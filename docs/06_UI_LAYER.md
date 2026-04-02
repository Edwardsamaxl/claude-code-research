# 07 - UI 层 (React + Ink)

> Claude Code 使用 React + Ink 在终端中渲染 UI

## 技术栈

| 技术 | 用途 |
|------|------|
| React | UI 组件框架 |
| Ink | 终端渲染 (React for CLI) |
| React Reconciler | 自定义渲染器 |
| zustand | 状态管理 |

## 核心文件

```
src/
├── screens/
│   └── REPL.tsx          # 主屏幕
│
├── components/
│   ├── PromptInput/     # 输入框
│   ├── messages/        # 消息渲染
│   ├── diff/           # Diff 视图
│   ├── permissions/    # 权限对话框
│   └── ...
│
├── hooks/               # React Hooks
│   └── use*.ts
│
├── ink/                # 自定义 Ink 组件
│   ├── components/
│   ├── layout/
│   └── hooks/
```

## REPL 屏幕

```typescript
// src/screens/REPL.tsx

export function REPL() {
  return (
    <ink.Text>
      {/* 消息列表 */}
      <MessageList messages={messages} />

      {/* 输入框 */}
      <PromptInput
        onSubmit={handleSubmit}
        onInterrupt={handleInterrupt}
      />

      {/* 状态栏 */}
      <StatusLine
        model={currentModel}
        tokens={tokenCount}
        context={contextLength}
      />
    </ink.Text>
  )
}
```

## 组件层次

```
App
├── REPL (主屏幕)
│   ├── Header (Logo, 版本)
│   ├── MessageList
│   │   ├── Message (用户消息)
│   │   ├── Message (AI 响应)
│   │   │   ├── ToolUse (工具调用)
│   │   │   ├── Thinking (思考过程)
│   │   │   └── Text (文本输出)
│   │   └── ...
│   ├── BackgroundTasksIndicator
│   ├── PermissionDialog
│   ├── PromptInput
│   │   ├── Prefix (模型指示)
│   │   ├── Input (文本输入)
│   │   └── Suggestions (建议)
│   └── StatusLine
```

## Ink 自定义组件

```typescript
// src/ink/components/

// 自定义 Box 支持更多样式
export function Box(props) {
  return <ink.Box {...props} />
}

// 自定义 Text 支持颜色
export function Text(props) {
  return <ink.Text {...props} />
}
```

## 状态管理

```typescript
// src/state/AppStateStore.ts (zustand)

export const useAppStore = create<AppState>((set, get) => ({
  // 状态
  messages: [],
  tasks: {},
  toolPermissionContext: { mode: 'auto' },

  // 操作
  addMessage: (msg) => set(state => ({
    messages: [...state.messages, msg]
  })),

  setToolPermission: (mode) => set(state => ({
    toolPermissionContext: { ...state.toolPermissionContext, mode }
  })),
}))
```

## 消息渲染

```typescript
// src/components/messages/

interface Message {
  id: string
  type: 'user' | 'assistant' | 'system'
  content: ContentBlock[]
}

interface ContentBlock {
  type: 'text' | 'tool_use' | 'tool_result' | 'thinking'
}
```

## 特殊 UI

### Diff 视图

```typescript
// src/components/diff/

// 渲染 git diff
export function DiffView({ diff }) {
  return (
    <Box>
      {diff.map(part => (
        <DiffLine
          type={part.type}  // 'add' | 'remove' | 'context'
          content={part.content}
        />
      ))}
    </Box>
  )
}
```

### 权限对话框

```typescript
// src/components/permissions/

// 显示权限请求
export function PermissionDialog({ tool, input }) {
  return (
    <Box borderStyle="round">
      <Text bold>Allow {tool}?</Text>
      <Text>{JSON.stringify(input, null, 2)}</Text>
      <Text dimColor>[y/n/a]</Text>
    </Box>
  )
}
```

### 进度指示

```typescript
// src/components/Spinner.tsx

// 工具执行中的进度指示
export function Spinner({ label }) {
  return (
    <Box>
      <Text color="cyan">⠋</Text>
      <Text>{label}</Text>
    </Box>
  )
}
```

## 事件处理

```typescript
// src/hooks/useGlobalKeybindings.tsx

// 全局快捷键处理
useEffect(() => {
  // Ctrl+C - 中断
  // Ctrl+Z - 挂起
  // Ctrl+L - 清屏
  // Tab - 自动补全
}, [])
```

## 响应式设计

```typescript
// Ink 使用固定宽度，默认为终端宽度
// 但支持动态宽度

<Box width={process.stdout.columns}>
  <Text>响应式内容</Text>
</Box>
```

---

## 待深入

- [ ] Ink 自定义渲染器的实现
- [ ] 消息的流式渲染
- [ ] 权限对话框的状态管理
