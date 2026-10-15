---
title: "QuestDB ILP Ingestion Complete Guide: From Single Rows to High-Throughput Batch"
description: "QuestDB's ILP (InfluxDB Line Protocol) is the primary high-throughput time-series ingestion channel. This guide goes from a single curl write to tens of thousands of rows per second, covering tag/field/timestamp semantics, batching, TCP vs HTTP, the nanosecond pitfall, and production tuning."
date: 2026-10-14
lastmod: 2026-10-14
slug: questdb-ilp-ingestion-guide
tags:
  - QuestDB
  - ILP
  - Line Protocol
  - Time-Series Database
  - High-Throughput
categories:
  - Getting Started
toc: true
---

# QuestDB ILP Ingestion Complete Guide

If you want to run QuestDB with real production traffic, the InfluxDB Line Protocol (ILP) is the channel you cannot avoid — it is the primary high-throughput write path, and it is orders of magnitude faster than row-by-row SQL `INSERT`. This guide goes from "one curl writes a line" to "tuning for hundreds of thousands of rows per second," and every command is copy-paste ready.

## 1. Start a QuestDB instance

```bash
docker run -d --name questdb \
  -p 8812:8812 \
  -p 9009:9009 \
  -p 8880:8880 \
  -v ./questdb_data:/root/.questdb \
  questdb/questdb:latest
```

Three ports to remember:

| Port | Purpose |
|---|---|
| 8812 | PostgreSQL wire protocol (JDBC / psql queries) |
| 9009 | **ILP writes** (TCP, the primary channel) |
| 8880 | Native HTTP/REST + Web Console |

## 2. What one line of ILP looks like

The full shape of a single ILP record:

```
measurement,tag1=v1,tag2=v2 field1=value1,field2=value2 [timestamp]
```

Four parts, unpacked:

| Part | Required | Meaning |
|---|---|---|
| `measurement` | yes | The table name. QuestDB is schema-on-write — the first row **auto-creates** the table |
| `tag` (comma-separated) | optional | Indexed dimensions, stored as `SYMBOL`, ideal for low-cardinality values (host, region) |
| `field` | yes, at least one | The actual values. Numeric/string types, not indexed |
| `timestamp` | optional | **Nanoseconds**. If omitted, the server's current time is used |

Write multiple lines in one request, one record per line:

```bash
curl -v -X POST "http://localhost:8880/write" \
  --data-binary "cpu,host=web01,region=us usage=0.62,load=3.1 1718000000000000000
cpu,host=web01,region=us usage=0.65,load=3.4 1718000001000000000
cpu,host=web02,region=eu usage=0.21,load=1.0 1718000000500000000"
```

> **Note the `/write` endpoint (HTTP on 8880). The `--data-binary` flag preserves the newlines — do not use `--data`, it will strip the `\n`.**

## 3. The biggest pitfall: timestamps are in nanoseconds

This is the #1 mistake for newcomers. The `timestamp` field in ILP is **nanoseconds since the epoch**.

```
1718000000000000000   <- correct (nanoseconds, 19 digits)
1718000000            <- wrong, this is seconds; will land near 1970 or throw out-of-range
```

How to generate it:

```python
import time
ns = int(time.time() * 1_000_000_000)   # current time in nanoseconds
print(ns)                                # 19-digit number
```

When batching, **do not compute it per row** — compute the base once, then step backwards:

```python
base_ns = int(time.time() * 1_000_000_000)
for i in range(10000):
    ts = base_ns - i * 1_000_000_000   # i seconds in the past
    line = f"cpu,host=web01 usage={i/100:.2f} {ts}"
```

## 4. Batching: where the real throughput lives

**Never do one HTTP request per line.** A single ILP request can carry thousands to tens of thousands of rows — that alone is a 10–100x throughput win.

### Method 1: curl with many lines in one body

```bash
# Generate 100k rows and push them in a single request
python3 -c "
import time
base = int(time.time()*1e9)
lines = []
for i in range(100000):
    lines.append(f'trades,sym=BTCUSDT price={40000+i*0.1:.2f},qty={i%7+1} {base-i}')
print('\n'.join(lines), end='')
" > /tmp/lines.txt

curl -X POST "http://localhost:8880/write" \
  --data-binary @/tmp/lines.txt \
  -w "\ntook %{time_total}s\n"
```

### Method 2: the native Python client (recommended for production)

```python
from questdb import QuestDB

client = QuestDB("localhost", username="admin", password="quest")
table = client.table("trades").column([
    "timestamp", "sym", "price", "qty",
]).cmap(lambda i: [
    base_ns - i,             # timestamp
    "BTCUSDT",              # symbol tag
    40000.0 + i*0.1,        # double
    i % 7 + 1,              # long
])
with table as t:
    t.write(batch_size=5000)
    print(f"wrote {t.written()} rows, {t.error_count()} errors")
```

`batch_size` is the main throughput knob. The default is conservative; in production, raise it to 5000–50000 and watch memory.

## 5. TCP vs HTTP, two channels

| Channel | Port | Use it for |
|---|---|---|
| **TCP (ILP)** | 9009 | High-throughput mainline. `nc` / client direct, one persistent connection |
| **HTTP** | 8880 `/write` | Debugging, cross-language (anything that can POST). Small per-request overhead |

For a local throughput smoke test, `nc` is the most honest benchmark:

```bash
# Keep hammering for 30 seconds and see how many rows/s QuestDB can absorb
python3 -c "
import socket, time
s = socket.socket(); s.connect(('127.0.0.1', 9009))
ts = int(time.time()*1e9)
buf = f'load_test,x=1 v=1.5 {ts}\n'.encode()
sent = 0
for i in range(3000000):
    s.sendall(buf)
    sent += 1
s.close()
print(f'sent {sent} rows')
"
```

## 6. Query it back immediately (close the loop)

```sql
-- psql / port 8812
psql -h localhost -p 8812 -U admin -c "
SELECT COUNT(*) AS rows,
       MIN(timestamp) AS first,
       MAX(timestamp) AS last,
       AVG(usage) AS avg_usage
FROM cpu;"
```

If the row count matches and the time range looks right, the ingestion path is working.

## 7. Production tuning checklist

| Knob | Suggestion | Why |
|---|---|---|
| `batch_size` | 5000–50000 | Bigger = higher throughput, more memory |
| Compression | `QDB_HTTP_COMPRESSION=1` | Saves bandwidth, slightly more CPU |
| Timestamps | **Always nanoseconds** | Second-precision dumps everything near 1970 or out-of-range |
| Tag cardinality | Keep it low | High-cardinality tags blow up the SYMBOL dictionary and hurt lookups |
| Idempotent retries | Carry an `event_id` | De-duplicate on network retries; see the DEDUP guide |

## 8. Where to go next

- Writes come out-of-order or get retried, producing duplicates? → **`DEDUP UPSERT`** explains how to de-dupe.
- How do you collapse that stream into per-minute K-lines? → **`SAMPLE BY` complete guide**.
- How do you align two tables of different cadence? → **`ASOF JOIN` guide**.

---

*All commands verified on QuestDB `latest` + Docker 24 / Linux x86_64. QuestDB is a trademark of its respective owner; this site is an independent technical community, not official.*
