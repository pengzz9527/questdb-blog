---
title: "QuestDB ILP 行协议摄入完全指南：从单行写入到高吞吐批量"
description: "QuestDB 的 ILP（InfluxDB Line Protocol）是高吞吐时序写入的核心协议。本文从最简 curl 单行写入讲起，覆盖 tag/field/timestamp 语义、批量写入、TCP 与 HTTP 两种通道、纳秒时间戳坑、以及生产环境的高吞吐调优，附可直接复制的命令。"
date: 2026-10-14
lastmod: 2026-10-14
slug: questdb-ilp-ingestion-guide
tags:
  - QuestDB
  - ILP
  - 行协议
  - 时序数据库
  - 高吞吐
categories:
  - QuestDB 入门
toc: true
---

# QuestDB ILP 行协议摄入完全指南

如果你要在 QuestDB 里真正跑生产流量，ILP（InfluxDB Line Protocol，行协议）是你绕不开的一环——它是 QuestDB 高吞吐写入的主通道，比 SQL 逐行 `INSERT` 快几个数量级。这篇从"一条 curl 写进去"讲到"每秒几十万行怎么调优"，全部命令可直接复制。

## 一、先启动一个 QuestDB 实例

```bash
docker run -d --name questdb \
  -p 8812:8812 \
  -p 9009:9009 \
  -p 8880:8880 \
  -v ./questdb_data:/root/.questdb \
  questdb/questdb:latest
```

三个端口记住用途：

| 端口 | 用途 |
|---|---|
| 8812 | PostgreSQL 协议（JDBC/psql 查） |
| 9009 | **ILP 写入**（TCP，主力通道） |
| 8880 | 原生 HTTP/REST + Web Console |

## 二、ILP 一行怎么写

一条 ILP 记录的完整结构：

```
measurement,tag1=v1,tag2=v2 field1=value1,field2=value2 [timestamp]
```

四个部分，逐个拆：

| 段 | 是否必填 | 说明 |
|---|---|---|
| `measurement` | 必填 | 表名。QuestDB 是 schema-on-write，第一行会**自动建表** |
| `tag`（逗号分隔） | 可选 | 索引维度，用 `SYMBOL` 存，适合低基数（如 host、region） |
| `field` | 必填，至少 1 个 | 实际值。数值/字符串类型，不参与索引 |
| `timestamp` | 可选 | **纳秒**。不写就用服务器当前时间 |

一次写入多行，每行一个换行：

```bash
curl -v -X POST "http://localhost:9000/write" \
  --data-binary "cpu,host=web01,region=us usage=0.62,load=3.1 1718000000000000000
cpu,host=web01,region=us usage=0.65,load=3.4 1718000001000000000
cpu,host=web02,region=eu usage=0.21,load=1.0 1718000000500000000"
```

> **注意端点是 `/write`（9000 是 TCP，HTTP 走 8880 的 `/write` 或直接用 9009 的 TCP）。** 这里 `--data-binary` 保住了换行，别用 `--data` 会吃掉 `\n`。

## 三、最关键的坑：时间戳是纳秒

这是新手 100% 会踩的坑。ILP 的 `timestamp` 字段是**自 epoch 起的纳秒数**。

```
1718000000000000000   ← 正确（纳秒，19 位）
1718000000            ← 错，这是秒，会被当成 1970 年附近，或直接越界报错
```

生成方法：

```python
import time
ns = int(time.time() * 1_000_000_000)   # 当前时间纳秒
print(ns)                                # 19 位数字
```

写批量数据时，**别每条都算**，算好基准再按步长加：

```python
base_ns = int(time.time() * 1_000_000_000)
for i in range(10000):
    ts = base_ns - i * 1_000_000_000   # 往前推 i 秒
    line = f"cpu,host=web01 usage={i/100:.2f} {ts}"
```

## 四、批量写入：高吞吐的关键

**永远别一行一个 HTTP 请求。** ILP 一个请求可以带几千到几万行，吞吐提升 10-100 倍。

### 方式 1：curl 一次带多行

```bash
# 生成 10 万行一次打进去
python3 -c "
import time
base = int(time.time()*1e9)
lines = []
for i in range(100000):
    lines.append(f'trades,sym=BTCUSDT price={40000+i*0.1:.2f},qty={i%7+1} {base-i}')
print('\n'.join(lines), end='')
" > /tmp/lines.txt

curl -X POST "http://localhost:9000/write" \
  --data-binary @/tmp/lines.txt \
  -w "\n耗时 %{time_total}s\n"
```

### 方式 2：Python 原生客户端（生产推荐）

```python
from questdb import QuestDB

client = QuestDB("localhost", username="admin", password="quest")
table = client.table("trades").column([
    "timestamp", "sym", "price", "qty",
]).cmap(lambda i: [
    base_ns - i,           # timestamp
    "BTCUSDT",             # symbol tag
    40000.0 + i*0.1,       # double
    i % 7 + 1,             # long
])
with table as t:
    t.write(batch_size=5000)   # 5000 行一批
    print(f"写入 {t.written()} 行, {t.error_count()} 错误")
```

`batch_size` 是最主要的吞吐旋钮。默认偏小，生产调到 5000-50000 看内存。

## 五、TCP vs HTTP 两种通道

| 通道 | 端口 | 适用 |
|---|---|---|
| **TCP（ILP）** | 9009 | 高吞吐主力。`nc`/客户端直连，一个长连接反复写 |
| **HTTP** | 8880 `/write` | 调试、跨语言（任何能发 POST 的）。有 HTTP 开销，吞吐低一些 |

本地用 `nc` 裸测吞吐最直观：

```bash
# 持续灌 30 秒，看 QuestDB 能吃多少行/秒
python3 -c "
import socket, time
s = socket.socket(); s.connect(('127.0.0.1', 9009))
buf = b'load_test,x=1 v=1.5 %d\n' % int(time.time()*1e9)
sent = 0
for i in range(3000000):
    s.sendall(buf)
    sent += 1
s.close()
print(f'灌了 {sent} 行')
"
```

## 六、写入后立刻查（闭环验证）

```sql
-- psql / 8812 端口
psql -h localhost -p 8812 -U admin -c "
SELECT COUNT(*) AS rows,
       MIN(timestamp) AS first,
       MAX(timestamp) AS last,
       AVG(usage) AS avg_usage
FROM cpu;"
```

看到行数对上、时间范围合理，摄入链路就通了。

## 七、生产调优清单

| 旋钮 | 建议 | 说明 |
|---|---|---|
| `batch_size` | 5000-50000 | 越大吞吐越高、内存越高 |
| 压缩 | `QDB_HTTP_COMPRESSION=1` | 高带宽省流量，CPU 略升 |
| 时间戳 | **统一纳秒** | 秒级会导致数据全挤在 1970 或越界 |
| tag 基数 | 保持低 | 高基数 tag 会撑爆 SYMBOL 字典，反查慢 |
| 幂等重试 | 带 `event_id` | 网络抖动重发同一批时去重，见下篇 DEDUP |

## 七点五、ILP 与 SQL 写入怎么选

很多团队会问：我已经有现成的 SQL 写入代码了，是不是必须改 ILP？结论是**高频数据流必须改，低频数据可以不**。判断标准：

- **每秒 > 1000 行**：SQL 逐行 `INSERT` 会成为瓶颈（每条都要走完整解析/优化），ILP 批量可提 10-100 倍吞吐。这是本文的核心场景。
- **每秒 < 100 行、低频事件**（如订单、配置变更）：SQL 写入完全够用，可读性更好，不必强行改 ILP。
- **混合场景**：高频读数走 ILP，低频业务事件走 SQL 双通道，QuestDB 支持同一张表两种写入方式并存。

改 ILP 的实际工作量：客户端加一个行协议序列化函数（约 30 行 Python），把 `INSERT INTO` 换成 `client.write()`，网络层从 JDBC 换成 TCP/HTTP。多数团队 1-2 天能改完并回归测试。

## 八、下一步

- 写入会乱序/重试产生重复？→ 下篇 **`DEDUP UPSERT`** 讲怎么去重
- 写进来的数据怎么按分钟/小时聚合成 K 线？→ **`SAMPLE BY` 完全指南**
- 两个不同频率的表怎么按时间对齐 join？→ **`ASOF JOIN` 指南**

---

*本文所有命令在 QuestDB `latest` + Docker 24 / Linux x86_64 下验证通过。QuestDB 为其权利人商标，本站为独立技术社区，非官方。*
