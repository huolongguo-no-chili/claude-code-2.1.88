# Claude Code API 通信层深度分析报告

## 一、架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                      API 通信层架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   认证层      │    │   客户端层    │    │   重试层     │      │
│  │  auth.ts     │───▶│  client.ts   │───▶│  withRetry.ts│      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│         │                   │                   │               │
│         ▼                   ▼                   ▼               │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │  OAuth 流程   │    │ API 请求构建 │    │ 流式响应处理  │      │
│  │  oauth/client│    │  claude.ts   │    │  Stream<T>   │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│         │                   │                   │               │
│         ▼                   ▼                   ▼               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    模型管理层                              │  │
│  │  model.ts │ modelStrings.ts │ providers.ts │ betas.ts   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、认证机制

### 2.1 认证源优先级

```typescript
// 认证源优先级（从高到低）
1. --bare 模式: 仅 ANTHROPIC_API_KEY 或 apiKeyHelper
2. ANTHROPIC_AUTH_TOKEN 环境变量
3. CLAUDE_CODE_OAUTH_TOKEN 环境变量
4. CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR (文件描述符)
5. CCR_OAUTH_TOKEN_FILE (CCR 子进程磁盘回退)
6. apiKeyHelper (配置的辅助脚本)
7. claude.ai OAuth tokens (订阅用户)
8. ANTHROPIC_API_KEY (已批准的 API key)
9. macOS Keychain 或配置文件中的 key
```

### 2.2 认证类型判断

```typescript
// isAnthropicAuthEnabled() 决定是否启用 OAuth 认证
function isAnthropicAuthEnabled(): boolean {
  // --bare 模式: 仅 API key，无 OAuth
  if (isBareMode()) return false
  
  // SSH 远程会话: 通过 Unix socket 隧道
  if (process.env.ANTHROPIC_UNIX_SOCKET) {
    return !!process.env.CLAUDE_CODE_OAUTH_TOKEN
  }
  
  // 检查是否使用第三方服务
  const is3P = CLAUDE_CODE_USE_BEDROCK || CLAUDE_CODE_USE_VERTEX || CLAUDE_CODE_USE_FOUNDRY
  
  // 外部 API key 或 auth token 时禁用 OAuth
  return !is3P && !hasExternalAuth
}
```

### 2.3 OAuth Token 刷新机制

```typescript
// refreshOAuthToken - 自动刷新过期的 token
async function refreshOAuthToken(refreshToken: string): Promise<OAuthTokens> {
  const response = await axios.post(TOKEN_URL, {
    grant_type: 'refresh_token',
    refresh_token: refreshToken,
    client_id: CLIENT_ID,
    scope: CLAUDE_AI_OAUTH_SCOPES.join(' ')
  })
  
  return {
    accessToken: data.access_token,
    refreshToken: data.refresh_token ?? refreshToken,
    expiresAt: Date.now() + data.expires_in * 1000,
    scopes: parseScopes(data.scope)
  }
}
```

### 2.4 多 Provider 认证

| Provider | 认证方式 | 环境变量 |
|----------|---------|---------|
| **First Party** | API Key / OAuth | `ANTHROPIC_API_KEY` |
| **AWS Bedrock** | AWS Credentials / Bearer Token | `AWS_BEARER_TOKEN_BEDROCK` |
| **Google Vertex** | GCP Service Account / ADC | `GOOGLE_APPLICATION_CREDENTIALS` |
| **Azure Foundry** | API Key / Azure AD | `ANTHROPIC_FOUNDRY_API_KEY` |

---

## 三、API 客户端

### 3.1 客户端创建流程

```typescript
async function getAnthropicClient(options): Promise<Anthropic> {
  // 1. 构建默认 headers
  const defaultHeaders = {
    'x-app': 'cli',
    'User-Agent': getUserAgent(),
    'X-Claude-Code-Session-Id': getSessionId(),
    ...customHeaders,
    ...(containerId && { 'x-claude-remote-container-id': containerId }),
    ...(remoteSessionId && { 'x-claude-remote-session-id': remoteSessionId })
  }
  
  // 2. 刷新 OAuth token
  await checkAndRefreshOAuthTokenIfNeeded()
  
  // 3. 配置 API key headers (非订阅用户)
  if (!isClaudeAISubscriber()) {
    await configureApiKeyHeaders(defaultHeaders)
  }
  
  // 4. 根据 provider 创建对应客户端
  if (CLAUDE_CODE_USE_BEDROCK) return new AnthropicBedrock(...)
  if (CLAUDE_CODE_USE_FOUNDRY) return new AnthropicFoundry(...)
  if (CLAUDE_CODE_USE_VERTEX) return new AnthropicVertex(...)
  
  // 5. 第一方客户端
  return new Anthropic({
    apiKey: isClaudeAISubscriber() ? null : apiKey,
    authToken: isClaudeAISubscriber() ? oauthAccessToken : undefined,
    ...config
  })
}
```

### 3.2 请求超时配置

```typescript
const ARGS = {
  maxRetries,
  timeout: parseInt(process.env.API_TIMEOUT_MS || '600000'), // 默认 10 分钟
  dangerouslyAllowBrowser: true,
  fetchOptions: getProxyFetchOptions({ forAnthropicAPI: true })
}
```

### 3.3 Client Request ID 注入

```typescript
// 为超时请求生成客户端请求 ID，便于服务端日志关联
function buildFetch(fetchOverride, source) {
  return (input, init) => {
    const headers = new Headers(init?.headers)
    if (injectClientRequestId && !headers.has(CLIENT_REQUEST_ID_HEADER)) {
      headers.set(CLIENT_REQUEST_ID_HEADER, randomUUID())
    }
    return inner(input, { ...init, headers })
  }
}
```

---

## 四、API 请求格式

### 4.1 请求参数构建

```typescript
// claude.ts - paramsFromContext()
function paramsFromContext(retryContext) {
  return {
    model: normalizeModelStringForAPI(options.model), // 移除 [1m] 后缀
    messages: addCacheBreakpoints(messagesForAPI, ...),
    system: buildSystemPromptBlocks(systemPrompt, enablePromptCaching, ...),
    tools: allTools,
    tool_choice: options.toolChoice,
    
    // Beta headers
    ...(useBetas && { betas: betasParams }),
    
    // 元数据
    metadata: getAPIMetadata(),
    
    // 输出限制
    max_tokens: maxOutputTokens,
    
    // Thinking 配置
    thinking: thinkingConfig,
    
    // 温度 (thinking 禁用时才有效)
    ...(temperature !== undefined && { temperature }),
    
    // 上下文管理
    ...(contextManagement && { context_management }),
    
    // 额外参数
    ...extraBodyParams,
    
    // 输出配置
    ...(Object.keys(outputConfig).length > 0 && { output_config: outputConfig }),
    
    // Fast Mode
    ...(speed !== undefined && { speed })
  }
}
```

### 4.2 系统提示构建

```typescript
systemPrompt = asSystemPrompt([
  getAttributionHeader(fingerprint),
  getCLISyspromptPrefix({
    isNonInteractive: options.isNonInteractiveSession,
    hasAppendSystemPrompt: options.hasAppendSystemPrompt
  }),
  ...systemPrompt,
  ...(advisorModel ? [ADVISOR_TOOL_INSTRUCTIONS] : []),
  ...(injectChromeHere ? [CHROME_TOOL_SEARCH_INSTRUCTIONS] : [])
].filter(Boolean))
```

### 4.3 消息标准化流程

```typescript
// 消息标准化管道
messagesForAPI = normalizeMessagesForAPI(messages, filteredTools)

// 后处理步骤:
1. stripToolReferenceBlocksFromUserMessage(msg)  // 移除 tool_reference blocks
2. stripCallerFieldFromAssistantMessage(msg)      // 移除 caller 字段
3. ensureToolResultPairing(messagesForAPI)        // 修复 tool_use/tool_result 配对
4. stripAdvisorBlocks(messagesForAPI)             // 移除 advisor blocks (无 beta header 时)
5. stripExcessMediaItems(messagesForAPI, 100)     // 限制媒体数量 ≤ 100
```

### 4.4 Prompt Caching 配置

```typescript
function getCacheControl({ scope, querySource }) {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),
    ...(scope === 'global' && { scope })
  }
}

// 1h TTL 条件:
// 1. Bedrock: ENABLE_PROMPT_CACHING_1H_BEDROCK=true
// 2. 第一方: ant 用户 或 订阅用户(非 overage)
// 3. querySource 在 GrowthBook allowlist 中
```

---

## 五、流式响应处理

### 5.1 Stream 类实现

```typescript
class Stream<T> implements AsyncIterator<T> {
  private queue: T[] = []
  private readResolve?: (value: IteratorResult<T>) => void
  private isDone: boolean = false
  
  // 入队元素
  enqueue(value: T): void {
    if (this.readResolve) {
      this.readResolve({ done: false, value })
      this.readResolve = undefined
    } else {
      this.queue.push(value)
    }
  }
  
  // 完成流
  done() {
    this.isDone = true
    if (this.readResolve) {
      this.readResolve({ done: true, value: undefined })
    }
  }
  
  // 错误处理
  error(error: unknown) {
    if (this.readReject) {
      this.readReject(error)
    }
  }
}
```

### 5.2 流式事件处理

```typescript
// 事件类型处理
switch (part.type) {
  case 'message_start':
    partialMessage = part.message
    ttftMs = Date.now() - start
    usage = updateUsage(usage, part.message?.usage)
    break
    
  case 'content_block_start':
    // 初始化 content block (tool_use, text, thinking, server_tool_use)
    contentBlocks[part.index] = { ...part.content_block, input: '' }
    break
    
  case 'content_block_delta':
    // 累加 delta 内容
    switch (delta.type) {
      case 'text_delta':
        contentBlock.text += delta.text
        break
      case 'input_json_delta':
        contentBlock.input += delta.partial_json
        break
      case 'thinking_delta':
        contentBlock.thinking += delta.thinking
        break
    }
    break
    
  case 'content_block_stop':
    // 产出完整的 assistant message
    yield {
      type: 'assistant',
      message: { ...partialMessage, content: [contentBlock] }
    }
    break
    
  case 'message_delta':
    usage = updateUsage(usage, part.usage)
    stopReason = part.delta.stop_reason
    break
}
```

### 5.3 流式超时保护

```typescript
// Stream Idle Timeout Watchdog
const STREAM_IDLE_TIMEOUT_MS = 90_000  // 默认 90 秒

streamIdleTimer = setTimeout(() => {
  streamIdleAborted = true
  logEvent('tengu_streaming_idle_timeout', { model, request_id, timeout_ms })
  releaseStreamResources()  // 取消流
}, STREAM_IDLE_TIMEOUT_MS)

// 每收到一个 chunk 重置计时器
for await (const part of stream) {
  resetStreamIdleTimer()  // 重置超时
  // ... 处理事件
}
```

### 5.4 流式 → 非流式回退

```typescript
// 当流式失败时，回退到非流式请求
try {
  // 流式处理...
} catch (streamingError) {
  // 检查是否禁用回退
  const disableFallback = CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK ||
    getFeatureValue('tengu_disable_streaming_to_non_streaming_fallback')
  
  if (disableFallback) {
    throw streamingError
  }
  
  // 执行非流式回退
  didFallBackToNonStreaming = true
  const result = yield* executeNonStreamingRequest(...)
  yield { type: 'assistant', message: result }
}
```

---

## 六、模型管理

### 6.1 模型选择优先级

```typescript
function getMainLoopModel(): ModelName {
  // 1. 会话中的模型覆盖
  const modelOverride = getMainLoopModelOverride()
  if (modelOverride !== undefined) return parseUserSpecifiedModel(modelOverride)
  
  // 2. --model 标志
  // 3. ANTHROPIC_MODEL 环境变量
  // 4. 设置文件中的 model
  const specifiedModel = process.env.ANTHROPIC_MODEL || settings.model
  if (specifiedModel && isModelAllowed(specifiedModel)) {
    return parseUserSpecifiedModel(specifiedModel)
  }
  
  // 5. 内置默认值
  return getDefaultMainLoopModel()
}
```

### 6.2 默认模型逻辑

```typescript
function getDefaultMainLoopModelSetting(): ModelName | ModelAlias {
  // Ant 用户: 默认 Opus 1M
  if (process.env.USER_TYPE === 'ant') {
    return getAntModelOverrideConfig()?.defaultModel ?? 
           getDefaultOpusModel() + '[1m]'
  }
  
  // Max 和 Team Premium: 默认 Opus
  if (isMaxSubscriber() || isTeamPremiumSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }
  
  // PAYG, Enterprise, Team Standard, Pro: 默认 Sonnet
  return getDefaultSonnetModel()
}
```

### 6.3 模型字符串解析

```typescript
// 支持别名和 [1m] 后缀
function parseUserSpecifiedModel(modelInput: string): ModelName {
  const has1mTag = has1mContext(normalizedModel)
  const modelString = normalizedModel.replace(/\[1m]$/i, '').trim()
  
  // 别名解析
  switch (modelString) {
    case 'opusplan': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
    case 'sonnet': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
    case 'haiku': return getDefaultHaikuModel() + (has1mTag ? '[1m]' : '')
    case 'opus': return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
  }
  
  // 原始模型名保留 [1m] 后缀
  return modelInput.trim()
}
```

### 6.4 Provider 映射

```typescript
type APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry'

function getAPIProvider(): APIProvider {
  return CLAUDE_CODE_USE_BEDROCK ? 'bedrock' :
         CLAUDE_CODE_USE_VERTEX  ? 'vertex'  :
         CLAUDE_CODE_USE_FOUNDRY ? 'foundry' : 'firstParty'
}

// 模型 ID 映射示例
const ALL_MODEL_CONFIGS = {
  opus46: {
    firstParty: 'claude-opus-4-6-20250519',
    bedrock: 'anthropic.claude-opus-4-6-v1:0',
    vertex: 'claude-opus-4-6@20250519',
    foundry: 'claude-opus-4-6'
  },
  sonnet46: {
    firstParty: 'claude-sonnet-4-6-20250514',
    bedrock: 'anthropic.claude-sonnet-4-6-v1:0',
    // ...
  }
}
```

---

## 七、Beta Headers 管理

### 7.1 核心 Beta Headers

```typescript
// 常量定义
CONTEXT_1M_BETA_HEADER = 'context-1m-2025-04-01'
FAST_MODE_BETA_HEADER = 'fast-mode-2025-05-07'
EFFORT_BETA_HEADER = 'effort-2025-05-19'
STRUCTURED_OUTPUTS_BETA_HEADER = 'structured-outputs-2025-05-20'
TASK_BUDGETS_BETA_HEADER = 'task-budgets-2026-03-13'
TOOL_SEARCH_BETA_HEADER_1P = 'advanced-tool-use-2025-11-20'
TOOL_SEARCH_BETA_HEADER_3P = 'tool-search-tool-2025-10-19'
ADVISOR_BETA_HEADER = 'advisor-2026-03-01'
```

### 7.2 动态 Beta 管理

```typescript
// 会话级 latching - 防止中途切换导致 cache break
let fastModeHeaderLatched = getFastModeHeaderLatched()
if (!fastModeHeaderLatched && isFastMode) {
  fastModeHeaderLatched = true
  setFastModeHeaderLatched(true)  // 持久化到会话状态
}

// 条件性添加 beta headers
if (fastModeHeaderLatched && !betasParams.includes(FAST_MODE_BETA_HEADER)) {
  betasParams.push(FAST_MODE_BETA_HEADER)
}
```

### 7.3 模型能力检查

```typescript
// 结构化输出支持检查
function modelSupportsStructuredOutputs(model: string): boolean {
  const provider = getAPIProvider()
  if (provider !== 'firstParty' && provider !== 'foundry') return false
  
  const canonical = getCanonicalName(model)
  return canonical.includes('claude-sonnet-4-6') ||
         canonical.includes('claude-opus-4-6') ||
         // ...
}

// 上下文管理支持检查
function modelSupportsContextManagement(model: string): boolean {
  if (provider === 'foundry') return true
  if (provider === 'firstParty') return !canonical.includes('claude-3-')
  return canonical.includes('claude-opus-4') || canonical.includes('claude-sonnet-4')
}
```

---

## 八、错误处理与重试

### 8.1 重试策略

```typescript
async function* withRetry<T>(getClient, operation, options): AsyncGenerator {
  const maxRetries = getMaxRetries(options)  // 默认 10 次
  
  for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
    try {
      return await operation(client, attempt, retryContext)
    } catch (error) {
      // 判断是否应重试
      if (!shouldRetry(error)) {
        throw new CannotRetryError(error, retryContext)
      }
      
      // 计算延迟
      const delayMs = getRetryDelay(attempt, retryAfterHeader)
      
      // yield 系统 API 错误消息
      yield createSystemAPIErrorMessage(error, delayMs, attempt, maxRetries)
      
      await sleep(delayMs, signal)
    }
  }
}
```

### 8.2 可重试错误判断

```typescript
function shouldRetry(error: APIError): boolean {
  // Mock 错误不重试
  if (isMockRateLimitError(error)) return false
  
  // Persistent 模式: 429/529 始终重试
  if (isPersistentRetryEnabled() && isTransientCapacityError(error)) return true
  
  // CCR 模式: 401/403 临时错误重试
  if (CLAUDE_CODE_REMOTE && (error.status === 401 || error.status === 403)) return true
  
  // x-should-retry header 判断
  const shouldRetryHeader = error.headers?.get('x-should-retry')
  if (shouldRetryHeader === 'true' && (!isSubscriber || isEnterprise)) return true
  if (shouldRetryHeader === 'false' && !(ant && is5xxError)) return false
  
  // 状态码判断
  if (error.status === 408) return true   // Request Timeout
  if (error.status === 409) return true   // Conflict
  if (error.status === 429) return !isSubscriber || isEnterprise
  if (error.status === 401) { clearApiKeyHelperCache(); return true }
  if (error.status >= 500) return true    // Server errors
  
  return false
}
```

### 8.3 重试延迟计算

```typescript
function getRetryDelay(attempt: number, retryAfterHeader?: string, maxDelayMs = 32000): number {
  // 优先使用 server 指定的延迟
  if (retryAfterHeader) {
    const seconds = parseInt(retryAfterHeader, 10)
    if (!isNaN(seconds)) return seconds * 1000
  }
  
  // 指数退避 + 抖动
  const baseDelay = Math.min(500 * Math.pow(2, attempt - 1), maxDelayMs)
  const jitter = Math.random() * 0.25 * baseDelay
  return baseDelay + jitter
}
```

### 8.4 529 错误处理

```typescript
// 连续 529 错误计数
if (is529Error(error) && shouldFallback) {
  consecutive529Errors++
  if (consecutive529Errors >= MAX_529_RETRIES) {
    if (fallbackModel) {
      throw new FallbackTriggeredError(model, fallbackModel)
    }
    throw new CannotRetryError(new Error(REPEATED_529_ERROR_MESSAGE))
  }
}

// Fast mode 特殊处理
if (wasFastModeActive && is529Error(error)) {
  const retryAfterMs = getRetryAfterMs(error)
  if (retryAfterMs < SHORT_RETRY_THRESHOLD_MS) {
    await sleep(retryAfterMs)  // 短延迟保持 fast mode
    continue
  }
  triggerFastModeCooldown(Date.now() + cooldownMs, 'overloaded')
  retryContext.fastMode = false  // 切换到标准速度
}
```

### 8.5 错误分类

```typescript
// 错误消息常量
const API_ERROR_MESSAGE_PREFIX = 'API Error'
const INVALID_API_KEY_ERROR_MESSAGE = 'Not logged in · Please run /login'
const CREDIT_BALANCE_TOO_LOW_ERROR_MESSAGE = 'Credit balance is too low'
const ORG_DISABLED_ERROR_MESSAGE = 'Your ANTHROPIC_API_KEY belongs to a disabled organization'
const TOKEN_REVOKED_ERROR_MESSAGE = 'OAuth token revoked · Please run /login'
const REPEATED_529_ERROR_MESSAGE = 'Repeated 529 Overloaded errors'
const CUSTOM_OFF_SWITCH_MESSAGE = 'Opus is experiencing high load, please use /model to switch to Sonnet'

// 媒体大小错误检测
function isMediaSizeError(raw: string): boolean {
  return (raw.includes('image exceeds') && raw.includes('maximum')) ||
         /maximum of \d+ PDF pages/.test(raw)
}
```

---

## 九、日志与遥测

### 9.1 API 查询日志

```typescript
function logAPIQuery({
  model, messagesLength, temperature, betas,
  permissionMode, querySource, queryTracking,
  thinkingType, effortValue, fastMode, previousRequestId
}) {
  logEvent('tengu_api_query', {
    model,
    messagesLength,
    temperature,
    provider: getAPIProviderForStatsig(),
    buildAgeMins: getBuildAgeMinutes(),
    betas: betas?.join(','),
    permissionMode,
    querySource,
    queryChainId: queryTracking?.chainId,
    queryDepth: queryTracking?.depth,
    thinkingType,
    effortValue,
    fastMode,
    previousRequestId
  })
}
```

### 9.2 API 成功日志

```typescript
function logAPISuccessAndDuration({
  model, usage, durationMs, ttftMs, costUSD, ...
}) {
  logEvent('tengu_api_success', {
    model,
    inputTokens: usage.input_tokens,
    outputTokens: usage.output_tokens,
    cachedInputTokens: usage.cache_read_input_tokens,
    durationMs,
    ttftMs,
    costUSD,
    stop_reason,
    didFallBackToNonStreaming,
    gateway: detectGateway({ headers, baseUrl }),
    globalCacheStrategy,
    fastMode
  })
}
```

### 9.3 网关检测

```typescript
// 从响应 headers 检测 AI 网关
const GATEWAY_FINGERPRINTS = {
  litellm: { prefixes: ['x-litellm-'] },
  helicone: { prefixes: ['helicone-'] },
  portkey: { prefixes: ['x-portkey-'] },
  'cloudflare-ai-gateway': { prefixes: ['cf-aig-'] },
  kong: { prefixes: ['x-kong-'] },
  braintrust: { prefixes: ['x-bt-'] }
}

const GATEWAY_HOST_SUFFIXES = {
  databricks: ['.cloud.databricks.com', '.azuredatabricks.net']
}
```

---

## 十、关键特性总结

| 特性 | 实现 | 文件 |
|------|------|------|
| **多 Provider 支持** | 动态客户端创建 | `client.ts` |
| **OAuth 认证** | PKCE 流程 + 自动刷新 | `oauth/client.ts` |
| **流式响应** | AsyncGenerator + 事件驱动 | `claude.ts`, `stream.ts` |
| **智能重试** | 指数退避 + 条件判断 | `withRetry.ts` |
| **Prompt Caching** | 1h TTL + Global Cache | `claude.ts` |
| **模型别名** | 别名解析 + [1m] 后缀 | `model.ts` |
| **Beta 管理** | 会话级 latching | `betas.ts` |
| **错误分类** | 详细错误类型 + 用户友好消息 | `errors.ts` |
| **遥测** | 完整的请求/响应日志 | `logging.ts` |

---

## 十一、数据流总结

```
用户输入
    │
    ▼
┌─────────────────┐
│ 认证检查        │ ◀── auth.ts
│ - OAuth/API Key │
│ - Token 刷新    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 模型解析        │ ◀── model.ts
│ - 别名解析      │
│ - Provider 选择 │
│ - [1m] 后缀处理 │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 消息标准化      │ ◀── messages.ts
│ - 配对修复      │
│ - 媒体限制      │
│ - Cache 断点    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 请求构建        │ ◀── claude.ts
│ - Beta Headers  │
│ - System Prompt │
│ - Tools Schema  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 重试循环        │ ◀── withRetry.ts
│ - 错误分类      │
│ - 延迟计算      │
│ - 状态恢复      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 流式处理        │ ◀── Stream<T>
│ - 事件解析      │
│ - 内容累加      │
│ - 超时保护      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 响应产出        │
│ - Assistant Msg │
│ - Usage Stats   │
│ - Cost 计算     │
└─────────────────┘
```
