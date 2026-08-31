---
name: pay-trade
description: Use when creating 支付单, unified order, 预下单, 支付二维码过期, pay status vs order status. Do not mix payment state machine with order fulfillment states (backend-order-state).
---

# 支付单 / 预下单

## When to Invoke

调微信统一下单、生成 QR/小程序支付参数、支付中状态、二维码过期重开。面试：「订单和支付单为什么要分开」。

## When NOT

订单发货收货 → `order-state`。回调入账 → `pay-notify`。

## 风险（面试考点）

订单状态（CREATED/PAID/SHIPPED）和 **支付单状态**（CREATED/PAYING/SUCCESS/CLOSED/REFUNDING）不是一张表一个字段能讲清的。一笔订单可以对应多次支付尝试（过期重开、多付退余）。混在订单上会无法解释第二笔 `out_trade_no`。

预下单成功只代表渠道接受了单，用户还没付钱。此时是 PAYING，不是 PAID。把预下单当支付成功会发货。

QR 有过期时间。过期后必须 **关渠道单** 再新开，或同号续期（看渠道）。旧码仍可能被付，见 `pay-duplicate`。

## 方案选型（轻量优先）

```text
t_order     履约
t_pay_trade 一次渠道交互
  out_trade_no  商户支付单号（我们生成）
  order_no      业务订单
  channel       WX/ALI/BALANCE
  amount_fen
  status        CREATED → PAYING → SUCCESS | CLOSED
  expire_at
```

下单：本地订单 CREATED + 支付单 CREATED。调渠道 unified order 成功 → PAYING，把 prepay_id/code_url 给前端。失败保持 CREATED 可重试同一 out_trade_no。

金额、标题、notify_url 在预下单时定死，回调按此校验。

0 元单：不走渠道，本地直接 SUCCESS（仍要唯一支付单，防重复发货）。

## 默认方案

```java
PayTrade t = new PayTrade();
t.setOutTradeNo(idGenerator.nextPayNo()); // 不要用纯订单号若允许重开支付
t.setOrderNo(orderNo);
t.setAmountFen(order.getPayFen());
t.setStatus("CREATED");
tradeMapper.insert(t);

PrepayResp r = wx.unifiedOrder(t);
t.setStatus("PAYING");
t.setExpireAt(now.plusMinutes(15));
tradeMapper.updateById(t);
return r.getPayParams();
```

重开支付：先 close 旧 PAYING（渠道 close + 本地 CLOSED），再插新支付单。

## 反例

错误：订单表一个 `pay_status` 兼顾预下单和履约。
正确：支付单独立。

错误：unified order 成功就把订单改 PAID。
正确：只改支付单 PAYING，PAID 留给成功入账。

错误：每次刷新收银台新单号，旧码不关。
正确：关旧或复用。

## 验证

- 预下单后杀进程，订单仍 CREATED，可查单或重入。
- 过期关单后旧 QR 付款进入差错而不是覆盖履约。
- 0 元单不调渠道且只成功一次。

## 评审清单

- [ ] 支付单与订单分离
- [ ] 预下单 ≠ 已支付
- [ ] out_trade_no 稳定可查
- [ ] 重开支付有关旧单
---
