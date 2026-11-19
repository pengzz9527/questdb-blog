---
title: "QuestDB 异常检测实战：滚动 Z-score 与金融 tick 去重回测"
description: "QuestDB 异常检测实战：用窗口聚合实现滚动 Z-score 抓时序尖峰，再对金融 tick 做按时间戳去重回测。覆盖窗口帧写法、去重语义、回测指标计算与常见坑，含可运行 SQL，附 Grafana 告警阈值建议。"
date: 2026-11-04
lastmod: 2026-11-04
slug: questdb-anomaly-detection-zscore
tags:
  - QuestDB
  - 异常检测
  - Z-score
  - 金融 tick
  - 窗口聚合
categories:
  - 实战
toc: true
---

# QuestDB 异常检测实战：滚动 Z-score 与金融 tick 去重回测

有了数据之后，下一步几乎都是"哪个点不对劲"。时序数据里，"不对劲"通常是**相对自己过去 N 秒/分的均值和标准差偏离太远**——这就是滚动 Z-score。在金融 tick 场景里，它还叠一个前置动作：交易所重发、网关去重不彻底，同一个 tick 会有重复时间戳，回测前必须先去重。

这篇给两条可跑的 SQL：一条做滚动 Z-score 抓尖峰，一条做 tick 去重回测。两张表都能直接建，阈值怎么定也讲清楚。

## 一、表结构：先想清楚"窗口帧"落在哪

滚动 Z-score 需要两样东西：当前值、以及它前面 N 个采样点的均值与标准差。在 QuestDB 里用窗口聚合（`OVER (ORDER BY ts ROWS ...)`）实现。

```sql
CREATE TABLE sensor (
    ts TIMESTAMP,
    host SYMBOL CAPACITY 1024 INDEX,
    value DOUBLE
) PARTITION BY DAY;
```

金融 tick 表多一列 `seq`（交易所序号）和 `price`/`vol`，并保留原始 `ts` 到纳秒：

```sql
CREATE TABLE tick (
    ts TIMESTAMP,
    sym SYMBOL CAPACITY 512 INDEX,
    seq BIGINT,
    price DOUBLE,
    vol DOUBLE
) PARTITION BY DAY;
```

## 二、滚动 Z-score 抓尖峰

思路：每个点取它前面 60 个点（约 60 秒，取决于采样率）的均值 `mu` 和标准差 `sigma`，算 `z = (value - mu)/sigma`，`|z|` 超过阈值就标异常。

```sql
SELECT
    ts, host, value,
    -- 窗口：当前点之前 60 行（不含当前）
    AVG(value)   OVER (PARTITION BY host ORDER BY ts
                       ROWS BETWEEN 60 PRECEDING AND 1 PRECEDING) AS mu,
    STDEVP(value) OVER (PARTITION BY host ORDER BY ts
                       ROWS BETWEEN 60 PRECEDING AND 1 PRECEDING) AS sigma
FROM sensor
ORDER BY host, ts
```

再套一层算 Z 并筛选（QuestDB 不支持直接在 OVER 里引用别名，所以用 CTE）：

```sql
WITH w AS (
    SELECT ts, host, value,
        AVG(value)    OVER w AS mu,
        STDEVP(value) OVER w AS sigma
    FROM sensor
    WINDOW w AS (PARTITION BY host ORDER BY ts
                 ROWS BETWEEN 60 PRECEDING AND 1 PRECEDING)
)
SELECT ts, host, value,
       (value - mu) / NULLIF(sigma, 0) AS z
FROM w
WHERE ABS(value - mu) / NULLIF(sigma, 0) > 3
ORDER BY ts;
```

几个要点：

- `NULLIF(sigma, 0)` 防除零——平稳序列方差接近 0，否则整列炸成 inf。
- 前 60 行还没凑够窗口时 `mu`/`sigma` 为 NULL，`ABS` 比较会被跳过，正好是"预热期不告警"。
- 阈值 3 是统计常态，但金融 tick 分布厚尾，常用 4~5 或按分位数调，别死卡 3。

## 三、金融 tick 去重：回测前必做

交易所流常带**重复时间戳**（重发、多网关）。直接用原始 tick 回测，同一时刻算多次，成交与信号都会偏。去重原则：**同一 `sym` 下同一 `ts`，按 `seq` 取最大**（序号最大=最终状态），其余丢掉。

```sql
-- 每个 (sym, ts) 只保留 seq 最大的那条
SELECT ts, sym, seq, price, vol
FROM (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY sym, ts
            ORDER BY seq DESC
        ) AS rn
    FROM tick
)
WHERE rn = 1
ORDER BY sym, ts;
```

如果同一 `ts` 下 `seq` 也不保证单调（网关乱序），改用 `seq` 严格去重：

```sql
-- 去重：同 sym 同 seq 只留一条（防彻底重复投递）
SELECT DISTINCT sym, seq, ts, price, vol FROM tick;
```

**别用 `DISTINCT` 全行去重当去重主力**——它按所有列比，价格/量有一点点抖动就留成两行。按 `seq`（或 `(sym,ts)` 取最新）才稳。

## 四、去重回测：算信号与命中

去重后的 tick 上回测"价格突破阈值"，并统计命中次数与盈亏：

```sql
WITH clean AS (
    SELECT ts, sym, seq, price,
        LAG(price) OVER (PARTITION BY sym ORDER BY seq) AS prev_price
    FROM (
        SELECT * FROM (
            SELECT *, ROW_NUMBER() OVER (PARTITION BY sym, ts ORDER BY seq DESC) AS rn
            FROM tick
        ) WHERE rn = 1
    )
)
SELECT sym,
       COUNT(*) AS n_signals,
       SUM(CASE WHEN price > prev_price * 1.001 THEN 1 ELSE 0 END) AS up_breaks,
       AVG(price - prev_price) AS avg_delta
FROM clean
WHERE prev_price IS NOT NULL
GROUP BY sym;
```

`n_signals` 应明显小于原始行数——差值就是被去重掉的重复 tick，这个数要写进回测报告，证明你没在用脏数据。

## 五、坑与调优

- **窗口帧太大**：`ROWS 10000 PRECEDING` 会把整个历史摊进每个点，窗口聚合开销随帧线性涨。按采样率定帧：1 秒采样用 60~300 行，毫秒 tick 用 100~500 行。
- **SYMBOL 基数爆炸**：`PARTITION BY host/sym` 的分片数 = 基数。设备上万时给 SYMBOL 设 `CAPACITY`（建表时），不然运行时扩字典会卡。
- **厚尾别用 Z-score 死标准**：tick 收益分布尖峰厚尾，Z>3 的"异常"可能一大片正常波动。改用**中位数 + MAD**（绝对偏差中位数）更抗离群：

```sql
SELECT ts, host, value,
    value - MEDIAN(value) OVER w AS dev,
    MEDIAN(ABS(value - MEDIAN(value) OVER w)) OVER w AS mad  -- 简化示意
FROM sensor
WINDOW w AS (PARTITION BY host ORDER BY ts ROWS 60 PRECEDING);
```

（实际生产建议先聚合成 MAD 序列再取中位数，避免嵌套窗口。）

- **Grafana 告警阈值**：Z-score 面板配两个阈值——`|z|>3` 黄、`|z|>4.5` 红，窗口用"最近 5 分钟且连续 3 个点"降噪，避免单点毛刺误报。

## 六、下一步

上一篇讲了数据保留与冷热分层，这篇把保留期内的数据用起来做检测。再往下可以接：把 Z-score 结果写回一张 `anomaly` 表，配 `SAMPLE BY` 出每小时异常计数面板，形成"检测→告警→回溯"闭环。

## 参考

- [QuestDB 官方文档：SELECT](https://questdb.io/docs/reference/sql/select/)
- [Python statistics 标准库文档](https://docs.python.org/3/library/statistics.html)

## 相关文章

- [QuestDB + Kafka 实时管道：从 ILP 写入到 SAMPLE BY 聚合](/zh/post/questdb-kafka-connect-pipeline/)
- [QuestDB SAMPLE BY 完全指南：时序数据的灵魂聚合语法](/zh/post/questdb-sample-by-complete-guide/)
- [QuestDB 加密交易异常检测：tick 数据实时告警实战](/zh/post/questdb-crypto-trading-anomaly-detection/)
