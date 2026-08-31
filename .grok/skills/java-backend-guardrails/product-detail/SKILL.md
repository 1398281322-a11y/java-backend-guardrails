---
name: product-detail
description: Use when designing 商品详情页, SKU specs, 静态化, CDN, 商详缓存与库存分离. For penetration/breakdown/avalanche read backend-cache-trap. For flash-sale pages read backend-seckill.
---

# 商品详情页

## When to Invoke

商详、SKU 规格、详情高 QPS、静态化、详情缓存脏了。电商面试：「详情页怎么抗高并发」。

## When NOT

缓存三件套细节 → `cache-trap`。秒杀活动页 → `seckill`。购物车 → `cart`。OSS 直传/STS/私有桶 → `file-oss`。

## 风险（面试考点）

详情 QPS 高但变更少：必须 **页面静态/CDN + 多级缓存**。把库存、价格强绑在同一超长 TTL 缓存里，会卖过期价或显示有货实际无货。

SKU：规格（颜色/尺码）组合爆炸，不要为每个组合生成一张详情 HTML。SPU 静态，SKU 价与库存动态接口。

## 方案选型（轻量优先）

| 数据 | 默认 |
|------|------|
| 标题、图文、规格定义 | CDN / 静态化，TTL 长，发布时刷新 |
| 价格、活动标签 | Redis 短 TTL 或主动删 |
| 可售库存 | 不进长缓存；短 TTL 或单独接口，允许最终一致 |
| SKU 列表 | `spu_id` 一次取出，前端选规格 |

读路径：CDN → 本地 Caffeine → Redis → DB。写：改 DB 后删对应 key（`cache-trap`）。

## 默认方案

```text
GET /spu/{id}          可缓存
GET /spu/{id}/live     价格+可售，Cache-Control 短或协商缓存
key: spu:{id}:info
key: spu:{id}:live
```

SKU 表：`uk(spu_id, spec_md5)`，库存在 sku 行或库存中心，不要复制进详情 JSON 长期存。

运营改图：发版本号或删 CDN。不要指望等 TTL。

## 反例

错误：详情 JSON 里带 stock=1234，缓存 1 小时。
正确：库存单独短缓存或实时。

错误：每个 SKU 一张静态页 × 上万规格。
正确：SPU 一页 + SKU 接口。

错误：秒杀也走普通商详集群。
正确：秒杀页静态独立，见 `seckill`。

## 验证

- 压测商详，DB QPS 远低于入口。
- 改价后详情 live 接口很快变；图文允许稍延迟。
- 无货 SKU 在 live 接口体现，静态页仍可打开。

## 评审清单

- [ ] 图文与价格库存分离缓存
- [ ] 写后删缓存或版本号
- [ ] 未把秒杀流量接在普通商详上
---
