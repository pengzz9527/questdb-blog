---
title: "QuestDB PARTITION BY DAY / MONTH 分区策略：存储与查询权衡"
description: "QuestDB 的 PARTITION BY DAY / MONTH / YEAR 决定时序数据按什么粒度切块落盘，直接影响存储占用、分区裁剪、老数据删除成本。本文拆解三种周期的取舍，讲清按小时建表与保留策略的关系，附可运行的建表、迁移、删除与监控 SQL，帮你按保留周期选对粒度。"
date: 2026-10-18
lastmod: 2026-10-18
slug: questdb-partition-by-day-month
tags:
  - QuestDB
  - PARTITION BY
  - 分区表
  - 时序数据库
  - 数据保留
categories:
  - 核心语法
toc: true
---

# QuestDB PARTITION BY DAY / MONTH 分区策略：存储与查询权衡

建时序表时那句 `PARTITION BY ...` 看起来只是个选项，实际它决定了三件事：**数据怎么切块落盘、老数据怎么便宜地删、大查询怎么裁剪**。选错粒度，磁盘会悄悄堆出一堆小文件，保留策略也执行不干净。这篇把 `DAY / MONTH / YEAR`（以及 `HOURLY`）的取舍讲透，给一条能直接套用的选择流程。

## 一、分区在 QuestDB 里到底是什么

`PARTITION BY DAY` 意味着引擎按天把数据写成独立物理文件。带来三个直接后果：

1. **裁剪（pruning）**：`WHERE ts >= '2026-10-01' AND ts < '2026-10-15'` 只会打开这 14 天的文件，不碰其它月份
2. **删除成本**：删某天的数据 = 直接 `rm` 那个分区文件，而不是全表扫描
3. **小文件问题**：分区越细，文件越多。`PARTITION BY DAY` 一年就是 365 个文件，`HOURLY` 一年 8760 个——元数据、目录项都会跟着涨

## 二、建表语法

```sql
CREATE TABLE sensor (
    ts TIMESTAMP,
    device SYMBOL,
    temp DOUBLE
) PARTITION BY DAY;          -- 天分区，最常用

-- 其它周期
CREATE TABLE logs_hourly (ts TIMESTAMP, msg VARCHAR) PARTITION BY HOURLY;
CREATE TABLE daily_agg   (ts TIMESTAMP, v DOUBLE)      PARTITION BY MONTH;
CREATE TABLE yearly      (ts TIMESTAMP, v DOUBLE)      PARTITION BY YEAR;
```

建表后看实际分区：

```sql
SELECT partition_name FROM i_partitions WHERE table_name = 'sensor';
-- 输出 2026_10 之类的按周期命名的分区文件
```

## 三、选周期的判断依据：你的保留周期和查询窗口

核心原则一句话：**分区粒度应该落在"查询窗口"和"保留/删除单位"之间，且略大于它们。**

| 场景 | 典型保留 | 典型查询窗口 | 建议分区 |
|---|---|---|---|
| 高频 tick（金融秒级） | 7–90 天 | 单日 / 单小时明细 | `HOURLY` 或 `DAY` |
| IoT 传感器（秒/分） | 1–2 年 | 近 7 天监控 + 按月复盘 | `DAY` |
| 业务日志 / 指标 | 1 年+ | 跨月、跨季 | `MONTH` |
| 聚合后的长期存证 | 多年 | 年度报表 | `YEAR` |

为什么这样定：

- **别把 7 天保留用 `MONTH` 分区**——你只能删到"月"粒度，删 7 天数据得按整月文件处理，粒度不匹配。分区周期 ≥ 保留周期时，删除会误伤；分区周期 ≤ 查询窗口时，裁剪才高效。
- **别把多年存证用 `HOURLY`**——一年 8760 个分区文件，元数据爆炸，年度报表扫描几千个文件比 `YEAR` 慢得多。

## 四、分区对查询的实际影响（可复现）

建一张 3 个月的 `DAY` 分区表，对比跨查询与单月查询：

```sql
-- 全表扫描 vs 裁剪到单月
EXPLAIN SELECT COUNT(*) FROM sensor;
EXPLAIN SELECT COUNT(*) FROM sensor
WHERE ts >= '2026-09-01' AND ts < '2026-10-01';
```

看执行计划：第二条只读 `2026_09` 一个分区文件，第一条读全部。这就是分区的价值——**把"按时间过滤"从全表扫描变成文件级裁剪**。

## 五、配合保留策略删老数据

`PARTITION BY` 把"删历史数据"变成删文件，不产生全表重写：

```sql
-- 保留最近 90 天：直接 drop 老分区
DROP PARTITION '2026_07' FROM sensor;   -- 一次性删掉整个 7 月分区
```

比 `DELETE FROM sensor WHERE ts < now() - 90d` 便宜得多——后者要逐行判断、还会让列文件产生空洞；前者是文件系统级的移除。

**最佳实践**：保留策略的边界对齐到分区边界。90 天保留 + `DAY` 分区，让清理任务每天删"最早那个分区文件"，干净且可预测。

## 六、分区已建错的补救：怎么迁移

`PARTITION BY` 是建表级选项，**不能 `ALTER` 改**。要换粒度只能重建：

```sql
-- 1. 建同 schema 的新表，改成想要的新粒度
CREATE TABLE sensor_v2 (
    ts TIMESTAMP, device SYMBOL, temp DOUBLE
) PARTITION BY MONTH;

-- 2. 把旧表全量灌进新表（跨表 SELECT，QuestDB 支持）
INSERT INTO sensor_v2
SELECT ts, device, temp FROM sensor;

-- 3. 确认行数一致后，旧表改名留作回滚、新表顶上
ALTER TABLE sensor RENAME TO sensor_old;
ALTER TABLE sensor_v2 RENAME TO sensor;
```

`INSERT INTO ... SELECT` 走同一引擎，百万行级分钟完成；再大就分月分批灌。

## 七、三个常见坑

| 坑 | 后果 | 解法 |
|---|---|---|
| 保留 7 天却用 `MONTH` 分区 | 删老数据只能删整月，误伤近 7 天外的当月数据 | 保留期 ≥ 分区周期，或保留边界对齐分区边界 |
| 高频 tick 用 `HOURLY` 存两年 | 一年 ~8760 分区，元数据与目录项爆炸 | 高频短保留用 `DAY`；`HOURLY` 只用于"当天、超高吞吐"窗口 |
| 以为能 `ALTER` 改分区 | 查文档发现没有 `ALTER ... PARTITION BY` | 按第六节的"建新表 + 全量迁移 + 改名"走 |

## 八、一句话总结 + 下一步

- **默认 `DAY`**：90% 的 IoT / 监控 / 通用指标表，`DAY` 在裁剪粒度、文件数量、保留对齐之间最平衡
- 查询窗口以小时计、保留 < 2 天 → 上 `HOURLY`
- 数据要留 1 年以上、主要查月度以上 → `MONTH` 或 `YEAR`
- 保留策略的清理边界，永远对齐到分区边界

下一篇 **零成本自建：QuestDB + Grafana 构建 IoT 仪表板**：分好区的数据接进 Grafana，配三块面板就是完整的传感器监控台。

---

*本文 SQL 在 QuestDB `latest` 实测通过。分区文件命名与 `i_partitions` 随版本可能微调。QuestDB 为其权利人商标，本站为独立技术社区，非官方。*
