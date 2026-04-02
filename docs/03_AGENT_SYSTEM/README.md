# 03 - Agent 编排系统

> 学习 Claude Code 如何实现多 Agent 协作：启动、控制、状态管理与并行策略

## 1. 模块概览

| 文档 | 内容 |
|------|------|
| `01_CORE_CONCEPTS.md` | Agent 核心概念：定义、类型、生命周期 |
| `02_RUN_AGENT.md` | runAgent 执行流程：AsyncGenerator、消息流向 |
| `03_AGENT_ORIGINS.md` | Agent 来源：内置、自定义、Team Agent |
| `04_TASK_SYSTEM.md` | 任务系统：七种任务类型、状态机、task-notification |
| `05_MULTI_AGENT.md` | 多 Agent 协作：并行策略、错误恢复 |
| `06_CONTEXT_COMPRESSION.md` | 上下文压缩：三层压缩体系详解 |

## 2. 核心问题

Agent 编排系统围绕三个核心问题展开：

**1. Agent 是什么？如何控制它的生命周期？**
- Agent 是独立运行的 AI 实例，有自己的上下文和工具池
- 通过 AgentTool 启动、SendMessage 继续、TaskStop 停止

**2. 多个 Agent 如何协作？**
- 同步模式：消息实时返回给父 Agent
- 异步模式：通过 task-notification 异步通知
- 官方版并行：伪并行（共享 AppState/文件系统）
- 隐藏并行模式：worktree（文件隔离）、Agent Team（进程隔离）

**3. 如何管理 Agent 的执行？**
- 七种任务类型通过 TaskState 联合类型统一管理
- 状态机：pending → running → completed/failed/killed
- task-notification 在 Agent 完成时发送，包含状态、结果、统计信息

## 3. 关键文件

| 文件 | 作用 |
|------|------|
| `src/tools/AgentTool/AgentTool.tsx` | Agent 工具定义和调用入口 |
| `src/tools/AgentTool/runAgent.ts` | Agent 执行核心逻辑（AsyncGenerator） |
| `src/tools/AgentTool/loadAgentsDir.ts` | Agent 定义加载：内置 + 自定义 |
| `src/tools/SendMessageTool/` | Agent 间通信 |
| `src/tools/TaskStopTool/` | 停止 Agent |
| `src/tasks/types.ts` | 七种任务类型定义 |
| `src/tasks/LocalAgentTask/LocalAgentTask.ts` | 异步 Agent 生命周期管理 |
| `src/coordinator/coordinatorMode.ts` | Coordinator 模式 |
| `src/tools/shared/spawnMultiAgent.ts` | Team Agent 机制 |

## 4. 学习路径

1. [01_CORE_CONCEPTS.md](01_CORE_CONCEPTS.md) — Agent 是什么？有哪些类型？如何控制生命周期？
2. [02_RUN_AGENT.md](02_RUN_AGENT.md) — runAgent 如何执行？消息如何流转？
3. [03_AGENT_ORIGINS.md](03_AGENT_ORIGINS.md) — Agent 从哪来？内置、自定义、Team Agent 的区别？
4. [04_TASK_SYSTEM.md](04_TASK_SYSTEM.md) — 七种任务如何管理？task-notification 如何工作？
5. [05_MULTI_AGENT.md](05_MULTI_AGENT.md) — 如何并行？各并行模式有什么代价？
6. [06_CONTEXT_COMPRESSION.md](06_CONTEXT_COMPRESSION.md) — 上下文如何压缩？三层压缩体系详解

---

## 下一步

← [02_STARTUP.md](../02_STARTUP.md) 学习启动流程
→ [01_CORE_CONCEPTS.md](01_CORE_CONCEPTS.md) 开始学习 Agent 核心概念
