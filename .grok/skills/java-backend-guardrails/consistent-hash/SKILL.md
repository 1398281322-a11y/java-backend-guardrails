---
name: consistent-hash
description: Use when discussing 一致性哈希, virtual nodes, load balancing sticky keys, or comparing hash-mod vs consistent hash. Do not confuse with Redis Cluster hash slots (backend-redis-ops) or DB sharding keys (backend-sharding).
---

# 一致性哈希

## When to Invoke

缓存节点扩缩容、按 key 粘滞到实例、面试问「加机器要不要全量迁移」。

## When NOT

选 MySQL 分片键 → `sharding`。Redis Cluster 槽位 → `redis-ops`。两者都 **不是** 经典一致性哈希。

## 风险（面试考点）

普通 `hash(key) % N`：N 从 8 变 16，**几乎所有 key 换节点** → 缓存雪崩式回源。

一致性哈希：key 和节点都映射到环上，顺时针找最近节点。加一台只迁移邻居一段，约 `1/N`。

没有虚拟节点时，真实节点少会严重不均。生产用 **虚拟节点**（一台机器挂很多环上点）。

Redis Cluster：`CRC16(key) % 16384` 固定槽，槽再分配给节点。扩容是 **迁移槽**，算法不是一致性哈希。面试把两者说成一回事是错的。

## 方案选型（轻量优先）

| 问题 | 默认 |
|------|------|
| Redis 官方集群 | 槽位，不要自己造一致性哈希客户端（除非 Codis 老方案） |
| 本地缓存节点 / 某些负载 | 一致性哈希 + 虚拟节点 |
| DB 分片 | 取模或范围；扩容用双写/迁移，不靠环 |

## 默认方案

需要自研粘滞时：虚拟节点数足够（每物理节点数百）、哈希用稳定算法（如 Murmur）、节点摘除要摘它的全部虚节点。

哈希环不解决「节点内存大小不同」——用权重虚节点数量表达容量。

## 反例

错误：Redis Cluster 16384 槽 = 一致性哈希。
正确：固定槽表。

错误：DB `user_id % 8` 改 `% 16` 当一致性哈希无感扩容。
正确：取模扩容要迁移约一半或更多，必须有迁移方案。

## 验证

- 加 1 个缓存节点，回源只升高对应那一段 key，不是全量 miss。
- 节点分布用虚节点后标准差可接受。

## 评审清单

- [ ] 没把 Cluster 槽位说成一致性哈希
- [ ] 没把 DB 取模扩容说成无感
- [ ] 自研环有虚拟节点
---
