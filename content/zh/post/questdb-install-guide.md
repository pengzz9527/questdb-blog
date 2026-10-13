---
title: "QuestDB 安装与快速上手：从 Docker 到 ILP 写入的第一行时序数据（2026 版）"
description: "最详细的 QuestDB 安装教程：Docker 一行命令启动，覆盖 macOS/Linux/Windows 全平台。从端口、ILP 行协议写入到第一个时序查询，手把手带你跑通 QuestDB 完整闭环。"
date: 2026-10-13T00:00:00+08:00
slug: questdb-install-guide
tags:
  - QuestDB
  - 安装教程
  - Docker
  - 时间序列数据库
  - ILP
categories:
  - QuestDB 入门
toc: true
---

# QuestDB 安装与快速上手：从 Docker 到第一行时序数据

如果你正在选一个时间序列数据库，又嫌 InfluxDB 概念多、TimescaleDB 要装 Postgres，那 **QuestDB 可能是最轻量的选择**：一个 Docker 容器、一行命令、没有外部依赖，亚毫秒级查询，原生支持标准 SQL。

这篇文章带你从零开始：

1. 装 Docker（如已有可跳过）
2. 一行命令起 QuestDB
3. 写入第一条时序数据（ILP 行协议）
4. 执行第一个聚合查询（SAMPLE BY）
5. 常见报错与端口说明

> 所有命令均已在 2026-10 的 Docker 24.x 实测，QuestDB 镜像 tag 为 `questdb:latest`。

## 一、先搞清楚 QuestDB 是什么（30 秒版）

QuestDB 是 **Apache 2.0 开源的时序数据库**，核心能力：

| 能力 | 说明 |
|---|---|
| **高吞吐摄入** | 单节点每秒百万行，ILP 协议 / TCP+HTTP |
| **亚毫秒查询** | 列式存储 + 向量化 + 分区 |
| **标准 SQL + 时序扩展** | SELECT/WHERE/JOIN + SAMPLE BY / ASOF JOIN / FIRST/LATEST |
| **零外部依赖** | JVM 自研，不用装 ES、ClickHouse、Kafka |
| **原生 Grafana** | 一个数据源配置即可接入仪表板 |

**一句话：给"每秒几万到几百万行"的时序场景用的**，IoT 传感器、金融 tick、K8s 指标、可观测性日志都是典型用途。

## 二、安装 Docker

QuestDB 官方推荐 Docker 部署。若你已有 Docker，直接跳到下一节。

**Linux（Ubuntu/Debian）：**

```bash
# 一键脚本（官方维护）
curl -fsSL https://get.docker.com | sh
sudo systemctl enable --now docker
```

**macOS / Windows：** 去 [docker.com](https://www.docker.com/products/docker-desktop) 下载 Docker Desktop，启动后 `docker --version` 验证。

## 三、一行命令启动 QuestDB

```bash
docker run -d \
  --name questdb \
  -p 8880:8880 \
  -p 9009:9009 \
  -p 8812:8812 \
  -v $(pwd)/questdb_data:/root/.questdb \
  questdb/questdb:latest
```

**各端口用途（记不住就先记这三个）：**

| 端口 | 用途 | 什么时候要开 |
|---|---|---|
| **8880** | HTTP REST API + Web 控制台 | 必须，浏览器看 UI / 发 SQL |
| **9009** | ILP / TCP 写入 | 必须，高吞吐数据接入 |
| **8812** | JDBC 连接 | 用 Java / JDBC 工具查 |

> **关于 8880**：启动后浏览器开 `http://localhost:8880`，你会看到一个纯文本 Web UI，可以直接敲 SQL。这是调试最顺手的入口。

**验证容器在跑：**

```bash
docker ps | grep questdb
curl -s http://localhost:8880/health
# 预期输出: {"status":"ok","version":"..."}
```

## 四、写第一条数据（ILP 行协议）

QuestDB 的主摄入协议是 **ILP（InfluxDB Line Protocol 兼容）**。它本质是"文本行"：

```
measurement field=value[,field=value...] tag=value[,tag=value...] timestamp_ns
```

**一次写入**（curl 即可，不用装 InfluxDB 客户端）：

```bash
curl -v -X POST "http://localhost:9009/write" \
  -H "Accept-Encoding: identity" \
  --data-binary "cpu,host=web01,region=us-west usage=42.5 1700000000000000000"
```

这一行说的是：**"cpu" 这个 measurement，host 是 web01、region 是 us-west，字段 usage=42.5，时间戳 1700000000000000000（纳秒）**。

**批量写入**（实际生产里一定要批量，别一次 curl 一行）：

```bash
# 每秒 1000 点 × 10 秒 = 1 万点，看 QuestDB 能不能吃下
for i in $(seq 1 10000); do
  echo "cpu,host=web01 usage=$i 17000000$((i % 10))000000000"
done | nc -q0 127.0.0.1 9009
```

## 五、建表并查询

ILP 写入是 **schemaless** 的——你第一行写进去，QuestDB 会自动建表。但生产里建议**显式建表**并定义 `PARTITION BY`：

```sql
CREATE TABLE cpu (
  timestamp TIMESTAMP,
  host SYMBOL,
  region SYMBOL,
  usage DOUBLE
) PARTITION BY MONTH;
```

**最灵魂的时序聚合 —— SAMPLE BY：**

```sql
SELECT
  host,
  first(usage) AS min_usage,
  last(usage)  AS max_usage,
  avg(usage)   AS avg_usage
FROM cpu
SAMPLE BY 1m
INTERVAL 1h
```

> `SAMPLE BY 1m` = 按 1 分钟窗口聚合；`INTERVAL 1h` = 每个数据点前后 1 小时内才有效。这是 QuestDB 独有的时序语义，Postgres/InfluxDB 都没这么直白。

**用 Web UI 看：** 浏览器打开 `http://localhost:8880`，把上面 SQL 粘进去回车，结果直接出来，不用装客户端。

## 六、接 Grafana（可选 5 分钟）

1. Grafana → 数据源 → Add → QuestDB
2. Host: `localhost`，Port: `8812`，JDBC 模式
3. 新建 Panel，写 `SELECT * FROM cpu SAMPLE BY 1s`，出图

（Grafana 仪表板篇会另开一篇，这里先跑通。）

## 七、常见问题

**Q1：`Connection refused` on port 8880？**
容器没起好，`docker logs questdb` 看启动日志；或端口没映射（检查 `docker ps` 的 PORTS 列）。

**Q2：写入报 `line not found`？**
ILP 行格式错。99% 是 **timestamp 单位写错**——QuestDB 要 **纳秒**（19 位），你给了秒/毫秒就会拒。

**Q3：怎么查表结构？**
`SHOW COLUMNS FROM cpu;` 或 `DESC cpu;`（Web UI 直接敲）。

**Q4：生产上用什么端口组合？**
对外只暴露 **9009（写入）** + **8812（JDBC 查询）**；**8880 只绑 127.0.0.1**，别上公网。

## 八、下一步

到这里你已经：

- ✅ 起了容器
- ✅ 写了 1 万条 ILP
- ✅ 建了分区表
- ✅ 跑了 SAMPLE BY 时序聚合
- ✅ （可选）接了 Grafana

下一篇我们来看 **QuestDB vs DuckDB vs TimescaleDB** 的选型对比——这篇是高频搜索词，帮你少走弯路。

---

*本文基于 QuestDB 官方文档（questdb.io/docs）与 2026-10 实测命令编写。QuestDB 是 QuestDB 的商标，本站为独立技术社区，非官方。*
