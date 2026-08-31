---
name: jvm-prod
description: Use when diagnosing Full GC, OOM, Metaspace, direct memory, heap dump, jstat, 频繁 GC, CPU 飙高 from JVM. Do not use for business oversell or Redis. Skip ordinary CRUD with no latency/GC symptoms.
---

# JVM 生产排查

## When to Invoke

RT 周期性尖刺、Full GC、OOM、Metaspace 涨、堆外内存、容器里 RSS 远大于 `-Xmx`。

## When NOT

业务超卖、SQL 慢（先看 SQL）。不要一上来「把堆调到 16G」。

## 风险（面试考点）

堆：新生代（Eden + Survivor）+ 老年代。**Metaspace**（Java 8+）在堆外，类加载泄漏会涨。直接内存（Netty/NIO）不在堆里，`-Xmx` 管不到。

OOM 类型要分：`Java heap space`、`Metaspace`、`Direct buffer memory`、`unable to create native thread`。处理不同。

Full GC 不是先加内存。路径：指标 → GC 日志 → 是谁撑堆 → 引用链。容器要设 `-XX:+UseContainerSupport`，别让 JVM 看见整机内存。

G1 常见；ZGC 低停顿但对 JDK 版本和调参有要求。面试别背收集器名单，要能说「看停顿目标和版本」。

## 方案选型（轻量优先）

| 症状 | 先做什么 |
|------|----------|
| 周期性 STW | 开 GC 日志，看 Young/Full/Mixed 频率与 pause |
| heap OOM | dump（低峰 `jcmd GC.heap_dump`），MAT 看 Dominator |
| Metaspace OOM | 查重复加载、热部署、`cglib` 无界 |
| 线程创建失败 | 线程池泄漏、ulimit、栈太大 |
| CPU 高 | 先 `top`/`async-profiler`，再决定是 GC 还是热点方法 |

## 默认方案

```text
-Xms / -Xmx 相同，避免运行时扩堆
-XX:+UseG1GC
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=20M
容器：MaxRAMPercentage，不要盲目 Xmx=宿主机
```

排查顺序：QPS/RT → GC pause/count → 老年代占比 → dump 或 profiler。生产 `jmap -dump:live` 会 STW，能 `jcmd` 就 `jcmd`。

## 反例

错误：Full GC 就重启 + 加大堆。
正确：先看是泄漏还是高峰。泄漏加大只会更晚爆。

错误：OOM 一律 dump 堆。
正确：Metaspace/直接内存 dump 堆看不见根因。

错误：线程池用无界队列（见 `thread-pool`）导致 heap OOM，却去调 G1。

## 验证

- GC 日志能对应上 RT 尖刺时间。
- dump 里能指出占内存的业务对象类型。
- 容器 limit 与 JVM 堆+元空间+直接内存预算匹配。

## 评审清单

- [ ] 分清 heap / metaspace / direct / thread
- [ ] 有 GC 日志，不是只靠重启
- [ ] 未把业务锁问题当 GC 问题
---
