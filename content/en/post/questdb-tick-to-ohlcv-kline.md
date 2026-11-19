---
title: "Building K-lines From Financial Ticks: QuestDB SAMPLE BY Computes OHLCV"
description: "Store tick-level financial data in QuestDB and compute your own K-lines: one SAMPLE BY line yields 1m candles, FIRST/MAX/MIN/LAST give OHLC, SUM gives volume. Covers the ticks table, ILP writes, per-symbol sampling, daily candles, the big-candle resampling trap, pre-aggregating a candle table for charts, and ASOF JOIN with an index."
date: 2026-11-15
lastmod: 2026-11-15
slug: questdb-tick-to-ohlcv-kline
tags:
  - QuestDB
  - Financial Data
  - Candlesticks
  - OHLCV
  - SAMPLE BY
categories:
  - In Practice
toc: true
---

# Building K-lines From Financial Ticks: QuestDB SAMPLE BY Computes OHLCV

Candlesticks (K-lines) are the most basic data structure in markets. Most teams pull candles from a quote API, and the cost is that you give up control of the candle period, session boundaries, timezone, and volume semantics. If you already hold tick-level data in QuestDB — exchange raw feeds, backtest fills, crypto order-book snapshots — you can compute candles yourself: one `SAMPLE BY` clause produces OHLCV, and you pick the period whenever you want.

## 1. What you gain by computing candles yourself

- **Arbitrary periods**: 7-minute, 30-minute, 4-hour, weekly — just change the `SAMPLE BY` argument.
- **Session and timezone boundaries**: 09:00–16:00 Asia session, US session, 24h crypto — sliced exactly to your business.
- **Volume semantics**: traded volume, trade count, average fill price, all in the same aggregation.
- **No vendor lock-in**: the candle tier of a quote API is usually the expensive one.

The cost is that you must nail two conventions about the candle timestamp: the `ts` on a candle is the **window start** (open time), not the close time. Get this wrong and every chart shifts by one window, and every backtest signal misaligns.

## 2. Data prep: ticks table + ILP writes

```sql
CREATE TABLE ticks (
    ts TIMESTAMP,
    symbol SYMBOL,
    price DOUBLE,
    volume DOUBLE
) TIMESTAMP ts PARTITION BY DAY;
```

Write with ILP — one trade per line, multiple fields on the same line:

```
ticks,symbol=BTCUSD price=67213.4,volume=0.0042 1700000000123456789
ticks,symbol=BTCUSD price=67215.1,volume=0.0110 1700000000456789012
```

Before backtesting, dedupe on (ts, symbol): duplicate timestamps distort OHLC. The dedup pattern is covered in our rolling Z-score and tick-dedup article. Below we assume the input is already clean.

## 3. Core SQL: 1-minute candles

```sql
SELECT
    ts,
    FIRST(price) AS open,
    MAX(price)   AS high,
    MIN(price)   AS low,
    LAST(price)  AS close,
    SUM(volume)  AS volume
FROM ticks
WHERE symbol = 'BTCUSD'
SAMPLE BY 1m;
```

Notes:

- **Window anchoring**: `SAMPLE BY 1m` aligns windows to the timestamp of the first row in the window; the output `ts` is the window start.
- **Empty windows are skipped by default**: windows without data produce no row (equivalent to `SKIP EMPTY`), so the candle sequence stays clean.
- **Keep the gap**: to carry the previous window's close into an empty window so charts don't break, write `SAMPLE BY 1m FILL PREVIOUS`.

## 4. Multiple symbols in one query

Add the sampling partition clause and each symbol is sampled independently:

```sql
SELECT
    symbol,
    ts,
    FIRST(price) AS open,
    MAX(price)   AS high,
    MIN(price)   AS low,
    LAST(price)  AS close,
    SUM(volume)  AS volume
FROM ticks
SAMPLE BY 1m PARTITION BY symbol;
```

Watch out: `PARTITION BY symbol` here is the sampling partition of `SAMPLE BY`, not the physical partition declared at table creation. Two different things.

## 5. Larger periods: 30m / 1h / daily

Just change the argument:

```sql
SELECT
    ts,
    FIRST(price) AS open,
    MAX(price)   AS high,
    MIN(price)   AS low,
    LAST(price)  AS close,
    SUM(volume)  AS volume
FROM ticks
WHERE symbol = 'BTCUSD' AND ts >= now() - INTERVAL 30 DAY
SAMPLE BY 30m;
```

`SAMPLE BY 1d` aligns to UTC midnight by default. For session-boundary candles (say, 09:00–16:00 only), filter by session first, sample at 1m, and stitch in the application layer. Also: the candle for the *currently open* period is a live candle — its close is not the true close, so exclude it from backtests.

## 6. The big trap: resampling large candles from small ones

If you already pre-aggregate 1m candles, building 1h candles works, but high/low must come from the small candles' high/low:

```sql
SELECT
    ts,
    FIRST(open)  AS open,
    MAX(high)    AS high,
    MIN(low)     AS low,
    LAST(close)  AS close,
    SUM(volume)  AS volume
FROM candles_1m
SAMPLE BY 1h;
```

The common mistake is `MAX(close)` / `MIN(close)`: close is just one point inside the window and misses the true extremes. A second, subtler trap: if the small period does not divide the large period evenly (7-minute candles into an hour), alignment breaks and you have no choice but to recompute from the raw ticks.

## 7. Pre-aggregate a candle table for charts

Dashboards should not scan billions of ticks on every render. Pre-aggregating into a 1m candle table is the standard move:

```sql
CREATE TABLE candles_1m (
    ts TIMESTAMP,
    symbol SYMBOL,
    open DOUBLE,
    high DOUBLE,
    low DOUBLE,
    close DOUBLE,
    volume DOUBLE
) TIMESTAMP ts PARTITION BY DAY;

INSERT INTO candles_1m
SELECT
    ts,
    symbol,
    FIRST(price),
    MAX(price),
    MIN(price),
    LAST(price),
    SUM(volume)
FROM ticks
WHERE ts < now() - INTERVAL 1 MINUTE
SAMPLE BY 1m PARTITION BY symbol;
```

Run this on a cron schedule to incrementally write the last completed window. If your version does not accept `SAMPLE BY` inside `INSERT`, split it: SELECT into a temp table, then move the rows.

## 8. ASOF JOIN against an index

To correlate your stock's candles with an index whose sampling points do not line up, use `ASOF JOIN` to align to the nearest point on the same candle axis:

```sql
SELECT
    t.ts,
    t.close AS close,
    i.close AS index_close
FROM candles_1m t
ASOF JOIN candles_1m i ON t.ts = i.ts
WHERE t.symbol = '600519' AND i.symbol = 'SPY';
```

## 9. Performance intuition

`SAMPLE BY` runs on the columnar store with partition pruning: a time filter in `WHERE` touches only the matching partition, so a day of 1m candles finishes in milliseconds to seconds even with billions of ticks. The same job done with a row-by-row Python loop is orders of magnitude slower — push all aggregation into SQL.

## 10. What next

- **Charts**: put the candle table into a Grafana candlestick panel; see our zero-cost IoT dashboard article for wiring Grafana to QuestDB end to end.
- **Backtesting**: read 1m candles, align with `ASOF JOIN`, and compute strategy signals in your backtester.
- **Outliers**: rolling Z-score on 1m closes — see our anomaly detection article for the exact pattern.
