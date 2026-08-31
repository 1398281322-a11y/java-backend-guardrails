---
name: delay-job
description: Use when implementing 延迟关单, 超时未支付取消, delayed messages, RocketMQ delay levels, timing wheel, or Redis ZSET delay. Do not scan huge unpaid-order tables with @Scheduled.
---

# 延迟任务 / 关单

## When to Invoke

下单 N 分钟未支付关单、核销码过期、延迟双删、预约提醒。

## When NOT

立刻异步解耦 → `mq-reliability`。库存怎么扣 → `concurrency-sell`。关单时仍要幂等。日切对账、周期性补偿、`@Scheduled` / XXL-JOB → `schedule-job`。

## 风险（面试考点）

`@Scheduled` 每分钟 `SELECT * FROM t_order WHERE status=0 AND created_at < now()-30min`：数据一大就扫表、重复关、漏关。

RocketMQ 4.x 开源：**18 个固定级别**（1s … 2h），不是任意秒。内部 `SCHEDULE_TOPIC_XXXX` 一档一个队列，同档 FIFO，避免全局排序。5.x 才有时间轮任意延迟。没有 3 分钟档就选 2m 或 4m，或自己补级。

到期消息会重复投，关单必须 `WHERE status=CREATED`。已支付的到期消息应空操作。

## 方案选型（轻量优先）

| 量级 | 默认 |
|------|------|
| 已有 RocketMQ，30min 关单 | `setDelayTimeLevel(16)`（30m） |
| 任意时间且 5.x | 定时/时间轮消息 |
| 量很小 | Redis ZSET + 轮询 |
| 禁止 | 全表扫描未支付订单 |

## 默认方案

```java
Message msg = new Message("order-cancel", orderNo.getBytes());
msg.setKeys(orderNo);
msg.setDelayTimeLevel(16); // 30min
producer.send(msg);
```

```java
public void onCancel(String orderNo) {
    int n = orderMapper.update(null, Wrappers.<Order>lambdaUpdate()
        .set(Order::getStatus, "CLOSED")
        .eq(Order::getOrderNo, orderNo)
        .eq(Order::getStatus, "CREATED"));
    if (n == 1) {
        restoreStock(orderNo); // 条件加回，幂等
    }
}
```

关单与回库存同一本地事务，或事务消息。回库存仍 `stock=stock+n` 且防重复回。

## 反例

错误：定时扫全表未支付。
正确：下单时发延迟消息。

错误：到期不看状态直接关。
正确：只关 CREATED。

错误：用 `Thread.sleep(30min)` 在请求线程里等。
正确：立刻返回，到点再处理。

## 验证

- 未支付到期关单且库存加回一次。
- 到期前已支付，消息到达不关单、不加库存。
- 重复到期消息不双倍回库存。

## 评审清单

- [ ] 不是扫表关单
- [ ] 延迟档位与业务时长匹配
- [ ] 到期处理幂等 + 状态条件
---
