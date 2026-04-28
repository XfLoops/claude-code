# 权限系统详解

## 权限系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    权限系统架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 types/permissions.ts                     │   │
│  │              (权限类型定义)                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Tool.ts (ToolUseContext)                    │   │
│  │              (权限上下文)                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              utils/permissions/                          │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│  │  │ Permission  │  │ Permission  │  │ autoMode    │     │   │
│  │  │   Mode      │  │   Rules     │  │ State.ts    │     │   │
│  │  │ (权限模式)  │  │ (规则匹配)   │  │ (自动模式)   │     │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│  │  ┌─────────────┐  ┌─────────────┐                      │   │
│  │  │ Filesystem  │  │  denial     │                      │   │
│  │  │ Permissions │  │ Tracking.ts │                      │   │
│  │  │ (文件权限)   │  │ (拒绝跟踪)   │                      │   │
│  │  └─────────────┘  └─────────────┘                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              components/permissions/                     │   │
│  │         PermissionRequest.tsx (权限请求UI)               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 权限模式

### 权限模式类型

```typescript
// src/types/permissions.ts

/**
 * 外部权限模式（用户可设置）
 */
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',      // 接受所有编辑（只读工具自动通过）
  'bypassPermissions', // 绕过所有权限检查（危险）
  'default',          // 默认：询问用户
  'dontAsk',          // 静默模式：只读通过，其他拒绝
  'plan',             // 计划模式：进入计划流程
] as const

export type ExternalPermissionMode = (typeof EXTERNAL_PERMISSION_MODES)[number]

/**
 * 内部权限模式（包含系统模式）
 */
export type InternalPermissionMode = 
  | ExternalPermissionMode 
  | 'auto'     // 自动分类器决策
  | 'bubble'   // 冒泡模式（子 agent）

export type PermissionMode = InternalPermissionMode

/**
 * 运行时验证集合
 */
export const INTERNAL_PERMISSION_MODES = [
  ...EXTERNAL_PERMISSION_MODES,
  ...(feature('TRANSCRIPT_CLASSIFIER') ? ['auto'] as const : []),
] as const satisfies readonly PermissionMode[]

export const PERMISSION_MODES = INTERNAL_PERMISSION_MODES
```

### 权限行为

```typescript
// src/types/permissions.ts

export type PermissionBehavior = 'allow' | 'deny' | 'ask'

/**
 * 规则来源
 */
export type PermissionRuleSource =
  | 'userSettings'    // 用户设置
  | 'projectSettings' // 项目设置
  | 'localSettings'   // 本地设置
  | 'flagSettings'    // Flag 设置
  | 'policySettings'  // 策略设置
  | 'cliArg'         // CLI 参数
  | 'command'        // 斜杠命令
  | 'session'        // 会话临时

/**
 * 权限规则值
 */
export type PermissionRuleValue = {
  toolName: string      // 工具名称
  ruleContent?: string  // 可选内容匹配模式
}

/**
 * 权限规则
 */
export type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: PermissionBehavior
  ruleValue: PermissionRuleValue
}
```

## 权限上下文

### ToolPermissionContext

```typescript
// src/Tool.ts

export type ToolPermissionContext = DeepImmutable<{
  // 当前模式
  mode: PermissionMode
  
  // 额外工作目录
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  
  // 规则集合
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  
  // 功能可用性
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  
  // 自动模式相关
  strippedDangerousRules?: ToolPermissionRulesBySource
  
  // 子 agent 相关
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>

export type ToolPermissionRulesBySource = {
  [K in PermissionRuleSource]?: PermissionRuleValue[]
}

/**
 * 创建空权限上下文
 */
export const getEmptyToolPermissionContext: () => ToolPermissionContext = () => ({
  mode: 'default',
  additionalWorkingDirectories: new Map(),
  alwaysAllowRules: {},
  alwaysDenyRules: {},
  alwaysAskRules: {},
  isBypassPermissionsModeAvailable: false,
})
```

### 权限决策结果

```typescript
// src/types/permissions.ts

export type PermissionDecision =
  | { type: 'allowed' }
  | { type: 'denied'; reason: string }
  | { type: 'needs_permission'; message: string }

export type PermissionResult = {
  allowed: boolean
  reason?: string
  mode?: PermissionMode
  behavior?: PermissionBehavior
}
```

## 权限检查流程

### 主要检查函数

```typescript
// src/utils/permissions/permissionSetup.ts

export async function checkToolPermission(
  tool: Tool,
  input: unknown,
  context: ToolUseContext
): Promise<PermissionDecision> {
  const appState = context.getAppState()
  const permissionContext = appState.toolPermissionContext
  
  // 1. 检查 bypass 模式
  if (permissionContext.mode === 'bypassPermissions') {
    return { type: 'allowed' }
  }
  
  // 2. 检查工具特定权限
  if (tool.checkPermissions) {
    const result = await tool.checkPermissions(input, context)
    if (result.type !== 'allowed') {
      return result
    }
  }
  
  // 3. 检查是否为只读工具（acceptEdits 模式）
  if (permissionContext.mode === 'acceptEdits' && tool.isReadOnly) {
    return { type: 'allowed' }
  }
  
  // 4. 检查 dontAsk 模式
  if (permissionContext.mode === 'dontAsk') {
    return tool.isReadOnly
      ? { type: 'allowed' }
      : { type: 'denied', reason: 'dontAsk mode' }
  }
  
  // 5. 检查路径匹配规则
  const path = tool.getPath?.(input)
  if (path) {
    const ruleDecision = checkPathRules(tool, path, permissionContext)
    if (ruleDecision) return ruleDecision
  }
  
  // 6. 检查自动模式
  if (permissionContext.mode === 'auto' && permissionContext.isAutoModeAvailable) {
    return await checkAutoPermission(tool, input, context)
  }
  
  // 7. 默认行为
  return { type: 'needs_permission', message: 'Confirmation required' }
}
```

### 路径规则检查

```typescript
// src/utils/permissions/permissionSetup.ts

function checkPathRules(
  tool: Tool,
  path: string | string[],
  context: ToolPermissionContext
): PermissionDecision | null {
  const paths = Array.isArray(path) ? path : [path]
  
  for (const p of paths) {
    // 检查 alwaysDeny 规则
    const denyRule = findMatchingRule(
      context.alwaysDenyRules,
      tool.name,
      p
    )
    if (denyRule) {
      return { 
        type: 'denied', 
        reason: `Denied by rule from ${denyRule.source}` 
      }
    }
    
    // 检查 alwaysAllow 规则
    const allowRule = findMatchingRule(
      context.alwaysAllowRules,
      tool.name,
      p
    )
    if (allowRule) {
      return { type: 'allowed' }
    }
    
    // 检查 alwaysAsk 规则
    const askRule = findMatchingRule(
      context.alwaysAskRules,
      tool.name,
      p
    )
    if (askRule) {
      return { 
        type: 'needs_permission', 
        message: `Rule from ${askRule.source} requires confirmation` 
      }
    }
  }
  
  return null
}

function findMatchingRule(
  rulesBySource: ToolPermissionRulesBySource,
  toolName: string,
  path: string
): { source: PermissionRuleSource; rule: PermissionRuleValue } | null {
  for (const [source, rules] of Object.entries(rulesBySource)) {
    for (const rule of rules ?? []) {
      if (rule.toolName === toolName || rule.toolName === '*') {
        if (!rule.ruleContent || matchWildcardPattern(rule.ruleContent, path)) {
          return { source: source as PermissionRuleSource, rule }
        }
      }
    }
  }
  return null
}
```

## 自动分类器模式

### 自动权限决策

```typescript
// src/utils/permissions/autoModeState.ts

export interface AutoModeState {
  enabled: boolean
  classifier: AutoClassifier
  confidenceThreshold: number
  denyThreshold: number
}

export async function checkAutoPermission(
  tool: Tool,
  input: unknown,
  context: ToolUseContext
): Promise<PermissionDecision> {
  const state = getAutoModeState()
  if (!state.enabled) {
    return { type: 'needs_permission', message: 'Auto mode not enabled' }
  }
  
  // 构建分类器输入
  const classifierInput = buildClassifierInput(tool, input, context)
  
  // 调用分类器
  const result = await state.classifier.classify(classifierInput)
  
  // 根据置信度决策
  if (result.confidence >= state.confidenceThreshold) {
    switch (result.decision) {
      case 'allow':
        return { type: 'allowed' }
      case 'deny':
        return { 
          type: 'denied', 
          reason: `Auto-classifier denied (confidence: ${result.confidence})` 
        }
      case 'ask':
        return { 
          type: 'needs_permission', 
          message: 'Auto-classifier requests confirmation' 
        }
    }
  }
  
  // 低置信度，请求用户确认
  return { 
    type: 'needs_permission', 
    message: `Uncertain classification (confidence: ${result.confidence})` 
  }
}

function buildClassifierInput(
  tool: Tool,
  input: unknown,
  context: ToolUseContext
): ClassifierInput {
  const inputText = tool.toAutoClassifierInput?.(input) ?? JSON.stringify(input)
  
  return {
    toolName: tool.name,
    toolDescription: typeof tool.description === 'string' 
      ? tool.description 
      : '',
    input: inputText,
    userContext: context.messages.slice(-3).map(m => 
      getMessageText(m)
    ).join('\n'),
  }
}
```

## 文件系统权限

### 文件权限检查

```typescript
// src/utils/permissions/filesystem.ts

export interface FilesystemPermissionResult {
  allowed: boolean
  reason?: string
  matchingRule?: PermissionRule
}

export async function checkReadPermissionForTool(
  tool: Tool,
  input: { file_path: string },
  context: ToolPermissionContext
): Promise<PermissionDecision> {
  const filePath = expandPath(input.file_path)
  
  // 1. 检查是否在允许的工作目录内
  if (!isWithinWorkingDirectory(filePath, context)) {
    return {
      type: 'denied',
      reason: `Path ${filePath} is outside the allowed working directory`,
    }
  }
  
  // 2. 检查路径规则
  const ruleResult = checkPathRules(tool, filePath, context)
  if (ruleResult) return ruleResult
  
  // 3. 读取权限通常允许
  return { type: 'allowed' }
}

export async function checkWritePermissionForTool(
  tool: Tool,
  input: { file_path: string },
  context: ToolPermissionContext
): Promise<PermissionDecision> {
  const filePath = expandPath(input.file_path)
  
  // 1. 检查是否在允许的工作目录内
  if (!isWithinWorkingDirectory(filePath, context)) {
    return {
      type: 'denied',
      reason: `Path ${filePath} is outside the allowed working directory`,
    }
  }
  
  // 2. 检查是否在额外工作目录内
  const additionalDir = context.additionalWorkingDirectories.get(filePath)
  if (additionalDir) {
    return { type: 'allowed' }
  }
  
  // 3. 检查路径规则
  const ruleResult = checkPathRules(tool, filePath, context)
  if (ruleResult) return ruleResult
  
  // 4. 检查是否 protected namespace
  if (isInProtectedNamespace(filePath)) {
    return {
      type: 'needs_permission',
      message: `Path ${filePath} is in a protected namespace`,
    }
  }
  
  // 5. 需要确认
  return { type: 'needs_permission', message: 'Write confirmation required' }
}

function isWithinWorkingDirectory(
  filePath: string,
  context: ToolPermissionContext
): boolean {
  const cwd = getCwd()
  
  // 检查是否在 CWD 内
  if (filePath.startsWith(cwd)) {
    return true
  }
  
  // 检查额外工作目录
  for (const [dirPath] of context.additionalWorkingDirectories) {
    if (filePath.startsWith(dirPath)) {
      return true
    }
  }
  
  return false
}
```

### 通配符匹配

```typescript
// src/utils/permissions/shellRuleMatching.ts

export function matchWildcardPattern(
  pattern: string,
  path: string
): boolean {
  // 转换为正则
  const regex = globToRegex(pattern)
  return regex.test(path)
}

function globToRegex(pattern: string): RegExp {
  let regex = ''
  let inGroup = false
  
  for (let i = 0; i < pattern.length; i++) {
    const c = pattern[i]
    
    switch (c) {
      case '*':
        if (pattern[i + 1] === '*') {
          // ** 匹配任意路径
          regex += '.*'
          i++
        } else {
          // * 匹配任意字符（除了 /）
          regex += '[^/]*'
        }
        break
      
      case '?':
        regex += '[^/]'
        break
      
      case '[':
        regex += '['
        inGroup = true
        break
      
      case ']':
        regex += ']'
        inGroup = false
        break
      
      case '{':
        regex += '(?:'
        break
      
      case '}':
        regex += ')'
        break
      
      case ',':
        if (inGroup) {
          regex += ','
        } else {
          regex += '|'
        }
        break
      
      case '/':
        regex += '/'
        break
      
      default:
        // 转义特殊字符
        if (/[\\^$.*+?()[\]{}|]/.test(c)) {
          regex += '\\' + c
        } else {
          regex += c
        }
    }
  }
  
  return new RegExp('^' + regex + '$')
}
```

## 拒绝跟踪

### 拒绝统计

```typescript
// src/utils/permissions/denialTracking.ts

export interface DenialTrackingState {
  // 每工具的拒绝计数
  toolDenials: Map<string, number>
  
  // 总计数
  totalDenials: number
  
  // 连续拒绝
  consecutiveDenials: number
  
  // 最后拒绝时间
  lastDenialAt: number | null
}

export function createDenialTrackingState(): DenialTrackingState {
  return {
    toolDenials: new Map(),
    totalDenials: 0,
    consecutiveDenials: 0,
    lastDenialAt: null,
  }
}

export function recordDenial(
  state: DenialTrackingState,
  toolName: string
): void {
  // 增加工具特定计数
  const current = state.toolDenials.get(toolName) ?? 0
  state.toolDenials.set(toolName, current + 1)
  
  // 增加总计
  state.totalDenials++
  state.consecutiveDenials++
  state.lastDenialAt = Date.now()
}

export function recordAllow(
  state: DenialTrackingState
): void {
  // 重置连续计数
  state.consecutiveDenials = 0
}

export function shouldSuggestAutoMode(
  state: DenialTrackingState
): boolean {
  // 如果连续拒绝超过阈值，建议自动模式
  return state.consecutiveDenials >= 5
}
```

## 权限更新

### 权限变更

```typescript
// src/utils/permissions/PermissionUpdate.ts

export type PermissionUpdate =
  | {
      type: 'addRules'
      destination: PermissionUpdateDestination
      rules: PermissionRuleValue[]
      behavior: PermissionBehavior
    }
  | {
      type: 'removeRules'
      destination: PermissionUpdateDestination
      rules: PermissionRuleValue[]
      behavior: PermissionBehavior
    }
  | {
      type: 'setMode'
      destination: PermissionUpdateDestination
      mode: ExternalPermissionMode
    }
  | {
      type: 'addDirectories'
      destination: PermissionUpdateDestination
      directories: string[]
    }
  | {
      type: 'removeDirectories'
      destination: PermissionUpdateDestination
      directories: string[]
    }

export type PermissionUpdateDestination =
  | 'userSettings'
  | 'projectSettings'
  | 'localSettings'
  | 'session'
  | 'cliArg'

export function applyPermissionUpdate(
  context: ToolPermissionContext,
  update: PermissionUpdate
): ToolPermissionContext {
  switch (update.type) {
    case 'addRules':
      return addRules(context, update)
    case 'removeRules':
      return removeRules(context, update)
    case 'setMode':
      return { ...context, mode: update.mode }
    case 'addDirectories':
      return addDirectories(context, update)
    case 'removeDirectories':
      return removeDirectories(context, update)
    default:
      return context
  }
}

export async function persistPermissionUpdate(
  update: PermissionUpdate
): Promise<void> {
  switch (update.destination) {
    case 'userSettings':
      await updateUserSettings(settings => ({
        ...settings,
        permissions: applyUpdateToSettings(settings.permissions, update),
      }))
      break
    case 'projectSettings':
      await updateProjectSettings(settings => ({
        ...settings,
        permissions: applyUpdateToSettings(settings.permissions, update),
      }))
      break
    case 'localSettings':
      await updateLocalSettings(settings => ({
        ...settings,
        permissions: applyUpdateToSettings(settings.permissions, update),
      }))
      break
    case 'session':
    case 'cliArg':
      // 不持久化
      break
  }
}
```

## UI 权限请求

### 权限请求组件

```typescript
// src/components/permissions/PermissionRequest.tsx

export interface ToolUseConfirm {
  tool: Tool
  input: unknown
  toolUseId: string
  onAllow: () => void
  onDeny: () => void
  onAlwaysAllow: () => void
  onAlwaysDeny: () => void
}

export const PermissionRequest: React.FC<{
  permission: ToolUseConfirm
}> = ({ permission }) => {
  const { tool, input, onAllow, onDeny, onAlwaysAllow, onAlwaysDeny } = permission
  
  const [showDetails, setShowDetails] = useState(false)
  const [selectedScope, setSelectedScope] = useState<'session' | 'project' | 'global'>('session')
  
  return (
    <Box borderStyle="single" borderColor="yellow" padding={1}>
      <Text bold>Permission Request</Text>
      
      <Box marginTop={1}>
        <Text>Tool: {tool.userFacingName}</Text>
        <Text>Input: {JSON.stringify(input, null, 2).slice(0, 200)}...</Text>
      </Box>
      
      {showDetails && (
        <Box marginTop={1}>
          <Text dimColor>Full input:</Text>
          <Text>{JSON.stringify(input, null, 2)}</Text>
        </Box>
      )}
      
      <Box marginTop={1}>
        <Text>Scope for Always Allow/Deny:</Text>
        <Box>
          <Button onPress={() => setSelectedScope('session')}>
            Session
          </Button>
          <Button onPress={() => setSelectedScope('project')}>
            Project
          </Button>
          <Button onPress={() => setSelectedScope('global')}>
            Global
          </Button>
        </Box>
      </Box>
      
      <Box marginTop={1}>
        <Button onPress={onAllow}>Allow</Button>
        <Button onPress={onDeny}>Deny</Button>
        <Button onPress={() => onAlwaysAllow(selectedScope)}>
          Always Allow
        </Button>
        <Button onPress={() => onAlwaysDeny(selectedScope)}>
          Always Deny
        </Button>
      </Box>
      
      <Button onPress={() => setShowDetails(!showDetails)}>
        {showDetails ? 'Hide' : 'Show'} Details
      </Button>
    </Box>
  )
}
```
