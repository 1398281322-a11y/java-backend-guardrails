---
name: distributed-session
description: Use when cluster login state, 分布式 Session, JWT, SSO, Redis session, sticky sessions, or gateway auth is in play. Do not use for SQL 越权 resource checks (backend-safe-check).
---

# 分布式会话

## When to Invoke

多实例登录态、网关鉴权、C 端 token、管理端 SSO、Session 粘滞。

## When NOT

资源是否属于当前商户 → `safe-check`。限流 → `rate-limit`。

## 风险（面试考点）

Tomcat Session 默认在单机内存，集群会丢登录或要粘滞。粘滞：实例挂了那批用户掉线，扩缩容痛苦。

JWT 无状态好扩，但 **无法立刻作废**（除非黑名单/短 TTL+刷新）。敏感权限变更要能踢人。

Session 进 Redis：好作废，但 Redis 挂了全员掉线，要当缓存依赖来保。

Cookie 域、HTTPS、`HttpOnly`、`SameSite`；跨子域 SSO 不要把 token 放 URL。

## 方案选型（轻量优先）

| 场景 | 默认 |
|------|------|
| C 端 API | 短 TTL Access Token + Refresh；网关验签 |
| 要立刻踢人 | Redis 存 session 或 token 黑名单 |
| 管理端 | 同 Redis session 或统一认证 |
| 禁止当默认 | Nginx ip_hash 粘滞当长期方案 |

## 默认方案

```text
Authorization: Bearer <jwt>
JWT 只放 userId、过期时间；权限实时查或短缓存
登出 / 改密：把 jti 写入 Redis 黑名单直到过期
```

网关验签失败 401。业务服务仍要做 **水平越权**（`safe-check`），有 token ≠ 能改任意订单。

## 反例

错误：JWT 里塞权限列表，运营改角色后 24h 仍是管理员。
正确：短 TTL 或 Redis 版本号。

错误：token 放 query string 方便 H5。
正确：头或 HttpOnly cookie。

错误：多实例内存 Session。
正确：无状态或 Redis。

## 验证

- 杀一个实例，已登录用户仍可用。
- 登出后原 token 立即 401。
- 用户 A 的 token 不能改用户 B 的订单。

## 评审清单

- [ ] 无粘滞依赖
- [ ] 能作废或足够短 TTL
- [ ] 鉴权之后仍有资源级校验
---
