---
name: inventory-occupy
description: Use when deciding 下单扣库存 vs 支付扣库存 vs 预占/锁定库存, 可售库存, 锁定库存, occupy TTL. Do not replace the SQL condition stock>=n (backend-concurrency-sell). Do not use for 秒杀分层 (backend-seckill).
---

# 库存预占 / 扣减时机

## When to Invoke

面试问「库存什么时候扣」、下单占库存支付才确认、超时释放、可售=总-锁-售。电商/票务/酒店都是这个模型。

## When NOT

已经确定「一行 stock 条件扣」的实现细节 → `concurrency-sell`。秒杀分层 → `seckill`。售后回库存 → `refund-aftersale`。

## 风险（面试考点）

三种时机：

| 时机 | 优点 | 坑 |
|------|------|-----|
| 下单扣 | 支付时一定有货 | 未支付占死库存，必须超时释放 |
| 支付扣 | 不占库存 | 支付成功可能无货，要赔/取消 |
| 预占（锁定） | 可售与占用分开 | 字段多，超时释放要幂等 |

面试没有唯一答案。零售电商默认 **下单预占 + 超时释放 + 支付确认**；秒杀常用 Redis 预扣再异步下单（`seckill`）。

「先查库存再扣」不是预占，那是丢失更新。预占必须是一条条件更新或 Lua。

## 方案选型（轻量优先）

文旅门票/核销名额：与电商下单预占相同，`CLOSED` 时释放。

字段（推荐预占模型）：

```text
total / locked / sold
available = total - locked - sold
下单：locked += n  WHERE available >= n
支付：locked -= n, sold += n  WHERE locked >= n
取消：locked -= n  WHERE locked >= n
```

或单字段 `stock` 当可售：下单直接减，取消再加。简单，但「已售」靠订单汇总。

## 默认方案

```sql
-- 预占
UPDATE sku SET locked = locked + #{n}
WHERE sku_id = #{id} AND (total - locked - sold) >= #{n};

-- 支付确认
UPDATE sku SET locked = locked - #{n}, sold = sold + #{n}
WHERE sku_id = #{id} AND locked >= #{n};

-- 关单释放（只能释放本单占用量，用流水防多释放）
```

占用来源记 `t_stock_log(order_no, sku_id, n, type)`，`uk(order_no, sku_id, type)`。释放/确认都插流水，重复空操作。

超时释放走 `delay-job`，不要扫全表 locked>0。

## 反例

错误：支付扣库存，回调成功才发现 stock=0，订单 PAID 无货。
正确：要嘛预占，要嘛支付失败/赔付流程写清。

错误：关单 `stock=stock+n` 无流水，MQ 重试加两次。
正确：释放流水唯一键。

错误：预占用分布式锁包读改写。
正确：一条条件 UPDATE。

## 验证

- 库存 1，两单同时下，只有一单预占成功。
- 预占后关单，可售回到 1，只回一次。
- 支付确认后 locked 下降、sold 上升，总和不变。

## 评审清单

- [ ] 扣减时机写进方案，不是默认「反正会扣」
- [ ] 预占/释放有流水唯一键
- [ ] 超时释放不扫全表
---
