# 第 17 章：自治与续跑——`/goal` 与 `/loop`

> 从这一章起，进入一个新模块：泄露快照之后的新功能。
>
> 前十六章读的是那份 3 月底流出的源码快照，但 `/goal`、`/loop`、dynamic workflow、auto mode 这些能力全在快照之后，没有源码可看。搞清楚它们只能换个办法：装上最新版真的去用，把它发出的网络请求抓下来，再对照官方文档，三头对上。所以下面凡是引号里的原文，都是这么抓出来的；凡是讲到“它内部大概怎么调度”，那是从行为反推的，讲到时会说明。这一章说清楚 `/goal` 和 `/loop` 怎么让 Claude 自己接着干，用到的办法写在文末，照着你也能自己去抓下一个功能。

## 17.1 两种“让它接着干”的范式

到 2026 年，Claude Code 早已不只是“一问一答”。它有一整族能力，让 agent 跨 turn、跨时间、跨会话地自己接着干。这一族里最外层、你最常碰到的两个入口，是 `/goal` 和 `/loop`——而它俩恰好是两种相反的思路。

`/goal` 是“盯着一个条件、不达成不罢休”。你给它一句完成条件，它就一轮一轮干下去，每轮结束由一个独立的裁判判一次“达成没有”，没达成就带着裁判给的理由再来一轮，达成了才停。它是被动的：什么时候停，交给裁判说了算。

`/loop` 是“定个闹钟、反复来”。你给它一个间隔（或者让它自己定节奏），它就按点反复跑同一件事。它是主动的：什么时候再来，由调度决定，跟“有没有干成”无关。

一个靠“守门人”决定何时收手，一个靠“闹钟”决定何时再来——抓住这条区别，也就抓住了 Claude Code 自治的两条主线。下面分别拆。

## 17.2 `/goal`：一个守门的裁判

### 它其实是 Stop hook 的语法糖

官方文档一句话点明了机制：`/goal` 是对一个会话级 Stop hook 的封装。每当一个 turn 结束，系统就把“你设的条件 + 到目前为止的对话”发给一个评估器模型（官方称“小快模型”、默认 Haiku；一次抓包看到的实际模型见下），它回一个“是 / 否 + 一句理由”；“否”就让 Claude 带着这句理由再干一轮，“是”就清掉目标、并在会话记录里记一笔达成。

抓包印证了这套流程。你设目标那一刻，主模型收到的消息里有这么一段，几乎是把机制写在了明面上（下引为关键片段，末尾省略）：

> A session-scoped Stop hook is now active with condition: "<你的条件>". Briefly acknowledge the goal, then immediately start (or continue) working toward it — treat the condition itself as your directive and do not pause to ask the user what to do. The hook will block stopping until the condition holds. It auto-clears once the condition is met…

“设目标即启动一轮，把条件本身当指令。”这就是为什么你不用再单独发一句提示。

### 裁判的判决：三种结果，一道死循环刹车

真正有意思的是那个裁判。把它发出的真实请求抓下来，它的系统提示词逐字就是下面这段——不长，值得整段读一遍，因为一个自治循环的全部分寸都压在这几行里：

> You are evaluating a stop-condition hook in Claude Code. Read the conversation transcript carefully, then judge whether the user-provided condition is satisfied.
>
> Your response must be a JSON object with one of these shapes:
> - `{"ok": true, "reason": "<quote evidence from the transcript that satisfies the condition>"}`
> - `{"ok": false, "reason": "<quote what is missing or what blocks the condition>"}`
> - `{"ok": false, "impossible": true, "reason": "<explain why the condition can never be satisfied>"}`
>
> Always include a "reason" field, quoting specific text from the transcript whenever possible. If the transcript does not contain clear evidence that the condition is satisfied, return `{"ok": false, "reason": "insufficient evidence in transcript"}`.
>
> Only use `{"ok": false, "impossible": true}` when the condition is genuinely unachievable in this session — for example: the condition is self-contradictory, it depends on a resource or capability that is unavailable, or the assistant has explicitly tried, exhausted reasonable approaches, and stated it cannot be done. Apply your own judgment when deciding this — the assistant claiming the goal is impossible is evidence, not proof; independently confirm the condition is genuinely unachievable rather than deferring to the assistant's self-assessment. Do not use it just because the goal has not been reached yet or because progress is slow. When in doubt, return `{"ok": false}` without "impossible".

三种结果——达成、没达成、判定不可能——就是这段里的三个 JSON 形状。前两种直白；关键在第三种。`impossible` 是一道精心设计的死循环刹车，而整整一段都在提防同一件事：别让主 agent 把裁判忽悠着提前认输。“主 agent 说干不成，只算证据、不算铁证；裁判得自己独立确认，拿不准就返回 `{"ok": false}`、别加 `impossible`。”一个自治循环最怕的就两头——要么停不下来，要么被内部说服着草草收场，这段提示词正是同时冲着这两头写的。它甚至连“进度慢”都点名排除：慢不等于不可能。

### 三个抓包才看得到的工程细节

把裁判的真实请求整个拆开，还能看到官方文档没提的三件事。

第一，判决是 API 层强制的，不只是提示词请求。这条请求带了一个 `output_config`，用 JSON schema 把输出死死约束成 `{ok, reason, impossible}` 这个形状（`ok` 和 `reason` 必填、不许有别的字段）。提示词只是说明，schema 才是护栏——就算模型想自由发挥，也发挥不出这个形状之外。

第二，裁判不给工具、只让它看对话。请求里的 `tools` 是空的。这印证了官方那句“裁判不调用工具，只能判断已经出现在对话里的内容”：它虽然被塞了一个 transcript 路径，却没有任何工具去读那个文件。

第三，裁判跑在高推理档。请求里 `effort: "high"`。判“到底达没达成”这件事，系统舍得花算力。

关于用哪个模型，有一处得说老实话：官方文档说裁判用“你配置的小快模型，默认 Haiku”，但我这台机器抓到的实际是 `claude-fable-5`——因为本机把小快模型配成了它。所以别把一次抓包看到的模型名当成默认值。还要澄清一点：是客户端的 hook 运行时在本地组装并发出这条请求，模型推理仍在 Anthropic 那边跑，不是你本地在跑模型。

### 它在追踪里长什么样

`/goal` 每一轮的进度都会落进会话记录：一条 `goal_status`，带着条件、迭代了几轮、耗时、烧了多少 token、达成没。所以“一个目标跨多少 turn、每轮什么状态”是完全可回放的——这正是上一章讲的那份“默认就在”的追踪的一个活例子，详见[第 16 章](16-observability.md)。

## 17.3 `/loop`：一个自己排程的闹钟

`/loop` 和 `/goal` 骨子里不一样。它不是一个被动 hook，而是一大段由主模型执行的编排提示词。你敲 `/loop …`，系统注入一段以 `# /loop — schedule a recurring or self-paced prompt` 开头的指令，让主模型自己去解析、选调度方式、调用通用的调度工具。换句话说，`/loop` 的“聪明”写在提示词里，不是一个硬编码的调度器——不过要补一句，真正的执行、生命周期和保护栏仍然硬编码在运行时，这点后面会看到。

### 它怎么解析你输入的

抓到的这段编排提示词，把解析规则逐字写死了——直接看原文，比我转述清楚：

> \# /loop — schedule a recurring or self-paced prompt
>
> \#\# Parsing (in priority order)
>
> 1. **Leading token**: if the first whitespace-delimited token matches `^\d+[smhd]$` (e.g. `5m`, `2h`), that's the interval; the rest is the prompt.
> 2. **Trailing "every" clause**: otherwise, if the input ends with `every <N><unit>` or `every <N> <unit-word>` (e.g. `every 20m`, `every 5 minutes`), extract that as the interval and strip it from the prompt. Only match when what follows "every" is a time expression — `check every PR` has no interval.
> 3. **No interval**: otherwise, the entire input is the prompt and you'll self-pace dynamically.
>
> Examples:
> - `check the deploy every 20m` → interval `20m`, prompt `check the deploy` (rule 2)
> - `check every PR` → no interval → dynamic mode, prompt `check every PR` (rule 3 — "every" not followed by time)
> - `5m` → empty prompt → show usage

三条规则一目了然：先看开头是不是 `5m` 这种间隔，再看结尾有没有 `every 20m` 从句，都没有就整句当 prompt、进自定节奏。最见功力的是 `check every PR` 那个反例——提示词特意声明“every 后面不是时间表达式就不算间隔”，免得把“检查每个 PR”误读成“每隔一个 PR 跑一次”。把边界情形连同用例一起写进提示词，这本身就是一课。

### 三条执行路径

解析出间隔和 prompt 后，走哪条路也是提示词说了算。

如果间隔在一小时以上、或者你用的是“每天早上”“每晚”这种日级措辞，提示词让模型先弹一个问题，问你要不要改成能脱离会话长期运行的云端计划（`/schedule`）——因为纯 `/loop` 一关会话就没了。你若选“只在本会话”，它才继续往下。

要是就在本会话按固定间隔跑，模型会调 `CronCreate`。这一步我抓到了真实调用：

```text
CronCreate({"cron": "*/1 * * * *", "prompt": "…", "recurring": true})
```

工具返回的关键部分是：`Scheduled recurring job 7e7d1261 (Every minute). Session-only (not written to disk, dies when Claude exits). Auto-expires after 7 days. Use CronDelete to cancel sooner.` 一句话里印证了好几点——`1m` 被转成了标准 cron 表达式 `*/1 * * * *`，任务有个 8 位 id，只活在会话里、不落盘（所以 Claude 一退出就没了、也不留垃圾），还会 7 天后自动过期。

有个容易误解的点值得单独点出：建完 cron 之后，编排提示词要求模型**立刻先把解析出的任务跑一次**——原文是 “Then immediately execute the parsed prompt now — don't wait for the first cron fire.” 所以 `/loop 1m X` 是先安排好定时、再马上执行一次 `X`，不是等一分钟后才第一次跑；之后才由 cron 继续按点触发。

如果你没给间隔，走的是“自定节奏”：模型用一个叫 `ScheduleWakeup` 的工具自己排下一次醒来。这个工具的描述里把设计取舍讲得很透，值得引一段原文：

> Do NOT schedule a short-interval wakeup to poll for background work you started — when harness-tracked work finishes, you are re-invoked automatically, so polling is wasted. Instead schedule a long fallback (1200s+) so the loop survives if the work hangs or never notifies.

翻成人话：别拿短间隔去反复查那些 harness 本来就会在完成时通知你的后台活——那纯属白轮询；应该排一个较长（一千多秒起）的兜底唤醒，只为防止任务挂死或永远不回信。每次还要把同一个 `/loop` prompt 通过参数传回去，下一次醒来才接着干同一件事。一个让模型自己给自己定闹钟的工具，居然要先教模型“别把闹钟定太勤”，这恰恰暴露了自排程最容易踩的坑。

四种输入、四条路径，摆到一起更清楚：

| 你输入 | 走哪条 | 第一步 | 之后怎么继续 |
|---|---|---|---|
| `1m X`（有间隔） | 建 session cron | 先立刻跑一次 `X` | cron 按点重投 `X` |
| `X`（无间隔） | 自定节奏 | 先立刻跑一次 `X` | `ScheduleWakeup` / `Monitor` 排下一次 |
| 空 | 内置维护 / `loop.md` | 跑默认维护 prompt | 同上，按节奏继续 |
| 间隔≥1h 或“每天…” | 先问 | 提议改云端计划 | 你选本会话才继续上面几条 |

### 什么都不给它，它跑维护脚本

`/loop` 后面要是空着，据官方文档它会跑一个内置的维护提示词（或者你自己写的 `loop.md`）：优先接着做没做完的事，照看当前分支的 PR，闲下来做点清理；并守一条底线——不开新战线，推送、删除这类不可逆动作只在对话里已授权时才做。这段是照官方文档讲的：二进制里能搜到相邻的 babysit、autofix 片段，但不足以逐字坐实“修失败 CI、回 review、解合并冲突”就是空 `/loop` 的通用默认——所以按官方描述转述，别把它当成从产物里直接确认过的行为。

### 背后的调度守护进程

本地的 `/loop`、cron、动态唤醒和通知路径，背后能看到一组同族的后台守护进程遥测（内部叫 KAIROS）。至于云端计划是不是也由这个客户端 daemon 管，证据不足——而且官方口径是云端跑在 Anthropic 那边，所以这里只把 KAIROS 当作本地调度/唤醒底座的一部分。它的内部调度和持久化逻辑是纯客户端后台代码，抓不到、只能推断。

准确的说法是这样：`/loop` 的高层决策（怎么解析、选哪条路、什么时候收敛）由提示词主导，但执行、生命周期和保护栏在运行时里写死——比如 `ScheduleWakeup` 的延迟被夹在一个固定区间内，加上保活兜底、太久没动就回收、cron 存储、调度器本身，这些都不是模型能改的。提示词负责“决策”，运行时负责“兜底”。

最后一处值得记一笔，因为它是“文档和实测对不上”的一个例子。关于 jitter（把触发时间随机错开、防止一堆会话同一秒打 API），官方文档说重复任务最多可以滞后半小时；但我这台 v2.1.201 机器上，`CronCreate` 工具自己的描述写的是“滞后周期的 10%、最多 15 分钟”。两个口径对不上，多半是版本差异——引用时得注明是哪个来源。

## 17.4 两个入口，一族底座

`/goal` 和 `/loop` 只是最外层的两个入口，它俩摆在一起，正好照出这一族自治能力的两个维度：谁触发下一轮，以及靠什么调度。下面点到的其它成员是同一族里相邻的能力，并非每一个都在本章里被抓包验证过——这里只把它们跟 `/goal`、`/loop` 的关系串一下，完整分析留给后面的章节。

触发这一侧，除了 `/goal` 封装的 Stop hook，还有 auto mode——它逐个自动放行工具调用，把每一步的确认省掉，好让续跑的每一轮都能无人值守地跑完。调度这一侧，除了 `/loop` 用到的 cron 和 `ScheduleWakeup`，还有 `Monitor`（盯着一个长跑脚本、有输出就推回来，比轮询更省），以及把这些串起来的 KAIROS 守护进程。再往外，还有云端计划（跑在 Anthropic 那边、关了会话也不停）、后台 agent（`/bg`、`claude agents`）、以及 `--resume` 时把没做完的目标和没过期的定时任务一起复活。

一句话概括这一族：一个后台调度基座，加一套通用调度工具，加提示词编排。`/goal` 和 `/loop` 是这套底座露在外面的两个语法糖，后面的章节会接着拆这一族的其它成员。

## 17.5 附录：我们怎么知道的（逆向方法，可复现）

本章能确认的东西，都来自两层办法。下面把过程写全，任何人都能复现——你甚至可以把这一节直接丢给 Claude Code，让它照着帮你重做一遍。

第一层是静态抽取，零成本、发版当天就能做。要提醒一句：2.1.x 起，Claude Code 已经不是一个明文的 `cli.js`，而是编译成了原生二进制（用 Bun 打包）。但 JS 源仍作为字符串嵌在里面，`strings` 照样抠得出来。抠出全部字符串后，用 `grep` 找某个功能的遥测事件名（当功能地图用），再找它的提示词和命令本体。有一个坑：在这个几十万行的大文件上用 `.{0,N}` 这种正则会疯狂回溯、直接超时，要用定长字符串 `grep -F`。这一层能拿到提示词片段、工具描述、事件名，都是逐字能核对的；但拿不到这些字符串怎么拼装、控制流长什么样——那些只能靠推断。

第二层是抓真实网络请求，看运行时最终组装出来的完整报文（系统提示、消息、工具、模型、`output_config` 一应俱全）。做法是架一个明文反向代理：让 `claude` 明文连本地（这样免去证书那套麻烦），代理把请求体记下来，再转发到真的 API。跑的时候从一个“已信任”的目录起（`/goal` 这类 hook 系功能要求工作目录接受过信任对话），把 `ANTHROPIC_BASE_URL` 指到本地代理，然后 `claude -p "/goal …"`，在抓到的一堆请求里找带那句“stopping condition”的，就是裁判那条。先拿一句 `-p "reply with PONG"` 冒烟验证代理通了、能抓到，再跑目标功能；探针条件挑那种一句话就能满足、很快收敛的，别动工具权限、别长跑烧 token。

还有两层顺手就能补。会话记录：跑完直接读 `~/.claude/projects/` 下的 JSONL，里面有 `goal_status`、每一步的 token 用量、消息间的父子链——追踪那一层的证据，零成本就能拿到。OTel 遥测：开 `CLAUDE_CODE_ENABLE_TELEMETRY` 加 console 导出器跑一遍，能看到那些内部 `tengu_` 事件对外导成 `claude_code.*` 指标真的在发。

最后是纪律。提示词、报文、字符串是能逐字核对的硬证据；内部怎么调度、怎么放行，是从行为反推出来的，只能当推断，不能拿它当确凿的事实来引。模型名、耗时这种只观测到一次的值，要写明是“这一次抓到的”，别当成通例。事件名要逐个列全，别用 `a|b|c` 这种正则式简写——一压缩就可能拼出一个根本不存在的名字。还有一条安全提醒：抓下来的报文里有账号、设备、会话的 id，还有提示词和工具结果，可能夹着密钥——只自己私下留着，要引用先把敏感信息抹掉。

想复现的话，一句话就能交给 Claude Code：“读这一节的方法，帮我复现 `/goal` 裁判的逆向——`strings` 抠本机 claude 二进制里 goal 相关的字符串；写一个明文反代，让 `claude -p '/goal …'` 走它、抓到裁判那条请求；把裁判的系统提示、`output_config`、`tools`、模型抠出来，跟官方 `/goal` 文档对一遍，标清哪些是抓到的原文、哪些是推断出来的。”这就是全部办法，你现在也能自己去抓下一个新功能了。

---

## 附录：完整提示词原文（逐字）

正文只摘了关键处；下面把从抓包里逐字取出的完整原文放全，供想深挖或复现的人对照。来源是对 2.1.201 客户端的明文反代抓包，凡属注入槽位（你的条件、你的输入）已替换成占位符，其余一字未改。

### `/goal` 评估器系统提示词（stop-condition evaluator system prompt）

```text
You are evaluating a stop-condition hook in Claude Code. Read the conversation transcript carefully, then judge whether the user-provided condition is satisfied.

Your response must be a JSON object with one of these shapes:
- {"ok": true, "reason": "<quote evidence from the transcript that satisfies the condition>"}
- {"ok": false, "reason": "<quote what is missing or what blocks the condition>"}
- {"ok": false, "impossible": true, "reason": "<explain why the condition can never be satisfied>"}

Always include a "reason" field, quoting specific text from the transcript whenever possible. If the transcript does not contain clear evidence that the condition is satisfied, return {"ok": false, "reason": "insufficient evidence in transcript"}.

Only use {"ok": false, "impossible": true} when the condition is genuinely unachievable in this session — for example: the condition is self-contradictory, it depends on a resource or capability that is unavailable, or the assistant has explicitly tried, exhausted reasonable approaches, and stated it cannot be done. Apply your own judgment when deciding this — the assistant claiming the goal is impossible is evidence, not proof; independently confirm the condition is genuinely unachievable rather than deferring to the assistant's self-assessment. Do not use it just because the goal has not been reached yet or because progress is slow. When in doubt, return {"ok": false} without "impossible".
```

### `/goal` Stop hook 激活消息（设目标时注入主模型；`<your condition>` 是你填的条件）

```text
A session-scoped Stop hook is now active with condition: "<your condition>". Briefly acknowledge the goal, then immediately start (or continue) working toward it — treat the condition itself as your directive and do not pause to ask the user what to do. The hook will block stopping until the condition holds. It auto-clears once the condition is met — do not tell the user to run `/goal clear` after success; that's only for clearing a goal early.
```

### `/loop` 命令编排提示词（`## Input` 段是把你的输入拼进去的位置）

```text
# /loop — schedule a recurring or self-paced prompt

Parse the input below into `[interval] <prompt…>` and schedule it.

## Parsing (in priority order)

1. **Leading token**: if the first whitespace-delimited token matches `^\d+[smhd]$` (e.g. `5m`, `2h`), that's the interval; the rest is the prompt.
2. **Trailing "every" clause**: otherwise, if the input ends with `every <N><unit>` or `every <N> <unit-word>` (e.g. `every 20m`, `every 5 minutes`, `every 2 hours`), extract that as the interval and strip it from the prompt. Only match when what follows "every" is a time expression — `check every PR` has no interval.
3. **No interval**: otherwise, the entire input is the prompt and you'll self-pace dynamically (see "Dynamic mode" below).

If the resulting prompt is empty, show usage `/loop [interval] <prompt>` and stop.

Examples:
- `5m /babysit-prs` → interval `5m`, prompt `/babysit-prs` (rule 1)
- `check the deploy every 20m` → interval `20m`, prompt `check the deploy` (rule 2)
- `run tests every 5 minutes` → interval `5m`, prompt `run tests` (rule 2)
- `check the deploy` → no interval → dynamic mode, prompt `check the deploy` (rule 3)
- `check every PR` → no interval → dynamic mode, prompt `check every PR` (rule 3 — "every" not followed by time)
- `5m` → empty prompt → show usage

## Offer cloud first

Before any scheduling step, check whether EITHER is true:
- the parsed interval (rule 1 or 2) is **≥60 minutes**, or
- regardless of which rule matched, the original input uses daily phrasing ("every morning", "daily", "every day", "each night", "every weekday")

If either is true, call AskUserQuestion first:
- `question`: "This loop stops when you close this session. Set it up as a cloud schedule instead so it keeps running?"
- `header`: "Schedule"
- `options`: `[{label: "Cloud schedule (recommended)", description: "Runs in Anthropic's cloud even after you close this session"}, {label: "This session only", description: "Runs in this terminal until you exit"}]`

If they pick **Cloud schedule**: do NOT call CronCreate. Invoke the `schedule` skill directly via the Skill tool with `args` set to their original input verbatim (e.g. `Skill({skill: "schedule", args: "every morning tell me a joke"})`), then follow that skill's instructions to completion. Do NOT tell the user to run /schedule themselves. **Then stop — do not continue to any section below** (no CronCreate, no ScheduleWakeup, no "execute the prompt now").
If they pick **This session only**:
- If the trigger was a parsed ≥60-minute interval (rule 1 or 2): continue below with that interval.
- If the trigger was daily phrasing only (rule 3, no parsed interval): do NOT call CronCreate. Explain that a daily-cadence loop won't fire before this session closes, so there's nothing useful to schedule locally — suggest they either pick Cloud schedule, or re-run `/loop` with an explicit shorter interval (e.g. `/loop 1h <prompt>`) if they want a session loop. Then stop.
If neither trigger condition was met: continue below.

## Fixed-interval mode (rules 1 and 2)

Convert the interval to a cron expression:

| Interval pattern      | Cron expression     | Notes                                    |
|-----------------------|---------------------|------------------------------------------|
| `Nm` where N ≤ 59   | `*/N * * * *`     | every N minutes                          |
| `Nm` where N ≥ 60   | `0 */H * * *`     | round to hours (H = N/60, must divide 24)|
| `Nh` where N ≤ 23   | `0 */N * * *`     | every N hours                            |
| `Nd`                | `0 0 */N * *`     | every N days at midnight local           |
| `Ns`                | treat as `ceil(N/60)m` | cron minimum granularity is 1 minute  |

**If the interval doesn't cleanly divide its unit** (e.g. `7m` → `*/7 * * * *` gives uneven gaps at :56→:00; `90m` → 1.5h which cron can't express), pick the nearest clean interval and tell the user what you rounded to before scheduling.

Then:
1. Call CronCreate with: `cron` (the expression above), `prompt` (the parsed prompt verbatim), `recurring: true`.
2. Briefly confirm: what's scheduled, the cron expression, the human-readable cadence, that recurring tasks auto-expire after 7 days, and that the user can cancel sooner with CronDelete (include the job ID). Only if you did NOT show the cloud-offer AskUserQuestion above (i.e., neither trigger condition applied), end the confirmation with this exact line on its own, italicized: `_Runs until you close this session · For durable cloud-based loops, use /schedule_`. If the user already answered that question, omit this line.
3. **Then immediately execute the parsed prompt now** — don't wait for the first cron fire. If it's a slash command, invoke it via the Skill tool; otherwise act on it directly.

## Dynamic mode (rule 3 — no interval)

The user wants you to self-pace. Decide what makes the next iteration worth running — a passage of time, or an observable event.

1. **Run the parsed prompt now.** If it's a slash command, invoke it via the Skill tool; otherwise act on it directly.
2. **If the next run is gated on an event** (CI finishing, a log line matching, a file changing, a PR comment) and no Monitor is already running for it: arm one now with `persistent: true`. Its events arrive as `<task-notification>` messages and wake this loop immediately — you do not wait for the ScheduleWakeup deadline. Arm once; on later iterations call TaskList first and skip this step if a monitor is already running.
3. **Briefly confirm**: that you're self-pacing, whether a Monitor is the primary wake signal, that you ran the task now, and what fallback delay you're about to pick. Write this as text *before* calling ScheduleWakeup — the turn ends as soon as that tool returns.
4. **Then, as the last action of this turn, call ScheduleWakeup** with:
   - `delaySeconds`: with a Monitor armed this is the **fallback heartbeat** — how long to wait if no event fires (lean 1200–1800s; idle ticks past the 5-minute cache window are pure overhead). Without a Monitor this is the cadence — pick based on what you observed. Read the tool's own description for cache-aware delay guidance.
   - `reason`: one short sentence on why you picked that delay.
   - `prompt`: the full original /loop input verbatim, prefixed with `/loop ` so the next firing re-enters this skill and continues the loop. For example, if the user typed `/loop check the deploy`, pass `/loop check the deploy` as the prompt.
5. **If you were woken by a `<task-notification>`** rather than this prompt: handle the event in the context of the loop task, then call ScheduleWakeup again with the same `prompt` and the same 1200–1800s `delaySeconds` from step 4 — the Monitor remains the wake signal; this only resets the safety net.
6. **To stop the loop**, omit the ScheduleWakeup call and TaskStop any Monitor you armed (use TaskList to find the task ID if it is no longer in context). Before you stop, send a one-line outcome via PushNotification — the user may be away and waiting to hear it's done. Skip this if you're stopping because the user just told you to; they're already here.

## Input

<your /loop input is spliced in here>
```

### `ScheduleWakeup` 工具描述（dynamic 自定节奏模式的核心）

```text
Schedule when to resume work in /loop dynamic mode — the user invoked /loop without an interval, asking you to self-pace iterations of a specific task.

Do NOT schedule a short-interval wakeup to poll for background work you started — when harness-tracked work finishes, you are re-invoked automatically, so polling is wasted. Instead schedule a long fallback (1200s+) so the loop survives if the work hangs or never notifies. The exception is external work the harness cannot track (a CI run, a deploy, a remote queue) — there, pick a delay matched to how fast that state actually changes.

Pass the same /loop prompt back via `prompt` each turn so the next firing repeats the task. For an autonomous /loop (no user prompt), pass the literal sentinel `<<autonomous-loop-dynamic>>` as `prompt` instead — the runtime resolves it back to the autonomous-loop instructions at fire time. (There is a similar `<<autonomous-loop>>` sentinel for CronCreate-based autonomous loops; do not confuse the two — ScheduleWakeup always uses the `-dynamic` variant.) Omit the call to end the loop.

## Picking delaySeconds

The Anthropic prompt cache has a 5-minute TTL. Sleeping past 300 seconds means the next wake-up reads your full conversation context uncached — slower and more expensive. So the natural breakpoints:

- **Under 5 minutes (60s–270s)**: cache stays warm. Right for actively polling external state the harness can't notify you about — a CI run, a deploy, a remote queue.
- **5 minutes to 1 hour (300s–3600s)**: pay the cache miss. Right when there's no point checking sooner — waiting on something that takes minutes to change, genuinely idle, or as the long fallback heartbeat when something else is the primary wake signal.

**Don't pick 300s.** It's the worst-of-both: you pay the cache miss without amortizing it. If you're tempted to "wait 5 minutes," either drop to 270s (stay in cache) or commit to 1200s+ (one cache miss buys a much longer wait). Don't think in round-number minutes — think in cache windows.

For idle ticks with no specific signal to watch, default to **1200s–1800s** (20–30 min). The loop checks back, you don't burn cache 12× per hour for nothing, and the user can always interrupt if they need you sooner.

Think about what you're actually waiting for, not just "how long should I sleep." If you're polling a CI run that takes ~8 minutes, sleeping 60s burns the cache 8 times before it finishes — sleep ~270s twice instead.

The runtime clamps to [60, 3600], so you don't need to clamp yourself.

## The reason field

One short sentence on what you chose and why. Goes to telemetry and is shown back to the user. "watching CI run" beats "waiting." The user reads this to understand what you're doing without having to predict your cadence in advance — make it specific.
```

### `CronCreate` 工具描述（固定间隔 / 定时任务）

```text
Schedule a prompt to be enqueued at a future time. Use for both recurring schedules and one-shot reminders.

Uses standard 5-field cron in the user's local timezone: minute hour day-of-month month day-of-week. "0 9 * * *" means 9am local — no timezone conversion needed.

## One-shot tasks (recurring: false)

For "remind me at X" or "at <time>, do Y" requests — fire once then auto-delete.
Pin minute/hour/day-of-month/month to specific values:
  "remind me at 2:30pm today to check the deploy" → cron: "30 14 <today_dom> <today_month> *", recurring: false
  "tomorrow morning, run the smoke test" → cron: "57 8 <tomorrow_dom> <tomorrow_month> *", recurring: false

## Recurring jobs (recurring: true, the default)

For "every N minutes" / "every hour" / "weekdays at 9am" requests:
  "*/5 * * * *" (every 5 min), "0 * * * *" (hourly), "0 9 * * 1-5" (weekdays at 9am local)

## Avoid the :00 and :30 minute marks when the task allows it

Every user who asks for "9am" gets `0 9`, and every user who asks for "hourly" gets `0 *` — which means requests from across the planet land on the API at the same instant. When the user's request is approximate, pick a minute that is NOT 0 or 30:
  "every morning around 9" → "57 8 * * *" or "3 9 * * *" (not "0 9 * * *")
  "hourly" → "7 * * * *" (not "0 * * * *")
  "in an hour or so, remind me to..." → pick whatever minute you land on, don't round

Only use minute 0 or 30 when the user names that exact time and clearly means it ("at 9:00 sharp", "at half past", coordinating with a meeting). When in doubt, nudge a few minutes early or late — the user will not notice, and the fleet will.

## Session-only

Jobs live only in this Claude session — nothing is written to disk, and the job is gone when Claude exits.

## Not for live watching

CronCreate re-runs a prompt at fixed wall-clock intervals. To watch a log file, process, or command output and be notified the moment something changes, use the Monitor tool instead — Monitor streams events as they happen; cron polls on a schedule.

## Runtime behavior

Jobs only fire while the REPL is idle (not mid-query). The scheduler adds a small deterministic jitter on top of whatever you pick: recurring tasks fire up to 10% of their period late (max 15 min); one-shot tasks landing on :00 or :30 fire up to 90 s early. Picking an off-minute is still the bigger lever.

Recurring tasks auto-expire after 7 days — they fire one final time, then are deleted. This bounds session lifetime. Tell the user about the 7-day limit when scheduling recurring jobs.

Returns a job ID you can pass to CronDelete.
```
