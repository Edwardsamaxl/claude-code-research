# 04 - 工具系统

> 工具是 Claude Code 能力的边界。AI 只能在工具范围内行动，工具定义了 Agent 能做什么、不能做什么。

## 1. 学习目标

学完本章后，你将理解：

- 为什么说"工具是 Claude Code 的核心"
- 工具的完整分类体系
- Agent 调用工具的完整流程（决策 → 权限 → 执行 → 结果）
- MCP 如何扩展工具系统（通过 MCPTool 模板适配）
- 工具系统的设计优势（对比其他 AI CLI）

## 2. 为什么工具是核心

| 对比 | 传统 CLI | Claude Code |
|------|----------|-------------|
| 流程 | 用户 → API → 文本输出 → 用户手动执行 | 用户 → AI推理 → 工具执行 → 结果 → AI继续 → 完成 |
| 本质 | AI 只能建议 | AI 直接调用工具自动完成 |

Claude Code 的本质区别在于：**工具调用是原子的能力单元，AI 在工具边界内做决策**。

| 对比 | AI CLI | Claude Code |
|------|---------|-------------|
| 能力边界 | 无边界，AI 只能建议 | 工具定义边界，AI 只能在边界内行动 |
| 执行方式 | 用户复制粘贴执行 | AI 直接调用工具自动完成 |
| 工具组合 | 无 | AI 推理决定并行/串行组合 |

## 3. 工具总览

### 3.1 工具分类表

| 类别 | 工具 | 说明 |
|------|------|------|
| **文件读取** | FileRead | 读取文件内容 |
| **文件编辑** | FileEdit | 对文件进行修改 |
| **文件写入** | FileWrite | 创建或覆盖文件 |
| **模式匹配** | Glob, Grep | 按模式搜索文件/内容 |
| **Shell执行** | Bash, PowerShell | 执行系统命令 |
| **网络搜索** | WebSearch, WebFetch | 搜索和获取网页 |
| **Agent管理** | Agent, SendMessage, TaskStop, TaskCreate, TaskGet, TaskUpdate, TaskList, TaskOutput | 启动子Agent、进程间通信、任务管理 |
| **团队协作** | TeamCreate, TeamDelete | 多Agent团队管理 |
| **配置管理** | Config, Skill | 技能调用、配置管理 |
| **计划控制** | EnterPlanMode, ExitPlanMode | 进入/退出计划审批模式 |
| **交互模式** | REPL, Brief | 交互式REPL、Brief模式 |
| **其他** | NotebookEdit, LSP, WebBrowser, CtxInspect | Jupyter、Language Server、浏览器控制 |
| **MCP** | MCPTool | 封装所有MCP服务器工具 |

### 3.2 工具数量

Claude Code 内置 **50+ 工具**，通过 MCP 可连接更多外部工具。

### 3.3 简单模式

`CLAUDE_CODE_SIMPLE=1` 只暴露核心工具：

| 工具 | 说明 |
|------|------|
| Bash | Shell 命令执行 |
| FileRead | 文件读取 |
| FileEdit | 文件编辑 |
| Agent, TaskStop, SendMessage | Coordinator 模式下额外暴露 |

## 4. 统一接口设计

### 4.1 buildTool 工厂

所有工具通过 `buildTool()` 创建，遵循统一接口：

```typescript
// src/Tool.ts
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,           // 提供安全默认值
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

### 4.2 工具接口

```typescript
// src/Tool.ts
export type Tool<Input, Output, P> = {
  name: string                    // 工具唯一标识
  inputSchema: Input              // Zod 输入模式
  maxResultSizeChars: number      // 超过此大小存磁盘（默认50KB）

  call(args, context): Promise<ToolResult<Output>>  // 核心执行
  description(input, options): Promise<string>     // 工具描述（给AI看）
  checkPermissions(input, context): Promise<PermissionResult>  // 权限检查

  // 工具特性
  isConcurrencySafe?(input): boolean   // 是否可并行
  isReadOnly?(input): boolean          // 是否只读
  isDestructive?(input): boolean       // 是否破坏性
  isEnabled(): boolean                  // 是否启用
}
```

### 4.3 默认值

```typescript
// src/Tool.ts
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,  // 默认不允许并行
  isReadOnly: () => false,         // 默认假设会写
  isDestructive: () => false,
  checkPermissions: () => ({ behavior: 'allow' }),
}
```

**设计价值**：`buildTool()` 提供默认值，工具开发者只需关注核心逻辑，代码复用率高。

## 5. 工具执行流程

工具调用由 AI 推理驱动，完整流程如下：

| 步骤 | 说明 |
|------|------|
| 1 | 用户输入 |
| 2 | AI 推理选择工具 |
| 3 | 权限检查 |
| 4 | 执行工具 |
| 5 | 结果处理 |
| 6 | AI 继续/结束 |

### 5.1 步骤详解

| 步骤 | 说明 | 代码位置 |
|------|------|----------|
| 1. AI推理 | 根据工具描述和场景选择工具 | model推理 |
| 2. 权限检查 | validateInput() 检查输入合法性，checkPermissions() 检查授权 | buildTool默认值 |
| 3. 工具执行 | tool.call() 执行具体逻辑 | 各工具实现 |
| 4. 结果处理 | 小于50KB直接返回，大于50KB存磁盘返回预览 | toolResultStorage.ts |

### 5.2 并行与串行决策

AI 根据工具间的依赖关系决定并行或串行：

| 场景 | 方式 | 原因 |
|------|------|------|
| 读取5个无关文件 | 并行 | 无依赖，同时执行更快 |
| 读取文件A后编辑A | 串行 | 有依赖，必须先读后写 |
| 搜索后编辑搜索结果 | 先并行后串行 | 先独立搜索，再按序编辑 |

**系统提示词**中告诉 AI：无关依赖的工具可以并行，有依赖的要串行。

### 5.3 权限检查链

```typescript
// 工具执行前的检查链
validateInput()   // 输入是否合法（路径存在？参数正确？）
       ↓
checkPermissions()  // 用户是否授权此操作
       ↓
canUseTool()     // 权限规则匹配（alwaysAllow/alwaysDeny）
```

## 6. MCP 扩展机制

### 6.1 MCP 不是工具，是工具扩展协议

MCP (Model Context Protocol) 是 Anthropic 定义的**模型与外部服务交互协议**。它的核心价值在于：**让外部工具可以像内置工具一样被 AI 调用**。

### 6.2 MCPTool 模板适配

Claude Code 通过 MCPTool 模板实现 MCP 工具：

```typescript
// src/tools/MCPTool/MCPTool.ts
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',
  maxResultSizeChars: 100_000,
  async call() { return { data: '' } },
  // ...其他方法由 MCP 客户端动态填充
})
```

当 MCP 服务器连接时：

```typescript
// src/services/mcp/client.ts
// 从 MCP 服务器获取工具列表
const result = await client.client.request({ method: 'tools/list' }, ListToolsResultSchema)

// 为每个 MCP 工具创建适配实例
return toolsToProcess.map((tool): Tool => ({
  ...MCPTool,  // 复用模板
  name: buildMcpToolName(client.name, tool.name),  // mcp__server__tool
  mcpInfo: { serverName: client.name, toolName: tool.name },
  description() { return tool.description ?? '' },
  inputSchema: tool.inputSchema,
  call(args, context, canUseTool, parentMessage, onProgress) {
    // 调用 MCP 服务器执行工具
    return client.client.request({ method: 'tools/call', params: { name: tool.name, arguments: args } })
  }
}))
```

### 6.3 MCP 扩展架构

| 层级 | 说明 |
|------|------|
| MCP 服务器 | 文件系统、Git、数据库等外部服务 |
| MCP 协议 | JSON-RPC 通信 |
| MCPTool 适配层 | 模板填充 |
| Claude Code 工具 | 成为可调用的工具 |

| 对比 | 内置工具 | MCP 工具 |
|------|----------|----------|
| 定义位置 | src/tools/*.ts | MCP 服务器 |
| 工具名 | FileRead, Bash | mcp__server__tool |
| 描述来源 | 源代码 | MCP 服务器返回 |
| 执行方式 | 直接调用 | 通过 MCP 协议调用远程 |

### 6.4 MCP 工具的特殊处理

MCP 工具继承 MCPTool 模板的特性：

| 特性 | 说明 |
|------|------|
| 权限控制 | 通过 `mcpInfo` 匹配权限规则 |
| 结果处理 | 超过 100KB 存磁盘 |
| 进度展示 | 支持 progress 消息回传 |
| 并发安全 | 使用 MCP 服务器声明的 `readOnlyHint` |

## 7. 工具系统设计优势

### 7.1 对比其他 AI CLI

| 维度 | 传统 AI CLI | Claude Code |
|------|-------------|-------------|
| 工具定义 | 每个工具独立实现，无统一标准 | 统一 `Tool` 接口，`buildTool()` 工厂 |
| 工具组合 | 无，只能串行调用 | 工具可嵌套，AI 决定并行/串行 |
| 权限控制 | 无或简单 | 完整的权限检查和用户确认机制 |
| 结果处理 | 纯文本 | 统一渲染机制，支持进度、错误、拒绝等状态 |
| 扩展方式 | 修改代码 | Feature Flag 和 MCP 协议 |
| 工具发现 | 全部加载 | ToolSearch 延迟加载 |

### 7.2 统一接口的设计价值

**代码复用**：`buildTool()` 提供默认值，工具开发者只需关注核心逻辑。

```typescript
// 创建一个工具变得非常简单
export const FileEditTool = buildTool({
  name: 'Edit',
  async call(input, context) {
    // 核心逻辑：读写文件、生成 patch
    return { data: { filePath, ... } }
  },
  // 其他方法使用默认值
})
```

**权限系统一致性**：所有工具共享同一套权限检查机制。

**UI 渲染一致性**：所有工具结果通过相同的渲染流程展示。

### 7.3 工具可嵌套组合

这是 Claude Code 区别于其他 AI CLI 的关键：

```typescript
// 工具调用可以嵌套
Read(file="src/main.ts")
  ↓
结果传递给 Grep(pattern="TODO")
  ↓
结果传递给 Edit(old_string="TODO", new_string="DONE")
```

AI 根据依赖关系自动决定并行或串行执行，无需人工干预。

## 8. 关键源文件

| 文件 | 作用 |
|------|------|
| `src/Tool.ts` | 工具接口定义与 buildTool 工厂 |
| `src/tools.ts` | 工具注册与组装、assembleToolPool |
| `src/tools/MCPTool/MCPTool.ts` | MCP 工具模板 |
| `src/services/mcp/client.ts` | MCP 客户端，工具适配层 |
| `src/tools/BashTool/BashTool.tsx` | Shell 命令执行 |
| `src/tools/FileEditTool/FileEditTool.ts` | 文件编辑 |
| `src/tools/AgentTool/AgentTool.tsx` | Agent 启动与管理 |

## 9. 本章小结

- **工具是核心**：工具定义能力边界，AI 只能在边界内行动
- **统一接口**：所有 50+ 工具通过 `buildTool()` 创建，遵循相同契约
- **AI 驱动**：工具调用由 AI 推理决定（选什么工具、并行还是串行）
- **权限可控**：每次执行都经过 `validateInput` 和 `checkPermissions` 检查
- **MCP 扩展**：MCPTool 模板适配外部工具，通过 MCP 协议调用
- **可扩展**：通过 Feature Flag 和 MCP 支持动态扩展

---

## 下一步

← [03_AGENT_SYSTEM/README.md](03_AGENT_SYSTEM/README.md) 返回 Agent 系统
→ [05_COMMANDS.md](05_COMMANDS.md) 学习命令系统
