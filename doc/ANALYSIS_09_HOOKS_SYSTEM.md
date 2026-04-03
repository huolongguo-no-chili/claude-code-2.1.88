# Claude Code Hooks 钩子系统深度分析报告

## 1. 系统概述

Claude Code 的 Hooks 系统是一个强大的生命周期事件机制，允许用户在特定事件发生时执行自定义命令、LLM 提示、HTTP 请求或 Agent 验证。

### 1.1 架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Hook 配置来源                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  userSettings    │  projectSettings  │  localSettings  │  policySettings    │
│  (~/.claude/)    │  (.claude/)       │  (.claude/)     │  (managed)         │
├─────────────────────────────────────────────────────────────────────────────┤
│  pluginHooks     │  sessionHooks     │  builtinHook    │  frontmatterHooks  │
│  (插件注册)       │  (内存临时)        │  (内部回调)      │  (Agent/Skill)     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      hooksConfigSnapshot.ts                                  │
│                   (启动时捕获配置快照，支持热更新)                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        hooks.ts - 核心执行引擎                                │
├─────────────────────────────────────────────────────────────────────────────┤
│  getMatchingHooks()  →  匹配器过滤 + `if` 条件评估                            │
│  executeHooks()      →  并行执行钩子，聚合结果                                  │
│  execCommandHook()   →  Shell 命令执行 (spawnAsync)                           │
│  execPromptHook()    →  LLM 单轮验证 (haiku)                                  │
│  execAgentHook()     →  LLM 多轮 Agent 验证                        │
│  execHttpHook()      →  HTTP POST 请求 (带 SSRF 防护)                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        钩子输出处理                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  Exit Code 0   →  成功，stdout 可能注入到对话上下文                            │
│  Exit Code 2   →  阻塞，阻止操作继续执行                                       │
│  Exit Code N   →  非阻塞错误，仅显示给用户                                      │
│  JSON Output   →  结构化控制 (continue, decision, permissionDecision)         │
│  Async Output  →  后台执行，不阻塞主流程                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 钩子事件类型

### 2.1 完整事件列表

| 事件 | 触发时机 | Matcher 字段 | 可阻塞 |
|------|----------|-------------|--------|
| `PreToolUse` | 工具执行前 | `tool_name` | ✅ (exit 2) |
| `PostToolUse` | 工具执行后 | `tool_name` | ❌ |
| `PostToolUseFailure` | 工具执行失败后 | `tool_name` | ❌ |
| `PermissionDenied` | 自动模式拒绝工具调用后 | `tool_name` | ❌ |
| `PermissionRequest` | 权限对话框显示时 | `tool_name` | ✅ |
| `Notification` | 发送通知时 | `notification_type` | ❌ |
| `UserPromptSubmit` | 用户提交提示时 | 无 | ✅ (exit 2) |
| `SessionStart` | 新会话开始时 | `source` | ❌ |
| `SessionEnd` | 会话结束时 | `reason` | ❌ |
| `Stop` | Claude 结束响应前 | 无 | ✅ (exit 2) |
| `StopFailure` | API 错误导致停止时 | `error` | ❌ |
| `SubagentStart` | 子 Agent 启动时 | `agent_type` | ❌ |
| `SubagentStop` | 子 Agent 停止前 | `agent_type` | ✅ |
| `PreCompact` | 对话压缩前 | `trigger` | ✅ (exit 2) |
| `PostCompact` | 对话压缩后 | `trigger` | ❌ |
| `Setup` | 仓库初始化/维护时 | `trigger` | ❌ |
| `TeammateIdle` | Teammate 即将空闲时 | 无 | ✅ (exit 2) |
| `TaskCreated` | 任务创建时 | 无 | ✅ (exit 2) |
| `TaskCompleted` | 任务完成时 | 无 | ✅ (exit 2) |
| `Elicitation` | MCP 服务器请求用户输入时 | `mcp_server_name` | ✅ |
| `ElicitationResult` | 用户响应 MCP 询问后 | `mcp_server_name` | ✅ |
| `ConfigChange` | 配置文件变更时 | `source` | ✅ (exit 2) |
| `InstructionsLoaded` | 指令文件加载时 | `load_reason` | ❌ |
| `WorktreeCreate` | 创建工作树时 | 无 | ❌ |
| `WorktreeRemove` | 移除工作树时 | 无 | ❌ |
| `CwdChanged` | 工作目录变更后 | 无 | ❌ |
| `FileChanged` | 监视文件变更时 | 文件名 | ❌ |

### 2.2 事件元数据定义

```typescript
// hooksConfigManager.ts
export const getHookEventMetadata = memoize(
  function (toolNames: string[]): Record<HookEvent, HookEventMetadata> {
    return {
      PreToolUse: {
        summary: 'Before tool execution',
        description: 'Input to command is JSON of tool call arguments.\n...',
        matcherMetadata: { fieldToMatch: 'tool_name', values: toolNames },
      },
      // ... 其他事件
    }
  }
)
```

---

## 3. 钩子类型详解

### 3.1 Command Hook (Shell 命令)

```typescript
type BashCommandHook = {
  type: 'command'
  command: string           // Shell 命令
  if?: string              // 条件过滤，如 "Bash(git *)"
  shell?: 'bash' | 'powershell'
  timeout?: number         // 秒
  statusMessage?: string   // Spinner 显示文本
  once?: boolean          // 执行一次后移除
  async?: boolean         // 后台执行
  asyncRewake?: boolean   // 后台执行 + exit 2 唤醒模型
}
```

**执行流程**:
1. 替换 `${CLAUDE_PLUGIN_ROOT}` 和 `${CLAUDE_PLUGIN_DATA}` 变量
2. Windows 上自动为 `.sh` 文件添加 `bash` 前缀
3. 设置环境变量 (CLAUDE_PROJECT_DIR, CLAUDE_ENV_FILE 等)
4. 通过 stdin 传入 JSON 输入
5. 解析 stdout 首行检测 async 响应

### 3.2 Prompt Hook (LLM 单轮验证)

```typescript
type PromptHook = {
  type: 'prompt'
  prompt: string           // 提示词，支持 $ARGUMENTS 占位符
  if?: string
  timeout?: number
  model?: string          // 默认使用 Haiku
  statusMessage?: string
  once?: boolean
}
```

**执行流程**:
1. 替换 `$ARGUMENTS` 为钩子输入 JSON
2. 使用快速模型 (haiku) 进行验证
3. 期望返回 `{ "ok": true }` 或 `{ "ok": false, "reason": "..." }`

### 3.3 Agent Hook (多轮 Agent 验证)

```typescript
type AgentHook = {
  type: 'agent'
  prompt: string          // 验证任务描述
  if?: string
  timeout?: number        // 默认 60s
  model?: string
  statusMessage?: string
  once?: boolean
}
```

**执行流程**:
1. 创建独立的 Agent 会话
2. 提供工具访问能力 (最多 50 轮)
3. Agent 必须调用 `SyntheticOutputTool` 返回结果
4. 支持 `StructuredOutputEnforcement` 强制结构化输出

### 3.4 HTTP Hook

```typescript
type HttpHook = {
  type: 'http'
  url: string             // POST 目标 URL
  if?: string
  timeout?: number        // 默认 10 分钟
  headers?: Record<string, string>  // 支持 $VAR_NAME 插值
  allowedEnvVars?: string[]         // 允许插值的环境变量
  statusMessage?: string
  once?: boolean
}
```

**安全特性**:
- SSRF 防护: 阻止私有/链路本地地址
- URL 白名单: `allowedHttpHookUrls` 策略
- 环境变量插值: 仅允许 `allowedEnvVars` 中的变量
- CRLF 注入防护: 清理 header 值中的 `\r\n\x00`

### 3.5 Function Hook (内存回调)

```typescript
type FunctionHook = {
  type: 'function'
  id?: string
  timeout?: number
  callback: (messages: Message[], signal?: AbortSignal) => boolean | Promise<boolean>
  errorMessage: string
  statusMessage?: string
}
```

**特点**: 仅限会话作用域，无法持久化到 settings.json

### 3.6 Callback Hook (内部回调)

```typescript
type HookCallback = {
  type: 'callback'
  callback: (input: HookInput, toolUseID: string, signal?: AbortSignal, 
             index?: number, context?: HookCallbackContext) => Promise<HookJSONOutput>
  internal?: boolean  // 内部标记，跳过日志
  timeout?: number
  statusMessage?: string
}
```

---

## 4. 钩子配置系统

### 4.1 配置来源优先级

```
userSettings < projectSettings < localSettings < policySettings < pluginHooks < sessionHooks
```

### 4.2 配置快照机制

```typescript
// hooksConfigSnapshot.ts
let initialHooksConfig: HooksSettings | null = null

export function captureHooksConfigSnapshot(): void {
  initialHooksConfig = getHooksFromAllowedSources()
}

export function getHooksConfigFromSnapshot(): HooksSettings | null {
  if (initialHooksConfig === null) {
    captureHooksConfigSnapshot()
  }
  return initialHooksConfig
}
```

**策略控制**:
- `allowManagedHooksOnly: true` → 仅运行 managed hooks
- `disableAllHooks: true` (policySettings) → 禁用所有 hooks
- `disableAllHooks: true` (non-managed) → 仅禁用非 managed hooks
- `strictPluginOnlyCustomization` → 阻止 user/project/local hooks

### 4.3 Session Hooks 管理

```typescript
// sessionHooks.ts
export function addSessionHook(
  setAppState: (updater: (prev: AppState) => AppState) => void,
  sessionId: string,
  event: HookEvent,
  matcher: string,
  hook: HookCommand,
  onHookSuccess?: OnHookSuccess,
  skillRoot?: string,
): void

export function addFunctionHook(
  setAppState: ...,
  sessionId: string,
  event: HookEvent,
  matcher: string,
  callback: FunctionHookCallback,
  errorMessage: string,
  options?: { timeout?: number; id?: string }
): string
```

**使用场景**:
- Agent/Skill frontmatter hooks 注册
- `StructuredOutputEnforcement` 强制结构化输出
- 一次性钩子 (`once: true`) 自动清理

---

## 5. 钩子执行流程

### 5.1 核心执行函数

```typescript
// hooks.ts
async function* executeHooks({
  hookInput,
  toolUseID,
  matchQuery,
  signal,
  timeoutMs,
  toolUseContext,
  messages,
  forceSyncExecution,
  requestPrompt,
}): AsyncGenerator<AggregatedHookResult>
```

**执行步骤**:

1. **信任检查**: `shouldSkipHookDueToTrust()` - 所有 hooks 需要工作区信任
2. **匹配过滤**: `getMatchingHooks()` - 按 matcher 和 `if` 条件过滤
3. **并行执行**: 所有钩子并行运行，聚合结果
4. **结果处理**: 
   - 收集 `blockingError`
   - 合并 `additionalContext`
   - 处理 `permissionBehavior` (deny > ask > allow)
   - 调用 `onHookSuccess` 回调

### 5.2 匹配器逻辑

```typescript
// hooks.ts
function matchesPattern(matchQuery: string, matcher: string): boolean {
  if (!matcher || matcher === '*') return true
  
  // 简单字符串或管道分隔列表
  if (/^[a-zA-Z0-9_|]+$/.test(matcher)) {
    if (matcher.includes('|')) {
      return matcher.split('|').map(p => normalizeLegacyToolName(p.trim()))
                   .includes(matchQuery)
    }
    return matchQuery === normalizeLegacyToolName(matcher)
  }
  
  // 正则表达式
  try {
    const regex = new RegExp(matcher)
    return regex.test(matchQuery) || 
           getLegacyToolNames(matchQuery).some(n => regex.test(n))
  } catch { return false }
}
```

### 5.3 `if` 条件评估

```typescript
// hooks.ts
async function prepareIfConditionMatcher(
  hookInput: HookInput,
  tools: Tools | undefined,
): Promise<IfConditionMatcher | undefined> {
  // 仅适用于 PreToolUse / PostToolUse / PostToolUseFailure / PermissionRequest
  
  const toolName = normalizeLegacyToolName(hookInput.tool_name)
  const tool = tools && findToolByName(tools, hookInput.tool_name)
  const input = tool?.inputSchema.safeParse(hookInput.tool_input)
  const patternMatcher = input?.success && tool?.preparePermissionMatcher
    ? await tool.preparePermissionMatcher(input.data)
    : undefined

  return ifCondition => {
    const parsed = permissionRuleValueFromString(ifCondition)
    if (normalizeLegacyToolName(parsed.toolName) !== toolName) return false
    if (!parsed.ruleContent) return true
    return patternMatcher ? patternMatcher(parsed.ruleContent) : false
  }
}
```

---

## 6. 异步钩子系统

### 6.1 Async Hook Registry

```typescript
// AsyncHookRegistry.ts
const pendingHooks = new Map<string, PendingAsyncHook>()

export function registerPendingAsyncHook({
  processId,
  hookId,
  asyncResponse,
  hookName,
  hookEvent,
  command,
  shellCommand,
}): void {
  const timeout = asyncResponse.asyncTimeout || 15000
  pendingHooks.set(processId, {
    processId, hookId, hookName, hookEvent, command,
    startTime: Date.now(),
    timeout,
    responseAttachmentSent: false,
    shellCommand,
    stopProgressInterval: startHookProgressInterval({...}),
  })
}
```

### 6.2 异步检测协议

```typescript
// hooks.ts - execCommandHook 中
if (!initialResponseChecked) {
  const firstLine = firstLineOf(stdout).trim()
  if (!firstLine.includes('}')) return
  initialResponseChecked = true
  
  try {
    const parsed = jsonParse(firstLine)
    if (isAsyncHookJSONOutput(parsed) && !forceSyncExecution) {
      const backgrounded = executeInBackground({
        processId: `async_hook_${child.pid}`,
        hookId,
        shellCommand,
        asyncResponse: parsed,
        ...
      })
      if (backgrounded) {
        asyncResolve?.({ stdout, stderr, output, status: 0 })
      }
    }
  } catch { /* ... */ }
}
```

### 6.3 Async Rewake 模式

```typescript
// hooks.ts - executeInBackground
if (asyncRewake) {
  // 完全绕过 registry
  void shellCommand.result.then(async result => {
    const stdout = await shellCommand.taskOutput.getStdout()
    const stderr = shellCommand.taskOutput.getStderr()
    shellCommand.cleanup()
    
    if (result.code === 2) {
      // Exit code 2: 唤醒模型
      enqueuePendingNotification({
        value: wrapInSystemReminder(
          `Stop hook blocking error from command "${hookName}": ${stderr || stdout}`
        ),
        mode: 'task-notification',
      })
    }
  })
  return true
}
```

---

## 7. 钩子输出处理

### 7.1 JSON 输出 Schema

```typescript
// types/hooks.ts
const hookJSONOutputSchema = z.object({
  continue: z.boolean().optional(),
  suppressOutput: z.boolean().optional(),
  stopReason: z.string().optional(),
  decision: z.enum(['approve', 'block']).optional(),
  reason: z.string().optional(),
  systemMessage: z.string().optional(),
  permissionDecision: z.enum(['allow', 'deny', 'ask']).optional(),
  hookSpecificOutput: z.object({
    hookEventName: z.string(),
    // 事件特定字段...
  }).optional(),
})
```

### 7.2 Exit Code 语义

| Exit Code | 含义 | 行为 |
|-----------|------|------|
| 0 | 成功 | stdout 可能注入到对话 |
| 2 | 阻塞 | 阻止操作，stderr 显示给模型 |
| 其他 | 非阻塞错误 | stderr 仅显示给用户 |

### 7.3 Permission Behavior 优先级

```typescript
// hooks.ts
switch (result.permissionBehavior) {
  case 'deny':
    permissionBehavior = 'deny'  // 最高优先级
    break
  case 'ask':
    if (permissionBehavior !== 'deny') {
      permissionBehavior = 'ask'
    }
    break
  case 'allow':
    if (!permissionBehavior) {
      permissionBehavior = 'allow'
    }
    break
}
```

---

## 8. 文件监视钩子

### 8.1 FileChanged Watcher

```typescript
// fileChangedWatcher.ts
let watcher: FSWatcher | null = null
let dynamicWatchPaths: string[] = []

export function initializeFileChangedWatcher(cwd: string): void {
  const config = getHooksConfigFromSnapshot()
  hasEnvHooks = (config?.CwdChanged?.length ?? 0) > 0 ||
                (config?.FileChanged?.length ?? 0) > 0
  
  const paths = resolveWatchPaths(config)
  if (paths.length === 0) return
  
  watcher = chokidar.watch(paths, {
    persistent: true,
    ignoreInitial: true,
    awaitWriteFinish: { stabilityThreshold: 500, pollInterval: 200 },
  })
  
  watcher.on('change', p => handleFileEvent(p, 'change'))
  watcher.on('add', p => handleFileEvent(p, 'add'))
  watcher.on('unlink', p => handleFileEvent(p, 'unlink'))
}
```

### 8.2 动态路径更新

```typescript
// 钩子可返回 watchPaths 更新监视列表
if (result.watchPaths && result.watchPaths.length > 0) {
  updateWatchPaths(result.watchPaths)
}
```

---

## 9. 事件广播系统

### 9.1 Hook Events

```typescript
// hookEvents.ts
export type HookExecutionEvent =
  | { type: 'started'; hookId: string; hookName: string; hookEvent: string }
  | { type: 'progress'; hookId: string; hookName: string; hookEvent: string;
      stdout: string; stderr: string; output: string }
  | { type: 'response'; hookId: string; hookName: string; hookEvent: string;
      output: string; stdout: string; stderr: string; exitCode?: number;
      outcome: 'success' | 'error' | 'cancelled' }

const pendingEvents: HookExecutionEvent[] = []
let eventHandler: HookEventHandler | null = null

export function registerHookEventHandler(handler: HookEventHandler | null): void {
  eventHandler = handler
  if (handler && pendingEvents.length > 0) {
    for (const event of pendingEvents.splice(0)) handler(event)
  }
}
```

### 9.2 SDK 集成

```typescript
// 仅 SessionStart 和 Setup 事件始终发送
// 其他事件需要 includeHookEvents 选项或 CLAUDE_CODE_REMOTE 模式
export function setAllHookEventsEnabled(enabled: boolean): void {
  allHookEventsEnabled = enabled
}
```

---

## 10. 安全机制

### 10.1 SSRF 防护

```typescript
// ssrfGuard.ts
export function isBlockedAddress(address: string): boolean {
  const v = isIP(address)
  if (v === 4) return isBlockedV4(address)
  if (v === 6) return isBlockedV6(address)
  return false
}

// 阻止的 IPv4 范围:
// - 0.0.0.0/8, 10.0.0.0/8, 100.64.0.0/10 (CGNAT)
// - 169.254.0.0/16 (云元数据)
// - 172.16.0.0/12, 192.168.0.0/16
// 允许: 127.0.0.0/8 (本地开发)
```

### 10.2 工作区信任

```typescript
// hooks.ts
export function shouldSkipHookDueToTrust(): boolean {
  const isInteractive = !getIsNonInteractiveSession()
  if (!isInteractive) return false  // SDK 模式隐式信任
  
  const hasTrust = checkHasTrustDialogAccepted()
  return !hasTrust  // 交互模式需要显式信任
}
```

### 10.3 环境变量隔离

```typescript
// execHttpHook.ts
function interpolateEnvVars(value: string, allowedEnvVars: ReadonlySet<string>): string {
  return value.replace(/\$\{([A-Z_][A-Z0-9_]*)\}|\$([A-Z_][A-Z0-9_]*)/g, 
    (_, braced, unbraced) => {
      const varName = braced ?? unbraced
      if (!allowedEnvVars.has(varName)) return ''  // 阻止未授权变量
      return process.env[varName] ?? ''
    }
  )
}
```

---

## 11. 总结

Claude Code 的 Hooks 系统是一个设计精良的生命周期事件框架：

1. **多类型支持**: Command / Prompt / Agent / HTTP / Function / Callback
2. **灵活配置**: 支持多来源、条件过滤、动态注册
3. **安全优先**: SSRF 防护、信任检查、环境变量隔离
4. **异步能力**: 后台执行、唤醒模型、进度报告
5. **可观测性**: 事件广播、遥测日志、调试输出

该系统为用户提供了强大的扩展能力，同时保持了安全性和可维护性。
