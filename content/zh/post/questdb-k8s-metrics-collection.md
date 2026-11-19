---
title: "Kubernetes 指标采集实战：Prometheus 直写 QuestDB + Grafana 监控"
description: "Kubernetes 监控链路里堆了 cAdvisor、Prometheus、VictoriaMetrics 三层存储，本文给一套只留 QuestDB 的写法：Prometheus remote_write 转 ILP 边车、Helm 部署 QuestDB 单节点、Grafana 配三块容器面板，附保留策略和 3 个坑。"
date: 2026-10-28
lastmod: 2026-10-28
slug: questdb-k8s-metrics-collection
tags:
  - QuestDB
  - Kubernetes
  - Prometheus
  - Grafana
  - 可观测性
categories:
  - 实战
toc: true
---

# Kubernetes 指标采集实战：Prometheus 直写 QuestDB + Grafana 监控

K8s 监控的标准堆栈是 cAdvisor → Prometheus → Grafana，中间那层本地存储撑不过 90 天，于是团队要么加 VictoriaMetrics，要么把 Prometheus 迁到 Thanos。两条路都多一个要运维的组件。如果你的诉求是"容器指标长期保存 + 能写 SQL 做容量分析 + Grafana 出图"，可以只留 QuestDB，把 Prometheus 当采集器，指标经 remote_write 转 ILP 写进去。

这里先说清楚两个前提，避免读完发现对不上场景：一是你已经有跑起来的 Prometheus，不想动采集端；二是你不需要 Prometheus 的 Recording Rules 告警，只要把指标存下来做长期查询。告警继续留在 Prometheus 里（2h 短存够用），长期数据和 SQL 分析交给 QuestDB，两边职责分开，边车只负责"搬运"，不做任何聚合。

整条链路：

```
cAdvisor/Pod 指标 → Prometheus（短存 2h）→ remote_write → 边车（protobuf→ILP）→ QuestDB ILP 口（9009）→ Grafana 查询 QuestDB
```

## 1. Helm 部署 QuestDB 单节点

先用单节点验证链路，等指标量上来再上多节点模式。部署时两个参数要盯住：一是 `persistence.size`，3000 Pod 的集群一年约 25GB，留 100Gi 足够；二是内存 limits，QuestDB 写入时会在内存里攒批，4Gi 是 5k QPS 下的安全线，再高就得上多节点。

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
# Service 暴露：8812（Postgres 查询）/ 9009（ILP）/ 8880（Web 控制台）
```

Pod 内起完先用 Web 控制台确认能建表，再往下走。

## 2. 建容器指标表

表结构按 cAdvisor 的真实字段来，SYMBOL 存低基数字段：

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
ALTER TABLE k8s_pod_metrics ALTER COLUMN memory_bytes TYPE BIGINT;
```

> `RETENTION RESERVE '12m' FIRST` 表示保留 12 个月数据，超过 12 个月的分区在下次备份/重启时自动清理。分区按月，单集群 3000 个 Pod、每 30 秒采一次，一年大约 2.5 亿行，压缩后落盘 20~30GB，单盘放得下。

## 3. 边车：Prometheus remote_write 转 ILP

QuestDB 的 ILP 口收文本行，Prometheus remote_write 推的是 protobuf，中间放一个 60 行左右的边车。用官方示例的 `prometheus-remote-write-to-influxdb` 也行，下面给 Python 版，部署就一个容器：

```python
# ilp_sidecar.py — 部署为 K8s Deployment，监听 9300
import http.server, gzip, struct, socket, urllib.request

class Handler(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        length = int(self.headers['Content-Length'])
        body = self.rfile.read(length)
        if self.headers.get('Content-Encoding') == 'gzip':
            body = gzip.decompress(body)
        # 真实环境用 protobuf 解码 TimeSeries，这里用简化接口示意
        ts_list = parse_remote_write(body)  # → [(labels, samples), ...]
        ilp = []
        for labels, samples in ts_list:
            tag_str = ','.join(f'{k}={v}' for k, v in labels.items()
                               if k in ('cluster','namespace','pod','node','container'))
            for ts_ns, val in samples:
                kind = labels.get('__name__', 'metric').replace('_total','')
                ilp.append(f'k8s_pod_metrics,{tag_str} {kind}={val} {ts_ns}')
        # 直写 QuestDB ILP 口，TCP 9009
        s = socket.create_connection(('qdb.questdb.svc', 9009), timeout=10)
        s.sendall(('\n'.join(ilp) + '\n').encode())
        s.close()
        self.send_response(204)
        self.end_headers()
```

Prometheus 侧配置：

```yaml
# prometheus.yaml
remote_write:
  - url: "http://ilp-sidecar.ilp-system.svc:9300/write"
    queue_config:
      max_samples_per_send: 10000
      capacity: 20000
```

## 4. Grafana 接 QuestDB 出三块容器面板

Grafana 装 QuestDB 插件（PostgreSQL 数据源走 8812），三块面板的 SQL 直接可跑：

```sql
-- 面板 1：单 Pod 内存 24h 曲线
SELECT timestamp, memory_bytes/1024/1024 AS mem_mb
FROM k8s_pod_metrics
WHERE namespace='default' AND pod='nginx-7d9f8'
  AND timestamp > now() - 24h
SAMPLE BY 1m

-- 面板 2：节点 CPU 排行（最近 1h，每 10s 采样）
SELECT node, SAMPLE AVG(cpu_usage_cores) BY 10s AS cpu_avg
FROM k8s_pod_metrics
WHERE timestamp > now() - 1h
GROUP BY node ORDER BY cpu_avg DESC

-- 面板 3：集群网络收发对比
SELECT timestamp,
       SAMPLE SUM(network_rx_bps) BY 1m AS rx,
       SAMPLE SUM(network_tx_bps) BY 1m AS tx
FROM k8s_pod_metrics
WHERE timestamp > now() - 6h
SAMPLE BY 1m
```

Grafana 面板里时间列直接绑 `timestamp`，rx/tx 两条曲线画一张图，节点 CPU 排行用 Bar 面板。

## 5. 三个坑

**坑 1：cAdvisor 指标名和 Prometheus 指标名不一致。** cAdvisor 采出来是 `container_cpu_usage_seconds_total`，Prometheus 默认重命名为 `container_cpu_usage_seconds_total`，但 Pod 标签挂在 `pod_cpu` 上。边车做转换时以 `__name__` 为准映射到字段，别拿标签直接当字段名。

**坑 2：SYMBOL 基数控制。** 如果标签里混进了带 UUID 的字段（比如 Job 的 `job_id`），SYMBOL 内部字典会无限膨胀，查询变慢。边车里把这类字段过滤掉，或者降级存 VARCHAR。

**坑 3：remote_write 断连重试。** 边车写 QuestDB 失败时别直接丢数据，Prometheus 的 `remote_write` 自带重试队列（`capacity: 20000`），但边车自己崩了得让 K8s 拉起来——Deployment 挂个 liveness 探针，POST 端点 5 分钟无流量即重启。另外边车和 QuestDB 之间走 TCP 9009，如果跨集群部署，记得在 NetworkPolicy 里放行，别等上线后才发现 9009 被默认拒绝。

## 下一步

单节点撑住一个月后，把 QuestDB 升到多节点模式（3 副本 + 对象存储冷备），指标保留拉到 2 年；容量分析用 SQL 聚合出月报，比如"每个 namespace 的 P95 内存"，不用再去翻 Prometheus 的 `topk`。

## 参考

- [Grafana 官方文档](https://grafana.com/docs/grafana/latest/)
- [QuestDB GitHub 仓库](https://github.com/questdb/questdb)

## 相关文章

- [Prometheus 指标直写 QuestDB：零中间件采集链路实操](/zh/post/questdb-prometheus-direct-write/)
- [零成本自建：QuestDB + Grafana 构建 IoT 传感器仪表板](/zh/post/questdb-grafana-iot-dashboard-zero-cost/)
- [QuestDB + Kafka 实时管道：从 ILP 写入到 SAMPLE BY 聚合](/zh/post/questdb-kafka-connect-pipeline/)
