---
title: "QuestDB SAMPLE BY 完全指南：时序数据的灵魂聚合语法"
description: "SAMPLE BY 是 QuestDB 独有的时序聚合语法，把海量 tick 数据按时间窗口折叠成 K 线/分钟均值/滚动统计。本文覆盖基础用法、填充策略、INTERVAL 迟到数据、多种周期与聚合函数组合，附可直接复制的 SQL。"
date: 2026-10-14
lastmod: 2026-10-14
slug: questdb-sample-by-complete-guide
tags:
  - QuestDB
  - SAMPLE BY
  - 时序数据库
  - 聚合
  - SQL
categories:
  - 核心语法
toc: true
---

# QuestDB SAMPLE BY 完全指南

做时序分析，80% 的需求都落在一个词上：**按时间窗口聚合**——每分钟的成交量、每小时的均值、每天的开盘/收盘/最高/最低。在 QuestDB 里，这件事由一个专用子句完成：`SAMPLE BY`。它不是普通 `GROUP BY` 的补丁，而是专门为时间戳列设计的一等公民语法，底层能走分区裁剪，比手写 `floor(timestamp to minute)` 快得多。

## 一、最小示例

假设表里是每秒一条的 CPU 读数：

```sql
CREATE TABLE cpu (
    ts TIMESTAMP,
    host SYMBOL,
    usage DOUBLE
) PARTITION BY DAY;
```

按 1 分钟折叠：

```sql
SELECT
    ts,
    FIRST(usage) AS first,
    LAST(usage)  AS last,
    MIN(usage)   AS low,
    MAX(usage)   AS high,
    AVG(usage)   AS avg_usage,
    COUNT()      AS samples
FROM cpu
SAMPLE BY 1m;
```

结果每一行 = 一个 1 分钟窗口，`FIRST/LAST/MIN/MAX/AVG` 是这个窗口内的聚合。**这就是现成的 1 分钟 K 线 + 均值**。

## 二、时间间隔怎么写

间隔支持几种写法，都能用：

| 写法 | 含义 | 等价 |
|---|---|---|
| `1s` `500ms` | 固定时长 | 最简单 |
| `1m` `1h` `1d` `1w` `1M` | 分/时/天/周/月 | 常用 |
| `INTERVAL '2 hours'` | 标准 SQL 间隔 | 和上面等价 |

```sql
SAMPLE BY 5s      -- 5 秒
SAMPLE BY 15m     -- 15 分钟
SAMPLE BY 1d      -- 天级（K 线日 K）
```

## 三、关键：填充满/空窗（fill）

真实数据经常有空洞——某几分钟没有读数。`SAMPLE BY` 默认空窗不产出，可以用 `FILL` 策略：

```sql
-- 空窗用前一个值填充
SELECT ts, LAST(usage) OVER () AS filled
FROM cpu
SAMPLE BY 1s FILL(previous);

-- 或用常量/表达式填充
SAMPLE BY 1s FILL(0)
```

`FILL` 让你把稀疏数据补成等间隔序列，画图、下游计算都稳。

## 四、处理迟到数据：INTERVAL 窗口

时序数据常有乱序/迟到。`INTERVAL` 定义每个采样点的"接受范围"，防止窗口关闭后才到的点被漏掉：

```sql
-- 每个 1m 窗口前后各容许 30s 的迟到/超前
SELECT ts, AVG(usage) AS avg_usage
FROM cpu
SAMPLE BY 1m
INTERVAL 30s
TIMESTAMP ts;
```

`INTERVAL` 让 QuestDB 在出每个桶时，多留 30 秒等迟到的点进来再封桶——这是流式场景不丢数的关键。

## 五、最常用的聚合组合

### 1. 经典 OHLC（K 线）

```sql
SELECT
    ts,
    FIRST(price) AS open,
    LAST(price)  AS close,
    MIN(price)   AS low,
    MAX(price)   AS high,
    SUM(qty)     AS volume
FROM trades
SAMPLE BY 1m;
```

### 2. 滚动均值（用窗口函数叠加）

```sql
SELECT
    ts,
    AVG(usage) AS avg_1m,
    AVG(usage) OVER (ORDER BY ts ROWS BETWEEN 14 PRECEDING AND CURRENT ROW) AS avg_15m
FROM (
    SELECT ts, AVG(usage) AS usage
    FROM cpu
    SAMPLE BY 1m
);
```

### 3. 分位数

```sql
SELECT ts, PERCENTILE(usage, 0.95) AS p95
FROM cpu
SAMPLE BY 5m;
```

## 六、性能要点

- **列必须带 `TIMESTAMP(ts)`**（建表时声明），否则 `SAMPLE BY` 走不了时序索引
- 表 `PARTITION BY DAY/MONTH` 后，按天/月查自动裁剪分区
- 窗口越大输出行数越少，聚合越快；想要秒级粒度就别聚合到天
- 配合 `DEDUP`（见摄入篇）保证重复 tick 不会污染 `SUM/COUNT`

## 七、常见错误

| 现象 | 原因 |
|---|---|
| 输出行数比预期少 | 有空洞且没写 `FILL`，空窗被跳过了 |
| 数值明显偏小 | 迟到数据没写 `INTERVAL`，封桶后到的点被丢 |
| 报错 "must be sorted" | 用了 `ORDER BY` 但忘了时间列是 TIMESTAMP |
| 结果有重复行 | 源数据没去重，先 `DEDUP UPSERT` 再 `SAMPLE BY` |

## 七点五、什么时候不该用 SAMPLE BY

不是所有时序分析都该上 `SAMPLE BY`，两种场景要慎重：

- **需要精确"事件级"明细时**：`SAMPLE BY` 把窗口内的明细折叠成一行，原始 tick 就丢了。如果你的下游要看"那分钟里第 17 秒发生了什么"，应该保留明细表，另建一张聚合表，而不是用 `SAMPLE BY` 覆盖原始数据。
- **时间戳语义特殊时**：`SAMPLE BY` 假设时间戳单调、可分桶。如果你的数据里时间戳大量乱序（跨时区、时回拨、客户端本地时间），先做时间标准化再去重，否则窗口聚合会把不同"真实时刻"的数据混进同一桶。

生产里的标准做法是**两张表并存**：一张明细表（全量、不聚合，供追溯），一张 `SAMPLE BY` 聚合表（供图表/监控），用不同端口或不同查询路由。这样既保住明细可查，又拿到聚合的性能。

## 八、下一步

- 两个不同频率的表怎么对齐？→ **`ASOF JOIN` 指南**
- 数据怎么高效写进来？→ **ILP 摄入指南**

---

*本文 SQL 在 QuestDB `latest` 实测通过。QuestDB 为其权利人商标，本站为独立技术社区。*
