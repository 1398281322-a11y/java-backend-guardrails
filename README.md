# Java Backend Guardrails

Interview-grade Java microservice pitfalls as **production checks** — not a flashcard dump, and not a reason to reach for TCC.

Default stack: **Spring Boot + MyBatis-Plus + Redisson + RocketMQ**. Built for orders, payments, verification (核销), settlement, and callbacks.

The parent skill is a router. Agents load **at most 2–4 child skills**. Ordinary single-table CRUD does not get sharding, Seata, or a global lock.

中文定位：把面试考点变成写代码时的风险扫描。轻量默认，升级要写触发条件。

## Install (skills.sh)

```bash
npx skills add 1398281322-a11y/java-backend-guardrails
```

List what would be installed:

```bash
npx skills add 1398281322-a11y/java-backend-guardrails --list
```

Install one child only:

```bash
npx skills add 1398281322-a11y/java-backend-guardrails --skill pay-notify
```

## Layout

```
skills/java-backend-guardrails/   # marketplace pack (npx skills add)
  SKILL.md                        # router
  idempotent/
  pay-notify/
  ...
.grok/skills/java-backend-guardrails/  # same pack for Grok
```

Grok / local: open this repo, or copy `skills/java-backend-guardrails` to `.grok/skills/java-backend-guardrails`.

## How agents should use it

1. Read the parent `SKILL.md`.
2. Match keywords; load **2–4** children.
3. Output: 判定 / 风险 / 方案 / 落地 / 评审.
4. Prefer unique keys and conditional updates over distributed locks.
5. Never default to TCC, Seata, or sharding.

## Child skills

One topic per file. Indexes ≠ sharding. Delayed cancel ≠ cron. Channel profit-sharing ≠ merchant settlement ≠ GMV. OSS upload ≠ file-type / SSRF checks.

See the routing table in [`skills/java-backend-guardrails/SKILL.md`](skills/java-backend-guardrails/SKILL.md).

## License

MIT
