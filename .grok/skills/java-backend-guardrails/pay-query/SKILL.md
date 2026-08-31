---
name: pay-query
description: Use when payment HTTP times out, 回调丢失, 主动查单, PAYING 补偿扫描, or 支付成功不等于收到 notify. Do not treat timeout as failure and create a new out_trade_no.
---

# 支付查单 / 丢回调

## When to Invoke

统一下单超时、用户付了但订单仍待支付、渠道 notify 丢失、面试「回调丢了怎么办」。

## When NOT

回调本身怎么写 → `pay-notify`。对账文件 → `reconciliation`。超时关单 → `delay-job`（关单前必须先查单）。

## 风险（面试考点）

支付请求超时 **只表示没收到应答，渠道可能已扣款**。再换一个 `out_trade_no` 重下单 = 用户付两笔。

不能只靠回调。生产三道：异步 notify + **主动查单** + 日终对账。缺任何一道都会漏单或错账。

关单任务若不管渠道状态，会把已付款订单关掉，然后回调到来变成「已取消却收了钱」。

## 方案选型（轻量优先）

| 时机 | 动作 |
|------|------|
| 下单/支付 HTTP 超时 | 用 **同一个** out_trade_no 调渠道 query |
| 用户回前台 | 查本地，仍 PAYING 则再 query 一次 |
| 定时 | 扫 PAYING 超过 N 分钟（如 2～5 分钟）的支付单，query |
| 关单前 | 先 query，已支付则推进 SUCCESS，不要关 |
| T+1 | 对账兜底 |

Query 结果当「官方回调」走同一套入账逻辑（验金额、唯一流水、状态机），不要写第二套。

## 默认方案

```java
public void compensate(PayTrade t) {
    ChannelQuery q = channel.query(t.getOutTradeNo()); // 超时当未知，有限重试
    if (q.isSuccess()) {
        payNotifyService.applySuccess(q); // 复用入账
    } else if (q.isClosedOrNotPay() && t.expired()) {
        payTradeService.markClosed(t.getOutTradeNo());
    }
    // PROCESSING / UNKNOWN → 下次再扫
}
```

关单：

```text
到期 → query
  SUCCESS → 走入账（可能回调晚到）
  NOTPAY → 关支付单 + 关订单 CREATED
  UNKNOWN → 不关，继续 PAYING
```

## 反例

错误：超时立刻 `out_trade_no = UUID` 再调统一下单。
正确：同一单号查单或关旧单后再开新单（产品允许时）。

错误：只扫「创建超过 30 分钟」直接关，不 query。
正确：先问渠道。

错误：查单成功自己写一套 update，和回调逻辑分叉。
正确：同一 applySuccess。

## 验证

- 模拟 notify 丢失，补偿任务能把订单打成已支付。
- 超时后没有第二笔渠道订单（除非产品明确允许）。
- 关单窗口内付款，订单不会停在 CLOSED 且钱在渠道。

## 评审清单

- [ ] 超时当未知
- [ ] PAYING 有补偿扫描
- [ ] 关单前查单
- [ ] 查单与回调共用入账
---
