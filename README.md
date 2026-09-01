# Java Backend Guardrails

把 Java 后端面试里会炸的坑，收成写代码时的防御包。不是背题库，也不是上来就上 TCC。

Interview-grade Java microservice pitfalls as **production checks** — not flashcards, and not an excuse to reach for TCC.

默认栈 / Default stack：**Spring Boot + MyBatis-Plus + Redisson + RocketMQ**。面向订单、支付、核销、结算、回调。

总控是路由器：按关键词最多加载 **2–4** 个子 skill。普通单表 CRUD 不加分库分表、Seata、全局锁。

The parent skill is a router. Agents load **at most 2–4** children. Ordinary CRUD does not get sharding, Seata, or a global lock.

- 技能页 / Page：https://skills.sh/1398281322-a11y/java-backend-guardrails/java-backend-guardrails
- 仓库 / Repo：https://github.com/1398281322-a11y/java-backend-guardrails

## 安装 / Install

```bash
npx skills add 1398281322-a11y/java-backend-guardrails
```

只看有哪些 skill / List without installing：

```bash
npx skills add 1398281322-a11y/java-backend-guardrails --list
```

Grok / Cursor 打开本仓库即可发现 `skills/java-backend-guardrails`。其他项目把该目录拷到：

- Grok：`.grok/skills/java-backend-guardrails`
- 或用户目录：`~/.grok/skills/java-backend-guardrails`

Open this repo in Grok/Cursor, or copy `skills/java-backend-guardrails` into `.grok/skills/` (or `~/.grok/skills/`).

## 怎么用 / How agents should use it

1. 先读总控 `SKILL.md` / Read the parent first.
2. 按关键词加载 2–4 个子 skill / Load 2–4 matching children.
3. 输出：判定 / 风险 / 方案 / 落地 / 评审。
4. 写库优先唯一键和条件更新，不要默认分布式锁 / Unique keys and conditional updates beat distributed locks.
5. 禁止无脑 TCC、Seata、分库分表 / Never default to TCC, Seata, or sharding.

显式指定也可以，例如：「用幂等 skill 看这段回调」。

You can also name a skill: “use the idempotent skill on this callback”.

## 分包原则 / Split rules

一个考点一个文件。不要混：

| 不要混 / Do not mix | 原因 |
|---------------------|------|
| 索引 vs 分库分表 | 慢 SQL ≠ 要拆库 |
| 延迟关单 vs 定时批 | 一次性延迟 ≠ cron |
| 渠道分账 vs 商家结算 vs GMV | 划款 ≠ 账期账单 ≠ 报表口径 |
| OSS 直传 vs 上传校验 | STS/分片 ≠ magic bytes / SSRF |
| 接口规范 vs 接口安全 | REST/错误码 ≠ HMAC/防重放 |

完整路由表见 [`skills/java-backend-guardrails/SKILL.md`](skills/java-backend-guardrails/SKILL.md)。

## 目录 / Layout

```
skills/java-backend-guardrails/
  SKILL.md          # 总控 / router
  idempotent/       # 幂等
  pay-notify/       # 支付回调
  concurrency-sell/ # 超卖
  ...
```

## License

MIT
