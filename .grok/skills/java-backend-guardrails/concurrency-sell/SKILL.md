---
name: concurrency-sell
description: Use when deducting stock, quotas, tickets, 名额, or commission-related remaining amounts under concurrent requests; oversell, 超卖, 扣库存, 核销占名额. Do not use for ordinary CRUD that does not mutate a remaining counter.
---

# 并发超卖 / 扣库存

## When to Invoke

扣库存、扣名额、余票、卡券可核销次数、分佣池剩余、秒杀预扣。

## When NOT

不改「剩余数量」的读写；纯展示库存（可缓存，不在这里做扣减）。

## 风险（面试考点）

`stock = stock - 1` 无条件更新 → 超卖。读库存再写回 → 丢失更新。只加 Redis 锁但 DB 无条件更新 → 锁失效仍超卖。面试常把「分布式锁」当成超卖银弹，这是错的。

## 适用场景

- 下单扣 SKU 库存
- 核销占用「可核销次数 / 场次名额」
- 分佣从可分配余额划出

## 方案选型（轻量优先）

| 优先级 | 方案 | 用在 | 代价 |
|--------|------|------|------|
| 1 | `UPDATE ... SET stock=stock-n WHERE stock>=n` | 中低并发，默认 | 一行热点时吞吐有上限 |
| 2 | 唯一键防同一用户重复下单/核销 | 一人一单、一券一核 | 要有业务唯一键 |
| 3 | Redis Lua 预扣 + MQ 异步落单 | 高 QPS 秒杀 | 缓存与 DB 要对账 |
| 4 | 队列化（单 SKU 单消费者） | 热点 SKU | 延迟上升 |

默认：条件更新 + 唯一键。不要用 Redisson 锁当主防超卖。锁可以串行化，但库存正确性必须写在 `WHERE stock>=n` 或 Lua 原子扣减里。

需要乐观锁版本号时加载 `optimistic-lock`。不要把「锁」和「库存条件更新」混成一件事。

## 默认方案

```sql
UPDATE sku_stock
SET stock = stock - #{n}, updated_at = NOW()
WHERE sku_id = #{skuId} AND stock >= #{n};
-- 影响行数 = 0 → 库存不足，不要再扣
```

MyBatis-Plus：

```java
int rows = stockMapper.update(null, Wrappers.<SkuStock>lambdaUpdate()
    .setSql("stock = stock - " + n)
    .eq(SkuStock::getSkuId, skuId)
    .ge(SkuStock::getStock, n));
if (rows == 0) {
    throw new BizException("库存不足");
}
```

一人一单 / 一券一核：

```sql
UNIQUE KEY uk_order_user_sku (user_id, sku_id, activity_id)
UNIQUE KEY uk_verify_voucher (voucher_id)
```

高 QPS 预扣（升级条件：单 SKU 条件更新扛不住）：

```lua
-- KEYS[1]=stock key, KEYS[2]=user bought key, ARGV[1]=n
if redis.call('EXISTS', KEYS[2]) == 1 then return -1 end
local left = tonumber(redis.call('GET', KEYS[1]) or '0')
if left < tonumber(ARGV[1]) then return 0 end
redis.call('DECRBY', KEYS[1], ARGV[1])
redis.call('SET', KEYS[2], 1, 'EX', 86400)
return 1
```

预扣成功 → 发 RocketMQ → 消费者用「条件更新 + 唯一键」落 DB。定时对账 Redis 与 DB。

## 反例

错误：

```java
int stock = mapper.selectById(id).getStock();
if (stock >= n) {
    entity.setStock(stock - n);
    mapper.updateById(entity);
}
```

并发两个请求都读到 1，都写成 0 以外的错值或都扣成功。

错误：`RLock.lock(); stock--; unlock();` 但 SQL 仍是无条件 `setStock`。
正确：锁不能替代 `WHERE stock>=n`。多数情况去掉锁，只留条件更新。

错误：Redis `DECR` 成负数还继续下单。
正确：Lua 里判断后再减，或 `DECR` 后发现 <0 再 `INCR` 回滚并失败。

## 验证

- 库存 1，200 并发下单，成交恰好 1，其余明确失败。
- 同一券并发核销两次，只成功一次。
- 预扣路径：Redis 扣了但 DB 失败，有回补或对账任务。

## 评审清单

- [ ] 扣减 SQL 带 `stock >= n`（或等价条件）
- [ ] 重复下单/核销有唯一键
- [ ] 没有「先读后写无版本号」
- [ ] 未把分布式锁当作唯一防超卖手段
---
