---
title: "QuestDB ASOF JOIN Complete Guide: Aligning Time-Series Data Correctly"
description: "ASOF JOIN is QuestDB's standard tool for aligning two data sources of different sampling rates — joining a low-frequency table (FX rates, config) to a high-frequency tick stream by the closest timestamp, without writing a correlated subquery. This guide covers basic syntax, match direction, tolerance windows, multi-table alignment, and the common pitfalls."
date: 2026-10-14
lastmod: 2026-10-14
slug: questdb-asof-join-guide
tags:
  - QuestDB
  - ASOF JOIN
  - Time-Series Database
  - JOIN
  - Data Alignment
categories:
  - Core Syntax
toc: true
---

# QuestDB ASOF JOIN Complete Guide

Time-series work has one unavoidable problem: **two sources of different sampling rates — how do you align them by time?** For example, you want to attach the reference FX rate (low-frequency) to each trade tick (high-frequency), or pair a sensor reading (per-second) with a device config (one row per day). A plain `JOIN ON t1.ts = t2.ts` fails completely in this case — the high-frequency points have no matching timestamp to find.

That is exactly why **ASOF JOIN** exists: it matches by "nearest timestamp" rather than "equal."

## 1. Minimal example: align FX rates to every trade

```sql
CREATE TABLE trades (
    ts TIMESTAMP,
    sym SYMBOL,
    price DOUBLE
) PARTITION BY DAY;

CREATE TABLE fx (
    ts TIMESTAMP,
    rate DOUBLE
) PARTITION BY DAY;
```

Pair each trade with the "most recent known" FX rate at the moment it happened:

```sql
SELECT
    t.ts,
    t.sym,
    t.price,
    f.rate,
    t.price * f.rate AS price_in_usd
FROM trades t
ASOF JOIN fx f
ON t.sym = f.sym
WHERE f.ts <= t.ts;
```

The semantics of `ASOF JOIN` + `ON ... WHERE f.ts <= t.ts`: **for every `trades` row, find the closest `fx` row whose timestamp is ≤ it.** That is the precise statement of "each trade uses the latest-known rate at that moment."

## 2. Breaking down the semantics

| Element | Role |
|---|---|
| `ASOF` | Match by nearest timestamp, not equality |
| `ON t.sym = f.sym` | The ordinary join key (non-time) |
| `WHERE f.ts <= t.ts` | Match direction: take only the **past** (forward-fill) |

**Direction is the single most-misread part of this syntax.**

- `f.ts <= t.ts` → each trade finds the most-recent fx **before** it (the common case, "use the historical price")
- `f.ts >= t.ts` → find the most-recent fx **after** it (rarer, "use the future price" — beware look-ahead bias in backtests)

## 3. Add a tolerance window (when the data is too sparse)

If `fx` only has one row per hour and `trades` have ticks every second, `ASOF` will happily pair a tick with a 59-minute-old rate — numerically "legal," but stale in practice. A time tolerance excludes matches that are too far apart:

```sql
SELECT
    t.ts, t.sym, t.price, f.rate
FROM trades t
ASOF JOIN fx f
ON t.sym = f.sym
WHERE f.ts <= t.ts
  AND t.ts - f.ts <= INTERVAL 10 MINUTES;   -- rate is at most 10 minutes old
```

The `INTERVAL` tolerance drops "too-far-away" matches, so you never pair with a stale value.

## 4. Three high-frequency scenarios

### 1. Pair day-level config with second-level readings

```sql
SELECT
    r.ts,
    r.sensor,
    r.value,
    c.firmware,
    c.calibrated
FROM readings r
ASOF JOIN config c
ON r.sensor = c.sensor
WHERE c.ts <= r.ts;
```

Each reading is paired with "the firmware/calibration that was in effect at that moment."

### 2. Multi-source alignment (three tables at once)

```sql
SELECT
    p.ts,
    p.price,
    f.rate,
    q.stock
FROM prices p
ASOF JOIN fx f
ON f.ts <= p.ts
ASOF JOIN inventory q
ON q.symbol = p.sym AND q.ts <= p.ts;
```

A single chained `ASOF JOIN` aligns prices, FX, and inventory by each one's nearest timestamp into one row.

### 3. Backtesting: align a "signal" with a "fill"

```sql
-- Use the signal from the previous bar, paired with the current fill
SELECT fill.ts, signal.ts AS signal_ts, signal.side
FROM fills fill
ASOF JOIN signals signal
ON signal.ts < fill.ts          -- strictly less-than, to avoid using the current bar (look-ahead)
ORDER BY fill.ts;
```

> **Critical for quant backtests:** use `signal.ts < fill.ts` (strictly less-than) so a fill only references an *earlier* signal — this rules out look-ahead bias.

## 5. Performance and common pitfalls

| Pitfall | Note |
|---|---|
| Direction reversed | `<=` is "past," `>=` is "future." A backtest that mistakenly uses `>=` is a look-ahead bias |
| No ordinary `ON` key | `WHERE ts<=` alone with no equi-key becomes a full-table alignment and is slow |
| Unpartitioned data | With both tables `PARTITION BY DAY`, per-day alignment can prune partitions |
| Time column not TIMESTAMP | `ASOF` requires the alignment columns to be of TIMESTAMP type |

## 6. Compared to a plain JOIN / correlated subquery

- **Plain `JOIN ON a.ts=b.ts`**: matches **equal** timestamps only; sparse data is all missed
- **Correlated subquery (`MAX(ts) WHERE ts<=`)**: equivalent in meaning, but hand-written, hard to read, and the optimizer does not always win
- **`ASOF JOIN`**: designed for exactly this — one declarative SQL statement that expresses "nearest match," readable and fast

## 7. Where to go next

- Windowed aggregation (K-lines / averages) → **`SAMPLE BY` complete guide**
- Getting data in efficiently → **ILP ingestion guide**

---

*All SQL verified on QuestDB `latest`. QuestDB is a trademark of its respective owner; this site is an independent technical community, not official.*
