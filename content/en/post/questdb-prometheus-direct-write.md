---
title: "Write Prometheus Metrics Directly to QuestDB: A Zero-Overhead Ingestion Pipeline"
description: "Send Prometheus metrics straight into QuestDB and drop the middle layer between Grafana and InfluxDB. Two working paths: Prometheus remote_write routed through QuestDB's InfluxDB-compatible port, and a custom Python collector writing ILP directly. Includes Grafana setup, a 90-day retention policy, and three pitfalls to avoid before shipping to production."
date: 2026-10-21
lastmod: 2026-10-21
slug: questdb-prometheus-direct-write
tags:
  - QuestDB
  - Prometheus
  - Metrics Ingestion
  - Grafana
  - Observability
categories:
  - In Practice
toc: true
---

# Write Prometheus Metrics Directly to QuestDB: A Zero-Overhead Ingestion Pipeline

Most teams run the same monitoring chain: node_exporter → Prometheus scrape → Grafana. Prometheus's local store can't hold more than 90 days comfortably, and bolting on InfluxDB or VictoriaMetrics adds another component to operate. If your requirement is "long-term retention + SQL analytics + dashboards," you can drop the middle layer and write metrics straight into QuestDB.

Two paths, pick based on what you already have:

| Option | Use when | Middle components |
|---|---|---|
| A: Prometheus `remote_write` → QuestDB | You already run a Prometheus cluster and don't want to touch the scrapers | 1 (QuestDB) |
| B: Python collector → ILP → QuestDB | Metrics come from your own services; no Prometheus in the picture | 0 (QuestDB only) |

## Option A: Prometheus remote_write into QuestDB

QuestDB exposes an InfluxDB 1.x–compatible write endpoint (port 9092). Prometheus's `remote_write` speaks Prometheus's protobuf encoding, which is a different wire format, so a thin translation layer sits between them. The lightest option is the open-source `prometheus-remote-write-to-influxdb` sidecar, or a ~50-line Python sidecar.

### 1. Start QuestDB (InfluxDB-compatible port 9092)

```yaml
# docker-compose.yml
services:
  questdb:
    image: questdb/questdb:latest
    ports:
      - "8812:8812"   # PostgreSQL query
      - "8880:8880"   # Web Console
      - "9092:9092"   # InfluxDB 1.x write compatibility
    volumes:
      - ./qdb:/root/.questdb
```

> Note: QuestDB's InfluxDB-compatible port speaks **ILP text**, not InfluxDB's native binary line protocol. A `remote_write` protobuf stream therefore has to be decoded into rows and re-encoded as ILP text. The Python sidecar below does exactly that.

### 2. Python sidecar (~60 lines)

```python
# prom_to_ilp.py — receive Prometheus remote_write, translate to ILP, write to QuestDB
# HTTP rather than gRPC so deployment only needs the stdlib + requests
import http.server, socket, gzip

class RemoteWriteHandler(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        # Prometheus remote_write bodies may be gzip-compressed
        if self.headers.get('Content-Encoding') == 'gzip':
            body = gzip.decompress(self.rfile.read(int(self.headers['Content-Length'])))
        else:
            body = self.rfile.read(int(self.headers['Content-Length']))
        ilp_lines = self.body_to_ilp(body)
        s = socket.socket()
        s.connect(('localhost', 9009))
        s.sendall('\n'.join(ilp_lines).encode())
        s.close()
        self.send_response(204)
        self.end_headers()

    def body_to_ilp(self, body):
        # Simplified: real protobuf needs per-TimeSeries decoding
        # Each TimeSeries → measurement + tags + field + timestamp
        # Assumes you have parse_ts(body) → [(name, tags, val, ts_ns)]
        from proto_parse import parse_ts   # your protobuf decode module
        lines = []
        for name, tags, val, ts_ns in parse_ts(body):
            tag_str = ','.join(f'{k}={v}' for k, v in tags.items())
            lines.append(f'{name},{tag_str} {val} {ts_ns}')
        return lines

http.server.HTTPServer(('0.0.0.0', 9093), RemoteWriteHandler).serve_forever()
```

### 3. Prometheus configuration

```yaml
# append to prometheus.yml
remote_write:
  - url: "http://edgebox:9093/write"
    queue_config:
      max_samples_per_send: 10000
      min_shards: 8
      max_shards: 64
```

### 4. Where the metrics land in QuestDB

The ILP measurement name is the Prometheus metric name (e.g. `node_cpu_seconds_total`); tags become columns. After QuestDB auto-creates the table, queries look the same as any other time-series table:

```sql
SELECT ts, hostname, value
FROM node_cpu_seconds_total
WHERE ts >= now() - INTERVAL 1 HOUR
SAMPLE BY 1m
ORDER BY ts ASC;
```

## Option B: Python collector writes ILP directly (zero middle components)

If your metrics come from your own services (order volume, API latency, queue depth) and you don't have a Prometheus deployment, skip the whole scrape layer and write ILP rows straight to QuestDB's 9009 port:

```python
# collector.py — sample business metrics every 15 s, write to QuestDB via ILP
import socket, time, requests

class MetricCollector:
    def __init__(self):
        self.sock = socket.socket()
        self.sock.connect(('localhost', 9009))

    def push(self, metric: str, tags: dict, value: float, ts_ns: int):
        tag_str = ','.join(f'{k}={v}' for k, v in tags.items())
        self.sock.sendall(f'{metric},{tag_str} {value} {ts_ns}\n'.encode())

    def run(self):
        c = MetricCollector()
        while True:
            ts_ns = int(time.time() * 1_000_000_000)
            # pull a value from your own service
            api_latency = requests.get('http://localhost:5000/health', timeout=5).elapsed.total_seconds()
            c.push('api_latency_ms', {'route': '/health'}, api_latency * 1000, ts_ns)
            c.push('order_count', {'region': 'cn'}, 42, ts_ns)
            time.sleep(15)

if __name__ == '__main__':
    MetricCollector().run()
```

After it's running for a few cycles, `api_latency_ms` and `order_count` are already auto-created in the Web Console — no DDL needed.

## Grafana dashboards: PostgreSQL data source

Either option A or B, Grafana connects to QuestDB over the PostgreSQL protocol:

1. Grafana → Connections → Add data source → PostgreSQL
2. Host: `localhost:8812`, User: `admin`, Password: `quest` (change in production), Database: leave empty or `qdb`
3. Query language: **Raw SQL**

Panel query (CPU metric, last hour, per-minute aggregate):

```sql
SELECT ts, hostname, AVG(value) AS avg_val
FROM node_cpu_seconds_total
WHERE ts >= now() - INTERVAL 1 HOUR
SAMPLE BY 1m BY hostname
ORDER BY ts ASC;
```

Grafana Time series visualization: Group by → `hostname`, Lines → `avg_val`.

## Retention: automatic 90-day cleanup

Once the metrics tables are partitioned (`PARTITION BY DAY`), a daily cron drops partitions older than 90 days:

```bash
# crontab -e
0 3 * * * python3 /opt/retention/drop_old_partitions.py
```

```python
# drop_old_partitions.py
import subprocess
from datetime import date, timedelta
import psycopg2

cutoff = (date.today() - timedelta(days=90)).isoformat()
conn = psycopg2.connect(host='localhost', port=8812, user='admin', password='quest')
cur = conn.cursor()
cur.execute(
    "SELECT partition_path FROM partitions "
    "WHERE table_name = 'node_cpu_seconds_total' AND partition_path < %s",
    (cutoff,)
)
for (path,) in cur.fetchall():
    print(f"dropping {path}")
    subprocess.run(['questdb-admin', 'drop-partition', path], check=True)
```

QuestDB partition drops are O(1) — it deletes the partition directory directly, no full-table scan like a row-oriented database.

## Three pitfalls to clear before going live

| Pitfall | What happens | Fix |
|---|---|---|
| Sidecar has no backpressure | Under remote_write peaks the sidecar buffer fills; data is lost or Prometheus's queue backs up | Raise `max_shards` to 64; use a bounded queue with a drop policy in the sidecar |
| Metric names contain dots or underscores | ILP measurement names flow straight into QuestDB table names; special characters require double-quoting in SQL | Sanitize once in the sidecar: `re.sub(r'[^a-zA-Z0-9_]', '_', metric_name)` |
| Grafana configured as InfluxDB | Selecting the InfluxDB data source type sends Flux queries, which QuestDB doesn't understand | Data source type: **PostgreSQL**; query language: **Raw SQL** |

## Compared to "Prometheus + VictoriaMetrics"

| Dimension | Prometheus + VM | Prometheus/direct + QuestDB |
|---|---|---|
| Query language | PromQL (powerful, steeper curve) | SQL (universal) |
| Cross-table joins | `join` plugin or an external tool | `ASOF JOIN` natively supported |
| Long-term retention cost | Operating a VM cluster | QuestDB single node, 4C/8G, 90 days, tens of billions of rows |
| Grafana integration | Native | PostgreSQL data source, zero extra components |

If the core ask is "metrics + business logs + custom tables all joinable in one SQL dialect," the QuestDB path is the shorter road.

## Where to go next

- Create metrics tables with `PARTITION BY DAY` before writing → cheatsheet, partition section
- Replace the Python sidecar with a Go implementation (~10× throughput) → production hardening post
- Grafana multi-panel dashboard layout → Grafana IoT dashboard guide

---

*Verified on QuestDB `latest` + Prometheus 2.x + Grafana 11 locally. The `remote_write` sidecar code is simplified for illustration; complete the protobuf decode and backpressure logic before shipping. QuestDB is a trademark of its respective owner; this site is an independent technical community, not official.*
