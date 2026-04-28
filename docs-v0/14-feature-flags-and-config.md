# Claude Code CLI 功能开关与配置详解

## 概述

Claude Code CLI 采用了多层级的配置系统，包括：
1. **构建时功能开关** (Build-time Feature Flags) - 通过 `feature()` 函数控制
2. **环境变量** (Environment Variables) - 运行时配置
3. **动态配置** (Dynamic Config) - GrowthBook 远程配置
4. **用户设置** (Settings) - `settings.json` 文件配置
5. **Beta 功能头** (Beta Headers) - API 功能开关

## 1. 构建时功能开关 (Feature Flags)

### 1.1 核心架构

```
┌─────────────────────────────────────────────────────────────────┐
│                  构建时功能开关架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐      ┌─────────────────────────────────┐  │
│  │  bun:bundle     │─────►│  feature(name: string): boolean │  │
│  │  (构建时宏)      │      │                                 │  │
│  └─────────────────┘      │  • 构建时确定返回值              │  │
│                           │  • 死代码消除 (DCE)              │  │
│                           │  • 外部版本默认为 false          │  │
│                           └─────────────────────────────────┘  │
│                                                                 │
│  运行时 polyfill (cli.tsx):                                     │
│  const feature = (_name: string) => false;                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 功能开关分类

#### Anthropic 内部功能 (Ant-only)

| 功能开关 | 说明 | 模块 |
|---------|------|------|
| `TRANSCRIPT_CLASSIFIER` | 自动模式/分类器 | 权限系统 |
| `KAIROS` | AI 助手模式 | 助手系统 |
| `KAIROS_BRIEF` | Brief 工具支持 | 助手系统 |
| `KAIROS_CHANNELS` | 多频道支持 | 助手系统 |
| `PROACTIVE` | 主动式 AI |  proactive 模块 |
| `COORDINATOR_MODE` | 协调器模式 | 协调器 |
| `CACHED_MICROCOMPACT` | 缓存微压缩 | 压缩系统 |
| `REACTIVE_COMPACT` | 响应式压缩 | 压缩系统 |
| `HISTORY_SNIP` | 历史剪枝 | 压缩系统 |
| `CONTEXT_COLLAPSE` | 上下文折叠 | 上下文管理 |
| `BREAK_CACHE_COMMAND` | 缓存破坏调试 | 调试 |
| `ABLATION_BASELINE` | L0 消融基线 | 实验 |
| `CHICAGO_MCP` | Computer Use MCP | MCP |
| `AGENT_MEMORY_SNAPSHOT` | Agent 内存快照 | Agent |
| `AGENT_TRIGGERS` | Agent 触发器 | Agent |
| `FORK_SUBAGENT` | Fork 子 Agent | Agent |
| `UDS_INBOX` | Unix Domain Socket 收件箱 | 桥接 |
| `BRIDGE_MODE` | 桥接模式 | 桥接 |
| `DAEMON` | 守护进程模式 | 守护进程 |
| `BG_SESSIONS` | 后台会话 | 会话管理 |
| `CCR_MIRROR` | CCR 镜像 | CCR |
| `DIRECT_CONNECT` | 直连模式 | 连接 |
| `SSH_REMOTE` | SSH 远程 | 远程 |
| `LODESTONE` | Lodestone 功能 | 核心 |
| `WEB_BROWSER_TOOL` | Web 浏览器工具 | 工具 |
| `CONNECTOR_TEXT` | 连接器文本摘要 | API |
| `DUMP_SYSTEM_PROMPT` | 转储系统提示 | 调试 |
| `PERFETTO_TRACING` | Perfetto 追踪 | 遥测 |

#### 实验性功能

| 功能开关 | 说明 | 状态 |
|---------|------|------|
| `EXPERIMENTAL_SKILL_SEARCH` | 实验性技能搜索 | Beta |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | 内置探索计划 Agent | Beta |
| `VERIFICATION_AGENT` | 验证 Agent | Beta |
| `ULTRAPLAN` | 超级计划模式 | Alpha |
| `ULTRATHINK` | 深度思考模式 | Alpha |
| `TORCH` | Torch 模式 | Alpha |
| `POWERSHELL_AUTO_MODE` | PowerShell 自动模式 | Beta |
| `BASH_CLASSIFIER` | Bash 分类器 | Beta |

#### 平台特定功能

| 功能开关 | 说明 |
|---------|------|
| `IS_LIBC_GLIBC` | glibc 检测 |
| `IS_LIBC_MUSL` | musl 检测 |
| `NATIVE_CLIPBOARD_IMAGE` | 原生剪贴板图像 |
| `NATIVE_CLIENT_ATTESTATION` | 原生客户端证明 |

### 1.3 功能开关使用模式

```typescript
// 模式 1: 简单条件
if (feature('KAIROS')) {
  // Kairos 特定代码
}

// 模式 2: 条件导入 (死代码消除)
const proactiveModule = feature('PROACTIVE') || feature('KAIROS')
  ? require('./proactive/index.js')
  : null

// 模式 3: 动态模块加载
const autoModeStateModule = feature('TRANSCRIPT_CLASSIFIER')
  ? require('./utils/permissions/autoModeState.js')
  : null

// 模式 4: Schema 条件
const PermissionsSchema = z.object({
  ...(feature('TRANSCRIPT_CLASSIFIER')
    ? { disableAutoMode: z.enum(['disable']).optional() }
    : {}),
})
```

## 2. 环境变量

### 2.1 API 配置

| 环境变量 | 说明 | 默认值 |
|---------|------|--------|
| `ANTHROPIC_API_KEY` | Anthropic API 密钥 | - |
| `ANTHROPIC_AUTH_TOKEN` | Anthropic 认证令牌 | - |
| `ANTHROPIC_BASE_URL` | API 基础 URL | https://api.anthropic.com |
| `ANTHROPIC_MODEL` | 默认模型 | claude-sonnet-4-6 |
| `ANTHROPIC_SMALL_FAST_MODEL` | 小型快速模型 | claude-haiku-4-5-20251001 |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Opus 模型 | claude-opus-4-6 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Sonnet 模型 | claude-sonnet-4-6 |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Haiku 模型 | claude-haiku-4-5-20251001 |
| `ANTHROPIC_BETAS` | 额外 Beta headers | - |
| `ANTHROPIC_CUSTOM_HEADERS` | 自定义 HTTP 头 | - |

### 2.2 AWS Bedrock 配置

| 环境变量 | 说明 |
|---------|------|
| `CLAUDE_CODE_USE_BEDROCK` | 启用 Bedrock |
| `ANTHROPIC_BEDROCK_BASE_URL` | Bedrock 基础 URL |
| `AWS_DEFAULT_REGION` | AWS 区域 |
| `AWS_BEARER_TOKEN_BEDROCK` | Bedrock Bearer Token |
| `BEDROCK_BASE_URL` | Bedrock URL |

### 2.3 Google Vertex 配置

| 环境变量 | 说明 |
|---------|------|
| `CLAUDE_CODE_USE_VERTEX` | 启用 Vertex |
| `ANTHROPIC_VERTEX_PROJECT_ID` | Vertex 项目 ID |
| `VERTEX_BASE_URL` | Vertex 基础 URL |

### 2.4 Token 优化配置

| 环境变量 | 说明 | 默认值 |
|---------|------|--------|
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | 禁用 1M 上下文 | - |
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS` | 最大上下文 (Ant only) | - |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | 自动压缩窗口 | - |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 自动压缩百分比 | - |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | 阻塞限制覆盖 | - |
| `DISABLE_COMPACT` | 禁用压缩 | - |
| `DISABLE_AUTO_COMPACT` | 禁用自动压缩 | - |
| `DISABLE_PROMPT_CACHING` | 禁用提示缓存 | - |
| `CLAUDE_CODE_DISABLE_PRECOMPACT_SKIP` | 禁用预压缩跳过 | - |

### 2.5 内存系统配置

| 环境变量 | 说明 |
|---------|------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 禁用自动记忆 |
| `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` | 记忆路径覆盖 |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | 远程记忆目录 |

### 2.6 功能开关

| 环境变量 | 说明 | 默认值 |
|---------|------|--------|
| `CLAUDE_CODE_DISABLE_THINKING` | 禁用思考模式 | - |
| `CLAUDE_CODE_DISABLE_FAST_MODE` | 禁用快速模式 | - |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | 禁用后台任务 | - |
| `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` | 禁用文件检查点 | - |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | 禁用 CLAUDE.md | - |
| `CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS` | 禁用 Git 指令 | - |
| `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` | 禁用实验性 Beta | - |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 禁用自动记忆 | - |
| `CLAUDE_CODE_DISABLE_CRON` | 禁用定时任务 | - |
| `CLAUDE_CODE_ENABLE_TASKS` | 启用任务系统 | - |
| `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING` | 启用细粒度工具流 | - |
| `CLAUDE_CODE_ENABLE_TOKEN_USAGE_ATTACHMENT` | 启用 Token 使用附件 | - |

### 2.7 调试配置

| 环境变量 | 说明 |
|---------|------|
| `CLAUDE_DEBUG` | 调试模式 |
| `DEBUG` | 调试命名空间 |
| `DEBUG_SDK` | SDK 调试 |
| `CLAUDE_CODE_DEBUG_LOGS_DIR` | 调试日志目录 |
| `CLAUDE_CODE_DEBUG_LOG_LEVEL` | 调试日志级别 |
| `CLAUDE_CODE_DEBUG_REPAINTS` | 调试重绘 |
| `CLAUDE_CODE_DIAGNOSTICS_FILE` | 诊断文件 |
| `CLAUDE_CODE_PROFILE_QUERY` | 查询性能分析 |
| `CLAUDE_CODE_PROFILE_STARTUP` | 启动性能分析 |
| `CLAUDE_CODE_PERFETTO_TRACE` | Perfetto 追踪 |

### 2.8 远程/桥接配置

| 环境变量 | 说明 |
|---------|------|
| `CLAUDE_CODE_REMOTE` | 远程模式 |
| `CLAUDE_CODE_IS_COWORK` | Cowork 模式 |
| `CLAUDE_CODE_BRIDGE_BASE_URL` | 桥接基础 URL |
| `CLAUDE_CODE_USE_CCR_V2` | 使用 CCR V2 |
| `CLAUDE_CODE_CCR_MIRROR` | CCR 镜像 |

### 2.9 OAuth 配置

| 环境变量 | 说明 |
|---------|------|
| `CLAUDE_CODE_OAUTH_TOKEN` | OAuth 令牌 |
| `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` | OAuth 刷新令牌 |
| `CLAUDE_CODE_OAUTH_CLIENT_ID` | OAuth 客户端 ID |
| `CLAUDE_CODE_OAUTH_SCOPES` | OAuth 范围 |
| `CLAUDE_LOCAL_OAUTH_API_BASE` | 本地 OAuth API 基础 |
| `USE_LOCAL_OAUTH` | 使用本地 OAuth |
| `USE_STAGING_OAUTH` | 使用 Staging OAuth |

### 2.10 MCP 配置

| 环境变量 | 说明 |
|---------|------|
| `CLAUDE_CODE_ENABLE_CFC` | 启用 CFC |
| `CLAUDE_CODE_MCP_INSTR_DELTA` | MCP 指令 Delta |
| `ENABLE_CLAUDEAI_MCP_SERVERS` | 启用 Claude AI MCP 服务器 |
| `MCP_TIMEOUT` | MCP 超时 |
| `MCP_TOOL_TIMEOUT` | MCP 工具超时 |
| `MCP_SERVER_CONNECTION_BATCH_SIZE` | 连接批大小 |
| `MAX_MCP_OUTPUT_TOKENS` | 最大 MCP 输出 Token |

### 2.11 Agent 配置

| 环境变量 | 说明 |
|---------|------|
| `CLAUDE_CODE_AGENT` | Agent 模式 |
| `CLAUDE_CODE_SUBAGENT_MODEL` | 子 Agent 模型 |
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` | 最大工具并发 |
| `CLAUDE_CODE_PLAN_V2_AGENT_COUNT` | Plan V2 Agent 数量 |
| `CLAUDE_CODE_PLAN_V2_EXPLORE_AGENT_COUNT` | 探索 Agent 数量 |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` | 禁用内置 Agent |

### 2.12 特殊模式

| 环境变量 | 说明 |
|---------|------|
| `CLAUDE_CODE_SIMPLE` | 简单模式 |
| `CLAUDE_CODE_BRIEF` | Brief 模式 |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | 强制计划模式 |
| `CLAUDE_CODE_UNDERCOVER` | 隐身模式 |
| `USER_TYPE` | 用户类型 (ant/external) |
| `CLAUDE_CODE_ABLATION_BASELINE` | 消融基线 |

### 2.13 内部覆盖 (Ant only)

| 环境变量 | 说明 |
|---------|------|
| `CLAUDE_INTERNAL_FC_OVERRIDES` | GrowthBook 功能覆盖 |
| `CLAUDE_CODE_ATTRIBUTION_HEADER` | 归因头 |
| `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT` | 始终启用 Effort |
| `CLAUDE_CODE_EFFORT_LEVEL` | Effort 级别 |

## 3. GrowthBook 动态配置

### 3.1 配置系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                   GrowthBook 配置系统                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────┐      ┌───────────────┐      ┌─────────────┐ │
│  │   远程评估    │─────►│   本地缓存    │─────►│   环境覆盖   │ │
│  │  (API 调用)   │      │  (磁盘缓存)    │      │ (Ant only)  │ │
│  └───────────────┘      └───────────────┘      └─────────────┘ │
│          │                       │                      │       │
│          ▼                       ▼                      ▼       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    功能值解析                            │   │
│  │  • getFeatureValue_CACHED_MAY_BE_STALE()                 │   │
│  │  • getFeatureValue_CACHED_WITH_REFRESH()                 │   │
│  │  • checkStatsigFeatureGate_CACHED_MAY_BE_STALE()         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 功能标志列表

| 功能标志 | 说明 | 类型 |
|---------|------|------|
| `tengu_auto_mode_config` | 自动模式配置 | JSON |
| `tengu_tool_pear` | 严格工具模式 | boolean |
| `tengu_fgts` | 细粒度工具流 | boolean |
| `tengu_satin_quoll` | 工具结果持久化阈值 | Map<string, number> |
| `tengu_hawthorn_window` | 消息预算限制 | number |
| `tengu_quartz_lantern` | 原子写入重试 | boolean |
| `tengu_hive_evidence` | 任务更新工具 | boolean |
| `tengu_plum_vx3` | WebSearch 使用 Haiku | boolean |
| `tengu_agent_list_attach` | Agent 列表附件 | boolean |
| `tengu_amber_stoat` | 内置 Agent 启用 | boolean |
| `tengu_cobalt_raccoon` | 响应式压缩 | boolean |
| `tengu_lodestone_enabled` | Lodestone 启用 | boolean |
| `tengu_coral_reef_sonnet` | Sonnet 1M 上下文 | boolean |
| `tengu_ccr_bridge` | CCR 桥接 | boolean |
| `tengu_ccr_mirror` | CCR 镜像 | boolean |
| `tengu_chrome_auto_enable` | Chrome 自动启用 | boolean |
| `tengu_immediate_model_command` | 立即模型命令 | boolean |
| `tengu_1p_event_batch_config` | 1P 事件批处理 | JSON |
| `tengu_bramble_lintel` | 远程 MCP 服务器 | JSON |
| `tengu_copper_panda` | MCP 工具权限 | boolean |
| `tengu_moth_copse` | 技能搜索 | boolean |
| `tengu_scratch` | 实验性功能 | boolean |
| `tengu_trace_lantern` | 追踪采样 | number |
| `tengu_kairos_cron` | Kairos 定时任务 | boolean |
| `tengu_kairos_cron_durable` | 持久化定时任务 | boolean |
| `tengu_session_memory` | 会话记忆 | boolean |
| `tengu_amber_prism` | 工具使用差异 | boolean |
| `tengu_slate_thimble` | 计划模式 Agent | boolean |
| `tengu_thinkback` | 思考回放 | boolean |
| `tengu_cobalt_harbor` | 自动模式模型 | string |
| `tengu_slim_subagent_claudemd` | 子 Agent CLAUDE.md | boolean |
| `tengu_harbor_permissions` | 港口权限 | boolean |
| `tengu_harbor` | 港口功能 | boolean |
| `tengu_collage_kaleidoscope` | 多 Agent | boolean |
| `tengu_lapis_finch` | 文件检查点 | boolean |
| `tengu_birch_trellis` | 市场源 | JSON |
| `tengu_marble_fox` | 探索计划 Agent | boolean |
| `tengu_marble_sandcastle` | 沙盒 Agent | boolean |
| `tengu_passport_quail` | 远程管理设置 | JSON |
| `tengu_pebble_leaf_prune` | 剪枝策略 | boolean |
| `tengu_strap_foyer` | 桥接模式 | boolean |
| `tengu_surreal_dali` | 艺术模式 | boolean |
| `tengu_turtle_carbon` | 碳追踪 | boolean |
| `tengu_copper_bridge` | 铜桥 | boolean |
| `tengu_chair_sermon` | 椅子布道 | boolean |
| `tengu_chomp_inflection` | 咀嚼拐点 | boolean |
| `tengu_coral_fern` | 珊瑚蕨 | boolean |
| `tengu_amber_flint` | 琥珀燧石 | boolean |
| `tengu_amber_quartz_disabled` | 禁用琥珀石英 | boolean |
| `tengu_amber_json_tools` | JSON 工具 | boolean |
| `tengu_slate_prism` | 石板棱镜 | boolean |

### 3.3 环境变量覆盖 (Ant only)

```typescript
// CLAUDE_INTERNAL_FC_OVERRIDES 用于测试
// 格式: JSON object
// 示例: CLAUDE_INTERNAL_FC_OVERRIDES='{"tengu_tool_pear": true}'

function getEnvOverrides(): Record<string, unknown> | null {
  if (process.env.USER_TYPE === 'ant') {
    const raw = process.env.CLAUDE_INTERNAL_FC_OVERRIDES
    if (raw) {
      return JSON.parse(raw)
    }
  }
  return null
}
```

## 4. Settings.json 配置

### 4.1 配置文件位置

| 级别 | 路径 |
|------|------|
| 用户级 | `~/.claude/settings.json` |
| 项目级 | `.claude/settings.json` (项目根目录) |
| 本地级 | `.claude/settings.json` (当前目录) |
| 托管级 | `/etc/claude/managed-settings.json` |

### 4.2 配置优先级

```
托管设置 (最高优先级)
    ↓
CLI 参数
    ↓
环境变量
    ↓
项目设置
    ↓
用户设置 (最低优先级)
```

### 4.3 配置项详解

#### 4.3.1 权限配置

```json
{
  "permissions": {
    "allow": [
      { "toolName": "BashTool", "ruleContent": "/safe/path/*" }
    ],
    "deny": [
      { "toolName": "FileWriteTool", "ruleContent": "*.env" }
    ],
    "ask": [
      { "toolName": "BashTool", "ruleContent": "rm -rf *" }
    ],
    "defaultMode": "acceptEdits",
    "additionalDirectories": ["/extra/project"]
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `allow` | PermissionRule[] | 始终允许的规则 |
| `deny` | PermissionRule[] | 始终拒绝的规则 |
| `ask` | PermissionRule[] | 始终询问的规则 |
| `defaultMode` | string | 默认权限模式 |
| `disableBypassPermissionsMode` | string | 禁用绕过权限模式 |
| `disableAutoMode` | string | 禁用自动模式 (需 TRANSCRIPT_CLASSIFIER) |
| `additionalDirectories` | string[] | 额外工作目录 |

#### 4.3.2 自动更新配置

```json
{
  "autoUpdaterStatus": "enabled",
  "autoUpdatePlugins": true
}
```

#### 4.3.3 插件市场配置

```json
{
  "marketplaces": {
    "official": {
      "source": { "type": "official" },
      "autoUpdate": true
    }
  },
  "extraMarketplaces": [
    {
      "source": { "type": "github", "repo": "org/repo" },
      "autoUpdate": false
    }
  ]
}
```

#### 4.3.4 MCP 配置

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem"],
      "env": { "KEY": "value" }
    }
  },
  "enableAllProjectMcpServers": true
}
```

#### 4.3.5 钩子配置

```json
{
  "hooks": {
    "bashCommand": [
      {
        "matcher": { "command": "git commit" },
        "commands": [{ "type": "shell", "command": "echo 'Committing...'" }]
      }
    ],
    "prompt": [
      {
        "matcher": { "promptRegex": ".*" },
        "commands": [{ "type": "http", "url": "https://api.example.com/hook" }]
      }
    ]
  }
}
```

#### 4.3.6 主题配置

```json
{
  "theme": "dark",
  "autoCompactEnabled": true
}
```

#### 4.3.7 环境变量配置

```json
{
  "env": {
    "MY_VAR": "value",
    "ANOTHER_VAR": "123"
  }
}
```

#### 4.3.8 实验性配置

```json
{
  "experiments": {
    "terminalTool USE BASH TOOL": "enabled"
  }
}
```

#### 4.3.9 企业策略配置

```json
{
  "policy": {
    "allowMcpServers": [
      { "serverName": "allowed-server" },
      { "serverCommand": ["npx", "-y", "@scope/server"] }
    ],
    "denyMcpServers": [
      { "serverName": "blocked-server" }
    ],
    "allowRemoteMcpServers": false
  },
  "strictPluginOnlyCustomization": {
    "lockedSurfaces": ["skills", "agents", "hooks"]
  }
}
```

## 5. Beta Headers

### 5.1 API Beta 功能

| Beta Header | 说明 | 状态 |
|------------|------|------|
| `claude-code-20250219` | Claude Code 基础功能 | GA |
| `interleaved-thinking-2025-05-14` | 交错思考 | GA |
| `context-1m-2025-08-07` | 1M 上下文 | Beta |
| `context-management-2025-06-27` | 上下文管理 | Beta |
| `structured-outputs-2025-12-15` | 结构化输出 | Beta |
| `web-search-2025-03-05` | Web 搜索 | Beta |
| `advanced-tool-use-2025-11-20` | 高级工具使用 (1P) | Beta |
| `tool-search-tool-2025-10-19` | 工具搜索 (3P) | Beta |
| `effort-2025-11-24` | Effort 控制 | Beta |
| `task-budgets-2026-03-13` | 任务预算 | Beta |
| `prompt-caching-scope-2026-01-05` | 提示缓存范围 | Beta |
| `fast-mode-2026-02-01` | 快速模式 | Beta |
| `redact-thinking-2026-02-12` | 思考内容脱敏 | Beta |
| `token-efficient-tools-2026-03-28` | Token 高效工具 | Beta |
| `advisor-tool-2026-03-01` | Advisor 工具 | Beta |
| `cli-internal-2026-02-09` | CLI 内部 (Ant) | Internal |
| `afk-mode-2026-01-31` | AFK 模式 (TC) | TC only |

### 5.2 Provider 特定限制

```typescript
// Bedrock 限制的 Betas
const BEDROCK_EXTRA_PARAMS_HEADERS = new Set([
  'interleaved-thinking-2025-05-14',
  'context-1m-2025-08-07',
  'tool-search-tool-2025-10-19',
])

// Vertex 允许的 Betas
const VERTEX_COUNT_TOKENS_ALLOWED_BETAS = new Set([
  'claude-code-20250219',
  'interleaved-thinking-2025-05-14',
  'context-management-2025-06-27',
])
```

## 6. 配置加载流程

```
启动流程
    │
    ▼
┌─────────────────┐
│ cli.tsx (polyfill) │
│ • feature() = false │
│ • 环境变量初筛     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ init.ts         │
│ • 配置系统初始化 │
│ • GrowthBook 连接│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ main.tsx        │
│ • 合并所有配置   │
│ • 功能开关检查   │
│ • 动态模块加载   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ QueryEngine     │
│ • 运行时配置     │
│ • 会话级覆盖     │
└─────────────────┘
```

## 7. 配置最佳实践

### 7.1 开发环境

```bash
# .envrc (direnv)
export CLAUDE_DEBUG=1
export DEBUG=claude-code:*
export CLAUDE_CODE_DISABLE_AUTO_COMPACT=1
export CLAUDE_CODE_MAX_CONTEXT_TOKENS=100000
```

### 7.2 CI/CD 环境

```bash
# 非交互式模式
export CI=1
export CLAUDE_CODE_SIMPLE=1
export CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY=1
export CLAUDE_CODE_DISABLE_MOUSE=1
```

### 7.3 远程/容器环境

```bash
export CLAUDE_CODE_REMOTE=true
export CLAUDE_CODE_DISABLE_TERMINAL_TITLE=1
export CLAUDE_CODE_NO_FLICKER=1
```

### 7.4 企业部署

```json
// managed-settings.json
{
  "policy": {
    "allowMcpServers": [
      { "serverName": "enterprise-server" }
    ]
  },
  "autoUpdaterStatus": "disabled",
  "strictPluginOnlyCustomization": {
    "lockedSurfaces": ["skills", "agents", "hooks", "mcp"]
  }
}
```

## 8. 故障排除

### 8.1 配置不生效

1. 检查配置优先级 (托管 > CLI > 环境变量 > 项目 > 用户)
2. 验证 JSON 格式
3. 检查 feature() 开关状态
4. 查看 `CLAUDE_DEBUG` 输出

### 8.2 GrowthBook 连接失败

1. 检查网络连接
2. 验证认证令牌
3. 使用 `CLAUDE_INTERNAL_FC_OVERRIDES` 本地覆盖 (Ant)
4. 查看缓存状态 `~/.claude/.growthbook-cache.json`

### 8.3 功能开关调试

```bash
# 查看所有生效配置
claude /config

# 查看状态
claude /status

# 启用调试日志
DEBUG=claude-code:* claude
```

## 9. 安全注意事项

1. **敏感信息**: 不要在 `settings.json` 中存储 API 密钥
2. **托管设置**: 企业部署使用 managed-settings.json 强制策略
3. **环境变量**: 使用 `.env` 文件并添加到 `.gitignore`
4. **权限规则**: 谨慎配置 `allow` 规则，避免过度授权

---

**注意**: 部分功能仅对 Anthropic 内部员工 (USER_TYPE=ant) 可用。外部版本的功能受 `feature()` 函数限制，默认返回 `false`。
