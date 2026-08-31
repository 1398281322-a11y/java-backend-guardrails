---
name: file-oss
description: Use when implementing 文件存储, OSS, S3, MinIO, STS 直传, 预签名 URL, 分片上传, CDN, 私有桶, or 文件不进 MySQL. Do not use for magic-byte/SSRF/SVG (backend-file-upload).
---

# 对象存储 / OSS

## When to Invoke

头像、证件、门票 PDF、商户资质、视频；面试「文件放哪」「为什么不进 MySQL」「前端怎么直传还不泄露 AK」。

## When NOT

上传类型、路径穿越、SVG/XSS、从 URL 拉文件（SSRF）→ `file-upload`。商详页静态化/库存缓存分离 → `product-detail`。OSS 回调幂等落库细节可再加 `idempotent`。

## 风险（面试考点）

BLOB 进 MySQL：备份炸、主从延迟、无法 CDN。本地盘：多实例对不上、磁盘满、无冷备。默认 **对象存储**（阿里云 OSS / 兼容 S3 的 MinIO）。

**AccessKey 禁止进前端、Git、日志。** 直传三选一（阿里云官方）：

| 方式 | 能限制大小/类型/目录 | 分片续传 | 用在 |
|------|----------------------|----------|------|
| STS 临时凭证 | RAM Policy 管前缀权限 | 能 | App / 大文件 |
| PostObject + 服务端 Policy/签名 | Policy 能限 content-length、key 前缀、Content-Type | **不能** | Web 表单小文件 |
| 预签名 PUT URL | 签死 object key 和部分头 | 单对象简单传 | 单文件、有效期内可被转发 |

预签名 URL 是 **持有即授权**（bearer）。STS 与 URL 都有过期时间，**取较短的那个**。Bucket 要配 CORS。

直传后服务端不知道成功：配 **Callback**，OSS POST 到你的接口；先验签再落 `t_file`。不要信前端「我传完了」。

公私有：商品图可 public + CDN；证件/合同 **私有桶**，下载用短时 GET 签名 URL，不要永久公网链接。防盗链 Referer 可绕，不能当唯一防线。

大文件：Initiate → UploadPart → Complete；未完成的 Part **占容量计费**，要 `AbortMultipartUpload` 或生命周期清理。服务端中转必须 **流式**，禁止 `byte[]` 把 500MB 读进堆。

Object key **服务端生成**：`biz/yyyyMMdd/{uuid}.ext`，不要用用户原始文件名（覆盖、路径、特殊字符）。DB 存 `object_key`、bucket、size、content_type、sha256，不存二进制。

## 方案选型（轻量优先）

| 场景 | 默认 |
|------|------|
| 本地/测试 | MinIO（S3 API），同一套 SDK |
| C 端小图 | PostObject 或 STS，Policy 限 2～10MB |
| 证件私有 | 私有桶 + 登录后签 GET URL（分钟级） |
| 视频/大包 | STS + 分片；完成回调再转码 |
| 公开图 | OSS + CDN，改图换 key 或带版本号 |

```text
1. 登录用户向 API 申请上传：业务、contentType、size
2. 服务端生成 objectKey，签发 STS 或 Policy（过期 15min，前缀写死）
3. 客户端直传 OSS
4. OSS Callback → 验签 → INSERT t_file uk(object_key) status=READY
5. 业务单绑定 file_id（头像、订单附件）
```

上传成功、业务单没绑：孤儿文件，日批按 `READY` 且无引用删除。先删 DB 再删 OSS，或反过来要对失败补偿，禁止只删一边。

## 反例

错误：前端写死 `accessKeyId/secret`。
正确：STS 或服务端签名，RAM 只授该前缀 `PutObject`。

错误：`INSERT` 图片 BLOB；或 Tomcat 存 `D:\upload`。
正确：OSS + 表只存 key。

错误：前端说成功就改头像，无回调/无 HeadObject。
正确：以 OSS 回调或服务端 Head 为准。

错误：证件用 `https://bucket.oss.../idcard.jpg` 永久公开。
正确：私有 + 短时签名 GET。

## 验证

- 前端包和日志无长期 AK。
- 直传到错误前缀/超 Policy 大小被 OSS 拒。
- 回调连打，`t_file` 一行；未回调则业务未绑定。
- 未完成分片会被 abort/生命周期清掉。

## 评审清单

- [ ] 文件不进库、不进应用盘当主存储
- [ ] 密钥不进前端；key 服务端生成
- [ ] 私有对象走签名 GET
- [ ] 回调验签 + 唯一键；分片有 abort
---
