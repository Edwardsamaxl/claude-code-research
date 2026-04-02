# 05 - 工具系统

## 概览

Claude Code 的工具系统是核心抽象之一，50+ 工具都遵循统一接口。

## 工具接口

```typescript
// src/Tool.ts

export interface Tool {
  name: string
  searchHint?: string
  aliases?: string[]

  prompt: (params: PromptParams) => Promise<string>
  description: (() => string) | string

  inputSchema: z.ZodType
  outputSchema: z.ZodType

  call: (
    args: any,
    toolUseContext: ToolUseContext,
    canUseTool: CanUseToolFn,
    assistantMessage: Message | undefined,
    onProgress?: (progress: Progress) => void
  ) => Promise<ToolResult>
}

export function buildTool(definition: ToolDefinition): Tool {
  return {
    name: definition.name,
    prompt: definition.prompt,
    // ...
  }
}
```

## 工具分类

### 文件操作

| 工具 | 文件 | 用途 |
|------|------|------|
| `FileRead` | `src/tools/FileReadTool/` | 读取文件 |
| `FileEdit` | `src/tools/FileEditTool/` | 编辑文件 |
| `FileWrite` | `src/tools/FileWriteTool/` | 写入文件 |
| `Glob` | `src/tools/GlobTool/` | 文件搜索 |
| `Grep` | `src/tools/GrepTool/` | 内容搜索 |

### 执行环境

| 工具 | 文件 | 用途 |
|------|------|------|
| `Bash` | `src/tools/BashTool/` | 执行 shell 命令 |
| `PowerShell` | `src/tools/PowerShellTool/` | PowerShell 命令 |
| `REPL` | `src/tools/REPLTool/` | 交互式 REPL |

### 网络

| 工具 | 文件 | 用途 |
|------|------|------|
| `WebFetch` | `src/tools/WebFetchTool/` | HTTP GET |
| `WebSearch` | `src/tools/WebSearchTool/` | 网络搜索 |
| `WebBrowser` | `src/tools/WebBrowserTool/` | 浏览器控制 |

### AI 协作

| 工具 | 文件 | 用途 |
|------|------|------|
| `Agent` | `src/tools/AgentTool/` | 启动 subagent |
| `SendMessage` | `src/tools/SendMessageTool/` | 继续 agent |
| `TaskStop` | `src/tools/TaskStopTool/` | 停止 agent |
| `TaskCreate` | `src/tools/TaskCreateTool/` | 创建任务 |
| `TaskList` | `src/tools/TaskListTool/` | 列出任务 |
| `TaskGet` | `src/tools/TaskGetTool/` | 获取任务详情 |
| `TaskUpdate` | `src/tools/TaskUpdateTool/` | 更新任务 |
| `TaskOutput` | `src/tools/TaskOutputTool/` | 任务输出 |
| `TeamCreate` | `src/tools/TeamCreateTool/` | 创建团队 |
| `TeamDelete` | `src/tools/TeamDeleteTool/` | 删除团队 |

### MCP

| 工具 | 文件 | 用途 |
|------|------|------|
| `MCP` | `src/tools/MCPTool/` | 调用 MCP 工具 |
| `McpAuth` | `src/tools/McpAuthTool/` | MCP 认证 |
| `ListMcpResources` | `src/tools/ListMcpResourcesTool/` | 列出资源 |
| `ReadMcpResource` | `src/tools/ReadMcpResourceTool/` | 读取资源 |

### 其他

| 工具 | 文件 | 用途 |
|------|------|------|
| `Skill` | `src/tools/SkillTool/` | 调用技能 |
| `Config` | `src/tools/ConfigTool/` | 配置管理 |
| `ScheduleCron` | `src/tools/ScheduleCronTool/` | 定时任务 |
| `NotebookEdit` | `src/tools/NotebookEditTool/` | Jupyter 编辑 |
| `LSP` | `src/tools/LSPTool/` | 语言服务器 |

## 工具构建示例

```typescript
// src/tools/BashTool/BashTool.tsx

export const BashTool = buildTool({
  name: 'Bash',
  aliases: ['Shell', 'Terminal'],
  description: 'Execute shell commands',

  async prompt({ tools, getToolPermissionContext }) {
    // 返回工具描述 prompt
  },

  inputSchema: z.object({
    command: z.string(),
    timeout: z.number().optional(),
    cwd: z.string().optional(),
    run_in_background: z.boolean().optional(),
  }),

  outputSchema: z.object({
    stdout: z.string(),
    stderr: z.string(),
    exitCode: z.number(),
  }),

  async call(args, toolUseContext, canUseTool) {
    // 执行逻辑
    const result = await executeBash(args.command, {
      cwd: args.cwd,
      timeout: args.timeout,
    })
    return { stdout: result.stdout, stderr: result.stderr, exitCode: result.exitCode }
  },
})
```

## 工具权限系统

```typescript
// src/hooks/useCanUseTool.ts

export type CanUseToolFn = (
  toolName: string,
  input: unknown
) => Promise<PermissionResult>

// 权限检查流程
// 1. 检查 tool 是否在 alwaysAllow 列表
// 2. 检查 permissionMode
// 3. 如果需要提示，发送 UI 事件
// 4. 用户决定 allow/deny
```

## 工具解析

```typescript
// src/tools/AgentTool/agentToolUtils.ts

export function resolveAgentTools(
  agentDefinition: AgentDefinition,
  availableTools: Tools,
  isAsync: boolean,
): { resolvedTools: Tools; deniedTools: Tool[] } {
  // 1. 基础工具直接可用
  // 2. MCP 工具检查连接状态
  // 3. 自定义工具检查权限

  // 返回解析后的工具列表和拒绝的工具列表
}
```

## 工具使用追踪

```typescript
// 工具执行时会记录
interface ToolUse {
  name: string
  input: unknown
  output: unknown
  duration: number
  tokenUsage?: {
    inputTokens: number
    outputTokens: number
  }
}
```

---

## 待深入

- [ ] 工具的 Zod schema 验证流程
- [ ] 工具结果的渲染机制
- [ ] MCP 工具的动态发现
