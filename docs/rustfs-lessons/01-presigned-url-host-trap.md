# 经验教训：S3 预签名 URL 与域名替换导致签名失效

**日期：** 2026-04-13  
**环境：** RustFS（私有化部署）  
**问题级别：** P2 - 功能不可用  
**标签：** `s3`, `presigned-url`, `signature`, `host`, `domain`

---

## 问题描述

RustFS 使用内网 IP 生成的预签名 URL 可以正常访问：

```
http://<RUSTFS_INTERNAL_IP>:9000/meeting-minutes/voiceprints/.../xxx.wav?X-Amz-Signature=...
```

将 URL 中的 IP 替换为域名后，访问返回签名错误：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Error><Code>SignatureDoesNotMatch</Code></Error>
```

---

## 根本原因

**S3 预签名 URL 的签名是基于完整请求 URL（包含 Host）计算的。**

签名生成时将 `Host` 头纳入签名范围（`X-Amz-SignedHeaders: host`），一旦 Host 发生变化，服务端重新验签时就会不匹配。

```
生成时：Host = <RUSTFS_INTERNAL_IP>:9000  →  签名 A
访问时：Host = fcg-aim             →  服务端计算签名 B
结果：签名 A ≠ 签名 B  →  SignatureDoesNotMatch
```

**字符串替换 URL 的 Host 部分不会重新计算签名，因此必然失败。**

---

## 解决方案

### 正确做法：生成 URL 时直接使用域名作为 endpoint

在服务端配置 S3 客户端时，`endpoint_url` 填写对外暴露的域名而非内网 IP，这样生成的预签名 URL 中 Host 就已经是域名，客户端无需做任何替换：

```python
# 正确：使用对外域名初始化客户端
s3_client = boto3.client(
    "s3",
    endpoint_url="https://your-domain.example.com",  # 对外域名
    aws_access_key_id="your-access-key",
    aws_secret_access_key="your-secret-key",
    region_name="us-east-1",
)
```

对应到本项目，检查 `.env` 中 RustFS 相关配置：

```ini
# 应填写客户端可访问的域名，而非内网 IP
RUSTFS_ENDPOINT=https://your-domain.example.com
```

### 无效做法（不要使用）

| 方案 | 原因 |
|------|------|
| 生成后做字符串替换 Host | 签名已固定，替换必然导致 SignatureDoesNotMatch |
| Nginx 反向代理透传 Host | 预签名 URL 签名时的 Host 已固定，代理无法修正 |

---

## 核心原理

S3 签名（AWS Signature V4）的计算过程：

```
StringToSign = 算法 + 时间戳 + Scope + Hash(CanonicalRequest)

CanonicalRequest 包含：
  - HTTP Method
  - URI Path
  - Query String
  - Headers（包括 Host）   ← 关键：Host 被纳入签名
  - SignedHeaders
  - Body Hash
```

只要 `X-Amz-SignedHeaders` 包含 `host`（默认都包含），Host 的任何变动都会导致签名验证失败。

---

## 预防措施

1. **环境配置规范**：私有化部署时，S3 客户端的 `endpoint_url` 必须配置为客户端实际访问的地址（域名或 IP），而非存储服务的内网地址。
2. **禁止 URL 字符串替换**：任何对预签名 URL 的 Host 部分做字符串替换的做法都是错误的，会在签名验证时失败。
3. **多环境测试**：内网 IP 直连测试通过不代表域名访问也没问题，上线前需用实际对外域名完整测试预签名 URL 的生成和访问。

---

## 相关链接

- [AWS S3 预签名 URL 文档](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [AWS Signature Version 4](https://docs.aws.amazon.com/general/latest/gr/signature-version-4.html)
