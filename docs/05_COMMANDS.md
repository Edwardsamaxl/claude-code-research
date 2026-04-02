# 06 - 命令系统

> 斜杠命令 (`/`) 的实现

## 命令注册

```typescript
// src/commands.ts

interface Command {
  name: string
  description: string
  isEnabled: () => boolean

  // 执行函数
  execute: (context: CommandContext) => Promise<void> | void
}

// 注册命令
export function registerCommand(cmd: Command) {
  commands.push(cmd)
}

// 获取所有启用的命令
export function getEnabledCommands(): Command[] {
  return commands.filter(cmd => cmd.isEnabled())
}
```

## 命令执行流程

```
User: /help

       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. 输入解析                                               │
│    - 识别 /help 前缀                                       │
│    - 提取命令名称和参数                                     │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 命令查找                                               │
│    - 在 commands 列表中匹配                                 │
│    - 检查 isEnabled()                                      │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 权限检查                                               │
│    - 检查命令是否需要认证                                   │
│    - 检查是否有权限执行                                     │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. 执行                                                   │
│    - 调用 cmd.execute(context)                             │
│    - 渲染结果                                              │
└─────────────────────────────────────────────────────────────┘
```

## 内置命令

### 基础命令

| 命令 | 文件 | 功能 |
|------|------|------|
| `/help` | `commands/help/` | 显示帮助 |
| `/clear` | `commands/clear/` | 清除对话 |
| `/exit` | `commands/exit/` | 退出 |
| `/cancel` | - | 取消当前操作 |

### 配置命令

| 命令 | 文件 | 功能 |
|------|------|------|
| `/config` | `commands/config/` | 查看/修改配置 |
| `/model` | `commands/model/` | 切换模型 |
| `/permissions` | `commands/permissions/` | 权限管理 |

### Agent 命令

| 命令 | 文件 | 功能 |
|------|------|------|
| `/agent` | - | 使用 Agent 工具 |
| `/task` | - | 使用 Task 工具 |

### Git 命令

| 命令 | 文件 | 功能 |
|------|------|------|
| `/commit` | skills (bundled) | 提交 |
| `/diff` | `commands/diff/` | 查看变更 |
| `/branch` | `commands/branch/` | 分支管理 |

### 其他命令

| 命令 | 文件 | 功能 |
|------|------|------|
| `/compact` | `commands/compact/` | 压缩上下文 |
| `/mcp` | `commands/mcp/` | MCP 管理 |
| `/doctor` | `commands/doctor/` | 诊断问题 |
| `/feedback` | `commands/feedback/` | 发送反馈 |

## 命令实现示例

```typescript
// src/commands/help/index.ts

export default {
  name: 'help',
  description: 'Show help information',

  isEnabled: () => true,  // 始终可用

  async execute(context) {
    const commands = getEnabledCommands()

    // 渲染帮助信息
    const helpText = commands
      .map(cmd => `/${cmd.name} - ${cmd.description}`)
      .join('\n')

    // 通过工具结果返回
    context.print(helpText)
  }
}
```

## 动态命令 (Feature Gated)

```typescript
// src/commands.ts

// 只有启用相应 feature 才能使用的命令
const workflowsCmd = feature('WORKFLOW_SCRIPTS')
  ? require('./commands/workflows/index.js').default
  : null

const forkCmd = feature('FORK_SUBAGENT')
  ? require('./commands/fork/index.js').default
  : null

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

const buddy = feature('BUDDY')
  ? require('./commands/buddy/index.js').default
  : null
```

## 特殊命令类型

### 1. 立即命令 (Immediate Commands)

```typescript
// src/utils/immediateCommand.ts

// 不通过消息发送，立即执行
// 用于 /help, /clear 等基础命令
export function isImmediateCommand(input: string): boolean {
  return ['help', 'clear', 'exit', 'cancel'].includes(input)
}
```

### 2. 持续命令 (Continuous Commands)

```typescript
// 需要保持运行的命令
// 如 /monitor, /tail 等
// 使用 background task 执行
```

### 3. 对话命令

```typescript
// 需要 AI 处理的命令
// 如 /review, /debug
// 通过 AgentTool 执行
```

## 命令别名

```typescript
// src/commands.ts

// 命令别名
const ALIASES: Record<string, string> = {
  '?': 'help',
  'h': 'help',
  'ls': 'files',
  'll': 'files --long',
}

function resolveAlias(input: string): string {
  return ALIASES[input] || input
}
```

## 命令参数解析

```typescript
// 用户输入: /commit "fix: null pointer"
// 解析为:
// command: 'commit'
// args: ['fix: null pointer']

// 或使用解析器
const parser = createCommandParser({
  '/commit': {
    options: [
      { name: 'all', short: 'a', type: 'boolean' },
      { name: 'message', short: 'm', type: 'string', required: true },
    ]
  }
})
```

---

## 待深入

- [ ] 命令的 UI 渲染机制
- [ ] 命令帮助的动态生成
- [ ] 自定义命令注册
