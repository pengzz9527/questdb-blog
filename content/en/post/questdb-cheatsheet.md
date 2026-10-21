---
title: "QuestDB SQL Cheatsheet: SAMPLE BY, ASOF JOIN, SYMBOL & Partitioning in One Page"
description: "A one-page QuestDB SQL cheatsheet covering SAMPLE BY (all modes including FILL, ALIGN, OFFSET), ASOF JOIN directions and tolerance, SYMBOL index sizing and cardinality tuning, PARTITION BY creation and retention, ILP write templates, and the 20 most-asked time-series queries. Bookmark this page instead of flipping through the docs."
date: 2026-10-21
lastmod: 2026-10-21
slug: questdb-cheatsheet
tags:
  - QuestDB
  - Cheatsheet
  - SQL
  - Time-Series Database
categories:
  - In Practice
toc: true
---

# QuestDB SQL Cheatsheet: SAMPLE BY, ASOF JOIN, SYMBOL & Partitioning in One Page

This page pulls together the most frequently-screenshotted snippets from the previous guides and reorders them into a single reference, arranged as **schema → write → query → align → retain**. The goal: on day one of a new QuestDB project, run the core operations without opening the docs.

## Schemas: the three patterns you actually use

### Pattern 1: sensor / metrics table (low-cardinality tags + numeric fields)

```sql
CREATE TABLE metrics (
    ts TIMESTAMP,
    host SYMBOL,              -- machine name, dozens to thousands
    metric SYMBOL,            -- metric name, dozens
    val DOUBLE
) PARTITION BY DAY;
```

`host` and `metric` are low-cardinality; QuestDB encodes them as SYMBOL automatically, no extra index cost.

### Pattern 2: market / tick table (high-frequency symbol column, needs an index)

```sql
CREATE TABLE ticks (
    ts TIMESTAMP,
    sym SYMBOL INDEX 256,     -- trading pair, hundreds to thousands
    px DOUBLE,
    vol DOUBLE
) PARTITION BY HOUR;
```

`INDEX 256` inlines the symbol dictionary per 256-row block, which is a step faster than the default external index on hot queries; `PARTITION BY HOUR` keeps "last hour" scans to a single partition.

### Pattern 3: event stream (high-cardinality tag, use VARCHAR as a fallback)

```sql
CREATE TABLE events (
    ts TIMESTAMP,
    user_id VARCHAR,          -- unique per row, SYMBOL would blow up
    event VARCHAR,
    payload JSON
);
```

When the distinct-value count of a column exceeds ~10k, skip SYMBOL and use VARCHAR — a per-row symbol dictionary costs more memory than it saves.

## Writing: ILP, three-line templates

```bash
# single row, no timestamp (server time is used)
echo 'cpu,host=srv1,metric=usage val=42.5' | nc localhost 9009

# batch + explicit nanosecond timestamp
echo 'cpu,host=srv1,metric=usage val=42.5 1729555200000000000' | nc localhost 9009

# auto-DDL: QuestDB creates the table on first sight of the measurement
echo 'mem,host=srv1,metric=used_kb val=1048576' | nc localhost 9009
```

Caveat: when ILP auto-creates a table, **the first non-ts string column encountered becomes the SYMBOL column**, and subsequent string columns of the same type follow that decision. If you want to control which column is the symbol, pre-create the table with DDL before writing.

## Query: the 20 you reach for most

```sql
-- 1. per-5-minute average over the last hour
SELECT ts, AVG(val) AS avg_val
FROM metrics WHERE ts >= now() - INTERVAL 1 HOUR
SAMPLE BY 5m;

-- 2. per-5-minute max + the timestamp of that max
SELECT ts, MAX(val) AS peak, MAX_AGGREGATE_TIMESTAMP(val) AS peak_ts
FROM metrics WHERE ts >= now() - INTERVAL 1 DAY
SAMPLE BY 5m;

-- 3. fill gaps with the previous value (no misleading zeros)
SELECT ts, val FROM cpu SAMPLE BY 1m FILL(previous);

-- 4. fill gaps with linear interpolation
SELECT ts, val FROM cpu SAMPLE BY 1m FILL(linear);

-- 5. independent aggregation per host, results aligned by host
SELECT ts, host, AVG(val)
FROM metrics WHERE ts >= now() - INTERVAL 1 HOUR
SAMPLE BY 1m BY host;

-- 6. snap timestamps to the 10-second grid
SELECT ts, val FROM cpu SAMPLE BY 10s ALIGN TO SECOND;

-- 7. 5-minute windows offset by 50 seconds (windows start at :05, not :00)
SELECT ts, AVG(val) FROM cpu SAMPLE BY 5m OFFSET 50s;

-- 8. P95 per host
SELECT host, PERCENTILE(val, 0.95) AS p95
FROM metrics WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY host;

-- 9. two tables aligned by nearest timestamp (ASOF, the common case)
SELECT t.ts, t.px, f.rate
FROM ticks t ASOF JOIN fx f ON f.ts <= t.ts;

-- 10. ASOF + tolerance (FX rate is at most 10 minutes stale)
SELECT t.ts, t.px, f.rate
FROM ticks t ASOF JOIN fx f ON f.ts <= t.ts
WHERE t.ts - f.ts <= INTERVAL 10 MINUTE;

-- 11. ASOF + equi-key (multi-source alignment)
SELECT r.ts, r.val, c.firmware
FROM readings r ASOF JOIN config c
ON r.host = c.host AND c.ts <= r.ts;

-- 12. backtest look-ahead guard (strictly less-than)
SELECT fill.ts, signal.side
FROM fills fill ASOF JOIN signals signal
ON signal.ts < fill.ts;

-- 13. window function: pair each row with itself 5 minutes earlier
SELECT ts, val, LAG(val, 5) OVER (ORDER BY ts) AS val_5m_ago
FROM cpu WHERE ts >= now() - INTERVAL 1 HOUR;

-- 14. rolling mean (100-row window)
SELECT ts, AVG(val) OVER (ORDER BY ts ROWS 100 PRECEDING) AS roll_avg
FROM cpu WHERE ts >= now() - INTERVAL 1 HOUR;

-- 15. count
SELECT COUNT(*) FROM ticks WHERE ts >= now() - INTERVAL 1 DAY;

-- 16. distinct value count
SELECT COUNT(DISTINCT sym) FROM ticks;

-- 17. count grouped by hour
SELECT ts, COUNT(*)
FROM ticks WHERE ts >= now() - INTERVAL 24 HOUR
SAMPLE BY 1h;

-- 18. JSON field extraction
SELECT ts, payload->>'region' AS region FROM events
WHERE ts >= now() - INTERVAL 1 HOUR;

-- 19. list columns of a table
SELECT * FROM columns WHERE table_name = 'metrics';

-- 20. partition listing
SELECT * FROM partitions WHERE table_id =
  (SELECT id FROM tables WHERE name = 'metrics');
```

## ASOF JOIN direction, at a glance

| Syntax | Semantics | Use |
|---|---|---|
| `f.ts <= t.ts` | each t pairs with the most recent f **before** it | most common: historical price, historical config |
| `f.ts >= t.ts` | each t pairs with the most recent f **after** it | rare; in a backtest this is look-ahead bias |
| `f.ts < t.ts` | strictly before (excludes same-timestamp) | signal/fill alignment |
| `+ INTERVAL x` | tolerance window | excludes matches that are too stale |

## SYMBOL sizing: three tiers

| Cardinality estimate | DDL | Note |
|---|---|---|
| < 64 | `SYMBOL INDEX 64` (default) | inlined dictionary, fastest |
| 64 – 10k | `SYMBOL INDEX 256` or `512` | step up by one tier |
| > 10k | `VARCHAR` | skip the symbol dictionary to avoid a memory blow-up |

Measure cardinality once: `SELECT COUNT(DISTINCT col) FROM tbl`, then pick the INDEX tier.

## Partitioning and retention

```sql
-- declare at creation time (ILP auto-DDL has no partitioning; pre-create with DDL)
CREATE TABLE cpu (...) PARTITION BY DAY;
CREATE TABLE ticks (...) PARTITION BY HOUR;
CREATE TABLE logs (...) PARTITION BY MONTH;

-- drop one partition (date string format varies with partition granularity)
SELECT DROP_PARTITION('cpu', '2026_01_15');   -- DAY granularity
SELECT DROP_PARTITION('ticks', '2026_01');     -- MONTH granularity

-- inspect partition storage
SELECT * FROM partitions WHERE table_id = (SELECT id FROM tables WHERE name='cpu');
```

In production, a daily cron that drops partitions older than 90 days pins disk usage to a constant.

## Common errors and how to diagnose them

| Error | Cause | Check |
|---|---|---|
| `column not found` | typo, or column type mismatch at creation | `SELECT * FROM columns WHERE table_name='...'` |
| `ASOF JOIN requires TIMESTAMP` | alignment column is not TIMESTAMP | confirm both tables' ts columns are TIMESTAMP |
| result N× larger than expected | `SAMPLE BY` without `WHERE` swept the full table | always carry a time predicate |
| ILP write returns 400 | field type conflicts with the existing column (e.g. a string into a DOUBLE) | `DESCRIBE` the table first; ILP values must match the column type |
| query suddenly slow | time predicate stopped pruning partitions | check the `WHERE ts >= now() - INTERVAL ...` syntax |

## Off-page links

- Full `SAMPLE BY` mode breakdown → SAMPLE BY complete guide
- Three ASOF JOIN scenarios in depth → ASOF JOIN guide
- ILP protocol field-by-field → ILP ingestion guide
- Persistence and scaling after install → install guide

---

*This cheatsheet is verified against QuestDB `latest`. `DROP_PARTITION` date formats and `INDEX N` upper bounds evolve with the version — before shipping, check the exact syntax your deployed version supports with `SELECT * FROM functions`. QuestDB is a trademark of its respective owner; this site is an independent technical community, not official.*
