# Claude Code 工具系统深度分析报告

## 一、工具系统架构概述

Claude Code 的工具系统是一个高度模块化、可扩展的工具调用框架，采用 TypeScript 实现，基于 Zod schema 进行参数验证，支持并发执行和权限控制。

### 核心文件位置

- **工具接口定义**: `src/Tool.ts`
- **工具注册中心**: `src/tools.ts`
- **工具执行引擎**: `src/services/tools/toolExecution.ts`
- **工具调度器**: `src/services/tools/toolOrchestration.ts`
- **工具实现目录**: `src/tools/`

---

## 二、工具基础接口定义

### 2.1 Tool 接口核心结构

```typescript
type Tool<Input, Output, Progress> = {
  // 元数据
  name: string
  aliases?: string[]
  searchHint?: string
  maxResultSizeChars: number
  strict?: boolean
  shouldDefer?: boolean
  alwaysLoad?: boolean
  mcpInfo?: { serverName: string; toolName: string }
  
  // Schema 定义
  inputSchema: z.ZodType<Input>
  outputSchema?: z.ZodType<Output>
  inputJSONSchema?: ToolInputJSONSchema
  
  // 核心方法
  call(args, context, canUseTool, parentMessage, onProgress): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  prompt(options): Promise<string>
  
  // 权限与验证
  validateInput?(input, context): Promise<ValidationResult>
  checkPermissions(input, context): Promise<PermissionResult>
  
  // 并发与安全
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isEnabled(): boolean
  
  // UI 渲染
  userFacingName(input): string
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, progress, options): React.ReactNode
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
}
```

### 2.2 buildTool 工厂函数

`buildTool` 函数提供默认值填充，确保所有工具具有一致的接口：

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,  // 保守默认：不安全
  isReadOnly: () => false,         // 保守默认：写操作
  isDestructive: () => false,
  checkPermissions: (input) => Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',
  userFacingName: () => '',
}
```

### 2.3 ToolUseContext 上下文

工具执行时的上下文对象，包含所有运行时依赖：

```typescript
type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    agentDefinitions: AgentDefinitionsResult
    // ...
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  messages: Message[]
  // ...
}
```

---

## 三、工具注册与发现机制

### 3.1 工具注册

工具通过 `getAllBaseTools()` 函数统一注册：

```typescript
function getAllBaseTools(): Tools {
  return [
    AgentTool,
    BashTool,
    GlobTool,
    GrepTool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    WebSearchTool,
    SkillTool,
    // ... 更多工具
  ]
}
```

### 3.2 工具发现

通过 `findToolByName` 函数查找工具，支持别名匹配：

```typescript
function findToolByName(tools: Tools, name: string): Tool | undefined {
  return tools.find(t => toolMatchesName(t, name))
}

function toolMatchesName(tool: { name: string; aliases?: string[] }, name: string): boolean {
  return tool.name === name || (tool.aliases?.includes(name) ?? false)
}
```

### 3.3 工具过滤

根据权限上下文过滤可用工具：

```typescript
function filterToolsByDenyRules(tools, permissionContext): T[] {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
}
```

---

## 四、工具调用流程

### 4.1 调用链路

```
用户消息 → API 返回 tool_use → runTools() 
    → partitionToolCalls() [并发分区]
    → runToolsConcurrently() / runToolsSerially()
    → runToolUse() 
    → checkPermissionsAndCallTool()
    → tool.call()
    → mapToolResultToToolResultBlockParam()
```

### 4.2 并发控制

工具调用被分区为两类批次：

```typescript
function partitionToolCalls(toolUseMessages, context): Batch[] {
  // 将工具调用分区：
  // 1. 并发安全批次：多个连续的只读工具可并发执行
  // 2. 非安全批次：单个非只读工具串行执行
}
```

默认最大并发数：`CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (默认 10)

### 4.3 权限检查流程

```typescript
async function checkPermissionsAndCallTool(tool, toolUseID, input, context, ...) {
  // 1. Zod schema 验证
  const parsedInput = tool.inputSchema.safeParse(input)
  
  // 2. 工具自定义验证
  const validationResult = await tool.validateInput?.(parsedInput.data, context)
  
  // 3. 权限检查
  const permissionResult = await canUseTool(tool, parsedInput.data, context)
  
  // 4. 执行工具调用
  const result = await tool.call(parsedInput.data, context, canUseTool, parentMessage, onProgress)
  
  // 5. 结果映射
  return tool.mapToolResultToToolResultBlockParam(result.data, toolUseID)
}
```

---

## 五、内置工具完整列表

### 5.1 文件操作工具

| 工具名 | 功能 | 并发安全 | 只读 |
|--------|------|----------|------|
| **Read** | 读取文件内容，支持图片、PDF、Jupyter Notebook | ✅ | ✅ |
| **Edit** | 原地编辑文件（字符串替换） | ❌ | ❌ |
| **Write** | 创建或覆写文件 | ❌ | ❌ |
| **NotebookEdit** | 编辑 Jupyter Notebook 单元格 | ❌ | ❌ |

### 5.2 搜索工具

| 工具名 | 功能 | 并发安全 | 只读 |
|--------|------|----------|------|
| **Glob** | 文件名模式匹配搜索 | ✅ | ✅ |
| **Grep** | 文件内容正则搜索 | ✅ | ✅ |
| **ToolSearch** | 搜索和发现工具（延迟加载） | ✅ | ✅ |

### 5.3 Shell 执行工具

| 工具名 | 功能 | 并发安全 | 只读 |
|--------|------|----------|------|
| **Bash** | 执行 shell 命令 | 视命令而定 | 视命令而定 |
| **PowerShell** | 执行 PowerShell 命令 | 视命令而定 | 视命令而定 |

### 5.4 网络工具

| 工具名 | 功能 | 并发安全 | 只读 |
|--------|------|----------|------|
| **WebFetch** | 获取 URL 内容 | ✅ | ✅ |
| **WebSearch** | 网络搜索 | ✅ | ✅ |

### 5.5 代理与子任务工具

| 工具名 | 功能 | 并发安全 | 只读 |
|--------|------|----------|------|
| **Agent** | 启动子代理执行复杂任务 | ❌ | 视代理类型 |
| **TaskOutput** | 获取异步代理输出 | ✅ | ✅ |
| **TaskStop** | 停止运行中的代理 | ❌ | ❌ |

### 5.6 任务管理工具 (TodoV2)

| 工具名 | 功能 | 并发安全 | 只读 |
|--------|------|----------|------|
| **TaskCreate** | 创建新任务 | ❌ | ❌ |
| **TaskGet** | 获取任务详情 | ✅ | ✅ |
| **TaskUpdate** | 更新任务状态 | ❌ | ❌ |
| **TaskList** | 列出所有任务 | ✅ | ✅ |
| **TodoWrite** | 更新待办列表（旧版） | ❌ | ❌ |

### 5.7 调度工具

| 工具名 | 功能 | 并发安全 | 只读 |
|--------|------|----------|------|
| **CronCreate** | 创建定时任务 | ❌ | ❌ |
| **CronDelete** | 删除定时任务 | ❌ | ❌ |
| **CronList** | 列出定时任务 | ✅ | ✅ |
| **RemoteTrigger** | 触发远程代理 | ❌ | ❌ |

### 5.8 技能工具

| 工具名 | 功能 | 并发安全 | 只读 |
|--------|------|----------|------|
| **Skill** | 执行 slash-command 技能 | ❌ | 视技能 |

### 5.9 MCP 工具

| 工具名 | 功能 | 并发安全 | 只读 |
|--------|------|----------|------|
| **mcp__*__(tool)** | MCP 服务器提供的工具 | 视工具 | 视工具 |
| **ListMcpResources** | 列出 MCP 资源 | ✅ | ✅ |
| **ReadMcpResource** | 读取 MCP 资源 | ✅ | ✅ |

### 5.10 LSP 工具

| 工具名 | 功能 | 并发安全 | 只读 |
|--------|------|----------|------|
| **LSP** | 代码智能（定义跳转、引用查找等） | ✅ | ✅ |

### 5.11 其他工具

| 工具名 | 功能 | 并发安全 | 只读 |
|--------|------|----------|------|
| **AskUserQuestion** | 向用户提问 | ❌ | ✅ |
| **EnterPlanMode** | 进入计划模式 | ❌ | ✅ |
| **ExitPlanMode** | 退出计划模式 | ❌ | ✅ |
| **EnterWorktree** | 进入 git worktree | ❌ | ❌ |
| **ExitWorktree** | 退出 worktree | ❌ | ❌ |
| **Config** | 配置设置 | ❌ | ❌ |
| **Brief** | 发送消息 | ❌ | ❌ |
| **SendMessage** | 发送消息给队友 | ❌ | ❌ |
| **TeamCreate** | 创建团队 | ❌ | ❌ |
| **TeamDelete** | 删除团队 | ❌ | ❌ |

---

## 六、工具参数验证机制

### 6.1 Zod Schema 验证

每个工具使用 Zod 定义输入 schema：

```typescript
const inputSchema = lazySchema(() =>
  z.strictObject({
    pattern: z.string().describe('The regular expression pattern...'),
    path: z.string().optional().describe('File or directory to search in...'),
    output_mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
    // ...
  })
)
```

### 6.2 自定义验证

工具可实现 `validateInput` 方法进行额外验证：

```typescript
async validateInput({ path }, context): Promise<ValidationResult> {
  if (path) {
    const absolutePath = expandPath(path)
    if (!await fs.exists(absolutePath)) {
      return {
        result: false,
        message: `Path does not exist: ${path}`,
        errorCode: 1,
      }
    }
  }
  return { result: true }
}
```

### 6.3 验证结果类型

```typescript
type ValidationResult =
  | { result: true }
  | { result: false; message: string; errorCode: number; behavior?: 'ask' }
```

---

## 七、权限控制系统

### 7.1 权限检查流程

```
validateInput() → checkPermissions() → canUseTool() → 用户确认(如需要) → 执行
```

### 7.2 PermissionResult 类型

```typescript
type PermissionResult = 
  | { behavior: 'allow'; updatedInput: unknown }
  | { behavior: 'deny'; message: string }
  | { behavior: 'ask'; message: string; suggestions?: Suggestion[] }
  | { behavior: 'passthrough'; message: string }
```

### 7.3 权限规则来源

- `cliArg`: 命令行参数
- `session`: 会话级别规则
- `localSettings`: 本地设置
- `userSettings`: 用户设置
- `policySettings`: 策略设置

---

## 八、工具结果处理

### 8.1 结果映射

每个工具实现 `mapToolResultToToolResultBlockParam` 将结果转换为 API 格式：

```typescript
mapToolResultToToolResultBlockParam(output, toolUseID) {
  return {
    tool_use_id: toolUseID,
    type: 'tool_result',
    content: `Found ${output.numFiles} files...`,
  }
}
```

### 8.2 结果存储与大文件处理

工具结果超过 `maxResultSizeChars` 会被持久化到磁盘：

```typescript
// 默认限制
GrepTool: 20,000 chars
GlobTool: 100,000 chars
FileEditTool: 100,000 chars
```

### 8.3 上下文修改

工具可返回 `contextModifier` 修改后续工具的上下文：

```typescript
return {
  data: result,
  contextModifier: (ctx) => ({
    ...ctx,
    options: { ...ctx.options, mainLoopModel: newModel }
  })
}
```

---

## 九、特殊工具机制

### 9.1 延迟加载工具

`shouldDefer: true` 的工具在 ToolSearch 启用时不会立即发送 schema：

```typescript
ToolSearchTool: { shouldDefer: true }
LSPTool: { shouldDefer: true }
WebSearchTool: { shouldDefer: true }
```

### 9.2 透明包装器

`isTransparentWrapper()` 返回 true 的工具不显示自身 UI：

```typescript
// REPL 工具包装内部工具调用
isTransparentWrapper: () => true
```

### 9.3 子代理工具

Agent 工具支持多种内置代理类型：

- **Explore**: 只读探索代理
- **Plan**: 计划编写代理
- **Verification**: 验证代理
- **Fork**: 分叉代理（共享 prompt cache）

---

## 十、设计模式总结

### 10.1 工厂模式

`buildTool` 函数统一创建工具实例，提供默认值填充。

### 10.2 策略模式

每个工具实现自己的权限检查、验证和执行策略。

### 10.3 观察者模式

`onProgress` 回调实现进度报告，支持实时 UI 更新。

### 10.4 责任链模式

权限检查链：`validateInput → checkPermissions → canUseTool → hooks`

### 10.5 适配器模式

MCP 工具通过 `MCPTool` 基类适配到统一接口。

---

## 十一、关键设计决策

1. **保守默认策略**: `isConcurrencySafe` 和 `isReadOnly` 默认 false，确保安全
2. **Zod 严格模式**: 使用 `z.strictObject` 防止额外字段
3. **延迟 Schema 加载**: `lazySchema` 避免模块加载时的初始化开销
4. **文件状态追踪**: `readFileState` 缓存防止陈旧写入
5. **权限规则优先级**: CLI > session > local > user > policy
6. **并发分区执行**: 只读工具并发，写操作串行

---

此报告全面覆盖了 Claude Code 工具系统的核心架构、实现细节和设计决策，可作为深入理解和扩展工具系统的参考文档。
