---
name: optimistic-lock
description: Use when preventing lost updates with version/CAS, MyBatis-Plus @Version, or comparing 乐观锁 vs 悲观锁. Do not use for stock oversell by itself (backend-concurrency-sell) or Redis locks (backend-distributed-lock).
---

# 乐观锁

## When to Invoke

并发改同一行配置/订单状态/结算单，冲突少，丢失更新不可接受。MyBatis-Plus `@Version`。

## When NOT

高冲突热点库存（应用条件更新 `stock>=n`，见 `concurrency-sell`）。需要强互斥的短临界区才考虑悲观锁 `SELECT FOR UPDATE`。跨进程锁见 `distributed-lock`。

## 风险（面试考点）

丢失更新：两人读到 version=1，后写覆盖先写。乐观锁不是「不加锁」，是提交时用版本号 CAS。冲突时要重试或让用户刷新。冲突率高时重试会打爆 DB，应改排队或悲观锁。

InnoDB 行锁是悲观；`@Version` 是应用层 CAS，两者不是同一个东西。

## 适用场景

- 订单状态流转（配 version 或状态条件更新）
- 分佣账单人工调整
- 配置实体被两个运营同时编辑

## 方案选型（轻量优先）

| 冲突率 | 默认 |
|--------|------|
| 低 | `@Version` 或 `WHERE version=?` |
| 中 | 状态机条件更新 `WHERE status='CREATED'`（不必 version） |
| 高 | 不要乐观锁死磕；排队 / `SELECT FOR UPDATE` 短事务 |

库存扣减优先 `stock=stock-n WHERE stock>=n`，不必每次上 version。

## 默认方案

```java
@TableName("t_order")
public class Order {
    @TableId(type = IdType.ASSIGN_ID)
    private Long id;
    @Version
    private Integer version;
    private Integer status;
}
```

```java
boolean ok = orderMapper.updateById(order) > 0; // MP 自动 WHERE version=? SET version=version+1
if (!ok) {
    throw new BizException("数据已变更，请刷新后重试");
}
```

手写等价：

```sql
UPDATE t_settle
SET amount = #{amount}, version = version + 1, updated_at = NOW()
WHERE id = #{id} AND version = #{version};
```

状态机可替代 version：

```sql
UPDATE t_order SET status = 2 WHERE id = ? AND status = 1;
```

重试：最多 2–3 次，带间隔。无限重试 = 热点放大。

## 反例

错误：`select` 出对象，改字段，`updateById` 不带 version。
正确：实体带 `@Version`，或条件更新。

错误：乐观锁失败就在 for 循环里立即重试 100 次。
正确：有限次；冲突高则换方案。

错误：用乐观锁替代库存 `WHERE stock>=n`。
正确：库存用条件扣减。

## 验证

- 两个事务读同一 version，只有一个 update 成功。
- 失败方能感知并提示刷新。
- 热点路径没有乐观锁空转。

## 评审清单

- [ ] 更新带 version 或等价状态条件
- [ ] 冲突处理明确（报错/有限重试）
- [ ] 未在超卖路径上只用 version 不加 stock 条件
---
