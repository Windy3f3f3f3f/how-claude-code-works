# Chapter 17: Autonomy & Continuation — `/goal` and `/loop`

> From here on, a new module: features that postdate the source snapshot.
>
> The first sixteen chapters read the source snapshot that leaked in late March 2026, but `/goal`, `/loop`, dynamic workflows, and auto mode all postdate it, with no source to read. The only way to understand them is to switch methods: install the latest build and actually use it, capture the network requests it sends, and cross-check against the official docs. So anything below in quotes is text pulled that way; anything about "how it probably schedules internally" is reasoned back from behavior, and is flagged as such where it comes up. This chapter explains how `/goal` and `/loop` let Claude keep going on its own, with the method written out at the end so you can go capture the next feature yourself.

## 17.1 Two paradigms for "keep going"

By 2026, Claude Code is long past being just "one question, one answer." It has a whole family of ways to keep an agent going on its own — across turns, across time, across sessions. The two outermost, most-encountered entry points are `/goal` and `/loop`, and they happen to be two opposite ideas.

`/goal` is "fix on a condition and don't quit until it's met." You give it a completion condition, and it works round after round; at the end of each round a separate judge rules "met or not," and if not, Claude comes back for another round carrying the judge's reason, stopping only once it's met. It's passive: when to stop is the judge's call.

`/loop` is "set an alarm and come back." You give it an interval (or let it set its own pace), and it reruns the same thing on schedule. It's active: when to return is decided by the schedule, independent of whether the work got done.

One uses a "gatekeeper" to decide when to stop, the other an "alarm clock" to decide when to return — grasp that distinction and you've grasped the two spines of Claude Code's autonomy. We'll take them one at a time.

## 17.2 `/goal`: a gatekeeping judge

### It's syntactic sugar over a Stop hook

The official docs name the mechanism in a sentence: `/goal` is a wrapper around a session-scoped Stop hook. Each time a turn ends, the system sends "your condition + the conversation so far" to the configured evaluator model (officially the "small/fast" model, defaulting to Haiku; the model actually seen in one capture is discussed below), which returns "yes / no + a reason"; "no" sends Claude back for another round carrying that reason, "yes" clears the goal and records an achievement in the session record.

Traffic capture confirms the flow. The moment you set a goal, the main model's message contains this stretch, which all but writes the mechanism out in the open (a key fragment, with the tail elided):

> A session-scoped Stop hook is now active with condition: "<your condition>". Briefly acknowledge the goal, then immediately start (or continue) working toward it — treat the condition itself as your directive and do not pause to ask the user what to do. The hook will block stopping until the condition holds. It auto-clears once the condition is met…

"Setting a goal starts a round, treating the condition itself as the directive." That's why you don't send a separate prompt.

### The judge's verdict: three outcomes, and a loop brake

The interesting part is that judge. Capture its actual request and its system prompt reads, verbatim, as the block below — short enough to read in full, because an autonomous loop's whole sense of restraint is packed into these few lines:

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

The three outcomes — met, not met, judged impossible — are the three JSON shapes in that block. The first two are plain; the point is the third. `impossible` is a carefully designed loop brake, and a whole paragraph guards one thing: don't let the main agent talk the judge into quitting early. "The main agent saying it can't be done is only evidence, not proof; the judge confirms it independently, and when unsure returns `{"ok": false}` without `impossible`." What an autonomous loop fears most is exactly two things — never stopping, or being talked into quitting early — and this stretch of prompt is written against both at once. It even rules out "slow" by name: slow progress is not impossibility.

### Three details you only see by capturing the request

Take the judge's real request apart and three things surface that the docs don't mention.

First, the verdict is enforced at the API layer, not merely requested in the prompt. The request carries an `output_config` that constrains the output, via JSON schema, into exactly the shape `{ok, reason, impossible}` (`ok` and `reason` required, no other fields allowed). The prompt explains; the schema is the guardrail — even if the model wanted to freelance, it couldn't produce anything outside that shape.

Second, the judge gets no tools and only reads the conversation. The request's `tools` is empty. This confirms the official line that the judge doesn't call tools and can only judge what's already in the conversation: it's handed a transcript path, but has no tool to read that file.

Third, the judge runs at high reasoning effort — `effort: "high"` in the request. Judging "is it actually done" is something the system is willing to spend compute on.

On which model, one thing has to be said plainly: the docs say the judge uses "your configured small/fast model, which defaults to Haiku," but the model I captured was `claude-fable-5` — because this machine configured its small/fast model to that. So don't take one capture's model name for the default. And to be precise: it's the client-side hook runtime that assembles and sends this request locally; the model still runs on Anthropic's side, not on your machine.

### What it looks like in the trace

Every round of a `/goal` run lands in the session record: a `goal_status` entry carrying the condition, how many rounds it iterated, duration, how many tokens it burned, whether it was met. So "how many turns a goal spanned, and the state of each round" is fully replayable — a live example of the "always there" trace from the previous chapter (see [Chapter 16](16-observability.md)).

## 17.3 `/loop`: an alarm clock that schedules itself

`/loop` is fundamentally unlike `/goal`. It isn't a passive hook but a large orchestration prompt executed by the main model. You type `/loop …`, the system injects an instruction block beginning `# /loop — schedule a recurring or self-paced prompt`, and the main model itself parses it, picks a scheduling method, and calls generic scheduling tools. Put differently, `/loop`'s "smarts" live in a prompt, not in a hardcoded scheduler — though one caveat: the actual execution, lifecycle, and guardrails are still hardcoded in the runtime, as we'll see.

### How it parses your input

The captured orchestration prompt pins the parsing rules down verbatim — clearer read straight from the source than paraphrased:

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

The three rules are plain enough: check whether the start is an interval like `5m`, then whether there's a trailing `every 20m` clause, and if neither, the whole line is the prompt and it self-paces. The craft is in that `check every PR` counter-example — the prompt explicitly declares that "every" not followed by a time expression is not an interval, so "check every PR" isn't misread as "run once per PR." Writing the edge case, with its worked example, right into the prompt is itself a lesson.

### Three execution paths

Once the interval and prompt are parsed, which path to take is also the prompt's call.

If the interval is an hour or more, or you used day-level phrasing like "every morning" or "each night," the prompt has the model first pop a question asking whether you'd rather make it a cloud schedule (`/schedule`) that can run long-term independent of the session — because a plain `/loop` dies the moment the session closes. Only if you pick "this session only" does it continue.

To run on a fixed interval within the session, the model calls `CronCreate`. I captured a real call of this:

```text
CronCreate({"cron": "*/1 * * * *", "prompt": "…", "recurring": true})
```

The key part of the tool's reply reads: `Scheduled recurring job 7e7d1261 (Every minute). Session-only (not written to disk, dies when Claude exits). Auto-expires after 7 days. Use CronDelete to cancel sooner.` One sentence confirms several things — `1m` was converted to the standard cron expression `*/1 * * * *`, the job has an 8-character id, it lives only in the session and isn't written to disk (so it dies when Claude exits and leaves no litter), and it auto-expires after seven days.

One point worth calling out, because it's easy to get wrong: after creating the cron, the orchestration prompt has the model **run the parsed task once immediately** — verbatim, "Then immediately execute the parsed prompt now — don't wait for the first cron fire." So `/loop 1m X` schedules the timer and then runs `X` right away, rather than waiting a minute for the first run; cron takes over on schedule after that.

If you give no interval, it goes "self-paced": the model uses a tool called `ScheduleWakeup` to schedule its own next wake. That tool's description lays out the trade-off so clearly it's worth quoting:

> Do NOT schedule a short-interval wakeup to poll for background work you started — when harness-tracked work finishes, you are re-invoked automatically, so polling is wasted. Instead schedule a long fallback (1200s+) so the loop survives if the work hangs or never notifies.

In plain terms: don't keep short-polling for background work the harness will notify you about on completion — that's wasted; schedule a longer fallback wake (upward of a thousand seconds) only as a hang guard, and pass the same `/loop` prompt back through a parameter each time so the next wake continues the same task. A tool that lets the model set its own alarm has to first teach the model not to set it too often — which is exactly the pitfall self-scheduling walks into.

Four inputs, four paths — clearer side by side:

| You type | Path | First step | How it continues |
|---|---|---|---|
| `1m X` (interval) | session cron | run `X` once now | cron re-fires `X` on schedule |
| `X` (no interval) | self-paced | run `X` once now | `ScheduleWakeup` / `Monitor` schedules the next |
| empty | built-in maintenance / `loop.md` | run the default maintenance prompt | same, on its own cadence |
| interval ≥1h or "every…" | ask first | offer a cloud schedule | only continues the above if you pick this-session |

### Give it nothing, and it runs a maintenance script

If `/loop` is left empty, then per the official docs it runs a built-in maintenance prompt (or your own `loop.md`): continue unfinished work first, tend the current branch's PR, and do some cleanup when idle, keeping to one floor — no new initiatives, and irreversible actions like push or delete proceed only where the conversation already authorized them. This is the docs' account: the binary has adjacent babysit/autofix fragments, but they aren't enough to pin "fix failed CI, answer reviews, resolve merge conflicts" verbatim as the general default of a bare `/loop` — so it's relayed from the docs, not presented as behavior confirmed from the artifact.

### The scheduling daemon behind it

Behind the local `/loop`, cron, dynamic wakeups, and notification paths sits a family of telemetry from one background daemon (internally, KAIROS). Whether cloud schedules are also run by this client-side daemon isn't clear from the evidence — and the official line is that cloud runs on Anthropic's side — so treat KAIROS here as part of the local scheduling/wakeup substrate only. Its internal scheduling and persistence logic is pure client-side background code: unreachable, only inferable.

The precise framing is this: `/loop`'s high-level decisions (how to parse, which path to take, when to converge) are prompt-driven, but execution, lifecycle, and guardrails are hardwired in the runtime — `ScheduleWakeup`'s delay is clamped to a fixed range, plus keepalive fallback, age-out on inactivity, the cron store, the scheduler itself, none of which the model can change. The prompt handles "decisions," the runtime handles "backstops."

One last thing worth noting, because it's an instance of "the docs and reality don't line up." On jitter (randomly offsetting fire times so a pile of sessions don't all hit the API in the same second), the official docs say a recurring task can be delayed up to half an hour; but on my v2.1.201 machine, `CronCreate`'s own tool description says "10% of the period late, up to 15 minutes." The two don't match — probably a version difference — so cite the source when you rely on it.

## 17.4 Two entry points, one family of substrate

`/goal` and `/loop` are only the two outermost entry points, and set side by side they reveal the two dimensions of this whole family of autonomy: what triggers the next round, and what does the scheduling. The other members named below are neighboring capabilities in the same family, not each one verified by capture in this chapter — here we just sketch how they relate to `/goal` and `/loop`, leaving the full analysis to later chapters.

On the trigger side, beyond the Stop hook that `/goal` wraps, there's auto mode — which auto-approves tool calls one by one, removing every step's confirmation so each continuation round can run unattended. On the scheduling side, beyond the cron and `ScheduleWakeup` that `/loop` uses, there's `Monitor` (watch a long-running script and push its output back as it comes, cheaper than polling), and the KAIROS daemon that ties these together. Further out still are cloud schedules (running on Anthropic's side, alive after the session closes), background agents (`/bg`, `claude agents`), and `--resume` bringing back both unfinished goals and unexpired scheduled tasks.

The family in one line: a background scheduling substrate, plus a set of generic scheduling tools, plus prompt orchestration. `/goal` and `/loop` are two pieces of syntactic sugar this substrate exposes; later chapters take on the rest of the family.

## 17.5 Appendix: how we know this (the reverse-engineering method, reproducible)

This chapter's verified facts come from two layers of method. The full process is below — anyone can reproduce it, and you can even hand this section straight to Claude Code and have it redo the work for you.

The first layer is static extraction, zero cost and doable the day a version ships. One caveat: since 2.1.x, Claude Code is no longer a plaintext `cli.js` but compiled into a native binary (packed with Bun). The JS source is still embedded as strings, though, so `strings` recovers it. Dump all the strings, then `grep` for a feature's telemetry event names (a feature map) and for its prompt and command bodies. One pitfall: a `.{0,N}` regex on this hundreds-of-thousands-of-lines file backtracks wildly and times out — use fixed-string `grep -F`. This layer gets you prompt fragments, tool descriptions, and event names, all verbatim; but how those strings assemble, and the control flow, it can't give — those are inference.

The second layer is capturing real network requests, to see the fully assembled runtime payload (system prompt, messages, tools, model, `output_config`, all of it). The way to do it is to stand up a plaintext reverse proxy: let `claude` talk plain HTTP to localhost (which sidesteps the certificate hassle), have the proxy record the request body, then forward to the real API. Run it from a "trusted" directory (`/goal`-type hook features require the working directory to have accepted a trust dialog), point `ANTHROPIC_BASE_URL` at the local proxy, then `claude -p "/goal …"`, and among the captured requests find the one containing "stopping condition" — that's the judge's. Smoke-test with `-p "reply with PONG"` first to confirm the proxy works and captures, then run the target feature; pick a probe condition that's satisfiable in one line and converges fast, don't touch tool permissions, and don't burn tokens on a long run.

Two more layers are easy add-ons. The session record: after a run, just read the JSONL under `~/.claude/projects/`, which has `goal_status`, per-step token usage, and the parent-child chain between messages — trace-layer facts, zero cost. OTel telemetry: turn on `CLAUDE_CODE_ENABLE_TELEMETRY` with a console exporter and run once, and you can watch those internal `tengu_` events actually fire, exported outward as `claude_code.*` metrics.

Last, the discipline. Prompts, payloads, and strings are facts; internal scheduling and gating logic is inference, and the latter is never cited alone as fact. One-off observations like a model name or a duration get tagged "this capture." Enumerate event names in full rather than by regex shorthand (`a|b|c` can, once compressed, spell out an identifier that doesn't exist). And a security note: a captured payload carries account, device, and session ids, plus prompts and tool results that may contain secrets — keep it private and redact before quoting.

To reproduce, one sentence hands it to Claude Code: "Read the method in this section and reproduce the reverse engineering of `/goal`'s judge — `strings` the local claude binary for goal-related strings; write a plaintext reverse proxy, run `claude -p '/goal …'` through it, and capture the judge's request; extract the judge's system prompt, `output_config`, `tools`, and model, cross-check against the official `/goal` docs, and label what's fact versus inference." That's the whole method; you can now go capture the next new feature yourself.

---

## Appendix: full prompts (verbatim)

The body quotes only the key parts; below is the complete text taken verbatim from the capture, for anyone who wants to dig in or reproduce it. Source is a plaintext reverse-proxy capture of the 2.1.201 client; injection slots (your condition, your input) are replaced with placeholders, and nothing else is changed.

### The `/goal` stop-condition evaluator's system prompt

```text
You are evaluating a stop-condition hook in Claude Code. Read the conversation transcript carefully, then judge whether the user-provided condition is satisfied.

Your response must be a JSON object with one of these shapes:
- {"ok": true, "reason": "<quote evidence from the transcript that satisfies the condition>"}
- {"ok": false, "reason": "<quote what is missing or what blocks the condition>"}
- {"ok": false, "impossible": true, "reason": "<explain why the condition can never be satisfied>"}

Always include a "reason" field, quoting specific text from the transcript whenever possible. If the transcript does not contain clear evidence that the condition is satisfied, return {"ok": false, "reason": "insufficient evidence in transcript"}.

Only use {"ok": false, "impossible": true} when the condition is genuinely unachievable in this session — for example: the condition is self-contradictory, it depends on a resource or capability that is unavailable, or the assistant has explicitly tried, exhausted reasonable approaches, and stated it cannot be done. Apply your own judgment when deciding this — the assistant claiming the goal is impossible is evidence, not proof; independently confirm the condition is genuinely unachievable rather than deferring to the assistant's self-assessment. Do not use it just because the goal has not been reached yet or because progress is slow. When in doubt, return {"ok": false} without "impossible".
```

### The `/goal` Stop-hook activation message injected into the main model (`<your condition>` is your condition)

```text
A session-scoped Stop hook is now active with condition: "<your condition>". Briefly acknowledge the goal, then immediately start (or continue) working toward it — treat the condition itself as your directive and do not pause to ask the user what to do. The hook will block stopping until the condition holds. It auto-clears once the condition is met — do not tell the user to run `/goal clear` after success; that's only for clearing a goal early.
```

### The `/loop` command orchestration prompt (the `## Input` section is where your input is spliced in)

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

### The `ScheduleWakeup` tool description (core of dynamic self-paced mode)

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

### The `CronCreate` tool description (fixed-interval / scheduled jobs)

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
