# API 层设计

## 文件位置

- `src/services/api/claude.ts` - 核心 API 客户端
- `src/services/api/client.ts` - SDK 客户端配置
- `src/services/api/retry.ts` - 重试机制
- `src/services/api/cache.ts` - 缓存控制

## 1. API 层概述

API 层是 Claude Code 与 Anthropic API 交互的核心，负责请求构建、流式处理、错误恢复等功能。

### 1.1 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         API 层架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    QueryEngine                            │ │
│  │  submitMessage()                                          │ │
│  └────────────────────┬──────────────────────────────────────┘ │
│                       │                                         │
│                       ▼                                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                   query() 函数                            │ │
│  │  - 状态管理                                               │ │
│  │  - 工具执行                                               │ │
│  └────────────────────┬──────────────────────────────────────┘ │
│                       │                                         │
│                       ▼                                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │            queryModelWithStreaming()                      │ │
│  │  - VCR 录制/回放                                          │ │
│  │  - 错误处理                                               │ │
│  └────────────────────┬──────────────────────────────────────┘ │
│                       │                                         │
│                       ▼                                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                queryModel() 核心                          │ │
│  │  - Beta Headers 管理                                      │ │
│  │  - Prompt Caching                                         │ │
│  │  - 工具搜索                                               │ │
│  │  - 重试机制                                               │ │
│  └────────────────────┬──────────────────────────────────────┘ │
│                       │                                         │
│                       ▼                                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │               Anthropic SDK                               │ │
│  │  - 流式 API 调用                                          │ │
│  │  - 事件解析                                               │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 核心流程

```
QueryEngine.submitMessage()
    │
    ▼
构建请求参数
    │
    ├─► 系统提示词（包含工具描述、上下文）
    ├─► 消息历史（经过预处理）
    ├─► 工具列表（可能经过搜索过滤）
    └─► Beta Headers（功能开关）
    │
    ▼
调用 Anthropic SDK
    │
    ├─► 流式响应处理
    ├─► 内容块解析
    ├─► 工具调用识别
    └─► Token 使用统计
    │
    ▼
返回生成器
    │
    ├─► 文本内容（增量）
    ├─► 工具调用
    ├─► 工具结果
    └─► 完成事件
```

## 2. 核心 API 函数

### 2.1 流式查询函数

**文件**: `src/services/api/claude.ts`

```typescript
import Anthropic from '@anthropic-ai/sdk';
import type { Message, MessageStreamEvent, ContentBlock, ToolUseBlock } from '@anthropic-ai/sdk/resources/messages';

/**
 * 核心 API 查询函数（带流式响应）
 * 
 * 这是 API 层的主要入口，提供完整的流式处理能力。
 * 
 * @param params - 查询参数
 * @returns 异步生成器，产出流式事件
 */
export async function* queryModelWithStreaming(
  params: QueryModelParams
): AsyncGenerator<StreamEvent | AssistantMessage | SystemAPIErrorMessage> {
  const { messages, systemPrompt, tools, signal, options } = params;
  
  // 1. 使用 VCR 包装（支持录制/回放）
  return yield* withStreamingVCR(messages, async function* () {
    yield* queryModel(messages, systemPrompt, tools, signal, options);
  });
}

// 查询参数类型
type QueryModelParams = {
  messages: Message[];
  systemPrompt: string;
  tools?: Tool[];
  signal?: AbortSignal;
  options?: QueryOptions;
};

type QueryOptions = {
  model?: string;
  maxTokens?: number;
  temperature?: number;
  thinking?: ThinkingConfig;
  enablePromptCaching?: boolean;
  enableToolSearch?: boolean;
  retryCount?: number;
};

// 流式事件类型
export type StreamEvent =
  | { type: 'message_start'; message: Message }
  | { type: 'content_block_start'; index: number; content_block: ContentBlock }
  | { type: 'content_block_delta'; index: number; delta: ContentDelta }
  | { type: 'content_block_stop'; index: number }
  | { type: 'message_delta'; delta: MessageDelta; usage: Usage }
  | { type: 'message_stop'; message: Message }
  | { type: 'error'; error: APIError };
```

### 2.2 核心 API 调用

```typescript
/**
 * 底层 API 调用实现
 * 
 * 处理：
 * - Beta Headers 注入
 * - Prompt Caching 配置
 * - 工具搜索
 * - 重试机制
 * - 流式事件解析
 */
async function* queryModel(
  messages: Message[],
  systemPrompt: string,
  tools: Tool[] = [],
  signal?: AbortSignal,
  options: QueryOptions = {}
): AsyncGenerator<StreamEvent> {
  const client = getAnthropicClient();
  
  // 1. 构建请求参数
  const requestParams = buildRequestParams({
    messages,
    systemPrompt,
    tools,
    options,
  });
  
  // 2. 获取重试包装器
  const retry = createRetryWrapper({
    maxRetries: options.retryCount ?? 3,
    shouldRetry: shouldRetryError,
  });
  
  // 3. 执行调用（带重试）
  const stream = await retry(async () => {
    return client.messages.create({
      ...requestParams,
      stream: true,
      signal,
    });
  });
  
  // 4. 处理流式响应
  for await (const event of stream) {
    // 检查中止信号
    if (signal?.aborted) {
      stream.controller.abort();
      throw new AbortError('Request aborted');
    }
    
    // 解析并转换事件
    const streamEvent = parseStreamEvent(event);
    yield streamEvent;
    
    // 处理特殊事件
    if (streamEvent.type === 'error') {
      yield* handleAPIError(streamEvent.error);
      return;
    }
  }
}

/**
 * 构建 API 请求参数
 */
function buildRequestParams(params: BuildParams): MessageCreateParams {
  const { messages, systemPrompt, tools, options } = params;
  
  // 1. 基础参数
  const baseParams: MessageCreateParams = {
    model: options.model ?? 'claude-3-sonnet-20240229',
    max_tokens: options.maxTokens ?? 4096,
    messages,
    system: systemPrompt,
  };
  
  // 2. 添加工具（如果启用工具搜索，可能只包含部分工具）
  if (tools.length > 0 && options.enableToolSearch !== false) {
    // 过滤延迟加载的工具
    const activeTools = tools.filter(t => !t.shouldDefer || t.alwaysLoad);
    
    baseParams.tools = activeTools.map(tool => ({
      name: tool.name,
      description: tool.description,
      input_schema: zodToJSONSchema(tool.inputSchema),
    }));
  }
  
  // 3. 添加思维配置（如果支持）
  if (options.thinking) {
    baseParams.thinking = {
      type: 'enabled',
      budget_tokens: options.thinking.budgetTokens,
    };
  }
  
  // 4. 添加 Beta Headers
  baseParams.extra_headers = buildBetaHeaders({
    enablePromptCaching: options.enablePromptCaching ?? true,
    enableToolSearch: options.enableToolSearch ?? true,
  });
  
  return baseParams;
}
```

## 3. Beta Headers 管理

### 3.1 Headers 构建

```typescript
/**
 * Beta Headers 构建
 * 
 * Anthropic API 使用 HTTP Headers 启用实验性功能。
 */
function buildBetaHeaders(options: HeaderOptions): Record<string, string> {
  const headers: Record<string, string> = {};
  
  // 1. Prompt Caching（提示词缓存）
  // 减少重复请求的成本
  if (options.enablePromptCaching) {
    headers['anthropic-beta'] = 'prompt-caching-2024-07-31';
  }
  
  // 2. Tool Search（工具搜索）
  // 支持动态工具加载
  if (options.enableToolSearch) {
    headers['anthropic-beta'] = appendBetaHeader(
      headers['anthropic-beta'],
      'tool-search-2024-11-01'
    );
  }
  
  // 3. 1-hour Cache TTL（延长缓存有效期）
  // 需要用户资格检查
  if (is1hCacheEligible()) {
    headers['anthropic-beta'] = appendBetaHeader(
      headers['anthropic-beta'],
      'prompt-caching-1h-ttl-2024-10-01'
    );
  }
  
  // 4. Extended Output（扩展输出长度）
  // 支持 8192 token 输出
  if (options.enableExtendedOutput) {
    headers['anthropic-beta'] = appendBetaHeader(
      headers['anthropic-beta'],
      'max-tokens-3-5-sonnet-2024-07-15'
    );
  }
  
  return headers;
}

function appendBetaHeader(existing: string | undefined, header: string): string {
  if (!existing) return header;
  return `${existing},${header}`;
}

// 1小时缓存资格检查
function is1hCacheEligible(): boolean {
  // 检查全局状态
  let eligible = getPromptCache1hEligible();
  
  if (eligible === null) {
    // 首次判断
    // 资格门控：防止 mid-session 切换
    eligible = check1hCacheEligibility();
    setPromptCache1hEligible(eligible);
  }
  
  return eligible ?? false;
}

function check1hCacheEligibility(): boolean {
  // 检查用户类型、使用历史等
  // 简化示例：仅 Anthropic 内部用户
  return process.env.USER_TYPE === 'ant' || false;
}
```

### 3.2 Beta Headers 说明

| Header | 功能 | 状态 |
|--------|------|------|
| `prompt-caching-2024-07-31` | 提示词缓存 | 公开 Beta |
| `prompt-caching-1h-ttl-2024-10-01` | 1小时缓存 TTL | 限制访问 |
| `tool-search-2024-11-01` | 工具搜索 | 公开 Beta |
| `max-tokens-3-5-sonnet-2024-07-15` | 扩展输出（8k） | 公开 Beta |
| `computer-use-2024-10-22` | Computer Use | 公开 Beta |

## 4. Prompt Caching

### 4.1 缓存控制

```typescript
/**
 * Prompt Caching 配置
 * 
 * 通过 cache_control 参数控制缓存行为。
 */
export function getCacheControl(params: CacheParams): CacheControl {
  const { scope, querySource } = params;
  
  return {
    type: 'ephemeral',  // 临时缓存
    
    // 1小时 TTL（如果符合条件）
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),
    
    // 全局缓存（跨查询共享）
    ...(scope === 'global' && { scope: 'global' }),
  };
}

type CacheControl = {
  type: 'ephemeral';
  ttl?: '1h';
  scope?: 'global';
};

// 是否应该使用 1 小时 TTL
function should1hCacheTTL(querySource: QuerySource): boolean {
  // 只在特定查询源使用
  return querySource === 'initial' || querySource === 'system';
}

// 应用缓存控制到消息
function applyCacheControl(
  messages: Message[],
  cacheConfig: CacheConfig
): Message[] {
  return messages.map((msg, index) => {
    // 为特定消息添加缓存控制
    if (cacheConfig.cacheIndices?.includes(index)) {
      return {
        ...msg,
        content: addCacheControlToContent(msg.content, cacheConfig.control),
      };
    }
    return msg;
  });
}

function addCacheControlToContent(
  content: ContentBlock[],
  control: CacheControl
): ContentBlock[] {
  // 为最后一个内容块添加缓存控制
  if (content.length === 0) return content;
  
  const lastIndex = content.length - 1;
  const lastBlock = content[lastIndex];
  
  return [
    ...content.slice(0, lastIndex),
    {
      ...lastBlock,
      cache_control: control,
    },
  ];
}
```

### 4.2 缓存策略

```typescript
/**
 * 默认缓存策略
 * 
 * 为不同类型的内容设置缓存策略
 */
export const defaultCacheStrategy: CacheStrategy = {
  // 系统提示词：全局缓存
  system: {
    scope: 'global',
    ttl: '1h',
  },
  
  // 工具描述：全局缓存
  tools: {
    scope: 'global',
    ttl: '1h',
  },
  
  // 多轮对话历史：渐进式缓存
  conversation: {
    // 为每 N 条消息设置一个缓存点
    cacheEveryN: 4,
    scope: 'global',
  },
  
  // 上下文信息（git status、文件列表等）：不缓存
  context: {
    cache: false,
  },
};

/**
 * 应用缓存策略
 */
export function applyCacheStrategy(
  messages: Message[],
  strategy: CacheStrategy
): Message[] {
  const result = [...messages];
  
  // 应用对话缓存策略
  if (strategy.conversation.cacheEveryN) {
    const n = strategy.conversation.cacheEveryN;
    
    for (let i = n - 1; i < result.length; i += n) {
      result[i] = {
        ...result[i],
        content: addCacheControlToContent(result[i].content, {
          type: 'ephemeral',
          scope: strategy.conversation.scope,
        }),
      };
    }
  }
  
  return result;
}
```

### 4.3 缓存统计

```typescript
/**
 * 缓存使用统计
 */
type CacheStats = {
  // 缓存创建数
  cacheCreationInputTokens: number;
  
  // 缓存读取数
  cacheReadInputTokens: number;
  
  // 非缓存输入数
  inputTokens: number;
  
  // 输出数
  outputTokens: number;
};

/**
 * 计算缓存节省
 */
export function calculateCacheSavings(stats: CacheStats): Savings {
  // 缓存成本：1.25x 基础价格
  const cacheWriteCost = stats.cacheCreationInputTokens * 1.25;
  
  // 缓存命中成本：0.1x 基础价格
  const cacheReadCost = stats.cacheReadInputTokens * 0.1;
  
  // 正常成本：1x 基础价格
  const normalCost = stats.inputTokens * 1;
  
  // 如果没有缓存的总成本
  const withoutCache = (stats.cacheCreationInputTokens + stats.cacheReadInputTokens) * 1;
  
  // 实际成本
  const actualCost = cacheWriteCost + cacheReadCost + normalCost;
  
  // 节省
  const saved = withoutCache - actualCost;
  const percentage = (saved / withoutCache) * 100;
  
  return {
    savedTokens: stats.cacheReadInputTokens,
    savedCost: saved,
    percentage,
  };
}
```

## 5. 重试机制

### 5.1 重试包装器

**文件**: `src/services/api/retry.ts`

```typescript
/**
 * 创建重试包装器
 * 
 * 自动处理可重试错误，支持指数退避。
 */
export function createRetryWrapper(options: RetryOptions) {
  const { maxRetries, shouldRetry, onRetry, baseDelay = 1000 } = options;
  
  return async function retry<T>(fn: () => Promise<T>): Promise<T> {
    let lastError: Error;
    
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        return await fn();
      } catch (error) {
        lastError = error as Error;
        
        // 检查是否应该重试
        if (attempt === maxRetries || !shouldRetry(error)) {
          throw error;
        }
        
        // 计算退避延迟
        const delay = calculateBackoff(attempt, baseDelay);
        
        // 通知重试
        onRetry?.(error, attempt + 1, delay);
        
        // 等待
        await sleep(delay);
      }
    }
    
    throw lastError!;
  };
}

// 计算指数退避延迟
function calculateBackoff(attempt: number, baseDelay: number): number {
  // 指数退避 + 随机抖动
  const exponentialDelay = baseDelay * Math.pow(2, attempt);
  const jitter = Math.random() * 1000;
  return Math.min(exponentialDelay + jitter, 30000);  // 最大 30 秒
}

// 重试选项
type RetryOptions = {
  maxRetries: number;
  shouldRetry: (error: any) => boolean;
  onRetry?: (error: any, attempt: number, delay: number) => void;
  baseDelay?: number;
};
```

### 5.2 错误分类

```typescript
/**
 * 判断是否应该重试
 */
export function shouldRetryError(error: any): boolean {
  // 网络错误
  if (error.code === 'ECONNREFUSED') return true;
  if (error.code === 'ETIMEDOUT') return true;
  if (error.code === 'ENOTFOUND') return true;
  
  // HTTP 错误
  if (error.status === 429) return true;  // 速率限制
  if (error.status === 500) return true;  // 服务器错误
  if (error.status === 502) return true;  // 网关错误
  if (error.status === 503) return true;  // 服务不可用
  if (error.status === 504) return true;  // 网关超时
  if (error.status === 529) return true;  // Anthropic 过载
  
  // API 特定错误
  if (error.error?.code === 'rate_limit') return true;
  if (error.error?.code === 'overloaded') return true;
  
  // 不重试的错误
  if (error.status === 400) return false;  // 请求错误
  if (error.status === 401) return false;  // 认证错误
  if (error.status === 403) return false;  // 权限错误
  if (error.status === 404) return false;  // 未找到
  
  return false;
}
```

### 5.3 降级策略

```typescript
/**
 * 模型降级策略
 * 
 * 当高级模型不可用时，自动降级到可用模型。
 */
const MODEL_FALLBACK_CHAIN: Record<string, string[]> = {
  'claude-3-opus-20240229': [
    'claude-3-opus-20240229',
    'claude-3-sonnet-20240229',
    'claude-3-haiku-20240307',
  ],
  'claude-3-sonnet-20240229': [
    'claude-3-sonnet-20240229',
    'claude-3-haiku-20240307',
  ],
  'claude-3-5-sonnet-20241022': [
    'claude-3-5-sonnet-20241022',
    'claude-3-sonnet-20240229',
    'claude-3-haiku-20240307',
  ],
};

export async function queryWithFallback(
  params: QueryParams,
  preferredModel: string
): AsyncGenerator<StreamEvent> {
  const fallbackChain = MODEL_FALLBACK_CHAIN[preferredModel] ?? [preferredModel];
  
  for (const model of fallbackChain) {
    try {
      return yield* queryModel({
        ...params,
        model,
      });
    } catch (error) {
      // 如果是最后一种模型，抛出错误
      if (model === fallbackChain[fallbackChain.length - 1]) {
        throw error;
      }
      
      // 如果不是过载错误，不重试
      if (error.status !== 529 && error.error?.code !== 'overloaded') {
        throw error;
      }
      
      // 降级到下一个模型
      console.warn(`Model ${model} overloaded, falling back to next model`);
    }
  }
}
```

## 6. 流式 VCR

### 6.1 VCR 模式

```typescript
/**
 * VCR (Video Cassette Recorder) 模式
 * 
 * 支持录制真实 API 调用并在后续回放，用于：
 * - 测试（无需真实 API 调用）
 * - 调试（重现特定问题）
 * - 开发（离线工作）
 */

const VCR_ENABLED = process.env.CLAUDE_VCR === 'true';
const VCR_MODE = process.env.CLAUDE_VCR_MODE || 'record';  // 'record' | 'playback'
const VCR_DIR = process.env.CLAUDE_VCR_DIR || './.vcr';

/**
 * VCR 包装器
 */
export async function* withStreamingVCR<T>(
  messages: Message[],
  fn: () => AsyncGenerator<T>
): AsyncGenerator<T> {
  if (!VCR_ENABLED) {
    // VCR 禁用，直接执行
    return yield* fn();
  }
  
  // 生成 cassette ID（基于消息哈希）
  const cassetteId = generateCassetteId(messages);
  const cassettePath = path.join(VCR_DIR, `${cassetteId}.json`);
  
  if (VCR_MODE === 'playback') {
    // 回放模式
    if (await fs.access(cassettePath).then(() => true).catch(() => false)) {
      return yield* playbackCassette(cassettePath);
    }
    
    throw new Error(`Cassette not found: ${cassetteId}`);
  }
  
  // 录制模式
  const events: T[] = [];
  
  for await (const event of fn()) {
    events.push(event);
    yield event;
  }
  
  // 保存 cassette
  await fs.mkdir(VCR_DIR, { recursive: true });
  await fs.writeFile(
    cassettePath,
    JSON.stringify(events, null, 2)
  );
}

async function* playbackCassette<T>(path: string): AsyncGenerator<T> {
  const data = await fs.readFile(path, 'utf-8');
  const events: T[] = JSON.parse(data);
  
  for (const event of events) {
    // 模拟流式延迟
    await sleep(10);
    yield event;
  }
}

function generateCassetteId(messages: Message[]): string {
  // 基于消息内容生成哈希
  const content = JSON.stringify(messages);
  return crypto.createHash('md5').update(content).digest('hex').slice(0, 16);
}
```

## 7. 错误处理

### 7.1 API 错误处理

```typescript
/**
 * 处理 API 错误
 * 
 * 将不同类型的错误转换为统一格式。
 */
export async function* handleAPIError(
  error: any
): AsyncGenerator<StreamEvent> {
  const apiError = parseAPIError(error);
  
  yield {
    type: 'error',
    error: apiError,
  };
}

function parseAPIError(error: any): APIError {
  // Anthropic API 错误格式
  if (error.error) {
    return {
      code: error.error.code || 'unknown',
      message: error.error.message,
      status: error.status,
      type: error.error.type,
    };
  }
  
  // HTTP 错误
  if (error.status) {
    return {
      code: `http_${error.status}`,
      message: error.message || `HTTP ${error.status}`,
      status: error.status,
    };
  }
  
  // 网络错误
  if (error.code) {
    return {
      code: error.code,
      message: error.message,
    };
  }
  
  // 未知错误
  return {
    code: 'unknown',
    message: error.message || 'Unknown error',
  };
}

type APIError = {
  code: string;
  message: string;
  status?: number;
  type?: string;
};
```

### 7.2 流式错误处理

```typescript
/**
 * 安全流式包装器
 * 
 * 捕获流式过程中的错误，转换为 error 事件。
 */
export async function* safeStream<T>(
  stream: AsyncGenerator<T>
): AsyncGenerator<T | ErrorEvent> {
  try {
    yield* stream;
  } catch (error) {
    yield {
      type: 'error',
      error: parseAPIError(error),
    };
  }
}
```

## 8. SDK 客户端配置

### 8.1 客户端创建

**文件**: `src/services/api/client.ts`

```typescript
import Anthropic from '@anthropic-ai/sdk';

/**
 * Anthropic SDK 客户端配置
 */
let client: Anthropic | null = null;

export function getAnthropicClient(): Anthropic {
  if (!client) {
    const apiKey = getAPIKey();
    
    client = new Anthropic({
      apiKey,
      baseURL: getBaseURL(),
      timeout: 60000,  // 60 秒超时
      maxRetries: 0,   // 我们有自己的重试逻辑
    });
  }
  
  return client;
}

/**
 * 获取 API Key
 */
function getAPIKey(): string {
  // 1. 环境变量
  if (process.env.ANTHROPIC_API_KEY) {
    return process.env.ANTHROPIC_API_KEY;
  }
  
  // 2. 配置文件
  const config = loadConfig();
  if (config.apiKey) {
    return config.apiKey;
  }
  
  // 3. Keychain（macOS）
  if (process.platform === 'darwin') {
    const keychainKey = getKeychainKey();
    if (keychainKey) {
      return keychainKey;
    }
  }
  
  throw new Error('ANTHROPIC_API_KEY not found');
}

/**
 * 获取 Base URL
 * 
 * 支持自定义端点（如代理、内部部署）
 */
function getBaseURL(): string | undefined {
  // 1. 环境变量
  if (process.env.ANTHROPIC_BASE_URL) {
    return process.env.ANTHROPIC_BASE_URL;
  }
  
  // 2. 配置文件
  const config = loadConfig();
  if (config.baseURL) {
    return config.baseURL;
  }
  
  // 使用默认端点
  return undefined;
}

/**
 * 重置客户端（用于配置变更后）
 */
export function resetClient(): void {
  client = null;
}
```

### 8.2 多提供商支持

```typescript
/**
 * 支持多个 API 提供商
 */
type Provider = 'anthropic' | 'bedrock' | 'vertex' | 'azure';

interface ProviderConfig {
  provider: Provider;
  apiKey: string;
  baseURL?: string;
  region?: string;  // Bedrock/Vertex 需要
  projectId?: string;  // Vertex 需要
}

export function createClientForProvider(config: ProviderConfig): APIClient {
  switch (config.provider) {
    case 'anthropic':
      return new AnthropicClient(config);
    
    case 'bedrock':
      return new BedrockClient(config);
    
    case 'vertex':
      return new VertexClient(config);
    
    case 'azure':
      return new AzureClient(config);
    
    default:
      throw new Error(`Unknown provider: ${config.provider}`);
  }
}

// 通用客户端接口
interface APIClient {
  createMessage(params: MessageParams): AsyncGenerator<StreamEvent>;
  countTokens(messages: Message[]): Promise<number>;
}
```

## 9. 总结

API 层的设计体现了以下架构原则：

1. **流式优先**: Generator 模式实现真正的流式处理
2. **可恢复性**: 重试机制和降级策略保证服务可用性
3. **成本优化**: Prompt Caching 显著降低 API 成本
4. **可测试性**: VCR 模式支持离线测试
5. **多提供商**: 统一的接口支持多种 API 提供商
6. **类型安全**: TypeScript 提供完整的类型支持

这些设计使得 Claude Code 能够可靠、高效地与 Anthropic API 交互，同时保持良好的可维护性和可扩展性。
