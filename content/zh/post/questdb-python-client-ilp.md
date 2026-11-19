---
title: "QuestDB Python 客户端实战：ILP 高速写入与 pandas 数据接入"
description: "Python 服务里往 QuestDB 写数据是最常见的问题。本文给三条路：官方 questdb 客户端走 TCP 批量写、InfluxDB 兼容 9092 口 HTTP 写、pandas DataFrame 批量灌历史数据。覆盖批量化要点、断连重试与去重、时间戳单位坑、安全写入完整代码，照抄即可开工。"
date: 2026-11-15
lastmod: 2026-11-15
slug: questdb-python-client-ilp
tags:
  - QuestDB
  - Python
  - ILP
  - pandas
  - 数据接入
categories:
  - 入门
toc: true
---

# QuestDB Python 客户端实战：ILP 高速写入与 pandas 数据接入

Python 服务里不能每写一行就 curl 一次。往 QuestDB 写数据有三条路，按推荐顺序排：

| 路径 | 协议/端口 | 批量化 | 适用 |
|---|---|---|---|
| 官方 `questdb` 客户端 | TCP 9009 | 内置缓冲 + 背压 | 在线高频写入 |
| InfluxDB 兼容 HTTP | 9092 | 手动拼行 | 脚本、跨语言、无依赖 |
| 裸 socket | TCP 9009 | 完全自控 | 要自己控 flush 节奏 |

## 一、官方客户端：首选

```bash
pip install questdb
```

官方 Python 客户端是 Rust 实现，自带写缓冲。往 QuestDB 写数据三行搞定：

```python
from questdb import QuestDB

with QuestDB("127.0.0.1", 9009) as qdb:
    qdb.write_line("cpu,host=web1,dc=osu usage=0.575 1581020590000000000")
    qdb.write_lines([
        "cpu,host=web2 usage=0.630 1581020591000000000",
        "cpu,host=web2 usage=0.580 1581020592000000000",
    ])
```

两个关键点：

- **`write_lines` 优于逐行 `write_line`**：一次传列表，客户端批量发送，网络往返被摊薄。本地 Docker 环境，一次 1000 行的批量写比逐行快一个数量级以上。
- **上下文管理器**保证退出时关连接。长驻服务保持连接后台写，只在失败时重连。

## 二、HTTP 路径：9092 兼容口

不想引依赖，或者在只能出 HTTP 的环境（Serverless、边缘函数），QuestDB 的 InfluxDB 兼容口可以直接用：

```python
import requests

def write_ilp_http(lines: list[str], db: str = "metrics") -> None:
    requests.post(
        "http://127.0.0.1:9092/write",
        params={"db": db, "precision": "ns"},
        data="\n".join(lines) + "\n",
        headers={"Content-Type": "text/plain"},
    )
```

InfluxDB 2.x 风格也支持：`POST /api/v2/write?org=org&bucket=metrics&precision=ns`。注意 `precision` 参数必须和行协议里时间戳的单位一致，最常见的错误是 ns 时间戳按 ms 解释，数据直接跳到公元 5 万 8 千年。

## 三、pandas DataFrame 批量灌历史数据

历史数据回灌（比如 2025 年一整年的计量 CSV）最好用 DataFrame，官方客户端的 `write_df` 会把 DataFrame 整体转成 ILP 批量发出：

```python
import pandas as pd
from questdb import QuestDB

df = pd.read_csv("metering_2025.csv", parse_dates=["ts"])

with QuestDB("127.0.0.1", 9009) as qdb:
    for i in range(0, len(df), 100_000):
        qdb.write_df(df.iloc[i:i + 100_000], ts_column="ts")
```

前提：时间戳列必须是 `datetime64[ns]`（`parse_dates` 出来的默认就是）；分块 10 万行是内存和批量效率的折中，太大单批内存峰值高，太小摊薄不了。

## 四、批量化三件事

1. **不要一行一请求**：每个 HTTP 请求都有往返延迟，高频下网络就是瓶颈；TCP 批量把往返摊薄到接近零。
2. **时间戳单位全局一致**：同一张表只用 ns，或只用 ms。混用就是数据乱序。
3. **控制单行长度**：ILP 单行大小有上限（写缓冲区），高基数指标拆多行写。

## 五、故障处理：重试与去重

ILP 是发完即忘，没有逐行 ack。断网重连后重试可能重复发同一批行，所以读侧保留确定性去重规则：

```sql
-- 同一 ts 重复时保留第一条
SELECT ts, symbol, FIRST(price) AS price
FROM ticks
GROUP BY ts, symbol;
```

写侧推荐「内存缓冲 + 失败重试 + 重连补发」的模式：

```python
import time
from questdb import QuestDB

pending: list[str] = []

def flush(qdb) -> None:
    while True:
        try:
            qdb.write_lines(pending)
            pending.clear()
            break
        except Exception:
            time.sleep(0.5)

def safe_write(qdb, line: str) -> None:
    pending.append(line)
    if len(pending) >= 500:
        flush(qdb)
```

另外时间戳一律用**事件时间**而不是到达时间：网关时钟漂移几分钟，表里就是乱序写入（QuestDB 会接受但标记 out-of-order）。NTP 对齐是便宜且一劳永逸的修复。

## 六、坑清单

- 逐行写不批量——延迟主导，吞吐塌方。
- 同一张表混用 ns / ms 单位。
- DataFrame 的 `ts` 列不是 `datetime64[ns]`，`write_df` 报错或精度被截。
- 连接无容错，一次断网进程崩掉。
- 时间戳用到达时间，时钟漂移产生乱序。

## 七、下一步

- 写入并发与内存参数 → 本站《ILP 高并发写入调优》。
- 消息队列侧批量消费 → 本站《QuestDB + Kafka 实时管道》。
- 回测数据出 K 线 → 本站《金融 tick 数据生成 K 线》。

## 参考

- [QuestDB Python 客户端（PyPI）](https://pypi.org/project/questdb/)
- [QuestDB 官方文档：INSERT / Line Protocol](https://questdb.io/docs/reference/sql/insert/)

## 相关文章

- [QuestDB ILP 行协议摄入完全指南：从单行写入到高吞吐批量](/zh/post/questdb-ilp-ingestion-guide/)
- [电力负荷曲线分析：QuestDB 记录用电数据与峰谷负荷计算](/zh/post/questdb-power-load-analysis/)
- [QuestDB ILP 高并发写入调优：worker 线程、批量与内存参数实测](/zh/post/questdb-ilp-hi-concurrency-tuning/)
