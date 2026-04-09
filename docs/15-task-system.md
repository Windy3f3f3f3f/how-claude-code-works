# 15. 任务管理系统

> **一句话总结**：Claude Code 的任务系统（TodoV2）用**文件级存储 + 锁机制**实现了一个支持多 Agent 并发的轻量任务管理器，让 AI Agent 能拆解复杂工作、追踪进度、并在团队中协调分工。

## 为什么需要任务系统？

想象一个场景：你要求 Claude Code「重构整个认证模块」。这涉及十几个步骤——修改数据模型、更新 API 端点、调整前端组件、写测试、更新文档……如果没有任务管理，Agent 很可能丢失上下文、遗漏步骤，或者做了一半忘记还有什么没做。

任务系统解决的核心问题：

1. **复杂任务拆解** — 单个 Agent 将大任务分解为可追踪的小步骤
2. **进度可见性** — 用户在终端 UI 中实时看到每个步骤的状态
3. **多 Agent 协调** — 团队中的多个 Agent 共享任务列表、认领工作、避免重复

### 从 TodoV1 到 TodoV2 的演进

早期的 `TodoWriteTool` 是一个简单的单一 JSON 文件方案——所有待办事项写在一个列表里。这在单 Agent 场景下够用，但当多个 Agent 同时读写同一个 JSON 文件时，就会出现竞争条件和数据丢失。

TodoV2 做了一个关键的架构决策：**每个任务一个独立文件**。这让锁粒度从「整个列表」细化到了「单个任务」，是支撑多 Agent 并发的基础。

## 15.1 四个核心工具

任务系统通过四个工具暴露给模型，分工明确：

| 工具 | 职责 | 只读 |
|------|------|------|
| **TaskCreate** | 创建新任务 | 否 |
| **TaskGet** | 获取单个任务的完整信息 | 是 |
| **TaskList** | 列出所有任务摘要 | 是 |
| **TaskUpdate** | 更新状态、owner、依赖等 | 否 |

### 参数设计的巧思

```typescript
// TaskCreate 的输入参数
{
  subject: string       // "Fix authentication bug in login flow"
  description: string   // 详细描述
  activeForm?: string   // "Fixing authentication bug" — 用于 spinner 显示
  metadata?: Record<string, unknown>  // 可扩展的元数据
}
```

几个值得注意的设计：

- **subject 要求命令式**（如 "Fix bug" 而非 "Fixing bug"）— 系统提示词明确引导 LLM 用这种格式，因为它更适合作为标题显示
- **activeForm 是可选的进行时形式** — 当任务正在执行时，终端 spinner 显示 "Fixing authentication bug" 而非 "Fix authentication bug"，这种状态感的区分让 UI 更自然
- **metadata 支持 `_internal` 标记** — 设置 `metadata._internal = true` 的任务不会出现在 TaskList 结果中，允许系统创建用户不可见的内部任务

### 状态机

任务的生命周期是一个简单的状态机：

```
pending ──→ in_progress ──→ completed
                              │
              deleted ←───────┘ (特殊状态，直接删除文件)
```

- `pending`：刚创建，等待认领
- `in_progress`：正在执行
- `completed`：已完成
- `deleted`：不是真正的状态——调用 `deleteTask()` 直接删除文件，并清理其他任务中对它的引用

### 依赖追踪：blocks / blockedBy

任务间可以声明依赖关系：

```typescript
// "任务 2 被任务 1 阻塞" — 即必须先完成任务 1
TaskUpdate({ taskId: "2", addBlockedBy: ["1"] })
```

这在底层是**双向更新**（`blockTask` 函数）：

```typescript
// src/utils/tasks.ts
export async function blockTask(taskListId, fromTaskId, toTaskId) {
  // A blocks B → 同时更新两端
  // fromTask.blocks 加入 toTaskId
  // toTask.blockedBy 加入 fromTaskId
}
```

为什么要双向维护？因为 `TaskList` 需要快速判断一个任务是否可以被认领（检查 `blockedBy` 是否为空），而 `TaskGet` 需要展示一个任务阻塞了哪些下游任务（查看 `blocks`）。双向冗余避免了遍历全部任务来计算关系。

`TaskList` 还会**自动过滤已完成的 blocker**——如果任务 1 已经 completed，任务 2 的 `blockedBy` 在显示时不会包含它，避免误导模型以为仍然被阻塞。

## 15.2 文件级存储：为并发而生

这是任务系统最核心的设计决策，值得深入分析。

### 存储结构

```
~/.claude/tasks/
  └── {taskListId}/        # 每个会话/团队一个目录
      ├── .lock            # 目录级锁文件
      ├── .highwatermark   # 最高 ID 记录
      ├── 1.json           # 任务 1
      ├── 2.json           # 任务 2
      └── 3.json           # 任务 3
```

每个任务文件的内容：

```json
{
  "id": "1",
  "subject": "Fix authentication bug",
  "description": "The login endpoint returns 500...",
  "status": "in_progress",
  "owner": "teammate-1",
  "blocks": ["3"],
  "blockedBy": [],
  "metadata": {}
}
```

### 为什么一个文件一个任务？

这是对比 TodoV1 的关键改进。考虑多 Agent 场景：

**单文件方案的问题**：Agent A 读取 `tasks.json`，Agent B 也读取 `tasks.json`。A 修改任务 1 并写回，B 修改任务 2 并写回——B 的写入覆盖了 A 对任务 1 的修改。要解决这个问题，需要对整个文件加锁，意味着所有 Agent 的任务操作都是串行的。

**一文件一任务的优势**：Agent A 锁住 `1.json`，Agent B 锁住 `2.json`，两者可以并行操作互不干扰。只有在需要跨任务原子操作时（如创建新任务需要分配 ID），才需要使用目录级的 `.lock` 文件。

### TaskListId：谁共享同一个任务列表？

任务列表的隔离通过 `taskListId` 实现，解析时有 5 层优先级：

```typescript
// src/utils/tasks.ts
export function getTaskListId(): string {
  // 1. 显式指定（环境变量）
  if (process.env.CLAUDE_CODE_TASK_LIST_ID) return it
  // 2. 进程内队友 → 使用 leader 的 team name
  const teammateCtx = getTeammateContext()
  if (teammateCtx) return teammateCtx.teamName
  // 3. 进程式队友 → CLAUDE_CODE_TEAM_NAME
  // 4. Leader 创建的 team name
  // 5. 兜底 → session ID（独立会话）
  return getTeamName() || leaderTeamName || getSessionId()
}
```

这个设计确保了：
- **独立会话**各自隔离（用 session ID）
- **团队中的所有成员**（无论是进程内还是进程间）共享同一个任务列表（用 team name）
- **外部工具**可以通过环境变量强制指定

### 高水位标记：防止 ID 重用

任务 ID 是自增整数（"1", "2", "3"...），而非 UUID。这是有意的选择：

- **可读性**：在对话中 "#1" 比 "a7f3b2c1-..." 容易引用
- **顺序性**：系统提示词建议模型按 ID 顺序处理任务，因为早期任务往往为后续任务建立上下文

但自增 ID 遇到删除时有问题：如果任务 3 被删除后创建新任务，新任务不应该再次获得 ID "3"——这会让对话中之前引用的 "#3" 产生歧义。

解决方案是 `.highwatermark` 文件：

```typescript
// 删除任务时更新高水位
export async function deleteTask(taskListId, taskId) {
  const numericId = parseInt(taskId, 10)
  const currentMark = await readHighWaterMark(taskListId)
  if (numericId > currentMark) {
    await writeHighWaterMark(taskListId, numericId)
  }
  // ... 删除文件
}

// 创建任务时同时参考文件和高水位
async function findHighestTaskId(taskListId) {
  const [fromFiles, fromMark] = await Promise.all([
    findHighestTaskIdFromFiles(taskListId),
    readHighWaterMark(taskListId),
  ])
  return Math.max(fromFiles, fromMark)
}
```

即使所有任务文件都被删除，高水位仍然记录着历史最大 ID，新任务从它之后开始编号。

### 锁策略：两种粒度

系统使用 `proper-lockfile` 库，配置了相当激进的重试策略：

```typescript
// 为 ~10+ 个并发 swarm agent 设计
const LOCK_OPTIONS = {
  retries: {
    retries: 30,       // 最多重试 30 次
    minTimeout: 5,     // 最短 5ms
    maxTimeout: 100,   // 最长 100ms
  },
}
// 总等待时间约 2.6 秒——足够处理 10 路竞争
```

两种锁粒度服务于不同场景：

| 锁粒度 | 锁对象 | 使用场景 |
|--------|--------|----------|
| **任务级** | `{taskId}.json` | 更新单个任务（如修改状态、设置 owner） |
| **目录级** | `.lock` | 需要跨任务原子操作（如创建新任务分配 ID、带忙碌检查的认领） |

特别值得一提的是 `claimTaskWithBusyCheck`——它用目录级锁来**原子地**执行「检查 agent 是否空闲 + 认领任务」两步操作。如果用任务级锁，两个 agent 可能同时通过忙碌检查然后都认领成功，破坏了「一个 agent 同时只做一件事」的约束。

## 15.3 实时 UI：三层变更检测

任务状态的变化需要实时反映在终端 UI 中。Claude Code 用了**三层检测机制**来确保不遗漏任何更新：

### 第一层：文件系统事件（fs.watch）

```typescript
// src/hooks/useTasksV2.ts
#rewatch(dir: string): void {
  this.#watcher = watch(dir, this.#debouncedFetch)
  this.#watcher.unref()  // 不阻止进程退出
}
```

最快的通知方式——操作系统的文件系统事件。加了 50ms 去抖，因为一次任务操作可能触发多个文件事件（如修改任务文件 + 更新高水位）。

**局限**：`fs.watch` 不是 100% 可靠（不同操作系统/文件系统行为不同），而且如果任务目录还不存在时无法监听。

### 第二层：进程内信号（onTasksUpdated）

```typescript
// src/utils/tasks.ts 中，每次写操作后调用
notifyTasksUpdated()

// useTasksV2.ts 中订阅
this.#unsubscribeTasksUpdated = onTasksUpdated(this.#debouncedFetch)
```

当同一个进程内的代码修改了任务（如 Agent 自己创建的任务），通过内存中的信号直接通知 UI，不依赖文件系统。

**覆盖场景**：同进程的即时更新，零延迟。

### 第三层：轮询兜底（5 秒间隔）

```typescript
// 只有存在未完成任务时才轮询
if (hasIncomplete) {
  this.#pollTimer = setTimeout(this.#debouncedFetch, FALLBACK_POLL_MS)
  this.#pollTimer.unref()
}
```

终极兜底——每 5 秒重新读取一次任务列表。主要覆盖**跨进程更新**场景（如另一个 tmux 窗口中的 Agent 修改了任务），以及 fs.watch 不可靠的边缘情况。

**优化**：只在有未完成任务时才轮询。所有任务完成后停止轮询，避免不必要的 I/O。

### 为什么需要三层？

每一层覆盖不同的失败模式：

| 层 | 覆盖场景 | 延迟 | 可靠性 |
|----|----------|------|--------|
| fs.watch | 同机器文件变更 | ~50ms | 中（平台差异） |
| 进程内信号 | 同进程操作 | 即时 | 高 |
| 轮询 | 跨进程、fs.watch 失效 | ≤5s | 高 |

三层组合的结果是：正常情况下变更几乎即时可见，极端情况下最多延迟 5 秒。

### Singleton Store 模式

```typescript
let _store: TasksV2Store | null = null
function getStore(): TasksV2Store {
  return (_store ??= new TasksV2Store())
}
```

所有 React 组件共享同一个 `TasksV2Store` 实例。为什么不让每个组件各自创建 watcher？

源码注释说得很直接：「Spinner mounts/unmounts every turn — per-hook watchers caused constant watch/unwatch churn.」Spinner 组件在每一轮对话中都会挂载和卸载，如果它自己维护 watcher，就会不停地创建和销毁文件监听——既浪费资源，又可能错过卸载和重新挂载之间的事件。

Singleton 模式让 watcher 的生命周期与「是否有人在看」解耦。REPL 组件始终挂载，保持至少一个订阅者存在，Singleton 就不会被销毁。

### 自动隐藏与重置

任务全部完成后的行为很优雅：

1. 检测到所有任务变为 `completed`
2. 等待 5 秒（`HIDE_DELAY_MS`）——给用户时间看到完成状态
3. 再次确认仍然全部完成（防止新任务在等待期间创建）
4. 调用 `resetTaskList()` 清空任务文件
5. UI 自动折叠任务面板

这个 5 秒延迟是个细节但很重要——如果任务完成后立即消失，用户无法确认是否真的都做完了。

### 任务显示优先级

终端空间有限（最多显示约 10 条任务），`TaskListV2.tsx` 用优先级排序决定哪些任务可见：

1. **最近完成的**（30 秒内）— 让用户看到刚完成的成果
2. **进行中的** — 当前正在做什么
3. **待办（未阻塞）** — 接下来可以做什么
4. **待办（被阻塞）** — 需要等待的
5. **更早完成的** — 最低优先级

超出显示限制的任务用摘要代替：「... +2 in progress, 3 pending, 1 completed」。

## 15.4 多 Agent 协调

> 更多多 Agent 架构细节请参考 [第 7 章：多 Agent 架构](07-multi-agent.md)。

任务系统在多 Agent 场景下展现出最大的设计深度。

### 共享任务列表

通过 `taskListId` 解析机制（见 15.2），团队中的所有成员——无论是进程内队友（in-process teammates）还是进程间队友（tmux/iTerm2）——都指向同一个任务目录。Leader 创建的任务，teammate 立刻可见。

### 自动 Ownership

当一个 Agent 将任务标记为 `in_progress` 时，如果没有显式指定 owner，系统自动分配：

```typescript
// TaskUpdateTool.ts
if (isAgentSwarmsEnabled() && statusChanged && newStatus === 'in_progress') {
  if (!input.owner && context.agentName) {
    updates.owner = context.agentName
  }
}
```

这避免了一个常见的遗忘——Agent 开始做任务但忘记声明自己是 owner，导致其他 Agent 重复认领。

### Mailbox 通知

当任务的 owner 变更时，新 owner 会通过 mailbox 收到通知：

```typescript
// 通知内容包含完整的任务上下文
{
  taskId, subject, description,
  assignedBy: context.agentName,
  timestamp: new Date().toISOString()
}
```

这让被分配任务的 Agent 不需要主动轮询就能知道有新工作，减少了不必要的 TaskList 调用。

### 忙碌检测与原子认领

`claimTask` 函数支持一个重要的选项——`checkAgentBusy`：

```typescript
// 原子地检查 + 认领，防止 TOCTOU 竞态
async function claimTaskWithBusyCheck(taskListId, taskId, claimantAgentId) {
  // 获取目录级锁（不是任务级！）
  release = await lockfile.lock(lockPath, LOCK_OPTIONS)
  
  // 在锁内检查该 agent 是否还有未完成的任务
  const allTasks = await listTasks(taskListId)
  const busyTasks = allTasks.filter(
    t => t.owner === claimantAgentId && t.status !== 'completed'
  )
  if (busyTasks.length > 0) {
    return { success: false, reason: 'agent_busy', busyWithTasks: ... }
  }
  
  // 检查通过，在锁内完成认领
  await updateTaskUnsafe(taskListId, taskId, { owner: claimantAgentId })
}
```

这里使用**目录级锁**而非任务级锁的原因：忙碌检查需要扫描所有任务的 owner 字段，如果只锁住目标任务文件，另一个 Agent 可能在扫描期间修改其他任务，导致检查结果不准确（经典的 TOCTOU 问题）。

### 退出清理

当一个 teammate 退出时，`unassignTeammateTasks` 会释放它持有的所有未完成任务：

```typescript
export async function unassignTeammateTasks(taskListId, agentId) {
  const tasks = await listTasks(taskListId)
  for (const task of tasks) {
    if (task.owner === agentId && task.status !== 'completed') {
      await updateTask(taskListId, task.id, {
        owner: undefined,
        status: 'pending',  // 回到 pending，让其他 agent 认领
      })
    }
  }
}
```

这防止了「僵尸任务」——一个 Agent 崩溃或被终止后，它正在做的任务不会永远卡在 `in_progress` 状态。

## 15.5 验证提醒（Verification Nudge）

这是一个巧妙的质量保证机制：

```typescript
// TaskUpdateTool.ts — 简化后的逻辑
if (allTasksCompleted && totalTasks >= 3 && !hasVerificationTask) {
  result.verificationNudgeNeeded = true
  // → 提示模型生成一个独立的验证 Agent
}
```

**触发条件**：
1. 当前是主线程 Agent（不是子 Agent）
2. 所有任务都已完成
3. 任务总数 ≥ 3
4. 没有任何 subject 包含 "verif" 的任务

**设计意图**：当 Agent 完成了一系列任务后，提醒它生成一个**独立的验证 Agent** 来检查工作质量。关键是验证者必须是独立 Agent——如果让同一个 Agent 自己验证自己的工作，它倾向于确认自己的实现是正确的（一种 AI 版本的确认偏误）。

这个功能通过 feature flag 双重控制（`VERIFICATION_AGENT` + `tengu_hive_evidence`），属于渐进发布的实验性功能。

## 15.6 Hook 集成

> 更多 Hook 系统细节请参考 [第 6 章：Hooks 与可扩展性](06-hooks-extensibility.md)。

任务系统在两个生命周期节点触发 Hook：

| 事件 | 触发时机 | 可阻塞？ |
|------|----------|----------|
| `TaskCreated` | 任务创建后 | 是（exit code 2） |
| `TaskCompleted` | 任务标记完成时 | 是（exit code 2） |

**阻塞型 Hook** 的应用场景：
- **合规检查**：在任务完成前验证是否满足某些条件
- **外部同步**：将任务状态同步到 Jira/Linear 等外部系统，失败时阻止状态变更
- **自动化流程**：任务创建时触发 CI/CD 流程

如果 `TaskCreated` Hook 返回阻塞错误，系统会**回滚**——删除刚创建的任务文件，并将错误信息返回给模型。

## 15.7 系统提示词中的任务引导

### 条件启用

任务工具不是在所有场景下都可用：

```typescript
export function isTodoV2Enabled(): boolean {
  // SDK 用户可以通过环境变量强制启用
  if (isEnvTruthy(process.env.CLAUDE_CODE_ENABLE_TASKS)) return true
  // 默认：交互模式启用，非交互模式（SDK/CI）禁用
  return !getIsNonInteractiveSession()
}
```

在非交互模式下默认禁用的原因：SDK 用户通常有自己的任务管理逻辑，不需要 Claude Code 内置的任务系统。但提供了 `CLAUDE_CODE_ENABLE_TASKS` 环境变量作为显式启用的入口。

### 提示词策略

系统提示词对任务工具的使用给出了精确的引导：

**何时创建任务**：
- 复杂的多步骤任务（3 个以上不同步骤）
- 用户提供了多个任务（编号列表或逗号分隔）
- 非平凡的复杂任务

**何时不用**：
- 单一直接的任务
- 可以在 3 步以内完成的简单任务
- 纯对话或信息查询

**关键行为引导**：
- 开始工作前标记 `in_progress`（不是开始后）
- 完成后**立即**标记 `completed`（不要批量标记）
- 只有**完全完成**才标记——测试失败、实现不完整、有未解决的错误都不算完成

**多 Agent 模式额外引导**：
- 按 ID 顺序处理任务（早期任务往往为后续建立上下文）
- 完成一个任务后调用 `TaskList` 获取下一个
- 任务描述要足够详细，让其他 Agent 也能理解和执行

## 小结

Claude Code 的任务系统看似是一个简单的待办列表，实际上是一个为多 Agent 并发协调而设计的分布式任务管理器。它的核心设计思路值得总结：

| 设计决策 | 原因 |
|----------|------|
| 文件级存储（一任务一文件） | 细粒度锁，支持多 Agent 并发 |
| 高水位标记 | 防止删除后 ID 重用，保持引用一致性 |
| 三层变更检测 | 覆盖进程内、跨进程、平台差异等所有场景 |
| 双向依赖追踪 | 快速判断任务是否可认领，同时展示阻塞关系 |
| Singleton Store | 避免 Spinner 挂载/卸载导致的 watcher 抖动 |
| 原子认领（目录级锁） | 防止多 Agent 同时通过忙碌检查的竞态 |
| 验证提醒 | 防止 Agent 自我验证，鼓励独立验证 |
| 5 秒延迟隐藏 | 给用户确认全部完成的时间窗口 |
