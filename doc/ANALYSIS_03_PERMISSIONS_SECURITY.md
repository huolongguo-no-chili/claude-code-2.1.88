# Claude Code 权限和安全模型深度分析报告

## 目录
1. [概述](#1-概述)
2. [权限规则定义](#2-权限规则定义)
3. [权限检查流程](#3-权限检查流程)
4. [权限模式系统](#4-权限模式系统)
5. [沙箱隔离机制](#5-沙箱隔离机制)
6. [敏感操作保护](#6-敏感操作保护)
7. [用户确认机制](#7-用户确认机制)
8. [钩子系统安全](#8-钩子系统安全)
9. [安全架构图](#9-安全架构图)

---

## 1. 概述

Claude Code 实现了一个多层防御的权限和安全模型，主要包含以下核心组件：

### 核心架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      用户请求 (工具调用)                          │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    权限检查流程 (permissions.ts)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  规则匹配     │→│  模式判断     │→│  分类器评估   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      沙箱执行 (sandbox-adapter.ts)               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  文件系统隔离  │  │  网络隔离     │  │  进程隔离     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 权限规则定义

### 2.1 规则数据结构

权限规则定义在 `types/permissions.ts` 中：

```typescript
// 权限行为类型
type PermissionBehavior = 'allow' | 'deny' | 'ask'

// 规则值 - 指定工具和可选内容
type PermissionRuleValue = {
  toolName: string        // 工具名称，如 "Bash", "Edit", "Read"
  ruleContent?: string    // 可选的具体内容，如 "npm install:*"
}

// 完整的权限规则
type PermissionRule = {
  source: PermissionRuleSource    // 规则来源
  ruleBehavior: PermissionBehavior // 行为：allow/deny/ask
  ruleValue: PermissionRuleValue   // 规则值
}
```

### 2.2 规则来源层次

```typescript
type PermissionRuleSource =
  | 'userSettings'     // 用户全局设置 (~/.claude/settings.json)
  | 'projectSettings'  // 项目设置
  | 'localSettings'    // 本地设置
  | 'policySettings'   // 策略设置 (企业管控)
  | 'flagSettings'     // CLI 标志设置
  | 'cliArg'          // 命令行参数
  | 'command'         // 会话命令
  | 'session'         // 会话级别
```

### 2.3 规则解析

规则字符串格式为 `ToolName(content)`，支持转义括号：

```typescript
// 示例规则
'Bash'                        // 整个 Bash 工具
'Bash(npm install)'          // 特定命令
'Bash(npm run *:*)'          // 前缀匹配
'Edit(.git/**)'              // 路径模式
```

解析器 (`permissionRuleParser.ts`) 处理转义字符：
- `\(` 和 `\)` 用于在内容中包含括号
- 支持工具名称别名映射（如 `Task` → `Agent`）

---

## 3. 权限检查流程

### 3.1 主检查函数

核心检查函数 `hasPermissionsToUseTool` 在 `permissions.ts` 中实现：

```typescript
export const hasPermissionsToUseTool: CanUseToolFn = async (
  tool,
  input,
  context,
  assistantMessage,
  toolUseID,
): Promise<PermissionDecision>
```

### 3.2 检查流程顺序

```
步骤 1: 工具级别拒绝检查
├── 1a. 整个工具是否被拒绝规则阻止
├── 1b. 整个工具是否需要询问
├── 1c. 工具特定权限检查 (checkPermissions)
├── 1d. 工具实现拒绝检查
├── 1e. 用户交互要求检查
├── 1f. 内容特定询问规则
└── 1g. 安全检查 (绕过免疫)

步骤 2: 允许规则检查
├── 2a. bypassPermissions 模式检查
├── 2b. 整个工具允许规则检查
├── 2c. 工具特定允许规则检查
└── 2d. 沙箱自动允许检查

步骤 3: 模式转换
├── dontAsk 模式: 'ask' → 'deny'
└── auto 模式: 运行分类器评估

步骤 4: 用户交互
└── 显示权限请求对话框
```

### 3.3 规则匹配逻辑

```typescript
// 工具整体匹配检查
function toolMatchesRule(tool, rule): boolean {
  // 规则不能有内容才能匹配整个工具
  if (rule.ruleValue.ruleContent !== undefined) return false
  
  // 直接工具名匹配
  if (rule.ruleValue.toolName === toolName) return true
  
  // MCP 服务器级别权限: "mcp__server1" 匹配 "mcp__server1__tool1"
  // 支持通配符: "mcp__server1__*" 匹配服务器1的所有工具
}
```

---

## 4. 权限模式系统

### 4.1 模式类型定义

```typescript
// 外部可配置模式
type ExternalPermissionMode = 
  | 'acceptEdits'     // 自动接受编辑
  | 'bypassPermissions' // 绕过权限检查
  | 'default'         // 默认模式
  | 'dontAsk'         // 不询问模式
  | 'plan'            // 计划模式

// 内部模式 (包含 ant-only)
type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
```

### 4.2 模式行为说明

| 模式 | 行为 | 风险等级 |
|------|------|---------|
| `default` | 标准权限检查，询问用户 | 低 |
| `acceptEdits` | 自动接受工作目录内的编辑 | 中 |
| `bypassPermissions` | 绕过所有权限检查 | 高 |
| `dontAsk` | 将所有 'ask' 转为 'deny' | 低 |
| `plan` | 计划模式，只读操作 | 低 |
| `auto` | AI 分类器自动决策 (ant-only) | 中 |

### 4.3 Auto 模式分类器

Auto 模式使用 AI 分类器评估操作安全性：

```typescript
// 拒绝追踪限制
const DENIAL_LIMITS = {
  maxConsecutive: 3,  // 连续拒绝上限
  maxTotal: 20,       // 总拒绝上限
}

// 当达到限制时，回退到手动提示
function shouldFallbackToPrompting(state): boolean {
  return state.consecutiveDenials >= 3 || state.totalDenials >= 20
}
```

---

## 5. 沙箱隔离机制

### 5.1 沙箱架构

沙箱系统 (`sandbox-adapter.ts`) 包装 `@anthropic-ai/sandbox-runtime`：

```typescript
interface ISandboxManager {
  // 初始化和配置
  initialize(sandboxAskCallback?): Promise<void>
  isSandboxingEnabled(): boolean
  checkDependencies(): SandboxDependencyCheck
  
  // 文件系统隔离
  getFsReadConfig(): FsReadRestrictionConfig
  getFsWriteConfig(): FsWriteRestrictionConfig
  
  // 网络隔离
  getNetworkRestrictionConfig(): NetworkRestrictionConfig
  getAllowUnixSockets(): string[] | undefined
  
  // 命令包装
  wrapWithSandbox(command, binShell?, customConfig?): Promise<string>
  
  // 清理
  cleanupAfterCommand(): void
}
```

### 5.2 文件系统隔离

```typescript
// 转换设置到沙箱运行时配置
function convertToSandboxRuntimeConfig(settings): SandboxRuntimeConfig {
  // 允许写入的路径
  const allowWrite: string[] = ['.', getClaudeTempDir()]
  
  // 拒绝写入的路径 (设置文件)
  denyWrite.push(...settingsPaths)
  denyWrite.push(getManagedSettingsDropInDir())
  
  // 阻止 .claude/skills 写入
  denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
  
  // 防止 Git 仓库逃逸攻击
  // 阻止 HEAD, objects, refs, hooks, config 写入
  
  return {
    filesystem: { denyRead, allowRead, allowWrite, denyWrite },
    network: { allowedDomains, deniedDomains, ... },
    ...
  }
}
```

### 5.3 网络隔离

```typescript
// 从 WebFetch 规则提取允许的域名
for (const ruleString of permissions.allow || []) {
  const rule = permissionRuleValueFromString(ruleString)
  if (rule.toolName === WEB_FETCH_TOOL_NAME && 
      rule.ruleContent?.startsWith('domain:')) {
    allowedDomains.push(rule.ruleContent.substring('domain:'.length))
  }
}

// 企业策略限制
if (shouldAllowManagedSandboxDomainsOnly()) {
  // 仅使用 policySettings 中的域名
}
```

### 5.4 平台支持

```typescript
// 支持的平台: macOS, Linux, WSL2+
function isSupportedPlatform(): boolean {
  return BaseSandboxManager.isSupportedPlatform()
}

// WSL1 不支持
if (platform === 'wsl') {
  return 'sandbox.enabled is set but WSL1 is not supported (requires WSL2)'
}
```

---

## 6. 敏感操作保护

### 6.1 危险文件和目录

```typescript
// 危险文件
const DANGEROUS_FILES = [
  '.gitconfig', '.gitmodules',    // Git 配置
  '.bashrc', '.bash_profile',     // Shell 配置
  '.zshrc', '.zprofile',
  '.profile', '.ripgreprc',
  '.mcp.json', '.claude.json',    // MCP/Claude 配置
]

// 危险目录
const DANGEROUS_DIRECTORIES = [
  '.git', '.vscode', '.idea', '.claude',
]
```

### 6.2 路径安全检查

```typescript
function checkPathSafetyForAutoEdit(path): 
  | { safe: true } 
  | { safe: false; message: string; classifierApprovable: boolean } {
  
  // 1. 检查可疑 Windows 路径模式
  // - NTFS 备用数据流 (file.txt::$DATA)
  // - 8.3 短名称 (GIT~1)
  // - 长路径前缀 (\\?\C:\...)
  // - 尾随点/空格
  // - DOS 设备名
  
  // 2. 检查 Claude 配置文件
  if (isClaudeConfigFilePath(path)) {
    return { safe: false, classifierApprovable: true }
  }
  
  // 3. 检查危险文件/目录
  if (isDangerousFilePathToAutoEdit(path)) {
    return { safe: false, classifierApprovable: true }
  }
  
  return { safe: true }
}
```

### 6.3 危险 Bash 命令模式

```typescript
const DANGEROUS_BASH_PATTERNS = [
  // 解释器
  'python', 'python3', 'node', 'deno', 'ruby', 'perl', 'php', 'lua',
  
  // 包运行器
  'npx', 'bunx', 'npm run', 'yarn run', 'pnpm run', 'bun run',
  
  // Shell
  'bash', 'sh', 'zsh', 'fish',
  
  // 危险命令
  'eval', 'exec', 'env', 'xargs', 'sudo', 'ssh',
  
  // Ant-only: 额外风险命令
  'gh', 'curl', 'wget', 'git', 'kubectl', 'aws', 'gcloud',
]
```

### 6.4 PowerShell 危险命令

```typescript
const patterns = [
  // 嵌套 Shell
  'pwsh', 'powershell', 'cmd', 'wsl',
  
  // 代码执行
  'iex', 'invoke-expression', 'invoke-command',
  'start-process', 'start-job', 'start-threadjob',
  
  // .NET 逃逸
  'add-type', 'new-object',
]
```

---

## 7. 用户确认机制

### 7.1 权限请求流程

```
工具调用需要权限
       │
       ▼
检查规则和模式
       │
       ├── allow → 直接执行
       ├── deny  → 返回拒绝消息
       └── ask   → 显示权限对话框
                      │
                      ▼
              ┌───────────────────┐
              │  权限请求对话框     │
              │  ┌─────────────┐  │
              │  │ 命令描述     │  │
              │  │ 风险等级     │  │
              │  │ 选项列表     │  │
              │  └─────────────┘  │
              └───────────────────┘
                      │
           ┌─────────┼─────────┐
           ▼         ▼         ▼
        允许一次   允许始终   拒绝
```

### 7.2 权限更新类型

```typescript
type PermissionUpdate =
  | { type: 'addRules', destination, rules, behavior }
  | { type: 'replaceRules', destination, rules, behavior }
  | { type: 'removeRules', destination, rules, behavior }
  | { type: 'setMode', destination, mode }
  | { type: 'addDirectories', destination, directories }
  | { type: 'removeDirectories', destination, directories }
```

### 7.3 企业策略控制

```typescript
// 仅允许托管权限规则
function shouldAllowManagedPermissionRulesOnly(): boolean {
  return getSettingsForSource('policySettings')
    ?.allowManagedPermissionRulesOnly === true
}

// 仅允许托管沙箱域名
function shouldAllowManagedSandboxDomainsOnly(): boolean {
  return getSettingsForSource('policySettings')
    ?.sandbox?.network?.allowManagedDomainsOnly === true
}
```

---

## 8. 钩子系统安全

### 8.1 钩子类型

```typescript
type HookEvent =
  | 'PreToolUse'       // 工具使用前
  | 'PostToolUse'      // 工具使用后
  | 'PostToolUseFailure' // 工具失败后
  | 'UserPromptSubmit' // 用户提交提示时
  | 'SessionStart'     // 会话开始
  | 'SessionEnd'       // 会话结束
  | 'PermissionRequest' // 权限请求时
  | 'PermissionDenied' // 权限拒绝时
  | 'Notification'     // 通知时
  | 'Stop'             // 停止时
  | 'SubagentStart'    // 子代理开始
  | 'SubagentStop'     // 子代理停止
  | ...
```

### 8.2 钩子信任检查

```typescript
// 所有钩子都需要工作区信任
function shouldSkipHookDueToTrust(): boolean {
  const isInteractive = !getIsNonInteractiveSession()
  if (!isInteractive) return false  // SDK 模式隐式信任
  
  const hasTrust = checkHasTrustDialogAccepted()
  return !hasTrust  // 交互模式需要信任
}
```

### 8.3 HTTP 钩子安全

```typescript
// SSRF 防护
function isBlockedAddress(address): boolean {
  // 阻止的 IPv4 范围
  // 0.0.0.0/8, 10.0.0.0/8, 169.254.0.0/16 (云元数据)
  // 172.16.0.0/12, 192.168.0.0/16, 100.64.0.0/10
  
  // 允许: 127.0.0.0/8 (本地开发)
  
  // 阻止的 IPv6 范围
  // ::, fc00::/7, fe80::/10
  // 允许: ::1
}

// URL 白名单
if (policy.allowedUrls !== undefined) {
  const matched = policy.allowedUrls.some(p => urlMatchesPattern(hook.url, p))
  if (!matched) return { ok: false, error: 'URL not in allowlist' }
}

// 环境变量保护
function interpolateEnvVars(value, allowedEnvVars): string {
  // 仅解析白名单中的环境变量
  // 防止通过钩子配置泄露密钥
}
```

### 8.4 钩子输出处理

```typescript
type SyncHookJSONOutput = {
  continue?: boolean      // 是否继续
  suppressOutput?: boolean // 隐藏输出
  stopReason?: string     // 停止原因
  decision?: 'approve' | 'block'  // 决策
  systemMessage?: string  // 警告消息
  hookSpecificOutput?: {  // 钩子特定输出
    hookEventName: string
    permissionDecision?: 'allow' | 'deny' | 'ask'
    updatedInput?: Record<string, unknown>
    ...
  }
}
```

---

## 9. 安全架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Claude Code 安全架构                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         外层防御                                     │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐          │   │
│  │  │  设置来源验证  │  │  工作区信任    │  │  企业策略控制  │          │   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                       │
│                                    ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         权限层                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                    权限规则引擎                              │   │   │
│  │  │  allow ──────────────────────────────────────────────► 执行  │   │   │
│  │  │  deny ───────────────────────────────────────────────► 拒绝  │   │   │
│  │  │  ask ────────────────────────────────────────────────► 提示  │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                      │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐          │   │
│  │  │  Auto 模式     │  │  安全检查     │  │  危险模式检测  │          │   │
│  │  │  分类器        │  │  (绕过免疫)   │  │  (解释器/Shell)│          │   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                       │
│                                    ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         沙箱层                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                   sandbox-runtime                            │   │   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │   │
│  │  │  │ 文件系统隔离 │  │  网络隔离   │  │  进程隔离    │         │   │   │
│  │  │  │ (bubblewrap) │  │ (proxy)     │  │ (namespace)  │         │   │   │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘         │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                      │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐          │   │
│  │  │  设置文件保护  │  │  Git 仓库保护 │  │  临时目录隔离  │          │   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                       │
│                                    ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         钩子层                                       │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐          │   │
│  │  │  PreToolUse   │  │  权限决策     │  │  SSRF 防护    │          │   │
│  │  │  事件拦截      │  │  覆盖        │  │  (HTTP 钩子)  │          │   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 总结

Claude Code 的权限和安全模型采用了**多层防御**策略：

1. **规则层**：通过灵活的权限规则系统（allow/deny/ask）进行细粒度控制
2. **模式层**：提供多种操作模式适应不同安全需求
3. **沙箱层**：使用 OS 级隔离（bubblewrap/seatbelt）限制文件和网络访问
4. **检查层**：对敏感路径、危险命令进行专门检测
5. **钩子层**：允许企业自定义安全策略和审计

关键安全特性：
- **绕过免疫**：安全检查不受 bypassPermissions 模式影响
- **企业管控**：支持策略设置锁定权限配置
- **分类器保护**：Auto 模式有拒绝限制防止无限拒绝
- **路径规范化**：防止通过大小写、短名称、ADS 等绕过检查
