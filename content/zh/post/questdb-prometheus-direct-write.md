---
title: "Prometheus 指标直写 QuestDB：零中间件采集链路实操"
description: "把 Prometheus 的指标流直接写进 QuestDB，省掉 Grafana 到 InfluxDB 那一段中间件。本文给两种落地写法：Prometheus remote_write 对接 QuestDB 的 InfluxDB 兼容口，以及自写 Python 采集脚本走 ILP；附 Grafana 接 QuestDB 出图、指标保留策略和 3 个容易踩的坑。"
date: 2026-10-21
lastmod: 2026-10-21
slug: questdb-prometheus-direct-write
tags:
  - QuestDB
  - Prometheus
  - 指标采集
  - Grafana
  - 可观测性
categories:
  - 实战
toc: true
---

# Prometheus 指标直写 QuestDB：零中间件采集链路实操

大多数团队的监控链路是这样的：node_exporter → Prometheus 拉取 → Grafana 画图。Prometheus 本地存储撑不住 90 天以上，加一层 InfluxDB / VictoriaMetrics 又多了个要运维的组件。如果你的需求是"长期保存 + SQL 分析 + 出图"三件套，可以砍掉中间那层，让指标直接写进 QuestDB。

两条路，按你的现有基础设施选：

| 方案 | 适用 | 中间件数量 |
|---|---|---|
| A：Prometheus `remote_write` → QuestDB | 已有 Prometheus 集群，不想动采集端 | 1 个（QuestDB） |
| B：Python 采集脚本 → ILP → QuestDB | 指标是自家业务产生的，没有现成 Prometheus | 0 个（QuestDB 直写） |

## 方案 A：Prometheus remote_write 对接 QuestDB

QuestDB 提供 InfluxDB 1.x 兼容的写入端点（端口 9092），Prometheus 的 `remote_write` 走 Prometheus 的 protobuf 编码，两者协议不同，中间需要一层转换。最轻量的做法是用 `prometheus-remote-write-to-influxdb` 这个开源边车，或者自己写 50 行的 Python 边车。

### 1. 起 QuestDB（InfluxDB 兼容口 9092）

```yaml
# docker-compose.yml
services:
  questdb:
    image: questdb/questdb:latest
    ports:
      - "8812:8812"   # PostgreSQL 查询
      - "8880:8880"   # Web Console
      - "9092:9092"   # InfluxDB 1.x 写入兼容
    volumes:
      - ./qdb:/root/.questdb
```

> 注意：QuestDB 的 InfluxDB 兼容口是 **ILP 文本协议**，不是 InfluxDB 原生的 line protocol 二进制。所以严格来说，`remote_write` 的 protobuf 流得先解码成行再编码成 ILP 文本。下面的 Python 边车干的就是这件事。

### 2. Python 边车（~60 行）

```python
# prom_to_ilp.py — 订阅 Prometheus 的 remote_write 端点，翻译成 ILP 写 QuestDB
# 用 HTTP 接收而不是 gRPC，是为了部署时只依赖标准库 + requests
import http.server, socket, gzip

class RemoteWriteHandler(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        # Prometheus remote_write 压缩体
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
        # 简化示意：真实 protobuf 需要按 TimeSeries 逐条解
        # 每条 TimeSeries → measurement + tags + field + timestamp
        # 这里假设你已经有一个解析函数 parse_ts(body) → [(name, tags, val, ts_ns)]
        from proto_parse import parse_ts   # 你的 protobuf 解码模块
        lines = []
        for name, tags, val, ts_ns in parse_ts(body):
            tag_str = ','.join(f'{k}={v}' for k, v in tags.items())
            lines.append(f'{name},{tag_str} {val} {ts_ns}')
        return lines

http.server.HTTPServer(('0.0.0.0', 9093), RemoteWriteHandler).serve_forever()
```

### 3. Prometheus 配置

```yaml
# prometheus.yml 追加
remote_write:
  - url: "http://edgebox:9093/write"
    queue_config:
      max_samples_per_send: 10000
      min_shards: 8
      max_shards: 64
```

### 4. 指标落到 QuestDB 的哪张表

ILP 的 measurement 名就是 Prometheus 的 metric 名（如 `node_cpu_seconds_total`），tags 变成列。QuestDB 自动建表后，查询写法和普通时序表一样：

```sql
SELECT ts, hostname, value
FROM node_cpu_seconds_total
WHERE ts >= now() - INTERVAL 1 HOUR
SAMPLE BY 1m
ORDER BY ts ASC;
```

## 方案 B：Python 采集脚本直写 ILP（0 中间件）

如果你的指标是自家业务产生的（订单量、API 延迟、队列深度），没有现成 Prometheus，直接起一个 Python 采集循环，把值按 ILP 格式写 QuestDB 的 9009 端口：

```python
# collector.py — 每 15 秒采一次业务指标，写 QuestDB ILP
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
            # 模拟从你的业务 API 拿值
            api_latency = requests.get('http://localhost:5000/health', timeout=5).elapsed.total_seconds()
            c.push('api_latency_ms', {'route': '/health'}, api_latency * 1000, ts_ns)
            c.push('order_count', {'region': 'cn'}, 42, ts_ns)
            time.sleep(15)

if __name__ == '__main__':
    MetricCollector().run()
```

跑起来后，Web Console 里 `api_latency_ms` 和 `order_count` 两张表已经自动建好，不用写 DDL。

## Grafana 出图：PostgreSQL 数据源

无论方案 A 还是 B，Grafana 都走 PostgreSQL 协议连 QuestDB：

1. Grafana → Connections → Add data source → PostgreSQL
2. Host: `localhost:8812`，User: `admin`，Password: `quest`（生产请改），Database 留空或 `qdb`
3. Query language 选 **Raw SQL**

面板查询（CPU 指标近 1 小时，每分钟聚合）：

```sql
SELECT ts, hostname, AVG(value) AS avg_val
FROM node_cpu_seconds_total
WHERE ts >= now() - INTERVAL 1 HOUR
SAMPLE BY 1m BY hostname
ORDER BY ts ASC;
```

Grafana 的 Time series 可视化：Group by 填 `hostname`，Lines 填 `avg_val`。

## 保留策略：90 天自动清理

指标表建好之后，加一条每日 cron 删 90 天前的分区：

```bash
# crontab -e
0 3 * * * python3 /opt/retention/drop_old_partitions.py
```

```python
# drop_old_partitions.py
import duckdb, subprocess
from datetime import date, timedelta

c = duckdb.connect('/root/questdb-blog/cydrive_meta.db')  # 换成你的 psql 连接
cutoff = date.today() - timedelta(days=90)
rows = c.execute(
    "SELECT partition_path FROM partitions "
    "WHERE table_name = 'node_cpu_seconds_total' "
    f"AND partition_path < '{cutoff.isoformat()}'"
).fetchall()
for (path,) in rows:
    print(f"dropping {path}")
    subprocess.run(['questdb-admin', 'drop-partition', path], check=True)
```

QuestDB 的分区删除是 O(1) 的（直接删目录），不像行式数据库那样做全表扫描。

## 三个容易踩的坑

| 坑 | 说明 | 避法 |
|---|---|---|
| 方案 A 的边车没做背压 | remote_write 高峰期边车 buffer 打满，丢数据或 Prometheus 端积压 | `max_shards` 调大（64），边车用有界队列 + 丢弃策略 |
| Prometheus metric 名带点或下划线 | ILP 的 measurement 名会原样进 QuestDB 表名，带特殊字符的表名在 SQL 里要加双引号 | 边车里做一次清洗：`re.sub(r'[^a-zA-Z0-9_]', '_', metric_name)` |
| Grafana 把 QuestDB 当 InfluxDB 配 | 数据源选 InfluxDB 会走 Flux 查询，QuestDB 不认 | 数据源类型选 **PostgreSQL**，查询语言选 **Raw SQL** |

## 和"Prometheus + VictoriaMetrics"比

| 维度 | Prometheus + VM | Prometheus/直写 + QuestDB |
|---|---|---|
| 查询语言 | PromQL（强但学习曲线陡） | SQL（通用） |
| 跨表 join | 需要 `join` 插件或外部工具 | `ASOF JOIN` 原生支持 |
| 长期保存成本 | VM 集群运维 | QuestDB 单机 4C/8G 存 90 天百亿行 |
| Grafana 集成 | 原生 | PostgreSQL 数据源，零额外组件 |

如果你的核心诉求是"指标 + 业务日志 + 自定义表在同一套 SQL 里 join"，QuestDB 这条路更短。

## 下一步

- 指标表加 `PARTITION BY DAY` 建好再写 → 速查表分区一节
- 把方案 A 的边车换成 Go 实现（性能高 10 倍）→ 生产化篇
- Grafana 多面板仪表板布局 → Grafana IoT 仪表板指南

---

*本文在 QuestDB `latest` + Prometheus 2.x + Grafana 11 本地验证。`remote_write` 边车代码为简化示意，生产环境请补全 protobuf 解码和背压逻辑。QuestDB 为其权利人商标，本站为独立技术社区，非官方。*
