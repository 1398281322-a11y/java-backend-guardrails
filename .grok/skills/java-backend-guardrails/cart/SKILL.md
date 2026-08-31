---
name: cart
description: Use when implementing 购物车, add to cart, 未登录购物车, 登录合并, cart checkout. Cart does not deduct stock. Do not use for order creation itself (backend-order-state / backend-inventory-occupy).
---

# 购物车

## When to Invoke

加购、改数量、勾选结算、登录合并游客车。电商面试常问「购物车放哪、未登录怎么办」。

## When NOT

结算创单、扣库存、用券 → 对应订单/库存/优惠 skill。购物车 **不扣库存**。

## 风险（面试考点）

- 车里价格/库存会过期：结算必须重查 SKU 现价、可售、上下架。  
- 未登录 Cookie 明文可改数量和价格。  
- 登录合并把游客车覆盖会员车，或合并后超库存。  
- 车放 MySQL 行爆炸；全放 Cookie 丢数据、有大小限制。

## 方案选型（轻量优先）

| 用户 | 默认存储 |
|------|----------|
| 未登录 | Redis `cart:guest:{token}` Hash skuId→qty（TTL 7–30 天） |
| 已登录 | Redis `cart:u:{userId}`，必要时异步落库防丢 |
| 结算 | 读车 → 服务端拉最新价库存 → 生成确认单，不信任车里的 price |

加购只校验「可售>0 且不超过限购」，不预占。预占发生在提交订单。

## 默认方案

```java
// Hash: skuId -> qty
BoundHashOperations<String, String, String> cart = redis.opsForHash()
    .bind(login ? "cart:u:" + uid : "cart:g:" + guestId);
cart.increment(skuId, qty);
```

登录合并：游客 Hash 逐条 `increment` 到用户 Hash，删游客键。冲突数量封顶为限购或可售。

结算页：

```text
for sku in selected:
  load sku 现价、可售、状态
  if 下架 or 可售 < qty → 标失效，不进入创单
应付 = 服务端算（promo-coupon）
```

## 反例

错误：购物车表存 `price`，下单直接用。
正确：下单再读商品中心。

错误：加购就 `stock-1`。
正确：加购不占库存。

错误：Cookie 存整个商品 JSON 含价格。
正确：只存 skuId+qty。

## 验证

- 改 Redis 里的伪造价格，结算金额仍是服务端价。
- 登录后游客车商品出现在会员车，不丢不双倍超限购。
- 可售 1、车里 5，结算提示不足，不能创单。

## 评审清单

- [ ] 车内无可信价格
- [ ] 加购不扣库存
- [ ] 登录合并策略写清
---
