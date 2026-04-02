# 08.1 - KAIROS

> 从被动问答到主动工作：Channels 通道、梦境系统、Cron 调度

## 1. 学习目标

学完本章后，你将理解：
- KAIROS 的核心使命：让 Claude 从被动响应变成主动工作
- Channels：MCP 服务器主动向会话推送消息
- Dream：会话空闲时自动整理记忆
- Cron：定时任务让 Claude 主动执行操作

## 2. 核心内容

### 2.1 KAIROS 是什么

**KAIROS** 是一套让 Claude 从"被动回答问题"转变为"主动规划、执行、记忆"的框架。

| 对比 | Legacy Claude Code | KAIROS |
|------|-------------------|--------|
| 交互方式 | 用户问 → Claude 答 | Claude 可主动推送消息 |
| 记忆整理 | 手动 | 会话空闲时自动后台整理 |
| 任务执行 | 等待用户指令 | 定时自动执行 |
| 输出方式 | 任意文本 | 必须通过 SendUserMessage 工具 |

**KAIROS 通过四个核心机制实现"主动"：**

| 机制 | 说明 | 特点 |
|------|------|------|
| **Channels** | MCP 服务器/插件主动向会话推送消息 | 外部推入，Claude 接收 |
| **PROACTIVE** | Claude 主动向用户发送建议或提醒 | Claude 主动发出 |
| **Dream** | 会话空闲时自动整理记忆 | 后台运行 |
| **Cron** | 定时任务让 Claude 自动执行操作 | 定时触发，后台运行 |

**PROACTIVE 状态机**：active / paused / contextBlocked，通过 `subscribeToProactiveChanges` 订阅状态变化。

### 2.2 Channels — 通道机制

**Channels** 允许 MCP 服务器和插件主动向当前会话推送消息，Claude 不再只能被动等待用户输入。

#### 工作原理

```
传统模式:  用户 → Claude Code → MCP Server（单向请求-响应）
Channel:   MCP Server/Plugin → 入站消息 → Claude 会话（双向）
```

#### 通道类型

| 类型 | 说明 |
|------|------|
| `plugin` | 通过 marketplace 分发的插件通道 |
| `server` | MCP 服务器通道 |

#### 状态与提示

| 状态 | 显示信息 |
|------|----------|
| disabled | `--channels ignored` |
| noAuth | 需要运行 `/login` |
| policyBlocked | 组织策略阻止 |
| listening | 正常监听通道消息 |

> **安全警告**：入站消息会被推入当前会话，存在 prompt injection 风险。

### 2.3 Dream — 梦境系统

**Dream** 在会话空闲时自动运行子 agent 整理记忆文件，让下次会话能快速 orient。

#### 触发条件

| 条件 | 值 |
|------|-----|
| 时间门槛 | 距上次整合 ≥24 小时 |
| 会话门槛 | ≥5 个会话有更新 |
| 锁 | 无其他进程正在整合 |

#### 四阶段工作流

| 阶段 | 做什么 | 具体操作 |
|------|--------|----------|
| Orient | 定位现有结构 | 读 memory 目录、`MEMORY.md` 入口、已有 topic 文件 |
| Gather | 收集信号 | 读日志、找矛盾事实、grep 会话 JSONL |
| Consolidate | 写入记忆 | 合并/新建/删除、改相对日期→绝对日期 |
| Prune | 清理索引 | 删矛盾、更新入口文件（≤25KB） |

#### KAIROS vs 标准模式

| 对比 | 标准模式 | KAIROS 模式 |
|------|----------|-------------|
| 实现 | Forked Subagent | Disk-Skill |
| 记忆布局 | `memory/*.md` | `logs/YYYY/MM/YYYY-MM-DD.md` |

### 2.4 Cron — 定时调度

**Cron** 让 Claude 能在指定时间自动执行任务，不只是被动回答问题。

#### 任务类型

| 类型 | 说明 | 持久化 |
|------|------|--------|
| One-shot | 单次执行后删除 | 可选磁盘 |
| Recurring | 周期性执行 | 写入 `scheduled_tasks.json` |
| Permanent | 永久任务（不自动过期） | 内置任务专属 |
| Session-only | 仅当前会话有效 | 内存中 |

#### 内置永久任务

| 任务 | 用途 |
|------|------|
| dream | 定时记忆整合 |
| catch-up | 会话恢复后补发 |
| morning-checkin | 每日签到 |

#### 使用限制

- 仅在 REPL 空闲时触发
- 定时任务会在 5 分钟内刷新 GrowthBook 开关
- 循环任务自动过期前会发送通知

## 3. 关键源文件

| 功能 | 文件 |
|------|------|
| 全局状态 | `src/bootstrap/state.ts` |
| PROACTIVE 状态机 | `src/proactive/index.ts` |
| Channels 通知 | `src/components/LogoV2/ChannelsNotice.tsx` |
| Dream 引擎 | `src/services/autoDream/autoDream.ts` |
| Dream 四阶段提示 | `src/services/autoDream/consolidationPrompt.ts` |
| Dream 任务定义 | `src/tasks/DreamTask/DreamTask.ts` |
| Cron 调度器 | `src/utils/cronScheduler.ts` |
| Cron 任务持久化 | `src/utils/cronTasks.ts` |

## 4. 本章小结

- **KAIROS 的核心目标**：让 Claude 从被动响应变成主动工作
- **Channels**：MCP 可主动推送消息，打破单向请求-响应模式
- **PROACTIVE**：Claude 主动向用户发送建议，三态状态机控制开关
- **Dream**：会话空闲时自动整理记忆，下次启动更快 orient
- **Cron**：定时任务让 Claude 能主动执行操作（不只是回答问题）

---

## 下一步

← [README](README.md) 返回隐藏功能总览
→ [02_BRIDGE_CCR.md](02_BRIDGE_CCR.md) 继续探索远程控制全家桶
