---
title: "从 InfluxDB 迁到 QuestDB：schema 映射、line protocol 与查询改写"
description: "InfluxDB 迁移 QuestDB 的完整路径：line protocol 如何直接复用、measurement/tag/field 与表/SYMBOL/列的映射、InfluxQL 的 GROUP BY time 怎么改成 SAMPLE BY、Flux 查询如何重写，以及精度、时区与删除语义差异，附校验 SQL。"
date: 2026-10-25
lastmod: 2026-10-25
slug: questdb-influxdb-migration
tags:
  - QuestDB
  - InfluxDB
  - 数据迁移
  - line protocol
  - SAMPLE BY
categories:
  - 实战
toc: true
---

# 从 InfluxDB 迁到 QuestDB：schema 映射、line protocol 与查询改写

InfluxDB 用户迁到 QuestDB，通常不是因为某一个功能不行，而是三件事一起压过来：**高基数下内存吃不消、跨月查询开始变慢、保留策略的成本失控**。

好消息是迁移的技术难度比想象中低——QuestDB 原生支持 InfluxDB line protocol，写入侧几乎不用改。真正的工作量在**查询改写**和**语义差异处理**上：`GROUP BY time()` 不是 `SAMPLE BY` 的等价物，`DELETE` 和 `DROP PARTITION` 说的是完全不同的事。这篇把三块拆开讲：schema、写入、查询，每块都给出改写前后的对照。

## 一、概念映射：先建立一张对照表

| InfluxDB | QuestDB | 说明 |
|---|---|---|
| database / bucket | 同一实例内的表集合 | QuestDB 用 `CREATE TABLE` 建表，没有 bucket 层 |
| measurement | table | 一一对应 |
| tag | `SYMBOL` 列 | 低基数字符串，可加 `INDEX` |
| field | `DOUBLE` / `LONG` / `VARCHAR` 列 | 数值、布尔、字符串 |
| timestamp | `TIMESTAMP` 列 | 必须显式声明 `TIMESTAMP(ts)` |
| retention policy | `PARTITION BY` + `DROP PARTITION` | 手工或调度删除，见保留策略文章 |
| continuous query | `SAMPLE BY` + `INSERT INTO ... SELECT` | 定时任务执行 |
| `GROUP BY time(5m)` | `SAMPLE BY 5m` | 语法不同，语义接近 |
| `fill(linear)` | 无内建等价物 | 在应用层或 Grafana 补 |

一个 InfluxDB 写入长这样：

```text
cpu,host=server01,region=us-west usage_idle=92.1,usage_user=5.3 1698307200000000000
```

对应 QuestDB 建表：

```sql
CREATE TABLE cpu (
    ts          TIMESTAMP,
    host        SYMBOL CAPACITY 4096 INDEX,
    region      SYMBOL CAPACITY 64,
    usage_idle  DOUBLE,
    usage_user  DOUBLE
) TIMESTAMP(ts) PARTITION BY DAY
  WAL
  DEDUP UPSERT KEYS(ts, host, region);
```

注意 `host` 和 `region` 都是 tag，所以都是 `SYMBOL`；`usage_idle`、`usage_user` 是 field，所以是 `DOUBLE`。QuestDB 不会从 line protocol 自动推断出理想 schema——它会按"字符串就是 VARCHAR"去建，所以在迁移时**显式建表**这一步不能省。

## 二、写入：line protocol 基本可以原样发

QuestDB 在 9009 端口直接接受 ILP，格式与 InfluxDB 1.x 的 line protocol 高度兼容：

```bash
echo "cpu,host=server01,region=us-west usage_idle=92.1,usage_user=5.3 1698307200000000000" \
  | nc -q 0 localhost 9009
```

在 Grafana 或 Telegraf 里改的只是 URL：

```toml
# telegraf.conf：从 InfluxDB 换到 QuestDB
[[outputs.influxdb]]
  urls = ["http://questdb:9009"]
  database = "qdb"
  skip_database_creation = true
```

三个必须知道的差异：

**1. 时间戳精度。** InfluxDB 默认纳秒精度，line protocol 尾部的整数是纳秒。QuestDB 的 `TIMESTAMP` 是微秒精度——纳秒部分会被截断（不是四舍五入）。同一微秒内来自同一 tag 组合的多条数据，在 `DEDUP` 键上会冲突，后写覆盖先写。高频 tick 数据（比如交易所撮合每毫秒上百条）要留意这一点，必要时把纳秒的一部分挪进一个额外字段做区分。

**2. 数据类型一旦定下不能改。** InfluxDB 里同名 field 在不同 measurement 间类型可以不同；QuestDB 是列式强类型，某一列建了 `DOUBLE`，灌入整数值没问题（自动提升），但灌入字符串会整批被拒。迁移前把所有 field 的取值类型扫一遍。

**3. 字段是新列的开销不同。** InfluxDB 里新 tag/field 是免费的；QuestDB 里灌一条带新字段的数据会触发 `ALTER TABLE ADD COLUMN`，第一次写入会有轻微延迟。如果上游字段集合不稳定（比如动态标签），建议先固定一个宽表 schema，或者把不固定部分塞进 `VARCHAR` 里存 JSON。

## 三、批量回灌历史数据

历史数据迁移不要走 HTTP 一条条写，用 line protocol 文件批量导入：

```bash
# 导出 InfluxDB 为 line protocol
influx export --bucket metrics --output influx_export.lp

# 灌入 QuestDB（按文件批量，走 9009）
curl -X POST "http://questdb:9009/write?db=qdb" \
  --data-binary @influx_export.lp
```

数据量大时用 `curl` 分段：

```bash
split -l 500000 influx_export.lp chunk_
for f in chunk_*; do
  curl -s -X POST "http://questdb:9009/write?db=qdb" --data-binary "@$f"
  echo "imported $f"
done
```

两点经验：

- **每批 50 万行左右**。太小则是连接开销占主导，太大则单次失败要重放整批。
- **先关 DEDUP 灌历史，灌完再打开**。历史数据通常已经去重，开着 DEDUP 回灌会让吞吐下降 15–25%（每个批次都要维护键索引）。改法：

```sql
ALTER TABLE cpu DEDUP DISABLE;
-- 回灌
ALTER TABLE cpu DEDUP ENABLE UPSERT KEYS(ts, host, region);
```

灌完之后校验行数：

```sql
SELECT count(), min(ts), max(ts) FROM cpu;
```

和 InfluxDB 侧的 `count()` 对比。数量对不上，先查时间戳精度截断造成的 `DEDUP` 合并，再查是否有类型不匹配导致整批被拒（这类错误在 HTTP 响应里，日志里能看到）。

## 四、查询改写：从 InfluxQL 到 QuestDB SQL

这是迁移里真正花时间的部分。逐类对照。

### 4.1 基础聚合 + 时间窗口

InfluxQL：

```sql
SELECT mean("usage_idle") AS "avg_idle"
FROM "cpu"
WHERE time >= '2026-10-18T00:00:00Z' AND time < '2026-10-19T00:00:00Z'
  AND "host" = 'server01'
GROUP BY time(5m), "region" fill(null)
```

QuestDB：

```sql
SELECT ts, region, avg(usage_idle) AS avg_idle
FROM cpu
WHERE ts >= '2026-10-18' AND ts < '2026-10-19'
  AND host = 'server01'
SAMPLE BY 5m ALIGN TO CALENDAR;
```

差异点：

- `GROUP BY time(...)` → `SAMPLE BY ...`，且时间戳列自动出现在结果里
- `fill(null)` 在 QuestDB 里**默认就是空窗口不出行**，不需要也不支持这个子句
- `ALIGN TO CALENDAR` 是必须加的。InfluxQL 的 `GROUP BY time()` 默认按时间边界对齐，QuestDB 的 `SAMPLE BY` 不加 `ALIGN TO CALENDAR` 时以第一条数据为起点，两组查询结果会错开

### 4.2 多聚合 + 降采样

InfluxQL 用多个 `SELECT` 语句或者 `percentile`：

```sql
SELECT max("usage_user"), min("usage_user"), percentile("usage_user", 95)
FROM "cpu" WHERE time > now() - 1h GROUP BY time(1m)
```

QuestDB：

```sql
SELECT ts,
       max(usage_user) AS p_max,
       min(usage_user) AS p_min,
       approx_percentile(usage_user, 0.95) AS p95
FROM cpu
WHERE ts > dateadd('h', -1, now())
SAMPLE BY 1m ALIGN TO CALENDAR;
```

`percentile` 改成 `approx_percentile`，第二个参数是 0–1 的比例而不是百分数。要精确分位数用 `percentile` 但成本高，监控场景 `approx_percentile` 足够。

### 4.3 按 tag 过滤与 LAST 查询

InfluxQL 的 `LAST()` 查最新值：

```sql
SELECT last("usage_idle") FROM "cpu" WHERE "host" = 'server01'
```

QuestDB 用 `ORDER BY ... LIMIT`：

```sql
SELECT ts, usage_idle FROM cpu
WHERE host = 'server01'
ORDER BY ts DESC LIMIT 1;
```

`host` 上有 `INDEX`，这个查询走索引定位再取最后一行，比全表扫快得多。**索引该建在哪一列，取决于面板里最常用的过滤条件**，这是迁移时要重新看一遍的事。

### 4.4 GROUP BY 多个 tag

InfluxQL：

```sql
SELECT mean("usage_user") FROM "cpu" GROUP BY "host", "region"
```

QuestDB：`GROUP BY` 直接列列名，但**如果同时要时间窗口，就用 `SAMPLE BY` + `GROUP BY` 组合**：

```sql
SELECT ts, host, region, avg(usage_user)
FROM cpu
WHERE ts IN '2026-10-18'
SAMPLE BY 1h ALIGN TO CALENDAR;
```

QuestDB 的 `SAMPLE BY` 会自动把表里其它 `SYMBOL` 列当作分组键（这个行为可以配置）。不想要这个行为时，把 `GROUP BY` 显式写出来更安全。

### 4.5 Flux 查询

Flux 没有语法兼容，只能按语义重写。一个典型 Flux：

```flux
from(bucket: "metrics")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "cpu" and r.host == "server01")
  |> aggregateWindow(every: 1h, fn: mean)
```

对应：

```sql
SELECT ts, avg(_value) AS mean
FROM cpu
WHERE host = 'server01' AND ts > dateadd('h', -24, now())
SAMPLE BY 1h ALIGN TO CALENDAR;
```

Flux 的 `aggregateWindow` 有 `createEmpty` 参数控制是否补空窗口；QuestDB 侧没有这个开关，需要补空就交给 Grafana 的 `fill()`。

## 五、三个容易踩的语义差异

| 差异 | InfluxDB 行为 | QuestDB 行为 | 迁移建议 |
|---|---|---|---|
| 时区 | 全部 UTC 存储 | `TIMESTAMP` 无时区，查询里写 `'2026-10-18'` 按 UTC 解释 | 明确统一 UTC，应用层做展示时区转换 |
| 删除 | `DELETE FROM` 按条件删 | 逐行标记再重写，开销大 | 分区表用 `DROP PARTITION` |
| 保留过期 | retention policy 自动过期 | 无内建 TTL（开源版），要外部调度 | 写定时脚本删分区 |
| 空值语义 | 缺失即为 null，`fill()` 补 | 缺失即无行 | 面板侧处理连续性 |
| 字段类型冲突 | 同名字段允许不同类型 | 列类型固定，冲突批次整批失败 | 迁移前扫描类型 |

**时区这条最隐蔽**。InfluxDB 里所有时间都是 UTC，而查询里的 `'2026-10-18T00:00:00Z'` 带 `Z`，语义明确。QuestDB 里 `'2026-10-18'` 这种不带时区的写法按 UTC 解释——两边碰巧一致，但如果原来有人写的是本地时间字符串（某些客户端会这么干），迁移后会整体偏移 8 小时。迁移校验里必须包含"抽查若干行原始时间戳与业务时间对照"。

**删除语义这条最容易在生产上出事**。InfluxDB 用户习惯 `DELETE WHERE time < ...` 直接删，迁到 QuestDB 后照抄，几亿行历史数据删起来没有上限。正确做法是第七章讲的分区删除。

## 六、校验迁移是否成功

三个维度的校验，缺一个都不算完成：

```sql
-- 1. 总量与时间范围
SELECT count() AS rows, min(ts) AS first_ts, max(ts) AS last_ts FROM cpu;

-- 2. 每个 tag 组合的行数分布（对照 InfluxDB 侧 group by）
SELECT host, region, count() AS c
FROM cpu GROUP BY host, region ORDER BY c DESC LIMIT 20;

-- 3. 抽样比对具体数值（选一个已知时间窗，两边查同一段）
SELECT ts, host, usage_idle FROM cpu
WHERE host = 'server01' AND ts IN '2026-10-18'
ORDER BY ts LIMIT 10;
```

第 3 条最关键。总量对得上不代表内容对得上——时间戳截断、字段错位、单位不一致，都会在总量层面看不出来。抽十几个点逐一核对，比看总量数字靠谱。

## 七、迁移后的必做三件事

**1. 建好保留策略。** InfluxDB 的 retention policy 是声明式的，QuestDB 要自己调度：

```sql
ALTER TABLE cpu DROP PARTITION LIST '2026-09';
```

配个 cron，或者参考数据保留策略那篇的脚本。

**2. 重新审视索引。** InfluxDB 的 series index 是自动的，QuestDB 的 `INDEX` 要手工给。上线后从慢查询日志里找出高频 `WHERE`，再决定建哪个索引。给所有 `SYMBOL` 加索引是常见的过度优化——每个索引都在写入时付出维护成本。

**3. 演练一次故障恢复。** QuestDB 的数据目录是 `var/lib/questdb`，备份方式就是停库 + 拷贝目录，或者分区级拷贝。InfluxDB 的 `influxd backup` 那套工具在 QuestDB 里没有对应物——存储架构不同，这点必须提前告诉运维。

## 八、下一步

- **line protocol 灌入的细节**：批量、并发、错误重试 → ILP 摄入完全指南
- **时间窗口聚合**：`SAMPLE BY` 的全部参数与陷阱 → SAMPLE BY 完全指南
- **保留与分区删除**：`DROP PARTITION`、降采样、冷热分层 → 数据保留策略
- **面板迁移**：把 InfluxQL 面板改到 QuestDB 数据源 → Grafana IoT 仪表板

---

*本文 SQL 在 QuestDB latest（Docker）验证，`approx_percentile` 需 QuestDB ≥ 6.1，`ALTER TABLE ... DEDUP` 需 ≥ 7.3。InfluxDB 侧语法以 1.8 与 2.x 的 InfluxQL 为准；Flux 需按语义手工重写。QuestDB 为其权利人商标，本站为独立技术社区，非官方。*
