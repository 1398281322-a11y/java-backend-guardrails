---
name: promo-coupon
description: Use when implementing 优惠券领取/核销, 超发, 满减叠加, 促销计算, 优惠金额以服务端为准. Do not trust frontend prices. For 秒杀券 also consider backend-seckill.
---

# 优惠券 / 促销

## When to Invoke

发券、领券、下单用券、满减、折扣、优惠叠加。电商面试高频：「怎么防优惠券超发」「价格谁说了算」。

## When NOT

支付金额对账 → `reconciliation`。券当库存秒杀 → 加读 `seckill`，本文件仍管超发。

## 风险（面试考点）

1. **超发**：`remain-1` 无条件，领 1 万张预算 1000。  
2. **重复领**：无 `(user_id, activity_id)` 唯一键。  
3. **前端改价**：提交 `payAmount=0.01`。必须服务端用商品现价 + 规则重算，比对差额。  
4. **叠加爆炸**：满减+券+会员折没有互斥，算成负数。  
5. **用券不落核销**：下单失败券已核，或支付失败券回退两次。

## 方案选型（轻量优先）

| 问题 | 默认 |
|------|------|
| 批次剩余 | `UPDATE batch SET remain=remain-1 WHERE remain>=1` |
| 每人限领 | `UNIQUE(user_id, batch_id)` 插入领取记录 |
| 下单占用券 | 券 `UNUSED→LOCKED` 带 `order_no`；支付 `LOCKED→USED`；关单 `LOCKED→UNUSED` |
| 价格 | 服务端计算引擎，客户端金额只展示 |
| 叠加 | 明确优先级：券互斥 / 满减与券是否可叠，写在规则表 |

不必一上来规则引擎 drools。if 能列清就先 if。规则 >5 且运营常改再引擎。

## 默认方案

```sql
UPDATE t_coupon_batch SET remain = remain - 1
WHERE id = #{batchId} AND remain >= 1 AND status = 1;

INSERT INTO t_coupon_user(user_id, batch_id, coupon_no, status)
VALUES (#{uid}, #{batchId}, #{no}, 'UNUSED');
-- uk_user_batch (user_id, batch_id)
```

下单：

```sql
UPDATE t_coupon_user SET status='LOCKED', order_no=#{orderNo}
WHERE coupon_no=#{no} AND user_id=#{uid} AND status='UNUSED';
```

应付金额 = 服务端算。`abs(clientPay - serverPay) > 0` 则拒绝（或允许运费等白名单差）。优惠分摊到 SKU 行，方便退款按行退。

## 反例

错误：相信前端的 `couponAmount`、`payAmount`。
正确：只用券 ID，重算。

错误：领券 `if (remain>0) remain--` 非原子。
正确：条件 UPDATE。

错误：关单把券改 UNUSED，MQ 重试改两次把已使用券冲回。
正确：只从 LOCKED 回 UNUSED，且 `order_no` 匹配。

## 验证

- 批次 100，并发领 1000，成功恰好 100。
- 同一用户并发领，只一张。
- 改前端金额下单失败。
- 关单只释放本单锁定的券。

## 评审清单

- [ ] 批次剩余条件扣
- [ ] 领取唯一键
- [ ] 金额服务端重算
- [ ] 券状态与订单状态同进退
---
