# 08.3 - ULTRAPLAN

> 本地不想跑规划？放到云端去跑。CCR 是"云端 Claude Code"，ULTRAPLAN 让 CCR 帮你做规划，结果可以本地执行或继续在云端执行。

## 1. 学习目标

学完本章后，你将理解：
- ULTRAPLAN 的使用方式
- 本地 CLI 与 CCR 云端会话的通讯机制
- 安全机制（permission mode、OAuth）
- 状态流程（发起 → 规划 → 审批 → 执行）

## 2. 核心内容

### 2.1 是什么

ULTRAPLAN = **Ultra + Plan**，把规划任务交给 CCR（Claude Code on the Web，云端容器里的 Claude Code）来跑。

**为什么不用本地跑**：
- 本地模型不够强（CCR 用 Opus 4.6）
- 复杂规划耗时太长，终端被阻塞
- 规划期间想继续其他工作

**三种结果**：
| 用户选择 | 计划去向 |
|---------|---------|
| Approve | 在 CCR 远程执行，结果以 PR 返回 |
| Teleport back | 计划发回本地，本地执行 |
| 拒绝/超时 | 继续迭代或关闭会话 |

### 2.2 怎么用

**方式一：Slash 命令**
```
/ultraplan 帮我规划一个微服务重构方案
```

**方式二：提示中包含 ultraplan 关键词**
```
请 ultraplan 一下这个重构方案
```
（会自动拦截并路由到 /ultraplan，ultraplan 替换为 plan 保持语法正确）

**前置条件**：必须已登录（`/login`）

### 2.3 本地与 CCR 的通讯

本地每 3 秒调用 `pollRemoteSessionEvents()` 拉取 CCR 事件流。事件类型包括 `assistant`、`user`、`result`。本地通过 `ExitPlanModeScanner` 扫描事件流判断用户是否已审批。

**超时**：30 分钟无响应则终止。
- 本地每 3 秒调用一次 `pollRemoteSessionEvents()` 获取 CCR 事件流
- 事件类型包括：`assistant`、`user`、`result`
- 本地的 `ExitPlanModeScanner` 扫描事件流，判断用户是否已审批

**超时**：30 分钟无响应则终止

### 2.4 安全机制

**1. Permission Mode = 'plan'**
创建 CCR 会话时指定 `permissionMode: 'plan'`，这会让 CCR 容器启动时就处于规划模式，ExitPlanModeTool 可用，用户可以审批计划。

**2. OAuth 认证**
创建会话需要 OAuth token，poll 过程中 token 可能过期（30 分钟超时的问题）。

**3. 前置条件检查**
`checkRemoteAgentEligibility()` 检查登录状态，不满足则拒绝发起。

### 2.5 状态流程

1. **发起**：用户输入 `/ultraplan` 或提示中含 `ultraplan`
2. **创建会话**：`teleportToRemote` 创建 CCR 会话（permissionMode: 'plan'，模型: Opus 4.6）
3. **Poll 循环**：本地每 3 秒拉取 CCR 事件，CCR 在云端跑规划，用户可在网页端交互
4. **用户决策**：Approve（CCR 执行，结果以 PR 返回）/ Teleport（计划发回本地）/ 拒绝/超时

## 3. 关键源文件

| 功能 | 文件 |
|------|------|
| 命令入口 | `src/commands/ultraplan.tsx` |
| Poll 与 Scanner | `src/utils/ultraplan/ccrSession.ts` |
| 关键词检测 | `src/utils/ultraplan/keyword.ts` |
| 会话创建 | `src/utils/teleport.tsx` |
| Remote Task 状态 | `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` |

## 4. 本章小结

- ULTRAPLAN 把规划任务放到 CCR 云端，本地终端保持可用
- 触发方式：`/ultraplan` 或提示中含 `ultraplan` 关键词
- 两者通过 HTTP Poll 机制通讯，本地每 3 秒拉取 CCR 事件流
- 用户可在 CCR 网页端审批或让计划 Teleport 回本地执行
- 安全基于 permission mode 和 OAuth 认证

---

## 下一步

← [02_BRIDGE_CCR.md](02_BRIDGE_CCR.md)
→ [04_BUDDY.md](04_BUDDY.md)
