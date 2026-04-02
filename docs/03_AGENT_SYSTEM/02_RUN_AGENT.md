# 02 - runAgent 执行流程

> 学习 Agent 执行的核心函数 runAgent() 的完整执行流程

## 1. 学习目标

学完本章后，你将理解：

- runAgent() 启动 Agent 后的完整执行流程
- 同步 Agent 和异步 Agent 在接收消息时的差异
- runAgent() 的核心机制与执行流程

## 2. 概述

`runAgent()` 是 Agent 执行的核心函数，位于 `src/tools/AgentTool/runAgent.ts:248-860`。

### 2.1 runAgent 是什么

`runAgent()` 是一个 **AsyncGenerator**，它：

1. **启动一个独立的 AI Agent**
2. **逐条 yield 出结构化的 Message 对象**
3. **每条 Message 包含 Agent 的完整执行信息**

```typescript
// 返回类型
export async function* runAgent({...}): AsyncGenerator<Message, void>

// Message 的结构
interface Message {
  type: 'assistant' | 'user' | 'system' | 'stream_event' | ...
  message: { content: [...] }
  uuid: string
  timestamp: string
  // ...
}
```

### 2.2 同步 vs 异步 Agent

**关键问题**：如何控制生成同步还是异步的 Agent？

在 `AgentTool.tsx` 中，通过 `shouldRunAsync` 变量决定：

```typescript
// AgentTool.tsx:567
const shouldRunAsync = (
  run_in_background === true ||           // 用户传入参数
  selectedAgent.background === true ||     // Agent 定义强制后台
  isCoordinator ||                       // Coordinator 模式
  forceAsync ||                          // Fork 实验模式
  isAssistantMode
) && !isBackgroundTasksDisabled;
```

| 控制方式 | 说明 |
|---------|------|
| `run_in_background: true` | 用户明确要求后台执行 |
| `Agent.background: true` | Agent 定义强制后台（如 Verification Agent） |
| Coordinator 模式 | 自动后台执行 |
| Fork 模式 | 自动后台执行 |

### 2.3 消息的流向：同步 vs 异步

**核心事实**：无论同步还是异步，`runAgent()` 本身总是 AsyncGenerator，因为 Claude API 是流式的——它一块一块地返回数据，不是等全部完成才返回。

区别在于：**谁来消费这个 Generator，消息就流向哪里**。

#### 同步模式：直接消费

```typescript
// AgentTool.tsx 中的同步调用
for await (const message of runAgent({ isAsync: false, ... })) {
  // message 直接返回给父 Agent
  // 每条消息实时可见
}
```

同步模式下，Generator 被直接遍历，**每 yield 一条消息，父 Agent 立即收到**。父 Agent 的执行循环暂停在这里，等这条消息处理完，才继续下一次遍历。

| 步骤 | 说明 |
|------|------|
| 1 | `runAgent()` yield message |
| 2 | 父 Agent 立即处理 |
| 3 | 继续下一次遍历 |

#### 异步模式：后台消费

```typescript
// AgentTool.tsx 中的异步调用
runAsyncAgentLifecycle({ agentId, runAgentPromise })
```

异步模式下，`runAgent()` 被包装成 Promise 传给 `runAsyncAgentLifecycle()`，它在后台启动一个独立的任务来消费 Generator。

| 步骤 | 说明 |
|------|------|
| 1 | `runAgent()` yield message |
| 2 | 后台任务消费 |
| 3 | 存入队列 |
| 4 | 不阻塞父 Agent |

**注意**：这些消息**不会**直接返回给父 Agent。父 Agent 不会看到中间的 progress message、assistant message 等。父 Agent 只会收到最终的 `task-notification`。

#### 两种模式接收的消息对比

| 消息类型 | 同步 | 异步 |
|---------|------|------|
| `stream_event` (TTFT/OTP) | 实时转发给父 Agent | 丢弃（后台任务自己记录） |
| `assistant` (含 tool_use) | 实时 yield | 存入后台队列，不返回 |
| `user` (子 Agent 的追问) | 实时 yield | 存入后台队列，不返回 |
| `attachment` (max_turns_reached) | yield 给父 Agent | **可以**通过 task-notification 通知 |
| 最终结果 | 直接返回 | 通过 task-notification 返回 |

#### 实际场景举例

**同步 Agent**（用户直接调用 `/agent --sync "修 bug"`）：

| 阶段 | 消息类型 | 说明 |
|------|----------|------|
| 1 | `stream_event` (TTFT) | 实时 yield |
| 2 | `assistant` (thinking) | 实时 yield |
| 3 | `assistant` (tool_use: Read) | 实时 yield |
| 4 | `user` (继续追问) | 实时 yield |
| 5 | 最终结果 | 直接返回 |

**异步 Agent**（用户直接调用 `/agent --background "修 bug"`）：

| 阶段 | 操作 | 说明 |
|------|------|------|
| 1 | `runAgent()` 后台运行 | 所有消息后台消费 |
| 2 | `stream_event` / `assistant` | 丢弃，不返回 |
| 3 | 完成后发送 | `task-notification` |
| 4 | 用户收到通知 | "Agent 已完成：修复了 3 个文件" |

## 3. 函数签名与核心机制

### 3.1 函数签名

```typescript
// src/tools/AgentTool/runAgent.ts:248
export async function* runAgent({
  agentDefinition,       // Agent 定义
  promptMessages,        // 初始消息（用户 prompt）
  toolUseContext,        // 父级的工具上下文
  canUseTool,           // 工具权限检查函数
  isAsync,              // 是否异步执行（关键控制参数）
  forkContextMessages,   // Fork 时的父级消息历史
  querySource,          // 查询来源标识
  override,             // 覆盖参数（systemPrompt、abortController 等）
  model,                // 模型选择
  maxTurns,            // 最大执行轮次
  availableTools,       // 可用工具池
  allowedTools,         // 允许的工具列表
  worktreePath,         // Worktree 隔离路径
  description,          // Agent 描述
  transcriptSubdir,      // 消息记录子目录
  onQueryProgress,      // 进度回调
  // ...
}): AsyncGenerator<Message, void>
```

### 3.2 消息类型

Agent 执行过程中产生的消息类型：

```typescript
type Message =
  | AssistantMessage    // AI 的回复（包含 tool_use 块）
  | UserMessage         // 用户消息
  | ProgressMessage     // 进度消息
  | SystemMessage       // 系统消息
  | StreamEvent        // 流事件（TTFT、OTP 等）
  | TombstoneMessage    // 墓碑消息（删除标记）
  | ToolUseSummaryMessage  // 工具使用摘要
  | AttachmentMessage   // 附件消息（如 max_turns_reached）
```

### 3.3 工具解析机制

根据 Agent 类型和权限模式，过滤出该 Agent 可用的工具：

```typescript
// src/tools/AgentTool/agentToolUtils.ts
function resolveAgentTools(
  agentDefinition: AgentDefinition,
  availableTools: Tools,
  isAsync: boolean,
): { resolvedTools: Tools; deniedTools: Tool[] } {
  // 1. 内置 Agent 直接可用
  // 2. 自定义 Agent 需要权限检查
  // 3. 异步 Agent 有额外限制（某些工具不可用）
}
```

**工具过滤规则**：

| 规则 | 说明 |
|------|------|
| `ALL_AGENT_DISALLOWED_TOOLS` | 所有 Agent 都不可用 |
| `CUSTOM_AGENT_DISALLOWED_TOOLS` | 自定义 Agent 不可用 |
| `ASYNC_AGENT_ALLOWED_TOOLS` | 异步 Agent 限制工具池 |

### 3.4 权限处理

每个 Agent 都有独立的权限上下文：

```typescript
const agentGetAppState = () => {
  const state = toolUseContext.getAppState()
  let toolPermissionContext = state.toolPermissionContext

  // 1. 覆盖 permission mode
  if (agentPermissionMode && parentMode !== 'bypassPermissions') {
    toolPermissionContext = { ...toolPermissionContext, mode: agentPermissionMode }
  }

  // 2. 异步 Agent 避免权限弹窗
  if (shouldAvoidPrompts) {
    toolPermissionContext = { ...toolPermissionContext, shouldAvoidPermissionPrompts: true }
  }

  return { ...state, toolPermissionContext }
}
```

**权限模式**：

| 模式 | 说明 |
|------|------|
| `auto` | 自动决定是否提示 |
| `manual` | 总是提示 |
| `bypassPermissions` | 跳过所有提示 |
| `acceptEdits` | 自动接受编辑 |
| `plan` | 需要计划审批 |

### 3.5 Sidechain Transcript

**Sidechain Transcript 是子 Agent 独立的消息记录**：

| 文件 | 说明 |
|------|------|
| `session-<sessionId>.jsonl` | 主会话文件 |
| `session-<sessionId>/subagents/agent-<agentId>.jsonl` | 子 Agent 独立记录 |

**作用**：

- 子 Agent 的消息独立记录，不混入主会话
- 支持 `--resume` 恢复子 Agent 的执行
- 每个 Agent 可独立查看自己的消息历史

```typescript
// 记录逻辑
if (isRecordableMessage(message)) {
  await recordSidechainTranscript([message], agentId, lastRecordedUuid)
  yield message  // 同时 yield 给父级
}
```

### 3.6 上下文构建

**User Context vs System Context**：

| 概念 | 来源 | 内容 |
|------|------|------|
| userContext | `getUserContext()` | gitStatus、项目信息、IDE 状态等 |
| systemContext | `getSystemContext()` | 环境变量详情等 |

**优化策略**：

```typescript
// Explore/Plan Agent 排除 gitStatus（节省 5-15 Gtok/week）
const shouldOmitClaudeMd = agentDefinition.omitClaudeMd && !override?.userContext

// Explore/Plan Agent 排除 gitStatus（节省 1-3 Gtok/week）
const resolvedSystemContext =
  agentDefinition.agentType === 'Explore' || agentDefinition.agentType === 'Plan'
    ? systemContextNoGit
    : baseSystemContext
```

### 3.7 Sync vs Async 的状态共享

```typescript
// 创建 subagentContext
const agentToolUseContext = createSubagentContext(toolUseContext, {
  abortController: agentAbortController,
  getAppState: agentGetAppState,
  shareSetAppState: !isAsync,  // ← 关键区别
})
```

| 维度 | 同步 (isAsync=false) | 异步 (isAsync=true) |
|------|---------------------|---------------------|
| setAppState | 共享父级 | 独立副本 |
| abortController | 共享父级 | 新建独立 |
| 状态同步 | 实时 | 最终一致 |

## 4. 执行流程详解

### 4.1 完整流程

| 步骤 | 阶段 | 主要操作 |
|------|------|----------|
| 1 | 初始化 | 创建 agentId、解析模型和工具、初始化 MCP 服务器、注册 frontmatter hooks、预加载技能 |
| 2 | 构建上下文 | `getUserContext()` / `getSystemContext()`、解析工具权限、构建 system prompt、决定 permission mode |
| 3 | 创建 Agent Context | `createSubagentContext()`、克隆文件状态缓存、设置 abort controller |
| 4 | 执行循环 (query) | `for await` 遍历 query、处理 stream_event/assistant/user 等消息、记录到 sidechain transcript |
| 5 | 清理 (finally) | MCP cleanup()、清理 session hooks、清理 prompt cache tracking、释放文件状态缓存、清理 Perfetto 注册、清理 todos entry、killShellTasksForAgent() |

### 4.2 Step 1: 初始化

```typescript
// 创建唯一 agentId
const agentId = override?.agentId ?? createAgentId()

// 解析模型
const resolvedAgentModel = getAgentModel(
  agentDefinition.model,
  toolUseContext.options.mainLoopModel,
  model,
  permissionMode,
)

// 注册 Perfetto 追踪
if (isPerfettoTracingEnabled()) {
  registerPerfettoAgent(agentId, agentDefinition.agentType, parentId)
}

// 初始化 MCP 服务器（Agent 可定义自己的 MCP）
const { clients: mergedMcpClients, tools: agentMcpTools, cleanup: mcpCleanup } =
  await initializeAgentMcpServers(agentDefinition, toolUseContext.options.mcpClients)

// 注册 frontmatter hooks
if (agentDefinition.hooks) {
  registerFrontmatterHooks(rootSetAppState, agentId, agentDefinition.hooks, ...)
}

// 预加载技能
if (agentDefinition.skills?.length) {
  const skills = await getSkillToolCommands(getProjectRoot())
  // 加载每个 skill 的内容
}
```

### 4.3 Step 2: 构建上下文

```typescript
// 用户上下文（可选择排除 CLAUDE.md）
const resolvedUserContext = shouldOmitClaudeMd
  ? userContextNoClaudeMd
  : baseUserContext

// 系统上下文（Explore/Plan 排除 gitStatus）
const resolvedSystemContext =
  agentDefinition.agentType === 'Explore' || agentDefinition.agentType === 'Plan'
    ? systemContextNoGit
    : baseSystemContext

// 构建 system prompt
const agentSystemPrompt = override?.systemPrompt
  ?? asSystemPrompt(await getAgentSystemPrompt(...))

// 解析工具
const resolvedTools = useExactTools
  ? availableTools
  : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools
```

### 4.4 Step 3: 创建 Agent Context

```typescript
// 决定 abortController
const agentAbortController = override?.abortController
  ?? (isAsync ? new AbortController() : toolUseContext.abortController)

// 创建 subagentContext
const agentToolUseContext = createSubagentContext(toolUseContext, {
  options: agentOptions,
  agentId,
  agentType: agentDefinition.agentType,
  messages: initialMessages,
  readFileState: agentReadFileState,
  abortController: agentAbortController,
  getAppState: agentGetAppState,
  shareSetAppState: !isAsync,
})
```

### 4.5 Step 4: 执行循环

```typescript
try {
  for await (const message of query({
    messages: initialMessages,
    systemPrompt: agentSystemPrompt,
    userContext: resolvedUserContext,
    systemContext: resolvedSystemContext,
    canUseTool,
    toolUseContext: agentToolUseContext,
    querySource,
    maxTurns: maxTurns ?? agentDefinition.maxTurns,
  })) {
    // 处理不同类型的消息
    if (message.type === 'stream_event') {
      // 转发 TTFT/OTPS 到父级 metrics
      toolUseContext.pushApiMetricsEntry?.(message.ttftMs)
    }

    if (message.type === 'attachment') {
      yield message  // 如 max_turns_reached
    }

    if (isRecordableMessage(message)) {
      // 记录到 sidechain transcript
      await recordSidechainTranscript([message], agentId, lastRecordedUuid)
      yield message  // 同时 yield 给父级
    }
  }
} finally {
  // 清理资源
}
```

### 4.6 Step 5: 清理

```typescript
finally {
  // 1. MCP 清理
  await mcpCleanup()

  // 2. 清理 session hooks
  clearSessionHooks(agentId)

  // 3. 清理 prompt cache tracking
  cleanupAgentTracking(agentId)

  // 4. 释放文件状态缓存
  // ...

  // 5. 清理 Perfetto 注册
  if (isPerfettoTracingEnabled()) {
    unregisterPerfettoAgent(agentId)
  }

  // 6. 清理 todos entry
  // ...

  // 7. 清理 shell tasks
  killShellTasksForAgent(agentId)
}
```

## 5. 关键源文件

| 功能 | 文件 |
|------|------|
| runAgent 主实现 | `src/tools/AgentTool/runAgent.ts` |
| AgentTool 入口 | `src/tools/AgentTool/AgentTool.tsx` |
| 工具解析 | `src/tools/AgentTool/agentToolUtils.ts` |
| Subagent Context | `src/utils/forkedAgent.ts` |
| 查询执行 | `src/query.ts` |
| Sidechain Transcript | `src/utils/sessionStorage.ts` |
| MCP 客户端 | `src/services/mcp/client.ts` |

## 6. 本章小结

- `runAgent()` 是一个 **AsyncGenerator**，因为 Claude API 是流式的——一块一块返回数据
- 同步模式：父 Agent 直接 `for await` 遍历，消息实时 yield 回来
- 异步模式：后台任务消费 Generator，消息不返回父 Agent，通过 task-notification 通知结果
- 完整执行流程分 5 步：初始化 → 构建上下文 → 创建 Agent Context → 执行循环 → finally 清理
- 工具解析根据 Agent 类型和权限模式过滤可用工具
- Sidechain Transcript 独立记录子 Agent 的消息历史

---

## 下一步

← [01_CORE_CONCEPTS.md](01_CORE_CONCEPTS.md) 学习 Agent 核心概念
→ [03_AGENT_ORIGINS.md](03_AGENT_ORIGINS.md) 学习 Agent 来源与构造
