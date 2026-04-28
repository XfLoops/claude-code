# 工具系统架构

## 文件位置

- `src/Tool.ts` - 工具类型定义和接口
- `src/tools.ts` - 工具注册表和加载逻辑
- `src/tools/<ToolName>/` - 各工具实现目录

## 1. 工具系统概述

Claude Code 的工具系统是核心架构组件，提供类型安全、权限控制、并发管理等功能。

### 1.1 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                      工具系统层级                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   UI 层      │  │  权限层      │  │  执行层      │      │
│  │              │  │              │  │              │      │
│  │ renderTool   │  │checkPermiss- │  │    call()    │      │
│  │ UseMessage   │  │   ions()     │  │              │      │
│  │              │  │              │  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│          │                │                │               │
│          └────────────────┼────────────────┘               │
│                           │                                │
│                    ┌──────────────┐                        │
│                    │   Tool 定义  │                        │
│                    │              │                        │
│                    │  name        │                        │
│                    │  inputSchema │                        │
│                    │  outputSchema│                        │
│                    │  ...         │                        │
│                    └──────────────┘                        │
│                           │                                │
│                    ┌──────────────┐                        │
│                    │  Tool 注册表 │                        │
│                    │              │                        │
│                    │ findToolBy   │                        │
│                    │   Name()     │                        │
│                    │ getTools()   │                        │
│                    └──────────────┘                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 工具分类

Claude Code 包含 40+ 个工具，按功能分类：

| 类别 | 工具示例 | 说明 |
|------|----------|------|
| **文件操作** | FileReadTool, FileWriteTool, FileEditTool | 读写编辑文件 |
| **Shell 执行** | BashTool, PowerShellTool | 执行 shell 命令 |
| **搜索** | GrepTool, GlobTool, WebSearchTool | 内容/文件搜索 |
| **网络** | WebFetchTool, WebBrowserTool | HTTP/Web 交互 |
| **系统** | ConfigTool, Task*Tool, Memory*Tool | 配置和状态管理 |
| **MCP** | MCPTool, ListMcpResourcesTool | MCP 服务器集成 |
| **高级** | AgentTool, SkillTool | 子代理和技能调用 |

## 2. Tool 类型定义

### 2.1 完整接口定义

**文件**: `src/Tool.ts`

```typescript
import { z } from 'zod';

// 工具输入/输出类型
export type AnyObject = z.ZodObject<any, any>;
export type ToolProgressData = Record<string, unknown>;

// 工具结果类型
export type ToolResult<T = unknown> = {
  result: T;
  newMessages?: Message[];
  metadata?: Record<string, unknown>;
};

// 工具使用上下文
export type ToolUseContext = {
  messageId: string;
  conversationId: string;
  abortSignal: AbortSignal;
  // ... 其他上下文
};

// 权限结果类型
export type PermissionResult =
  | { behavior: 'allow'; updatedInput?: Record<string, unknown> }
  | { behavior: 'deny'; reason: string }
  | { behavior: 'ask'; message: string; metadata?: Record<string, unknown> }
  | { behavior: 'auto-allowed'; classifierDecision: ClassifierDecision };

// 验证结果类型
export type ValidationResult =
  | { success: true }
  | { success: false; error: string };

// 核心 Tool 类型定义
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData
> = {
  // ==================== 基础元数据 ====================
  /** 工具唯一名称 */
  name: string;
  
  /** 别名（向后兼容） */
  aliases?: string[];
  
  /** 输入 Schema（Zod 类型） */
  inputSchema: Input;
  
  /** 输出 Schema（可选） */
  outputSchema?: z.ZodType<Output>;
  
  /** 进度数据类型 */
  progressSchema?: z.ZodType<P>;
  
  // ==================== 核心执行方法 ====================
  /** 
   * 执行工具
   * @param input - 已验证的输入
   * @param context - 工具使用上下文
   * @param canUseTool - 权限检查回调
   * @returns 工具执行结果
   */
  call(
    input: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    options?: ToolCallOptions
  ): Promise<ToolResult<Output>>;
  
  /**
   * 生成工具描述
   * 用于构建发送到 LLM 的工具定义
   */
  description(
    input: z.infer<Input>,
    options: DescriptionOptions
  ): Promise<string>;
  
  // ==================== 权限与安全 ====================
  /**
   * 检查权限
   * 在执行前调用，决定是否需要用户确认
   */
  checkPermissions(
    input: z.infer<Input>,
    context: ToolUseContext
  ): Promise<PermissionResult>;
  
  /**
   * 验证输入（可选）
   * 在执行前进行额外的输入验证
   */
  validateInput?(
    input: z.infer<Input>,
    context: ToolUseContext
  ): Promise<ValidationResult>;
  
  /**
   * 是否只读操作
   * 用于权限系统的默认决策
   */
  isReadOnly(input: z.infer<Input>): boolean;
  
  /**
   * 是否破坏性操作
   * 用于标记需要额外确认的操作
   */
  isDestructive?(input: z.infer<Input>): boolean;
  
  // ==================== 并发与行为控制 ====================
  /**
   * 是否线程安全
   * 决定是否可以与其他工具并行执行
   * 默认：false（fail-closed）
   */
  isConcurrencySafe(input: z.infer<Input>): boolean;
  
  /**
   * 中断行为
   * 当被用户中断时的行为
   * 'cancel' - 取消执行
   * 'block' - 阻塞直到完成
   */
  interruptBehavior?(): 'cancel' | 'block';
  
  /**
   * 最大执行时间（毫秒）
   */
  timeout?: number;
  
  // ==================== UI 渲染 ====================
  /**
   * 渲染工具使用消息
   * 在用户确认前显示
   */
  renderToolUseMessage(
    input: Partial<z.infer<Input>>,
    options: RenderOptions
  ): React.ReactNode;
  
  /**
   * 渲染工具结果消息（可选）
   * 在工具执行完成后显示
   */
  renderToolResultMessage?(
    content: Output,
    context: ToolResultContext
  ): React.ReactNode;
  
  // ==================== MCP/延迟加载 ====================
  /**
   * 是否应该延迟加载
   * 减少初始提示词大小
   */
  shouldDefer?: boolean;
  
  /**
   * 是否始终加载
   * 即使 shouldDefer 为 true 也加载
   */
  alwaysLoad?: boolean;
  
  /**
   * MCP 服务器信息
   */
  mcpInfo?: {
    serverName: string;
    toolName: string;
  };
  
  /**
   * 用户可编辑的输入字段
   * 在权限对话框中允许用户修改
   */
  userFacingFields?: (keyof z.infer<Input>)[];
};

// 权限检查函数类型
type CanUseToolFn = (
  tool: Tool,
  input: Record<string, unknown>
) => Promise<PermissionResult>;
```

### 2.2 设计原则

1. **Fail-Closed 默认**:
   - `isConcurrencySafe` 默认 `false`
   - `isReadOnly` 默认 `false`
   - 需要显式声明安全属性

2. **类型安全**:
   - 使用 Zod 进行运行时验证
   - TypeScript 类型推断

3. **职责分离**:
   - 权限检查、输入验证、执行、渲染分离

## 3. 工具创建

### 3.1 Builder 模式

**文件**: `src/Tool.ts`

```typescript
/**
 * 构建工具的工厂函数
 * 提供合理的默认值，简化工具定义
 */
export function buildTool<
  Input extends AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData
>(config: ToolConfig<Input, Output, P>): Tool<Input, Output, P> {
  return {
    // 基础元数据
    name: config.name,
    aliases: config.aliases,
    inputSchema: config.inputSchema,
    outputSchema: config.outputSchema,
    progressSchema: config.progressSchema,
    
    // 核心方法
    call: config.call,
    description: config.description ?? defaultDescription,
    
    // 权限与安全（Fail-Closed 默认）
    checkPermissions: config.checkPermissions ?? defaultCheckPermissions,
    validateInput: config.validateInput,
    isReadOnly: config.isReadOnly ?? (() => false),
    isDestructive: config.isDestructive,
    
    // 并发控制（Fail-Closed）
    isConcurrencySafe: config.isConcurrencySafe ?? (() => false),
    interruptBehavior: config.interruptBehavior,
    timeout: config.timeout,
    
    // UI 渲染
    renderToolUseMessage: config.renderToolUseMessage ?? defaultRender,
    renderToolResultMessage: config.renderToolResultMessage,
    
    // MCP/延迟加载
    shouldDefer: config.shouldDefer,
    alwaysLoad: config.alwaysLoad,
    mcpInfo: config.mcpInfo,
    userFacingFields: config.userFacingFields,
  };
}

// 默认实现
async function defaultCheckPermissions(): Promise<PermissionResult> {
  return { behavior: 'ask' };  // 默认询问用户
}

async function defaultDescription(input: unknown): Promise<string> {
  return JSON.stringify(input);
}

function defaultRender(input: unknown): React.ReactNode {
  return <Text>{JSON.stringify(input)}</Text>;
}
```

### 3.2 工具实现示例

**文件**: `src/tools/BashTool/tool.ts`

```typescript
import { z } from 'zod';
import { buildTool } from '../../Tool.js';
import { Text } from 'ink';

// 1. 定义输入 Schema
const BashInputSchema = z.object({
  command: z.string().describe('The shell command to execute'),
  timeout: z.number().optional().describe('Timeout in milliseconds'),
  cwd: z.string().optional().describe('Working directory'),
  env: z.record(z.string()).optional().describe('Environment variables'),
});

// 2. 定义输出类型
type BashOutput = {
  stdout: string;
  stderr: string;
  exitCode: number;
  duration: number;
};

// 3. 构建工具
export const BashTool = buildTool({
  name: 'Bash',
  aliases: ['bash', 'shell'],
  inputSchema: BashInputSchema,
  
  // ==================== 执行逻辑 ====================
  async call(input, context, canUseTool) {
    const { command, timeout = 60000, cwd, env } = input;
    
    // 执行命令
    const startTime = Date.now();
    const result = await executeCommand(command, {
      timeout,
      cwd,
      env,
      signal: context.abortSignal,
    });
    const duration = Date.now() - startTime;
    
    return {
      result: {
        stdout: result.stdout,
        stderr: result.stderr,
        exitCode: result.exitCode,
        duration,
      },
    };
  },
  
  // ==================== 权限控制 ====================
  async checkPermissions(input) {
    const { command } = input;
    
    // 检查危险命令
    if (isDangerousCommand(command)) {
      return {
        behavior: 'ask',
        message: `This command may be destructive: ${command}`,
      };
    }
    
    // 检查只读命令
    if (isReadOnlyCommand(command)) {
      return { behavior: 'allow' };
    }
    
    // 默认询问
    return {
      behavior: 'ask',
      message: `Execute command: ${command}`,
    };
  },
  
  isReadOnly: (input) => isReadOnlyCommand(input.command),
  
  isDestructive: (input) => isDangerousCommand(input.command),
  
  // ==================== 并发控制 ====================
  isConcurrencySafe: () => false,  // Shell 命令不安全
  interruptBehavior: () => 'cancel',  // 可取消
  timeout: 60000,  // 60秒超时
  
  // ==================== UI 渲染 ====================
  renderToolUseMessage(input) {
    return (
      <Text>
        <Text color="cyan">$</Text> {input.command}
      </Text>
    );
  },
  
  renderToolResultMessage(output) {
    return (
      <Box flexDirection="column">
        {output.stdout && <Text>{output.stdout}</Text>}
        {output.stderr && <Text color="red">{output.stderr}</Text>}
        <Text dimColor>Exit code: {output.exitCode}</Text>
      </Box>
    );
  },
  
  // ==================== 用户可编辑字段 ====================
  userFacingFields: ['command', 'timeout'],
});

// 辅助函数
function isDangerousCommand(command: string): boolean {
  const dangerous = ['rm', 'drop', 'delete', 'destroy', 'kill'];
  return dangerous.some(d => command.includes(d));
}

function isReadOnlyCommand(command: string): boolean {
  const readOnly = ['ls', 'cat', 'echo', 'pwd', 'grep', 'find'];
  return readOnly.some(c => command.startsWith(c));
}

async function executeCommand(
  command: string,
  options: { timeout: number; cwd?: string; env?: Record<string, string>; signal: AbortSignal }
): Promise<{ stdout: string; stderr: string; exitCode: number }> {
  // 实现命令执行逻辑
  // ...
}
```

## 4. 工具注册表

### 4.1 注册表结构

**文件**: `src/tools.ts`

```typescript
import { Tool } from './Tool.js';

// 核心工具（始终加载）
import { BashTool } from './tools/BashTool/index.js';
import { FileReadTool } from './tools/FileReadTool/index.js';
import { FileWriteTool } from './tools/FileWriteTool/index.js';
import { FileEditTool } from './tools/FileEditTool/index.js';
import { GlobTool } from './tools/GlobTool/index.js';
import { GrepTool } from './tools/GrepTool/index.js';
import { WebFetchTool } from './tools/WebFetchTool/index.js';

// 延迟加载工具
const DeferredTools = new Map<string, () => Promise<Tool>>([
  ['WebBrowserTool', async () => {
    const { WebBrowserTool } = await import('./tools/WebBrowserTool/index.js');
    return WebBrowserTool;
  }],
  ['MCPTool', async () => {
    const { MCPTool } = await import('./tools/MCPTool/index.js');
    return MCPTool;
  }],
]);

// 工具注册表
class ToolRegistry {
  private tools: Map<string, Tool> = new Map();
  private loadedDeferred: Set<string> = new Set();
  
  constructor() {
    // 注册核心工具
    this.register(BashTool);
    this.register(FileReadTool);
    this.register(FileWriteTool);
    this.register(FileEditTool);
    this.register(GlobTool);
    this.register(GrepTool);
    this.register(WebFetchTool);
  }
  
  register(tool: Tool): void {
    this.tools.set(tool.name, tool);
    
    // 注册别名
    if (tool.aliases) {
      for (const alias of tool.aliases) {
        this.tools.set(alias, tool);
      }
    }
  }
  
  async loadDeferredTool(name: string): Promise<Tool | undefined> {
    if (this.tools.has(name)) {
      return this.tools.get(name);
    }
    
    const loader = DeferredTools.get(name);
    if (loader && !this.loadedDeferred.has(name)) {
      const tool = await loader();
      this.register(tool);
      this.loadedDeferred.add(name);
      return tool;
    }
    
    return undefined;
  }
  
  get(name: string): Tool | undefined {
    return this.tools.get(name);
  }
  
  getAll(): Tool[] {
    return Array.from(this.tools.values());
  }
  
  /**
   * 搜索工具（支持别名和模糊匹配）
   */
  findByName(name: string): Tool | undefined {
    // 精确匹配
    if (this.tools.has(name)) {
      return this.tools.get(name);
    }
    
    // 大小写不敏感匹配
    const lowerName = name.toLowerCase();
    for (const [key, tool] of this.tools) {
      if (key.toLowerCase() === lowerName) {
        return tool;
      }
    }
    
    return undefined;
  }
}

// 全局注册表实例
export const toolRegistry = new ToolRegistry();

// 便捷函数
export function findToolByName(name: string): Tool | undefined {
  return toolRegistry.findByName(name);
}

export function getAllTools(): Tool[] {
  return toolRegistry.getAll();
}

export async function loadDeferredTool(name: string): Promise<Tool | undefined> {
  return toolRegistry.loadDeferredTool(name);
}
```

### 4.2 工具搜索机制

```typescript
/**
 * 工具搜索 - 确定需要加载的延迟工具
 * 
 * 使用轻量级模型或启发式搜索确定用户查询需要哪些延迟加载的工具
 */
export async function searchTools(
  query: string,
  availableTools: Tool[]
): Promise<Tool[]> {
  const deferredTools = availableTools.filter(t => t.shouldDefer && !t.alwaysLoad);
  const alwaysLoaded = availableTools.filter(t => !t.shouldDefer || t.alwaysLoad);
  
  if (deferredTools.length === 0) {
    return alwaysLoaded;
  }
  
  // 1. 关键词匹配
  const keywordMatches = deferredTools.filter(tool => {
    const keywords = getToolKeywords(tool);
    return keywords.some(kw => query.toLowerCase().includes(kw.toLowerCase()));
  });
  
  // 2. 启发式匹配（简单的模式匹配）
  const heuristicMatches = deferredTools.filter(tool => {
    if (tool.name === 'WebBrowserTool' && /(browser|website|url|page)/i.test(query)) {
      return true;
    }
    if (tool.name === 'WebSearchTool' && /(search|google|find)/i.test(query)) {
      return true;
    }
    return false;
  });
  
  // 3. 合并结果（去重）
  const matches = new Set([...keywordMatches, ...heuristicMatches]);
  
  return [...alwaysLoaded, ...matches];
}

function getToolKeywords(tool: Tool): string[] {
  // 从工具描述中提取关键词
  const desc = tool.description({} as any, {} as any);
  // 简化的关键词提取
  return tool.name.toLowerCase().split(/(?=[A-Z])/);
}
```

## 5. 工具执行流程

### 5.1 执行时序

```
用户输入/AI 请求
    │
    ▼
┌────────────────┐
│ 1. 输入验证     │
│    validateInput│
└────────────────┘
    │ success
    ▼
┌────────────────┐
│ 2. 权限检查     │
│    checkPermis- │
│    sions()      │
└────────────────┘
    │
    ├─► allow ───┐
    ├─► deny ────┼──► 返回错误
    ├─► ask ─────┘
    │              ▼
    │         ┌────────────────┐
    │         │ 3. 用户确认     │
    │         │    显示对话框   │
    │         └────────────────┘
    │              │
    └──────────────┘
                   │
                   ▼
            ┌────────────────┐
            │ 4. 执行工具     │
            │    call()       │
            └────────────────┘
                   │
                   ▼
            ┌────────────────┐
            │ 5. 结果渲染     │
            │    renderTool   │
            │    ResultMessage│
            └────────────────┘
```

### 5.2 并发执行

```typescript
/**
 * 并行执行工具
 * 
 * 只并行执行标记为 isConcurrencySafe 的工具
 */
export async function executeToolsConcurrently(
  toolCalls: ToolCall[],
  context: ToolUseContext
): Promise<ToolResult[]> {
  // 分离可并发和不可并发的工具
  const concurrentSafe: ToolCall[] = [];
  const sequential: ToolCall[] = [];
  
  for (const call of toolCalls) {
    const tool = findToolByName(call.name);
    if (tool?.isConcurrencySafe(call.input)) {
      concurrentSafe.push(call);
    } else {
      sequential.push(call);
    }
  }
  
  const results: ToolResult[] = [];
  
  // 1. 并行执行安全工具
  if (concurrentSafe.length > 0) {
    const concurrentResults = await Promise.all(
      concurrentSafe.map(call => executeSingleTool(call, context))
    );
    results.push(...concurrentResults);
  }
  
  // 2. 顺序执行不安全工具
  for (const call of sequential) {
    const result = await executeSingleTool(call, context);
    results.push(result);
  }
  
  return results;
}

async function executeSingleTool(
  call: ToolCall,
  context: ToolUseContext
): Promise<ToolResult> {
  const tool = findToolByName(call.name);
  if (!tool) {
    throw new Error(`Tool not found: ${call.name}`);
  }
  
  // 验证输入
  const validated = tool.inputSchema.parse(call.input);
  
  if (tool.validateInput) {
    const validation = await tool.validateInput(validated, context);
    if (!validation.success) {
      throw new Error(`Validation failed: ${validation.error}`);
    }
  }
  
  // 检查权限
  const permission = await tool.checkPermissions(validated, context);
  if (permission.behavior === 'deny') {
    throw new PermissionDeniedError(permission.reason);
  }
  
  // 执行
  return await tool.call(validated, context, canUseTool);
}
```

### 5.3 Streaming Tool Executor

```typescript
/**
 * 流式工具执行器
 * 
 * 支持增量返回工具结果，适用于长时间运行的工具
 */
class StreamingToolExecutor {
  private pendingTools: Map<string, ToolExecution> = new Map();
  private completedResults: ToolExecutionResult[] = [];
  
  /**
   * 添加工具执行
   */
  addTool(toolBlock: ToolUseBlock, assistantMessage: AssistantMessage): void {
    const execution = this.executeTool(toolBlock, assistantMessage);
    this.pendingTools.set(toolBlock.id, execution);
  }
  
  /**
   * 获取已完成的结果（同步）
   */
  getCompletedResults(): ToolExecutionResult[] {
    return this.completedResults.splice(0);  // 取出并清空
  }
  
  /**
   * 获取剩余结果（异步流式）
   */
  async *getRemainingResults(): AsyncGenerator<ToolExecutionResult> {
    while (this.pendingTools.size > 0) {
      // 等待任意一个工具完成
      const completed = await Promise.race(
        Array.from(this.pendingTools.entries()).map(async ([id, execution]) => {
          try {
            const result = await execution.promise;
            return { id, result, error: null };
          } catch (error) {
            return { id, result: null, error };
          }
        })
      );
      
      this.pendingTools.delete(completed.id);
      yield completed;
    }
  }
  
  /**
   * 放弃未完成的执行
   */
  discard(): void {
    for (const [id, execution] of this.pendingTools) {
      execution.abort();
    }
    this.pendingTools.clear();
  }
  
  private executeTool(
    toolBlock: ToolUseBlock,
    assistantMessage: AssistantMessage
  ): ToolExecution {
    const abortController = new AbortController();
    
    const promise = this.runTool(toolBlock, assistantMessage, abortController.signal);
    
    return {
      promise,
      abort: () => abortController.abort(),
    };
  }
  
  private async runTool(
    toolBlock: ToolUseBlock,
    assistantMessage: AssistantMessage,
    signal: AbortSignal
  ): Promise<ToolResult> {
    const tool = findToolByName(toolBlock.name);
    if (!tool) {
      throw new Error(`Tool not found: ${toolBlock.name}`);
    }
    
    const context: ToolUseContext = {
      messageId: toolBlock.id,
      conversationId: assistantMessage.conversationId,
      abortSignal: signal,
    };
    
    return await tool.call(toolBlock.input, context, canUseTool);
  }
}
```

## 6. 权限系统

### 6.1 权限决策流程

```
┌──────────────────────────────────────────────┐
│              权限决策流程                       │
├──────────────────────────────────────────────┤
│                                              │
│  用户输入                                      │
│     │                                        │
│     ▼                                        │
│  ┌──────────────┐                            │
│  │ 1. 检查全局   │                            │
│  │    权限设置   │                            │
│  └──────────────┘                            │
│     │                                        │
│     ├─► 全局禁止 ────► deny                  │
│     ├─► 全局允许 ────► allow                 │
│     └─► 需要检查 ────► ┌──────────────┐      │
│                      │ 2. 调用工具     │      │
│                      │    checkPermis-│      │
│                      │    sions()      │      │
│                      └──────────────┘      │
│                         │                   │
│                         ├─► allow           │
│                         ├─► deny            │
│                         └─► ask             │
│                            │                │
│                            ▼                │
│                      ┌──────────────┐      │
│                      │ 3. 显示确认   │      │
│                      │    对话框     │      │
│                      └──────────────┘      │
│                         │                   │
│                         ├─► 用户允许       │
│                         └─► 用户拒绝       │
│                                              │
└──────────────────────────────────────────────┘
```

### 6.2 权限配置

```typescript
// 权限模式
enum PermissionMode {
  /** 所有工具都需要确认 */
  ASK_EVERY_TIME = 'ask_every_time',
  
  /** 只读工具自动允许，其他需要确认 */
  AUTO_ALLOW_READ_ONLY = 'auto_allow_read_only',
  
  /** 基于分类器自动决策 */
  AUTO_ALLOW = 'auto_allow',
}

// 权限配置
type PermissionConfig = {
  mode: PermissionMode;
  
  // 白名单（始终允许）
  allowList: string[];
  
  // 黑名单（始终禁止）
  denyList: string[];
  
  // 自动允许阈值（分类器分数）
  autoAllowThreshold: number;
};

// 全局权限检查函数
async function canUseTool(
  tool: Tool,
  input: Record<string, unknown>,
  config: PermissionConfig
): Promise<PermissionResult> {
  // 1. 检查黑名单
  if (config.denyList.includes(tool.name)) {
    return { behavior: 'deny', reason: 'Tool is blacklisted' };
  }
  
  // 2. 检查白名单
  if (config.allowList.includes(tool.name)) {
    return { behavior: 'allow' };
  }
  
  // 3. 根据模式决策
  switch (config.mode) {
    case 'ask_every_time':
      return { behavior: 'ask', message: `Use ${tool.name}?` };
      
    case 'auto_allow_read_only':
      if (tool.isReadOnly(input)) {
        return { behavior: 'allow' };
      }
      return { behavior: 'ask', message: `Use ${tool.name}?` };
      
    case 'auto_allow':
      // 使用分类器决策
      const decision = await classifier.classify(tool, input);
      if (decision.score >= config.autoAllowThreshold) {
        return { behavior: 'auto-allowed', classifierDecision: decision };
      }
      return { behavior: 'ask', message: decision.reason };
      
    default:
      return { behavior: 'ask', message: `Use ${tool.name}?` };
  }
}
```

### 6.3 权限对话框

```typescript
// 权限确认对话框组件
function PermissionDialog({
  tool,
  input,
  onAllow,
  onDeny,
  onEdit,
}: PermissionDialogProps) {
  const [editedInput, setEditedInput] = useState(input);
  
  // 渲染工具使用消息
  const toolMessage = tool.renderToolUseMessage(editedInput, {});
  
  return (
    <Box flexDirection="column">
      <Text>Claude wants to use the {tool.name} tool:</Text>
      <Box marginLeft={2}>{toolMessage}</Box>
      
      {/* 可编辑字段 */}
      {tool.userFacingFields?.map(field => (
        <EditableField
          key={field}
          name={field}
          value={editedInput[field]}
          onChange={(value) => setEditedInput({ ...editedInput, [field]: value })}
        />
      ))}
      
      <Box>
        <Button onPress={() => onAllow(editedInput)}>Allow</Button>
        <Button onPress={onDeny}>Deny</Button>
      </Box>
    </Box>
  );
}
```

## 7. 特殊工具类型

### 7.1 MCP 工具

```typescript
// MCP (Model Context Protocol) 工具
// 动态从 MCP 服务器加载

const MCPTool = buildTool({
  name: 'MCP',
  inputSchema: z.object({
    serverName: z.string(),
    toolName: z.string(),
    input: z.record(z.unknown()),
  }),
  
  mcpInfo: {
    serverName: '{{dynamic}}',
    toolName: '{{dynamic}}',
  },
  
  async call(input, context) {
    const { serverName, toolName, input: toolInput } = input;
    
    // 调用 MCP 服务器
    const server = await getMCPClient(serverName);
    const result = await server.callTool(toolName, toolInput);
    
    return { result };
  },
  
  // MCP 工具默认延迟加载
  shouldDefer: true,
});
```

### 7.2 Agent 工具

```typescript
// 子 Agent 工具
// 派生独立的 QueryEngine 实例

const AgentTool = buildTool({
  name: 'Agent',
  inputSchema: z.object({
    task: z.string().describe('Task description for the agent'),
    files: z.array(z.string()).optional(),
    instructions: z.string().optional(),
  }),
  
  async call(input, context) {
    const { task, files, instructions } = input;
    
    // 创建子引擎
    const agentEngine = new QueryEngine({
      parentContext: context,
      instructions,
    });
    
    // 执行子任务
    const stream = agentEngine.submitMessage(task);
    const results = [];
    
    for await (const message of stream) {
      results.push(message);
    }
    
    return {
      result: {
        output: formatResults(results),
        files: files,
      },
    };
  },
  
  isConcurrencySafe: () => true,  // Agent 是独立的
  interruptBehavior: () => 'block',  // 阻塞直到完成
});
```

## 8. 总结

工具系统的设计体现了以下架构原则：

1. **类型安全**: 使用 Zod 实现运行时验证和 TypeScript 类型
2. **Fail-Closed**: 默认保守，需要显式声明安全属性
3. **职责分离**: 权限、验证、执行、渲染分离
4. **灵活扩展**: Builder 模式 + 延迟加载支持
5. **并发控制**: 精细的并发安全控制
6. **用户友好**: 可编辑字段和清晰的权限对话框
