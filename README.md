# Claude Code 源码学习版

> 深入剖析世界上最先进的 AI Agent 系统的构造与特性

![预览](preview.png)

> [!WARNING]
> **免责声明**：本仓库为**非官方**学习版本，基于公开 npm 包的 source map 还原。源码版权归 [Anthropic](https://www.anthropic.com) 所有，本仓库**仅供研究学习使用**，请勿用于商业用途。如有侵权，请联系删除。

---

## 源码来源

本仓库的学习内容（解释文档）由本人撰写，源码还原自以下仓库：

- **原始还原仓库**：[pengchengneo/Claude-Code](https://github.com/pengchengneo/Claude-Code)
- **npm 包来源**：[@anthropic-ai/claude-code](https://www.npmjs.com/package/@anthropic-ai/claude-code)

> [!NOTE]
> 部分模块因 source map 无法完整还原，已用兼容 shim 或降级实现替代，行为可能与原版有所不同。

## 学习路径

### 必读核心

| 文档 | 内容 |
|------|------|
| [docs/00_OVERVIEW.md](docs/00_OVERVIEW.md) | 学习路线图总纲 |
| [docs/01_ARCHITECTURE.md](docs/01_ARCHITECTURE.md) | 项目架构、设计模式、核心模块 |

### Agent 系统（核心重点）

| 文档 | 内容 |
|------|------|
| [Agent 核心概念](docs/03_AGENT_SYSTEM/01_CORE_CONCEPTS.md) | Agent 定义、类型、生命周期 |
| [Agent 运行机制](docs/03_AGENT_SYSTEM/02_RUN_AGENT.md) | runAgent 执行流程、AsyncGenerator |
| [Agent 来源](docs/03_AGENT_SYSTEM/03_AGENT_ORIGINS.md) | 内置、自定义、Team Agent |
| [任务系统](docs/03_AGENT_SYSTEM/04_TASK_SYSTEM.md) | 七种任务类型、状态机 |
| [多 Agent 协作](docs/03_AGENT_SYSTEM/05_MULTI_AGENT.md) | 并行策略、错误恢复 |
| [上下文压缩](docs/03_AGENT_SYSTEM/06_CONTEXT_COMPRESSION.md) | 三层压缩体系详解 |

### 进阶

| 文档 | 内容 |
|------|------|
| [docs/02_STARTUP.md](docs/02_STARTUP.md) | 启动流程、状态初始化 |
| [docs/04_TOOLS.md](docs/04_TOOLS.md) | 工具系统原理 |
| [docs/05_COMMANDS.md](docs/05_COMMANDS.md) | 斜杠命令系统 |
| [docs/06_UI_LAYER.md](docs/06_UI_LAYER.md) | React + Ink 终端 UI |
| [docs/07_SERVICES.md](docs/07_SERVICES.md) | 服务层（API、MCP、压缩） |

### 隐藏功能探索

| 文档 | 内容 |
|------|------|
| [docs/08_HIDDEN_FEATURES/README.md](docs/08_HIDDEN_FEATURES/README.md) | 实验性功能索引 |
| [KAIROS 模式](docs/08_HIDDEN_FEATURES/01_KAIROS.md) | 助手模式 |
| [Bridge CCR](docs/08_HIDDEN_FEATURES/02_BRIDGE_CCR.md) | 远程桥接控制 |
| [UltraPlan](docs/08_HIDDEN_FEATURES/03_ULTRAPLAN.md) | 超级规划模式 |
| [Buddy 系统](docs/08_HIDDEN_FEATURES/04_BUDDY.md) | 伴侣精灵 |
| [Orchestration](docs/08_HIDDEN_FEATURES/05_ORCHESTRATION.md) | 编排机制 |

### 实践验证

| 文档 | 内容 |
|------|------|
| [docs/09_PRACTICE.md](docs/09_PRACTICE.md) | 练习题和学习验证 |

---

## 快速开始

```bash
# 安装依赖
bun install
# 启动还原后的 CLI
bun run dev
# 输出版本号
bun run version
```

---

## 项目信息

| 项目 | 说明 |
|------|------|
| 源码文件数 | 1,987 个 TypeScript/TSX 文件 |
| UI 框架 | React + Ink（终端 UI） |
| 运行时要求 | Bun ≥ 1.3.5 / Node.js ≥ 24 |

---

## 声明

- 源码版权归 [Anthropic](https://www.anthropic.com) 所有
- 本仓库仅用于技术研究与学习，请勿用于商业用途
- 如有侵权，请联系删除
