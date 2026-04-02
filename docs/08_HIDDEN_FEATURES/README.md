# 08 - 隐藏功能

> Claude Code 官方版未提供或默认关闭的实验性功能，通过 Feature Flag 控制

## 1. 模块定位

本模块属于学习路线图**第四阶段 · 探索**，面向已掌握核心架构的学习者。

| 前置模块 | 学习目的 |
|---------|---------|
| [01_ARCHITECTURE](../01_ARCHITECTURE.md) | 理解项目结构和组件关系 |
| [03_AGENT_SYSTEM](../03_AGENT_SYSTEM/) | 掌握 Agent 编排基础 |

## 2. Feature Flag 机制

所有实验性功能通过 `feature('XXX')` 启用，定义在 `bun:bundle` 中：

```typescript
import { feature } from 'bun:bundle'

if (feature('KAIROS')) {
  // KAIROS 相关代码
}
```

### 2.1 环境变量覆盖

部分功能支持环境变量直接启用：

```bash
CLAUDE_CODE_COORDINATOR_MODE=1  # 启用 Coordinator Mode（多代理编排）
CLAUDE_CODE_PROACTIVE=1         # 启用主动模式
```

### 2.2 绕过 Feature Gate

由于 `feature('XXX')` 需要 Statsig Feature Gate 授权，可通过修改源码绕过检查。

以 Coordinator Mode 为例：

**文件：** `src/coordinator/coordinatorMode.ts`

**修改：** 将 `isCoordinatorMode()` 函数改为：

```typescript
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE') || true) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

**启动命令：**

```powershell
cd E:\CursorProject\claude-code-research; $env:CLAUDE_CODE_COORDINATOR_MODE=1; bun run dev
```

其他隐藏功能可依此类推，修改对应文件中的 `feature('XXX')` 检查即可。

## 3. 内容总览

| 序号 | 文档 | 核心内容 |
|------|------|---------|
| 01 | [KAIROS](01_KAIROS.md) | 下一代渲染引擎、Channels 机制、主动模式、Dream、Cron |
| 02 | [BRIDGE_CCR](02_BRIDGE_CCR.md) | 远程控制全家桶、自动连接、会话镜像 |
| 03 | [ULTRAPLAN](03_ULTRAPLAN.md) | 云端规划执行、Poll 机制、超时处理 |
| 04 | [BUDDY](04_BUDDY.md) | 宠物系统：物种生成、稀有度、ASCII 精灵动画 |
| 05 | [ORCHESTRATION](05_ORCHESTRATION.md) | COORDINATOR 与 FORK 两种编排范式对比 |

## 4. 已知问题

### 4.1 Coordinator 模式下的 Agent Tool

即使通过上述方法启用了 Coordinator 模式，调用 `Agent` 工具时使用 `subagent_type: "worker"` 可能会报错：

```
Agent type 'worker' not found. Available agents: general-purpose, ...
```

这是因为 `worker` 是一个独立的 subagent type，需要在 AgentTool 的配置中注册才能使用。

**说明：** `isCoordinatorMode()` 控制的是协调者**系统提示词**（见 `getCoordinatorSystemPrompt()`），而 `subagent_type: "worker"` 是 Agent 工具的**子类型**配置。两者是独立机制——Coordinator 模式提供协调策略，但 worker 子类型的注册属于工具层面配置。

如果需要使用多 agent 协作功能，可以改用 `general-purpose` 作为 `subagent_type`，效果类似但不具备 worker 的专属优化。

## 下一步

→ [01_KAIROS.md](01_KAIROS.md) 开始探索隐藏功能
