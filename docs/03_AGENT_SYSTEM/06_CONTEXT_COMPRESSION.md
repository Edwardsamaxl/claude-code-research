# 06 - 上下文压缩

> 学习 Claude Code 如何管理对话上下文大小，防止超出模型限制

## 1. 学习目标

学完本章后，你将理解：

- Claude Code 上下文压缩的整体架构（三层体系）
- 工具结果压缩：Microcompact 的两种路径（Time-based / Cached）
- 会话记忆压缩：持续提取结构化事实的机制
- 上下文折叠：AI 总结历史对话的完整流程
- 自动压缩的触发条件、阈值配置和熔断机制

## 2. 压缩体系架构

Claude Code 的上下文压缩分为**三个层次**，在不同粒度上管理上下文大小：

| 层次 | 名称 | 触发条件 | 机制 | 粒度 |
|------|------|----------|------|------|
| 第一层 | 工具结果压缩 | 单个工具结果 > 50KB，或单次 API 调用 > 200KB | 持久化到磁盘，替换为路径+预览 | 单个工具结果 |
| 第二层 | 会话记忆压缩 | 上下文中积累的结构化事实 | 持续提取到文件，压缩时复用 | 结构化事实 |
| 第三层 | 上下文折叠 | 上下文接近上限 | AI 总结历史对话，生成摘要 | 整个会话历史 |

**设计思想**：越上层的压缩代价越高，尽量用下层机制解决问题。

## 3. 第一层：工具结果压缩

工具结果压缩处理**单个工具调用产生的大结果**，防止撑爆单次请求。

### 3.1 Microcompact（微压缩）

当单个工具结果超过 **50KB** 时，将结果持久化到磁盘，只给 AI 留路径和预览。

**可压缩的工具类型**包括：文件读取、Shell 命令、搜索、文件匹配、网页搜索、网页抓取、文件编辑、文件写入等。

### 3.2 Time-based MC（时间触发）

当距离上次 AI 响应超过阈值（默认 **30 分钟**）时，清除旧的工具结果。

**为什么需要 Time-based MC？**

服务器端 Prompt Cache 有约 30 分钟的过期时间。Cache 过期后，发送相同前缀会重新计算，浪费 tokens。Time-based MC 在 Cache 过期前主动清除旧结果，避免浪费。

### 3.3 Cached MC（缓存感知）

对于支持 `cache_edits` API 的模型，使用**缓存编辑**方式删除工具结果，**保持 Prompt Cache 命中**。

#### 问题背景

Claude API 有 Prompt Cache 功能，会缓存请求前缀。如果多次发送相同前缀的内容，可以享受缓存优惠（减少实际传输的 tokens）。

但如果直接删除旧工具结果来压缩内容，会**破坏 Prompt Cache**——因为消息内容变了，缓存就失效了。

#### Cached MC 的核心思路

Cached MC 不直接修改消息本身，而是在 **API 层**做手脚：

1. 标记哪些工具结果需要删除
2. 通过 `cache_edits` API 告诉 API："请在缓存中删除这些内容"
3. 消息本身没有变化，Prompt Cache 依然有效

#### 实现示例

```typescript
// 1. 注册工具结果到状态
for (const message of messages) {
  if (message.type === 'user') {
    mod.registerToolResult(state, block.tool_use_id)
  }
}

// 2. 获取需要删除的工具 ID
const toolsToDelete = mod.getToolResultsToDelete(state)

// 3. 创建 cache_edits block，由 API 层添加到请求中
if (toolsToDelete.length > 0) {
  const cacheEdits = mod.createCacheEditsBlock(state, toolsToDelete)
  pendingCacheEdits = cacheEdits
  // 消息本身不修改，只在 API 层删除缓存引用
  return { messages }
}
```

#### 对比 Time-based MC

| 维度 | Cached MC | Time-based MC |
|------|-----------|--------------|
| 消息本身 | 不变 | 被修改（删除内容） |
| 缓存效果 | **保持命中** | **失效** |
| 机制 | API 层 `cache_edits` | 直接修改消息内容 |
| 模型要求 | 需要支持 `cache_edits` | 所有模型 |

**简单说**：Cached MC 是在不破坏缓存的前提下"假装"删除内容，实际删除由 API 层面处理。

### 3.4 Per-Message Budget（单消息预算）

单次 API 调用中，**所有工具结果的总和**不能超过 **200KB**。

Claude Code 将一次 API 调用的所有消息划为一组。边界在新的 assistant 消息出现时触发，这样可以正确处理单用户消息多工具调用的"agentic"模式。

## 4. 第二层：会话记忆压缩

会话记忆压缩是一个**持续运行的轻量级机制**，在后台提取结构化事实到文件，压缩时直接复用。

### 4.1 工作原理

会话记忆压缩在后台持续运行，通过 forked subagent 定期提取对话中的结构化事实到 `session-memory.md` 文件。当上下文需要压缩时，直接复用已提取的记忆，无需 AI 总结。

| 阶段 | 说明 |
|------|------|
| 对话进行中 | Session Memory Agent 在后台定期提取结构化事实 |
| 提取结果 | 存储到 `~/.claude/session-memory/session-memory.md` |
| 压缩触发 | 上下文接近上限时，使用已提取的记忆作为摘要 |

### 4.2 记忆模板

Session Memory 使用预定义的 Markdown 模板，包含以下章节：

| 章节 | 内容 |
|------|------|
| Session Title | 5-10 词的会话描述标题 |
| Current State | 当前工作状态和待处理任务 |
| Task specification | 用户需求和设计决策 |
| Files and Functions | 重要文件及其用途 |
| Workflow | Bash 命令及其执行顺序 |
| Errors & Corrections | 遇到的错误和修复方法 |
| Codebase and System Documentation | 重要系统组件 |
| Learnings | 经验总结 |
| Key results | 用户要求的精确输出 |
| Worklog | 逐步执行摘要 |

### 4.3 提取机制

提取在以下条件同时满足时触发：

1. **初始化阈值**：token 数量达到最低要求
2. **更新阈值**：token 增长量 + tool calls 数量超过阈值
3. **自然断点**：上次 assistant turn 有 tool calls（确保在自然断点提取）

```typescript
// 触发条件：token 阈值 + (tool calls 阈值 或 无 tool calls)
return hasMetTokenThreshold &&
  (hasMetToolCallThreshold || !hasToolCallsInLastTurn)
```

优势在于比传统 AI 总结更快、更轻量，因为不需要调用额外的 API。

### 4.4 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| minTokens | 10,000 | 最小保留 token 数 |
| minTextBlockMessages | 5 | 最小文本消息数 |
| maxTokens | 40,000 | 最大保留 token 数 |

## 5. 第三层：上下文折叠

当上下文接近上限且会话记忆不足以解决问题时，使用**传统的 AI 总结**。

### 5.1 折叠流程

压缩在选定位置将对话分为两部分：较旧的消息被总结，较新的消息保留。

| 阶段 | 说明 |
|------|------|
| 压缩前 | 完整的对话历史，所有消息都保留 |
| 压缩点 | 选择在某个位置之前的消息需要总结 |
| 压缩后 | 早期消息 → 折叠边界 + 摘要 → 后期消息保留 |

### 5.2 完整流程

上下文折叠的执行步骤：

1. **执行 Pre-Compact Hooks**：触发钩子函数，允许外部介入
2. **构建总结 prompt**：根据自定义指令生成总结要求
3. **调用 AI 生成摘要**：最多生成 20,000 tokens
4. **清理文件状态缓存**：折叠会清除文件状态缓存
5. **创建折叠边界标记**：标记压缩位置和元数据
6. **重新注入关键上下文**：将必要信息重新注入压缩后的上下文

### 5.3 总结 Prompt 设计

AI 需要输出的摘要包含以下内容：

| 要求 | 说明 |
|------|------|
| Primary Request and Intent | 主要需求和意图 |
| Key Technical Concepts | 关键技术概念 |
| Files and Code Sections | 重要文件和代码段（带代码片段） |
| Errors and fixes | 错误和修复方法 |
| Problem Solving | 问题解决方法 |
| All user messages | 所有用户消息 |
| Pending Tasks | 待处理任务 |
| Current Work | 当前工作 |
| Optional Next Step | 可选的下一个步骤 |

**格式要求**：AI 输出 `<analysis>` 块（思考过程，会被剥离）和 `<summary>` 块（实际摘要内容）。

### 5.4 折叠边界标记

折叠后插入边界标记，包含以下元数据：

| 字段 | 说明 |
|------|------|
| trigger | 触发方式：auto 或 manual |
| preCompactTokenCount | 压缩前 token 数 |
| summaryTokenCount | 摘要 token 数 |

### 5.5 折叠后上下文恢复

折叠会清除文件状态缓存，需要重新注入关键上下文：

| 类型 | 内容 | 预算 |
|------|------|------|
| 文件附件 | 最近访问的 5 个文件 | 最多 50,000 tokens |
| 异步 Agent 输出 | 后台任务摘要 | - |
| 计划 | 当前 Plan 内容 | - |
| Plan Mode 指令 | 保持 Plan Mode | - |
| 技能 | 已使用技能内容 | 最多 25,000 tokens |
| 工具/MCP delta | 工具列表变更 | - |
| SessionStart Hooks | 钩子消息 | - |

## 6. 自动压缩触发机制

### 6.1 触发条件

自动压缩在以下条件满足时触发：

1. **未禁用**：环境变量 `DISABLE_COMPACT` 和 `DISABLE_AUTO_COMPACT` 未设置
2. **查询源限制**：不是来自 session_memory 或 compact 自身的查询
3. **Token 阈值**：上下文 token 数量达到模型有效上下文窗口减去 13,000 buffer

```typescript
// 计算阈值：有效上下文窗口 - buffer
const effectiveContextWindow = getEffectiveContextWindowSize(model)
const threshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS  // 13,000
```

**阈值配置**：

| 参数 | 值 | 说明 |
|------|-----|------|
| AUTOCOMPACT_BUFFER_TOKENS | 13,000 | 保留给输出的空间 |
| WARNING_THRESHOLD_BUFFER_TOKENS | 20,000 | 警告阈值 |
| ERROR_THRESHOLD_BUFFER_TOKENS | 20,000 | 错误阈值 |
| MANUAL_COMPACT_BUFFER_TOKENS | 3,000 | 手动压缩的缓冲区 |

### 6.2 熔断机制

连续失败 3 次后停止自动压缩尝试，防止无限重试浪费资源。

### 6.3 自动压缩流程

1. 检查是否应该压缩
2. 尝试 Session Memory 压缩（轻量级优先）
3. 如果 Session Memory 不可用，执行完整 Context Collapse（AI 总结）

## 7. 触发条件速查表

| 机制 | 触发条件 | 是否自动 | 可配置 |
|-------|----------|----------|--------|
| Microcompact (Time-based) | 距离上次响应 > 30 分钟 | 是 | 是，gapThresholdMinutes |
| Microcompact (Cached) | 工具结果数量 > 阈值 | 是 | 是，GrowthBook |
| Per-Message Budget | 工具结果总和 > 200KB | 是 | 否 |
| Session Memory 提取 | token 增长 + tool calls 阈值 | 是 | 是，GrowthBook |
| Session Memory 压缩 | 上下文接近阈值 | 是 | 是 |
| Context Collapse | Token 数量 >= 阈值 - 13,000 | 是 | 是，AUTO_COMPACT_WINDOW |

## 8. 关键设计决策

### 8.1 为什么需要三层压缩？

越上层的压缩代价越高：

- **工具结果压缩**：本地操作，无需 API 调用
- **会话记忆压缩**：利用已有的结构化数据，无需重新总结
- **上下文折叠**：需要额外的 API 调用来生成摘要

Claude Code 尽量用下层机制解决问题，只有在下层不够用时才触发上层。

### 8.2 为什么让 AI 自己总结？

1. **质量更高**：AI 理解对话的语义和意图，生成的摘要更准确
2. **单次 API 调用**：不需要额外的 LLM
3. **上下文完整**：AI 可以看到完整的对话历史

### 8.3 会话记忆 vs AI 总结

| 维度 | Session Memory | AI 总结 |
|------|---------------|---------|
| 速度 | 快（复用已有数据） | 慢（需要额外 API 调用） |
| 成本 | 低 | 高 |
| 实时性 | 持续更新 | 压缩时才生成 |
| 准确性 | 可能过时 | 压缩时最新 |

## 9. 关键源文件

| 功能 | 文件 |
|------|------|
| 微压缩核心 | `src/services/compact/microCompact.ts` |
| Cached MC | `src/services/compact/cachedMicrocompact.js` |
| 压缩主逻辑 | `src/services/compact/compact.ts` |
| 自动压缩触发 | `src/services/compact/autoCompact.ts` |
| API Round 分组 | `src/services/compact/grouping.ts` |
| 压缩 Prompt | `src/services/compact/prompt.ts` |
| 会话记忆 | `src/services/SessionMemory/sessionMemory.ts` |
| 会话记忆模板 | `src/services/SessionMemory/prompts.ts` |
| 会话记忆压缩 | `src/services/compact/sessionMemoryCompact.ts` |
| Token 估算 | `src/services/tokenEstimation.ts` |

## 10. 本章小结

- Claude Code 采用三层压缩：工具结果压缩 → 会话记忆压缩 → 上下文折叠
- **工具结果压缩**包括 Microcompact（Time-based / Cached）和 Per-Message Budget
- **Cached MC** 通过 API 层的 `cache_edits` 实现"假删除"，保持 Prompt Cache 命中
- **会话记忆压缩**持续提取结构化事实到文件，压缩时直接复用
- **上下文折叠**让 AI 总结历史对话，是最后的手段
- 自动压缩通过 `shouldAutoCompact` 检查触发条件，熔断机制防止无限重试
- Session Memory 相比 AI 总结更快、更轻量，但可能包含过时信息

---

## 下一步

← [05_MULTI_AGENT.md](05_MULTI_AGENT.md) 学习多 Agent 协作
→ [README.md](../README.md) 返回 Agent System 总览
