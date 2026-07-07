# 第 19 章：Dynamic Workflows——用确定性脚本指挥 agent 舰队

> 第 8 章讲过 Claude Code 的多 Agent：主 Agent 派子 Agent、协调器分派 worker、Swarm 点对点。但那几种编排有个共同点——谁来派、何时派、何时收，都是模型在推理里临场决定的。碰到真正的大活（把一个改动铺到几十个文件、审一整条依赖链、对一大批候选逐一核实），你想要的不是"让模型边想边调度"，而是一套确定性的编排：一段脚本一次性铺开几十到几百个 subagent，带并发上限、token 预算、断点续跑。这就是 Dynamic Workflows，触发词 `ultracode`。
>
> 这一章的主证据是 Workflow 这个工具的描述——那是一段注入主模型、教它"怎么写一个 workflow 脚本"的说明。它就在运行时注入给模型的工具集里，其中的关键句子（并发上限、item 上限、agent 总数上限的原话）在 2.1.201 的二进制里能逐字对上，所以我们手里这份就是它的原文。加上二进制里泄露的运行时常量和一族遥测事件，这一章能落到相当实的地方；至于某一次 workflow 具体怎么分支收敛，那写在脚本里、由模型现写，属于拿不到的部分，后面会讲清楚。

## 19.1 两种编排：模型当协调器，还是脚本当协调器

先接上第 8 章。那一章的三种多 Agent 模式——子 Agent、协调器、Swarm——本质都是模型驱动的编排：由主模型或协调器在每一轮推理里决定"现在派谁去、拿到结果后怎么综合"。协调器那节讲的"提示词设计精要"，讲的就是怎么用提示词引导模型把这个调度judgment 做好。

Dynamic Workflows 换了个思路：把那个"当协调器的模型"换成一段确定性的 JS 脚本。fan-out 出去多少个 agent、它们的结果怎么串、什么时候收敛，全写死在脚本的控制流里，不靠模型每轮临场拍板。而底下那些原语——一个 subagent 怎么派生、异步任务怎么用 `<task-notification>` 通知回来、并行写文件怎么用 worktree 隔离——原封不动复用第 8 章那套（见[第 8 章](07-multi-agent.md) 8.2、8.3）。

跟第 17 章对照，两者正好是两条轴。`/loop` 是时间轴上的串行续跑：同一个主模型隔一段时间回来再干一轮。Workflow 是空间轴上的并行 fan-out：一段脚本此刻就规划出几十上百个 agent，由运行时按并发上限成批地并行跑。它们共享同一套哲学——上层用脚本或提示词表达"要干什么"，下层由运行时兜底"怎么调度、怎么保住运行状态"——但不是一个东西，也不在一条轴上。

这个"脚本当协调器"的设计，顺带解释了逆向它的边界：能拿到的是那段教模型写脚本的工具描述、以及每个被派生 agent 的请求；拿不到的是某一次 run 的确切分支逻辑——因为它就在那次模型现写的脚本里，不是一段固定的提示词。

## 19.2 一段 workflow 脚本长什么样

Workflow 工具做的事，是让主模型现写一段 JS 脚本，交给运行时确定性地执行。脚本必须以一个 `meta` 声明开头——纯字面量，写清楚名字、一句话描述、有哪些 phase（这几样会显示在权限弹窗和 `/workflows` 面板里）：

```javascript
export const meta = {
  name: 'review-changes',
  description: 'Review changed files across dimensions, verify each finding',
  phases: [{ title: 'Review' }, { title: 'Verify' }],
}
```

`meta` 之后是脚本体，用一组内置 hook 来编排：

- `agent(prompt, opts?)`——派生一个 subagent，返回它的结果。不带 schema 时返回它的终文本；带 `schema` 时返回一个校验过的对象（下节讲）。`opts` 能指定它的 label、归属哪个 phase、用哪个模型、多高的 reasoning、要不要 worktree 隔离、用哪种 agent 类型。
- `pipeline(items, stage1, stage2, …)`——让每个 item 独立穿过一串 stage。
- `parallel(thunks)`——并发跑一批任务，等它们全部完成。
- `phase(title)` 分组、`log(msg)` 往面板打进度、`budget` 查 token 预算、`workflow(nameOrRef, args?)` 内联跑另一个 workflow——可以是存好的 workflow，也可以是一个脚本路径；子 workflow 共用这一 run 的并发上限、agent 计数、中止信号和 token 预算，而且只准嵌套一层。

其中 `agent()` 那个 `agentType` 也值得一提：它跟 Agent 工具走同一个注册表解析，还能跟 `schema` 组合——于是你能派一个自定义类型的子 agent、又强制它返回结构化结果。脚本是纯 JS 不是 TS（写类型注解会解析失败），而且 `Date.now()`、`Math.random()`、不带参的 `new Date()` 都被禁用——因为它们会让"断点续跑"（19.6）算出不一样的结果。这几条限制本身就透出这套东西的定位：脚本要可重放，所以不许有不确定性。

一段真正的 workflow 长这样——审查一批改动文件、每条发现随即对抗式复核：

```javascript
const results = await pipeline(
  DIMENSIONS,
  d => agent(d.prompt, { phase: 'Review', schema: FINDINGS }),
  review => parallel(review.findings.map(f => () =>
    agent(`Adversarially verify: ${f.title}`, { phase: 'Verify', schema: VERDICT })
      .then(v => ({ ...f, verdict: v })))),
)
```

派生出去的一个 subagent 到底是什么？抓一次真实 workflow 的网络请求就看清了。它是一个独立的客户端请求，系统提示头一句就是 "You are a subagent spawned by a workflow orchestration script"，还特意交代：你的最终文本会被原样当成字符串返回给调用它的脚本——那是返回值，不是给人看的消息。它还是个叶子 worker。主模型手里那几个能再往下编排、排程的工具——`Agent`、`Workflow`、`ScheduleWakeup`，加上 `TaskOutput`——都从子 agent 的工具集里摘掉了（抓包里子 agent 76 个工具、主模型 80 个，差的正是这四个）。所以它只能干活，不能再 fan-out。而对主模型来说，整个 Workflow 调用是个后台任务：一调就立刻返回一个 task id，workflow 跑完再用 `<task-notification>` 通知回来。

## 19.3 默认用 pipeline，parallel 是个 barrier——为什么

这是整套设计里最见功力的一点，也是工具描述里花最多篇幅讲的：什么时候用 `pipeline`、什么时候用 `parallel`。

`parallel` 是一道 barrier：它并发跑一批任务，然后等所有任务都完成才返回。`pipeline` 不设 barrier：每个 item 独立地穿过所有 stage，item A 可以已经在第 3 个 stage，而 item B 还在第 1 个。差别落到墙钟时间上很实在——`pipeline` 的总耗时约等于"最慢那一条单项链"，而不是"每个 stage 里最慢那个之和"。如果 5 个 finder 并行跑、最慢的比最快的慢三倍，用 barrier 就白白浪费掉快 finder 三分之二的空闲。

所以工具描述把话说死了：默认用 `pipeline`。只有一种情况才该用 barrier——第 N 个 stage 真的需要"前一个 stage 全部结果凑齐"才能干，比如在昂贵的下游步骤前先对全量结果去重、或者"零发现就整轮跳过"、或者某个 stage 的提示词要引用"其它所有发现"来比对。它甚至给了个判据：如果你写的是"并发跑一批 → 把结果 flatten/map/filter 一下 → 再并发跑一批"，那中间那步根本不需要 barrier，把它塞进 pipeline 的一个 stage 就行。

一个把编程模型做成工具、还在工具描述里教模型"别乱设 barrier、拿不准就 pipeline"的系统，其实是在替模型把一类经典的并发反模式提前堵掉。

还有一点错误语义值得记：`parallel` 里某个任务抛错，那一格就变成 null（用结果前先 `.filter(Boolean)` 滤掉）；`pipeline` 里某个 stage 抛错，那个 item 就落下、跳过它剩下的 stage。失败被隔离成单点，不会把整批带崩。

## 19.4 让 subagent 返回结构化数据

多个 agent 要串成流水线，前一个的输出得能喂给后一个。纯文本不好办——你还得解析。Workflow 的做法是给 `agent()` 一个 `schema`（一份 JSON Schema）。带上它，运行时就往这个 subagent 的工具集里塞一个叫 `StructuredOutput` 的工具，而且把这份 schema 作为该工具的 `input_schema`；工具描述写着 "You MUST call this tool exactly once at the end"。subagent 干完活必须调它一次，参数当场按你的 schema 校验（`required`、`additionalProperties: false` 都生效），`agent()` 于是拿到一个校验过的对象，不用你再解析。（这套机制是抓一次带 schema 的真实 workflow、在子 agent 的请求里看到那个 `StructuredOutput` 工具确认的。）

关键在于校验发生在工具调用那一层：成功路径下 `agent()` 直接拿到校验过的对象；形状不对会先 nudge、再重试，但要是 subagent 到末尾还是不调用、或超了结构化输出的重试上限，这次 `agent()` 就失败——它若在 `parallel`/`pipeline` 的 slot 里，失败被隔离成 null；直接 `await agent(...)` 则会抛错。于是"上一个 agent 产出一批 `{file, line, severity}`、下一个 agent 挨个 verify"这种拼装在成功路径上是可靠的——stage 之间流动的是结构化数据，不是等着被正则的字符串。19.2 那个例子里，`FINDINGS` 和 `VERDICT` 两个 schema 就是这么让 review 阶段和 verify 阶段对接上的。

## 19.5 三道上限加一个预算

一段脚本能铺开几百个 agent，就必须有护栏，否则一个写错的循环能把机器和账单一起拖垮。运行时压了三道硬上限，工具描述里逐字写着：

- 并发：同时在跑的 `agent()` 调用被夹在 `min(16, cpu cores - 2)`（16 和"CPU 核数减 2"取小），超出的排队，慢慢腾出槽位再跑。
- 单次：一次 `parallel()` 或 `pipeline()` 最多收 4096 个 item，再多直接报错，不是静默截断。
- 总数：一个 workflow 整个生命周期最多派 1000 个 agent（运行时里就是个常量 `1000`），描述里管它叫"防失控循环的兜底，设得远高于任何真实 workflow"。

第四道不是上限而是预算。如果你给这轮任务下过一个 token 目标（那种 `+500k` 式的指令），`budget` 就把它变成一个硬天花板：`budget.total` 是目标，`budget.spent()` 是已花，`budget.remaining()` 是剩余；一旦花到顶，再调 `agent()` 就抛错。这让脚本能按预算动态decide深度——描述里给的写法是 `while (budget.total && budget.remaining() > 50_000) { … }`，边花边看还剩多少、决定要不要再派一轮。撞上上限或预算，分别记 `tengu_workflow_agent_cap_exceeded` 和 `tengu_workflow_budget_cap_exceeded` 两个事件。

## 19.6 隔离与断点续跑

几百个 agent 并行干活，两件事会出岔子：它们同时改同一个文件会互相覆盖；而这么大的一次运行跑到一半挂了，重头再来太亏。Workflow 对这两件事各有一手。

隔离靠 worktree。给 `agent()` 传 `isolation: 'worktree'`，运行时就给这个 agent 开一个独立的 git worktree，让它在自己那份副本里改，跑完没改动就自动删、有改动就把路径交回来。这正是第 8 章讲的 worktree 隔离底座（见 8.2），Workflow 直接拿来用。描述里也提醒它贵——每个 agent 多花几百毫秒加一份磁盘，所以只在"并行写、会互相冲突"时才开。

断点续跑靠 journal 和 runId。每次 run 有个 id（形如 `wf_000d0acb-383`），脚本每个 `agent()` 都往 journal 里记两条——一条 `started`、一条 `result`——都带一个 `v2:<哈希>` 的 key，那个 key 就是这次调用（prompt 加 opts）的内容哈希。带 `resumeFromRunId` 重跑时，最长的那一段"key 没变"的 `agent()` 前缀直接命中 journal 里上次存的 result，从第一个哈希变了的调用起才真重跑。（这些都是从一次真跑 workflow 的落盘里看到的：模型现写的那段脚本本身会被存成一个可编辑的 `.js` 文件，run 目录 `<会话>/subagents/workflows/wf_<runId>/` 下有 `journal.jsonl`，每个子 agent 的完整对话再另存一份 `agent-<id>.jsonl`。）二进制里能读到这套机制的骨架：运行时按 `workflowRunId` 注册和管理这个 `local_workflow` 任务，缓存命中则由同一 runId 的 journal 提供（相关事件 `tengu_workflow_journal_started_hit_respawn`、`tengu_workflow_saved`）。所以一个 100 个 agent 的迁移跑到第 60 个断了，接着跑不会重做前 59 个。这也解释了 19.2 为什么禁 `Date.now()`/`Math.random()`——脚本必须可重放，缓存命中才对得上。

## 19.7 把可靠性编排进流程

Workflow 真正有意思的地方，不只是"并行快"，而是它把一套追求可信度的编排范式写进了工具描述，鼓励模型照着搭。几个典型的：

- 对抗式复核：每条发现，另派 N 个独立的怀疑者，各自被要求去反驳它；多数反驳成立就把这条发现剔除。这防的是"看着有道理其实站不住"的发现混过去。
- 视角分散的复核：一条发现能从多个角度出问题时，给每个复核者一个不同的镜头（正确性、安全、性能、能不能复现），而不是派 N 个一样的。
- judge panel：从不同切入点独立生成 N 个方案，并行打分，从赢家里综合、再嫁接亚军的好点子。
- loop-until-dry：对不知道有多少的发现（bug、边角情况），一直派 finder，直到连续 K 轮没有新东西才停——简单的"跑 N 轮"会漏掉长尾。
- completeness critic：最后派一个 agent 专门问"还缺什么——哪种搜法没跑、哪条断言没验、哪份材料没读"，它找出来的就是下一轮的活。

这些范式合起来是一个朴素的主张：多个 agent 的价值不在于快，而在于能用结构化的冗余把"单个 agent 容易疏漏"这件事系统地纠回来。

这些范式也不是纸上谈兵。Claude Code 自带的 `/deep-research` 就是照着搭的一个 workflow；二进制里它的 phase 元数据显示的结构是：先 Scope，把问题拆成 5 个搜索角度；再 Search，5 个并行 WebSearch agent、每个角度一个；再 Fetch，把 URL 去重、取前 15 个源、抽出可证伪的断言；再 Verify，每条断言派 3 票做对抗验证、2/3 反驳才判死；最后 Synthesize，合并语义重复、按置信度排序、附上引用。数一下就对上了：多路搜索是 multi-modal sweep，3 票 2/3 是对抗式复核，取源那步的 URL 去重正是"先凑齐再往下"、需要 barrier 的地方——一个 workflow 把前面几节讲的范式几乎全用上了。

## 19.8 ultracode、面板、deep-research 与关停

平时你未必手写 workflow 脚本。有两个入口都带 `ultracode` 这个词，得分清。一是在 prompt 里带上它——这让当前这一轮 opt-in 成 Workflow 工具，模型就自己写一段 workflow 并跑起来，把一个大活拆开、并行铺出去，只作用于这一次。二是会话级的 `/effort ultracode`，那才是官方说的"xhigh 推理 + 常驻的自动 workflow 编排"，在整个会话里都开着。`/workflows` 面板看每个 workflow 的实时进度（`meta` 里的 phase 和 `log()` 打的消息就显示在这儿）。`/deep-research` 则是一个官方打包好的 workflow，多路搜索、深读、对抗式核实再综合成带引用的报告——它本身就是这套编排能力的一个成品。整个功能能用 `CLAUDE_CODE_DISABLE_WORKFLOWS=1` 关掉。

## 19.9 从 ultraplan 到 dynamic workflows：一条可对账的演进线

这套能力也能两端对账。快照那头（v2.1.88）已经有它的前身——`utils/ultraplan/` 目录：一个 `keyword.ts` 负责在你的输入里检测触发词 `ultraplan`，细致到跳过引号、路径、问号后、slash 命令里的假阳性。另一个 `ccrSession.ts` 管云端会话这一头：`/ultraplan` 启动时会 teleport 到一个远程会话、把它设成 plan 模式，之后 `ccrSession` 每 3 秒轮询事件流、等你在浏览器里批准一个 `ExitPlanMode`，再把 plan 文本抽出来。也就是说，当时这个功能叫 `ultraplan`，靠关键词触发，干的是"到云端跑一遍 plan 模式、等你点批准、取回一份 plan"。

当前这头（2.1.201）：触发词换成了 `ultracode`，遥测从 `tengu_ultraplan_*` 一族变成了 `tengu_workflow_*` 一族，能力也从"云端出 plan"长成了"一段确定性 JS 脚本铺开 agent 舰队"。关键词触发那套一脉相承地留了下来，而 fan-out 编排的运行时是快照之后才长出来的。触发的骨架来自快照源码，成型的编排来自当前二进制——又一条"没随快照泄露不等于分析不了"的线。

## 19.10 附录：我们怎么知道的（逆向方法，可复现）

这一章的证据来自几条能自己复现的路。

第一条，读工具描述。Workflow 是注入给主模型的一个工具，它的描述就是"怎么写 workflow 脚本"的完整说明——编程模型、pipeline 与 parallel 的语义、三道上限、预算、隔离、续跑、那套质量范式，全在里面。它在运行时的工具集里，而其中的关键句子（`min(16, cpu cores - 2)`、`at most 4096 items`、`capped at 1000`）在 2.1.201 的二进制里能逐字对上，两相印证。

第二条，从二进制抽字符串和运行时常量。`grep` 全部 `tengu_workflow_*` 和 `tengu_ultraplan_*` 事件名当功能地图；压缩后的运行时代码里还能读到上限常量 `1000`、以及 `workflowRunId` / `resume` / 任务表这套续跑机制的骨架。

第三条，抓 fan-out。架一个明文反向代理（配方见[第 17 章](17-autonomy-goal-loop.md) 17.5），跑一个很小的 workflow，就能看到一段脚本并发派出去的 subagent 请求。本章就真跑了两个探针。2-agent 并行那个，抓到两个子 agent 的请求时间戳只差 0.01 秒——并发派生坐实，也看到 "You are a subagent spawned by a workflow orchestration script" 的系统提示。带 schema 的那个，看到子 agent 工具集里被注入的 `StructuredOutput` 工具，它的 `input_schema` 就是你传的那份。要注意分寸：workflow 会 spawn 大量 agent、很烧 token，探针要限成两三个 agent、只读任务、跑完就停。

第四条，读快照前身。`utils/ultraplan/` 在 v2.1.88 快照里就有，用来做 ultraplan → dynamic-workflows 的演进对账。

诚实边界：工具描述、上限、事件、续跑机制都是能直读或直接确认的；但某一次 workflow 的确切收敛与分支逻辑写在模型现写的脚本里、不是固定提示词，运行时并发调度器的内部、以及服务端，都拿不到。`ultracode` 背后用哪个模型档会随灰度变，正文不写死。

---

## 附录：Workflow 工具描述（关键节选）

正文只摘了要点。下面节选工具描述的几段，标清哪些是逐字、哪些是压缩：上限与 pipeline/parallel 那两段是逐字原文（已与 2.1.201 二进制逐句核对）；编程模型骨架和模式清单是忠实压缩——保留原意和关键短语，省掉完整的类型签名与细则（比如 `agent()` 的 `null` 返回、`workflow()` 的共享与嵌套限制，这些正文里讲过）。来源是运行时注入的那段"教主模型写 workflow"的说明。

### 脚本骨架与 hooks

```text
Every script must begin with `export const meta = {...}`:
  export const meta = {
    name: 'find-flaky-tests',
    description: 'Find flaky tests and propose fixes',
    phases: [ { title: 'Scan', … }, { title: 'Fix', … } ],
  }

Script body hooks:
- agent(prompt, opts?): spawn a subagent. Without schema, returns its final text
  as a string. With schema (a JSON Schema), the subagent is forced to call a
  StructuredOutput tool and agent() returns the validated object. opts: label,
  phase, schema, model, effort, isolation:'worktree', agentType.
- pipeline(items, stage1, stage2, …): run each item through all stages
  independently, NO barrier between stages. … Wall-clock = slowest single-item
  chain, not sum-of-slowest-per-stage.
- parallel(thunks): run tasks concurrently. This is a BARRIER: awaits all thunks
  before returning. … `.filter(Boolean)` before using the results.
- phase(title) / log(message) / budget / workflow(nameOrRef, args?)
```

### pipeline vs parallel 的取舍

```text
DEFAULT TO pipeline(). Only reach for a barrier (parallel between stages) when
you genuinely need ALL prior-stage results together.

A barrier is correct ONLY when stage N needs cross-item context from all of
stage N-1: dedup/merge before expensive downstream work; early-exit if the total
count is zero; stage N's prompt references "the other findings".

Smell test: if you wrote
  const a = await parallel(...)
  const b = transform(a)        // flatten, map, filter — no cross-item dependency
  const c = await parallel(b.map(...))
that middle transform doesn't need the barrier. Rewrite as a pipeline. When in
doubt: pipeline.
```

### 上限与预算

```text
Concurrent agent() calls are capped at min(16, cpu cores - 2) per workflow —
excess calls queue and run as slots free up. … Total agent count across a
workflow's lifetime is capped at 1000 — a runaway-loop backstop set far above any
real workflow. A single parallel()/pipeline() call accepts at most 4096 items.

budget: {total, spent(), remaining()} — the turn's token target … a HARD ceiling:
once spent() reaches total, further agent() calls throw.
  while (budget.total && budget.remaining() > 50_000) { … }
```

### 一套质量编排范式（清单，细则省略）

```text
- Adversarial verify: spawn N independent skeptics per finding, each prompted to
  REFUTE. Kill if ≥majority refute.
- Perspective-diverse verify: give each verifier a distinct lens.
- Judge panel: generate N independent attempts, score with parallel judges,
  synthesize from the winner.
- Loop-until-dry: keep spawning finders until K consecutive rounds return nothing.
- Multi-modal sweep / Completeness critic: "what's missing?" becomes next round.
```

### 子 agent 系统提示与 StructuredOutput 工具（逐字，反代抓包 2.1.202）

一个被 workflow 派生的 subagent，它系统提示的头部：

```text
You are a subagent spawned by a workflow orchestration script. Use the tools available to complete the task.

CRITICAL: Your final text response is returned **verbatim** as a string to the calling script — it is your return value, not a message to a human. Output the literal result (data, JSON, text). Do NOT output confirmation…
```

带 `schema` 时，被注入进子 agent 工具集的 `StructuredOutput` 工具（它的 `input_schema` 就是调用方传入的 schema）：

```text
StructuredOutput — Use this tool to return your final response in the requested
structured format. You MUST call this tool exactly once at the end of your
response to provide the structured output.
```

> 来源：工具描述节选来自运行时注入的 Workflow 工具，`min(16, cpu cores - 2)` / `at most 4096 items` / `capped at 1000` 等句已与 2.1.201 二进制逐字核对；子 agent 系统提示与 `StructuredOutput` 工具来自对 2.1.202 客户端真跑一个探针 workflow 的明文反代抓包。示例与模式细则用 `…` 省略。某次 workflow 的具体脚本、运行时并发调度器内部、以及 `ultracode` 背后的模型档，均不在可见范围内。
