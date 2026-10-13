---
title: "QuestDB 速查表"
layout: "page"
description: "QuestDB SQL 速查：SAMPLE BY / FIRST / LATEST / ASOF JOIN / symbol / PARTITION BY / 时区处理 / ILP 写入语法，一页汇总常用命令。"
date: 2026-10-13
---

# QuestDB 速查表（建设中）

完整速查表将包含：

- **基础查询**：SELECT / WHERE / ORDER BY 与标准 SQL 差异
- **时间处理**：`SAMPLE BY 1s`、`fill(previous)`、时区转换
- **窗口函数**：FIRST / LATEST / NTH_VALUE / PERCENTILE
- **特殊 JOIN**：`ASOF JOIN`（时序数据对齐）、`JOIN BY`
- **数据类型**：`SYMBOL` vs `VARCHAR` 的性能与使用场景
- **ILP 摄入**：行协议语法、LIN 编码、批量写入
- **分区策略**：`PARTITION BY DAY` / `MONTH` 对查询与存储的影响

> 速查表正在编写，预计 2 周内上线。
