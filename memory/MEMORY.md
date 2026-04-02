# 项目记忆

## Claude Code 源码学习项目

### 教程文档格式规范

| 序号 | 模块名 | 路径 |
|------|--------|------|
| 01 | ARCHITECTURE | 01_ARCHITECTURE.md |
| 02 | BOOTSTRAP | 02_BOOTSTRAP.md |
| 03 | AGENT_SYSTEM | 03_AGENT_SYSTEM/README.md |
| 04 | TOOLS | 04_TOOLS.md |
| 05 | COMMANDS | 05_COMMANDS.md |
| 06 | UI_LAYER | 06_UI_LAYER.md |
| 07 | SERVICES | 07_SERVICES.md |
| 08 | HIDDEN_FEATURES | 08_HIDDEN_FEATURES.md |
| 09 | PRACTICE | 09_PRACTICE.md |

### 排版规范

1. **不用 ASCII box drawing**（`┌ ─ ┐`），用 Markdown 表格
2. **标题用数字编号**（1. 2. 3. 或 1.1 1.2）
3. **代码用 ``` 包裹**，注释用 //
4. **表格用 `| col | col |`** 格式

### 文档结构模板

```markdown
# XX - 模块名称

> 简短描述

## 1. 学习目标

学完本章后，你将理解：
- 要点1
- 要点2

## 2. 核心内容

### 2.1 小节标题

内容...

## 3. 关键源文件

| 功能 | 文件 |
|------|------|
| xxx | path |

## 4. 本章小结

- 总结1
- 总结2

---

## 下一步

→ [00_OVERVIEW.md](../00_OVERVIEW.md) 查看学习路线图
```

### 已完成修改

- 01_ARCHITECTURE.md: 改版完成，移除 ASCII art，使用标准表格
- 00_OVERVIEW.md: 序号修正，添加学习路径导航
- 08_SERVICES.md: 简化压缩服务描述，指向 01_ARCHITECTURE.md 详阅
- 03_BUILTIN_AGENTS.md → 03_AGENT_ORIGINS.md: 重命名并重构内容，涵盖内置/自定义/Team Agent

### 待完成

- [ ] 重命名 05_TOOLS.md → 04_TOOLS.md
- [ ] 重命名 06_COMMANDS.md → 05_COMMANDS.md
- [ ] 重命名 07_UI_LAYER.md → 06_UI_LAYER.md
- [ ] 重命名 08_SERVICES.md → 07_SERVICES.md
- [ ] 重命名 09_HIDDEN_FEATURES.md → 08_HIDDEN_FEATURES.md
- [ ] 重命名 PRACTICE.md → 09_PRACTICE.md
