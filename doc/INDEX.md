# Claude Code 源码分析文档

> 版本: 2.1.88 | 分析日期: 2026-04-03

## 文档索引

### 入口文档

| 文件 | 说明 |
|------|------|
| [README_CN.md](./README_CN.md) | Claude Code 中文介绍 |
| [CLAUDE_CODE_ARCHITECTURE_REPORT.md](./CLAUDE_CODE_ARCHITECTURE_REPORT.md) | **综合架构报告** - 整体概览、模块索引、设计模式总结 |

### 模块详细分析

| 编号 | 文件 | 模块 | 主要内容 |
|------|------|------|---------|
| 01 | [ANALYSIS_01_MCP_INTEGRATION.md](./ANALYSIS_01_MCP_INTEGRATION.md) | MCP 集成 | MCP 协议实现、服务器生命周期、工具发现、OAuth 认证、Elicitation 机制 |
| 02 | [ANALYSIS_02_CONFIG_SETTINGS.md](./ANALYSIS_02_CONFIG_SETTINGS.md) | 配置系统 | 多层配置源、MDM 集成、热更新机制、环境变量处理 |
| 03 | [ANALYSIS_03_PERMISSIONS_SECURITY.md](./ANALYSIS_03_PERMISSIONS_SECURITY.md) | 权限安全 | 权限规则、沙箱隔离、敏感操作保护、安全架构 |
| 04 | [ANALYSIS_04_UI_TERMINAL_RENDERING.md](./ANALYSIS_04_UI_TERMINAL_RENDERING.md) | UI 渲染 | Ink 框架、Vim 模式状态机、键绑定系统、双缓冲渲染 |
| 05 | [ANALYSIS_05_TOOL_SYSTEM.md](./ANALYSIS_05_TOOL_SYSTEM.md) | 工具系统 | Tool 接口定义、工具注册发现、并发执行、权限检查 |
| 06 | [ANALYSIS_06_AGENT_RUNTIME.md](./ANALYSIS_06_AGENT_RUNTIME.md) | Agent 运行时 | 主循环实现、子代理系统、消息处理、压缩机制 |
| 07 | [ANALYSIS_07_SESSION_PERSISTENCE.md](./ANALYSIS_07_SESSION_PERSISTENCE.md) | 会话持久化 | 消息存储格式、会话恢复、历史管理、会话记忆 |
| 08 | [ANALYSIS_08_API_COMMUNICATION.md](./ANALYSIS_08_API_COMMUNICATION.md) | API 通信 | 多 Provider 认证、流式响应、模型管理、重试策略 |
| 09 | [ANALYSIS_09_HOOKS_SYSTEM.md](./ANALYSIS_09_HOOKS_SYSTEM.md) | 钩子系统 | 生命周期事件、钩子类型、执行引擎、安全机制 |

## 推荐阅读顺序

### 初学者
1. `README_CN.md` - 了解 Claude Code 是什么
2. `CLAUDE_CODE_ARCHITECTURE_REPORT.md` - 掌握整体架构
3. `ANALYSIS_06_AGENT_RUNTIME.md` - 理解核心运行机制
4. `ANALYSIS_05_TOOL_SYSTEM.md` - 了解工具如何工作

### 进阶开发者
1. `ANALYSIS_03_PERMISSIONS_SECURITY.md` - 安全模型设计
2. `ANALYSIS_01_MCP_INTEGRATION.md` - MCP 扩展机制
3. `ANALYSIS_09_HOOKS_SYSTEM.md` - 生命周期扩展
4. `ANALYSIS_02_CONFIG_SETTINGS.md` - 配置管理

### 前端开发者
1. `ANALYSIS_04_UI_TERMINAL_RENDERING.md` - 终端 UI 实现
2. `ANALYSIS_06_AGENT_RUNTIME.md` - React 状态管理

### 后端/架构师
1. `ANALYSIS_08_API_COMMUNICATION.md` - API 层设计
2. `ANALYSIS_07_SESSION_PERSISTENCE.md` - 数据持久化
3. `ANALYSIS_03_PERMISSIONS_SECURITY.md` - 安全架构

## 关键源码目录

```
src/
├── entrypoints/           # 入口点
│   ├── cli.tsx           # CLI 主入口
│   └── mcp.ts            # MCP 服务器模式
├── main.ts               # 主逻辑入口
├── REPL.tsx              # 主循环组件
├── query.ts              # 查询循环核心
├── tools/                # 工具实现
├── services/             # 服务层
│   ├── api/              # API 服务
│   ├── mcp/              # MCP 服务
│   └── compact/          # 压缩服务
├── utils/                # 工具函数
│   ├── permissions/      # 权限管理
│   ├── settings/         # 设置管理
│   └── hooks/            # 钩子系统
├── components/           # UI 组件
└── ink/                  # Ink 终端 UI
```

## 文档统计

| 指标 | 数值 |
|------|------|
| 文档总数 | 11 份 |
| 总大小 | ~226 KB |
| 涵盖模块 | 9 个核心模块 |
| 代码示例 | 200+ 个 |

---

*本分析基于 Claude Code v2.1.88 从 npm 发布包的 source map 中恢复的源码*
