---
name: mysql-mvcc
description: Use when debugging lost updates, phantom reads, deadlocks, SELECT FOR UPDATE, InnoDB isolation, MVCC, gap lock, next-key lock, or 当前读/快照读. Do not use for secondary index design (backend-mysql-index) or sharding.
---

# MySQL MVCC / 隔离 / 间隙锁

## When to Invoke

事务里看到「改了却读不到」「幻行」「间隙锁死锁」、`FOR UPDATE`、RR vs RC、核销/结算并发改同一范围。

## When NOT

只是慢查询走哪条索引 → `mysql-index`。分库 → `sharding`。库存扣减条件更新 → `concurrency-sell`。

## 风险（面试考点）

InnoDB 默认 **REPEATABLE READ**。ANSI 说 RR 仍可能幻读；InnoDB 用 MVCC + Next-Key 把常见幻读挡住，但要分读法：

| 读法 | SQL | 机制 |
|------|-----|------|
| 快照读 | 普通 `SELECT` | MVCC Read View，不加锁 |
| 当前读 | `SELECT ... FOR UPDATE/SHARE`、`UPDATE`、`DELETE` | 加锁读最新已提交版本 |

- **RR**：事务内第一次快照读建立 Read View，之后复用 → 不可重复读/快照幻读被挡住。
- **RC**：每条语句新 Read View → 能看到他事务已提交的新值。
- **Next-Key** = 行锁 + 间隙锁。范围当前读会锁间隙，挡住插入。唯一索引等值命中通常只锁行、不锁间隙。
- 无索引条件的更新可能锁大量间隙甚至退化成表级效果，并发崩、死锁多。

## 方案选型（轻量优先）

| 场景 | 默认 |
|------|------|
| 普通查询 | 快照读，别乱加 `FOR UPDATE` |
| 改一行（有主键） | `UPDATE ... WHERE id=? AND status=?` |
| 必须锁行再改 | `SELECT ... WHERE id=? FOR UPDATE`，事务尽量短 |
| 热点插入死锁 | 考虑 RC（少间隙锁）或减少范围当前读 |

## 默认方案

```sql
-- 核销：主键等值当前读，不要锁整个商户的券
SELECT * FROM t_voucher WHERE id = ? FOR UPDATE;

-- 更好：条件更新，少开长事务
UPDATE t_voucher SET status = 2
WHERE id = ? AND merchant_id = ? AND status = 1;
```

死锁：看 `SHOW ENGINE INNODB STATUS` 的 `LATEST DETECTED DEADLOCK`。两边按同一顺序加锁。

## 反例

错误：「RR 绝对不会幻读」。
正确：快照读基本不会；当前读靠间隙锁。范围 `FOR UPDATE` 仍可能和插入打架。

错误：`SELECT * FROM t_order WHERE merchant_id=? FOR UPDATE` 再循环改。
正确：锁范围过大。按主键逐条或条件更新。

错误：把间隙锁写成「索引失效」。
正确：那是另一套问题。

## 验证

- 两个事务，一个范围当前读，一个插入同一间隙，后者阻塞或死锁可解释。
- 普通 SELECT 在 RR 下两次结果一致（他事务提交后）。
- 核销 SQL 走主键/唯一索引，不是全表间隙。

## 评审清单

- [ ] 没有无必要的 `FOR UPDATE`
- [ ] 当前读条件带主键/唯一键
- [ ] 事务短
- [ ] 未和索引/分片 skill 混写
---
