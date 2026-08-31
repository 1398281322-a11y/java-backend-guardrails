---
name: api-security
description: Use when hardening APIs: HTTPS, HMAC 签名, timestamp+nonce 防重放, CORS, 脱敏, JWT none 算法, 密钥不进 URL. For SQL injection and resource 越权 WHERE, use backend-safe-check.
---

# 接口安全

## When to Invoke

开放平台、App 签名、防抓包重放、敏感字段、CORS、面试「怎么防重放」「水平越权」「HMAC 和 SHA256 区别」。

## When NOT

SQL `${}`、订单必须带 `user_id` → `safe-check`（本文件点过去，不写归属 SQL）。限流数值 → `rate-limit`。支付回调验签细节 → `pay-notify`。登录态存储 → `distributed-session`。上传类型/路径/SSRF → `file-upload`。

## 风险（面试考点）

认证 ≠ 授权。登录了仍能改别人 `orderId`（水平越权 / IDOR）；普通用户打管理接口（垂直越权）。**签名过了 ≠ 有权碰这张单。**

签名防 **篡改+身份**；timestamp+nonce 防 **重放**。只有签名、窗口内同一包可打无数次。只有 timestamp 一秒内可重放。nonce 必须不可预测，**秒级时间戳当 nonce 不够**。

**HMAC-SHA256（带密钥）≠ 裸 SHA256/MD5。** 哈希只能校验内容没被改，任何人都能重算；HMAC 证明持有 secret。HMAC **不是加密**，报文明文仍要 HTTPS。

Secret **不进 URL、不进日志、不进前端包、不进 Git**。GET query 会被浏览器/CDN/access log 全文记下。

JWT：禁止信任客户端 `alg`。`alg=none` 可去签；`alg` 从 RS256 改成 HS256 会用公钥当 HMAC 密钥（CVE-2015-9235）。服务端 **写死允许算法**。JWT 签的是 token，**不签这次 HTTP body**——改订单金额要靠业务授权或 HMAC，不能只验 Bearer。

HTTPS 强制。Cookie `HttpOnly`+`SameSite`。Cookie 会话还要防 CSRF；CORS 不要 `*` 加凭证。日志脱敏：手机、证件、卡号、token。密码不可逆哈希+盐，不是可逆 AES 存密码。

IP 白名单、设备指纹是加分项，**不能替代签名和授权**。

## 方案选型（轻量优先）

| 场景 | 默认 |
|------|------|
| C 端登录接口 | HTTPS + JWT/Session；写操作要登录 |
| 开放/服务间 | HMAC-SHA256 + timestamp + nonce |
| 资金写 | 签名防重放 **还要** 业务幂等（`idempotent`） |
| 浏览器跨域 | 白名单 Origin，不 `*`+cookie |

防重放：

```text
|now - timestamp| <= 60s~5min   // 对时；过宽等于没防
Redis SETNX nonce TTL=窗口*2
canonical = METHOD\nPATH\nsortedQuery\nbodyHash\ntimestamp\nnonce
sign = HMAC_SHA256(secret, canonical)   // 双方拼串必须完全一致
```

比较签名用常量时间（`MessageDigest.isEqual`），防时序侧信道。密钥按 `kid` 轮换，旧密钥宽限期后再下线。

越权：查改删都 `WHERE id=? AND user_id=?`（或商户），见 `safe-check`。不能只信 body 里的 userId。

## 默认方案

网关：TLS、鉴权、限流、CORS。服务内：资源级 WHERE。开放 API：签名过滤器。

```java
if (Math.abs(now - ts) > 300_000) reject();
if (!Boolean.TRUE.equals(redis.opsForValue()
        .setIfAbsent("nonce:" + nonce, "1", Duration.ofMinutes(10)))) reject();
if (!MessageDigest.isEqual(hmac, expected)) reject();
// 再鉴权：这个身份能不能碰这个 orderId
```

## 反例

错误：签名过了就当授权过了。
正确：签名只证明没被改；仍要验这个用户能不能碰这张单。

错误：nonce 用时间戳；或只校验 ±5 分钟不存 nonce。
正确：随机 16+ 字节 + SETNX。

错误：GET 把 token、secret 放 query；日志打印完整 Authorization。
正确：头或 body；日志打码。

错误：CORS `Access-Control-Allow-Origin: *` 且允许 cookie。
正确：精确 Origin。

错误：JWT 按 header 里的 `alg` 选验证器。
正确：服务端允许列表，拒绝 `none`。

## 验证

- 改 body 不改 sign → 拒。
- 合法包重放第二次 → 拒。
- 用户 A token + 用户 B orderId → 403。
- 日志无完整手机号/token。
- `alg=none` / 改 HS256 的 JWT → 拒。

## 评审清单

- [ ] 写接口有认证；开放接口 HMAC+时间窗+nonce
- [ ] 资源级授权（配合 safe-check）
- [ ] JWT 算法写死；密钥与敏感日志不泄露
- [ ] CORS 白名单；HTTPS
---
