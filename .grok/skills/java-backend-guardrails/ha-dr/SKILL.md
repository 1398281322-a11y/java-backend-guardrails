---
name: ha-dr
description: Use when designing 容灾, 高可用, failover, RPO/RTO, Redis/MySQL/RocketMQ downtime, multi-AZ, or degradation. Do not treat HA and DR as the same thing. Skip for feature CRUD with no availability target.
---

# 容灾 / 高可用

## When to Invoke

定可用性目标、主从切换、机房故障、Redis/MQ/DB 挂了怎么办、降级兜底。

## When NOT

普通功能开发且没有 SLO。不要用「上集群」代替备份恢复演练。

## 风险（面试考点）

HA ≠ DR。HA 是同城故障自动切；DR 是异地，允许更大 RPO/RTO。没有演练的备份等于没有备份。Redis 当缓存挂了应降级，当锁/库存账本挂了会停业务——这是选型问题。

RPO：能丢多久的数据。RTO：多久恢复服务。

## 适用场景

- 生产 Redis / MySQL / RocketMQ 部署
- 支付核心 vs 推荐非核心的分级
- 机房级预案

## 方案选型（轻量优先）

| 组件 | 默认 HA | 容灾 |
|------|---------|------|
| MySQL | 主从 + 自动/半自动切主 | binlog 备份；定期 restore 演练 |
| Redis 缓存 | 哨兵或云主从 | 可丢；宕机走 DB + 限流 |
| Redis 账本/锁 | 不要当唯一账本 | 数据应在 MySQL |
| RocketMQ | 多副本 / DLedger | 堆积可接受；不能单 Broker |
| 应用 | 多实例 + 健康检查 | 跨 AZ |

分级：支付/核销不可降级为成功；推荐/广告可关。

## 默认方案

缓存挂了：熔断 Redis，读 DB，并限流（`rate-limit` + `cache-trap`）。

DB 从库延迟：写后读走主库或会话粘滞，不要默认「一律读从」。

切主：应用用 VIP/域名，禁止写死 IP。

备份：只说「每天 RDB」不够。要有恢复步骤和上次演练时间。缓存类 Redis 可用 RDB；不能丢的数据不要只放 Redis。Redis 持久化：缓存可关 AOF；类数据要用 AOF `everysec` + 副本，仍不要当 MySQL 替代。

## 反例

错误：Redis 单点，当库存唯一存储。
正确：库存在 DB，Redis 只加速。

错误：从库延迟 10s 仍把支付结果查询打到从库。
正确：资金读主。

错误：从未做过 backup restore，却宣称 RPO=1min。
正确：RPO 以演练为准。

## 验证

- 杀一个应用实例，接口仍可用。
- 杀 Redis 主，哨兵切换，缓存业务可恢复；核心写不依赖它。
- 用备份在预发恢复出库。

## 评审清单

- [ ] 核心路径不把 Redis 当账本
- [ ] 有超时/熔断/降级，且资金路径不假成功
- [ ] 备份有恢复演练
- [ ] HA 和异地容灾没有混为一谈
---
