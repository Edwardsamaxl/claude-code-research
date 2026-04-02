# 07 - 服务层

> 后台服务和基础设施，为 Claude Code 的核心功能提供支撑

## 1. 学习目标

学完本章后，你将理解：

- API 服务的重试策略和速率限制处理
- 分析服务如何避免循环依赖
- LSP 服务的异步初始化架构
- 工具执行钩子系统的设计

## 2. 服务概览

```
src/services/
├── api/               # Anthropic API 客户端
├── analytics/         # 事件日志和 A/B 测试
├── compact/          # 上下文压缩
├── contextCollapse/   # 上下文折叠
├── SessionMemory/    # 会话记忆
├── mcp/              # MCP 协议（详见 04_TOOLS.md）
├── lsp/              # 语言服务器协议
├── tools/            # 工具执行服务
├── oauth/            # OAuth 认证
├── autoDream/        # 自动分析服务
├── voice/            # 语音服务
├── tokenEstimation/  # Token 估算
└── ...
```

## 3. API 服务

API 服务封装了与 Claude API 的通信，是 Claude Code 的核心基础设施。

### 3.1 重试机制设计

Claude Code 的重试针对不同错误有不同策略：

| 错误类型 | 策略 | 原理 |
|----------|------|------|
| 429 (限速) | 订阅用户不重试，普通用户重试 | 订阅用户有更高配额 |
| 529 (过载) | 前台请求重试，后台请求放弃 | 避免放大过载 |
| 401 (认证) | 清除缓存后重试 | token 可能过期 |
| 400 (上下文溢出) | 调整 max_tokens 后重试 | 动态适配上下文 |
| 5xx (服务端错误) | 指数退避重试 | 等待恢复 |

**模型降级（529 处理）：**

连续 3 次 529 错误后，自动切换到备用模型。这是一种熔断机制，避免持续请求过载的模型。

### 3.2 速率限制层级

| 层级 | 来源 | 行为 |
|------|------|------|
| API 层面 | `Retry-After` 响应头 | 等待指定时间 |
| 用户层面 | 429 + 订阅状态 | 订阅用户显示提示 |
| 容量层面 | 529 + 查询来源 | 前台重试/后台放弃 |
| 持久模式 | UNATTENDED_RETRY | 后台无限重试 |

## 4. 分析服务

分析服务负责事件追踪和 A/B 测试。

### 4.1 零依赖设计

Claude Code 的分析模块不 import 任何其他模块，避免循环依赖：

```typescript
/**
 * DESIGN: This module has NO dependencies to avoid import cycles.
 * Events are queued until attachAnalyticsSink() is called during app initialization.
 */
export function logEvent(eventName: string, metadata: LogEventMetadata): void {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}
```

### 4.2 事件队列机制

在 sink 附加之前，事件暂存到队列：

```typescript
const eventQueue: QueuedEvent[] = []

// 应用启动时附加 sink，队列异步清空
export function attachAnalyticsSink(newSink: AnalyticsSink): void {
  sink = newSink
  if (eventQueue.length > 0) {
    const queuedEvents = [...eventQueue]
    eventQueue.length = 0
    queueMicrotask(() => {
      for (const event of queuedEvents) {
        sink!.logEvent(event.eventName, event.metadata)
      }
    })
  }
}
```

**设计好处：**
- 启动早期的事件不丢失
- 不阻塞启动流程
- 避免模块间循环依赖

### 4.3 数据安全标记

分析服务使用类型标记确保不泄露敏感数据：

| 标记 | 用途 |
|------|------|
| `I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` | 验证不包含代码/路径 |
| `I_VERIFIED_THIS_IS_PII_TAGGED` | PII 标记字段 |

### 4.4 A/B 测试框架

GrowthBook 是一个开源的 A/B 测试和功能开关框架。在 Claude Code 中用于：

| 用途 | 说明 |
|------|------|
| 功能开关 | 通过配置控制功能开启/关闭，无需发版 |
| A/B 测试 | 对不同用户群体实验新功能 |
| 远程配置 | 不重新部署也能调整参数 |

```typescript
// 获取实验参数值（有缓存，可能过期）
const value = getFeatureValue_CACHED_MAY_BE_STALE('feature_name', defaultValue)
```

## 5. LSP 服务

LSP (Language Server Protocol) 服务提供代码诊断和被动反馈。

### 5.1 为什么需要 LSP？

工具执行只能看到当前调用的结果，但代码可能有语法错误、类型问题、lint 警告等。LSP 服务在后台持续分析，提供这些反馈。

### 5.2 异步初始化

LSP 服务不阻塞应用启动：

```typescript
// manager.ts
type InitializationState = 'not-started' | 'pending' | 'success' | 'failed'

// 初始化状态
let initializationState: InitializationState = 'not-started'

// 获取管理器实例（可能未初始化）
export function getLspServerManager(): LSPServerManager | undefined {
  if (initializationState === 'failed') {
    return undefined  // 失败也不阻塞
  }
  return lspManagerInstance
}
```

**好处：**
- 启动不受 LSP 影响
- LSP 失败不影响核心功能
- 调用方优雅处理未初始化状态

### 5.3 核心组件

| 组件 | 职责 |
|------|------|
| LSPServerManager | 管理服务器生命周期 |
| LSPServerInstance | 封装单个服务器进程 |
| LSPClient | 与服务器通信 |
| LSPDiagnosticRegistry | 存储诊断结果 |
| passiveFeedback | 处理被动反馈 |

### 5.4 核心文件

| 文件 | 职责 |
|------|------|
| `lsp/manager.ts` | 单例管理、异步初始化 |
| `lsp/LSPServerManager.ts` | 服务器实例管理 |
| `lsp/LSPServerInstance.ts` | 单个服务器封装 |
| `lsp/passiveFeedback.ts` | 被动反馈处理 |

## 6. 工具执行钩子

工具执行服务管理工具的完整生命周期。

### 6.1 执行流程

| 步骤 | 说明 |
|------|------|
| 1 | 权限检查 |
| 2 | 执行前钩子 (preToolUse) |
| 3 | 执行工具 |
| 4 | 执行后钩子 (postToolUse) |
| 5 | 结果渲染 |

### 6.2 钩子接口

```typescript
interface ToolHooks {
  preToolUse?: (tool: Tool, input: unknown) => void
  postToolUse?: (tool: Tool, input: unknown, result: ToolResult) => void
}
```

### 6.3 使用场景

| 钩子 | 使用场景 |
|------|----------|
| preToolUse | 记录开始时间、权限检查 |
| postToolUse | 记录执行时长、更新统计、触发压缩检查 |

### 6.4 核心文件

| 文件 | 职责 |
|------|------|
| `tools/toolExecution.ts` | 工具执行流程 |
| `tools/toolHooks.ts` | 钩子接口定义 |

## 7. 其他服务

| 服务 | 目录 | 用途 |
|------|------|------|
| OAuth | `oauth/` | 用户认证和 token 管理 |
| autoDream | `autoDream/` | 后台自动分析任务 |
| SessionMemory | `SessionMemory/` | 持续提取结构化记忆 |
| compact | `compact/` | 工具结果压缩 |
| contextCollapse | `contextCollapse/` | 上下文折叠 |
| MCP | `mcp/` | 外部工具扩展（详见 04_TOOLS.md） |
| tokenEstimation | `tokenEstimation.ts` | Token 数量估算 |
| voice | `voice.ts` | 语音输入支持 |
| MagicDocs | `MagicDocs/` | 文档生成服务 |
| PromptSuggestion | `PromptSuggestion/` | 提示建议 |
| rateLimitMessages | `rateLimitMessages.ts` | 限速提示文案 |
| claudeAiLimits | `claudeAiLimits.ts` | Claude AI 配额管理 |
| settingsSync | `settingsSync/` | 配置同步服务 |
| skillSearch | `skillSearch/` | 技能搜索 |

## 8. 关键源文件

| 服务 | 文件 | 设计亮点 |
|------|------|----------|
| API 重试 | `src/services/api/withRetry.ts` | 指数退避、模型降级、熔断机制 |
| 分析入口 | `src/services/analytics/index.ts` | 零依赖设计、事件队列 |
| GrowthBook | `src/services/analytics/growthbook.ts` | 功能开关配置 |
| LSP 管理 | `src/services/lsp/manager.ts` | 异步初始化 |
| 工具钩子 | `src/services/tools/toolHooks.ts` | 生命周期管理 |

## 9. 本章小结

- **API 服务**通过智能重试策略处理各种故障，指数退避 + 模型降级 + 熔断机制
- **分析服务**采用零依赖设计避免循环依赖，事件队列保证早于 sink 的事件不丢失
- **LSP 服务**异步初始化不阻塞启动，失败不影响核心功能
- **工具执行钩子**提供完整的生命周期管理，支持 pre/post 回调
- **MCP 机制**在 04_TOOLS.md 中详细讲解，核心是模板 + 适配器模式

---

## 下一步

← [06_UI_LAYER.md](06_UI_LAYER.md) 返回 UI 层
→ [08_HIDDEN_FEATURES.md](08_HIDDEN_FEATURES.md) 继续学习隐藏功能
