# Claude Code 源码架构深度分析报告

> 版本: 2.1.88 | 分析日期: 2026-04-03

---

## 目录

1. [项目概述](#1-项目概述)
2. [整体架构](#2-整体架构)
3. [入口点与启动流程](#3-入口点与启动流程)
4. [Agent 运行机制与主循环](#4-agent-运行机制与主循环)
5. [工具系统设计](#5-工具系统设计)
6. [权限与安全模型](#6-权限与安全模型)
7. [MCP 服务器集成](#7-mcp-服务器集成)
8. [配置与设置系统](#8-配置与设置系统)
9. [会话管理与持久化](#9-会话管理与持久化)
10. [Hooks 钩子系统](#10-hooks-钩子系统)
11. [API 通信层](#11-api-通信层)
12. [UI 与终端渲染](#12-ui-与终端渲染)
13. [设计模式总结](#13-设计模式总结)
14. [关键文件索引](#14-关键文件索引)

---

## 1. 项目概述

### 1.1 项目定位

Claude Code 是 Anthropic 官方推出的终端 AI 编程助手，是一个**代理式编码工具 (Agentic Coding Tool)**。它具备以下核心能力：

- **代码理解**: 阅读和分析代码库
- **文件操作**: 编辑、创建、删除文件
- **Shell 执行**: 运行终端命令
- **Git 工作流**: 处理版本控制任务
- **MCP 集成**: 扩展外部工具能力

### 1.2 技术栈

| 层级 | 技术 |
|------|------|
| **语言** | TypeScript |
| **运行时** | Node.js 18+ / Bun |
| **UI 框架** | React + Ink (终端 UI) |
| **布局引擎** | Yoga (Flexbox) |
| **验证** | Zod |
| **API 客户端** | @anthropic-ai/sdk |
| **MCP SDK** | @modelcontextprotocol/sdk |

### 1.3 项目结构

```
src/
├── entrypoints/           # 入口点
│   ├── cli.tsx           # CLI 主入口
│   ├── mcp.ts            # MCP 服务器模式
│   └── sdk/              # SDK 入口
├── main.ts               # 主逻辑入口
├── REPL.tsx              # 主循环组件
├── query.ts              # 查询循环核心
├── tools/                # 工具实现
│   ├── Agent/            # Agent 工具
│   ├── Bash/             # Bash 工具
│   ├── Edit/             # 编辑工具
│   └── ...
├── services/             # 服务层
│   ├── api/              # API 服务
│   ├── mcp/              # MCP 服务
│   └── compact/          # 压缩服务
├── utils/                # 工具函数
│   ├── permissions/      # 权限管理
│   ├── settings/         # 设置管理
│   ├── hooks/            # 钩子系统
│   └── ...
├── components/           # UI 组件
├── ink/                  # Ink 终端 UI
├── vim/                  # Vim 模式
├── types/                # 类型定义
├── constants/            # 常量定义
└── bootstrap/            # 启动状态
```

---

## 2. 整体架构

### 2.1 架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户界面层                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    REPL.tsx (React + Ink)                           │   │
│  │  - AppState 状态管理                                                 │   │
│  │  - 消息渲染 (Messages.tsx)                                           │   │
│  │  - 输入处理 (PromptInput.tsx)                                        │   │
│  │  - Vim 模式 (VimTextInput.tsx)                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              核心循环层                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      query.ts (查询循环)                             │   │
│  │  - while(true) 无限循环                                              │   │
│  │  - 消息处理 → API 调用 → 工具执行 → 继续判断                           │   │
│  │  - 自动压缩 / 上下文折叠                                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              工具执行层                                      │
│  ┌──────────────┬──────────────┬──────────────┬──────────────────────┐    │
│  │   BashTool   │   EditTool   │   AgentTool  │   MCPTool / 其他     │    │
│  │  (Shell)     │  (文件编辑)   │  (子代理)    │   (扩展工具)         │    │
│  └──────────────┴──────────────┴──────────────┴──────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    权限检查 (permissions.ts)                         │   │
│  │  - 规则匹配 (allow/deny/ask)                                         │   │
│  │  - 沙箱执行 (sandbox-adapter.ts)                                     │   │
│  │  - 用户确认对话框                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              服务支撑层                                      │
│  ┌──────────────┬──────────────┬──────────────┬──────────────────────┐    │
│  │  API Client  │  MCP Client  │  Settings    │  Session Storage     │    │
│  │  (claude.ts) │  (client.ts) │  (settings/) │  (sessionStorage.ts) │    │
│  └──────────────┴──────────────┴──────────────┴──────────────────────┘    │
│  ┌──────────────┬──────────────┬──────────────┬──────────────────────┐    │
│  │  Hooks       │  Auth        │  Compact     │  Memory              │    │
│  │  (hooks/)    │  (auth.ts)   │  (compact/)  │  (SessionMemory/)    │    │
│  └──────────────┴──────────────┴──────────────┴──────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 数据流

```
用户输入 → processUserInput() → onQuery() → query()
                                              │
                                              ▼
                                        API 调用 (streaming)
                                              │
                                              ▼
                                        Assistant Message
                                              │
                                   ┌──────────┴──────────┐
                                   │                     │
                              无 tool_use            有 tool_use
                                   │                     │
                                   ▼                     ▼
                              返回用户            runTools() 执行
                                                         │
                                                         ▼
                                                   Tool Result
                                                         │
                                                         ▼
                                                    继续循环
```

---

## 3. 入口点与启动流程

### 3.1 CLI 入口 (entrypoints/cli.tsx)

```typescript
async function main(): Promise<void> {
  const args = process.argv.slice(2);
  
  // 1. 快速路径: --version
  if (args[0] === '--version' || args[0] === '-v') {
    console.log(`${MACRO.VERSION} (Claude Code)`);
    return;
  }
  
  // 2. 特殊模式分发
  if (args[0] === '--daemon-worker') {
    await runDaemonWorker(args[1]);  // 守护进程 worker
    return;
  }
  if (args[0] === 'remote-control' || args[0] === 'rc') {
    await bridgeMain(args.slice(1));  // 远程控制模式
    return;
  }
  if (args[0] === 'daemon') {
    await daemonMain(args.slice(1));  // 守护进程模式
    return;
  }
  if (['ps', 'logs', 'attach', 'kill'].includes(args[0])) {
    await bg.handleBgFlag(args);  // 后台会话管理
    return;
  }
  
  // 3. 正常 CLI 模式
  const { main: cliMain } = await import('../main.js');
  await cliMain();
}
```

### 3.2 启动流程

```
cli.tsx
    │
    ├── 检查 --version / 特殊标志
    │
    ▼
main.ts
    │
    ├── enableConfigs() - 启用配置系统
    ├── initSinks() - 初始化日志接收器
    ├── startCapturingEarlyInput() - 捕获早期输入
    │
    ▼
REPL.tsx
    │
    ├── 初始化 AppState
    ├── 加载 MCP 服务器
    ├── 恢复历史会话 (如有)
    │
    ▼
等待用户输入
```

---

## 4. Agent 运行机制与主循环

### 4.1 查询循环核心 (query.ts)

主循环是一个 `AsyncGenerator`，实现无限循环：

```typescript
async function* queryLoop(params) {
  let state = initialState;
  
  while (true) {
    const { messages, toolUseContext } = state;
    
    // ===== 1. 预处理阶段 =====
    // - 应用工具结果预算
    // - 微压缩 (microcompact)
    // - 上下文折叠
    // - 自动压缩
    
    // ===== 2. API 调用阶段 =====
    for await (const message of callModel({ messages, tools, ... })) {
      yield message;
      if (message.type === 'assistant' && hasToolUse(message)) {
        needsFollowUp = true;
        toolUseBlocks.push(...extractToolUse(message));
      }
    }
    
    // ===== 3. 停止条件判断 =====
    if (!needsFollowUp) {
      // 执行 Stop Hooks
      // Token 预算检查
      return { reason: 'completed' };
    }
    
    // ===== 4. 工具执行阶段 =====
    for await (const update of runTools(toolUseBlocks, ...)) {
      yield update.message;
      toolResults.push(update.message);
    }
    
    // ===== 5. 状态更新 =====
    state = {
      messages: [...messages, ...assistantMessages, ...toolResults],
      turnCount: turnCount + 1,
      ...
    };
  }
}
```

### 4.2 停止条件

| 条件 | 说明 | 恢复机制 |
|------|------|---------|
| `!needsFollowUp` | 助手无工具调用 | 正常结束 |
| `maxTurns` | 达到最大轮次 | 注入提示 |
| `signal.aborted` | 用户中断 | 返回中断消息 |
| `prompt_too_long` | 上下文超限 | 压缩后重试 |
| `max_output_tokens` | 输出超限 | 升级限制重试 |
| `stop_hook` | Hook 阻止 | 返回 hook 消息 |

### 4.3 子代理 (AgentTool)

```typescript
// Agent 定义
type AgentDefinition = {
  agentType: string          // 'general-purpose' | 'explore' | 'plan' | ...
  model?: 'sonnet' | 'opus' | 'haiku'
  permissionMode?: PermissionMode
  maxTurns?: number
  tools?: string[]
  background?: boolean       // 异步模式
  isolation?: 'worktree' | 'remote'  // 隔离模式
}

// 执行模式
1. 同步模式: 阻塞主循环，实时流式输出
2. 异步模式: 后台执行，通知驱动
3. 可中途转换: 同步 → 异步
```

---

## 5. 工具系统设计

### 5.1 Tool 接口

```typescript
type Tool<Input, Output> = {
  // 元数据
  name: string
  inputSchema: z.ZodType<Input>
  outputSchema?: z.ZodType<Output>
  
  // 核心方法
  call(args, context, canUseTool, ...): Promise<ToolResult<Output>>
  prompt(): Promise<string>  // 工具描述
  
  // 权限与并发
  checkPermissions(input, context): Promise<PermissionResult>
  isConcurrencySafe(input): boolean  // 是否可并发
  isReadOnly(input): boolean         // 是否只读
  
  // UI 渲染
  renderToolUseMessage(input): React.ReactNode
  renderToolResultMessage(output): React.ReactNode
}
```

### 5.2 内置工具列表

| 工具 | 功能 | 并发安全 | 只读 |
|------|------|:--------:|:----:|
| **Read** | 读取文件 (图片/PDF/Notebook) | ✅ | ✅ |
| **Edit** | 原地编辑文件 | ❌ | ❌ |
| **Write** | 创建/覆写文件 | ❌ | ❌ |
| **Glob** | 文件名模式搜索 | ✅ | ✅ |
| **Grep** | 内容正则搜索 | ✅ | ✅ |
| **Bash** | Shell 命令执行 | 视命令 | 视命令 |
| **WebFetch** | 获取 URL 内容 | ✅ | ✅ |
| **WebSearch** | 网络搜索 | ✅ | ✅ |
| **Agent** | 启动子代理 | ❌ | 视代理 |
| **TaskCreate/Update/List/Get** | 任务管理 | 视操作 | 视操作 |
| **Skill** | 执行技能 | ❌ | 视技能 |
| **mcp__*__(tool)** | MCP 工具 | 视工具 | 视工具 |

### 5.3 工具执行流程

```
模型返回 tool_use
       │
       ▼
validateInput() - Zod schema 验证
       │
       ▼
checkPermissions() - 权限检查
       │
       ├── allow → 执行
       ├── deny  → 返回拒绝消息
       └── ask   → 显示确认对话框
                         │
                         ▼
                   用户选择
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
   允许一次          允许始终           拒绝
       │                 │                 │
       ▼                 ▼                 ▼
   执行工具         添加规则          返回拒绝
       │
       ▼
tool.call()
       │
       ▼
返回 tool_result
```

---

## 6. 权限与安全模型

### 6.1 权限规则

```typescript
type PermissionRule = {
  source: 'userSettings' | 'projectSettings' | 'localSettings' | 'policySettings'
  ruleBehavior: 'allow' | 'deny' | 'ask'
  ruleValue: {
    toolName: string       // 'Bash', 'Edit', 'mcp__github__*', ...
    ruleContent?: string   // 'npm install', '.git/**', ...
  }
}

// 示例规则
'Bash(npm install)'     // 特定命令
'Bash(git *)'           // 前缀匹配
'Edit(.git/**)'         // 路径模式
'mcp__github__*'        // MCP 服务器级别
```

### 6.2 权限模式

| 模式 | 行为 | 风险 |
|------|------|------|
| `default` | 标准检查，询问用户 | 低 |
| `acceptEdits` | 自动接受工作目录编辑 | 中 |
| `bypassPermissions` | 绕过所有检查 | 高 |
| `dontAsk` | ask → deny | 低 |
| `plan` | 只读模式 | 低 |
| `auto` | AI 分类器决策 (ant-only) | 中 |

### 6.3 沙箱隔离

```typescript
// sandbox-adapter.ts
interface ISandboxManager {
  // 文件系统隔离
  getFsReadConfig(): { allowRead, denyRead }
  getFsWriteConfig(): { allowWrite, denyWrite }
  
  // 网络隔离
  getNetworkRestrictionConfig(): { allowedDomains, deniedDomains }
  
  // 命令包装
  wrapWithSandbox(command): Promise<string>
}

// 平台支持: macOS (seatbelt), Linux (bubblewrap), WSL2+
```

### 6.4 安全检查

```typescript
// 危险文件检测
const DANGEROUS_FILES = [
  '.gitconfig', '.bashrc', '.zshrc', '.mcp.json', '.claude.json'
]

// 危险命令检测
const DANGEROUS_BASH_PATTERNS = [
  'python', 'node', 'npx', 'eval', 'exec', 'sudo', 'ssh'
]

// 绕过免疫: 安全检查不受 bypassPermissions 影响
```

---

## 7. MCP 服务器集成

### 7.1 传输类型

| 类型 | 用途 | 特点 |
|------|------|------|
| `stdio` | 本地服务器 | stdin/stdout 通信 |
| `sse` | 远程 SSE | Server-Sent Events + OAuth |
| `http` | HTTP Streamable | MCP 2025-03-26 规范 |
| `ws` | WebSocket | 实时双向 |
| `sdk` | SDK 内置 | Agent SDK 集成 |

### 7.2 连接生命周期

```
加载配置 → 检查认证 → 创建传输层 → 连接服务器
                                          │
                                          ▼
                                    能力协商
                                          │
                                          ▼
                              注册 Elicitation 处理器
                                          │
                                          ▼
                                   获取工具/资源
                                          │
                                          ▼
                                   更新 AppState
                                          │
                           ┌──────────────┴──────────────┐
                           │                              │
                       正常运行                      连接断开
                           │                              │
                           ▼                              ▼
                      处理请求                      指数退避重连
```

### 7.3 工具命名

```typescript
// 格式: mcp__<server_name>__<tool_name>
// 示例: mcp__github__add_comment_to_issue

// 服务器级别权限
'mcp__github'           // 匹配 github 服务器的所有工具
'mcp__github__*'        // 同上
```

---

## 8. 配置与设置系统

### 8.1 设置来源优先级

```
插件设置基线
    ↓
用户设置 (~/.claude/settings.json)
    ↓
项目设置 (.claude/settings.json)
    ↓
本地设置 (.claude/settings.local.json)
    ↓
标志设置 (CLI --settings)
    ↓
策略设置 (企业 MDM) [最高优先级]
```

### 8.2 配置文件位置

| 文件 | 路径 | 用途 |
|------|------|------|
| 全局配置 | `~/.claude.json` | 会话状态、缓存 |
| 用户设置 | `~/.claude/settings.json` | 用户全局设置 |
| 项目设置 | `.claude/settings.json` | 项目共享设置 |
| 本地设置 | `.claude/settings.local.json` | 私有设置 |
| 托管设置 | `/etc/claude-code/managed-settings.json` | 企业策略 |

### 8.3 热更新机制

```typescript
// chokidar 文件监听
watcher = chokidar.watch(dirs, {
  persistent: true,
  ignoreInitial: true,
  awaitWriteFinish: { stabilityThreshold: 1000 }
});

// 变更时
handleChange(path) → resetSettingsCache() → settingsChanged.emit()
```

---

## 9. 会话管理与持久化

### 9.1 会话状态

```typescript
type State = {
  sessionId: string           // UUID
  parentSessionId: string     // 父会话 (lineage)
  projectRoot: string         // 项目根目录
  cwd: string                 // 当前工作目录
  startTime: number           // 启动时间
  lastInteractionTime: number // 最后交互
  isRemoteMode: boolean       // 远程模式
  sessionTrustAccepted: boolean
  ...
}
```

### 9.2 消息存储

```typescript
// JSONL 格式
// ~/.claude/projects/<project>/<session-id>.jsonl

type Entry = 
  | TranscriptMessage      // 用户/助手消息
  | SummaryMessage         // 压缩摘要
  | CustomTitleMessage     // 自定义标题
  | TaskSummaryMessage     // 任务摘要
  | FileHistorySnapshotMessage  // 文件历史快照
  | ...

// 消息链: parentUuid 形成链表
TranscriptMessage = {
  uuid: string
  parentUuid: string | null
  isSidechain: boolean
  ...message
}
```

### 9.3 自动压缩

```typescript
// 触发条件
const threshold = contextWindow - 13_000;  // 缓冲区
if (tokenCount >= threshold) {
  // 尝试会话记忆压缩
  // 失败则标准压缩
}

// 会话记忆提取
// - 最少 10,000 tokens 才初始化
// - 每 5,000 tokens 或 3 次工具调用更新
```

---

## 10. Hooks 钩子系统

### 10.1 钩子事件

| 事件 | 触发时机 | 可阻塞 |
|------|----------|:------:|
| `PreToolUse` | 工具执行前 | ✅ |
| `PostToolUse` | 工具执行后 | ❌ |
| `PermissionRequest` | 权限对话框显示时 | ✅ |
| `UserPromptSubmit` | 用户提交提示时 | ✅ |
| `SessionStart` | 会话开始时 | ❌ |
| `SessionEnd` | 会话结束时 | ❌ |
| `Stop` | 助手结束响应前 | ✅ |
| `SubagentStart/Stop` | 子代理生命周期 | 视事件 |
| `PreCompact/PostCompact` | 压缩前后 | ✅/❌ |

### 10.2 钩子类型

```typescript
type Hook = 
  | { type: 'command', command: string }      // Shell 命令
  | { type: 'prompt', prompt: string }        // LLM 验证
  | { type: 'agent', prompt: string }         // Agent 验证
  | { type: 'http', url: string }             // HTTP POST
  | { type: 'function', callback: Function }  // 内存回调
```

### 10.3 执行结果

```typescript
// Exit Code 语义
0   → 成功，stdout 可能注入对话
2   → 阻塞，阻止操作继续
其他 → 非阻塞错误

// JSON 输出控制
{
  "continue": false,           // 是否继续
  "decision": "block",         // approve/block
  "permissionDecision": "deny" // allow/deny/ask
}
```

---

## 11. API 通信层

### 11.1 认证优先级

```
1. --bare 模式: ANTHROPIC_API_KEY 或 apiKeyHelper
2. ANTHROPIC_AUTH_TOKEN 环境变量
3. CLAUDE_CODE_OAUTH_TOKEN 环境变量
4. apiKeyHelper 配置脚本
5. claude.ai OAuth tokens (订阅用户)
6. ANTHROPIC_API_KEY
7. macOS Keychain / 配置文件
```

### 11.2 多 Provider 支持

| Provider | 认证方式 | 环境变量 |
|----------|---------|---------|
| First Party | API Key / OAuth | `ANTHROPIC_API_KEY` |
| AWS Bedrock | AWS Credentials | `CLAUDE_CODE_USE_BEDROCK=true` |
| Google Vertex | GCP ADC | `CLAUDE_CODE_USE_VERTEX=true` |
| Azure Foundry | API Key / Azure AD | `CLAUDE_CODE_USE_FOUNDRY=true` |

### 11.3 流式处理

```typescript
// Stream<T> 类
class Stream<T> implements AsyncIterator<T> {
  private queue: T[] = []
  private isDone = false
  
  enqueue(value: T) { ... }
  done() { this.isDone = true }
  
  next(): Promise<IteratorResult<T>> { ... }
}

// 流式事件处理
for await (const part of stream) {
  switch (part.type) {
    case 'content_block_delta':
      yield deltaContent
    case 'message_delta':
      usage = updateUsage(usage, part.usage)
  }
}

// 空闲超时保护 (90s)
// 流式 → 非流式回退
```

### 11.4 重试策略

```typescript
// 重试条件
- 408 Request Timeout
- 429 Rate Limit (非订阅用户)
- 401 Unauthorized (清除缓存重试)
- 5xx Server Error
- 529 Overloaded (可能切换模型)

// 延迟计算
const delay = min(500 * 2^(attempt-1), 32000) + jitter
```

---

## 12. UI 与终端渲染

### 12.1 Ink 框架

```typescript
// 核心类
class Ink {
  private frontFrame: Frame;  // 当前显示
  private backFrame: Frame;   // 下一帧
  private container: FiberRoot;  // React 根节点
  
  // 渲染流程
  React 更新 → Reconciler → DOM 树 → Yoga 布局 → 渲染帧 → Diff → 终端输出
}

// 双缓冲 + 增量更新
const diff = log.render(prevFrame, frame);
writeDiffToTerminal(optimized);
```

### 12.2 Vim 模式

```typescript
// 状态机设计
type VimState =
  | { mode: 'INSERT', insertedText: string }
  | { mode: 'NORMAL', command: CommandState }

type CommandState =
  | { type: 'idle' }
  | { type: 'count', digits: string }
  | { type: 'operator', op: Operator, count: number }
  | { type: 'operatorTextObj', ... }
  | { type: 'find', ... }
  | ...

// 支持的操作符: d, c, y, p, x, r, >, <
// 支持的文本对象: w, W, b, B, (, ), [, ], {, }, ", ', `
```

### 12.3 键绑定系统

```typescript
// 上下文分层
type KeybindingContextName = 
  | 'Global' | 'Chat' | 'Autocomplete' | 'Confirmation' | 'Settings' | ...

// 默认绑定
{
  'ctrl+c': 'app:interrupt',
  'ctrl+d': 'app:exit',
  'enter': 'chat:submit',
  'escape': 'chat:cancel',
  ...
}

// 和弦支持
'ctrl+k' → 'ctrl+d' → 动作
```

---

## 13. 设计模式总结

### 13.1 核心模式

| 模式 | 应用场景 |
|------|---------|
| **异步生成器** | query.ts 主循环，流式消息处理 |
| **状态机** | QueryGuard 并发控制，Vim 模式 |
| **工厂模式** | buildTool() 工具创建 |
| **策略模式** | 权限检查，沙箱适配 |
| **观察者模式** | 钩子事件，设置变更通知 |
| **责任链** | 权限检查链，钩子执行链 |
| **对象池** | Ink 样式池，字符池 |

### 13.2 架构亮点

1. **类型驱动**: TypeScript 严格模式，Zod schema 验证
2. **流式优先**: AsyncGenerator 实现真正的流式处理
3. **安全优先**: 多层防御，沙箱隔离，绕过免疫
4. **可扩展性**: MCP 集成，钩子系统，插件机制
5. **企业友好**: MDM 配置，策略控制，审计日志

---

## 14. 关键文件索引

| 功能 | 文件 | 职责 |
|------|------|------|
| **入口** | `entrypoints/cli.tsx` | CLI 启动分发 |
| **主循环** | `query.ts` | 查询循环核心 |
| **UI 入口** | `REPL.tsx` | React 主组件 |
| **工具系统** | `tools.ts`, `Tool.ts` | 工具注册与接口 |
| **Agent** | `tools/Agent/AgentTool.tsx` | 子代理管理 |
| **权限** | `utils/permissions/` | 权限检查与沙箱 |
| **MCP** | `services/mcp/client.ts` | MCP 客户端 |
| **设置** | `utils/settings/` | 配置管理 |
| **会话** | `utils/sessionStorage.ts` | 持久化 |
| **钩子** | `utils/hooks/` | 生命周期钩子 |
| **API** | `services/api/claude.ts` | API 通信 |
| **认证** | `utils/auth.ts` | 认证管理 |
| **压缩** | `services/compact/` | 自动压缩 |
| **UI** | `ink/`, `components/` | 终端渲染 |

---

## 附录: 学习建议

1. **入门路径**:
   - 从 `entrypoints/cli.tsx` 开始理解启动流程
   - 阅读 `query.ts` 掌握核心循环
   - 研究 `tools/` 目录了解工具设计

2. **深入主题**:
   - 权限系统: `utils/permissions/` 全目录
   - MCP 集成: `services/mcp/client.ts`
   - 钩子系统: `utils/hooks/hooks.ts`

3. **调试技巧**:
   - 使用 `--debug` 标志启用调试日志
   - 查看 `~/.claude/logs/` 目录下的日志文件
   - 使用 `/doctor` 命令诊断配置问题

---

*本报告基于 Claude Code v2.1.88 源码分析生成*
