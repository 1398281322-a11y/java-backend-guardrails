---
name: distributed-transaction
description: Use when a business write spans services or datasources (order then coupon, pay then commission, verify then settle), or the user mentions 本地消息表, 事务消息, TCC, Seata, Saga. Do not use for a single-service single-DB transaction. Never default to TCC.
---

# 分布式事务

## When to Invoke

一次业务要改两个服务或两个库：下单后发券、支付成功后分佣、核销后结算、退款后回库存。

## When NOT

单服务单库：用本地 `@Transactional`。不要为了「微服务」上分布式事务。

## 风险（面试考点）

服务 A 成功、服务 B 失败 → 钱货不一致。面试常逼问 2PC/TCC/Seata。那些方案有锁时间和耦合代价。业务体量够用「本地事务 + 可靠消息」时上 TCC 就是过度设计。

## 适用场景

- 订单服务写订单，营销服务发券
- 支付成功通知分销结算
- 核销服务改券状态，订单服务改核销状态

## 方案选型（轻量优先）

| 优先级 | 方案 | 一致性 | 用在 | 不要用在 |
|--------|------|--------|------|----------|
| 1 | 本地事务 + RocketMQ 事务消息 | 最终 | 已有 RocketMQ，跨服务通知 | 需要同步强一致返回 |
| 2 | 本地消息表（outbox） | 最终 | 不能用事务消息时 | 愿意引入 Seata 之前 |
| 3 | Saga（成功路径 + 补偿） | 最终 | 长流程、可补偿 | 补偿写不清的资金流 |
| 4 | TCC | 近似强一致 | 预占资源必须 Confirm/Cancel | 默认业务 |
| 5 | Seata AT / XA | 强一致表象 | 明确要求且能接受锁 | 默认；长事务、高并发 |

默认：RocketMQ 事务消息（或 outbox）。禁止无脑 TCC/Seata。

升级到 TCC 的触发条件：必须预占（库存/余额冻结），Confirm/Cancel 有明确对账，团队能运维空回滚、悬挂。

## 默认方案（RocketMQ 事务消息）

本地 DB 提交与「半消息」绑定：本地成功 Commit 消息，本地失败 Rollback 消息。Broker 回查本地状态。

```java
transactionMQProducer.sendMessageInTransaction(msg, order);
// executeLocalTransaction: 写订单（本地事务）
// checkLocalTransaction: SELECT 订单是否已提交 → COMMIT / ROLLBACK
```

消费者（发券 / 分佣）必须幂等，见 `idempotent`。消息会重复。

Outbox 等价实现：

```sql
-- 与订单同一本地事务
INSERT INTO outbox(biz_no, topic, body, status) VALUES (?, ?, ?, 'NEW');
```

定时/CDC 扫描 `NEW` 发 RocketMQ，成功标 `SENT`。消费者幂等。

补偿（Saga）只在「能反向操作」时写：发券失败可关单；分佣失败可重试入账，不可随便再扣一次。

## 反例

错误：A 服务 HTTP 调 B，A 本地已提交，B 超时后 A 再重试却把超时当失败回滚——对端可能已成功。
正确：超时当未知；B 幂等；用消息而不是同步链。

错误：一上来 Seata 全局锁包下单+支付+分佣。
正确：支付回调已是异步，用事务消息即可。

错误：TCC 的 Cancel 不可幂等，悬挂时误取消别人的 Try。
正确：TCC 每个阶段都要幂等，用冻结单号关联。

## 验证

- 本地事务回滚，下游收不到消息。
- 本地成功但消息未 Commit，Broker 回查后仍能投出。
- 下游重复消息，券/分佣只入账一次。
- 下游持续失败，有重试上限、死信、人工对账，而不是悬挂半成功订单。

## 评审清单

- [ ] 是否真的跨服务写？单库请退回本地事务
- [ ] 默认是事务消息或 outbox，不是 TCC
- [ ] 下游幂等
- [ ] 有对账或回查，不只靠同步 RPC
---
