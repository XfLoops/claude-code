# 上下文构建与系统提示详解

## 上下文系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                   上下文构建系统                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     context.ts                          │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│  │  │ System      │  │   User      │  │  CLAUDE.md  │     │   │
│  │  │ Context     │  │  Context    │  │  Loading    │     │   │
│  │  │ (系统上下文) │  │ (用户上下文) │  │(项目记忆)   │     │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 constants/prompts.ts                     │   │
│  │              (系统提示生成与组装)                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 utils/claudemd.ts                        │   │
│  │              (CLAUDE.md 文件发现与加载)                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 系统上下文构建

### System Context

```typescript
// src/context.ts

const MAX_STATUS_CHARS = 2000

/**
 * 系统提示注入（仅 ant 内部使用，用于缓存破坏调试）
 */
let systemPromptInjection: string | null = null

export function setSystemPromptInjection(value: string | null): void {
  systemPromptInjection = value
  // 清除上下文缓存
  getUserContext.cache.clear?.()
  getSystemContext.cache.clear?.()
}

/**
 * 获取系统上下文（每会话缓存）
 */
export const getSystemContext = memoize(
  async (): Promise<Record<string, string>> => {
    const startTime = Date.now()
    logForDiagnosticsNoPII('info', 'system_context_started')
    
    // 获取 Git 状态
    const gitStatus = 
      isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) ||
      !shouldIncludeGitInstructions()
        ? null
        : await getGitStatus()
    
    // 系统提示注入（仅 ant）
    const injection = feature('BREAK_CACHE_COMMAND')
      ? getSystemPromptInjection()
      : null
    
    logForDiagnosticsNoPII('info', 'system_context_completed', {
      duration_ms: Date.now() - startTime,
      has_git_status: gitStatus !== null,
      has_injection: injection !== null,
    })
    
    return {
      ...(gitStatus && { gitStatus }),
      ...(feature('BREAK_CACHE_COMMAND') && injection
        ? { cacheBreaker: `[CACHE_BREAKER: ${injection}]` }
        : {}),
    }
  }
)

/**
 * 获取 Git 状态
 */
export const getGitStatus = memoize(async (): Promise<string | null> => {
  if (process.env.NODE_ENV === 'test') {
    return null
  }
  
  const startTime = Date.now()
  
  const isGit = await getIsGit()
  if (!isGit) {
    return null
  }
  
  try {
    const [branch, mainBranch, status, log, userName] = await Promise.all([
      getBranch(),
      getDefaultBranch(),
      execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short'])
        .then(({ stdout }) => stdout.trim()),
      execFileNoThrow(
        gitExe(),
        ['--no-optional-locks', 'log', '--oneline', '-n', '5'],
        { preserveOutputOnError: false }
      ).then(({ stdout }) => stdout.trim()),
      execFileNoThrow(gitExe(), ['config', 'user.name'], {
        preserveOutputOnError: false,
      }).then(({ stdout }) => stdout.trim()),
    ])
    
    // 截断过长的状态
    const truncatedStatus =
      status.length > MAX_STATUS_CHARS
        ? status.substring(0, MAX_STATUS_CHARS) +
          '\n... (truncated)'
        : status
    
    return [
      'This is the git status at the start of the conversation.',
      'Current branch: ${branch}',
      'Main branch: ${mainBranch}',
      ...(userName ? [`Git user: ${userName}`] : []),
      `Status:\n${truncatedStatus || '(clean)'}`,
      `Recent commits:\n${log}`,
    ].join('\n\n')
  } catch (error) {
    logError(error)
    return null
  }
})
```

### 用户上下文

```typescript
// src/context.ts

/**
 * 用户上下文（每会话缓存）
 */
export const getUserContext = memoize(
  async (): Promise<Record<string, string>> => {
    const startTime = Date.now()
    logForDiagnosticsNoPII('info', 'user_context_started')
    
    const context: Record<string, string> = {}
    
    // 获取平台信息
    const platform = getPlatform()
    context.platform = `${platform.os} ${platform.version}`
    
    // 获取 shell
    context.shell = getDefaultShell()
    
    // 获取终端信息
    const terminal = getTerminalInfo()
    context.terminal = terminal.name
    
    // 获取项目 CLAUDE.md
    const claudeMdContent = await getClaudeMdContent()
    if (claudeMdContent) {
      context.claudeMd = claudeMdContent
    }
    
    logForDiagnosticsNoPII('info', 'user_context_completed', {
      duration_ms: Date.now() - startTime,
      has_claude_md: !!claudeMdContent,
    })
    
    return context
  }
)
```

## CLAUDE.md 加载系统

### 文件发现

```typescript
// src/utils/claudemd.ts

export interface ClaudeMdFile {
  path: string
  content: string
  scope: 'project' | 'local' | 'user' | 'flag'
  priority: number
}

/**
 * 发现和加载所有 CLAUDE.md 文件
 */
export async function getClaudeMds(): Promise<ClaudeMdFile[]> {
  const files: ClaudeMdFile[] = []
  
  // 1. 用户级配置
  const userClaudeMd = await loadUserClaudeMd()
  if (userClaudeMd) {
    files.push({
      path: userClaudeMd.path,
      content: userClaudeMd.content,
      scope: 'user',
      priority: 0,
    })
  }
  
  // 2. 项目配置（从当前目录向上查找）
  const projectRoot = await findProjectRoot()
  if (projectRoot) {
    const projectClaudeMd = await loadProjectClaudeMd(projectRoot)
    if (projectClaudeMd) {
      files.push({
        path: projectClaudeMd.path,
        content: projectClaudeMd.content,
        scope: 'project',
        priority: 1,
      })
    }
  }
  
  // 3. 本地配置
  const localClaudeMd = await loadLocalClaudeMd()
  if (localClaudeMd) {
    files.push({
      path: localClaudeMd.path,
      content: localClaudeMd.content,
      scope: 'local',
      priority: 2,
    })
  }
  
  // 4. Flag 配置（通过 --claude-md 指定）
  const flagClaudeMd = await loadFlagClaudeMd()
  if (flagClaudeMd) {
    files.push({
      path: flagClaudeMd.path,
      content: flagClaudeMd.content,
      scope: 'flag',
      priority: 3,
    })
  }
  
  // 按优先级排序
  return files.sort((a, b) => a.priority - b.priority)
}

/**
 * 获取记忆文件
 */
export async function getMemoryFiles(): Promise<MemoryFile[]> {
  const memories: MemoryFile[] = []
  const claudeMds = await getClaudeMds()
  
  for (const claudeMd of claudeMds) {
    // 解析 CLAUDE.md 中的记忆引用
    const memoryRefs = parseMemoryReferences(claudeMd.content)
    
    for (const ref of memoryRefs) {
      const resolvedPath = resolve(claudeMd.path, '..', ref.path)
      const content = await readFileSafe(resolvedPath)
      if (content !== null) {
        memories.push({
          path: resolvedPath,
          content,
          source: claudeMd.path,
          mtimeMs: (await stat(resolvedPath)).mtimeMs,
        })
      }
    }
  }
  
  return memories
}

function parseMemoryReferences(content: string): MemoryReference[] {
  const refs: MemoryReference[] = []
  
  // 匹配 <!-- mem: path/to/file.md --> 注释
  const memRegex = /<!--\s*mem:\s*(.+?)\s*-->/g
  let match
  
  while ((match = memRegex.exec(content)) !== null) {
    refs.push({ path: match[1].trim() })
  }
  
  return refs
}
```

## 系统提示生成

### Prompts 主模块

```typescript
// src/constants/prompts.ts

/**
 * 动态内容边界标记
 * 标记之前的内容可使用全局缓存
 * 标记之后的内容包含用户/会话特定内容，不应缓存
 */
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = 
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'

// 前沿模型名称
const FRONTIER_MODEL_NAME = 'Claude Opus 4.6'

// 模型家族 ID
const CLAUDE_4_5_OR_4_6_MODEL_IDS = {
  opus: 'claude-opus-4-6',
  sonnet: 'claude-sonnet-4-6',
  haiku: 'claude-haiku-4-5-20251001',
}

/**
 * 获取系统提示
 */
export async function getSystemPrompt(
  memoryAttachments: MemoryAttachment[],
  model: string,
  options?: SystemPromptOptions
): Promise<SystemPrompt> {
  const sections: string[] = []
  
  // 1. 基础身份提示
  sections.push(getIdentitySection())
  
  // 2. 工具使用指南
  sections.push(await getToolUseGuidelines(model, options))
  
  // 3. 思考指南（支持 thinking 的模型）
  if (shouldEnableThinkingByDefault(model)) {
    sections.push(getThinkingSection())
  }
  
  // 4. 系统提醒
  sections.push(getSystemRemindersSection())
  
  // 5. Hooks 说明
  sections.push(getHooksSection())
  
  // === 动态边界 ===
  sections.push(SYSTEM_PROMPT_DYNAMIC_BOUNDARY)
  
  // 6. 当前上下文
  const context = await getCurrentContextSection(model, options)
  sections.push(context)
  
  // 7. 记忆文件
  if (memoryAttachments.length > 0) {
    sections.push(getMemorySection(memoryAttachments))
  }
  
  // 8. 语言偏好
  const languageSection = getLanguageSection(options?.languagePreference)
  if (languageSection) {
    sections.push(languageSection)
  }
  
  // 9. 当前时间
  sections.push(getCurrentDateSection())
  
  return sections
}

/**
 * 获取工具使用指南
 */
async function getToolUseGuidelines(
  model: string,
  options?: SystemPromptOptions
): Promise<string> {
  const guidelines: string[] = []
  
  // 工具使用总体原则
  guidelines.push(`
# Tool Use

You have access to a set of tools. Follow these guidelines:

1. **Choose the right tool** for each task
2. **Always prefer specialized tools** over BashTool for file operations
3. **Batch multiple tool calls** when they are independent
4. **Wait for tool results** before making dependent calls
5. **Respect file size limits**: ${MAX_FILE_SIZE_MB}MB max
  `)
  
  // 文件操作工具优先级
  guidelines.push(`
## File Operations Priority

When modifying files, use in this order:
1. **FileEditTool** - for precise edits with old_string/new_string
2. **FileWriteTool** - for creating new files or complete rewrites
3. **BashTool** - only when file tools are insufficient
  `)
  
  // Agent 工具说明
  guidelines.push(`
## Agent Tool

Spawn subagents for parallel tasks or complex multi-step operations.
- Each agent runs in isolation
- Agents can use most tools (except nested AgentTool)
- Always provide clear, specific instructions
  `)
  
  // 任务工具说明
  guidelines.push(`
## Task Tools

Use TodoWriteTool to track progress on complex tasks.
Use TaskCreateTool to spawn background tasks.
  `)
  
  return guidelines.join('\n\n')
}

/**
 * 获取当前上下文
 */
async function getCurrentContextSection(
  model: string,
  options?: SystemPromptOptions
): Promise<string> {
  const sections: string[] = ['# Current Context']
  
  // 系统上下文
  const systemContext = await getSystemContext()
  if (systemContext.gitStatus) {
    sections.push('## Git Status', systemContext.gitStatus)
  }
  
  // 用户上下文
  const userContext = await getUserContext()
  if (userContext.claudeMd) {
    sections.push('## Project Guidelines (CLAUDE.md)', userContext.claudeMd)
  }
  
  return sections.join('\n\n')
}

/**
 * 获取记忆部分
 */
function getMemorySection(attachments: MemoryAttachment[]): string {
  const lines = ['# Relevant Context from Memory']
  
  for (const attachment of attachments) {
    lines.push(
      `## ${attachment.path}`,
      '```',
      attachment.content.slice(0, MAX_MEMORY_CONTENT_LENGTH),
      '```'
    )
  }
  
  return lines.join('\n\n')
}
```

## 系统提示部分

### 各部分详解

```typescript
// src/constants/systemPromptSections.ts

/**
 * 系统提示部分构建器
 */
export function systemPromptSection(
  id: string,
  builder: () => string
): SystemPromptSection {
  return {
    id,
    build: builder,
    // 缓存策略
    cacheable: id.startsWith('static_'),
  }
}

/**
 * 身份部分
 */
export const identitySection = systemPromptSection('identity', () => `
You are Claude Code, an AI assistant created by Anthropic.
Your goal is to help users with software engineering tasks.

Key principles:
- Be concise and direct
- Focus on code quality and best practices
- Explain complex concepts clearly
- Ask clarifying questions when needed
`)

/**
 * 工具使用部分
 */
export const toolUseSection = systemPromptSection('tool_use', () => `
# Tools

You have access to tools that help you accomplish tasks:

1. **File Tools** - Read, write, and edit files
2. **Search Tools** - Find files and search content
3. **Execution Tools** - Run shell commands
4. **Web Tools** - Search and fetch web content
5. **Agent Tools** - Spawn subagents for parallel work
6. **Task Tools** - Manage todo lists and background tasks

Use the most appropriate tool for each task.
`)

/**
 * 思考部分
 */
export const thinkingSection = systemPromptSection('thinking', () => `
# Thinking Mode

When thinking mode is enabled:
1. Show your reasoning before providing answers
2. Consider multiple approaches
3. Explain trade-offs
4. Be thorough in complex problem-solving

Format your thinking with <thinking> tags.
`)

/**
 * 安全部分
 */
export const securitySection = systemPromptSection('security', () => `
# Security Guidelines

1. Never commit secrets or credentials
2. Validate all user inputs
3. Use parameterized queries for databases
4. Be cautious with file paths and user input
5. Follow OWASP best practices
`)

/**
 * 解决系统提示部分
 */
export function resolveSystemPromptSections(
  options: ResolveOptions
): string[] {
  const sections: string[] = []
  
  // 始终包含的部分
  sections.push(identitySection.build())
  sections.push(toolUseSection.build())
  
  // 条件部分
  if (options.includeThinking) {
    sections.push(thinkingSection.build())
  }
  
  if (options.includeSecurity) {
    sections.push(securitySection.build())
  }
  
  // 模型特定部分
  if (isClaude4Model(options.model)) {
    sections.push(getClaude4SpecificPrompt())
  }
  
  return sections
}
```

## 提示缓存优化

### 缓存策略

```typescript
// src/utils/api.ts

export type CacheScope = 'global' | 'local'

export interface CacheConfig {
  scope: CacheScope
  ttlSeconds?: number
}

/**
 * 分割系统提示前缀
 * 用于区分可缓存和不可缓存部分
 */
export function splitSysPromptPrefix(
  prompt: SystemPrompt
): { prefix: string; suffix: string } {
  const boundaryIndex = prompt.indexOf(SYSTEM_PROMPT_DYNAMIC_BOUNDARY)
  
  if (boundaryIndex === -1) {
    // 无边界标记，全部视为动态
    return { prefix: '', suffix: prompt.join('\n') }
  }
  
  const prefix = prompt.slice(0, boundaryIndex).join('\n')
  const suffix = prompt.slice(boundaryIndex + 1).join('\n')
  
  return { prefix, suffix }
}

/**
 * 构建系统提示块（带缓存）
 */
export function buildSystemPromptBlocks(
  prompt: SystemPrompt
): BetaTextBlockParam[] {
  const { prefix, suffix } = splitSysPromptPrefix(prompt)
  const blocks: BetaTextBlockParam[] = []
  
  // 前缀块（可缓存）
  if (prefix) {
    blocks.push({
      type: 'text',
      text: prefix,
      cache_control: { type: 'ephemeral' },
    })
  }
  
  // 后缀块（动态，不缓存）
  if (suffix) {
    blocks.push({
      type: 'text',
      text: suffix,
    })
  }
  
  return blocks
}

/**
 * 构建缓存控制头
 */
export function buildCacheControlHeaders(
  useGlobalCache: boolean
): Record<string, string> {
  const headers: Record<string, string> = {}
  
  if (useGlobalCache) {
    headers['anthropic-beta'] = 'prompt-caching-2024-07-31'
    headers['anthropic-prompt-caching'] = '1'
  }
  
  return headers
}
```

## 上下文统计

### Stats 收集

```typescript
// src/context/stats.ts

export interface ContextStats {
  // 令牌统计
  tokens: {
    systemPrompt: number
    messages: number
    contextWindow: number
    available: number
  }
  
  // 消息统计
  messages: {
    total: number
    user: number
    assistant: number
    system: number
    toolResults: number
  }
  
  // 上下文来源
  sources: {
    claudeMdFiles: number
    memoryFiles: number
    gitStatus: boolean
    systemContext: boolean
  }
}

export async function getContextStats(
  messages: Message[],
  systemPrompt: SystemPrompt
): Promise<ContextStats> {
  const systemPromptText = systemPrompt.join('\n')
  const messagesText = JSON.stringify(messages)
  
  return {
    tokens: {
      systemPrompt: await countTokens(systemPromptText),
      messages: await countTokens(messagesText),
      contextWindow: getContextWindowForModel(getMainLoopModel()),
      available: 0, // 计算得出
    },
    messages: {
      total: messages.length,
      user: messages.filter(m => m.type === 'user').length,
      assistant: messages.filter(m => m.type === 'assistant').length,
      system: messages.filter(m => m.type === 'system').length,
      toolResults: messages.filter(m => m.type === 'user' && m.toolUseResult).length,
    },
    sources: {
      claudeMdFiles: (await getClaudeMds()).length,
      memoryFiles: (await getMemoryFiles()).length,
      gitStatus: !!(await getGitStatus()),
      systemContext: Object.keys(await getSystemContext()).length > 0,
    },
  }
}
```

## 调试与诊断

### 提示转储

```typescript
// src/services/api/dumpPrompts.ts

export function createDumpPromptsFetch(
  messages: Message[],
  systemPrompt: SystemPrompt
): () => void {
  return () => {
    const dump = {
      timestamp: new Date().toISOString(),
      sessionId: getSessionId(),
      systemPrompt,
      messages: messages.map(m => ({
        type: m.type,
        uuid: m.uuid,
        content: truncate(m.message?.content?.toString() ?? '', 1000),
      })),
    }
    
    const dumpPath = join(os.tmpdir(), `claude-prompt-dump-${Date.now()}.json`)
    writeFileSync(dumpPath, JSON.stringify(dump, null, 2))
    
    logForDebugging(`Prompt dump saved to: ${dumpPath}`)
  }
}

export function dumpPromptsToConsole(
  messages: Message[],
  systemPrompt: SystemPrompt
): void {
  // biome-ignore lint/suspicious/noConsole: 调试用途
  console.log('=== SYSTEM PROMPT ===')
  console.log(systemPrompt.join('\n'))
  console.log('\n=== MESSAGES ===')
  for (const msg of messages) {
    console.log(`\n[${msg.type.toUpperCase()}] ${msg.uuid}`)
    console.log(msg.message?.content?.toString()?.slice(0, 500))
  }
}
```
