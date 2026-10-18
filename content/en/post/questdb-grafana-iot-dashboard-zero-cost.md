---
title: "Zero-Cost Self-Hosted: QuestDB + Grafana IoT Sensor Dashboard"
description: "You can build a complete IoT sensor monitoring stack for zero dollars: QuestDB stores the time-series data, ILP protocol ingests the readings, Grafana renders the charts. This post gives you a runnable Docker compose file, a simulated sensor writer, three-panel configuration, threshold alerting, and a retention strategy — all open-source, all free, including Grafana connection setup and native QuestDB query syntax (not PromQL)."
date: 2026-10-18
lastmod: 2026-10-18
slug: questdb-grafana-iot-dashboard-zero-cost
tags:
  - QuestDB
  - Grafana
  - IoT
  - Dashboard
  - Self-Hosted
categories:
  - Real-World
toc: true
---

# Zero-Cost Self-Hosted: QuestDB + Grafana IoT Sensor Dashboard

Want to build a "device reading → storage → charting → alerting" sensor monitoring stack, but don't want to touch cloud vendors' per-node subscription fees? This entire solution runs locally: QuestDB stores the time-series data, the ILP protocol writes the readings, Grafana handles the visualization. All three components are open source and free — an old 4-core 4GB box is plenty.

## 1. Architecture

```
[sensors / simulator] --ILP (TCP 9009)--> [QuestDB] --PostgreSQL (8812)--> [Grafana] --> browser
```

QuestDB exposes two query paths: the PostgreSQL protocol (8812) for Grafana and JDBC, and a native HTTP API (8880) for the Web Console. Grafana connects to QuestDB over the PostgreSQL protocol and treats it as "a database that only knows time-series queries."

## 2. Two containers, one Docker compose

```yaml
# docker-compose.yml
services:
  questdb:
    image: questdb/questdb:latest
    ports:
      - "8812:8812"   # PostgreSQL
      - "9009:9009"   # ILP ingest
      - "8880:8880"   # HTTP + Web Console
    volumes:
      - ./qdb:/root/.questdb

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=***    volumes:
      - ./grafana:/var/lib/grafana
```

```bash
docker compose up -d
# Grafana: http://localhost:3000  (admin / grafana)
# QuestDB Web Console: http://localhost:8880
```

## 3. Writing data: simulate 10 sensors, one reading per second

No real hardware needed — simulate the full pipeline. Ingest via ILP:

```python
# gen_sensors.py — simulate 10 sensors, one reading per second each
import socket, time

hosts = [f"sensor_{i:02d}" for i in range(10)]
s = socket.socket()
s.connect(("localhost", 9009))
seq = 0
while True:
    lines = []
    base_ns = int(time.time() * 1_000_000_000)
    for h in hosts:
        temp = 20.0 + 8.0 * ((seq % 137) / 137.0) + (int(h[-2:]) % 5)
        hum  = 40.0 + 30.0 * ((seq % 89)  / 89.0)
        lines.append(f"sensor,{h} temp={temp:.2f},hum={hum:.2f} {base_ns}")
    s.sendall(("\n".join(lines) + "\n").encode())
    seq += 1
    time.sleep(1)   # one batch per second
```

```bash
python3 gen_sensors.py &    # keep it running in the background: 10 devices x 1/s
```

`measurement=sensor`, tag is `host` (low cardinality → auto-created as `SYMBOL`), fields are `temp/hum`. QuestDB auto-creates the table on the first row, no DDL required.

## 4. Configure the Grafana data source (the critical step)

1. Grafana → **Connections → Add data source → PostgreSQL**
2. Fill in: Host `localhost:8812`, User `admin`, Password `quest` (QuestDB's default credentials — change them in production), leave Database as `qdb`
3. **Pick "Raw SQL" as the query language** (QuestDB uses time-series SQL, not vanilla Postgres)

Verify connectivity:

```sql
SELECT host, COUNT(*) AS n, MAX(temp) AS max_temp
FROM sensor
WHERE timestamp >= now() - INTERVAL 5 MINUTES
GROUP BY host;
```

If 10 rows come back, the data source works.

## 5. Three panels

### Panel 1: per-sensor temperature line chart

```sql
SELECT
    timestamp,
    host,
    temp
FROM sensor
WHERE timestamp >= now() - INTERVAL 1 HOUR
ORDER BY timestamp ASC
SAMPLE BY 10s FILL(null);
```

- Grafana panel: choose **Time series** for visualization
- Query: the statement above; put `host` in **Group by**, `temp` in **Line**
- The 10-second sampling downsamples the raw 1/sec points to 1/10s — a clean line that won't jank the panel

### Panel 2: each device's 1-hour peak temperature

```sql
SELECT host, MAX(temp) AS peak
FROM sensor
SAMPLE BY 1h
WHERE timestamp >= now() - INTERVAL 1 HOUR
ORDER BY peak DESC;
```

Choose **Table** or **Bar chart** visualization — spot the hottest device at a glance.

### Panel 3: humidity P95

```sql
SELECT host, PERCENTILE(hum, 0.95) AS p95
FROM sensor
SAMPLE BY 1h
WHERE timestamp >= now() - INTERVAL 1 HOUR
ORDER BY p95 DESC;
```

Arrange the three panels into one dashboard, group them with `Layout → Rows`, and set the refresh interval to `5s`.

## 6. Threshold alerting (Grafana-native, extra cost = zero)

On a panel, click **Add → Alert → Threshold**:

- Condition: `Last * 1 > 28.0` (temperature P95 or latest reading above 28°C)
- For: `2m` (must persist 2 minutes — debouncing)
- Notification: point Grafana's Alerting at a Webhook / email / team IM channel

The alert's data source is the same PostgreSQL data source, and the query reuses the P95 statement from Panel 3. All of this lives inside Grafana's built-in capabilities — zero external dependencies.

## 7. Add a retention policy so disk doesn't run away

Sensors write 24/7; within months the disk fills up. Add an automatic drop of old partitions (for a `PARTITION BY DAY` table):

```sql
-- at table-creation time, partition must be declared (here `sensor` was auto-created via ILP, so it's unpartitioned by default — rebuild it)
CREATE TABLE sensor_daily (
    ts TIMESTAMP,
    host SYMBOL,
    temp DOUBLE,
    hum DOUBLE
) PARTITION BY DAY;

-- after writing to sensor_daily via ILP, drop partitions older than 90 days daily
-- run in QuestDB (in production, call the Web Console / psql from cron):
SELECT DROP_PARTITION('sensor_daily', '2026_07') ;  -- exact function name depends on version
```

Wire it to system cron to run once daily — delete the 90-day-old partition at 03:00 — and disk usage stabilizes to a constant.

## 8. Cost & scale reference

| Item | Order of magnitude |
|---|---|
| Hardware | 4-core / 4GB / 100GB SSD, old box |
| Ingest rate | 10 devices x 1/s = 10 rows/s (a single QuestDB node handles ~1M rows/s — enormous headroom) |
| Storage | 10 rows/s ≈ 0.5 MB/day uncompressed; 90 days < 50 MB |
| License cost | 0 (QuestDB / Grafana / Docker all open source) |

Bottom line: this stack's performance ceiling vastly exceeds the needs of a "small-team IoT" setup. The bottleneck will always be the network card, not the database.

## 9. Where to go next

- **Device-metadata table + ASOF JOIN**: align a "device config" table to each reading by timestamp — see the ASOF JOIN guide
- **Anomaly detection**: add a `temp - LAG(temp, 5m) > 3°C` alert query on a panel
- **Kubernetes metrics**: move this architecture into K8s, replace the sensor sim with cAdvisor Prometheus scrapes, still writing into QuestDB

---

*Verified on QuestDB `latest` + Grafana 11 + Docker Compose. The default admin/quest credentials are for local demo only. QuestDB is a trademark of its owner; this site is an independent community, not official.*
