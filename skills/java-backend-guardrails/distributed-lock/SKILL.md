---
name: distributed-lock
description: Use when multiple instances need mutual exclusion with Redisson or Redis SET NX PX, 看门狗, fencing. Do not use as the default for oversell/stock (backend-concurrency-sell) or as a substitute for unique keys (backend-idempotent).
---

# 分布式锁

## When to Invoke

多实例必须互斥执行：定时任务只跑一次、库存预热、无法用唯一键表达的临界区。

## When NOT

能靠唯一约束或 `WHERE stock>=n` 解决的，不要上锁。锁不是幂等，也不是超卖的主方案。

## 风险（面试考点）

面试标准错答：`SETNX` + 过期 + `DEL`。坑：

1. `SETNX` 与 `EXPIRE` 非原子，进程崩溃会死锁。
2. `DEL` 不校验持有者，会删掉别人的锁。
3. 业务超时锁过期，两个客户端同时进临界区。看门狗续期不能防止 GC 停顿后「自己以为还持有」。
4. 正确性要求高时需要 fencing token（资源侧拒绝旧令牌），只靠 Redis 锁不够。见 Kleppmann 对 Redlock 的分析；Redis 官方文档也要求认真对待 fencing。

官方单实例加锁：`SET key token NX PX ttl`，释放用 Lua/`DELEX` 比对 token。

## 适用场景

- 集群定时对账，同一时刻只允许一个实例
- 热点 key 回源单飞的跨进程互斥（通常单飞更轻）
- 无法加唯一键的「创建若不存在」跨实例

## 方案选型（轻量优先）

| 优先级 | 方案 |
|--------|------|
| 1 | 不上锁，用唯一键 / 条件更新 |
| 2 | Redisson 锁（lease + 看门狗）做**尽力互斥** |
| 3 | 资金级正确：DB 约束 + fencing，或不依赖 Redis 锁的正确性 |

不要把 Redlock 当面试必上。单 Redis 实例锁 + 业务侧约束通常更清晰。

## 默认方案

```java
RLock lock = redisson.getLock("lock:job:settle:" + bizDate);
boolean locked = lock.tryLock(0, 30, TimeUnit.SECONDS);
if (!locked) {
    return;
}
try {
    runSettle();
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

手写等价（不要 SETNX 再 EXPIRE）：

```text
SET lock:xxx <uuid> NX PX 30000
-- 释放：Lua GET==uuid 才 DEL
```

锁内业务必须仍幂等。锁丢失时唯一键还能挡第二笔。

## 反例

错误：用锁包住「读库存、减一、写回」。
正确：`UPDATE stock=stock-1 WHERE stock>=1`。

错误：`DEL lock` 不看 token。
正确：比对 token 再删。

错误：锁超时 10s，业务可能 30s，无续期也无 fencing。
正确：lease 足够或看门狗；正确性靠 DB。

## 验证

- 两实例同时抢，临界区不重叠（尽力而为）。
- 持有者崩溃，ttl 后他人能获得锁。
- 非持有者 unlock 不影响新持有者。
- 库存/支付路径即使去掉锁仍正确。

## 评审清单

- [ ] 是否其实不需要锁
- [ ] SET NX PX + token 释放，或 Redisson
- [ ] 锁内业务仍幂等/条件更新
- [ ] 未把锁当超卖银弹
---
