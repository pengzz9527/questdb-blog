---
title: "QuestDB Data Retention: DROP PARTITION, TTL, and Hot/Cold Tiering"
description: "How to implement QuestDB data retention properly: drop partitions physically with DROP PARTITION, inspect partition sizes, automate cleanup with an external scheduler or the enterprise TTL clause, then add hot/cold tiering with Parquet exports. Runnable SQL, real pitfalls, and disk-space verification for long-running time-series tables."
date: 2026-10-25
lastmod: 2026-10-25
slug: questdb-data-retention-lifecycle
tags:
  - QuestDB
  - Data Retention
  - DROP PARTITION
  - Partitioned Table
  - Tiering
categories:
  - In Practice
toc: true
---

# QuestDB Data Retention: DROP PARTITION, TTL, and Hot/Cold Tiering

Time-series tables only grow. That has no exceptions. One device writing once per second produces 31.5 million rows a year; a thousand devices produce 315 billion. Disk space never frees itself — and "you can delete" versus "deleting is clean and cheap" are different problems.

QuestDB's retention story has exactly one core action: **drop a partition**. Everything else — cron jobs, TTL clauses, hot/cold tiering — ultimately resolves to that operation. Understand it and retention is straightforward; skip it and you end up running `DELETE` over hundreds of millions of rows, saturating CPU while the disk usage barely moves.

## 1. Why DELETE is the wrong tool

A typical mistake:

```sql
DELETE FROM sensor_reading WHERE ts < '2026-04-01';
```

That parses and runs, but on a partitioned table it works row-by-row: find matches, write new versions, leave old versions for background cleanup. Deleting 300 million rows that way is expensive in time and I/O — and people often check disk usage afterwards and see it unchanged, because the old segments are still waiting for GC.

The right tool:

```sql
ALTER TABLE sensor_reading DROP PARTITION LIST '2026-03';
```

This is a metadata operation: the `2026-03` partition is unlinked from the table, file handles close, the directory entry disappears, and **space returns to the filesystem immediately**. Cost is independent of partition size — milliseconds to a few hundred milliseconds.

This is the real payoff of partitioning. The earlier post on `PARTITION BY` covered the mechanism; here is the consequence: **your retention granularity is physically locked to your partition granularity.** A `PARTITION BY DAY` table can only drop whole days, so "keep 45 days" is impossible — it's 45 or 46. A `MONTH` partitioned table can only drop by month.

## 2. See what partitions you have

```sql
-- single table
SELECT partition_name, num_rows, disk_size
FROM partitions('sensor_reading')
ORDER BY partition_name;

-- whole database
SELECT table_name, partition_name, num_rows
FROM table_partitions()
ORDER BY table_name, partition_name;
```

`partitions('table')` is a function scoped to one table; `table_partitions()` is a system view across tables. Both expose `num_rows` and `disk_size` — and `disk_size` is the field that actually drives retention decisions. What matters is not "how many days do I keep" but "which partition is eating the disk".

If one partition holds 500 million rows and another 3 million, the data distribution is skewed — a load test dumped into one day, or one device reported at an abnormal rate. Dropping purely by date treats both the same.

## 3. Drop one partition by hand and verify the space is back

```bash
docker compose exec questdb df -h /var/lib/questdb
# /dev/...  20G  14G  5G  74% /var/lib/questdb
```

```sql
ALTER TABLE sensor_reading DROP PARTITION LIST '2026-03';

SELECT * FROM partitions('sensor_reading') WHERE partition_name = '2026-03';
-- 0 rows
```

```bash
docker compose exec questdb df -h /var/lib/questdb
# /dev/...  20G  8G  11G  42% /var/lib/questdb
```

14G down to 8G: 6G returned to the filesystem. Do this check every time — any "deletion" after which disk usage doesn't fall was not a clean deletion.

## 4. Automating it without TTL

Open-source QuestDB has no built-in TTL clause (the enterprise edition offers `CREATE TABLE ... TTL 90 DAYS`, where the engine schedules partition drops itself). On the open-source build you schedule externally: a script computes which partitions are past their expiry and drops each one.

The script is shorter than expected:

```bash
#!/usr/bin/env bash
# prune_partitions.sh — run daily at 03:00
set -euo pipefail

QDB="http://localhost:9000/exec"
TABLE="sensor_reading"
RETAIN_DAYS=180

# cutoff partition: the DAY partition name (YYYY-MM-DD) RETAIN_DAYS ago
CUTOFF=$(date -u -d "-${RETAIN_DAYS} days" +%Y-%m-%d)

mapfile -t PARTS < <(
  curl -sG "$QDB" \
    --data-urlencode "query=SELECT partition_name FROM partitions('${TABLE}') WHERE partition_name < '${CUTOFF}' ORDER BY partition_name" \
  | jq -r '.dataset[] | .[0]'
)

for p in "${PARTS[@]}"; do
  echo "dropping ${TABLE} partition ${p}"
  curl -sG "$QDB" \
    --data-urlencode "query=ALTER TABLE ${TABLE} DROP PARTITION LIST '${p}'" > /dev/null
done
```

Three rules:

1. **Use `<`, not `=`.** A script that fails for a few days will catch every missed partition on the next run — that's where idempotency comes from.
2. **Never drop the partition being written.** Comparing against the cutoff naturally excludes today.
3. **Compute, log, then delete.** Dropping is irreversible; the log is your only record of what happened (the only real rollback is a backup).

Wire it into cron:

```cron
0 3 * * * /opt/qdb/prune_partitions.sh >> /var/log/qdb-prune.log 2>&1
```

On WAL tables, `DROP PARTITION` is executed asynchronously by the sequencer and does not block ingestion. That beats a hand-written `DELETE` by a wide margin — the hardest part of bulk deletion is "keep accepting writes while deleting".

## 5. Retention and partition granularity must line up

This is where most setups go wrong. Keep a table like this handy:

| Retention target | Partition granularity must be | Actually achievable | Note |
|---|---|---|---|
| 7 days | `DAY` or `HOUR` | 7–8 days | drops by day, may hold one extra |
| 30 days | `DAY` | 30–31 days | standard pairing |
| 90 days | `DAY` | 90–91 days | recommended for fine-grained tables |
| 6 months | `DAY` or `MONTH` | 6 months ± 1 month | `MONTH` means fewer files but coarser deletes |
| 45 days | `DAY` | 45–46 days | `MONTH` cannot do this |
| 2 years | `MONTH` | 24–25 months | 730 daily partitions is too many |

If a table already has the wrong granularity, `ALTER TABLE ... SET PARTITION BY ...` changes it:

```sql
ALTER TABLE sensor_reading SET PARTITION BY MONTH;
```

This does not rewrite history. Existing partitions keep their current layout; only new writes follow the new granularity. To fully restructure, create a new table, migrate with `INSERT INTO ... SELECT`, then rename. Decide the retention window before creating the table — an order of magnitude cheaper than fixing it later.

## 6. The other half of retention: downsample instead of delete

Dropping after 180 days throws data away. Most real requirements are actually "keep the trend, lose the detail". That makes retention a two-stage policy:

- 0–30 days: full detail (raw per-second data)
- 30–180 days: 5-minute rollups
- beyond 180 days: hourly rollups, kept long-term

In QuestDB this is `SAMPLE BY` plus `INSERT INTO ... SELECT`:

```sql
-- hourly table, long retention
CREATE TABLE sensor_reading_1h (
    ts          TIMESTAMP,
    device      SYMBOL,
    avg_temp    DOUBLE,
    max_temp    DOUBLE,
    min_temp    DOUBLE,
    samples     LONG
) TIMESTAMP(ts) PARTITION BY MONTH
  DEDUP UPSERT KEYS(ts, device);

-- collapse yesterday's detail into hourly rows (idempotent, safe to re-run)
INSERT INTO sensor_reading_1h
SELECT
    ts,
    device,
    avg(temperature),
    max(temperature),
    min(temperature),
    count()
FROM sensor_reading
WHERE ts IN '2026-10-24'
SAMPLE BY 1h ALIGN TO CALENDAR;
```

`DEDUP UPSERT KEYS(ts, device)` makes this re-runnable: replaying a day overwrites aggregate rows with identical values instead of duplicating them. The cron job doesn't need "exactly once" semantics — retry on failure.

Measure the savings:

```sql
SELECT
    (SELECT count() FROM sensor_reading WHERE ts IN '2026-10-24')     AS raw_rows,
    (SELECT count() FROM sensor_reading_1h WHERE ts IN '2026-10-24')  AS hourly_rows;
-- raw_rows >> hourly_rows; the ratio is your compression gain
```

Hourly rollups typically shrink row counts by up to 3600× for per-second data. Disk shrinks proportionally, while "peak temperature per hour over the past year" is still answerable.

## 7. Hot/cold tiering: keep hot locally, let cold leave

One step further: migrate old partitions off local disk. Two approaches.

**Approach A — export then drop.** QuestDB can `COPY` a query result to Parquet:

```sql
COPY (
    SELECT * FROM sensor_reading WHERE ts IN '2026-02'
) TO '/var/lib/questdb/export/2026-02.parquet' WITH FORMAT PARQUET;
```

Upload to object storage (`mc cp`, `aws s3 cp`, or a mounted bucket), verify file size and row count, then `DROP PARTITION`. Cold queries go to DuckDB or Athena reading Parquet directly — "look at this once a year" workloads do not belong in a hot database.

**Approach B — relocate the partition directory.** A QuestDB partition is an ordinary directory. Stop the table → `mv` the partition directory to a cold-storage mount → `mv` it back when needed. Stopping is required because the engine holds open file handles; moving a live partition makes it unreadable. Rehearse this on a copy before touching production. Modifying live partition directories without rehearsal is one of the few operations that can genuinely corrupt a table.

The trade-off is blunt:

| | All hot | Hot/cold tiering |
|---|---|---|
| Historical queries | fast | slow (mount or external tool) |
| Disk cost | grows linearly | cold data at object-storage prices |
| Operational load | low | medium (migration job + verification) |
| Restoring old data | not needed | needs a documented procedure |

While your total data is still in the tens of gigabytes, don't tier yet. Get partition dropping and downsampling right first; that's already most of the benefit.

## 8. Four things to verify before going live

| Check | How | Consequence if skipped |
|---|---|---|
| Disk actually decreases | `df -h` before and after a drop | You think you're saving space while segments queue for GC |
| Script is idempotent | run twice; the second run should print nothing | Retries either double-drop or abort midway |
| Today's partition is never dropped | print the candidate list including today | Today's data disappears and dashboards go dark |
| Retention matches audit/legal requirements | confirm with the business owner, not just engineers | You delete records you were required to keep |

The last one is the most commonly skipped. Engineering decides "180 days is plenty" while legal requires "transaction-related data for three years". Retention policy inputs should not come only from disk capacity.

## 9. Next steps

- **Choosing granularity** — `DAY` / `MONTH` / `HOURLY` trade-offs → PARTITION BY strategy
- **Aggregation syntax** — `ALIGN TO CALENDAR`, missing windows, multi-column rollups → SAMPLE BY guide
- **Getting data in** — Kafka pipelines and idempotent writes with DEDUP → Kafka real-time pipeline
- **Visualizing retention** — chart partition counts and disk usage in Grafana → Grafana IoT dashboard

---

*SQL and scripts verified against QuestDB latest (Docker) on Linux. `partitions()` requires QuestDB ≥ 6.6, `COPY ... WITH FORMAT PARQUET` ≥ 7.0, async partition drops on WAL tables ≥ 7.3. Enterprise TTL syntax differs from the open-source build; consult the official docs. QuestDB is a trademark of its owner; this site is an independent technical community, not affiliated with QuestDB.*
