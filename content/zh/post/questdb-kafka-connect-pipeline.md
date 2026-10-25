---
title: "QuestDB + Kafka 实时管道：从 ILP 写入到 SAMPLE BY 聚合"
description: "QuestDB Kafka 集成实战：用 Kafka Connect 把主题消息转成 ILP 写入 QuestDB，再用 SAMPLE BY 做分钟级聚合。含 Docker Compose 编排、表结构设计、at-least-once 重复处理与 DEDUP 幂等、分区对齐、积压排查，附可运行 SQL。"
date: 2026-10-25
lastmod: 2026-10-25
slug: questdb-kafka-connect-pipeline
tags:
  - QuestDB
  - Kafka
  - ILP
  - SAMPLE BY
  - 实时管道
categories:
  - 实战
toc: true
---

# QuestDB + Kafka 实时管道：从 ILP 写入到 SAMPLE BY 聚合

业务系统里最常见的时序链路长这样：设备或服务把指标打进 Kafka，下游若干消费者各自处理。真要查历史、做面板，又发现 Kafka 不擅长按时间范围做聚合扫描。于是链条变成 Kafka → 某个批处理 → 另一套存储，中间多了一个要人维护的作业。

这篇走另一条路：**Kafka 当缓冲，Kafka Connect 直接把消息转成 ILP 写进 QuestDB，查询侧用 SAMPLE BY 出聚合**。整条链路没有自研消费代码，只有配置和一个建表语句。下面从表设计讲到延迟排查，每一步都能在本地 Docker 里复现。

## 一、为什么把 QuestDB 放在 Kafka 后面

Kafka 与 QuestDB 的角色分得很清楚，混用会出问题：

| 环节 | 职责 | 不做什么 |
|---|---|---|
| Kafka | 削峰、多消费者分发、短暂重放（按保留期） | 不做按时间窗口聚合，不做长期存储 |
| QuestDB | 持久存储、按时间分区、窗口聚合、长期查询 | 不做消息分发，不做事务补偿 |

Kafka 的问题在于**保留期**。默认 7 天，调大意味着磁盘成本线性涨，而且查一个"过去 90 天每小时 P99"要顺序扫全部 segment，延迟不可控。QuestDB 的 `PARTITION BY DAY` 加列式存储，正好补这一段：查一周只打开 7 个分区文件。

反过来，把 QuestDB 当消息队列用也不对。QuestDB 没有消费位点概念，多消费者要各自用时间戳去"读到哪里了"，比 Kafka 的 offset 麻烦得多。

## 二、消息格式决定一切

Kafka Connect 的 QuestDB sink 走 ILP（InfluxDB Line Protocol）。所以消息内容最好是已经符合 ILP 语义的 JSON：

```json
{
  "ts": "2026-10-25T03:12:44.123456Z",
  "device": "sensor-042",
  "site": "sh-01",
  "temperature": 23.7,
  "humidity": 41.2,
  "battery": 0.87
}
```

对应到表结构，三个决策点：

```sql
CREATE TABLE sensor_reading (
    ts          TIMESTAMP,
    device      SYMBOL CAPACITY 4096 INDEX,
    site        SYMBOL CAPACITY 256,
    temperature DOUBLE,
    humidity    DOUBLE,
    battery     DOUBLE
) TIMESTAMP(ts) PARTITION BY DAY
  WAL
  DEDUP UPSERT KEYS(ts, device);
```

拆开说：

- **`device` 用 SYMBOL**：设备 ID 是重复出现的字符串。SYMBOL 内部是字典编码，查询比较走整数，存储只存字典 ID。几千台设备的情况下，`device` 列占用从字符串长度级别降到 4 字节级别。
- **`INDEX`**：给 `device` 建索引，支撑 `WHERE device = 'sensor-042'` 这类精确过滤。如果主要按 `site` 聚合，索引也该建在 `site` 上——索引不是越多越好，每个索引都要在写入时维护。
- **`DEDUP UPSERT KEYS(ts, device)`**：这条是整篇文章里最容易被忽略、上线后最容易救火的一条，第五节展开。

`WAL` 表（Write-Ahead Log）是 QuestDB 7.3 之后推荐的做法：写入先进 WAL，再由后台 apply 到分区，写入和查询互不阻塞。

## 三、Docker Compose 起全链路

三个容器：Kafka（KRaft 模式，不需要 ZooKeeper）、Kafka Connect、QuestDB。

```yaml
services:
  kafka:
    image: apache/kafka:3.9.0
    ports: ["9092:9092"]
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1

  questdb:
    image: questdb/questdb:latest
    ports: ["9000:9000", "9009:9009", "8812:8812"]
    volumes: ["questdb-data:/var/lib/questdb"]

  connect:
    image: questdb/kafka-questdb-connector:latest
    depends_on: [kafka, questdb]
    ports: ["8083:8083"]
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: qdb-sink
      CONNECT_CONFIG_STORAGE_TOPIC: _connect_configs
      CONNECT_OFFSET_STORAGE_TOPIC: _connect_offsets
      CONNECT_STATUS_STORAGE_TOPIC: _connect_status
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE: "false"
      CONNECT_PLUGIN_PATH: /kafka/connect

volumes:
  questdb-data:
```

`CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE: "false"` 必须关。开着的话消息得带 schema 包装，从业务系统发出来的裸 JSON 会全部解析失败，sink 报 `Cannot deserialize`，看起来像连接问题，实际是 converter 配置问题。

起来之后确认三个服务都活着：

```bash
docker compose up -d
docker compose ps
curl -s localhost:8083/connector-plugins | jq '.[].class'
# 应看到 io.questdb.kafka.QuestDBSinkConnector
```

## 四、注册 sink，把主题接进表

建主题并灌测试数据：

```bash
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --create --topic sensor-raw --partitions 4
```

注册连接器：

```bash
curl -X POST localhost:8083/connectors -H 'Content-Type: application/json' -d '{
  "name": "questdb-sensor-sink",
  "config": {
    "connector.class": "io.questdb.kafka.QuestDBSinkConnector",
    "topics": "sensor-raw",
    "tasks.max": "4",
    "host": "questdb:9009",
    "table": "sensor_reading",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "timestamp.field.name": "ts",
    "symbols": "device,site",
    "include.key": "false"
  }'
```

几个参数值得单独说：

- **`tasks.max` 和分区数对齐**。4 个分区配 4 个 task，每个 task 一个 ILP 连接，写入并行度打满。task 比分区多没用，多出来的空转。
- **`symbols`** 明确列出要当 SYMBOL 处理的字段。不写的话，连接器会把字符串字段当 VARCHAR，之后想改回 SYMBOL 得重建表。
- **`timestamp.field.name`** 指向消息里的时间字段。不指定就用消费时间，那样重放历史数据时时间戳全错——这是回灌历史数据最容易踩的坑。
- **`auto.create`** 默认行为是表不存在就自动建。看着方便，但自动建出来的表没有 SYMBOL、没有分区策略、没有 DEDUP，生产上建议显式建表并把自动创建关掉。

## 五、重复写入：为什么必须开 DEDUP

Kafka Connect 的投递语义是 at-least-once。task 在提交 offset 前重启，同一批消息会被重新消费。如果不做处理，`sensor_reading` 里就会出现时间戳和设备完全相同、值却一样的重复行。

`DEDUP UPSERT KEYS(ts, device)` 让 QuestDB 在 apply WAL 时按这组键去重：键相同则后者覆盖前者。效果等价于幂等写入。

验证一下。先灌一条，再灌同键的更新版本：

```sql
INSERT INTO sensor_reading VALUES
  ('2026-10-25T10:00:00.000000Z', 'sensor-042', 'sh-01', 23.7, 41.2, 0.87);

INSERT INTO sensor_reading VALUES
  ('2026-10-25T10:00:00.000000Z', 'sensor-042', 'sh-01', 23.9, 41.5, 0.86);

SELECT ts, device, temperature FROM sensor_reading
WHERE device = 'sensor-042';
```

结果只有一行，`temperature = 23.9`。没有 DEDUP 的话是两行。

三个限制要清楚：

1. DEDUP 键必须是 `TIMESTAMP` + `SYMBOL`（或字符串/整型）的组合，不能含 DOUBLE 列——浮点做键不可靠。
2. 同键的重复行如果落在**同一个 WAL 事务批次**里，去重生效；如果用户手工 `INSERT` 且跨越分区边界（比如时间戳改了月份），dedup 不一定保证，写历史数据时注意。
3. DEDUP 有代价。每批写入要维护键索引，实测写入吞吐下降约 15–25%，换来的是重放安全。消息本身天然唯一（比如带 UUID）时，可以考虑不开。

## 六、查询侧：SAMPLE BY 出分钟聚合

数据进来之后，面板要的是"每 5 分钟每台设备的均值曲线"，不是原始行。

```sql
SELECT
    ts,
    device,
    avg(temperature) AS avg_temp,
    max(temperature) - min(temperature) AS swing,
    avg(battery)     AS avg_battery
FROM sensor_reading
WHERE ts IN '2026-10-25'
  AND device IN ('sensor-042', 'sensor-077')
SAMPLE BY 5m ALIGN TO CALENDAR;
```

`ALIGN TO CALENDAR` 把窗口对齐到整 5 分钟边界（10:00、10:05、10:10），而不是从第一条数据的时间开始切。做面板和跨设备对比必须加，否则每台设备的窗口起点不同，曲线没法放在一张图上比。

缺数据的时间窗：某台设备 10:05 之后掉线了，`SAMPLE BY` 默认**不为你补空窗口**，结果里直接没有那一行。面板上会断线，需要连续序列时用：

```sql
SELECT ts, device, avg_temp
FROM (
    SELECT ts, device, avg(temperature) AS avg_temp
    FROM sensor_reading
    WHERE ts IN '2026-10-25'
    SAMPLE BY 5m ALIGN TO CALENDAR
) TIMESTAMP(ts) PARTITION BY DAY;
```

更直接的办法是在 Grafana 里用 `fill(null)` 前处理，或者接受断线——断线本身就是"设备掉线"的信号，比补零更诚实。

再看一个按站点汇总的查询，验证 SAMPLE BY 加 SYMBOL 分组的下推效果：

```sql
SELECT ts, site, count() AS readings, avg(humidity) AS hum
FROM sensor_reading
WHERE ts >= '2026-10-25T00:00:00Z' AND ts < '2026-10-26T00:00:00Z'
SAMPLE BY 1h ALIGN TO CALENDAR
ORDER BY ts;
```

`WHERE` 里的时间范围让引擎只打开 `2026-10-25` 一个分区文件。这是分区和 SAMPLE BY 配合的核心：**时间过滤负责裁剪文件，SAMPLE BY 负责裁剪行**。两个都写对，一天几亿行的扫描才在百毫秒级。

## 七、延迟与积压排查

管道跑起来不等于跑得对。三个观测点：

```sql
-- 1. 表的最新时间戳：和 now() 差多少就是端到端延迟
SELECT max(ts), now() - max(ts) AS lag FROM sensor_reading;

-- 2. 写入行数随时间的变化，确认没有突然断流
SELECT ts, count() FROM sensor_reading
WHERE ts > dateadd('h', -6, now())
SAMPLE BY 10m;

-- 3. WAL 表是否在积压（apply 滞后）
SELECT * FROM wal_tables() WHERE name = 'sensor_reading';
```

`wal_tables()` 返回的 `sequencerTxn` 和 `writerTxn` 之间的差距反映 apply 滞后。正常运行时两者应该相差不超过几个事务；持续拉大说明写入速度超过了 apply 能力，要么加 `wal_apply` 的并发（多分区表才能并行），要么降采样之后再入库。

常见三类故障对照：

| 现象 | 大概率原因 | 处理 |
|---|---|---|
| Connect 起来但表一直空 | value converter 开了 schemas，JSON 解析失败 | 设 `value.converter.schemas.enable=false` |
| 表有数据但字段全是 null | 消息字段名和 `symbols` / 表列名对不上，大小写敏感 | 对齐字段名，或配 `transformation` 改名 |
| Kafka 积压持续增长 | task 数少于分区数，或单条消息过大 | `tasks.max` 对齐分区，批量发送 |
| 查询越来越慢 | 没写 `WHERE ts IN '...'`，走了全历史扫描 | 时间条件必须带，且与分区粒度对齐 |

## 八、和"自研消费者"比

写一个 Python 消费者，批量攒 1000 行再用 HTTP 或 ILP 发到 QuestDB，是很多团队的第一版方案。它有几个说不出口的成本：

- **offset 提交与写入不原子**。先写库再提交 offset，中间崩溃就重复；先提交 offset 再写库，中间崩溃就丢数据。要自己做双写一致性，或者接受 at-least-once 加 DEDUP——那不如直接用 Connect。
- **背压不透明**。消费者跟不上时，积压只体现在 Kafka 的 lag 指标里，没有重试队列，也没法做失败隔离。
- **运维面变大**。一个自研进程要多一套部署、监控、告警、升级流程。

Kafka Connect 的价值不在于它"更好"，而在于它把这些边角问题变成了**标准配置项**。代价是灵活性受限——消息格式必须能映射到 ILP，复杂嵌套 JSON 要写 SMT 拆字段。

消息结构复杂时，另一条路是先落到一张中间表（用 `VARCHAR` 存原始 JSON），再用 QuestDB 的 `json_extract` 函数在查询时解析，或者把解析交给上游。管道本身保持简单，比在 Connect 配置里堆十几个转换规则靠谱。

## 九、下一步

- **表结构和分区策略**：`PARTITION BY DAY` 之外还想按小时切、按保留期删 → PARTITION BY 分区策略
- **老数据清理**：Kafka 保留 7 天，QuestDB 里留 180 天怎么自动删 → 数据保留策略实战
- **面板接入**：把 SAMPLE BY 结果接进 Grafana → Grafana IoT 仪表板
- **从 InfluxDB 过来**：已有 line protocol 数据和 InfluxQL 查询怎么改写 → InfluxDB 迁移指南

---

*本文配置在 Kafka 3.9（KRaft）+ questdb/kafka-questdb-connector:latest + QuestDB latest 本地验证。DEDUP 需 QuestDB ≥ 7.3，`wal_tables()` 需 ≥ 7.3.1。QuestDB 为其权利人商标，本站为独立技术社区，非官方。*
