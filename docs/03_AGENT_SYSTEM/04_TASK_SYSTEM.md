# 04 - 任务系统

> 学习 Claude Code 如何通过任务状态机管理七种任务的完整生命周期

## 1. 学习目标

学完本章后，你将理解：

- 七种任务类型的分类与各自职责
- 任务状态机的设计：状态与转换
- 状态转换的触发条件：start、stop、abort、complete
- task-notification 的触发时机、数据结构与消费者处理

## 2. 七种任务类型

所有任务类型通过 `TaskState` 联合类型统一管理：

```typescript
// src/tasks/types.ts

export type TaskState =
  | LocalShellTaskState      // 本地 shell 命令
  | LocalAgentTaskState      // 本地 agent 任务
  | RemoteAgentTaskState     // 远程 agent 任务
  | InProcessTeammateTaskState  // 进程内队友
  | LocalWorkflowTaskState   // 本地工作流
  | MonitorMcpTaskState      // MCP 监控
  | DreamTaskState          // 异步梦境任务
```

| 任务类型 | 说明 | 触发方式 |
|----------|------|----------|
| **LocalShellTask** | 执行本地 shell 命令（`npm install`、`git status` 等） | BashTool |
| **LocalAgentTask** | 本地子 Agent 任务 | AgentTool (isAsync=true) |
| **RemoteAgentTask** | 连接远程 Agent 实例进行协作 | AgentTool + 远程配置 |
| **InProcessTeammateTask** | 进程内队友，共享进程状态 | spawnTeammate (in-process 模式) |
| **LocalWorkflowTask** | 执行预定义的多步骤本地工作流 | 内部工作流引擎 |
| **MonitorMcpTask** | 监控 MCP 服务器的连接状态和健康检查 | MCP 连接管理 |
| **DreamTask** | AI 自动分析任务，反思和规划 | 空闲时触发 |

## 3. 任务状态机

### 3.1 状态定义

每个任务都有以下核心状态字段：

```typescript
interface BaseTaskState {
  taskId: string
  status: 'pending' | 'running' | 'completed' | 'failed' | 'killed'
  createdAt: number
  startedAt?: number
  completedAt?: number
}
```

| 状态 | 说明 |
|------|------|
| `pending` | 任务已创建，等待启动 |
| `running` | 任务正在执行 |
| `completed` | 任务成功完成 |
| `failed` | 任务执行失败（出错） |
| `killed` | 任务被手动终止 |

### 3.2 状态转换

**转换路径**：

- 创建任务 → pending → start() → running
- running → complete() → completed（正常完成）
- running → abort() 或 stop() → killed（被终止）
- running → error() → failed（执行出错）

**触发来源**：

| 来源 | 触发条件 | 状态变化 |
|------|----------|---------|
| 用户操作（点击停止、Ctrl+C） | `stop()` | running → killed |
| Agent 操作（TaskStopTool） | `abort()` | running → killed |
| 系统自动（maxTurns 达到、abort signal） | `abort()` | running → killed |
| 任务正常结束 | `complete()` | running → completed |
| 任务执行出错 | `error()` | running → failed |

## 4. task-notification 机制

task-notification 是异步 Agent 向父级报告完成的主要机制。

### 4.1 触发时机

当异步 Agent 执行完成时（无论成功、失败还是被 kill），都会发送 task-notification：

```typescript
// runAgent.ts 执行循环结束后
finally {
  // 发送 task-notification
  await completeAsyncAgent(taskId, {
    status: isAborted ? 'killed' : hasError ? 'failed' : 'completed',
    result: finalTextResponse,
  })
}
```

### 4.2 数据结构

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

**各字段含义**：

| 字段 | 说明 |
|------|------|
| `task-id` | Agent 的唯一 ID |
| `status` | 执行结果：completed（成功）、failed（失败）、killed（被终止） |
| `summary` | 人类可读的状态摘要 |
| `result` | Agent 的最终文本响应 |
| `usage.total_tokens` | 消耗的总 token 数 |
| `usage.tool_uses` | 工具调用次数 |
| `usage.duration_ms` | 执行耗时（毫秒） |

### 4.3 消费者处理

task-notification 被发送到父 Agent 后，父 Agent 的执行循环会处理它：

```typescript
// AgentTool.tsx 中的处理逻辑
for await (const message of runAgent({ ... })) {
  if (message.type === 'task-notification') {
    // 解析 notification
    const { status, result, usage } = parseTaskNotification(message)

    // 更新 UI 显示
    // 决定是否继续（SendMessage）还是结束
  }
}
```

**处理流程**：

```
Agent 执行完成
    │
    ▼
completeAsyncAgent() 写入 notification
    │
    ▼
父 Agent 的执行循环收到 notification
    │
    ▼
解析 status：
  ├── completed：打印结果给用户
  ├── failed：打印错误信息
  └── killed：打印被终止信息
    │
    ▼
决定后续操作：
  ├── 继续对话（用户输入新指令）
  └── 结束（无更多指令）
```

## 5. 各任务类型详解

### 5.1 LocalShellTask

执行本地 shell 命令：

```typescript
// src/tasks/LocalShellTask/LocalShellTask.tsx

export interface LocalShellTaskState extends BaseTaskState {
  type: 'local_shell'
  command: string      //执行的命令
  cwd: string         //工作目录
  exitCode?: number   //退出码
  agentId?: string    //如果是 agent 启动的
}
```

**典型场景**：`npm install`、`git status`、`cargo build` 等。

### 5.2 LocalAgentTask

管理异步 Agent 任务的生命周期：

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.ts

export interface LocalAgentTaskState extends BaseTaskState {
  type: 'local_agent'
  agentId: string
  agentType: string
  description: string
  prompt: string
  outputFile?: string
  isBackgrounded: boolean
}
```

**核心操作**：

```typescript
// 注册异步任务
registerAsyncAgent({ taskId, agentId, description, prompt, ... })

// 完成异步任务
completeAsyncAgent(taskId, { status: 'completed', result: '...' })

// 终止异步任务
killAsyncAgent(taskId, toolUseContext, rootSetAppState)
```

### 5.3 RemoteAgentTask

连接远程 Agent 实例（CCR）进行协作：

```typescript
// src/tasks/RemoteAgentTask/RemoteAgentTask.ts

export interface RemoteAgentTaskState extends BaseTaskState {
  type: 'remote_agent'
  remoteId: string
  connectionUrl: string
}
```

### 5.4 InProcessTeammateTask

进程内队友，与主 Agent 共享进程状态：

```typescript
// src/tasks/InProcessTeammateTask/types.ts

export interface InProcessTeammateTaskState extends BaseTaskState {
  type: 'in_process_teammate'
  identity: {
    agentId: string
    agentName: string
    teamName: string
    color: string
    parentSessionId: string
  }
  prompt: string
  permissionMode: PermissionMode
  shutdownRequested: boolean
}
```

### 5.5 LocalWorkflowTask

执行预定义的多步骤工作流：

```typescript
// src/tasks/LocalWorkflowTask/LocalWorkflowTask.ts

export interface LocalWorkflowTaskState extends BaseTaskState {
  type: 'local_workflow'
  workflowDefinition: string
  currentStep: number
  steps: WorkflowStep[]
}
```

### 5.6 MonitorMcpTask

监控 MCP 服务器的健康状态：

```typescript
// src/tasks/MonitorMcpTask/MonitorMcpTask.ts

export interface MonitorMcpTaskState extends BaseTaskState {
  type: 'monitor_mcp'
  mcpServerName: string
  checkIntervalMs: number
  lastHealthCheck?: number
}
```

### 5.7 DreamTask

空闲时自动运行的反思和分析任务：

```typescript
// src/tasks/DreamTask/DreamTask.ts

export interface DreamTaskState extends BaseTaskState {
  type: 'dream'
  trigger: 'idle' | 'manual'
  analysis: string
}
```

## 6. 关键源文件

| 功能 | 文件 |
|------|------|
| Task 类型定义 | `src/tasks/types.ts` |
| LocalAgentTask | `src/tasks/LocalAgentTask/LocalAgentTask.ts` |
| LocalShellTask | `src/tasks/LocalShellTask/LocalShellTask.tsx` |
| RemoteAgentTask | `src/tasks/RemoteAgentTask/RemoteAgentTask.ts` |
| InProcessTeammateTask | `src/tasks/InProcessTeammateTask/types.ts` |
| LocalWorkflowTask | `src/tasks/LocalWorkflowTask/LocalWorkflowTask.ts` |
| MonitorMcpTask | `src/tasks/MonitorMcpTask/MonitorMcpTask.ts` |
| DreamTask | `src/tasks/DreamTask/DreamTask.ts` |

## 7. 本章小结

- 七种任务类型通过 `TaskState` 联合类型统一管理
- 任务状态机：pending → running → completed/failed/killed
- 状态转换通过 `start()`、`complete()`、`abort()`、`stop()` 触发
- task-notification 在 Agent 执行完成后发送，包含状态、结果、token 统计
- 父 Agent 收到 notification 后解析 status，决定后续操作

---

## 下一步

← [03_AGENT_ORIGINS.md](03_AGENT_ORIGINS.md) 学习 Agent 来源与构造
→ [05_MULTI_AGENT.md](05_MULTI_AGENT.md) 学习多 Agent 协作
