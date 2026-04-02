# 05 - 命令系统

> 学习 Claude Code 的斜杠命令（`/`）系统：命令类型、注册机制和执行流程

## 1. 学习目标

学完本章后，你将理解：

- 斜杠命令的两种执行方式（直接执行 vs 交给 AI 执行）
- 命令的来源分类（内置命令、技能、插件、MCP）
- Skill 与命令的关系（它们是同构的）
- 命令解析和执行流程
- Feature Flag 控制命令的启用/禁用

## 2. 命令系统概述

斜杠命令是 Claude Code 的用户交互入口。用户输入 `/commandName` 即可触发特定功能。

### 2.1 命令与工具的区别

| 维度 | 斜杠命令 | 工具（Tool） |
|------|----------|--------------|
| 触发方式 | 用户输入 `/xxx` | AI 推理决定调用 |
| 执行者 | 用户主动触发 | AI Agent 调用后执行 |
| 用途 | 配置、导航、系统控制 | 完成任务（读文件、执行命令等） |

**本质区别**：命令是用户主动触发的系统功能，工具是 AI 推理后调用的能力。

### 2.2 命令数量

Claude Code 内置了 **100+ 个斜杠命令**，涵盖基础操作、配置、Git 操作、上下文管理、技能等各个方面。

## 3. 命令的两种执行方式

斜杠命令按执行方式分为两类：

| 执行方式 | 说明 | 对应 type | 示例命令 |
|----------|------|-----------|----------|
| **直接执行** | Claude Code 直接执行，不涉及 AI | `local`, `local-jsx` | `/help`, `/compact`, `/diff`, `/config` |
| **交给 AI 执行** | 命令展开为 prompt，发送给 AI 处理 | `prompt` | `/review`, `/init`, `/statusline` |

### 3.1 直接执行的命令

这类命令由 Claude Code 直接执行，返回结果给用户，不经过 AI 推理。

| type | 说明 | 示例命令 | 功能 |
|------|------|----------|------|
| `local` | 执行后返回文本结果 | `/compact` | 压缩上下文 |
| `local-jsx` | 渲染 React 组件 | `/help`, `/diff`, `/config`, `/model` | 显示帮助、变更、配置面板 |

典型例子：

- `/help` - 显示帮助界面（local-jsx）
- `/compact` - 执行上下文压缩（local）
- `/diff` - 查看未提交的变更（local-jsx）
- `/config` - 显示配置面板（local-jsx）

### 3.2 交给 AI 执行的命令

这类命令会展开为 prompt，发送给 AI 处理。AI 根据 prompt 内容执行相应操作。

| 命令 | type | 功能 |
|------|------|------|
| `/review` | prompt | 审查 Pull Request |
| `/init` | prompt | 初始化项目 |
| `/statusline` | prompt | 设置状态栏 |
| `/insights` | prompt | 分析会话 |

以 `/review` 为例：

| 步骤 | 说明 |
|------|------|
| 1 | 用户输入 `/review` |
| 2 | 展开 prompt："审查这个 Pull Request..." |
| 3 | 发送给 AI |
| 4 | AI 执行 PR 审查操作 |

## 4. 命令的来源

### 4.1 命令分类

命令按来源分为以下几类：

| 来源 | 说明 | 示例 |
|------|------|------|
| 内置命令 | 代码中直接定义的命令 | `/help`, `/compact`, `/config`, `/diff` |
| 技能（Skill） | 从 skills/ 目录加载的 Markdown 文件 | 自定义技能 |
| 插件命令 | 从插件包加载的命令 | 第三方插件提供 |
| MCP 命令 | 从 MCP 服务器工具转换 | MCP 服务器暴露的工具 |

### 4.2 内置命令

内置命令在 `commands.ts` 中直接导入注册，覆盖核心功能。

按功能分类的常用命令：

| 分类 | 命令 |
|------|------|
| 基础操作 | `/help`, `/clear`, `/exit`, `/cancel` |
| 配置管理 | `/config`, `/model`, `/theme`, `/color` |
| 上下文 | `/compact`, `/context`, `/resume`, `/rewind` |
| 会话管理 | `/session`, `/rename`, `/memory` |
| 文件操作 | `/files`, `/diff` |
| MCP | `/mcp` |
| 技能 | `/skills`, `/hooks` |
| 反馈 | `/feedback`, `/doctor` |

### 4.3 技能命令

技能是从 Markdown 文件加载的命令，本质上是 `type: 'prompt'` 的命令。

```yaml
---
name: my-skill
description: Do something useful
when_to_use: When you need to do X
---

# 技能内容
这里是可以被 AI 理解的指令...
```

技能加载路径：

| 路径 | 说明 | loadedFrom |
|------|------|------------|
| `~/.claude/skills/` | 用户级技能 | `skills` |
| `{project}/.claude/skills/` | 项目级技能 | `skills` |
| 插件提供 | 插件打包的技能 | `plugin` |
| MCP 服务器 | MCP 暴露的工具 | `mcp` |
| 内置技能 | 代码内置的技能 | `bundled` |

### 4.4 技能与命令的关系

**技能本质上就是命令**——它们都实现同一个 `Command` 接口：

```typescript
// 技能被转换为 Command 对象
{
  type: 'prompt',
  name: 'my-skill',
  description: 'Do something useful',
  loadedFrom: 'skills',
  async getPromptForCommand(args, context) {
    return [{ type: 'text', text: markdownContent }]
  }
}
```

由于技能和内置命令都是 `Command` 对象，它们被统一注册、查找和执行。

## 5. 命令注册机制

### 5.1 Command 接口

所有命令都实现统一的 `Command` 接口：

```typescript
interface Command {
  // 基本信息
  name: string
  description: string
  aliases?: string[]           // 命令别名

  // 可用性控制
  availability?: CommandAvailability[]  // 'claude-ai' | 'console'
  isEnabled?: () => boolean
  isHidden?: boolean

  // 执行方式
  type: 'local' | 'local-jsx' | 'prompt'

  // 来源追踪
  source?: 'builtin' | 'mcp' | 'plugin' | 'bundled'
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'bundled' | 'mcp'
}
```

### 5.2 启用检查

命令启用状态通过两个维度检查：

```typescript
// 1. 可用性检查（什么用户可以使用）
meetsAvailabilityRequirement(cmd)

// 2. 启用检查（是否开启）
isCommandEnabled(cmd)
// → 执行 cmd.isEnabled() ?? true
```

### 5.3 Feature Flag 控制

某些命令通过 Feature Flag 控制是否包含在构建中：

```typescript
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

这种方式实现**死代码消除**——未启用的功能不会增加最终产物体积。

## 6. 命令执行流程

### 6.1 完整流程

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 解析命令 | `parseSlashCommand()` → `{ commandName, args, isMcp }` |
| 2 | 查找命令 | `findCommand()` → `Command | undefined` |
| 3 | 检查可用性 | `meetsAvailabilityRequirement()` + `isCommandEnabled()` |
| 4a | 执行 local | 直接执行，返回文本 |
| 4b | 执行 local-jsx | 渲染 React 组件 |
| 4c | 执行 prompt | 展开为 prompt，发送给 AI |

### 6.2 命令解析

`parseSlashCommand()` 解析用户输入：

```typescript
parseSlashCommand('/compact')
// → { commandName: 'compact', args: '', isMcp: false }

parseSlashCommand('/mcp:github (MCP) repo clone')
// → { commandName: 'mcp:github (MCP)', args: 'repo clone', isMcp: true }
```

### 6.3 立即命令

某些命令在任何时候都可以执行，不需要等待 AI 响应完成：

```typescript
const IMMEDIATE_COMMANDS = ['help', 'clear', 'exit', 'cancel']
```

## 7. 远程模式安全

当使用 CCR（Claude Code Remote）远程访问功能时，只有特定白名单命令可以执行。

```typescript
// 远程安全命令
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim, cost, usage,
  copy, btw, feedback, plan, keybindings, statusline, stickers, mobile
])
```

## 8. 关键源文件

| 功能 | 文件 |
|------|------|
| 命令注册中心 | `src/commands.ts` |
| Command 类型定义 | `src/types/command.ts` |
| 命令解析 | `src/utils/slashCommandParsing.ts` |
| 命令执行流程 | `src/utils/processUserInput/processSlashCommand.tsx` |
| 技能加载 | `src/skills/loadSkillsDir.ts` |
| 插件命令加载 | `src/utils/plugins/loadPluginCommands.ts` |

## 9. 本章小结

- 命令按执行方式分为两类：**直接执行**（local/local-jsx）和**交给 AI 执行**（prompt）
- `/review`、`/init`、`/statusline` 等命令通过展开 prompt 让 AI 执行相应操作
- 命令来源包括：内置命令、技能（从 Markdown 加载）、插件、MCP 服务器
- **技能本质上就是命令**——它们都实现同一个 `Command` 接口，区别只是来源不同
- 命令执行流程：解析 → 查找 → 检查可用性 → 分发执行
- 通过 Feature Flag 控制命令的构建时包含/排除
- 远程模式（CCR）下只有特定白名单命令可以执行

---

## 下一步

← [04_TOOLS.md](04_TOOLS.md) 学习工具系统
→ [06_UI_LAYER.md](06_UI_LAYER.md) 学习 UI 层
