---
name: perf-api
description: Use when optimizing 接口性能, N+1 SQL/RPC, 循环远程调用, Hikari 连接池, 批量写入, or 同步转异步. Do not use for GC (backend-jvm-prod), EXPLAIN (backend-mysql-index), or incident 排查顺序 (backend-observability).
---

# 接口性能

## When to Invoke

接口 RT 高、QPS 上不去、面试「怎么优化一个接口」「N+1」「连接池怎么设」「循环里调 Feign」。已经知道慢在 **业务代码路径**（多次 IO、池配错、同步多余活）。

## When NOT

还没定位瓶颈 → `observability`（TraceId、P99、先测再改）。单条 SQL 慢 / EXPLAIN → `mysql-index`。Full GC / 堆 → `jvm-prod`。线程池无界 → `thread-pool`。Feign 超时数值 → `rpc-remote`。缓存击穿 → `cache-trap`。

## 风险（面试考点）

**先测后改。** 没有火焰图/Trace 就加缓存、加线程，是面试扣分项。

N+1：列表 20 条，循环里再查子表或调一次 Feign = 1+20 次网络。MyBatis `<collection select="...">` 默认就是 N+1。嵌套结果 JOIN 或 **两次查询 + 内存拼**（主列表 + `IN` 子表）次数固定。

循环 `insert` / `save`：MyBatis-Plus `saveBatch` 仍可能只是 JDBC batch；MySQL 要 `rewriteBatchedStatements=true` 才会合成多值 INSERT。`foreach` 一条超大 SQL 会撞 `max_allowed_packet`，按 200～1000 行切开。

线程池 200、Hikari **默认 10**：请求线程堵在 `getConnection`，报 `Connection is not available, request timed out`。Hikari 作者：**小池更好**，起点约 `(CPU核×2)+盘`（常 10～20），不是跟 Tomcat 线程数对齐。一个线程持有多连接时按 `Tn×(Cm-1)+1` 防死锁。

`maxLifetime` 必须比 MySQL `wait_timeout`（及中间 NAT/LB 超时）**至少短 30s**，否则池里是已死连接，表现为偶发 Communications link failure。

用户请求线程里做：发短信、算报表、写一堆日志 JSON、`SELECT *` 回 50 列、深 `offset` 翻页。用户不等的活丢 MQ。热路径日志不要 `toJson(huge)`。

## 方案选型（轻量优先）

| 症状 | 默认 |
|------|------|
| 循环查库 / 循环 Feign | 批量 `IN` / 批量 RPC，内存 Map 组装 |
| 循环 insert | `saveBatch` + `rewriteBatchedStatements` + 分批 |
| 获取连接超时 | 查泄漏（事务/连接未关）；池不是先加到 100 |
| 用户 RT 含非关键 IO | 同步只做成功路径，其余 MQ |
| IN 列表过大 | 分批，不要上万 ID 一条 SQL |

```java
List<Order> orders = orderMapper.selectPage(uid, cursor, 20);
List<Long> ids = orders.stream().map(Order::getId).toList();
Map<Long, List<Item>> items = itemMapper.selectByOrderIds(ids).stream()
    .collect(Collectors.groupingBy(Item::getOrderId));
orders.forEach(o -> o.setItems(items.getOrDefault(o.getId(), List.of())));
```

```yaml
spring.datasource.hikari:
  maximum-pool-size: 10
  connection-timeout: 30000
  max-lifetime: 1800000   # < wait_timeout - 30s
  leak-detection-threshold: 20000
```

JDBC URL 加 `rewriteBatchedStatements=true`（仅当确有批量写）。

## 反例

错误：for 每条订单 `feign.getMerchant(id)`。
正确：一次 `listByIds`，或下单时冗余必要字段。

错误：接口慢就把 Hikari 调成 200。
正确：先看是 SQL、泄漏、还是下游 RT；库侧连接过多会更慢。

错误：列表接口 `SELECT *` + 循环补用户名 + 同步写操作日志到 DB。
正确：必要列、批量补、日志异步。

错误：没压测就上本地缓存「肯定快」。
正确：Trace 看时间花在哪一段。

## 验证

- 一页 20 条，SQL/RPC 次数不随 N 线性涨（大约 2～3 次）。
- 连接泄漏检测能抓到未关闭的连接。
- 压测拐点：再加池或线程，RT 上升而不是 QPS 上升则停。

## 评审清单

- [ ] 无循环 IO（SQL/RPC/Redis）
- [ ] 批量写真正 batch，且分批
- [ ] 连接池小于 wait_timeout；泄漏检测打开
- [ ] 非关键路径不在用户线程
---
