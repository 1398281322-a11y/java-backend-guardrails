---
name: schedule-job
description: Use when implementing 定时任务, @Scheduled, cron, XXL-JOB, Quartz, ShedLock, or cluster duplicate job execution. Do not use for 延迟关单 (backend-delay-job). Jobs must be idempotent.
---

# 定时任务 / 调度

## When to Invoke

日切对账、账期结算批、补偿扫描、报表、面试「@Scheduled 集群会怎样」「Timer 和线程池调度区别」「XXL-JOB 分片」。

## When NOT

下单 N 分钟关单、券过期这类 **一次性延迟** → `delay-job`（发延迟消息，禁止扫全表）。立刻异步解耦 → `mq-reliability`。任务线程池参数细节 → `thread-pool`。

## 风险（面试考点）

`java.util.Timer`：**单线程**；一个任务卡住后面全延迟；未捕获异常 **杀死 Timer 线程**，后续任务全停；基于绝对时钟，改系统时间会乱。JDK 文档写明用 `ScheduledThreadPoolExecutor` 替代。

Spring `@Scheduled` 默认 **线程池 size=1**（`spring.task.scheduling.pool.size`，Boot 2.1+）。一个慢 Job 堵住所有 cron。同一方法在单线程上不会重叠；**多实例每人一份调度器，任务会跑 N 遍**——这不是 Bug。

ShedLock / Redis 锁只能保证「同时最多一个」；`lockAtMostFor` 到期会把锁交给别人，长任务必须大于最坏耗时或可续期。锁不是幂等：任务仍要按业务键可重入。

XXL-JOB（官方能力，别背错）：

- **调度过期**：忽略（推荐）/ 立即补偿一次。补偿可能双跑，业务必须幂等。
- **阻塞处理**：单机串行（默认排队）；丢弃后续；覆盖之前（会打断正在跑的）。
- **分片广播**：所有执行器都触发，用 `shardIndex/shardTotal` 切数据；不是「只跑一台」。
- 超时会中断任务，循环里要响应中断。失败重试次数 >0 时当重复执行。

Cron 时区写死 `Asia/Shanghai`。Spring 是 6 域（含秒）。`fixedRate` 按开始时间对齐，跑超周期会积压；`fixedDelay` 从上一次 **结束** 再等。

禁止用定时任务当关单主路径（见 `delay-job`）。补偿扫描只捞小窗口、带索引、LIMIT 批次。

## 方案选型（轻量优先）

| 场景 | 默认 |
|------|------|
| 单实例、几个轻量 Job | `@Scheduled` + 调大 scheduling pool |
| 多实例、Job 少 | `@Scheduled` + ShedLock（或 Redisson 锁） |
| 多实例、要控制台/分片/告警 | XXL-JOB |
| 一次性延迟 | 不走本文件 |

```java
@Scheduled(cron = "0 0 2 * * ?", zone = "Asia/Shanghai")
@SchedulerLock(name = "settleDaily", lockAtMostFor = "50m", lockAtLeastFor = "1m")
public void settle() {
    // 批次 LIMIT；幂等 uk；失败可重跑
}
```

XXL-JOB 大批量：

```java
@XxlJob("verifyCompensate")
public void compensate() {
    int idx = XxlJobHelper.getShardIndex();
    int total = XxlJobHelper.getShardTotal();
    // WHERE MOD(id, total)=idx 或按号段，禁止每台全表扫
}
```

## 反例

错误：三台机器 `@Scheduled` 无锁，日结打款打三次。
正确：ShedLock / XXL-JOB 路由一台，且打款单号唯一。

错误：用 Timer 跑对账，任务抛 NPE 后全站定时死。
正确：调度线程池；单任务异常不能杀调度器。

错误：每分钟 `SELECT * FROM t_order WHERE status=CREATED` 关单。
正确：下单发延迟消息。

错误：Job 跑 40 分钟，锁 10 分钟，第二台接着跑同一批。
正确：`lockAtMostFor` > 超时；或任务可安全重叠。

## 验证

- 两实例同时到点，业务副作用只有一次。
- 人为让 Job 跑超周期：串行排队或丢弃符合配置，不双写资金。
- 分片广播两台，数据无交集、无漏。

## 评审清单

- [ ] 不是延迟关单误用扫表
- [ ] 多实例有互斥或调度中心；任务本身幂等
- [ ] 调度池 >1 或确认 Job 都很短
- [ ] cron 带时区；长任务有超时/阻塞策略
---
