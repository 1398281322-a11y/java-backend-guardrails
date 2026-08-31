---
name: api-contract
description: Use when defining API 接口规范, REST methods, HTTP status, 错误码, 版本, pagination, 金额单位分, 时区, Idempotency-Key, or request_id. Do not use for HMAC/replay (backend-api-security) or SQL 越权 (backend-safe-check).
---

# 接口规范

## When to Invoke

新接口、前后端联调约定、开放 API、面试「REST 怎么设计」「错误码怎么定」「金额用元还是分」。

## When NOT

验签防重放 → `api-security`。资源归属越权 → `safe-check`。支付回调报文 → `pay-notify`。幂等落库细节 → `idempotent`。

## 风险（面试考点）

**方法语义（RFC 9110）：** GET/HEAD **安全且幂等**（只读、可缓存、可重试）。PUT/DELETE **幂等但不安全**。POST/PATCH **默认不幂等**。GET 写库、GET 带 body、`GET /order/create` 都是扣分项。

业务失败也返回 HTTP 200 且 `code=0`，监控当成功。或反过来所有错误都 500。401 与 403 不分：没登录 vs 没权限。404 可用来隐瞒资源存在（对越权列表也返回 404，不泄露「这个 ID 存在但不属于你」）。

金额用 `double`：对账差一分。时间无时区：报表差一天。分页用巨大 `offset`：深翻页打穿。列表 `null` 当空数组：前端 NPE。创建成功不给 `201` + `Location`，客户端无法跟资源。

版本：路径 `/v1` 或头；**不兼容变更必须新版本**，加字段可兼容。改金额单位、改枚举含义、删必填 = 不兼容。

`Idempotency-Key`（draft-ietf-httpapi）：同一 key 必须返回首次结果。同一 key 不同 body → **409/422**，禁止当新单。进行中的同一 key → 409，客户端等，不要再开一单。

## 方案选型（轻量优先）

| 项 | 默认 |
|----|------|
| URL | 复数名词 `/orders/{id}`，动词在方法上 |
| 方法 | GET 查、POST 创建、PUT 全量幂等更新、PATCH 部分、DELETE 删 |
| HTTP | 2xx 成功（创建 **201**+Location；异步 **202**）；400 参数；401 未登录；403 无权限；404 无资源；409 冲突/幂等处理中；422 语义无法处理；429 限流；5xx 服务器 |
| 业务码 | body 里 `code` 给业务细分；**不要**用 200 包装未登录 |
| 金额 | 整数 **分**，字段名 `amountFen`；禁止 float |
| 时间 | ISO-8601 带时区或统一 UTC；切日时区写进口径 |
| 分页 | 默认 `id` 游标；管理后台小数据可用 page；禁止 `page=10000` |
| 幂等 | 写接口要 `Idempotency-Key` 或业务号，见 `idempotent` |
| 追踪 | 每个响应 `requestId` / `traceId`；5xx **不回堆栈** |

```json
{
  "code": 0,
  "message": "ok",
  "requestId": "…",
  "data": { "amountFen": 19900, "createdAt": "2026-09-01T08:00:00+08:00" }
}
```

列表：`items: []`（空也是数组），`nextCursor`。枚举用字符串并文档化全集。字段命名全链路统一（对外 camelCase 或 snake_case 二选一）。

兼容：只增可选字段、不改语义、不删必填、不改类型。客户端忽略未知字段。批量写：约定「全成或全败」或逐条结果（Zalando 用 207）；禁止静默丢一半。

客户端重试：只对超时/429/5xx；**4xx 除 409/429 不自动重试**。POST 重试必须带同一 Idempotency-Key。

## 反例

错误：`GET /order/create?price=1.99`。
正确：`POST /orders`，body `amountFen`，头 `Idempotency-Key`。

错误：业务库存不足返回 200 + 文案。
正确：约定业务码（如 40901）或 409，监控能扫到。

错误：`page=10000&size=20`。
正确：`cursor` + limit。

错误：同一 Idempotency-Key 换金额再打，当成第二笔。
正确：指纹不一致直接拒。

## 验证

- 未登录 401，越权 403，缺资源 404，可区分。
- 同一 Idempotency-Key 创建订单只有一条；换 body 被拒。
- 金额全链路整数分，对账无浮点误差。
- 空列表是 `[]` 不是 `null`。

## 评审清单

- [ ] 方法语义正确；POST 写有幂等键
- [ ] 金额分、时间带时区
- [ ] HTTP 与业务码可监控；5xx 无堆栈
- [ ] 分页不是巨大 offset
- [ ] 不兼容变更走新版本
---
