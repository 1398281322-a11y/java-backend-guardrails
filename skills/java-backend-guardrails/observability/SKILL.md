---
name: observability
description: Use when adding tracing, metrics, 链路追踪, TraceId, 慢接口排查, 全链路压测, or production incident workflow. Skip when the task is greenfield CRUD with no performance or incident context.
---

# 可观测 / 线上排查

## When to Invoke

慢接口、偶发失败、要加监控、事故复盘、全链路压测。高级面常问「线上 RT 高你怎么查」。

## When NOT

还没写业务就上完整 APM 平台当任务本身。先有 TraceId 和关键指标。

## 风险（面试考点）

没有 **同一 TraceId** 穿网关→服务→MQ→DB，日志对不上。指标只看 CPU 会误判（可能是锁、GC、慢 SQL、依赖超时）。压测不带数据隔离会打脏生产。

排查顺序（固定）：

1. 是全站还是单接口、从何时开始  
2. 错误率 / RT / QPS 哪个变了  
3. 依赖 RT（DB、Redis、Feign、MQ 积压）  
4. 实例是否均衡、GC、线程池拒绝  
5. 单请求日志（TraceId）看卡在哪一段  

不要先 dump 堆。

## 方案选型（轻量优先）

| 层 | 默认 |
|----|------|
| 日志 | 每条带 `traceId`、`userId`、`orderNo` |
| 指标 | QPS、RT P99、错误率、线程池、DB 连接池、MQ 积压 |
| 追踪 | 网关生成 TraceId，Feign/MQ 透传 |
| 告警 | 错误率、P99、积压、拒绝次数，不是 CPU 单指标 |

## 默认方案

MDC 放 TraceId；HTTP 头 `X-Trace-Id`；RocketMQ 属性同样透传。慢 SQL 阈值 1s 先打日志。核心接口有黄金指标看板。

压测：影子表/影子标，或预发全量。生产全链路压测必须能识别压测流量并隔离写。

## 反例

错误：RT 高先重启再 dump。
正确：按上面 1–5。

错误：日志没有业务单号，只能搜时间。
正确：`orderNo` 可检索。

错误：只监控机器 CPU。
正确：先看接口和依赖。

## 验证

- 任意失败请求能用 TraceId 串起网关到 DB。
- 人为把从库打慢，看板能显示是 DB 不是 JVM。

## 评审清单

- [ ] 写路径日志有业务键 + TraceId
- [ ] 核心接口有 QPS/RT/错误
- [ ] 排查顺序不是先重启
---
