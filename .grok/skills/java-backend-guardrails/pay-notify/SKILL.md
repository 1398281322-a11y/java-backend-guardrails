---
name: pay-notify
description: Use when implementing 支付异步回调, WeChat/Alipay notify, 验签, 金额校验, notify 返回 SUCCESS. Do not use user JWT on callbacks. Unique-key mechanics also in backend-idempotent.
---

# 支付异步回调

## When to Invoke

微信/支付宝/聚合支付 **异步通知**、return_url 与 notify 的区别、回调重试。资金面试第一题。

## When NOT

用户点支付、预下单 → `pay-trade`。丢回调去查单 → `pay-query`。重复付了两笔 → `pay-duplicate`。通用幂等表结构可再读 `idempotent`，本文件管支付回调流水线。

## 风险（面试考点）

渠道收不到成功应答会重试（微信约 24h 内多次）。不幂等 = 重复入账、重复发货。

回调 **不是用户浏览器**，不能验 JWT。必须 **验签 → 校验商户号/金额/币种/单号 → 落唯一流水 → 推状态**。验签失败直接拒绝，不要落业务。

在回调线程里调渠道退款、发券、RPC：超时会导致渠道以为失败再打，把线程池打满。回调只做短事务，重活丢 MQ。

return_url（同步跳回）只展示，**不能当入账依据**。入账只认异步 notify 或主动查单。

金额单位：渠道常用 **分**。本地订单是元会错 100 倍，必须统一成分比较。

## 方案选型（轻量优先）

```text
1. 验签（官方 SDK，禁止手写拼串除非渠道要求）
2. 校验 mchId、out_trade_no、amount、currency 与本地支付单一致
3. INSERT 回调流水 uk(channel, transaction_id 或 notify_id)
   冲突 → 已处理，仍返回 SUCCESS
4. 支付单 PAYING/CREATED → SUCCESS（条件更新）
5. 发 MQ：改订单、发货、发券（消费者幂等）
6. HTTP 返回渠道要求的 SUCCESS/XML
```

业务处理失败要返回 FAIL 让渠道重试 **仅当还没落成功流水**。已经 SUCCESS 的重复通知必须 SUCCESS，否则会重试到天荒。

## 默认方案

```java
public String notify(HttpServletRequest req) {
    if (!wxPay.verifyNotifySign(req)) return "FAIL"; // 不落业务
    WxNotify n = parse(req);
    PayTrade t = tradeMapper.selectByOutNo(n.getOutTradeNo());
    if (t == null || !t.getMchId().equals(n.getMchId())
        || t.getAmountFen() != n.getAmountFen()) {
        log.error("回调与本地不一致");
        return "FAIL";
    }
    try {
        logMapper.insert(new PayNotifyLog(n.getTransactionId(), n.getRaw()));
    } catch (DuplicateKeyException e) {
        return "SUCCESS";
    }
    int rows = tradeMapper.update(null, Wrappers.<PayTrade>lambdaUpdate()
        .set(PayTrade::getStatus, "SUCCESS")
        .set(PayTrade::getChannelTxnId, n.getTransactionId())
        .eq(PayTrade::getOutTradeNo, n.getOutTradeNo())
        .in(PayTrade::getStatus, "CREATED", "PAYING"));
    if (rows == 1) {
        mq.send("pay-success", t.getOrderNo()); // 异步改订单
    } else {
        handleNotifyConflict(t.getOutTradeNo()); // 见 pay-duplicate
    }
    return "SUCCESS";
}
```

## 反例

错误：先查是否已支付再 insert，并发双入账。
正确：先插唯一流水。

错误：验签失败也返回 SUCCESS 以免重试。
正确：假通知必须 FAIL/拒绝。

错误：回调里同步发券、扣库存。
正确：短事务 + MQ。

错误：只处理同步 return_url。
正确：用户关页面就丢单。

## 验证

- 同一 notify 连打 10 次，支付单 SUCCESS 一次，下游 MQ 消费幂等。
- 改金额的伪造回调被拒。
- 验签失败无业务副作用。

## 评审清单

- [ ] 先验签再业务
- [ ] 金额/商户号/单号比对，单位为分
- [ ] 唯一流水 + 状态条件更新
- [ ] 已成功的重复通知返回 SUCCESS
- [ ] 回调内无耗时 RPC
---
