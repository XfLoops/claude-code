# Claude Code 架构文档

## 文档概述

本文档详细描述 Claude Code CLI 工具的架构设计、工程实践和问题解决方案。本版本为逆向工程/反编译版本，保留了核心功能架构。

## 文档结构

| 文件 | 内容 |
|------|------|
| `00-overview.md` | 本文档 - 项目概述与结构说明 |
| `01-entry-and-bootstrap.md` | 入口与启动流程 |
| `02-tool-system.md` | 工具系统架构 |
| `03-query-loop.md` | 核心查询循环 |
| `04-ui-system.md` | UI 系统 (Ink + React) |
| `05-state-management.md` | 状态管理 |
| `06-api-layer.md` | API 层设计 |
| `07-engineering-practices.md` | 工程实践 |
| `08-problem-solutions.md` | 问题解决方案 |
| `09-performance-optimization.md` | 性能优化策略 |

## 项目统计

- **源代码文件**: ~2,794 个 (src/)
- **工具数量**: 40+
- **模块数量**: 500+
- **技术栈**: Bun + TypeScript + React + Ink
- **构建目标**: Bun 运行时
- **模块系统**: ESM

## 关键技术决策

1. **Bun 运行时**: 替代 Node.js，利用 Bun 的高性能 API 和构建工具
2. **React + Ink**: 终端 UI 使用 React 组件模型，通过 Ink 渲染到终端
3. **Feature Flag 系统**: 构建时死代码消除，实现多配置构建
4. **类型优先**: 使用 Zod 实现运行时类型安全

## 注意事项

- 本项目包含 ~1341 个 TypeScript 类型错误，来自反编译过程
- 这些错误不影响 Bun 运行时执行
- `feature()` 函数始终返回 `false`，禁用 Anthropic 内部功能

---

## 阅读建议

建议按照以下顺序阅读：

1. **01-entry-and-bootstrap.md** - 理解启动流程
2. **02-tool-system.md** - 理解工具系统架构
3. **03-query-loop.md** - 理解核心查询循环
4. **04-ui-system.md** - 理解 UI 渲染
5. **05-state-management.md** - 理解状态管理
6. **06-api-layer.md** - 理解 API 交互
7. **07-09** - 工程实践和优化策略

每个章节包含详细的代码示例和架构说明。
