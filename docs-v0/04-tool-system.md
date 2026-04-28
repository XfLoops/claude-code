# 工具系统详解

## 工具系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      工具系统架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Tool.ts                              │   │
│  │              (工具接口定义)                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   tools.ts                              │   │
│  │               (工具注册表)                               │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│  │  │核心工具 │ │Agent工具│ │Web工具  │ │Task工具 │       │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              src/tools/<ToolName>/                      │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│  │  │ <Tool>.tsx  │  │  prompt.ts  │  │  types.ts   │     │   │
│  │  │ (执行逻辑)   │  │ (描述/提示) │  │(输入/输出)  │     │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Tool.ts 核心接口

### Tool 类型定义

```typescript
export type Tool = {
  // 基本信息
  name: string
  description: string | (() => Promise<string>)
  prompt?: string | (() => Promise<string>)
  
  // JSON Schema 输入定义
  inputSchema: ToolInputJSONSchema
  
  // 输出 Schema（用于验证）
  outputSchema?: z.ZodType<any>
  
  // 核心执行方法
  call: (
    input: unknown,
    context: ToolUseContext
  ) => Promise<ToolResult>
  
  // 权限相关
  checkPermissions?: (
    input: unknown,
    context: ToolUseContext
  ) => Promise<PermissionDecision>
  
  // 路径提取（用于权限检查）
  getPath?: (input: any) => string | string[]
  
  // UI 渲染
  userFacingName: string
  getActivityDescription?: (input: any) => string
  getToolUseSummary?: (input: any) => string | null
  
  // 渲染函数（可选）
  renderToolUseMessage?: (input: any) => string
  renderToolResultMessage?: (result: ToolResult) => string
  renderToolUseRejectedMessage?: (input: any) => string
  renderToolUseErrorMessage?: (error: Error, input: any) => string
  
  // 分类器输入转换
  toAutoClassifierInput?: (input: any) => string
  
  // 其他元数据
  searchHint?: string
  maxResultSizeChars?: number
  strict?: boolean
  isReadOnly?: boolean
  isInteractive?: boolean
}

export type Tools = readonly Tool[]
```

### ToolUseContext

```typescript
export type ToolUseContext = {
  options: {
    commands: Command[]
    debug: boolean
    mainLoopModel: string
    tools: Tools
    verbose: boolean
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    customSystemPrompt?: string
    appendSystemPrompt?: string
    querySource?: QuerySource
    refreshTools?: () => Tools
  }
  
  // 状态管理
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  setAppStateForTasks?: (f: (prev: AppState) => AppState) => void
  
  // UI 回调
  setToolJSX?: SetToolJSXFn
  addNotification?: (notif: Notification) => void
  appendSystemMessage?: (msg: Exclude<SystemMessage, SystemLocalCommandMessage>) => void
  sendOSNotification?: (opts: { message: string; notificationType: string }) => void
  
  // 消息和跟踪
  messages: Message[]
  toolDecisions?: Map<string, { source: string; decision: 'accept' | 'reject'; timestamp: number }>
  queryTracking?: QueryChainTracking
  
  // 提示请求
  requestPrompt?: (sourceName: string, toolInputSummary?: string | null) => (request: PromptRequest) => Promise<PromptResponse>
  
  // 其他
  toolUseId?: string
  nestedMemoryAttachmentTriggers?: Set<string>
  loadedNestedMemoryPaths?: Set<string>
  dynamicSkillDirTriggers?: Set<string>
  discoveredSkillNames?: Set<string>
  userModified?: boolean
  setInProgressToolUseIDs: (f: (prev: Set<string>) => Set<string>) => void
  setHasInterruptibleToolInProgress?: (v: boolean) => void
  setResponseLength: (f: (prev: number) => number) => void
  pushApiMetricsEntry?: (ttftMs: number) => void
  setStreamMode?: (mode: SpinnerMode) => void
  onCompactProgress?: (event: CompactProgressEvent) => void
  setSDKStatus?: (status: SDKStatus) => void
  openMessageSelector?: () => void
  updateFileHistoryState: (updater: (prev: FileHistoryState) => FileHistoryState) => void
  updateAttributionState: (updater: (prev: AttributionState) => AttributionState) => void
  setConversationId?: (id: UUID) => void
  agentId?: AgentId
  agentType?: string
  requireCanUseTool?: boolean
  localDenialTracking?: DenialTrackingState
  contentReplacementState?: ContentReplacementState
  renderedSystemPrompt?: SystemPrompt
}
```

### ToolResult 类型

```typescript
export type ToolResult = {
  // 成功结果
  content?: string | ContentBlockParam[]
  
  // 错误信息
  error?: string
  isError?: boolean
  
  // 元数据
  _type?: 'tool_result'
  tool_use_id?: string
  
  // 额外数据（用于内部传递）
  [key: string]: unknown
}
```

## tools.ts 注册表

### 工具组装

```typescript
export function getTools(options: ToolOptions): Tools {
  const tools: Tool[] = [
    // 核心文件工具
    FileReadTool,
    FileWriteTool,
    FileEditTool,
    GlobTool,
    GrepTool,
    
    // 执行工具
    BashTool,
    
    // Web 工具
    WebSearchTool,
    WebFetchTool,
    
    // 任务工具
    TaskCreateTool,
    TaskGetTool,
    TaskUpdateTool,
    TaskListTool,
    TaskStopTool,
    TaskOutputTool,
    
    // Agent 工具
    AgentTool,
    SkillTool,
    
    // 交互工具
    AskUserQuestionTool,
    TodoWriteTool,
    
    // MCP 工具
    ListMcpResourcesTool,
    ReadMcpResourceTool,
    
    // 其他
    NotebookEditTool,
    BriefTool,
    ConfigTool,
    ExitPlanModeV2Tool,
    EnterPlanModeTool,
    EnterWorktreeTool,
    ExitWorktreeTool,
    LSPTool,
    TungstenTool,
    ToolSearchTool,
    TestingPermissionTool,
    ...getConditionalTools(options),
  ]
  
  return tools.filter((tool): tool is Tool => tool !== null)
}
```

### 条件工具加载

```typescript
function getConditionalTools(options: ToolOptions): (Tool | null)[] {
  return [
    // Ant-only 工具
    process.env.USER_TYPE === 'ant' ? REPLTool : null,
    process.env.USER_TYPE === 'ant' ? SuggestBackgroundPRTool : null,
    
    // Feature-gated 工具
    feature('PROACTIVE') || feature('KAIROS') ? SleepTool : null,
    feature('AGENT_TRIGGERS') ? CronCreateTool : null,
    feature('AGENT_TRIGGERS') ? CronDeleteTool : null,
    feature('AGENT_TRIGGERS') ? CronListTool : null,
    feature('AGENT_TRIGGERS_REMOTE') ? RemoteTriggerTool : null,
    feature('MONITOR_TOOL') ? MonitorTool : null,
    feature('KAIROS') ? SendUserFileTool : null,
    feature('KAIROS') || feature('KAIROS_PUSH_NOTIFICATION') ? PushNotificationTool : null,
    feature('KAIROS_GITHUB_WEBHOOKS') ? SubscribePRTool : null,
    
    // 环境变量控制
    process.env.CLAUDE_CODE_VERIFY_PLAN === 'true' ? VerifyPlanExecutionTool : null,
  ]
}
```

## 核心工具详解

### BashTool

```typescript
// src/tools/BashTool/BashTool.tsx

export const BashTool = buildTool({
  name: BASH_TOOL_NAME,
  searchHint: 'execute bash commands',
  maxResultSizeChars: 100_000,
  isReadOnly: false,
  
  async description() {
    return getBashToolDescription()
  },
  
  async prompt() {
    return getBashToolPrompt()
  },
  
  userFacingName: 'Bash',
  
  getActivityDescription(input) {
    return `Running ${input.command.slice(0, 50)}...`
  },
  
  get inputSchema() {
    return inputSchema()
  },
  
  getPath(input) {
    return input.cwd ?? '.'
  },
  
  toAutoClassifierInput(input) {
    return input.command
  },
  
  async checkPermissions(input, context): Promise<PermissionDecision> {
    return checkBashPermission(input, context)
  },
  
  async validateInput(input: BashInput, context: ToolUseContext) {
    // 验证输入
    if (!input.command || input.command.trim() === '') {
      return { result: false, message: 'Command is required', errorCode: 0 }
    }
    
    // 检查命令是否在允许列表
    const validation = validateBashCommand(input.command, context)
    if (!validation.valid) {
      return { result: false, message: validation.error, errorCode: 1 }
    }
    
    return { result: true }
  },
  
  async call(input: BashInput, context: ToolUseContext): Promise<ToolResult> {
    const { command, cwd, timeout, description } = input
    
    // 设置工作目录
    const workingDir = cwd || context.options.cwd
    
    // 执行命令
    const result = await executeCommand({
      command,
      cwd: workingDir,
      timeout: timeout ?? DEFAULT_TIMEOUT,
      description,
      abortController: context.abortController,
    })
    
    // 记录到历史
    prependToShellHistoryCache(command, workingDir)
    
    // 构建结果
    return {
      content: formatCommandOutput(result),
      tool_use_id: context.toolUseId,
    }
  },
  
  renderToolUseMessage(input) {
    return `$ ${input.command}`
  },
  
  renderToolResultMessage(result) {
    return result.content?.toString() || ''
  },
})
```

### FileEditTool

```typescript
// src/tools/FileEditTool/FileEditTool.ts

export const FileEditTool = buildTool({
  name: FILE_EDIT_TOOL_NAME,
  searchHint: 'modify file contents in place',
  maxResultSizeChars: 100_000,
  strict: true,
  
  async description() {
    return 'A tool for editing files'
  },
  
  async prompt() {
    return getEditToolDescription()
  },
  
  userFacingName,
  getToolUseSummary,
  
  getActivityDescription(input) {
    const summary = getToolUseSummary(input)
    return summary ? `Editing ${summary}` : 'Editing file'
  },
  
  get inputSchema() {
    return inputSchema()
  },
  
  get outputSchema() {
    return outputSchema()
  },
  
  toAutoClassifierInput(input) {
    return `${input.file_path}: ${input.new_string}`
  },
  
  getPath(input) {
    return input.file_path
  },
  
  backfillObservableInput(input) {
    // 展开路径
    if (typeof input.file_path === 'string') {
      input.file_path = expandPath(input.file_path)
    }
  },
  
  async preparePermissionMatcher({ file_path }) {
    return pattern => matchWildcardPattern(pattern, file_path)
  },
  
  async checkPermissions(input, context): Promise<PermissionDecision> {
    const appState = context.getAppState()
    return checkWritePermissionForTool(
      FileEditTool,
      input,
      appState.toolPermissionContext,
    )
  },
  
  async validateInput(input: FileEditInput, context: ToolUseContext) {
    const { file_path, old_string, new_string, replace_all = false } = input
    const fullFilePath = expandPath(file_path)
    
    // 检查团队内存密钥
    const secretError = checkTeamMemSecrets(fullFilePath, new_string)
    if (secretError) {
      return { result: false, message: secretError, errorCode: 0 }
    }
    
    // 检查是否有变化
    if (old_string === new_string) {
      return {
        result: false,
        message: 'old_string and new_string are identical. You must provide different strings to make an edit.',
        errorCode: 1,
      }
    }
    
    // 检查文件存在性
    const fs = getFsImplementation()
    const exists = await fs.exists(fullFilePath)
    if (!exists) {
      return handleFileNotFound(fullFilePath, context)
    }
    
    // 检查文件大小
    const stats = await fs.stat(fullFilePath)
    if (stats.size > MAX_EDIT_FILE_SIZE) {
      return {
        result: false,
        message: `File too large (${formatFileSize(stats.size)}). Maximum size is ${formatFileSize(MAX_EDIT_FILE_SIZE)}.`,
        errorCode: 2,
      }
    }
    
    // 读取文件内容
    const { content, lineEnding } = await readFileSyncWithMetadata(fullFilePath)
    
    // 验证 old_string 匹配
    if (!replace_all) {
      const matchResult = findActualString(content, old_string)
      if (!matchResult.found) {
        return {
          result: false,
          message: buildNoMatchError(content, old_string),
          errorCode: 3,
        }
      }
    }
    
    return { result: true }
  },
  
  async call(input: FileEditInput, context: ToolUseContext): Promise<ToolResult> {
    const { file_path, old_string, new_string, replace_all = false } = input
    const fullFilePath = expandPath(file_path)
    
    // 读取当前内容
    const fs = getFsImplementation()
    const { content, lineEnding } = await readFileSyncWithMetadata(fullFilePath)
    
    // 执行编辑
    let newContent: string
    if (replace_all) {
      newContent = content.split(old_string).join(new_string)
    } else {
      newContent = getPatchForEdit(content, old_string, new_string)
    }
    
    // 保留引号风格
    newContent = preserveQuoteStyle(content, newContent)
    
    // 写入文件
    await writeTextContent(fullFilePath, newContent, lineEnding)
    
    // 更新文件历史
    if (fileHistoryEnabled()) {
      fileHistoryTrackEdit(fullFilePath, old_string, new_string)
    }
    
    // 通知 VSCode
    notifyVscodeFileUpdated(fullFilePath)
    
    // 记录分析
    logFileOperation('edit', fullFilePath, countLinesChanged(old_string, new_string))
    
    // 激活条件技能
    const skillDirs = discoverSkillDirsForPaths([fullFilePath])
    addSkillDirectories(skillDirs)
    
    // 清除诊断
    clearDeliveredDiagnosticsForFile(fullFilePath)
    
    // 获取 git diff
    const diff = await fetchSingleFileGitDiff(fullFilePath)
    
    return {
      content: renderToolResultMessage({ filePath: fullFilePath, diff }),
      tool_use_id: context.toolUseId,
    }
  },
  
  renderToolUseMessage,
  renderToolResultMessage,
  renderToolUseRejectedMessage,
  renderToolUseErrorMessage,
})
```

### AgentTool

```typescript
// src/tools/AgentTool/AgentTool.ts

export const AgentTool = buildTool({
  name: AGENT_TOOL_NAME,
  searchHint: 'spawn subagents for parallel or complex tasks',
  maxResultSizeChars: 500_000,
  isReadOnly: false,
  
  async description() {
    return getAgentToolDescription()
  },
  
  async prompt() {
    return getAgentToolPrompt()
  },
  
  userFacingName: 'Agent',
  
  getActivityDescription(input) {
    return `Running agent: ${input.description.slice(0, 50)}...`
  },
  
  get inputSchema() {
    return inputSchema()
  },
  
  toAutoClassifierInput(input) {
    return `${input.agent_type}: ${input.description}`
  },
  
  async validateInput(input: AgentInput, context: ToolUseContext) {
    // 验证 agent 类型
    if (!isValidAgentType(input.agent_type, context)) {
      return {
        result: false,
        message: `Invalid agent type: ${input.agent_type}`,
        errorCode: 0,
      }
    }
    
    // 验证深度限制
    const currentDepth = context.queryTracking?.depth ?? 0
    if (currentDepth >= MAX_AGENT_DEPTH) {
      return {
        result: false,
        message: `Maximum agent nesting depth (${MAX_AGENT_DEPTH}) exceeded`,
        errorCode: 1,
      }
    }
    
    return { result: true }
  },
  
  async call(input: AgentInput, context: ToolUseContext): Promise<ToolResult> {
    const { agent_type, description, artifacts } = input
    
    // 创建子 agent 上下文
    const agentContext = createSubagentContext(context, {
      agentType: agent_type,
      description,
    })
    
    // 创建 agent 任务
    const task = createAgentTask({
      agentType: agent_type,
      description,
      artifacts,
      parentContext: agentContext,
    })
    
    // 启动 agent
    const result = await runAgentTask(task)
    
    return {
      content: result.output,
      tool_use_id: context.toolUseId,
    }
  },
  
  renderToolUseMessage(input) {
    return `🤖 Agent (${input.agent_type}): ${input.description}`
  },
  
  renderToolResultMessage(result) {
    return result.content?.toString() || ''
  },
})
```

## 工具构建辅助函数

### buildTool 辅助函数

```typescript
// src/Tool.ts

export function buildTool<TInput, TOutput>(
  config: ToolConfig<TInput, TOutput>
): Tool {
  return {
    // 必需字段
    name: config.name,
    description: config.description,
    inputSchema: config.inputSchema,
    call: config.call,
    userFacingName: config.userFacingName,
    
    // 可选字段
    prompt: config.prompt,
    outputSchema: config.outputSchema,
    checkPermissions: config.checkPermissions,
    getPath: config.getPath,
    getActivityDescription: config.getActivityDescription,
    getToolUseSummary: config.getToolUseSummary,
    renderToolUseMessage: config.renderToolUseMessage,
    renderToolResultMessage: config.renderToolResultMessage,
    renderToolUseRejectedMessage: config.renderToolUseRejectedMessage,
    renderToolUseErrorMessage: config.renderToolUseErrorMessage,
    toAutoClassifierInput: config.toAutoClassifierInput,
    backfillObservableInput: config.backfillObservableInput,
    preparePermissionMatcher: config.preparePermissionMatcher,
    validateInput: config.validateInput,
    searchHint: config.searchHint,
    maxResultSizeChars: config.maxResultSizeChars,
    strict: config.strict,
    isReadOnly: config.isReadOnly,
    isInteractive: config.isInteractive,
  }
}
```

### 工具查找

```typescript
export function findToolByName(tools: Tools, name: string): Tool | undefined {
  return tools.find(tool => toolMatchesName(tool, name))
}

export function toolMatchesName(tool: Tool, name: string): boolean {
  return tool.name === name || tool.aliases?.includes(name)
}
```

## 工具权限系统

### 权限决策类型

```typescript
export type PermissionDecision =
  | { type: 'allowed' }
  | { type: 'denied'; reason: string }
  | { type: 'needs_permission'; message: string }
```

### 权限检查流程

```typescript
async function checkToolPermission(
  tool: Tool,
  input: unknown,
  context: ToolUseContext
): Promise<PermissionDecision> {
  // 1. 检查工具特定权限
  if (tool.checkPermissions) {
    const result = await tool.checkPermissions(input, context)
    if (result.type !== 'allowed') {
      return result
    }
  }
  
  // 2. 检查全局权限上下文
  const appState = context.getAppState()
  const permissionContext = appState.toolPermissionContext
  
  // 3. 检查模式
  switch (permissionContext.mode) {
    case 'acceptEdits':
      if (!tool.isReadOnly) {
        return { type: 'allowed' }
      }
      break
    case 'bypassPermissions':
      return { type: 'allowed' }
    case 'plan':
      // 计划模式下需要用户确认
      return { type: 'needs_permission', message: 'Plan mode requires confirmation' }
    case 'auto':
      // 自动分类器决策
      return await autoClassifierDecision(tool, input, context)
  }
  
  // 4. 检查规则
  const path = tool.getPath?.(input)
  if (path) {
    const rule = matchingRuleForInput(permissionContext, tool.name, path)
    if (rule) {
      switch (rule.behavior) {
        case 'allow':
          return { type: 'allowed' }
        case 'deny':
          return { type: 'denied', reason: `Denied by rule: ${rule.source}` }
        case 'ask':
          return { type: 'needs_permission', message: 'Rule requires confirmation' }
      }
    }
  }
  
  // 5. 默认行为
  if (tool.isReadOnly) {
    return { type: 'allowed' }
  }
  
  return { type: 'needs_permission', message: 'Confirmation required' }
}
```

## 工具执行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      工具执行流程                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. 查找工具                                                     │
│    - 通过 name 匹配 tools.ts 中的工具列表                        │
│    - 支持别名匹配                                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 验证输入                                                     │
│    - JSON Schema 验证                                           │
│    - validateInput() 自定义验证                                 │
│    - 路径展开和规范化                                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 权限检查                                                     │
│    - checkPermissions() 工具特定检查                            │
│    - 全局权限上下文检查                                         │
│    - 规则匹配                                                   │
│    - 模式决策（acceptEdits/bypassPermissions/plan/auto）         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 执行工具                                                     │
│    - 调用 tool.call(input, context)                             │
│    - 支持 abortController 取消                                  │
│    - 流式进度报告                                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 处理结果                                                     │
│    - 格式化输出                                                 │
│    - 更新文件历史                                               │
│    - 触发技能发现                                               │
│    - 记录分析                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## 工具结果存储

### ContentReplacementState

```typescript
export type ContentReplacementState = {
  replacements: Map<string, ReplacementRecord>
  maxSize: number
}

type ReplacementRecord = {
  originalSize: number
  replacedSize: number
  replacementId: string
  timestamp: number
}

export function applyToolResultBudget(
  content: string,
  state: ContentReplacementState
): { content: string; wasReplaced: boolean } {
  if (content.length <= state.maxSize) {
    return { content, wasReplaced: false }
  }
  
  const replacementId = randomUUID()
  const truncated = content.slice(0, state.maxSize)
  const placeholder = `[Content replaced: ${replacementId}]`
  
  state.replacements.set(replacementId, {
    originalSize: content.length,
    replacedSize: truncated.length,
    replacementId,
    timestamp: Date.now(),
  })
  
  return { content: truncated + placeholder, wasReplaced: true }
}
```
