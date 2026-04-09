# Claude Code Agent 上下文动态加载架构技术文档

> 基于源码 `claude-code-2.1.88` 逆向分析
>
> 生成日期：2026-04-09

---

## 目录

1. [架构总览](#1-架构总览)
2. [上下文窗口计算与弹性扩缩](#2-上下文窗口计算与弹性扩缩)
3. [会话级上下文静态加载](#3-会话级上下文静态加载)
4. [Turn 级动态附件注入](#4-turn-级动态附件注入)
5. [增量 Diff 机制详解](#5-增量-diff-机制详解)
6. [Compaction 上下文压缩机制](#6-compaction-上下文压缩机制)
7. [Agent 上下文隔离与继承](#7-agent-上下文隔离与继承)
8. [Fork Agent 缓存共享设计](#8-fork-agent-缓存共享设计)
9. [Agent Memory 持久化](#9-agent-memory-持久化)
10. [Agent 定义动态加载](#10-agent-定义动态加载)
11. [Skill 动态发现与条件激活](#11-skill-动态发现与条件激活)
12. [Workload 上下文标记](#12-workload-上下文标记)
13. [上下文分析与建议系统](#13-上下文分析与建议系统)
14. [架构全景图](#14-架构全景图)
15. [核心设计原则](#15-核心设计原则)
16. [关键源码索引](#16-关键源码索引)

---

## 1. 架构总览

Claude Code 的 Agent 上下文动态加载系统是一个多层、多机制的复杂系统，解决的核心问题是：

- **如何在有限的上下文窗口中高效加载和维持必要的上下文信息**
- **如何在多个并发 Agent 之间隔离上下文同时共享缓存**
- **如何在上下文溢出时优雅地压缩并重建关键信息**

系统采用"静态基底 + 动态增量 + 按需压缩"的三层架构：

```
静态层（会话级）: System Context + User Context（memoize 缓存）
动态层（Turn级）: 20+ 种 Attachment 类型（增量 diff + 状态驱动）
压缩层（溢出时）: Auto/Manual/Reactive Compaction + 重建
```

---

## 2. 上下文窗口计算与弹性扩缩

### 2.1 核心文件

- `src/utils/context.ts`
- `src/utils/model/contextWindowUpgradeCheck.ts`

### 2.2 上下文窗口优先级决策链

上下文窗口大小的计算采用优先级瀑布式决策链，高优先级命中后短路返回：

```
优先级 1: 环境变量覆盖
  └─ CLAUDE_CODE_MAX_CONTEXT_TOKENS（仅 ant 内部用户）
     └─ 任何正整数，覆盖所有后续决策

优先级 2: 模型名 [1m] 后缀
  └─ 检测模型名中是否包含 [1m]（如 sonnet[1m]）
     └─ 命中 → 直接返回 1,000,000

优先级 3: 模型能力表
  └─ getModelCapability(model).max_input_tokens
     └─ 仅当 >= 100,000 时采用
        └─ 若 > 200K 且 1M 被禁用 → 截断为 200K

优先级 4: Beta 标志
  └─ betas 包含 CONTEXT_1M_BETA_HEADER 且模型支持 1M
     └─ 返回 1,000,000

优先级 5: Sonnet 1M 实验组
  └─ globalConfig.clientDataCache['coral_reef_sonnet'] === 'true'
     └─ 仅对 sonnet-4-6 有效
     └─ 返回 1,000,000

优先级 6: Ant 内部模型解析
  └─ resolveAntModel(model).contextWindow

优先级 7: 默认值
  └─ MODEL_CONTEXT_WINDOW_DEFAULT = 200,000
```

### 2.3 1M 上下文控制

```typescript
// 关键函数
function is1mContextDisabled(): boolean    // HIPAA 合规开关
function has1mContext(model: string)       // 检测 [1m] 后缀
function modelSupports1M(model: string)    // 模型是否支持 1M
```

- `CLAUDE_CODE_DISABLE_1M_CONTEXT` 环境变量可在任何层级截断为 200K
- 当前支持 1M 的模型：`claude-sonnet-4`、`opus-4-6`

### 2.4 有效上下文窗口

有效窗口 ≠ 总窗口，需要预留输出空间：

```typescript
function getEffectiveContextWindowSize(model: string): number {
  const reservedTokensForSummary = Math.min(
    getMaxOutputTokensForModel(model),  // 模型最大输出 token
    20_000                               // 上限：基于 p99.99 的摘要输出
  )
  let contextWindow = getContextWindowForModel(model, getSdkBetas())

  // 环境变量覆盖：进一步限制自动压缩的触发窗口
  const autoCompactWindow = process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW
  if (autoCompactWindow) {
    const parsed = parseInt(autoCompactWindow, 10)
    if (!isNaN(parsed) && parsed > 0) {
      contextWindow = Math.min(contextWindow, parsed)
    }
  }

  return contextWindow - reservedTokensForSummary
}
```

### 2.5 动态上下文升级

当上下文逼近上限时，系统检查用户是否有资格升级到 1M 上下文模型：

```typescript
function getAvailableUpgrade(): { alias: string; name: string; multiplier: number } | null {
  const currentModelSetting = getUserSpecifiedModelSetting()
  if (currentModelSetting === 'opus' && checkOpus1mAccess()) {
    return { alias: 'opus[1m]', name: 'Opus 1M', multiplier: 5 }
  } else if (currentModelSetting === 'sonnet' && checkSonnet1mAccess()) {
    return { alias: 'sonnet[1m]', name: 'Sonnet 1M', multiplier: 5 }
  }
  return null
}
```

这是运行时动态升级，而非静态配置。用户可通过 `/model opus[1m]` 主动切换。

### 2.6 Max Output Token 分层

不同模型有不同的输出 token 上限，影响上下文预算：

| 模型 | 默认输出 | 上限输出 |
|------|---------|---------|
| opus-4-6 | 64,000 | 128,000 |
| sonnet-4-6 | 32,000 | 128,000 |
| opus-4-5 / sonnet-4 / haiku-4 | 32,000 | 64,000 |
| opus-4-1 / opus-4 | 32,000 | 32,000 |
| claude-3-opus | 4,096 | 4,096 |

---

## 3. 会话级上下文静态加载

### 3.1 核心文件

- `src/context.ts`

### 3.2 System Context

在对话开始时通过 `memoize` 一次性计算并缓存：

```typescript
export const getSystemContext = memoize(async (): Promise<{ [k: string]: string }> => {
  // Git 状态快照（CCR 环境或禁用 Git 指令时跳过）
  const gitStatus = ...await getGitStatus()...

  // 缓存破坏器（ant 内部调试用，需 BREAK_CACHE_COMMAND feature flag）
  const injection = feature('BREAK_CACHE_COMMAND') ? getSystemPromptInjection() : null

  return {
    ...(gitStatus && { gitStatus }),
    ...(feature('BREAK_CACHE_COMMAND') && injection ? { cacheBreaker: `[CACHE_BREAKER: ${injection}]` } : {}),
  }
})
```

**Git 状态包含**：
- 当前分支名
- 主分支名（用于 PR 创建）
- `git status --short`（超过 2000 字符截断）
- 最近 5 条 commit（`git log --oneline -n 5`）
- Git 用户名

**缓存策略**：整个对话生命周期内缓存。当 `systemPromptInjection` 变更时手动清缓存：

```typescript
export function setSystemPromptInjection(value: string | null): void {
  systemPromptInjection = value
  getUserContext.cache.clear?.()
  getSystemContext.cache.clear?.()
}
```

**注意**：Git 状态是**快照**，不会在对话过程中更新。注释明确说明：
> "This is the git status at the start of the conversation. Note that this status is a snapshot in time, and will not update during the conversation."

### 3.3 User Context

同样使用 `memoize` 缓存：

```typescript
export const getUserContext = memoize(async (): Promise<{ [k: string]: string }> => {
  // CLAUDE.md 层级内容
  const shouldDisableClaudeMd =
    isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
    (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)

  const claudeMd = shouldDisableClaudeMd
    ? null
    : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))

  // 缓存到 bootstrap state（供 yoloClassifier 等模块使用，避免循环依赖）
  setCachedClaudeMdContent(claudeMd || null)

  return {
    ...(claudeMd && { claudeMd }),
    currentDate: `Today's date is ${getLocalISODate()}.`,
  }
})
```

**CLAUDE.md 加载规则**：
- `CLAUDE_CODE_DISABLE_CLAUDE_MDS`: 硬关闭，总是跳过
- `--bare` 模式：跳过自动发现（cwd walk），但保留 `--add-dir` 指定的目录
- `--bare` 的语义是"跳过我没要求的"，而非"忽略我要求的"

---

## 4. Turn 级动态附件注入

### 4.1 核心文件

- `src/utils/attachments.ts` → `getAttachments()` → `getAttachmentMessages()`

### 4.2 执行流程

每个用户 turn 发出时，系统通过 `getAttachments()` 并行收集动态附件，然后通过 `getAttachmentMessages()` 生成器逐个 yield 为 `AttachmentMessage`：

```typescript
export async function* getAttachmentMessages(
  input, toolUseContext, ideSelection, queuedCommands, messages, querySource, options
): AsyncGenerator<AttachmentMessage, void> {
  const attachments = await getAttachments(...)
  for (const attachment of attachments) {
    yield createAttachmentMessage(attachment)
  }
}
```

`getAttachments()` 内部使用 `Promise.all` 并行处理所有附件类型，每种类型都有 1 秒超时保护。

### 4.3 附件分类

#### 4.3.1 用户输入驱动型

| 附件类型 | 触发条件 | 数据来源 | 说明 |
|---------|---------|---------|------|
| `at_mentioned_files` | 用户输入 `@filepath` | FileReadTool | 读取文件内容，支持行范围 |
| `mcp_resources` | 用户输入 `@mcp://resource` | MCP Server | 从 MCP 服务器读取资源 |
| `agent_mentions` | 用户输入 `@agent-name` | Agent 定义列表 | 触发特定 agent 启动 |
| `skill_discovery` | Turn 0 用户输入 | Haiku 语义匹配 | 基于 skill 搜索信号发现相关 skill |

#### 4.3.2 系统状态驱动型

| 附件类型 | 触发条件 | 机制 | 增量 |
|---------|---------|------|------|
| `deferred_tools_delta` | 工具池变化 | 扫描 deferred tools diff | 是 |
| `agent_listing_delta` | Agent 定义变化 | 扫描历史 delta diff | 是 |
| `mcp_instructions_delta` | MCP 连接/指令变化 | 扫描历史指令 diff | 是 |
| `date_change` | 日期跨天 | 对比 lastEmittedDate | 否 |
| `changed_files` | 文件系统变更 | mtime 检测 | 否 |
| `nested_memory` | 触发嵌套 memory | 子目录 CLAUDE.md | 否 |
| `dynamic_skill` | 运行时发现 skill | 文件路径遍历 | 否 |
| `skill_listing` | 可用 skill 变化 | 增量 skill 清单 | 是 |
| `plan_mode` | 处于 plan mode | 计划文件读取 | 否 |
| `plan_mode_exit` | 退出 plan mode | 状态标记 | 否 |
| `auto_mode` | 处于 auto mode | 模式状态 | 否 |
| `auto_mode_exit` | 退出 auto mode | 状态标记 | 否 |
| `todo_reminders` | TODO 列表存在 | Task 系统 | 否 |
| `critical_system_reminder` | Agent 定义配置 | 每轮重注入 | 否 |
| `compaction_reminder` | 上下文使用率高 | token 阈值检测 | 否 |
| `context_efficiency` | 历史裁剪开启 | 效率分析 | 否 |
| `teammate_mailbox` | Swarm 收到消息 | 邮箱系统 | 否 |
| `team_context` | Swarm 活跃 | 团队上下文 | 否 |
| `agent_pending_messages` | 子 agent 有待处理消息 | Agent 队列 | 否 |
| `ultrathink_effort` | 请求超深度思考 | 用户输入检测 | 否 |
| `queued_commands` | 队列中有命令 | 命令队列 | 否 |
| `companion_intro` | Buddy 模式 | 首次对话 | 否 |

---

## 5. 增量 Diff 机制详解

### 5.1 设计动机

工具描述变更曾占 fleet `cache_creation` 的 10.2%：MCP 异步连接、`/reload-plugins`、权限模式变化 → AgentTool description 变化 → 完整 tool-schema cache 失效。

解决方案：将动态内容从 tool description 中移出，改为 attachment message 增量通知。

### 5.2 通用 Diff 算法

三种 delta attachment 使用相同的 diff 模式：

```typescript
function computeDelta(currentSet: Set<string>, messages: Message[], attachmentType: string): { added: string[], removed: string[] } {
  // 1. 扫描历史消息，重建"已公告集合"
  const announced = new Set<string>()
  for (const msg of messages) {
    if (msg.type !== 'attachment') continue
    if (msg.attachment.type !== attachmentType) continue
    for (const name of msg.attachment.addedNames ?? msg.attachment.addedTypes) announced.add(name)
    for (const name of msg.attachment.removedNames ?? msg.attachment.removedTypes) announced.delete(name)
  }

  // 2. 与当前状态 diff
  const added = [...currentSet].filter(n => !announced.has(n))
  const removed = [...announced].filter(n => !currentSet.has(n))

  // 3. 无变化则不发送
  if (added.length === 0 && removed.length === 0) return null

  return { added, removed }
}
```

### 5.3 Deferred Tools Delta

**核心文件**: `src/utils/toolSearch.ts` → `getDeferredToolsDelta()`

**触发条件**：`isDeferredToolsDeltaEnabled()` 返回 true（ant 内部或 GrowthBook gate `tengu_glacier_2xr`）

**特殊逻辑**：
- 一个名称如果曾经公告为 deferred 但现在不再是 → **不报告为 removed**（因为它现在已在 base pool 中直接加载，告诉模型"不再可用"是错误的）
- 使用 `isToolSearchEnabledOptimistic()` 跳过异步阈值检查（避免双重触发）

### 5.4 Agent Listing Delta

**核心文件**: `src/utils/attachments.ts` → `getAgentListingDeltaAttachment()`

**过滤链**（镜像 AgentTool.prompt() 的过滤）：
```
activeAgents → filterAgentsByMcpRequirements → filterDeniedAgents → allowedAgentTypes restriction
```

**输出格式**：
```typescript
{
  type: 'agent_listing_delta',
  addedTypes: string[],       // 新增的 agent 类型名
  addedLines: string[],       // 格式化的 agent 行: "type: whenToUse (Tools: ...)"
  removedTypes: string[],     // 移除的 agent 类型名
  isInitial: boolean,         // 是否为首次公告（announced.size === 0）
  showConcurrencyNote: boolean // 是否显示并发提示
}
```

### 5.5 MCP Instructions Delta

**核心文件**: `src/utils/mcpInstructionsDelta.ts` → 通过 `getMcpInstructionsDeltaAttachment()` 调用

**特殊逻辑**：
- Chrome MCP 的 ToolSearch 提示是客户端合成的，作为 `ClientSideInstruction` 传入 diff
- 实际服务器的 `instructions` 是无条件的

---

## 6. Compaction 上下文压缩机制

### 6.1 核心文件

- `src/services/compact/autoCompact.ts` — 自动压缩触发逻辑
- `src/services/compact/compact.ts` — 压缩执行
- `src/services/compact/prompt.ts` — 压缩提示词

### 6.2 触发链

```
shouldAutoCompact()
  ├── 递归守卫
  │   ├── querySource === 'session_memory' → false（会死锁）
  │   ├── querySource === 'compact' → false（会死锁）
  │   └── querySource === 'marble_origami' → false（会破坏主线程状态）
  │
  ├── 功能开关
  │   ├── DISABLE_COMPACT → false
  │   ├── DISABLE_AUTO_COMPACT → false
  │   ├── userConfig.autoCompactEnabled === false → false
  │   └── Reactive-only 模式（tengu_cobalt_raccoon gate）→ false
  │
  ├── Context Collapse 模式
  │   └── isContextCollapseEnabled() → false（压缩与 collapse 竞争）
  │
  └── Token 阈值检查
      └── tokenCountWithEstimation(messages) - snipTokensFreed >= autoCompactThreshold
         └── threshold = effectiveContextWindow - 13,000 (AUTOCOMPACT_BUFFER_TOKENS)
```

### 6.3 压缩执行流程

```
compactConversation()
  │
  ├── 1. 执行 PreCompact Hooks
  │   └── 可修改 customInstructions
  │
  ├── 2. 流式生成摘要
  │   ├── 构建摘要请求消息
  │   ├── 消息预处理
  │   │   ├── stripImagesFromMessages() — 图片替换为 [image]/[document] 标记
  │   │   └── stripReinjectedAttachments() — 移除 skill_discovery/skill_listing
  │   │
  │   └── PTL 重试循环（最多 3 次）
  │       ├── 如果摘要请求本身 prompt-too-long
  │       ├── truncateHeadForPTLRetry() — 丢弃最老的 API round groups
  │       └── 重试直到成功或放弃
  │
  ├── 3. 清空状态缓存
  │   ├── readFileState.clear()
  │   └── loadedNestedMemoryPaths.clear()
  │
  ├── 4. 并行重建附件
  │   ├── createPostCompactFileAttachments() — 恢复关键文件
  │   ├── createAsyncAgentAttachmentsIfNeeded()
  │   ├── 计划附件 + 计划模式附件
  │   ├── Skill 附件（带 token 预算）
  │   ├── getDeferredToolsDeltaAttachment() — 全量重新公告
  │   ├── getAgentListingDeltaAttachment() — 全量重新公告
  │   └── getMcpInstructionsDeltaAttachment() — 全量重新公告
  │
  ├── 5. SessionStart Hooks（压缩后执行）
  │
  ├── 6. PostCompact Hooks
  │
  └── 7. 返回 CompactionResult
      ├── boundaryMarker — 压缩边界标记
      ├── summaryMessages — 摘要消息
      ├── attachments — 重建的附件
      ├── hookResults — 钩子结果
      └── token 统计信息
```

### 6.4 压缩后 Token 预算

| 项目 | 预算 | 说明 |
|------|------|------|
| 文件附件总数 | 最多 5 个 | `POST_COMPACT_MAX_FILES_TO_RESTORE` |
| 文件附件总 token | 50,000 | `POST_COMPACT_TOKEN_BUDGET` |
| 单文件 token | 5,000 | `POST_COMPACT_MAX_TOKENS_PER_FILE` |
| 单 Skill token | 5,000 | `POST_COMPACT_MAX_TOKENS_PER_SKILL` |
| Skill 总 token | 25,000 | `POST_COMPACT_SKILLS_TOKEN_BUDGET` |

### 6.5 压缩后上下文结构

```
[compactBoundaryMarker]           ← 压缩边界（含 metadata: auto/manual, preCompactTokenCount）
[summaryMessages]                 ← 对话摘要（<analysis> + <summary> 结构）
[messagesToKeep]                  ← 部分压缩保留的近期消息
[attachments]                     ← 重建的附件（文件/Skill/工具/MCP）
[hookResults]                     ← 钩子产生的消息
```

### 6.6 熔断器

```typescript
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

连续 3 次自动压缩失败后停止重试，避免在不可恢复的上下文溢出场景中浪费 API 调用。之前有会话产生高达 3,272 次连续失败，浪费约 250K API 调用/天。

### 6.7 部分压缩

支持两个方向：

- **`from`**：从指定位置向后摘要，保留更早的消息（保留 prompt cache）
- **`up_to`**：从开头到指定位置摘要，保留更近的消息（cache 失效）

`up_to` 方向会清除保留部分中的旧 compact boundary/summary，避免向后扫描时找到旧边界导致新的摘要被丢弃。

### 6.8 Session Memory Compaction

自动压缩首先尝试 `trySessionMemoryCompaction()`，这是一种更轻量的压缩方式。如果成功则跳过完整的 `compactConversation()`。

### 6.9 压缩提示词结构

```
NO_TOOLS_PREAMBLE     ← 禁止使用工具（关键：强制纯文本输出）
DETAILED_ANALYSIS     ← <analysis> 分析块指令
SUMMARY_SECTIONS      ← 9 个摘要段落
  1. Primary Request and Intent
  2. Key Technical Concepts
  3. Files and Code Sections
  4. Errors and fixes
  5. Problem Solving
  6. All user messages
  7. Pending Tasks
  8. Current Work
  9. Optional Next Step
```

---

## 7. Agent 上下文隔离与继承

### 7.1 核心文件

- `src/utils/agentContext.ts`

### 7.2 AsyncLocalStorage 上下文传递

使用 Node.js 的 `AsyncLocalStorage`（ALS）实现异步链路安全的上下文隔离：

```typescript
const agentContextStorage = new AsyncLocalStorage<AgentContext>()

export function runWithAgentContext<T>(context: AgentContext, fn: () => T): T {
  return agentContextStorage.run(context, fn)
}

export function getAgentContext(): AgentContext | undefined {
  return agentContextStorage.getStore()
}
```

### 7.3 为什么不用 AppState

当 agent 被 backgrounded（ctrl+b）时，多个 agent 在同一进程中并发运行。AppState 是单一共享状态，会被覆盖，导致 Agent A 的事件错误使用 Agent B 的上下文。ALS 隔离每个异步执行链，并发 agent 互不干扰。

### 7.4 两种 Agent 上下文类型

#### SubagentContext

```typescript
type SubagentContext = {
  agentId: string                    // 子 agent 的 UUID
  parentSessionId?: string           // 父会话 ID
  agentType: 'subagent'              // 类型标记
  subagentName?: string              // Agent 类型名（如 "Explore", "code-reviewer"）
  isBuiltIn?: boolean                // 是否为内置 agent
  invokingRequestId?: string         // 触发此 agent 的 request_id
  invocationKind?: 'spawn' | 'resume' // 是初始启动还是后续恢复
  invocationEmitted?: boolean        // 是否已发送遥测事件
}
```

#### TeammateAgentContext

```typescript
type TeammateAgentContext = {
  agentId: string                    // 完整 ID（如 "researcher@my-team"）
  agentName: string                  // 显示名（如 "researcher"）
  teamName: string                   // 团队名
  agentColor?: string                // UI 颜色
  planModeRequired: boolean          // 是否必须先 plan
  parentSessionId: string            // 团队负责人的 session ID
  isTeamLead: boolean                // 是否为团队负责人
  agentType: 'teammate'              // 类型标记
  invokingRequestId?: string         // 同 SubagentContext
  invocationKind?: 'spawn' | 'resume' // 同 SubagentContext
  invocationEmitted?: boolean        // 同 SubagentContext
}
```

### 7.5 跨进程 Agent

对于通过 tmux/iTerm2 运行的独立进程 Swarm 队友，使用环境变量而非 ALS：

- `CLAUDE_CODE_AGENT_ID`
- `CLAUDE_CODE_PARENT_SESSION_ID`

### 7.6 一次性 Request ID 消费

```typescript
export function consumeInvokingRequestId(): { invokingRequestId: string; invocationKind: string } | undefined {
  const context = getAgentContext()
  if (!context?.invokingRequestId || context.invocationEmitted) return undefined
  context.invocationEmitted = true  // 标记已消费
  return { invokingRequestId: context.invokingRequestId, invocationKind: context.invocationKind }
}
```

这确保每个 spawn/resume 边界只产生一个遥测事件。

---

## 8. Fork Agent 缓存共享设计

### 8.1 核心文件

- `src/utils/forkedAgent.ts`
- `src/tools/AgentTool/forkSubagent.ts`

### 8.2 CacheSafeParams

Fork 子 agent 的核心设计是字节级精确共享父 agent 的 prompt cache：

```typescript
type CacheSafeParams = {
  systemPrompt: SystemPrompt        // 必须匹配父请求
  userContext: { [k: string]: string }  // 影响消息前缀
  systemContext: { [k: string]: string } // 影响系统提示
  toolUseContext: ToolUseContext     // 包含 tools, model, options
  forkContextMessages: Message[]     // 父级上下文消息
}
```

Anthropic API 的 cache key 由以下组成：system prompt、tools、model、messages (prefix)、thinking config。CacheSafeParams 携带前五个。

### 8.3 Fork 消息构造

```typescript
function buildForkedMessages(directive: string, assistantMessage: AssistantMessage): Message[] {
  // 1. 克隆完整的父 assistant 消息（保留所有 tool_use blocks）
  const fullAssistantMessage = { ...assistantMessage, uuid: randomUUID() }

  // 2. 为每个 tool_use 构建统一占位符 tool_result
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result',
    tool_use_id: block.id,
    content: [{ type: 'text', text: 'Fork started — processing in background' }]
  }))

  // 3. 构建单条 user 消息：占位符 results + 子 agent 指令
  const toolResultMessage = createUserMessage({
    content: [...toolResultBlocks, { type: 'text', text: buildChildMessage(directive) }]
  })

  return [fullAssistantMessage, toolResultMessage]
}
```

**关键**：所有 fork 子 agent 的占位符 `tool_result` 文本完全相同，只有最后的 `directive text block` 不同。这最大化了 cache hit。

### 8.4 Fork 子 Agent 指令格式

```
<fork_boilerplate>
STOP. READ THIS FIRST.

You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Your system prompt says "default to forking." IGNORE IT — that's for the parent.
2. Do NOT converse, ask questions, or suggest next steps
3. Do NOT editorialize or add meta-commentary
4. USE your tools directly: Bash, Read, Write, etc.
5. If you modify files, commit your changes before reporting
6. Do NOT emit text between tool calls
7. Stay strictly within your directive's scope
8. Keep your report under 500 words
9. Your response MUST begin with "Scope:"
10. REPORT structured facts, then stop

Output format:
  Scope: <echo back assigned scope>
  Result: <answer or key findings>
  Key files: <relevant file paths>
  Files changed: <list with commit hash>
  Issues: <list>
</fork_boilerplate>

FORK_DIRECTIVE: <user's directive>
```

### 8.5 防止递归 Fork

```typescript
function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false
    const content = m.message.content
    if (!Array.isArray(content)) return false
    return content.some(block => block.type === 'text' && block.text.includes('<fork_boilerplate>'))
  })
}
```

Fork 子 agent 保留 Agent tool 在 tool pool 中（为了 cache key 匹配），但在调用时检测递归并拒绝。

### 8.6 Fork Agent 定义

```typescript
const FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],            // 使用 useExactTools：继承父级完整 tool pool
  maxTurns: 200,
  model: 'inherit',         // 继承父级模型（保持上下文长度一致）
  permissionMode: 'bubble', // 权限提示上浮到父终端
  getSystemPrompt: () => '', // 不使用：直接传入父级已渲染的 system prompt bytes
}
```

**`getSystemPrompt` 返回空字符串**：Fork 路径通过 `toolUseContext.renderedSystemPrompt` 传入父级已渲染的 system prompt bytes。重新调用 `getSystemPrompt()` 可能因 GrowthBook 冷/热状态差异而产生不同结果，破坏 cache。

### 8.7 缓存安全注意事项

Fork 不设置 `maxOutputTokens`，因为它会通过 `claude.ts` 中的 clamping 改变 `budget_tokens`，而 thinking config 是 cache key 的一部分。不同 `budget_tokens` = cache miss。

---

## 9. Agent Memory 持久化

### 9.1 核心文件

- `src/tools/AgentTool/agentMemory.ts`
- `src/tools/AgentTool/agentMemorySnapshot.ts`

### 9.2 三层作用域

| 作用域 | 路径 | 特点 | VCS |
|-------|------|------|-----|
| `user` | `~/.claude/agent-memory/<agentType>/` | 跨项目通用 | 不提交 |
| `project` | `.claude/agent-memory/<agentType>/` | 项目级 | 可提交 |
| `local` | `.claude/agent-memory-local/<agentType>/` | 项目级 | 不提交 |

`local` 作用域支持 `CLAUDE_CODE_REMOTE_MEMORY_DIR` 环境变量，将持久化重定向到远程挂载点（带项目命名空间）。

### 9.3 Memory 加载

```typescript
function loadAgentMemoryPrompt(agentType: string, scope: AgentMemoryScope): string {
  // 作用域特定的指南
  const scopeNote = {
    user: 'keep learnings general since they apply across all projects',
    project: 'tailor your memories to this project',
    local: 'tailor your memories to this project and machine',
  }[scope]

  const memoryDir = getAgentMemoryDir(agentType, scope)

  // 异步创建目录（fire-and-forget：spawn 时同步调用，不能 await）
  void ensureMemoryDirExists(memoryDir)

  return buildMemoryPrompt({
    displayName: 'Persistent Agent Memory',
    memoryDir,
    extraGuidelines: [scopeNote, ...(coworkExtraGuidelines ? [coworkExtraGuidelines] : [])],
  })
}
```

### 9.4 快照机制

项目可在 `.claude/agent-memory-snapshots/<agentType>/` 中预置 agent 记忆快照：

```
agent-memory-snapshots/
  code-reviewer/
    snapshot.json         ← { updatedAt: "2026-01-01T00:00:00Z" }
    CODE_STYLE.md         ← 记忆文件
    COMMON_PATTERNS.md
```

**生命周期**：

1. **首次启动**（`action: 'initialize'`）：本地无记忆 → 从快照完整复制
2. **快照更新**（`action: 'prompt-update'`）：快照比本地新 → 标记 `pendingSnapshotUpdate`
3. **已同步**（`action: 'none'`）：跳过

### 9.5 Memory 注入到 System Prompt

```typescript
// 在 parseAgentFromMarkdown / parseAgentFromJson 中
getSystemPrompt: () => {
  if (isAutoMemoryEnabled() && memory) {
    const memoryPrompt = loadAgentMemoryPrompt(agentType, memory)
    return systemPrompt + '\n\n' + memoryPrompt
  }
  return systemPrompt
}
```

Memory 提示在每次调用 `getSystemPrompt()` 时动态拼接，而非静态嵌入。

---

## 10. Agent 定义动态加载

### 10.1 核心文件

- `src/tools/AgentTool/loadAgentsDir.ts`

### 10.2 定义来源与优先级

```typescript
function getActiveAgentsFromList(allAgents: AgentDefinition[]): AgentDefinition[] {
  const agentGroups = [
    builtInAgents,     // 内置 agent（Explore, Plan, code-reviewer 等）
    pluginAgents,      // 插件 agent
    userAgents,        // 用户级设置 (~/.claude/agents/)
    projectAgents,     // 项目级设置 (.claude/agents/)
    flagAgents,        // Feature flag 设置
    managedAgents,     // 管理策略设置
  ]

  const agentMap = new Map<string, AgentDefinition>()
  for (const agents of agentGroups) {
    for (const agent of agents) {
      agentMap.set(agent.agentType, agent)  // 后覆盖前
    }
  }
  return Array.from(agentMap.values())
}
```

优先级：`built-in < plugin < userSettings < projectSettings < flagSettings < policySettings`

### 10.3 Agent 定义类型

```typescript
// 内置 Agent：动态 prompt
type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  getSystemPrompt: (params: { toolUseContext: Pick<ToolUseContext, 'options'> }) => string
}

// 自定义 Agent：prompt 存储在闭包中
type CustomAgentDefinition = BaseAgentDefinition & {
  source: SettingSource
  getSystemPrompt: () => string
}

// 插件 Agent：带插件元数据
type PluginAgentDefinition = BaseAgentDefinition & {
  source: 'plugin'
  plugin: string
  getSystemPrompt: () => string
}
```

### 10.4 Agent 核心字段

```typescript
type BaseAgentDefinition = {
  agentType: string                    // Agent 类型名
  whenToUse: string                    // 使用场景描述
  tools?: string[]                     // 允许的工具列表
  disallowedTools?: string[]           // 禁止的工具列表
  skills?: string[]                    // 预加载的 skill
  mcpServers?: AgentMcpServerSpec[]    // 专用 MCP 服务器
  hooks?: HooksSettings               // 会话级钩子
  color?: AgentColorName              // UI 颜色
  model?: string                       // 使用的模型（'inherit' 继承父级）
  effort?: EffortValue                // 推理力度
  permissionMode?: PermissionMode      // 权限模式
  maxTurns?: number                   // 最大 agentic 轮次
  omitClaudeMd?: boolean              // 是否省略 CLAUDE.md（节省 ~5-15 Gtok/周）
  memory?: AgentMemoryScope           // 持久记忆作用域
  isolation?: 'worktree' | 'remote'   // 隔离模式
  background?: boolean                // 是否总是后台运行
  initialPrompt?: string              // 首轮用户 turn 前置内容
  criticalSystemReminder_EXPERIMENTAL?: string  // 每轮重注入的短提示
  requiredMcpServers?: string[]       // 必须配置的 MCP 服务器
  pendingSnapshotUpdate?: { snapshotTimestamp: string }
}
```

### 10.5 缓存策略

```typescript
export const getAgentDefinitionsWithOverrides = memoize(async (cwd: string): Promise<AgentDefinitionsResult> => {
  // ...加载逻辑
})

export function clearAgentDefinitionsCache(): void {
  getAgentDefinitionsWithOverrides.cache.clear?.()
  clearPluginAgentCache()
}
```

使用 `lodash-es/memoize` 缓存。`/reload-plugins` 等操作调用 `clearAgentDefinitionsCache()` 刷新。

### 10.6 MCP 要求过滤

```typescript
function hasRequiredMcpServers(agent: AgentDefinition, availableServers: string[]): boolean {
  if (!agent.requiredMcpServers || agent.requiredMcpServers.length === 0) return true
  return agent.requiredMcpServers.every(pattern =>
    availableServers.some(server => server.toLowerCase().includes(pattern.toLowerCase()))
  )
}
```

Agent 可以声明所需的 MCP 服务器，未满足的 agent 不会出现在可用列表中。

---

## 11. Skill 动态发现与条件激活

### 11.1 核心文件

- `src/skills/loadSkillsDir.ts`

### 11.2 三层加载机制

#### 启动时加载

从五个目录并行加载：

```typescript
const [managedSkills, userSkills, projectSkillsNested, additionalSkillsNested, legacyCommands] = await Promise.all([
  loadSkillsFromSkillsDir(managedSkillsDir, 'policySettings'),    // 管理策略
  loadSkillsFromSkillsDir(userSkillsDir, 'userSettings'),          // 用户级
  projectSkillsDirs.map(dir => loadSkillsFromSkillsDir(dir, 'projectSettings')),  // 项目级
  additionalDirs.map(dir => loadSkillsFromSkillsDir(join(dir, '.claude', 'skills'), 'projectSettings')),  // 附加目录
  loadSkillsFromCommandsDir(cwd),                                   // 遗留命令
])
```

去重使用 `realpath` 解析符号链接后的 canonical path。

#### 运行时动态发现

当用户操作触及新文件路径时：

```typescript
async function discoverSkillDirsForPaths(filePaths: string[], cwd: string): Promise<string[]> {
  for (const filePath of filePaths) {
    let currentDir = dirname(filePath)
    // 沿文件路径向上遍历到 cwd（不含 cwd 本身）
    while (currentDir.startsWith(resolvedCwd + pathSep)) {
      const skillDir = join(currentDir, '.claude', 'skills')
      if (!dynamicSkillDirs.has(skillDir)) {
        dynamicSkillDirs.add(skillDir)  // 记录已检查路径（含不存在的）
        // 检查目录是否存在 + 是否被 gitignore
        if (await fs.stat(skillDir) && !await isPathGitignored(currentDir, resolvedCwd)) {
          newDirs.push(skillDir)
        }
      }
      currentDir = dirname(currentDir)
    }
  }
  // 深度优先排序：更接近文件的 skill 优先级更高
  return newDirs.sort((a, b) => b.split(pathSep).length - a.split(pathSep).length)
}
```

**安全检查**：通过 `isPathGitignored` 检查 `.gitignore`，阻止如 `node_modules/pkg/.claude/skills` 被静默加载。

#### 条件激活

带有 `paths` frontmatter 的 skill 在匹配的文件被操作时才激活：

```typescript
function activateConditionalSkillsForPaths(filePaths: string[], cwd: string): string[] {
  for (const [name, skill] of conditionalSkills) {
    const skillIgnore = ignore().add(skill.paths)  // gitignore 风格匹配
    for (const filePath of filePaths) {
      const relativePath = isAbsolute(filePath) ? relative(cwd, filePath) : filePath
      if (skillIgnore.ignores(relativePath)) {
        dynamicSkills.set(name, skill)      // 移入动态 skill
        conditionalSkills.delete(name)       // 从条件列表移除
        activatedConditionalSkillNames.add(name)  // 记录已激活（缓存清空后仍保留）
        break
      }
    }
  }
}
```

### 11.3 Skill 通知机制

动态 Skill 发现后通过 `createSignal()` 通知订阅者：

```typescript
const skillsLoaded = createSignal()

export function onDynamicSkillsLoaded(callback: () => void): () => void {
  return skillsLoaded.subscribe(() => {
    try { callback() } catch (error) { logError(error) }
  })
}
```

订阅者可清空相关缓存。Skill 内容本身不直接注入上下文，而是通过下一轮 turn 的 `skill_listing` 附件增量告知模型。

---

## 12. Workload 上下文标记

### 12.1 核心文件

- `src/utils/workloadContext.ts`

### 12.2 设计动机

使用 ALS（而非全局变量）标记 workload 类型，因为 void-detached 后台 agent 在首个 await 时让出执行权，父 turn 的同步续体（包括 finally block）先于 detached 闭包恢复运行。全局变量 `setWorkload('cron')` 会被父 turn 确定性覆盖。

### 12.3 实现

```typescript
type Workload = 'cron'

const workloadStorage = new AsyncLocalStorage<{ workload: string | undefined }>()

export function getWorkload(): string | undefined {
  return workloadStorage.getStore()?.workload
}

export function runWithWorkload<T>(workload: string | undefined, fn: () => T): T {
  return workloadStorage.run({ workload }, fn)  // 总是创建新边界，即使 workload 为 undefined
}
```

**关键**：即使 `workload` 为 `undefined` 也调用 `.run()`，而不是 `return fn()` 直通。直通会导致 ALS 上下文泄漏（REPL 的 notify → React subscriber → 调度链 → useQueueProcessor effect → executeQueuedInput），一旦泄漏就永远粘滞。

---

## 13. 上下文分析与建议系统

### 13.1 核心文件

- `src/utils/contextAnalysis.ts` — Token 统计分析
- `src/utils/contextSuggestions.ts` — 上下文优化建议

### 13.2 Token 统计

`analyzeContext()` 遍历所有消息，按类型统计 token 使用：

```typescript
type TokenStats = {
  toolRequests: Map<string, number>         // 按工具名统计请求 token
  toolResults: Map<string, number>          // 按工具名统计结果 token
  humanMessages: number                     // 人类消息 token
  assistantMessages: number                 // 助手消息 token
  localCommandOutputs: number               // 本地命令输出 token
  other: number                             // 其他类型 token
  attachments: Map<string, number>          // 按类型统计附件数
  duplicateFileReads: Map<string, { count: number; tokens: number }>  // 重复文件读取
  total: number                             // 总 token 数
}
```

### 13.3 上下文建议

`generateContextSuggestions()` 根据统计数据生成优化建议：

| 检查 | 阈值 | 建议 |
|------|------|------|
| 上下文接近容量 | >= 80% | 提前压缩 |
| Bash 结果过大 | >= 15% 且 >= 10K token | 使用 head/tail/grep 管道 |
| Read 结果过多 | >= 5% 且 >= 10K token | 使用 offset/limit |
| Grep 结果过多 | >= 15% 且 >= 10K token | 加更具体的模式 |
| WebFetch 结果过大 | >= 15% 且 >= 10K token | 提取特定信息 |
| Memory 文件膨胀 | >= 5% 且 >= 5K token | 使用 /memory 清理 |
| AutoCompact 禁用 | 50-80% 之间 | 建议启用 |

---

## 14. 架构全景图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        上下文动态加载全景图                               │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────┐  会话启动时  ┌──────────────────────────────────┐  │
│  │  System Context  │◄─memoize───│ Git Status + CacheBreaker        │  │
│  └─────────────────┘             └──────────────────────────────────┘  │
│  ┌─────────────────┐  会话启动时  ┌──────────────────────────────────┐  │
│  │  User Context    │◄─memoize───│ CLAUDE.md Hierarchy + Date       │  │
│  └─────────────────┘             └──────────────────────────────────┘  │
│                                                                          │
│  ┌─────────────────┐  每轮 Turn    ┌──────────────────────────────────┐ │
│  │  Attachments     │◄─parallel───│ 20+ 种动态附件                    │ │
│  │                  │             │ (增量 diff / 状态驱动)             │ │
│  └────────┬─────────┘             └──────────────────────────────────┘ │
│           │                                                              │
│      ┌────┴──────────────────────────────────────┐                      │
│      │  增量 Diff 三驾马车                         │                      │
│      │  ├── deferred_tools_delta (工具池变化)      │                      │
│      │  ├── agent_listing_delta (Agent 列表变化)   │                      │
│      │  └── mcp_instructions_delta (MCP 指令变化)  │                      │
│      ├────────────────────────────────────────────┤                      │
│      │  状态驱动附件                               │                      │
│      │  ├── changed_files / nested_memory         │                      │
│      │  ├── skill_listing / dynamic_skill         │                      │
│      │  ├── teammate_mailbox / team_context       │                      │
│      │  ├── plan_mode / todo_reminders            │                      │
│      │  ├── critical_system_reminder              │                      │
│      │  ├── compaction_reminder / context_efficiency│                     │
│      │  └── ...                                   │                      │
│      └────────────────────────────────────────────┘                      │
│                                                                          │
│  ┌─────────────────┐  上下文溢出    ┌──────────────────────────────────┐ │
│  │  Compaction      │◄─threshold──│ Auto / Manual / Reactive         │ │
│  │                  │             │ + SessionMemory 路径              │ │
│  └────────┬─────────┘             └──────────────────────────────────┘ │
│           │ 重建                                                         │
│      ┌────┴──────────────────────────────────────┐                      │
│      │  文件附件(5个/50K) + Skill附件(25K)        │                      │
│      │  工具/Agent/MCP 全量重新公告                │                      │
│      │  Plan + Todo + HookResults                │                      │
│      └────────────────────────────────────────────┘                      │
│                                                                          │
│  ┌─────────────────┐  Agent 隔离    ┌──────────────────────────────────┐ │
│  │  AgentContext    │◄─ALS─────────│ Subagent / Teammate 上下文        │ │
│  └─────────────────┘               └──────────────────────────────────┘ │
│                                                                          │
│  ┌─────────────────┐  Fork 缓存     ┌──────────────────────────────────┐ │
│  │  CacheSafeParams │◄─byte-exact──│ 父级 prompt cache 共享            │ │
│  │                  │              │ 占位符 results + 子 agent 指令     │ │
│  └─────────────────┘               └──────────────────────────────────┘ │
│                                                                          │
│  ┌─────────────────┐  记忆持久化    ┌──────────────────────────────────┐ │
│  │  Agent Memory    │◄─3-scope─────│ user / project / local + 快照     │ │
│  └─────────────────┘               └──────────────────────────────────┘ │
│                                                                          │
│  ┌─────────────────┐  Skill 发现    ┌──────────────────────────────────┐ │
│  │  Dynamic Skills  │◄─path-walk───│ 启动加载 + 运行时发现             │ │
│  │                  │              │ + 条件激活(paths 匹配)            │ │
│  └─────────────────┘               └──────────────────────────────────┘ │
│                                                                          │
│  ┌─────────────────┐  Workload      ┌──────────────────────────────────┐ │
│  │  WorkloadContext │◄─ALS─────────│ cron 标记（防泄漏边界）            │ │
│  └─────────────────┘               └──────────────────────────────────┘ │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 15. 核心设计原则

### 原则 1：增量优于全量

工具、Agent、MCP 指令都通过 delta diff 机制只传递变化部分，保护 prompt cache。这解决了工具描述变更导致 10.2% fleet cache_creation 的问题。

### 原则 2：延迟优于预加载

- Agent 的 `getSystemPrompt()` 是闭包延迟求值
- Skill 按条件激活（`paths` frontmatter）
- 工具延迟加载（deferred tools + ToolSearch）
- Agent Memory 在首次调用 `getSystemPrompt()` 时才拼接

### 原则 3：并行优于串行

- `getAttachments()` 中所有附件类型通过 `Promise.all` 并行收集
- Agent 定义加载：plugin agents + memory snapshot init 并行
- Skill 加载：managed/user/project/additional/legacy 五路并行
- 压缩后附件重建：文件附件 + agent 附件并行

### 原则 4：缓存优于重算

- `memoize` 缓存：system context、user context、agent 定义、skill 目录
- `FileStateCache` 缓存文件读取状态
- `lastCacheSafeParams` 缓存 fork 参数，供 post-turn forks 共享
- `dynamicSkillDirs` 记录已检查路径，避免重复 stat

### 原则 5：隔离优于共享

- `AsyncLocalStorage` 隔离 agent 上下文，并发 agent 互不干扰
- Fork 通过字节精确匹配共享缓存但不共享可变状态
- Workload ALS 总是创建新边界，防止上下文泄漏
- 压缩时 `readFileState.clear()` + `cloneFileStateCache()` 隔离子 agent 状态

### 原则 6：优雅降级优于硬失败

- 压缩 PTL 重试：摘要请求本身超长时截断最老消息组重试
- 熔断器：连续 3 次压缩失败后停止重试
- 上下文升级：接近上限时建议切换到 1M 模型
- 文件截断：Git status 超 2K 字符时截断并提示

---

## 16. 关键源码索引

| 功能模块 | 源码路径 |
|---------|---------|
| 上下文窗口计算 | `src/utils/context.ts` |
| 上下文升级检查 | `src/utils/model/contextWindowUpgradeCheck.ts` |
| System/User Context | `src/context.ts` |
| 动态附件注入 | `src/utils/attachments.ts` |
| 上下文分析 | `src/utils/contextAnalysis.ts` |
| 上下文建议 | `src/utils/contextSuggestions.ts` |
| 自动压缩 | `src/services/compact/autoCompact.ts` |
| 压缩执行 | `src/services/compact/compact.ts` |
| 压缩提示词 | `src/services/compact/prompt.ts` |
| Agent 上下文隔离 | `src/utils/agentContext.ts` |
| Fork Agent | `src/utils/forkedAgent.ts` |
| Fork 子 Agent | `src/tools/AgentTool/forkSubagent.ts` |
| Agent Memory | `src/tools/AgentTool/agentMemory.ts` |
| Agent Memory 快照 | `src/tools/AgentTool/agentMemorySnapshot.ts` |
| Agent 定义加载 | `src/tools/AgentTool/loadAgentsDir.ts` |
| Agent Tool Prompt | `src/tools/AgentTool/prompt.ts` |
| Skill 动态发现 | `src/skills/loadSkillsDir.ts` |
| Workload 上下文 | `src/utils/workloadContext.ts` |
| 延迟工具 Diff | `src/utils/toolSearch.ts` |
| MCP 指令 Diff | `src/utils/mcpInstructionsDelta.ts` |
| CLAUDE.md 加载 | `src/utils/claudemd.ts` |
