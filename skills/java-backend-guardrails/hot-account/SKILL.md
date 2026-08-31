---
name: hot-account
description: Use when a single DB row or Redis key becomes a hotspot (热点账户, 热点行, 爆款 SKU, 钱包余额行锁排队). Do not use as a substitute for ordinary stock WHERE stock>=n (backend-concurrency-sell) unless that row is the bottleneck.
---

# 热点账户 / 热点行

## When to Invoke

爆款 SKU 一行库存、商户钱包一行余额、全局计数器，InnoDB 行锁排队、RT 随并发线性变差。

## When NOT

普通多 SKU 扣减用条件更新即可。没测出单行瓶颈不要拆。缓存热点 → `cache-trap` / `redis-ops`。

## 风险（面试考点）

InnoDB 行锁在单行上串行。`UPDATE wallet SET bal=bal-n WHERE id=1 AND bal>=n` 正确但不快。分布式锁再包一层更慢。面试常让「热点账户」和「超卖」混在一起：超卖靠条件更新；热点靠 **拆行 / 排队 / 异步入账**。

## 方案选型（轻量优先）

| 优先级 | 方案 | 说明 |
|--------|------|------|
| 1 | 不拆，条件更新 | 并发不够高 |
| 2 | 分段库存 / 分片余额 | `stock_0..15`，下单哈希到一段 |
| 3 | 单账户单队列串行 | 钱包流水入 MQ，单消费者改余额 |
| 4 | 内存预扣 + 异步落库 | 秒杀；必须对账 |

分段后每段自己 `WHERE stock>=n`。卖完一段换段或汇总失败再试。

## 默认方案

```text
sku:{id}:seg:{0..15}     Redis 预扣或 DB 16 行
下单：seg = userId % 16
扣减只打那一行
查询库存：SUM(seg) 或单独汇总键（最终一致）
```

钱包：禁止所有订单同步打同一 `merchant_id` 余额行。改成「流水表插入（唯一键）+ 异步汇总」或按日分片余额。

## 反例

错误：热点行上加 Redisson 锁再 `select` 改 `update`。
正确：先条件更新；仍排队再拆行。

错误：拆 16 段但不原子扣，只读汇总再减。
正确：每段独立条件扣。

## 验证

- 单 SKU 压测，拆段后 TPS 明显高于单行锁。
- 各段之和不错卖。
- 钱包流水不丢、不双花（唯一流水号）。

## 评审清单

- [ ] 先确认真是单行瓶颈
- [ ] 拆分后每段仍条件更新
- [ ] 有汇总/对账
---
