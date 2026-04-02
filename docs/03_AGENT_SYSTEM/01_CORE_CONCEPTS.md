# 01 - Agent 核心概念

> 学习 Claude Code 中 Agent 的定义、类型和核心工具三角

## 1. 学习目标

学完本章后，你将理解：

- Agent 的定义和与普通函数的本质区别
- 核心工具三角：AgentTool、SendMessage、TaskStop
- Agent 的分类方式（按来源、按用途）
- 同步 vs 异步 Agent 的状态共享差异
- toolUseContext 的作用
- Agent 的生命周期

## 2. Agent 是什么

在 Claude Code 中，**Agent** 是一个独立运行的 AI 实例，有自己的：

| 特性 | 说明 |
|------|------|
| 对话上下文 | 独立的 messages 历史 |
| 工具池 | 可用的工具集合 |
| 执行循环 | 自己的 query → response 循环 |
| 结果输出 | 通过 task-notification 返回结果 |

**本质区别**：Agent 是 AI 驱动执行的，而不是程序员在代码里写死调用顺序。

## 3. 核心工具三角

Agent 的生命周期由三个核心工具控制：

| 工具 | 作用 | 使用场景 |
|------|------|----------|
| Agent Tool | 启动新 Agent | 创建子 Agent、并行任务 |
| SendMessage Tool | 继续执行 | 向已启动的 Agent 发送消息 |
| TaskStop Tool | 停止 Agent | 取消任务、结束执行 |

### 3.1 Agent Tool — 启动 Agent

```typescript
// src/tools/AgentTool/AgentTool.tsx
const inputSchema = z.object({
  prompt: z.string(),                    // 任务描述
  subagent_type: z.string(),           // Agent 类型
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),  // 是否后台运行
  // 多 Agent 团队参数
  name: z.string().optional(),         // 队友名称（可寻址）
  team_name: z.string().optional(),    // 团队名称
  mode: permissionModeSchema().optional(),
  isolation: z.enum(['worktree']).optional(),
  cwd: z.string().optional(),
})
```

### 3.2 SendMessage Tool — 继续 Agent

```typescript
// src/tools/SendMessageTool/
const inputSchema = z.object({
  to: z.string(),       // 目标 Agent ID
  message: z.string(),  // 要发送的消息
})
```

### 3.3 TaskStop Tool — 停止 Agent

```typescript
// src/tools/TaskStopTool/
const inputSchema = z.object({
  task_id: z.string(),  // 要停止的 Agent ID
})
```

## 4. Agent 类型

### 4.1 按来源分类

| 类型 | 说明 | 示例 |
|------|------|------|
| 内置 (built-in) | 代码中预定义 | `general-purpose`、`Explore`、`Plan` |
| 自定义 (custom) | 用户/插件提供 | 通过 `agents/` 目录或 frontmatter |
| 队友 (teammate) | 多 Agent 团队成员 | `spawnTeammate()` 创建 |

### 4.2 按用途分类

| 类型 | 说明 | 代码位置 |
|------|------|----------|
| Worker | 执行具体任务 | `src/coordinator/workerAgent.ts` |
| Explorer | 只读代码搜索 | `built-in/exploreAgent.ts` |
| Planner | 架构规划 | `built-in/planAgent.ts` |
| Verifier | 对抗性验证 | `built-in/verificationAgent.ts` |
| Coordinator | 任务编排 | `src/coordinator/coordinatorMode.ts` |

## 5. 权限模式

每个 Agent 运行时都有权限模式，控制工具执行的提示行为：

```typescript
// src/utils/permissions/PermissionMode.ts
type PermissionMode =
  | 'auto'                    // 自动决定是否提示
  | 'manual'                  // 总是提示
  | 'bypassPermissions'       // 跳过所有提示
  | 'acceptEdits'             // 自动接受编辑
  | 'plan'                    // 需要计划审批
```

## 6. 上下文传递

### 6.1 Fork 机制 — 完整上下文继承

```typescript
// src/tools/AgentTool/forkSubagent.ts

// Fork 的特点：
// 1. 子 Agent 继承父 Agent 的完整对话历史
// 2. 用于并行执行同一任务的不同变体
// 3. 最大化 prompt cache 命中率

buildForkedMessages(directive, assistantMessage)
// → [...history, assistant(all_tool_uses), user(placeholder_results..., directive)]
```

### 6.2 普通 Subagent — 独立上下文

```typescript
// src/tools/AgentTool/runAgent.ts

// 普通 subagent：
// 1. 从初始消息开始
// 2. 可以访问工具和 MCP
// 3. 独立执行循环
// 4. 结果通过 task-notification 返回
```

### 6.3 task-notification 格式

Agent 执行完成后，通过 task-notification 返回结果：

```xml
<task-notification>
<task-id>{agentId}</task-id>
<status>completed|failed|killed</status>
<summary>{human-readable status summary}</summary>
<result>{agent's final text response}</result>
<usage>
  <total_tokens>N</total_tokens>
  <tool_uses>N</tool_uses>
  <duration_ms>N</duration_ms>
</usage>
</task-notification>
```

## 7. 内置 Agent 列表

| Agent | 说明 | 启用条件 |
|-------|------|----------|
| `GENERAL_PURPOSE_AGENT` | 通用任务执行 | 始终启用 |
| `STATUSLINE_SETUP_AGENT` | 状态栏设置 | 始终启用 |
| `EXPLORE_AGENT` | 只读代码搜索 | Feature Flag `BUILTIN_EXPLORE_PLAN_AGENTS` |
| `PLAN_AGENT` | 架构规划 | Feature Flag `BUILTIN_EXPLORE_PLAN_AGENTS` |
| `VERIFICATION_AGENT` | 代码验证 | Feature Flag `VERIFICATION_AGENT` |
| `COORDINATOR_AGENT` | 任务编排 | Feature Flag `COORDINATOR_MODE` + 环境变量 |

详细实现见 [03_AGENT_ORIGINS.md](03_AGENT_ORIGINS.md)。

## 8. 关键概念

### 8.1 Async vs Sync

| 维度 | 同步 (isAsync=false) | 异步 (isAsync=true) |
|------|---------------------|---------------------|
| **setAppState** | **共享**父级的 | 独立的副本 |
| **abortController** | 共享父级的 | 新建独立 |
| **执行方式** | 调用时阻塞等待 | 立即返回 agentId |
| **结果返回** | 直接返回 | 通过 task-notification |
| **多个并行** | 不支持（会阻塞） | 支持后台同时运行 |
| **状态同步** | 实时共享 | 最终一致 |

**同步模式下**，subagent 和父 agent 共享 AppState，调用 setAppState 会直接修改父级的状态。

**异步模式下**，subagent 有独立的 AppState 副本，完全隔离，通过 task-notification 异步通知完成。

```typescript
// runAgent.ts 核心逻辑
const agentToolUseContext = createSubagentContext(toolUseContext, {
  shareSetAppState: !isAsync,  // 同步共享，异步隔离
  abortController: isAsync ? new AbortController() : toolUseContext.abortController,
})
```

### 8.2 toolUseContext

每个 Agent 都有独立的 toolUseContext：

```typescript
interface ToolUseContext {
  options: {
    tools: Tools              // 可用工具
    mcpClients: ...           // MCP 客户端
    mainLoopModel: ...        // 使用的模型
  }
  getAppState: () => AppState
  setAppState: (updater) => void
  abortController: AbortController
  // ...
}
```

### 8.3 生命周期

```
启动 → 初始化 → 执行循环 → 完成/失败 → 清理
  │         │           │
  │         │           └─► query() yield 消息
  │         │
  │         └─► MCP 连接、frontmatter hooks、技能预加载
  │
  └─► AgentTool.call() 返回 agentId
```

## 9. 关键源文件

| 功能 | 文件 |
|------|------|
| Agent Tool 主实现 | `src/tools/AgentTool/AgentTool.tsx` |
| 启动 Subagent | `src/tools/AgentTool/runAgent.ts` |
| Fork Subagent | `src/tools/AgentTool/forkSubagent.ts` |
| 内置 Agent 加载 | `src/tools/AgentTool/builtInAgents.ts` |
| SendMessage Tool | `src/tools/SendMessageTool/` |
| TaskStop Tool | `src/tools/TaskStopTool/` |
| Task 类型定义 | `src/tasks/types.ts` |

并行协作机制（run_in_background、worktree、Coordinator、Agent Team）见 [05_MULTI_AGENT.md](05_MULTI_AGENT.md)。

## 10. 本章小结

- Agent 是独立运行的 AI 实例，有自己的上下文、工具池和执行循环
- 核心工具三角控制 Agent 生命周期：启动 (AgentTool)、继续 (SendMessage)、停止 (TaskStop)
- Agent 按来源分为内置、自定义、队友；按用途分为 Worker、Explorer、Planner、Verifier
- Fork 机制继承完整上下文以优化 prompt cache；普通 Subagent 有独立上下文
- 同步 Agent 共享父级 AppState，异步 Agent 有独立副本
- task-notification 是完成通知，包含状态、结果、统计信息

---

## 下一步

← [README.md](README.md) 返回 Agent System 总览
→ [02_RUN_AGENT.md](02_RUN_AGENT.md) 学习 Agent 的详细执行流程
