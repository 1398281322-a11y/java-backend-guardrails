---
name: split-order
description: Use when an order must split by merchant, warehouse, or 门店/景区: 拆单, 父单子单, 多商家结算. Do not use for sharding tables (backend-sharding).
---

# 拆单

## When to Invoke

购物车跨店、多仓发货、文旅一单里多个景区/门店要分别履约和结算。面试：「怎么拆单、支付怎么付」。

## When NOT

分库分表 → `sharding`。普通单店单仓不要拆。

## 风险（面试考点）

父单支付成功、子单创一半：钱与货不一致。拆完每子单独立扣自己仓库存，取消要按子单释放。客服只拿父单号找不到子单。

支付两种：父单一次支付再分账；子单分别支付。C 端默认 **父单一次支付**，内部拆子单履约。

## 方案选型（轻量优先）

```text
t_order_parent  支付、用户、应付总额
t_order_split   parent_no + merchant_id/warehouse_id, 自己的状态、金额、运费
```

拆分键：`merchant_id`（电商多店）或 `warehouse_id` 或 `poi_id`（文旅门店）。同一键一行子单。

下单本地事务：写父单 + 所有子单 + 各子单预占。失败全滚。支付回调只改父单 PAID，再消息通知子单 PAID（子单幂等）。

履约、发货、售后挂在 **子单**。退款金额不能超过该子单实付。

## 默认方案

```sql
UNIQUE KEY uk_parent_merchant (parent_no, merchant_id)
```

父单号对外展示；子单号内部履约。查询始终能用父单号查出全部子单。

部分子单售后：父单状态 `PARTIAL_REFUND`，不要整单关。

## 反例

错误：一个订单表 `merchant_ids` 逗号存多家，发货无法拆。
正确：子单行。

错误：子单各自调支付，用户付 N 次。
正确：C 端一次支付（除非业务就是到店分付）。

错误：取消父单只改父状态，子单库存不释放。
正确：遍历子单走释放流水。

## 验证

- 两店结算，生成 1 父 2 子，支付一次，两店都能看到待发货。
- 一店售后，另一店不受影响。
- 关父单，两店预占都释放且只一次。

## 评审清单

- [ ] 有父单号和子单号
- [ ] 库存按子单预占
- [ ] 支付次数与产品设计一致
---
