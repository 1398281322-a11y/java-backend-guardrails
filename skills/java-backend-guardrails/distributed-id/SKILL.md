---
name: distributed-id
description: Use when generating order numbers, primary keys, or globally unique IDs (雪花, ASSIGN_ID, segment). Do not use UUID as a clustered InnoDB primary key. Do not use for shard-key design (backend-sharding).
---

# 分布式 ID

## When to Invoke

订单号、主键、分片后的全局唯一 ID、券码。

## When NOT

单库自增已经够且永不拆库的内部表可以继续 AUTO_INCREMENT。分片键怎么选 → `sharding`。

## 风险（面试考点）

UUID 做 InnoDB 主键：无序，页分裂，插入变慢。单机 AUTO_INCREMENT 无法跨库。雪花 ID 依赖时钟，回拨会撞号。号段模式要防 Redis/DB 号段丢失导致重复。

## 适用场景

- 微服务多库的主键
- 订单号对外暴露（可含时间/分片信息）
- 分库分表后的全局唯一

## 方案选型（轻量优先）

| 优先级 | 方案 | 说明 |
|--------|------|------|
| 1 | MyBatis-Plus `IdType.ASSIGN_ID`（雪花） | 默认 |
| 2 | 号段（DB/Redis 一次取一段） | 要可控、趋势递增 |
| 3 | 数据库 AUTO_INCREMENT | 单库内部表 |
| 禁止 | UUID 主键 | 可当业务展示码，不当聚簇主键 |

对外订单号可以：时间 + 分片 + 序列，和内部主键分开。

## 默认方案

```java
@TableId(type = IdType.ASSIGN_ID)
private Long id;
```

时钟：NTP 约束；workerId 按实例稳定分配（配置/K8s ordinal），不要每次随机。回拨：拒绝启动或等待，不要默默发号。

号段：

```sql
UPDATE id_segment SET max_id = max_id + step, version = version + 1
WHERE biz = 'order' AND version = #{version};
-- 本地内存发号，用完再取
```

## 反例

错误：`UUID.randomUUID()` 当主键。
正确：雪花 Long，或 UUID 只做非主键列。

错误：多实例共用 `workerId=1`。
正确：workerId 唯一。

错误：用雪花当用户分片键（时间热点）。
正确：分片键用 `user_id`，ID 只保证唯一。

## 验证

- 多实例并发插入无主键冲突。
- 主键大致时间有序，插入不疯狂页分裂。
- 人为时钟回拨有保护。

## 评审清单

- [ ] 主键不是 UUID
- [ ] workerId 不冲突
- [ ] 订单号与主键是否需要分离已想过
- [ ] 没有用 ID 算法冒充分片设计
---
