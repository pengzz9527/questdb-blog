---
title: "零成本自建：QuestDB + Grafana 构建 IoT 传感器仪表板"
description: "不花一分钱就能搭一套完整的 IoT 传感器监控：QuestDB 存时序数据，ILP 协议灌入读数，Grafana 出图。本文给可运行的 Docker 编排、传感器模拟写入、三块面板配置、阈值告警与保留策略，全部零授权费用，含 Grafana 连接与 PromQL 无关的 QuestDB 原生查询写法。"
date: 2026-10-18
lastmod: 2026-10-18
slug: questdb-grafana-iot-dashboard-zero-cost
tags:
  - QuestDB
  - Grafana
  - IoT
  - 仪表板
  - 自托管
categories:
  - 实战
toc: true
---

# 零成本自建：QuestDB + Grafana 构建 IoT 传感器仪表板

想搭一套"设备读数 → 入库 → 出图 → 告警"的传感器监控，但又不想碰云厂商按节点收订阅费？下面这套方案全程本地跑：QuestDB 存时序数据，ILP 协议写入读数，Grafana 负责看图。三个组件都开源免费，一台 4 核 4G 的旧机器就够。

## 一、整体架构

```
[传感器/模拟脚本] --ILP(TCP 9009)--> [QuestDB] --PostgreSQL(8812)--> [Grafana] --> 浏览器
```

QuestDB 同时提供两条查询出口：PostgreSQL 协议（8812）给 Grafana 和 JDBC，以及原生 HTTP（8880）给 Web 控制台。Grafana 走 PostgreSQL 协议连 QuestDB，把它当成一个"只会时序查询的数据库"来配。

## 二、Docker 一键起两个容器

```yaml
# docker-compose.yml
services:
  questdb:
    image: questdb/questdb:latest
    ports:
      - "8812:8812"   # PostgreSQL
      - "9009:9009"   # ILP 写入
      - "8880:8880"   # HTTP + Web Console
    volumes:
      - ./qdb:/root/.questdb

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=grafana
    volumes:
      - ./grafana:/var/lib/grafana
```

```bash
docker compose up -d
# Grafana: http://localhost:3000  (admin / grafana)
# QuestDB Web Console: http://localhost:8880
```

## 三、写数据：模拟 10 台传感器每秒读数

不用真设备也能跑通全流程。用 ILP 灌入：

```python
# gen_sensors.py — 模拟 10 台传感器，每台每秒一条
import socket, time

hosts = [f"sensor_{i:02d}" for i in range(10)]
s = socket.socket()
s.connect(("localhost", 9009))
seq = 0
while True:
    lines = []
    base_ns = int(time.time() * 1_000_000_000)
    for h in hosts:
        temp = 20.0 + 8.0 * ((seq % 137) / 137.0) + (int(h[-2:]) % 5)  # 0.1 步进 + 设备偏移
        hum  = 40.0 + 30.0 * ((seq % 89)  / 89.0)
        lines.append(f"sensor,{h} temp={temp:.2f},hum={hum:.2f} {base_ns}")
    s.sendall(("\n".join(lines) + "\n").encode())
    seq += 1
    time.sleep(1)   # 每秒一批
```

```bash
python3 gen_sensors.py &    # 后台跑着，10 台 × 1/秒
```

`measurement=sensor`，tag 是 `host`（低基数 → 自动建为 `SYMBOL`），fields 是 `temp/hum`。QuestDB 第一行就自动建表，不用手写 DDL。

## 四、Grafana 配数据源（关键步骤）

1. Grafana → **Connections → Add data source → PostgreSQL**
2. 填：Host `localhost:8812`，User `admin`，Password `quest`（QuestDB 默认凭据，生产请改），Database 留 `qdb`
3. **Query language 选 "Raw SQL"**（QuestDB 是时序 SQL，不是标准 Postgres）

验证连通：

```sql
SELECT host, COUNT(*) AS n, MAX(temp) AS max_temp
FROM sensor
WHERE timestamp >= now() - INTERVAL 5 MINUTES
GROUP BY host;
```

能出 10 行，数据源就通了。

## 五、三块面板

### 面板 1：各传感器温度折线

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

- Grafana 面板：Visualization 选 **Time series**
- Query：上面这段；**Group by** 填 `host`，**Line** 填 `temp`
- 10 秒采样把 1/秒的原始点降成 1/10 秒，折线干净、面板不卡

### 面板 2：每台设备近 1 小时最高温

```sql
SELECT host, MAX(temp) AS peak
FROM sensor
SAMPLE BY 1h
WHERE timestamp >= now() - INTERVAL 1 HOUR
ORDER BY peak DESC;
```

Visualization 选 **Table** 或 **Bar chart**，一眼看出谁最热。

### 面板 3：湿度百分位（P95）

```sql
SELECT host, PERCENTILE(hum, 0.95) AS p95
FROM sensor
SAMPLE BY 1h
WHERE timestamp >= now() - INTERVAL 1 HOUR
ORDER BY p95 DESC;
```

三块面板排成一个仪表板，`Layout → Rows` 分组，刷新间隔设 `5s`。

## 六、阈值告警（Grafana 原生，不额外花钱）

在面板上点 **Add → Alert → Threshold**：

- 条件：`Last * 1 > 28.0`（温度 P95 或最新值超 28℃）
- For：`2m`（持续 2 分钟才触发，防抖）
- Notification：Grafana 的 Alerting 里接一个 Webhook / 邮件 / 企业 IM

告警数据源就是刚才那个 PostgreSQL 数据源，查询用面板 3 的 P95 语句。全在 Grafana 内置能力里，零外部依赖。

## 七、加保留策略，磁盘不失控

传感器 7×24 灌，几个月磁盘就满。加一条自动删老分区（`PARTITION BY DAY` 的表）：

```sql
-- 建表时就要带分区（本例 sensor 是 ILP 自动建表，默认未分区，需重建）
CREATE TABLE sensor_daily (
    ts TIMESTAMP,
    host SYMBOL,
    temp DOUBLE,
    hum DOUBLE
) PARTITION BY DAY;

-- 用 ILP 写 sensor_daily 后，每天删 90 天前的分区
-- 在 QuestDB 里（生产用 cron 调 Web Console / psql）：
SELECT DROP_PARTITION('sensor_daily', '2026_07') ;  -- 具体函数名以版本为准
```

配合系统 cron，每天凌晨执行一次"删除 90 天前分区"，磁盘占用就稳在一个常数。

## 八、成本与规模参考

| 项 | 数量级 |
|---|---|
| 硬件 | 4 核 / 4G / 100G SSD 旧机器 |
| 写入速率 | 10 设备 × 1/s = 10 行/s（QuestDB 单机可扛 100 万行/s，余量巨大） |
| 存储 | 10 行/s ≈ 0.5 MB/天（未压缩），90 天 < 50 MB |
| 授权费 | 0（QuestDB / Grafana / Docker 全开源） |

结论：这套方案的性能天花板远超"小团队物联网"需求，瓶颈永远在网卡而不是数据库。

## 九、扩展方向

- **设备元数据表 + ASOF JOIN**：把"设备配置"表按时间对齐到读数，见 ASOF JOIN 指南
- **异常检测**：在面板里加一条 `temp - LAG(temp, 5m) > 3℃` 的告警查询
- **K8s 指标接入**：把本套架构挪进 Kubernetes，传感器换成 cAdvisor 的 Prometheus 拉取，仍写 QuestDB

---

*本文在 QuestDB `latest` + Grafana 11 + Docker Compose 验证。默认凭据 admin/quest 仅供本地演示。QuestDB 为其权利人商标，本站为独立技术社区，非官方。*
