# 性能优化策略

## 1. Prompt Caching 优化

### 1.1 缓存策略

```typescript
// 缓存控制头生成
export function getCacheControl({ scope, querySource }): CacheControl {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),
    ...(scope === 'global' && { scope }),
  };
}

// 1小时 TTL 资格检查
let userEligible = getPromptCache1hEligible();
if (userEligible === null) {
  userEligible = process.env.USER_TYPE === 'ant' || ...;
  setPromptCache1hEligible(userEligible);
}
```

### 1.2 缓存配置

| 内容类型 | 缓存策略 | 说明 |
|----------|----------|------|
| System Prompt | 全局缓存 | 跨查询共享 |
| Tools Description | 全局缓存 | 减少重复传输 |
| Conversation History | 渐进式缓存 | 新消息附加 |
| Context (git status) | 不缓存 | 经常变化 |

### 1.3 缓存统计

```typescript
type CacheStats = {
  cacheCreationInputTokens: number;
  cacheReadInputTokens: number;
  inputTokens: number;
  outputTokens: number;
};

function calculateCacheSavings(stats: CacheStats): Savings {
  const cacheWriteCost = stats.cacheCreationInputTokens * 1.25;
  const cacheReadCost = stats.cacheReadInputTokens * 0.1;
  const normalCost = stats.inputTokens * 1;
  
  const withoutCache = (stats.cacheCreationInputTokens + stats.cacheReadInputTokens) * 1;
  const actualCost = cacheWriteCost + cacheReadCost + normalCost;
  
  return {
    savedCost: withoutCache - actualCost,
    percentage: ((withoutCache - actualCost) / withoutCache) * 100,
  };
}
```

## 2. 消息历史压缩

### 2.1 Auto Compact

```typescript
async function autoCompact(messages: Message[]): Promise<Message[]> {
  const tokenCount = await countTokens(messages);
  if (tokenCount < THRESHOLD) return messages;
  
  const recent = messages.slice(-KEEP_RECENT);
  const older = messages.slice(0, -KEEP_RECENT);
  const summary = await summarizeMessages(older);
  
  return [
    { role: 'system', content: `Previous conversation summary:\n${summary}` },
    ...recent
  ];
}
```

### 2.2 History Snip

```typescript
function snipMessages(messages: Message[], maxTokens: number): Message[] {
  let total = 0;
  const result = [];
  
  for (let i = messages.length - 1; i >= 0; i--) {
    const tokens = estimateTokens(messages[i]);
    if (total + tokens > maxTokens) break;
    total += tokens;
    result.unshift(messages[i]);
  }
  
  return result;
}
```

### 2.3 Context Collapse

```typescript
async function collapseContext(messages: Message[]): Promise<Message[]> {
  const topics = extractTopics(messages);
  const resolved = topics.filter(t => t.isResolved);
  const active = topics.filter(t => !t.isResolved);
  
  const resolvedSummary = await summarizeTopics(resolved);
  const activeMessages = getMessagesForTopics(active, messages);
  
  return [
    { role: 'system', content: `Resolved topics:\n${resolvedSummary}` },
    ...activeMessages
  ];
}
```

## 3. 流式处理优化

### 3.1 Streaming Tool Executor

```typescript
class StreamingToolExecutor {
  private pendingTools: Map<string, ToolExecution> = new Map();
  private completedResults: ToolExecutionResult[] = [];
  
  addTool(toolBlock: ToolUseBlock, assistantMessage: AssistantMessage): void {
    const execution = this.executeTool(toolBlock, assistantMessage);
    this.pendingTools.set(toolBlock.id, execution);
  }
  
  getCompletedResults(): ToolExecutionResult[] {
    return this.completedResults.splice(0);
  }
  
  async *getRemainingResults(): AsyncGenerator<ToolExecutionResult> {
    while (this.pendingTools.size > 0) {
      const completed = await Promise.race(
        Array.from(this.pendingTools.entries()).map(async ([id, execution]) => {
          const result = await execution.promise;
          return { id, result };
        })
      );
      
      this.pendingTools.delete(completed.id);
      yield completed.result;
    }
  }
  
  discard(): void {
    for (const [id, execution] of this.pendingTools) {
      execution.abort();
    }
    this.pendingTools.clear();
  }
}
```

### 3.2 并发执行

```typescript
async function executeToolsConcurrently(
  toolCalls: ToolCall[],
  tools: Tool[]
): Promise<ToolResult[]> {
  const concurrentSafe = toolCalls.filter(call => {
    const tool = findToolByName(call.name);
    return tool?.isConcurrencySafe(call.input);
  });
  
  const sequential = toolCalls.filter(call => {
    const tool = findToolByName(call.name);
    return !tool?.isConcurrencySafe(call.input);
  });
  
  const results: ToolResult[] = [];
  
  if (concurrentSafe.length > 0) {
    const concurrentResults = await Promise.all(
      concurrentSafe.map(call => executeSingleTool(call, tools))
    );
    results.push(...concurrentResults);
  }
  
  for (const call of sequential) {
    const result = await executeSingleTool(call, tools);
    results.push(result);
  }
  
  return results;
}
```

## 4. 工具延迟加载

### 4.1 实现机制

```typescript
const WebSearchTool = buildTool({
  name: 'WebSearch',
  shouldDefer: true,  // 不随初始提示词发送
  // ...
});

async function searchTools(query: string): Promise<Tool[]> {
  const allTools = getAllTools();
  const loadedTools = allTools.filter(t => !t.shouldDefer);
  const deferredTools = allTools.filter(t => t.shouldDefer);
  
  const needed = await determineNeededTools(query, deferredTools);
  
  return [...loadedTools, ...needed];
}
```

### 4.2 延迟加载工具列表

- `WebBrowserTool` - 浏览器工具
- `WebSearchTool` - 搜索工具
- `MCPTool` - MCP 集成
- `ComputerUseTool` - 计算机使用
- `VoiceModeTool` - 语音模式

## 5. 启动优化

### 5.1 快速路径

```typescript
// 零导入快速路径
if (args.length === 1 && args[0] === "--version") {
  console.log(`${MACRO.VERSION} (Claude Code)`);
  return;  // 立即退出
}
```

### 5.2 并行初始化

```typescript
// 启动时并行预取
startMdmRawRead();        // 并行 MDM 检查
startKeychainPrefetch();  // 并行 Keychain 预取

// 然后继续加载
const { QueryEngine } = await import('./QueryEngine.js');
```

### 5.3 动态导入

```typescript
// 条件功能懒加载
const analyticsModule = feature('ANALYTICS')
  ? await import('./services/analytics/index.js')
  : null;

// 大型模块延迟加载
const { LargeModule } = await import('./LargeModule.js');
```

## 6. 内存管理

### 6.1 工具结果限制

```typescript
const MAX_RESULT_SIZE = 100_000;

async function executeTool(tool, input): Promise<ToolResult> {
  const result = await tool.call(input);
  
  if (result.length > MAX_RESULT_SIZE) {
    const filePath = await persistToDisk(result);
    return {
      result: `[Result too large, saved to ${filePath}]`,
      filePath
    };
  }
  
  return { result };
}
```

### 6.2 虚拟列表

```typescript
function useVirtualList<T>({ items, estimateSize, overscan = 5 }) {
  const { virtualItems, scrollToIndex } = useMemo(() => {
    const start = Math.floor(scrollOffset / estimateSize());
    const end = Math.ceil((scrollOffset + containerHeight) / estimateSize());
    
    const overscanStart = Math.max(0, start - overscan);
    const overscanEnd = Math.min(items.length - 1, end + overscan);
    
    const virtualItems = [];
    for (let i = overscanStart; i <= overscanEnd; i++) {
      virtualItems.push({
        index: i,
        offset: i * estimateSize(),
        size: estimateSize(),
        item: items[i],
      });
    }
    
    return { virtualItems, scrollToIndex };
  }, [items, scrollOffset, containerHeight]);
  
  return { virtualItems, scrollToIndex };
}
```

### 6.3 消息引用

```typescript
class QueryEngine {
  private messages: Message[];
  
  addMessage(message: Message) {
    // 直接引用，不复制
    this.messages.push(message);
  }
}
```

## 7. Token 预算管理

```typescript
function calculateTokenBudget(
  messages: Message[],
  maxTokens?: number
): TokenBudget {
  const modelLimit = 200000;
  const max = maxTokens ?? modelLimit;
  
  const inputTokens = estimateTokens(messages);
  const systemTokens = 4000;
  const safetyBuffer = 1000;
  
  const forResponse = max - inputTokens - systemTokens - safetyBuffer;
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
```

## 8. 重试与降级

### 8.1 指数退避

```typescript
function calculateBackoff(attempt: number, baseDelay: number): number {
  const exponentialDelay = baseDelay * Math.pow(2, attempt);
  const jitter = Math.random() * 1000;
  return Math.min(exponentialDelay + jitter, 30000);
}
```

### 8.2 模型降级链

```typescript
const MODEL_FALLBACK_CHAIN: Record<string, string[]> = {
  'claude-3-opus-20240229': [
    'claude-3-opus-20240229',
    'claude-3-sonnet-20240229',
    'claude-3-haiku-20240307',
  ],
  'claude-3-5-sonnet-20241022': [
    'claude-3-5-sonnet-20241022',
    'claude-3-sonnet-20240229',
    'claude-3-haiku-20240307',
  ],
};
```

## 9. 总结

性能优化核心策略：

1. **Prompt Caching**: 全局缓存 + 1小时 TTL，降低 API 成本
2. **消息压缩**: Auto Compact + History Snip + Context Collapse
3. **流式处理**: Generator 模式 + 并行工具执行
4. **工具延迟加载**: shouldDefer 减少初始提示词
5. **启动优化**: 快速路径 + 并行初始化 + 动态导入
6. **内存管理**: 结果大小限制 + 虚拟列表 + 引用传递
7. **Token 预算**: 精细分配，预留响应空间
8. **错误恢复**: 指数退避 + 模型降级
