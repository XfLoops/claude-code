# 记忆系统详解

## 系统概述

Claude Code CLI 的记忆系统采用**双轨架构**：
1. **CLAUDE.md 系统** - 嵌套式项目规则文件（类似 `.cursorrules`）
2. **自动记忆系统 (Auto-Memory)** - AI 持久化记忆（学习用户偏好和项目上下文）

```
┌─────────────────────────────────────────────────────────────────┐
│                    记忆系统架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────┐      ┌─────────────────────────────┐  │
│  │   CLAUDE.md 系统    │      │    Auto-Memory 系统         │  │
│  │                     │      │                             │  │
│  │  ┌───────────────┐  │      │  ┌─────────────────────┐   │  │
│  │  │ 嵌套规则加载   │  │      │  │ Memory Scan         │   │  │
│  │  │ (claudemd.ts) │  │      │  │ (memoryScan.ts)     │   │  │
│  │  └───────────────┘  │      │  └─────────────────────┘   │  │
│  │          │          │      │             │              │  │
│  │          ▼          │      │             ▼              │  │
│  │  ┌───────────────┐  │      │  ┌─────────────────────┐   │  │
│  │  │ @include 解析  │  │      │  │ Relevance Search    │   │  │
│  │  │ (100+ 扩展名)  │  │      │  │ (findRelevant*)     │   │  │
│  │  └───────────────┘  │      │  └─────────────────────┘   │  │
│  │          │          │      │             │              │  │
│  │          ▼          │      │             ▼              │  │
│  │  ┌───────────────┐  │      │  ┌─────────────────────┐   │  │
│  │  │ Context 注入   │  │      │  │ Attachment 构建     │   │  │
│  │  │ (attachments)  │  │      │  │ (attachments.ts)    │   │  │
│  │  └───────────────┘  │      │  └─────────────────────┘   │  │
│  └─────────────────────┘      └─────────────────────────────┘  │
│                                                                 │
│                         │                                       │
│                         ▼                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              System Prompt 注入                          │   │
│  │         (src/context.ts, constants/prompts.ts)           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 自动记忆系统 (Auto-Memory)

### 目录结构

```
~/.claude/projects/{sanitized-git-root}/memory/
├── MEMORY.md              # 入口索引文件（自动维护）
├── user/
│   ├── user_role.md
│   └── feedback_style.md
├── feedback/
│   └── testing_approach.md
├── project/
│   ├── auth_refactor.md
│   └── api_v2_migration.md
└── reference/
    └── linear_ingest.md
```

### 路径解析

```typescript
// src/memdir/paths.ts

export function getAutoMemPath(): string | null {
  const projectRoot = getProjectRoot()
  if (!projectRoot) return null

  // 对项目路径进行清理，生成唯一标识
  const slug = sanitizePathSegment(projectRoot)
  const memoryDir = join(homedir(), '.claude', 'projects', slug, 'memory')
  return memoryDir
}

// 支持环境变量覆盖
const COWORK_MEMORY_PATH_OVERRIDE = process.env.CLAUDE_COWORK_MEMORY_PATH_OVERRIDE
const DISABLE_AUTO_MEMORY = isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_AUTO_MEMORY)

export function isAutoMemoryEnabled(): boolean {
  if (DISABLE_AUTO_MEMORY) return false
  if (isFeatureEnabled('DISABLE_MEMORY')) return false
  return true
}
```

**安全验证**：
- 拒绝相对路径（`../`）
- 拒绝根目录
- 拒绝 UNC 路径（Windows）
- 清理特殊字符

### 四种记忆类型

```typescript
// src/memdir/memoryTypes.ts

type MemoryType = 'user' | 'feedback' | 'project' | 'reference'

interface MemoryFile {
  path: string           // 绝对路径
  content: string        // 文件内容
  scope: 'project' | 'local' | 'user' | 'flag'  // 来源范围
  priority: number       // 加载优先级
}
```

| 类型 | 用途 | 示例 |
|------|------|------|
| **user** | 用户信息、角色、技能 | "用户是后端工程师，新接触前端" |
| **feedback** | 用户给出的反馈和偏好 | "不要添加末尾总结" |
| **project** | 项目状态、决策、目标 | "正在重构 auth 模块" |
| **reference** | 外部系统引用 | "Pipeline  bugs 在 Linear 项目 INGEST" |

### 文件格式

```markdown
---
name: 用户角色
description: 用户的技能背景和职责
type: user
---

用户是高级后端工程师，专注于分布式系统。
这是用户第一次接触这个项目的 React 前端代码。
```

### 扫描与发现

```typescript
// src/memdir/memoryScan.ts

export interface MemoryHeader {
  filename: string
  path: string
  mtime: number
  description: string
  type: MemoryType
}

export async function scanMemoryFiles(): Promise<MemoryHeader[]> {
  const memoryDir = getAutoMemPath()
  if (!memoryDir) return []

  const files: MemoryHeader[] = []
  const mdFiles = await glob('**/*.md', {
    cwd: memoryDir,
    ignore: ['MEMORY.md'],  // 排除入口文件
    absolute: true,
  })

  for (const file of mdFiles.slice(0, MAX_MEMORY_FILES)) {
    const stat = await safeStat(file)
    const frontmatter = await parseFrontmatter(file)

    files.push({
      filename: basename(file),
      path: file,
      mtime: stat?.mtimeMs ?? 0,
      description: frontmatter.description ?? '',
      type: frontmatter.type ?? 'project',
    })
  }

  return files.sort((a, b) => b.mtime - a.mtime)  // 最新优先
}
```

**限制**：
- 最大 200 个记忆文件
- 只读取 `.md` 文件
- 解析前 30 行作为 frontmatter

### 相关性选择

```typescript
// src/memdir/findRelevantMemories.ts

export async function findRelevantMemories(
  query: string,
  memories: MemoryHeader[],
  options?: FindOptions
): Promise<MemoryHeader[]> {
  if (!memories.length) return []

  const manifest = formatMemoryManifest(memories)
  const systemPrompt = SELECT_MEMORIES_SYSTEM_PROMPT

  // 使用 sideQuery + Sonnet 模型选择相关记忆
  const response = await sideQuery({
    systemPrompt,
    userPrompt: `Query: ${query}\n\nAvailable memories:\n${manifest}`,
    model: 'claude-sonnet-4-6',
    maxTokens: 500,
  })

  // 解析 JSON 响应，获取选中的记忆路径
  const selected = parseSelectionResponse(response)
  return memories.filter(m => selected.includes(m.path))
}

// 系统提示引导模型选择最多 5 个最相关的记忆
const SELECT_MEMORIES_SYSTEM_PROMPT = `
You are a memory selection assistant. Given a user query and a list of available memories,
select the most relevant memories (max 5) that would help answer the query.

Return your response as a JSON array of memory filenames.
Example: ["user_role.md", "project_status.md"]

Guidelines:
- Select memories that provide context about the user's role, preferences, or project state
- Avoid selecting memories that are not relevant to the current query
- Return [] if no memories seem relevant
`
```

### 年龄与新鲜度

```typescript
// src/memdir/memoryAge.ts

export function memoryAgeDays(mtime: number): number {
  const now = Date.now()
  const diff = now - mtime
  return Math.floor(diff / (1000 * 60 * 60 * 24))
}

export function memoryFreshnessText(mtime: number): string {
  const days = memoryAgeDays(mtime)
  if (days === 0) return 'from today'
  if (days === 1) return 'from yesterday'
  return `from ${days} days ago`
}

export function memoryFreshnessNote(mtime: number): string {
  const days = memoryAgeDays(mtime)
  if (days > 1) {
    return `<system-reminder>Memory ${memoryFreshnessText(mtime)} - may be stale</system-reminder>`
  }
  return ''
}
```

**新鲜度规则**：
- 超过 1 天的记忆显示陈旧警告
- 用于提醒 AI 记忆可能过时

### 注入系统提示

```typescript
// src/memdir/memdir.ts

export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000
export const MAX_MEMORY_CONTENT_LENGTH = 4000

export async function loadMemoryPrompt(): Promise<string | null> {
  const memoryDir = getAutoMemPath()
  if (!memoryDir) return null

  const entrypoint = join(memoryDir, ENTRYPOINT_NAME)
  if (!(await fileExists(entrypoint))) {
    return null
  }

  const content = await readFile(entrypoint, 'utf8')
  return truncateEntrypointContent(content)
}

function truncateEntrypointContent(content: string): string {
  const lines = content.split('\n')

  // 限制 200 行
  let truncated = lines.slice(0, MAX_ENTRYPOINT_LINES).join('\n')

  // 限制 25KB
  const bytes = Buffer.byteLength(truncated, 'utf8')
  if (bytes > MAX_ENTRYPOINT_BYTES) {
    let end = truncated.length
    while (Buffer.byteLength(truncated.slice(0, end), 'utf8') > MAX_ENTRYPOINT_BYTES) {
      end--
    }
    truncated = truncated.slice(0, end) + '\n... (truncated)'
  }

  return truncated
}
```

## CLAUDE.md 系统

### 文件发现层级

```typescript
// src/utils/claudemd.ts

export async function getClaudeMds(): Promise<ClaudeMdFile[]> {
  const files: ClaudeMdFile[] = []

  // 1. 用户级配置 (~/.claude/CLAUDE.md)
  const userClaudeMd = await loadUserClaudeMd()
  if (userClaudeMd) {
    files.push({ scope: 'user', priority: 0, ...userClaudeMd })
  }

  // 2. 项目配置（从当前目录向上查找）
  const projectRoot = await findProjectRoot()
  if (projectRoot) {
    const projectClaudeMd = await loadProjectClaudeMd(projectRoot)
    if (projectClaudeMd) {
      files.push({ scope: 'project', priority: 1, ...projectClaudeMd })
    }
  }

  // 3. 本地配置（当前工作目录）
  const localClaudeMd = await loadLocalClaudeMd()
  if (localClaudeMd) {
    files.push({ scope: 'local', priority: 2, ...localClaudeMd })
  }

  // 4. Flag 配置（通过 --claude-md 指定）
  const flagClaudeMd = await loadFlagClaudeMd()
  if (flagClaudeMd) {
    files.push({ scope: 'flag', priority: 3, ...flagClaudeMd })
  }

  // 按优先级排序（低数值 = 高优先级）
  return files.sort((a, b) => a.priority - b.priority)
}
```

### 嵌套加载与 @include

```typescript
// src/utils/claudemd.ts

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

### 支持的文件扩展名

```typescript
// src/utils/claudemd.ts

export const TEXT_FILE_EXTENSIONS = new Set([
  // 文档
  '.md', '.mdx', '.txt', '.rst',
  // Web
  '.js', '.ts', '.jsx', '.tsx', '.vue', '.svelte',
  '.html', '.htm', '.css', '.scss', '.sass', '.less',
  // 后端
  '.py', '.rb', '.go', '.rs', '.java', '.kt', '.scala',
  '.php', '.cs', '.fs', '.swift', '.m', '.mm',
  // 数据
  '.json', '.yaml', '.yml', '.toml', '.ini', '.cfg',
  '.sql', '.graphql', '.gql',
  // 配置
  '.dockerfile', '.dockerignore', '.gitignore',
  '.nginx', '.htaccess',
  // 脚本
  '.sh', '.bash', '.zsh', '.fish', '.ps1', '.bat',
  '.cmd', '.vbs', '.applescript',
  // 其他
  '.graphql', '.prisma', '.proto', '.thrift',
])
```

## 记忆附件系统

### 附件类型

```typescript
// src/utils/attachments.ts

export type AttachmentType =
  | 'nested_memory'      // CLAUDE.md 嵌套文件
  | 'relevant_memories'  // AI 选择的相关记忆
  | 'memory_file'        // 单个记忆文件

export interface MemoryAttachment {
  type: AttachmentType
  path: string
  content: string
  displayPath?: string
  mtimeMs?: number
}
```

### 预取机制

```typescript
// src/utils/attachments.ts

let relevantMemoriesPromise: Promise<MemoryAttachment[]> | null = null

export function startRelevantMemoryPrefetch(query: string): void {
  if (!isAutoMemoryEnabled()) return

  relevantMemoriesPromise = fetchRelevantMemories(query)
}

export async function getPrefetchedRelevantMemories(): Promise<MemoryAttachment[]> {
  if (!relevantMemoriesPromise) return []

  try {
    const memories = await relevantMemoriesPromise
    return memories.slice(0, 5)  // 最多 5 个
  } catch {
    return []
  } finally {
    relevantMemoriesPromise = null
  }
}

async function fetchRelevantMemories(query: string): Promise<MemoryAttachment[]> {
  const scanned = await scanMemoryFiles()
  const relevant = await findRelevantMemories(query, scanned)

  const attachments: MemoryAttachment[] = []
  let totalBytes = 0
  const MAX_SESSION_BYTES = 60 * 1024  // 60KB

  for (const memory of relevant) {
    const content = await readFileSafe(memory.path)
    if (!content) continue

    const bytes = Buffer.byteLength(content, 'utf8')
    if (totalBytes + bytes > MAX_SESSION_BYTES) break

    attachments.push({
      type: 'memory_file',
      path: memory.path,
      content: truncateMemoryContent(content),
      displayPath: relative(getAutoMemPath() ?? '', memory.path),
    })

    totalBytes += bytes
  }

  return attachments
}
```

### 上下文注入

```typescript
// src/context.ts

export const getUserContext = memoize(async () => {
  const context: Record<string, string> = {}

  // 获取项目 CLAUDE.md
  const claudeMdContent = await getClaudeMdContent()
  if (claudeMdContent) {
    context.claudeMd = claudeMdContent
  }

  // 获取记忆文件
  const memoryFiles = await getMemoryFiles()
  if (memoryFiles.length > 0) {
    context.memories = formatMemoryFiles(memoryFiles)
  }

  return context
})
```

## 记忆使用流程

### 完整流程图

```
用户输入
    │
    ▼
┌─────────────────┐
│  预取相关记忆   │◄─────────────────────────┐
│ (startPrefetch) │                          │
└─────────────────┘                          │
    │                                        │
    ▼                                        │
┌─────────────────┐     ┌──────────────┐    │
│ 扫描记忆文件    │────►│ 相关性选择   │────┤
│ (scanMemory)    │     │ (sideQuery)  │    │
└─────────────────┘     └──────────────┘    │
    │                                        │
    ▼                                        │
┌─────────────────┐                          │
│ 构建系统提示    │                          │
│ (getSystemPrompt)                         │
└─────────────────┘                          │
    │                                        │
    ▼                                        │
┌─────────────────┐     ┌──────────────┐     │
│ 注入 CLAUDE.md  │────►│ 注入记忆附件 │─────┘
│ (嵌套规则)      │     │ (相关记忆)   │
└─────────────────┘     └──────────────┘
    │
    ▼
┌─────────────────┐
│ 调用 Claude API │
│ (流式响应)      │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ 保存新记忆      │
│ (自动学习)      │
└─────────────────┘
```

### 记忆更新流程

```typescript
// src/memdir/PermissionUpdate.ts (简化)

export type PermissionUpdate =
  | { type: 'addMemory'; memory: MemoryContent }
  | { type: 'updateMemory'; path: string; content: string }
  | { type: 'deleteMemory'; path: string }

// 记忆持久化
export async function persistMemory(
  update: PermissionUpdate
): Promise<void> {
  const memoryDir = getAutoMemPath()
  if (!memoryDir) return

  switch (update.type) {
    case 'addMemory':
      await writeMemoryFile(memoryDir, update.memory)
      break
    case 'updateMemory':
      await updateMemoryFile(update.path, update.content)
      break
    case 'deleteMemory':
      await deleteMemoryFile(update.path)
      break
  }
}
```

## 配置与覆盖

### 环境变量

| 变量 | 作用 | 示例 |
|------|------|------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 禁用自动记忆 | `1` |
| `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` | 自定义记忆路径 | `/path/to/memory` |

### settings.json

```json
{
  "memory": {
    "enabled": true,
    "maxMemoriesPerSession": 5,
    "maxMemorySize": 4096,
    "autoSave": true
  }
}
```

## 限制与约束

### 硬性限制

| 项目 | 限制 |
|------|------|
| MEMORY.md 行数 | 200 行 |
| MEMORY.md 大小 | 25,000 bytes |
| 单个记忆文件 | 4,000 bytes |
| 每会话记忆数 | 5 个 |
| 每会话记忆总量 | 60,000 bytes |
| 扫描记忆文件数 | 200 个 |
| 扫描目录深度 | 无限（受限于 glob） |

### 不应保存的内容

```typescript
// src/memdir/memoryTypes.ts - WHAT_NOT_TO_SAVE_SECTION

/**
## What NOT to save in memory

- Code patterns, conventions, architecture — derive from current project state
- Git history, recent changes, who-changed-what — use git commands
- Debugging solutions or fix recipes — the fix is in the code
- Anything already documented in CLAUDE.md files
- Ephemeral task details: in-progress work, temporary state
*/
```

## 最佳实践

### 1. 记忆粒度

**推荐**：小而专注的记忆文件

```markdown
---
name: 测试策略
description: 用户对测试框架的偏好
type: feedback
---
用户偏好使用 Vitest 而不是 Jest。
用户要求集成测试必须连接真实数据库，而不是 mock。
```

**避免**：大而全的 monolithic 记忆

### 2. 命名规范

```
user/
├── role_backend_engineer.md
├── new_to_react.md
└── prefers_terse_responses.md

project/
├── auth_refactor_2024.md
├── api_v2_migration.md
└── migration_to_supabase.md

reference/
├── linear_ingest_project.md
└── grafana_latency_dashboard.md
```

### 3. 版本控制

- **项目记忆** (`project/`) - 建议提交到 git
- **用户记忆** (`user/`, `feedback/`) - 保留本地或同步到 dotfiles
- **引用记忆** (`reference/`) - 可选提交

### 4. 新鲜度管理

- 定期回顾记忆文件
- 删除过时记忆
- 更新变化的信息
- 关注系统提示中的陈旧警告

## 调试与诊断

### 查看加载的记忆

```bash
# 查看记忆目录
ls -la ~/.claude/projects/$(echo $PWD | tr '/' '_')/memory/

# 查看 MEMORY.md
bat ~/.claude/projects/*/memory/MEMORY.md
```

### 调试日志

```typescript
// 启用记忆系统调试
process.env.DEBUG = 'claude-code:memory:*'

// 或特定模块
process.env.DEBUG = 'claude-code:memdir,claude-code:attachments'
```

### 手动触发记忆刷新

```typescript
// 清除记忆缓存
import { getUserContext } from 'src/context'
getUserContext.cache.clear?.()

// 重新加载
await getUserContext()
```

## 与其他系统的关系

### 权限系统

- 记忆系统需要读取文件系统权限
- 记忆更新需要写入权限
- 路径受限于项目工作目录

### MCP 系统

- MCP 服务器配置可存储在 `reference/` 记忆中
- 外部工具发现信息可自动保存

### 上下文构建

- 记忆是系统提示的一部分
- 与 CLAUDE.md 文件合并后注入
- 受 token 预算限制

---

**注意**: 本文档基于 Claude Code CLI 源码分析，记忆系统实现可能随版本更新而变化。
