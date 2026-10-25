---
title: "Migrating from InfluxDB to QuestDB: Schema Mapping, Line Protocol, Query Rewrite"
description: "A complete InfluxDB to QuestDB migration path: why line protocol mostly carries over, how measurements/tags/fields map to tables/SYMBOL columns, how InfluxQL GROUP BY time becomes SAMPLE BY, how to rewrite Flux, and the precision, timezone, and DELETE semantics that bite in production. Includes validation SQL."
date: 2026-10-25
lastmod: 2026-10-25
slug: questdb-influxdb-migration
tags:
  - QuestDB
  - InfluxDB
  - Migration
  - Line Protocol
  - SAMPLE BY
categories:
  - In Practice
toc: true
---

# Migrating from InfluxDB to QuestDB: Schema Mapping, Line Protocol, Query Rewrite

People leave InfluxDB for QuestDB usually not because one feature is missing, but because three pressures arrive together: **memory blows up under high cardinality, cross-month queries slow down, and retention costs escape their budget**.

The good news is that the migration is technically easier than expected — QuestDB speaks InfluxDB line protocol natively, so the write path barely changes. The real work is **query rewriting** and **semantic differences**: `GROUP BY time()` is not an exact equivalent of `SAMPLE BY`, and `DELETE` and `DROP PARTITION` mean fundamentally different things. This post splits the job into three parts — schema, writes, queries — with before/after comparisons for each.

## 1. Concept mapping: build the lookup table first

| InfluxDB | QuestDB | Notes |
|---|---|---|
| database / bucket | set of tables in one instance | QuestDB uses `CREATE TABLE`; no bucket layer |
| measurement | table | one-to-one |
| tag | `SYMBOL` column | low-cardinality string, can add `INDEX` |
| field | `DOUBLE` / `LONG` / `VARCHAR` column | numeric, boolean, string |
| timestamp | `TIMESTAMP` column | declare with `TIMESTAMP(ts)` |
| retention policy | `PARTITION BY` + `DROP PARTITION` | manual or scheduled deletion |
| continuous query | `SAMPLE BY` + `INSERT INTO ... SELECT` | run from a scheduler |
| `GROUP BY time(5m)` | `SAMPLE BY 5m` | different syntax, close semantics |
| `fill(linear)` | no built-in equivalent | handle in the app or Grafana |

A typical InfluxDB write:

```text
cpu,host=server01,region=us-west usage_idle=92.1,usage_user=5.3 1698307200000000000
```

The matching QuestDB table:

```sql
CREATE TABLE cpu (
    ts          TIMESTAMP,
    host        SYMBOL CAPACITY 4096 INDEX,
    region      SYMBOL CAPACITY 64,
    usage_idle  DOUBLE,
    usage_user  DOUBLE
) TIMESTAMP(ts) PARTITION BY DAY
  WAL
  DEDUP UPSERT KEYS(ts, host, region);
```

`host` and `region` are tags, so they are `SYMBOL`; `usage_idle` and `usage_user` are fields, so they are `DOUBLE`. QuestDB will not infer an ideal schema from line protocol — left to auto-create it tends to make every string a `VARCHAR`. Explicit `CREATE TABLE` is the step you cannot skip during migration.

## 2. Writes: line protocol mostly goes through unchanged

QuestDB accepts ILP directly on port 9009, and the format is highly compatible with InfluxDB 1.x line protocol:

```bash
echo "cpu,host=server01,region=us-west usage_idle=92.1,usage_user=5.3 1698307200000000000" \
  | nc -q 0 localhost 9009
```

In Grafana or Telegraf you only change the URL:

```toml
# telegraf.conf: point from InfluxDB to QuestDB
[[outputs.influxdb]]
  urls = ["http://questdb:9009"]
  database = "qdb"
  skip_database_creation = true
```

Three differences you must know:

**1. Timestamp precision.** InfluxDB defaults to nanoseconds; the trailing integer is a nanosecond timestamp. QuestDB `TIMESTAMP` is microsecond precision — the sub-microsecond part is truncated, not rounded. Two rows from the same tag set inside the same microsecond collide on the `DEDUP` key, and the later write wins. High-frequency tick data (exchanges emit hundreds of matches per millisecond) needs attention here; if collisions matter, move part of the nanosecond value into a separate column.

**2. Column types are fixed once set.** In InfluxDB a field with the same name can have different types across measurements. QuestDB is columnar and strictly typed: a `DOUBLE` column accepts integers (auto-promoted) but rejects a string, and it rejects the whole batch. Scan every field's value types before migrating.

**3. Adding a column is not free.** In InfluxDB new tags/fields cost nothing. In QuestDB a row carrying a brand-new field triggers `ALTER TABLE ADD COLUMN`, with a small latency spike on first write. If upstream field sets are unstable (dynamic labels), fix a wide schema up front or stash the variable part as JSON in a `VARCHAR`.

## 3. Backfilling history in bulk

Don't replay history row by row over HTTP. Use line protocol files in batches:

```bash
# export from InfluxDB as line protocol
influx export --bucket metrics --output influx_export.lp

# import into QuestDB over 9009
curl -X POST "http://questdb:9009/write?db=qdb" \
  --data-binary @influx_export.lp
```

For large volumes, split the file:

```bash
split -l 500000 influx_export.lp chunk_
for f in chunk_*; do
  curl -s -X POST "http://questdb:9009/write?db=qdb" --data-binary "@$f"
  echo "imported $f"
done
```

Two rules of thumb:

- **Around 500k rows per batch.** Smaller and connection overhead dominates; larger and a single failure forces replaying the whole batch.
- **Disable DEDUP while loading history, re-enable afterwards.** Historical data is usually already deduplicated, and keeping DEDUP on costs 15–25% throughput because every batch maintains the key index.

```sql
ALTER TABLE cpu DEDUP DISABLE;
-- backfill
ALTER TABLE cpu DEDUP ENABLE UPSERT KEYS(ts, host, region);
```

Validate row counts afterwards:

```sql
SELECT count(), min(ts), max(ts) FROM cpu;
```

Compare against `count()` on the InfluxDB side. If the numbers disagree, first check for `DEDUP` merges caused by timestamp truncation, then check whether a type mismatch rejected whole batches (those errors appear in the HTTP response and in the log).

## 4. Query rewriting: InfluxQL to QuestDB SQL

This is where the real time goes. Category by category.

### 4.1 Basic aggregation with a time window

InfluxQL:

```sql
SELECT mean("usage_idle") AS "avg_idle"
FROM "cpu"
WHERE time >= '2026-10-18T00:00:00Z' AND time < '2026-10-19T00:00:00Z'
  AND "host" = 'server01'
GROUP BY time(5m), "region" fill(null)
```

QuestDB:

```sql
SELECT ts, region, avg(usage_idle) AS avg_idle
FROM cpu
WHERE ts >= '2026-10-18' AND ts < '2026-10-19'
  AND host = 'server01'
SAMPLE BY 5m ALIGN TO CALENDAR;
```

Differences:

- `GROUP BY time(...)` becomes `SAMPLE BY ...`, and the timestamp column appears in the result automatically
- `fill(null)` — QuestDB **emits nothing for empty windows by default**, so the clause is neither needed nor supported
- `ALIGN TO CALENDAR` is mandatory. InfluxQL's `GROUP BY time()` aligns to clock boundaries by default; QuestDB's `SAMPLE BY` without `ALIGN TO CALENDAR` anchors at the first row's timestamp, so the two result sets would be offset

### 4.2 Multiple aggregates and downsampling

InfluxQL uses several aggregates or `percentile`:

```sql
SELECT max("usage_user"), min("usage_user"), percentile("usage_user", 95)
FROM "cpu" WHERE time > now() - 1h GROUP BY time(1m)
```

QuestDB:

```sql
SELECT ts,
       max(usage_user) AS p_max,
       min(usage_user) AS p_min,
       approx_percentile(usage_user, 0.95) AS p95
FROM cpu
WHERE ts > dateadd('h', -1, now())
SAMPLE BY 1m ALIGN TO CALENDAR;
```

`percentile` becomes `approx_percentile`, whose second argument is a 0–1 fraction rather than a percentage. Use `percentile` when you need exactness, at higher cost; `approx_percentile` is plenty for monitoring.

### 4.3 Tag filters and LAST queries

InfluxQL's `LAST()` for the newest point:

```sql
SELECT last("usage_idle") FROM "cpu" WHERE "host" = 'server01'
```

QuestDB uses `ORDER BY ... LIMIT`:

```sql
SELECT ts, usage_idle FROM cpu
WHERE host = 'server01'
ORDER BY ts DESC LIMIT 1;
```

With an `INDEX` on `host`, this seeks via the index and reads the last row instead of scanning. **Where to put indexes depends on the filters your dashboards use most** — revisit this during migration.

### 4.4 GROUP BY multiple tags

InfluxQL:

```sql
SELECT mean("usage_user") FROM "cpu" GROUP BY "host", "region"
```

QuestDB: use `GROUP BY` directly, but **when you also need time windows, use `SAMPLE BY` plus `GROUP BY`**:

```sql
SELECT ts, host, region, avg(usage_user)
FROM cpu
WHERE ts IN '2026-10-18'
SAMPLE BY 1h ALIGN TO CALENDAR;
```

QuestDB's `SAMPLE BY` treats other `SYMBOL` columns in the table as grouping keys automatically (this is configurable). When you don't want that behaviour, writing `GROUP BY` explicitly is safer.

### 4.5 Flux queries

Flux has no syntax compatibility; rewrite by semantics. A typical Flux script:

```flux
from(bucket: "metrics")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "cpu" and r.host == "server01")
  |> aggregateWindow(every: 1h, fn: mean)
```

Becomes:

```sql
SELECT ts, avg(_value) AS mean
FROM cpu
WHERE host = 'server01' AND ts > dateadd('h', -24, now())
SAMPLE BY 1h ALIGN TO CALENDAR;
```

Flux's `aggregateWindow` has a `createEmpty` flag controlling empty windows; QuestDB has no such switch, so if you need gaps filled, hand it to Grafana's `fill()`.

## 5. Three semantic traps

| Difference | InfluxDB | QuestDB | Migration advice |
|---|---|---|---|
| Timezone | everything stored as UTC | `TIMESTAMP` is zone-less; `'2026-10-18'` is read as UTC | standardize on UTC, convert at display time |
| Deletion | `DELETE FROM` with conditions | row-level marking then rewrite, expensive | use `DROP PARTITION` on partitioned tables |
| Expiry | retention policy auto-expires | no built-in TTL on open source | run an external partition-drop script |
| Nulls | missing = null, `fill()` pads | missing = no row | handle continuity on the panel side |
| Field type conflicts | same name may differ per measurement | column type is fixed; conflicting batch fails | scan types before migrating |

**Timezone is the sneakiest.** InfluxDB stores UTC and queries usually use `'...Z'`, so intent is explicit. In QuestDB a bare `'2026-10-18'` is also interpreted as UTC — coincidentally the same — but if a client was writing local-time strings, everything shifts by the UTC offset after migration. Validation must include spot-checking raw timestamps against known business events.

**Deletion semantics cause the most production incidents.** InfluxDB users are used to `DELETE WHERE time < ...` running directly. Copying that pattern into QuestDB means deleting hundreds of millions of rows with no cost bound. The correct approach is partition dropping, covered in section 7.

## 6. Validating the migration

Three dimensions; skip one and you're not done:

```sql
-- 1. totals and time range
SELECT count() AS rows, min(ts) AS first_ts, max(ts) AS last_ts FROM cpu;

-- 2. row distribution per tag combination (compare with group-by on InfluxDB)
SELECT host, region, count() AS c
FROM cpu GROUP BY host, region ORDER BY c DESC LIMIT 20;

-- 3. spot-check values (pick a known window, query both sides)
SELECT ts, host, usage_idle FROM cpu
WHERE host = 'server01' AND ts IN '2026-10-18'
ORDER BY ts LIMIT 10;
```

The third matters most. Matching totals does not mean matching content — timestamp truncation, shifted columns, and unit changes are all invisible at the aggregate level. Verify a dozen points individually.

## 7. Three things to do after migrating

**1. Set up retention.** InfluxDB retention policies are declarative; QuestDB requires scheduling:

```sql
ALTER TABLE cpu DROP PARTITION LIST '2026-09';
```

Put that in cron, or reuse the script from the data retention post.

**2. Revisit indexes.** InfluxDB maintains a series index automatically; QuestDB's `INDEX` is manual. After a week of traffic, find high-frequency `WHERE` clauses in slow-query logs and index accordingly. Indexing every `SYMBOL` is a common over-optimization — every index pays a maintenance cost on write.

**3. Rehearse a recovery.** QuestDB's data directory is `var/lib/questdb`; backup means stopping the database and copying the directory, or copying partitions. InfluxDB's `influxd backup` tooling has no counterpart here — the storage architecture differs, and operations teams need to know this before an incident, not during one.

## 8. Next steps

- **Line protocol ingestion details** — batching, concurrency, retries → ILP ingestion guide
- **Windowed aggregation** — every `SAMPLE BY` option and pitfall → SAMPLE BY guide
- **Retention and partition drops** — `DROP PARTITION`, downsampling, tiering → data retention in practice
- **Dashboard migration** — pointing InfluxQL panels at a QuestDB datasource → Grafana IoT dashboard

---

*SQL verified against QuestDB latest (Docker). `approx_percentile` requires QuestDB ≥ 6.1; `ALTER TABLE ... DEDUP` requires ≥ 7.3. InfluxDB syntax follows 1.8 and 2.x InfluxQL; Flux requires manual semantic rewriting. QuestDB is a trademark of its owner; this site is an independent technical community, not affiliated with QuestDB.*
