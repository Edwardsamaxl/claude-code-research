# 08 - 服务层

> 后台服务和基础设施

## 概览

```
src/services/
├── api/               # Anthropic API 客户端
├── mcp/              # MCP 协议实现
├── compact/          # 会话压缩
├── analytics/        # 遥测和 A/B 测试
├── SessionMemory/    # 会话记忆
├── tools/           # 工具相关服务
├── lsp/             # 语言服务器协议
└── ...
```

## API 服务

```typescript
// src/services/api/claude.ts

// Anthropic API 客户端
export class ClaudeClient {
  constructor(config: APIConfig) {
    this.baseURL = config.baseURL || 'https://api.anthropic.com'
    this.apiKey = config.apiKey
  }

  async createMessage(params: CreateMessageParams): Promise<Message> {
    // 发送消息到 Claude API
    // 处理重试和速率限制
  }

  async streamMessage(params: CreateMessageParams): AsyncGenerator<Message> {
    // 流式响应
  }
}
```

### 重试机制

```typescript
// src/services/api/retry.ts

interface RetryConfig {
  maxRetries: number
  initialDelay: number
  backoffMultiplier: number
}

async function withRetry<T>(
  fn: () => Promise<T>,
  config: RetryConfig
): Promise<T> {
  let lastError: Error
  for (let i = 0; i < config.maxRetries; i++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error
      await sleep(config.initialDelay * (config.backoffMultiplier ** i))
    }
  }
  throw lastError
}
```

### 速率限制

```typescript
// src/services/api/rateLimit.ts

export class RateLimiter {
  private tokens: number
  private lastRefill: number

  async acquire(): Promise<void> {
    // 检查是否超过速率限制
    // 等待直到有可用 tokens
  }
}
```

## MCP 服务

```typescript
// src/services/mcp/

// MCP 客户端
export async function connectToServer(
  name: string,
  config: McpServerConfig
): Promise<MCPServerConnection> {
  // 1. 建立连接
  // 2. 发送 initialize
  // 3. 返回连接对象
}

// 获取 MCP 工具
export async function fetchToolsForClient(
  client: MCPServerConnection
): Promise<Tool[]> {
  // 1. 发送 tools/list
  // 2. 转换工具定义
  // 3. 返回 Tools
}
```

### MCP 通道管理

```typescript
// src/services/mcp/channel.ts

interface MCPServerConnection {
  name: string
  type: 'connected' | 'disconnected' | 'error'
  tools: Tool[]

  // 清理函数
  cleanup: () => Promise<void>
}
```

## 压缩服务

Claude Code 的上下文管理是分层设计的，包含三种压缩机制：

```typescript
// src/services/compact/
// src/services/contextCollapse/
// src/utils/toolResultStorage.ts
```

### 压缩三层体系

```
┌─────────────────────────────────────────────────────────────┐
│                    上下文管理三层                          │
├─────────────────────────────────────────────────────────────┤
│  1. Microcompact（微压缩）                               │
│     - 单个工具结果太大时存磁盘，只给 AI 留路径           │
│     - 触发：工具结果 > 50KB                              │
│     - 工具：FileRead, Grep, Glob, Bash, WebSearch 等      │
│     - 保留最近的 N 个结果（可配置）                      │
│                                                              │
│  2. Per-Message Budget（单消息预算）                      │
│     - 单次 API 调用中所有工具结果的总和限制              │
│     - 触发：工具结果总和 > 200KB                         │
│     - 逻辑：选最大的结果存磁盘，保持总大小在预算内       │
│                                                              │
│  3. Context Collapse（上下文折叠）                        │
│     - 多条消息合并为一个摘要                              │
│     - 触发：上下文接近上限（通过 GrowthBook 配置）       │
│     - 方式：**让 AI 自己总结**（不是 LLM 做摘要）        │
│     - 最大输出：20,000 tokens（COMPACT_MAX_OUTPUT_TOKENS）│
└─────────────────────────────────────────────────────────────┘
```

### 1. 微压缩（Microcompact）

**核心逻辑**：工具结果太大时，存到磁盘，AI 只看到预览路径

```typescript
// src/services/compact/microCompact.ts

// 可压缩的工具类型
const COMPACTABLE_TOOLS = new Set([
  FILE_READ_TOOL_NAME,   // 文件读取
  GREP_TOOL_NAME,        // 搜索
  GLOB_TOOL_NAME,        // 文件匹配
  WEB_SEARCH_TOOL_NAME,  // 网页搜索
  WEB_FETCH_TOOL_NAME,   // 网页抓取
  FILE_EDIT_TOOL_NAME,   // 文件编辑
  FILE_WRITE_TOOL_NAME,  // 文件写入
  ...SHELL_TOOL_NAMES,   // Shell 命令
])

// 触发：超过 50KB 就持久化
const DEFAULT_MAX_RESULT_SIZE_CHARS = 50_000
```

**两种微压缩路径**：

| 类型 | 触发条件 | 机制 | 缓存影响 |
|------|----------|------|----------|
| Cached MC | 支持 cache editing 的模型 | 删除旧结果，API 层用 cache_edits | 保持缓存命中 |
| Time-based MC | 距离上次响应超过阈值（默认 30 分钟） | 直接清空旧结果内容 | 缓存失效 |

### 2. Per-Message Budget（单消息预算）

**核心逻辑**：单次 API 调用中所有工具结果加起来不能太大

```typescript
// src/utils/toolResultStorage.ts

// 200KB 限制
export const MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200_000

// 实际处理
async function enforceToolResultBudget(messages, state) {
  // 1. 按"API 级用户消息"分组（并行工具结果合并计算）
  // 2. 选最大的 FRESH 结果存磁盘
  // 3. 保持总大小在预算内
}
```

**关键洞察**：
- 状态必须稳定，否则破坏 prompt cache
- once seen, once replaced → 决定是永久的
- 最多约 400KB 的工具结果（10 × 40KB）

### 3. Context Collapse（上下文折叠）

**核心逻辑**：让 AI 自己总结历史对话

```typescript
// src/services/compact/compact.ts

// 折叠流程
async function collapse(messages, direction) {
  // 1. 选择要折叠的消息范围
  // 2. 构建摘要提示词
  // 3. 调用 AI 生成摘要（最多 20,000 tokens）
  // 4. 替换原消息为摘要 + 边界标记
}

// 折叠方向
type Direction = 'from' | 'up_to'
// 'from': 折叠指定位置之后的消息，保留之前的
// 'up_to': 折叠指定位置之前的消息，保留之后的
```

**折叠机制**：

```
折叠前：
[用户消息1] → [助手响应1] → [用户消息2] → [助手响应2] → [用户消息3] → [助手响应3]
                                                    ↑
                                              选择这里

折叠后（direction='up_to'）：
[用户消息1] → [助手响应1] → [助手响应2]
                                          │
                         [上下文折叠边界 - 包含摘要]
                                          │
                                 [用户消息3] → [助手响应3]
```

### 触发条件

| 机制 | 触发条件 | 是否自动 |
|------|----------|----------|
| 微压缩 | 单个工具结果 > 50KB | ✅ 自动 |
| Per-Message Budget | 单次调用工具结果总和 > 200KB | ✅ 自动 |
| Context Collapse | 上下文 token 接近上限 | ✅ 自动（可通过 `/compact` 手动） |
| Time-based MC | 距离上次响应 > 30 分钟 | ✅ 自动 |

### 关于"让 AI 做总结"

**Context Collapse 是让 AI（Claude）自己总结，而不是用一个单独的 LLM 做摘要。**

流程：
1. 把要折叠的消息发给 Claude
2. 请求："请总结以下对话的关键信息"
3. Claude 生成摘要（最多 20,000 tokens）
4. 原消息替换为摘要 + 边界标记

好处：
- 保持总结的连贯性和准确性
- 只需要一个 API 调用
- 摘要质量高（上下文完整）

详细原理和代码示例请参阅 [01_ARCHITECTURE.md](01_ARCHITECTURE.md) 的「上下文管理」章节。

## 分析服务

```typescript
// src/services/analytics/

// 事件日志
export function logEvent(
  event: string,
  properties: Record<string, any>
): void {
  // 发送到分析后端
}

// GrowthBook A/B 测试
export function getFeatureValue<T>(
  feature: string,
  defaultValue: T
): T {
  return getFeatureValue_CACHED_MAY_BE_STALE(feature, defaultValue)
}
```

## 会话记忆

```typescript
// src/services/SessionMemory/

interface SessionMemory {
  importantFacts: string[]
  userPreferences: Record<string, any>
  projectContext: string[]
}

// 提取记忆
export async function extractMemories(
  messages: Message[]
): Promise<SessionMemory> {
  // AI 分析对话，提取重要信息
}

// 存储记忆
export function saveMemory(memory: SessionMemory): void {
  // 持久化到磁盘
}
```

## 工具执行服务

```typescript
// src/services/tools/toolExecution.ts

// 工具执行前后处理
export async function executeTool(
  tool: Tool,
  input: unknown,
  context: ToolUseContext
): Promise<ToolResult> {
  // 1. 权限检查
  // 2. 记录开始
  // 3. 执行
  // 4. 记录结束
  // 5. 结果渲染
}
```

---

## 待深入

- [ ] API 客户端的完整重试逻辑
- [ ] MCP OAuth 实现
- [ ] 压缩算法细节（详见 01_ARCHITECTURE.md）
