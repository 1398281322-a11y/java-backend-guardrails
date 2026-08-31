---
name: rate-limit
description: Use when protecting public or hotspot APIs with 限流, 熔断, 降级, Sentinel, gateway rate limit, or shedding traffic. Do not use for ordinary internal single-table CRUD with no burst risk.
---

# 限流 / 熔断 / 降级

## When to Invoke

公开接口、秒杀、登录、短信、支付回调之外的营销入口、依赖方变慢、需要保命。

## When NOT

内网管理端普通 CRUD、低流量内部任务。不要给每个列表查询加一套 Sentinel 规则当默认模板。

## 风险（面试考点）

流量或依赖故障把线程池、连接池打满，雪崩。限流算法：固定窗口有突刺；滑动窗口更平滑；漏桶匀速；令牌桶允许突发。熔断：失败率高时 fail-fast，半开探测。降级：返回缓存/底图/空列表，而不是把异常抛到用户再拖垮自己。

## 适用场景

- 网关入口（全局限流）
- 热点 SKU / 热点用户参数限流
- 调第三方（票务、支付查询）变慢时熔断
- 非核心：推荐、积分展示可降级

## 方案选型（轻量优先）

| 层级 | 默认 | 升级 |
|------|------|------|
| 接入 | 网关 QPS 限流（Nginx/Gateway） | 按 IP/用户维度 |
| 应用 | Sentinel 资源规则（QPS + 慢调用比） | 热点参数限流 |
| 依赖 | 熔断 + 超时（见 `rpc-remote`） | 舱壁隔离线程池 |
| 业务 | 降级开关（配置中心） | 核心/非核心分级 |

令牌桶：允许短突发（营销开抢）。漏桶：保护下游匀速（写库、发短信）。不要两个都上到同一接口除非能说清。

## 默认方案

公开写接口、高 QPS 读：网关一层 + 应用 Sentinel。内部 CRUD：不加成套规则。

Sentinel 资源名用业务名：`POST:/order/create`、`RPC:ticket-query`。

```java
@SentinelResource(value = "createOrder", blockHandler = "createOrderBlock")
public OrderId createOrder(cmd) { ... }

public OrderId createOrderBlock(cmd, BlockException e) {
    throw new BizException("活动火爆，请稍后重试");
}
```

降级：非核心读失败返回空或本地缓存，核心写失败要明确错误，不许假装成功。

热点：对 `skuId` 单独限流，避免一个爆款拖死集群。

## 反例

错误：只在 Controller 里 `if (count++ > 100)` 单机计数。多实例无效。
正确：网关或 Redis/Sentinel 集群规则。

错误：熔断后仍同步重试把依赖打得更死。
正确：熔断打开期间 fail-fast；半开小流量。

错误：降级吞掉支付失败返回成功。
正确：资金路径不可降级为成功。

## 验证

- 压测超过阈值返回限流，不打满 DB 连接。
- 依赖 100% 失败，熔断打开，本服务线程池不耗尽。
- 非核心降级后核心下单仍可用。

## 评审清单

- [ ] 公开/热点接口有入口限流
- [ ] 外部依赖有超时 + 熔断
- [ ] 资金/核销路径没有「失败当成功」的降级
- [ ] 简单内部 CRUD 没有堆限流模板
---
