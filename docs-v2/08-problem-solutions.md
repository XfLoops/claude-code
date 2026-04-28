# 问题解决方案

## 1. TSC 错误处理

### 1.1 问题描述

- **数量**: ~1341 个 TypeScript 错误
- **来源**: 反编译/逆向工程过程
- **类型**: `unknown`/`never`/`{}` 类型推断失败

### 1.2 解决方案

```typescript
// 策略：接受错误，不影响运行时
// 原因：Bun 运行时直接执行 TypeScript，不依赖 tsc 输出

// 1. 新代码使用严格类型
function newFunction(): string {
  return "typed";
}

// 2. 遗留代码保留宽松类型
const legacyData: any = fetchData();

// 3. 渐进式改进
// - 新模块使用严格类型
// - 修改遗留代码时添加类型
// - 不主动修复未改动代码
```

### 1.3 配置调整

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": false,     // 关闭严格模式
    "noEmit": true,      // 不输出文件
    "skipLibCheck": true // 跳过库类型检查
  }
}
```

## 2. Feature Flag 死代码消除

### 2.1 问题描述

需要支持多种构建配置，内部功能不能出现在外部构建中。

### 2.2 解决方案

```typescript
// 构建时宏（bun:bundle）
import { feature } from 'bun:bundle';

// 条件代码块在构建时消除
if (feature('COORDINATOR_MODE')) {
  // 这段代码在 external 构建中不存在
  await import('./coordinator/mode.js');
}

// 运行时 polyfill（开发/测试）
const feature = (_name: string) => false;
```

### 2.3 实现机制

```
构建时（bun:bundle）：
- feature() 被评估为常量
- false 分支的代码被完全移除
- 不进入最终产物

运行时（cli.tsx polyfill）：
- feature() 始终返回 false
- 代码存在但不执行
- 零运行时开销
```

## 3. 工具系统权限三层架构

### 3.1 问题描述

需要在不同层级控制工具执行权限，确保安全。

### 3.2 解决方案

```typescript
// 1. 输入验证层
async validateInput(input, context): Promise<ValidationResult> {
  if (input.command.includes('sudo')) {
    return { success: false, error: 'sudo not allowed' };
  }
  return { success: true };
}

// 2. 权限检查层
async checkPermissions(input, context): Promise<PermissionResult> {
  if (isDestructive(input)) {
    return { 
      behavior: 'ask', 
      message: 'This will delete files. Continue?' 
    };
  }
  return { behavior: 'allow' };
}

// 3. 运行时决策层
type CanUseToolFn = (tool: Tool, input: Record<string, unknown>) => Promise<PermissionResult>;

const permission = await canUseTool(this, input);
if (permission.behavior === 'deny') {
  throw new PermissionDeniedError(permission.reason);
}
```

## 4. 错误传播与恢复

### 4.1 问题描述

复杂异步流程中需要正确传播和恢复错误。

### 4.2 解决方案

```typescript
// 1. 工具错误转换为消息
async function executeTool(tool, input): Promise<ToolResult> {
  try {
    const result = await tool.call(input, ...);
    return { type: 'success', result };
  } catch (error) {
    return { 
      type: 'error', 
      newMessages: [{
        role: 'user',
        content: [{ type: 'tool_result', content: `Error: ${error.message}` }]
      }]
    };
  }
}

// 2. API 错误特殊消息类型
export type AssistantAPIErrorMessage = {
  type: 'assistant_api_error';
  error: APIError;
  usage: Usage;
};

// 3. 流式错误传播
async function* queryModel(): AsyncGenerator<StreamEvent> {
  try {
    yield* streamAPI();
  } catch (error) {
    yield { type: 'error', error };
  }
}
```

## 5. 循环依赖打破

### 5.1 问题描述

工具之间可能存在循环依赖。

### 5.2 解决方案

```typescript
// 1. 懒加载函数模式
import type { ToolBType } from './ToolB.js';

export const ToolA = buildTool({
  name: 'A',
  async call() {
    // 运行时动态加载
    const { ToolB } = await import('./ToolB.js');
    await ToolB.call(...);
  }
});

// 2. 类型导入与实际导入分离
import type { SomeType } from './module.js';  // 仅类型
const module = await import('./module.js');   // 运行时加载
```

## 6. Token 超限处理

### 6.1 问题描述

长对话超出模型上下文窗口。

### 6.2 解决方案

```typescript
// 1. Auto Compact
async function autoCompact(messages: Message[]): Promise<Message[]> {
  const tokenCount = await countTokens(messages);
  if (tokenCount < THRESHOLD) return messages;
  
  const recent = messages.slice(-KEEP_RECENT);
  const older = messages.slice(0, -KEEP_RECENT);
  const summary = await summarizeMessages(older);
  
  return [
    { role: 'system', content: `Summary: ${summary}` },
    ...recent
  ];
}

// 2. History Snip
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

// 3. Context Collapse
async function collapseContext(messages: Message[]): Promise<Message[]> {
  const topics = extractTopics(messages);
  const resolved = topics.filter(t => t.isResolved);
  const active = topics.filter(t => !t.isResolved);
  
  const resolvedSummary = await summarizeTopics(resolved);
  const activeMessages = getMessagesForTopics(active, messages);
  
  return [
    { role: 'system', content: `Resolved: ${resolvedSummary}` },
    ...activeMessages
  ];
}
```

## 7. 长消息流式处理

### 7.1 问题描述

大文件或长输出导致内存增长。

### 7.2 解决方案

```typescript
// 1. Generator 模式实现可中断流式
async function* queryLoop(params: QueryLoopParams): AsyncGenerator<QueryResult> {
  while (true) {
    const stream = await callAPI(params);
    
    for await (const event of stream) {
      if (params.signal.aborted) {
        stream.cancel();
        throw new AbortError();
      }
      
      yield event;
    }
  }
}

// 2. 工具结果大小限制
const MAX_RESULT_SIZE = 100_000;

async function executeTool(tool, input): Promise<ToolResult> {
  const result = await tool.call(input);
  
  if (result.length > MAX_RESULT_SIZE) {
    const filePath = await persistToDisk(result);
    return { result: `[Saved to ${filePath}]`, filePath };
  }
  
  return { result };
}
```

## 8. 快速启动优化

### 8.1 问题描述

大型应用启动时间过长。

### 8.2 解决方案

```typescript
// 1. 零导入快速路径
if (args.length === 1 && args[0] === "--version") {
  console.log(MACRO.VERSION);
  return;  // 立即退出，无额外加载
}

// 2. 并行初始化
startMdmRawRead();        // 并行
startKeychainPrefetch();  // 并行

// 主流程继续
const { QueryEngine } = await import('./QueryEngine.js');

// 3. 动态导入延迟加载
const { LargeModule } = await import('./LargeModule.js');
```

## 9. 总结

核心问题解决方案：

1. **TSC 错误**: 接受并渐进改进，不影响运行时
2. **Feature Flag**: 构建时死代码消除
3. **权限控制**: 三层架构（验证/检查/决策）
4. **错误处理**: 转换为消息，流式传播
5. **循环依赖**: 懒加载 + 类型导入分离
6. **Token 超限**: 三层压缩策略
7. **流式处理**: Generator + 中止信号
8. **启动优化**: 快速路径 + 并行 + 动态导入
