---
name: refund-aftersale
description: Use when implementing 售后, 仅退款, 退货退款, 退库存, 退优惠券, reverse order flow. Do not reuse 正向下单代码硬改状态. Payment callback idempotency is backend-idempotent.
---

# 售后 / 逆向

## When to Invoke

未发货退款、已发货退货退款、部分退、退库存、退券、退分佣。电商面试：「正向和逆向怎么走」。

## When NOT

支付回调入账 → `idempotent`。订单正向跳转 → `order-state`。对账 → `reconciliation`。

## 风险（面试考点）

逆向不是 `status=REFUND` 一行改完。要按 **是否已发货 / 是否已核销** 分路径。重复退款、库存加回两次、券退回后被已使用订单再占用，都是资损。

售后单自己做状态机，不要和订单状态搅在一个字段里打补丁。

## 方案选型（轻量优先）

| 场景 | 默认路径 |
|------|----------|
| 未发货仅退款 | 售后单 → 调渠道退款 → 回调成功 → 订单 PAID→REFUNDED，释放预占，券 LOCKED/USED 按规则回退 |
| 已发货退货 | 审核 → 用户寄回 → 确认收货 → 再退款 |
| 已核销（票务） | 通常不能原路退库存名额，走废票/人工规则 |
| 部分退 | 按订单行退，不整单关 |

退款调用渠道：同一 `refundNo` 重试，禁止每次新单号。超时查单，见 `rpc-remote`。

## 默认方案

```text
t_aftersale: aftersale_no uk, order_no, type, status, refund_amount
允许：CREATED → APPROVED → REFUNDING → REFUNDED
        CREATED → REJECTED
退款回调：uk(refund_no, channel)
```

```java
int n = aftersaleMapper.update(null, Wrappers.<Aftersale>lambdaUpdate()
    .set(Aftersale::getStatus, "REFUNDED")
    .eq(Aftersale::getAftersaleNo, no)
    .eq(Aftersale::getStatus, "REFUNDING"));
if (n == 1) {
    releaseStockOnce(orderNo);  // 流水 uk
    unlockCouponOnce(orderNo);
    orderMapper.update(..., .eq(Order::getStatus, "PAID")); // 整单退
}
```

库存释放用 `inventory-occupy` 的释放流水。分佣已入账要记负向流水，不要 delete。

## 反例

错误：退款成功直接 `stock++`，回调重试加两次。
正确：释放流水唯一键。

错误：已发货仅退款不走仓库，货还在路上钱已退。
正确：分路径。

错误：把订单从 COMPLETED 直接改 CLOSED。
正确：售后单推进，订单到 REFUNDED/PARTIAL_REFUND。

## 验证

- 同一退款回调 10 次，钱只退一次、库存只加一次。
- 未发货退款后可售恢复；已核销票按规则不恢复或走废票。
- 正向 PAID 与售后 REFUNDING 并发，状态机不丢钱。

## 评审清单

- [ ] 售后单独立状态机
- [ ] 退款单号稳定可重试
- [ ] 回库存/回券有流水
- [ ] 发货前后路径分开
---
