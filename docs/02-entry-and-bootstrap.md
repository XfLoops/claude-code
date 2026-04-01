# 入口与启动流程详解

## 启动流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        启动流程                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 1: 顶层 Polyfill 注入 (cli.tsx 顶部)                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. feature() polyfill → 始终返回 false                  │   │
│  │ 2. globalThis.MACRO 注入 (VERSION, BUILD_TIME等)        │   │
│  │ 3. BUILD_TARGET/ENV/INTERFACE_TYPE 全局常量             │   │
│  │ 4. process.env.COREPACK_ENABLE_AUTO_PIN = "0"          │   │
│  │ 5. CLAUDE_CODE_REMOTE 堆内存设置                        │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 2: Fast-Path 路由 (cli.tsx main())                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ --version/-v      → 输出版本，零模块加载                 │   │
│  │ --dump-system-prompt → 输出系统提示（ant-only）          │   │
│  │ --claude-in-chrome-mcp → Chrome MCP 服务器              │   │
│  │ --chrome-native-host → Chrome Native Host               │   │
│  │ --computer-use-mcp → Computer Use MCP（ant-only）       │   │
│  │ --daemon-worker   → Daemon Worker（内部）                │   │
│  │ remote-control    → 桥接模式                             │   │
│  │ daemon            → Daemon 主进程                       │   │
│  │ ps/logs/attach/kill → 后台会话管理                       │   │
│  │ new/list/reply    → 模板作业（ant-only）                 │   │
│  │ environment-runner → BYOC Runner（ant-only）            │   │
│  │ self-hosted-runner → Self-hosted Runner（ant-only）     │   │
│  │ --worktree + --tmux → Tmux Worktree 快速路径            │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 3: 主 CLI 初始化 (main.tsx)                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. 启动性能分析器 (startupProfiler)                      │   │
│  │ 2. 并行启动 MDM 读取和 Keychain 预取                     │   │
│  │ 3. 执行 init() 初始化函数                                │   │
│  │ 4. 解析 CLI 参数 (Commander.js)                         │   │
│  │ 5. 启动 REPL 或执行管道模式                              │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## cli.tsx 详解

### 1. 顶层 Polyfill 注入

```typescript
// Runtime polyfill for bun:bundle (build-time macros)
const feature = (_name: string) => false;

if (typeof globalThis.MACRO === "undefined") {
    (globalThis as any).MACRO = {
        VERSION: "2.1.888",
        BUILD_TIME: new Date().toISOString(),
        FEEDBACK_CHANNEL: "",
        ISSUES_EXPLAINER: "",
        NATIVE_PACKAGE_URL: "",
        PACKAGE_URL: "",
        VERSION_CHANGELOG: "",
    };
}

(globalThis as any).BUILD_TARGET = "external";
(globalThis as any).BUILD_ENV = "production";
(globalThis as any).INTERFACE_TYPE = "stdio";
```

**关键作用**：
- `feature()`: 控制特性开关，在此版本中始终返回 `false`
- `MACRO`: 构建时宏，包含版本信息
- `BUILD_TARGET/ENV`: 构建目标标识

### 2. 快速路径路由

```typescript
async function main(): Promise<void> {
    const args = process.argv.slice(2);

    // Fast-path for --version/-v: zero module loading needed
    if (args.length === 1 && (args[0] === "--version" || args[0] === "-v")) {
        console.log(`${MACRO.VERSION} (Claude Code)`);
        return;
    }

    // 其他快速路径...
    if (args[0] === "remote-control") { /* ... */ }
    if (args[0] === "daemon") { /* ... */ }
    if (args[0] === "ps" || args[0] === "logs") { /* ... */ }
    
    // 加载完整 CLI
    const { main: cliMain } = await import("../main.jsx");
    await cliMain();
}
```

### 3. 特殊模式详解

#### Daemon 模式
```typescript
if (feature("DAEMON") && args[0] === "daemon") {
    const { enableConfigs } = await import("../utils/config.js");
    enableConfigs();
    const { daemonMain } = await import("../daemon/main.js");
    await daemonMain(args.slice(1));
    return;
}
```

用于启动长期运行的 supervisor 进程，管理后台任务。

#### 桥接模式 (Bridge Mode)
```typescript
if (feature("BRIDGE_MODE") && args[0] === "remote-control") {
    // 1. 启用配置
    // 2. 检查 OAuth 认证
    // 3. 检查 GrowthBook 开关
    // 4. 检查策略限制
    // 5. 启动桥接主程序
    await bridgeMain(args.slice(1));
}
```

允许远程控制本地机器，用于 Claude Desktop 等场景。

## init.ts 详解

### 初始化流程

```typescript
export const init = memoize(async (): Promise<void> => {
  // 1. 启用配置系统
  enableConfigs();
  
  // 2. 应用安全环境变量
  applySafeConfigEnvironmentVariables();
  applyExtraCACertsFromConfig();
  
  // 3. 设置优雅关闭
  setupGracefulShutdown();
  
  // 4. 初始化事件日志（延迟加载）
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(/* ... */);
  
  // 5. 预填充 OAuth 信息
  void populateOAuthAccountInfoIfNeeded();
  
  // 6. 检测 IDE 环境
  void initJetBrainsDetection();
  
  // 7. 检测 GitHub 仓库
  void detectCurrentRepository();
  
  // 8. 初始化远程管理设置
  if (isEligibleForRemoteManagedSettings()) {
    initializeRemoteManagedSettingsLoadingPromise();
  }
  
  // 9. 配置 mTLS
  configureGlobalMTLS();
  
  // 10. 配置代理
  configureGlobalAgents();
  
  // 11. 预连接 Anthropic API
  preconnectAnthropicApi();
  
  // 12. 初始化上游代理（CCR 环境）
  if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE)) {
    await initUpstreamProxy();
  }
  
  // 13. 初始化 scratchpad 目录
  if (isScratchpadEnabled()) {
    await ensureScratchpadDir();
  }
});
```

### 关键初始化点

#### 配置系统
```typescript
enableConfigs();  // 启用 settings.json 配置
applySafeConfigEnvironmentVariables();  // 应用安全环境变量
applyExtraCACertsFromConfig();  // 应用自定义 CA 证书
```

#### 网络配置
```typescript
configureGlobalMTLS();  // 配置 mTLS
configureGlobalAgents();  // 配置 HTTP 代理
preconnectAnthropicApi();  // TCP+TLS 预连接
```

#### 远程设置
```typescript
if (isEligibleForRemoteManagedSettings()) {
  initializeRemoteManagedSettingsLoadingPromise();
}
if (isPolicyLimitsEligible()) {
  initializePolicyLimitsLoadingPromise();
}
```

## main.tsx 详解

### 启动性能优化

```typescript
// These side-effects must run before all other imports
import { profileCheckpoint } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');

import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();  // 并行启动 MDM 读取

import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();  // 并行启动 Keychain 预取
```

### CLI 定义

```typescript
const program = new CommanderCommand()
  .name('claude')
  .description('Claude Code CLI')
  .version(MACRO.VERSION)
  .option('-p, --print', 'Print mode (pipe input)')
  .option('--model <model>', 'Specify model')
  .option('--permission-mode <mode>', 'Set permission mode')
  // ... 更多选项
  .action(async (options) => {
    // 启动逻辑
  });
```

### 启动模式

#### 交互模式 (REPL)
```typescript
// 启动 REPL 屏幕
await renderAndRun(
  <App>
    <REPL /* props */ />
  </App>
);
```

#### 管道模式
```typescript
if (options.print || peekForStdinData()) {
  // 读取 stdin 并执行
  const input = await readStdin();
  await runHeadless(input);
}
```

## 性能优化策略

### 1. 延迟加载

```typescript
// 使用动态 import 延迟加载重型模块
const { initializeTelemetry } = await import(
  '../utils/telemetry/instrumentation.js'
);
```

### 2. 并行启动

```typescript
// 并行执行独立初始化任务
await Promise.all([
  getBranch(),
  getDefaultBranch(),
  execFileNoThrow(gitExe(), ['status']),
  execFileNoThrow(gitExe(), ['log']),
]);
```

### 3. 预取

```typescript
// 预取可能需要的资源
startKeychainPrefetch();
preconnectAnthropicApi();
```

### 4. Memoization

```typescript
export const init = memoize(async (): Promise<void> => {
  // 确保只执行一次
});
```

## 错误处理

### 配置错误处理
```typescript
try {
  enableConfigs();
} catch (error) {
  if (error instanceof ConfigParseError) {
    if (getIsNonInteractiveSession()) {
      process.stderr.write(`Configuration error: ${error.message}\n`);
      gracefulShutdownSync(1);
    } else {
      showInvalidConfigDialog({ error });
    }
  }
}
```

### 优雅关闭
```typescript
setupGracefulShutdown();

// 注册清理函数
registerCleanup(shutdownLspServerManager);
registerCleanup(async () => {
  const { cleanupSessionTeams } = await import('../utils/swarm/teamHelpers.js');
  await cleanupSessionTeams();
});
```
