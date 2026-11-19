---
title: "Power Load Curve Analysis: QuestDB Metering Data and Peak Demand"
description: "Power load curves are the archetypal time-series workload: fixed-interval sampling, many independent meters, cross-meter aggregation, peak/off-peak pricing windows. This article stores 15-minute metering data in QuestDB and gives four core query families: load curves, coincident and non-coincident peak demand, load factor, and peak/off-peak energy split."
date: 2026-11-15
lastmod: 2026-11-15
slug: questdb-power-load-analysis
tags:
  - QuestDB
  - Energy
  - Metering
  - Load Curve
  - SAMPLE BY
categories:
  - In Practice
toc: true
---

# Power Load Curve Analysis: QuestDB Metering Data and Peak Demand

A power load curve is the archetypal time-series workload: fixed-interval sampling, a large number of independent meters, aggregation across meters, and peak/off-peak pricing windows. If you run demand-charge billing or energy cost analysis, the queries below are the ones you run every day. This article goes from the data model to the four core analyses — load curve, peak demand, load factor, peak/off-peak energy — all as runnable SQL.

## 1. Data model: two signals, do not mix them

Metering gateways usually emit two kinds of signals:

| Signal | Unit | Meaning | Used for |
|---|---|---|---|
| `power_kw` | kW | interval-average power | load curve, peak demand |
| `energy_kwh` | kWh | interval consumption (already differenced) | energy cost, peak/off-peak split |

If your meters only send a **cumulative register** (a monotonically increasing counter), difference it on the gateway side to get interval consumption; if you must keep the counter in the database, you can difference it in SQL instead (section 6).

```sql
CREATE TABLE meter (
    ts TIMESTAMP,
    site SYMBOL,
    meter SYMBOL,
    power_kw DOUBLE,
    energy_kwh DOUBLE
) TIMESTAMP ts PARTITION BY MONTH;
```

Writing via ILP (15-minute interval, one meter one line):

```
meter,site=HQ,meter=m01 power_kw=128.4,energy_kwh=31.6 1700000000000000000
meter,site=HQ,meter=m02 power_kw=96.2,energy_kwh=23.9  1700000000000000000
```

100 meters × 15-minute interval is 9.6 million rows a year — a small table for QuestDB; thousands of meters is still fine.

## 2. Load curve: 15-minute granularity

```sql
SELECT
    ts,
    SUM(power_kw) AS power_kw
FROM meter
WHERE site = 'HQ'
  AND ts >= '2026-11-14'::timestamp
SAMPLE BY 15m;
```

One row is one 15-minute window; `power_kw` is the sum of every meter in the site during the window — that is the load curve. To see the daily shape, switch to `SAMPLE BY 1h`.

## 3. Peak demand: two definitions

**Non-coincident peak** — the maximum interval power of each meter:

```sql
SELECT site, meter, power_kw AS peak_kw, ts AS peak_ts
FROM meter
ORDER BY power_kw DESC
LIMIT 100;
```

Demand-charge billing uses exactly this: demand charge = peak demand × demand tariff, so you need the peak and the moment it happened for every meter.

**Coincident peak** — the window in which the sum across all meters is largest:

```sql
SELECT ts, SUM(power_kw) AS total_kw
FROM meter
WHERE ts >= '2026-11-01'::timestamp
SAMPLE BY 15m
ORDER BY total_kw DESC
LIMIT 10;
```

Row one is the month's coincident peak, and `ts` tells you exactly when the peak happened.

## 4. Load factor

Load factor = average load / peak load. Near 1 means a flat load and high capacity utilization; too low means the peak was a short spike and you may have paid a heavy demand charge for nothing:

```sql
SELECT AVG(load) / MAX(load) AS load_factor
FROM (
    SELECT SUM(power_kw) AS load
    FROM meter
    WHERE site = 'HQ' AND ts >= '2026-11-01'::timestamp
    SAMPLE BY 15m
);
```

## 5. Peak/off-peak energy: CASE WHEN on time windows

Peak/off-peak tariffs follow **local time**, but the database stores UTC. Convert with `AT TIME ZONE`:

```sql
SELECT
    SUM(CASE WHEN EXTRACT(HOUR FROM ts AT TIME ZONE 'Asia/Shanghai')
             BETWEEN 8 AND 19 THEN energy_kwh ELSE 0 END) AS peak_energy_kwh,
    SUM(CASE WHEN EXTRACT(HOUR FROM ts AT TIME ZONE 'Asia/Shanghai')
             BETWEEN 8 AND 19 THEN 0 ELSE energy_kwh END) AS offpeak_energy_kwh
FROM meter
WHERE site = 'HQ'
  AND ts >= '2026-11-01'::timestamp;
```

Set the peak/off-peak boundary to your region's policy (the example uses 08:00–20:00). Multiply the two energy buckets by the peak and off-peak tariffs and you have the power bill.

## 6. Register-based variant: LAG difference + reset detection

If you stored the counter, difference it with `LAG`:

```sql
SELECT
    ts,
    meter,
    energy_kwh - LAG(energy_kwh) OVER (PARTITION BY meter ORDER BY ts) AS interval_kwh
FROM meter
WHERE site = 'HQ';
```

Two traps: each meter's first row has no previous value, so the difference is NULL — filter it out; and a **meter reset** drops the counter to zero, making the difference negative. Flag negatives as resets and distrust that difference.

## 7. Long-term retention: two years raw, then roll down to hourly

The 15-minute raw data is worth keeping for two years; after that, drop old monthly partitions (the full retention workflow is in our data retention article):

```sql
ALTER TABLE meter DROP PARTITION LIST '2024-03';
```

Anything older than two years is rolled down to hourly granularity into `meter_1h` (`SUM(energy_kwh)`, `AVG(power_kw)`, window-start timestamps), and the detail table's old partitions are dropped. Long-term storage cost drops to about a twelfth.

## 8. Pitfall list

- Mixing interval consumption and cumulative registers in one table — energy analysis is silently wrong.
- A gateway outage leaves 15-minute gaps: `SAMPLE BY` skips empty windows by default, so the curve has holes; use `SAMPLE BY 15m FILL PREVIOUS` if you want them filled.
- Peak/off-peak windows computed in UTC — you must `AT TIME ZONE`, or the split is off by an hour during DST periods.
- `MAX(power_kw)` is a single-point peak, not a window peak — align with the billing interval and use 15-minute granularity.

## 9. What next

- Load curve + coincident peak in Grafana: our zero-cost IoT dashboard article covers wiring the data source and panels.
- Rolling Z-score on the hourly curve to catch load spikes: our anomaly detection article.
- Write concurrency at thousands of meters: our ILP high-concurrency tuning article.

## References

- [QuestDB docs: SAMPLE BY](https://questdb.io/docs/reference/sql/sample-by/)
- [QuestDB docs: INSERT / Line Protocol](https://questdb.io/docs/reference/sql/insert/)

## Related reading

- [QuestDB SAMPLE BY Complete Guide: The Soul of Time-Series Aggregation](/en/post/questdb-sample-by-complete-guide/)
- [QuestDB ILP Ingestion Complete Guide: From Single Rows to High-Throughput Batch](/en/post/questdb-ilp-ingestion-guide/)
- [QuestDB + Kafka Real-Time Pipeline: ILP Ingestion to SAMPLE BY Rollups](/en/post/questdb-kafka-connect-pipeline/)
