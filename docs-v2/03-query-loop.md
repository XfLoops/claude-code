# 核心查询循环

## 文件位置

- `src/query.ts` - 核心查询循环实现
- `src/QueryEngine.ts` - 查询引擎封装
- `src/query/config.ts` - 查询配置
- `src/query/tokenBudget.ts` - Token 预算管理
- `src/query/transitions.ts` - 状态转换逻辑

## 1. 查询循环概述

核心查询循环是 Claude Code 的核心组件，负责管理对话流程、处理流式响应、执行工具调用。

### 1.1 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        查询循环架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                                               │
│  │  QueryEngine │────────────────────────────────────┐          │
│  │              │                                    │          │
│  │ - 会话管理   │                                    │          │
│  │ - 系统提示   │                                    │          │
│  │ - 结果格式化 │                                    │          │
│  └──────────────┘                                    │          │
│          │                                           │          │
│          │ submitMessage()                           │          │
│          ▼                                           │          │
│  ┌──────────────┐    ┌──────────────┐               │          │
│  │   query()    │───▶│   queryLoop  │               │          │
│  │              │    │   (Generator)│               │          │
│  │ - 预处理     │    │              │               │          │
│  │ - 循环调用   │    │ while (true) │               │          │
│  └──────────────┘    └──────────────┘               │          │
│                               │                      │          │
│                               ▼                      │          │
│  ┌──────────────────────────────────────────────┐   │          │
│  │                   迭代处理                      │   │          │
│  │                                                │   │          │
│  │  1. 预处理消息 (compact, snip, collapse)       │   │          │
│  │  2. API 调用与流式处理                         │   │          │
│  │  3. 工具执行                                   │   │          │
│  │  4. 状态转换                                   │   │          │
│  │                                                │   │          │
│  └──────────────────────────────────────────────┘   │          │
│                               │                      │          │
│                               └──────────────────────┘          │
│                               ▼                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │ API 客户端   │    │ 工具执行器   │    │ 状态管理     │      │
│  │              │    │              │    │              │      │
│  │ claude.ts    │    │ executeTools │    │ Auto Compact │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 数据流

```
用户输入
    │
    ▼
┌──────────────┐
│ 预处理       │
│ - 斜杠命令   │
│ - 输入格式化 │
└──────────────┘
    │
    ▼
┌──────────────┐
│ 消息历史     │
│ 管理         │
└──────────────┘
    │
    ▼
┌──────────────┐     ┌──────────────┐
│ 系统提示词   │────▶│ API 请求     │
│ 构建         │     │ 构造         │
└──────────────┘     └──────────────┘
                            │
                            ▼
┌──────────────┐     ┌──────────────┐
│ 流式响应     │◀────│ Anthropic    │
│ 处理         │     │ API          │
└──────────────┘     └──────────────┘
    │
    ▼
┌──────────────┐
│ 工具调用     │
│ 处理         │
└──────────────┘
    │
    ├─► 无工具调用 ────► 返回最终回答
    │
    └─► 有工具调用 ────► 执行工具
                              │
                              ▼
                       ┌──────────────┐
                       │ 工具结果     │
                       │ 添加到历史   │
                       └──────────────┘
                              │
                              └──────────────► 继续循环
```

## 2. QueryEngine 封装

### 2.1 类定义

**文件**: `src/QueryEngine.ts`

```typescript
export class QueryEngine {
  // 配置
  private config: QueryEngineConfig;
  
  // 消息历史（可变状态）
  private mutableMessages: Message[];
  
  // 会话级状态
  private conversationId: string;
  private abortController: AbortController;
  private totalUsage: NonNullableUsage;
  
  // 工具权限追踪
  private permissionHistory: PermissionRecord[];
  
  constructor(config: QueryEngineConfig) {
    this.config = config;
    this.mutableMessages = [];
    this.conversationId = generateConversationId();
    this.abortController = new AbortController();
    this.totalUsage = { input_tokens: 0, output_tokens: 0 };
    this.permissionHistory = [];
  }
  
  /**
   * 提交用户消息
   * 
   * 核心入口方法，返回流式结果生成器
   */
  async *submitMessage(
    prompt: string | ContentBlockParam[],
    options?: SubmitOptions
  ): AsyncGenerator<SDKMessage> {
    // 1. 构建系统提示词
    const systemPrompt = await this.buildSystemPrompt();
    
    // 2. 处理用户输入
    const processedPrompt = await this.processUserInput(prompt);
    
    // 3. 添加到消息历史
    this.mutableMessages.push({
      role: 'user',
      content: processedPrompt,
      timestamp: Date.now(),
    });
    
    // 4. 调用核心查询循环
    const stream = query({
      messages: this.mutableMessages,
      systemPrompt,
      tools: await this.getTools(),
      signal: this.abortController.signal,
      options: {
        ...this.config.queryOptions,
        ...options,
      },
    });
    
    // 5. 消费流并格式化输出
    const assistantMessages: AssistantMessage[] = [];
    
    for await (const result of stream) {
      // 处理不同类型的结果
      switch (result.type) {
        case 'content':
          // 流式内容块
          yield this.formatContentBlock(result);
          break;
          
        case 'tool_use':
          // 工具调用开始
          yield this.formatToolUse(result);
          break;
          
        case 'tool_result':
          // 工具执行结果
          yield this.formatToolResult(result);
          break;
          
        case 'message_complete':
          // 消息完成，保存到历史
          assistantMessages.push(result.message);
          break;
          
        case 'usage':
          // Token 使用统计
          this.updateUsage(result.usage);
          break;
      }
    }
    
    // 6. 保存完成的助理消息到历史
    for (const msg of assistantMessages) {
      this.mutableMessages.push(msg);
    }
    
    // 7. 持久化对话记录
    await this.persistTranscript();
  }
  
  /**
   * 构建系统提示词
   * 
   * 包含：系统指令、工具描述、上下文信息
   */
  private async buildSystemPrompt(): Promise<string> {
    const parts: string[] = [];
    
    // 1. 基础系统提示
    parts.push(SYSTEM_PROMPT_BASE);
    
    // 2. 加载 CLAUDE.md（项目特定指令）
    const claudeMd = await loadClaudeMd();
    if (claudeMd) {
      parts.push('Project Context:');
      parts.push(claudeMd);
    }
    
    // 3. 加载记忆（用户偏好）
    const memories = await loadMemories();
    if (memories.length > 0) {
      parts.push('User Preferences:');
      parts.push(memories.join('\n'));
    }
    
    // 4. 工具描述（延迟加载的会被过滤）
    const tools = await this.getTools();
    const toolDescriptions = await Promise.all(
      tools.map(t => t.description({} as any, {} as any))
    );
    parts.push('Available Tools:');
    parts.push(toolDescriptions.join('\n\n'));
    
    // 5. 当前上下文（git status、文件列表等）
    const context = await buildContext();
    parts.push('Current Context:');
    parts.push(context);
    
    return parts.join('\n\n');
  }
  
  /**
   * 处理用户输入
   * 
   * 解析斜杠命令、格式化输入
   */
  private async processUserInput(
    prompt: string | ContentBlockParam[]
  ): Promise<ContentBlockParam[]> {
    if (typeof prompt === 'string') {
      // 检查斜杠命令
      if (prompt.startsWith('/')) {
        return await this.handleSlashCommand(prompt);
      }
      
      // 普通文本
      return [{ type: 'text', text: prompt }];
    }
    
    // 已经是 ContentBlock 数组
    return prompt;
  }
  
  /**
   * 获取可用工具
   * 
   * 支持动态工具搜索
   */
  private async getTools(): Promise<Tool[]> {
    const allTools = getAllTools();
    
    // 如果需要，执行工具搜索
    if (this.shouldSearchTools()) {
      const query = this.getLastUserMessage();
      return await searchTools(query, allTools);
    }
    
    return allTools.filter(t => !t.shouldDefer || t.alwaysLoad);
  }
  
  /**
   * 中止查询
   */
  abort(): void {
    this.abortController.abort();
    this.abortController = new AbortController();
  }
}
```

### 2.2 QueryEngine 配置

```typescript
interface QueryEngineConfig {
  // API 配置
  apiKey: string;
  model: string;
  baseURL?: string;
  
  // 查询选项
  queryOptions: QueryOptions;
  
  // 上下文配置
  workingDirectory: string;
  projectRoot?: string;
  
  // 权限配置
  permissionConfig: PermissionConfig;
  
  // 持久化配置
  persistence: {
    enabled: boolean;
    directory: string;
  };
  
  // 特性开关
  features: {
    autoCompact: boolean;
    historySnip: boolean;
    contextCollapse: boolean;
    toolSearch: boolean;
  };
}
```

## 3. 核心查询函数

### 3.1 函数签名

**文件**: `src/query.ts`

```typescript
/**
 * 核心查询函数
 * 
 * 管理完整的对话循环，包括：
 * - 消息预处理
 * - API 调用
 * - 工具执行
 * - 状态转换
 * 
 * 使用 Generator 实现可中断的流式处理
 */
export async function* query(
  params: QueryParams
): AsyncGenerator<QueryResult> {
  const { messages, systemPrompt, tools, signal, options } = params;
  
  // 初始化状态
  const initialState: QueryState = {
    messages: [...messages],
    toolUseContext: createToolUseContext(signal),
    autoCompactTracking: options.autoCompact ? createAutoCompactState() : undefined,
    maxOutputTokensRecoveryCount: 0,
    hasAttemptedReactiveCompact: false,
    transition: undefined,
  };
  
  // 进入查询循环
  return yield* queryLoop({ ...params, initialState });
}

// 查询参数
type QueryParams = {
  messages: Message[];
  systemPrompt: string;
  tools: Tool[];
  signal: AbortSignal;
  options: QueryOptions;
};

// 查询选项
type QueryOptions = {
  maxIterations?: number;           // 最大迭代次数（默认 50）
  maxTokens?: number;               // 最大 token 数
  temperature?: number;             // 温度参数
  autoCompact?: boolean;            // 自动压缩
  enablePromptCaching?: boolean;    // 启用 prompt caching
  toolSearchThreshold?: number;     // 工具搜索阈值
};

// 查询结果类型
export type QueryResult =
  | { type: 'content'; content: ContentBlock }
  | { type: 'tool_use'; tool: ToolUseBlock }
  | { type: 'tool_result'; result: ToolResult }
  | { type: 'message_complete'; message: AssistantMessage }
  | { type: 'usage'; usage: Usage }
  | { type: 'error'; error: QueryError };
```

### 3.2 查询循环实现

```typescript
/**
 * 查询循环
 * 
 * 核心状态机循环，持续直到满足停止条件
 */
async function* queryLoop(
  params: QueryLoopParams
): AsyncGenerator<QueryResult> {
  let state = params.initialState;
  let iteration = 0;
  const maxIterations = params.options.maxIterations ?? 50;
  
  while (iteration < maxIterations) {
    iteration++;
    
    // ==================== 阶段 1: 预处理 ====================
    
    // 1.1 Token 预算检查
    const tokenBudget = calculateTokenBudget(state.messages, params.options.maxTokens);
    
    if (tokenBudget.remaining < tokenBudget.minRequired) {
      // Token 不足，尝试压缩
      if (state.autoCompactTracking && !state.hasAttemptedReactiveCompact) {
        yield { type: 'info', message: 'Token budget exceeded, compacting...' };
        const compacted = await autoCompact(state.messages);
        state = { ...state, messages: compacted, hasAttemptedReactiveCompact: true };
        continue;  // 重新尝试
      } else {
        // 无法压缩，报错
        throw new QueryError('Token budget exceeded and cannot be compacted');
      }
    }
    
    // 1.2 应用 History Snip（如果启用）
    if (params.options.historySnip) {
      state = { ...state, messages: snipMessages(state.messages, tokenBudget) };
    }
    
    // 1.3 应用 Context Collapse（如果启用）
    if (params.options.contextCollapse) {
      state = { ...state, messages: await collapseContext(state.messages) };
    }
    
    // ==================== 阶段 2: API 调用 ====================
    
    // 构建 API 请求
    const apiParams = buildAPIParams({
      messages: state.messages,
      systemPrompt: params.systemPrompt,
      tools: params.tools,
      signal: params.signal,
      options: {
        maxTokens: tokenBudget.forResponse,
        temperature: params.options.temperature,
        enableCaching: params.options.enablePromptCaching,
      },
    });
    
    // 调用 API
    const stream = queryModelWithStreaming(apiParams);
    
    // 收集助理消息
    const assistantContent: ContentBlock[] = [];
    const toolUseBlocks: ToolUseBlock[] = [];
    let usage: Usage = { input_tokens: 0, output_tokens: 0 };
    
    // 处理流式响应
    for await (const event of stream) {
      switch (event.type) {
        case 'content_block_start':
        case 'content_block_delta':
          // 流式内容
          if (event.delta?.text) {
            yield { type: 'content', content: event.delta };
            assistantContent.push(event.delta);
          }
          break;
          
        case 'tool_use':
          // 工具调用
          yield { type: 'tool_use', tool: event.tool };
          toolUseBlocks.push(event.tool);
          break;
          
        case 'message_stop':
          // 消息完成，获取使用统计
          usage = event.usage;
          yield { type: 'usage', usage };
          break;
          
        case 'error':
          // 处理 API 错误
          yield* handleAPIError(event.error, state, params);
          return;  // 错误时退出
      }
    }
    
    // ==================== 阶段 3: 处理工具调用 ====================
    
    if (toolUseBlocks.length === 0) {
      // 没有工具调用，对话结束
      const assistantMessage: AssistantMessage = {
        role: 'assistant',
        content: assistantContent,
        usage,
      };
      
      yield { type: 'message_complete', message: assistantMessage };
      return;  // 正常退出
    }
    
    // 有工具调用，需要执行
    const toolResults = await executeToolCalls(
      toolUseBlocks,
      params.tools,
      state.toolUseContext
    );
    
    // 生成工具结果消息
    const toolResultMessages: Message[] = [];
    for (const result of toolResults) {
      yield { type: 'tool_result', result };
      
      // 构建工具结果消息
      const toolResultMessage: ToolResultMessage = {
        role: 'user',
        content: [{
          type: 'tool_result',
          tool_use_id: result.toolUseId,
          content: result.result,
        }],
      };
      
      toolResultMessages.push(toolResultMessage);
    }
    
    // ==================== 阶段 4: 状态更新 ====================
    
    // 构建助理消息
    const assistantMessage: AssistantMessage = {
      role: 'assistant',
      content: [
        ...assistantContent,
        ...toolUseBlocks.map(b => ({ type: 'tool_use', ...b })),
      ],
      usage,
    };
    
    // 更新消息历史
    state = {
      ...state,
      messages: [
        ...state.messages,
        assistantMessage,
        ...toolResultMessages,
      ],
      transition: { type: 'tool_execution', toolCount: toolUseBlocks.length },
    };
    
    // 更新 Auto Compact 追踪
    if (state.autoCompactTracking) {
      state.autoCompactTracking = updateAutoCompactState(
        state.autoCompactTracking,
        usage
      );
    }
    
    // 继续下一次迭代
  }
  
  // 达到最大迭代次数
  throw new QueryError(`Maximum iterations (${maxIterations}) reached`);
}

// 查询状态
type QueryState = {
  messages: Message[];
  toolUseContext: ToolUseContext;
  autoCompactTracking?: AutoCompactTrackingState;
  maxOutputTokensRecoveryCount: number;
  hasAttemptedReactiveCompact: boolean;
  transition?: Continue;
};

type Continue =
  | { type: 'tool_execution'; toolCount: number }
  | { type: 'compact_applied'; savedTokens: number }
  | { type: 'error_recovery'; error: string };
```

## 4. 消息预处理

### 4.1 Auto Compact（自动压缩）

```typescript
/**
 * 自动压缩消息历史
 * 
 * 当 token 数接近上限时，总结旧消息并替换为摘要
 */
async function autoCompact(
  messages: Message[],
  options?: CompactOptions
): Promise<Message[]> {
  const tokenCount = await countTokens(messages);
  const threshold = options?.threshold ?? 100000;
  
  if (tokenCount < threshold) {
    return messages;  // 不需要压缩
  }
  
  // 保留最近的 N 条消息（通常是最近的对话轮次）
  const keepRecent = options?.keepRecent ?? 10;
  const recentMessages = messages.slice(-keepRecent);
  const olderMessages = messages.slice(0, -keepRecent);
  
  // 总结旧消息
  const summary = await summarizeMessages(olderMessages, {
    maxSummaryTokens: options?.maxSummaryTokens ?? 500,
  });
  
  // 构建压缩后的消息列表
  const compacted: Message[] = [
    {
      role: 'system',
      content: [{
        type: 'text',
        text: `Previous conversation summary:\n${summary}`,
      }],
    },
    ...recentMessages,
  ];
  
  return compacted;
}

/**
 * 总结消息
 * 
 * 使用轻量级模型或启发式方法生成摘要
 */
async function summarizeMessages(
  messages: Message[],
  options: SummaryOptions
): Promise<string> {
  // 1. 提取关键信息
  const keyPoints = extractKeyPoints(messages);
  
  // 2. 生成摘要
  const summaryPrompt = `Summarize the following conversation concisely:
${formatMessagesForSummary(messages)}`;
  
  const response = await queryModel({
    messages: [{ role: 'user', content: summaryPrompt }],
    maxTokens: options.maxSummaryTokens,
    model: 'claude-3-haiku-20240307',  // 使用轻量级模型
  });
  
  return response.content;
}

// Auto Compact 状态追踪
type AutoCompactTrackingState = {
  totalTokensBefore: number;
  totalTokensAfter: number;
  compactCount: number;
  lastCompactTime: number;
};

function createAutoCompactState(): AutoCompactTrackingState {
  return {
    totalTokensBefore: 0,
    totalTokensAfter: 0,
    compactCount: 0,
    lastCompactTime: 0,
  };
}

function updateAutoCompactState(
  state: AutoCompactTrackingState,
  usage: Usage
): AutoCompactTrackingState {
  return {
    ...state,
    totalTokensAfter: usage.input_tokens + usage.output_tokens,
    lastCompactTime: Date.now(),
  };
}
```

### 4.2 History Snip（历史截断）

```typescript
/**
 * 截断消息历史
 * 
 * 当消息超出 token 预算时，移除旧消息
 */
function snipMessages(
  messages: Message[],
  budget: TokenBudget
): Message[] {
  let currentTokens = 0;
  const snipped: Message[] = [];
  
  // 从后向前遍历（保留最新消息）
  for (let i = messages.length - 1; i >= 0; i--) {
    const message = messages[i];
    const messageTokens = estimateTokens(message);
    
    if (currentTokens + messageTokens > budget.maxHistory) {
      // 超出预算，停止添加
      break;
    }
    
    currentTokens += messageTokens;
    snipped.unshift(message);  // 添加到开头（保持顺序）
  }
  
  // 如果截断了消息，添加系统提示说明
  if (snipped.length < messages.length) {
    snipped.unshift({
      role: 'system',
      content: [{
        type: 'text',
        text: `[Earlier messages omitted due to length constraints]`,
      }],
    });
  }
  
  return snipped;
}
```

### 4.3 Context Collapse（上下文折叠）

```typescript
/**
 * 上下文折叠
 * 
 * 将已解决的讨论话题归档为摘要
 */
async function collapseContext(messages: Message[]): Promise<Message[]> {
  // 1. 提取话题
  const topics = extractTopics(messages);
  
  // 2. 分类话题
  const active = topics.filter(t => !t.isResolved);
  const resolved = topics.filter(t => t.isResolved);
  
  if (resolved.length === 0) {
    return messages;  // 没有可折叠的内容
  }
  
  // 3. 总结已解决话题
  const resolvedSummary = await summarizeTopics(resolved);
  
  // 4. 获取活跃话题相关的消息
  const activeMessages = getMessagesForTopics(active, messages);
  
  // 5. 构建折叠后的消息列表
  return [
    {
      role: 'system',
      content: [{
        type: 'text',
        text: `Resolved topics:\n${resolvedSummary}`,
      }],
    },
    ...activeMessages,
  ];
}
```

## 5. Token 预算管理

### 5.1 预算计算

```typescript
/**
 * Token 预算管理
 * 
 * 管理 token 分配：历史消息 vs 响应空间
 */
function calculateTokenBudget(
  messages: Message[],
  maxTokens?: number
): TokenBudget {
  const modelLimit = 200000;  // Claude 3 的上下文窗口
  const max = maxTokens ?? modelLimit;
  
  // 为当前查询分配 token
  const inputTokens = estimateTokens(messages);
  const systemTokens = 4000;  // 系统提示预留
  const safetyBuffer = 1000;   // 安全缓冲
  
  // 可用于响应的 token
  const forResponse = max - inputTokens - systemTokens - safetyBuffer;
  
  // 最小响应空间
  const minResponseTokens = 4000;
  
  return {
    total: max,
    used: inputTokens,
    remaining: max - inputTokens - systemTokens,
    forResponse: Math.max(forResponse, minResponseTokens),
    minRequired: inputTokens + systemTokens + minResponseTokens,
    maxHistory: max - systemTokens - minResponseTokens - safetyBuffer,
  };
}

type TokenBudget = {
  total: number;           // 总预算
  used: number;            // 已使用
  remaining: number;       // 剩余
  forResponse: number;     // 为响应急留
  minRequired: number;     // 最小必需
  maxHistory: number;      // 最大历史消息 token
};

/**
 * 估计消息 token 数
 * 
 * 使用简单的启发式估计（实际使用 API 计数）
 */
function estimateTokens(message: Message): number {
  let tokens = 0;
  
  for (const block of message.content) {
    switch (block.type) {
      case 'text':
        // 粗略估计：1 token ≈ 4 字符
        tokens += Math.ceil(block.text.length / 4);
        break;
      case 'image':
        // 图片有固定 token 成本
        tokens += 1600;  // base64 编码图片的估计
        break;
      case 'tool_use':
      case 'tool_result':
        // 工具调用/结果的估计
        tokens += 500;
        break;
    }
  }
  
  // 消息开销
  tokens += 10;
  
  return tokens;
}
```

## 6. 工具执行

### 6.1 执行流程

```typescript
/**
 * 执行工具调用
 * 
 * 处理权限检查、并发控制、错误处理
 */
async function executeToolCalls(
  toolUseBlocks: ToolUseBlock[],
  availableTools: Tool[],
  context: ToolUseContext
): Promise<ToolExecutionResult[]> {
  // 1. 验证工具存在
  const validated = toolUseBlocks.map(block => {
    const tool = findToolByName(block.name);
    if (!tool) {
      throw new Error(`Tool not found: ${block.name}`);
    }
    return { block, tool };
  });
  
  // 2. 分离可并发和不可并发的工具
  const concurrentSafe: typeof validated = [];
  const sequential: typeof validated = [];
  
  for (const { block, tool } of validated) {
    if (tool.isConcurrencySafe(block.input)) {
      concurrentSafe.push({ block, tool });
    } else {
      sequential.push({ block, tool });
    }
  }
  
  const results: ToolExecutionResult[] = [];
  
  // 3. 并行执行安全工具
  if (concurrentSafe.length > 0) {
    const concurrentResults = await Promise.all(
      concurrentSafe.map(({ block, tool }) =>
        executeSingleTool(block, tool, context)
      )
    );
    results.push(...concurrentResults);
  }
  
  // 4. 顺序执行不安全工具
  for (const { block, tool } of sequential) {
    const result = await executeSingleTool(block, tool, context);
    results.push(result);
  }
  
  return results;
}

async function executeSingleTool(
  block: ToolUseBlock,
  tool: Tool,
  context: ToolUseContext
): Promise<ToolExecutionResult> {
  // 1. 验证输入
  const validated = tool.inputSchema.parse(block.input);
  
  // 2. 额外验证（如果有）
  if (tool.validateInput) {
    const validation = await tool.validateInput(validated, context);
    if (!validation.success) {
      return {
        toolUseId: block.id,
        result: `Validation error: ${validation.error}`,
        error: true,
      };
    }
  }
  
  // 3. 检查权限
  const permission = await tool.checkPermissions(validated, context);
  
  if (permission.behavior === 'deny') {
    return {
      toolUseId: block.id,
      result: `Permission denied: ${permission.reason}`,
      error: true,
    };
  }
  
  if (permission.behavior === 'ask') {
    // 需要用户确认 - 这里简化处理
    throw new Error('Permission required');
  }
  
  // 4. 执行工具
  try {
    const result = await tool.call(
      validated,
      context,
      canUseTool  // 全局权限函数
    );
    
    return {
      toolUseId: block.id,
      result: result.result,
      newMessages: result.newMessages,
    };
  } catch (error) {
    return {
      toolUseId: block.id,
      result: `Error: ${error.message}`,
      error: true,
    };
  }
}
```

## 7. 错误处理

### 7.1 API 错误处理

```typescript
/**
 * 处理 API 错误
 * 
 * 支持错误恢复和重试
 */
async function* handleAPIError(
  error: APIError,
  state: QueryState,
  params: QueryLoopParams
): AsyncGenerator<QueryResult> {
  switch (error.code) {
    case 'max_output_tokens':
      // 输出 token 超限，尝试恢复
      yield* handleMaxOutputTokensError(error, state, params);
      break;
      
    case 'prompt_too_long':
      // 提示词太长，尝试压缩
      if (!state.hasAttemptedReactiveCompact) {
        yield { type: 'info', message: 'Prompt too long, compacting...' };
        const compacted = await autoCompact(state.messages);
        
        // 重新尝试（通过修改状态）
        state = {
          ...state,
          messages: compacted,
          hasAttemptedReactiveCompact: true,
        };
        
        yield { type: 'transition', transition: { type: 'error_recovery', error: 'prompt_too_long' } };
      } else {
        yield { type: 'error', error };
      }
      break;
      
    case 'rate_limit':
      // 速率限制，重试
      yield { type: 'info', message: 'Rate limited, retrying...' };
      await delay(1000);
      // 重新进入循环
      break;
      
    default:
      yield { type: 'error', error };
  }
}

/**
 * 处理 max_output_tokens 错误
 * 
 * 通过减小响应空间重试
 */
async function* handleMaxOutputTokensError(
  error: APIError,
  state: QueryState,
  params: QueryLoopParams
): AsyncGenerator<QueryResult> {
  const recoveryCount = state.maxOutputTokensRecoveryCount;
  
  if (recoveryCount >= 3) {
    yield { type: 'error', error };
    return;
  }
  
  // 减小响应空间
  const reducedOptions = {
    ...params.options,
    maxTokens: (params.options.maxTokens ?? 4000) / 2,
  };
  
  yield { type: 'info', message: `Reducing output tokens (attempt ${recoveryCount + 1})` };
  
  // 更新状态
  state = {
    ...state,
    maxOutputTokensRecoveryCount: recoveryCount + 1,
  };
  
  // 递归调用（简化示例）
  yield* queryLoop({ ...params, options: reducedOptions, initialState: state });
}
```

### 7.2 错误分类

```typescript
type QueryError =
  | { code: 'token_budget_exceeded'; message: string }
  | { code: 'max_iterations_reached'; maxIterations: number }
  | { code: 'aborted'; message: string }
  | { code: 'api_error'; error: APIError }
  | { code: 'tool_error'; tool: string; error: Error }
  | { code: 'permission_denied'; tool: string; reason: string };

// API 错误（来自 Anthropic SDK）
type APIError =
  | { code: 'max_output_tokens'; message: string }
  | { code: 'prompt_too_long'; message: string }
  | { code: 'rate_limit'; message: string; retryAfter?: number }
  | { code: 'invalid_api_key'; message: string }
  | { code: 'unknown'; message: string; status?: number };
```

## 8. 流式处理

### 8.1 Generator 模式

```typescript
/**
 * 流式处理的优势
 * 
 * 1. 可中断：用户可以随时中止
 * 2. 低延迟：内容逐步显示
 * 3. 低内存：不需要缓存全部结果
 */

// 消费者代码示例
async function consumeStream(stream: AsyncGenerator<QueryResult>) {
  try {
    for await (const result of stream) {
      switch (result.type) {
        case 'content':
          // 流式显示内容
          ui.appendText(result.content.text);
          break;
          
        case 'tool_use':
          // 显示工具调用
          ui.showToolUse(result.tool);
          break;
          
        case 'tool_result':
          // 显示工具结果
          ui.showToolResult(result.result);
          break;
          
        case 'error':
          // 显示错误
          ui.showError(result.error);
          break;
      }
    }
  } catch (error) {
    if (error.name === 'AbortError') {
      console.log('Stream aborted by user');
    } else {
      throw error;
    }
  }
}
```

### 8.2 中止处理

```typescript
/**
 * 中止信号传递
 * 
 * 通过 AbortSignal 实现跨层中止
 */

// 1. 创建中止控制器
const abortController = new AbortController();

// 2. 传递 signal 到查询
const stream = query({
  messages,
  systemPrompt,
  tools,
  signal: abortController.signal,
  options,
});

// 3. 用户触发中止
function onUserAbort() {
  abortController.abort();
}

// 4. 查询循环中检查
async function* queryLoop(params: QueryLoopParams) {
  const { signal } = params;
  
  while (true) {
    // 检查中止
    if (signal.aborted) {
      throw new AbortError('Query aborted');
    }
    
    // API 调用也传递 signal
    const stream = queryModelWithStreaming({
      ...params,
      signal,
    });
    
    for await (const event of stream) {
      if (signal.aborted) {
        stream.cancel();  // 取消流
        throw new AbortError('Query aborted');
      }
      
      yield event;
    }
  }
}
```

## 9. 状态转换

### 9.1 状态机

```
┌─────────────────────────────────────────────────────────────┐
│                      查询状态机                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────┐                                              │
│   │ INITIAL  │                                              │
│   └────┬─────┘                                              │
│        │                                                    │
│        │ 开始查询                                            │
│        ▼                                                    │
│   ┌──────────┐     有工具      ┌──────────┐                 │
│   │STREAMING │────────────────▶│TOOL_EXEC │                 │
│   │          │    调用         │          │                 │
│   └────┬─────┘                 └────┬─────┘                 │
│        │                            │                       │
│        │ 无工具                     │ 执行完成               │
│        ▼                            ▼                       │
│   ┌──────────┐                 ┌──────────┐                 │
│   │ COMPLETE │◀────────────────│ STREAMING│                 │
│   └──────────┘                 └──────────┘                 │
│        │                            │                       │
│        │                            │ Token 超限             │
│        │ 错误                       ▼                       │
│        ▼                       ┌──────────┐                 │
│   ┌──────────┐                 │ COMPACT  │                 │
│   │  ERROR   │                 │          │                 │
│   └──────────┘                 └────┬─────┘                 │
│                                     │                       │
│                                     │ 压缩完成               │
│                                     ▼                       │
│                                ┌──────────┐                 │
│                                │ STREAMING│                 │
│                                └──────────┘                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 转换逻辑

```typescript
/**
 * 确定下一个状态
 */
function determineNextState(
  currentState: QueryState,
  result: IterationResult
): StateTransition {
  if (result.error) {
    return { type: 'error', error: result.error };
  }
  
  if (result.toolCalls.length > 0) {
    return {
      type: 'tool_execution',
      toolCount: result.toolCalls.length,
    };
  }
  
  if (result.isComplete) {
    return { type: 'complete' };
  }
  
  if (result.shouldCompact) {
    return { type: 'compact', savedTokens: result.savedTokens };
  }
  
  return { type: 'continue' };
}

type StateTransition =
  | { type: 'continue' }
  | { type: 'tool_execution'; toolCount: number }
  | { type: 'compact'; savedTokens: number }
  | { type: 'complete' }
  | { type: 'error'; error: QueryError };
```

## 10. 总结

核心查询循环的设计体现了以下架构原则：

1. **Generator 模式**: 使用 AsyncGenerator 实现可中断的流式处理
2. **不可变状态**: 状态更新使用展开操作符，避免副作用
3. **错误恢复**: 支持多种错误场景的自动恢复（token 超限、prompt 过长等）
4. **Token 预算**: 精细的 token 分配和管理
5. **预处理管道**: 压缩、截断、折叠多层预处理
6. **并发控制**: 智能的工具并行执行
7. **中止传播**: 通过 AbortSignal 实现跨层中止

这些设计使得查询循环能够处理复杂的对话场景，同时保持良好的性能和用户体验。
