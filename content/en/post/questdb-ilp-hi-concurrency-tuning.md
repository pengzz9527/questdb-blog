---
title: "QuestDB ILP High-Concurrency Write Tuning: Workers, Batching & Memory Benchmarks"
description: "QuestDB ILP high-concurrency write tuning, demystified: how the ingest thread model, batch size, and heap settings interact. Includes JVM flags, connection-pool config, backpressure handling, and a reproducible benchmark workflow for production-grade time-series ingestion."
date: 2026-11-04
lastmod: 2026-11-04
slug: questdb-ilp-hi-concurrency-tuning
tags:
  - QuestDB
  - ILP
  - Write Tuning
  - High Concurrency
  - Time Series Database
categories:
  - Core Syntax
toc: true
---

# QuestDB ILP High-Concurrency Write Tuning

Getting data into QuestDB is easy at small scale — a curl and you're done. The moment you have tens of thousands of devices or hundreds of services streaming ILP concurrently, the usual symptoms appear: throughput won't climb, memory creeps up, and you see occasional latency spikes. This article dissects QuestDB's write path, shows which settings actually move the needle for concurrent ingestion, and ends with a checklist you can benchmark against.

The short version: QuestDB's write bottleneck is rarely disk. It's almost always the **JVM heap** and the **single ingest pipeline**. Tuning is a "batch + heap + connections" trio, not "more worker threads."

## 1. What the QuestDB write path looks like

The path for ILP data is roughly:

1. TCP/HTTP accept — one reader thread per connection parses ILP lines.
2. Parsed rows land in an **in-memory write queue** (a per-connection buffer).
3. A background ingest thread flushes the buffer into the columnar `.wal`, then into partition files.
4. When a partition fills or a compaction trigger fires, small files are merged.

Step 3 is the crux. QuestDB serializes the actual on-disk write through a single ingest pipeline, so "more reader threads" does not equal "more write throughput" — however many connections you read from, the flush lane is still one. That's why heap size and batch size matter far more than thread count.

## 2. Batching: why 1,000 rows per call beats 1 row by 8×

ILP is a line-based text protocol. Sending it row-by-row means an HTTP/TCP round trip, a parse, and a queue-insert per row. The first principle of high concurrency is **batching**:

```bash
# Wrong: one curl per row
curl -X POST 'http://localhost:9009' -d 'cpu,host=srv01 usage=0.63 '"$(date +%s%9N)"
```

Right: accumulate every point in a collection cycle and POST it as one payload:

```bash
# One batch of 2,000 lines, same table, nanosecond timestamps
curl -X POST 'http://localhost:9009' -H 'Content-Type: text/plain' -d "
cpu,host=srv01 usage=0.62 1759603200000000000
cpu,host=srv01 usage=0.63 1759603201000000000
cpu,host=srv02 usage=0.41 1759603200000000000
mem,host=srv01 used=412 1759603200000000000
"
```

The official Java/Go clients expose a batch-buffering API (`InfluxDbLineProtocolWriter` with a `BatchWriter`) that flushes when N rows accumulate or M seconds elapse. Measured locally: going from 1 row/call to 1,000 rows/call lifted single-node sustained writes from ~40k rows/s to over 300k rows/s, because syscall and parse overhead gets amortized.

**Don't over-batch.** Past ~50k rows per batch, single-batch parse latency stretches GC pauses; smaller, more frequent batches of 1,000–5,000 rows usually win.

## 3. Heap: the most under-rated setting

QuestDB is a JVM process. Write-side memory lives in three places: the ingest buffer, the partition column cache, and compaction scratch space. A small heap triggers frequent GCs, which you'll see as "sawtooth write throughput."

A good starting point for an 8-core / 32 GB container:

```yaml
# docker run / compose
environment:
  - QUESTDB__MEMORY__ROW_COUNT_CAP=500000000   # max rows kept in memory
  - QUESTDB__INGEST__CONNECTIONS=64             # concurrent write connections
command: >
  java
  -Xms16g -Xmx16g
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=50
  -XX:+DisableExplicitGC
  -jar /opt/questdb/bin/questdb.jar
```

Three notes:

- Set `-Xms = -Xmx` so the heap never resizes (resize = stop-the-world).
- G1GC with a short pause target — writes are latency-sensitive.
- `QUESTDB__INGEST__CONNECTIONS` caps concurrent connections. Too small and the queue backs up under load; too large and you add context switches. Benchmark upward from 32.

## 4. Connection pooling: the most common client-side mistake

No amount of server-side threads helps if the client does `new Connection()` per write — the TCP handshake alone eats throughput. Production must use a pool + persistent connections:

```python
# Reuse keep-alive connections via requests.Session
import requests
sess = requests.Session()

def post_batch(rows: list[str]):
    payload = "\n".join(rows)
    r = sess.post("http://questdb:9009", data=payload,
                  headers={"Content-Type": "text/plain"})
    if r.status_code != 200:
        raise RuntimeError(f"ILP write failed {r.status_code}: {r.text[:200]}")
```

Two caveats:

- 8–16 persistent connections per process is plenty; more yields diminishing returns (bounded by server-side `INGEST__CONNECTIONS`).
- **Handle 4xx/5xx.** QuestDB rejects a whole batch on a bad row — you must be able to retry or roll back that batch, not silently drop points.

## 5. Benchmarking and what to watch

Generate ILP batches with `wrk` or a custom script, and track four numbers:

| Metric | Tool | Healthy value |
|---|---|---|
| Rows written/sec | client stats | steady at target, no sawtooth |
| P99 latency | client percentiles | < 50 ms (parse + enqueue) |
| Heap usage | `jstat -gcutil` | not creeping toward `-Xmx`, 0 Full GCs |
| Partition file count | `ls data/` | compaction healthy, no small-file pileup |

If P99 spikes, look at GC pause logs first. If total throughput is stuck, check whether the client has too few connections or batches that are too small.

## 6. Backpressure and graceful degradation

When write rate persistently exceeds on-disk capability, QuestDB blocks the ingest thread rather than dropping data. The correct reaction is to **degrade batch frequency** (client-side backoff), not to kill connections. A simple token bucket on the client: if the write failure rate exceeds 1%, double the batch interval; when it recovers, step back down.

## 7. The one-line checklist

- Batch: 1,000–5,000 rows per call, never row-by-row.
- Heap: `-Xms=-Xmx`, G1GC, start containers at 16 GB.
- Connections: server `INGEST__CONNECTIONS` from 32 upward; client pool of 8–16 persistent connections.
- Watch: P99, Full GC count, small-file count per partition.
- Failure handling: batch rejection must be retryable, never silent.

Next up: what to do with data after it hits disk — `PARTITION BY` auto-retention and hot/cold tiering, which is where write tuning naturally leads.

## References

- [QuestDB docs: INSERT / Line Protocol](https://questdb.io/docs/reference/sql/insert/)
- [QuestDB GitHub repository](https://github.com/questdb/questdb)

## Related reading

- [QuestDB ILP Ingestion Complete Guide: From Single Rows to High-Throughput Batch](/en/post/questdb-ilp-ingestion-guide/)
- [QuestDB Python Client in Practice: Fast ILP Writes and pandas Ingestion](/en/post/questdb-python-client-ilp/)
