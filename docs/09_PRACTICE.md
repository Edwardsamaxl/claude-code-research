# 练习题

> 验证你对 Claude Code 源码的理解

## 初级

### P1: 追踪 Agent 启动流程

问题：当用户输入 `/agent` 命令时，代码是如何路由到 `AgentTool.call()` 的？

**提示**：
- 查看 `src/commands.ts` 的命令注册
- 查看 `src/tools/AgentTool/AgentTool.tsx` 的工具定义

<details>
<summary>答案要点</summary>

1. `src/commands.ts` 中命令通过 `commands.ts` 注册
2. 斜杠命令路由到对应的 command handler
3. Command handler 可以调用 Tool
4. `AgentTool` 是一个 `buildTool()` 创建的工具
5. `call()` 方法是实际执行逻辑
</details>

---

### P2: 识别 Feature Flag

问题：以下代码使用了哪个 feature flag？

```typescript
if (feature('COORDINATOR_MODE')) {
  if (isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)) {
    return getCoordinatorAgents()
  }
}
```

<details>
<summary>答案</summary>

`COORDINATOR_MODE` - 通过 `process.env.CLAUDE_CODE_COORDINATOR_MODE` 环境变量控制
</details>

---

### P3: 理解权限模式

问题：创建一个 Agent 时，如何设置它的权限模式为 `bypassPermissions`？

**提示**：查看 `AgentTool.tsx` 的 `call()` 方法参数

<details>
<summary>答案要点</summary>

在 `AgentTool.call()` 时，通过 `mode` 参数设置：
```typescript
AgentTool.call({
  prompt: "...",
  mode: 'bypassPermissions'
})
```
</details>

---

## 中级

### P4: 实现一个简单 Agent

问题：假设你要实现一个 "Code Reviewer" Agent，需要：

1. 只使用 `FileRead`, `Bash`, `Grep` 工具
2. 设置系统提示为 "你是一个专业的代码审查员"
3. 最大执行 30 轮

请写出这个 Agent 的定义代码。

**提示**：参考 `src/tools/AgentTool/built-in/generalPurposeAgent.ts`

<details>
<summary>答案要点</summary>

```typescript
const CODE_REVIEWER_AGENT: BuiltInAgentDefinition = {
  agentType: 'code-reviewer',
  whenToUse: '当需要进行代码审查时使用',
  tools: ['FileRead', 'Bash', 'Grep'],
  maxTurns: 30,
  source: 'built-in',
  baseDir: 'built-in',
  getSystemPrompt: () => '你是一个专业的代码审查员...',
}
```
</details>

---

### P5: 分析 Fork 机制

问题：解释 `buildForkedMessages()` 函数为什么要过滤 `incompleteToolCalls`？

<details>
<summary>答案要点</summary>

1. Fork 需要将父 Agent 的对话历史传递给子 Agent
2. 如果父 Agent 有未完成的 tool_use 块（没有 tool_result）
3. 直接传递给子 Agent 会导致 API 错误
4. 所以需要过滤掉这些不完整的 tool_use
5. 使用 `filterIncompleteToolCalls()` 函数
</details>

---

### P6: Coordinator 决策

问题：在 Coordinator 模式下，以下场景应该选择 "继续 (SendMessage)" 还是 "新启动 (AgentTool)"？

| 场景 | 选择 |
|------|------|
| Worker A 刚完成研究，发现了 5 个需要修复的文件 | ? |
| Worker B 报告测试失败，需要修正断言 | ? |
| 需要验证 Worker C 刚写的代码 | ? |
| 用户的请求完全不相关 | ? |

<details>
<summary>答案</summary>

| 场景 | 选择 | 原因 |
|------|------|------|
| 研究完成，文件确定 | **继续** | 上下文重叠高，worker 已加载文件 |
| 测试失败修正 | **继续** | 有错误上下文，需继续同一思路 |
| 验证新代码 | **新启动** | 避免实现假设影响验证 |
| 不相关任务 | **新启动** | 无上下文重叠 |
</details>

---

## 高级

### P7: 设计新的多 Agent 模式

问题：设计一个 "Pipeline" 模式，多个 Agent 按顺序执行，每个的输出是下一个的输入。

**要求**：
1. 定义数据结构
2. 说明如何传递上下文
3. 如何处理中间失败

<details>
<summary>答案要点</summary>

```typescript
interface PipelineStage {
  name: string
  agentType: string
  prompt: string
}

interface PipelineState {
  currentStage: number
  results: Record<string, any>
}

// 1. 按顺序执行每个 stage
// 2. 每个 stage 的输出存入 results
// 3. 下一个 stage 通过 prompt 注入前一个结果
// 4. 任何 stage 失败可以停止或重试
```
</details>

---

### P8: 分析任务通知格式

问题：`task-notification` 使用 XML 格式而不是 JSON，有什么考虑？

**提示**：考虑 Claude Code 的输出特性和解析复杂度

<details>
<summary>答案要点</summary>

1. **避免与内容冲突**：用户消息可能包含 JSON 字符串
2. **可读性**：XML 有明确的标签结构
3. **流式友好**：可以逐步解析
4. **与 AI 输出格式一致**：Claude 更容易生成结构化 XML
5. **历史兼容**：可能沿用早期设计决策
</details>

---

## 验证清单

完成练习后，确认你理解：

- [ ] Agent 工具的注册和调用流程
- [ ] Feature flag 的检查机制
- [ ] 权限模式的设置和传递
- [ ] Fork 和普通 subagent 的区别
- [ ] Coordinator 的任务分解策略
- [ ] 何时选择继续 vs 新启动

---

## 思考题：为什么 Claude Code 不一样？

### Q1: 架构的本质区别

Kimi CLI 和 Claude Code 都调用 Claude API，但架构本质不同：

```
Kimi CLI: 用户 → API → 输出文本 → 用户执行
Claude Code: 用户 → AI决策 → 工具执行 → 反馈 → AI继续 → 完成
```

问题：这种"工具循环调用"的能力，从架构上看，需要解决哪些核心问题？

<details>
<summary>思考要点</summary>

1. **循环终止条件** - AI 何时知道任务完成了？（tools_completed、max_turns）
2. **状态传递** - 每次工具调用后，AI 需要看到什么？（工具结果）
3. **错误恢复** - 工具执行失败怎么办？（重试、跳过，回退）
4. **权限控制** - 哪些工具可以调用，哪些需要用户确认？（permissionMode）
5. **上下文管理** - 循环调用会导致上下文暴涨，如何压缩？（microcompact、context_collapse）
</details>

---

### Q2: 为什么工具接口要统一？

```typescript
// Claude Code 的工具接口
interface Tool {
  name: string
  prompt(): Promise<string>      // 描述工具
  inputSchema: z.ZodType        // 输入验证
  outputSchema: z.ZodType        // 输出验证
  call(args, context): Promise   // 执行逻辑
}
```

问题：如果工具接口不统一（比如每个工具自己定义 call 签名），会出现什么问题？

<details>
<summary>思考要点</summary>

1. **工具组合** - AgentTool 调用其他工具，需要统一的调用方式
2. **权限管理** - 统一的权限检查点
3. **日志/追踪** - 统一的工具调用记录
4. **UI 渲染** - 统一的工具结果展示
5. **MCP 集成** - 外部工具需要适配统一接口
</details>

---

### Q3: Agent 调用 Agent 的意义

```typescript
// Coordinator 模式下的调用链
Coordinator
  └→ Worker A (研究)
  └→ Worker B (实现)
  └→ Worker C (验证)
```

问题：为什么不直接在 Coordinator 里顺序执行所有任务，而要拆成多个 Agent？

<details>
<summary>思考要点</summary>

1. **并行化** - 研究和验证可以并行
2. **上下文隔离** - 避免实现污染研究视角
3. **错误恢复** - 一个 worker 失败不影响其他
4. **专业分工** - 不同 agent 可以有不同的工具集
5. **可扩展性** - 可以动态增加 worker
</details>

---

### Q4: "工具是核心" 与 "AI 是大脑" 的区别

| 说法 | 解释 |
|------|------|
| "AI 是大脑，工具是手脚" | AI 产生意图，工具执行 |
| "工具是核心，AI 是控制器" | 工具是能力边界，AI 在边界内决策 |

问题：Claude Code 更接近哪种？为什么？

<details>
<summary>思考要点</summary>

Claude Code 更接近"工具是核心"：

1. **工具定义了能力边界** - AI 只能在可用工具范围内行动
2. **工具是可编程接口** - AgentTool 本身也是一个工具，可以递归
3. **工具调用是原子的** - 每个调用都是完整的能力单元
4. **工具组合是显式的** - AI 决定调用什么工具，而不是直接"做"什么

"AI 是大脑"的模式更像：AI 产生自然语言描述的行动，然后...谁来执行？
</details>
