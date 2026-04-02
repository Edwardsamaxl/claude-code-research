# 08.2 - BRIDGE & CCR

> 让你在 claude.ai 上看到本地 Claude Code 的操作

## 1. 学习目标

学完本章后，你将理解：
- BRIDGE 的核心机制：本地主动连接 claude.ai 建立反向隧道
- 传输协议：CCR v1 (WebSocket) vs CCR v2 (SSE + HTTP POST)
- 安全机制：OAuth 认证、JWT、信任设备
- 崩溃恢复和自动重连

## 2. 核心内容

### 2.1 是什么

BRIDGE 让你在 claude.ai 网页上看到本地 Claude Code 的实时操作，并可以远程发送命令。

实现方式是：**本地 Claude Code 主动连接到 claude.ai**（反向隧道），而不是 claude.ai 去连接本地机器。这样就不需要暴露端口、不需要 ngrok。

claude.ai → 本地发送命令 → 本地执行 → 结果通过同一隧道返回 → claude.ai 显示

### 2.2 两种运行模式

**standalone**：`claude remote-control`，作为后台 daemon 运行，可以托管多个 session，支持 git worktree 隔离。

**REPL**：`/remote-control`，嵌入本地正在运行的 CLI，本地和远程用户同时可见同一个 session。

### 2.3 传输协议

| | CCR v1 | CCR v2 |
|--|--|--|
| 读取 | WebSocket | SSE |
| 写入 | WebSocket POST | HTTP POST /worker/events |
| 认证 | OAuth | Session Ingress JWT |
| 断线重连 | 无序列号，可能重放 | SSE 序列号，精确续传 |

v2 的核心优势是 SSE 带了序列号，断线重连后不会重复消费消息。

### 2.4 安全机制

- 需要 claude.ai OAuth 登录（不支持 API Key、Claude Code 账号）
- v2 使用 JWT 而非 OAuth token（因为 v2 端点验证 `session_id` claim）
- 可选设备信任：`X-Trusted-Device-Token` header

### 2.5 崩溃恢复

`bridgePointer.json` 记录 session 信息，TTL 4 小时。进程崩溃后重启，如果指针未过期则自动恢复。

### 2.6 自动重连

连接失败时指数退避重试（2s → 4s → ... → 上限 2min，10min 放弃）。系统休眠恢复后，poll 间隔异常（>240s），识别为休眠而非故障，快速重连。

## 3. 关键源文件

| 功能 | 文件 |
|------|------|
| Standalone 主循环 | `src/bridge/bridgeMain.ts` |
| REPL 桥接核心 | `src/bridge/replBridge.ts` |
| 传输抽象（v1/v2） | `src/bridge/replBridgeTransport.ts` |
| REST API 客户端 | `src/bridge/bridgeApi.ts` |
| 崩溃恢复指针 | `src/bridge/bridgePointer.ts` |
| JWT 刷新 | `src/bridge/jwtUtils.ts` |
| poll 配置 | `src/bridge/pollConfig.ts` |

## 4. 本章小结

- BRIDGE 通过"本地主动连接 claude.ai"的反向隧道实现远程观测/控制
- standalone 模式 daemon 托管多 session，REPL 模式嵌入本地 CLI
- CCR v2 用 SSE + HTTP POST + JWT 替代旧版 WebSocket，支持精确断线重连
- 崩溃恢复指针 + 指数退避重试保障连接可靠性

---

## 下一步

← [01_KAIROS.md](01_KAIROS.md)
→ [03_ULTRAPLAN.md](03_ULTRAPLAN.md)
