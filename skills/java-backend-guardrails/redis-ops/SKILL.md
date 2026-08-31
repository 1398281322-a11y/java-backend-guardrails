---
name: redis-ops
description: Use when operating Redis persistence (RDB/AOF), Sentinel vs Cluster, 大 key, 热 key, fork blocking, 16384 slots. Do not use for cache penetration/breakdown/avalanche (backend-cache-trap) or distributed locks (backend-distributed-lock).
---

# Redis 运行与集群

## When to Invoke

持久化怎么选、哨兵还是 Cluster、大 key 把实例打满、热 key 打爆网卡、槽位迁移、fork 引起延迟。

## When NOT

穿透/击穿/雪崩/删缓存 → `cache-trap`。分布式锁 → `distributed-lock`。一致性哈希算法本身 → `consistent-hash`（Cluster **不是**一致性哈希）。

## 风险（面试考点）

- 命令执行基本仍是 **单线程**；Redis 6+ 的多线程是 **网络 IO**，不是并行执行 SET。
- **RDB**：恢复快，丢上次快照后的写。**AOF everysec**：约 1 秒窗口。**混合**（AOF 带 RDB preamble）生产常见。缓存可关持久化；当账本就不要只用 Redis。
- **哨兵**：一个主的故障转移。**Cluster**：16384 slot，CRC16(key) 分片 + 每槽有副本。跨 slot 多 key 要 `{hash tag}`。
- 大 key：`DEL`/`HGETALL` 阻塞；用 `UNLINK`、`SCAN`、拆 key。
- 热 key：单 slot 单实例打满；本地缓存或拆 key。
- `BGSAVE`/`AOF rewrite` 要 **fork**，内存大时主线程卡顿。

## 方案选型（轻量优先）

| 用途 | 默认 |
|------|------|
| 纯缓存 | 可无持久化；淘汰 `allkeys-lru`；挂了走 DB+限流 |
| 要少丢的缓存 | AOF everysec + 副本 |
| 数据量单机够 | 哨兵 |
| 要分片 | Cluster；key 设计带 hash tag |

## 默认方案

```text
prod:{id}           # 普通缓存 key
lock:{orderId}      # 锁，别和超大 hash 混在一起
{user:123}:cart     # 同用户多 key 要同槽才用 tag
```

大 key：Hash 按 field 拆；List 分页；删除 `UNLINK`。禁用生产 `KEYS *`。

热 key：Caffeine 本地短 TTL + Redis；或把库存拆成 `stock:{sku}:0..15` 再汇总（见 `hot-account`）。

## 反例

错误：Cluster 用了一致性哈希所以加节点几乎不迁移。
正确：Cluster 是槽位映射。加节点要迁移部分 slot。一致性哈希是另一算法。

错误：缓存实例 `noeviction` 写满报错，业务当库。
正确：缓存用淘汰；不能丢的数据在 MySQL。

错误：`HGETALL` 百万 field 的详情。
正确：拆，或只取需要的 field。

## 验证

- `MEMORY USAGE` / `--bigkeys` 无异常大对象。
- 主挂，哨兵或 Cluster 切主，缓存可恢复；核心写不依赖它。
- fork 期间监控延迟，数据集过大要拆实例。

## 评审清单

- [ ] 未把 Redis 当订单账本
- [ ] 大 key / 热 key 有拆分或本地缓存
- [ ] Cluster 与哨兵没混为一谈
- [ ] 未把穿透击穿写进本文件
---
