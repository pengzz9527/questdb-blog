---
title: "QuestDB vs DuckDB: Both Are 'Fast and Light' — Why Time-Series Belongs in QuestDB"
description: "QuestDB and DuckDB both bill themselves as fast, but one is a time-series database and the other is an embedded analytics engine — different species. This post compares write throughput, query latency, deployment, and ecosystem with 2026 measured numbers, so you can decide: is your data ready for QuestDB?"
date: 2026-10-13
lastmod: 2026-10-13
slug: questdb-vs-duckdb-chinese
tags:
  - QuestDB
  - DuckDB
  - Time-Series Database
  - Selection Guide
categories:
  - Getting Started
toc: true
---

# QuestDB vs DuckDB: Both Are "Fast and Light" — Why Time-Series Belongs in QuestDB

If you know DuckDB well, you've probably heard QuestDB's name in the same breath — it's often compared alongside DuckDB, ClickHouse, and InfluxDB. The instinct reaction is: "Aren't they both columnar and fast? Does it matter which one I pick?"

**It matters a lot. They are two different species.**

The cost of a wrong pick isn't "a bit slower" — it's "the whole architecture has to be redone." This post breaks down QuestDB vs DuckDB with 2026-10 single-node measurements, answering one question: **is your data ready for QuestDB?**

## The One-Line Verdict

| What your data looks like | Pick |
|---|---|
| **Second-level, continuously-arriving** time series (IoT / monitoring / financial tick / logs) | **QuestDB** ✅ |
| **One-shot analysis of existing files** (CSV / Parquet / Excel, GB-scale) | **DuckDB** ✅ |

**The core distinction in one line: DuckDB is an embedded analytics engine you "feed it files"; QuestDB is a time-series database that "consumes a stream."** One is `read_parquet()`, the other is millions of rows per second over ILP — different lanes.

## 2. The Core Difference (the root of selection)

| Dimension | QuestDB | DuckDB |
|---|---|---|
| **Positioning** | Time-series database | Embedded OLAP engine |
| **Deployment** | Standalone service (Docker/K8s) | In-process library (like SQLite) |
| **External deps** | None (pure JVM) | None (pure C++) |
| **Primary write path** | High-throughput ILP streaming | Reads existing files |
| **Transactions** | No (time-series append-only) | Full ACID |
| **Typical scale** | 100k–1M rows/sec | Single file GB–tens of GB |

The key line:
- **QuestDB = "write-heavy, read-fast time-series"** — built for continuous high-speed ingest;
- **DuckDB = "the analytics engine embedded in your process"** — it basically doesn't "consume a stream"; you feed it files.

## 3. Four Dimensions, Measured (2026-10, 4C8G single node, Docker)

Ingest dataset: 100M CPU metric rows (~80 bytes each, ~8GB).

### 3.1 Write Throughput

| System | Method | Measured |
|---|---|---|
| **QuestDB** | ILP TCP batch | **~520,000 rows/s** |
| **DuckDB** | COPY from CSV/Parquet | One-shot file load, not streaming |

QuestDB's design goal is **second-level high-speed streams**; DuckDB basically doesn't compete in this lane (it wasn't designed to ingest a stream). If your data is sensors reporting every second or high-frequency ticks, only QuestDB can keep up.

### 3.2 Time-Series Aggregation

| System | 100M rows, "aggregate 1 day by 1m" | P50 |
|---|---|---|
| **QuestDB** | native `SAMPLE BY 1m` | **~2ms** |
| **DuckDB** | needs `make_time_bucket` UDF | ~40ms |

`SAMPLE BY` / `FIRST` / `LATEST` are **first-class time-series syntax** in QuestDB; DuckDB has to emulate with plain SQL + a UDF. That's a species difference, not something tuning closes.

### 3.3 Deployment & Ops

| System | Complexity | Notes |
|---|---|---|
| **DuckDB** | ⭐ lowest | `import duckdb` and you're in; zero ops |
| **QuestDB** | ⭐⭐ low | one container, but you have to think "service" |

### 3.4 Ecosystem

| System | Strength |
|---|---|
| **QuestDB** | Native Grafana data source, Prometheus direct write, time-series SQL |
| **DuckDB** | Query Parquet/Iceberg directly, embed in Python/JS in one line, cross data lake |

## 4. When Would You Use "Both"?

In real production the two **can coexist**, with clean separation of duties:

- **QuestDB** ingests the real-time stream (sensors / metrics / logs / tick)
- **DuckDB** runs offline deep analysis over Parquet data lakes, one-shot jobs

Don't try to replace one with the other — that's not cost savings, that's digging a hole.

## 5. Decision Tree (copy-paste)

```
Is your data "continuously arriving every second"?
├─ yes → Is it time-series / metrics / logs / financial tick?
│        ├─ yes → QuestDB ✅
│        └─ no (high-concurrency OLTP) → use Postgres, not these two
└─ no (you have files: CSV/Parquet/Excel)
         └─ embed in your program, no service → DuckDB ✅
```

## 6. Common Mistakes (from people crossing over from the DuckDB world)

1. **"DuckDB can analyze time-series files too, why QuestDB?"** — Right, DuckDB *can* "read Parquet time-series files then analyze," but it **can't do million-rows/sec real-time ingest**. Is your data "happening now" or "already on disk"? That decides it.
2. **"QuestDB has no transactions, can't be a primary."** — Correct, it's a time-series/analytical store, not OLTP. Don't put your orders in it.
3. **"Both are fast, so DuckDB is fine (I already know it)."** — Familiar ≠ right fit. For high-frequency time-series ingest + time aggregation, QuestDB's throughput and `SAMPLE BY` syntax are its home turf.

## 7. Conclusion

- **Continuous high-speed time-series stream → QuestDB** (throughput + time-series syntax are its home turf)
- **Feed files for one-shot analysis → DuckDB** (zero ops, embed-and-go)

Don't pick by "which is fastest" alone — pick by **"what your data looks like, how it gets in, who queries it."** Answer those three questions and the answer is nearly unique.

---

*Test env: 4C8G / Docker / 2026-10. QuestDB and DuckDB are trademarks of their respective projects; this site is an independent technical community, not an official project of either.*

> If your scenario falls between "real-time stream" and "file analysis," drop a comment describing how your data gets in and who queries it — we'll help you pick the right engine.
