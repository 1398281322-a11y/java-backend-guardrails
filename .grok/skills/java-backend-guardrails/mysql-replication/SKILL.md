---
name: mysql-replication
description: Use when designing 读写分离, MySQL 主从, binlog, GTID, 半同步, 主从延迟, write-then-read. Do not use for sharding (backend-sharding) or secondary indexes (backend-mysql-index).
---

# 读写分离 / 主从复制

## When to Invoke

读多写少要拆读流量、支付后立刻查订单、从库延迟、主从切换。

## When NOT

单表太大要拆库 → `sharding`。慢 SQL → `mysql-index`。备份演练细节也可看 `ha-dr`。

## 风险（面试考点）

复制默认 **异步**：主库提交成功，从库可能还没回放。半同步只保证从库 **relay log 收到**，不保证已经 apply。超时会 **降级回异步**。`Seconds_Behind_Master` 是估算。

写后读：下单成功立刻 `SELECT` 打到从库 → 空单。资金、核销结果、支付状态 **读主**。

Binlog 生产用 **ROW**。STATEMENT 在不确定函数下会不一致。

## 方案选型（轻量优先）

| 优先级 | 做法 |
|--------|------|
| 1 | 不上读写分离 |
| 2 | 只把可脏读的列表/报表打从库；详情/写后读走主 |
| 3 | 半同步（至少 1 个从 ACK）提高切主时少丢 |
| 4 | MGR/组复制 | 明确要求且能运维才上 |

延迟常见原因：大事务、从库单线程回放（老版本）、没索引的回放 SQL、DDL。先拆大事务、从库并行复制，不要先上分库。

## 默认方案

```text
写 → 主库
读：
  - 下单后查询、支付状态、核销结果 → 主库
  - 商户后台报表、可延迟列表 → 从库
```

MyBatis 标注：

```java
@Transactional(readOnly = true) // 若框架按此路由从库，资金查询不要标这个
public Order detail(Long id) { ... } // 资金详情强制主库数据源
```

写后读三种：强制主库（默认）；等 GTID 追平（`WAIT_FOR_EXECUTED_GTID_SET`）；会话粘滞。文旅订单详情用强制主库。

## 反例

错误：所有 SELECT 走从库，支付回调后马上查从库改状态。
正确：写后读走主。

错误：半同步 = 同步强一致。
正确：ACK 的是 relay log；超时会降级。

错误：用读写分离解决单表 2 亿行。
正确：那是归档/分区/分片。

## 验证

- 写后立即读，读主可见。
- 人为制造从库延迟，列表可旧，资金详情仍对。
- 切主后 GTID 能接上，不丢已确认写（半同步未降级时）。

## 评审清单

- [ ] 资金/核销读路径不走从库
- [ ] 没有把延迟当「分片问题」
- [ ] 大事务已拆或有解释
---
