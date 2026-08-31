---
name: idempotent
description: Use when handling duplicate HTTP submits, payment/refund callbacks, RocketMQ reconsume, Feign retries, 核销, 结算, or 防重. Do not use for read-only queries or simple unique-free CRUD with no retry/callback.
---

# 幂等

## When to Invoke

重复提交、支付/退款/第三方回调、MQ 重试、超时后客户端重试、核销、分佣入账。

## When NOT

只读查询；确定不会重试、没有回调、没有唯一业务键的一次性写。

## 风险（面试考点）

同一笔业务被执行两次：重复扣款、重复核销、重复分佣、重复发券。HTTP 超时、MQ 至少一次投递、回调重放都会制造重复。面试常考「如何保证消息不被重复消费」——本质是消费侧幂等，不是 MQ 只投一次。

## 适用场景

- 支付/退款回调（同一 `notifyId` / `outTradeNo` 会打多次）
- 订单提交、核销、结算入账
- RocketMQ 消费失败 reconsume
- Feign/HTTP 超时后重试

## 方案选型（轻量优先）

| 优先级 | 方案 | 用在 | 代价 |
|--------|------|------|------|
| 1 | 数据库唯一约束 + 状态机 | 所有写库业务 | 要设计好唯一键 |
| 2 | 业务单号查重（先查后写，仍靠唯一约束兜底） | 回调、核销 | 并发下必须靠唯一键 |
| 3 | Redis `SET key NX EX` 短窗口 | 按钮连点、秒级防抖 | 不能当唯一资金防线 |
| 4 | 分布式锁 | 无法用唯一键表达的临界区 | 重，见 `distributed-lock` |

默认：唯一约束 + 状态机。不要只用 Redis 防重当资金安全网。

## 默认方案

1. 每笔写操作有业务单号：`requestId` / `outTradeNo` / `verifyNo`。
2. 表上建唯一键，插入冲突视为「已处理」，返回首次结果。
3. 状态只允许单向跳：`CREATED → PAID → REFUNDED`，非法跳转直接忽略或报业务错误。
4. MQ：消费成功后再 ACK；失败重试；处理函数按业务单号幂等。

```sql
-- 支付回调 / 核销 落库
UNIQUE KEY uk_pay_callback (out_trade_no, channel)
UNIQUE KEY uk_verify (order_id, voucher_id)
UNIQUE KEY uk_commission (order_id, rule_id)
```

```java
@Transactional
public void onPayNotify(PayNotify n) {
    PayRecord row = new PayRecord();
    row.setOutTradeNo(n.getOutTradeNo());
    row.setChannel(n.getChannel());
    row.setPayload(n.getRaw());
    try {
        payRecordMapper.insert(row); // uk 冲突即重复回调
    } catch (DuplicateKeyException e) {
        return; // 已处理
    }
    int ok = orderMapper.update(null, Wrappers.<Order>lambdaUpdate()
        .set(Order::getStatus, "PAID")
        .eq(Order::getId, n.getOrderId())
        .eq(Order::getStatus, "CREATED"));
    if (ok == 0) {
        return; // 已支付或状态不允许
    }
}
```

RocketMQ 消费：

```java
public void onMessage(MessageExt msg) {
    String bizNo = msg.getKeys(); // 生产时必须设 keys=业务单号
    try {
        handleOnce(bizNo, msg);
        // 正常返回 → ACK
    } catch (DuplicateKeyException e) {
        // 已处理，ACK
    }
    // 抛其他异常 → reconsume
}
```

接口层短窗口（防连点，不是资金防线）：

```java
String key = "idem:" + uid + ":" + action + ":" + bizNo;
boolean first = Boolean.TRUE.equals(stringRedisTemplate.opsForValue()
    .setIfAbsent(key, "1", Duration.ofSeconds(10)));
if (!first) {
    throw new BizException("请勿重复提交");
}
```

## 反例

错误：只靠 Redis `SETNX` 防重复支付；Redis 重启/过期会双花。
正确：DB 唯一键是底线，Redis 只挡连点。

错误：MQ 自动 ACK，处理失败也丢，然后用「不会重复」当幂等。
正确：至少一次投递 + 消费幂等。重复是正常的。

错误：先查「有没有记录」再 insert，没有唯一键。
正确：并发两个请求都会查到空。必须唯一约束兜底。

## 验证

- 同一回调连打 10 次，业务表只变一次。
- 消费端丢异常重试，最终状态正确且金额不翻倍。
- 超时后客户端重放，返回首次成功结果而不是第二笔。

## 评审清单

- [ ] 写路径有业务唯一键
- [ ] 状态机单向，重复到达可安全忽略
- [ ] MQ keys 用业务单号，消费幂等
- [ ] 超时当未知，不把「没收到响应」当失败再开新单
---
