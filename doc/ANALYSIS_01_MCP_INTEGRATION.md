# Claude Code MCP (Model Context Protocol) 服务器集成深度分析报告

## 目录
1. [架构概述](#1-架构概述)
2. [MCP 协议实现](#2-mcp-协议实现)
3. [MCP 服务器生命周期](#3-mcp-服务器生命周期)
4. [MCP 工具集成](#4-mcp-工具集成)
5. [传输层实现](#5-传输层实现)
6. [认证与授权](#6-认证与授权)
7. [Elicitation 机制](#7-elicitation-机制)
8. [资源管理](#8-资源管理)

---

## 1. 架构概述

### 1.1 核心目录结构

```
src/
├── entrypoints/
│   └── mcp.ts                    # MCP 服务器模式入口
├── services/mcp/
│   ├── client.ts                 # MCP 客户端核心实现 (~3000行)
│   ├── config.ts                 # 配置管理 (~1200行)
│   ├── types.ts                  # 类型定义
│   ├── auth.ts                   # OAuth 认证 (~2200行)
│   ├── useManageMCPConnections.ts # React Hook 连接管理
│   ├── elicitationHandler.ts     # Elicitation 请求处理
│   ├── InProcessTransport.ts     # 进程内传输
│   ├── mcpStringUtils.ts         # 名称解析工具
│   ├── utils.ts                  # 通用工具函数
│   └── ...
├── tools/MCPTool/
│   └── MCPTool.ts               # MCP 工具包装器
└── utils/mcp/
    ├── elicitationValidation.ts  # 输入验证
    └── dateTimeParser.ts         # 日期时间解析
```

### 1.2 核心类型定义

```typescript
// MCP 服务器配置类型
type Transport = 'stdio' | 'sse' | 'sse-ide' | 'http' | 'ws' | 'sdk'

type McpServerConfig =
  | McpStdioServerConfig      // 本地进程通信
  | McpSSEServerConfig        // Server-Sent Events
  | McpHTTPServerConfig       // HTTP Streamable
  | McpWebSocketServerConfig  // WebSocket
  | McpSdkServerConfig        // SDK 内置
  | McpClaudeAIProxyServerConfig // claude.ai 代理

// 服务器连接状态
type MCPServerConnection =
  | ConnectedMCPServer   // 已连接
  | FailedMCPServer      // 连接失败
  | NeedsAuthMCPServer   // 需要认证
  | PendingMCPServer     // 等待中
  | DisabledMCPServer    // 已禁用
```

---

## 2. MCP 协议实现

### 2.1 协议层架构

Claude Code 实现了完整的 MCP 协议栈，基于 `@modelcontextprotocol/sdk`:

```
┌─────────────────────────────────────────────────────────┐
│                   Claude Code 应用层                     │
├─────────────────────────────────────────────────────────┤
│                   MCP Client (SDK)                      │
│  ┌─────────────┬──────────────┬──────────────────────┐ │
│  │   Tools     │  Resources   │     Prompts/Skills   │ │
│  └─────────────┴──────────────┴──────────────────────┘ │
├─────────────────────────────────────────────────────────┤
│                   Transport Layer                       │
│  ┌────────┬────────┬─────────┬───────────┬──────────┐ │
│  │ Stdio  │  SSE   │ HTTP    │ WebSocket │ SDK/IDE  │ │
│  └────────┴────────┴─────────┴───────────┴──────────┘ │
├─────────────────────────────────────────────────────────┤
│                   MCP Server (外部)                     │
└─────────────────────────────────────────────────────────┘
```

### 2.2 客户端能力声明

```typescript
const client = new Client(
  {
    name: 'claude-code',
    title: 'Claude Code',
    version: MACRO.VERSION,
    description: "Anthropic's agentic coding tool",
    websiteUrl: PRODUCT_URL,
  },
  {
    capabilities: {
      roots: {},           // 支持根目录列表
      elicitation: {},     // 支持 Elicitation 请求
    },
  }
)
```

### 2.3 JSON-RPC 消息处理

MCP 使用 JSON-RPC 2.0 协议进行通信：

```typescript
// 消息类型
type JSONRPCMessage = {
  jsonrpc: '2.0'
  method?: string
  params?: unknown
  id?: string | number
  result?: unknown
  error?: { code: number; message: string; data?: unknown }
}

// 请求处理器注册
client.setRequestHandler(ListRootsRequestSchema, async () => ({
  roots: [{ uri: `file://${getOriginalCwd()}` }]
}))
```

---

## 3. MCP 服务器生命周期

### 3.1 连接流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        服务器连接生命周期                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐      │
│  │ 加载配置     │───▶│ 检查禁用状态  │───▶│ 检查认证缓存     │      │
│  │ getAllMcp    │    │ isDisabled?  │    │ isMcpAuthCached? │      │
│  │ Configs()    │    └──────────────┘    └────────┬─────────┘      │
│  └──────────────┘                                  │                 │
│         │                                          ▼                 │
│         │                              ┌──────────────────┐         │
│         │                              │ 缓存命中？       │         │
│         │                              │ needs-auth      │         │
│         │                              └──────────────────┘         │
│         ▼                                                           │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐      │
│  │ 创建传输层   │───▶│ 连接服务器   │───▶│ 能力协商         │      │
│  │ Transport    │    │ client.connect│   │ getServerCapabi  │      │
│  └──────────────┘    └──────────────┘    │ lities()        │      │
│                                            └────────┬─────────┘      │
│                                                     │                │
│                                                     ▼                │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐      │
│  │ 更新 AppState│◀───│ 获取工具/资源│◀───│ 注册 Elicitation │      │
│  │ updateServer │    │ fetchTools   │    │ Handler          │      │
│  └──────────────┘    └──────────────┘    └──────────────────┘      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 连接实现核心代码

```typescript
export const connectToServer = memoize(
  async (name: string, serverRef: ScopedMcpServerConfig): Promise<MCPServerConnection> => {
    // 1. 根据配置类型创建对应的传输层
    let transport: Transport
    
    if (serverRef.type === 'sse') {
      transport = new SSEClientTransport(new URL(serverRef.url), {
        authProvider: new ClaudeAuthProvider(name, serverRef),
        fetch: wrapFetchWithTimeout(...)
      })
    } else if (serverRef.type === 'http') {
      transport = new StreamableHTTPClientTransport(new URL(serverRef.url), {
        authProvider: new ClaudeAuthProvider(name, serverRef),
        ...
      })
    } else if (serverRef.type === 'ws') {
      transport = new WebSocketTransport(await createNodeWsClient(...))
    } else {
      // stdio 默认
      transport = new StdioClientTransport({
        command: serverRef.command,
        args: serverRef.args,
        env: { ...subprocessEnv(), ...serverRef.env }
      })
    }
    
    // 2. 创建客户端并连接
    const client = new Client({...}, { capabilities: {...} })
    await client.connect(transport)
    
    // 3. 获取服务器能力
    const capabilities = client.getServerCapabilities()
    
    // 4. 注册 Elicitation 处理器
    client.setRequestHandler(ElicitRequestSchema, async request => {...})
    
    // 5. 设置错误和关闭处理器
    client.onerror = (error) => { /* 处理连接错误 */ }
    client.onclose = () => { 
      // 清除缓存，触发重连
      connectToServer.cache.delete(getServerCacheKey(name, serverRef))
    }
    
    return { name, client, type: 'connected', capabilities, ... }
  }
)
```

### 3.3 重连机制

```typescript
// 指数退避重连参数
const MAX_RECONNECT_ATTEMPTS = 5
const INITIAL_BACKOFF_MS = 1000
const MAX_BACKOFF_MS = 30000

// 重连逻辑
const reconnectWithBackoff = async () => {
  for (let attempt = 1; attempt <= MAX_RECONNECT_ATTEMPTS; attempt++) {
    updateServer({ ...client, type: 'pending', reconnectAttempt: attempt })
    
    try {
      const result = await reconnectMcpServerImpl(client.name, client.config)
      if (result.client.type === 'connected') {
        // 重连成功
        return
      }
    } catch (error) {
      // 计算退避时间
      const backoff = Math.min(
        INITIAL_BACKOFF_MS * Math.pow(2, attempt - 1),
        MAX_BACKOFF_MS
      )
      await sleep(backoff)
    }
  }
}
```

### 3.4 进程清理（stdio 服务器）

```typescript
const cleanup = async () => {
  if (serverRef.type === 'stdio') {
    const childPid = stdioTransport.pid
    
    // 1. 先发送 SIGINT (Ctrl+C)
    process.kill(childPid, 'SIGINT')
    await sleep(100)
    
    // 2. 如果进程还在，发送 SIGTERM
    if (processExists(childPid)) {
      process.kill(childPid, 'SIGTERM')
      await sleep(400)
    }
    
    // 3. 最后手段：SIGKILL
    if (processExists(childPid)) {
      process.kill(childPid, 'SIGKILL')
    }
  }
  
  await client.close()
}
```

---

## 4. MCP 工具集成

### 4.1 工具发现与注册

```typescript
export const fetchToolsForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<Tool[]> => {
    if (client.type !== 'connected') return []
    
    // 调用 tools/list 获取工具列表
    const result = await client.client.request(
      { method: 'tools/list' },
      ListToolsResultSchema
    )
    
    // 转换为内部 Tool 格式
    return result.tools.map(tool => {
      const fullyQualifiedName = buildMcpToolName(client.name, tool.name)
      
      return {
        ...MCPTool,  // 基础模板
        name: fullyQualifiedName,
        mcpInfo: { serverName: client.name, toolName: tool.name },
        isMcp: true,
        
        async call(args, context, ...) {
          const connectedClient = await ensureConnectedClient(client)
          const mcpResult = await callMCPToolWithUrlElicitationRetry({
            client: connectedClient,
            tool: tool.name,
            args,
            ...
          })
          return { data: mcpResult.content }
        },
        
        // 工具注解
        isConcurrencySafe: () => tool.annotations?.readOnlyHint ?? false,
        isReadOnly: () => tool.annotations?.readOnlyHint ?? false,
        isDestructive: () => tool.annotations?.destructiveHint ?? false,
        isOpenWorld: () => tool.annotations?.openWorldHint ?? false,
      }
    })
  }
)
```

### 4.2 工具命名规范

```typescript
// 格式: mcp__<normalized_server_name>__<normalized_tool_name>
// 示例: mcp__github__add_comment_to_issue

export function buildMcpToolName(serverName: string, toolName: string): string {
  return `mcp__${normalizeNameForMCP(serverName)}__${normalizeNameForMCP(toolName)}`
}

// 解析
export function mcpInfoFromString(toolString: string) {
  const parts = toolString.split('__')
  const [mcpPart, serverName, ...toolNameParts] = parts
  if (mcpPart !== 'mcp' || !serverName) return null
  return { serverName, toolName: toolNameParts.join('__') }
}
```

### 4.3 工具调用流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                         工具调用流程                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐      │
│  │ 模型请求工具 │───▶│ 查找工具定义 │───▶│ MCPTool.call()   │      │
│  │ tool_use     │    │ findToolByName│   │                  │      │
│  └──────────────┘    └──────────────┘    └────────┬─────────┘      │
│                                                     │                │
│                                                     ▼                │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐      │
│  │ 返回结果     │◀───│ 处理响应内容 │◀───│ client.request   │      │
│  │ tool_result  │    │ transformRes │    │ tools/call       │      │
│  └──────────────┘    └──────────────┘    └──────────────────┘      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.4 MCPTool 基础模板

```typescript
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',  // 被覆盖
  maxResultSizeChars: 100_000,
  
  // 允许任意输入（MCP 工具自定义 schema）
  inputSchema: z.object({}).passthrough(),
  
  // 输出为字符串
  outputSchema: z.string(),
  
  // 渲染方法
  renderToolUseMessage,
  renderToolUseProgressMessage,
  renderToolResultMessage,
  
  // 结果截断检测
  isResultTruncated(output) {
    return isOutputLineTruncated(output)
  },
})
```

---

## 5. 传输层实现

### 5.1 传输类型对比

| 传输类型 | 用途 | 特点 |
|---------|------|------|
| `stdio` | 本地 MCP 服务器 | 通过 stdin/stdout 通信，最常用 |
| `sse` | 远程 SSE 服务器 | Server-Sent Events，支持 OAuth |
| `http` | HTTP Streamable | MCP 2025-03-26 规范，支持双向 |
| `ws` | WebSocket | 实时双向通信 |
| `sse-ide` | IDE 集成 | VS Code 等 IDE 内置 |
| `ws-ide` | IDE WebSocket | IDE WebSocket 连接 |
| `sdk` | SDK 内置 | Agent SDK 集成 |
| `claudeai-proxy` | claude.ai 代理 | 云端 MCP 服务 |

### 5.2 WebSocket 传输实现

```typescript
export class WebSocketTransport implements Transport {
  private started = false
  private opened: Promise<void>
  private isBun = typeof Bun !== 'undefined'
  
  constructor(private ws: WebSocketLike) {
    this.opened = new Promise((resolve, reject) => {
      if (ws.readyState === WS_OPEN) {
        resolve()
      } else if (this.isBun) {
        // Bun 原生 WebSocket
        nws.addEventListener('open', () => resolve())
        nws.addEventListener('error', (e) => reject(e))
      } else {
        // Node ws 包
        nws.on('open', () => resolve())
        nws.on('error', (e) => reject(e))
      }
    })
  }
  
  async send(message: JSONRPCMessage): Promise<void> {
    if (this.ws.readyState !== WS_OPEN) {
      throw new Error('WebSocket is not open')
    }
    this.ws.send(jsonStringify(message))
  }
  
  async close(): Promise<void> {
    this.ws.close()
  }
}
```

### 5.3 进程内传输

```typescript
// 用于内置 MCP 服务器（Chrome MCP、Computer Use）
class InProcessTransport implements Transport {
  private peer: InProcessTransport | undefined
  
  async send(message: JSONRPCMessage): Promise<void> {
    // 异步投递到对端
    queueMicrotask(() => {
      this.peer?.onmessage?.(message)
    })
  }
  
  async close(): Promise<void> {
    this.onclose?.()
    // 同时关闭对端
    if (this.peer && !this.peer.closed) {
      this.peer.onclose?.()
    }
  }
}

// 创建双向传输对
export function createLinkedTransportPair(): [Transport, Transport] {
  const a = new InProcessTransport()
  const b = new InProcessTransport()
  a._setPeer(b)
  b._setPeer(a)
  return [a, b]  // [clientTransport, serverTransport]
}
```

### 5.4 请求超时处理

```typescript
// 每个 HTTP 请求使用新的超时信号，避免过期问题
export function wrapFetchWithTimeout(baseFetch: FetchLike): FetchLike {
  return async (url: string | URL, init?: RequestInit) => {
    // GET 请求不设置超时（长连接 SSE 流）
    if (method === 'GET') return baseFetch(url, init)
    
    const controller = new AbortController()
    const timer = setTimeout(() => controller.abort(), 60000)
    
    try {
      return await baseFetch(url, { ...init, signal: controller.signal })
    } finally {
      clearTimeout(timer)
    }
  }
}
```

---

## 6. 认证与授权

### 6.1 OAuth 流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                        OAuth 认证流程                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐      │
│  │ 发现认证服务器│───▶│ 动态客户端注册│───▶│ 用户授权        │      │
│  │ RFC 9728/8414│    │ DCR          │    │ 浏览器打开      │      │
│  └──────────────┘    └──────────────┘    └────────┬─────────┘      │
│                                                     │                │
│                                                     ▼                │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐      │
│  │ 存储令牌     │◀───│ 交换授权码   │◀───│ 回调接收 code   │      │
│  │ Keychain     │    │ tokens()     │    │ localhost:port   │      │
│  └──────────────┘    └──────────────┘    └──────────────────┘      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 认证提供者实现

```typescript
export class ClaudeAuthProvider implements OAuthClientProvider {
  async clientInformation(): Promise<OAuthClientInformation | undefined> {
    // 从安全存储读取已注册的客户端信息
    return getStoredClientInfo(this.serverName)
  }
  
  async saveClientInformation(info: OAuthClientInformation): Promise<void> {
    // 保存 DCR 客户端信息
    await saveClientInfo(this.serverName, info)
  }
  
  async tokens(): Promise<OAuthTokens | undefined> {
    // 从 Keychain 读取令牌
    return getStoredTokens(this.serverName)
  }
  
  async saveTokens(tokens: OAuthTokens): Promise<void> {
    // 安全存储令牌
    await saveTokensToKeychain(this.serverName, tokens)
  }
  
  async redirectUrl(): Promise<string | URL> {
    // 生成回调 URL
    const port = await findAvailablePort()
    return buildRedirectUri(port)
  }
}
```

### 6.3 认证缓存

```typescript
// 15 分钟 TTL，避免重复认证请求
const MCP_AUTH_CACHE_TTL_MS = 15 * 60 * 1000

async function isMcpAuthCached(serverId: string): Promise<boolean> {
  const cache = await getMcpAuthCache()
  const entry = cache[serverId]
  return entry && Date.now() - entry.timestamp < MCP_AUTH_CACHE_TTL_MS
}

// 401 错误时缓存
function handleRemoteAuthFailure(name: string, serverRef: ScopedMcpServerConfig): MCPServerConnection {
  setMcpAuthCacheEntry(name)
  return { name, type: 'needs-auth', config: serverRef }
}
```

### 6.4 claude.ai 代理认证

```typescript
export function createClaudeAiProxyFetch(innerFetch: FetchLike): FetchLike {
  return async (url, init) => {
    const tokens = getClaudeAIOAuthTokens()
    
    // 添加 Bearer 令牌
    headers.set('Authorization', `Bearer ${tokens.accessToken}`)
    const response = await innerFetch(url, { ...init, headers })
    
    // 401 时尝试刷新令牌
    if (response.status === 401) {
      const tokenChanged = await handleOAuth401Error(tokens.accessToken)
      if (tokenChanged) {
        // 用新令牌重试
        return doRequest()
      }
    }
    
    return response
  }
}
```

---

## 7. Elicitation 机制

### 7.1 概述

Elicitation 是 MCP 的服务器主动请求能力，允许 MCP 服务器向用户请求输入或确认。

### 7.2 请求类型

```typescript
type ElicitationMode = 'form' | 'url'

type ElicitRequestParams = 
  | {  // 表单模式
      mode: 'form'
      message: string
      requestedSchema: PrimitiveSchemaDefinition
    }
  | {  // URL 模式
      mode: 'url'
      message: string
      url: string
      elicitationId: string
    }
```

### 7.3 处理流程

```typescript
export function registerElicitationHandler(
  client: Client,
  serverName: string,
  setAppState: (f: (prev: AppState) => AppState) => void
): void {
  client.setRequestHandler(ElicitRequestSchema, async (request, extra) => {
    // 1. 先执行 Hooks（可能自动响应）
    const hookResponse = await runElicitationHooks(
      serverName, request.params, extra.signal
    )
    if (hookResponse) return hookResponse
    
    // 2. 添加到队列等待 UI 处理
    return new Promise(resolve => {
      setAppState(prev => ({
        ...prev,
        elicitation: {
          queue: [...prev.elicitation.queue, {
            serverName,
            requestId: extra.requestId,
            params: request.params,
            signal: extra.signal,
            respond: (result) => resolve(result)
          }]
        }
      }))
    })
  })
  
  // 注册完成通知处理器（URL 模式）
  client.setNotificationHandler(
    ElicitationCompleteNotificationSchema,
    notification => {
      setAppState(prev => {
        const idx = findElicitationInQueue(prev.elicitation.queue, ...)
        queue[idx].completed = true
        return { ...prev, elicitation: { queue } }
      })
    }
  )
}
```

### 7.4 输入验证

```typescript
export function validateElicitationInput(
  stringValue: string,
  schema: PrimitiveSchemaDefinition
): ValidationResult {
  const zodSchema = getZodSchema(schema)  // 根据 schema 生成 Zod 验证器
  const parseResult = zodSchema.safeParse(stringValue)
  
  if (parseResult.success) {
    return { value: parseResult.data, isValid: true }
  }
  return { isValid: false, error: parseResult.error.issues.map(e => e.message).join('; ') }
}

// 支持自然语言日期解析
export async function validateElicitationInputAsync(
  stringValue: string,
  schema: PrimitiveSchemaDefinition,
  signal: AbortSignal
): Promise<ValidationResult> {
  const syncResult = validateElicitationInput(stringValue, schema)
  if (syncResult.isValid) return syncResult
  
  // 尝试自然语言日期解析
  if (isDateTimeSchema(schema) && !looksLikeISO8601(stringValue)) {
    const parseResult = await parseNaturalLanguageDateTime(stringValue, schema.format, signal)
    if (parseResult.success) {
      return validateElicitationInput(parseResult.value, schema)
    }
  }
  
  return syncResult
}
```

---

## 8. 资源管理

### 8.1 资源发现

```typescript
export const fetchResourcesForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<ServerResource[]> => {
    if (!client.capabilities?.resources) return []
    
    const result = await client.client.request(
      { method: 'resources/list' },
      ListResourcesResultSchema
    )
    
    // 添加服务器名标识
    return result.resources.map(resource => ({
      ...resource,
      server: client.name
    }))
  }
)
```

### 8.2 资源订阅

```typescript
// 检查服务器是否支持资源订阅
const hasResourceSubscribe = !!capabilities?.resources?.subscribe

// 注册资源变更通知处理器
client.setNotificationHandler(
  ResourceListChangedNotificationSchema,
  async () => {
    const resources = await fetchResourcesForClient(client)
    updateServer({ ...client, resources })
  }
)
```

### 8.3 Prompts 和 Skills

```typescript
export const fetchCommandsForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<Command[]> => {
    if (!client.capabilities?.prompts) return []
    
    const result = await client.client.request(
      { method: 'prompts/list' },
      ListPromptsResultSchema
    )
    
    return result.prompts.map(prompt => ({
      type: 'prompt',
      name: `mcp__${normalizeNameForMCP(client.name)}__${prompt.name}`,
      isMcp: true,
      async getPromptForCommand(args) {
        const result = await client.getPrompt({
          name: prompt.name,
          arguments: zipObject(argNames, args.split(' '))
        })
        return transformResultContent(result.messages)
      }
    }))
  }
)
```

### 8.4 MCP Skills 发现

```typescript
// 从 skill:// 资源发现 MCP 技能
if (feature('MCP_SKILLS') && supportsResources) {
  const mcpSkills = await fetchMcpSkillsForClient(client)
  commands.push(...mcpSkills)
}
```

---

## 9. MCP 服务器模式入口

### 9.1 入口实现 (`entrypoints/mcp.ts`)

Claude Code 本身可以作为 MCP 服务器运行：

```typescript
export async function startMCPServer(cwd: string, debug: boolean, verbose: boolean): Promise<void> {
  setCwd(cwd)
  
  const server = new Server(
    { name: 'claude/tengu', version: MACRO.VERSION },
    { capabilities: { tools: {} } }
  )
  
  // 工具列表处理器
  server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: (await getTools(toolPermissionContext)).map(tool => ({
      name: tool.name,
      description: await tool.prompt(),
      inputSchema: zodToJsonSchema(tool.inputSchema),
      outputSchema: zodToJsonSchema(tool.outputSchema)
    }))
  }))
  
  // 工具调用处理器
  server.setRequestHandler(CallToolRequestSchema, async ({ params }) => {
    const tool = findToolByName(tools, params.name)
    const result = await tool.call(params.arguments, toolUseContext, ...)
    return { content: [{ type: 'text', text: result.data }] }
  })
  
  // 启动服务器
  const transport = new StdioServerTransport()
  await server.connect(transport)
}
```

---

## 10. 关键设计模式总结

### 10.1 连接管理
- **Memoization**: 连接结果缓存，避免重复连接
- **批量处理**: 区分本地/远程服务器，使用不同并发度
- **自动重连**: 指数退避重连机制

### 10.2 工具集成
- **模板模式**: MCPTool 作为基础模板，动态覆盖属性
- **命名空间**: `mcp__<server>__<tool>` 避免冲突
- **注解支持**: readOnlyHint、destructiveHint 等

### 10.3 认证架构
- **RFC 9728/8414**: 标准 OAuth 发现
- **DCR**: 动态客户端注册
- **安全存储**: macOS Keychain 集成

### 10.4 Elicitation
- **队列模式**: 请求入队，UI 异步处理
- **Hook 机制**: 支持自动化响应
- **表单/URL 双模式**: 灵活的用户交互

---

这份分析报告全面涵盖了 Claude Code 的 MCP 服务器集成实现，从协议层到应用层，从连接管理到资源处理，展现了其作为 MCP 客户端和服务器的完整能力。
