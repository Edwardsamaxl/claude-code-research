# Claude Code 源码学习路线图

> 本文档是学习 Claude Code 源码的总纲，每个模块的具体内容在对应文档中。

## 项目信息

| 项目 | 说明 |
|------|------|
| 名称 | Claude Code (还原版) |
| 源码 | 1,987 个 TypeScript/TSX 文件 |
| 运行时 | Bun ≥ 1.3.5 / Node.js ≥ 24 |
| UI框架 | React + Ink (终端 UI) |
| 来源 | npm 包 source map 还原 |

## 学习目标

1. **理解架构设计** - 了解 Claude Code 如何组织如此复杂的代码库
2. **掌握 Agent 编排** - 多 Agent 协作、任务分派、上下文管理
3. **熟悉工具系统** - 50+ 工具的实现原理
4. **理解 UI 层** - React + Ink 的终端渲染
5. **探索隐藏功能** - 实验性功能的用途和启用方式

## 学习路径

### 第一阶段：奠基

| 序号 | 模块 | 内容 | 优先级 |
|------|------|------|--------|
| 01 | [ARCHITECTURE](01_ARCHITECTURE.md) | 项目结构、核心组件、设计模式 | 必读 |
| 02 | [STARTUP](02_STARTUP.md) | 启动流程、状态初始化 | 推荐 |

### 第二阶段：核心

| 序号 | 模块 | 内容 | 优先级 |
|------|------|------|--------|
| 03 | [AGENT_SYSTEM](03_AGENT_SYSTEM/) | Agent 编排系统 | 核心重点 |
|  | └─ [01_CORE_CONCEPTS](03_AGENT_SYSTEM/01_CORE_CONCEPTS.md) | Agent 核心概念 | |
|  | └─ [02_RUN_AGENT](03_AGENT_SYSTEM/02_RUN_AGENT.md) | Agent 运行机制 | |
|  | └─ [03_AGENT_ORIGINS](03_AGENT_SYSTEM/03_AGENT_ORIGINS.md) | Agent 的来源与构造 | |
|  | └─ [04_TASK_SYSTEM](03_AGENT_SYSTEM/04_TASK_SYSTEM.md) | 任务类型与生命周期 | |
|  | └─ [05_MULTI_AGENT](03_AGENT_SYSTEM/05_MULTI_AGENT.md) | 多 Agent 协作 | |
|  | └─ [06_CONTEXT_COMPRESSION](03_AGENT_SYSTEM/06_CONTEXT_COMPRESSION.md) | 上下文压缩机制 | |
| 04 | [TOOLS](04_TOOLS.md) | 工具系统原理 | 重要 |

### 第三阶段：进阶

| 序号 | 模块 | 内容 | 优先级 |
|------|------|------|--------|
| 05 | [COMMANDS](05_COMMANDS.md) | 斜杠命令系统 | 了解 |
| 06 | [UI_LAYER](06_UI_LAYER.md) | React + Ink 终端 UI | 了解 |
| 07 | [SERVICES](07_SERVICES.md) | 服务层（API、MCP、压缩） | 了解 |
| 08 | [HIDDEN_FEATURES](08_HIDDEN_FEATURES/) | 实验性功能详解 | 探索 |

### 第四阶段：实践

| 序号 | 模块 | 内容 |
|------|------|------|
| 09 | [PRACTICE](09_PRACTICE.md) | 练习题和学习验证 |

## 下一步

→ [01_ARCHITECTURE.md](01_ARCHITECTURE.md) 开始学习架构设计
