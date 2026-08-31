---
name: rpc-remote
description: Use when calling other services or third parties via Feign, HTTP, RPC: 超时, 重试, 仓壁, deadline, 第三方对接. Do not retry non-idempotent writes. Circuit breaking details also in backend-rate-limit.
---

# 远程调用

## When to Invoke

Feign 调内部服务、HTTP 调票务/支付/短信、任何同步出站调用。

## When NOT

进程内方法调用。熔断阈值与网关限流的细则见 `rate-limit`；本文件管超时、重试、线程隔离、把超时当未知。

## 风险（面试考点）

无超时 = 线程被慢依赖拖死。无脑重试非幂等接口 = 重复下单/重复扣款。超时后当失败再开新单 = 对端可能已成功。雪崩：一个慢依赖占满 Tomcat/Feign 线程池。

## 适用场景

- 订单服务调库存、营销
- 支付查询、退款申请
- 第三方核销/出票

## 方案选型（轻量优先）

| 项 | 默认 |
|----|------|
| 超时 | 必设；小于调用方剩余 deadline |
| 重试 | 仅幂等读或明确幂等写；指数退避 + 抖动；有限次 |
| 熔断 | 错误率/慢调用比过高则 fail-fast |
| 隔离 | 按依赖分线程池（舱壁），避免票务拖死支付 |
| 语义 | 超时 = 未知，不是失败 |

资金写：宁可查单，不要立刻重放一笔新的。

## 默认方案

```yaml
# Feign / HTTP
feign.client.config.ticket-service.connectTimeout: 200
feign.client.config.ticket-service.readTimeout: 800
```

```java
@Retryable(
    retryFor = RetryableException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 50, multiplier = 2, random = true)
)
public Ticket queryTicket(String id) { ... } // GET 幂等，可重试
```

不要给 `createOrder` / `refund` / `verify` 加自动重试，除非对端按业务单号幂等。

超时后：

```java
try {
    payClient.pay(req); // req 带 outTradeNo
} catch (TimeoutException e) {
    PayStatus s = payClient.query(req.getOutTradeNo()); // 查，不重开
}
```

舱壁：票务、短信、支付各一套线程池/连接池。

## 反例

错误：`readTimeout=30s` 且无限重试。
正确：毫秒到两秒级（视 SLA），有限次，只重试安全接口。

错误：支付 HTTP 失败立刻换一个新的 `outTradeNo` 再扣。
正确：同一单号查询或重放。

错误：所有 Feign 共用一个线程池。
正确：按依赖隔离。

## 验证

- 对端 sleep 超过超时，调用方在超时内返回，线程不堆死。
- 非幂等写在故障注入下不会变成两笔。
- 一个依赖全挂，其他依赖接口仍可用。

## 评审清单

- [ ] 每个出站调用有超时
- [ ] 非幂等写无自动重试
- [ ] 超时走查单而不是新单
- [ ] 关键依赖有隔离
---
