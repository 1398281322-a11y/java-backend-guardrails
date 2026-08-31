---
name: pay-refund-channel
description: Use when calling WeChat/Alipay 退款 API, refund notify, 部分退, stable refundNo, refund timeout query. Business aftersale flow is backend-refund-aftersale; do not mix the two files.
---

# 渠道退款

## When to Invoke

向微信/支付宝发起退款、退款异步通知、部分退、退款超时。面试：「退款怎么保证只退一次」。

## When NOT

售后审核、退货物流 → `refund-aftersale`。那是业务单；本文件是 **渠道资金退回**。对账 → `reconciliation`。

## 风险（面试考点）

退款 API 超时 ≠ 没退。再换 `refund_no` 会退两次。必须 **同一退款单号重试 + 查退款**。

退款也有异步通知，同样验签、幂等、金额校验。

部分退：累计退款不能超过原支付成功金额。多次部分退每笔一个 refund_no。

原路退回到支付渠道，不要默认退到余额除非产品就是钱包。

回调里同步调退款会把支付线程拖死，且可能和关单逻辑缠死。退款走独立任务。

## 方案选型（轻量优先）

```text
t_pay_refund
  refund_no uk          我们生成，终身不变
  out_trade_no
  amount_fen
  status CREATED → REFUNDING → SUCCESS | FAIL
```

```java
public void refund(PayRefund r) {
    if (r.getStatus() == SUCCESS) return;
    ChannelRefundResp resp = channel.refund(r.getRefundNo(), r.getOutTradeNo(), r.getAmountFen());
    if (resp.acceptedOrSuccess()) {
        markRefundingOrSuccess(r, resp);
    } else if (resp.unknownOrTimeout()) {
        // 保持 REFUNDING，补偿任务 queryRefund(refund_no)
    } else {
        markFail(r, resp.getMsg());
    }
}
```

成功后才推进售后单 REFUNDED（消息通知 `refund-aftersale`）。渠道成功、售后单失败：对账补推进，不要再退一笔。

## 反例

错误：超时 newRefundNo 再调一次。
正确：同号重试 + queryRefund。

错误：退款通知不验签。
正确：与支付回调同等安全。

错误：部分退不记账已退金额，第三次把整单退超。
正确：`sum(success refund) + 本次 <= payAmount`。

## 验证

- 同一 refund_no 连调 5 次，渠道只受理一次，本地 SUCCESS 一次。
- 超时后补偿 query 能对上。
- 部分退累计不超过实付。

## 评审清单

- [ ] refund_no 稳定
- [ ] 超时查退款不是新单
- [ ] 有已退累计
- [ ] 与售后状态通过消息对齐，不在支付回调里同步退
---
