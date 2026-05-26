# DevSecOps 安全扫描工具全景

> 关注语言：Go, Rust, Python, TypeScript/JavaScript
> 更新时间：2025-05

---

## 一、通用型 SAST 工具（多语言支持）

| 工具 | 开源 | Go | Rust | Python | TS/JS | 特点 |
|------|------|:--:|:----:|:------:|:-----:|------|
| **Semgrep** | ✅ | ✅ | ✅ | ✅ | ✅ | 轻量、YAML 规则可自定义、速度快、CI/CD 友好、Taint 分析 |
| **SonarQube** | ✅（社区版） | ✅ | ✅ | ✅ | ✅ | 生态成熟、规则库丰富、企业首选、Quality Gate |
| **CodeQL** | ✅ | ✅ | ✅ | ✅ | ✅ | GitHub 原生、语义分析精准、跨过程数据流分析 |
| **Snyk Code** | ❌ | ✅ | ✅ | ✅ | ✅ | AI 驱动、低误报、实时 IDE 扫描、DeepCode 修复建议 |
| **GitLab SAST** | ✅ | ✅ | ✅ | ✅ | ✅ | 内置模板、与 GitLab CI 深度集成 |
| **Bearer** | ✅ | ✅ | ❌ | ✅ | ✅ | 数据流分析、隐私/合规导向（PII 检测、GDPR） |

---

## 二、Go 语言专用工具

| 工具 | 开源 | 类型 | 特点 |
|------|------|------|------|
| **gosec** | ✅ | SAST | Go 安全扫描事实标准，Securego 维护，检查 30+ 种漏洞模式 |
| **govulncheck** | ✅ | 漏洞扫描 | Google 官方出品，检测已知漏洞（依赖 + 代码可达性分析） |
| **staticcheck** | ✅ | 静态分析 | 侧重代码质量，包含安全相关规则 |
| **nilaway** | ✅ | 静态分析 | Uber 出品，专注 nil 解引用安全 |

**Go 推荐组合**：`gosec`（安全）+ `govulncheck`（漏洞）+ `staticcheck`（质量）

**CI 示例**：
```yaml
# .github/workflows/security.yml
- name: Run gosec
  uses: securego/gosec@master
  with:
    args: '-fmt sarif -out gosec.sarif ./...'

- name: Run govulncheck
  uses: golang/govulncheck-action@v1
```

---

## 三、Rust 语言专用工具

| 工具 | 开源 | 类型 | 特点 |
|------|------|------|------|
| **cargo-audit** | ✅ | 依赖漏洞 | 检查 Cargo.lock 中的已知 CVE，Rust 安全工作组维护 |
| **cargo-deny** | ✅ | 合规/安全 | 检查许可证、漏洞、crate 来源，可自定义策略 |
| **clippy** | ✅ | 静态分析 | Rust 官方 linter，包含安全相关 lint（如 unsafe 使用） |
| **cargo-geiger** | ✅ | unsafe 检测 | 统计依赖树中的 `unsafe` 代码块，评估风险面 |
| **MIRAI** | ✅ | 符号执行 | Meta 出品，静态分析发现运行时错误（实验性） |

**Rust 推荐组合**：`cargo-audit`（漏洞）+ `cargo-deny`（合规）+ `clippy`（质量/安全）

**CI 示例**：
```yaml
- name: Audit dependencies
  run: cargo install cargo-audit && cargo audit

- name: Run clippy
  run: cargo clippy -- -D warnings

- name: Check deny policies
  run: cargo install cargo-deny && cargo deny check
```

---

## 四、Python 语言专用工具

| 工具 | 开源 | 类型 | 特点 |
|------|------|------|------|
| **Bandit** | ✅ | SAST | Python 安全扫描事实标准，PyCQA 维护，AST 分析，30+ 漏洞模式 |
| **Pysa** | ✅ | SAST（Taint） | Meta 出品，污点追踪分析，适合大型代码库（Instagram 级别） |
| **Ruff（S 规则）** | ✅ | Linter/SAST-lite | Rust 编写极快，实现了 Bandit 规则子集（S 前缀），可部分替代 |
| **pip-audit** | ✅ | SCA | PyPA/Trail of Bits 维护，使用 OSV 数据库，支持自动修复建议 |
| **Safety** | 部分 | SCA | 专有漏洞数据库（免费版受限），Safety 3.x 更新较大 |
| **Dlint** | ✅ | SAST（Flake8） | 检查标准库不安全用法（xml, pickle, eval），活跃度下降 |

**Python 推荐组合**：`Bandit`（安全）+ `pip-audit`（漏洞）+ `Ruff`（质量+基础安全）

**CI 示例**：
```yaml
- name: Run Bandit
  run: pip install bandit && bandit -r src/ -f sarif -o bandit.sarif

- name: Audit dependencies
  run: pip install pip-audit && pip-audit -r requirements.txt

- name: Ruff security checks
  run: pip install ruff && ruff check --select S .
```

---

## 五、TypeScript/JavaScript 专用工具

### SAST 工具

| 工具 | 开源 | 类型 | 特点 |
|------|------|------|------|
| **eslint-plugin-security** | ✅ | Linter | 检测 eval、非字面量 require、SQL 注入、ReDoS |
| **@microsoft/eslint-plugin-sdl** | ✅ | Linter | 微软 SDL 安全规则，基于安全开发生命周期实践 |
| **eslint-plugin-no-unsanitized** | ✅ | Linter | Mozilla 出品，防止未净化的 DOM 操作，XSS 防护 |
| **NodeJsScan / njsscan** | ✅ | SAST | 检测 SQLi、XSS、命令注入、SSRF，njsscan 基于 Semgrep |

### 依赖/供应链安全

| 工具 | 开源 | 类型 | 特点 |
|------|------|------|------|
| **npm audit** | ✅ | SCA | 内置于 npm，扫描 package-lock.json，支持 `npm audit fix` |
| **Socket** | 部分 | 供应链 | 行为分析（非仅 CVE），检测恶意脚本、混淆代码、typosquatting |
| **Retire.js** | ✅ | SCA | 扫描 JS 文件中已知漏洞库，支持浏览器扩展和 Burp Suite |
| **lockfile-lint** | ✅ | 供应链 | 验证 lockfile 安全策略，确保可信注册源，检测 HTTP 降级 |
| **Renovate** | ✅ | 依赖管理 | 自动化依赖更新，安全感知，支持 npm/yarn/pnpm，高度可配置 |

**TS/JS 推荐组合**：`eslint-plugin-security`（SAST）+ `npm audit`/`Socket`（SCA）+ `Semgrep`（深度分析）

**CI 示例**：
```yaml
- name: ESLint security
  run: npx eslint --plugin security --rule 'security/detect-eval-with-expression: error' .

- name: npm audit
  run: npm audit --audit-level=high

- name: Socket analysis
  uses: SocketDev/socket-security-action@v1
```

---

## 六、Secret Detection（密钥泄露检测）

| 工具 | 开源 | 特点 | 适用场景 |
|------|------|------|----------|
| **Gitleaks** | ✅（MIT） | Go 编写极快，150+ 内置规则，支持 pre-commit，SARIF 输出 | CI 首选，速度优先 |
| **TruffleHog** | ✅（AGPL） | 700+ 凭证类型，**验证密钥是否存活**，支持扫描 S3/Slack/Jira | 需要验证密钥有效性 |
| **detect-secrets** | ✅（Apache） | Python 编写，baseline 文件减少误报，插件架构 | Python 生态、pre-commit |

**推荐**：`Gitleaks`（CI 速度快）+ `TruffleHog`（深度扫描+验证）

**CI 示例**：
```yaml
- name: Gitleaks
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 七、Container/Image Scanning（容器镜像扫描）

| 工具 | 开源 | 特点 | 适用场景 |
|------|------|------|----------|
| **Trivy** | ✅（Apache） | 全能：镜像+文件系统+K8s+IaC+SBOM，VEX 支持，离线 DB | 瑞士军刀，一个工具覆盖多场景 |
| **Grype** | ✅（Apache） | 专注漏洞扫描，与 Syft 配合，轻量快速 | 组合式工作流（Syft+Grype） |
| **Clair** | ✅（Apache） | API 驱动服务模式，适合注册中心集成（Quay 原生） | 私有镜像仓库集成 |

**推荐**：`Trivy`（通用首选）或 `Syft + Grype`（SBOM 驱动工作流）

**CI 示例**：
```yaml
- name: Trivy image scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:latest'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
```

---

## 八、IaC Scanning（基础设施即代码扫描）

| 工具 | 开源 | 支持格式 | 特点 |
|------|------|----------|------|
| **Checkov** | ✅（Apache） | Terraform, K8s, Helm, CloudFormation, Dockerfile | 1000+ 策略，图分析理解资源关系，Python/YAML 自定义策略 |
| **Trivy config** | ✅（Apache） | Terraform, CloudFormation, Dockerfile, K8s | tfsec 已合并入 Trivy，Rego 自定义策略 |
| **KICS** | ✅（Apache） | Terraform, Ansible, K8s, Docker, OpenAPI, gRPC | 3000+ 查询，Rego 编写，支持自动修复 |

**推荐**：`Checkov`（策略最丰富）或 `Trivy config`（统一工具链）

**CI 示例**：
```yaml
- name: Checkov
  uses: bridgecrewio/checkov-action@master
  with:
    directory: ./terraform
    framework: terraform
    output_format: sarif
```

---

## 九、SBOM 生成（软件物料清单）

| 工具 | 开源 | 特点 |
|------|------|------|
| **Syft** | ✅（Apache） | 30+ 包生态系统，输出 SPDX/CycloneDX，与 Grype 联动 |
| **cdxgen** | ✅ | CycloneDX 通用生成器，单工具支持 20+ 语言 |
| **Trivy** | ✅（Apache） | `trivy image --format cyclonedx` 一步生成 SBOM |

**CI 示例**：
```yaml
- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    image: myapp:latest
    format: spdx-json
    output-file: sbom.spdx.json
```

---

## 十、DAST（动态应用安全测试）

| 工具 | 开源 | 特点 | 适用场景 |
|------|------|------|----------|
| **OWASP ZAP** | ✅（Apache） | 全功能 Web 扫描器，API 扫描（OpenAPI/GraphQL），Automation Framework | 综合 Web 安全测试 |
| **Nuclei** | ✅（MIT） | 模板驱动，8000+ 社区模板，极快（Go 编写），支持多协议 | CI 中快速已知漏洞扫描 |

**推荐**：`Nuclei`（CI 速度快，已知漏洞）+ `ZAP`（全面测试，Staging 环境）

**CI 示例**：
```yaml
- name: Nuclei scan
  uses: projectdiscovery/nuclei-action@main
  with:
    target: https://staging.example.com
    templates: cves/,misconfiguration/
    sarif-export: nuclei.sarif

- name: ZAP baseline scan
  uses: zaproxy/action-baseline@v0.12.0
  with:
    target: 'https://staging.example.com'
```

---

## 十一、Supply Chain Security（供应链安全）

| 工具 | 开源 | 类型 | 特点 |
|------|------|------|------|
| **Sigstore/Cosign** | ✅（Apache） | 签名/验证 | 无密钥签名（OIDC），容器镜像签名，透明日志 |
| **SLSA Framework** | ✅（Apache） | 构建溯源 | L0-L4 安全级别，GitHub Generator 生成 L3 溯源证明 |
| **OSV-Scanner** | ✅（Apache） | SCA | Google 出品，聚合多源漏洞数据库，引导式修复，多语言 |
| **OpenSSF Scorecard** | ✅（Apache） | 评估 | 评估开源项目安全态势（分支保护、CI/CD、签名发布等） |

**CI 示例**：
```yaml
- name: Sign image with Cosign
  run: cosign sign --yes ${{ env.IMAGE }}

- name: Verify SLSA provenance
  uses: slsa-framework/slsa-verifier/actions/installer@v2.6.0
```

---

## 十二、SCA 通用工具（跨语言依赖扫描）

| 工具 | 开源 | Go | Rust | Python | TS/JS | 特点 |
|------|------|:--:|:----:|:------:|:-----:|------|
| **Trivy** | ✅ | ✅ | ✅ | ✅ | ✅ | 全能扫描器，一个工具覆盖所有 |
| **OSV-Scanner** | ✅ | ✅ | ✅ | ✅ | ✅ | Google 出品，OSV 数据库，引导式修复 |
| **Snyk Open Source** | ❌ | ✅ | ✅ | ✅ | ✅ | 优先级评分、修复 PR、可达性分析 |
| **Dependabot** | ❌（GitHub 免费） | ✅ | ✅ | ✅ | ✅ | GitHub 原生，自动安全更新 PR |
| **OWASP Dependency-Check** | ✅ | ⚠️ | ❌ | ⚠️ | ✅ | NVD 数据库，Java/JS 支持最好 |

---

## 十三、推荐工具组合方案

### 方案 A：开源全栈（零成本）

```
SAST:       Semgrep + 语言专用工具（gosec/Bandit/clippy/eslint-plugin-security）
SCA:        Trivy / OSV-Scanner
Secret:     Gitleaks
Container:  Trivy
IaC:        Checkov
SBOM:       Syft
DAST:       Nuclei + ZAP
Supply:     Cosign + SLSA
```

### 方案 B：GitHub 生态深度集成

```
SAST:       CodeQL（GitHub Advanced Security）
SCA:        Dependabot + OSV-Scanner
Secret:     GitHub Secret Scanning + Gitleaks
Container:  Trivy
IaC:        Checkov
SBOM:       Syft（anchore/sbom-action）
DAST:       Nuclei
Supply:     Cosign + SLSA GitHub Generator
输出:       统一到 GitHub Security Tab（SARIF）
```

### 方案 C：GitLab 生态集成

```
SAST:       GitLab SAST（内置 Semgrep）
SCA:        GitLab Dependency Scanning
Secret:     GitLab Secret Detection
Container:  GitLab Container Scanning（基于 Trivy）
IaC:        KICS / Checkov
SBOM:       CycloneDX
DAST:       GitLab DAST（基于 ZAP）
输出:       统一到 GitLab Security Dashboard
```

---

## 十四、关键趋势（2024-2025）

1. **工具整合**：Trivy 吸收 tfsec，Semgrep 扩展为 SAST+SCA+Secrets 全平台
2. **无密钥签名**：Sigstore OIDC 方式成为标准，无需管理私钥
3. **SBOM 强制化**：美国行政令 + 欧盟 CRA 推动 SBOM 采纳
4. **行为分析 > CVE**：Socket 引领从被动 CVE 扫描到主动行为检测的转变
5. **可达性分析**：Semgrep Supply Chain、OSV-Scanner 只告警代码实际可达的漏洞
6. **AI 修复建议**：Snyk DeepCode AI、SonarQube 提供 AI 生成的修复方案
7. **SARIF 标准化**：几乎所有工具都支持 SARIF 输出，统一接入 GitHub/GitLab Security Tab
