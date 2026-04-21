# 速查参考

> 一页搞定：核心概念、常用工具、关键源码入口。

## 核心概念速查

| 概念 | 一句话解释 | 详见 |
|------|-----------|------|
| **Agent Loop** | 用户输入 → 模型决策 → 工具执行 → 结果注入的循环，直到模型返回纯文本 | [第 2 章](./02-agent-loop.md) |
| **query()** | 核心循环的异步生成器实现，包含 7 个 continue site 处理不同恢复策略 | [2.4 节](./02-agent-loop.md#_24-query核心循环的实现) |
| **QueryEngine** | 会话级管理器，驱动 query() 并处理预算、权限、结构化输出 | [2.3 节](./02-agent-loop.md#_23-queryengine会话生命周期管理) |
| **Autocompact** | Token 使用量接近上下文窗口时的自动压缩机制（~93.5% 利用率触发） | [3.6 节](./03-context-engineering.md#_36-autocompact-自动全量压缩) |
| **Context Collapse** | 投影式只读上下文折叠，不修改原始消息，可安全回退 | [3.7 节](./03-context-engineering.md#_37-context-collapse-上下文折叠) |
| **CLAUDE.md** | 项目级指令文件，从 CWD 向上遍历目录树发现，支持多层级 | [3.2 节](./03-context-engineering.md#_32-系统提示词的构建) |
| **buildTool()** | 工具工厂函数，合并 TOOL_DEFAULTS（fail-closed 默认值）和工具定义 | [4.1 节](./04-tool-system.md#_41-tool-接口定义) |
| **MCP** | Model Context Protocol，外部工具扩展协议，支持 7 种传输机制 | [4.9 节](./04-tool-system.md#_49-mcp-工具集成) |
| **ToolSearch** | 延迟加载机制，50+ 工具中只按需加载，减少每次 API 调用的 prompt 体积 | [4.10 节](./04-tool-system.md#_410-工具搜索与延迟加载) |
| **search-and-replace** | FileEditTool 的编辑策略，要求 old_string 在文件中唯一匹配 | [第 10 章](./05-code-editing-strategy.md) |
| **纵深防御** | 7 层独立安全检查，任一层被绕过不致命 | [第 12 章](./11-permission-security.md) |
| **Plan 模式** | 两阶段执行：只读探索 → 用户审批 → 可写实施 | [8.6 节](./07-multi-agent.md#_86-plan-模式两阶段执行) |
| **协调器模式** | 主 Agent 只编排不执行，通过 Worker 完成实际任务 | [8.3 节](./07-multi-agent.md#_83-协调器模式coordinator) |
| **Hooks** | 事件驱动扩展机制，在工具执行生命周期的关键节点注入自定义逻辑 | [第 7 章](./06-hooks-extensibility.md) |

## 常用工具清单

### 文件操作

| 工具 | 只读 | 并发安全 | 说明 |
|------|:----:|:-------:|------|
| **Read** (FileReadTool) | ✅ | ✅ | 读取文件，支持行范围、PDF、图片 |
| **Write** (FileWriteTool) | ❌ | ❌ | 写入/创建文件 |
| **Edit** (FileEditTool) | ❌ | ❌ | search-and-replace 编辑，要求唯一匹配 |
| **NotebookEdit** | ❌ | ❌ | Jupyter Notebook 编辑 |

### 搜索与导航

| 工具 | 只读 | 并发安全 | 说明 |
|------|:----:|:-------:|------|
| **Glob** (GlobTool) | ✅ | ✅ | 文件名模式匹配搜索 |
| **Grep** (GrepTool) | ✅ | ✅ | 文件内容正则搜索（基于 ripgrep） |
| **ToolSearch** (ToolSearchTool) | ✅ | ✅ | 动态发现延迟加载的工具 |

### 执行与系统

| 工具 | 只读 | 并发安全 | 说明 |
|------|:----:|:-------:|------|
| **Bash** (BashTool) | ❌ | ❌ | 执行 Shell 命令，7 层安全验证 |
| **Agent** (AgentTool) | ❌ | ❌ | 派生子 Agent 执行独立任务 |
| **SendMessage** | ❌ | ❌ | 向已有 Agent 或队友发送消息 |
| **TaskStop** | ❌ | ❌ | 终止子 Agent |

### 模式控制

| 工具 | 说明 |
|------|------|
| **EnterPlanMode** | 进入 Plan 模式（只读探索阶段） |
| **ExitPlanMode** | 退出 Plan 模式并提交计划供审批 |

## 关键源码入口

| 模块 | 入口文件 | 行数 | 职责 |
|------|---------|------|------|
| **CLI 入口** | `src/main.tsx` | ~4,700 | Commander.js 参数解析，运行模式分发 |
| **Agent 循环** | `src/query.ts` | ~1,730 | 核心循环的异步生成器实现 |
| **会话管理** | `src/QueryEngine.ts` | ~1,160 | 对话生命周期管理 |
| **工具接口** | `src/Tool.ts` | ~200 | Tool 类型定义和 buildTool 工厂 |
| **系统提示词** | `src/constants/prompts.ts` | ~2,400 | 完整的系统提示词模板 |
| **权限系统** | `src/utils/permissions/` | ~多文件 | 多层权限检查和规则匹配 |
| **Bash 安全** | `src/tools/BashTool/bashSecurity.ts` | ~1,200 | 23 项静态安全验证器 |
| **上下文组装** | `src/context.ts` | ~190 | 系统/用户上下文构建 |
| **压缩服务** | `src/services/compact/` | ~多文件 | Autocompact、Snip、Context Collapse |
| **MCP 客户端** | `src/services/mcp/client.ts` | ~3,350 | MCP 连接管理和工具注册 |
| **Hooks 引擎** | `src/hooks/` | ~多文件 | Hook 事件分发和执行 |
| **多 Agent** | `src/coordinator/` | ~多文件 | 协调器模式实现 |
| **Swarm 后端** | `src/utils/swarm/backends/` | ~多文件 | Tmux/iTerm2/InProcess 执行后端 |

## 关键阈值与常量

| 常量 | 值 | 来源 | 用途 |
|------|---|------|------|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | autoCompact.ts | 自动压缩触发缓冲 |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | autoCompact.ts | 压缩熔断器阈值 |
| `CAPPED_DEFAULT_MAX_TOKENS` | 8,000 | context.ts | 默认输出 token 上限（节省 slot） |
| `ESCALATED_MAX_TOKENS` | 64,000 | context.ts | 截断后升级的输出上限 |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | autoCompact.ts | 压缩摘要预留输出空间 |
| `DEFAULT_MAX_RESULT_SIZE_CHARS` | 50,000 | toolLimits.ts | 工具结果最大字符数 |
| `MAX_TOOL_RESULT_TOKENS` | 100,000 | toolLimits.ts | 工具结果最大 token 数 |
| `DENIAL_LIMITS.maxConsecutive` | 3 | denialTracking.ts | 连续拒绝后回退到交互确认 |
| `DENIAL_LIMITS.maxTotal` | 20 | denialTracking.ts | 总拒绝数上限 |
| `WARNING_THRESHOLD` | 0.7 (70%) | rateLimitMessages.ts | 速率限制警告阈值 |
| `POST_MAX_RETRIES` | 10 | SSETransport.ts | POST 请求最大重试次数 |
| `RECONNECT_GIVE_UP_MS` | 600,000 (10min) | SSETransport.ts | SSE 重连放弃时间 |
| `LIVENESS_TIMEOUT_MS` | 45,000 | SSETransport.ts | 心跳超时（服务端每 15s 发送） |

---

返回：[快速入门](./quick-start.md) | [首页](/)
