# Quick Reference

> All in one page: core concepts, common tools, key source code entry points.

## Core Concepts Quick Reference

| Concept | One-Sentence Explanation | Details |
|---------|--------------------------|---------|
| **Agent Loop** | A cycle of user input → model decision → tool execution → result injection, until the model returns plain text | [Chapter 2](/en/docs/02-agent-loop.md) |
| **query()** | The async generator implementation of the core loop, containing 7 continue sites handling different resume strategies | [Section 2.4](/en/docs/02-agent-loop.md#_24-query核心循环的实现) |
| **QueryEngine** | A session-level manager that drives query() and handles budgets, permissions, and structured output | [Section 2.3](/en/docs/02-agent-loop.md#_23-queryengine会话生命周期管理) |
| **Autocompact** | An automatic compaction mechanism triggered when token usage approaches the context window (~93.5% utilization) | [Section 3.6](/en/docs/03-context-engineering.md#_36-autocompact-自动全量压缩) |
| **Context Collapse** | A projection-based read-only context folding mechanism that doesn't modify original messages and can safely roll back | [Section 3.7](/en/docs/03-context-engineering.md#_37-context-collapse-上下文折叠) |
| **CLAUDE.md** | A project-level instruction file, discovered by traversing the directory tree upward from CWD, supporting multiple levels | [Section 3.2](/en/docs/03-context-engineering.md#_32-系统提示词的构建) |
| **buildTool()** | A tool factory function that merges TOOL_DEFAULTS (fail-closed defaults) with tool definitions | [Section 4.1](/en/docs/04-tool-system.md#_41-tool-接口定义) |
| **MCP** | Model Context Protocol, an external tool extension protocol supporting 7 transport mechanisms | [Section 4.9](/en/docs/04-tool-system.md#_49-mcp-工具集成) |
| **ToolSearch** | A lazy-loading mechanism that loads only needed tools on demand from 50+, reducing prompt size per API call | [Section 4.10](/en/docs/04-tool-system.md#_410-工具搜索与延迟加载) |
| **search-and-replace** | The editing strategy of FileEditTool, requiring old_string to match uniquely within the file | [Chapter 10](/en/docs/05-code-editing-strategy.md) |
| **Defense in Depth** | 7 independent security check layers, where bypassing any single layer is not fatal | [Chapter 12](/en/docs/11-permission-security.md) |
| **Plan Mode** | Two-phase execution: read-only exploration → user approval → writable implementation | [Section 8.6](/en/docs/07-multi-agent.md#_86-plan-mode-two-phase-execution) |
| **Coordinator Mode** | The main Agent only orchestrates without executing, completing actual tasks through Workers | [Section 8.3](/en/docs/07-multi-agent.md#_83-coordinator-mode) |
| **Hooks** | An event-driven extension mechanism that injects custom logic at key points in the tool execution lifecycle | [Chapter 7](/en/docs/06-hooks-extensibility.md) |

## Common Tools List

### File Operations

| Tool | Read-Only | Concurrency-Safe | Description |
|------|:---------:|:----------------:|-------------|
| **Read** (FileReadTool) | ✅ | ✅ | Read files, supports line ranges, PDF, images |
| **Write** (FileWriteTool) | ❌ | ❌ | Write/create files |
| **Edit** (FileEditTool) | ❌ | ❌ | search-and-replace editing, requires unique match |
| **NotebookEdit** | ❌ | ❌ | Jupyter Notebook editing |

### Search and Navigation

| Tool | Read-Only | Concurrency-Safe | Description |
|------|:---------:|:----------------:|-------------|
| **Glob** (GlobTool) | ✅ | ✅ | Filename pattern matching search |
| **Grep** (GrepTool) | ✅ | ✅ | File content regex search (based on ripgrep) |
| **ToolSearch** (ToolSearchTool) | ✅ | ✅ | Dynamically discover lazily-loaded tools |

### Execution and System

| Tool | Read-Only | Concurrency-Safe | Description |
|------|:---------:|:----------------:|-------------|
| **Bash** (BashTool) | ❌ | ❌ | Execute shell commands, 7-layer security validation |
| **Agent** (AgentTool) | ❌ | ❌ | Spawn sub-Agents to execute independent tasks |
| **SendMessage** | ❌ | ❌ | Send messages to existing Agents or teammates |
| **TaskStop** | ❌ | ❌ | Terminate a sub-Agent |

### Mode Control

| Tool | Description |
|------|-------------|
| **EnterPlanMode** | Enter Plan mode (read-only exploration phase) |
| **ExitPlanMode** | Exit Plan mode and submit the plan for approval |

## Key Source Code Entry Points

| Module | Entry File | Lines | Responsibility |
|--------|-----------|-------|----------------|
| **CLI Entry** | `src/main.tsx` | ~4,700 | Commander.js argument parsing, run mode dispatching |
| **Agent Loop** | `src/query.ts` | ~1,730 | Async generator implementation of the core loop |
| **Session Management** | `src/QueryEngine.ts` | ~1,160 | Conversation lifecycle management |
| **Tool Interface** | `src/Tool.ts` | ~200 | Tool type definitions and buildTool factory |
| **System Prompts** | `src/constants/prompts.ts` | ~2,400 | Complete system prompt templates |
| **Permission System** | `src/utils/permissions/` | ~multiple files | Multi-layer permission checks and rule matching |
| **Bash Security** | `src/tools/BashTool/bashSecurity.ts` | ~1,200 | 23 static security validators |
| **Context Assembly** | `src/context.ts` | ~190 | System/user context construction |
| **Compaction Service** | `src/services/compact/` | ~multiple files | Autocompact, Snip, Context Collapse |
| **MCP Client** | `src/services/mcp/client.ts` | ~3,350 | MCP connection management and tool registration |
| **Hooks Engine** | `src/hooks/` | ~multiple files | Hook event dispatching and execution |
| **Multi-Agent** | `src/coordinator/` | ~multiple files | Coordinator mode implementation |
| **Swarm Backend** | `src/utils/swarm/backends/` | ~multiple files | Tmux/iTerm2/InProcess execution backends |

## Key Thresholds and Constants

| Constant | Value | Source | Purpose |
|----------|-------|--------|---------|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | autoCompact.ts | Auto-compaction trigger buffer |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | autoCompact.ts | Compaction circuit breaker threshold |
| `CAPPED_DEFAULT_MAX_TOKENS` | 8,000 | context.ts | Default output token cap (saves slots) |
| `ESCALATED_MAX_TOKENS` | 64,000 | context.ts | Escalated output cap after truncation |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | autoCompact.ts | Reserved output space for compaction summary |
| `DEFAULT_MAX_RESULT_SIZE_CHARS` | 50,000 | toolLimits.ts | Maximum characters for tool results |
| `MAX_TOOL_RESULT_TOKENS` | 100,000 | toolLimits.ts | Maximum tokens for tool results |
| `DENIAL_LIMITS.maxConsecutive` | 3 | denialTracking.ts | Fall back to interactive confirmation after consecutive denials |
| `DENIAL_LIMITS.maxTotal` | 20 | denialTracking.ts | Upper limit on total denials |
| `WARNING_THRESHOLD` | 0.7 (70%) | rateLimitMessages.ts | Rate limit warning threshold |
| `POST_MAX_RETRIES` | 10 | SSETransport.ts | Maximum retry count for POST requests |
| `RECONNECT_GIVE_UP_MS` | 600,000 (10min) | SSETransport.ts | SSE reconnection give-up time |
| `LIVENESS_TIMEOUT_MS` | 45,000 | SSETransport.ts | Heartbeat timeout (server sends every 15s) |

---

Back to: [Quick Start](/en/docs/quick-start.md) | [Home](/)
