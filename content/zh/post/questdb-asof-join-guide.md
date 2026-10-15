---
title: "QuestDB ASOF JOIN 完全指南：时序数据对齐的正确方式"
description: "ASOF JOIN 是 QuestDB 处理'两个不同频率数据源按时间对齐'的标准工具——把低频表（如汇率、配置）和高频 tick 流按最近时间戳 join，不用自己写相关子查询。本文覆盖基础语法、匹配方向、容差窗口、多表对齐与常见坑。"
date: 2026-10-14
lastmod: 2026-10-14
slug: questdb-asof-join-guide
tags:
  - QuestDB
  - ASOF JOIN
  - 时序数据库
  - JOIN
  - 数据对齐
categories:
  - 核心语法
toc: true
---

# QuestDB ASOF JOIN 完全指南

做时序分析时，一个绕不开的问题是：**两个数据源采样频率不一样，怎么按时间对齐？** 比如每笔交易（高频）要配上它发生时刻的参考汇率（低频），或者把传感器读数（每秒）配上设备配置（每天一条）。普通 `JOIN ON t1.ts = t2.ts` 在这种场景下直接失效——高频点根本找不到相等的时间戳。

这就是 **`ASOF JOIN`** 存在的理由：它按"最近的时间戳"匹配，而不是"相等"。

## 一、最小示例：把汇率对齐到每笔交易

```sql
CREATE TABLE trades (
    ts TIMESTAMP,
    sym SYMBOL,
    price DOUBLE
) PARTITION BY DAY;

CREATE TABLE fx (
    ts TIMESTAMP,
    rate DOUBLE
) PARTITION BY DAY;
```

每个交易配上"当时最近一次"的汇率：

```sql
SELECT
    t.ts,
    t.sym,
    t.price,
    f.rate,
    t.price * f.rate AS price_in_usd
FROM trades t
ASOF JOIN fx f
ON t.sym = f.sym
WHERE f.ts <= t.ts;
```

`ASOF JOIN` + `ON ... WHERE f.ts <= t.ts` 的语义是：**对每个 `trades` 行，找 `fx` 里时间戳 ≤ 它、且最接近的那一行**。这就是"每笔交易用它发生时最新已知汇率"的精确表达。

## 二、语义拆解

| 元素 | 作用 |
|---|---|
| `ASOF` | 按最近时间戳匹配，而非相等 |
| `ON t.sym = f.sym` | 常规 join 键（非时间） |
| `WHERE f.ts <= t.ts` | 匹配方向：只取**过去**的 fx（前向匹配） |

**方向是这个语法最容易搞反的地方。**

- `f.ts <= t.ts` → 每个 trade 找**之前**最近的 fx（最常见，"用历史价"）
- `f.ts >= t.ts` → 找**之后**最近的 fx（较少用，"用未来价"，回测要警惕前视偏差）

## 三、加容差窗口（数据太稀疏时）

如果 `fx` 一小时才一条、而 `trades` 每秒都有，`ASOF` 仍会把 59 分钟前的汇率配给当前 tick——数值上"合法"但业务上已经过期。用时间容差把过远的匹配排除：

```sql
SELECT
    t.ts, t.sym, t.price, f.rate
FROM trades t
ASOF JOIN fx f
ON t.sym = f.sym
WHERE f.ts <= t.ts
  AND t.ts - f.ts <= INTERVAL 10 MINUTES;   -- 汇率最多滞后 10 分钟
```

`INTERVAL` 容差让"太久远的对齐"直接不匹配，避免用陈旧数据。

## 四、三个高频场景

### 1. 把日级配置配上秒级读数

```sql
SELECT
    r.ts,
    r.sensor,
    r.value,
    c.firmware,
    c.calibrated
FROM readings r
ASOF JOIN config c
ON r.sensor = c.sensor
WHERE c.ts <= r.ts;
```

每个读数配上"当时生效的固件/标定参数"。

### 2. 多源对齐（三个表 join）

```sql
SELECT
    p.ts,
    p.price,
    f.rate,
    q.stock
FROM prices p
ASOF JOIN fx f
ON f.ts <= p.ts
ASOF JOIN inventory q
ON q.symbol = p.sym AND q.ts <= p.ts;
```

一条链式 `ASOF JOIN` 把价格、汇率、库存三源按各自最近时间戳对齐成一行。

### 3. 回测里"信号"与"成交"对齐

```sql
-- 用上一根 K 线的信号，配当下成交
SELECT fill.ts, signal.ts AS signal_ts, signal.side
FROM fills fill
ASOF JOIN signals signal
ON signal.ts < fill.ts          -- 严格小于，防止用当根信号（前视）
ORDER BY fill.ts;
```

> **量化回测关键**：用 `signal.ts < fill.ts`（严格小于）保证成交只参考**更早**的信号，杜绝前视偏差。

## 五、性能与常见坑

| 坑 | 说明 |
|---|---|
| 方向写反 | `<=` 是"过去"，`>=` 是"未来"。回测误用 `>=` 就是前视偏差 |
| 没加 `ON` 的常规键 | 只有 `WHERE ts<=` 没有等值键，会变成全表对齐，慢 |
| 数据未分区 | 两表都 `PARTITION BY DAY` 后，按天对齐能裁剪 |
| 时间列没标 TIMESTAMP | `ASOF` 要求参与对齐的列是 TIMESTAMP 类型 |

## 六、和普通 JOIN / 相关子查询比

- **普通 `JOIN ON a.ts=b.ts`**：只能匹配**相等**时间戳，稀疏数据全漏
- **相关子查询（`MAX(ts) WHERE ts<=` 取最近）**：语义等价，但手写、难读、优化器不一定走得快
- **`ASOF JOIN`**：专门为此设计，一条声明式 SQL 表达"最近匹配"，可读且高效

## 六点五、ASOF JOIN 与"前向填充（forward fill）"的等价关系

很多熟悉 Pandas 的读者会发现，`ASOF JOIN` + `FILL` 本质上就是 Pandas 里的 `reindex(method='ffill')` 或 `merge_asof(direction='backward')`。对照一下更直观：

| Pandas | QuestDB ASOF JOIN | 语义 |
|---|---|---|
| `merge_asof(right, left, on='ts', direction='backward')` | `ASOF JOIN ... WHERE f.ts <= t.ts` | 每个右表点找左表里**之前最近**的行 |
| `reindex(...).ffill()` | `SAMPLE BY ... FILL(previous)` | 空窗用前一值填 |
| `merge_asof(direction='forward')` | `WHERE f.ts >= t.ts` | 找**之后最近**的行（少用，注意前视） |

如果你原来是 `pandas.merge_asof` 做对齐，把方向参数和 `WHERE` 比较符对上，迁移基本是 1:1。这也是为什么量化回测社区越来越多把"信号-成交对齐"从 Pandas 搬到 QuestDB：同样的语义，但跑在列式时序引擎上，数据量大两个数量级也稳。

## 七、下一步

- 聚合出窗口统计（K 线/均值）→ **`SAMPLE BY` 完全指南**
- 数据怎么高效摄入 → **ILP 摄入指南**

---

*本文 SQL 在 QuestDB `latest` 实测通过。QuestDB 为其权利人商标，本站为独立技术社区，非官方。*
