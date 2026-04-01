# Claude Code CLI 源码架构概述

## 项目简介

这是一个**逆向工程/反编译版本**的 Anthropic Claude Code CLI 工具，目标是恢复核心功能，同时裁剪次要特性。代码使用 **Bun** 运行时（而非 Node.js），采用 React + Ink 构建终端 UI。

### 技术栈

| 组件 | 技术 |
|------|------|
| 运行时 | Bun (替代 Node.js) |
| 构建 | `bun build` 单文件打包 (~25MB) |
| 模块系统 | ESM + TSX |
| Monorepo | Bun workspaces |
| UI 框架 | React + Ink (终端 UI) |
| 状态管理 | Zustand-style store |
| API 客户端 | @anthropic-ai/sdk |

### 项目结构

```
src/
├── entrypoints/       # 入口点
│   ├── cli.tsx       # 主入口（polyfill + 快速路径）
│   └── init.ts       # 一次性初始化
├── main.tsx          # CLI 定义与启动
├── query.ts          # 核心 API 查询循环
├── QueryEngine.ts    # 查询生命周期管理
├── Tool.ts           # 工具接口定义
├── tools.ts          # 工具注册表
├── context.ts        # 系统/用户上下文构建
│
├── tools/            # 各工具实现
├── components/       # React 组件
├── screens/          # 屏幕组件（REPL等）
├── ink/              # 自定义 Ink 框架
├── state/            # 状态管理
├── services/         # 服务层（API、MCP等）
├── utils/            # 工具函数
└── types/            # 类型定义
```

## 核心架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Entry Layer                               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │ cli.tsx     │───→│  main.tsx   │───→│   REPL / Pipe Mode  │  │
│  │ (polyfills) │    │  (Commander)│    │                     │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Query Engine Layer                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │ QueryEngine │───→│  query.ts   │───→│   Claude API        │  │
│  │ (状态管理)   │    │ (核心循环)   │    │   (流式响应)         │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Tool System Layer                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────────┐  │
│  │BashTool  │ │File*Tool │ │GlobTool  │ │  AgentTool         │  │
│  └──────────┘ └──────────┘ └──────────┘ └────────────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────────┐  │
│  │Web*Tool  │ │Task*Tool │ │MCP Tools │ │  SkillTool         │  │
│  └──────────┘ └──────────┘ └──────────┘ └────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                          UI Layer                                │
│  ┌─────────────────┐    ┌─────────────┐    ┌─────────────────┐  │
│  │  Ink Framework  │───→│ Components  │───→│  REPL Screen    │  │
│  │ (自定义reconciler)│    │ (React组件)  │    │  (交互主界面)    │  │
│  └─────────────────┘    └─────────────┘    └─────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## 关键设计原则

### 1. 特性开关系统 (Feature Flags)

```typescript
// cli.tsx 中 polyfill 为始终返回 false
const feature = (_name: string) => false;
```

所有 Anthropic 内部特性（COORDINATOR_MODE、KAIROS、PROACTIVE 等）通过 `feature()` 函数控制，在此版本中全部禁用。

### 2. 快速路径优化

```typescript
// cli.tsx: 各种特殊标志的快速路径
if (args[0] === "--version") {
    console.log(`${MACRO.VERSION} (Claude Code)`);
    return;
}
```

- `--version`: 零模块加载
- `--daemon-worker`: 内部 worker 快速路径
- `remote-control`: 桥接模式快速路径
- `ps/logs/attach/kill`: 后台会话管理

### 3. 死代码消除 (Dead Code Elimination)

```typescript
// 使用条件 require 确保构建时消除未使用代码
const coordinatorModeModule = feature('COORDINATOR_MODE') 
  ? require('./coordinator/coordinatorMode.js') 
  : null;
```

### 4. 流式架构

- API 响应采用流式处理
- 使用 AsyncGenerator 实现渐进式输出
- 支持工具调用的交错执行

### 5. 模块化工具系统

每个工具是独立的模块，包含：
- `name`: 工具标识
- `description`: 功能描述
- `inputSchema`: JSON Schema 输入验证
- `call()`: 执行逻辑
- 可选的 React 组件用于渲染结果

## 数据流

```
User Input → REPL → QueryEngine → query.ts → API Client → Claude API
                                              ↓
Tool Results ← Tool Executor ← Tool Registry ← Streaming Response
```

## 状态管理

采用分层状态管理：

1. **AppState**: React 上下文 + Zustand-style store
2. **Bootstrap State**: 模块级单例（session ID、CWD等）
3. **ToolUseContext**: 工具执行上下文
4. **FileStateCache**: 文件状态缓存

## 构建系统

```bash
# 开发模式
bun run dev
# 等价于: bun run src/entrypoints/cli.tsx

# 生产构建
bun run build
# 等价于: bun build src/entrypoints/cli.tsx --outdir dist --target bun
```

构建输出 `dist/cli.js` (~25MB)，包含所有依赖的单文件可执行脚本。

## 运行时特性

- **Bun 特定 API**: 使用 `bun:bundle` 等 Bun 特有模块
- **Polyfill 注入**: 在 cli.tsx 顶部注入 `feature()` 和 `MACRO` 等全局变量
- **模块热替换**: 开发模式下支持快速重载
