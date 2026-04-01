# Claude Code CLI 源码分析文档

本目录包含 Claude Code CLI 项目的详细源码分析文档。

## 文档目录

### 1. [项目概述](00-summary.md)
项目总结、核心特性、架构亮点、开发指南和学习资源。

### 2. [架构概述](01-architecture-overview.md)
- 项目简介与技术栈
- 核心架构图
- 关键设计原则
- 数据流图
- 构建系统

### 3. [入口与启动流程](02-entry-and-bootstrap.md)
- 启动流程图
- cli.tsx 详解（polyfill、快速路径）
- init.ts 详解（初始化流程）
- main.tsx 详解（CLI 定义）
- 性能优化策略
- 错误处理

### 4. [核心查询循环](03-core-query-loop.md)
- QueryEngine 详解
- query.ts 详解（核心查询函数）
- 流式处理架构
- 工具执行流程
- 压缩与上下文管理
- Token 预算管理
- 错误恢复机制
- 依赖注入

### 5. [工具系统](04-tool-system.md)
- Tool.ts 核心接口
- tools.ts 注册表
- 核心工具详解（BashTool、FileEditTool、AgentTool）
- 工具构建辅助函数
- 工具权限系统
- 工具执行流程
- 工具结果存储

### 6. [UI 系统 - Ink 框架](05-ui-system-ink.md)
- Ink 架构概述
- 核心组件（Box、Text）
- 自定义 Hooks（useInput、useTerminalSize、useSearchHighlight）
- REPL 屏幕组件
- 消息渲染组件
- 虚拟列表
- 主题系统
- 渲染输出
- 性能优化

### 7. [状态管理](06-state-management.md)
- 分层状态架构
- AppState 详解
- Store 实现（Zustand-style）
- React 集成（hooks）
- Bootstrap State
- ToolUseContext
- FileStateCache
- 状态持久化
- 状态变更监听

### 8. [API 层](07-api-layer.md)
- claude.ts 主 API
- 多提供商支持
- 重试机制
- 错误处理
- 流事件处理
- 令牌使用跟踪
- 启动数据获取
- 调试与日志
- 模型管理

### 9. [MCP 系统](08-mcp-system.md)
- MCP 架构
- MCP 核心类型
- MCP 客户端管理
- MCP 工具封装
- MCP 配置管理
- OAuth 认证
- Elicitation 流程
- 官方 MCP 注册表

### 10. [上下文构建](09-context-building.md)
- 系统上下文构建
- 用户上下文
- CLAUDE.md 加载系统
- 系统提示生成
- 系统提示各部分详解
- 提示缓存优化
- 上下文统计
- 调试与诊断

### 11. [记忆系统](11-memory-system.md)
- 系统架构与双轨设计
- 自动记忆系统 (Auto-Memory)
- CLAUDE.md 嵌套规则系统
- 四种记忆类型详解
- 相关性选择算法
- 记忆注入与附件系统
- 路径安全与限制
- 配置与最佳实践

### 12. [权限系统](10-permission-system.md)
- 权限模式类型
- 权限上下文
- 权限检查流程
- 自动分类器模式
- 文件系统权限
- 拒绝跟踪
- 权限更新
- UI 权限请求

### 13. [Token 优化策略](13-token-optimization.md)
- 提示缓存 (Prompt Caching)
- 上下文压缩 (Context Compaction)
- 微压缩 (Micro-Compaction)
- 工具结果管理
- Token 估算与预算管理
- 记忆系统 Token 优化
- 1M 上下文窗口支持
- 成本节省分析

### 14. [功能开关与配置](14-feature-flags-and-config.md)
- 构建时功能开关 (feature())
- 环境变量全览
- GrowthBook 动态配置
- Settings.json 配置
- Beta Headers
- 配置加载流程
- 最佳实践与安全

## 快速开始

### 阅读建议
1. **新手**: 先阅读 [架构概述](01-architecture-overview.md)，了解整体架构
2. **开发者**: 按顺序阅读 [入口与启动](02-entry-and-bootstrap.md) → [核心查询循环](03-core-query-loop.md) → [工具系统](04-tool-system.md)
3. **深入**: 根据兴趣阅读特定模块（UI、MCP、权限等）

### 关键概念
- **QueryEngine**: 对话生命周期管理
- **ToolUseContext**: 工具执行上下文
- **AppState**: 全局应用状态
- **MCP**: Model Context Protocol
- **Ink**: 终端 UI 框架
- **Memory System**: 双轨记忆系统（Auto-Memory + CLAUDE.md）
- **Prompt Caching**: 提示缓存优化（global/org scope）
- **Context Compaction**: 上下文压缩（自动/手动）
- **Micro-Compaction**: 微压缩（工具结果清理）
- **Feature Flags**: 构建时功能开关（feature()）
- **GrowthBook**: 动态配置系统

### 常用文件路径
```
src/
├── entrypoints/cli.tsx    # 入口点
├── main.tsx               # CLI 定义
├── query.ts               # 核心查询循环
├── QueryEngine.ts         # 查询引擎
├── Tool.ts                # 工具接口
├── tools.ts               # 工具注册表
├── context.ts             # 上下文构建
├── state/AppState.tsx     # 状态管理
├── services/api/claude.ts # API 层
├── memdir/                # 记忆系统
│   ├── memdir.ts          # 核心记忆函数
│   ├── memoryTypes.ts     # 记忆类型定义
│   ├── paths.ts           # 路径解析
│   ├── memoryScan.ts      # 记忆扫描
│   ├── findRelevantMemories.ts  # 相关性选择
│   └── memoryAge.ts       # 记忆新鲜度
└── tools/                 # 各工具实现
```

## 图表说明

文档中使用的主要图表类型：

### 架构图
```
┌─────────────┐
│   Module    │
└─────────────┘
```

### 流程图
```
Step 1 → Step 2 → Step 3
```

### 数据流
```
Source → [Process] → Destination
```

## 术语表

| 术语 | 说明 |
|------|------|
| MCP | Model Context Protocol，模型上下文协议 |
| REPL | Read-Eval-Print Loop，交互式环境 |
| Tool | 工具，可被 Claude 调用的功能单元 |
| Query | 查询，一次完整的对话交互 |
| Turn | 轮次，用户输入 + AI 响应 |
| Compact | 压缩，上下文窗口管理 |
| Stream | 流，渐进式数据传输 |
| Auto-Memory | 自动记忆系统，AI 持久化记忆 |
| MEMORY.md | 记忆索引入口文件 |
| CLAUDE.md | 项目规则配置文件 |
| Prompt Caching | 提示缓存，API 级别的缓存优化 |
| Context Compaction | 上下文压缩，对话历史摘要 |
| Micro-Compaction | 微压缩，细粒度工具结果清理 |
| Token Budget | Token 预算，上下文窗口管理 |
| 1M Context | 1M Token 上下文窗口支持 |
| Feature Flag | 功能开关，构建时条件编译 |
| GrowthBook | 动态配置系统，远程功能开关 |

## 更新日志

### 2024-04-01
- 创建初始文档集
- 完成 11 个核心模块分析

### 2024-04-01
- 新增记忆系统文档 [记忆系统详解](11-memory-system.md)
- 覆盖 Auto-Memory 和 CLAUDE.md 双轨架构

### 2024-04-01
- 新增功能开关文档 [功能开关与配置](14-feature-flags-and-config.md)
- 全面梳理构建时功能开关、环境变量、GrowthBook 配置

## 参考资料

- [Bun 文档](https://bun.sh/docs)
- [Ink 文档](https://github.com/vadimdemedes/ink)
- [React 文档](https://react.dev)
- [MCP 规范](https://modelcontextprotocol.io)

---

**注意**: 本文档基于 Claude Code CLI 的逆向工程版本分析，仅供学习研究使用。
