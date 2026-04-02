# 09 - 练习题

> 验证你对 Claude Code 核心概念的理解，而不是代码追踪能力

## 练习说明

本练习题围绕 Claude Code 的**核心设计理念**设计，考察你对 Agent 系统、工具架构、上下文管理等概念的理解深度。

---

## 第一部分：Agent 系统

### E1: Agent 的本质

**问题**：Claude Code 的 Agent 与普通 API 调用的本质区别是什么？

<details>
<summary>参考答案</summary>

| 对比维度 | 普通 API 调用 | Claude Code Agent |
|----------|--------------|-------------------|
| 执行驱动 | 开发者代码 | AI 推理决定 |
| 工具调用 | 无（只能返回文本） | AI 推理决定调用什么工具 |
| 执行循环 | 单次请求-响应 | 循环直到任务完成或达到 maxTurns |
| 结果返回 | 一次性返回 | 通过 task-notification 异步返回 |

**核心洞察**：Agent 的本质是"AI 驱动的执行循环"，而不是"带工具的 API 调用"。

</details>

---

### E2: 同步 vs 异步 Agent

**场景**：你需要在后台运行一个代码分析任务，同时继续与用户对话。

**问题**：
1. 你应该选择同步还是异步 Agent？为什么？
2. 异步 Agent 的消息会返回给父 Agent 吗？
3. 异步 Agent 完成后如何通知父 Agent？

<details>
<summary>参考答案</summary>

1. **选择异步 Agent** — 因为需要"后台运行，同时继续对话"
2. **不会** — 异步模式下，中间消息（stream_event、assistant、user）全部丢弃，不返回父 Agent
3. **通过 task-notification** — 异步 Agent 完成后发送 `<task-notification>`，包含状态、结果、token 统计

**关键理解**：同步/异步的区别不是"能不能并行"，而是"消息流向哪里"。

</details>

---

### E3: Fork vs Subagent

**问题**：什么场景下应该使用 Fork 而不是普通 Subagent？

<details>
<summary>参考答案</summary>

| 场景 | 选择 | 原因 |
|------|------|------|
| 并行检查 PR 的安全性和性能 | **Fork** | 继承完整上下文，prompt cache 优化 |
| 创建长期存在的"代码审查员"角色 | **Subagent (spawnTeammate)** | Fork 是任务级，Teammate 是会话级 |
| 同一个任务需要多个 Worker 同时研究 | **Fork** | 继承上下文，避免重复探索 |
| Worker 之间需要 Mailbox 通信 | **Agent Team (spawnTeammate)** | Fork 通过 task-notification，Team 通过 Mailbox |

**核心理解**：Fork 是"并行执行同一任务的不同变体"，Agent Team 是"创建有持久身份的可寻址角色"。

</details>

---

### E4: 内置 Agent 的设计意图

**问题**：Explore Agent 和 Plan Agent 被设计为"One-Shot"类型，跳过 agentId/SendMessage 等字段。

1. 这样做的好处是什么？
2. 什么场景下你不应该用 One-Shot Agent？

<details>
<summary>参考答案</summary>

1. **节省 token** — 约 135 chars × 34M+ runs/week，避免为一次性任务传输交互开销
2. **需要继续对话时** — One-Shot Agent 执行完就结束，无法通过 SendMessage 继续交互。如果需要多轮对话，应该用普通 Subagent

**设计思想**：根据任务性质选择 Agent 类型，而不是总是用通用 Agent。

</details>

---

## 第二部分：工具系统

### E5: 工具接口统一的意义

**问题**：如果工具接口不统一（每个工具自己定义 call 签名），会出现什么问题？

<details>
<summary>参考答案</summary>

| 问题领域 | 具体影响 |
|----------|----------|
| 工具组合 | AgentTool 无法统一调用其他工具（每个工具 call 签名不同） |
| 权限管理 | 无法统一在入口点做权限检查（checkPermissions 无统一接口） |
| UI 渲染 | 无法统一渲染工具结果（进度、错误、拒绝等状态） |
| MCP 扩展 | 外部工具无法适配统一接口 |

**核心洞察**：`buildTool()` 工厂的真正价值不是"少写代码"，而是"保证所有工具遵循同一契约"。

</details>

---

### E6: 权限模式的选择

**场景**：你需要在一个无人值守的服务器上运行 Claude Code（cron job 场景）。

**问题**：
1. 应该选择什么权限模式？
2. 各权限模式的特点是什么？

<details>
<summary>参考答案</summary>

| 权限模式 | 特点 | 适用场景 |
|----------|------|----------|
| auto | 自动决定是否提示 | 常规交互 |
| manual | 总是提示 | 需要用户确认高风险操作 |
| bypassPermissions | 跳过所有提示 | 信任环境，如 cron job |
| acceptEdits | 自动接受编辑 | 非破坏性的简单操作 |
| plan | 需要计划审批 | 高风险变更需要人工审核 |

**答案 1**：应该选择 `bypassPermissions`（无人值守无法响应提示）

</details>

---

### E7: MCP 的本质

**问题**：MCP (Model Context Protocol) 的核心价值是什么？它与内置工具的本质区别是什么？

<details>
<summary>参考答案</summary>

**核心价值**：让外部工具可以像内置工具一样被 AI 调用（协议统一）

**本质区别**：

| 维度 | 内置工具 | MCP 工具 |
|------|----------|----------|
| 定义位置 | 源代码 | MCP 服务器 |
| 工具名 | FileRead, Bash | mcp__server__tool |
| 描述来源 | 源代码 | MCP 服务器返回 |
| 执行方式 | 直接调用 | 通过 MCP 协议调用远程 |

**理解**：MCPTool 是一个"适配器模板"，把 MCP 服务器返回的工具适配成 Claude Code 的统一工具接口。

</details>

---

## 第三部分：多 Agent 协作

### E8: Coordinator 的调度策略

**场景**：Coordinator 模式下一个任务需要"研究代码库 → 实施变更 → 验证"。

**问题**：
1. 为什么研究阶段要并行，但实施阶段要串行？
2. 验证为什么要用新的 Agent（Spawn）而不是继续（Continue）？

<details>
<summary>参考答案</summary>

1. **研究并行、实施串行**：
   - 研究阶段：多角度同时探索效率高，无冲突风险
   - 实施阶段：同一文件同时修改会冲突，必须串行

2. **验证用新 Agent**：
   - 避免实现假设影响验证（独立视角）
   - 验证者不应该知道"打算怎么修"，只应该知道"修后对不对"

**核心理解**：Coordinator 的调度策略是为了"效率"和"客观性"的平衡。

</details>

---

### E9: Mailbox vs task-notification

**问题**：Agent Team 中队友之间的通信用 Mailbox，而不是 task-notification。为什么？

<details>
<summary>参考答案</summary>

| 通信特性 | task-notification | Mailbox |
|----------|-------------------|---------|
| 通信方向 | 单向（子 → 父） | 双向（任意队友） |
| 消息类型 | 结构化通知（完成/失败） | 任意文本消息 |
| 实时性 | 异步（任务结束时） | 持续轮询 |
| 适用场景 | 任务结果汇报 | 队友间协作对话 |

**关键理解**：task-notification 是"完成任务通知"，Mailbox 是"进程间通信机制"。

</details>

---

## 第四部分：上下文管理

### E10: 三层压缩体系的设计思想

**问题**：Claude Code 为什么需要三层压缩体系？它们各自的代价是什么？

<details>
<summary>参考答案</summary>

| 压缩层 | 机制 | 触发条件 | 代价 |
|--------|------|----------|------|
| 第一层 | 工具结果压缩 | 单个 > 50KB 或总和 > 200KB | 低（本地操作） |
| 第二层 | 会话记忆压缩 | 结构化事实积累 | 低（复用已有提取） |
| 第三层 | 上下文折叠 | 接近上下文上限 | 高（需要 AI 总结） |

**设计思想**：越上层的压缩代价越高，尽量用下层机制解决问题。

**Cached MC 的巧妙设计**：通过 `cache_edits` API "假删除"内容，保持 Prompt Cache 命中。

</details>

---

### E11: Microcompact vs Context Collapse

**问题**：什么时候应该触发 Microcompact，什么时候应该触发 Context Collapse？

<details>
<summary>参考答案</summary>

| 场景 | 选择 | 原因 |
|------|------|------|
| 单个工具结果很大（如读取大文件） | **Microcompact** | 只压缩单个工具结果 |
| 整个对话历史太长 | **Context Collapse** | 需要总结历史 |
| 需要保持 Prompt Cache 命中 | **Cached MC** | 不破坏缓存 |
| 上下文即将溢出 | **Context Collapse** | 最后手段 |

**理解**：Microcompact 是"手术刀"，Context Collapse 是"大锤"。

</details>

---

## 第五部分：隐藏功能与架构决策

### E12: KAIROS 的核心使命

**问题**：KAIROS 想解决什么问题？它通过哪四个机制实现"主动"？

<details>
<summary>参考答案</summary>

**核心问题**：Claude Code 传统上是"被动响应"模式，用户问一句，Claude 答一句。

**四个机制**：

| 机制 | 说明 | 角色 |
|------|------|------|
| Channels | MCP 服务器主动推送消息 | 外部 → Claude |
| PROACTIVE | Claude 主动向用户发送建议 | Claude → 用户 |
| Dream | 空闲时自动整理记忆 | 后台自动 |
| Cron | 定时任务自动执行 | 定时触发 |

**本质转变**：从"用户驱动"到"事件驱动 + 时间驱动"。

</details>

---

### E13: BRIDGE 的反向隧道设计

**问题**：BRIDGE 为什么采用"本地主动连接 claude.ai"的设计，而不是"claude.ai 连接本地"？

<details>
<summary>参考答案</summary>

**原因**：

1. **不需要暴露端口** — 本地机器在防火墙后，无法从外部连接
2. **不需要 ngrok** — 传统远程控制需要内网穿透工具
3. **用户体验** — 用户只需运行命令，连接由 Claude Code 维护

**架构思想**：让连接方向适应网络拓扑，而不是要求用户配置网络。

</details>

---

### E14: Feature Flag 的死代码消除

**问题**：`feature('XXX')` 在构建时内联的好处是什么？

<details>
<summary>参考答案</summary>

**好处**：

1. **最终构建不包含未启用代码** — 减少产物体积
2. **零运行时判断开销** — 判断在构建时完成
3. **精确控制** — 每个 feature 独立开关

**对比**：如果用运行时检查（`if (isFeatureEnabled('XXX')`），未启用代码也会打包进最终产物。

**理解**：`feature()` 是编译时常量，不是运行时变量。

</details>

---

## 第六部分：设计决策分析

### E15: 为什么工具是核心

**问题**：文档说"工具是 Claude Code 的核心"，而不是"AI 是核心"或"模型是核心"。谈谈你的理解。

<details>
<summary>参考答案</summary>

**三层对比**：

| 说法 | 意味着 | Claude Code 的选择 |
|------|--------|-------------------|
| AI 是核心 | AI 产生意图，工具执行 | 不够，AI 无法直接作用于世界 |
| 模型是核心 | 能力取决于模型 | 不够，模型无法读写文件、执行命令 |
| **工具是核心** | **工具定义能力边界，AI 在边界内决策** | **正确，工具是原子能力单元** |

**核心洞察**：
- 工具定义了 Claude Code 能做什么（能力边界）
- AI 决定怎么组合工具（策略）
- 模型提供推理能力（燃料）

**类比**：工具是"手脚"，AI 是"大脑"，模型是"能量"。但只有手脚没有大脑是僵尸，只有大脑没有手脚是幽灵。Claude Code 选择了让"手脚"成为边界。

</details>

---

### E16: 架构分组的哲学

**问题**：Claude Code 用"能力分组"而不是"层分组"（Controller → Service → Repository）。为什么？

<details>
<summary>参考答案</summary>

| 分组方式 | 优点 | 缺点 |
|----------|------|------|
| 层分组 | 清晰、职责分明 | 适合固定调用链 |
| 能力分组 | 灵活、可动态组合 | 需要开发者理解能力边界 |

**Claude Code 选择能力分组的原因**：

1. **业务逻辑太分散** — 工具本身既是接口又是逻辑，放在 Service 层会很别扭
2. **数据源太杂** — 文件系统、API、用户输入，数据访问模式完全不同
3. **需要动态组合** — 工具可以嵌套调用（Read → Grep → Edit），调用链无法用固定分层约束

**架构哲学**：根据问题的本质形态选择架构，而不是套用标准模式。

</details>

---

## 综合验证

### 验证清单

完成练习后，确认你理解：

- [ ] Agent 是 AI 驱动的执行循环，不是带工具的 API 调用
- [ ] 同步/异步 Agent 的区别是"消息流向哪里"
- [ ] Fork 继承完整上下文用于并行变体，Agent Team 创建持久身份
- [ ] 工具接口统一是工具组合、权限管理、UI 渲染的基础
- [ ] Coordinator 的调度策略：研究并行、实施串行、验证独立
- [ ] 三层压缩体系：工具结果 → 会话记忆 → 上下文折叠，代价递增
- [ ] KAIROS 的目标是让 Claude 从"被动响应"变成"主动工作"
- [ ] Feature Flag 在构建时内联实现死代码消除

---

## 开放思考

### Q: 如果让你重新设计

**问题**：如果让你重新设计 Claude Code 的架构，你会做什么不同的决策？

思考方向：
1. Agent 的同步/异步设计是否最优？
2. 三层压缩是否足够？有什么改进空间？
3. 工具接口统一是否限制了某些能力？

---

## 下一步

← [08_HIDDEN_FEATURES/README.md](08_HIDDEN_FEATURES/README.md) 返回隐藏功能总览
