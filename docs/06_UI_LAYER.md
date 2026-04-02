# 06 - UI 层

> Claude Code 使用 React + Ink 在终端中渲染交互式 UI

## 1. 学习目标

学完本章后，你将理解：

- Ink 自定义渲染器的核心实现原理
- UI 组件层次结构和状态管理架构
- 消息渲染系统的设计
- 权限对话框的交互模式
- UI 层的设计考量和性能优化策略

## 2. 名词解释

在深入之前，先理解几个核心概念：

| 名词 | 解释 |
|------|------|
| **Reconciler** | React 的核心算法，负责比较虚拟 DOM 的变化并决定如何更新真实 DOM。Reconciler 就像翻译员，把 React 组件"翻译"成能显示在屏幕上的内容 |
| **Ink** | 一个让你在终端中使用 React 的库。本质上，Ink 实现了一套自己的 Reconciler，能把 React 组件渲染成终端可显示的 ANSI 转义序列 |
| **Yoga** | Facebook 开源的 flexbox 布局引擎。Ink 使用 Yoga 来计算 UI 元素的排列位置 |
| **ANSI 转义序列** | 终端控制码，用于控制光标位置、颜色等。比如 `\x1b[31m` 表示红色文字 |
| **Virtual DOM (虚拟 DOM)** | React 使用的虚拟表示，描述 UI 应该长什么样 |
| **Re-render (重新渲染)** | 当状态变化时，组件重新执行以生成新的 UI |
| **zustand** | 一个轻量级状态管理库，类似 Redux 但更简洁 |
| **useState** | React Hook，用于在组件内部存储会变化的数据 |
| **useAppState** | Claude Code 封装的 Hook，用于读取全局状态 |

## 3. 核心内容

### 3.1 技术栈概览

Claude Code 的 UI 层建立在四个核心技术之上：

| 技术 | 角色 | 用途 |
|------|------|------|
| React | UI 组件框架 | 定义组件"长什么样"，比如一个按钮、一个列表 |
| react-reconciler | 自定义 Reconciler | React 与 Ink 之间的"翻译官" |
| Ink | 终端渲染引擎 | 把 React 组件转成终端能显示的文字和颜色 |
| zustand | 状态管理 | 存储和管理整个应用的数据 |

### 3.2 Ink 渲染架构

Ink 是 Claude Code 自定义渲染器的核心。普通浏览器用 DOM 显示内容，Ink 用终端显示内容。

**核心流程：**

```
用户输入 → React 组件定义 UI 样式
    ↓
虚拟 DOM (React 创建的 UI 描述)
    ↓  Reconciler 转换
Ink DOM 节点 (ink-box, ink-text)
    ↓  Yoga 计算
每个元素的位置和大小
    ↓  转义序列
终端输出 (带颜色、光标控制的文字)
```

**Ink 源码目录结构：**

| 目录/文件 | 说明 |
|-----------|------|
| `src/ink/ink.tsx` | 入口，导出 Box、Text 等 UI 组件 |
| `src/ink/reconciler.ts` | 核心：把 React 组件转换成终端能理解的结构 |
| `src/ink/renderer.ts` | 渲染器主逻辑，负责协调渲染流程 |
| `src/ink/dom.ts` | 管理虚拟 DOM 节点 |
| `src/ink/layout/` | 布局计算，使用 Yoga 引擎 |
| `src/ink/components/` | 内置组件：Box、Text、Button 等 |
| `src/ink/events/` | 键盘事件、鼠标事件处理 |

### 3.3 核心组件

#### Box 组件 - 布局容器

Box 是最基本的布局组件，类似浏览器里的 `<div style="display: flex">`。所有其他组件都放在 Box 里面。

```typescript
// src/ink/components/Box.tsx
function Box({
  children,           // Box 内部包含的其他组件
  flexDirection = 'row',  // 'row'=横向排列，'column'=纵向排列
  flexGrow = 0,       // 是否自动扩展填充可用空间
  flexShrink = 1,     // 空间不足时是否自动缩小
  flexWrap = 'nowrap', // 是否换行
  ...style            // 其他样式：margin、padding、backgroundColor 等
}: Props): React.ReactNode {

  // ink-box 是实际渲染的标签
  // style 对象会被 Yoga 布局引擎处理
  return (
    <ink-box
      style={{
        flexWrap,           // 子元素是否换行
        flexDirection,      // 排列方向
        flexGrow,           // 扩展权重
        flexShrink,         // 缩小权重
        ...style,           // 展开其他样式属性
      }}
    >
      {children}
    </ink-box>
  )
}
```

**使用示例：**

```typescript
// 纵向排列的容器
<Box flexDirection="column">
  <Text>第一行</Text>
  <Text>第二行</Text>
</Box>

// 横向排列的容器
<Box flexDirection="row">
  <Text>左边</Text>
  <Text>右边</Text>
</Box>

// 带间距和背景
<Box
  flexDirection="column"
  padding={1}
  backgroundColor="cyan"
>
  <Text>内容</Text>
</Box>
```

#### Text 组件 - 文字显示

Text 组件用于显示文字，可以设置颜色、粗细等样式。

```typescript
// 支持的样式属性
<Text color="cyan">青色文字</Text>    // 文字颜色
<Text bold>粗体文字</Text>         // 加粗
<Text dimColor>暗淡的文字</Text>    // 降低亮度
<Text underline>下划线</Text>      // 下划线
```

### 3.4 消息渲染系统

消息是用户和 Claude Code 对话的记录。每条消息有不同的类型：用户输入、AI 回复、工具执行结果等。

#### 消息组件层次

```
Messages (总容器)
├── UserPromptMessage        # 用户的输入
├── UserToolResultMessage   # 工具执行结果
│   ├── UserToolSuccessMessage  # 成功
│   └── UserToolErrorMessage    # 失败
├── AssistantTextMessage     # AI 的文字回复
├── AssistantToolUseMessage  # AI 调用了哪些工具
├── AssistantThinkingMessage # AI 的思考过程
├── SystemTextMessage        # 系统提示
└── ...
```

#### 消息渲染示例

```typescript
// src/components/messages/UserPromptMessage.tsx
export function UserPromptMessage({
  addMargin,      // 是否在顶部添加间距
  param: { text } // 用户输入的文本内容
}: Props): React.ReactNode {

  // 订阅"简洁模式"状态 - 查看 Claude Code 是否有省空间模式
  const isBriefOnly = useAppState(s => s.isBriefOnly)

  // 大文本截断：防止大文件导致界面卡顿
  // 如果文本超过 10000 字符，只显示头尾各 2500 字符
  const displayText = useMemo(() => {
    // 文本不长，直接返回
    if (text.length <= MAX_DISPLAY_CHARS) return text

    // 截取头部
    const head = text.slice(0, TRUNCATE_HEAD_CHARS)  // 前 2500 字符
    // 截取尾部
    const tail = text.slice(-TRUNCATE_TAIL_CHARS)    // 后 2500 字符
    // 返回 "头部 ... 隐藏的行数 ... 尾部"
    return `${head}\n… +${hiddenLines} lines …\n${tail}`
  }, [text])

  // 返回渲染结果
  return (
    // 蓝色背景的容器
    <Box backgroundColor="userMessageBackground">
      {/* 带语法高亮的文本 */}
      <HighlightedThinkingText
        text={displayText}
        useBriefLayout={isBriefOnly}
      />
    </Box>
  )
}
```

### 3.5 权限对话框

当 Claude Code 需要用户确认时，会显示权限对话框。比如执行 `rm -rf` 之前会问"确定要删除吗？"

```typescript
// src/components/permissions/PermissionPrompt.tsx
export function PermissionPrompt({
  options,      // 选项数组，比如 [Allow, Deny, Allow with feedback]
  onSelect,     // 用户选择后的回调函数
  onCancel,     // 用户取消时的回调
  question = "Do you want to proceed?",  // 默认问题文本
}: PermissionPromptProps): React.ReactNode {

  // ===== 组件内部状态 =====
  const [acceptFeedback, setAcceptFeedback] = useState("")    // 接受时的反馈输入
  const [rejectFeedback, setRejectFeedback] = useState("")   // 拒绝时的反馈输入
  const [focusedValue, setFocusedValue] = useState(null)     // 当前聚焦的选项

  // ===== 快捷键绑定 =====
  // 把键盘按键（如 y、n）绑定到对应的选项
  useKeybindings(keybindingHandlers, { context: "Confirmation" })

  // ===== 渲染对话框 =====
  return (
    <Box flexDirection="column">
      {/* 问题文本 */}
      <Text>{question}</Text>

      {/* 选项列表 */}
      <Select
        options={selectOptions}       // 选项数据
        onChange={handleSelect}       // 选择时调用
        onCancel={handleCancel}       // 取消时调用
      />

      {/* 底部提示：Esc 取消，Tab 添加反馈 */}
      <Text dimColor>Esc to cancel</Text>
    </Box>
  )
}
```

**权限选项的结构：**

```typescript
// 每个选项包含的值
type PermissionPromptOption<T extends string> = {
  value: T,                    // 选项的唯一标识，如 "allow"
  label: ReactNode,            // 显示文本，如 "Allow"

  // 可选：允许用户输入反馈
  feedbackConfig?: {
    type: 'accept' | 'reject',  // 这是接受还是拒绝的反馈
    placeholder?: string,       // 输入框的占位文字
  },

  // 可选：快捷键
  keybinding?: KeybindingAction  // 如 'y' 代表选择此项
}
```

### 3.6 REPL 主屏幕

REPL (Read-Eval-Print Loop) 是 Claude Code 的主界面，类似命令行提示符。它把各个组件组装在一起。

```typescript
// src/screens/REPL.tsx
export function REPL() {
  return (
    // 纵向排列的根容器，flexGrow=1 表示填充全部可用空间
    <Box flexDirection="column" flexGrow={1}>

      {/* 1. 消息列表 - 显示对话历史 */}
      <MessageList messages={messages} />

      {/* 2. 工具执行进度 - 当有工具在运行时显示 */}
      <AgentProgressLine />

      {/* 3. 权限对话框 - 仅当需要用户确认时显示 */}
      {/* && 表示条件渲染：有 toolPermissionContext 才显示 */}
      {toolPermissionContext && (
        <PermissionDialog {...toolPermissionContext} />
      )}

      {/* 4. 输入框 - 用户输入命令的地方 */}
      <PromptInput
        onSubmit={handleSubmit}     // 提交时调用
        onInterrupt={handleInterrupt} // Ctrl+C 中断时调用
      />

      {/* 5. 状态栏 - 底部显示模型、token 数量等 */}
      <StatusLine
        model={currentModel}       // 当前使用的模型
        tokens={tokenCount}        // 已消耗的 token 数
      />
    </Box>
  )
}
```

**REPL 布局示意：**

```
┌─────────────────────────────────────┐
│                                     │
│         消息列表区域                 │
│    (显示对话历史、工具执行结果)       │
│                                     │
│                                     │
├─────────────────────────────────────┤
│         权限对话框 (必要时显示)       │
├─────────────────────────────────────┤
│  > 用户输入区域                      │
├─────────────────────────────────────┤
│  Model: claude-3-opus  Tokens: 1024 │
└─────────────────────────────────────┘
```

### 3.7 设计考量

#### 为什么选择 React + Ink？

| 方案 | 优点 | 缺点 |
|------|------|------|
| 原生 CLI (curses) | 性能高 | 开发效率低，难以维护 |
| Ink (React for CLI) | **开发效率高**，组件化 | 有运行时开销 |
| Claude Code 选择 | 平衡两者 | - |

**核心权衡：UI 层的设计目标不是启动速度，而是开发效率。**

终端应用的 UI 复杂度远低于浏览器应用，Ink 的性能开销（~10%）不是瓶颈。真正的启动瓶颈在于插件加载（~40%）和 MCP 连接（~20%）。

#### 性能优化策略

**1. React Compiler 自动缓存**

```typescript
// React Compiler 会自动优化组件，避免重复计算
import { c as _c } from "react/compiler-runtime";
```
React Compiler 会分析组件，只在输入变化时才重新渲染。

**2. 选择性状态订阅**

```typescript
// 只读取关心的部分状态，避免无关变化触发重新渲染
const statusLineText = useAppState(s => s.statusLineText)
const tasks = useAppState(s => s.tasks)
```

**3. 大文本截断**

用户可能通过管道输入大文件内容（如 `cat big.log | claude`），这会导致终端卡顿。Claude Code 截断超过 10000 字符的文本。

**4. 混合渲染模式**

- **非全屏模式**：消息直接输出到终端滚动区，不需要完整渲染
- **全屏模式**：使用完整渲染器，支持焦点管理、键盘导航

#### 独特设计

**1. 状态驱动的 UI**

UI 不是用命令控制，而是用状态描述：

```typescript
// UI 由这些状态决定，而不是直接调用 showDialog() / hideDialog()
toolPermissionContext: ToolPermissionContext | null  // null = 不显示对话框
expandedView: 'none' | 'tasks' | 'teammates'       // 显示哪个面板
footerSelection: FooterItem | null                  // 哪个 footer 被选中
```

**2. 统一的交互模式**

所有权限对话框都使用 `PermissionPrompt` 组件，用户体验一致。

**3. 消息类型专用组件**

每种消息都有专门的渲染组件，代码职责清晰，易于维护。

## 4. 关键源文件

| 功能 | 文件 |
|------|------|
| 主屏幕 | `src/screens/REPL.tsx` |
| 状态管理 | `src/state/AppStateStore.ts` |
| 消息渲染 | `src/components/Messages.tsx` |
| Ink 渲染器 | `src/ink/reconciler.ts` |
| Box 组件 | `src/ink/components/Box.tsx` |
| 权限对话框 | `src/components/permissions/PermissionPrompt.tsx` |
| 状态栏 | `src/components/StatusLine.tsx` |
| 输入框 | `src/components/PromptInput/PromptInput.tsx` |

## 5. 本章小结

- Claude Code 使用自定义 Ink 渲染器将 React 组件渲染到终端
- Ink 实现了自己的 Reconciler，把 React 组件转成终端可显示的内容
- 消息系统通过专用组件树渲染不同类型消息
- 权限对话框使用统一的 PermissionPrompt 组件
- UI 层的设计核心是开发效率，而非启动速度
- 性能优化通过 React Compiler 缓存、状态切片订阅、大文本截断等策略实现

---

## 下一步

← [05_COMMANDS.md](05_COMMANDS.md) 返回命令系统
→ [07_SERVICES.md](07_SERVICES.md) 继续学习服务层
