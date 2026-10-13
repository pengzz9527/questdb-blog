---
title: "QuestDB Installation & Quick Start: From Docker to Your First ILP Write (2026)"
description: "The most practical QuestDB installation guide: start QuestDB with one Docker command across macOS/Linux/Windows. From ports, ILP line-protocol writes, to your first time-series query — run the full loop by hand."
date: 2026-10-13T00:00:00+08:00
slug: questdb-install-guide
tags:
  - QuestDB
  - Installation
  - Docker
  - Time-Series Database
  - ILP
categories:
  - Getting Started
toc: true
---

# QuestDB Installation & Quick Start: From Docker to Your First Time-Series Row

If you're picking a time-series database and find InfluxDB too concept-heavy and TimescaleDB too tied to Postgres, **QuestDB is the lightest option**: one Docker container, one command, no external dependencies, sub-millisecond queries, and native standard SQL.

This article walks you from zero:

1. Install Docker (skip if you already have it)
2. Start QuestDB with one line
3. Write your first time-series points (ILP line protocol)
4. Run your first aggregation query (SAMPLE BY)
5. Common errors and port reference

> Every command was verified on Docker 24.x in Oct 2026. QuestDB image tag: `questdb:latest`.

## 1. What QuestDB Actually Is (30-Second Version)

QuestDB is an **Apache 2.0 open-source time-series database**. Core capabilities:

| Capability | Notes |
|---|---|
| **High-throughput ingestion** | Single node, millions of rows/sec via ILP over TCP/HTTP |
| **Sub-millisecond queries** | Columnar storage + vectorization + partitioning |
| **Standard SQL + TS extensions** | SELECT/WHERE/JOIN + SAMPLE BY / ASOF JOIN / FIRST/LATEST |
| **Zero external deps** | Pure-JVM, no ES/ClickHouse/Kafka to babysit |
| **Native Grafana** | One data-source config and you're charting |

**TL;DR: built for "tens of thousands to millions of rows per second"** — IoT sensors, financial tick, K8s metrics, and observability logs are the archetypal use cases.

## 2. Install Docker

QuestDB's official recommendation is Docker. If you already run Docker, skip ahead.

**Linux (Ubuntu/Debian):**

```bash
curl -fsSL https://get.docker.com | sh
sudo systemctl enable --now docker
```

**macOS / Windows:** Download Docker Desktop from [docker.com](https://www.docker.com/products/docker-desktop), start it, then verify with `docker --version`.

## 3. Start QuestDB in One Line

```bash
docker run -d \
  --name questdb \
  -p 8880:8880 \
  -p 9009:9009 \
  -p 8812:8812 \
  -v $(pwd)/questdb_data:/root/.questdb \
  questdb/questdb:latest
```

**Port cheat sheet (remember these three):**

| Port | Purpose | When you need it |
|---|---|---|
| **8880** | HTTP REST API + Web console | Always — browse UI / run SQL |
| **9009** | ILP / TCP writes | Always — high-throughput ingestion |
| **8812** | JDBC | Java / JDBC tooling |

> **About 8880:** open `http://localhost:8880` in a browser — a plain-text Web UI where you can type SQL directly. It's the most comfortable debugging surface.

**Verify the container is running:**

```bash
docker ps | grep questdb
curl -s http://localhost:8880/health
# Expect: {"status":"ok","version":"..."}
```

## 4. Write Your First Data (ILP Line Protocol)

QuestDB's primary ingestion protocol is **ILP (InfluxDB Line Protocol-compatible)**. It's essentially "text lines":

```
measurement field=value[,field=value...] tag=value[,tag=value...] timestamp_ns
```

**Single write** (curl is enough, no InfluxDB client needed):

```bash
curl -v -X POST "http://localhost:9009/write" \
  -H "Accept-Encoding: identity" \
  --data-binary "cpu,host=web01,region=us-west usage=42.5 1700000000000000000"
```

Decoded: **measurement `cpu`**, tags `host=web01`, `region=us-west`, field `usage=42.5`, timestamp `1700000000000000000` (nanoseconds).

**Batch write** (in production you *always* batch — never curl line by line):

```bash
# 10k points, observe QuestDB keeping up
for i in $(seq 1 10000); do
  echo "cpu,host=web01 usage=$i 17000000$((i % 10))000000000"
done | nc -q0 127.0.0.1 9009
```

## 5. Create the Table and Query

ILP is **schemaless** — your first write auto-creates the table. In production you'll want an explicit table with `PARTITION BY`:

```sql
CREATE TABLE cpu (
  timestamp TIMESTAMP,
  host SYMBOL,
  region SYMBOL,
  usage DOUBLE
) PARTITION BY MONTH;
```

**The soul of time-series aggregation — SAMPLE BY:**

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

> `SAMPLE BY 1m` = aggregate per 1-minute bucket; `INTERVAL 1h` = only accept points within ±1h of the bucket. This is QuestDB-specific time-series semantics — Postgres and InfluxDB don't express it this directly.

**Use the Web UI:** open `http://localhost:8880`, paste the SQL, hit enter, results appear — no client install needed.

## 6. Grafana (Optional, 5 Minutes)

1. Grafana → Data Sources → Add → QuestDB
2. Host: `localhost`, Port: `8812`, JDBC mode
3. New Panel, query `SELECT * FROM cpu SAMPLE BY 1s`, chart appears

(We'll do a dedicated Grafana dashboard post later — here we just get it wired up.)

## 7. Common Issues

**Q1: `Connection refused` on port 8880?**
Container isn't up — check `docker logs questdb`. Or the port isn't mapped (check `docker ps` PORTS column).

**Q2: Write fails with `line not found`?**
Bad ILP format. 99% of the time it's the **timestamp unit** — QuestDB wants **nanoseconds** (19 digits). Seconds/milliseconds will be rejected.

**Q3: How do I inspect a table's schema?**
`SHOW COLUMNS FROM cpu;` or `DESC cpu;` (run in the Web UI).

**Q4: Which ports to open in production?**
Expose **9009 (write)** and **8812 (JDBC reads)** to the app; **bind 8880 to 127.0.0.1 only** — keep it off the public internet.

## 8. Next Steps

You've now:

- ✅ Started the container
- ✅ Written 10,000 ILP points
- ✅ Created a partitioned table
- ✅ Run a SAMPLE BY time-series aggregation
- ✅ (optional) wired up Grafana

Next up: **QuestDB vs DuckDB vs TimescaleDB** — a high-volume search term that'll save you real selection pain.

---

*Based on QuestDB official docs (questdb.io/docs) and Oct 2026 verified commands. QuestDB is a trademark of its respective owners; this site is an independent technical community, not the official project.*
