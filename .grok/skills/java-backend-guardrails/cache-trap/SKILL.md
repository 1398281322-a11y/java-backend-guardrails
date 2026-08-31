---
name: cache-trap
description: Use when adding or debugging Redis cache, cache-aside, 穿透, 击穿, 雪崩, cache stampede, hot key, or cache-DB consistency. Do not use for Redis distributed locks (use backend-distributed-lock) or as a message queue replacement.
---

# 缓存陷阱

## When to Invoke

商品详情、活动页、配置、用户信息等读多写少走 Redis；出现击穿/穿透/雪崩/不一致。

## When NOT

分布式锁、库存扣减主账本、MQ。缓存不是数据源。

## 风险（面试考点）

| 名称 | 是什么 | 不是什么 |
|------|--------|----------|
| 穿透 | 查不存在的数据，缓存和 DB 都没有，每次打 DB | 热点 key 过期 |
| 击穿 | 一个热点 key 过期，并发打穿 DB | 大批 key 一起过期 |
| 雪崩 | 大批 key 同时过期，或 Redis 宕机，流量打到 DB | 单个 key |

另：先更新缓存再写 DB、或删缓存后再写 DB 导致把旧值填回缓存。

## 适用场景

- 商品/景点/场次详情
- 活动开关、字典
- 热点商户信息

## 方案选型（轻量优先）

读写：Cache-Aside。应用读 Redis → miss 读 DB → 回填。写：**先写 DB，再删缓存**（不是改缓存）。删失败靠 TTL 自愈，可延迟再删一次。

| 问题 | 默认 | 升级 |
|------|------|------|
| 穿透 | 参数校验 + 缓存空值短 TTL（30–120s） | 布隆过滤器挡明确不存在的 ID |
| 击穿 | 热点永不过期或逻辑过期 + 单飞（同 key 只回源一次） | 互斥锁回填 |
| 雪崩 | TTL 加随机抖动；Redis 主从/哨兵 | 本地 Caffeine 做二级；DB 限流；熔断 |
| 热点 | 本地缓存 + Redis | 拆 key、副本读 |
| 大 key | 拆分；删除用 `UNLINK` | 禁止 `KEYS` |

## 默认方案

```java
public Product get(Long id) {
    String key = "prod:" + id;
    String cached = redis.opsForValue().get(key);
    if (cached != null) {
        return cached.equals(NULL) ? null : readJson(cached);
    }
    Product db = productMapper.selectById(id);
    if (db == null) {
        redis.opsForValue().set(key, NULL, Duration.ofSeconds(60));
        return null;
    }
    int ttl = 600 + ThreadLocalRandom.current().nextInt(120);
    redis.opsForValue().set(key, writeJson(db), Duration.ofSeconds(ttl));
    return db;
}

@Transactional
public void update(Product p) {
    productMapper.updateById(p);
    redis.delete("prod:" + p.getId()); // 先 DB 后删缓存
}
```

击穿单飞（同进程）：

```java
private final ConcurrentHashMap<Long, CompletableFuture<Product>> inflight = new ConcurrentHashMap<>();

Product load(Long id) {
    return inflight.computeIfAbsent(id, k -> CompletableFuture.supplyAsync(() -> loadFromDb(k)))
        .whenComplete((r, e) -> inflight.remove(id))
        .join();
}
```

Redis 挂了：降级读 DB 前必须有限流/熔断，见 `rate-limit`。否则雪崩发生在「Redis 宕机」而不是 TTL。

## 反例

错误：写库前先删缓存。并发读把旧 DB 值填回缓存。
正确：先写 DB，再删缓存。

错误：不存在的 ID 不缓存，被刷库。
正确：空值短 TTL + 入参校验（ID>0、存在性）。

错误：一批 key 同一个 `EX 3600` 在整点一起过期。
正确：TTL + 随机抖动。

错误：把 Redis 当库存账本且无对账。
正确：库存见 `concurrency-sell`。

## 验证

- 不存在 ID 高频打，DB QPS 被空值/布隆挡住。
- 热点 key 过期瞬间，回源次数约为 1（单飞/锁）。
- 更新后读到新值（删缓存后回源），或最多一个 TTL 窗口的脏读。
- Redis 停掉，DB 有保护，不至于连接打满。

## 评审清单

- [ ] Cache-Aside：写 DB 后删缓存
- [ ] 空值或布隆防穿透
- [ ] TTL 有抖动
- [ ] 热点有单飞或逻辑过期
- [ ] 未把 Redis 当唯一账本
---
