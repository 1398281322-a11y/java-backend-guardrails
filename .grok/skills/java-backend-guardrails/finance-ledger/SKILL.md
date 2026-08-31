---
name: finance-ledger
description: Use when implementing 记账流水, 账户余额, 借贷, 日切, 不可改历史流水, or 余额必须能用流水重放。Not for GMV dashboards (backend-finance-stats) or merchant payout bills (backend-finance-settlement).
---

# 财务流水 / 记账

## When to Invoke

用户余额、平台备付金、积分、佣金账户、面试「余额怎么保证对得上」。

## When NOT

报表口径 → `finance-stats`。商家账期打款 → `finance-settlement`。支付渠道回调 → `pay-notify`。

## 风险（面试考点）

只 UPDATE 余额、没有流水：对不上、无法审计、并发丢更新。只插流水不算余额：每次都 SUM，账户热、还可能漏。

正确：**流水是事实，余额是物化视图**。入账 = 同一本地事务里 `INSERT 流水(uk 业务号)` + `余额 += n WHERE` 或乐观版本。重复业务号插流水失败则整单回滚。

禁止改、删已入账流水。冲正用反向流水。日切后当日流水只读。

金额分；禁止 float。账户类型分开：用户余额、商家待结算、平台佣金，不要一个字段打天下。

## 方案选型（轻量优先）

```sql
t_account (id, type, owner_id, balance_fen, version)
t_ledger  (id, account_id, biz_no, delta_fen, balance_after, uk(account_id, biz_no))
```

```java
@Transactional
void credit(long accountId, String bizNo, long fen) {
    try {
        ledgerMapper.insert(row(accountId, bizNo, fen));
    } catch (DuplicateKeyException e) { return; }
    int n = accountMapper.update(null, Wrappers.<Account>lambdaUpdate()
        .setSql("balance_fen = balance_fen + " + fen)
        .eq(Account::getId, accountId));
    if (n != 1) throw new IllegalStateException("account missing");
}
```

扣减必须 `balance_fen >= fen`。热点账户见 `hot-account`。

日终：对每个账户 `期初 + SUM(当日流水) = 期末`，不平进差错（`reconciliation`）。

## 反例

错误：`setBalance(getBalance()+n)` 无版本无流水。
正确：流水 uk + 条件更新。

错误：客服改余额 UPDATE 一行。
正确：调账也走流水，类型 `ADJUST`，留审核。

错误：统计直接 SUM 流水当实时余额给交易用。
正确：交易读账户表；SUM 只给对账。

## 验证

- 同一 bizNo 入账两次，余额只加一次。
- 并发扣到 0，不会扣成负。
- 用流水重放得到的余额 = 账户表。

## 评审清单

- [ ] 有流水且业务号唯一
- [ ] 余额与流水同事务
- [ ] 不改历史、冲正走反向
- [ ] 单位分
---
