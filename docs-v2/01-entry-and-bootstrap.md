# 入口与启动流程

## 文件位置

- `src/entrypoints/cli.tsx` - 主入口点
- `src/entrypoints/init.ts` - 初始化逻辑
- `src/main.tsx` - Commander.js CLI 定义

## 1. 入口架构概述

Claude Code 采用多层入口架构，实现快速启动路径和模块化加载。

```
┌─────────────────────────────────────────────────────────────┐
│                     入口层 (Entry Layer)                      │
├─────────────────────────────────────────────────────────────┤
│  cli.tsx                                                      │
│  ├── Runtime Polyfill 注入                                    │
│  ├── Feature Flag 初始化                                      │
│  ├── 零导入快速路径 (--version, --help)                       │
│  └── 动态导入 main.tsx                                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    启动层 (Bootstrap Layer)                   │
├─────────────────────────────────────────────────────────────┤
│  main.tsx                                                     │
│  ├── Commander.js CLI 定义                                    │
│  ├── 并行初始化 (MDM, Keychain)                               │
│  ├── 服务初始化 (Auth, Analytics)                             │
│  └── 模式路由 (REPL, Pipe, Init)                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    运行时层 (Runtime Layer)                   │
├─────────────────────────────────────────────────────────────┤
│  REPL / Pipe / Init                                          │
│  ├── QueryEngine 实例化                                       │
│  ├── 状态管理初始化                                           │
│  └── 主循环启动                                               │
└─────────────────────────────────────────────────────────────┘
```

## 2. Runtime Polyfill 机制

### 2.1 问题背景

Claude Code 使用 Bun 的构建时宏系统 (`bun:bundle`) 注入构建信息。在开发/测试环境中需要模拟这些宏。

### 2.2 Polyfill 实现

**文件**: `src/entrypoints/cli.tsx`

```typescript
// 1. Feature Flag Polyfill
// 构建时通过 bun:bundle 导入，开发时 polyfill
const feature = (_name: string): boolean => false;

// 2. MACRO 全局对象注入
if (typeof globalThis.MACRO === "undefined") {
  (globalThis as any).MACRO = {
    VERSION: "2.1.888",
    BUILD_TIME: new Date().toISOString(),
    COMMIT: "unknown",
    IS_DEV: false,
    PLATFORM: process.platform,
  };
}

// 3. 构建目标常量
(globalThis as any).BUILD_TARGET = "external";
(globalThis as any).BUILD_ENV = "production";
(globalThis as any).INTERFACE_TYPE = "terminal";

// 4. 内部标识符（external 构建中不使用）
(globalThis as any).K8S_CLUSTER = null;
(globalThis as any).ANTHROPIC_INTERNAL = false;
```

### 2.3 设计原理

```typescript
// 构建时（bun:bundle）代码示例：
import { feature } from 'bun:bundle';

// feature() 在构建时被评估
if (feature('COORDINATOR_MODE')) {
  // 这段代码在 feature 返回 false 时会被完全移除
  await import('./coordinator/mode.js');
}

// 运行时（cli.tsx polyfill）行为：
// - feature() 始终返回 false
// - 条件代码块不会被执行
// - 但代码存在于运行时的包中
```

**关键区别**：
- **构建时 DCE**: `bun:bundle` 的 `feature()` 在构建时评估，false 则删除代码
- **运行时 polyfill**: `cli.tsx` 的 `feature()` 在运行时评估，代码存在但不会执行

## 3. 快速路径设计

### 3.1 零导入快速路径

```typescript
// cli.tsx 中的快速路径实现
async function main(): Promise<void> {
  const args = process.argv.slice(2);
  
  // 快速路径：无需加载其他模块
  if (args.length === 1) {
    if (args[0] === "--version" || args[0] === "-v") {
      console.log(`${MACRO.VERSION} (Claude Code)`);
      return;  // 立即退出，零额外模块加载
    }
    
    if (args[0] === "--help" || args[0] === "-h") {
      console.log("Claude Code - AI-powered coding assistant");
      console.log("Usage: claude [options] [prompt]");
      return;
    }
  }
  
  // 慢速路径：动态导入 main.tsx
  const { run } = await import("../main.tsx");
  await run();
}
```

### 3.2 性能影响

| 命令 | 模块加载 | 启动时间 |
|------|----------|----------|
| `claude --version` | ~1 个 | <50ms |
| `claude "hello"` | ~500 个 | ~200ms |
| `claude` (REPL) | ~1000 个 | ~500ms |

## 4. 动态导入模式

### 4.1 条件功能加载

```typescript
// main.tsx 中的条件加载

// 1. 并行启动非关键初始化
startMdmRawRead();        // MDM 检查（并行）
startKeychainPrefetch();  // Keychain 预取（并行）

// 2. 动态导入可选功能
const analyticsModule = feature('ANALYTICS')
  ? await import('./services/analytics/index.js')
  : null;

// 3. 懒加载大型模块
const { QueryEngine } = await import('./QueryEngine.js');
const { REPL } = await import('./screens/REPL.tsx');
```

### 4.2 动态导入工具

```typescript
// tools.ts 中的工具动态加载

// 基础工具始终加载
import { BashTool } from './tools/BashTool/index.js';
import { FileReadTool } from './tools/FileReadTool/index.js';

// 可选工具条件加载
async function loadOptionalTools(): Promise<Tool[]> {
  const tools: Tool[] = [];
  
  if (feature('WEB_BROWSER_TOOL')) {
    const { WebBrowserTool } = await import('./tools/WebBrowserTool/index.js');
    tools.push(WebBrowserTool);
  }
  
  if (feature('MCP')) {
    const { MCPTool } = await import('./tools/MCPTool/index.js');
    tools.push(MCPTool);
  }
  
  return tools;
}
```

## 5. Commander.js CLI 定义

### 5.1 CLI 结构

**文件**: `src/main.tsx`

```typescript
import { Command } from 'commander';

export async function run(): Promise<void> {
  const program = new Command()
    .name('claude')
    .version(MACRO.VERSION)
    .description('Claude Code - AI-powered coding assistant')
    .option('-p, --pipe', 'Run in pipe mode (non-interactive)')
    .option('--init', 'Run initialization wizard')
    .option('-d, --dir <path>', 'Working directory')
    .option('--model <model>', 'Model to use')
    .argument('[prompt]', 'Initial prompt');
  
  program.action(async (prompt: string | undefined, options: CLIOptions) => {
    // 1. 初始化日志
    setupLogging();
    
    // 2. 加载配置
    const config = await loadConfig(options);
    
    // 3. 模式路由
    if (options.init) {
      await runInitWizard();
    } else if (options.pipe || prompt) {
      await runPipeMode(prompt, config);
    } else {
      await runREPLMode(config);
    }
  });
  
  await program.parseAsync();
}
```

### 5.2 模式路由

```typescript
// 三种运行模式

// 1. REPL 模式（默认）
async function runREPLMode(config: Config): Promise<void> {
  const { QueryEngine } = await import('./QueryEngine.js');
  const { REPL } = await import('./screens/REPL.tsx');
  
  const engine = new QueryEngine(config);
  const repl = new REPL(engine);
  
  await repl.start();
}

// 2. Pipe 模式（非交互式）
async function runPipeMode(prompt: string | undefined, config: Config): Promise<void> {
  const input = prompt || await readStdin();
  
  const { QueryEngine } = await import('./QueryEngine.js');
  const engine = new QueryEngine(config);
  
  const stream = engine.submitMessage(input);
  
  // 流式输出到 stdout
  for await (const message of stream) {
    process.stdout.write(formatMessage(message));
  }
}

// 3. Init 模式（初始化向导）
async function runInitWizard(): Promise<void> {
  const { init } = await import('./entrypoints/init.ts');
  await init();
}
```

## 6. 初始化流程

### 6.1 一次性初始化

**文件**: `src/entrypoints/init.ts`

```typescript
export async function init(): Promise<void> {
  // 1. 检查是否已初始化
  if (await isInitialized()) {
    console.log('Claude Code is already initialized');
    return;
  }
  
  // 2. 显示信任对话框
  const trustGranted = await showTrustDialog();
  if (!trustGranted) {
    process.exit(1);
  }
  
  // 3. 配置 API 密钥
  await configureAPIKey();
  
  // 4. 配置遥测（可选）
  await configureTelemetry();
  
  // 5. 创建配置文件
  await createConfigFile();
  
  // 6. 标记已初始化
  await markInitialized();
}

async function showTrustDialog(): Promise<boolean> {
  const { TrustDialog } = await import('./components/TrustDialog/index.js');
  
  return new Promise((resolve) => {
    render(
      <TrustDialog
        onAccept={() => resolve(true)}
        onReject={() => resolve(false)}
      />
    );
  });
}
```

### 6.2 信任对话框

```typescript
// 信任对话框向用户说明：
// 1. Claude Code 可以执行 shell 命令
// 2. Claude Code 可以读写文件
// 3. 用户应审查每个操作
// 4. 敏感操作需要确认
```

## 7. 并行初始化优化

### 7.1 非阻塞启动

```typescript
// main.tsx 中的并行初始化

export async function run(): Promise<void> {
  // 1. 启动并行任务（不等待）
  const mdmPromise = startMdmRawRead();
  const keychainPromise = startKeychainPrefetch();
  
  // 2. 继续主流程（并行加载关键模块）
  const [{ QueryEngine }, { REPL }] = await Promise.all([
    import('./QueryEngine.js'),
    import('./screens/REPL.tsx'),
  ]);
  
  // 3. 等待并行任务完成（如果需要结果）
  const [mdmResult, keychainResult] = await Promise.all([
    mdmPromise,
    keychainPromise,
  ]);
  
  // 4. 启动主程序
  const engine = new QueryEngine({ mdmResult, keychainResult });
  await engine.start();
}
```

### 7.2 预取策略

```typescript
// 预取 API 密钥（避免运行时延迟）
async function startKeychainPrefetch(): Promise<string | null> {
  try {
    const key = await keychain.getPassword('claude-code', 'api-key');
    return key;
  } catch {
    return null;
  }
}

// 预取 MDM 状态（企业环境检查）
async function startMdmRawRead(): Promise<MDMState> {
  try {
    const state = await checkMDMStatus();
    return state;
  } catch {
    return { managed: false };
  }
}
```

## 8. 服务初始化

### 8.1 服务层初始化

```typescript
// services/ 目录初始化顺序

async function initializeServices(): Promise<Services> {
  // 1. 配置服务（最先）
  const config = await initializeConfig();
  
  // 2. 日志服务
  const logger = initializeLogger(config.logLevel);
  
  // 3. 认证服务
  const auth = await initializeAuth(config);
  
  // 4. API 客户端
  const apiClient = createAPIClient(auth, config);
  
  // 5. 可选服务（Analytics、Sentry 等）
  const analytics = feature('ANALYTICS')
    ? await initializeAnalytics(config)
    : null;
  
  return { config, logger, auth, apiClient, analytics };
}
```

### 8.2 失败处理

```typescript
// 服务初始化失败处理

async function initializeWithFallback(): Promise<Services> {
  try {
    return await initializeServices();
  } catch (error) {
    if (error.code === 'AUTH_FAILED') {
      // 重试认证
      await reconfigureAuth();
      return await initializeServices();
    }
    
    if (error.code === 'NETWORK_ERROR') {
      // 离线模式
      return initializeOfflineMode();
    }
    
    throw error;
  }
}
```

## 9. 启动时序图

```
用户命令行输入
    │
    ▼
┌──────────────┐
│   cli.tsx    │
│ - polyfill   │
│ - 快速路径   │
└──────────────┘
    │
    ▼（动态导入）
┌──────────────┐
│  main.tsx    │
│ - CLI 定义   │
│ - 并行初始化 │
└──────────────┘
    │
    ├──────────────────┬──────────────────┐
    ▼                  ▼                  ▼
┌──────────┐    ┌──────────┐    ┌──────────┐
│   Init   │    │   REPL   │    │   Pipe   │
│   模式   │    │   模式   │    │   模式   │
└──────────┘    └──────────┘    └──────────┘
                     │
                     ▼
              ┌──────────────┐
              │ QueryEngine  │
              │ - 状态初始化 │
              │ - API 准备   │
              └──────────────┘
                     │
                     ▼
              ┌──────────────┐
              │   REPL 循环  │
              │ - 用户输入   │
              │ - 查询处理   │
              │ - 结果展示   │
              └──────────────┘
```

## 10. 配置加载

### 10.1 配置来源优先级

```typescript
// 配置来源（优先级从高到低）

1. 命令行参数 (--model, --dir 等)
2. 环境变量 (ANTHROPIC_API_KEY, CLAUDE_MODEL 等)
3. 项目级配置 (.claude/config.json)
4. 用户级配置 (~/.config/claude/settings.json)
5. 默认值
```

### 10.2 配置加载代码

```typescript
async function loadConfig(options: CLIOptions): Promise<Config> {
  // 1. 加载默认配置
  const defaults = getDefaultConfig();
  
  // 2. 加载用户配置
  const userConfig = await loadUserConfig();
  
  // 3. 加载项目配置
  const projectConfig = await loadProjectConfig(options.dir);
  
  // 4. 环境变量覆盖
  const envConfig = loadEnvConfig();
  
  // 5. 命令行参数覆盖
  const cliConfig = options;
  
  // 6. 合并配置（优先级：cli > env > project > user > defaults）
  return mergeConfigs(defaults, userConfig, projectConfig, envConfig, cliConfig);
}
```

## 11. 错误处理

### 11.1 启动错误分类

```typescript
enum StartupError {
  // 配置错误
  CONFIG_NOT_FOUND = 'CONFIG_NOT_FOUND',
  CONFIG_INVALID = 'CONFIG_INVALID',
  
  // 认证错误
  AUTH_MISSING_KEY = 'AUTH_MISSING_KEY',
  AUTH_INVALID_KEY = 'AUTH_INVALID_KEY',
  
  // 网络错误
  NETWORK_UNAVAILABLE = 'NETWORK_UNAVAILABLE',
  API_UNREACHABLE = 'API_UNREACHABLE',
  
  // 系统错误
  PERMISSION_DENIED = 'PERMISSION_DENIED',
  DISK_FULL = 'DISK_FULL',
}
```

### 11.2 错误恢复策略

```typescript
async function handleStartupError(error: StartupError): Promise<void> {
  switch (error.code) {
    case 'AUTH_MISSING_KEY':
      console.error('API key not found. Run: claude --init');
      process.exit(1);
      
    case 'CONFIG_INVALID':
      console.error('Invalid configuration file. Please check ~/.config/claude/settings.json');
      process.exit(1);
      
    case 'NETWORK_UNAVAILABLE':
      console.warn('Network unavailable. Starting in offline mode...');
      // 继续启动
      break;
      
    default:
      console.error('Startup error:', error.message);
      process.exit(1);
  }
}
```

## 12. 总结

入口与启动流程的设计体现了以下架构原则：

1. **快速路径**: 常用命令走零导入快速路径
2. **按需加载**: 使用动态导入延迟加载非关键模块
3. **并行初始化**: 启动任务并行化，减少启动时间
4. **失败隔离**: 服务初始化失败不阻断主流程
5. **配置分层**: 多层级配置，灵活覆盖
