---
title: "QuestDB 数据保留策略：DROP PARTITION、TTL 与冷热分层"
description: "QuestDB 数据保留怎么做才干净：用 DROP PARTITION 按分区物理删除、判断分区是否可删、用外部定时任务或 QuestDB 企业版 TTL 自动清理，再配合冷热分层把热数据留在本地、冷数据搬去对象存储。含可运行 SQL、分区删除的坑与磁盘回收验证，适合长期跑的时序表。"
date: 2026-10-25
lastmod: 2026-10-25
slug: questdb-data-retention-lifecycle
tags:
  - QuestDB
  - 数据保留
  - DROP PARTITION
  - 分区表
  - 冷热分层
categories:
  - 实战
toc: true
---

# QuestDB 数据保留策略：DROP PARTITION、TTL 与冷热分层

时序表的行数只增不减，这件事没有例外。一台设备每秒一条，一年 3150 万行；一千台设备一年 315 亿行。磁盘不会自己空出来，而"能删"和"删得干净、删得便宜"是两件不同的事。

QuestDB 的保留手段在结构上只有一个核心动作：**删分区**。其它所有做法——定时任务、TTL、冷热分层——最后都落到这一个操作上。理解它，就能把保留策略写对；不理解它，用 `DELETE` 一遍遍扫全表，几亿行删下来 CPU 打满，磁盘空间还不一定还给你。

## 一、为什么不能直接用 DELETE

先看一次错误的删除：

```sql
DELETE FROM sensor_reading WHERE ts < '2026-04-01';
```

这条语句在 QuestDB 里能执行，但在分区表上是**逐行标记再重写**的过程：找到匹配的行、写新版本、旧版本等待清理。三亿行删下来，无论耗时还是资源占用都不划算，而且用户往往在删完之后看磁盘占用没变——因为旧数据段还在等 GC（后台清理）。

正确姿势：

```sql
ALTER TABLE sensor_reading DROP PARTITION LIST '2026-03';
```

这是一次元数据操作：直接从表的分区目录里摘掉 `2026-03` 这个分区，**文件句柄关闭、目录项删除，空间立即归还文件系统**。耗时和分区大小无关，几毫秒到几百毫秒。

这是分区策略真正的价值所在。第一篇文章讲 `PARTITION BY` 时说的是原理，这里说后果：**你的保留粒度，物理上就被分区粒度锁死了**。`PARTITION BY DAY` 的表只能按天删，想保留 45 天做不到，只能删到"整 45 天"或"46 天"。`MONTH` 分区只能按月删。

## 二、先看有哪些分区

```sql
-- 单表看
SELECT partition_name, num_rows, disk_size
FROM partitions('sensor_reading')
ORDER BY partition_name;

-- 全库看
SELECT table_name, partition_name, num_rows
FROM table_partitions()
ORDER BY table_name, partition_name;
```

`partitions('table')` 是函数形式、按表查，`table_partitions()` 是系统视图、跨表查。两个都能拿到 `num_rows` 和 `disk_size`——后者是这一篇里做保留决策最关键的字段：不是看"有多少天分区"，而是看"哪个分区占了多少磁盘"。

一个分区里有 5 亿行，另一个只有 300 万行，说明数据分布不均，可能是某天压测灌入或者某台设备上报频率异常。光按时间删，会把两个都当天数据对待。

## 三、手工删一个分区，验证空间真的回来了

```bash
docker compose exec questdb df -h /var/lib/questdb
# /dev/...  20G  14G  5G  74% /var/lib/questdb
```

```sql
ALTER TABLE sensor_reading DROP PARTITION LIST '2026-03';

SELECT * FROM partitions('sensor_reading') WHERE partition_name = '2026-03';
-- 0 rows
```

```bash
docker compose exec questdb df -h /var/lib/questdb
# /dev/...  20G  8G  11G  42% /var/lib/questdb
```

14G 到 8G，6G 直接还给了文件系统。这一步值得每次都做一遍——凡是磁盘没下降的"删除"，都是没删干净。

## 四、把删除自动化：不用 TTL 也能做

开源版 QuestDB 没有内置 TTL 子句（企业版提供 `CREATE TABLE ... TTL 90 DAYS`，会由引擎自己调度分区删除）。开源版的做法是**外部调度**：一个定时脚本，算出该删哪些分区，逐个 `DROP PARTITION`。

脚本逻辑比想象中少：

```bash
#!/usr/bin/env bash
# prune_partitions.sh — 每天 03:00 跑一次
set -euo pipefail

QDB="http://localhost:9000/exec"
TABLE="sensor_reading"
RETAIN_DAYS=180

# 截止分区：今天往前推 180 天所在的分区名（DAY 分区格式 YYYY-MM-DD）
CUTOFF=$(date -u -d "-${RETAIN_DAYS} days" +%Y-%m-%d)

# 查出所有早于截止日的分区
mapfile -t PARTS < <(
  curl -sG "$QDB" \
    --data-urlencode "query=SELECT partition_name FROM partitions('${TABLE}') WHERE partition_name < '${CUTOFF}' ORDER BY partition_name" \
  | jq -r '.dataset[] | .[0]'
)

for p in "${PARTS[@]}"; do
  echo "dropping ${TABLE} partition ${p}"
  curl -sG "$QDB" \
    --data-urlencode "query=ALTER TABLE ${TABLE} DROP PARTITION LIST '${p}'" > /dev/null
done
```

三个要点：

1. **用 `<` 而不是 `=`**。脚本可能因为故障停几天，漏掉的分区下次会一起清掉——幂等性靠这个。
2. **不要删正在写的分区**。判断依据是分区名与当天比较，`partition_name < CUTOFF` 天然排除今天。
3. **先算后删**，把待删列表打到日志里再执行。删除不可逆，日志是唯一的回滚线索（回滚手段只有备份）。

挂到 cron：

```cron
0 3 * * * /opt/qdb/prune_partitions.sh >> /var/log/qdb-prune.log 2>&1
```

`DROP PARTITION` 在 WAL 表上由 sequencer 异步执行，不会阻塞写入。这一点比自己写 `DELETE` 强得多——批量删除最难的就是"删的时候还能写"。

## 五、保留策略和分区粒度必须对得上

这是最容易出错的地方，单独列一张表：

| 保留需求 | 分区粒度必须是 | 实际能达到的保留 | 说明 |
|---|---|---|---|
| 保留 7 天 | `DAY` 或 `HOUR` | 7–8 天 | 按天删，最多多留一天 |
| 保留 30 天 | `DAY` | 30–31 天 | 标准组合 |
| 保留 90 天 | `DAY` | 90–91 天 | 细粒度表推荐 |
| 保留 6 个月 | `DAY` 或 `MONTH` | 6 个月 ± 一个月 | `MONTH` 分区省文件，但删除粒度粗 |
| 保留 45 天 | `DAY` | 45–46 天 | 不能用 `MONTH`，只能按天删 |
| 保留 2 年 | `MONTH` | 24–25 个月 | 730 个日分区太多 |

如果已经建了错粒度的表，只能用 `ALTER TABLE ... SET PARTITION BY ...` 改分区粒度：

```sql
ALTER TABLE sensor_reading SET PARTITION BY MONTH;
```

注意这条语句本身不重写历史数据——已有分区保持原样，只有新写入的数据按新粒度切。想彻底调整得建新表、`INSERT INTO ... SELECT` 迁移、再改表名。所以建表前把保留期想清楚，比事后改表便宜一个数量级。

## 六、保留策略的另一半：降采样而不是删

180 天后删除是"丢数据"。多数业务真实的需求其实是"明细丢掉，趋势留下"。这就变成两段式策略：

- 0–30 天：完整明细（秒级原始数据）
- 30–180 天：5 分钟聚合
- 180 天以上：1 小时聚合（可长期保留）

QuestDB 里这件事用 `SAMPLE BY` 加 `INSERT INTO ... SELECT` 完成：

```sql
-- 日聚合表，长期保留
CREATE TABLE sensor_reading_1h (
    ts          TIMESTAMP,
    device      SYMBOL,
    avg_temp    DOUBLE,
    max_temp    DOUBLE,
    min_temp    DOUBLE,
    samples     LONG
) TIMESTAMP(ts) PARTITION BY MONTH
  DEDUP UPSERT KEYS(ts, device);

-- 把昨天尚未聚合的明细压成小时表（幂等，可重复执行）
INSERT INTO sensor_reading_1h
SELECT
    ts,
    device,
    avg(temperature),
    max(temperature),
    min(temperature),
    count()
FROM sensor_reading
WHERE ts IN '2026-10-24'
SAMPLE BY 1h ALIGN TO CALENDAR;
```

`DEDUP UPSERT KEYS(ts, device)` 让这段可以反复跑：重跑同一天不会产生重复聚合行，只是把值覆盖成同样的结果。定时任务不需要"只跑一次"的精确控制，失败重试就行。

指标对照，看聚合省了多少：

```sql
SELECT
    (SELECT count() FROM sensor_reading WHERE ts IN '2026-10-24')     AS raw_rows,
    (SELECT count() FROM sensor_reading_1h WHERE ts IN '2026-10-24')  AS hourly_rows;
-- raw_rows 远大于 hourly_rows，比例就是压缩收益
```

一小时聚合通常把行数压到原始的 1/3600 以下（秒级原始数据）。磁盘占用按同样比例缩，而"过去一年每小时峰值温度"这种问题依然能答。

## 七、冷热分层：热数据本地，冷数据出走

再往前一步，是让老分区离开本地磁盘。两种做法：

**做法一：导出后删除。** QuestDB 支持 `COPY` 把查询结果导成 Parquet：

```sql
COPY (
    SELECT * FROM sensor_reading WHERE ts IN '2026-02'
) TO '/var/lib/questdb/export/2026-02.parquet' WITH FORMAT PARQUET;
```

导出到对象存储（挂载或 `mc cp`、`aws s3 cp`），校验文件大小与行数，然后 `DROP PARTITION`。冷数据查询交给 DuckDB / Athena 之类的工具直接读 Parquet——这类"偶尔查一次历史"的需求不值得留在热库里。

**做法二：分区目录外挂。** QuestDB 的分区是普通目录。停表 → 把整个分区目录 `mv` 到冷存储挂载点 → 需要时 `mv` 回来。要停表是因为引擎持有打开的文件句柄，运行中直接移动会让分区读不到。这条路上线前务必在一份拷贝上演练，不做演练直接动生产的分区目录，是少数能把表搞坏的操作。

冷热分层的取舍很直接：

| | 全留热库 | 冷热分层 |
|---|---|---|
| 查询历史 | 快 | 慢（要挂载或走外部工具） |
| 磁盘成本 | 随时间线性增长 | 冷数据成本降到对象存储级别 |
| 运维复杂度 | 低 | 中（要管迁移任务和校验） |
| 恢复历史数据 | 无需 | 需要规程 |

一年以内的数据量还在几十 G 量级时，别急着做分层——先把手上的分区删除和降采样做对，收益已经足够。

## 八、三个上线前必须验证的点

| 检查项 | 怎么验 | 不通过的后果 |
|---|---|---|
| 删除后磁盘真的下降 | 删前删后各跑一次 `df -h` | 以为在省空间，实际在堆积待 GC 的段 |
| 脚本幂等 | 连跑两次，第二次应无输出 | 定时任务重试时误删或报错中断 |
| 不删当天的分区 | 在待删列表里 print 出今天的分区名 | 当天数据被删，面板断流 |
| 保留期与合同/审计要求一致 | 和业务方对一次，不只是问工程师 | 删掉了本该留档的数据 |

最后一条最容易被忽略。技术团队按"180 天够了"删数据，法务那边要求的是"交易相关数据留三年"。保留策略的输入不该只来自磁盘容量。

## 九、下一步

- **分区粒度怎么选**：`DAY` / `MONTH` / `HOURLY` 的取舍 → PARTITION BY 分区策略
- **聚合语法**：SAMPLE BY 的 `ALIGN TO CALENDAR`、缺窗口、多列聚合 → SAMPLE BY 完全指南
- **数据怎么进来**：Kafka 管道和 DEDUP 幂等写入 → Kafka 实时管道
- **面板上看到保留效果**：把分区数量和磁盘占用做成 Grafana 曲线 → Grafana IoT 仪表板

---

*本文 SQL 与脚本在 QuestDB latest（Docker）+ Linux 环境验证。`partitions()` 函数需 QuestDB ≥ 6.6，`COPY ... WITH FORMAT PARQUET` 需 ≥ 7.0，WAL 异步删分区需 ≥ 7.3。企业版 TTL 语法与开源版不同，以官方文档为准。QuestDB 为其权利人商标，本站为独立技术社区，非官方。*
