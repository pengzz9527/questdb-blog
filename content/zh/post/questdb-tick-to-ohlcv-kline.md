---
title: "金融 tick 数据生成 K 线：QuestDB SAMPLE BY 计算 OHLCV"
description: "金融 tick 数据存进 QuestDB 就能自己算 K 线：SAMPLE BY 一行出 1 分钟 K 线，FIRST/MAX/MIN/LAST 取 OHLC，SUM 出成交量。覆盖建表、ILP 写入、多 symbol 采样、日线、大周期重采样坑与 ASOF JOIN"
date: 2026-11-15
lastmod: 2026-11-15
slug: questdb-tick-to-ohlcv-kline
tags:
  - QuestDB
  - 金融数据
  - K线
  - OHLCV
  - SAMPLE BY
categories:
  - 实战
toc: true
---

# 金融 tick 数据生成 K 线：QuestDB SAMPLE BY 计算 OHLCV

K 线是行情里最基础的数据结构。多数团队从行情 API 拿 K 线，代价是 candle 周期、交易时段边界、时区、成交量口径全部失去控制权，换个供应商历史数据就要重新对账。如果你的 QuestDB 里已经有 tick 级数据——交易所原始 feed、回测成交记录、加密订单簿快照——K 线可以自己算：一行 `SAMPLE BY` 出 OHLCV，周期想换就换。

## 一、自己算 K 线买到什么

- **任意周期**：7 分钟、30 分钟、4 小时、周线，改一下 SAMPLE BY 的参数。
- **时段与时区边界**：亚洲盘 09:00–16:00、美盘、加密 24h，按业务精确切。
- **成交量口径**：成交量、笔数、成交均价，同一套聚合里一起算。
- **不依赖供应商**：行情 API 的 K 线档通常是最贵的一档。

代价是要钉死 K 线时间戳的两个约定：K 线上的 `ts` 是**窗口开始时间**（open time），不是收盘时间。搞混之后每张图都会偏移一个窗口，回测信号全部错位。

## 二、数据准备：ticks 表 + ILP 写入

```sql
CREATE TABLE ticks (
    ts TIMESTAMP,
    symbol SYMBOL,
    price DOUBLE,
    volume DOUBLE
) TIMESTAMP ts PARTITION BY DAY;
```

用 ILP 写入，一笔成交一行，同一行挂多个字段：

```
ticks,symbol=BTCUSD price=67213.4,volume=0.0042 1700000000123456789
ticks,symbol=BTCUSD price=67215.1,volume=0.0110 1700000000456789012
```

回测前确认已按 (ts, symbol) 去重，重复时间戳会让 OHLC 失真——去重写法见本站《滚动 Z-score 与金融 tick 去重回测》一文。下面假设输入已去重。

## 三、核心 SQL：1 分钟 K 线

```sql
SELECT
    ts,
    FIRST(price) AS open,
    MAX(price)   AS high,
    MIN(price)   AS low,
    LAST(price)  AS close,
    SUM(volume)  AS volume
FROM ticks
WHERE symbol = 'BTCUSD'
SAMPLE BY 1m;
```

要点：

- **窗口锚定**：`SAMPLE BY 1m` 的窗口按该窗口内第一行时间戳对齐，输出的 `ts` 是窗口开始时间。
- **空窗默认跳过**：没有数据的窗口直接不输出（等价 SKIP EMPTY），K 线序列保持连续语义。
- **保留空洞**：想让空窗沿用上一窗口的收盘价（画图不断线），写 `SAMPLE BY 1m FILL PREVIOUS`。

## 四、多 symbol 一次出 K 线

给 SAMPLE BY 加分区子句，每个 symbol 独立采样：

```sql
SELECT
    symbol,
    ts,
    FIRST(price) AS open,
    MAX(price)   AS high,
    MIN(price)   AS low,
    LAST(price)  AS close,
    SUM(volume)  AS volume
FROM ticks
SAMPLE BY 1m PARTITION BY symbol;
```

注意 `PARTITION BY symbol` 是 SAMPLE BY 的采样分区，和建表时的物理分区是两回事，别混。

## 五、大周期：30 分钟 / 1 小时 / 日线

参数直接换：

```sql
SELECT
    ts,
    FIRST(price) AS open,
    MAX(price)   AS high,
    MIN(price)   AS low,
    LAST(price)  AS close,
    SUM(volume)  AS volume
FROM ticks
WHERE symbol = 'BTCUSD' AND ts >= now() - INTERVAL 30 DAY
SAMPLE BY 30m;
```

`SAMPLE BY 1d` 默认按 UTC 午夜对齐。想按交易时段出 K 线（比如只算 09:00–16:00），先按时段过滤再做 1m 采样，应用层拼。另外「当前未完成窗口」那根是活 K 线，close 不是真收盘，回测引擎里要剔除。

## 六、大坑：大周期 K 线不能从小周期「用 close 重采样」

已预聚合 1m K 线，要出 1h K 线可以，但 high/low 必须取小周期的 high/low：

```sql
SELECT
    ts,
    FIRST(open)  AS open,
    MAX(high)    AS high,
    MIN(low)     AS low,
    LAST(close)  AS close,
    SUM(volume)  AS volume
FROM candles_1m
SAMPLE BY 1h;
```

常见错误是 `MAX(close)` / `MIN(close)`：close 只是窗口内一个点，会漏掉窗口内真实极值。还有一个隐藏坑：小周期不能整除大周期时（7 分钟拼 1 小时），对齐就断了，只能回原始 ticks 重算。

## 七、预聚合 K 线表出图

图表和看板不需要每次看几十亿 tick，预聚合成 1m K 线表是标准做法：

```sql
CREATE TABLE candles_1m (
    ts TIMESTAMP,
    symbol SYMBOL,
    open DOUBLE,
    high DOUBLE,
    low DOUBLE,
    close DOUBLE,
    volume DOUBLE
) TIMESTAMP ts PARTITION BY DAY;

INSERT INTO candles_1m
SELECT
    ts,
    symbol,
    FIRST(price),
    MAX(price),
    MIN(price),
    LAST(price),
    SUM(volume)
FROM ticks
WHERE ts < now() - INTERVAL 1 MINUTE
SAMPLE BY 1m PARTITION BY symbol;
```

调度定时跑（cron），增量写入「最后一个完整窗口」。如果当前版本的 INSERT 不接受 SAMPLE BY 子句，拆两步：先 SELECT 进临时表再转存。

## 八、和指数数据 ASOF JOIN

个股 K 线要和指数联动，两边采样点不一定重合，用 ASOF JOIN 对齐到同一 K 线时间轴上最近的点：

```sql
SELECT
    t.ts,
    t.close AS close,
    i.close AS index_close
FROM candles_1m t
ASOF JOIN candles_1m i ON t.ts = i.ts
WHERE t.symbol = '600519' AND i.symbol = 'SPY';
```

## 九、性能直觉

SAMPLE BY 跑在列式存储 + 分区裁剪上：WHERE 时间过滤只扫对应分区，一天数据的 1m K 线聚合在毫秒到秒级完成，十亿 tick 也无所谓。同样的事用 Python 逐行循环会慢几个数量级，聚合全部进 SQL。

## 十、下一步

- **出图**：K 线表进 Grafana candlestick 面板，接 Grafana 与 QuestDB 数据源全流程见本站《零成本自建 IoT 仪表板》一文。
- **回测**：读 1m K 线 + ASOF JOIN 对齐，策略信号在回测引擎里算。
- **异动**：对 1m K 线的 close 做滚动 Z-score，写法见本站异常检测一文。

## 参考

- [QuestDB 官方文档：SAMPLE BY](https://questdb.io/docs/reference/sql/sample-by/)
- [QuestDB 官方文档：ASOF JOIN](https://questdb.io/docs/reference/sql/asof-join/)

## 相关文章

- [QuestDB SAMPLE BY 完全指南：时序数据的灵魂聚合语法](/zh/post/questdb-sample-by-complete-guide/)
- [QuestDB 加密交易异常检测：tick 数据实时告警实战](/zh/post/questdb-crypto-trading-anomaly-detection/)
- [QuestDB 异常检测实战：滚动 Z-score 与金融 tick 去重回测](/zh/post/questdb-anomaly-detection-zscore/)
