---
name: mysql-index
description: Use when diagnosing slow SQL, creating or changing secondary indexes, EXPLAIN, 最左前缀, covering index, or implicit conversion. Do not use for sharding or table status-machine design.
---

# MySQL 索引

## When to Invoke

慢查询、`EXPLAIN`、加索引、分页深翻、订单列表按用户+时间查。

## When NOT

分库分表、选分片键 → `sharding`。建表字段/唯一键/归档 → `table-design`。

## 风险（面试考点）

InnoDB 二级索引是 B+ 树。联合索引最左前缀：`(user_id, created_at)` 能用 `user_id`，不能单独用 `created_at`。回表：二级索引找到主键再回聚簇。覆盖索引：SELECT 列都在索引里可避免回表。隐式转换（`varchar` 列用数字比）会让索引失效。函数包列（`DATE(created_at)=`）同样失效。

面试常说「RR 靠间隙锁防幻读」——那是锁，不是索引本身；本 skill 不展开。当前读/间隙锁见业务里 `SELECT FOR UPDATE` 时再单独处理。

## 适用场景

- `WHERE user_id=? ORDER BY created_at DESC LIMIT 20`
- 核销列表：`merchant_id + status + created_at`
- 回调查重：应靠 UNIQUE，不是普通二级索引

## 方案选型（轻量优先）

| 场景 | 默认索引 |
|------|----------|
| 等值 + 时间排序 | `(user_id, created_at)` |
| 等值 + 过滤 + 时间 | `(merchant_id, status, created_at)` |
| 唯一业务键 | UNIQUE，不当成普通 INDEX |
| 低区分度列（status 单独） | 不要单建；放到联合索引第二列 |
| 深分页 | 延迟关联 / 记上次 ID，不 `LIMIT 100000,20` |

先 `EXPLAIN`，再加索引。禁止凭感觉给每个查询字段都建单列索引。

## 默认方案

```sql
EXPLAIN SELECT id, order_no, created_at
FROM t_order
WHERE user_id = ? AND created_at >= ?
ORDER BY created_at DESC
LIMIT 20;
-- 期望：type=range, key=idx_user_created, Extra 尽量 Using index 或至少非 filesort 全表
```

```sql
ALTER TABLE t_order ADD KEY idx_user_created (user_id, created_at);
```

分页：

```sql
-- 用上一页最小 id，不要大 offset
WHERE user_id = ? AND id < #{lastId}
ORDER BY id DESC LIMIT 20;
```

隐式转换：

```sql
-- 列是 VARCHAR 单号
WHERE order_no = 123        -- 可能失效
WHERE order_no = '123'      -- 正确
```

## 反例

错误：`WHERE DATE(created_at) = '2026-09-01'`。
正确：`created_at >= '2026-09-01' AND created_at < '2026-09-02'`。

错误：`OR` 两边不同列导致全表；或 `LIKE '%abc'`。
正确：改写 UNION；搜索走专门检索而不是硬上索引。

错误：一张表 12 个单列索引，写入变慢。
正确：按查询形状建联合索引，定期用 `unused index` 思路删。

错误：把「分片键」写进索引优化建议。
正确：分片是另一套问题。

## 验证

- `EXPLAIN` 走预期索引，`rows` 数量级合理。
- 慢日志里该 SQL 消失或耗时下降。
- 写入 QPS 未因索引过多明显变差。

## 评审清单

- [ ] 联合索引列顺序匹配 WHERE/ORDER
- [ ] 无函数包列、无隐式转换
- [ ] 深分页不是巨大 OFFSET
- [ ] 没有用分库分表冒充索引优化
---
