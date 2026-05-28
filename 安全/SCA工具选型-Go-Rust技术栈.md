# SCA 工具选型：Go/Rust 技术栈的现代化方案

> OWASP Dependency-Check 在 Go/Rust 技术栈中已不推荐使用
> 更新时间：2025-05

---

## 为什么抛弃 OWASP Dependency-Check

Dependency-Check 骨子里是为 Java/多层嵌套依赖设计的：

- 对 Go Mod 和 Rust Cargo 的支持远不如 Maven 丝滑
- CPE 模糊匹配在 Go/Rust 轻量级依赖里产生大量虚警
- 扫描速度慢，不适合现代 CI/CD 节奏

---

## 一、AI 时代的新一代安全工具

传统工具只管报漏洞，新一代工具强调"上下文理解"和"自动化修复"。

### 1. Snyk（现代 SCA 领头羊）

- **AI 赋能（DeepCode AI）**：不仅扫依赖，还能扫 Go/Rust 源码（SAST）
- **Reachability Analysis（可达性分析）**：如果 Go 包有漏洞但你没调用漏洞函数，自动降低优先级，拒绝"误报轰炸"
- **自动修复**：AI 评估升级是否破坏现有代码，直接提 PR
- **原生支持**：对 `go.mod` 和 `Cargo.toml` 解析精准

### 2. GitHub Dependabot / GitLab Security

- 零安装，直接集成在代码托管平台
- 微软安全情报 + AI 补丁建议
- 检测到 Go/Rust 依赖有已知 CVE → 自动生成修复 PR
- 对大部分 Go/Rust 项目已经足够

---

## 二、Go & Rust 原生现代化工具

### 1. Govulncheck（Go 官方降维打击）

- Go 官方团队出品
- **杀手锏**：分析编译后的 AST 和调用链，只有代码真正调用了漏洞函数才报错
- 误报率几乎为零
- 速度恐怖，几秒钟跑完

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```

### 2. Cargo-audit（Rust 生态标配）

- 基于 RustSec（Rust 专门漏洞数据库）
- 完美契合 `Cargo.lock`
- RustSec 更新和审核效率极高，比 NVD 快，无 CPE 误报

```bash
cargo install cargo-audit
cargo audit
```

---

## 三、云原生与 SBOM 持续监控架构

Go/Rust 常编译为静态二进制 → Docker 镜像（scratch/distroless），现代安全倾向"镜像扫描"+"SBOM 持续监控"。

### 1. Trivy（云原生绝对顶流）

- 扫 Go/Rust 应用依赖 + 容器镜像 + OS 依赖 + K8s YAML + Terraform
- 扫描速度秒级
- 适合放在 CI/CD 的 `docker build` 之后

```bash
trivy image myapp:latest
trivy fs --scanners vuln .
```

### 2. SBOM + Dependency-Track 模式（下一代架构）

> Dependency-Track 是 Dependency-Check 的精神续作，但架构完全不同

**工作流**：

1. **编译时**：CI/CD 中用 `syft` 生成 SBOM（`bom.json`）
2. **运行时**：SBOM 传给 OWASP Dependency-Track 后台
3. **持续监控**：新 CVE 爆发时自动匹配已有 SBOM，立即报警

**优势**：一次编译，持续监控。流水线不需要每次查漏洞库，不影响编译速度。

```bash
# 生成 SBOM
syft packages dir:. -o cyclonedx-json > sbom.json

# 上传到 Dependency-Track
curl -X POST "https://dtrack.internal/api/v1/bom" \
  -H "X-Api-Key: $DTRACK_API_KEY" \
  -F "project=$PROJECT_UUID" \
  -F "bom=@sbom.json"
```

---

## 四、落地建议

### 场景 A：公有云极致开发体验

```
GitHub Dependabot + Govulncheck (Go) + Cargo-audit (Rust)
```

### 场景 B：私有化部署/涉密内网

```
CI: syft 生成 SBOM → 内网 Dependency-Track
或: Trivy 离线版扫描二进制/镜像
```

### 场景 C：预算充足，AI 全家桶

```
Snyk（SAST + SCA 一网打尽）
```

---

## 五、工具对比总结

| 维度 | OWASP Dependency-Check | Govulncheck | Cargo-audit | Trivy | Snyk |
|------|:----------------------:|:-----------:|:-----------:|:-----:|:----:|
| Go 支持 | ⚠️ 勉强 | ✅ 官方 | — | ✅ | ✅ |
| Rust 支持 | ❌ | — | ✅ 官方 | ✅ | ✅ |
| 可达性分析 | ❌ | ✅ | ❌ | ❌ | ✅ |
| 误报率 | 高 | 极低 | 低 | 低 | 极低 |
| 速度 | 慢（分钟级） | 秒级 | 秒级 | 秒级 | 秒级 |
| 离线支持 | ✅ | ✅ | ✅ | ✅ | ❌ |
| AI 修复 | ❌ | ❌ | ❌ | ❌ | ✅ |
| 容器扫描 | ❌ | ❌ | ❌ | ✅ | ✅ |
| 开源 | ✅ | ✅ | ✅ | ✅ | ❌ |
