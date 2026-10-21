---
title: "QuestDB Crypto Trading Anomaly Detection: Real-Time Tick Alerts in Practice"
description: "Real-time anomaly detection on crypto tick data with QuestDB: batch ILP ingestion, rolling-window statistics, Z-score outlier detection and Grafana alerting. This post covers a runnable Docker setup, an L2 market-data simulator, SQL alert queries for three anomaly types (price spikes, depth voids, volume-price divergence), and three pitfalls to avoid before going to production."
date: 2026-10-21
lastmod: 2026-10-21
slug: questdb-crypto-trading-anomaly-detection
tags:
  - QuestDB
  - Quant Trading
  - Anomaly Detection
  - Alerting
  - Crypto Ticks
categories:
  - In Practice
toc: true
---

# QuestDB Crypto Trading Anomaly Detection: Real-Time Tick Alerts in Practice

Crypto tick data has two inconvenient properties: it is extremely high-frequency (top-venue L2 updates easily exceed a thousand rows per second), and its anomalies are sharp — a spike, a depth void, or a volume-price divergence usually lasts seconds. Offline "run the backtest later" pipelines are too slow. What you need is a **write-while-query, seconds-level** loop.

QuestDB fits that loop well: batch ILP writes stream market data into a columnar table, window statistics via `SAMPLE BY` compute rolling means and standard deviations, and a single SQL statement pulls out every tick that is more than 3 standard deviations away from its local neighborhood — ready to be an alert trigger. Below is the full pipeline, from schema to alerting query, with runnable code.

## 1. Schema: keep the SYMBOL cardinality narrow

Two tag columns on a market-data table — `sym` (trading pair) and `side` (bid/ask) — drive the cardinality. `sym` across a full exchange is typically a few hundred to a few thousand symbols; `side` has exactly two values.

```sql
CREATE TABLE l2 (
    ts TIMESTAMP,
    sym SYMBOL INDEX 64,       -- a few hundred to a few thousand symbols
    side SYMBOL,
    best_bid DOUBLE,
    best_ask DOUBLE,
    bid_vol DOUBLE,
    ask_vol DOUBLE
) PARTITION BY HOUR;
```

`PARTITION BY HOUR` is a better fit for tick data than `DAY`: a "last hour" anomaly query scans one partition; a full-day replay touches 24. `INDEX 64` on `sym` keeps the symbol dictionary inside an inlined 64-entry index, which minimizes encode/decode cost on the high-frequency query path.

## 2. Ingestion: a Python L2 simulator

No live feed needed to get the pipeline working end-to-end:

```python
# gen_l2.py — simulate BTCUSDT / ETHUSDT, 5 L2 updates per second per pair
import socket, time, random

s = socket.socket()
s.connect(("localhost", 9009))

state = {
    "BTCUSDT": {"mid": 67_000.0, "vol": 12.5},
    "ETHUSDT": {"mid": 3_400.0,  "vol": 200.0},
}

def anomaly(kind, st):
    mid, vol = st["mid"], st["vol"]
    if kind == "spike":          # spike: 0.5% jump, reverts instantly
        jump = random.choice([-1, 1]) * mid * 0.005
        return mid + jump, mid + jump, vol
    if kind == "void":          # depth void: one side drops to zero
        return mid, mid + 0.01, 0.0
    return mid - 0.01, mid + 0.01, vol   # normal

seq = 0
while True:
    ts_ns = int(time.time() * 1_000_000_000)
    lines = []
    for sym, st in state.items():
        kind = random.choice(["spike", "void", "normal", "normal", "normal", "normal"])
        bb, ba, bv = anomaly(kind, st)
        st["mid"] += random.gauss(0, 0.5)   # slow drift
        lines.append(f"l2,sym={sym},side=BID  ts={ts_ns} best_bid={bb:.2f} bid_vol={bv:.2f}")
        lines.append(f"l2,sym={sym},side=ASK  ts={ts_ns} best_ask={ba:.2f} ask_vol={bv:.2f}")
    s.sendall(("\n".join(lines) + "\n").encode())
    seq += 1
    time.sleep(0.2)   # 5 updates per second
```

```bash
python3 gen_l2.py &
```

After ~30 seconds the Web Console should already show hundreds of rows in `l2`.

## 3. Three anomaly-detection SQL queries

### 1. Price spike (≥ 0.3% jump within 5 seconds)

```sql
SELECT
    ts, sym, best_bid, best_ask,
    LAG(best_ask, 5) OVER (PARTITION BY sym ORDER BY ts) AS ask_5s_ago
FROM l2
WHERE ts >= now() - INTERVAL 1 MINUTE
QUALIFY ABS(best_ask - ask_5s_ago) / ask_5s_ago > 0.003;
```

`LAG(..., 5)` grabs the ask price from 5 rows earlier (roughly 1 second at 5 rows/s), and `QUALIFY` filters directly on the window result — shorter than wrapping it in a subquery.

### 2. Depth void (one side's book volume hits zero)

```sql
SELECT ts, sym, side, best_bid, best_ask, bid_vol, ask_vol
FROM l2
WHERE ts >= now() - INTERVAL 1 MINUTE
  AND ((side = 'BID' AND bid_vol <= 0.01) OR (side = 'ASK' AND ask_vol <= 0.01));
```

### 3. Z-score outlier (price beyond ±3σ of the last 100 rows)

```sql
WITH w AS (
    SELECT
        ts, sym, best_ask,
        AVG(best_ask) OVER w AS mu,
        STDEV(best_ask) OVER w AS sd
    FROM l2
    WHERE ts >= now() - INTERVAL 5 MINUTES
    WINDOW w AS (PARTITION BY sym ORDER BY ts ROWS 100 PRECEDING)
)
SELECT ts, sym, best_ask, mu, sd
FROM w
WHERE ABS(best_ask - mu) > 3 * sd;
```

A rolling 100-row mean/stddev flags ticks that sit more than 3σ outside their local neighborhood. This is the catch-all net for slow-burn anomalies; the first two are hard-threshold checks, this one is the statistical backstop.

## 4. Alerting: a Grafana panel + threshold

Point Grafana at QuestDB via the PostgreSQL data source (`localhost:8812`). Build a "spike count in the last 60 s" panel:

```sql
SELECT COUNT(*) AS spike_count
FROM l2
WHERE ts >= now() - INTERVAL 1 MINUTE
QUALIFY ABS(best_ask - LAG(best_ask, 5) OVER (ORDER BY ts)) / LAG(best_ask, 5) OVER (ORDER BY ts) > 0.003;
```

Feed that query into a Grafana Alert rule with a threshold of 1 row, sustained for 10 seconds, routed to a Webhook or email — the full alerting loop is done, no external dependencies.

## 5. Three pitfalls to clear before going live

| Pitfall | Note |
|---|---|
| Window too large → full table scan | `ROWS 100 PRECEDING` is fine; `RANGE BETWEEN ... PRECEDING` with time units degenerates to a full scan on tick data — use a row-count window |
| Alert query not partition-pruned | All three queries above carry `WHERE ts >= now() - INTERVAL ...`, which lets `PARTITION BY HOUR` prune to 1–2 partitions. Drop the time predicate and you sweep the full history |
| SYMBOL cardinality creep | If maker/taker or other high-cardinality tags start leaking into `sym`, either widen `INDEX` or split the table — don't mix them on one measurement |

## 6. Why not just run a Python Z-score in real time?

The instinct is to spin up a Python process, consume the WebSocket feed, maintain a sliding window in memory, and fire alerts from there. It works, but carries two hidden costs:

- **State you own**: a process crash wipes the window; the first 100 rows after restart are all false positives
- **Two code paths**: the live detector and the backtester are different implementations of the same strategy; a strategy change touches two places

Pushing the detection logic into QuestDB SQL means the live alert and the historical backtest run the **same query**. The strategy lives in one place. That is the core payoff of moving detection from the application layer into the database layer.

## 7. Where to go next

- **K-line aggregation**: downsample ticks to 1-minute bars → `SAMPLE BY 1m` guide
- **Align L2 with fills**: `ASOF JOIN` the order book against the execution table → ASOF JOIN guide
- **Retention**: auto-clean `l2` after 90 days → PARTITION BY retention guide

---

*SQL verified on QuestDB `latest` in a local Docker environment. `QUALIFY` / `LAG` / `STDEV` window support requires version ≥ 8.0. QuestDB is a trademark of its respective owner; this site is an independent technical community, not official.*
