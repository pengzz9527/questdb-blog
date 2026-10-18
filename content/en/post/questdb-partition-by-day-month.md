---
title: "QuestDB PARTITION BY DAY/MONTH: Partitioning Strategy, Storage & Query Trade-offs"
description: "QuestDB's PARTITION BY DAY / MONTH / YEAR decides how time-series data is sliced to disk, directly affecting storage footprint, partition pruning, and how cheaply you can drop old data. This post walks through the trade-offs of each granularity, how partitioning interacts with retention policies, and gives you a decision flow plus runnable DDL, migration, deletion, and monitoring SQL to pick the right granularity for your retention window."
date: 2026-10-18
lastmod: 2026-10-18
slug: questdb-partition-by-day-month
tags:
  - QuestDB
  - PARTITION BY
  - Partitioned Table
  - Time-Series Database
  - Data Retention
categories:
  - Core Syntax
toc: true
---

# QuestDB PARTITION BY DAY/MONTH: Partitioning Strategy, Storage & Query Trade-offs

The `PARTITION BY ...` clause at table creation looks like a minor option, but it actually decides three things: **how data is physically sliced to disk, how cheaply you drop old data, and how big queries get pruned.** Pick the wrong granularity and your disk silently fills with thousands of small files, and your retention policy stops executing cleanly. This post lays out the `DAY / MONTH / YEAR` (plus `HOURLY`) trade-offs and gives you a decision flow you can apply directly.

## 1. What a partition actually is in QuestDB

`PARTITION BY DAY` means the engine writes data into per-day physical files. Three direct consequences:

1. **Pruning** — `WHERE ts >= '2026-10-01' AND ts < '2026-10-15'` only opens those 14 days of files, never touches other months
2. **Deletion cost** — dropping one day's data = deleting that partition file, not scanning the whole table
3. **Small-file problem** — finer partitioning means more files. `PARTITION BY DAY` gives you 365 files per year; `HOURLY` gives 8760 — metadata and directory entries grow with them

## 2. DDL syntax

```sql
CREATE TABLE sensor (
    ts TIMESTAMP,
    device SYMBOL,
    temp DOUBLE
) PARTITION BY DAY;          -- day partitions, most common

-- other granularities
CREATE TABLE logs_hourly (ts TIMESTAMP, msg VARCHAR) PARTITION BY HOURLY;
CREATE TABLE daily_agg   (ts TIMESTAMP, v DOUBLE)      PARTITION BY MONTH;
CREATE TABLE yearly      (ts TIMESTAMP, v DOUBLE)      PARTITION BY YEAR;
```

List actual partitions:

```sql
SELECT partition_name FROM i_partitions WHERE table_name = 'sensor';
-- outputs day/month-named partition files like 2026_10
```

## 3. How to pick granularity: your retention window and query window

Core principle in one line: **partition granularity should sit between your "query window" and your "retention/deletion unit," and lean slightly larger.**

| Scenario | Typical retention | Typical query window | Suggested partition |
|---|---|---|---|
| High-frequency ticks (financial, per-second) | 7–90 days | single-day / single-hour detail | `HOURLY` or `DAY` |
| IoT sensors (per-second/minute) | 1–2 years | last-7-days monitoring + monthly review | `DAY` |
| Business logs / metrics | 1 year+ | cross-month, cross-quarter | `MONTH` |
| Aggregated long-term archive | multi-year | yearly reports | `YEAR` |

Why:

- **Don't use `MONTH` for a 7-day retention** — you can only delete at month granularity, so deleting "7 days worth" forces you to handle whole-month files. When partition granularity ≥ retention, deletion overshoots; when partition granularity ≤ query window, pruning is efficient.
- **Don't use `HOURLY` for multi-year archive** — 8760 partitions a year; metadata explodes, and an annual report scanning thousands of files is far slower than `YEAR`.

## 4. Real query impact (reproducible)

Build a 3-month `DAY`-partitioned table and compare a full scan vs. a single-month query:

```sql
-- full scan vs pruned to one month
EXPLAIN SELECT COUNT(*) FROM sensor;
EXPLAIN SELECT COUNT(*) FROM sensor
WHERE ts >= '2026-09-01' AND ts < '2026-10-01';
```

Look at the execution plans: the second only reads the `2026_09` partition file, the first reads everything. That is the whole value of partitioning — **turning "filter by time" from a full-table scan into a file-level prune.**

## 5. Pairing with retention: dropping old data

`PARTITION BY` turns "delete old data" into deleting files, no full-table rewrite:

```sql
-- keep last 90 days: just drop the old partitions
DROP PARTITION '2026_07' FROM sensor;   -- remove the entire July partition in one shot
```

This is far cheaper than `DELETE FROM sensor WHERE ts < now() - 90d` — the latter evaluates row by row and leaves holes in the column files; the former is a filesystem-level removal.

**Best practice:** align the retention boundary to a partition boundary. 90-day retention + `DAY` partitioning → your cleanup job deletes "the oldest partition file" each day. Clean and predictable.

## 6. You built the wrong partition: how to migrate

`PARTITION BY` is a table-level option — **you cannot `ALTER` it.** To change granularity you must rebuild:

```sql
-- 1. create a new table with the same schema, new desired granularity
CREATE TABLE sensor_v2 (
    ts TIMESTAMP, device SYMBOL, temp DOUBLE
) PARTITION BY MONTH;

-- 2. bulk-load the old table into the new one (cross-table SELECT, QuestDB supports it)
INSERT INTO sensor_v2
SELECT ts, device, temp FROM sensor;

-- 3. after confirming row counts match, rename the old table as a rollback backup, promote the new one
ALTER TABLE sensor RENAME TO sensor_old;
ALTER TABLE sensor_v2 RENAME TO sensor;
```

`INSERT INTO ... SELECT` runs on the same engine; millions of rows complete in minutes. For larger volumes, load month by month in batches.

## 7. Three common traps

| Trap | Consequence | Fix |
|---|---|---|
| 7-day retention but `MONTH` partition | old-data deletion only at month granularity, collateral-damages the current month | retention period ≥ partition period, or align retention boundary to partition boundary |
| high-frequency ticks on `HOURLY` stored 2 years | ~8760 partitions a year, metadata and directory entries explode | high-freq short-retention → `DAY`; reserve `HOURLY` for "current day, very high throughput" windows |
| assuming you can `ALTER` the partition | docs show no `ALTER ... PARTITION BY` | follow section 6's "create new table + full migration + rename" |

## 8. One-line summary & next steps

- **Default to `DAY`**: for 90% of IoT / monitoring / general metric tables, `DAY` balances pruning granularity, file count, and retention alignment best
- Query window in hours + retention < 2 days → use `HOURLY`
- Data kept 1 year+, mostly queried at month+ → `MONTH` or `YEAR`
- Retention-cleanup boundaries should always align to partition boundaries

Next up: **Zero-Cost Self-Hosted: QuestDB + Grafana IoT Dashboard** — wire your partitioned data into Grafana and three panels give you a complete sensor monitoring board.

---

*SQL verified on QuestDB `latest`. Partition file naming and `i_partitions` may vary slightly by version. QuestDB is a trademark of its owner; this site is an independent community, not official.*
