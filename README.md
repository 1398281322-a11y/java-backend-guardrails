# Java Backend Guardrails

把 Java 后端面试里会炸的坑，收成写代码时的防御包。给人和 AI 用：需求、建表、写接口、评审时扫描风险，给出**轻量默认方案**和升级条件。

Interview-grade Java microservice pitfalls as **production checks**. For humans and agents: scan risks while designing, creating tables, writing APIs, and reviewing — with a **lightweight default** and an upgrade trigger.

**不是**背题库，也**不是**自动写出完美代码，更**不是**上来就上 TCC / Seata / 分库分表。

Not a flashcard dump, not auto-perfect code, and not a reason to reach for TCC / Seata / sharding.

- 技能页 / Page：https://skills.sh/1398281322-a11y/java-backend-guardrails/java-backend-guardrails
- 仓库 / Repo：https://github.com/1398281322-a11y/java-backend-guardrails

## 这个包解决什么 / What this pack is for

国内 Java 面试和线上事故高度重合：重复支付、超卖、回调不幂等、缓存击穿、主从延迟、扫表关单、GMV 当收入、AccessKey 进前端……

这些题如果只背「标准答案」，写代码时容易无脑上分布式锁、TCC、分库分表。本包反过来：**一个考点一个 skill**，总控按关键词最多加载 2–4 个，普通 CRUD 不加戏。

Chinese Java interviews and production incidents overlap: duplicate pay, oversell, non-idempotent callbacks, cache stampede, replica lag, scanning unpaid orders, treating GMV as revenue, AccessKey in the frontend.

Memorized “standard answers” turn into TCC and sharding in real code. This pack inverts that: **one topic per skill**, the parent loads at most 2–4, and ordinary CRUD stays light.

默认技术栈 / Default stack：

- Java 微服务 / Java microservices
- Spring Boot + MyBatis-Plus
- Redisson
- RocketMQ
- MySQL + Redis
- 业务：订单、支付、核销、结算、文旅/电商 / Orders, payments, ticket verification, settlement, travel & commerce

## 安装 / Install

### skills.sh / npx

```bash
npx skills add 1398281322-a11y/java-backend-guardrails
```

只看会装哪些、不落地：

```bash
npx skills add 1398281322-a11y/java-backend-guardrails --list
```

CLI 默认会发现总控包 `java-backend-guardrails`（子 skill 在包内，由总控路由懒加载）。

The CLI discovers the parent pack; children stay inside it and are loaded by the router.

### 拷到项目里 / Copy into a project

本仓库打开 Grok / Cursor 即可。给别的项目用时，拷目录：

```text
skills/java-backend-guardrails/   →   目标项目/.grok/skills/java-backend-guardrails/
或 / or
skills/java-backend-guardrails/   →   ~/.grok/skills/java-backend-guardrails/
```

Cursor / Claude 同类路径：`.cursor/skills/`、`.claude/skills/`、`~/.agents/skills/`（以各工具文档为准）。

### 对本机 Grok

本仓库的 `.grok/` **不进 Git**（和 `skills/` 重复）。本地开发可把 `skills/java-backend-guardrails` 拷到 `.grok/skills/`，或依赖用户目录那份。

`.grok/` is gitignored. Copy `skills/java-backend-guardrails` locally if Grok should auto-discover it in this repo.

## 怎么跟 AI 说 / How to talk to the agent

语义触发（推荐）：直接说业务词，总控会路由。

Say the business words; the parent routes.

| 你说 / You say | 通常加载 / Typically loads |
|----------------|----------------------------|
| 支付回调、微信 notify | `pay-notify` + `idempotent` |
| 超时关单、30 分钟未支付 | `delay-job` + `concurrency-sell` + `idempotent` |
| 核销 | `idempotent` + `concurrency-sell` + `distributed-transaction` + `safe-check` |
| 慢 SQL、建索引 | 只 `mysql-index` |
| 今日 GMV | 只 `finance-stats` |
| OSS 直传、STS | 只 `file-oss` |
| 上传防 webshell、从 URL 拉图 | `file-upload` |

显式指定：

```text
用幂等 skill 看这段回调
用 pay-query 看支付超时
```

Name a skill: “use the idempotent skill on this callback”.

不要对列表查询说「按高并发标准做」。普通单表 CRUD 不应甩出限流 + 分库 + TCC。

Do not ask for “high-concurrency standard” on a list API. Ordinary CRUD should not get rate-limit + sharding + TCC.

## 硬规则 / Hard rules

人和 AI 都遵守：

1. **先读总控，再按需加载子 skill。** 禁止一次灌完全部。Read the parent, then load children. Never dump every file.
2. **最多 2–4 个子 skill。** At most 2–4 children.
3. **无钱、无库存、无回调、无跨服务写、无公开高 QPS → 不加载高并发 skill。** 只做 `#{}`、登录鉴权、必要唯一约束。Skip heavy skills for plain CRUD.
4. **先轻量。** TCC / Seata / 分库分表 / 全局锁必须写清升级条件。Lightweight first; heavy tools need a trigger.
5. **给风险 + 对比 + 默认路径，取舍交给人。** Risks, trade-offs, default path — you decide.
6. **写完按已加载 skill 的评审清单再扫一遍。** Re-scan with the loaded checklists.

## 工作流 / Workflow

| 阶段 / Stage | 做什么 | 典型加载 |
|--------------|--------|----------|
| 需求 / 设计 | 标风险、定默认、标升级条件 | 业务命中的 2–4 个 |
| 建表 | 唯一约束、状态机、版本号、索引 | `table-design` + `mysql-index`（真要拆才加 `sharding`） |
| 写接口 | 按默认方案落代码 | 同需求阶段 |
| 评审 / 存量扫描 | 对照反例和清单 | 同需求阶段 |

## AI 输出格式 / Agent output shape

```text
## 判定
- 场景:
- 命中关键词:
- 加载的子 skill: （≤4）
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

## 轻量默认 / Defaults even without a child skill

- 写操作：唯一约束 / 条件更新，优先于分布式锁。Unique keys / conditional updates beat locks.
- 重试：先幂等。超时当**未知**，禁止当失败再开一笔。Timeout = unknown, not failure.
- 缓存：不是数据源。写库成功后再删缓存。Cache is not the source of truth.
- 消息：至少一次投递，消费者必须幂等。At-least-once + consumer idempotency.
- 远程调用：必须超时。非幂等写禁止自动重试。Always set timeouts.
- 分库分表、TCC、Seata、异地多活：默认不上。Off by default.
- 读写分离后，资金 / 核销 / 写后读走主库。Money and read-your-writes go to primary.
- 文件不进 MySQL；OSS 密钥不进前端。Files stay in object storage; secrets stay off the client.

## 子 skill 目录 / Child catalog

一个考点一个文件。路径都在 `skills/java-backend-guardrails/<目录>/SKILL.md`。

One topic per file. Paths: `skills/java-backend-guardrails/<dir>/SKILL.md`.

### 基础防御 / Core defense

| 目录 | 管什么 / Covers | 命中词 / Triggers |
|------|-----------------|-------------------|
| [idempotent](skills/java-backend-guardrails/idempotent/SKILL.md) | 幂等：重复提交、回调、消费 | 防重、重试、重复回调 |
| [concurrency-sell](skills/java-backend-guardrails/concurrency-sell/SKILL.md) | 超卖、扣库存、名额 | 库存、余票、核销占名额 |
| [distributed-transaction](skills/java-backend-guardrails/distributed-transaction/SKILL.md) | 跨服务写；默认事务消息/outbox，禁止默认 TCC | 本地消息表、TCC、Seata |
| [optimistic-lock](skills/java-backend-guardrails/optimistic-lock/SKILL.md) | 版本号 / CAS，防丢失更新 | `@Version`、乐观锁 |
| [distributed-lock](skills/java-backend-guardrails/distributed-lock/SKILL.md) | Redisson / SET NX；不是超卖银弹 | 看门狗、fencing |
| [safe-check](skills/java-backend-guardrails/safe-check/SKILL.md) | SQL 注入、水平/垂直越权、回调验签 | `user_id` WHERE、`${}` |

### 中间件 / Middleware

| 目录 | 管什么 / Covers | 命中词 / Triggers |
|------|-----------------|-------------------|
| [cache-trap](skills/java-backend-guardrails/cache-trap/SKILL.md) | 穿透、击穿、雪崩、Cache-Aside | 缓存一致性、热点 key |
| [redis-ops](skills/java-backend-guardrails/redis-ops/SKILL.md) | RDB/AOF、哨兵、Cluster 16384 槽、大 key | fork、热 key |
| [mq-reliability](skills/java-backend-guardrails/mq-reliability/SKILL.md) | 丢失、重复、顺序、积压、死信 | RocketMQ、延迟消息 |
| [rpc-remote](skills/java-backend-guardrails/rpc-remote/SKILL.md) | Feign 超时、重试、仓壁 | 第三方 HTTP、deadline |
| [rate-limit](skills/java-backend-guardrails/rate-limit/SKILL.md) | 限流、熔断、降级 | Sentinel、网关保护 |

### MySQL

| 目录 | 管什么 / Covers | 命中词 / Triggers |
|------|-----------------|-------------------|
| [mysql-index](skills/java-backend-guardrails/mysql-index/SKILL.md) | 慢 SQL、联合索引、最左前缀 | EXPLAIN、回表 |
| [mysql-mvcc](skills/java-backend-guardrails/mysql-mvcc/SKILL.md) | RR、当前读、间隙锁、死锁 | FOR UPDATE、幻读 |
| [mysql-replication](skills/java-backend-guardrails/mysql-replication/SKILL.md) | 主从、GTID、写后读不到 | 半同步、从库延迟 |
| [table-design](skills/java-backend-guardrails/table-design/SKILL.md) | 状态机字段、归档、分区 | 建表、大表 |
| [sharding](skills/java-backend-guardrails/sharding/SKILL.md) | 分库分表；默认不上 | ShardingSphere、跨库 join |
| [distributed-id](skills/java-backend-guardrails/distributed-id/SKILL.md) | 雪花 / 号段；UUID 不当聚簇主键 | 订单号、ASSIGN_ID |

### 运行时与稳定性 / Runtime

| 目录 | 管什么 / Covers | 命中词 / Triggers |
|------|-----------------|-------------------|
| [jvm-prod](skills/java-backend-guardrails/jvm-prod/SKILL.md) | OOM、Full GC、Metaspace、直接内存 | dump、GC 日志 |
| [thread-pool](skills/java-backend-guardrails/thread-pool/SKILL.md) | 有界队列、拒绝策略、ThreadLocal | 无界队列、虚拟线程 |
| [observability](skills/java-backend-guardrails/observability/SKILL.md) | TraceId、P99、排查顺序 | 链路追踪、全链路压测 |
| [ha-dr](skills/java-backend-guardrails/ha-dr/SKILL.md) | 高可用 ≠ 容灾；RPO/RTO | 主从切换、降级 |
| [multi-live](skills/java-backend-guardrails/multi-live/SKILL.md) | 异地多活、单元化；默认不上 | 双活、写冲突 |
| [consistent-hash](skills/java-backend-guardrails/consistent-hash/SKILL.md) | 虚节点；≠ Redis Cluster 槽 | 加机器迁移 |
| [hot-account](skills/java-backend-guardrails/hot-account/SKILL.md) | 热点行、分段库存 | 爆款 SKU、钱包一行 |

### 任务 / Jobs

| 目录 | 管什么 / Covers | 命中词 / Triggers |
|------|-----------------|-------------------|
| [delay-job](skills/java-backend-guardrails/delay-job/SKILL.md) | 延迟关单；禁止扫全表 | delayTimeLevel、时间轮 |
| [schedule-job](skills/java-backend-guardrails/schedule-job/SKILL.md) | `@Scheduled` 集群重复、XXL-JOB 分片 | cron、ShedLock |

### 电商 / Commerce

| 目录 | 管什么 / Covers | 命中词 / Triggers |
|------|-----------------|-------------------|
| [seckill](skills/java-backend-guardrails/seckill/SKILL.md) | 秒杀分层、Lua 预扣、MQ 创单 | 抢购、限时抢 |
| [order-state](skills/java-backend-guardrails/order-state/SKILL.md) | 正向状态机、非法跳转 | 待支付、已支付 |
| [inventory-occupy](skills/java-backend-guardrails/inventory-occupy/SKILL.md) | 下单占 / 支付扣 / 预占 TTL | 锁定库存、可售库存 |
| [promo-coupon](skills/java-backend-guardrails/promo-coupon/SKILL.md) | 领券超发、价格以服务端为准 | 满减、叠加 |
| [cart](skills/java-backend-guardrails/cart/SKILL.md) | 加购、登录合并；购物车不扣库存 | 未登录购物车 |
| [refund-aftersale](skills/java-backend-guardrails/refund-aftersale/SKILL.md) | 售后逆向、退库存退券 | 仅退款、退货退款 |
| [split-order](skills/java-backend-guardrails/split-order/SKILL.md) | 父单支付、子单履约 | 多商家、多仓 |
| [product-detail](skills/java-backend-guardrails/product-detail/SKILL.md) | 商详静态化，价库存分离 | CDN、SKU |

### 支付 / Payment

| 目录 | 管什么 / Covers | 命中词 / Triggers |
|------|-----------------|-------------------|
| [pay-trade](skills/java-backend-guardrails/pay-trade/SKILL.md) | 预下单 ≠ 已支付 | unified order、二维码过期 |
| [pay-notify](skills/java-backend-guardrails/pay-notify/SKILL.md) | 异步回调：验签、金额分、SUCCESS | notify、return_url |
| [pay-query](skills/java-backend-guardrails/pay-query/SKILL.md) | 超时当未知、补偿扫描 | 丢回调、PAYING |
| [pay-duplicate](skills/java-backend-guardrails/pay-duplicate/SKILL.md) | 两笔成功、关单撞车 | rows=0、多付退余 |
| [pay-refund-channel](skills/java-backend-guardrails/pay-refund-channel/SKILL.md) | 渠道退款 API、同 refund_no | 部分退、退款查单 |
| [pay-split-account](skills/java-backend-guardrails/pay-split-account/SKILL.md) | 渠道分账、延迟分账、回退 | 微信分账、完结 |

### 财务 / Finance

| 目录 | 管什么 / Covers | 命中词 / Triggers |
|------|-----------------|-------------------|
| [finance-stats](skills/java-backend-guardrails/finance-stats/SKILL.md) | GMV ≠ 收入；切日口径 | 支付GMV、结算GMV |
| [finance-settlement](skills/java-backend-guardrails/finance-settlement/SKILL.md) | 商家账期、履约后才可结算 | T+N、结算单 |
| [finance-ledger](skills/java-backend-guardrails/finance-ledger/SKILL.md) | 流水是事实，余额是物化 | 冲正、日切 |
| [reconciliation](skills/java-backend-guardrails/reconciliation/SKILL.md) | 对账、差错 | 渠道对账单 |

### 接口与文件 / API and files

| 目录 | 管什么 / Covers | 命中词 / Triggers |
|------|-----------------|-------------------|
| [api-contract](skills/java-backend-guardrails/api-contract/SKILL.md) | REST、错误码、金额分、时区、分页 | Idempotency-Key、版本 |
| [api-security](skills/java-backend-guardrails/api-security/SKILL.md) | HMAC、防重放、JWT alg、CORS | nonce、脱敏 |
| [distributed-session](skills/java-backend-guardrails/distributed-session/SKILL.md) | 集群登录态 | JWT、SSO、粘滞 |
| [perf-api](skills/java-backend-guardrails/perf-api/SKILL.md) | N+1、循环 Feign、Hikari | 批量 insert、连接池 |
| [file-oss](skills/java-backend-guardrails/file-oss/SKILL.md) | OSS/S3 直传、STS、分片、私有桶 | CDN、预签名 |
| [file-upload](skills/java-backend-guardrails/file-upload/SKILL.md) | magic bytes、路径、SVG、SSRF | Zip Slip、URL 拉图 |

总控路由表（给 AI 用的完整关键词）在 [`skills/java-backend-guardrails/SKILL.md`](skills/java-backend-guardrails/SKILL.md)。

## 常见组合（仍然 ≤4）/ Common combos

| 业务 / Case | 加载 / Load |
|-------------|-------------|
| 支付 / 退款回调 | `pay-notify` + `idempotent`（再加 `pay-duplicate`） |
| 支付超时 / 丢回调 | 只 `pay-query` |
| 用户付了两次 | `pay-duplicate` + `pay-notify` |
| 微信预下单 | `pay-trade` + `pay-notify` |
| 渠道退款 | `pay-refund-channel`（业务售后再加 `refund-aftersale`） |
| 多门店渠道分账 | `pay-split-account`（出账单再加 `finance-settlement`） |
| 核销 | `idempotent` + `concurrency-sell` + `distributed-transaction` + `safe-check` |
| 超时关单 | `delay-job` + `concurrency-sell` + `idempotent` |
| 日批对账 Job | `schedule-job` + `idempotent` |
| 下单扣库存 | `concurrency-sell` + `idempotent` + `mq-reliability` |
| 秒杀入口 | 只 `seckill`（扣减细节再加 `concurrency-sell`） |
| 下单预占 | `inventory-occupy` + `order-state` + `delay-job` |
| 售后退款 | `refund-aftersale` + `idempotent` |
| 慢查询 | 只 `mysql-index` |
| 接口慢未定位 | 只 `observability` |
| 已定位 N+1 / 连接池 | 只 `perf-api` |
| OSS 直传 | 只 `file-oss` |
| 上传校验 / URL 拉图 | `file-upload`（直传再加 `file-oss`） |

## 不要混 / Do not mix

| 不要当成一个问题 | 原因 |
|------------------|------|
| 索引 vs 分库分表 vs 主从 vs MVCC | 四个 skill，四个问题 |
| 缓存陷阱 vs Redis 锁 vs Redis 运维 vs 一致性哈希 | 穿透 ≠ 锁 ≠ 16384 槽 ≠ 哈希环 |
| 延迟关单 vs 定时批 | 一次性延迟消息 ≠ `@Scheduled` 扫表 |
| 渠道分账 vs 商家结算 vs GMV vs 流水 | 划款 ≠ 账期账单 ≠ 报表口径 ≠ 账户余额 |
| 支付回调幂等 vs 通用幂等 | `pay-notify` 管流水线，`idempotent` 管唯一键 |
| 售后状态机 vs 渠道退款 API | `refund-aftersale` vs `pay-refund-channel` |
| 接口规范 vs 接口安全 vs SQL 越权 | REST/分 vs HMAC vs WHERE 归属 |
| OSS 直传 vs 上传校验 | STS/分片 vs magic / SSRF |
| 接口性能 vs JVM vs 慢 SQL vs 排查顺序 | N+1 vs Full GC vs EXPLAIN vs Trace |

## 仓库结构 / Layout

```text
skills/java-backend-guardrails/
  SKILL.md              # 总控：硬规则、路由、组合、输出格式
  idempotent/SKILL.md
  pay-notify/SKILL.md
  ...                   # 每个子目录一个考点
README.md
LICENSE
```

每个子 skill 一般包含：何时用、何时不用、面试级风险、方案对比、Java 片段、反例、验证、评审清单。

Each child usually has: when / when not, interview risks, options, a Java snippet, anti-patterns, verification, checklist.

## 答案来源 / Sources

题目录来自常见 Java 后端面试体系（如 doocs/advanced-java、JavaGuide 的目录）。结论按 Redis / Kafka / MySQL 官方文档、RFC 9110、JWT BCP、微信支付分账、JDK Timer、Spring 调度默认池、XXL-JOB、HikariCP 池大小 wiki、阿里云 OSS、OWASP 文件处理校正，不照抄网文。

Catalog from common Java interview maps. Conclusions checked against official Redis / Kafka / MySQL docs, RFC 9110, JWT BCP, WeChat profit-sharing, JDK Timer, Spring scheduling, XXL-JOB, HikariCP, Aliyun OSS, OWASP file handling — not copied from blog posts.

## License

MIT
