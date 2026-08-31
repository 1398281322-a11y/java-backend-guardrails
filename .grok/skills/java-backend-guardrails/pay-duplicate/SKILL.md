---
name: pay-duplicate
description: Use when handling 重复支付, 同一订单两笔成功, 支付与超时关单撞车, 用户连点收银台, or rows=0 after pay update. Do not auto-refund without reading current status.
---

# 重复支付 / 关单撞车

## When to Invoke

用户点两次支付、微信+余额同时付、支付回调与 30 分钟关单并发、条件更新影响行数 0。支付面试里最容易答错的并发题。

## When NOT

普通重复回调（已 SUCCESS 再来）→ `pay-notify` 幂等返回即可。渠道退款怎么调 → `pay-refund-channel`。

## 风险（面试考点）

1. **连点**：两次 unified order，两个 QR，用户付两次。  
2. **关单撞车**：29:59 付款，30:00 关单。关单赢了钱还在。  
3. **rows=0 就退款**：第二次回调 rows=0 可能是重复成功，不是失败。误退会把第一笔也冲掉或平白出账。

一笔业务订单应对一笔 **进行中** 支付单。新支付前关掉/作废旧的 PAYING（调渠道 close，失败则查单）。

## 方案选型（轻量优先）

| 现象 | 处理 |
|------|------|
| 重复回调，已 SUCCESS | 返回成功，不退款 |
| 订单已 PAID，又来一笔不同 transaction_id 成功 | **多付**：记差错，对多余一笔原路退，主单保持 PAID |
| 订单已 CLOSED，钱后到 | **迟到款**：禁止改回 PAID 覆盖关单；入差错，退款或捞回（产品定） |
| 关单 UPDATE CREATED→CLOSED 与支付抢行 | 谁更新成功谁赢；输的一方二次查状态再决策 |

默认建议：钱到了优先 **认款**（捞回订单）还是 **退款** 由产品定。代码必须分出这两种，禁止 rows=0 无脑退。

## 默认方案

```java
int rows = orderMapper.markPaid(orderNo, "CREATED");
if (rows == 1) return;
Order o = orderMapper.selectByNo(orderNo);
switch (o.getStatus()) {
    case "PAID" -> { /* 重复回调 */ }
    case "CLOSED" -> reconMapper.insertLatePay(orderNo, txnId); // 差错，异步退或捞回
    default -> log.error("支付回调碰到非法状态 {}", o.getStatus());
}
```

同一 `order_no` 只允许一条 `PAYING` 支付单（部分唯一索引或应用关旧开新）。

```sql
-- 支付单：同一订单可有多条历史，但成功入账要对账
uk (channel, out_trade_no)
-- 成功渠道交易号全局唯一
uk (channel, transaction_id)
```

## 反例

错误：`if (rows==0) refund()`。
正确：先查当前状态。

错误：收银台每次点都新 out_trade_no，旧 QR 仍能付。
正确：关旧单或同号覆盖（渠道允许时）。

错误：已退款后又把迟到的支付成功改成 PAID。
正确：REFUNDED 不能倒回，进差错。

## 验证

- 两笔不同 transaction_id 成功，只履约一次，多余一笔进入退款且只退一次。
- 关单与回调对打，最终要么 PAID+履约，要么 CLOSED+退款，不出现 CLOSED 且钱留下无记录。
- 重复回调不触发退款。

## 评审清单

- [ ] 同时只有一笔 PAYING
- [ ] rows=0 有状态分支
- [ ] 多付/迟到款有差错单
- [ ] 已退款不能被成功回调改回去
---
