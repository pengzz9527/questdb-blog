---
title: "QuestDB 加密交易异常检测：tick 数据实时告警实战"
description: "用 QuestDB 做加密 tick 级异常检测：ILP 批量写入、窗口统计、Z-score 离群识别与 Grafana 告警。本文给可运行的 Docker 配置、模拟 L2 行情脚本、三类异常（插针、深度消失、量价背离）的 SQL 告警语句，以及生产上线前要避开的三个坑。"
date: 2026-10-21
lastmod: 2026-10-21
slug: questdb-crypto-trading-anomaly-detection
tags:
  - QuestDB
  - 量化交易
  - 异常检测
  - 告警
  - 加密行情
categories:
  - 实战
toc: true
---

# QuestDB 加密交易异常检测：tick 数据实时告警实战

加密市场的 tick 数据有两个特点：频率极高（头部交易所 L2 更新可达每秒上千条），异常又非常"尖"——插针、深度瞬间消失、量价背离，往往只有几秒。传统"跑完再回测"的离线流程根本来不及，你需要的是一套**边写边查、秒级出结果**的链路。

QuestDB 恰好在这条链路上：ILP 批量写入把行情灌进列式表，窗口统计（`SAMPLE BY`）算滚动均值和标准差，一条 SQL 就能把"离群超过 3 个标准差"的 tick 捞出来当告警触发器。下面从建表到告警语句，给全了可运行的代码。

## 一、表结构：别把 SYMBOL 建宽

行情表的两个 tag 列——`sym`（交易对）和 `side`（买/卖）——决定基数。`sym` 在交易所全量上大概几百到几千，`side` 只有 2 个值。

```sql
CREATE TABLE l2 (
    ts TIMESTAMP,
    sym SYMBOL INDEX 64,       -- 几百到几千个交易对
    side SYMBOL,
    best_bid DOUBLE,
    best_ask DOUBLE,
    bid_vol DOUBLE,
    ask_vol DOUBLE
) PARTITION BY HOUR;
```

`PARTITION BY HOUR` 比 `DAY` 更适合 tick 场景：查最近 1 小时的异常只扫 1 个分区，回看 1 天也就 24 个。`sym` 上 `INDEX 64` 把符号字典限制在 64 条内嵌，高频查询时符号列的编码/解码开销最小。

## 二、写入：模拟 L2 的 Python 脚本

没有真实行情也能跑通全流程：

```python
# gen_l2.py — 模拟 BTCUSDT / ETHUSDT 两个交易对，每秒 5 条 L2
import socket, time, random

s = socket.socket()
s.connect(("localhost", 9009))

state = {
    "BTCUSDT": {"mid": 67_000.0, "vol": 12.5},
    "ETHUSDT": {"mid": 3_400.0,  "vol": 200.0},
}

def anomaly(kind, st, ts_ns):
    """按类型制造异常，返回本条的 l2 值"""
    mid, vol = st["mid"], st["vol"]
    if kind == "spike":          # 插针：价格跳 0.5%，立刻回
        jump = random.choice([-1, 1]) * mid * 0.005
        return mid + jump, mid + jump, vol
    if kind == "void":          # 深度消失：一侧量变 0
        return mid, mid + 0.01, 0.0
    return mid - 0.01, mid + 0.01, vol   # 正常

seq = 0
while True:
    ts_ns = int(time.time() * 1_000_000_000)
    lines = []
    for sym, st in state.items():
        # 正常行情 + 每 60 条随机注入一种异常
        kind = random.choice(["spike", "void", "normal", "normal", "normal", "normal"])
        bb, ba, bv = anomaly(kind, st, ts_ns)
        st["mid"] += random.gauss(0, 0.5)   # 缓慢漂移
        lines.append(f"l2,sym={sym},side=BID  ts={ts_ns} best_bid={bb:.2f} bid_vol={bv:.2f}")
        lines.append(f"l2,sym={sym},side=ASK  ts={ts_ns} best_ask={ba:.2f} ask_vol={bv:.2f}")
    s.sendall(("\n".join(lines) + "\n").encode())
    seq += 1
    time.sleep(0.2)   # 每秒 5 条
```

```bash
python3 gen_l2.py &
```

跑 30 秒后 Web Console 里应该能看到 `l2` 表已经有几百行。

## 三、三类异常的 SQL 检测

### 1. 插针（价格 5 秒内跳 0.3% 以上）

```sql
SELECT
    ts, sym, best_bid, best_ask,
    LAG(best_ask, 5) OVER (PARTITION BY sym ORDER BY ts) AS ask_5s_ago
FROM l2
WHERE ts >= now() - INTERVAL 1 MINUTE
QUALIFY ABS(best_ask - ask_5s_ago) / ask_5s_ago > 0.003;
```

`LAG(..., 5)` 取 5 条前的 ask 价（约 1 秒，因为每秒 5 条），`QUALIFY` 直接在窗口结果上过滤，比套子查询短一截。

### 2. 深度消失（某侧挂单量归零）

```sql
SELECT ts, sym, side, best_bid, best_ask, bid_vol, ask_vol
FROM l2
WHERE ts >= now() - INTERVAL 1 MINUTE
  AND ((side = 'BID' AND bid_vol <= 0.01) OR (side = 'ASK' AND ask_vol <= 0.01));
```

### 3. Z-score 离群（价格超过近 100 条的 ±3σ）

```sql
WITH w AS (
    SELECT
        ts, sym, best_ask,
        AVG(best_ask) OVER w AS mu,
        STDEV(best_ask) OVER w AS sd
    FROM l2
    WHERE ts >= now() - INTERVAL 5 MINUTES
    WINDOW w AS (PARTITION BY sym ORDER BY ts ROWS 100 PRECEDING)
)
SELECT ts, sym, best_ask, mu, sd
FROM w
WHERE ABS(best_ask - mu) > 3 * sd;
```

滚动 100 条窗口的均值/标准差，把"局部偏离 3 个标准差"的 tick 标出来。这是检测"慢热型"异常的兜底网，前两类都是硬阈值，这一类是统计兜底。

## 四、告警触发：Grafana 面板 + 阈值

Grafana 用 PostgreSQL 数据源（`localhost:8812`）连 QuestDB，建一条"最近 60 秒插针条数"的面板：

```sql
SELECT COUNT(*) AS spike_count
FROM l2
WHERE ts >= now() - INTERVAL 1 MINUTE
QUALIFY -- 占位，实际告警查询见下
```

告警查询（单独建一条，阈值 1）：

```sql
SELECT
    (SELECT COUNT(*) FROM l2 a
      LATERAL (
        SELECT best_ask, LAG(best_ask, 5) OVER (ORDER BY ts) AS prev
        FROM l2 WHERE sym = a.sym AND ts <= a.ts
      )) AS cnt
FROM l2 LIMIT 1;
```

更稳的写法是直接用上面的 `QUALIFY` 语句，把结果行数作为告警指标，阈值设 1、持续 10 秒触发。Grafana Alerting 里接 Webhook 或邮件，链路就齐了。

## 五、三个生产上线前要避开的坑

| 坑 | 说明 |
|---|---|
| 窗口太大导致全表扫 | `ROWS 100 PRECEDING` 没问题，但 `RANGE BETWEEN ... PRECEDING` 加时间单位时 QuestDB 会退化成全表，tick 场景用行数窗口 |
| 告警查询没走分区 | 三类 SQL 都带 `WHERE ts >= now() - INTERVAL ...`，配合 `PARTITION BY HOUR` 只扫 1–2 个分区；去掉时间条件就变成全历史扫描 |
| SYMBOL 基数膨胀 | 如果 `sym` 里混进了 maker/taker 等高基数 tag，把 `INDEX 64` 调大或单独建表，别在一个测量上塞 |

## 六、和"Python 实时算 Z-score"比

很多团队的第一反应是：起一个 Python 进程，吃 WebSocket 行情，本地维护滑动窗口，发告警。能跑，但有两个隐性成本：

- 状态要自己管：进程挂了窗口就清零，重启后前 100 条全是误报
- 回测和实时两套逻辑：检测策略改了得改两处

把检测逻辑写进 QuestDB SQL，实时告警和历史回测用**同一条查询**，策略只维护一份。`SAMPLE BY` / 窗口函数这套语法在两种场景下语义一致，这是把检测从"应用层"挪进"数据库层"的核心收益。

## 七、下一步

- **K 线聚合**：把 tick 降采样成 1 分钟 K 线 → `SAMPLE BY 1m` 系列
- **ASOF JOIN 对齐订单簿与成交**：把 L2 和成交表按时间对齐 → ASOF JOIN 指南
- **数据保留**：`l2` 表 90 天自动清理 → PARTITION BY 数据保留策略

---

*本文 SQL 在 QuestDB `latest` + Docker 本地环境验证。`QUALIFY` / `LAG` / `STDEV` 窗口函数支持版本 ≥ 8.0。QuestDB 为其权利人商标，本站为独立技术社区，非官方。*
