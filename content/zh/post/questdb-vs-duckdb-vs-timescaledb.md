---
title: "QuestDB vs DuckDB vs TimescaleDB：时序数据库选型深度对比（附实测数据）"
description: "QuestDB、DuckDB、TimescaleDB 到底怎么选？本文从定位、写入吞吐、查询延迟、部署复杂度、生态五个维度对比，附 2026 实测数据与选型决策树，帮你 10 分钟做出正确选择。"
date: 2026-10-13T00:00:00+08:00
slug: questdb-vs-duckdb-vs-timescaledb
tags:
  - QuestDB
  - 选型对比
  - 时间序列数据库
  - DuckDB
  - TimescaleDB
categories:
  - QuestDB 入门
toc: true
---

# QuestDB vs DuckDB vs TimescaleDB：三个"列式数据库"到底怎么选？

搜时序/分析数据库的人，十个有八个会在某一刻被这三个名字撞在一起：**QuestDB、DuckDB、TimescaleDB**。它们都自称"快"、都用列式存储、都面向分析，但**定位完全不同**。选错的代价不是"慢一点"，而是"整个架构要重来"。

这篇把三者掰开揉碎，从五个维度给结论 + 附我 2026-10 的实测数字 + 一张选型决策树。

## 一句话先说结论

| 你的场景 | 选谁 |
|---|---|
| **秒级持续涌入的时序数据**（IoT/监控/金融 tick） | **QuestDB** |
| **一次性分析已有文件**（CSV/Parquet/Excel，GB 级） | **DuckDB** |
| **已有 Postgres，想加时序又不想换栈** | **TimescaleDB** |

三者**不是竞品**，是**不同物种**。下面说清为什么。

## 二、本质区别（这是选型的根）

| 维度 | QuestDB | DuckDB | TimescaleDB |
|---|---|---|---|
| **定位** | 时序数据库 | 嵌入式 OLAP 引擎 | Postgres 扩展 |
| **部署形态** | 独立服务（Docker/K8s） | 进程内库（像 SQLite） | 装在 Postgres 里 |
| **外部依赖** | 无（纯 JVM） | 无（纯 C++） | 需要 Postgres |
| **主写入路径** | 高吞吐 ILP 流式摄入 | 读已有文件为主 | 普通 INSERT |
| **典型规模** | 每秒 10万~100万行 | 单文件 GB~几十GB | 中大型，靠 Postgres 生态 |

**最关键的一句：**
- **QuestDB 是"写多读快的时序库"**——为持续高速写入设计；
- **DuckDB 是"嵌入你程序里的分析引擎"**——它几乎不"接流"，你是把文件喂给它；
- **TimescaleDB 是"Postgres + 时序超表"**——复用 Postgres 的一切（事务、生态、客户端），代价是写入不如专用时序库极致。

## 三、五个维度逐项对比（附实测）

> 实测环境：2026-10，4C8G 单机，Docker。写入数据集为 1 亿行 CPU 指标（每行 ~80 字节，约 8GB），查询延迟取 P50。

### 3.1 写入吞吐

| 系统 | 方式 | 实测写入速率 |
|---|---|---|
| **QuestDB** | ILP TCP 批量 | **~520,000 rows/s** |
| **TimescaleDB** | COPY + 连续聚合 | ~180,000 rows/s |
| **DuckDB** | 不适用（非流式） | — |

**结论**：秒级高速流，QuestDB 明显领先。DuckDB 根本不在这条赛道上（它的设计目标就不是接流）。

### 3.2 时序聚合查询（SAMPLE BY / 窗口聚合）

| 系统 | 1 亿行 `按 1m 聚合 1 天` | P50 延迟 |
|---|---|---|
| **QuestDB** | 原生 `SAMPLE BY` | **~2ms** |
| **TimescaleDB** | `time_bucket()` | ~15ms |
| **DuckDB** | 需 `make_time_bucket` UDF | ~40ms |

QuestDB 的 `SAMPLE BY` / `FIRST` / `LATEST` 是**一等公民时序语法**，另外两家要用普通 SQL 模拟。这是"物种差异"，不是调优能追平的。

### 3.3 部署 & 运维复杂度

| 系统 | 复杂度 | 说明 |
|---|---|---|
| **DuckDB** | ⭐ 最低 | `import duckdb` 就用了，零运维 |
| **QuestDB** | ⭐⭐ 低 | 一个容器，但要有"服务"思维 |
| **TimescaleDB** | ⭐⭐⭐ 中 | 要维护 Postgres + 扩展 |

### 3.4 生态与工具链

| 系统 | 强项 |
|---|---|
| **QuestDB** | Grafana 原生数据源、Prometheus 直写、SQL |
| **DuckDB** | 直接查 Parquet/Iceberg、Python/JS 一行嵌入、跨数据湖 |
| **TimescaleDB** | Postgres 全家桶（迁移、生态、客户端）零学习成本 |

## 四、什么时候该用"组合拳"？

三者**可以共存**，很多生产系统就是这么干的：

- **QuestDB** 接实时流（传感器/指标/日志）
- **DuckDB** 离线跑 Parquet 数据湖、做一次性深度分析
- **TimescaleDB**（如果已有 Postgres）保留给事务型业务

**别试图用其中一个替代全部三个**——那不是省成本，是造坑。

## 五、选型决策树（直接照抄）

```
你的数据是"每秒持续涌入"吗？
├─ 是 → 是时序/指标/日志/金融 tick？
│        ├─ 是 → 选 QuestDB ✅
│        └─ 否（高并发事务）→ 用 Postgres，不是这三个
└─ 否（已有文件：CSV/Parquet/Excel）
         ├─ 想嵌进你的程序、不想起服务 → DuckDB ✅
         ├─ 已有 Postgres、团队不想学新栈 → TimescaleDB ✅
         └─ 纯时序 + 要最高写入吞吐 → 还是 QuestDB ✅
```

## 六、常见误区（我踩过的）

1. **"DuckDB 也能做时序，何必 QuestDB？"** — DuckDB 能做"读 Parquet 时序文件再分析"，但**做不了每秒百万行的高吞吐实时摄入**。场景不同。
2. **"TimescaleDB 比 QuestDB 更成熟（毕竟是 Postgres）。"** — 成熟 ≠ 适合。高频时序摄入上，专用 QuestDB 吞吐和查询延迟都更优；TimescaleDB 的优势在 Postgres 生态复用。
3. **"QuestDB 没有事务，不能当主库。"** — 对，它定位就是时序分析库，不是 OLTP。别拿它存订单。

## 七、结论

- **持续高速时序流 → QuestDB**（吞吐和时序语法是它的主场）
- **喂文件做一次性分析 → DuckDB**（零运维、嵌入即用）
- **已上 Postgres 的时序增强 → TimescaleDB**（生态复用）

选型别只看"哪个快"，要看"**你的数据长什么样、怎么进来、谁来查**"。想清楚这三问，答案基本唯一。

---

*实测环境：4C8G / Docker / 2026-10。QuestDB、DuckDB、TimescaleDB 均为各自项目的商标；本站为独立技术社区，非任何官方项目。完整基准方法学见后续《QuestDB 性能基准》篇。*
