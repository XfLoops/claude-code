# 工程实践

## 1. 代码质量配置

### 1.1 Biome 配置

**文件**: `biome.json`

```json
{
  "$schema": "https://biomejs.dev/schemas/1.5.3/schema.json",
  "organizeImports": {
    "enabled": false
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "suspicious": {
        "noExplicitAny": "off",
        "noConsole": "off",
        "noDebugger": "warn"
      },
      "style": {
        "useConst": "off",
        "useTemplate": "off",
        "useNamingConvention": "off"
      },
      "complexity": {
        "noForEach": "off",
        "useArrowFunction": "off",
        "noStaticOnlyClass": "off"
      },
      "correctness": {
        "noUnusedImports": "warn",
        "noUnusedVariables": "warn"
      }
    }
  },
  "formatter": {
    "enabled": false
  },
  "assist": {
    "enabled": false
  }
}
```

**策略**: 
- 宽松规则适应遗留代码
- 仅启用关键规则（suspicious、correctness）
- 关闭 Formatter 使用其他工具

### 1.2 TypeScript 配置

**文件**: `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": false,
    "skipLibCheck": true,
    "noEmit": true,
    "paths": {
      "src/*": ["./src/*"]
    },
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**设计考量**:
- Bundler 模块解析
- `strict: false` 允许渐进迁移
- 路径别名简化导入

## 2. 项目配置

### 2.1 Bun 工作区

**文件**: `package.json`

```json
{
  "name": "claude-code",
  "type": "module",
  "workspaces": [
    "packages/*",
    "packages/@ant/*"
  ],
  "scripts": {
    "build": "bun build src/entrypoints/cli.tsx --outdir dist --target bun",
    "dev": "bun run src/entrypoints/cli.tsx",
    "test": "bun test"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.24.0",
    "react": "^19.0.0",
    "ink": "^5.0.0",
    "zod": "^3.22.0",
    "color-diff-napi": "workspace:*"
  },
  "devDependencies": {
    "@types/bun": "latest",
    "@types/react": "^19.0.0"
  }
}
```

**关键实践**:
- ESM 模块系统
- Bun 工作区共享内部包
- 使用 `workspace:*` 依赖

### 2.2 Bun 配置

**文件**: `bunfig.toml`

```toml
[test]
preload = "./test/setup.ts"

[install]
exact = true
```

## 3. 开发工作流

### 3.1 启动脚本

```bash
# 开发模式
bun run dev

# 构建
bun run build

# 管道模式
echo "hello" | bun run dev -p

# 初始化
bun run dev --init
```

### 3.2 并行初始化模式

```typescript
// 启动时并行预取
startMdmRawRead();        // MDM 检查（并行）
startKeychainPrefetch();  // Keychain 预取（并行）

// 非阻塞加载模块
const [{ QueryEngine }, { REPL }] = await Promise.all([
  import('./QueryEngine.js'),
  import('./screens/REPL.tsx'),
]);
```

### 3.3 动态导入策略

```typescript
// 条件功能懒加载
const analyticsModule = feature('ANALYTICS')
  ? await import('./services/analytics/index.js')
  : null;

// 工具动态加载
async function loadOptionalTools(): Promise<Tool[]> {
  const tools: Tool[] = [];
  
  if (feature('WEB_BROWSER_TOOL')) {
    const { WebBrowserTool } = await import('./tools/WebBrowserTool/index.js');
    tools.push(WebBrowserTool);
  }
  
  return tools;
}
```

## 4. 代码组织

### 4.1 模块化结构

```
src/
├── entrypoints/      # 入口点
│   ├── cli.tsx      # CLI 入口
│   ├── init.ts      # 初始化
│   └── mcp.ts       # MCP 入口
├── main.tsx         # Commander 定义
├── query.ts         # 核心查询循环
├── QueryEngine.ts   # 查询引擎
├── Tool.ts          # 工具类型
├── tools.ts         # 工具注册表
├── tools/           # 各工具实现
├── services/
│   ├── api/         # API 客户端
│   ├── mcp/         # MCP 服务
│   └── ...
├── state/           # 状态管理
├── ink/             # Ink 框架
├── components/      # React 组件
└── utils/           # 工具函数
```

### 4.2 工具目录约定

```
src/tools/ToolName/
├── index.ts         # 主导出
├── tool.ts          # 工具定义
├── render.tsx       # UI 渲染
├── permissions.ts   # 权限检查
├── validation.ts    # 输入验证
└── __tests__/       # 测试
    └── tool.test.ts
```

## 5. 类型安全

### 5.1 Zod 运行时验证

```typescript
import { z } from 'zod';

const BashInputSchema = z.object({
  command: z.string(),
  timeout: z.number().optional(),
  cwd: z.string().optional(),
});

// 运行时验证
const validated = BashInputSchema.parse(input);

// TypeScript 类型推断
type BashInput = z.infer<typeof BashInputSchema>;
```

### 5.2 类型导出模式

```typescript
// types.ts
export type { Tool } from './Tool.js';
export type { QueryResult } from './query.js';
export type { Message, AssistantMessage, UserMessage } from './types/message.js';

// 避免循环依赖
type Lazy<T> = () => T;
```

## 6. 错误处理

### 6.1 错误分类

```typescript
enum StartupError {
  CONFIG_NOT_FOUND = 'CONFIG_NOT_FOUND',
  AUTH_MISSING_KEY = 'AUTH_MISSING_KEY',
  NETWORK_UNAVAILABLE = 'NETWORK_UNAVAILABLE',
  PERMISSION_DENIED = 'PERMISSION_DENIED',
}
```

### 6.2 错误恢复

```typescript
async function initializeWithFallback(): Promise<Services> {
  try {
    return await initializeServices();
  } catch (error) {
    if (error.code === 'AUTH_FAILED') {
      await reconfigureAuth();
      return await initializeServices();
    }
    throw error;
  }
}
```

## 7. 配置管理

### 7.1 配置来源优先级

1. 命令行参数
2. 环境变量
3. 项目级配置 (`.claude/config.json`)
4. 用户级配置 (`~/.config/claude/settings.json`)
5. 默认值

### 7.2 配置加载

```typescript
async function loadConfig(options: CLIOptions): Promise<Config> {
  const defaults = getDefaultConfig();
  const userConfig = await loadUserConfig();
  const projectConfig = await loadProjectConfig(options.dir);
  const envConfig = loadEnvConfig();
  const cliConfig = options;
  
  return mergeConfigs(defaults, userConfig, projectConfig, envConfig, cliConfig);
}
```

## 8. 总结

工程实践的核心原则：

1. **宽松但有效**: 宽松规则 + 关键检查
2. **模块化**: 清晰的目录结构和职责分离
3. **类型安全**: Zod 运行时验证 + TypeScript 编译时检查
4. **性能**: 并行初始化、动态导入
5. **渐进改进**: 接受技术债务，新代码遵循高标准
