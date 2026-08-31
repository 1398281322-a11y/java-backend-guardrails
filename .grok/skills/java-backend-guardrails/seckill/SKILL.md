---
name: seckill
description: Use when designing 秒杀, 抢购, flash sale, 限时抢, 10万 QPS 活动. Do not use for ordinary product listing. Prefer layers of filtering; do not put all traffic on DB or distributed locks.
---

# 秒杀 / 抢购

## When to Invoke

限时、库存少、瞬时 QPS 远大于日常下单。文旅抢票、酒店今夜特价、券秒杀同样适用。

## When NOT

日常加购/下单 → `inventory-occupy` + `concurrency-sell`。不要把每个商品详情当秒杀。

## 风险（面试考点）

秒杀不是「Redis 锁 + 下单」。核心是 **把无效流量拦在数据库之外**，有效请求才扣库存。只靠 DB 行锁会打挂；只靠锁不带 `stock>=n` 会超卖。面试要能画出分层，而不是堆中间件名词。

## 分层（从外到内，默认全要，由外到内变严）

| 层 | 做什么 | 拦掉什么 |
|----|--------|----------|
| 1 页面 | 静态化 + CDN；开始前按钮置灰 | 刷新静态页 |
| 2 接入 | 网关按用户/IP/SKU 限流 | 刷子、超卖前的洪峰 |
| 3 资格 | 登录、活动时间、一人一单、验证码 | 不合格请求 |
| 4 预扣 | Redis Lua 原子扣 + 已买集合 | 无库存、重复抢 |
| 5 削峰 | 预扣成功才发 MQ 异步创单 | DB 写放大 |
| 6 落库 | `WHERE stock>=n` + 唯一键 | 缓存与 DB 最终一致 |

预扣失败直接售罄。禁止预扣失败还去锁 DB。

扣减细节读 `concurrency-sell`，热点拆段读 `hot-account`，关单读 `delay-job`。本文件不重复那些 SQL。本次最多再加 1–2 个，不要把上面三个全读进来。

## 默认方案

活动前把库存写入 Redis，按 SKU 分片（见 `hot-account`）。

```lua
-- KEYS[1] stock  KEYS[2] boughtSet  ARGV[1] userId  ARGV[2] n
if redis.call('SISMEMBER', KEYS[2], ARGV[1]) == 1 then return -1 end
local s = tonumber(redis.call('GET', KEYS[1]) or '0')
if s < tonumber(ARGV[2]) then return 0 end
redis.call('DECRBY', KEYS[1], ARGV[2])
redis.call('SADD', KEYS[2], ARGV[1])
return 1
```

成功 → RocketMQ（keys=userId+sku+activity）→ 消费者本地事务创单 + DB 条件扣。重复消息靠唯一键。

独立线程池、独立 Redis、必要时独立库。不要和商详、后台报表混部。

## 反例

错误：请求一进来 `RLock` + `select stock` + `update`。
正确：Lua 预扣，DB 只接预扣成功的量级。

错误：`DECR` 成负数还创单。
正确：Lua 里判断再减。

错误：MQ 消费者扣库存不带 `stock>=n`，重试把库存打成负数。
正确：条件更新 + 影响行数。

## 验证

- 库存 100，5 万 QPS，成交 ≤100，DB QPS 远小于入口。
- 同一用户并发只成一单。
- Redis 预扣了但创单失败，有回补或对账。

## 评审清单

- [ ] 有分层拦截，不是一把锁打天下
- [ ] Lua 预扣 + DB 条件更新两道
- [ ] 一人一单唯一键
- [ ] 秒杀集群与普通交易隔离
---
