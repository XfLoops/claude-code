# API 层详解

## API 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      API 架构                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    services/api/                        │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│  │  │  claude.ts  │  │ bootstrap.ts│  │  filesApi.ts│     │   │
│  │  │ (核心API)   │  │ (启动数据)  │  │ (文件下载)  │     │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│  │  │  withRetry  │  │   errors.ts │  │   dumpPrompt│     │   │
│  │  │ (重试逻辑)   │  │ (错误处理)  │  │ (调试)      │     │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  多提供商支持                            │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│  │  │Anthropic│ │ Bedrock │ │ Vertex  │ │  Azure  │       │   │
│  │  │ 直连    │ │  AWS    │ │ Google  │ │ 微软    │       │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 核心 API 客户端

### claude.ts 主 API

```typescript
// src/services/api/claude.ts

import { Anthropic } from '@anthropic-ai/sdk'
import type {
  BetaMessageStreamParams,
  BetaMessage,
  BetaRawMessageStreamEvent,
  BetaUsage,
} from '@anthropic-ai/sdk/resources/beta/messages/messages.mjs'

// ==================== 流式 API 调用 ====================

export async function* streamClaudeAPI(
  params: BetaMessageStreamParams,
  options: APIOptions
): AsyncGenerator<StreamEvent, void> {
  const client = getAPIClient(options.provider)
  
  // 记录请求开始
  const requestStartTime = Date.now()
  logAPIRequest(params)
  
  try {
    // 创建流
    const stream = await client.messages.create({
      ...params,
      stream: true,
    })
    
    // 首 token 时间
    let firstTokenTime: number | undefined
    
    // 处理流事件
    for await (const event of stream) {
      // 记录首 token
      if (!firstTokenTime) {
        firstTokenTime = Date.now() - requestStartTime
        logFirstTokenLatency(firstTokenTime)
      }
      
      // 转换事件
      const streamEvent = convertToStreamEvent(event)
      yield streamEvent
      
      // 处理使用统计
      if (event.type === 'message_delta' && event.usage) {
        accumulateUsage(event.usage)
      }
    }
    
    // 记录总耗时
    const totalDuration = Date.now() - requestStartTime
    logAPIDuration(totalDuration)
    
  } catch (error) {
    // 分类错误
    const categorized = categorizeAPIError(error)
    
    if (categorized.retryable) {
      throw new RetryableAPIError(error, categorized.delayMs)
    }
    
    throw error
  }
}

// ==================== 请求构建 ====================

export function buildAPIRequest(
  params: BuildAPIRequestParams
): BetaMessageStreamParams {
  const {
    messages,
    systemPrompt,
    tools,
    model,
    maxTokens,
    temperature,
    betas,
    thinkingConfig,
  } = params
  
  return {
    model,
    max_tokens: maxTokens,
    temperature,
    messages: normalizeMessagesForAPI(messages),
    system: systemPrompt.map(part => ({
      type: 'text',
      text: part,
    })),
    tools: tools.map(toolToAPISchema),
    tool_choice: { type: 'auto' },
    // Beta 功能
    betas: buildBetaHeaders(betas),
    // Thinking 配置
    ...(thinkingConfig && {
      thinking: {
        type: 'enabled',
        budget_tokens: thinkingConfig.budgetTokens,
      },
    }),
    // 其他选项
    metadata: {
      session_id: getSessionId(),
    },
  }
}

// ==================== Beta Headers ====================

function buildBetaHeaders(betas: string[]): Record<string, string> {
  const headers: Record<string, string> = {}
  
  for (const beta of betas) {
    switch (beta) {
      case 'prompt-caching':
        headers['anthropic-beta'] = 'prompt-caching-2024-07-31'
        break
      case 'computer-use':
        headers['anthropic-beta'] = 'computer-use-2024-10-22'
        break
      case 'extended-output':
        headers['anthropic-beta'] = 'output-128k-2025-02-19'
        break
      case '1m-context':
        headers['anthropic-beta'] = 'max-tokens-3-5-sonnet-2024-07-15'
        break
    }
  }
  
  return headers
}

// ==================== 工具转换 ====================

export function toolToAPISchema(tool: Tool): BetaToolUnion {
  return {
    name: tool.name,
    description: typeof tool.description === 'string'
      ? tool.description
      : tool.description(),
    input_schema: tool.inputSchema as ToolInputSchema,
  }
}

// ==================== 消息标准化 ====================

export function normalizeMessagesForAPI(
  messages: Message[]
): MessageParam[] {
  return messages
    .filter(shouldIncludeInAPI)
    .map(normalizeMessageForAPI)
    .filter(isValidMessageParam)
}

function normalizeMessageForAPI(message: Message): MessageParam | null {
  switch (message.type) {
    case 'user':
      return {
        role: 'user',
        content: normalizeContent(message.content),
      }
    case 'assistant':
      return {
        role: 'assistant',
        content: normalizeAssistantContent(message),
      }
    case 'system':
      // 系统消息在 API 中作为顶层参数
      return null
    default:
      return null
  }
}
```

## 多提供商支持

### 提供商选择

```typescript
// src/utils/model/providers.ts

export type APIProvider = 
  | 'anthropic' 
  | 'bedrock' 
  | 'vertex' 
  | 'azure'

export function getAPIProvider(): APIProvider {
  // 1. 检查环境变量
  if (process.env.CLAUDE_CODE_USE_BEDROCK) {
    return 'bedrock'
  }
  if (process.env.CLAUDE_CODE_USE_VERTEX) {
    return 'vertex'
  }
  if (process.env.CLAUDE_CODE_USE_AZURE) {
    return 'azure'
  }
  
  // 2. 检查配置
  const settings = getGlobalConfig()
  if (settings.apiProvider) {
    return settings.apiProvider
  }
  
  // 3. 默认 Anthropic
  return 'anthropic'
}

// ==================== Anthropic 客户端 ====================

import { Anthropic } from '@anthropic-ai/sdk'

function createAnthropicClient(): Anthropic {
  const apiKey = getAnthropicAPIKey()
  const baseURL = process.env.ANTHROPIC_API_BASE_URL
  
  return new Anthropic({
    apiKey,
    baseURL,
    httpAgent: getProxyAgent(),
    maxRetries: 3,
    timeout: 600_000, // 10分钟
  })
}

// ==================== Bedrock 客户端 ====================

import { AnthropicBedrock } from '@anthropic-ai/bedrock-sdk'

function createBedrockClient(): AnthropicBedrock {
  const region = process.env.AWS_REGION || 'us-east-1'
  
  return new AnthropicBedrock({
    awsRegion: region,
    awsAccessKey: getAwsAccessKey(),
    awsSecretKey: getAwsSecretKey(),
    awsSessionToken: getAwsSessionToken(),
  })
}

// ==================== Vertex 客户端 ====================

import { AnthropicVertex } from '@anthropic-ai/vertex-sdk'

function createVertexClient(): AnthropicVertex {
  const region = process.env.GOOGLE_CLOUD_REGION || 'us-east5'
  const projectId = getGcpProjectId()
  
  return new AnthropicVertex({
    region,
    projectId,
    // 使用默认的 GCP 认证链
  })
}

// ==================== Azure 客户端 ====================

function createAzureClient(): Anthropic {
  const endpoint = process.env.AZURE_OPENAI_ENDPOINT
  const apiKey = process.env.AZURE_OPENAI_API_KEY
  const deployment = process.env.AZURE_OPENAI_DEPLOYMENT
  
  return new Anthropic({
    apiKey,
    baseURL: `${endpoint}/openai/deployments/${deployment}`,
    defaultHeaders: {
      'api-key': apiKey,
    },
  })
}
```

## 重试机制

### withRetry 包装器

```typescript
// src/services/api/withRetry.ts

export class FallbackTriggeredError extends Error {
  constructor(
    message: string,
    public readonly originalError: Error,
    public readonly fallbackModel: string
  ) {
    super(message)
    this.name = 'FallbackTriggeredError'
  }
}

export interface RetryConfig {
  maxRetries: number
  baseDelayMs: number
  maxDelayMs: number
  backoffMultiplier: number
  retryableErrors: Class<Error>[]
}

const DEFAULT_RETRY_CONFIG: RetryConfig = {
  maxRetries: 3,
  baseDelayMs: 1000,
  maxDelayMs: 30000,
  backoffMultiplier: 2,
  retryableErrors: [
    APIConnectionTimeoutError,
    APIConnectionError,
  ],
}

export async function withRetry<T>(
  fn: () => Promise<T>,
  config: Partial<RetryConfig> = {}
): Promise<T> {
  const fullConfig = { ...DEFAULT_RETRY_CONFIG, ...config }
  const { maxRetries, baseDelayMs, maxDelayMs, backoffMultiplier } = fullConfig
  
  let lastError: Error | undefined
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error as Error
      
      // 检查是否应该重试
      if (attempt >= maxRetries) {
        break
      }
      
      if (!isRetryableError(error, fullConfig)) {
        throw error
      }
      
      // 计算延迟
      const delayMs = Math.min(
        baseDelayMs * Math.pow(backoffMultiplier, attempt),
        maxDelayMs
      )
      
      logForDebugging(`Retry ${attempt + 1}/${maxRetries} after ${delayMs}ms: ${errorMessage(error)}`)
      
      // 等待后重试
      await sleep(delayMs)
    }
  }
  
  throw lastError ?? new Error('Max retries exceeded')
}

function isRetryableError(
  error: unknown,
  config: RetryConfig
): boolean {
  if (!(error instanceof Error)) return false
  
  // 检查 HTTP 状态码
  if (error instanceof APIError) {
    // 429 (Rate Limit) 和 5xx 错误可以重试
    if (error.status === 429 || (error.status >= 500 && error.status < 600)) {
      return true
    }
  }
  
  // 检查特定错误类型
  return config.retryableErrors.some(ErrorClass => 
    error instanceof ErrorClass
  )
}
```

## 错误处理

### API 错误分类

```typescript
// src/services/api/errors.ts

export interface APIErrorInfo {
  type: 'rate_limit' | 'token_limit' | 'server_error' | 'connection_error' | 'unknown'
  message: string
  status?: number
  retryable: boolean
  delayMs?: number
  fallbackModel?: string
}

export function categorizeAPIError(error: unknown): APIErrorInfo {
  if (!(error instanceof APIError)) {
    if (error instanceof APIConnectionTimeoutError) {
      return {
        type: 'connection_error',
        message: 'Connection timeout',
        retryable: true,
        delayMs: 2000,
      }
    }
    
    return {
      type: 'unknown',
      message: errorMessage(error),
      retryable: false,
    }
  }
  
  const status = error.status
  
  // 429 Rate Limit
  if (status === 429) {
    const retryAfter = parseRetryAfter(error.headers?.['retry-after'])
    return {
      type: 'rate_limit',
      message: 'Rate limit exceeded',
      status,
      retryable: true,
      delayMs: retryAfter ?? 5000,
    }
  }
  
  // 413 Payload Too Large / 400 Token Limit
  if (status === 413 || (status === 400 && isTokenLimitError(error))) {
    return {
      type: 'token_limit',
      message: PROMPT_TOO_LONG_ERROR_MESSAGE,
      status,
      retryable: true,
      fallbackModel: getSmallerModel(),
    }
  }
  
  // 5xx Server Error
  if (status >= 500 && status < 600) {
    return {
      type: 'server_error',
      message: `Server error: ${error.message}`,
      status,
      retryable: true,
      delayMs: 1000,
    }
  }
  
  // 其他 4xx
  if (status >= 400 && status < 500) {
    return {
      type: 'unknown',
      message: error.message,
      status,
      retryable: false,
    }
  }
  
  return {
    type: 'unknown',
    message: error.message,
    status,
    retryable: false,
  }
}

export const PROMPT_TOO_LONG_ERROR_MESSAGE = `
You've exceeded your limit for the current conversation.
The conversation will be compacted to continue.
`

export function isPromptTooLongMessage(message: string): boolean {
  return message.includes("exceeded your limit") ||
         message.includes("prompt is too long") ||
         message.includes("context limit")
}
```

## 流事件处理

### 流事件转换

```typescript
// src/services/api/streaming.ts

export type StreamEvent =
  | TextDeltaEvent
  | ThinkingDeltaEvent
  | ToolUseStartEvent
  | ToolUseDeltaEvent
  | ToolUseStopEvent
  | MessageStopEvent
  | UsageEvent
  | ErrorEvent

interface TextDeltaEvent {
  type: 'text_delta'
  text: string
}

interface ThinkingDeltaEvent {
  type: 'thinking_delta'
  thinking: string
}

interface ToolUseStartEvent {
  type: 'tool_use_start'
  toolUse: ToolUseBlock
}

interface ToolUseDeltaEvent {
  type: 'tool_use_delta'
  toolUseId: string
  delta: {
    partial_json?: string
  }
}

interface ToolUseStopEvent {
  type: 'tool_use_stop'
  toolUseId: string
}

interface MessageStopEvent {
  type: 'message_stop'
  stopReason: BetaStopReason
}

interface UsageEvent {
  type: 'usage'
  usage: BetaUsage
}

interface ErrorEvent {
  type: 'error'
  error: Error
}

export function convertToStreamEvent(
  rawEvent: BetaRawMessageStreamEvent
): StreamEvent {
  switch (rawEvent.type) {
    case 'content_block_delta':
      if (rawEvent.delta.type === 'text_delta') {
        return {
          type: 'text_delta',
          text: rawEvent.delta.text,
        }
      }
      if (rawEvent.delta.type === 'thinking_delta') {
        return {
          type: 'thinking_delta',
          thinking: rawEvent.delta.thinking,
        }
      }
      if (rawEvent.delta.type === 'partial_json') {
        return {
          type: 'tool_use_delta',
          toolUseId: rawEvent.content_block.id,
          delta: { partial_json: rawEvent.delta.partial_json },
        }
      }
      break
      
    case 'content_block_start':
      if (rawEvent.content_block.type === 'tool_use') {
        return {
          type: 'tool_use_start',
          toolUse: {
            id: rawEvent.content_block.id,
            name: rawEvent.content_block.name,
            input: {},
          },
        }
      }
      break
      
    case 'content_block_stop':
      return {
        type: 'tool_use_stop',
        toolUseId: rawEvent.content_block.id,
      }
      
    case 'message_delta':
      if (rawEvent.stop_reason) {
        return {
          type: 'message_stop',
          stopReason: rawEvent.stop_reason,
        }
      }
      if (rawEvent.usage) {
        return {
          type: 'usage',
          usage: rawEvent.usage,
        }
      }
      break
  }
  
  throw new Error(`Unknown event type: ${rawEvent.type}`)
}
```

## 令牌使用跟踪

### 使用统计

```typescript
// src/services/api/claude.ts

let accumulatedUsage: BetaUsage = {
  input_tokens: 0,
  output_tokens: 0,
  cache_creation_input_tokens: 0,
  cache_read_input_tokens: 0,
}

export function accumulateUsage(usage: BetaUsage): void {
  accumulatedUsage.input_tokens += usage.input_tokens ?? 0
  accumulatedUsage.output_tokens += usage.output_tokens ?? 0
  accumulatedUsage.cache_creation_input_tokens += usage.cache_creation_input_tokens ?? 0
  accumulatedUsage.cache_read_input_tokens += usage.cache_read_input_tokens ?? 0
}

export function getAccumulatedUsage(): BetaUsage {
  return { ...accumulatedUsage }
}

export function resetAccumulatedUsage(): void {
  accumulatedUsage = {
    input_tokens: 0,
    output_tokens: 0,
    cache_creation_input_tokens: 0,
    cache_read_input_tokens: 0,
  }
}

export function updateUsage(
  total: NonNullableUsage,
  delta: BetaUsage
): NonNullableUsage {
  return {
    inputTokens: total.inputTokens + (delta.input_tokens ?? 0),
    outputTokens: total.outputTokens + (delta.output_tokens ?? 0),
    cacheCreationInputTokens: total.cacheCreationInputTokens + 
      (delta.cache_creation_input_tokens ?? 0),
    cacheReadInputTokens: total.cacheReadInputTokens + 
      (delta.cache_read_input_tokens ?? 0),
  }
}
```

## 启动数据获取

### Bootstrap API

```typescript
// src/services/api/bootstrap.ts

export interface BootstrapData {
  // 用户配置
  userSettings: UserSettings
  
  // 策略限制
  policyLimits: PolicyLimits
  
  // 远程管理设置
  remoteManagedSettings: RemoteManagedSettings
  
  // 配额状态
  quotaStatus: QuotaStatus
  
  // 模型可用性
  modelAvailability: Record<string, boolean>
  
  // 功能开关
  featureFlags: Record<string, boolean>
}

export async function fetchBootstrapData(): Promise<BootstrapData> {
  // 并行获取所有启动数据
  const [
    userSettings,
    policyLimits,
    remoteManagedSettings,
    quotaStatus,
  ] = await Promise.all([
    fetchUserSettings(),
    fetchPolicyLimits(),
    fetchRemoteManagedSettings(),
    checkQuotaStatus(),
  ])
  
  return {
    userSettings,
    policyLimits,
    remoteManagedSettings,
    quotaStatus,
    modelAvailability: getModelAvailability(),
    featureFlags: getFeatureFlags(),
  }
}

async function fetchUserSettings(): Promise<UserSettings> {
  try {
    const response = await fetch(`${API_BASE}/v1/settings`, {
      headers: {
        Authorization: `Bearer ${getAccessToken()}`,
      },
    })
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`)
    }
    
    return await response.json()
  } catch (error) {
    logForDebugging(`Failed to fetch user settings: ${errorMessage(error)}`)
    return getDefaultUserSettings()
  }
}
```

## 调试与日志

### API 日志记录

```typescript
// src/services/api/logging.ts

export interface APILogEntry {
  timestamp: string
  requestId: string
  method: string
  path: string
  model: string
  inputTokens: number
  outputTokens: number
  durationMs: number
  error?: string
}

export function logAPIRequest(params: BetaMessageStreamParams): void {
  if (!isDebugMode()) return
  
  logForDebugging('API Request:', {
    model: params.model,
    max_tokens: params.max_tokens,
    message_count: params.messages.length,
    tool_count: params.tools?.length,
    betas: params.betas,
  })
}

export function logAPIResponse(
  response: BetaMessage,
  durationMs: number
): void {
  if (!isDebugMode()) return
  
  logForDebugging('API Response:', {
    stop_reason: response.stop_reason,
    usage: response.usage,
    duration_ms: durationMs,
  })
}

export function logAPIError(error: Error, context?: Record<string, unknown>): void {
  logForDiagnosticsNoPII('error', 'api_error', {
    error: error.message,
    error_type: error.name,
    ...context,
  })
}
```

## 模型管理

### 模型配置

```typescript
// src/utils/model/model.ts

export interface ModelConfig {
  id: string
  name: string
  maxTokens: number
  contextWindow: number
  supportsThinking: boolean
  supportsEffort: boolean
  supportsPromptCaching: boolean
  costPer1KInput: number
  costPer1KOutput: number
}

const MODELS: Record<string, ModelConfig> = {
  'claude-opus-4-6': {
    id: 'claude-opus-4-6',
    name: 'Claude Opus 4.6',
    maxTokens: 8192,
    contextWindow: 200_000,
    supportsThinking: true,
    supportsEffort: true,
    supportsPromptCaching: true,
    costPer1KInput: 15.00,
    costPer1KOutput: 75.00,
  },
  'claude-sonnet-4-6': {
    id: 'claude-sonnet-4-6',
    name: 'Claude Sonnet 4.6',
    maxTokens: 8192,
    contextWindow: 200_000,
    supportsThinking: false,
    supportsEffort: true,
    supportsPromptCaching: true,
    costPer1KInput: 3.00,
    costPer1KOutput: 15.00,
  },
  'claude-haiku-4-5-20251001': {
    id: 'claude-haiku-4-5-20251001',
    name: 'Claude Haiku 4.5',
    maxTokens: 4096,
    contextWindow: 200_000,
    supportsThinking: false,
    supportsEffort: false,
    supportsPromptCaching: true,
    costPer1KInput: 0.25,
    costPer1KOutput: 1.25,
  },
}

export function getModelConfig(modelId: string): ModelConfig {
  return MODELS[modelId] ?? MODELS['claude-sonnet-4-6']
}

export function getDefaultMainLoopModel(): string {
  const settings = getGlobalConfig()
  return settings.defaultModel ?? 'claude-sonnet-4-6'
}

export function parseUserSpecifiedModel(
  modelString: string
): ModelConfig {
  // 解析模型字符串，支持别名
  const normalized = normalizeModelString(modelString)
  return getModelConfig(normalized)
}
```
