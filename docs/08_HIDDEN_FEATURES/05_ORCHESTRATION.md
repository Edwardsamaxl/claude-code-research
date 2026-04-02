# 08.5 - ORCHESTRATION

> 两种 Agent 编排范式对比：COORDINATOR_MODE vs FORK_SUBAGENT

> 本文档是 [03_05](../03_AGENT_SYSTEM/05_MULTI_AGENT.md) 的扩展，深入讲解两种高级编排模式

## 1. 学习目标

学完本章后，你将理解：
- COORDINATOR_MODE 的完整 System Prompt 架构
- FORK_SUBAGENT 的隐式分叉机制
- 两种模式的适用场景对比
- 互斥关系的原因

## 2. COORDINATOR_MODE

### 2.1 角色定位

| 角色 | 职责 | 特点 |
|------|------|------|
| **Coordinator** | 任务分解者 + 结果综合者 | 调用 AgentTool 调度 Worker，通过 Mailbox 通信 |
| **Worker** | 任务执行者 | 接收 prompt，执行具体工作，报告结果 |

**与普通 Agent Team 的本质区别**：

| 维度 | 普通 Agent Team | Coordinator Mode |
|------|-----------------|------------------|
| 调度方式 | 手动 spawnTeammate | Coordinator 智能分解任务 |
| 上下文 | 各自独立 | Worker 无父上下文（需完整描述） |
| 并行策略 | 自由发挥 | 研究并行、实施串行、验证并行 |

### 2.2 启用方式

```bash
CLAURE_CODE_COORDINATOR_MODE=1 bun run dev
```

- 通过 `feature('COORDINATOR_MODE')` 控制
- 与 FORK_SUBAGENT 互斥
- 可配合 `CLAURE_CODE_SIMPLE=1` 限制 Worker 工具范围

### 2.3 Worker 工具配置

| 模式 | Worker 工具 | 说明 |
|------|-------------|------|
| **完整模式** | ASYNC_AGENT_ALLOWED_TOOLS（所有标准工具 + MCP） | 默认模式 |
| **SIMPLE 模式** | 仅 Bash、Read、Edit | 通过 CLAURE_CODE_SIMPLE=1 启用 |

```typescript
// SIMPLE 模式配置
const workerTools = isEnvTruthy(process.env.CLAURE_CODE_SIMPLE)
  ? [BASH_TOOL_NAME, FILE_READ_TOOL_NAME, FILE_EDIT_TOOL_NAME]
  : Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
```

### 2.4 完整 System Prompt 架构

Coordinator System Prompt 分为 6 个 Section：

**Section 1: Your Role**
- Coordinator 是任务分解者，帮助用户达成目标
- 直接回答问题（不需要工具的简单问题）
- Worker 结果是内部信号，不是对话伙伴

**Section 2: Your Tools**

| 工具 | 用途 |
|------|------|
| AgentTool | 启动新 Worker（subagent_type: "worker"） |
| SendMessageTool | 继续已有 Worker（发送 follow-up） |
| TaskStopTool | 停止运行中的 Worker |

**Section 3: Workers**
- 使用 subagent_type: "worker" 创建
- Workers 自主执行任务（研究、实施、验证）

**Section 4: Task Workflow**

| 阶段 | 执行者 | 目的 |
|------|--------|------|
| Research | Workers（并行） | 探索代码库、理解问题 |
| Synthesis | **Coordinator** | 综合发现、制定实施规范 |
| Implementation | Workers | 按规范实施变更 |
| Verification | Workers | 测试变更是否有效 |

**并发控制原则**：
- 研究阶段：并行（多个 Worker 同时探索不同角度）
- 实施阶段：串行（同一文件一次只有一个 Worker 修改）
- 验证阶段：可并行（独立验证，不阻塞新实施）

**Section 5: Writing Worker Prompts**

这是 Coordinator 最重要的职责。核心原则：

| 要点 | 说明 |
|------|------|
| **Worker 看不到你的对话** | 每个 prompt 必须自包含 |
| **必须综合** | 理解 Worker 发现后再指挥下一步 |
| **避免"基于你的发现"** | 自己综合，不要委托理解 |

**Continue vs Spawn 选择**：

| 情况 | 机制 | 原因 |
|------|------|------|
| 研究正好覆盖要编辑的文件 | **Continue** | Worker 已有文件上下文 + 清晰计划 |
| 研究广泛但实施狭窄 | **Spawn fresh** | 避免探索噪声 |
| 修正失败或延续最近工作 | **Continue** | 有错误上下文 |
| 验证其他 Worker 刚写的代码 | **Spawn fresh** | 独立视角 |
| 第一次尝试方向完全错误 | **Spawn fresh** | 错误上下文会污染重试 |
| 完全无关任务 | **Spawn fresh** | 无可复用上下文 |

**Section 6: Example Session**

展示了完整的 Coordinator 工作流程：并行研究 → 收到结果 → 继续 Worker 实施 → 向用户报告进度。

### 2.5 Scratchpad 目录

当 Statsig Feature Gate `tengu_scratch` 启用时，Coordinator 可以指定 Worker 的 scratchpad 目录：

```bash
Scratchpad directory: {path}
Workers can read and write here without permission prompts.
```

用途：Worker 之间共享持久化知识。

### 2.6 task-notification 格式

Worker 结果通过 `<task-notification>` XML 标签返回：

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

## 3. FORK_SUBAGENT

### 3.1 隐式分叉机制

**触发条件**：省略 `subagent_type` 参数

```typescript
// Fork 触发
AgentTool({ prompt: "检查安全问题" })  // 无 subagent_type → Fork

// 普通 Subagent
AgentTool({ subagent_type: "worker", prompt: "..." })
```

**上下文继承**：Fork 子 Agent 继承父 Agent 的完整对话历史，包括：
- 用户原始请求
- 父 Agent 的分析过程
- 所有工具调用和返回结果

### 3.2 启用条件

```typescript
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false  // 与 Coordinator 互斥
    if (getIsNonInteractiveSession()) return false  // 非交互会话禁用
    return true
  }
  return false
}
```

**互斥原因**：Coordinator 已有自己的编排模型，Fork 无法在其基础上工作。

### 3.3 prompt cache 优化

Fork 的核心设计是**最大化 API prompt cache 命中率**。

所有 Fork 请求共享相同前缀，只有末尾 directive 不同：

```
Fork A: [history + tool_results + "检查安全问题"]
Fork B: [history + tool_results + "检查性能问题"]
             ↑
      相同前缀 → API 只传输差异部分
```

**实现机制**：使用相同的占位符结果

```typescript
const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'

// 所有 tool_result 块使用相同占位符
const toolResultBlocks = toolUseBlocks.map(block => ({
  type: 'tool_result' as const,
  tool_use_id: block.id,
  content: [{ type: 'text' as const, text: FORK_PLACEHOLDER_RESULT }],
}))
```

### 3.4 Fork 子 Agent 的指令模板

Fork 子 Agent 收到包含 `<FORK_BOILERPLATE_TAG>` 的消息，核心规则：

| 规则 | 说明 |
|------|------|
| 不要提问 | 直接执行 |
| 不要编辑 | 工具调用后直接报告结果 |
| 报告格式 | Scope: / Result: / Key files: / Files changed: |
| 500 字限制 | 除非 directive 另有指定 |
| 必须以 "Scope:" 开头 | 无前缀 |

### 3.5 Worktree 隔离

当 Fork 在独立 worktree 中运行时：

```
父 Agent: /home/user/project
Fork Child: /home/user/project/worktrees/feature-xyz

// Child 会收到路径转换说明
"You are operating in an isolated git worktree at {worktreeCwd}...
Paths in the inherited context refer to the parent's working directory;
translate them to your worktree root."
```

### 3.6 递归防护

Fork 子 Agent 不能进一步 Fork（防止无限递归）：

```typescript
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false
    const content = m.message.content
    if (!Array.isArray(content)) return false
    return content.some(
      block =>
        block.type === 'text' &&
        block.text.includes(`<${FORK_BOILERPLATE_TAG}>`),
    )
  })
}
```

检测到 `<FORK_BOILERPLATE_TAG>` 表示当前在 Fork 子 Agent 中。

### 3.7 /fork 命令

快捷触发 Fork 的 slash 命令：

```typescript
const forkCmd = feature('FORK_SUBAGENT')
```

## 4. 两种模式对比

| 维度 | COORDINATOR_MODE | FORK_SUBAGENT |
|------|------------------|---------------|
| **触发方式** | 环境变量 `CLAURE_CODE_COORDINATOR_MODE=1` | 省略 `subagent_type` |
| **上下文** | Worker 独立上下文（需完整描述） | **完整继承**父对话历史 |
| **主控权** | Coordinator 主导任务分解 | 父 Agent 主导（Fork 是执行者） |
| **工具池** | 可配置（SIMPLE 模式） | 继承父工具池（`*` 通配符） |
| **并行策略** | 研究并行、实施串行 | 并行变体探索 |
| **适用场景** | 复杂任务分解 | 同一任务的不同方面检查 |
| **互斥** | 与 FORK 互斥 | 与 COORDINATOR 互斥 |
| **会话类型** | 无限制 | 交互式会话 |

### 4.1 模式选择建议

**选择 COORDINATOR_MODE 当**：
- 任务需要多阶段（研究→实施→验证）
- 需要智能的任务分解和调度
- 实施阶段需要串行控制（避免文件冲突）

**选择 FORK_SUBAGENT 当**：
- 同一任务需要并行检查不同方面（PR 审查 + 安全检查）
- 已有分析结果，需要并行执行变体
- 需要最大化 prompt cache 效率

## 5. 关键源文件

| 功能 | 文件 |
|------|------|
| COORDINATOR System Prompt | `src/coordinator/coordinatorMode.ts` |
| COORDINATOR 启用检测 | `src/coordinator/coordinatorMode.ts:isCoordinatorMode()` |
| FORK 启用检测 | `src/tools/AgentTool/forkSubagent.ts:isForkSubagentEnabled()` |
| FORK 消息构建 | `src/tools/AgentTool/forkSubagent.ts:buildForkedMessages()` |
| FORK 子 Agent 指令 | `src/tools/AgentTool/forkSubagent.ts:buildChildMessage()` |
| FORK 递归防护 | `src/tools/AgentTool/forkSubagent.ts:isInForkChild()` |
| 模式切换警告 | `src/coordinator/coordinatorMode.ts:matchSessionMode()` |

## 6. 本章小结

- COORDINATOR_MODE 通过环境变量启用，Coordinator 负责任务分解和结果综合
- Coordinator System Prompt 包含 6 个 Section，核心是"必须综合 + 正确的 Continue/Spawn 选择"
- FORK_SUBAGENT 通过省略 `subagent_type` 触发，继承父 Agent 完整上下文
- Fork 的 prompt cache 优化通过占位符实现，所有 Fork 共享相同前缀
- 两种模式互斥：Coordinator 有自己的编排模型，Fork 需要父 Agent 主导
- SIMPLE 模式可限制 Worker 工具仅剩 Bash/Read/Edit

---

## 下一步

← [04_PROACTIVE_BUDDY.md](04_PROACTIVE_BUDDY.md)
→ [练习题](../09_PRACTICE.md)
