---
name: table-design
description: Use when creating or changing MySQL tables, status fields, unique keys, big-table archive, or partitioning. Do not use for index details (backend-mysql-index) or sharding (backend-sharding).
---

# 表设计

## When to Invoke

建表、改表、加状态、加大表归档/分区。需求阶段就把唯一键和状态机定下来。

## When NOT

怎么建二级索引 → `mysql-index`。要不要拆库拆表 → `sharding`。本文件只决定表长什么样、何时归档/分区。

## 风险（面试考点）

没有唯一键导致并发双写；状态用字符串随意改导致乱跳；大表只靠「以后分库」而从不归档；把分区和分库分表混为一谈。分区仍在同一实例，分库分表才跨实例。

## 适用场景

- 订单、支付单、核销单、分佣流水
- 流水型大表（日志、回调原始报文）
- 需要幂等的业务表

## 方案选型（轻量优先）

| 问题 | 默认 | 升级（不要跳级） |
|------|------|------------------|
| 幂等 | UNIQUE 业务键 | 见 `idempotent` |
| 并发改同一行 | `version` 字段或条件更新 | 见 `optimistic-lock` / `concurrency-sell` |
| 表变大 | 索引 + 冷热分离/归档 | 分区（按月） |
| 分区仍不够 | 才考虑 `sharding` | — |

禁止：新建业务表就按用户分 256 表。先估算：有索引的 InnoDB 单表千万级仍常见。

## 默认方案

订单/核销表必备：

- 主键：雪花 ID（见 `distributed-id`），不要无序 UUID 当聚簇主键
- 业务唯一键：`out_trade_no+channel`、`voucher_id`
- 状态：tinyint/枚举，代码里枚举单向跳转
- `created_at` / `updated_at`
- 需要乐观锁才加 `version`
- 回调原始报文另表存储，避免撑爆主表行

```sql
CREATE TABLE t_order (
  id            BIGINT PRIMARY KEY,
  order_no      VARCHAR(32) NOT NULL,
  user_id       BIGINT NOT NULL,
  merchant_id   BIGINT NOT NULL,
  status        TINYINT NOT NULL,
  pay_amount    BIGINT NOT NULL,
  version       INT NOT NULL DEFAULT 0,
  created_at    DATETIME NOT NULL,
  updated_at    DATETIME NOT NULL,
  UNIQUE KEY uk_order_no (order_no),
  KEY idx_user_created (user_id, created_at),
  KEY idx_merchant_status (merchant_id, status)
);
```

大表：先按时间归档到历史表（应用双写查询：近期主表，远期历史表）。单机时间范围查询再考虑 `PARTITION BY RANGE (TO_DAYS(created_at))`。还不够再读 `sharding`。

## 反例

错误：状态只靠应用判断，表无唯一键。
正确：唯一键是最后一道闸。

错误：把索引策略和分片键写进这张建表稿当同一段「优化」。
正确：索引走 `mysql-index`，分片走 `sharding`。

错误：流水表无限增长，三年数据全在热库。
正确：归档任务 + 保留窗口。

## 验证

- 重复业务单号插入失败（唯一键）。
- 非法状态更新影响行数为 0。
- 归档后热表体量可控，历史可查。

## 评审清单

- [ ] 写路径有 UNIQUE
- [ ] 状态字段 + 合法迁移
- [ ] 未把分库分表当建表默认项
- [ ] 大表有归档/分区说法，且与分片分开
---
