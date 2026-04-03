# Claude Code Agent 运行机制深度分析报告

## 一、整体架构概览

Claude Code 采用了**分层递归的 Agent 架构**，核心是一个**主循环**，通过工具系统扩展能力，支持**子代理**的启动和管理。

```
┌─────────────────────────────────────────────────────────────┐
│                      main.tsx (入口)                         │
│  - CLI 参数解析                                              │
│  - 初始化 (init, GrowthBook, 迁移)                           │
│  - 启动 REPL 或 Headless 模式                                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    query.ts (主循环)                         │
│  - 消息处理流                                                │
│  - 自动压缩 (autocompact)                                    │
│  - API 调用 & 流式响应                                       │
│  - 工具执行                                                  │
│  - Stop Hook 处理                                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              toolOrchestration.ts (工具编排)                 │
│  - 并发/串行执行决策                                         │
│  - 工具调用分发                                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              toolExecution.ts (工具执行)                     │
│  - 权限检查 (canUseTool)                                     │
│  - Hook 执行 (PreToolUse/PostToolUse)                        │
│  - 工具调用                                                  │
│  - 结果处理                                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、Agent 生命周期

### 2.1 入口流程

```typescript
// main.tsx 核心流程
async function main() {
  // 1. 安全设置
  process.env.NoDefaultCurrentDirectoryInExePath = '1'; // Windows PATH 劫持防护
  
  // 2. 早期预取 (MDM设置, Keychain)
  startMdmRawRead();
  startKeychainPrefetch();
  
  // 3. 确定客户端类型
  const clientType = /* cli/sdk/github-action/remote/... */;
  setClientType(clientType);
  
  // 4. 运行 CLI 命令处理器
  await run();
}
```

### 2.2 主循环状态机

```typescript
// 状态定义
type State = {
  messages: Message[]                    // 消息历史
  toolUseContext: ToolUseContext         // 工具使用上下文
  autoCompactTracking: AutoCompactTrackingState  // 自动压缩追踪
  maxOutputTokensRecoveryCount: number   // 输出token恢复计数
  hasAttemptedReactiveCompact: boolean   // 是否尝试过响应式压缩
  stopHookActive: boolean | undefined    // Stop Hook 是否激活
  turnCount: number                      // 回合数
  transition: Continue | undefined       // 上次迭代的原因
}
```

### 2.3 主循环流程

```typescript
async function* queryLoop(params: QueryParams) {
  // 初始化状态
  let state: State = { /* ... */ };
  
  while (true) {
    // 1. 消息预处理
    // - Snip Compact (历史裁剪)
    // - Microcompact (微压缩)
    // - Context Collapse (上下文折叠)
    // - Autocompact (自动压缩)
    
    // 2. 检查阻塞限制
    if (isAtBlockingLimit) {
      yield createAssistantAPIErrorMessage({ content: PROMPT_TOO_LONG_ERROR_MESSAGE });
      return { reason: 'blocking_limit' };
    }
    
    // 3. 调用模型 API
    for await (const message of deps.callModel({ /* ... */ })) {
      // 处理流式响应
      if (message.type === 'assistant') {
        assistantMessages.push(message);
        // 收集 tool_use 块
        if (msgToolUseBlocks.length > 0) {
          toolUseBlocks.push(...msgToolUseBlocks);
          needsFollowUp = true;
        }
      }
      yield message;
    }
    
    // 4. 如果不需要跟进 (无工具调用)，处理 Stop Hook 并结束
    if (!needsFollowUp) {
      const stopHookResult = yield* handleStopHooks(/* ... */);
      if (stopHookResult.preventContinuation) {
        return { reason: 'stop_hook_prevented' };
      }
      return { reason: 'completed' };
    }
    
    // 5. 执行工具
    const toolUpdates = streamingToolExecutor
      ? streamingToolExecutor.getRemainingResults()
      : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext);
    
    for await (const update of toolUpdates) {
      yield update.message;
    }
    
    // 6. 准备下一轮迭代
    state = {
      messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
      // ...
    };
  }
}
```

---

## 三、工具系统详解

### 3.1 Tool 接口定义

```typescript
type Tool<Input, Output, P extends ToolProgressData> = {
  name: string;
  aliases?: string[];
  inputSchema: Input;                    // Zod schema
  outputSchema?: z.ZodType<unknown>;
  
  // 核心方法
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>;
  checkPermissions(input, context): Promise<PermissionResult>;
  validateInput?(input, context): Promise<ValidationResult>;
  
  // 生命周期
  isEnabled(): boolean;
  isConcurrencySafe(input): boolean;     // 决定是否可并发执行
  isReadOnly(input): boolean;
  isDestructive?(input): boolean;
  
  // UI 渲染
  renderToolUseMessage(input, options): React.ReactNode;
  renderToolResultMessage?(content, progress, options): React.ReactNode;
  
  // 描述
  description(input, options): Promise<string>;
  prompt(options): Promise<string>;
  userFacingName(input): string;
}
```

### 3.2 工具执行流程

```typescript
async function* runToolUse(toolUse, assistantMessage, canUseTool, toolUseContext) {
  // 1. 查找工具
  let tool = findToolByName(toolUseContext.options.tools, toolUse.name);
  
  if (!tool) {
    yield { message: createUserMessage({ content: [/* error */] }) };
    return;
  }
  
  // 2. 解析输入
  const parsedInput = tool.inputSchema.safeParse(toolUse.input);
  if (!parsedInput.success) {
    yield { message: /* validation error */ };
    return;
  }
  
  // 3. 验证输入
  const isValidCall = await tool.validateInput?.(parsedInput.data, toolUseContext);
  if (isValidCall?.result === false) {
    yield { message: /* validation error */ };
    return;
  }
  
  // 4. 执行权限检查和调用
  for await (const update of streamedCheckPermissionsAndCallTool(/* ... */)) {
    yield update;
  }
}
```

### 3.3 权限检查与 Hook 执行

```typescript
async function checkPermissionsAndCallTool(/* ... */) {
  // 1. PreToolUse Hooks
  const hookResults = await runPreToolUseHooks(toolName, processedInput, toolUseContext);
  
  // 2. 权限检查
  const permissionResult = await canUseTool(tool, processedInput, toolUseContext);
  
  if (permissionResult.behavior === 'deny') {
    // 执行 PermissionDenied Hooks
    await executePermissionDeniedHooks(/* ... */);
    return [/* denied message */];
  }
  
  // 3. 调用工具
  const result = await tool.call(callInput, toolUseContext, canUseTool, assistantMessage, onProgress);
  
  // 4. PostToolUse Hooks
  await runPostToolUseHooks(toolName, processedInput, result, toolUseContext);
  
  return [/* result message */];
}
```

### 3.4 工具并发执行

```typescript
// 工具调用分区
function partitionToolCalls(toolUseMessages, toolUseContext): Batch[] {
  return toolUseMessages.reduce((acc, toolUse) => {
    const isConcurrencySafe = tool?.isConcurrencySafe(parsedInput.data) ?? false;
    
    // 连续的并发安全工具合并为一个批次
    if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
      acc[acc.length - 1].blocks.push(toolUse);
    } else {
      acc.push({ isConcurrencySafe, blocks: [toolUse] });
    }
    return acc;
  }, []);
}

// 执行策略
async function* runTools(toolUseMessages, /* ... */) {
  for (const { isConcurrencySafe, blocks } of partitionToolCalls(/* ... */)) {
    if (isConcurrencySafe) {
      // 并发执行
      yield* runToolsConcurrently(blocks, /* ... */);
    } else {
      // 串行执行
      yield* runToolsSerially(blocks, /* ... */);
    }
  }
}
```

---

## 四、子代理系统

### 4.1 Agent Tool 核心结构

```typescript
// AgentTool 输入 Schema
const inputSchema = z.object({
  description: z.string().describe('A short (3-5 word) description of the task'),
  prompt: z.string().describe('The task for the agent to perform'),
  subagent_type: z.string().optional().describe('The type of specialized agent'),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),
  
  // 多代理参数
  name: z.string().optional(),           // 用于 SendMessage 寻址
  team_name: z.string().optional(),      // 团队名
  mode: permissionModeSchema().optional(), // 权限模式
  
  // 隔离选项
  isolation: z.enum(['worktree', 'remote']).optional(),
  cwd: z.string().optional(),
});
```

### 4.2 子代理启动流程

```typescript
async function call({ prompt, subagent_type, description, /* ... */ }) {
  // 1. 解析代理类型
  const effectiveType = subagent_type ?? (isForkSubagentEnabled() ? undefined : 'general-purpose');
  const isForkPath = effectiveType === undefined;
  
  // 2. 检查 Fork 递归防护
  if (isForkPath && isInForkChild(toolUseContext.messages)) {
    throw new Error('Fork is not available inside a forked worker.');
  }
  
  // 3. 检查 MCP 服务器要求
  if (requiredMcpServers?.length) {
    // 等待连接完成并验证工具可用性
  }
  
  // 4. Fork 路径: 继承父级上下文
  if (isForkPath) {
    forkParentSystemPrompt = toolUseContext.renderedSystemPrompt;
    promptMessages = buildForkedMessages(directive, assistantMessage);
  }
  
  // 5. 同步/异步代理分发
  if (isAsync) {
    // 后台代理
    const agentTask = registerAsyncAgent({ agentId, description, prompt, /* ... */ });
    void runAsyncAgentLifecycle({ taskId, abortController, /* ... */ });
    return { data: { status: 'async_launched', agentId, outputFile } };
  }
  
  // 6. 同步代理
  const agentMessages = [];
  for await (const message of runAgent({ agentDefinition, promptMessages, /* ... */ })) {
    agentMessages.push(message);
    yield message;
  }
  
  return { data: finalizeAgentTool(agentMessages, agentId, metadata) };
}
```

### 4.3 runAgent 实现

```typescript
async function* runAgent({
  agentDefinition,
  promptMessages,
  toolUseContext,
  isAsync,
  // ...
}): AsyncGenerator<Message> {
  // 1. 创建代理 ID
  const agentId = override?.agentId ?? createAgentId();
  
  // 2. 初始化代理特定 MCP 服务器
  const { clients: mergedMcpClients, tools: agentMcpTools, cleanup: mcpCleanup } = 
    await initializeAgentMcpServers(agentDefinition, parentClients);
  
  // 3. 解析工具池
  const resolvedTools = useExactTools
    ? availableTools
    : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools;
  
  // 4. 构建系统提示
  const agentSystemPrompt = override?.systemPrompt ?? 
    asSystemPrompt(await getAgentSystemPrompt(agentDefinition, /* ... */));
  
  // 5. 注册 Frontmatter Hooks
  if (agentDefinition.hooks && hooksAllowedForThisAgent) {
    registerFrontmatterHooks(rootSetAppState, agentId, agentDefinition.hooks, /* ... */);
  }
  
  // 6. 预加载 Skills
  for (const skillName of skillsToPreload) {
    const content = await skill.getPromptForCommand('', toolUseContext);
    initialMessages.push(createUserMessage({ content: [/* skill content */] }));
  }
  
  // 7. 创建子代理上下文
  const agentToolUseContext = createSubagentContext(toolUseContext, {
    options: agentOptions,
    agentId,
    agentType: agentDefinition.agentType,
    messages: initialMessages,
    // ...
  });
  
  // 8. 记录 Transcript
  void recordSidechainTranscript(initialMessages, agentId);
  void writeAgentMetadata(agentId, { agentType, worktreePath, description });
  
  try {
    // 9. 执行主循环
    for await (const message of query({
      messages: initialMessages,
      systemPrompt: agentSystemPrompt,
      userContext: resolvedUserContext,
      systemContext: resolvedSystemContext,
      canUseTool,
      toolUseContext: agentToolUseContext,
      querySource,
      maxTurns: maxTurns ?? agentDefinition.maxTurns,
    })) {
      yield message;
    }
  } finally {
    // 10. 清理
    await mcpCleanup();
    clearSessionHooks(rootSetAppState, agentId);
    killShellTasksForAgent(agentId, /* ... */);
  }
}
```

### 4.4 Fork Subagent 机制

Fork 是一种特殊的子代理，继承父级的完整上下文：

```typescript
// Fork 子代理定义
export const FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],                          // 继承父级工具池
  maxTurns: 200,
  model: 'inherit',                      // 继承父级模型
  permissionMode: 'bubble',              // 权限提示冒泡到父终端
  source: 'built-in',
};

// 构建 Fork 消息
export function buildForkedMessages(directive, assistantMessage) {
  // 1. 克隆父级助手消息 (所有 tool_use 块)
  const fullAssistantMessage = { ...assistantMessage, uuid: randomUUID() };
  
  // 2. 构建占位符 tool_result
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result',
    tool_use_id: block.id,
    content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }],  // 相同占位符
  }));
  
  // 3. 添加指令
  const toolResultMessage = createUserMessage({
    content: [...toolResultBlocks, { type: 'text', text: buildChildMessage(directive) }],
  });
  
  return [fullAssistantMessage, toolResultMessage];
}
```

---

## 五、消息处理流程

### 5.1 消息类型

```typescript
type Message =
  | UserMessage           // 用户输入
  | AssistantMessage      // 助手响应
  | ProgressMessage       // 进度更新
  | SystemMessage         // 系统消息
  | AttachmentMessage     // 附件 (hook结果, memory等)
  | ToolUseSummaryMessage // 工具使用摘要
  | TombstoneMessage      // 墓碑 (已删除消息标记)
  | SystemCompactBoundaryMessage // 压缩边界
```

### 5.2 消息处理管道

```
用户输入
    │
    ▼
┌─────────────────┐
│ 挂起命令处理    │ ← getCommandsByMaxPriority()
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ 附件消息生成    │ ← getAttachmentMessages()
│ - Memory        │
│ - Skill Discovery│
│ - Hook结果      │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ 消息标准化      │ ← normalizeMessagesForAPI()
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ 工具结果预算    │ ← applyToolResultBudget()
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ 压缩处理        │
│ - Snip          │
│ - Microcompact  │
│ - Context Collapse│
│ - Autocompact   │
└─────────────────┘
    │
    ▼
API 请求
```

---

## 六、压缩机制

### 6.1 四级压缩策略

| 机制 | 触发条件 | 方式 | 特点 |
|------|----------|------|------|
| **Snip** | 历史过长 | 裁剪旧的助手消息 | 快速，低成本 |
| **Microcompact** | 缓存工具结果过大 | 按工具ID删除缓存 | 利用API的cache_deleted_input_tokens |
| **Context Collapse** | 上下文接近限制 | 提交折叠摘要 | 保留粒度信息 |
| **Autocompact** | 超过阈值 | 完整摘要 | 最后手段 |

### 6.2 响应式压缩

当遇到 413 (prompt-too-long) 错误时：

```typescript
// query.ts 中的处理
if (isWithheld413) {
  // 1. 尝试 Context Collapse 排空
  if (contextCollapse && state.transition?.reason !== 'collapse_drain_retry') {
    const drained = contextCollapse.recoverFromOverflow(messagesForQuery, querySource);
    if (drained.committed > 0) {
      state = { messages: drained.messages, transition: { reason: 'collapse_drain_retry' } };
      continue;  // 重试
    }
  }
  
  // 2. 尝试响应式压缩
  if (reactiveCompact) {
    const compacted = await reactiveCompact.tryReactiveCompact({ /* ... */ });
    if (compacted) {
      state = { messages: postCompactMessages, transition: { reason: 'reactive_compact_retry' } };
      continue;  // 重试
    }
  }
  
  // 3. 无法恢复，返回错误
  yield lastMessage;
  return { reason: 'prompt_too_long' };
}
```

---

## 七、多代理协作

### 7.1 Teammate 模式

```typescript
// Teammate 启动
if (teamName && name) {
  const result = await spawnTeammate({
    name,
    prompt,
    description,
    team_name: teamName,
    use_splitpane: true,
    plan_mode_required: spawnMode === 'plan',
    model,
    agent_type: subagent_type,
  });
  
  return { data: { status: 'teammate_spawned', teammate_id, agent_id, tmux_session_name, /* ... */ } };
}
```

### 7.2 隔离模式

- **worktree**: 创建临时 git worktree，代理在隔离的工作副本中工作
- **remote**: 启动远程 CCR 环境（仅限 ant 内部）

---

## 八、总结

Claude Code 的 Agent 系统是一个精心设计的多层架构：

1. **主循环** 负责消息流、压缩和工具执行的协调
2. **工具系统** 提供可扩展的能力，支持并发执行和权限控制
3. **子代理系统** 支持任务委托和并行处理
4. **压缩机制** 确保长对话的可持续性
5. **Hook 系统** 提供生命周期扩展点

关键设计特点：
- **递归架构**: 子代理通过相同的 `query()` 函数运行，形成自然的多层代理结构
- **流式处理**: 整个系统使用 AsyncGenerator 实现流式响应
- **权限冒泡**: Fork 子代理的权限请求可以冒泡到父终端
- **Prompt Cache 优化**: Fork 继承父级系统提示，确保缓存命中
- **灵活隔离**: worktree 和 remote 模式支持不同级别的隔离
