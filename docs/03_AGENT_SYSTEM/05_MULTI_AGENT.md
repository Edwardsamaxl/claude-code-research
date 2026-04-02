# 05 - 多 Agent 协作

> 学习 Claude Code 中的多 Agent 并行协作机制：Agent Team、Fork 与 Coordinator

## 1. 学习目标

学完本章后，你将理解：

- Agent Team 的核心概念：什么是队友、如何创建、如何通信
- 三种隔离模式的本质区别：in-process、split-pane、separate-window
- Fork Mode：继承上下文与 prompt cache 优化
- Coordinator Mode：任务分解与 Worker 调度策略
- 各模式的适用场景与资源消耗

## 2. Agent Team 核心概念

### 2.1 什么是 Agent Team

Agent Team 是一种**多 Agent 协作模式**，主 Agent（Leader）可以创建多个队友（Teammate）来并行完成复杂任务。

| 操作 | 说明 |
|------|------|
| `spawnTeammate("reviewer")` | Leader 创建 reviewer 队友 |
| `spawnTeammate("coder")` | Leader 创建 coder 队友 |
| Mailbox | 队友之间通过 Mailbox 通信 |

**Agent Team vs 普通 Subagent**：

| 维度 | 普通 Subagent | Agent Team |
|------|---------------|------------|
| 创建方式 | `AgentTool` 隐式创建 | `spawnTeammate` 显式创建 |
| 生命周期 | 任务级（TaskStop 即结束） | 会话级（可持久存在） |
| 通信方式 | task-notification | Mailbox（双向消息） |
| 身份 | 无固定身份 | 有可寻址名称（如 @reviewer） |

### 2.2 创建队友

```typescript
// 通过 spawnTeammate 创建队友
spawnTeammate({
  name: "reviewer",           // 可寻址名称
  prompt: "审查代码变更...",
  team_name: "pr-review",     // 所属团队
  plan_mode_required: false,
})
```

创建后的队友会出现在 UI 中，可以接收来自 Leader 或其他队友的消息。

### 2.3 通信机制：Mailbox

Agent Team 使用 **Mailbox** 进行进程间通信。Mailbox 是基于文件系统的消息队列：

```
~/.claude/teams/{team_name}/inboxes/{agent_name}.json
```

**消息类型**：

| 类型 | 用途 |
|------|------|
| 文本消息 | 队友之间的日常通信 |
| `permission_request` | Worker 请求权限（如执行危险操作） |
| `shutdown_request` | Leader 请求关闭队友 |
| `plan_approval_request` | Worker 请求计划审批 |

Mailbox 使用**文件锁**避免并发写入冲突，允许多个 Claude Code 实例同时操作。

## 3. 三种隔离模式

spawnTeammate 支持三种后端隔离模式，核心区别在于**资源消耗**和**隔离程度**。

### 3.1 模式对比

| 模式 | 隔离级别 | 资源消耗 | 通信延迟 | 适用场景 |
|------|----------|----------|----------|----------|
| **in-process** | AsyncLocalStorage | 低 | 低 | 快速迭代、频繁通信 |
| **split-pane** | 进程（tmux/iTerm2 分屏） | 中 | 中 | 视觉化协作 |
| **separate-window** | 进程（tmux 独立窗口） | 中 | 中 | 独立工作区 |

### 3.2 in-process 模式

**核心原理**：使用 Node.js 的 `AsyncLocalStorage` 在同一进程内模拟隔离。

```
主进程
├── Leader Agent（AsyncLocalStorage context: leader）
└── Teammate Agent（AsyncLocalStorage context: teammate）
    └── 共享文件系统和 AppState，但 setAppState 隔离
```

**关键特性**：
- 不创建新进程，资源消耗最低
- 通信通过函数调用而非文件 I/O
- 共享文件系统（可同时读写同一文件，有冲突风险）
- 适合需要频繁消息交换的场景

### 3.3 split-pane / separate-window 模式

**核心原理**：在 tmux 或 iTerm2 中创建独立进程。

```
tmux session
├── Leader pane（Claude Code 主进程）
└── Teammate pane（独立 Claude Code 进程）
```

**关键特性**：
- 每个队友是完全独立的操作系统进程
- 有独立的文件描述符、内存空间、网络连接
- 通信需要通过 Mailbox（文件 I/O）
- 可以独立调用 AI API，真正并行执行

### 3.4 模式选择建议

- **in-process**：快速原型、需要频繁通信、低资源预算
- **tmux 分屏/窗口**：需要真正并行、资源充足、需要独立工作区

## 4. Fork Mode

Fork 是一种**轻量级并行机制**，通过 `feature('FORK_SUBAGENT')` 启用。

### 4.1 Fork vs spawnTeammate

| 维度 | Fork | spawnTeammate |
|------|------|---------------|
| 触发方式 | 省略 `subagent_type` | 显式调用 spawn |
| 上下文 | **继承父 Agent 完整对话** | 独立上下文 |
| 用途 | 并行执行同一任务的不同变体 | 创建持久化角色 |
| 生命周期 | 任务级 | 会话级 |
| 通信 | task-notification | Mailbox |

### 4.2 Fork 的本质

当你对 Leader 说"帮我看看这个 PR，顺便也检查一下安全问题"，Leader 会 Fork 出两个子 Agent：

```
Leader 上下文（包含用户 PR 描述）
    ├── Fork A：PR 审查（继承完整上下文）
    └── Fork B：安全检查（继承完整上下文）
```

Fork 子 Agent **继承父 Agent 的完整对话历史**，包括：
- 用户最初的请求
- Leader 的分析和工具调用
- 所有工具返回结果

### 4.3 prompt cache 优化

Fork 的关键设计是**最大化 prompt cache 命中率**。

Claude API 会缓存相同前缀的请求。如果多个 Fork 请求只有末尾的 directive 不同，可以共享缓存：

```
Fork A 请求：[history + tool_results + "检查安全问题"]
Fork B 请求：[history + tool_results + "检查性能问题"]
                    ↑
              相同的前缀 → API 只传输差异部分
```

这大大减少了 token 消耗和 API 延迟。

## 5. Coordinator Mode

Coordinator 是一种**任务分解模式**，通过环境变量启用：

```bash
CLAUDE_CODE_COORDINATOR_MODE=1 bun run dev
```

### 5.1 角色分工

```
Coordinator（主 Agent）
├── 分解任务为子任务
├── 调度 Worker 执行
└── 综合结果返回用户

Worker（子 Agent）
├── 接收任务指令
├── 执行具体工作
└── 报告完成状态
```

### 5.2 调度策略

```
任务：修复认证模块的 NullPointerException

第一阶段：并行研究
    ├── Worker A：分析 auth/validate.ts 中的 null 检查
    └── Worker B：分析 auth/session.ts 中的状态管理

第二阶段：串行实施（避免文件冲突）
    └── Worker A：根据研究结果修复 validate.ts

第三阶段：并行验证
    └── Worker B：验证修复有效
```

**关键原则**：
- 研究阶段：并行（多个 Worker 同时探索）
- 实施阶段：串行（同一文件一次只有一个 Worker 修改）
- 验证阶段：可并行（不阻塞新实施）

### 5.3 何时继续 vs 重启

| 情况 | 决策 |
|------|------|
| Worker 研究了 X 文件，现在要修改 X 文件 | 继续（已有上下文） |
| Worker 研究了 X 文件，要修改 Y 文件 | 可以重启（避免噪声） |
| Worker 失败了，需要修正 | 继续（已有错误上下文） |
| 验证新代码 | 重启（独立视角） |

## 6. 各模式对比总览

| 维度 | in-process | tmux/iTerm2 | Fork | Coordinator |
|------|------------|-------------|------|-------------|
| **隔离级别** | AsyncLocalStorage | 进程 | 线程 | 线程 |
| **上下文继承** | 独立 | 独立 | **完整继承** | 独立 |
| **通信机制** | 函数调用 | Mailbox | task-notification | Mailbox + task-notification |
| **资源消耗** | 低 | 中-高 | 低 | 低 |
| **真正并行** | 否（共享进程） | **是** | 否 | 否 |
| **适用场景** | 快速迭代 | 独立并行任务 | 并行变体探索 | 复杂任务分解 |

> **延伸阅读**：Fork 和 Coordinator 两种编排模式在 [08_HIDDEN_FEATURES/05_ORCHESTRATION.md](../08_HIDDEN_FEATURES/05_ORCHESTRATION.md) 中有深入讲解，包括完整的 Coordinator System Prompt 解读、FORK_SUBAGENT 的隐式分叉机制、以及两种模式的适用场景对比。

## 7. 关键源文件

| 功能 | 文件 |
|------|------|
| spawnTeammate 入口 | `src/tools/shared/spawnMultiAgent.ts` |
| In-Process 实现 | `src/utils/swarm/spawnInProcess.ts` |
| Mailbox 通信 | `src/utils/teammateMailbox.ts` |
| Fork 实现 | `src/tools/AgentTool/forkSubagent.ts` |
| Coordinator 逻辑 | `src/coordinator/coordinatorMode.ts` |

## 8. 本章小结

- Agent Team 通过 `spawnTeammate` 创建有身份、可寻址的队友，通过 Mailbox 通信
- 三种隔离模式的核心区别：in-process（AsyncLocalStorage 低开销）、tmux/iTerm2（进程级真正并行）
- Fork 继承父 Agent 完整上下文，用于并行执行同一任务的不同变体，并优化 prompt cache
- Coordinator 通过任务分解实现伪并行：研究并行、实施串行、验证并行
- task-notification 是异步通知机制，详细格式见 [04_TASK_SYSTEM.md](04_TASK_SYSTEM.md)

---

## 下一步

← [04_TASK_SYSTEM.md](04_TASK_SYSTEM.md) 学习任务系统
→ [README.md](README.md) 返回 Agent System 总览
