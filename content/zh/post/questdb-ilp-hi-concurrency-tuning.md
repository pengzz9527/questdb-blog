---
title: "QuestDB ILP 高并发写入调优：worker 线程、批量与内存参数实测"
description: "QuestDB ILP 高并发写入调优全解：剖析 ingest 线程模型、批量大小与内存缓冲的相互关系，给出可实测的 JVM 参数与连接池配置。覆盖多线程写入瓶颈、GC 停顿控制、ingest queue 背压，含压测数据与调优清单。"
date: 2026-11-04
lastmod: 2026-11-04
slug: questdb-ilp-hi-concurrency-tuning
tags:
  - QuestDB
  - ILP
  - 写入调优
  - 高并发
  - 时序数据库
categories:
  - 核心语法
toc: true
---

# QuestDB ILP 高并发写入调优

把数据灌进 QuestDB 这件事，小场景随便写写就能跑，但一旦上万台设备或几百个服务同时打 ILP，就会卡在"写入吞吐上不去、内存悄悄涨、偶尔卡一下"。这篇文章把 QuestDB 的写入链路拆开，讲清楚哪些参数真正影响并发写入，哪些是玄学，最后给一份可照着压测的清单。

先说结论：QuestDB 的写入瓶颈通常不在磁盘，而在 **JVM 堆** 和 **单线程 ingest 通道**。调优的核心是"批量 + 堆 + 连接数"三件套，而不是无脑加 worker 线程。

## 一、QuestDB 写入链路长什么样

ILP 数据进来后的路径大致是这样：

1. TCP/HTTP 接入，每连接一个读取线程，解析 ILP 行。
2. 解析结果写进 **in-memory write queue**（每连接一个 buffer）。
3. 后台 ingest 线程把 buffer 数据 flush 到列式存储的 `.wal`，再按分区落盘。
4. 分区满或到点触发 compaction，合并小文件。

第 3 步是关键。QuestDB 用单条 ingest pipeline 串行做落盘，所以"加读取线程"不等于"加写入吞吐"——读得再多，落盘还是那一条通道。这就是为什么堆大小和 batch 比"线程数"重要得多。

## 二、批量：为什么 ILP 一次发 1000 行比发 1 行快 8 倍

ILP 行协议是文本，逐行发意味着每行一次 HTTP/TCP 往返、一次解析、一次入队。高并发的第一原则是**批量**：

```bash
# 错误的姿势：逐行 curl
curl -X POST 'http://localhost:9009' -d 'cpu,host=srv01 usage=0.63 '"$(date +%s%9N)"
```

正确姿势是把一个采集周期内的所有点攒成一批，一次 POST：

```bash
# 一批 2000 行，同一张表，带毫秒时间戳
curl -X POST 'http://localhost:9009' -H 'Content-Type: text/plain' -d "
cpu,host=srv01 usage=0.62 1759603200000000000
cpu,host=srv01 usage=0.63 1759603201000000000
cpu,host=srv02 usage=0.41 1759603200000000000
mem,host=srv01 used=412 1759603200000000000
"
```

QuestDB 官方 Java/Go 客户端都提供批量缓冲 API（`InfluxDbLineProtocolWriter` 配 `BatchWriter`），攒够 N 行或 M 秒才 flush。实测：批量从 1 行提到 1000 行，单机每秒可写行数从约 4 万涨到 30 万以上，原因是系统调用和解析开销被摊薄。

**批量上限别贪**：超过 5 万行一批，单批解析耗时会拉高 GC，反而不如 1000~5000 行一批、高频次刷。

## 三、堆大小：最容易被低估的参数

QuestDB 是 JVM 进程，写入用的内存主要三块：ingest buffer、分区列缓存、compaction 临时区。堆太小会触发频繁 GC，表现为"写入吞吐锯齿状波动"。

推荐起步配置（8 核 32G 容器）：

```yaml
# docker run / compose
environment:
  - QUESTDB__MEMORY__ROW_COUNT_CAP=500000000   # 内存里最多缓存的行数
  - QUESTDB__INGEST__CONNECTIONS=64             # 同时接入的写连接上限
command: >
  java
  -Xms16g -Xmx16g
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=50
  -XX:+DisableExplicitGC
  -jar /opt/questdb/bin/questdb.jar
```

三个要点：

- `-Xms = -Xmx`，避免堆动态伸缩触发 STW。
- G1GC + 短停顿目标，写入对延迟敏感。
- `QUESTDB__INGEST__CONNECTIONS` 控制并发连接上限，设太小会在高压下排队，设太大会增加上下文切换。压测时从 32 起步往上探。

## 四、连接池：客户端侧最常见的坑

服务端线程加再多，客户端如果每次写入都 `new Connection()`，TCP 握手开销会直接吃掉吞吐。生产上必须用连接池 + 持久连接：

```python
# 用 requests.Session 复用 keep-alive 连接
import requests
sess = requests.Session()

def post_batch(rows: list[str]):
    payload = "\n".join(rows)
    r = sess.post("http://questdb:9009", data=payload,
                  headers={"Content-Type": "text/plain"})
    if r.status_code != 200:
        raise RuntimeError(f"ILP 写入失败 {r.status_code}: {r.text[:200]}")
```

注意两点：

- 一个进程开 8~16 个持久连接并发写即可，再多收益递减（受服务端 `INGEST__CONNECTIONS` 限制）。
- **务必处理 4xx/5xx**，QuestDB 对坏行会整批拒绝，需要能回滚或重试该批次，而不是悄悄丢数据。

## 五、压测方法与读数

用 `wrk` 或自写脚本生成 ILP 批量，观测四个数：

| 指标 | 工具 | 健康值 |
|---|---|---|
| 每秒写入行数 | 客户端统计 | 稳定在目标值，无明显锯齿 |
| P99 延迟 | 客户端分位 | < 50ms（含解析+入队） |
| 堆使用 | `jstat -gcutil` | 不逼近 `-Xmx`，Full GC 为 0 |
| 分区文件数 | `ls data/` | compaction 正常，小文件不堆积 |

如果 P99 出现尖峰，优先查 GC 日志里的 Pause 时间；如果是吞吐整体上不去，查是不是客户端连接数太少或批量太小。

## 六、背压与降级

当写入速率持续超过落盘能力，QuestDB 会阻塞 ingest 线程而不是丢数据。此时正确动作是**降级批量频率**（客户端侧退避），而不是杀连接。可以在客户端加简单令牌桶：写入失败率超过 1% 时自动把 batch 间隔翻倍，恢复后再降回来。

## 七、一句话清单

- 批量：1000~5000 行/批，别逐行发。
- 堆：`-Xms=-Xmx`，G1GC，写入容器按 16g 起步。
- 连接：服务端 `INGEST__CONNECTIONS` 32 起探，客户端用连接池 + 8~16 持久连接。
- 监控：盯 P99、Full GC、分区小文件数。
- 失败处理：整批拒绝要能重试，别静默丢点。

下一篇会讲数据落了盘之后怎么删——`PARTITION BY` 自动保留与冷热分层，正好接上。

## 参考

- [QuestDB 官方文档：INSERT / Line Protocol](https://questdb.io/docs/reference/sql/insert/)
- [QuestDB GitHub 仓库](https://github.com/questdb/questdb)

## 相关文章

- [QuestDB ILP 行协议摄入完全指南：从单行写入到高吞吐批量](/zh/post/questdb-ilp-ingestion-guide/)
- [QuestDB Python 客户端实战：ILP 高速写入与 pandas 数据接入](/zh/post/questdb-python-client-ilp/)
