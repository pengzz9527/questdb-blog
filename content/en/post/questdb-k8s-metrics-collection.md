---
title: "Kubernetes Metrics Pipeline: Prometheus Writing Directly to QuestDB with Grafana"
description: "Three storage layers in the K8s monitoring chain - cAdvisor, Prometheus, and VictoriaMetrics. This article shows how to keep only QuestDB: a Prometheus remote_write-to-ILP sidecar, single-node QuestDB via Helm, and three Grafana container panels, plus retention strategy and three gotchas."
date: 2026-10-28
lastmod: 2026-10-28
slug: questdb-k8s-metrics-collection
tags:
  - QuestDB
  - Kubernetes
  - Prometheus
  - Grafana
  - Observability
categories:
  - Practical
toc: true
---

# Kubernetes Metrics Pipeline: Prometheus Writing Directly to QuestDB with Grafana

The standard K8s monitoring stack is cAdvisor → Prometheus → Grafana, and the local storage layer in the middle can't hold more than 90 days. So teams either add VictoriaMetrics or migrate Prometheus to Thanos. Both routes mean one more component to operate. If your requirement is "long-term container metrics storage + SQL-based capacity analysis + Grafana dashboards", you can keep only QuestDB and treat Prometheus purely as a collector: metrics flow out via `remote_write` into an ILP sidecar and land in QuestDB.

Two assumptions to rule out before reading: first, you already have a running Prometheus instance and don't want to touch the collection side; second, you don't need Prometheus recording-rule alerts — you just want to store metrics for long-term query. Alerts stay in Prometheus (a 2h short-term retention is enough); long-term data and SQL analytics go to QuestDB. The sidecar only "moves" data, it does no aggregation.

The full pipeline:

```
cAdvisor/Pod metrics → Prometheus (2h short retention) → remote_write → sidecar (protobuf → ILP) → QuestDB ILP port (9009) → Grafana queries QuestDB
```

## 1. Helm: Deploy a Single-Node QuestDB

Start with single-node to validate the pipeline; move to multi-node when metric volume grows. Two parameters to watch: `persistence.size` — a 3,000-pod cluster generates roughly 25 GB of compressed metrics over a year, so 100 Gi is plenty; and memory limits — QuestDB batches writes in memory, and 4 Gi is the safe line at 5k QPS. Beyond that, multi-node.

```yaml
# questdb-values.yaml
image:
  repository: questdb/questdb
  tag: latest
replicaCount: 1
resources:
  requests: { cpu: "500m", memory: "2Gi" }
  limits:   { cpu: "2",    memory: "4Gi" }
persistence:
  size: 100Gi
  storageClassName: standard
```

```bash
helm repo add questdb https://questdb.github.io/helm-charts
helm install qdb questdb/questdb -f questdb-values.yaml
# Service exposes: 8812 (Postgres query) / 9009 (ILP) / 8880 (Web console)
```

Once the Pod is up, verify table creation through the Web console before moving on.

## 2. Create the Container Metrics Table

Table schema mirrors cAdvisor's actual fields; SYMBOL for low-cardinality columns:

```sql
CREATE TABLE IF NOT EXISTS k8s_pod_metrics (
    timestamp        TIMESTAMP,
    cluster          SYMBOL,
    namespace        SYMBOL,
    pod              SYMBOL,
    node             SYMBOL,
    container        SYMBOL,
    cpu_usage_cores  DOUBLE,
    memory_bytes     BIGINT,
    network_rx_bps   DOUBLE,
    network_tx_bps   DOUBLE,
    INDEX idx_ts (timestamp)
) PARTITION BY MONTH(timestamp)
RETENTION RESERVE '12m' FIRST
```

> `RETENTION RESERVE '12m' FIRST` means 12 months of data are kept; partitions older than that are cleaned up at the next backup/restart. Monthly partitioning: one 3,000-pod cluster at 30-second intervals produces roughly 250 million rows per year, compressing to 20–30 GB on disk — a single volume handles it.

## 3. Sidecar: Prometheus `remote_write` to ILP

QuestDB's ILP port accepts text lines, but Prometheus `remote_write` pushes protobuf. A ~60-line sidecar bridges the gap. The official `prometheus-remote-write-to-influxdb` example works too; here's a Python version that deploys as a single container:

```python
# ilp_sidecar.py — deploy as a K8s Deployment, listening on 9300
import http.server, gzip, socket

class Handler(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        length = int(self.headers['Content-Length'])
        body = self.rfile.read(length)
        if self.headers.get('Content-Encoding') == 'gzip':
            body = gzip.decompress(body)
        # In production, decode protobuf TimeSeries; simplified interface here
        ts_list = parse_remote_write(body)  # → [(labels, samples), ...]
        ilp = []
        for labels, samples in ts_list:
            tag_str = ','.join(f'{k}={v}' for k, v in labels.items()
                               if k in ('cluster','namespace','pod','node','container'))
            for ts_ns, val in samples:
                kind = labels.get('__name__', 'metric').replace('_total','')
                ilp.append(f'k8s_pod_metrics,{tag_str} {kind}={val} {ts_ns}')
        # Write directly to QuestDB ILP over TCP 9009
        s = socket.create_connection(('qdb.questdb.svc', 9009), timeout=10)
        s.sendall(('\n'.join(ilp) + '\n').encode())
        s.close()
        self.send_response(204)
        self.end_headers()
```

Prometheus-side config:

```yaml
# prometheus.yaml
remote_write:
  - url: "http://ilp-sidecar.ilp-system.svc:9300/write"
    queue_config:
      max_samples_per_send: 10000
      capacity: 20000
```

## 4. Grafana with QuestDB: Three Container Panels

Install the QuestDB plugin for Grafana (PostgreSQL data source on 8812). The three panel SQL queries below are ready to run:

```sql
-- Panel 1: 24h memory curve for a single Pod
SELECT timestamp, memory_bytes/1024/1024 AS mem_mb
FROM k8s_pod_metrics
WHERE namespace='default' AND pod='nginx-7d9f8'
  AND timestamp > now() - 24h
SAMPLE BY 1m

-- Panel 2: Node CPU ranking (last 1h, 10s sampling)
SELECT node, SAMPLE AVG(cpu_usage_cores) BY 10s AS cpu_avg
FROM k8s_pod_metrics
WHERE timestamp > now() - 1h
GROUP BY node ORDER BY cpu_avg DESC

-- Panel 3: Cluster network RX/TX comparison
SELECT timestamp,
       SAMPLE SUM(network_rx_bps) BY 1m AS rx,
       SAMPLE SUM(network_tx_bps) BY 1m AS tx
FROM k8s_pod_metrics
WHERE timestamp > now() - 6h
SAMPLE BY 1m
```

In Grafana, bind the time axis directly to `timestamp`; plot rx/tx on the same line panel; use a Bar panel for the node CPU ranking.

## 5. Three Gotchas

**Gotcha 1: cAdvisor metric names don't match Prometheus metric names.** cAdvisor emits `container_cpu_usage_seconds_total`, but Prometheus renames it by default and attaches the Pod label to `pod_cpu`. The sidecar must map to fields using `__name__` as the source of truth — don't use labels directly as field names.

**Gotcha 2: SYMBOL cardinality control.** If a label with a UUID (like a Job's `job_id`) slips into a SYMBOL column, the internal dictionary grows unboundedly and queries slow down. Filter such fields in the sidecar, or downgrade them to VARCHAR.

**Gotcha 3: `remote_write` disconnect retries.** When the sidecar fails to write to QuestDB, don't just drop the data. Prometheus `remote_write` has a built-in retry queue (`capacity: 20000`), but if the sidecar itself crashes, K8s needs to restart it — add a liveness probe to the Deployment that restarts on 5 minutes of zero POST traffic. Also, the sidecar talks to QuestDB over TCP 9009; for cross-cluster deployment, allow that port in your NetworkPolicy so it's not silently denied by default.

## Next Steps

After single-node holds steady for a month, upgrade QuestDB to multi-node mode (3 replicas + object-storage cold backup) and extend retention to 2 years. Capacity analysis becomes SQL aggregation into monthly reports — e.g. "P95 memory per namespace" — no more digging through Prometheus `topk`.

## References

- [Grafana docs](https://grafana.com/docs/grafana/latest/)
- [QuestDB GitHub repository](https://github.com/questdb/questdb)

## Related reading

- [Write Prometheus Metrics Directly to QuestDB: A Zero-Overhead Ingestion Pipeline](/en/post/questdb-prometheus-direct-write/)
- [Zero-Cost Self-Hosted: QuestDB + Grafana IoT Sensor Dashboard](/en/post/questdb-grafana-iot-dashboard-zero-cost/)
- [QuestDB + Kafka Real-Time Pipeline: ILP Ingestion to SAMPLE BY Rollups](/en/post/questdb-kafka-connect-pipeline/)
