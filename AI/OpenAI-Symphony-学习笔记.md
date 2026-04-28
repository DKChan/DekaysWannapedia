# OpenAI Symphony 学习笔记

> 原文: https://openai.com/index/open-source-codex-orchestration-symphony/
> GitHub: https://github.com/openai/symphony
> 日期: 2026-04-28

---

## 什么是 Symphony?

**Symphony** 是 OpenAI 开源的 AI 编程 Agent 编排框架，用于将项目工作转化为**独立的、自主的实现运行**，让团队从"监督编码 Agent"转变为"管理工作本身"。

---

## 核心设计理念

### 1. 从"监督 Agent"到"管理工作"

| 传统方式 | Symphony 方式 |
|---------|--------------|
| 工程师监督 AI 写代码 | 工程师管理任务队列 |
| 手动运行 Agent | 自动派发任务给 Agent |
| 人工审查每一步 | Agent 自动提供工作证明 |

### 2. 工作流自动化

```
Linear 看板 → Symphony 监控 → Agent 派生 → 任务执行 → 工作证明 → PR 合并
```

**工作证明包括:**
- CI 状态
- PR 审查反馈
- 复杂度分析
- 演示视频

---

## 架构组件

### 核心组件 (8个)

```
┌─────────────────────────────────────────────────────────┐
│                    Symphony Service                      │
├─────────────┬─────────────┬─────────────┬───────────────┤
│   Workflow  │    Config   │   Issue     │  Orchestrator │
│   Loader    │    Layer    │   Tracker   │               │
│  (配置加载)  │  (配置解析)  │   Client    │   (调度核心)   │
├─────────────┴─────────────┴─────────────┴───────────────┤
│   Workspace Manager   │   Agent Runner   │   Status     │
│     (工作空间管理)      │   (Agent 启动)    │   Surface    │
│                       │                  │  (状态展示)   │
└─────────────────────────────────────────────────────────┘
```

### 组件职责

| 组件 | 职责 |
|-----|------|
| **Workflow Loader** | 读取 `WORKFLOW.md`，解析 YAML Front Matter 和 Prompt 模板 |
| **Config Layer** | 类型化配置获取、默认值、环境变量解析、验证 |
| **Issue Tracker Client** | 获取候选 Issue、状态刷新、数据规范化 |
| **Orchestrator** | 轮询调度、运行时状态管理、派发/重试/停止决策 |
| **Workspace Manager** | Issue 到工作空间路径映射、生命周期钩子管理 |
| **Agent Runner** | 创建工作空间、构建 Prompt、启动 Coding Agent |
| **Status Surface** | 运行时状态展示 (可选) |
| **Logging** | 结构化日志输出 |

---

## 配置系统 (WORKFLOW.md)

### 文件格式

```markdown
---
tracker:
  kind: linear
  api_key: $LINEAR_API_KEY
  project_slug: my-project
  active_states: ["Todo", "In Progress"]
  terminal_states: ["Closed", "Done"]

polling:
  interval_ms: 30000

workspace:
  root: ~/symphony_workspaces

hooks:
  after_create: |
    git clone $REPO_URL .
  before_run: |
    git pull origin main
  after_run: |
    echo "Run completed"

agent:
  max_concurrent_agents: 10
  max_turns: 20
  max_retry_backoff_ms: 300000

codex:
  command: codex app-server
  approval_policy: auto
  turn_timeout_ms: 3600000
---

# Prompt Template

你正在处理 Linear Issue: {{ issue.identifier }}
标题: {{ issue.title }}
描述: {{ issue.description }}

请完成以下任务...
```

### 配置层级

```
1. Policy Layer (repo-defined)     → WORKFLOW.md prompt
2. Configuration Layer (typed)     → YAML front matter
3. Coordination Layer              → Orchestrator 调度
4. Execution Layer                 → Workspace + Agent
5. Integration Layer               → Linear adapter
6. Observability Layer             → Logs + Dashboard
```

---

## 调度机制

### Issue 生命周期状态

```
Unclaimed → Claimed → Running → [RetryQueued] → Released
```

### 派发规则

**候选 Issue 必须满足:**
- ✅ 有 id, identifier, title, state
- ✅ 状态在 active_states 中
- ✅ 不在 running 中
- ✅ 不在 claimed 中
- ✅ 有可用并发槽位
- ✅ Todo 状态的 Issue 没有非终止状态的 blocker

**排序优先级:**
1. priority 升序 (数字小优先)
2. created_at 最早优先
3. identifier 字典序

### 重试机制

| 场景 | 延迟策略 |
|-----|---------|
| 正常退出后的连续运行 | 固定 1 秒 |
| 失败后的重试 | 指数退避: `min(10000 * 2^(attempt-1), max_backoff)` |

---

## 工作空间管理

### 安全不变式

```
Invariant 1: Agent 只能在 Issue 工作空间路径内运行
Invariant 2: 工作空间路径必须在 workspace.root 下
Invariant 3: 工作空间目录名使用 sanitized identifier ([A-Za-z0-9._-])
```

### 路径结构

```
<workspace.root>/<sanitized_issue_identifier>/
```

### 生命周期钩子

| 钩子 | 触发时机 | 失败行为 |
|-----|---------|---------|
| `after_create` | 工作空间首次创建 | 中止创建 |
| `before_run` | 每次运行前 | 中止当前尝试 |
| `after_run` | 每次运行后 | 记录并忽略 |
| `before_remove` | 工作空间删除前 | 记录并忽略 |

---

## Agent Runner 协议

### 启动参数

```bash
bash -lc "<codex.command>"  # 在工作空间目录执行
```

### Session 生命周期

```
1. 创建工作空间
2. 运行 before_run 钩子
3. 启动 app-server 子进程
4. 初始化 Session (thread_id + turn_id)
5. 发送 Prompt 启动第一轮
6. 流式处理 Agent 更新
7. 检查 Issue 状态
8. 如仍活跃且未达 max_turns，继续下一轮
9. 停止 Session
10. 运行 after_run 钩子
```

### 连续运行 (Continuation)

```
第一轮: 发送完整 Prompt
后续轮: 发送 continuation guidance (不重复原始 Prompt)
```

---

## 与 Codex 集成

### Codex App-Server 协议

- Symphony 启动 `codex app-server` 子进程
- 通过 stdio 进行 JSON Line 协议通信
- 支持 approval policy、sandbox policy 配置
- 支持 client-side tools (如 `linear_graphql`)

### Token 统计

```
- codex_input_tokens
- codex_output_tokens
- codex_total_tokens
- 聚合统计所有 Session
```

---

## 可观测性

### 日志要求

```
必须包含的上下文字段:
- issue_id
- issue_identifier
- session_id

格式: key=value 格式，包含动作结果和失败原因
```

### 状态 API (可选)

```
GET /api/v1/state      → 系统状态摘要
GET /api/v1/<issue>    → Issue 详情
POST /api/v1/refresh   → 立即触发轮询
```

---

## 安全考虑

### 信任边界

- 实现必须明确声明其信任边界
- 支持高信任环境 (auto-approve) 或严格环境
- 工作空间隔离是基线控制，不是完整沙箱

### 强化措施

- 使用专用 OS 用户运行
- 限制工作空间根目录权限
- 使用容器/VM 额外隔离
- 过滤可派发的问题范围
- 限制 `linear_graphql` 工具权限

---

## 实现选项

### 选项 1: 自己实现

```
告诉你的 Coding Agent:
"按照以下规范实现 Symphony:"
https://github.com/openai/symphony/blob/main/SPEC.md
```

### 选项 2: 使用参考实现

```
Elixir 实现:
https://github.com/openai/symphony/tree/main/elixir
```

---

## 关键设计决策

| 决策 | 说明 |
|-----|------|
| **无持久化数据库** | 重启后通过轮询 Tracker 和重用工作空间恢复 |
| **单权威 Orchestrator** | 所有状态变更串行化，避免重复派发 |
| **动态配置重载** | WORKFLOW.md 修改自动生效，无需重启 |
| **工作空间持久化** | 成功运行后不自动删除，支持连续运行 |
| **Agent 负责写操作** | Symphony 只读 Tracker，写操作由 Agent 完成 |

---

## 与 Harness Engineering 的关系

**Symphony 是 Harness Engineering 的下一步:**

- **Harness Engineering**: 管理 Coding Agent
- **Symphony**: 管理需要完成的工作

参考: https://openai.com/index/harness-engineering/

---

## 总结

Symphony 提供了一个**生产级的 Agent 编排框架**，核心优势:

1. **声明式配置** - WORKFLOW.md 定义完整工作流
2. **安全隔离** - 每个 Issue 独立工作空间
3. **可靠调度** - 并发控制、重试、状态协调
4. **可观测** - 结构化日志、状态 API
5. **可扩展** - 支持自定义 hooks、client-side tools

适合需要**规模化运行 Coding Agent** 的团队，从"手动监督"转向"自动化管理"。
