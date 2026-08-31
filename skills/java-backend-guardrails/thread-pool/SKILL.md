---
name: thread-pool
description: Use when configuring Java thread pools, 拒绝策略, unbounded queue OOM, ThreadLocal leaks, 虚拟线程, or isolating report vs order executors. Do not use for Redis or HTTP timeout values (backend-rpc-remote).
---

# 线程池 / ThreadLocal

## When to Invoke

自定义线程池、报表和下单抢线程、ThreadLocal 用户上下文在异步后串号、JDK21 虚拟线程。

## When NOT

Feign 超时数值 → `rpc-remote`。GC 堆分析 → `jvm-prod`。

## 风险（面试考点）

`Executors.newFixedThreadPool` / `newCachedThreadPool`：前者 **无界队列**（max 线程永远不创建，堆积 OOM），后者线程无界。阿里规约禁止用它们生产。

参数：core、max、keepAlive、**有界队列**、拒绝策略、线程名。

`CallerRunsPolicy` 会把任务打回调用线程，可能拖接口；`AbortPolicy` 要有降级。IO 密集用更大 max + 有界队列；CPU 密集约核数。

ThreadLocal：线程池里线程长期活着，value 强引用 → 泄漏；异步线程不 `remove` 会串用户。必须 `try/finally remove`。

虚拟线程（JDK21）：适合阻塞 IO 多；synchronized 可能 pinning；不要拿它当「不用再隔离线程池」的理由。CPU 忙任务仍要限制并行。

## 方案选型（轻量优先）

| 场景 | 默认 |
|------|------|
| 下单/核销 | 独立有界池，队列小，Abort+快速失败 |
| 报表/导出 | 另一个池，绝不能和下单共用 |
| 定时任务 | 单独池 |
| 虚拟线程 | 新项目 IO 可试；线程池隔离仍要 |

## 默认方案

```java
ThreadPoolExecutor orderPool = new ThreadPoolExecutor(
    8, 16, 60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(200),
    new ThreadFactoryBuilder().setNameFormat("order-%d").build(),
    new ThreadPoolExecutor.AbortPolicy());
```

```java
try {
    UserContext.set(uid);
    handle();
} finally {
    UserContext.remove(); // 线程池必须
}
```

## 反例

错误：`Executors.newFixedThreadPool(8)` 跑导出。
正确：有界队列 + 独立池。

错误：网关线程里 set ThreadLocal，丢给线程池不传、不 remove。
正确：任务入参显式传 userId，或装饰器拷贝并 finally 清。

错误：虚拟线程无限提交 CPU 计算。
正确：信号量/信号限制并行。

## 验证

- 报表打满时下单 RT 不明显上升。
- 压测队列满走拒绝策略，有监控。
- 线程池任务后 ThreadLocal 为空。

## 评审清单

- [ ] 不是 Executors 无界包装
- [ ] 核心与非核心池隔离
- [ ] ThreadLocal 有 remove
---
