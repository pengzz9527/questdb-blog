---
title: "QuestDB Python Client in Practice: Fast ILP Writes and pandas Ingestion"
description: "Writing to QuestDB from Python services is the most common integration question. Three paths: the official questdb client over TCP batch writes, the InfluxDB-compatible 9092 HTTP port, and bulk pandas DataFrame backfills. Covers batching, retry and dedup on reconnect, timestamp-unit pitfalls, and a complete safe-write pattern."
date: 2026-11-15
lastmod: 2026-11-15
slug: questdb-python-client-ilp
tags:
  - QuestDB
  - Python
  - ILP
  - pandas
  - Ingestion
categories:
  - Getting Started
toc: true
---

# QuestDB Python Client in Practice: Fast ILP Writes and pandas Ingestion

You cannot `curl` on every single write from a Python service. There are three paths for writing data into QuestDB, in recommended order:

| Path | Protocol / port | Batching | Use when |
|---|---|---|---|
| Official `questdb` client | TCP 9009 | built-in buffer + backpressure | high-frequency online writes |
| InfluxDB-compatible HTTP | 9092 | manual line joining | scripts, cross-language, zero-dependency |
| Raw socket | TCP 9009 | fully self-managed | you need to control flush cadence |

## 1. The official client: first choice

```bash
pip install questdb
```

The official Python client is a Rust binding with a built-in write buffer. Three lines get you writing:

```python
from questdb import QuestDB

with QuestDB("127.0.0.1", 9009) as qdb:
    qdb.write_line("cpu,host=web1,dc=osu usage=0.575 1581020590000000000")
    qdb.write_lines([
        "cpu,host=web2 usage=0.630 1581020591000000000",
        "cpu,host=web2 usage=0.580 1581020592000000000",
    ])
```

Two points that matter:

- **`write_lines` beats row-by-row `write_line`**: pass a list in one call and the client batches the network round-trip. On local Docker, a batch of 1,000 lines is an order of magnitude faster than a single `write_line` per row.
- **Context manager** closes the connection on exit. For long-running services, keep the connection and write in the background; only reconnect on failure.

## 2. The HTTP path: the 9092 compatible port

No dependency, or you are in an HTTP-only environment (Serverless, edge functions) — QuestDB's InfluxDB-compatible port works as-is:

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

The InfluxDB 2.x style is also supported: `POST /api/v2/write?org=org&bucket=metrics&precision=ns`. Watch out: the `precision` parameter must match the unit of the timestamps in the line protocol. The most common failure is a nanosecond timestamp interpreted as milliseconds — your data lands in the year 58,000 AD.

## 3. Bulk pandas backfills

Historical backfills (a full year of metering CSVs, for example) are best done with DataFrames. The official client's `write_df` turns a DataFrame into batched ILP:

```python
import pandas as pd
from questdb import QuestDB

df = pd.read_csv("metering_2025.csv", parse_dates=["ts"])

with QuestDB("127.0.0.1", 9009) as qdb:
    for i in range(0, len(df), 100_000):
        qdb.write_df(df.iloc[i:i + 100_000], ts_column="ts")
```

Prerequisites: the timestamp column must be `datetime64[ns]` (which is what `parse_dates` gives you by default); a 100k-row chunk is the sweet spot between memory pressure and batch efficiency — too big and single-batch memory spikes, too small and round-trips stop amortizing.

## 4. Three batching rules

1. **Never one row per request**: every HTTP request carries round-trip latency; at high frequency the network is the bottleneck. TCP batching amortizes it to near zero.
2. **One timestamp unit per table**: nanoseconds everywhere, or milliseconds everywhere. Mixing them is data disorder.
3. **Cap line length**: ILP single lines have a size limit (the write buffer); high-cardinality metrics should be split across lines.

## 5. Failure handling: retries and dedupe

ILP is fire-and-forget — there is no per-row ack. After a network blip and reconnect, a retry may have sent the same batch twice. So keep a deterministic dedupe rule on the read side:

```sql
-- on duplicate ts, keep the first row
SELECT ts, symbol, FIRST(price) AS price
FROM ticks
GROUP BY ts, symbol;
```

On the write side, the "in-memory buffer + retry on failure + resend after reconnect" pattern:

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

And use **event time** for timestamps, not arrival time: a gateway clock drifting by minutes means out-of-order writes (QuestDB accepts them but flags the table). NTP alignment is the cheap, one-time fix.

## 6. Pitfall list

- Row-by-row writes without batching — latency-bound, throughput collapses.
- Mixing ns/ms units inside one table.
- The DataFrame `ts` column is not `datetime64[ns]` — `write_df` errors or truncates precision.
- No connection resilience — one network blip kills the process.
- Arrival-time timestamps — clock drift creates out-of-order writes.

## 7. What next

- Write concurrency and memory knobs → our ILP high-concurrency tuning article.
- Batch consumption from message queues → our QuestDB + Kafka pipeline article.
- Candles from backtest data → our tick-to-K-line article.

## References

- [QuestDB Python client (PyPI)](https://pypi.org/project/questdb/)
- [QuestDB docs: INSERT / Line Protocol](https://questdb.io/docs/reference/sql/insert/)

## Related reading

- [QuestDB ILP Ingestion Complete Guide: From Single Rows to High-Throughput Batch](/en/post/questdb-ilp-ingestion-guide/)
- [Power Load Curve Analysis: QuestDB Metering Data and Peak Demand](/en/post/questdb-power-load-analysis/)
- [QuestDB ILP High-Concurrency Write Tuning: Workers, Batching & Memory Benchmarks](/en/post/questdb-ilp-hi-concurrency-tuning/)
