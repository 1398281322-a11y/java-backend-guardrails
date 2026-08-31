---
name: mq-reliability
description: Use when sending or consuming RocketMQ (or Kafka) messages: 消息丢失, 重复消费, 顺序, 积压, 死信, 延迟消息, transactional messages. Do not use Redis List as a production business queue.
---

# 消息队列可靠性

## When to Invoke

下单后异步发券/分佣/短信、支付结果广播、削峰、延迟关单。用户问丢失/重复/顺序/积压。

## When NOT

同步就能完成的单服务写。不要用 Redis List 当订单账本队列。

## 风险（面试考点）

三件套：不丢、可重复、可积压处理。RocketMQ 默认不是恰好一次。重复消费是正常现象，靠消费幂等（`idempotent`）。顺序只在同一队列/分区且单线程消费时成立。积压要扩消费者或降级，不是加大超时。

## 适用场景

- 支付成功 → 分佣 / 发券
- 核销成功 → 结算
- 超时关单（延迟消息）

## 方案选型（轻量优先）

| 问题 | 默认（RocketMQ） | 不要 |
|------|------------------|------|
| 生产不丢 | 同步发送；失败重试；事务消息/outbox | 发完不管返回 |
| Broker 不丢 | 刷盘 + 主从/DLedger | 单节点内存队列 |
| 消费不丢 | 业务成功后再 ACK；失败抛错 reconsume | 先 ACK 再处理 |
| 重复 | 消费者幂等 | 幻想 MQ 只投一次 |
| 顺序 | 同一 `shardingKey` 进同一队列，单线程消费 | 全局顺序 + 大吞吐 |
| 积压 | 扩消费者；非核心丢弃/降级；死信后人工 | 无限堆积 |
| 延迟 | RocketMQ 延迟级别 | 用 Thread.sleep |

Kafka 对照（若出现）：`acks=all` + `min.insync.replicas=2` + `enable.idempotence=true`；消费 `enable.auto.commit=false`，处理完再提交。`acks=all` 不是恰好一次。

## 默认方案

生产：

```java
SendResult r = producer.send(msg); // 同步
msg.setKeys(orderNo);              // 业务键，方便幂等和排查
```

跨服务与本地事务绑定：用事务消息，见 `distributed-transaction`。

消费：

```java
try {
    handleIdempotent(msg.getKeys(), body);
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
} catch (Exception e) {
    return ConsumeConcurrentlyStatus.RECONSUME_LATER;
}
```

顺序：同一订单号 `MessageQueueSelector` 选固定队列。

积压：先看消费耗时和线程数；慢依赖要拆开，避免一条消息里同步调三个 HTTP。超过最大重试进死信，告警，不要静默丢。

## 反例

错误：消费一进来就 SUCCESS，再异步处理。进程挂了消息丢。
正确：处理完再 SUCCESS。

错误：「开了事务消息就不会重复」。
正确：下游仍会收到重复，必须幂等。

错误：用 Redis 做百万级订单队列。
正确：Redis 适合缓存和短任务，不适合要堆积、回溯、投递保证的账类消息。

## 验证

- 杀消费者进程，未 ACK 的消息会再来，且业务不双写。
- Broker 短暂不可用，生产者有失败可感知、可重试或入 outbox。
- 故意堆积，扩消费者后水位下降；有死信告警。

## 评审清单

- [ ] 同步发送或 outbox，不发后即忘
- [ ] ACK 在业务成功之后
- [ ] keys=业务单号 + 消费幂等
- [ ] 有重试上限和死信
---
