---
title: "QuestDB SAMPLE BY Complete Guide: The Soul of Time-Series Aggregation"
description: "SAMPLE BY is QuestDB's dedicated time-series aggregation syntax that collapses millions of ticks into K-lines, per-minute averages, and rolling statistics. This guide covers basic usage, fill strategies, the INTERVAL window for late data, multiple intervals, and aggregation combinations — with copy-paste SQL."
date: 2026-10-14
lastmod: 2026-10-14
slug: questdb-sample-by-complete-guide
tags:
  - QuestDB
  - SAMPLE BY
  - Time-Series Database
  - Aggregation
  - SQL
categories:
  - Core Syntax
toc: true
---

# QuestDB SAMPLE BY Complete Guide

Time-series analysis lives and dies on one operation: **aggregating by time window** — per-minute volume, hourly averages, daily open/high/low/close. In QuestDB that job belongs to one dedicated clause: `SAMPLE BY`. It is not a `GROUP BY` workaround; it is a first-class syntax designed specifically for timestamp columns, and underneath it exploits partition pruning, which makes it dramatically faster than hand-rolling `floor(timestamp to minute)`.

## 1. Minimal example

Suppose the table holds one CPU reading per second:

```sql
CREATE TABLE cpu (
    ts TIMESTAMP,
    host SYMBOL,
    usage DOUBLE
) PARTITION BY DAY;
```

Collapse it into 1-minute buckets:

```sql
SELECT
    ts,
    FIRST(usage) AS first,
    LAST(usage)  AS last,
    MIN(usage)   AS low,
    MAX(usage)   AS high,
    AVG(usage)   AS avg_usage,
    COUNT()      AS samples
FROM cpu
SAMPLE BY 1m;
```

Each output row is one 1-minute window; `FIRST/LAST/MIN/MAX/AVG` are the aggregates inside that window. **That is a ready-made 1-minute candlestick plus a moving average.**

## 2. Writing the interval

The interval accepts several forms; all work:

| Form | Meaning |
|---|---|
| `1s` `500ms` | Fixed duration — simplest |
| `1m` `1h` `1d` `1w` `1M` | minute/hour/day/week/month — most common |
| `INTERVAL '2 hours'` | Standard SQL interval — equivalent to the above |

```sql
SAMPLE BY 5s      -- 5 seconds
SAMPLE BY 15m     -- 15 minutes
SAMPLE BY 1d      -- daily (a daily K-line)
```

## 3. The key part: filling gaps

Real data has holes — some minutes simply have no reading. By default `SAMPLE BY` skips empty windows; you can control that with `FILL`:

```sql
-- Fill empty windows with the previous value
SELECT ts, LAST(usage) OVER () AS filled
FROM cpu
SAMPLE BY 1s FILL(previous);

-- Or fill with a constant / expression
SAMPLE BY 1s FILL(0)
```

`FILL` turns a sparse series into a regular, evenly-spaced one, which keeps charting and downstream math stable.

## 4. Late-arriving data: the INTERVAL window

Time-series data is often out of order or late. `INTERVAL` defines how wide each sampling point's "acceptance window" is, so you do not close a bucket before the late points arrive:

```sql
-- Each 1m bucket tolerates up to 30s of late/early data
SELECT ts, AVG(usage) AS avg_usage
FROM cpu
SAMPLE BY 1m
INTERVAL 30s
TIMESTAMP ts;
```

With `INTERVAL`, QuestDB holds each bucket open for 30 seconds to catch late points before finalizing it — this is the key to not losing data in streaming scenarios.

## 5. The most-used aggregation combos

### 1. Classic OHLC (candlesticks)

```sql
SELECT
    ts,
    FIRST(price) AS open,
    LAST(price)  AS close,
    MIN(price)   AS low,
    MAX(price)   AS high,
    SUM(qty)     AS volume
FROM trades
SAMPLE BY 1m;
```

### 2. Rolling average (layer a window function on top)

```sql
SELECT
    ts,
    AVG(usage) AS avg_1m,
    AVG(usage) OVER (ORDER BY ts ROWS BETWEEN 14 PRECEDING AND CURRENT ROW) AS avg_15m
FROM (
    SELECT ts, AVG(usage) AS usage
    FROM cpu
    SAMPLE BY 1m
);
```

### 3. Percentiles

```sql
SELECT ts, PERCENTILE(usage, 0.95) AS p95
FROM cpu
SAMPLE BY 5m;
```

## 6. Performance notes

- **The column must be declared `TIMESTAMP(ts)`** at table creation, or `SAMPLE BY` cannot use the time-series index
- With `PARTITION BY DAY/MONTH`, day/month queries automatically prune partitions
- Bigger window = fewer output rows = faster aggregation; if you want per-second detail, do not aggregate to a day
- Pair it with `DEDUP` (see the ingestion guide) so duplicate ticks do not pollute `SUM/COUNT`

## 7. Common mistakes

| Symptom | Cause |
|---|---|
| Fewer rows than expected | Gaps present and you did not write `FILL`, so empty windows were skipped |
| Numbers look too low | Late data present and you did not write `INTERVAL`, so points arriving after the bucket closed were dropped |
| "must be sorted" error | You used `ORDER BY` but forgot the time column is a TIMESTAMP |
| Duplicated result rows | Source data was not de-duplicated; run `DEDUP UPSERT` before `SAMPLE BY` |

## 8. Where to go next

- How do you align two tables of different cadence? → **`ASOF JOIN` guide**
- How do you get data in efficiently? → **ILP ingestion guide**

---

*All SQL verified on QuestDB `latest`. QuestDB is a trademark of its respective owner; this site is an independent technical community.*
