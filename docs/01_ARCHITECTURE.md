# 01 - 架构设计

> 学习如何组织 2000+ 文件的复杂项目，以及为什么 Claude Code 的架构与众不同

## 1. 学习目标

学完本章后，你将理解：

- Claude Code 的整体架构和核心设计理念
- 项目核心模块的实现

## 2. 什么是 Claude Code

Claude Code 是一个 **AI Agent 平台**，而不是普通的 AI CLI 工具。

| 对比 | 普通 AI CLI | Claude Code |
|------|-------------|-------------|
| 工作方式 | 调用 API，输出文本建议 | AI 驱动工具执行，实际完成任务 |
| 执行模式 | 串行 | AI 决定并行或串行 |
| 上下文 | 简单截断 | 三层压缩体系 |
| 工作流 | 弱 | 多 Agent 协作 |

**核心洞察**：Claude Code 的强大不在于"有没有工具"，而在于**工具系统的精致程度**。

## 3. 架构与目录

Claude Code 采用**能力分组**架构，每个分组代表一个核心能力域：

| 分组 | 职责 | 示例内容 |
|------|------|----------|
| tools | 工具执行 | 50+ 工具的定义与实现（Read/Edit/Bash/Glob等） |
| components | 界面渲染 | React 组件的终端 UI 呈现 |
| services | 后台支撑 | API 调用、MCP 连接、压缩服务 |
| tasks | 生命周期 | 7 种任务类型的状态追踪与完成管理 |
| coordinator | 编排协作 | 多 Agent 任务的协调与路由 |
| state | 状态管理 | 全局状态存储与订阅发布 |
| skills | 技能系统 | 结构化知识、最佳实践、思考框架 |
| commands | 命令系统 | 100+ 斜杠命令的注册与执行 |

### 3.1 详细目录结构

| 路径 | 说明 |
|------|------|
| `src/main.tsx` | 主入口 |
| `src/commands.ts` | 斜杠命令注册 |
| `src/commands/` | 斜杠命令实现 (100+) |
| `src/tools/` | 工具实现 (50+) |
| `src/components/` | React UI 组件 (148 文件) |
| `src/hooks/` | React Hooks (87 文件) |
| `src/services/` | 服务层 |
| `src/coordinator/` | 多 Agent 协调器 |
| `src/tasks/` | 任务系统 |
| `src/state/` | 状态管理 |
| `src/types/` | 类型定义 |
| `src/utils/` | 工具函数 |
| `shims/` | 模块替代 |
| `vendor/` | 原生绑定 |

### 3.2 架构对比

**为什么按能力分组而不是按层分组？**

传统 Web 架构按层横向拆分：Controller → Service → Repository → Database，每层只关心下一层。Claude Code 则按能力垂直拆分，每个能力域独立演进。

选择能力分组的原因：

1. **业务逻辑太分散** — 工具本身既是接口又是逻辑，放在传统的 Service 层会很别扭
2. **数据源太杂** — 文件系统、API、用户输入，数据访问模式完全不同
3. **需要动态组合** — 工具可以嵌套调用（Read 的结果给 Grep，Grep 的结果给 Edit），这种调用链无法用固定的分层来约束

| 维度 | 传统 Web 架构 | 普通 CLI | Claude Code |
|------|---------------|---------|-------------|
| 组织方式 | 水平分层：Controller → Service → Repository | 线性：解析 → 执行 → 输出 | 能力垂直分组：工具/UI/状态/任务 |
| 数据流 | 单向分层，清晰但僵化 | 直线型，无状态 | 网状，工具可嵌套调用 |
| 状态管理 | Session/Database | 无状态 | 复杂多 Agent 共享状态 |
| 扩展方式 | 加层（Service/Repository） | 加函数 | 加工具/加 Agent |
| 工具调用 | 无 | 无 | AI 推理驱动并行/串行 |

**结论**：Claude Code 的架构**更适合 AI Agent 这种需要动态组合工具、多方协作的场景**。

几个关键概念的解释：

- **数据流**：传统 Web 是请求 → 处理 → 响应，然后归于平静；Claude Code 是网状结构，工具可以嵌套调用，形成动态调用链。
- **状态管理**：传统 Web 每个请求独立处理，状态存在 Database；Claude Code 是多 Agent 共享状态，会话期间 tasks、messages、settings 持续累积。
- **工具 vs 函数**：本质都是函数，但关键区别在于谁调用、什么时候调用。函数是程序员在代码里写死调用；工具是 AI 根据上下文推理决定何时调用、如何组合。

## 4. 核心设计模式

核心设计模式描述的是**模块间的联动关系和实现标准**。

### 4.1 Feature Flag — 功能开关标准

通过 `feature()` 函数控制实验性功能。代码提前合并，功能通过配置开启，支持灰度发布。

例如 Coordinator 模式、宠物系统、Voice Mode 等都是通过 `feature('XXX')` 控制的隐藏功能，代码已合并但默认不启用。

### 4.2 统一接口标准

Claude Code 要求所有工具和任务遵循统一接口标准：

**Tool Builder** — 所有工具用 `buildTool()` 创建，自动填充默认值（isEnabled、checkPermissions 等），工具只需定义差异化逻辑。

**TaskState 联合类型** — 7 种任务类型（LocalShell、LocalAgent、RemoteAgent 等）通过联合类型统一管理，共享生命周期逻辑。

### 4.3 状态传递标准

全局 AppStateStore 基于发布-订阅模式：

| 层级 | 内容 |
|------|------|
| AppStateStore | 全局状态根节点 |
| toolPermissionContext | 权限状态 |
| mcpClients | MCP 连接 |
| tasks | 任务状态 |
| todos | 待办事项 |
| ToolUseContext | 各组件订阅 |

状态通过 context 层层传递，组件通过 `useAppState()` 订阅。

### 4.4 资源访问标准

工具不直接 import 依赖，而是通过 `ToolUseContext` 获取所需资源。

**ToolUseContext 包含的内容**：

| 类别 | 内容 |
|------|------|
| 状态访问 | `getAppState()`、`setAppState()` |
| 可用资源 | `options.tools`、`options.commands`、`options.mcpClients` |
| 运行时 | `abortController`、`messages` |
| 功能回调 | `setToolJSX()`、`addNotification()`、`sendOSNotification()` |

**工作流程**：

| 步骤 | 说明 |
|------|------|
| 1 | 会话开始 → 构建 ToolUseContext（注入可用资源） |
| 2 | AI 推理决定调用工具 |
| 3 | 工具通过 context 获取所需资源 |
| 4 | 返回结果给 AI |

**好处**：
- **可测试**：测试时可以注入 mock 的 context
- **可替换**：可以轻松替换实现而不改工具代码
- **避免循环依赖**：工具不直接引用状态模块

## 5. 核心模块实现

Claude Code 的核心能力由以下模块协作实现。

### 5.1 工具系统

**职责**：定义和执行 50+ 工具，供 AI 调用完成任务。

**调用流程**：

| 步骤 | 说明 |
|------|------|
| 1 | 用户输入 |
| 2 | AI 推理 |
| 3 | 返回 tool_use 块 |
| 4 | Claude Code 执行 |

**并行与串行**：由 AI 模型根据系统提示词决定。Claude Code 在提示词中告诉 AI：无关依赖的工具可以并行，有依赖的要串行。

**权限管理**：工具通过 `ToolUseContext` 访问权限状态，权限检查由 `buildTool` 的默认 `checkPermissions` 处理。

### 5.2 上下文管理

**职责**：管理对话上下文的大小，防止超出模型限制。

Claude Code 采用**三层压缩体系**：

| 层级 | 名称 | 触发条件 | 机制 |
|------|------|----------|------|
| 第一层 | 工具结果压缩 | 单个工具结果 > 50KB 或总和 > 200KB | 持久化到磁盘，替换为路径+预览 |
| 第二层 | 会话记忆压缩 | 上下文中积累的结构化事实 | 持续提取到文件，压缩时复用 |
| 第三层 | 上下文折叠 | 上下文接近上限 | AI 总结历史对话 |

### 5.3 智能规划能力

**职责**：帮助 AI 更有效地规划和执行任务。

| 组件 | 作用 | 本质 |
|------|------|------|
| Skills | 结构化知识与最佳实践 | 指导框架（通过 system prompt 提供） |
| TaskTool | 逐个完成任务 | 行为约束 |
| System Prompts | 并行/串行决策 | 推理规则 |

**说明**：Skills 存储在 `skills/` 目录，会话开始时加载并构建成 system prompt 片段，供 AI 推理参考。

### 5.4 任务系统

**职责**：管理 7 种任务类型的生命周期和状态追踪。

| 任务类型 | 说明 |
|----------|------|
| LocalShellTask | 终端命令任务 |
| LocalAgentTask | 本地子 Agent 任务 |
| RemoteAgentTask | 远程 Agent 任务 |
| InProcessTeammateTask | 进程内队友任务 |
| LocalWorkflowTask | 本地工作流任务 |
| MonitorMcpTask | MCP 监控任务 |
| DreamTask | 做梦任务（自动分析） |

所有任务类型通过 `TaskState` 联合类型统一管理：

```typescript
export type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState
```

### 5.5 编排协作

**职责**：协调多个 Agent 之间的任务分配和执行。

通过 `feature('COORDINATOR_MODE')` 控制。Coordinator 模式下：
- 主 Agent 负责任务分解和分配
- 工作 Agent 并行执行子任务
- 主 Agent 汇总结果并推进整体流程

### 5.6 命令系统

**职责**：提供 100+ 斜杠命令供用户调用。

| 分类 | 命令示例 |
|------|----------|
| Git 相关 | commit.ts, branch.ts, diff.ts |
| 配置相关 | config.ts, model.ts |
| 上下文相关 | compact.ts, context.ts |
| 扩展相关 | mcp.ts, skills.ts |
| 其他 | 100+ 个命令 |

所有命令通过统一的 Command 接口注册：

```typescript
export type Command = {
  name: string
  description: string
  execute: (args, context) => Promise<CommandResult>
}
```

部分命令通过 Feature Flag 控制：
- `/voice` — 需要 `feature('VOICE_MODE')`
- `/bridge` — 需要 `feature('BRIDGE_MODE')`
- `/brief` — 需要 `feature('KAIROS')` 或 `feature('KAIROS_BRIEF')`

### 5.7 服务层

**职责**：提供后台支撑服务（API 调用、MCP 连接、压缩等）。

主要服务：

| 服务 | 作用 |
|------|------|
| API 服务 | 与 Claude API 通信 |
| MCP 服务 | 管理 MCP 服务器连接 |
| 压缩服务 | 实现三层上下文压缩 |
| 分析服务 | 事件追踪、性能监控 |

## 6. 本章小结

- Claude Code 是 AI Agent 平台，工具系统按能力分组
- 核心设计模式：Feature Flag 控制功能开关、统一接口标准、状态传递标准、资源访问标准
- 七大核心模块：工具系统、上下文管理、智能规划、任务系统、编排协作、命令系统、服务层

---

## 下一步

← [00_OVERVIEW.md](00_OVERVIEW.md) 回到学习路线图
→ [02_STARTUP.md](02_STARTUP.md) 学习启动流程
