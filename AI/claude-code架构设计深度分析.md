# Claude Code Agent 架构设计深度分析

> 基于 v2.1.88 反编译源码的架构分析，聚焦 agent 系统设计及其与普遍框架的差异。

---

## 一、整体架构：单进程 + 递归 query 循环

**核心循环位置**：`src/query.ts:307` — `while (true)` 循环

整个 agent 的执行本质是一个 `while(true)` 循环，每次迭代：

```
┌─ 1. snip/microcompact/contextCollapse 压缩历史 (query.ts:400-445)
├─ 2. autoCompact 检查是否需要全量压缩 (query.ts:454-543)
├─ 3. 调用模型 API 流式获取响应 (query.ts:659-863)
├─ 4. 收集 tool_use blocks (query.ts:829-835)
├─ 5. 执行工具调用 (query.ts:1380-1408)
├─ 6. 注入 attachments（内存、skill、通知等）(query.ts:1580-1628)
├─ 7. 检查 maxTurns 限制 (query.ts:1705-1711)
└─ 8. 组装新 messages，continue 下一轮 (query.ts:1715-1728)
```

**关键入口**：
- `src/query.ts:219` — `query()` 函数，AsyncGenerator，对外暴露的唯一入口
- `src/query.ts:241` — `queryLoop()` 内部实现，包含完整的 while(true) 循环
- `src/query.ts:200-217` — `State` 类型，循环的可变状态定义

**循环退出条件**（`return { reason: ... }`）：
- `needsFollowUp === false`（模型没有发出 tool_use）→ `query.ts:1062`
- 用户中断（abort）→ `query.ts:1015` / `query.ts:1485`
- maxTurns 达到上限 → `query.ts:1705`
- prompt_too_long 无法恢复 → `query.ts:1175`
- hook 阻止继续 → `query.ts:1519`

**循环继续条件**（`state = next; continue`）：
- 有 tool_use 需要下一轮 → `query.ts:1715`（`reason: 'next_turn'`）
- reactive compact 恢复 → `query.ts:1148`（`reason: 'reactive_compact_retry'`）
- max_output_tokens 恢复 → `query.ts:1230`（`reason: 'max_output_tokens_recovery'`）
- stop hook blocking → `query.ts:1289`（`reason: 'stop_hook_blocking'`）
- context collapse drain → `query.ts:1099`（`reason: 'collapse_drain_retry'`）
- token budget continuation → `query.ts:1321`（`reason: 'token_budget_continuation'`）

**与普遍设计的区别**：没有 DAG/状态机编排层。循环本身就是编排——模型通过是否发出 tool_use 来决定是否继续。

---

## 二、子 Agent 模型：同构递归

**核心文件**：
- `src/tools/AgentTool/runAgent.ts:248-860` — `runAgent()` 函数，所有子 agent 的统一入口
- `src/tools/AgentTool/agentToolUtils.ts:70-116` — `filterToolsForAgent()` 工具集裁剪
- `src/tools/AgentTool/agentToolUtils.ts:122-225` — `resolveAgentTools()` 工具解析
- `src/tools/AgentTool/agentToolUtils.ts:508-686` — `runAsyncAgentLifecycle()` 异步 agent 生命周期

**子 agent 也调用同一个 `query()`**：`runAgent.ts:748`

```typescript
for await (const message of query({
  messages: initialMessages,
  systemPrompt: agentSystemPrompt,
  userContext: resolvedUserContext,
  systemContext: resolvedSystemContext,
  canUseTool,
  toolUseContext: agentToolUseContext,
  querySource,
  maxTurns: maxTurns ?? agentDefinition.maxTurns,
})) { ... }
```

**同步 vs 异步隔离**：`runAgent.ts:524-528`
```typescript
const agentAbortController = override?.abortController
  ? override.abortController
  : isAsync
    ? new AbortController()  // 异步：独立生命周期
    : toolUseContext.abortController  // 同步：共享父级
```

**权限覆盖逻辑**：`runAgent.ts:416-498` — `agentGetAppState()` 闭包

**内置 agent 定义**：
- `src/tools/AgentTool/built-in/exploreAgent.ts` — Explore（只读搜索，Haiku 模型）
- `src/tools/AgentTool/built-in/planAgent.ts` — Plan（只读规划）
- `src/tools/AgentTool/built-in/generalPurposeAgent.ts` — 通用 agent
- `src/tools/AgentTool/built-in/verificationAgent.ts` — 验证 agent
- `src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts` — 文档查询 agent

**Agent 定义加载**：`src/tools/AgentTool/loadAgentsDir.ts` — 从 `.claude/agents/` 目录加载自定义 agent

**与普遍设计的区别**：
- LangGraph/CrewAI：agent 是预定义的异构角色（researcher/coder/reviewer），通过图/流程编排
- Claude Code：agent 是同构的 query() 循环实例，只是约束不同（工具集、权限、系统提示）

---

## 三、Coordinator 模式：纯 prompt 驱动的编排

**核心文件**：`src/coordinator/coordinatorMode.ts`

- `coordinatorMode.ts:36-41` — `isCoordinatorMode()` 判断是否启用
- `coordinatorMode.ts:80-109` — `getCoordinatorUserContext()` 注入 worker 工具列表到 userContext
- `coordinatorMode.ts:111-369` — `getCoordinatorSystemPrompt()` 完整的 coordinator 系统提示

**关键设计**：coordinator 不是代码编排器，而是一段 ~4000 字的系统提示，教模型如何：
- 用 `Agent` 工具 spawn worker（`coordinatorMode.ts:133`）
- 用 `SendMessage` 继续已有 worker（`coordinatorMode.ts:134`）
- 用 `TaskStop` 停止 worker（`coordinatorMode.ts:135`）
- 遵循 Research → Synthesis → Implementation → Verification 工作流（`coordinatorMode.ts:200-209`）
- 写好 worker prompt（`coordinatorMode.ts:252-335`）

**模式切换**：`coordinatorMode.ts:49-78` — `matchSessionMode()` 恢复 session 时匹配模式

**与普遍设计的区别**：
- LangGraph：编排是代码定义的图（节点 + 边 + 条件路由）
- CrewAI：编排是预定义的 Process（sequential/hierarchical）
- Claude Code：编排是 prompt 中的建议，模型自主决策何时并行、何时串行——编排是涌现的而非声明式的

---

## 四、Fork 子 Agent：Prompt Cache 共享优化

**核心文件**：`src/tools/AgentTool/forkSubagent.ts`

- `forkSubagent.ts:32-39` — `isForkSubagentEnabled()` feature gate
- `forkSubagent.ts:60-71` — `FORK_AGENT` 定义（`tools: ['*']`, `permissionMode: 'bubble'`, `model: 'inherit'`）
- `forkSubagent.ts:107-169` — `buildForkedMessages()` 构造 cache 友好的消息序列
- `forkSubagent.ts:171-198` — `buildChildMessage()` fork 子 agent 的指令模板
- `forkSubagent.ts:78-89` — `isInForkChild()` 防止递归 fork

**Fork agent 的实际运行**：`src/utils/forkedAgent.ts` — `runForkedAgent()` 函数

**Cache 共享的关键**：`forkSubagent.ts:142-148`
```typescript
// 所有 fork 子 agent 的 tool_result 用相同占位符，确保字节级一致
const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'

const toolResultBlocks = toolUseBlocks.map(block => ({
  type: 'tool_result',
  tool_use_id: block.id,
  content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }],
}))
```

**设计目标**：多个并行 fork 子 agent 的 API 请求前缀（system prompt + tools + 历史消息 + tool_results）完全字节一致，只有最后一个 directive text block 不同，从而最大化 prompt cache 命中率。

**与普遍设计的区别**：其他框架几乎不考虑 API 层的 prompt cache 优化。Claude Code 把 cache 命中率作为架构级约束，影响了消息构造、工具定义、甚至 compaction 策略。

---

## 五、Context Compaction：无限对话的关键

**核心文件目录**：`src/services/compact/`

| 文件 | 位置 | 作用 |
|------|------|------|
| `compact.ts:387-763` | `compactConversation()` | 主压缩函数 |
| `compact.ts:772-1106` | `partialCompactConversation()` | 部分压缩（保留选定消息） |
| `compact.ts:1136-1396` | `streamCompactSummary()` | 调用 LLM 生成摘要 |
| `compact.ts:1415-1464` | `createPostCompactFileAttachments()` | 压缩后恢复文件内容 |
| `compact.ts:1494-1534` | `createSkillAttachmentIfNeeded()` | 压缩后恢复 skill |
| `compact.ts:1568-1599` | `createAsyncAgentAttachmentsIfNeeded()` | 压缩后恢复 agent 状态 |
| `autoCompact.ts:72-91` | `getAutoCompactThreshold()` | 阈值计算 |
| `autoCompact.ts:160-239` | `shouldAutoCompact()` | 是否需要自动压缩 |
| `autoCompact.ts:241-351` | `autoCompactIfNeeded()` | 自动压缩入口 |
| `microCompact.ts` | — | 微压缩（tool_result 级别） |
| `sessionMemoryCompact.ts` | — | session memory 压缩 |
| `prompt.ts` | — | 压缩用的 prompt 模板 |

**在主循环中的调用位置**：`query.ts:454` — `deps.autocompact(...)`

**阈值逻辑**：`autoCompact.ts:63-64`
```typescript
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
// threshold = effectiveContextWindow - 13K - maxOutputTokens(20K)
```

**压缩后状态恢复**（`compact.ts:532-585`）：
1. 最近读取的文件（最多 5 个，50K token 预算）
2. Plan 文件
3. Plan mode 指令
4. 已调用的 skill 内容（每个 skill 最多 5K token，总预算 25K）
5. Deferred tools delta
6. Agent listing delta
7. MCP instructions delta
8. SessionStart hooks 结果

**多层压缩策略**（按优先级）：
1. **Snip**（`query.ts:400`）— 裁剪历史中的冗余 tool_result
2. **Microcompact**（`query.ts:414`）— tool_result 级别的精细压缩
3. **Context Collapse**（`query.ts:441`）— 分组折叠（实验性）
4. **Session Memory Compact**（`autoCompact.ts:288`）— 基于 session memory 的压缩
5. **Full Compact**（`autoCompact.ts:313`）— 完整的 LLM 摘要压缩
6. **Reactive Compact**（`query.ts:1119`）— API 返回 prompt_too_long 后的被动压缩

**与普遍设计的区别**：
- 大多数框架要么不处理长对话（靠截断），要么用简单的滑动窗口
- Claude Code 的 compaction 是有状态恢复的——压缩后模型不会"失忆"关键文件内容和进行中的任务
- 压缩本身也通过 fork agent 路径发送，复用主对话的 prompt cache

---

## 六、权限系统：分层 + 冒泡

**核心文件**：
- `src/types/permissions.ts` — 权限类型定义
- `src/utils/permissions/` — 权限工具目录
- `src/utils/permissions/PermissionMode.ts` — 权限模式定义
- `src/hooks/toolPermission/` — 工具权限 hooks

**权限模式层级**：
```
bypassPermissions > acceptEdits > auto > plan > default
```

**在 runAgent 中的实现**：`runAgent.ts:416-498` — `agentGetAppState()` 闭包

关键逻辑：
- `runAgent.ts:427-429` — 父级 `bypassPermissions`/`acceptEdits`/`auto` 不被子 agent 覆盖
- `runAgent.ts:443-449` — 异步 agent 设置 `shouldAvoidPermissionPrompts`（无法显示 UI）
- `runAgent.ts:456-463` — `bubble` 模式 + 异步 → `awaitAutomatedChecksBeforeDialog`
- `runAgent.ts:469-479` — `allowedTools` 替换 session 级规则，保留 SDK 级规则

**Handoff 安全分类**：`agentToolUtils.ts:389-481` — `classifyHandoffIfNeeded()`
- 异步 agent 完成后，用分类器检查其操作是否违反安全策略
- 如果分类器标记为危险，在结果前注入安全警告

---

## 七、Swarm（多 Agent 协作）

**核心文件目录**：`src/utils/swarm/`

| 文件 | 作用 |
|------|------|
| `backends/InProcessBackend.ts` | 同进程 teammate 后端 |
| `backends/TmuxBackend.ts` | tmux 分屏后端 |
| `backends/ITermBackend.ts` | iTerm2 分屏后端 |
| `backends/detection.ts` | 自动检测可用后端 |
| `backends/registry.ts` | 后端注册表 |
| `spawnInProcess.ts` | 进程内 spawn teammate |
| `spawnUtils.ts` | spawn 工具函数 |
| `leaderPermissionBridge.ts` | leader 权限桥接 |
| `permissionSync.ts` | 权限同步 |
| `reconnection.ts` | 断线重连 |
| `teammateInit.ts` | teammate 初始化 |
| `teammateModel.ts` | teammate 模型选择 |
| `teammatePromptAddendum.ts` | teammate 额外 prompt |

**Team 工具**：
- `src/tools/TeamCreateTool/` — 创建 team
- `src/tools/TeamDeleteTool/` — 删除 team
- `src/tools/SendMessageTool/` — agent 间消息传递

**Task 系统**：
- `src/tasks/LocalAgentTask/` — 本地 agent 任务（异步 agent 的状态管理）
- `src/tasks/InProcessTeammateTask/` — 进程内 teammate 任务
- `src/tasks/LocalShellTask/` — 后台 shell 任务
- `src/tasks/RemoteAgentTask/` — 远程 agent 任务

---

## 八、工具集的动态裁剪

**核心文件**：
- `src/tools/AgentTool/agentToolUtils.ts:70-116` — `filterToolsForAgent()`
- `src/constants/tools.ts` — 定义各类工具白名单/黑名单

**裁剪规则**（`agentToolUtils.ts:83-115`）：
```
1. MCP 工具 (mcp__*) → 所有 agent 都允许
2. ExitPlanMode + plan 模式 → 允许
3. ALL_AGENT_DISALLOWED_TOOLS → 所有 agent 禁用
4. CUSTOM_AGENT_DISALLOWED_TOOLS → 自定义 agent 额外禁用
5. 异步 agent → 只允许 ASYNC_AGENT_ALLOWED_TOOLS 白名单内的
6. In-process teammate → 额外允许 AgentTool + IN_PROCESS_TEAMMATE_ALLOWED_TOOLS
```

**Explore agent 的裁剪示例**（`exploreAgent.ts:64-83`）：
- `disallowedTools`: Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit
- `omitClaudeMd: true` — 省略 CLAUDE.md（`runAgent.ts:389-398`）
- 省略 gitStatus（`runAgent.ts:404-410`）
- `model: 'haiku'`（外部用户）/ `'inherit'`（内部用户）

**Agent 自定义 MCP 服务器**：`runAgent.ts:95-218` — `initializeAgentMcpServers()`
- Agent 可在 frontmatter 中定义自己的 MCP 服务器
- 支持引用已有服务器（字符串名称）或内联定义
- Agent 结束时自动清理新创建的连接

---

## 九、其他关键基础设施

| 模块 | 位置 | 作用 |
|------|------|------|
| 状态管理 | `src/state/AppStateStore.ts` | 全局应用状态 |
| 工具执行 | `src/services/tools/toolOrchestration.ts` | `runTools()` 并行/串行执行工具 |
| 流式工具执行 | `src/services/tools/StreamingToolExecutor.ts` | 边流式边执行工具（模型还在输出时就开始执行已完成的 tool_use） |
| API 调用 | `src/services/api/claude.ts` | `queryModelWithStreaming()` |
| Session 恢复 | `src/tools/AgentTool/resumeAgent.ts` | 恢复中断的 agent |
| Agent 摘要 | `src/services/AgentSummary/agentSummary.ts` | 后台 agent 进度摘要 |
| Transcript 记录 | `src/utils/sessionStorage.ts` | `recordSidechainTranscript()` |
| Token 预算 | `src/query/tokenBudget.ts` | +500K auto-continue |
| Stop hooks | `src/query/stopHooks.ts` | 模型停止时的 hook 处理 |
| QueryEngine | `src/QueryEngine.ts` | 上层封装（REPL/SDK 入口） |
| Skill 系统 | `src/skills/` | 可加载的技能模块 |
| Plugin 系统 | `src/plugins/` | 插件加载和管理 |
| Memory 系统 | `src/utils/memory/` | 持久化记忆 |

---

## 十、总结：设计哲学对比

| 维度 | 普遍框架（LangGraph/CrewAI/AutoGen） | Claude Code |
|------|------|------|
| 编排 | 代码定义的图/状态机 | 模型自主决策 + prompt 引导 |
| Agent 关系 | 异构角色（researcher/coder/reviewer） | 同构递归（同一个 query 循环，不同约束） |
| 上下文管理 | 截断/滑动窗口 | 6 层压缩策略 + 有状态恢复 |
| 性能优化 | 几乎不考虑 | Prompt cache 是架构级约束 |
| 权限 | 全有或全无 | 分层冒泡 + 动态裁剪 + 安全分类器 |
| 工具 | 静态注册 | 按 agent 类型/模式/来源动态裁剪 |
| 协作 | 消息传递/共享内存 | 进程内 + tmux 分屏 + 权限同步 |
| 错误恢复 | 通常不处理 | 多层自动恢复（escalate → compact → truncate → surface） |

**核心设计理念**：把尽可能多的决策权交给模型本身，代码层只负责约束（权限、工具集、token 预算）和优化（cache 共享、compaction）。这是一个"模型优先"的架构，而非"编排优先"的架构。
