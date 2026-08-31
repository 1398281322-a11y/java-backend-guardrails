---
name: order-state
description: Use when implementing 订单状态机, 待支付/已支付/已发货/已完成/已取消, illegal status jumps, or 正向履约. For 售后逆向 use backend-refund-aftersale. For stock occupy timing use backend-inventory-occupy.
---

# 订单状态机

## When to Invoke

下单、支付回调改状态、发货、确认收货、取消。电商面试必问「订单有哪些状态、怎么防乱跳」。

## When NOT

售后单自己的状态 → `refund-aftersale`。库存何时扣 → `inventory-occupy`。关单怎么触发 → `delay-job`。

## 风险（面试考点）

状态用一堆 if 散落各处：已取消还能支付成功、已支付被超时关单、发货接口不校验 PAID。资金事故多半是 **非法跳转**，不是少了 Seata。

更新必须 `WHERE id=? AND status=期望旧值`，影响行数 0 视为冲突，走幂等/告警，不要覆盖。

## 方案选型（轻量优先）

默认枚举 + 允许表，不必上状态机引擎。状态一多（>8 且多角色操作）再上框架。

正向默认：

```text
CREATED ──支付成功──► PAID ──发货──► SHIPPED ──收货──► COMPLETED
    │
    └──超时/用户取消──► CLOSED
```

允许边（写死在代码或表）：

| 从 | 到 | 条件 |
|----|----|------|
| CREATED | PAID | 支付回调验签、金额一致 |
| CREATED | CLOSED | 未支付 |
| PAID | SHIPPED | 商家发货 |
| PAID | REFUNDING | 未发货退款，见售后 |
| SHIPPED | COMPLETED | 确认收货或超期 |

禁止：CLOSED→PAID（钱到了要进差错/人工，不能默默改回）、COMPLETED→CLOSED。

## 默认方案

```java
int n = orderMapper.update(null, Wrappers.<Order>lambdaUpdate()
    .set(Order::getStatus, "PAID")
    .eq(Order::getOrderNo, no)
    .eq(Order::getStatus, "CREATED"));
if (n == 0) {
    Order cur = orderMapper.selectByNo(no);
    if ("PAID".equals(cur.getStatus())) return; // 重复回调
    throw new BizException("状态不允许支付确认: " + cur.getStatus());
}
```

支付回调先落 `t_pay_record` 唯一键，再推状态，见 `idempotent`。关单只关 CREATED，见 `delay-job`。

## 反例

错误：`order.setStatus("PAID"); updateById(order);` 不带旧状态。
正确：条件更新。

错误：超时关单和支付回调并发，关单赢了但钱已扣。
正确：关单 `status=CREATED`；若已 PAID 则关单空操作，钱以支付单为准，对账处理。

## 验证

- 已支付订单跑关单任务，状态仍为 PAID。
- 重复支付回调只成功一次跳转。
- 未支付不能发货，接口返回业务错误。

## 评审清单

- [ ] 所有改状态 SQL 带旧状态条件
- [ ] 有允许边列表，非法边有测试
- [ ] 关单与支付回调并发有结论
---
