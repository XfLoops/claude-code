# 状态管理

## 文件位置

- `src/state/AppState.tsx` - 应用状态类型和上下文
- `src/state/store.ts` - Zustand 风格 store 实现
- `src/bootstrap/state.ts` - 会话级全局状态
- `src/hooks/useAppState.ts` - React Hook 封装

## 1. 状态管理概述

Claude Code 使用分层状态管理架构，结合自定义 Zustand-style store 和 React Context。

### 1.1 架构分层

```
┌─────────────────────────────────────────────────────────────┐
│                     状态管理架构                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                   React 层 (UI 状态)                    │ │
│  │  - 组件本地状态 (useState)                            │ │
│  │  - 表单状态                                          │ │
│  │  - UI 临时状态（选中、展开等）                        │ │
│  └───────────────────────────────────────────────────────┘ │
│                           │                                 │
│                           │ useAppState()                   │
│                           ▼                                 │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                应用状态层 (App State)                   │ │
│  │  - 消息历史                                          │ │
│  │  - 当前会话                                          │ │
│  │  - 权限决策                                          │ │
│  │  - MCP 连接                                          │ │
│  └───────────────────────────────────────────────────────┘ │
│                           │                                 │
│                           │ Store.subscribe                 │
│                           ▼                                 │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                 Store 层 (业务状态)                     │ │
│  │  - 自定义 Zustand-style store                         │ │
│  │  - 不可变更新                                        │ │
│  │  - 订阅机制                                          │ │
│  │  - 持久化集成                                        │ │
│  └───────────────────────────────────────────────────────┘ │
│                           │                                 │
│                           │ 模块导入                        │
│                           ▼                                 │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                全局状态层 (Bootstrap)                   │ │
│  │  - 会话 ID                                           │ │
│  │  - 工作目录                                          │ │
│  │  - 项目根目录                                        │ │
│  │  - Token 计数                                        │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 状态分类

| 层级 | 状态类型 | 示例 | 生命周期 |
|------|----------|------|----------|
| **React 层** | UI 状态 | 输入框内容、选中项、弹窗开关 | 组件级 |
| **App State** | 业务状态 | 消息历史、工具权限、会话信息 | 会话级 |
| **Store** | 全局状态 | 主题、配置、用户信息 | 应用级 |
| **Bootstrap** | 运行时状态 | 会话ID、工作目录、token计数 | 进程级 |

## 2. Store 实现

### 2.1 核心 Store 类型

**文件**: `src/state/store.ts`

```typescript
/**
 * Store 类型定义
 * 
 * 基于 Zustand API 设计，但自定义实现
 */
export type Store<T> = {
  /** 获取当前状态 */
  getState: () => T;
  
  /** 
   * 更新状态
   * 使用函数式更新，支持不可变更新
   */
  setState: (updater: (prev: T) => T) => void;
  
  /** 
   * 订阅状态变化
   * 返回取消订阅函数
   */
  subscribe: (listener: Listener) => () => void;
};

type Listener = () => void;

type OnChange<T> = (change: { newState: T; oldState: T }) => void;

/**
 * 创建 Store
 * 
 * @param initialState - 初始状态
 * @param onChange - 可选的状态变化回调
 * @returns Store 实例
 */
export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>
): Store<T> {
  let state = initialState;
  const listeners = new Set<Listener>();
  
  return {
    getState: () => state,
    
    setState: (updater) => {
      const prev = state;
      const next = updater(prev);
      
      // 浅比较优化：如果状态未变，不触发更新
      if (Object.is(next, prev)) {
        return;
      }
      
      state = next;
      
      // 调用变化回调（用于持久化等）
      onChange?.({ newState: next, oldState: prev });
      
      // 通知所有订阅者
      for (const listener of listeners) {
        listener();
      }
    },
    
    subscribe: (listener) => {
      listeners.add(listener);
      
      // 返回取消订阅函数
      return () => {
        listeners.delete(listener);
      };
    },
  };
}
```

### 2.2 设计原理

```typescript
// ==================== 不可变更新 ====================

// ❌ 错误：直接修改状态
store.setState((prev) => {
  prev.messages.push(newMessage);  // 修改原对象
  return prev;
});

// ✅ 正确：创建新对象
store.setState((prev) => ({
  ...prev,
  messages: [...prev.messages, newMessage],  // 新数组
}));

// ==================== 浅比较优化 ====================

// 如果 updater 返回相同对象，不会触发更新
store.setState((prev) => prev);  // 无更新

// 如果对象内容相同但引用不同，会触发更新
store.setState((prev) => ({ ...prev }));  // 会触发更新

// ==================== 批量更新 ====================

// 多次 setState 会触发多次通知
store.setState((prev) => ({ ...prev, a: 1 }));
store.setState((prev) => ({ ...prev, b: 2 }));  // 两次通知

// 应该合并为一次更新
store.setState((prev) => ({ ...prev, a: 1, b: 2 }));  // 一次通知
```

### 2.3 React 集成

**文件**: `src/state/AppState.tsx`

```typescript
import React, { createContext, useContext, useCallback, useSyncExternalStore } from 'react';
import { createStore, Store } from './store.js';

// ==================== 应用状态类型 ====================

export type AppState = {
  // 会话信息
  session: Session;
  
  // 消息历史
  messages: Message[];
  
  // 工具权限
  permissions: PermissionState;
  
  // MCP 连接
  mcpConnections: MCPConnection[];
  
  // UI 状态
  ui: UIState;
  
  // 统计信息
  stats: Stats;
};

type Session = {
  id: string;
  startTime: number;
  workingDirectory: string;
  projectRoot: string | null;
};

type PermissionState = {
  mode: PermissionMode;
  decisions: Map<string, PermissionDecision>;  // toolName -> decision
};

type UIState = {
  isLoading: boolean;
  currentView: 'repl' | 'settings' | 'help';
  selectedMessageId: string | null;
};

type Stats = {
  totalInputTokens: number;
  totalOutputTokens: number;
  totalCost: number;
  toolCallCount: number;
};

// ==================== Context 定义 ====================

const AppStoreContext = createContext<Store<AppState> | null>(null);

export function AppStateProvider({
  children,
  initialState,
}: AppStateProviderProps): React.ReactElement {
  // 创建 store（只在首次渲染时创建）
  const [store] = React.useState(() =>
    createStore<AppState>(initialState ?? getDefaultState(), {
      onChange: ({ newState, oldState }) => {
        // 持久化到磁盘
        persistState(newState);
        
        // 记录状态变化日志
        logStateChange(oldState, newState);
      },
    })
  );
  
  return (
    <AppStoreContext.Provider value={store}>
      {children}
    </AppStoreContext.Provider>
  );
}

// ==================== Hooks ====================

/**
 * useAppStore
 * 
 * 获取 Store 实例（用于直接操作）
 */
export function useAppStore(): Store<AppState> {
  const store = useContext(AppStoreContext);
  if (!store) {
    throw new Error('useAppStore must be used within AppStateProvider');
  }
  return store;
}

/**
 * useAppState
 * 
 * 订阅状态片段（用于组件渲染）
 * 使用 useSyncExternalStore 实现与 React 的集成
 */
export function useAppState<R>(selector: (state: AppState) => R): R {
  const store = useAppStore();
  
  // 创建 getter 函数
  const get = useCallback(
    () => selector(store.getState()),
    [selector, store]
  );
  
  // 使用 React 18 的 useSyncExternalStore
  return useSyncExternalStore(
    store.subscribe,  // 订阅函数
    get,              // 客户端获取
    get               // 服务端获取（SSR）
  );
}

// 默认状态
function getDefaultState(): AppState {
  return {
    session: {
      id: generateSessionId(),
      startTime: Date.now(),
      workingDirectory: process.cwd(),
      projectRoot: null,
    },
    messages: [],
    permissions: {
      mode: 'auto_allow_read_only',
      decisions: new Map(),
    },
    mcpConnections: [],
    ui: {
      isLoading: false,
      currentView: 'repl',
      selectedMessageId: null,
    },
    stats: {
      totalInputTokens: 0,
      totalOutputTokens: 0,
      totalCost: 0,
      toolCallCount: 0,
    },
  };
}
```

### 2.4 状态操作 Hook

**文件**: `src/hooks/useAppState.ts`

```typescript
import { useCallback } from 'react';
import { useAppStore, useAppState } from '../state/AppState.js';

/**
 * useMessages
 * 
 * 消息历史操作
 */
export function useMessages() {
  const store = useAppStore();
  const messages = useAppState((state) => state.messages);
  
  const addMessage = useCallback((message: Message) => {
    store.setState((prev) => ({
      ...prev,
      messages: [...prev.messages, { ...message, id: generateId(), timestamp: Date.now() }],
    }));
  }, [store]);
  
  const updateLastMessage = useCallback((updater: (message: Message) => Message) => {
    store.setState((prev) => {
      if (prev.messages.length === 0) return prev;
      
      const lastIndex = prev.messages.length - 1;
      const updated = updater(prev.messages[lastIndex]);
      
      return {
        ...prev,
        messages: [...prev.messages.slice(0, lastIndex), updated],
      };
    });
  }, [store]);
  
  const appendToLastMessage = useCallback((content: string) => {
    store.setState((prev) => {
      if (prev.messages.length === 0) return prev;
      
      const lastIndex = prev.messages.length - 1;
      const lastMessage = prev.messages[lastIndex];
      
      if (lastMessage.role !== 'assistant') return prev;
      
      const updated: Message = {
        ...lastMessage,
        content: lastMessage.content + content,
      };
      
      return {
        ...prev,
        messages: [...prev.messages.slice(0, lastIndex), updated],
      };
    });
  }, [store]);
  
  const clearMessages = useCallback(() => {
    store.setState((prev) => ({
      ...prev,
      messages: [],
    }));
  }, [store]);
  
  const compactMessages = useCallback(async () => {
    const compacted = await autoCompact(messages);
    store.setState((prev) => ({
      ...prev,
      messages: compacted,
    }));
  }, [store, messages]);
  
  return {
    messages,
    addMessage,
    updateLastMessage,
    appendToLastMessage,
    clearMessages,
    compactMessages,
  };
}

/**
 * useStats
 * 
 * 统计信息操作
 */
export function useStats() {
  const store = useAppStore();
  const stats = useAppState((state) => state.stats);
  
  const addUsage = useCallback((usage: Usage) => {
    store.setState((prev) => ({
      ...prev,
      stats: {
        ...prev.stats,
        totalInputTokens: prev.stats.totalInputTokens + usage.input_tokens,
        totalOutputTokens: prev.stats.totalOutputTokens + usage.output_tokens,
        totalCost: prev.stats.totalCost + calculateCost(usage),
      },
    }));
  }, [store]);
  
  const incrementToolCallCount = useCallback(() => {
    store.setState((prev) => ({
      ...prev,
      stats: {
        ...prev.stats,
        toolCallCount: prev.stats.toolCallCount + 1,
      },
    }));
  }, [store]);
  
  return {
    stats,
    addUsage,
    incrementToolCallCount,
  };
}

/**
 * usePermissions
 * 
 * 权限管理
 */
export function usePermissions() {
  const store = useAppStore();
  const permissions = useAppState((state) => state.permissions);
  
  const recordPermissionDecision = useCallback((
    toolName: string,
    decision: PermissionDecision
  ) => {
    store.setState((prev) => {
      const newDecisions = new Map(prev.permissions.decisions);
      newDecisions.set(toolName, decision);
      
      return {
        ...prev,
        permissions: {
          ...prev.permissions,
          decisions: newDecisions,
        },
      };
    });
  }, [store]);
  
  const getPermissionForTool = useCallback((toolName: string) => {
    return permissions.decisions.get(toolName);
  }, [permissions.decisions]);
  
  const setPermissionMode = useCallback((mode: PermissionMode) => {
    store.setState((prev) => ({
      ...prev,
      permissions: {
        ...prev.permissions,
        mode,
      },
    }));
  }, [store]);
  
  return {
    permissions,
    recordPermissionDecision,
    getPermissionForTool,
    setPermissionMode,
  };
}

/**
 * useMCP
 * 
 * MCP 连接管理
 */
export function useMCP() {
  const store = useAppStore();
  const connections = useAppState((state) => state.mcpConnections);
  
  const addConnection = useCallback((connection: MCPConnection) => {
    store.setState((prev) => ({
      ...prev,
      mcpConnections: [...prev.mcpConnections, connection],
    }));
  }, [store]);
  
  const removeConnection = useCallback((serverName: string) => {
    store.setState((prev) => ({
      ...prev,
      mcpConnections: prev.mcpConnections.filter(
        (c) => c.serverName !== serverName
      ),
    }));
  }, [store]);
  
  return {
    connections,
    addConnection,
    removeConnection,
  };
}
```

## 3. 会话级全局状态

### 3.1 Bootstrap 状态

**文件**: `src/bootstrap/state.ts`

```typescript
/**
 * 会话级全局状态
 * 
 * 这些状态以模块级变量的形式存在，跨整个会话共享
 */

// ==================== 会话标识 ====================

let sessionId: string | null = null;
let sessionStartTime: number | null = null;

export function getSessionId(): string {
  if (!sessionId) {
    sessionId = generateSessionId();
    sessionStartTime = Date.now();
  }
  return sessionId;
}

export function getSessionStartTime(): number {
  if (!sessionStartTime) {
    sessionStartTime = Date.now();
  }
  return sessionStartTime;
}

// ==================== 路径状态 ====================

let currentWorkingDirectory: string = process.cwd();
let projectRoot: string | null = null;

export function getCwd(): string {
  return currentWorkingDirectory;
}

export function setCwd(cwd: string): void {
  currentWorkingDirectory = cwd;
  process.chdir(cwd);
}

export function getProjectRoot(): string | null {
  if (!projectRoot) {
    // 自动检测项目根目录
    projectRoot = detectProjectRoot(currentWorkingDirectory);
  }
  return projectRoot;
}

// ==================== Token 计数 ====================

let totalInputTokens = 0;
let totalOutputTokens = 0;

export function addTokenUsage(input: number, output: number): void {
  totalInputTokens += input;
  totalOutputTokens += output;
}

export function getTokenUsage(): { input: number; output: number } {
  return { input: totalInputTokens, output: totalOutputTokens };
}

// ==================== 全局配置 ====================

type GlobalConfig = {
  apiKey: string | null;
  model: string;
  maxTokens: number;
  temperature: number;
  theme: 'light' | 'dark';
  enableAnalytics: boolean;
};

let globalConfig: GlobalConfig = {
  apiKey: null,
  model: 'claude-3-sonnet-20240229',
  maxTokens: 4096,
  temperature: 0.7,
  theme: 'dark',
  enableAnalytics: true,
};

export function getGlobalConfig(): GlobalConfig {
  return { ...globalConfig };  // 返回副本防止外部修改
}

export function updateGlobalConfig(updates: Partial<GlobalConfig>): void {
  globalConfig = { ...globalConfig, ...updates };
  
  // 持久化到配置文件
  persistConfig(globalConfig);
}

// ==================== Prompt Caching 状态 ====================

let promptCache1hEligible: boolean | null = null;

export function getPromptCache1hEligible(): boolean | null {
  return promptCache1hEligible;
}

export function setPromptCache1hEligible(eligible: boolean): void {
  promptCache1hEligible = eligible;
}

// ==================== 初始化 ====================

export function initializeBootstrapState(): void {
  // 加载持久化配置
  const savedConfig = loadPersistedConfig();
  if (savedConfig) {
    globalConfig = { ...globalConfig, ...savedConfig };
  }
  
  // 初始化会话
  sessionId = generateSessionId();
  sessionStartTime = Date.now();
  
  // 检测项目根目录
  projectRoot = detectProjectRoot(process.cwd());
}
```

### 3.2 设计考量

```typescript
/**
 * 为什么使用模块级变量？
 * 
 * 1. 简单：不需要复杂的单例模式
 * 2. 性能：直接访问，无函数调用开销
 * 3. 跨层共享：任何模块都可以导入使用
 * 4. 会话级生命周期：随进程启动和结束
 * 
 * 注意事项：
 * - 这些状态不是响应式的，不会自动触发 UI 更新
 * - 需要 UI 响应的状态应该放在 AppState 中
 * - 多线程/多进程环境需要额外同步机制
 */

// 使用示例
import { getSessionId, getCwd, addTokenUsage } from '../bootstrap/state.js';

function logActivity() {
  console.log(`Session ${getSessionId()} in ${getCwd()}`);
}

function trackUsage(usage: Usage) {
  addTokenUsage(usage.input_tokens, usage.output_tokens);
}
```

## 4. 状态持久化

### 4.1 持久化策略

```typescript
/**
 * 状态持久化分层
 */

// ==================== 会话持久化 ====================
// 文件：~/.local/share/claude/sessions/<sessionId>.json

type PersistedSession = {
  sessionId: string;
  startTime: number;
  messages: Message[];
  workingDirectory: string;
  stats: Stats;
};

export async function persistSession(state: AppState): Promise<void> {
  const data: PersistedSession = {
    sessionId: state.session.id,
    startTime: state.session.startTime,
    messages: state.messages,
    workingDirectory: state.session.workingDirectory,
    stats: state.stats,
  };
  
  const filePath = getSessionFilePath(state.session.id);
  await fs.writeFile(filePath, JSON.stringify(data, null, 2));
}

export async function loadSession(sessionId: string): Promise<PersistedSession | null> {
  try {
    const filePath = getSessionFilePath(sessionId);
    const data = await fs.readFile(filePath, 'utf-8');
    return JSON.parse(data);
  } catch {
    return null;
  }
}

// ==================== 配置持久化 ====================
// 文件：~/.config/claude/settings.json

type PersistedConfig = {
  apiKey?: string;
  model?: string;
  theme?: 'light' | 'dark';
  permissionMode?: PermissionMode;
  enableAnalytics?: boolean;
  // ...
};

const CONFIG_PATH = path.join(os.homedir(), '.config/claude/settings.json');

export async function persistConfig(config: GlobalConfig): Promise<void> {
  const data: PersistedConfig = {
    // 不持久化敏感信息（如 API key 使用 keychain）
    model: config.model,
    theme: config.theme,
    enableAnalytics: config.enableAnalytics,
  };
  
  await fs.mkdir(path.dirname(CONFIG_PATH), { recursive: true });
  await fs.writeFile(CONFIG_PATH, JSON.stringify(data, null, 2));
}

export async function loadPersistedConfig(): Promise<PersistedConfig | null> {
  try {
    const data = await fs.readFile(CONFIG_PATH, 'utf-8');
    return JSON.parse(data);
  } catch {
    return null;
  }
}

// ==================== 用户偏好持久化 ====================
// 文件：~/.config/claude/memories.json

type PersistedMemory = {
  id: string;
  content: string;
  category: string;
  timestamp: number;
};

export async function saveMemory(memory: Memory): Promise<void> {
  const memories = await loadMemories();
  memories.push({
    id: memory.id,
    content: memory.content,
    category: memory.category,
    timestamp: Date.now(),
  });
  
  const filePath = path.join(os.homedir(), '.config/claude/memories.json');
  await fs.writeFile(filePath, JSON.stringify(memories, null, 2));
}

export async function loadMemories(): Promise<PersistedMemory[]> {
  try {
    const filePath = path.join(os.homedir(), '.config/claude/memories.json');
    const data = await fs.readFile(filePath, 'utf-8');
    return JSON.parse(data);
  } catch {
    return [];
  }
}
```

### 4.2 持久化触发时机

```typescript
/**
 * Store 的 onChange 回调中触发持久化
 */

const store = createStore<AppState>(initialState, {
  onChange: ({ newState, oldState }) => {
    // 防抖：避免频繁写入
    debouncedPersist(newState);
    
    // 立即持久化关键变化
    if (hasCriticalChange(oldState, newState)) {
      persistSession(newState);
    }
  },
});

// 防抖持久化
const debouncedPersist = debounce(
  (state: AppState) => persistSession(state),
  5000  // 5 秒防抖
);

// 判断是否有关键变化
function hasCriticalChange(oldState: AppState, newState: AppState): boolean {
  // 消息数量变化
  if (oldState.messages.length !== newState.messages.length) {
    return true;
  }
  
  // 最后一条消息变化（流式更新完成）
  const oldLast = oldState.messages[oldState.messages.length - 1];
  const newLast = newState.messages[newState.messages.length - 1];
  if (oldLast?.id !== newLast?.id) {
    return true;
  }
  
  return false;
}
```

## 5. 状态选择器优化

### 5.1 选择器缓存

```typescript
import { useMemo } from 'react';

/**
 * useMemoSelector
 * 
 * 缓存选择器结果，避免不必要的重渲染
 */
export function useMemoSelector<R>(
  selector: (state: AppState) => R,
  deps: React.DependencyList = []
): R {
  const state = useAppState((s) => s);  // 订阅整个状态
  
  return useMemo(() => selector(state), [state, ...deps]);
}

// 使用示例
function MessageList() {
  // ❌ 每次状态变化都会重新计算
  const messages = useAppState((state) =>
    state.messages.filter((m) => m.role === 'assistant')
  );
  
  // ✅ 只在 messages 变化时重新计算
  const assistantMessages = useMemoSelector(
    (state) => state.messages.filter((m) => m.role === 'assistant'),
    []
  );
  
  return <Messages messages={assistantMessages} />;
}
```

### 5.2 细粒度订阅

```typescript
/**
 * 细粒度状态订阅
 * 
 * 只订阅需要的字段，避免不必要的重渲染
 */

// ❌ 订阅整个消息数组
function BadComponent() {
  const messages = useAppState((state) => state.messages);
  const lastMessage = messages[messages.length - 1];
  return <Text>{lastMessage?.content}</Text>;
}

// ✅ 只订阅最后一条消息
function GoodComponent() {
  const lastMessage = useAppState((state) =>
    state.messages[state.messages.length - 1]
  );
  return <Text>{lastMessage?.content}</Text>;
}

// ✅ 使用自定义 hook 封装
function useLastMessage() {
  return useAppState((state) =>
    state.messages[state.messages.length - 1]
  );
}
```

## 6. 状态调试

### 6.1 状态日志

```typescript
/**
 * 状态变化日志
 */

const STORE_LOG_ENABLED = process.env.DEBUG_STORE === 'true';

export function logStateChange<T>(oldState: T, newState: T): void {
  if (!STORE_LOG_ENABLED) return;
  
  const changes = findChanges(oldState, newState);
  
  console.log('[Store Change]', {
    timestamp: new Date().toISOString(),
    changes: changes.map((c) => ({
      path: c.path,
      oldValue: truncate(c.oldValue),
      newValue: truncate(c.newValue),
    })),
  });
}

function findChanges(oldObj: any, newObj: any, path = ''): Change[] {
  const changes: Change[] = [];
  
  const keys = new Set([...Object.keys(oldObj), ...Object.keys(newObj)]);
  
  for (const key of keys) {
    const newPath = path ? `${path}.${key}` : key;
    const oldVal = oldObj[key];
    const newVal = newObj[key];
    
    if (typeof oldVal === 'object' && typeof newVal === 'object') {
      changes.push(...findChanges(oldVal, newVal, newPath));
    } else if (oldVal !== newVal) {
      changes.push({
        path: newPath,
        oldValue: oldVal,
        newValue: newVal,
      });
    }
  }
  
  return changes;
}

function truncate(value: any, maxLength = 100): any {
  if (typeof value === 'string' && value.length > maxLength) {
    return value.slice(0, maxLength) + '...';
  }
  return value;
}
```

### 6.2 Redux DevTools 集成

```typescript
/**
 * Redux DevTools 集成（开发环境）
 */

export function connectDevTools<T>(store: Store<T>): void {
  if (typeof window === 'undefined') return;
  
  const devTools = (window as any).__REDUX_DEVTOOLS_EXTENSION__;
  if (!devTools) return;
  
  const connection = devTools.connect({
    name: 'Claude Code',
    features: {
      pause: true,
      lock: true,
      persist: true,
      export: true,
      import: true,
    },
  });
  
  // 发送初始状态
  connection.init(store.getState());
  
  // 监听状态变化
  store.subscribe(() => {
    connection.send('STATE_UPDATE', store.getState());
  });
  
  // 监听 DevTools 消息
  connection.subscribe((message: any) => {
    if (message.type === 'DISPATCH') {
      switch (message.payload.type) {
        case 'RESET':
          // 重置状态
          store.setState(() => getDefaultState() as T);
          break;
        case 'COMMIT':
          // 提交状态
          connection.init(store.getState());
          break;
        case 'ROLLBACK':
          // 回滚状态
          const state = JSON.parse(message.state);
          store.setState(() => state);
          break;
      }
    }
  });
}
```

## 7. 总结

状态管理系统的设计体现了以下架构原则：

1. **分层管理**: React 层、App State、Store、Bootstrap 四层各司其职
2. **不可变更新**: 使用展开操作符保证状态不可变性
3. **细粒度订阅**: 只订阅需要的字段，避免不必要的重渲染
4. **浅比较优化**: Object.is 比较避免无意义更新
5. **响应式集成**: useSyncExternalStore 与 React 完美集成
6. **持久化支持**: 自动持久化到磁盘，支持会话恢复
7. **调试友好**: 状态变化日志、Redux DevTools 集成

这些设计使得 Claude Code 能够在复杂的交互场景中保持良好的性能和可维护性。
