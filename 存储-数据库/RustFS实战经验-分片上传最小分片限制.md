# 经验教训：S3 分片上传最小分片限制

**日期：** 2026-04-15  
**环境：** RustFS（私有化部署）  
**问题级别：** P2 - 功能不可用  
**标签：** `s3`, `multipart-upload`, `entity-too-small`, `5mb-limit`, `compatibility`

---

## 问题描述

客户端完成分片上传时，调用 `POST /api/v1/files/multipart/complete` 接口返回 500 错误：

```json
{"code": 4003, "message": "完成分片上传失败，请稍后重试", "data": null}
```

服务端日志：

```
S3Error: S3 operation failed; code: EntityTooSmall,
message: Your proposed upload is smaller than the minimum allowed object size.
```

---

## 根本原因

**S3 协议规定：除最后一个分片外，其他每个分片必须 ≥ 5MB。**

本次上传共 3 个分片，前两个分片均为 2MB，不满足最小限制：

| 分片 | 大小 | 是否合规 |
|------|------|----------|
| Part 1 | 2MB | ❌ 非最后分片，< 5MB |
| Part 2 | 2MB | ❌ 非最后分片，< 5MB |
| Part 3 | 0.xMB | ✅ 最后分片，无限制 |

---

## 为什么 upload_part 阶段不报错

这是 S3 协议的设计决定，不是 RustFS 的 bug：

- `upload_part` 阶段：服务端只知道当前分片的编号和大小，**不知道总共有多少片**，无法判断当前片是否为最后一片，因此不做大小校验。
- `complete` 阶段：客户端提交完整的 parts 列表，服务端此时才能确定哪些是中间片，并对非最后分片执行 ≥ 5MB 的校验。

---

## 为什么在 OBS / COS / OSS 上没有问题

各厂商对 S3 标准的最小分片限制实现不同：

| 存储服务 | 最小分片大小 | 说明 |
|---------|------------|------|
| AWS S3 | 5MB | S3 官方标准 |
| RustFS | 5MB | 严格遵循 S3 标准 |
| 华为 OBS | 100KB | 厂商主动放宽 |
| 腾讯 COS | 1MB | 厂商主动放宽 |
| 阿里 OSS | 100KB | 厂商主动放宽 |

客户端的分片策略在 OBS/COS/OSS 上"碰巧"能跑通，是因为这些厂商降低了门槛做了兼容，**并不代表客户端逻辑符合 S3 规范**。切换到严格遵循标准的 RustFS 后，问题暴露出来。

---

## 解决方案

### 推荐：客户端调整分片策略（成本最低）

- 文件 **< 10MB** 时，直接使用普通上传接口，不走分片上传
- 文件需要分片时，每片大小设置为 **≥ 5MB**（最后一片除外）

### 备选：服务端在 complete 时转换

收到 `complete` 请求后检测到 `EntityTooSmall` 错误，自动 abort 原分片上传，将所有分片数据拼合后走普通 `put_object` 重新上传，客户端无感知。（实现成本较高，且需要临时缓冲完整文件数据）

---

## 预防措施

1. **多云兼容测试**：新功能上线前，在 RustFS 私有化环境做完整测试，不能只依赖 OBS/COS/OSS 测试通过就认为没问题。
2. **客户端分片大小规范**：客户端分片上传组件应统一遵循 S3 标准（≥ 5MB/片），不依赖各厂商的兼容性差异。
3. **服务端前置校验（可选）**：在 `upload_part` 接口记录分片大小，或在文档中明确告知客户端最小分片要求。

---

## 相关链接

- [AWS S3 分片上传文档](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [S3 Multipart Upload Limits](https://docs.aws.amazon.com/AmazonS3/latest/userguide/qfacts.html)
- [RustFS 文档](https://github.com/rustfs/rustfs)
