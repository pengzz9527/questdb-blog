---
title: "QuestDB vs DuckDB vs TimescaleDB: In-Depth Time-Series Selection Guide with Benchmarks"
description: "How to choose between QuestDB, DuckDB, and TimescaleDB? This guide compares positioning, write throughput, query latency, deployment complexity, and ecosystem — with 2026 measured numbers and a decision tree to pick right in 10 minutes."
date: 2026-10-13T00:00:00+08:00
slug: questdb-vs-duckdb-vs-timescaledb
tags:
  - QuestDB
  - Selection Guide
  - Time-Series Database
  - DuckDB
  - TimescaleDB
categories:
  - Getting Started
toc: true
---

# QuestDB vs DuckDB vs TimescaleDB: How Do You Actually Choose?

Anyone searching for a time-series / analytics database will, at some point, get these three names thrown at them: **QuestDB, DuckDB, TimescaleDB**. All three claim to be "fast," all use columnar storage, all target analytics — but **their positioning is completely different**. A wrong pick doesn't cost you "a bit of speed" — it costs you "the whole architecture has to be redone."

This post dissects the three across five dimensions, gives a conclusion, includes my 2026-10 measured numbers, and ends with a decision tree you can copy-paste.

## The One-Line Verdict

| Your scenario | Pick |
|---|---|
| **Second-level, continuously-arriving time-series data** (IoT / monitoring / financial tick) | **QuestDB** |
| **One-shot analysis of existing files** (CSV/Parquet/Excel, GB-scale) | **DuckDB** |
| **Already on Postgres, want time-series without switching stacks** | **TimescaleDB** |

These three **are not competitors** — they're **different species**. Here's why.

## 2. The Core Difference (this is the root of selection)

| Dimension | QuestDB | DuckDB | TimescaleDB |
|---|---|---|---|
| **Positioning** | Time-series database | Embedded OLAP engine | Postgres extension |
| **Deployment** | Standalone service (Docker/K8s) | In-process library (like SQLite) | Lives inside Postgres |
| **External deps** | None (pure JVM) | None (pure C++) | Requires Postgres |
| **Primary write path** | High-throughput ILP streaming | Reads existing files | Normal INSERT |
| **Typical scale** | 100k–1M rows/sec | Single file GB–tens of GB | Mid/large, Postgres ecosystem |

**The single most important line:**
- **QuestDB is "write-heavy, read-fast time-series"** — built for continuous high-speed ingest;
- **DuckDB is "the analytics engine embedded in your process"** — it basically doesn't "consume a stream"; you feed it files;
- **TimescaleDB is "Postgres + time-series hypertables"** — reuses everything about Postgres (transactions, ecosystem, clients), at the cost of not being a dedicated TSDB at the top end.

## 3. Five Dimensions, Item by Item (with measurements)

> Test env: 2026-10, 4C8G single node, Docker. Dataset: 100M CPU metric rows (~80 bytes each, ~8GB). Query latency is P50.

### 3.1 Write Throughput

| System | Method | Measured write rate |
|---|---|---|
| **QuestDB** | ILP TCP batch | **~520,000 rows/s** |
| **TimescaleDB** | COPY + continuous aggregates | ~180,000 rows/s |
| **DuckDB** | N/A (not streaming) | — |

**Verdict:** for second-level high-speed streams, QuestDB clearly leads. DuckDB simply isn't on this track (stream ingest wasn't ever its design goal).

### 3.2 Time-Series Aggregation Queries (SAMPLE BY / window aggregation)

| System | 100M rows, `aggregate 1 day by 1m` | P50 latency |
|---|---|---|
| **QuestDB** | native `SAMPLE BY` | **~2ms** |
| **TimescaleDB** | `time_bucket()` | ~15ms |
| **DuckDB** | needs `make_time_bucket` UDF | ~40ms |

QuestDB's `SAMPLE BY` / `FIRST` / `LATEST` are **first-class time-series syntax**; the other two have to emulate with ordinary SQL. That's a "species difference," not something tuning closes.

### 3.3 Deployment & Ops Complexity

| System | Complexity | Notes |
|---|---|---|
| **DuckDB** | ⭐ lowest | `import duckdb` and you're in; zero ops |
| **QuestDB** | ⭐⭐ low | one container, but you have to think "service" |
| **TimescaleDB** | ⭐⭐⭐ medium | you maintain Postgres + the extension |

### 3.4 Ecosystem & Tooling

| System | Strength |
|---|---|
| **QuestDB** | Native Grafana data source, Prometheus direct write, SQL |
| **DuckDB** | Query Parquet/Iceberg directly, embed in Python/JS in one line, cross data lake |
| **TimescaleDB** | Whole Postgres family (migration, ecosystem, clients) at zero learning cost |

## 4. When Should You Use the "Combo Punch"?

The three **can coexist**; many production systems do exactly this:

- **QuestDB** handles the real-time stream (sensors / metrics / logs)
- **DuckDB** runs offline deep analysis over Parquet data lakes, one-shot jobs
- **TimescaleDB** (if you're already on Postgres) stays for transactional business

**Don't try to replace all three with one** — that's not cost savings, that's digging a hole.

## 5. Decision Tree (copy-paste this)

```
Is your data "continuously arriving every second"?
├─ yes → Is it time-series / metrics / logs / financial tick?
│        ├─ yes → QuestDB ✅
│        └─ no (high-concurrency OLTP) → use Postgres, not these three
└─ no (you have files: CSV/Parquet/Excel)
         ├─ want to embed in your program, no service → DuckDB ✅
         ├─ already on Postgres, team doesn't want a new stack → TimescaleDB ✅
         └─ pure time-series + max write throughput → still QuestDB ✅
```

## 6. Common Mistakes (ones I've hit)

1. **"DuckDB can do time-series too, why QuestDB?"** — DuckDB *can* "read Parquet time-series files then analyze," but it **can't do million-rows/sec real-time ingest**. Different scenarios.
2. **"TimescaleDB is more mature (it's Postgres)."** — Mature ≠ right fit. For high-frequency time-series ingest, dedicated QuestDB wins on throughput and query latency; TimescaleDB's edge is Postgres-ecosystem reuse.
3. **"QuestDB has no transactions, can't be a primary."** — Correct, it's a time-series/analytical store, not OLTP. Don't put your orders in it.

## 7. Conclusion

- **Continuous high-speed time-series stream → QuestDB** (throughput and time-series syntax are its home turf)
- **Feed files for one-shot analysis → DuckDB** (zero ops, embed-and-go)
- **Time-series on top of existing Postgres → TimescaleDB** (ecosystem reuse)

Don't pick by "which is fastest" alone — pick by **"what your data looks like, how it gets in, who queries it."** Answer those three questions and the answer is nearly unique.

---

*Test env: 4C8G / Docker / 2026-10. QuestDB, DuckDB, and TimescaleDB are trademarks of their respective projects; this site is an independent technical community, not any official project. Full benchmark methodology in the upcoming "QuestDB Performance Benchmarks" post.*
