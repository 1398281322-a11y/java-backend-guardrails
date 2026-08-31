---
name: java-backend-guardrails
description: Use when designing, implementing, or reviewing Java microservice or 电商/文旅/支付业务 involving 订单, 支付, 分账, 结算, OSS, 文件上传, STS, 直传, SSRF, 财务统计, GMV, 接口规范, 接口安全, 定时任务, XXL-JOB, 接口性能, N+1, 连接池, 防重放, 水平越权, 退款, 核销, 库存, 秒杀, 购物车, 优惠券, Redis, RocketMQ, 幂等, 高并发, 分布式事务, 限流, 缓存, 分库分表, 索引, 容灾, 远程调用, JVM, 对账, or 账期. Skip heavy concurrency skills for ordinary single-table CRUD.
---

# Java Backend Guardrails（总控）

定位：把面试考点变成写代码时的风险扫描，不是背题，也不是自动写出完美代码。
技术栈默认：Java 微服务 + Spring Boot + MyBatis-Plus + Redisson + RocketMQ。

## 硬规则

1. **先读本文件，再按需加载子 skill。** 禁止把所有子 skill 读进上下文。
2. **最多加载 2–4 个子 skill。** 用下面的路由表匹配，取相关度最高的。
3. **普通单表 CRUD（无钱、无库存、无回调、无跨服务写、无公开高 QPS）→ 不加载任何高并发子 skill。** 只做安全与表设计里的轻量检查（`#{}`、登录鉴权、必要唯一约束）。
4. **先给轻量方案。** 禁止无脑上 TCC、Seata、分库分表、全局分布式锁。升级重方案必须写清触发条件。
5. **skill 给风险 + 对比 + 默认路径，取舍交给人。** 不要假装只有一种正确答案。
6. **写完代码后按已加载子 skill 的评审清单再扫一遍。**

## 工作流

按当前阶段只加载该阶段相关的子 skill：

| 阶段 | 做什么 | 典型加载 |
|------|--------|----------|
| 需求 / 设计 | 标风险、定默认方案、标升级条件 | 业务命中的 2–4 个 |
| 建表 | 唯一约束、状态机、版本号、索引 | `table-design` + `mysql-index`（需要分片才加 `sharding`） |
| 写接口 | 按默认方案落代码 | 同需求阶段 |
| 代码评审 / 存量扫描 | 对照子 skill 反例和清单 | 同需求阶段 |

两种触发：

- **语义自动：** 识别到下方关键词就加载对应子 skill。
- **显式：** 用户点名某个 skill（如「用幂等 skill」）时只读那一个。

## 路由表（只读命中的文件）

路径相对本 skill 根目录。索引和分片是两个文件，禁止当作同一个问题处理。

| 子 skill | 读这个 | 命中关键词 / 症状 |
|----------|--------|-------------------|
| 幂等 | [idempotent/SKILL.md](idempotent/SKILL.md) | 重复提交、重复回调、重复消费、重试、支付回调、退款回调、核销、防重 |
| 超卖 / 扣库存 | [concurrency-sell/SKILL.md](concurrency-sell/SKILL.md) | 库存、超卖、扣减、名额、余票、分佣扣减、核销占名额 |
| 分布式事务 | [distributed-transaction/SKILL.md](distributed-transaction/SKILL.md) | 跨服务写、下单后发券、支付后分佣、核销后结算、本地消息表、TCC、Seata |
| 缓存陷阱 | [cache-trap/SKILL.md](cache-trap/SKILL.md) | Redis 缓存、穿透、击穿、雪崩、缓存一致性、热点 key |
| 限流熔断 | [rate-limit/SKILL.md](rate-limit/SKILL.md) | 限流、熔断、降级、打爆、Sentinel、网关保护 |
| 远程调用 | [rpc-remote/SKILL.md](rpc-remote/SKILL.md) | Feign、RPC、超时、重试、仓壁、第三方 HTTP |
| 消息可靠 | [mq-reliability/SKILL.md](mq-reliability/SKILL.md) | RocketMQ、消息丢失、重复消费、顺序、积压、死信、延迟消息 |
| 乐观锁 | [optimistic-lock/SKILL.md](optimistic-lock/SKILL.md) | 乐观锁、@Version、CAS、版本号、丢失更新 |
| 分布式锁 | [distributed-lock/SKILL.md](distributed-lock/SKILL.md) | Redisson、SET NX、分布式锁、看门狗 |
| 索引 | [mysql-index/SKILL.md](mysql-index/SKILL.md) | 索引、慢 SQL、EXPLAIN、最左前缀、回表 |
| 分库分表 | [sharding/SKILL.md](sharding/SKILL.md) | 分库、分表、分片、ShardingSphere、跨库 join |
| 表设计 | [table-design/SKILL.md](table-design/SKILL.md) | 建表、大表、状态机字段、归档、分区 |
| 安全 | [safe-check/SKILL.md](safe-check/SKILL.md) | SQL 注入、越权、签名、回调伪造、水平越权 |
| 容灾 | [ha-dr/SKILL.md](ha-dr/SKILL.md) | 容灾、高可用、RPO、RTO、主从切换、降级兜底 |
| 分布式 ID | [distributed-id/SKILL.md](distributed-id/SKILL.md) | 雪花 ID、订单号、全局唯一、ASSIGN_ID |
| MVCC / 间隙锁 | [mysql-mvcc/SKILL.md](mysql-mvcc/SKILL.md) | 幻读、RR、当前读、FOR UPDATE、死锁、间隙锁 |
| 读写分离 | [mysql-replication/SKILL.md](mysql-replication/SKILL.md) | 主从、binlog、GTID、半同步、写后读不到、从库延迟 |
| Redis 运维 | [redis-ops/SKILL.md](redis-ops/SKILL.md) | RDB、AOF、哨兵、Cluster、16384 槽、大 key、fork |
| JVM 排查 | [jvm-prod/SKILL.md](jvm-prod/SKILL.md) | OOM、Full GC、Metaspace、堆 dump、GC 日志 |
| 线程池 | [thread-pool/SKILL.md](thread-pool/SKILL.md) | 无界队列、拒绝策略、ThreadLocal 泄漏、虚拟线程 |
| 延迟任务 | [delay-job/SKILL.md](delay-job/SKILL.md) | 延迟关单、超时未支付、delayTimeLevel、时间轮 |
| 定时任务 | [schedule-job/SKILL.md](schedule-job/SKILL.md) | @Scheduled、cron、XXL-JOB、集群重复执行、ShedLock |
| 接口性能 | [perf-api/SKILL.md](perf-api/SKILL.md) | 接口慢、N+1、循环 Feign、Hikari、批量 insert |
| 热点账户 | [hot-account/SKILL.md](hot-account/SKILL.md) | 热点行、爆款 SKU、钱包一行、分段库存 |
| 对账 | [reconciliation/SKILL.md](reconciliation/SKILL.md) | 对账、差错、渠道对账单、分佣核对 |
| 分布式会话 | [distributed-session/SKILL.md](distributed-session/SKILL.md) | Session、JWT、SSO、粘滞、踢人 |
| 可观测 | [observability/SKILL.md](observability/SKILL.md) | TraceId、链路追踪、P99、线上排查 |
| 一致性哈希 | [consistent-hash/SKILL.md](consistent-hash/SKILL.md) | 一致性哈希、虚拟节点、加机器迁移 |
| 异地多活 | [multi-live/SKILL.md](multi-live/SKILL.md) | 异地多活、单元化、双活、写冲突 |
| 秒杀 | [seckill/SKILL.md](seckill/SKILL.md) | 秒杀、抢购、限时抢、活动预热、Lua 预扣 |
| 订单状态机 | [order-state/SKILL.md](order-state/SKILL.md) | 待支付、已支付、发货、非法跳转、正向履约 |
| 库存预占 | [inventory-occupy/SKILL.md](inventory-occupy/SKILL.md) | 下单扣还是支付扣、锁定库存、可售库存、预占释放 |
| 优惠券 | [promo-coupon/SKILL.md](promo-coupon/SKILL.md) | 领券、超发、满减、叠加、前端改价 |
| 购物车 | [cart/SKILL.md](cart/SKILL.md) | 加购、未登录购物车、登录合并、结算 |
| 售后逆向 | [refund-aftersale/SKILL.md](refund-aftersale/SKILL.md) | 仅退款、退货退款、退库存、退券 |
| 拆单 | [split-order/SKILL.md](split-order/SKILL.md) | 多商家、多仓、父单子单、门店拆单 |
| 商品详情 | [product-detail/SKILL.md](product-detail/SKILL.md) | 商详、静态化、CDN、SKU 规格、价库存分离 |
| 支付回调 | [pay-notify/SKILL.md](pay-notify/SKILL.md) | 异步通知、验签、金额分、notify SUCCESS |
| 支付查单 | [pay-query/SKILL.md](pay-query/SKILL.md) | 回调丢失、超时查单、PAYING 补偿、关单前 query |
| 重复支付 | [pay-duplicate/SKILL.md](pay-duplicate/SKILL.md) | 两笔成功、关单撞车、rows=0、多付退余 |
| 支付单 | [pay-trade/SKILL.md](pay-trade/SKILL.md) | 预下单、unified order、支付单状态、二维码过期 |
| 渠道退款 | [pay-refund-channel/SKILL.md](pay-refund-channel/SKILL.md) | 退款 API、refund_no、部分退、退款查单 |
| 支付分账 | [pay-split-account/SKILL.md](pay-split-account/SKILL.md) | 渠道分账、延迟分账、抽佣、微信分账 |
| 接口规范 | [api-contract/SKILL.md](api-contract/SKILL.md) | REST、状态码、错误码、版本、金额分、时区、分页 |
| 接口安全 | [api-security/SKILL.md](api-security/SKILL.md) | HMAC、防重放、nonce、CORS、脱敏、JWT |
| 财务统计 | [finance-stats/SKILL.md](finance-stats/SKILL.md) | GMV、实收、结算GMV、口径、切日 |
| 商家结算 | [finance-settlement/SKILL.md](finance-settlement/SKILL.md) | 账期、结算单、待结算、打款 |
| 财务流水 | [finance-ledger/SKILL.md](finance-ledger/SKILL.md) | 记账、余额、借贷、日切、冲正 |
| 对象存储 | [file-oss/SKILL.md](file-oss/SKILL.md) | OSS、S3、MinIO、STS、直传、预签名、分片、CDN、私有桶 |
| 上传安全 | [file-upload/SKILL.md](file-upload/SKILL.md) | magic bytes、路径穿越、SVG、Zip Slip、SSRF |

组合场景组合（仍然 ≤4）：

| 业务 | 加载 |
|------|------|
| 支付 / 退款回调 | `pay-notify` + `idempotent`（需要再加 `pay-duplicate`） |
| 支付超时 / 丢回调 | 只加载 `pay-query` |
| 用户付了两次 / 关单撞支付 | `pay-duplicate` + `pay-notify` |
| 接微信预下单 | `pay-trade` + `pay-notify` |
| 渠道退款 | `pay-refund-channel` + `refund-aftersale`（业务售后才加后者） |
| 多门店分账 | `pay-split-account`（渠道划款）需要账单再加 `finance-settlement` |
| 新接口约定 | 只加载 `api-contract` |
| 签名/防重放 | `api-security`（越权 SQL 再加 `safe-check`） |
| GMV/报表口径 | 只加载 `finance-stats` |
| 商家账期打款 | `finance-settlement` |
| 余额账户 | `finance-ledger` |
| 核销 | `idempotent` + `concurrency-sell` + `distributed-transaction` + `safe-check` |
| 分佣 / 结算 | `idempotent` + `distributed-transaction` + `optimistic-lock` |
| 下单扣库存 | `concurrency-sell` + `idempotent` + `mq-reliability` |
| 商品详情高 QPS | `cache-trap` + `rate-limit` |
| 慢查询 / 建索引 | 只加载 `mysql-index` |
| 数据量要拆库 | 只加载 `sharding`（先读 `table-design` 确认还不到拆） |
| 第三方 HTTP | `rpc-remote` + `idempotent` + `safe-check` |
| 超时关单 | `delay-job` + `concurrency-sell` + `idempotent` |
| 日批对账/结算 Job | `schedule-job` + `idempotent` |
| 接口慢未定位 | 只加载 `observability` |
| 已定位 N+1/循环 RPC/连接池 | 只加载 `perf-api` |
| 写完立刻读空 | 只加载 `mysql-replication` |
| 爆款单行排队 | `hot-account` + `concurrency-sell` |
| 支付/分佣错账 | `reconciliation` + `idempotent` |
| Full GC / OOM | 只加载 `jvm-prod`（线程池无界再加 `thread-pool`） |
| 秒杀入口 | 只加载 `seckill`（需要扣减细节再加 `concurrency-sell`） |
| 下单预占 | `inventory-occupy` + `order-state` + `delay-job` |
| 领券/用券 | `promo-coupon` + `idempotent` |
| 购物车 | 只加载 `cart`（结算创单再加 `order-state`） |
| 售后退款 | `refund-aftersale` + `idempotent` |
| 跨店结算 | `split-order` + `inventory-occupy` |
| 商详高 QPS | `product-detail` + `cache-trap` |
| 登录态集群 | 只加载 `distributed-session` |
| 异地双写 | 先 `ha-dr`；用户坚持多活再 `multi-live` |
| OSS 直传/私有下载 | 只加载 `file-oss` |
| 上传校验/URL 拉图 | `file-upload`（直传流程再加 `file-oss`） |

## 轻量默认（未加载子 skill 时也遵守）

- 写操作：数据库唯一约束 / 条件更新，优先于分布式锁。
- 重试：先幂等，再重试。超时当未知结果，禁止当失败直接再做一笔。
- 缓存：缓存不是数据源。写库成功后再删缓存。
- 消息：按至少一次投递设计，消费者必须幂等。
- 远程调用：必须设超时。非幂等接口禁止自动重试。
- 分库分表、TCC、Seata、异地多活：默认不上。
- 读写分离后，资金/核销/写后读走主库。
- Redis Cluster 槽位 ≠ 一致性哈希；索引 ≠ 分片 ≠ 主从。
- 一次性延迟 ≠ 周期性定时；接口慢先 Trace，不要先加缓存/加连接池。
- 文件不进 MySQL；OSS 密钥不进前端；用户文件名不当 object key。

## 输出形状

```
## 判定
- 场景:
- 命中关键词:
- 加载的子 skill: （≤4，写相对路径）
- 明确跳过: （如 TCC / 分库分表 / 分布式锁）

## 风险
- （只列本次相关的面试级坑）

## 方案
- 默认（轻量）:
- 何时升级:

## 落地
- 表 / 唯一键 / 状态机:
- 代码要点:

## 评审
- （按已加载子 skill 清单勾）
```

## 禁止

- 简单列表查询输出限流 + 分库分表 + TCC。
- 把索引方案写进分片 skill，或把分片方案写进索引 skill。
- 把扫表关单写进定时任务 skill（应走 `delay-job`）；把 Full GC 写进接口性能 skill。
- 把 STS/直传写进上传安全 skill，或把 magic/SSRF 写进 OSS skill。
- 用「面试标准答案」压过当前业务体量。
- 只给概念不给可落地的约束（唯一键、WHERE 条件、超时值来源）。
---
