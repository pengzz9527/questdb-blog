---
title: "QuestDB Time-Series Compression & Retention: PARTITION BY Auto-Purge and Hot-Cold Tiering"
description: "QuestDB retention and compression in practice: how PARTITION BY day/month auto-purges expired data, how hot-cold tiering archives old partitions to cheap object storage. Covers delete windows, compaction, and disk-usage control, plus capacity estimation and importable cold queries."
date: 2026-11-04
lastmod: 2026-11-04
slug: questdb-timeseries-compression-retention
tags:
  - QuestDB
  - Retention
  - PARTITION BY
  - Time-Series Compression
  - Hot-Cold Tiering
categories:
  - Getting Started
toc: true
---

# QuestDB Time-Series Compression & Retention

Time-series data has an inescapable property: it only grows. Billions of points a day means disk explodes over a year. The legacy answer is "dump it into one big table and delete it by hand occasionally" — but deleting from a time-organized table is slow and easy to botch. QuestDB turns this into a CREATE-TABLE clause — `PARTITION BY` — so expired data is cleaned up per-partition automatically, and columnar compression keeps disk usage low.

This post covers three things: how to partition by time so a delete becomes "rm a directory," how to estimate a table's year-long footprint, and how hot/cold tiering moves old data to cheaper storage.

## 1. Why day-partitioning saves disk and deletes fast

QuestDB is columnar — values of the same column sit contiguously, which compresses extremely well. `PARTITION BY` controls the time granularity at which data is split into partition files:

- `DAY`: one partition per calendar day. Good for small tables and high-frequency queries.
- `MONTH`: one partition per month. Good for long retention and wide-range queries.

The key win is **delete becomes rm-of-a-directory.** You drop the whole expired partition — QuestDB does not scan row by row, and disk is genuinely reclaimed (unlike engines that mark rows deleted and wait on compaction to free space).

```sql
-- Day-partitioned, fits ~30-90 days of high-frequency detail
CREATE TABLE otel_trace (
    ts      TIMESTAMP,
    trace   SYMBOL CAPACITY 8192 INDEX,
    span    SYMBOL CAPACITY 256,
    dur_ms  DOUBLE,
    status  INT
) PARTITION BY DAY;

-- Month-partitioned, fits long-term archives
CREATE TABLE device_daily (
    ts     TIMESTAMP,
    device SYMBOL CAPACITY 4096 INDEX,
    energy DOUBLE,
    peak   DOUBLE
) PARTITION BY MONTH;
```

## 2. Picking the partition granularity

Granularity trades two opposite things: partition-file count (more = more fragmentation, more compaction work) versus how much a single delete/query scans (larger = coarser, but cleaner deletes). Rules of thumb:

| Scenario | Granularity | Why |
|---|---|---|
| High-frequency detail, < 90-day retention | `DAY` | Queries locate by day; a day-delete is a day-drop |
| Monthly rollups, > 1-year retention | `MONTH` | Controllable file count; a month-delete is clean |
| Very high cardinality + long retention | `MONTH` or `YEAR` | Avoid small-file explosion |

When unsure, start on `DAY`, watch the partition directory count for 2–4 weeks, and move to `MONTH` if it grows too fast. Note QuestDB does not let you change the partition clause in place — you build a new table, backfill, and cut over.

## 3. Capacity estimation: how much disk is one year?

Estimate by computing bytes-per-row, multiplying by row count, then by compression ratio. Rough method:

1. The **logical size** of one row across all columns (by type: `TIMESTAMP` 8 B, `DOUBLE` 8 B, `SYMBOL` by distinct-cardinality cost, `INT` 4 B).
2. After columnar + dictionary encoding, on-disk is typically **1/3 to 1/5** of logical size.

Example: a `TIMESTAMP + SYMBOL(10k-class cardinality) + DOUBLE + INT` row is ~32 B logical. A year of ~3 billion rows (roughly 100 rows/s × year) is ~96 GB logical, ~20–30 GB compressed. So "keep a year" on SSD is not outrageous, provided you rely on columnar compression rather than storing raw text.

Monitor with `SHOW CATALOG` / `SHOW CREATE TABLE` for the partition layout, and `du -sh data/<table>/*` for real usage — that beats any estimate.

## 4. Hot/cold tiering: move old data to cheap storage

QuestDB has no built-in "auto-demote to cold," but retention is the hot tier, and the cold tier is an ETL move:

- **Hot tier** (QuestDB): keep the last N days, `PARTITION BY DAY`, always queryable.
- **Cold tier**: use `COPY` or a scheduled job to export partitions older than N days to Parquet on object storage (S3/OSS).

```sql
-- Export a month partition older than 30 days to Parquet
COPY (
    SELECT * FROM device_daily WHERE ts >= '2026-01-01' AND ts < '2026-02-01'
) TO 's3://archive/device_daily/2026-01.parquet' (FORMAT PARQUET, COMPRESSION ZSTD);
```

After export, run `DELETE FROM device_daily WHERE ts < '2026-02-01'` so the hot tier holds only the last 30 days. When you need cold data later, QuestDB's `SAMPLE BY` can read external Parquet via import — you don't have to pull the whole cold set back in.

## 5. Real delete semantics and gotchas

- `DELETE` in QuestDB is **lazy per partition**: dropping a whole partition is cleanest; scattered row-deletes generate tombstones that only reclaim on compaction. So "drop the whole partition if you can."
- Forgetting `PARTITION BY` at CREATE TABLE is not fatal, but deletes degrade to row scans and you lose the "rm-a-directory" speed.
- Match retention to the SLA: observability data (traces) usually needs only 7–30 days; business/metering data often wants 90+ days or a year.
- Don't fight the partitioning mechanism with app-side scheduled `DELETE`s — let partitions age out naturally and pair that with cold-tier export.

## 6. Next

The next post covers QuestDB anomaly detection in practice — connecting "can archived old data still be queried" with real-time alerting via a rolling Z-score that catches spikes inside the retention window.
