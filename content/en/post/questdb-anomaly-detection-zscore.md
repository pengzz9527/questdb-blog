---
title: "QuestDB Anomaly Detection in Practice: Rolling Z-Score & Financial Tick De-Dup Backtest"
description: "QuestDB anomaly detection in practice: a windowed rolling Z-score to catch time-series spikes, plus financial tick de-duplication before backtesting. Covers window-frame syntax, dedup semantics, backtest metrics, common traps, and runnable SQL with Grafana alert thresholds."
date: 2026-11-04
lastmod: 2026-11-04
slug: questdb-anomaly-detection-zscore
tags:
  - QuestDB
  - Anomaly Detection
  - Z-Score
  - Financial Tick
  - Window Functions
categories:
  - In Practice
toc: true
---

# QuestDB Anomaly Detection in Practice: Rolling Z-Score & Financial Tick De-Dup Backtest

Once data is landed, the next question is almost always "which point looks wrong?" In time series, "wrong" usually means **a value that deviates too far from its own mean and standard deviation over the last N seconds/minutes** — that's a rolling Z-score. In financial tick work it comes with a prerequisite: exchanges resend data and gateways dedup imperfectly, so the same tick can carry a duplicate timestamp. You must de-duplicate before backtesting.

This post gives two runnable SQL patterns: one for rolling Z-score spike detection, one for tick de-dup plus backtest. Both tables are ready to create, and threshold selection is spelled out.

## 1. Table design: decide what the "window frame" sits on

A rolling Z-score needs two things: the current value, and the mean/std of the N samples before it. In QuestDB you do this with window aggregation (`OVER (ORDER BY ts ROWS ...)`).

```sql
CREATE TABLE sensor (
    ts TIMESTAMP,
    host SYMBOL CAPACITY 1024 INDEX,
    value DOUBLE
) PARTITION BY DAY;
```

The financial tick table adds a `seq` column (exchange sequence number) plus `price`/`vol`, and keeps `ts` at nanosecond precision:

```sql
CREATE TABLE tick (
    ts TIMESTAMP,
    sym SYMBOL CAPACITY 512 INDEX,
    seq BIGINT,
    price DOUBLE,
    vol DOUBLE
) PARTITION BY DAY;
```

## 2. Rolling Z-score spike detection

Idea: for each point, take the mean `mu` and std `sigma` of the previous 60 points (about 60 seconds at 1 Hz), compute `z = (value - mu)/sigma`, and flag `|z|` above a threshold as anomalous.

```sql
SELECT
    ts, host, value,
    -- Window: 60 rows before the current one (excluding current)
    AVG(value)   OVER (PARTITION BY host ORDER BY ts
                       ROWS BETWEEN 60 PRECEDING AND 1 PRECEDING) AS mu,
    STDEVP(value) OVER (PARTITION BY host ORDER BY ts
                       ROWS BETWEEN 60 PRECEDING AND 1 PRECEDING) AS sigma
FROM sensor
ORDER BY host, ts
```

Then wrap in a CTE to compute Z and filter (QuestDB doesn't let you reference a window alias inside the same OVER, so use a CTE):

```sql
WITH w AS (
    SELECT ts, host, value,
        AVG(value)    OVER w AS mu,
        STDEVP(value) OVER w AS sigma
    FROM sensor
    WINDOW w AS (PARTITION BY host ORDER BY ts
                 ROWS BETWEEN 60 PRECEDING AND 1 PRECEDING)
)
SELECT ts, host, value,
       (value - mu) / NULLIF(sigma, 0) AS z
FROM w
WHERE ABS(value - mu) / NULLIF(sigma, 0) > 3
ORDER BY ts;
```

A few notes:

- `NULLIF(sigma, 0)` guards against division by zero — a flat series has near-zero variance and would otherwise blow up to inf.
- For the first 60 rows, `mu`/`sigma` are NULL, so the `ABS` comparison is skipped — that's a free "warm-up period, don't alert yet."
- A threshold of 3 is the statistical norm, but financial ticks are fat-tailed; 4–5, or a percentile cutoff, is common. Don't hard-code 3.

## 3. Financial tick de-dup: mandatory before backtesting

Exchange feeds often carry **duplicate timestamps** (resends, multiple gateways). Backtesting raw ticks double-counts the same instant and skews fills and signals. Dedup rule: **for the same `sym` and `ts`, keep the row with the max `seq`** (highest sequence = final state), drop the rest.

```sql
-- Keep only the max-seq row per (sym, ts)
SELECT ts, sym, seq, price, vol
FROM (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY sym, ts
            ORDER BY seq DESC
        ) AS rn
    FROM tick
)
WHERE rn = 1
ORDER BY sym, ts;
```

If `seq` is not guaranteed monotonic at the same `ts` (out-of-order gateways), fall back to strict `seq` dedup:

```sql
-- Dedup: one row per (sym, seq) to kill true duplicate delivery
SELECT DISTINCT sym, seq, ts, price, vol FROM tick;
```

**Don't use full-row `DISTINCT` as your main dedup** — it compares every column, so a tiny price/volume jitter keeps two rows. Keying on `seq` (or `(sym, ts)` keeping the latest) is reliable.

## 4. De-duped backtest: signals and hits

On cleaned ticks, backtest a "price breaks threshold" strategy and count hits and P&L:

```sql
WITH clean AS (
    SELECT ts, sym, seq, price,
        LAG(price) OVER (PARTITION BY sym ORDER BY seq) AS prev_price
    FROM (
        SELECT * FROM (
            SELECT *, ROW_NUMBER() OVER (PARTITION BY sym, ts ORDER BY seq DESC) AS rn
            FROM tick
        ) WHERE rn = 1
    )
)
SELECT sym,
       COUNT(*) AS n_signals,
       SUM(CASE WHEN price > prev_price * 1.001 THEN 1 ELSE 0 END) AS up_breaks,
       AVG(price - prev_price) AS avg_delta
FROM clean
WHERE prev_price IS NOT NULL
GROUP BY sym;
```

`n_signals` should be materially lower than the raw row count — the gap is the duplicate ticks removed, and that number belongs in your backtest report as proof you're not testing on dirty data.

## 5. Traps and tuning

- **Window frame too large**: `ROWS 10000 PRECEDING` smears the entire history into every point, and window cost grows linearly with the frame. Size by sample rate: 60–300 rows at 1 Hz, 100–500 for millisecond ticks.
- **SYMBOL cardinality explosion**: the number of `PARTITION BY host/sym` partitions equals cardinality. With tens of thousands of devices, set `CAPACITY` on the SYMBOL at CREATE TABLE time, or runtime dictionary growth will stall.
- **Don't trust Z-score blindly on fat tails**: tick return distributions are leptokurtic, so `Z>3` may flag a lot of normal noise. **Median + MAD** (median absolute deviation) is more outlier-robust:

```sql
SELECT ts, host, value,
    value - MEDIAN(value) OVER w AS dev,
    MEDIAN(ABS(value - MEDIAN(value) OVER w)) OVER w AS mad  -- simplified
FROM sensor
WINDOW w AS (PARTITION BY host ORDER BY ts ROWS 60 PRECEDING);
```

(In production, aggregate into a MAD series first, then take the median, to avoid nested windows.)

- **Grafana alert thresholds**: give the Z-score panel two thresholds — `|z|>3` amber, `|z|>4.5` red — and use a "last 5 minutes, 3 consecutive points" window for noise reduction, so single-point spikes don't page you.

## 6. Next

The previous post covered retention and hot/cold tiering; this one puts the data inside the retention window to work. A natural next step: write Z-score results back into an `anomaly` table, then use `SAMPLE BY` for an hourly anomaly-count panel — completing the detect → alert → review loop.

## References

- [QuestDB docs: SELECT](https://questdb.io/docs/reference/sql/select/)
- [Python statistics stdlib docs](https://docs.python.org/3/library/statistics.html)

## Related reading

- [QuestDB + Kafka Real-Time Pipeline: ILP Ingestion to SAMPLE BY Rollups](/en/post/questdb-kafka-connect-pipeline/)
- [QuestDB SAMPLE BY Complete Guide: The Soul of Time-Series Aggregation](/en/post/questdb-sample-by-complete-guide/)
- [QuestDB Crypto Trading Anomaly Detection: Real-Time Tick Alerts in Practice](/en/post/questdb-crypto-trading-anomaly-detection/)
