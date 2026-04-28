# MCP (Model Context Protocol) 系统详解

## MCP 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    MCP 系统架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   MCP 服务器                              │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│  │  │ VSCode  │ │ GitHub  │ │ Postgres│ │ Custom  │       │   │
│  │  │  Server │ │  Server │ │  Server │ │ Servers │       │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              services/mcp/client.ts                      │   │
│  │         ┌─────────────┐    ┌─────────────┐              │   │
│  │         │ 连接管理     │───→│ 工具发现    │              │   │
│  │         │ Connection  │    │ Discovery   │              │   │
│  │         └─────────────┘    └─────────────┘              │   │
│  │         ┌─────────────┐    ┌─────────────┐              │   │
│  │         │ 资源管理     │───→│ 请求路由    │              │   │
│  │         │ Resources   │    │ Routing     │              │   │
│  │         └─────────────┘    └─────────────┘              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              tools/MCPTool/                              │   │
│  │  ┌─────────────────┐    ┌─────────────────┐              │   │
│  │  │ ListMcpResources│    │ ReadMcpResource │              │   │
│  │  └─────────────────┘    └─────────────────┘              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## MCP 核心类型

### MCP 服务器配置

```typescript
// src/services/mcp/types.ts

export interface McpServerConfig {
  // 服务器名称
  name: string
  
  // 传输类型
  type: 'stdio' | 'sse' | 'websocket'
  
  // 命令配置（stdio）
  command?: string
  args?: string[]
  env?: Record<string, string>
  cwd?: string
  
  // URL 配置（sse/websocket）
  url?: string
  headers?: Record<string, string>
  
  // 认证
  auth?: {
    type: 'oauth' | 'api_key' | 'bearer'
    token?: string
    clientId?: string
    clientSecret?: string
  }
  
  // 权限
  permissions?: {
    allowedTools?: string[]
    blockedTools?: string[]
    allowedResources?: string[]
    blockedResources?: string[]
  }
  
  // 其他选项
  timeout?: number
  maxReconnects?: number
}

export interface MCPServerConnection {
  config: McpServerConfig
  client: MCPClient
  status: 'connecting' | 'connected' | 'error' | 'disconnected'
  tools: MCPTool[]
  resources: ServerResource[]
  lastError?: Error
}

export interface MCPTool {
  name: string
  description: string
  inputSchema: ToolInputSchema
  serverName: string
}

export interface ServerResource {
  uri: string
  name: string
  mimeType?: string
  description?: string
  serverName: string
}
```

## MCP 客户端管理

### MCP 客户端连接

```typescript
// src/services/mcp/client.ts

import { Client } from '@modelcontextprotocol/sdk/client/index.js'
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js'

const mcpConnections = new Map<string, MCPServerConnection>()

export async function connectMcpServer(
  config: McpServerConfig
): Promise<MCPServerConnection> {
  // 检查现有连接
  const existing = mcpConnections.get(config.name)
  if (existing?.status === 'connected') {
    return existing
  }
  
  // 创建客户端
  const client = new Client({
    name: 'claude-code',
    version: MACRO.VERSION,
  })
  
  // 创建传输层
  const transport = createTransport(config)
  
  // 建立连接
  try {
    await client.connect(transport)
    
    // 发现能力
    const capabilities = await client.getCapabilities()
    
    // 获取工具列表
    const tools = capabilities.tools?.tools ?? []
    
    // 获取资源列表
    const resources = capabilities.resources?.resources ?? []
    
    const connection: MCPServerConnection = {
      config,
      client,
      status: 'connected',
      tools: tools.map(t => ({ ...t, serverName: config.name })),
      resources: resources.map(r => ({ ...r, serverName: config.name })),
    }
    
    mcpConnections.set(config.name, connection)
    
    logEvent('tengu_mcp_server_connected', {
      server_name: config.name,
      tool_count: tools.length,
      resource_count: resources.length,
    })
    
    return connection
  } catch (error) {
    const failedConnection: MCPServerConnection = {
      config,
      client,
      status: 'error',
      tools: [],
      resources: [],
      lastError: error as Error,
    }
    
    mcpConnections.set(config.name, failedConnection)
    
    logEvent('tengu_mcp_server_error', {
      server_name: config.name,
      error: errorMessage(error),
    })
    
    throw error
  }
}

function createTransport(config: McpServerConfig) {
  switch (config.type) {
    case 'stdio':
      return new StdioClientTransport({
        command: config.command!,
        args: config.args ?? [],
        env: { ...process.env, ...config.env },
        cwd: config.cwd,
      })
      
    case 'sse':
      return new SSEClientTransport(new URL(config.url!), {
        headers: config.headers,
      })
      
    case 'websocket':
      return new WebSocketClientTransport(config.url!, {
        headers: config.headers,
      })
      
    default:
      throw new Error(`Unsupported transport type: ${config.type}`)
  }
}
```

### MCP 工具执行

```typescript
// src/services/mcp/client.ts

export async function callMcpTool(
  connection: MCPServerConnection,
  toolName: string,
  args: unknown
): Promise<McpToolResult> {
  const startTime = Date.now()
  
  try {
    const result = await connection.client.callTool({
      name: toolName,
      arguments: args as Record<string, unknown>,
    })
    
    const duration = Date.now() - startTime
    
    logEvent('tengu_mcp_tool_called', {
      server_name: connection.config.name,
      tool_name: toolName,
      duration_ms: duration,
    })
    
    return {
      content: result.content,
      isError: result.isError ?? false,
    }
  } catch (error) {
    // 检查是否是 Elicitation 错误
    if (isElicitationError(error)) {
      const elicitParams = extractElicitationParams(error)
      throw new MCPElicitationError(connection.config.name, elicitParams)
    }
    
    throw error
  }
}

export async function readMcpResource(
  connection: MCPServerConnection,
  uri: string
): Promise<McpResourceResult> {
  const result = await connection.client.readResource({ uri })
  
  return {
    contents: result.contents.map(content => ({
      uri: content.uri,
      mimeType: content.mimeType,
      text: content.text,
      blob: content.blob,
    })),
  }
}
```

## MCP 工具封装

### ListMcpResourcesTool

```typescript
// src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts

export const ListMcpResourcesTool = buildTool({
  name: LIST_MCP_RESOURCES_TOOL_NAME,
  searchHint: 'list available resources from MCP servers',
  
  async description() {
    return 'List available MCP resources'
  },
  
  get inputSchema() {
    return {
      type: 'object',
      properties: {
        server_name: {
          type: 'string',
          description: 'Filter by server name (optional)',
        },
        resource_type: {
          type: 'string',
          enum: ['file', 'api', 'database', 'documentation'],
          description: 'Filter by resource type (optional)',
        },
      },
    }
  },
  
  userFacingName: 'List MCP Resources',
  
  async call(input, context): Promise<ToolResult> {
    const { server_name, resource_type } = input as ListMcpResourcesInput
    const { mcpClients } = context.options
    
    // 过滤连接
    const connections = server_name
      ? mcpClients.filter(c => c.config.name === server_name)
      : mcpClients.filter(c => c.status === 'connected')
    
    if (connections.length === 0) {
      return {
        content: 'No MCP servers are currently connected.',
      }
    }
    
    // 收集资源
    const resources: ResourceEntry[] = []
    
    for (const connection of connections) {
      for (const resource of connection.resources) {
        if (resource_type && !matchesResourceType(resource, resource_type)) {
          continue
        }
        
        resources.push({
          server: connection.config.name,
          uri: resource.uri,
          name: resource.name,
          mimeType: resource.mimeType,
          description: resource.description,
        })
      }
    }
    
    // 格式化输出
    return {
      content: formatResourceList(resources),
    }
  },
})

function formatResourceList(resources: ResourceEntry[]): string {
  if (resources.length === 0) {
    return 'No resources found matching the criteria.'
  }
  
  const lines: string[] = [`Found ${resources.length} resources:`]
  
  for (const resource of resources) {
    lines.push(`\n**${resource.name}** (${resource.server})`)
    lines.push(`  URI: ${resource.uri}`)
    if (resource.mimeType) {
      lines.push(`  Type: ${resource.mimeType}`)
    }
    if (resource.description) {
      lines.push(`  Description: ${resource.description}`)
    }
  }
  
  return lines.join('\n')
}
```

### ReadMcpResourceTool

```typescript
// src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts

export const ReadMcpResourceTool = buildTool({
  name: READ_MCP_RESOURCE_TOOL_NAME,
  searchHint: 'read content from MCP resources',
  maxResultSizeChars: 100_000,
  isReadOnly: true,
  
  async description() {
    return 'Read content from an MCP resource'
  },
  
  get inputSchema() {
    return {
      type: 'object',
      properties: {
        uri: {
          type: 'string',
          description: 'The URI of the resource to read',
        },
        server_name: {
          type: 'string',
          description: 'The MCP server name (optional if URI is unique)',
        },
      },
      required: ['uri'],
    }
  },
  
  userFacingName: 'Read MCP Resource',
  
  async call(input, context): Promise<ToolResult> {
    const { uri, server_name } = input as ReadMcpResourceInput
    const { mcpClients } = context.options
    
    // 查找资源
    let connection: MCPServerConnection | undefined
    let resource: ServerResource | undefined
    
    if (server_name) {
      connection = mcpClients.find(c => c.config.name === server_name)
      resource = connection?.resources.find(r => r.uri === uri)
    } else {
      for (const c of mcpClients) {
        const r = c.resources.find(r => r.uri === uri)
        if (r) {
          connection = c
          resource = r
          break
        }
      }
    }
    
    if (!connection || !resource) {
      return {
        error: `Resource not found: ${uri}${server_name ? ` on server ${server_name}` : ''}`,
        isError: true,
      }
    }
    
    // 读取资源
    try {
      const result = await readMcpResource(connection, uri)
      
      // 格式化内容
      const contents = result.contents.map(c => {
        if (c.text) {
          return c.text
        }
        if (c.blob) {
          return `[Binary content: ${c.mimeType}]`
        }
        return '[Empty content]'
      })
      
      return {
        content: contents.join('\n\n---\n\n'),
      }
    } catch (error) {
      return {
        error: `Failed to read resource: ${errorMessage(error)}`,
        isError: true,
      }
    }
  },
})
```

## MCP 配置管理

### 配置加载

```typescript
// src/services/mcp/config.ts

export interface McpConfigFile {
  servers: Record<string, McpServerConfig>
}

export async function getClaudeCodeMcpConfigs(): Promise<McpServerConfig[]> {
  const configs: McpServerConfig[] = []
  
  // 1. 加载全局配置
  const globalConfig = await loadGlobalMcpConfig()
  configs.push(...globalConfig)
  
  // 2. 加载项目配置
  const projectConfig = await loadProjectMcpConfig()
  configs.push(...projectConfig)
  
  // 3. 加载环境变量配置
  const envConfig = loadEnvMcpConfig()
  configs.push(...envConfig)
  
  // 4. 去重（按名称）
  const seen = new Set<string>()
  return configs.filter(c => {
    if (seen.has(c.name)) return false
    seen.add(c.name)
    return true
  })
}

async function loadGlobalMcpConfig(): Promise<McpServerConfig[]> {
  const configPath = join(os.homedir(), '.claude', 'mcp.json')
  
  try {
    const content = await readFile(configPath, 'utf-8')
    const config = JSON.parse(content) as McpConfigFile
    return Object.entries(config.servers).map(([name, server]) => ({
      ...server,
      name,
    }))
  } catch {
    return []
  }
}

async function loadProjectMcpConfig(): Promise<McpServerConfig[]> {
  const configPath = join(process.cwd(), '.claude', 'mcp.json')
  
  try {
    const content = await readFile(configPath, 'utf-8')
    const config = JSON.parse(content) as McpConfigFile
    return Object.entries(config.servers).map(([name, server]) => ({
      ...server,
      name,
    }))
  } catch {
    return []
  }
}

function loadEnvMcpConfig(): McpServerConfig[] {
  const configs: McpServerConfig[] = []
  
  // 解析 CLAUDE_CODE_MCP_SERVERS 环境变量
  const envConfig = process.env.CLAUDE_CODE_MCP_SERVERS
  if (envConfig) {
    try {
      const parsed = JSON.parse(envConfig)
      configs.push(...Object.entries(parsed).map(([name, config]) => ({
        ...(config as McpServerConfig),
        name,
      })))
    } catch {
      // 忽略解析错误
    }
  }
  
  return configs
}
```

### MCP 工具集成

```typescript
// src/services/mcp/client.ts

export function getMcpTools(mcpClients: MCPServerConnection[]): Tools {
  const tools: Tool[] = []
  
  for (const connection of mcpClients) {
    if (connection.status !== 'connected') continue
    
    for (const mcpTool of connection.tools) {
      // 创建工具包装器
      const tool: Tool = buildTool({
        name: `${connection.config.name}_${mcpTool.name}`,
        searchHint: mcpTool.description,
        
        description: `MCP Tool from ${connection.config.name}: ${mcpTool.description}`,
        
        inputSchema: mcpTool.inputSchema,
        
        userFacingName: `${connection.config.name}: ${mcpTool.name}`,
        
        async call(input, context): Promise<ToolResult> {
          // 调用 MCP 工具
          const result = await callMcpTool(connection, mcpTool.name, input)
          
          return {
            content: result.content,
            isError: result.isError,
          }
        },
        
        // MCP 工具只读属性继承
        isReadOnly: isMcpToolReadOnly(mcpTool),
      })
      
      tools.push(tool)
    }
  }
  
  return tools
}
```

## OAuth 认证

### MCP OAuth 流程

```typescript
// src/services/mcp/oauth.ts

export async function initiateMcpOAuth(
  serverName: string,
  oauthConfig: OAuthConfig
): Promise<OAuthResult> {
  // 1. 生成 PKCE 参数
  const { codeVerifier, codeChallenge } = generatePKCE()
  
  // 2. 构建授权 URL
  const authUrl = new URL(oauthConfig.authorizationEndpoint)
  authUrl.searchParams.set('client_id', oauthConfig.clientId)
  authUrl.searchParams.set('redirect_uri', oauthConfig.redirectUri)
  authUrl.searchParams.set('response_type', 'code')
  authUrl.searchParams.set('code_challenge', codeChallenge)
  authUrl.searchParams.set('code_challenge_method', 'S256')
  authUrl.searchParams.set('state', generateState())
  authUrl.searchParams.set('scope', oauthConfig.scopes.join(' '))
  
  // 3. 打开浏览器
  await openBrowser(authUrl.toString())
  
  // 4. 启动本地回调服务器
  const authCode = await startCallbackServer(oauthConfig.redirectUri)
  
  // 5. 交换令牌
  const tokens = await exchangeCodeForTokens({
    tokenEndpoint: oauthConfig.tokenEndpoint,
    clientId: oauthConfig.clientId,
    clientSecret: oauthConfig.clientSecret,
    code: authCode,
    codeVerifier,
    redirectUri: oauthConfig.redirectUri,
  })
  
  // 6. 保存令牌
  await saveMcpTokens(serverName, tokens)
  
  return { success: true, tokens }
}

async function exchangeCodeForTokens(
  params: TokenExchangeParams
): Promise<OAuthTokens> {
  const response = await fetch(params.tokenEndpoint, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
    },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      client_id: params.clientId,
      client_secret: params.clientSecret ?? '',
      code: params.code,
      code_verifier: params.codeVerifier,
      redirect_uri: params.redirectUri,
    }),
  })
  
  if (!response.ok) {
    throw new Error(`Token exchange failed: ${response.status}`)
  }
  
  const data = await response.json()
  
  return {
    accessToken: data.access_token,
    refreshToken: data.refresh_token,
    expiresAt: data.expires_in 
      ? Date.now() + data.expires_in * 1000 
      : undefined,
  }
}
```

## Elicitation 流程

### URL Elicitation

```typescript
// src/services/mcp/elicitation.ts

export interface ElicitRequestURLParams {
  url: string
  method?: 'GET' | 'POST' | 'PUT' | 'DELETE'
  headers?: Record<string, string>
  body?: string
}

export interface ElicitResult {
  status: number
  headers: Record<string, string>
  body: string
}

export class MCPElicitationError extends Error {
  constructor(
    public readonly serverName: string,
    public readonly params: ElicitRequestURLParams
  ) {
    super(`MCP server ${serverName} requires user authorization`)
    this.name = 'MCPElicitationError'
  }
}

export async function handleElicitation(
  serverName: string,
  params: ElicitRequestURLParams,
  signal: AbortSignal
): Promise<ElicitResult> {
  // 显示授权对话框
  const result = await showElicitationDialog({
    serverName,
    url: params.url,
    method: params.method ?? 'GET',
  })
  
  if (!result.approved) {
    throw new Error('User denied authorization')
  }
  
  // 执行请求
  const response = await fetch(params.url, {
    method: params.method,
    headers: params.headers,
    body: params.body,
    signal,
  })
  
  return {
    status: response.status,
    headers: Object.fromEntries(response.headers.entries()),
    body: await response.text(),
  }
}
```

## 官方 MCP 注册表

### 预配置 MCP 服务器

```typescript
// src/services/mcp/officialRegistry.ts

export interface OfficialMCPServer {
  name: string
  displayName: string
  description: string
  category: 'version-control' | 'database' | 'cloud' | 'communication' | 'development'
  config: Omit<McpServerConfig, 'name'>
  oauth?: {
    authorizationEndpoint: string
    tokenEndpoint: string
    scopes: string[]
  }
}

export const OFFICIAL_MCP_SERVERS: OfficialMCPServer[] = [
  {
    name: 'github',
    displayName: 'GitHub',
    description: 'Access GitHub repositories, issues, and pull requests',
    category: 'version-control',
    config: {
      type: 'stdio',
      command: 'npx',
      args: ['-y', '@anthropic/mcp-github'],
      env: {
        GITHUB_TOKEN: '${GITHUB_TOKEN}',
      },
    },
    oauth: {
      authorizationEndpoint: 'https://github.com/login/oauth/authorize',
      tokenEndpoint: 'https://github.com/login/oauth/access_token',
      scopes: ['repo', 'read:user'],
    },
  },
  {
    name: 'vscode',
    displayName: 'VS Code',
    description: 'Read and edit files in VS Code',
    category: 'development',
    config: {
      type: 'stdio',
      command: 'code',
      args: ['--mcp-server'],
    },
  },
  {
    name: 'postgres',
    displayName: 'PostgreSQL',
    description: 'Query PostgreSQL databases',
    category: 'database',
    config: {
      type: 'stdio',
      command: 'npx',
      args: ['-y', '@anthropic/mcp-postgres'],
      env: {
        DATABASE_URL: '${DATABASE_URL}',
      },
    },
  },
]

export async function prefetchOfficialMcpUrls(): Promise<void> {
  // 预连接常用 MCP 服务器
  for (const server of OFFICIAL_MCP_SERVERS) {
    if (server.config.type === 'sse' && server.config.url) {
      // 预解析 DNS
      void prefetchDNS(server.config.url)
    }
  }
}
```
