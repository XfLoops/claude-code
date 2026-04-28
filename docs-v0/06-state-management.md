# 状态管理系统详解

## 状态管理架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    状态管理架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  分层状态架构                            │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │                                                         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│  │  │ AppState    │  │ Bootstrap   │  │  ToolUse    │     │   │
│  │  │ (React)     │  │  State      │  │  Context    │     │   │
│  │  │             │  │ (单例)       │  │             │     │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│  │         │                │                │             │   │
│  │         └────────────────┼────────────────┘             │   │
│  │                          ▼                              │   │
│  │              ┌─────────────────────┐                    │   │
│  │              │    FileStateCache   │                    │   │
│  │              │    (LRU 缓存)        │                    │   │
│  │              └─────────────────────┘                    │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## AppState 详解

### AppState 类型定义

```typescript
// src/state/AppStateStore.ts

export type AppState = {
  // ==================== 核心状态 ====================
  
  // 消息历史
  messages: Message[]
  
  // 处理状态
  isProcessing: boolean
  isWaitingForPermission: boolean
  abortController: AbortController
  
  // ==================== 权限状态 ====================
  
  toolPermissionContext: ToolPermissionContext
  pendingPermission: PendingPermission | null
  
  // ==================== 输入状态 ====================
  
  inputValue: string
  inputMode: PromptInputMode
  vimMode: VimMode
  
  // ==================== 显示状态 ====================
  
  verbose: boolean
  showCost: boolean
  showFPS: boolean
  theme: ThemeName
  
  // ==================== 会话状态 ====================
  
  sessionId: string
  conversationId: UUID | null
  
  // ==================== 工具相关 ====================
  
  inProgressToolUseIDs: Set<string>
  hasInterruptibleToolInProgress: boolean
  toolJSX: {
    jsx: React.ReactNode | null
    shouldHidePromptInput: boolean
    showSpinner: boolean
  }
  
  // ==================== MCP 相关 ====================
  
  mcpClients: MCPServerConnection[]
  mcpServerStates: Map<string, McpServerState>
  
  // ==================== Agent 相关 ====================
  
  activeAgents: Map<string, AgentState>
  teammateMode: TeammateModeState | null
  
  // ==================== 任务相关 ====================
  
  tasks: Map<string, Task>
  taskLists: Map<string, TaskList>
  
  // ==================== 历史记录 ====================
  
  fileHistoryState: FileHistoryState
  attributionState: AttributionState
  
  // ==================== 通知 ====================
  
  notifications: Notification[]
  
  // ==================== 统计信息 ====================
  
  usage: UsageState
  cost: CostState
  
  // ==================== 推测状态 ====================
  
  speculation: SpeculationState
}
```

### AppStateStore 创建

```typescript
// src/state/AppStateStore.ts

import { createStore, type Store } from './store.js'

export function getDefaultAppState(): AppState {
  return {
    messages: [],
    isProcessing: false,
    isWaitingForPermission: false,
    abortController: new AbortController(),
    
    toolPermissionContext: getEmptyToolPermissionContext(),
    pendingPermission: null,
    
    inputValue: '',
    inputMode: 'normal',
    vimMode: 'normal',
    
    verbose: false,
    showCost: true,
    showFPS: false,
    theme: 'dark',
    
    sessionId: getSessionId(),
    conversationId: null,
    
    inProgressToolUseIDs: new Set(),
    hasInterruptibleToolInProgress: false,
    toolJSX: {
      jsx: null,
      shouldHidePromptInput: false,
      showSpinner: false,
    },
    
    mcpClients: [],
    mcpServerStates: new Map(),
    
    activeAgents: new Map(),
    teammateMode: null,
    
    tasks: new Map(),
    taskLists: new Map(),
    
    fileHistoryState: createEmptyFileHistoryState(),
    attributionState: createEmptyAttributionState(),
    
    notifications: [],
    
    usage: createEmptyUsageState(),
    cost: createEmptyCostState(),
    
    speculation: IDLE_SPECULATION_STATE,
  }
}
```

## Store 实现

### Zustand-style Store

```typescript
// src/state/store.ts

type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()
  
  return {
    getState: () => state,
    
    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      
      // 使用 Object.is 进行浅比较
      if (Object.is(next, prev)) return
      
      state = next
      
      // 调用 onChange 回调
      onChange?.({ newState: next, oldState: prev })
      
      // 通知所有订阅者
      for (const listener of listeners) {
        listener()
      }
    },
    
    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

## React 集成

### AppStateProvider

```typescript
// src/state/AppState.tsx

import React, { useContext, useState, useSyncExternalStore } from 'react'
import { createStore, type Store } from './store.js'
import { type AppState, getDefaultAppState } from './AppStateStore.js'

export const AppStoreContext = React.createContext<Store<AppState> | null>(null)

export interface AppStateProviderProps {
  children: React.ReactNode
  initialState?: AppState
  onChangeAppState?: (args: { newState: AppState; oldState: AppState }) => void
}

export function AppStateProvider({
  children,
  initialState,
  onChangeAppState,
}: AppStateProviderProps) {
  const [store] = useState(() =>
    createStore(initialState ?? getDefaultAppState(), onChangeAppState)
  )
  
  return (
    <AppStoreContext.Provider value={store}>
      <MailboxProvider>
        <VoiceProvider>
          {children}
        </VoiceProvider>
      </MailboxProvider>
    </AppStoreContext.Provider>
  )
}
```

### useAppState Hook

```typescript
// src/state/AppState.tsx

/**
 * 订阅 AppState 的某个切片
 * 只在选择器返回值变化时重新渲染
 */
export function useAppState<R>(selector: (state: AppState) => R): R {
  const store = useContext(AppStoreContext)
  if (!store) {
    throw new ReferenceError('useAppState must be used within AppStateProvider')
  }
  
  const getSnapshot = () => {
    const state = store.getState()
    return selector(state)
  }
  
  return useSyncExternalStore(store.subscribe, getSnapshot, getSnapshot)
}

/**
 * 获取 setState 而不订阅任何状态
 */
export function useSetAppState(): Store<AppState>['setState'] {
  const store = useContext(AppStoreContext)
  if (!store) {
    throw new ReferenceError('useSetAppState must be used within AppStateProvider')
  }
  return store.setState
}

/**
 * 获取完整 store（用于非 React 代码）
 */
export function useAppStateStore(): Store<AppState> {
  const store = useContext(AppStoreContext)
  if (!store) {
    throw new ReferenceError('useAppStateStore must be used within AppStateProvider')
  }
  return store
}

/**
 * 安全版本，返回 undefined 如果不在 Provider 内
 */
export function useAppStateMaybeOutsideOfProvider<R>(
  selector: (state: AppState) => R
): R | undefined {
  const store = useContext(AppStoreContext)
  if (!store) return undefined
  
  const getSnapshot = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, getSnapshot, getSnapshot)
}
```

## Bootstrap State

### 模块级单例

```typescript
// src/bootstrap/state.ts

import { randomUUID, type UUID } from 'crypto'

// ==================== 会话状态 ====================

let sessionId: string = randomUUID()
let originalCwd: string = process.cwd()
let projectRoot: string | null = null
let isNonInteractiveSession: boolean = false

// ==================== 模型状态 ====================

let initialMainLoopModel: string | undefined
let mainLoopModelOverride: string | undefined
let sdkBetas: string[] = []

// ==================== Token 跟踪 ====================

let currentTurnTokenBudget: number | undefined
let turnOutputTokens: number = 0
let budgetContinuationCount: number = 0
let totalInputTokens: number = 0

// ==================== 统计计数器 ====================

interface AttributedCounter {
  add(value: number, additionalAttributes?: Attributes): void
}

let meter: Meter | undefined
let sessionCounter: AttributedCounter | undefined

// ==================== Getter/Setter ====================

export function getSessionId(): string {
  return sessionId
}

export function switchSession(newSessionId?: string): string {
  sessionId = newSessionId ?? randomUUID()
  return sessionId
}

export function getOriginalCwd(): string {
  return originalCwd
}

export function setOriginalCwd(cwd: string): void {
  originalCwd = cwd
}

export function getProjectRoot(): string | null {
  return projectRoot
}

export function setProjectRoot(root: string | null): void {
  projectRoot = root
}

export function getIsNonInteractiveSession(): boolean {
  return isNonInteractiveSession
}

export function setIsNonInteractiveSession(value: boolean): void {
  isNonInteractiveSession = value
}

// ==================== Token 预算 ====================

export function getCurrentTurnTokenBudget(): number | undefined {
  return currentTurnTokenBudget
}

export function setCurrentTurnTokenBudget(budget: number): void {
  currentTurnTokenBudget = budget
}

export function getTurnOutputTokens(): number {
  return turnOutputTokens
}

export function setTurnOutputTokens(tokens: number): void {
  turnOutputTokens = tokens
}

export function incrementBudgetContinuationCount(): number {
  return ++budgetContinuationCount
}

export function getTotalInputTokens(): number {
  return totalInputTokens
}

export function addToTotalInputTokens(tokens: number): void {
  totalInputTokens += tokens
}

// ==================== 遥测 ====================

export function setMeter(
  newMeter: Meter,
  createCounter: (name: string, options: MetricOptions) => AttributedCounter
): void {
  meter = newMeter
  sessionCounter = createCounter('tengu_sessions', {
    description: 'Number of sessions started',
  })
}

export function getSessionCounter(): AttributedCounter | undefined {
  return sessionCounter
}
```

## ToolUseContext 状态

### 上下文构建

```typescript
// src/Tool.ts

export function buildToolUseContext(
  params: BuildToolUseContextParams
): ToolUseContext {
  const {
    options,
    abortController,
    readFileState,
    getAppState,
    setAppState,
    setAppStateForTasks,
    handleElicitation,
    setToolJSX,
    addNotification,
    appendSystemMessage,
    sendOSNotification,
    requestPrompt,
    setSDKStatus,
    openMessageSelector,
    messages,
    agentId,
    agentType,
    requireCanUseTool,
    localDenialTracking,
    contentReplacementState,
    renderedSystemPrompt,
  } = params
  
  return {
    options,
    abortController,
    readFileState,
    getAppState,
    setAppState,
    setAppStateForTasks,
    handleElicitation,
    setToolJSX,
    addNotification,
    appendSystemMessage,
    sendOSNotification,
    nestedMemoryAttachmentTriggers: new Set(),
    loadedNestedMemoryPaths: new Set(),
    dynamicSkillDirTriggers: new Set(),
    discoveredSkillNames: new Set(),
    requestPrompt,
    messages,
    setInProgressToolUseIDs: (f) => {
      setAppState(prev => ({
        ...prev,
        inProgressToolUseIDs: f(prev.inProgressToolUseIDs),
      }))
    },
    setHasInterruptibleToolInProgress: setAppState
      ? (v) => setAppState(prev => ({ ...prev, hasInterruptibleToolInProgress: v }))
      : undefined,
    setResponseLength: (f) => {
      setAppState(prev => ({
        ...prev,
        responseLength: f(prev.responseLength ?? 0),
      }))
    },
    pushApiMetricsEntry: options.isNonInteractiveSession
      ? (ttftMs) => {
          setAppState(prev => ({
            ...prev,
            apiMetrics: [...(prev.apiMetrics ?? []), { ttftMs }],
          }))
        }
      : undefined,
    setStreamMode: setAppState
      ? (mode) => setAppState(prev => ({ ...prev, spinnerMode: mode }))
      : undefined,
    setSDKStatus,
    openMessageSelector,
    updateFileHistoryState: (updater) => {
      setAppState(prev => ({
        ...prev,
        fileHistoryState: updater(prev.fileHistoryState),
      }))
    },
    updateAttributionState: (updater) => {
      setAppState(prev => ({
        ...prev,
        attributionState: updater(prev.attributionState),
      }))
    },
    setConversationId: setAppState
      ? (id) => setAppState(prev => ({ ...prev, conversationId: id }))
      : undefined,
    agentId,
    agentType,
    requireCanUseTool,
    localDenialTracking,
    contentReplacementState,
    renderedSystemPrompt,
  }
}
```

## FileStateCache

### LRU 缓存实现

```typescript
// src/utils/fileStateCache.ts

import { LRUCache } from 'lru-cache'

export type FileState = {
  content: string
  mtimeMs: number
  size: number
  lineEnding: LineEndingType
  hash: string
}

export type FileStateCache = LRUCache<string, FileState>

export const READ_FILE_STATE_CACHE_SIZE = 1000

export function createFileStateCache(): FileStateCache {
  return new LRUCache({
    max: READ_FILE_STATE_CACHE_SIZE,
    // 自定义大小计算
    sizeCalculation: (value: FileState) => value.content.length,
    maxSize: 100 * 1024 * 1024, // 100MB
    updateAgeOnGet: true,
    updateAgeOnHas: true,
  })
}

export function createFileStateCacheWithSizeLimit(
  maxSize: number
): FileStateCache {
  return new LRUCache({
    max: READ_FILE_STATE_CACHE_SIZE,
    sizeCalculation: (value: FileState) => value.content.length,
    maxSize,
    updateAgeOnGet: true,
    updateAgeOnHas: true,
  })
}

export function cloneFileStateCache(cache: FileStateCache): FileStateCache {
  const newCache = createFileStateCache()
  
  for (const [key, value] of cache.entries()) {
    newCache.set(key, { ...value })
  }
  
  return newCache
}

export function mergeFileStateCaches(
  base: FileStateCache,
  overlay: FileStateCache
): FileStateCache {
  const merged = cloneFileStateCache(base)
  
  for (const [key, value] of overlay.entries()) {
    // 只合并更新的文件
    const existing = merged.get(key)
    if (!existing || existing.mtimeMs < value.mtimeMs) {
      merged.set(key, value)
    }
  }
  
  return merged
}

// 使用
export async function readFileWithCache(
  filePath: string,
  cache: FileStateCache
): Promise<FileState> {
  // 检查缓存
  const cached = cache.get(filePath)
  if (cached) {
    // 验证文件是否修改
    const stats = await stat(filePath)
    if (stats.mtimeMs === cached.mtimeMs) {
      return cached
    }
  }
  
  // 读取文件
  const content = await readFile(filePath, 'utf-8')
  const stats = await stat(filePath)
  const lineEnding = detectLineEnding(content)
  const hash = computeHash(content)
  
  const fileState: FileState = {
    content,
    mtimeMs: stats.mtimeMs,
    size: stats.size,
    lineEnding,
    hash,
  }
  
  // 更新缓存
  cache.set(filePath, fileState)
  
  return fileState
}
```

## 状态持久化

### Session Storage

```typescript
// src/utils/sessionStorage.ts

import { writeFile, readFile, mkdir } from 'fs/promises'
import { join } from 'path'
import { getSessionId } from '../bootstrap/state.js'

const SESSIONS_DIR = join(os.homedir(), '.claude', 'sessions')

export interface SessionRecord {
  sessionId: string
  title: string
  messages: Message[]
  fileHistoryState: FileHistoryState
  attributionState: AttributionState
  timestamp: number
  metadata: SessionMetadata
}

export async function saveSession(state: AppState): Promise<void> {
  const sessionId = getSessionId()
  const record: SessionRecord = {
    sessionId,
    title: generateSessionTitle(state.messages),
    messages: state.messages,
    fileHistoryState: state.fileHistoryState,
    attributionState: state.attributionState,
    timestamp: Date.now(),
    metadata: {
      cwd: process.cwd(),
      model: state.mainLoopModel,
      cost: state.cost,
    },
  }
  
  const filePath = join(SESSIONS_DIR, `${sessionId}.json`)
  await mkdir(SESSIONS_DIR, { recursive: true })
  await writeFile(filePath, JSON.stringify(record, null, 2))
}

export async function loadSession(sessionId: string): Promise<SessionRecord | null> {
  try {
    const filePath = join(SESSIONS_DIR, `${sessionId}.json`)
    const data = await readFile(filePath, 'utf-8')
    return JSON.parse(data) as SessionRecord
  } catch {
    return null
  }
}

export async function recordTranscript(
  messages: Message[],
  metadata: TranscriptMetadata
): Promise<void> {
  const sessionId = getSessionId()
  const logPath = join(SESSIONS_DIR, `${sessionId}.log`)
  
  const entry = {
    timestamp: new Date().toISOString(),
    messages,
    metadata,
  }
  
  await appendFile(logPath, JSON.stringify(entry) + '\n')
}
```

### 状态恢复

```typescript
// src/utils/sessionRestore.ts

export async function restoreSessionStateFromLog(
  sessionId: string
): Promise<Partial<AppState> | null> {
  const logPath = join(SESSIONS_DIR, `${sessionId}.log`)
  
  try {
    const content = await readFile(logPath, 'utf-8')
    const lines = content.trim().split('\n')
    
    // 找到最后一个完整的状态
    for (let i = lines.length - 1; i >= 0; i--) {
      try {
        const entry = JSON.parse(lines[i])
        if (entry.messages && entry.metadata) {
          return {
            messages: deserializeMessages(entry.messages),
            fileHistoryState: entry.metadata.fileHistoryState,
            attributionState: entry.metadata.attributionState,
          }
        }
      } catch {
        continue
      }
    }
    
    return null
  } catch {
    return null
  }
}

export function computeStandaloneAgentContext(
  parentContext: ToolUseContext,
  agentParams: AgentParams
): ToolUseContext {
  // 克隆文件状态缓存
  const readFileState = cloneFileStateCache(parentContext.readFileState)
  
  // 创建新的 AppState 存根
  const agentState = createAgentStateStub(parentContext.getAppState(), agentParams)
  
  return {
    ...parentContext,
    agentId: agentParams.agentId,
    agentType: agentParams.agentType,
    readFileState,
    getAppState: () => agentState,
    setAppState: createAgentSetState(agentParams.agentId),
    messages: [], // Agent 有自己的消息历史
  }
}
```

## 状态变更监听

### onChangeAppState

```typescript
// src/state/onChangeAppState.ts

import { logEvent } from '../services/analytics/index.js'
import { saveSession } from '../utils/sessionStorage.js'

export function onChangeAppState({
  newState,
  oldState,
}: {
  newState: AppState
  oldState: AppState
}): void {
  // 1. 分析状态变化
  const changes = analyzeChanges(oldState, newState)
  
  // 2. 记录分析事件
  if (changes.hasNewMessages) {
    logEvent('tengu_messages_changed', {
      count: newState.messages.length,
    })
  }
  
  if (changes.permissionModeChanged) {
    logEvent('tengu_permission_mode_changed', {
      from: oldState.toolPermissionContext.mode,
      to: newState.toolPermissionContext.mode,
    })
  }
  
  // 3. 自动保存会话
  if (changes.shouldAutoSave) {
    void saveSession(newState)
  }
  
  // 4. 处理推测状态
  if (changes.speculationChanged) {
    handleSpeculationChange(newState.speculation)
  }
}

function analyzeChanges(
  oldState: AppState,
  newState: AppState
): StateChanges {
  return {
    hasNewMessages: oldState.messages.length !== newState.messages.length,
    permissionModeChanged:
      oldState.toolPermissionContext.mode !== newState.toolPermissionContext.mode,
    shouldAutoSave:
      newState.messages.length > 0 &&
      Date.now() - (newState.lastSavedAt ?? 0) > AUTO_SAVE_INTERVAL,
    speculationChanged: oldState.speculation !== newState.speculation,
  }
}
```

## 性能优化

### 选择器优化

```typescript
// 好的实践：选择原始值
const count = useAppState(s => s.messages.length)
const firstMessage = useAppState(s => s.messages[0])

// 避免：创建新对象
// ❌ 每次都创建新对象，导致无限重渲染
const messageInfo = useAppState(s => ({
  count: s.messages.length,
  first: s.messages[0],
}))

// 使用 memoized 选择器
const selectMessageStats = useMemo(
  () => createSelector(
    (state: AppState) => state.messages,
    (messages) => ({
      count: messages.length,
      userCount: messages.filter(m => m.type === 'user').length,
    })
  ),
  []
)

const stats = useAppState(selectMessageStats)
```

### 批量更新

```typescript
// 批量状态更新
function batchStateUpdates(
  setAppState: SetAppState,
  updates: Array<(prev: AppState) => Partial<AppState>>
): void {
  setAppState(prev => {
    let next = prev
    for (const update of updates) {
      const partial = update(next)
      next = { ...next, ...partial }
    }
    return next
  })
}

// 使用
batchStateUpdates(setAppState, [
  prev => ({ isProcessing: true }),
  prev => ({ messages: [...prev.messages, newMessage] }),
  prev => ({ inputValue: '' }),
])
```
