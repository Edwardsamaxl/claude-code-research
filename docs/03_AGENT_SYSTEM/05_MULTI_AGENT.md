# 05 - 多 Agent 协作

> 学习 Claude Code 中的并行协作机制：官方版的 run_in_background 与 worktree 隔离，以及三种隐藏并行模式

## 1. 学习目标

学完本章后，你将理解：

- run_in_background 的伪并行机制（共享 AppState/主进程）
- worktree 隔离：文件隔离但执行仍是串行
- Coordinator Mode 的任务分解与伪并行策略
- Agent Team 的进程级真正并行（高资源消耗）
- 各模式的错误恢复机制

## 2. 官方版的并行能力

### 2.1 run_in_background — 伪并行

```typescript
AgentTool({
  prompt: "修复 bug",
  run_in_background: true,
})
```

启用后台运行后，多个 Agent 可以**同时存在**，但它们：

| 共享资源 | 说明 |
|---------|------|
| AppState | 所有 Agent 共享同一个全局状态 |
| 文件系统 | 没有协调机制避免文件冲突 |
| 工具调用 | 工具权限、文件系统操作都是串行的 |

**本质**：多个 Agent 可以"同时运行"，但对文件的操作是串行的。因为没有锁机制，如果两个 Agent 同时写入同一个文件，后写入的会覆盖前面的。

### 2.2 worktree 隔离

```typescript
AgentTool({
  prompt: "重构 auth 模块",
  isolation: "worktree",  // 启用 worktree 隔离
})
```

worktree 是 git 的功能，可以在同一仓库创建多个工作目录。Claude Code 利用这个机制实现**文件级别的隔离**：

```
主仓库 (main)
    ├── worktree-1 (agent-1 修改 auth/)
    ├── worktree-2 (agent-2 修改 api/)
    └── worktree-3 (agent-3 修改 ui/)
```

每个 Agent 有**独立的文件副本**，可以修改不同区域的文件，互不干扰。

**注意**：worktree 只解决文件冲突问题，但执行仍然是串行的。所有 Agent 仍然共享同一个 Claude Code 进程，调用 AI API 时仍然需要排队。

**限制**：
- 只适用于 git 仓库
- 需要支持 worktree 的文件系统
- worktree 创建和切换有开销

### 2.3 官方版的并行策略

| 模式 | 是否真正并行 | 限制 |
|------|-------------|------|
| run_in_background | 伪并行 | 共享 AppState 和文件系统 |
| isolation: worktree | 真正并行 | 仅限不同文件区域 |

**实际建议**：
- 无关依赖的任务 → run_in_background 并行
- 需要修改同一文件 → 必须串行
- 需要修改不同文件区域 → 可以用 worktree 隔离并行

## 3. Coordinator Mode

### 3.1 启用方式

```bash
feature('COORDINATOR_MODE') && CLAUDE_CODE_COORDINATOR_MODE=1 bun run dev
```

### 3.2 核心概念

Coordinator 模式下，主 Agent 变成**协调者**，负责任务分解和分配：

```typescript
// Coordinator 的职责
1. 分解任务为子任务
2. 并行启动多个 Worker
3. 收集 task-notification
4. 综合结果返回用户
```

### 3.3 并行化策略

| 阶段 | 策略 | 说明 |
|------|------|------|
| 研究 | 可并行 | 多个 Worker 同时探索不同角度 |
| 实施 | 串行 | 一次只在一个文件区域实施，避免冲突 |
| 验证 | 可并行 | 验证不阻塞新的实施 |

### 3.4 Worker 配置

```typescript
const workerTools = isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)
  ? [BASH, READ, EDIT]  // 简单模式：仅核心工具
  : ASYNC_AGENT_ALLOWED_TOOLS  // 完整模式：异步可用工具
```

## 4. Agent Team 模式

### 4.1 启用方式

```bash
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 bun run dev
```

### 4.2 核心概念

Agent Team 提供**进程级隔离**，每个队友运行在独立的 tmux 窗口或进程中：

```typescript
spawnTeammate({
  name: "reviewer",           // 可寻址名称
  prompt: "审查代码变更...",
  team_name: "pr-review",
})
```

### 4.3 进程级隔离

**Agent Teams 和 worktree 的核心区别**：

| 维度 | worktree | Agent Teams |
|------|----------|------------|
| 文件隔离 | 有（独立文件副本） | 可选 |
| 执行并行 | 无（共享主进程） | 有（独立进程） |
| 资源消耗 | 低（共享进程） | 高（每个队友一个完整 Claude Code 实例） |

**进程隔离的真正含义**：
- 每个队友是独立的操作系统进程
- 有自己的执行循环（query() loop）
- 可以独立调用 AI API、独立使用工具
- 不需要等待主 Agent 调度，可以同时工作

**资源消耗**：Agent Teams 资源消耗很大，每个队友都是一个完整的 Claude Code 实例（独立的 Node.js 进程、独立的 AI API 调用）。这是用资源换并行的代价。

```
worktree 模式：
主 Agent (同一进程，串行调度)
  └── Agent-1 (worktree-1) → 等待 → Agent-2 (worktree-2)
  （文件隔离，但执行还是等一个完成才能下一个）

Agent Teams 模式：
主 Agent (进程A) ←→ 队友-1 (进程B, 独立运行) ←→ 队友-2 (进程C, 独立运行)
  （真正并行，但每个进程都消耗资源）
```

### 4.4 通信机制

队友之间通过 **mailbox**（进程间通信）通信，不是 task-notification：

```typescript
// 发送消息给队友
await writeToMailbox(recipientName, {
  from: senderName,
  text: "请审查 PR #123",
  timestamp: new Date().toISOString(),
}, teamName)
```

## 5. 错误恢复机制

### 5.1 错误恢复策略

无论哪种模式，错误恢复的核心策略都是**继续并提供更清晰指令**：

```typescript
// 检测到失败
if (result.status === 'failed') {
  // 策略：继续同一 Worker/队友，提供修正指令
  sendMessage({
    to: result.agentId,
    message: `失败原因：${result.error}。修正方法：...`
  })
}
```

### 5.2 各模式的错误恢复

| 模式 | 通信方式 | 恢复操作 |
|------|---------|---------|
| 官方串行 | task-notification | SendMessage 继续 |
| Coordinator | task-notification | SendMessage 继续或启动新 Worker |
| Agent Team | mailbox | writeToMailbox 继续或重启队友 |

### 5.3 恢复决策

| 情况 | 决策 |
|------|------|
| 失败但上下文还有效 | 继续同一 Agent，提供修正指令 |
| 失败且上下文已过时 | 启动新 Agent |
| 部分成功 | 继续同一 Agent，补充剩余任务 |

## 6. 协作模式对比

| 维度 | run_in_background | worktree | Coordinator | Agent Team |
|------|------------------|-----------|--------------|-------------|
| 文件隔离 | 无 | 有 | 无 | 可选 |
| 执行并行 | 无 | 无 | 伪并行 | 真正并行 |
| 进程隔离 | 无 | 无 | 无 | 有 |
| 资源消耗 | 低 | 低 | 中 | 高 |
| 启用方式 | 默认 | `isolation: "worktree"` | Feature Flag | 环境变量 |
| 适用场景 | 独立后台任务 | 同仓库文件隔离 | 复杂任务分解 | 持久化角色 |

## 7. 关键源文件

| 功能 | 文件 |
|------|------|
| Coordinator 实现 | `src/coordinator/coordinatorMode.ts` |
| Worker Agent | `src/coordinator/workerAgent.ts` |
| Team Agent | `src/tools/shared/spawnMultiAgent.ts` |
| Fork Agent | `src/tools/AgentTool/forkSubagent.ts` |

## 8. 本章小结

- 官方版的 run_in_background 是伪并行（共享 AppState/文件系统/主进程）
- worktree 只解决文件冲突，执行仍是主 Agent 串行调度（伪并行）
- Agent Team 提供真正的进程级并行，但资源消耗大（每个队友一个完整 Claude Code 实例）
- Coordinator 通过任务分解实现伪并行：研究并行、实施串行、验证并行
- 错误恢复策略：优先继续同一 Agent，无效则启动新 Agent

---

## 下一步

← [04_TASK_SYSTEM.md](04_TASK_SYSTEM.md) 学习任务系统
→ [README.md](README.md) 返回 Agent System 总览
