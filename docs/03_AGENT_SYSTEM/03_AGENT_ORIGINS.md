# 03 - Agent 的来源与构造

> 学习 Claude Code 中 Agent 的来源分类、定义结构与特殊类型

## 1. 学习目标

学完本章后，你将理解：

- AgentDefinition 的核心字段与构造方式
- Agent 的三种来源：内置、自定义、队友
- 各内置 Agent（Explore、Plan、Verification）的实现与特点
- 内置 Agent 与自定义 Agent 的区别
- Team Agent 的创建与通信机制
- One-Shot Agent 和 Fork Agent 的特点

## 2. AgentDefinition — 统一的数据结构

所有 Agent（无论来源）都遵循统一的 `AgentDefinition` 接口：

```typescript
// src/tools/AgentTool/loadAgentsDir.ts

interface BaseAgentDefinition {
  agentType: string           // 唯一标识，如 "general-purpose"、"Explore"
  whenToUse: string            // 使用场景描述（给用户看）
  tools: string[]            // 可用工具列表，['*'] 表示全部
  disallowedTools?: string[]   // 明确禁止的工具
  model: string              // 'sonnet'、'opus'、'haiku' 或 'inherit'
  maxTurns: number          // 最大执行轮次
  permissionMode: PermissionMode  // 权限模式
  getSystemPrompt: () => string | Promise<string>  // system prompt 生成器
  hooks: FrontmatterHook[]   // 生命周期钩子
  skills: string[]            // 预加载的技能
  mcpServers: McpServerSpec[] // Agent 专用的 MCP 服务器
  omitClaudeMd: boolean       // 是否排除 CLAUDE.md
  background: boolean         // 是否强制后台运行
  isolation: 'worktree'       // 是否使用 git worktree 隔离
}
```

根据来源不同，AgentDefinition 分为三种类型：

| 类型 | source 字段 | prompt 来源 |
|------|-------------|-------------|
| BuiltInAgentDefinition | `'built-in'` | 代码中定义，通过 `getSystemPrompt()` 获取 |
| CustomAgentDefinition | `'userSettings'`、`'projectSettings'` 等 | 配置文件定义，通过 `getSystemPrompt()` 获取 |
| PluginAgentDefinition | `'plugin'` | 插件提供 |

## 3. Agent 的三种来源

### 3.1 内置 Agent (built-in)

代码中预定义的 Agent，加载自 `src/tools/AgentTool/built-in/`：

```typescript
// src/tools/AgentTool/builtInAgents.ts
export function getBuiltInAgents(): AgentDefinition[] {
  const agents = [
    GENERAL_PURPOSE_AGENT,
    STATUSLINE_SETUP_AGENT,
  ]

  if (areExplorePlanAgentsEnabled()) {
    agents.push(EXPLORE_AGENT, PLAN_AGENT)
  }

  if (feature('VERIFICATION_AGENT') && ...) {
    agents.push(VERIFICATION_AGENT)
  }

  return agents
}
```

| Agent | 用途 | 工具池 | 特点 |
|-------|------|--------|------|
| `general-purpose` | 通用任务执行 | 全部 | 默认的 subagent 类型 |
| `Explore` | 只读代码搜索 | Bash、Glob、Grep、Read | 排除 CLAUDE.md 节省 token |
| `Plan` | 架构规划 | Bash、Glob、Grep、Read | 排除 CLAUDE.md |
| `verification` | 代码验证 | 依赖配置 | 运行测试验证实现正确性 |
| `statusline-setup` | 状态栏设置 | 全部 | 仅内部使用 |

**内置 Agent 的特点**：
- prompt 通过 `getSystemPrompt()` 获取（即固定字符串）
- 通过 Feature Flag 控制是否启用（如 `BUILTIN_EXPLORE_PLAN_AGENTS`）
- 代码合并后，通过配置开启，不需重新部署

#### 3.1.1 Explore Agent

```typescript
// src/tools/AgentTool/built-in/exploreAgent.ts
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  disallowedTools: [AGENT_TOOL_NAME, FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, ...],
  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
  omitClaudeMd: true,
  getSystemPrompt: () => getExploreSystemPrompt(),
}
```

**特点**：
- 只读搜索 Agent，禁止任何文件修改
- 可用工具：Glob、Grep、Read、Bash（仅只读命令）
- 使用 Haiku 模型以提高速度（ant 用户继承父模型）

#### 3.1.2 Plan Agent

```typescript
// src/tools/AgentTool/built-in/planAgent.ts
export const PLAN_AGENT: BuiltInAgentDefinition = {
  agentType: 'Plan',
  disallowedTools: [AGENT_TOOL_NAME, FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, ...],
  model: 'inherit',
  omitClaudeMd: true,
  getSystemPrompt: () => getPlanV2SystemPrompt(),
}
```

**特点**：
- 软件架构师角色，设计实现计划
- 只读探索 + 输出实施策略
- 输出关键文件列表

#### 3.1.3 Verification Agent

```typescript
// src/tools/AgentTool/built-in/verificationAgent.ts
export const VERIFICATION_AGENT: BuiltInAgentDefinition = {
  agentType: 'verification',
  disallowedTools: [AGENT_TOOL_NAME, FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, ...],
  model: 'inherit',
  background: true,
  getSystemPrompt: () => VERIFICATION_SYSTEM_PROMPT,
}
```

**特点**：
- 验证实现是否正确，不是确认代码存在
- 必须运行命令验证，不能只读代码
- 输出格式：每个检查必须有命令输出，最后 `VERDICT: PASS/FAIL/PARTIAL`
- 对抗性探测：并发、边界值、幂等性等

### 3.2 自定义 Agent (custom)

用户或项目定义的 Agent，通过两种方式创建：

#### 方式一：markdown 文件

```
agents/my-agent.agent.md
---
name: my-agent
description: 用于审查代码变更
tools: [Bash, Glob, Grep, FileRead, FileEdit]
maxTurns: 100
permissionMode: auto
---

You are a code reviewer agent. Your job is to...
```

#### 方式二：JSON 配置

```json
// settings.json 或项目配置
{
  "agents": {
    "my-agent": {
      "description": "用于审查代码变更",
      "tools": ["Bash", "Glob", "Grep", "FileRead", "FileEdit"],
      "prompt": "You are a code reviewer agent..."
    }
  }
}
```

**自定义 Agent 的特点**：
- `source` 字段标识来源：`'userSettings'`、`'projectSettings'`、`'flagSettings'`、`'policySettings'`
- prompt 通过 `getSystemPrompt()` 获取（即固定字符串）
- 优先级：built-in < plugin < user < project < flag < policy（后者覆盖前者）

### 3.3 队友 Agent (teammate)

通过 `spawnTeammate()` 创建的持久化 Agent，有独立的进程或 tmux 窗口：

```typescript
// src/tools/shared/spawnMultiAgent.ts

spawnTeammate({
  name: "reviewer",           // 可寻址名称
  prompt: "审查代码变更...",
  team_name: "pr-review",
  model: "sonnet",
})
```

**队友 Agent vs 普通 Subagent**：

| 维度 | Subagent | Teammate |
|------|----------|----------|
| 生命周期 | 任务级 | 团队级 |
| 隔离方式 | 共享进程 | tmux 窗口或独立进程 |
| 通信方式 | task-notification | 邮箱 (mailbox) |
| 状态持久 | 无 | 有 |
| 寻址 | agentId | name@teamName |

**Mailbox 通信机制**：

Mailbox 不是真实邮箱，是**进程间通信机制**。队友运行在独立的 tmux 窗口或进程中，通过读写共享文件（team file）来传递消息：

```typescript
// 发送消息给队友（写入共享文件）
await writeToMailbox(recipientName, {
  from: senderName,
  text: "请审查 PR #123",
  timestamp: new Date().toISOString(),
}, teamName)

// 队友通过轮询 mailbox 接收消息
```

这是**隐藏功能**，需要启用：
- 环境变量 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 或
- CLI 参数 `--agent-teams`

```bash
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 bun run dev
```

**架构**：主从模式，Leader 通过 mailbox 向队友发消息，队友轮询接收指令。

## 4. 特殊 Agent 类型

### 4.1 One-Shot Agent

Explore 和 Plan Agent 属于 One-Shot 类型，执行一次返回报告，不返回继续交互：

```typescript
// src/tools/AgentTool/constants.ts

export const ONE_SHOT_BUILTIN_AGENT_TYPES: ReadonlySet<string> = new Set([
  'Explore',
  'Plan',
])
```

**优化点**：跳过 `agentId`/`SendMessage`/`usage trailer` 等字段，节省约 135 chars × 34M+ runs/week。

### 4.2 Fork Agent

Fork 是一种特殊的执行模式，不是独立的 Agent 类型：

```typescript
// src/tools/AgentTool/forkSubagent.ts

export const FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],
  maxTurns: 200,
  model: 'inherit',           // 继承父模型
  permissionMode: 'bubble',  // 权限冒泡到父
  getSystemPrompt: () => '',
}
```

**Fork 的用途**：并行执行同一任务的不同变体，继承父 Agent 的完整上下文，最大化 prompt cache 命中率。

**与普通 Subagent 的区别**：

| 特性 | Fork | 普通 Subagent |
|------|------|---------------|
| 上下文 | 继承父完整历史 | 独立消息 |
| Prompt Cache | 高（前缀相同） | 低 |
| 权限 | bubble 冒泡到父 | 独立 |
| 适用场景 | 并行变体 | 独立任务 |

## 5. 工具过滤

Agent 可用的工具不是简单地从父级继承，而是根据 Agent 类型重新计算：

```typescript
// src/tools/AgentTool/agentToolUtils.ts

function resolveAgentTools(
  agentDefinition: AgentDefinition,
  availableTools: Tools,
  isAsync: boolean,
): { resolvedTools: Tools; deniedTools: Tool[] } {
  // 1. 基础工具过滤
  // 2. 内置 Agent 直接可用
  // 3. 自定义 Agent 需要权限检查
  // 4. 异步 Agent 有额外限制
}
```

**过滤规则**：

| 规则 | 说明 |
|------|------|
| `ALL_AGENT_DISALLOWED_TOOLS` | 所有 Agent 都不可用（如 AgentTool、TaskStopTool，防止嵌套创建 Agent） |
| `CUSTOM_AGENT_DISALLOWED_TOOLS` | 自定义 Agent 不可用 |
| `ASYNC_AGENT_ALLOWED_TOOLS` | 异步 Agent 限制工具池 |

**MCP 过滤**：MCP (Model Context Protocol) 是一种扩展协议，允许 Agent 连接外部工具（如 Slack、GitHub）。如果 Agent 定义了 `requiredMcpServers`，但环境中没有对应的 MCP 服务器，该 Agent 会被过滤掉，不会出现在可用列表中：

```typescript
// 检查 MCP 要求
export function hasRequiredMcpServers(
  agent: AgentDefinition,
  availableServers: string[],
): boolean {
  if (!agent.requiredMcpServers) return true
  return agent.requiredMcpServers.every(pattern =>
    availableServers.some(server =>
      server.toLowerCase().includes(pattern.toLowerCase())
    )
  )
}
```

## 6. 关键源文件

| 功能 | 文件 |
|------|------|
| Agent 定义类型 | `src/tools/AgentTool/loadAgentsDir.ts` |
| 内置 Agent 加载 | `src/tools/AgentTool/builtInAgents.ts` |
| 内置 Agent 实现 | `src/tools/AgentTool/built-in/*.ts` |
| Team Agent 机制 | `src/tools/shared/spawnMultiAgent.ts` |
| Fork Agent | `src/tools/AgentTool/forkSubagent.ts` |
| 工具过滤 | `src/tools/AgentTool/agentToolUtils.ts` |
| One-Shot 常量 | `src/tools/AgentTool/constants.ts` |

## 7. 本章小结

- 所有 Agent 都遵循统一的 `AgentDefinition` 接口，区别在于 `source` 字段
- Agent 有三种来源：内置（代码预定义）、自定义（用户/项目配置）、队友（独立进程）
- 内置 Agent 通过 Feature Flag 控制加载，prompt 通过 `getSystemPrompt()` 获取
- 自定义 Agent 通过 markdown 文件或 JSON 配置创建，prompt 通过 `getSystemPrompt()` 获取
- Team Agent 有持久化身份，通过 mailbox（进程间通信）通信，不走 task-notification
- One-Shot Agent（Explore、Plan）跳过 agentId/SendMessage 等交互字段，节省 token
- Fork 是执行模式而非 Agent 类型，用于最大化 prompt cache
- 工具根据 Agent 类型和权限模式重新计算，MCP 缺失会导致 Agent 被过滤

---

## 下一步

← [02_RUN_AGENT.md](02_RUN_AGENT.md) 学习 Agent 执行流程
→ [04_TASK_SYSTEM.md](04_TASK_SYSTEM.md) 学习任务系统
