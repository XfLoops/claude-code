# 核心查询循环详解

## 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                      Query 架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────────┐   │
│  │   User      │────→│  QueryEngine │────→│    query.ts     │   │
│  │   Input     │     │  (生命周期)   │     │  (核心循环)      │   │
│  └─────────────┘     └─────────────┘     └─────────────────┘   │
│                              │                      │           │
│                              │                      ▼           │
│                              │           ┌─────────────────┐   │
│                              │           │   API Client    │   │
│                              │           │  (claude.ts)    │   │
│                              │           └─────────────────┘   │
│                              │                      │           │
│                              ▼                      ▼           │
│                     ┌─────────────┐     ┌─────────────────┐   │
│                     │   State     │←────│  Tool Executor  │   │
│                     │  (AppState) │     │  (runTools)     │   │
│                     └─────────────┘     └─────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## QueryEngine.ts 详解

### QueryEngine 职责

`QueryEngine` 管理单个对话的生命周期和状态，是 REPL 和 SDK 路径的共享核心。

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private totalUsage: NonNullableUsage
  private readFileState: FileStateCache
  private discoveredSkillNames = new Set<string>()
  
  // 核心方法
  async submitMessage(userInput: UserMessage): Promise<void>
  async *streamQuery(): AsyncGenerator<StreamEvent | Message>
  abort(): void
}
```

### QueryEngineConfig

```typescript
export type QueryEngineConfig = {
  cwd: string                    // 工作目录
  tools: Tools                   // 可用工具列表
  commands: Command[]            // 斜杠命令
  mcpClients: MCPServerConnection[]  // MCP 连接
  agents: AgentDefinition[]      // Agent 定义
  canUseTool: CanUseToolFn       // 工具权限检查
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  readFileCache: FileStateCache
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  jsonSchema?: Record<string, unknown>
  verbose?: boolean
  // ... 更多配置
}
```

### 消息处理流程

```typescript
async submitMessage(userInput: UserMessage): Promise<void> {
  // 1. 处理用户输入
  const processedInput = await processUserInput({
    text: userInput,
    commands: this.config.commands,
    /* ... */
  })
  
  // 2. 添加到消息历史
  this.mutableMessages.push(processedInput.message)
  
  // 3. 启动查询流
  const stream = this.streamQuery()
  
  // 4. 消费流并更新状态
  for await (const event of stream) {
    this.handleStreamEvent(event)
  }
}
```

## query.ts 详解

### 核心查询函数

```typescript
export async function* query(
  params: QueryParams
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  
  // 通知命令生命周期完成
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  
  return terminal
}
```

### QueryParams

```typescript
export type QueryParams = {
  messages: Message[]                    // 消息历史
  systemPrompt: SystemPrompt             // 系统提示
  userContext: { [k: string]: string }   // 用户上下文
  systemContext: { [k: string]: string } // 系统上下文
  canUseTool: CanUseToolFn               // 工具权限检查
  toolUseContext: ToolUseContext         // 工具执行上下文
  fallbackModel?: string                 // 备用模型
  querySource: QuerySource               // 查询来源
  maxOutputTokensOverride?: number       // 输出令牌限制
  maxTurns?: number                      // 最大轮次
  skipCacheWrite?: boolean               // 跳过缓存写入
  taskBudget?: { total: number }         // 任务预算
  deps?: QueryDeps                       // 依赖注入
}
```

### 查询循环状态

```typescript
type State = {
  messages: Message[]                                    // 消息列表
  toolUseContext: ToolUseContext                         // 工具上下文
  autoCompactTracking: AutoCompactTrackingState | undefined  // 自动压缩跟踪
  maxOutputTokensRecoveryCount: number                   // 恢复计数
  hasAttemptedReactiveCompact: boolean                   // 是否尝试反应式压缩
  maxOutputTokensOverride: number | undefined            // 输出限制覆盖
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined  // 待处理摘要
  stopHookActive: boolean | undefined                    // Stop hook 状态
  turnCount: number                                      // 轮次计数
  transition: Continue | undefined                      // 继续原因
}
```

### 查询循环详解

```typescript
async function* queryLoop(
  params: QueryParams,
  consumedCommandUuids: string[]
): AsyncGenerator<StreamEvent | RequestStartEvent | Message, Terminal> {
  // 1. 解构参数
  const {
    systemPrompt,
    userContext,
    systemContext,
    canUseTool,
    fallbackModel,
    querySource,
    maxTurns,
    skipCacheWrite,
  } = params
  
  // 2. 初始化依赖
  const deps = params.deps ?? productionDeps()
  
  // 3. 初始化状态
  let state: State = {
    messages: params.messages,
    toolUseContext: params.toolUseContext,
    maxOutputTokensOverride: params.maxOutputTokensOverride,
    autoCompactTracking: undefined,
    stopHookActive: undefined,
    maxOutputTokensRecoveryCount: 0,
    hasAttemptedReactiveCompact: false,
    turnCount: 1,
    pendingToolUseSummary: undefined,
    transition: undefined,
  }
  
  // 4. 创建预算跟踪器
  const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null
  
  // 5. 任务预算跟踪
  let taskBudgetRemaining: number | undefined = undefined
  
  // 6. 构建查询配置（一次）
  const config = buildQueryConfig()
  
  // 7. 启动内存预取
  using memoryPrefetch = startRelevantMemoryPrefetch(/* ... */)
  
  // 8. 主循环
  while (true) {
    // 检查轮次限制
    if (maxTurns && state.turnCount > maxTurns) {
      return { outcome: 'turns_exceeded' }
    }
    
    // 构建 API 请求
    const request = buildAPIRequest(state, systemPrompt, userContext, systemContext)
    
    // 调用 API
    const response = yield* callClaudeAPI(request, config)
    
    // 处理响应
    const result = yield* handleAPIResponse(response, state, canUseTool)
    
    // 处理结果
    switch (result.outcome) {
      case 'complete':
        return result
      case 'continue':
        state = result.nextState
        continue
      case 'error':
        yield* handleError(result.error, state)
        continue
    }
  }
}
```

## 流式处理架构

### 流事件类型

```typescript
type StreamEvent =
  | { type: 'text'; text: string }
  | { type: 'thinking'; thinking: string }
  | { type: 'tool_use'; toolUse: ToolUseBlock }
  | { type: 'tool_result'; toolResult: ToolResultBlockParam }
  | { type: 'error'; error: APIError }
  | { type: 'usage'; usage: BetaUsage }
  | { type: 'stop'; reason: BetaStopReason }
```

### 流处理流程

```typescript
async function* callClaudeAPI(
  request: BetaMessageStreamParams,
  config: QueryConfig
): AsyncGenerator<StreamEvent, void> {
  // 1. 发送请求
  const stream = await anthropic.messages.create({
    ...request,
    stream: true,
  })
  
  // 2. 处理流事件
  for await (const event of stream) {
    switch (event.type) {
      case 'content_block_delta':
        if (event.delta.type === 'text_delta') {
          yield { type: 'text', text: event.delta.text }
        }
        break
      case 'content_block_start':
        if (event.content_block.type === 'thinking') {
          yield { type: 'thinking', thinking: event.content_block.thinking }
        }
        break
      case 'message_stop':
        yield { type: 'stop', reason: event.message.stop_reason }
        break
    }
  }
}
```

## 工具执行流程

### 工具调用检测

```typescript
function* handleAPIResponse(
  response: AsyncGenerator<StreamEvent>,
  state: State,
  canUseTool: CanUseToolFn
): Generator<StreamEvent | Message, HandleResult> {
  const toolUseBlocks: ToolUseBlock[] = []
  const contentBlocks: ContentBlock[] = []
  
  for await (const event of response) {
    yield event
    
    if (event.type === 'tool_use') {
      toolUseBlocks.push(event.toolUse)
    } else if (event.type === 'text') {
      contentBlocks.push({ type: 'text', text: event.text })
    }
  }
  
  // 如果有工具调用
  if (toolUseBlocks.length > 0) {
    // 执行工具
    const results = yield* executeTools(toolUseBlocks, state.toolUseContext, canUseTool)
    
    // 构建下一轮消息
    const nextMessages = buildNextMessages(state.messages, contentBlocks, toolUseBlocks, results)
    
    return {
      outcome: 'continue',
      nextState: {
        ...state,
        messages: nextMessages,
        turnCount: state.turnCount + 1,
      }
    }
  }
  
  // 无工具调用，完成
  return { outcome: 'complete' }
}
```

### 工具执行器

```typescript
async function* executeTools(
  toolUses: ToolUseBlock[],
  context: ToolUseContext,
  canUseTool: CanUseToolFn
): AsyncGenerator<ProgressMessage, ToolResult[]> {
  const results: ToolResult[] = []
  
  for (const toolUse of toolUses) {
    // 查找工具
    const tool = findToolByName(context.options.tools, toolUse.name)
    if (!tool) {
      results.push({ error: `Tool ${toolUse.name} not found` })
      continue
    }
    
    // 权限检查
    const permission = await canUseTool(tool, toolUse.input, context)
    if (!permission.allowed) {
      results.push({ error: `Permission denied: ${permission.reason}` })
      continue
    }
    
    // 执行工具
    try {
      const result = await tool.call(toolUse.input, context)
      results.push(result)
    } catch (error) {
      results.push({ error: errorMessage(error) })
    }
  }
  
  return results
}
```

## 压缩与上下文管理

### 自动压缩

```typescript
function shouldAutoCompact(state: State): boolean {
  if (!isAutoCompactEnabled()) return false
  
  const tokenCount = tokenCountWithEstimation(state.messages)
  const threshold = getAutoCompactThreshold()
  
  return tokenCount > threshold
}

async function* autoCompact(
  state: State
): AsyncGenerator<CompactProgressEvent | Message, State> {
  // 1. 触发压缩钩子
  yield { type: 'hooks_start', hookType: 'pre_compact' }
  
  // 2. 执行压缩
  const compactResult = await buildPostCompactMessages({
    messages: state.messages,
    /* ... */
  })
  
  // 3. 更新状态
  return {
    ...state,
    messages: compactResult.messages,
    autoCompactTracking: compactResult.tracking,
  }
}
```

### 微压缩 (Microcompact)

```typescript
function* microCompact(state: State): Generator<Message, State> {
  // 快速压缩小片段
  const boundaryMessage = createMicrocompactBoundaryMessage({
    preservedCount: state.messages.length - MICROCOMPACT_THRESHOLD,
  })
  
  return {
    ...state,
    messages: [
      ...state.messages.slice(0, MICROCOMPACT_THRESHOLD),
      boundaryMessage,
    ],
  }
}
```

## Token 预算管理

### 预算跟踪

```typescript
function createBudgetTracker() {
  let totalTokens = 0
  let inputTokens = 0
  let outputTokens = 0
  
  return {
    addUsage(usage: BetaUsage) {
      totalTokens += usage.input_tokens + usage.output_tokens
      inputTokens += usage.input_tokens
      outputTokens += usage.output_tokens
    },
    getRemaining(): number {
      return MAX_TOKENS_PER_TURN - totalTokens
    },
    shouldContinue(): boolean {
      return totalTokens < MAX_TOKENS_PER_TURN && outputTokens < MAX_OUTPUT_TOKENS
    },
  }
}
```

### 预算检查

```typescript
function checkTokenBudget(
  state: State,
  budgetTracker: BudgetTracker | null
): { shouldContinue: boolean; reason?: string } {
  if (!budgetTracker) return { shouldContinue: true }
  
  const remaining = budgetTracker.getRemaining()
  if (remaining <= 0) {
    return { shouldContinue: false, reason: 'token_budget_exceeded' }
  }
  
  const outputTokens = getTurnOutputTokens()
  if (outputTokens > ESCALATED_MAX_TOKENS) {
    return { shouldContinue: false, reason: 'output_tokens_exceeded' }
  }
  
  return { shouldContinue: true }
}
```

## 错误恢复机制

### Max Output Tokens 恢复

```typescript
function isWithheldMaxOutputTokens(
  msg: Message | StreamEvent | undefined
): msg is AssistantMessage {
  return msg?.type === 'assistant' && msg.apiError === 'max_output_tokens'
}

async function* handleMaxOutputTokensError(
  state: State,
  error: APIError
): AsyncGenerator<Message, State> {
  if (state.maxOutputTokensRecoveryCount >= MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
    throw new Error('Max output tokens recovery limit exceeded')
  }
  
  // 尝试反应式压缩
  if (!state.hasAttemptedReactiveCompact && reactiveCompact) {
    const compacted = await reactiveCompact.compact(state.messages)
    return {
      ...state,
      messages: compacted,
      hasAttemptedReactiveCompact: true,
      maxOutputTokensRecoveryCount: state.maxOutputTokensRecoveryCount + 1,
    }
  }
  
  // 降级到更小模型
  return {
    ...state,
    fallbackModel: getSmallerModel(state.toolUseContext.options.mainLoopModel),
    maxOutputTokensRecoveryCount: state.maxOutputTokensRecoveryCount + 1,
  }
}
```

### API 错误处理

```typescript
function categorizeRetryableAPIError(error: APIError): {
  retryable: boolean
  message: string
  delayMs?: number
} {
  if (error.status === 429) {
    return { retryable: true, message: 'Rate limited', delayMs: 1000 }
  }
  if (error.status >= 500) {
    return { retryable: true, message: 'Server error', delayMs: 500 }
  }
  if (error instanceof APIConnectionTimeoutError) {
    return { retryable: true, message: 'Connection timeout', delayMs: 2000 }
  }
  return { retryable: false, message: error.message }
}
```

## 依赖注入

### QueryDeps 接口

```typescript
export type QueryDeps = {
  // API 调用
  callClaudeAPI: (
    params: BetaMessageStreamParams,
    options: APIOptions
  ) => AsyncGenerator<StreamEvent>
  
  // 工具执行
  runTools: (
    toolUses: ToolUseBlock[],
    context: ToolUseContext
  ) => AsyncGenerator<ProgressMessage, ToolResult[]>
  
  // 压缩
  compact: (messages: Message[]) => Promise<CompactResult>
  
  // Token 计数
  countTokens: (content: MessageContent) => Promise<number>
  
  // 钩子执行
  executeHooks: (
    type: HookType,
    context: HookContext
  ) => Promise<HookResult>
}

// 生产依赖
export function productionDeps(): QueryDeps {
  return {
    callClaudeAPI: streamingAPICall,
    runTools: toolOrchestration,
    compact: buildPostCompactMessages,
    countTokens: tokenCountWithEstimation,
    executeHooks: executeStopFailureHooks,
  }
}
```

### 测试依赖

```typescript
export function mockDeps(overrides: Partial<QueryDeps> = {}): QueryDeps {
  return {
    callClaudeAPI: mockStreamingCall,
    runTools: mockToolExecution,
    compact: mockCompact,
    countTokens: mockTokenCount,
    executeHooks: mockHooks,
    ...overrides,
  }
}
```

## 性能优化

### 内存预取

```typescript
function startRelevantMemoryPrefetch(context: ToolUseContext): MemoryPrefetchHandle {
  // 分析当前消息，预取相关记忆文件
  const relevantPaths = analyzeMessagesForMemoryPaths(context.messages)
  
  // 并行预取
  const promises = relevantPaths.map(path => 
    prefetchMemoryFile(path).catch(() => null)
  )
  
  return {
    async settledAt(): Promise<void> {
      await Promise.all(promises)
    },
    [Symbol.dispose](): void {
      // 清理
    },
  }
}
```

### 流式工具执行

```typescript
class StreamingToolExecutor {
  private activeExecutions = new Map<string, ToolExecution>()
  
  async executeStreaming(
    toolUse: ToolUseBlock,
    context: ToolUseContext
  ): AsyncGenerator<ToolProgressEvent, ToolResult> {
    const executionId = randomUUID()
    
    // 启动执行
    const execution = this.startExecution(toolUse, context)
    this.activeExecutions.set(executionId, execution)
    
    try {
      // 流式输出进度
      for await (const progress of execution.progressStream) {
        yield { type: 'progress', progress }
      }
      
      // 返回结果
      const result = await execution.result
      return result
    } finally {
      this.activeExecutions.delete(executionId)
    }
  }
}
```
