# Claude Code CLI Token 优化策略详解

## 概述

Claude Code CLI 采用了多层次的 Token 优化策略，从提示缓存、上下文压缩、工具结果管理到消息截断等多个维度来降低 API 调用成本。本文档详细分析这些优化措施的实现原理和工程实践。

## 1. 提示缓存 (Prompt Caching)

### 1.1 缓存架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                     提示缓存架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐      ┌─────────────────────────────────┐  │
│  │  Static Content │      │       Dynamic Content           │  │
│  │  (Cross-org)    │      │  (User/Session-specific)        │  │
│  │                 │      │                                 │  │
│  │  • 系统提示基础 │      │  • 当前工作目录                 │  │
│  │  • 工具定义     │      │  • Git 状态                     │  │
│  │  • 通用指令     │      │  • CLAUDE.md 内容               │  │
│  │                 │      │  • 记忆文件                     │  │
│  └────────┬────────┘      └─────────────────────────────────┘  │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              SYSTEM_PROMPT_DYNAMIC_BOUNDARY             │   │
│  │                    (动态边界标记)                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  缓存策略:                                                      │
│  • Static: cache_scope='global' (跨组织共享)                    │
│  • Static: cache_scope='org'    (组织内共享)                    │
│  • Dynamic: cache_scope=null    (不缓存)                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 动态边界分割

```typescript
// src/constants/prompts.ts
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'

// src/utils/api.ts
export function splitSysPromptPrefix(
  systemPrompt: SystemPrompt,
): SystemPromptBlock[] {
  const useGlobalCacheFeature = shouldUseGlobalCacheScope()
  
  if (useGlobalCacheFeature) {
    const boundaryIndex = systemPrompt.findIndex(
      s => s === SYSTEM_PROMPT_DYNAMIC_BOUNDARY,
    )
    
    if (boundaryIndex !== -1) {
      // 边界前的内容标记为 global scope (可缓存)
      // 边界后的内容标记为 null scope (不缓存)
      const staticBlocks = systemPrompt.slice(0, boundaryIndex)
      const dynamicBlocks = systemPrompt.slice(boundaryIndex + 1)
      
      return [
        { text: staticBlocks.join('\n\n'), cacheScope: 'global' },
        { text: dynamicBlocks.join('\n\n'), cacheScope: null },
      ]
    }
  }
  
  return [{ text: systemPrompt.join('\n\n'), cacheScope: 'org' }]
}
```

### 1.3 缓存控制标记

```typescript
// Anthropic API 缓存控制
interface CacheControl {
  type: 'ephemeral'
  scope?: 'global' | 'org'  // 缓存范围
  ttl?: '5m' | '1h'         // 缓存有效期
}

// 系统提示块结构
interface SystemPromptBlock {
  text: string
  cacheScope: CacheScope | null  // 'global' | 'org' | null
}

// 工具缓存控制
interface BetaToolWithExtras {
  name: string
  description: string
  input_schema: InputSchema
  cache_control?: CacheControl
}
```

### 1.4 缓存命中策略

| 缓存范围 | 共享级别 | 适用场景 | Beta Header |
|---------|---------|---------|-------------|
| `global` | 跨组织 | 系统提示静态部分 | `prompt-caching-scope-2026-01-05` |
| `org` | 组织内 | 项目特定配置 | 默认支持 |
| `null` | 不缓存 | 会话特定内容 | - |

**缓存节省效果**：
- 缓存写入：成本约为标准输入 token 的 1.25 倍
- 缓存读取：成本约为标准输入 token 的 0.1 倍
- 对于 100K 的系统提示，缓存命中可节省约 90% 成本

## 2. 上下文压缩 (Context Compaction)

### 2.1 自动压缩触发机制

```typescript
// src/services/compact/autoCompact.ts

// 关键阈值配置
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000

// 有效上下文窗口 = 模型窗口 - 预留输出 tokens
export function getEffectiveContextWindowSize(model: string): number {
  const reservedTokensForSummary = Math.min(
    getMaxOutputTokensForModel(model),
    MAX_OUTPUT_TOKENS_FOR_SUMMARY, // 20,000
  )
  const contextWindow = getContextWindowForModel(model)
  
  return contextWindow - reservedTokensForSummary
}

// 自动压缩阈值
export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  return effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
}
```

### 2.2 压缩决策流程

```
Token 使用量监控
       │
       ▼
┌─────────────────┐     否      ┌─────────────────┐
│ 超过警告阈值?   │────────────►│ 正常对话        │
│ (窗口 - 20K)    │             │                 │
└─────────────────┘             └─────────────────┘
       │ 是
       ▼
┌─────────────────┐     否      ┌─────────────────┐
│ 超过自动压缩    │────────────►│ 显示警告        │
│ 阈值?           │             │                 │
│ (窗口 - 13K)    │             └─────────────────┘
└─────────────────┘
       │ 是
       ▼
┌─────────────────┐
│ 触发自动压缩    │
│ (异步 fork)     │
└─────────────────┘
       │
       ▼
┌─────────────────┐
│ 生成对话摘要    │
│ 替换历史消息    │
└─────────────────┘
```

### 2.3 压缩实现细节

```typescript
// src/services/compact/compact.ts

export interface CompactMetadata {
  // 被保留的消息段 (最近消息)
  preservedSegment: {
    headUuid: string  // 段起始消息 UUID
    tailUuid: string  // 段结束消息 UUID
    messageCount: number
  }
  // 被压缩的消息段 (旧消息)
  summarizedSegment: {
    headUuid: string
    tailUuid: string
    messageCount: number
    summary: string    // 摘要内容
  }
  // Token 统计
  tokenStats: {
    preCompact: number
    postCompact: number
    tokensFreed: number
  }
}

// 压缩边界消息
export interface SystemCompactBoundaryMessage {
  type: 'system'
  subtype: 'compact_boundary'
  compactMetadata: CompactMetadata
}
```

### 2.4 图像剥离优化

```typescript
// src/services/compact/compact.ts

/**
 * 压缩前移除用户消息中的图像块
 * 图像对生成摘要不必要，且可能触发提示过长错误
 */
export function stripImagesFromMessages(messages: Message[]): Message[] {
  return messages.map(message => {
    if (message.type !== 'user') return message
    
    const content = message.message.content
    if (!Array.isArray(content)) return message
    
    return {
      ...message,
      message: {
        ...message.message,
        content: content.flatMap(block => {
          // 替换图像为文本标记
          if (block.type === 'image') {
            return [{ type: 'text', text: '[image]' }]
          }
          if (block.type === 'document') {
            return [{ type: 'text', text: '[document]' }]
          }
          // 递归处理 tool_result 中的图像
          if (block.type === 'tool_result') {
            return [stripImagesFromToolResult(block)]
          }
          return [block]
        })
      }
    }
  })
}
```

### 2.5 Token 节省效果

| 场景 | 原始 Token | 压缩后 Token | 节省率 |
|------|-----------|-------------|--------|
| 长对话 (50 轮) | ~150K | ~30K (摘要) | ~80% |
| 多轮工具调用 | ~100K | ~25K (摘要) | ~75% |
| 包含图像的对话 | ~120K | ~20K (摘要) | ~83% |

## 3. 微压缩 (Micro-Compaction)

### 3.1 微压缩策略

微压缩是一种细粒度的 token 优化，在保持对话完整性的同时清除旧工具结果。

```typescript
// src/services/compact/microCompact.ts

// 可压缩工具列表
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
])

// 图像 token 估算 (固定值)
const IMAGE_MAX_TOKEN_SIZE = 2000

// 工具结果 Token 计算
function calculateToolResultTokens(block: ToolResultBlockParam): number {
  if (!block.content) return 0
  
  if (typeof block.content === 'string') {
    return roughTokenCountEstimation(block.content)
  }
  
  return block.content.reduce((sum, item) => {
    if (item.type === 'text') {
      return sum + roughTokenCountEstimation(item.text)
    } else if (item.type === 'image' || item.type === 'document') {
      return sum + IMAGE_MAX_TOKEN_SIZE
    }
    return sum
  }, 0)
}
```

### 3.2 时间基于的微压缩配置

```typescript
// src/services/compact/timeBasedMCConfig.ts

export interface TimeBasedMCConfig {
  // 消息年龄阈值 (分钟)
  messageAgeThresholdMinutes: number
  // 最小 token 节省阈值
  minTokensThreshold: number
  // 最大清除比例
  maxClearRatio: number
}

export function getTimeBasedMCConfig(): TimeBasedMCConfig {
  return {
    messageAgeThresholdMinutes: 5,  // 5 分钟前的消息
    minTokensThreshold: 1000,       // 至少节省 1000 tokens
    maxClearRatio: 0.3,             // 最多清除 30% 的消息
  }
}
```

### 3.3 微压缩效果

| 指标 | 数值 |
|------|------|
| 触发频率 | 每轮对话后自动评估 |
| 平均清除 Token | ~5K-15K |
| 对对话连续性影响 | 无 (保留消息结构) |
| 缓存命中率影响 | 低 (仅清除内容，保留消息框架) |

## 4. 工具结果管理

### 4.1 工具结果持久化

大型工具结果会被持久化到磁盘，避免占用上下文窗口。

```typescript
// src/constants/toolLimits.ts

// 默认最大结果大小 (字符)
export const DEFAULT_MAX_RESULT_SIZE_CHARS = 50_000

// 最大结果 Token 数 (~400KB 文本)
export const MAX_TOOL_RESULT_TOKENS = 100_000

// Token 到字节的估算 (保守估计)
export const BYTES_PER_TOKEN = 4
export const MAX_TOOL_RESULT_BYTES = MAX_TOOL_RESULT_TOKENS * BYTES_PER_TOKEN

// 单条消息中所有工具结果的聚合限制
export const MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200_000
```

### 4.2 持久化实现

```typescript
// src/utils/toolResultStorage.ts

export interface PersistedToolResult {
  filepath: string        // 持久化文件路径
  originalSize: number    // 原始大小
  isJson: boolean         // 是否为 JSON
  preview: string         // 预览内容 (前 2000 bytes)
  hasMore: boolean        // 是否有更多内容
}

// 处理大工具结果
export async function maybePersistLargeToolResult(
  toolResultBlock: ToolResultBlockParam,
  toolName: string,
  threshold: number,
): Promise<ToolResultBlockParam> {
  const contentStr = extractContentString(toolResultBlock)
  
  if (contentStr.length <= threshold) {
    return toolResultBlock  // 未超过阈值，直接返回
  }
  
  // 持久化到磁盘
  const persisted = await persistToolResult(contentStr, toolUseId)
  
  // 替换为引用消息
  return {
    type: 'tool_result',
    tool_use_id: toolUseId,
    content: buildLargeToolResultMessage(persisted),
  }
}

// 预览大小配置
export const PREVIEW_SIZE_BYTES = 2000

function buildLargeToolResultMessage(result: PersistedToolResult): string {
  return `<persisted-output>
Output too large (${formatFileSize(result.originalSize)}). 
Full output saved to: ${result.filepath}

Preview (first ${formatFileSize(PREVIEW_SIZE_BYTES)}):
${result.preview}
...
</persisted-output>`
}
```

### 4.3 动态阈值覆盖

```typescript
// GrowthBook 动态配置
const PERSIST_THRESHOLD_OVERRIDE_FLAG = 'tengu_satin_quoll'

export function getPersistenceThreshold(
  toolName: string,
  declaredMaxResultSizeChars: number,
): number {
  // 支持按工具配置不同阈值
  const overrides = getFeatureValue_CACHED_MAY_BE_STALE<Record<string, number>>(
    PERSIST_THRESHOLD_OVERRIDE_FLAG, 
    {}
  )
  
  if (overrides?.[toolName] > 0) {
    return overrides[toolName]
  }
  
  return Math.min(declaredMaxResultSizeChars, DEFAULT_MAX_RESULT_SIZE_CHARS)
}
```

## 5. Token 估算与预算管理

### 5.1 粗略 Token 估算

```typescript
// src/services/tokenEstimation.ts

// 默认 4 字节/token (保守估计)
export function roughTokenCountEstimation(
  content: string,
  bytesPerToken: number = 4,
): number {
  return Math.round(content.length / bytesPerToken)
}

// 按文件类型的估算
export function bytesPerTokenForFileType(fileExtension: string): number {
  switch (fileExtension) {
    case 'json':
    case 'jsonl':
    case 'jsonc':
      return 2  // JSON 符号密集，实际比例约 2
    default:
      return 4
  }
}
```

### 5.2 精确 Token 计数

```typescript
// 使用 Anthropic API countTokens 端点
export async function countMessagesTokensWithAPI(
  messages: Anthropic.Beta.Messages.BetaMessageParam[],
  tools: Anthropic.Beta.Messages.BetaToolUnion[],
): Promise<number | null> {
  const anthropic = await getAnthropicClient()
  
  const response = await anthropic.beta.messages.countTokens({
    model: normalizeModelStringForAPI(model),
    messages,
    tools,
    betas: filteredBetas,
    ...(containsThinking && {
      thinking: {
        type: 'enabled',
        budget_tokens: TOKEN_COUNT_THINKING_BUDGET, // 1024
      },
    }),
  })
  
  return response.input_tokens
}

// Haiku 备用方案 (成本更低)
export async function countTokensViaHaikuFallback(
  messages: Anthropic.Beta.Messages.BetaMessageParam[],
): Promise<number | null> {
  const model = getSmallFastModel() // Haiku
  // ... 使用 Haiku 进行 token 计数
}
```

### 5.3 Token 预算分配

```typescript
// src/utils/context.ts

// 模型上下文窗口
export const MODEL_CONTEXT_WINDOW_DEFAULT = 200_000

// 输出 Token 预留
export const MAX_OUTPUT_TOKENS_DEFAULT = 32_000
export const MAX_OUTPUT_TOKENS_UPPER_LIMIT = 64_000

// 优化：槽位预留优化 (Capped Default)
// BQ p99 输出 = 4,911 tokens，32k/64k 默认过度预留 8-16 倍
export const CAPPED_DEFAULT_MAX_TOKENS = 8_000
export const ESCALATED_MAX_TOKENS = 64_000

export function getModelMaxOutputTokens(model: string): {
  default: number
  upperLimit: number
} {
  if (model.includes('opus-4-6')) {
    return { default: 64_000, upperLimit: 128_000 }
  } else if (model.includes('sonnet-4-6')) {
    return { default: 32_000, upperLimit: 128_000 }
  }
  // ... 其他模型
}
```

## 6. 记忆系统 Token 优化

### 6.1 MEMORY.md 截断

```typescript
// src/memdir/memdir.ts

export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000

export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  const contentLines = raw.split('\n')
  const lineCount = contentLines.length
  const byteCount = Buffer.byteLength(raw, 'utf8')
  
  // 先按行截断
  let truncated = lineCount > MAX_ENTRYPOINT_LINES
    ? contentLines.slice(0, MAX_ENTRYPOINT_LINES).join('\n')
    : raw
  
  // 再按字节截断 (在行边界处)
  if (Buffer.byteLength(truncated, 'utf8') > MAX_ENTRYPOINT_BYTES) {
    const cutAt = truncated.lastIndexOf('\n', MAX_ENTRYPOINT_BYTES)
    truncated = truncated.slice(0, cutAt > 0 ? cutAt : MAX_ENTRYPOINT_BYTES)
  }
  
  return {
    content: truncated,
    wasLineTruncated: lineCount > MAX_ENTRYPOINT_LINES,
    wasByteTruncated: byteCount > MAX_ENTRYPOINT_BYTES,
    originalLineCount: lineCount,
    originalByteCount: byteCount,
  }
}
```

### 6.2 相关记忆选择

```typescript
// src/memdir/findRelevantMemories.ts

export async function findRelevantMemories(
  query: string,
  memories: MemoryHeader[],
): Promise<MemoryHeader[]> {
  // 使用 Sonnet 模型选择相关记忆 (最多 5 个)
  const selected = await sideQuery({
    systemPrompt: SELECT_MEMORIES_SYSTEM_PROMPT,
    userPrompt: `Query: ${query}\n\nAvailable memories:\n${manifest}`,
    model: 'claude-sonnet-4-6',
    maxTokens: 500,
  })
  
  return memories
    .filter(m => selected.includes(m.path))
    .slice(0, 5)  // 限制数量
}

// 会话记忆总预算
const MAX_SESSION_BYTES = 60 * 1024  // 60KB
```

## 7. 提示太长错误处理 (Prompt Too Long)

### 7.1 PTL 重试机制

```typescript
// src/services/compact/compact.ts

const PTL_RETRY_MARKER = '[earlier conversation truncated for compaction retry]'

export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: string,
): Message[] | null {
  // 从错误响应解析需要减少的 token 数
  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  if (tokenGap === undefined) return null
  
  // 按 API 轮次分组
  const groups = groupMessagesByApiRound(messages)
  
  let acc = 0
  const preservedGroups: typeof groups = []
  
  // 从最早的轮次开始丢弃
  for (let i = groups.length - 1; i >= 0; i--) {
    const group = groups[i]
    const groupTokens = roughTokenCountEstimation(
      JSON.stringify(group.messages)
    )
    
    if (acc + groupTokens >= tokenGap) {
      // 已累积足够 token，保留剩余轮次
      preservedGroups.unshift(...groups.slice(0, i))
      break
    }
    acc += groupTokens
  }
  
  if (preservedGroups.length === groups.length) {
    return null  // 无法进一步截断
  }
  
  // 插入截断标记
  return [
    createSystemMessage(PTL_RETRY_MARKER),
    ...flattenGroups(preservedGroups),
  ]
}
```

### 7.2 PTL 预防策略

```typescript
// src/services/compact/autoCompact.ts

export function calculateTokenWarningState(
  tokenUsage: number,
  model: string,
): TokenWarningState {
  const autoCompactThreshold = getAutoCompactThreshold(model)
  
  // 多级阈值警告
  const warningThreshold = threshold - WARNING_THRESHOLD_BUFFER_TOKENS  // -20K
  const errorThreshold = threshold - ERROR_THRESHOLD_BUFFER_TOKENS      // -20K
  
  return {
    percentLeft: Math.round(((threshold - tokenUsage) / threshold) * 100),
    isAboveWarningThreshold: tokenUsage >= warningThreshold,
    isAboveErrorThreshold: tokenUsage >= errorThreshold,
    isAboveAutoCompactThreshold: tokenUsage >= autoCompactThreshold,
    isAtBlockingLimit: tokenUsage >= blockingLimit,
  }
}
```

## 8. 1M 上下文窗口支持

### 8.1 模型检测与回退

```typescript
// src/utils/context.ts

export function modelSupports1M(model: string): boolean {
  if (is1mContextDisabled()) return false
  
  const canonical = getCanonicalName(model)
  return canonical.includes('claude-sonnet-4') || 
         canonical.includes('opus-4-6')
}

export function getContextWindowForModel(
  model: string,
  betas?: string[],
): number {
  // 1. Ant 内部覆盖
  if (process.env.USER_TYPE === 'ant' && 
      process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS) {
    return parseInt(process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS, 10)
  }
  
  // 2. [1m] 后缀显式启用
  if (has1mContext(model)) return 1_000_000
  
  // 3. Beta header
  if (betas?.includes(CONTEXT_1M_BETA_HEADER) && modelSupports1M(model)) {
    return 1_000_000
  }
  
  // 4. 实验性治疗
  if (getSonnet1mExpTreatmentEnabled(model)) return 1_000_000
  
  return MODEL_CONTEXT_WINDOW_DEFAULT  // 200K
}
```

### 8.2 1M 上下文优化

| 功能 | 200K 窗口 | 1M 窗口 |
|------|----------|---------|
| 自动压缩触发 | ~187K | ~987K |
| 工具结果保留 | 短 | 更长 |
| 对话历史 | 压缩更频繁 | 更完整保留 |
| 文件读取 | 谨慎 | 更激进 |

## 9. 综合 Token 优化效果

### 9.1 典型场景节省分析

```
场景: 100 轮对话，包含 50 次文件读取，20 次 Bash 执行

┌─────────────────────┬──────────┬──────────┬──────────┐
│ 优化策略            │ 原始     │ 优化后   │ 节省     │
├─────────────────────┼──────────┼──────────┼──────────┤
│ 提示缓存 (全局)     │ 50K      │ 5K       │ 90%      │
│ 提示缓存 (组织)     │ 30K      │ 3K       │ 90%      │
│ 上下文压缩          │ 200K     │ 40K      │ 80%      │
│ 微压缩              │ 50K      │ 35K      │ 30%      │
│ 工具结果持久化      │ 100K     │ 20K      │ 80%      │
│ 记忆系统截断        │ 30K      │ 10K      │ 67%      │
├─────────────────────┼──────────┼──────────┼──────────┤
│ 总计                │ 460K     │ 113K     │ 75%      │
└─────────────────────┴──────────┴──────────┴──────────┘
```

### 9.2 成本计算示例

假设使用 Claude Sonnet 4.6 ($3/MTok 输入, $15/MTok 输出):

| 场景 | 原始成本 | 优化后成本 | 节省 |
|------|---------|-----------|------|
| 单次长对话 | $1.38 | $0.34 | $1.04 (75%) |
| 每日 50 会话 | $69 | $17 | $52 (75%) |
| 月度成本 | $2,070 | $517 | $1,553 (75%) |

**注**: 缓存读取成本为输入成本的 10%，缓存写入成本为输入成本的 125%

## 10. 配置与调优

### 10.1 环境变量覆盖

```bash
# 禁用自动压缩
export DISABLE_AUTO_COMPACT=1

# 自定义自动压缩窗口
export CLAUDE_CODE_AUTO_COMPACT_WINDOW=150000

# 禁用 1M 上下文 (HIPAA 合规)
export CLAUDE_CODE_DISABLE_1M_CONTEXT=1

# 最大上下文覆盖 (Ant only)
export CLAUDE_CODE_MAX_CONTEXT_TOKENS=150000

# 禁用自动记忆
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=1

# 强制紧凑预览大小
export CLAUDE_CODE_TOOL_RESULT_PREVIEW_SIZE=5000

# 禁用实验性 Beta
export CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1
```

### 10.2 用户设置

```json
{
  "autoCompactEnabled": true,
  "memory": {
    "enabled": true,
    "maxMemoriesPerSession": 5,
    "maxMemorySize": 4096
  }
}
```

## 11. 最佳实践

### 11.1 对话管理建议

1. **定期手动压缩**: 在长时间对话后使用 `/compact` 命令
2. **合理使用 `/clear`**: 在切换任务时清除上下文
3. **避免重复读取**: 使用文件缓存避免重复读取相同文件
4. **控制工具输出**: 使用 `head`/`tail` 限制 Bash 工具输出

### 11.2 Token 监控

```typescript
// 启用 Token 使用调试
process.env.DEBUG = 'claude-code:tokens:*'

// 查看 Token 统计
// 在对话中显示: [Tokens: 45.2K/200K (23%)]
```

### 11.3 故障排除

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 频繁自动压缩 | 对话过长 | 使用 `/compact` 手动压缩 |
| 提示太长错误 | 单条消息过大 | 工具结果持久化或分页 |
| 缓存未命中 | 动态内容变化 | 检查 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` |
| 成本过高 | 缓存策略不当 | 启用 global scope 缓存 |

---

**注意**: 本文档基于 Claude Code CLI 源码分析，实际优化效果可能因使用场景而异。
