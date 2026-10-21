---
title: "QuestDB SQL 速查表：SAMPLE BY / ASOF JOIN / SYMBOL / 分区一页汇总"
description: "QuestDB 高频 SQL 一页速查：SAMPLE BY 全模式（含 FILL/ALIGN/OFFSET）、ASOF JOIN 方向与容差、SYMBOL 索引与基数调优、PARTITION BY 建表与保留、ILP 写入模板，以及 20 条最常被问到的时序查询。收藏这一页，少翻文档。"
date: 2026-10-21
lastmod: 2026-10-21
slug: questdb-cheatsheet
tags:
  - QuestDB
  - 速查表
  - SQL
  - 时序数据库
categories:
  - 实战
toc: true
---

# QuestDB SQL 速查表：SAMPLE BY / ASOF JOIN / SYMBOL / 分区一页汇总

这页是把过去几篇指南里最常被截图收藏的片段拉出来，按"建表 → 写入 → 查询 → 对齐 → 保留"的顺序排好。目标是：接手一个 QuestDB 项目的第一天，不用翻官方文档，对着这页把核心操作跑通。

## 建表：三种最常见的模式

### 模式 1：传感器 / 指标表（低基数 tag + 数值 field）

```sql
CREATE TABLE metrics (
    ts TIMESTAMP,
    host SYMBOL,              -- 机器名，几十到几千
    metric SYMBOL,            -- 指标名，几十
    val DOUBLE
) PARTITION BY DAY;
```

`host` 和 `metric` 都是低基数列，QuestDB 自动走 SYMBOL 编码，不占额外索引空间。

### 模式 2：行情 / tick 表（符号列高频，需要索引）

```sql
CREATE TABLE ticks (
    ts TIMESTAMP,
    sym SYMBOL INDEX 256,     -- 交易对，几百到几千
    px DOUBLE,
    vol DOUBLE
) PARTITION BY HOUR;
```

`INDEX 256` 把符号字典塞进每 256 行的块内索引，比默认 256 外索引查得快一档；`PARTITION BY HOUR` 让"最近 1 小时"查询只扫 1 个分区。

### 模式 3：事件流（高基数 tag，用 VARCHAR 兜底）

```sql
CREATE TABLE events (
    ts TIMESTAMP,
    user_id VARCHAR,          -- 每行唯一，SYMBOL 会爆
    event VARCHAR,
    payload JSON
);
```

`user_id` 基数等于行数，建 SYMBOL 会让字典内存爆炸，直接用 VARCHAR。判断标准：**列里不同值的数量 > 1 万**，就别用 SYMBOL。

## 写入：ILP 三行模板

```bash
# 单行，不指定时间戳（用服务端时间）
echo 'cpu,host=srv1,metric=usage val=42.5' | nc localhost 9009

# 批量 + 显式纳秒时间戳
echo 'cpu,host=srv1,metric=usage val=42.5 1729555200000000000' | nc localhost 9009

# 自动建表：measurement 第一次出现时，QuestDB 根据字段类型自动建表
echo 'mem,host=srv1,metric=used_kb val=1048576' | nc localhost 9009
```

注意：ILP 自动建表时，**第一个出现的非 ts 字符串列会被当成 SYMBOL**，后续同类列跟随。如果想控制哪列是 SYMBOL，提前用 DDL 建好表再写。

## 查询：20 条最常用的

```sql
-- 1. 最近 1 小时每 5 分钟均值
SELECT ts, AVG(val) AS avg_val
FROM metrics WHERE ts >= now() - INTERVAL 1 HOUR
SAMPLE BY 5m;

-- 2. 每 5 分钟最大值 + 对应时间戳
SELECT ts, MAX(val) AS peak, MAX_AGGREGATE_TIMESTAMP(val) AS peak_ts
FROM metrics WHERE ts >= now() - INTERVAL 1 DAY
SAMPLE BY 5m;

-- 3. 空窗口用前一值填（不产生 0 值误导）
SELECT ts, val FROM cpu SAMPLE BY 1m FILL(previous);

-- 4. 空窗口用线性插值
SELECT ts, val FROM cpu SAMPLE BY 1m FILL(linear);

-- 5. 每个 host 独立聚合，结果按 host 对齐
SELECT ts, host, AVG(val)
FROM metrics WHERE ts >= now() - INTERVAL 1 HOUR
SAMPLE BY 1m BY host;

-- 6. 时间戳对齐到 10s 网格
SELECT ts, val FROM cpu SAMPLE BY 10s ALIGN TO SECOND;

-- 7. 带偏移的 5 分钟窗口（窗口从 :05 开始，不是 :00）
SELECT ts, AVG(val) FROM cpu SAMPLE BY 5m OFFSET 50s;

-- 8. 每个 host 的 P95
SELECT host, PERCENTILE(val, 0.95) AS p95
FROM metrics WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY host;

-- 9. 两表按最近时间戳对齐（ASOF，最常见）
SELECT t.ts, t.px, f.rate
FROM ticks t ASOF JOIN fx f ON f.ts <= t.ts;

-- 10. ASOF + 容差（汇率最多滞后 10 分钟）
SELECT t.ts, t.px, f.rate
FROM ticks t ASOF JOIN fx f ON f.ts <= t.ts
WHERE t.ts - f.ts <= INTERVAL 10 MINUTE;

-- 11. ASOF + 等值键（多源对齐）
SELECT r.ts, r.val, c.firmware
FROM readings r ASOF JOIN config c
ON r.host = c.host AND c.ts <= r.ts;

-- 12. 回测防前视（严格小于）
SELECT fill.ts, signal.side
FROM fills fill ASOF JOIN signals signal
ON signal.ts < fill.ts;

-- 13. 窗口函数：每行配 5 分钟前的自己
SELECT ts, val, LAG(val, 5) OVER (ORDER BY ts) AS val_5m_ago
FROM cpu WHERE ts >= now() - INTERVAL 1 HOUR;

-- 14. 滚动均值（100 行窗口）
SELECT ts, AVG(val) OVER (ORDER BY ts ROWS 100 PRECEDING) AS roll_avg
FROM cpu WHERE ts >= now() - INTERVAL 1 HOUR;

-- 15. 计数
SELECT COUNT(*) FROM ticks WHERE ts >= now() - INTERVAL 1 DAY;

-- 16. 唯一值数量
SELECT COUNT(DISTINCT sym) FROM ticks;

-- 17. 按小时分组计数
SELECT date_add_hour(CAST(ts AS TIMESTAMP), 0, 0) AS hr, COUNT(*)
FROM ticks WHERE ts >= now() - INTERVAL 24 HOUR
SAMPLE BY 1h;

-- 18. JSON 字段取值
SELECT ts, payload->>'region' AS region FROM events
WHERE ts >= now() - INTERVAL 1 HOUR;

-- 19. 表里有哪些列
SELECT * FROM columns WHERE table_name = 'metrics';

-- 20. 分区清单
SELECT * FROM partitions WHERE table_id =
  (SELECT id FROM tables WHERE name = 'metrics');
```

## ASOF JOIN 方向速记

| 写法 | 语义 | 用途 |
|---|---|---|
| `f.ts <= t.ts` | 每个 t 配**之前**最近的 f | 最常见：历史价、历史配置 |
| `f.ts >= t.ts` | 每个 t 配**之后**最近的 f | 少用；回测里用这就是前视偏差 |
| `f.ts < t.ts` | 严格之前（不含同时刻） | 信号与成交对齐 |
| `+ INTERVAL x` | 容差窗口 | 排除过陈的对齐 |

## SYMBOL 调优三档

| 基数估计 | 建表写法 | 说明 |
|---|---|---|
| < 64 | `SYMBOL INDEX 64`（默认） | 字典内嵌，最快 |
| 64 – 1 万 | `SYMBOL INDEX 256` 或 `512` | 按实际基数上调一档 |
| > 1 万 | 改用 `VARCHAR` | 不建符号字典，避免内存爆 |

判断基数：`SELECT COUNT(DISTINCT col) FROM tbl` 跑一次，再定 INDEX 档位。

## 分区与保留

```sql
-- 建表时指定（ILP 自动建表不带分区，需要提前 DDL）
CREATE TABLE cpu (...) PARTITION BY DAY;
CREATE TABLE ticks (...) PARTITION BY HOUR;
CREATE TABLE logs (...) PARTITION BY MONTH;

-- 删除一个分区（按日期字符串，格式因分区粒度而异）
SELECT DROP_PARTITION('cpu', '2026_01_15');   -- DAY 粒度
SELECT DROP_PARTITION('ticks', '2026_01');     -- MONTH 粒度

-- 查分区占用
SELECT * FROM partitions WHERE table_id = (SELECT id FROM tables WHERE name='cpu');
```

生产上配 cron 每天删 90 天前的分区，磁盘占用就锁在一个常数。

## 常见报错与查法

| 报错 | 原因 | 查法 |
|---|---|---|
| `column not found` | 列名拼错，或建表时字段类型没写 | `SELECT * FROM columns WHERE table_name='...'` |
| `ASOF JOIN requires TIMESTAMP` | 对齐列不是 TIMESTAMP 类型 | 检查两表的 ts 列都是 TIMESTAMP |
| 结果比预期多 N 倍 | `SAMPLE BY` 没加 `WHERE`，扫了全表 | 时间条件永远加 |
| ILP 写入 400 | 字段类型和已有表冲突（DOUBLE 列写进来的是字符串） | 先 `DESCRIBE` 看列类型，ILP 值要匹配 |
| 查询突然变慢 | 时间条件被优化器裁剪失效（跨分区） | 确认 `WHERE ts >= now() - INTERVAL ...` 写法 |

## 一页外链接

- `SAMPLE BY` 全模式详解 → SAMPLE BY 完全指南
- `ASOF JOIN` 三个场景 → ASOF JOIN 指南
- ILP 协议逐字段说明 → ILP 摄入指南
- 建表后怎么持久化和扩容 → 安装指南

---

*本速查表基于 QuestDB `latest` 实测。`DROP_PARTITION` 日期格式和 `INDEX N` 上限随版本演进，上线前用 `SELECT * FROM functions` 核对你在用的版本支持的写法。QuestDB 为其权利人商标，本站为独立技术社区，非官方。*
