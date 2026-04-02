# 02 - 启动流程

> 理解 Claude Code 如何初始化：从命令行入口到交互式 REPL 的完整流程

## 1. 学习目标

学完本章后，你将理解：

- Claude Code 的三层启动入口设计（dev-entry → cli → main）
- 快速路径机制：如何用零模块加载实现 `--version` 等快捷命令
- 完整路径的初始化流程：配置加载、认证、MCP、状态创建
- Feature Flag 在启动时的作用

## 2. 核心内容

### 2.1 三层启动入口

Claude Code 采用三层入口设计，每层职责分明：

```
dev-entry.ts      # 开发/生产启动器，检测缺失模块
    ↓
cli.tsx           # 命令行路由，快捷路径分发
    ↓
main.tsx          # 完整初始化，渲染 REPL
```

#### dev-entry.ts — 开发启动器

负责启动前的完整性检查：

```typescript
// src/dev-entry.ts

// 检测缺失的相对导入
const missingImports = collectMissingRelativeImports()

// 快速路径：--version 无需加载任何模块
if (args.includes('--version')) {
  console.log(pkg.version)
  process.exit(0)
}

// 如果有缺失模块，显示警告但不阻止启动
if (missingImports.length > 0) {
  console.log('Claude Code restored development workspace')
  // 继续启动，允许在恢复模式下工作
}

// 正常启动，转到 cli.tsx
await import('./entrypoints/cli.tsx')
```

#### cli.tsx — 命令行路由器

负责解析命令行参数，分发到不同处理路径。所有导入都是动态的，以实现真正的零模块加载快速路径：

```typescript
// src/entrypoints/cli.tsx

async function main(): Promise<void> {
  const args = process.argv.slice(2)

  // 快速路径 1：--version（零模块加载）
  if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
    console.log(`${MACRO.VERSION} (Claude Code)`)
    return
  }

  // 加载启动性能分析器
  const { profileCheckpoint } = await import('../utils/startupProfiler.js')
  profileCheckpoint('cli_entry')

  // 快速路径 2：--dump-system-prompt（需要加载配置）
  if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') {
    // ... 输出系统提示词后退出
    return
  }

  // 快速路径 3：--daemon-worker
  if (feature('DAEMON') && args[0] === '--daemon-worker') {
    // ... 后台工作进程
    return
  }

  // 快速路径 4：bridge/remote-control
  if (feature('BRIDGE_MODE') && args[0] === 'remote-control') {
    // ... 远程控制模式
    return
  }

  // 完整路径：加载主程序
  const { main: cliMain } = await import('../main.js')
  await cliMain()
}
```

### 2.2 初始化模块（init.ts）

`init.ts` 是真正的初始化逻辑，通过 `memoize` 确保只执行一次：

```typescript
// src/entrypoints/init.ts

export const init = memoize(async (): Promise<void> => {
  const initStartTime = Date.now()

  // 1. 启用配置系统
  enableConfigs()

  // 2. 应用安全的环境变量（信任对话框之前）
  applySafeConfigEnvironmentVariables()

  // 3. 应用 CA 证书（Bun 在启动时缓存 TLS 证书）
  applyExtraCACertsFromConfig()

  // 4. 设置优雅关闭
  setupGracefulShutdown()

  // 5. 初始化遥测（延迟加载，~400KB）
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(([fp, gb]) => {
    fp.initialize1PEventLogging()
    // ...
  })

  // 6. 填充 OAuth 账户信息
  void populateOAuthAccountInfoIfNeeded()

  // 7. 初始化远程托管设置
  if (isEligibleForRemoteManagedSettings()) {
    initializeRemoteManagedSettingsLoadingPromise()
  }

  // 8. 配置 mTLS 和代理
  configureGlobalMTLS()
  configureGlobalAgents()

  // 9. 预连接 Anthropic API（重叠 TCP+TLS 握手）
  preconnectAnthropicApi()

  // 10. 初始化 scratchpad 目录
  if (isScratchpadEnabled()) {
    await ensureScratchpadDir()
  }
})
```

### 2.3 主程序流程（main.tsx）

`main.tsx` 是完整的初始化入口，导入量巨大（约 200 行 import）：

```typescript
// src/main.tsx 的 main 函数核心流程

async function main() {
  // 启动前准备（并行执行）
  profileCheckpoint('main_tsx_entry')
  startMdmRawRead()        // MDM 子进程
  startKeychainPrefetch()  // macOS 钥匙串预读

  // 解析命令行参数
  const options = parseArgs()

  // 调用初始化
  await init()

  // 加载配置
  const config = await loadConfig()

  // 检查是否需要认证
  if (needsAuth) {
    await initAuth()
  }

  // 初始化 MCP
  await initMcp(config)

  // 加载设置
  const settings = await loadSettings()

  // 创建 AppState
  const appState = createAppState(settings)

  // 渲染 REPL
  await launchRepl(appState)
}
```

### 2.4 状态初始化（AppState）

全局状态通过 `bootstrap/state.ts` 管理：

```typescript
// src/bootstrap/state.ts

type State = {
  // 会话信息
  sessionId: SessionId
  startTime: number

  // 模型配置
  mainLoopModelOverride: ModelSetting | undefined
  initialMainLoopModel: ModelSetting
  modelStrings: ModelStrings | null

  // 成本追踪
  totalCostUSD: number
  totalAPIDuration: number
  totalToolDuration: number

  // 权限
  isBypassPermissionsModeAvailable: boolean
  permissionMode: PermissionMode

  // 遥测
  meter: Meter | null
  sessionCounter: AttributedCounter | null
}
```

### 2.5 Feature Flag 机制

启动时的功能开关通过 `feature()` 函数控制：

```typescript
// src/entrypoints/cli.tsx

// feature() 在构建时内联，实现死代码消除
if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') {
  // 外部构建会完全移除此代码
}

if (feature('DAEMON') && args[0] === 'daemon') {
  // 只有启用 DAEMON 特性的构建才包含此代码
}

if (feature('BRIDGE_MODE') && args[0] === 'remote-control') {
  // 只有启用 BRIDGE_MODE 特性的构建才包含此代码
}
```

### 2.6 启动性能分析

使用 `startupProfiler.js` 追踪各阶段耗时：

```typescript
// 标记各阶段检查点
profileCheckpoint('cli_entry')
profileCheckpoint('cli_before_main_import')
profileCheckpoint('cli_after_main_import')
profileCheckpoint('init_function_end')

// 输出启动耗时报告
// 用于识别性能瓶颈
```

## 3. 关键源文件

| 功能 | 文件 |
|------|------|
| 开发启动器 | `src/dev-entry.ts` |
| CLI 路由 | `src/entrypoints/cli.tsx` |
| 初始化逻辑 | `src/entrypoints/init.ts` |
| 主程序 | `src/main.tsx` |
| 全局状态 | `src/bootstrap/state.ts` |
| 启动分析 | `src/utils/startupProfiler.js` |
| 配置加载 | `src/utils/config.js` |
| 权限设置 | `src/utils/permissions/permissionSetup.js` |

## 4. 本章小结

- Claude Code 采用三层启动入口：dev-entry（完整性检查）→ cli（路由分发）→ main（完整初始化）
- 快速路径机制：使用动态导入和 Feature Flag，实现零模块加载快捷命令
- 初始化流程：配置 → 安全环境变量 → mTLS/代理 → MCP → 设置 → 状态 → REPL
- Feature Flag 在构建时内联，配合 `feature()` 函数实现精确的死代码消除
- 启动性能分析：通过 `profileCheckpoint` 追踪各阶段耗时

---

## 下一步

← [01_ARCHITECTURE.md](01_ARCHITECTURE.md) 回顾整体架构设计
→ [03_AGENT_SYSTEM/README.md](03_AGENT_SYSTEM/README.md) 学习 Agent 系统
