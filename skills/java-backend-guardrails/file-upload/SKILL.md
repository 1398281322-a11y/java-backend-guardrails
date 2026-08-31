---
name: file-upload
description: Use when hardening 文件上传/下载: 类型白名单, magic bytes, 大小限制, 路径穿越, SVG XSS, Zip Slip, 从 URL 拉文件 SSRF. Do not use for STS/直传/CDN (backend-file-oss).
---

# 上传安全 / 文件校验

## When to Invoke

任意用户上传、导入「图片 URL」、解压、下载接口、面试「怎么防上传 webshell」「SSRF」。

## When NOT

STS/直传/分片/私有桶 → `file-oss`（本文件管内容与路径，不管 AK）。SQL 越权 WHERE → `safe-check`。接口 HMAC → `api-security`。

## 风险（面试考点）

`Content-Type` 和扩展名都是攻击者填的。`image/jpeg` + `shell.php` 不能信。OWASP：用 **magic bytes**（JPEG `FF D8 FF`、PNG `89 50 4E 47`）+ **扩展名白名单**，图片再经图像库重编码（剥脚本/异常元数据）。黑名单 `.php` 会被 `.phtml` / `.php.jpg` 绕过。

Object key / 本地路径 **禁止用用户文件名**。`../../etc/passwd`、`avatar.jpg/x.php` 都是穿越。下载接口不要 `?path=` 拼磁盘。

SVG 是 XML：内嵌 `<script>` 造成 **存储 XSS**；DOCTYPE 外部实体造成 **XXE / SSRF**。默认 **拒绝 SVG**；必须收则消毒或当附件下载（`Content-Disposition: attachment` + `Content-Type: application/octet-stream`），且放到 **独立域**，不要跟主站同域打开。

Zip：**Zip Slip**（压缩包内 `../`）、Zip 炸弹。解压忽略用户路径，落盘只允许白名单目录 + 条目大小/数量上限。

**从 URL 拉图是 SSRF。** 禁止用户传 `http://169.254.169.254/`、`file://`、内网 IP、本机。只允许 https、解析后 IP 做内网黑名单、禁止 30x 跳到内网、超时短、体积上限。能直传 OSS 就不要服务端去拉。

大小：网关 `client_max_body_size`、应用、OSS Policy **三层都限**。只限前端等于没限。流式读，禁止整文件进内存。

下载：`Content-Disposition: attachment; filename="..."`（文件名编码、无 XSS）；不要 `inline` 打开 HTML/SVG。文件记录带 `user_id`，下载走归属（`safe-check`）。

## 方案选型（轻量优先）

| 场景 | 默认 |
|------|------|
| 头像/门票图 | jpg/png/webp 白名单 + magic + 重编码 + 2～5MB |
| 证件 PDF | pdf magic + 体积上限，私有桶 |
| 用户 SVG | 默认拒 |
| 导入 URL | 能不用就不用；用则 SSRF 防护清单 |
| 解压 | 不按压缩包内路径写盘 |

```java
String ext = switch (magicOf(headBytes)) {
    case JPEG -> "jpg";
    case PNG -> "png";
    default -> throw new BizException("不支持的文件类型");
};
String key = "avatar/" + Date.now() + "/" + UUID.randomUUID() + "." + ext;
if (file.getSize() > 5 * 1024 * 1024) throw new BizException("过大");
```

URL 导入：

```text
只 https → 禁止 127/10/172.16/192.168/169.254/::1
DNS 解析后再判 IP（防 DNS rebinding 可二次解析）
超时 3s，上限 5MB，不跟随可疑跳转
```

## 反例

错误：`if (filename.endsWith(".jpg"))` 或只看 `multipart Content-Type`。
正确：magic + 白名单 +（图片）重编码。

错误：`new File("/data/upload/" + originalFilename)`。
正确：服务端 UUID key；原名只当展示字段并转义。

错误：`restTemplate.getForObject(userUrl, byte[].class)` 拉头像。
正确：直传 OSS，或 SSRF 加固后的下载器。

错误：上传目录在 web 根下可执行。
正确：OSS/非站点目录，无脚本解释。

## 验证

- 改后缀/Content-Type 的 php 被拒；PNG 头 + php 尾被重编码或拒。
- `../` 文件名不会写出目录外。
- SVG 默认 400；若允许则浏览器当附件且无脚本执行。
- `http://169.254.169.254/latest/meta-data/` 被拒。

## 评审清单

- [ ] 类型白名单 + magic，不信客户端 MIME
- [ ] key/路径服务端生成
- [ ] 默认无 SVG；下载 attachment
- [ ] 无裸 URL 抓取；有则 SSRF 清单
- [ ] 大小三层限制；流式
---
