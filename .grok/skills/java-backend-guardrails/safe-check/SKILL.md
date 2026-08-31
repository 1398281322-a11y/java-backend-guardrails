---
name: safe-check
description: Use when writing or reviewing APIs that touch authz, 越权, SQL injection, third-party callback signatures, 核销 merchant ownership, or sensitive fields. Use on almost every write API; skip only for internal read prototypes with no user input.
---

# 安全检查

## When to Invoke

写接口、接回调、按 ID 查/改资源、拼 SQL、核销、退款、导出。存量评审默认扫一遍。

## When NOT

纯内部、无用户输入、无资源 ID 的一次性脚本。不要用这个 skill 去设计分库分表。

## 风险（面试考点）

- SQL 注入：MyBatis `${}` 拼字符串
- 水平越权：只验证登录，不验证资源属于当前用户/商户
- 垂直越权：普通用户打到管理接口
- 回调伪造：未验签、未校验金额/商户号
- 重放：签名通过但同一通知反复入账（配合 `idempotent`）

## 适用场景

- 核销：必须校验券属于当前门店/商户，不能只传 `voucherId`
- 订单查询：`orderId` + `userId`/`merchantId` 一起查
- 支付回调：验签 + 金额 + 商户号 + 幂等
- 列表筛选：禁止把用户输入拼进 `ORDER BY`

## 方案选型（轻量优先）

| 风险 | 默认 |
|------|------|
| SQL | `#{}` 参数绑定；动态排序白名单 |
| 水平越权 | 查询/更新 WHERE 带归属字段；不要 `selectById` 后直接改 |
| 管理接口 | 网关鉴权 + 角色；与 C 端分离 |
| 回调 | 官方验签 SDK；校验金额、订单号、商户号 |
| 敏感数据 | 日志脱敏；返回 DTO 不带证件号明文 |

## 默认方案

```xml
<!-- 正确 -->
AND merchant_id = #{merchantId}
ORDER BY ${orderBy}  <!-- 禁止：orderBy 必须来自白名单 -->
```

```java
private static final Set<String> ORDER = Set.of("id", "created_at", "amount");

String sort = ORDER.contains(req.getSort()) ? req.getSort() : "id";

int rows = voucherMapper.update(null, Wrappers.<Voucher>lambdaUpdate()
    .set(Voucher::getStatus, "USED")
    .eq(Voucher::getId, voucherId)
    .eq(Voucher::getMerchantId, currentMerchantId())
    .eq(Voucher::getStatus, "UNUSED"));
if (rows == 0) {
    throw new BizException("无权核销或券不可用");
}
```

回调：验签失败直接 4xx；验签成功后仍要核对金额、币种、商户号与本地订单一致。

## 反例

错误：`getById(orderId)` 返回给任意登录用户。
正确：`eq(Order::getUserId, currentUserId())`。

错误：`${merchantId}` 拼 SQL。
正确：`#{merchantId}`。

错误：回调只看 `status=SUCCESS` 不验签。
正确：先验签再入账。

## 验证

- 用别人的 `orderId`/`voucherId` 调接口，必须 403/业务失败。
- 改回调金额，入账被拒绝。
- 动态排序传入 `id;drop table` 被白名单挡掉。

## 评审清单

- [ ] 写/读资源带归属条件
- [ ] MyBatis 无 `${}` 用户输入
- [ ] 回调验签且核对金额
- [ ] 管理接口与 C 端隔离
---
