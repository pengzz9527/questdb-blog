---
title: "QuestDB + Kafka Real-Time Pipeline: ILP Ingestion to SAMPLE BY Rollups"
description: "A practical QuestDB Kafka pipeline: use Kafka Connect to convert topic messages into ILP writes, then roll up with SAMPLE BY. Includes Docker Compose orchestration, table design, at-least-once duplicate handling with DEDUP, partition alignment, and lag troubleshooting, plus runnable SQL for a production-ready real-time time-series path."
date: 2026-10-25
lastmod: 2026-10-25
slug: questdb-kafka-connect-pipeline
tags:
  - QuestDB
  - Kafka
  - ILP
  - SAMPLE BY
  - Streaming
categories:
  - In Practice
toc: true
---

# QuestDB + Kafka Real-Time Pipeline: ILP Ingestion to SAMPLE BY Rollups

The typical time-series path inside a business system looks like this: devices or services push metrics into Kafka, and several downstream consumers each do their own thing. When someone later asks for history or a dashboard, Kafka turns out to be poor at range-scanning by time. So the chain grows into Kafka → a batch job → yet another store, with one more job for a human to babysit.

This post takes a different route: **Kafka as the buffer, Kafka Connect converting messages straight into ILP writes against QuestDB, and SAMPLE BY producing the rollups on the query side.** No custom consumer code — just configuration and one CREATE TABLE. Below covers table design through lag troubleshooting, all reproducible locally with Docker.

## 1. Why put QuestDB behind Kafka

The two components have clearly separated jobs, and mixing them up causes trouble:

| Stage | Job | Not its job |
|---|---|---|
| Kafka | Absorb spikes, fan out to consumers, short replay (per retention) | Time-window aggregation, long-term storage |
| QuestDB | Land the data, partition by time, window aggregation, long-range queries | Message distribution, transactional compensation |

Kafka's weak point is **retention**. Default is 7 days; raising it costs disk linearly, and running "P99 per hour over the last 90 days" means sequentially scanning every segment with unpredictable latency. QuestDB's `PARTITION BY DAY` plus columnar storage covers exactly that gap: a one-week query opens 7 partition files.

The reverse is equally wrong. Using QuestDB as a message queue has no consumer-offset concept — multiple consumers have to track "how far did I read" with timestamps, far messier than a Kafka offset.

## 2. Message format decides everything

The QuestDB sink for Kafka Connect speaks ILP (InfluxDB Line Protocol). So the message should already carry ILP-shaped semantics:

```json
{
  "ts": "2026-10-25T03:12:44.123456Z",
  "device": "sensor-042",
  "site": "sh-01",
  "temperature": 23.7,
  "humidity": 41.2,
  "battery": 0.87
}
```

Mapped to a table, there are three decisions:

```sql
CREATE TABLE sensor_reading (
    ts          TIMESTAMP,
    device      SYMBOL CAPACITY 4096 INDEX,
    site        SYMBOL CAPACITY 256,
    temperature DOUBLE,
    humidity    DOUBLE,
    battery     DOUBLE
) TIMESTAMP(ts) PARTITION BY DAY
  WAL
  DEDUP UPSERT KEYS(ts, device);
```

One by one:

- **`device` as SYMBOL** — device IDs are repeated strings. SYMBOL is dictionary-encoded internally: comparisons are integer comparisons, and storage holds dictionary IDs. With a few thousand devices, that column drops from string-length bytes to 4 bytes per row.
- **`INDEX`** — supports `WHERE device = 'sensor-042'` style equality filters. If you mostly aggregate by `site`, put the index there instead. Indexes are not free: each one is maintained on every write.
- **`DEDUP UPSERT KEYS(ts, device)`** — the single most overlooked line in this article and the one that causes the most 3 a.m. incidents. Section 5 covers it.

`WAL` tables (write-ahead log) are the recommended default since QuestDB 7.3: writes land in the WAL first and a background process applies them to partitions, so ingestion and queries don't block each other.

## 3. Bring up the whole chain with Docker Compose

Three containers: Kafka in KRaft mode (no ZooKeeper), Kafka Connect, and QuestDB.

```yaml
services:
  kafka:
    image: apache/kafka:3.9.0
    ports: ["9092:9092"]
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1

  questdb:
    image: questdb/questdb:latest
    ports: ["9000:9000", "9009:9009", "8812:8812"]
    volumes: ["questdb-data:/var/lib/questdb"]

  connect:
    image: questdb/kafka-questdb-connector:latest
    depends_on: [kafka, questdb]
    ports: ["8083:8083"]
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: qdb-sink
      CONNECT_CONFIG_STORAGE_TOPIC: _connect_configs
      CONNECT_OFFSET_STORAGE_TOPIC: _connect_offsets
      CONNECT_STATUS_STORAGE_TOPIC: _connect_status
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE: "false"
      CONNECT_PLUGIN_PATH: /kafka/connect

volumes:
  questdb-data:
```

`CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE: "false"` is mandatory. With it on, messages must be wrapped with a schema envelope; bare JSON emitted by normal applications fails to deserialize. The sink logs `Cannot deserialize`, which looks like a connectivity problem but is actually a converter misconfiguration.

Confirm all three services are alive:

```bash
docker compose up -d
docker compose ps
curl -s localhost:8083/connector-plugins | jq '.[].class'
# expect io.questdb.kafka.QuestDBSinkConnector
```

## 4. Register the sink and wire a topic into the table

Create the topic and feed test data:

```bash
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --create --topic sensor-raw --partitions 4
```

Register the connector:

```bash
curl -X POST localhost:8083/connectors -H 'Content-Type: application/json' -d '{
  "name": "questdb-sensor-sink",
  "config": {
    "connector.class": "io.questdb.kafka.QuestDBSinkConnector",
    "topics": "sensor-raw",
    "tasks.max": "4",
    "host": "questdb:9009",
    "table": "sensor_reading",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "timestamp.field.name": "ts",
    "symbols": "device,site",
    "include.key": "false"
  }
}'
```

A few parameters deserve their own paragraph:

- **`tasks.max` should match partition count.** Four partitions with four tasks gives each task its own ILP connection and saturates write parallelism. Extra tasks beyond the partition count just idle.
- **`symbols`** explicitly lists fields to treat as SYMBOL. Omit it, and string fields become VARCHAR; changing back to SYMBOL later requires rebuilding the table.
- **`timestamp.field.name`** points at the time field inside the message. Without it the connector uses consumption time — so replaying historical data stamps everything with "now". This is the classic mistake when backfilling.
- **`auto.create`** defaults to creating a missing table. Convenient, but auto-created tables have no SYMBOL columns, no partition strategy, no DEDUP. In production, create the table explicitly and disable auto-creation.

## 5. Duplicates: why DEDUP is not optional

Kafka Connect delivers at-least-once. A task restarting before committing offsets means the same batch is consumed again. Without handling, `sensor_reading` accumulates rows with identical timestamps and identical devices.

`DEDUP UPSERT KEYS(ts, device)` makes QuestDB deduplicate on the key set while applying the WAL: same key means the later row overwrites the earlier one. The effect is idempotent writes.

Verify it. Insert a row, then insert an updated version with the same key:

```sql
INSERT INTO sensor_reading VALUES
  ('2026-10-25T10:00:00.000000Z', 'sensor-042', 'sh-01', 23.7, 41.2, 0.87);

INSERT INTO sensor_reading VALUES
  ('2026-10-25T10:00:00.000000Z', 'sensor-042', 'sh-01', 23.9, 41.5, 0.86);

SELECT ts, device, temperature FROM sensor_reading
WHERE device = 'sensor-042';
```

One row comes back with `temperature = 23.9`. Without DEDUP there would be two.

Three limits to keep in mind:

1. DEDUP keys must be `TIMESTAMP` plus `SYMBOL` (or string/integer) columns — no DOUBLE columns, floating point is an unreliable key.
2. Duplicates inside the same WAL transaction batch are deduplicated. Manual `INSERT`s that cross a partition boundary (say, a timestamp edited into another month) are not guaranteed, so watch out when backfilling history.
3. DEDUP costs something. Each batch maintains a key index; measured write throughput drops roughly 15–25% in exchange for replay safety. If messages are naturally unique (carry a UUID), you can skip it.

## 6. Query side: minute rollups with SAMPLE BY

What the dashboard needs is not raw rows but "5-minute average per device".

```sql
SELECT
    ts,
    device,
    avg(temperature) AS avg_temp,
    max(temperature) - min(temperature) AS swing,
    avg(battery)     AS avg_battery
FROM sensor_reading
WHERE ts IN '2026-10-25'
  AND device IN ('sensor-042', 'sensor-077')
SAMPLE BY 5m ALIGN TO CALENDAR;
```

`ALIGN TO CALENDAR` snaps windows to clock boundaries (10:00, 10:05, 10:10) instead of starting from the first row's timestamp. Panels and cross-device comparisons require it — otherwise each device's windows start at different points and the lines can't share an axis.

Missing windows: if a device goes offline after 10:05, `SAMPLE BY` by default **does not emit empty buckets**; those rows simply do not exist. The panel shows a gap. If you need a continuous series, either pre-process in Grafana with `fill(null)` or accept the gap — a gap is itself the "device offline" signal, which is more honest than padding with zeros.

Now a site-level rollup to see SAMPLE BY with a SYMBOL group-by:

```sql
SELECT ts, site, count() AS readings, avg(humidity) AS hum
FROM sensor_reading
WHERE ts >= '2026-10-25T00:00:00Z' AND ts < '2026-10-26T00:00:00Z'
SAMPLE BY 1h ALIGN TO CALENDAR
ORDER BY ts;
```

The `WHERE` range makes the engine open only the `2026-10-25` partition file. This is the heart of combining partitioning with SAMPLE BY: **the time filter prunes files, SAMPLE BY prunes rows.** Get both right and scanning hundreds of millions of rows in a day stays in the hundreds-of-milliseconds range.

## 7. Lag and backlog troubleshooting

A running pipeline is not necessarily a correct one. Three observations to make:

```sql
-- 1. latest timestamp vs now() = end-to-end lag
SELECT max(ts), now() - max(ts) AS lag FROM sensor_reading;

-- 2. rows written over time, to confirm no sudden stall
SELECT ts, count() FROM sensor_reading
WHERE ts > dateadd('h', -6, now())
SAMPLE BY 10m;

-- 3. WAL apply backlog
SELECT * FROM wal_tables() WHERE name = 'sensor_reading';
```

`wal_tables()` returns `sequencerTxn` and `writerTxn`; the gap between them is apply lag. In steady state they differ by at most a few transactions. A persistently growing gap means ingestion outpaces apply — either raise `wal_apply` concurrency (only possible with multiple partitions) or downsample before ingesting.

Common failure modes:

| Symptom | Likely cause | Fix |
|---|---|---|
| Connect is up but table stays empty | value converter has schemas enabled, JSON parse fails | set `value.converter.schemas.enable=false` |
| Rows arrive but fields are all null | message field names don't match `symbols` / column names (case-sensitive) | align names, or add a transformation |
| Kafka backlog keeps growing | fewer tasks than partitions, or oversized messages | set `tasks.max` to partition count, batch sends |
| Queries get slower over time | no `WHERE ts IN '...'`, so the scan covers all history | always include a time filter aligned with partition granularity |

## 8. Versus a hand-rolled consumer

Writing a Python consumer that batches 1,000 rows and pushes them over HTTP or ILP is most teams' first attempt. Its hidden costs:

- **Offset commit and write are not atomic.** Write then commit: a crash in between duplicates data. Commit then write: a crash loses data. You either build a dual-write scheme or accept at-least-once plus DEDUP — at which point you might as well use Connect.
- **Backpressure is opaque.** When the consumer can't keep up, the backlog only shows as Kafka lag, with no retry queue and no failure isolation.
- **More surface to operate.** Another deployment, another monitoring and alerting story, another upgrade path.

Kafka Connect's value is not that it is "better" — it is that it turns these edge cases into **configuration knobs**. The trade-off is flexibility: messages must map onto ILP, and deeply nested JSON needs SMT rules to flatten.

When messages are genuinely complex, an alternative is to land raw JSON in a staging table (`VARCHAR`) and either parse at query time with `json_extract` or push parsing upstream. Keeping the pipeline simple beats stacking a dozen transformation rules inside Connect config.

## 9. Next steps

- **Table design and partition strategy** — hourly partitions, retention-driven deletion → PARTITION BY strategy
- **Old-data cleanup** — Kafka keeps 7 days, QuestDB keeps 180 → data retention in practice
- **Dashboards** — feed SAMPLE BY results into Grafana → Grafana IoT dashboard
- **Coming from InfluxDB** — rewriting existing line protocol and InfluxQL → InfluxDB migration guide

---

*Configuration verified locally against Kafka 3.9 (KRaft), questdb/kafka-questdb-connector:latest, and QuestDB latest. DEDUP requires QuestDB ≥ 7.3; `wal_tables()` requires ≥ 7.3.1. QuestDB is a trademark of its owner; this site is an independent technical community, not affiliated with QuestDB.*
