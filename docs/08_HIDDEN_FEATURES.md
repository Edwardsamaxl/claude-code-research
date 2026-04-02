# 09 - 隐藏功能 (Feature Flags)

> 这些功能通过 `feature('XXX')` 或环境变量控制，大部分是实验性功能

## Feature Flag 机制

```typescript
// 所有 flag 来自 bun:bundle
import { feature } from 'bun:bundle'

// 检查 flag
if (feature('KAIROS')) {
  // KAIROS 相关代码
}

// 环境变量覆盖
// CLAUDE_CODE_XXX=1 可以启用某些功能
```

## 功能列表

### Agent 编排类

| Flag | 环境变量 | 功能 |
|------|----------|------|
| `COORDINATOR_MODE` | `CLAUDE_CODE_COORDINATOR_MODE=1` | 多 Worker 编排模式 |
| `FORK_SUBAGENT` | - | Fork 上下文继承 |
| `AGENT_TEAMS` | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | 多 Agent 团队 |
| `AGENT_TRIGGERS` | - | 定时任务调度 |
| `AGENT_MEMORY_SNAPSHOT` | - | Agent 记忆快照 |

### UI/交互类

| Flag | 环境变量 | 功能 |
|------|----------|------|
| `KAIROS` | - | 下一代 UI 框架 |
| `KAIROS_BRIEF` | - | Brief 模式 |
| `KAIROS_CHANNELS` | - | 通道功能 |
| `KAIROS_DREAM` | - | 梦境模式 |
| `KAIROS_GITHUB_WEBHOOKS` | - | GitHub Webhook |
| `PROACTIVE` | `CLAUDE_CODE_PROACTIVE=1` | 主动模式 |
| `BUDDY` | - | 宠物伴侣系统 |
| `VOICE_MODE` | - | 语音交互 |
| `ULTRAPLAN` | - | 高级计划模式 |

### 工具类

| Flag | 环境变量 | 功能 |
|------|----------|------|
| `WEB_BROWSER_TOOL` | - | 内置浏览器工具 |
| `WORKFLOW_SCRIPTS` | - | 工作流脚本 |
| `MONITOR_TOOL` | - | MCP 监控任务 |
| `EXPERIMENTAL_SKILL_SEARCH` | - | 改进的技能搜索 |
| `MCP_SKILLS` | - | MCP 技能 |

### 权限/分类类

| Flag | 环境变量 | 功能 |
|------|----------|------|
| `BASH_CLASSIFIER` | - | Bash 命令 AI 分类 |
| `TRANSCRIPT_CLASSIFIER` | - | 转录分类 |
| `AUTO_MODE` | - | 自动模式 |

### 远程/连接类

| Flag | 环境变量 | 功能 |
|------|----------|------|
| `BRIDGE_MODE` | - | 远程控制模式 |
| `CCR_AUTO_CONNECT` | - | CCR 自动连接 |
| `CCR_MIRROR` | - | CCR 镜像 |
| `DIRECT_CONNECT` | - | 直接连接 |
| `SSH_REMOTE` | - | SSH 远程 |
| `UDS_INBOX` | - | Unix Domain Socket |

### 存储/会话类

| Flag | 环境变量 | 功能 |
|------|----------|------|
| `BG_SESSIONS` | - | 后台会话管理 |
| `TEAMMEM` | - | 团队记忆共享 |
| `FILE_PERSISTENCE` | - | 文件持久化 |
| `SESSION_MEMORY` | - | 会话记忆 |

### 输出/显示类

| Flag | 环境变量 | 功能 |
|------|----------|------|
| `STREAMLINED_OUTPUT` | - | 简化输出 |
| `MESSAGE_ACTIONS` | - | 消息操作菜单 |
| `HISTORY_PICKER` | - | 历史记录选择器 |
| `QUICK_SEARCH` | - | 快速搜索 |
| `TERMINAL_PANEL` | - | 终端面板 |
| `AWAY_SUMMARY` | - | 离开摘要 |

### 压缩/优化类

| Flag | 环境变量 | 功能 |
|------|----------|------|
| `CONTEXT_COLLAPSE` | - | 智能上下文压缩 |
| `REACTIVE_COMPACT` | - | 反应式压缩 |
| `CACHED_MICROCOMPACT` | - | 缓存微压缩 |
| `TOKEN_BUDGET` | - | Token 预算管理 |
| `COMPACTION_REMINDERS` | - | 压缩提醒 |

### 其他

| Flag | 环境变量 | 功能 |
|------|----------|------|
| `COMMIT_ATTRIBUTION` | - | 提交归属 |
| `DOWNLOAD_USER_SETTINGS` | - | 下载用户设置 |
| `UPLOAD_USER_SETTINGS` | - | 上传用户设置 |
| `CONNECTOR_TEXT` | - | 连接器文本 |
| `SHOT_STATS` | - | 统计信息 |
| `AUTO_THEME` | - | 自动主题 |
| `BYOC_ENVIRONMENT_RUNNER` | - | BYOC 环境运行器 |
| `SELF_HOSTED_RUNNER` | - | 自托管运行器 |

## 环境变量速查

```bash
# Agent 相关
CLAUDE_CODE_COORDINATOR_MODE=1          # 启用 Coordinator Mode
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1  # 启用 Agent Teams
CLAUDE_CODE_SIMPLE=1                   # 简化 worker 工具
CLAUDE_CODE_PROACTIVE=1                 # 启用主动模式
CLAUDE_CODE_ASSISTANT_MODE=1            # 启用助手模式

# 输出相关
CLAUDE_CODE_BRIEF=1                    # Brief 模式
CLAUDE_CODE_STREAMLINED_OUTPUT=1       # 简化输出

# 远程相关
CLAUDE_CODE_REMOTE=1                    # 远程模式
CLAUDE_CODE_USE_CCR_V2=1               # 使用 CCR v2

# 调试
CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1  # 禁用后台任务
CLAUDE_CODE_DISABLE_FAST_MODE=1        # 禁用快速模式

# 其他
CLAUDE_CODE_DISABLE_COMPACT=1          # 禁用压缩
CLAUDE_CODE_DISABLE_TERMINAL_TITLE=1    # 禁用终端标题
```

## 实验性功能详解

### BUDDY - 宠物系统

```typescript
// src/buddy/types.ts

// 宠物属性
type Companion = {
  rarity: 'common' | 'uncommon' | 'rare' | 'epic' | 'legendary'
  species: 'duck' | 'goose' | 'blob' | 'cat' | 'dragon' | ...
  eye: '·' | '✦' | '×' | '◉' | '@' | '°'
  hat: 'none' | 'crown' | 'tophat' | 'propeller' | ...
  shiny: boolean
  stats: {
    DEBUGGING: number
    PATIENCE: number
    CHAOS: number
    WISDOM: number
    SNARK: number
  }
}
```

### PROACTIVE - 主动模式

```typescript
// src/proactive/

// AI 主动推断用户意图
// 在用户明确请求之前采取行动
```

### ULTRAPLAN - 高级计划

```typescript
// src/commands/ultraplan.tsx

// 更强大的计划能力
// 结构化规划输出
```

---

## 启用实验性功能的建议

```bash
# 方式 1: 环境变量
CLAUDE_CODE_COORDINATOR_MODE=1 bun run dev

# 方式 2: 编译时启用 (需要修改代码)
# 在 bun:bundle 配置中添加
```

---

## 待深入

- [ ] 各 feature 的 GrowthBook A/B 测试配置
- [ ] feature flag 的安全考虑
- [ ] 如何安全地添加新的 feature flag
