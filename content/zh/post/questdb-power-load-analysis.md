---
title: "电力负荷曲线分析：QuestDB 记录用电数据与峰谷负荷计算"
description: "电力负荷曲线是最典型时序负载：定间隔采样、多表独立、跨表聚合、峰谷价窗口。本文用 QuestDB 记录 15 分钟计量数据，给负载曲线、同时/非同时峰值、负荷率、峰谷电量拆分四段核心 SQL，外加累计读数差分与复位检测、长期保留降采样两条运维路径，附表设计与 ILP 写入示例，全部可运行。"
date: 2026-11-15
lastmod: 2026-11-15
slug: questdb-power-load-analysis
tags:
  - QuestDB
  - 电力
  - 计量
  - 负荷曲线
  - SAMPLE BY
categories:
  - 实战
toc: true
---

# 电力负荷曲线分析：QuestDB 记录用电数据与峰谷负荷计算

电力负荷曲线是最典型的时间序列负载：定间隔采样、大量独立电表、跨表聚合、峰谷电价窗口。做需量计费或者电费分析的话，下面这些查询是天天要跑的。本文从数据模型讲到四类核心分析（负载曲线、峰值需求、负荷率、峰谷电量），全部是可运行 SQL。

## 一、数据模型：两种信号别混

计量网关通常发两类信号：

| 信号 | 单位 | 含义 | 用途 |
|---|---|---|---|
| `power_kw` | kW | 区间平均功率 | 负载曲线、峰值需求 |
| `energy_kwh` | kWh | 区间用电量（已差分） | 电量成本、峰谷拆分 |

如果你的表只发**累计读数**（单调递增的计数器），在网关侧先差分出区间用量；实在要把计数器存进库里，SQL 里差分也可以（见第六节）。

```sql
CREATE TABLE meter (
    ts TIMESTAMP,
    site SYMBOL,
    meter SYMBOL,
    power_kw DOUBLE,
    energy_kwh DOUBLE
) TIMESTAMP ts PARTITION BY MONTH;
```

ILP 写入（15 分钟间隔，一表一行）：

```
meter,site=HQ,meter=m01 power_kw=128.4,energy_kwh=31.6 1700000000000000000
meter,site=HQ,meter=m02 power_kw=96.2,energy_kwh=23.9  1700000000000000000
```

100 块表 × 15 分钟间隔一年 960 万行，对 QuestDB 是小表；几千块表也扛得住。

## 二、负载曲线：15 分钟粒度

```sql
SELECT
    ts,
    SUM(power_kw) AS power_kw
FROM meter
WHERE site = 'HQ'
  AND ts >= '2026-11-14'::timestamp
SAMPLE BY 15m;
```

一行是一个 15 分钟窗口，`power_kw` 是该窗口内 site 全部电表的功率总和——这就是负载曲线。想看日峰形，改成 `SAMPLE BY 1h` 就行。

## 三、峰值需求：两个口径

**非同时峰值**——每块表自己的最大区间功率：

```sql
SELECT site, meter, power_kw AS peak_kw, ts AS peak_ts
FROM meter
ORDER BY power_kw DESC
LIMIT 100;
```

需量计费就用这个口径：需量费 = 峰值 × 需量单价，所以要看到每块表的峰值和发生时刻。

**同时峰值**——所有表加总最大的那个窗口：

```sql
SELECT ts, SUM(power_kw) AS total_kw
FROM meter
WHERE ts >= '2026-11-01'::timestamp
SAMPLE BY 15m
ORDER BY total_kw DESC
LIMIT 10;
```

第一行就是当月同时峰值，`ts` 直接告诉你峰值发生在几点。

## 四、负荷率

负荷率 = 平均负载 / 峰值负载。接近 1 说明负载平稳、电力容量利用率高；过低说明峰值是短时尖峰，可能白交了高额需量费：

```sql
SELECT AVG(load) / MAX(load) AS load_factor
FROM (
    SELECT SUM(power_kw) AS load
    FROM meter
    WHERE site = 'HQ' AND ts >= '2026-11-01'::timestamp
    SAMPLE BY 15m
);
```

## 五、峰谷电量：CASE WHEN 分时段

峰谷电价看**当地时区**，库里存的是 UTC，用 `AT TIME ZONE` 转：

```sql
SELECT
    SUM(CASE WHEN EXTRACT(HOUR FROM ts AT TIME ZONE 'Asia/Shanghai')
             BETWEEN 8 AND 19 THEN energy_kwh ELSE 0 END) AS peak_energy_kwh,
    SUM(CASE WHEN EXTRACT(HOUR FROM ts AT TIME ZONE 'Asia/Shanghai')
             BETWEEN 8 AND 19 THEN 0 ELSE energy_kwh END) AS offpeak_energy_kwh
FROM meter
WHERE site = 'HQ'
  AND ts >= '2026-11-01'::timestamp;
```

峰谷分界按当地政策设（示例用 08:00–20:00）。拿到两类电量后分别乘峰、谷单价，就是电费。

## 六、累计读数版：LAG 差分 + 复位检测

库里存的是计数器，用 `LAG` 差分：

```sql
SELECT
    ts,
    meter,
    energy_kwh - LAG(energy_kwh) OVER (PARTITION BY meter ORDER BY ts) AS interval_kwh
FROM meter
WHERE site = 'HQ';
```

两个坑：每块表第一行没有前值，差分为 NULL，要过滤；**电表复位**让计数器归零，差分变负——加一个判定，负值记为复位、该差分不可信。

## 七、长期保留：原始数据留两年，降采样到 1 小时

15 分钟原始数据价值够两年，再往后按月删分区（写法见本站《数据保留策略》一文）：

```sql
ALTER TABLE meter DROP PARTITION LIST '2024-03';
```

超过两年的数据降采样到 1 小时粒度写进 `meter_1h`（`SUM(energy_kwh)`、`AVG(power_kw)`，时间戳为窗口起点），明细表旧分区删掉，长期存储成本降到约 1/12。

## 八、坑清单

- 区间用量和累计读数混存同一张表，电量分析直接错。
- 网关故障缺 15 分钟——SAMPLE BY 默认跳空窗，曲线有洞；要填用 `SAMPLE BY 15m FILL PREVIOUS`。
- 峰谷窗口按 UTC 算——必须 `AT TIME ZONE`，否则夏令时期间峰谷差错一个小时。
- `MAX(power_kw)` 是单点峰值不是窗口峰值——口径要对齐计费周期，用 15 分钟粒度。

## 九、下一步

- 负载曲线 + 同时峰值进 Grafana：接数据源和面板的完整流程见本站《零成本自建 IoT 仪表板》。
- 对 1 小时曲线做滚动 Z-score 抓负载尖峰：见本站异常检测一文。
- 电表数量上千的写入并发：见本站《ILP 高并发写入调优》。
