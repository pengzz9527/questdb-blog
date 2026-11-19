---
title: "QuestDB 时序数据压缩与保留策略：PARTITION BY 自动删除 + 冷热分层"
description: "QuestDB 数据保留与压缩实战：PARTITION BY 按天/月分区如何自动清理过期数据，冷热分层怎么把旧数据归档到低成本存储。覆盖删除窗口、compaction、磁盘占用控制，含建表语法与 Docker 部署，附容量估算方法。"
date: 2026-11-04
lastmod: 2026-11-04
slug: questdb-timeseries-compression-retention
tags:
  - QuestDB
  - 数据保留
  - PARTITION BY
  - 时序压缩
  - 冷热分层
categories:
  - 入门
toc: true
---

# QuestDB 时序数据压缩与保留策略

时序数据有个躲不掉的问题：只进不出。一天几亿点，一年下来磁盘直接爆炸。传统做法是"存到一个大表里，偶尔手动删"，但删一个按时间组织的时序表又慢又容易误删。QuestDB 把这件事做成了建表时的一个子句——`PARTITION BY`——让过期数据按分区自动清理，配合列式存储压缩，磁盘占用可以压得很低。

这篇讲三件事：怎么按时间分区让删除变"删文件夹"，怎么估算一张表一年占多少盘，以及冷热分层怎么把老数据挪走。

## 一、为什么按天分区能省磁盘又能快删

QuestDB 是列式存储，同列的值连续排布，对压缩极友好。`PARTITION BY` 决定了数据按什么时间粒度切成分区文件（`.db` 目录下的子目录）：

- `DAY`：每个自然日一个分区，小表、高频查询用。
- `MONTH`：每月一个分区，长期数据、查询范围大时用。

关键收益是**删除变成删目录**。过期数据所在的分区整个丢掉即可，QuestDB 不需要逐行扫描删除，磁盘也真正释放（不像某些引擎删了行还要等 compaction 才回收空间）。

```sql
-- 按天分区，适合保留最近 30~90 天的高频明细
CREATE TABLE otel_trace (
    ts      TIMESTAMP,
    trace   SYMBOL CAPACITY 8192 INDEX,
    span    SYMBOL CAPACITY 256,
    dur_ms  DOUBLE,
    status  INT
) PARTITION BY DAY;

-- 按月分区，适合长期归档
CREATE TABLE device_daily (
    ts     TIMESTAMP,
    device SYMBOL CAPACITY 4096 INDEX,
    energy DOUBLE,
    peak   DOUBLE
) PARTITION BY MONTH;
```

## 二、分区粒度怎么选

粒度直接影响两件矛盾的事：分区文件数（越多越碎，compaction 越累）和单次删除/查询的扫描范围（分区越大越粗，删得越整）。经验规则：

| 场景 | 推荐粒度 | 理由 |
|---|---|---|
| 高频明细、保留 < 90 天 | `DAY` | 查询常按天定位，删一天就是一删 |
| 月度聚合、保留 > 1 年 | `MONTH` | 文件数可控，删一月干净 |
| 超高基数 + 长保留 | `MONTH` 或 `YEAR` | 避免小文件爆炸 |

拿不准就先用 `DAY` 起盘，观察 2~4 周分区文件数，如果目录数增长过猛再改 `MONTH`。QuestDB 建表后不能直接改分区子句，要改粒度得建新表、回灌、切换。

## 三、容量估算：一张表一年占多少盘

估磁盘先把"每行字节数"算出来，再乘以行数，再乘压缩比。粗算法：

1. 一行所有列的**逻辑大小**（按类型：`TIMESTAMP` 8B、`DOUBLE` 8B、`SYMBOL` 按去重后基数、`INT` 4B）。
2. 列式 + 字典编码后，实际落盘通常压缩到逻辑大小的 **1/3 ~ 1/5**。

举例：一行 `TIMESTAMP + SYMBOL(万级基数) + DOUBLE + INT` 逻辑约 32 字节，一年 30 亿行（每秒约 100 行 × 全年），逻辑 ~96 GB，压缩后约 20~30 GB。所以"保留一年"在 SSD 上并不是天价，前提是你用了列式压缩而不是把原始文本存着。

监控用 `SHOW CATALOG` / `SHOW CREATE TABLE` 看分区，配合 `du -sh data/<table>/*` 盯实际占用，比任何估算都准。

## 四、冷热分层：把老数据挪到便宜存储

QuestDB 本身没有内置"自动降冷"，但保留策略是热层，冷层靠 ETL 搬走：

- **热层**（QuestDB）：保留最近 N 天，`PARTITION BY DAY`，随时可查。
- **冷层**：用 `COPY` 或定时任务把超过 N 天的分区导出为 Parquet，丢进对象存储（S3/OSS）。

```sql
-- 把 30 天前的某月分区导出成 Parquet 归档
COPY (
    SELECT * FROM device_daily WHERE ts >= '2026-01-01' AND ts < '2026-02-01'
) TO 's3://archive/device_daily/2026-01.parquet' (FORMAT PARQUET, COMPRESSION ZSTD);
```

导出后再 `DELETE FROM device_daily WHERE ts < '2026-02-01'`，热层只留近 30 天。冷数据要查时，QuestDB 的 `SAMPLE BY` 也能读外部 Parquet（通过 import），不必把整份冷数据搬回来。

## 五、删除的真实语义与坑

- `DELETE` 在 QuestDB 上是**按分区惰性生效**：删一个整分区最干净，跨多行散删会生成 tombstone，等 compaction 才真正回收。所以"能整分区删就别散删"。
- 建表忘写 `PARTITION BY` 不是灾难，但删除会退化成行扫描，且无法享受"删目录"的快。
- 保留期要和业务 SLA 对齐：观测类数据（trace）通常 7~30 天就够，业务类（计量）要 90 天以上甚至一年。
- 别用应用层定时 `DELETE` 去对抗分区机制——让分区到点自然过期，配合冷层导出更稳。

## 六、下一步

下一篇讲 QuestDB 的异常检测实战——把"老数据归档后还能不能被查"和实时告警连起来，用滚动 Z-score 在保留期内发现尖峰。

## 参考

- [QuestDB 官方文档：CREATE TABLE](https://questdb.io/docs/reference/sql/create-table/)
- [QuestDB 官方文档：SHOW](https://questdb.io/docs/reference/sql/show/)

## 相关文章

- [QuestDB 数据保留策略：DROP PARTITION、TTL 与冷热分层](/zh/post/questdb-data-retention-lifecycle/)
- [QuestDB PARTITION BY DAY / MONTH 分区策略：存储与查询权衡](/zh/post/questdb-partition-by-day-month/)
