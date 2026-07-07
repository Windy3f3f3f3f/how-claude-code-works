# Chapter 19: Dynamic Workflows — Orchestrating an Agent Fleet with a Deterministic Script

> Chapter 8 covered Claude Code's multi-agent modes: a main agent spawning subagents, a coordinator dispatching workers, Swarm peer-to-peer. But those forms of orchestration share one thing — who to spawn, when, and when to merge are all decided by the model on the fly, in its reasoning. For genuinely big work (rolling a change across dozens of files, auditing a whole dependency chain, verifying a large batch of candidates one by one), what you want isn't "the model scheduling as it thinks" but deterministic orchestration: a script that fans out dozens to hundreds of subagents at once, with a concurrency cap, a token budget, and resume-from-checkpoint. This is Dynamic Workflows, triggered by the keyword `ultracode`.
>
> The main evidence for this chapter is the description of the Workflow tool — the block injected into the main model that teaches it how to write a workflow script. It's right there in the tool set the runtime injects, and its key sentences (the concurrency cap, the item cap, the total-agent cap) line up verbatim with the 2.1.201 binary, so the copy we have is the real thing. Add the runtime constants leaked in the binary and a family of telemetry events, and this chapter lands on fairly solid ground. As for exactly how any one workflow branches and converges — that's written in the script, authored by the model on the fly, and is the part we can't get; more on that below.

## 19.1 Two kinds of orchestration: the model as coordinator, or a script as coordinator

Start by connecting to Chapter 8. Its three multi-agent modes — subagent, coordinator, Swarm — are all fundamentally model-driven orchestration: the main model or coordinator decides, in each round of reasoning, "who to dispatch now, and how to synthesize once results come back." The "prompt design essentials" in the coordinator section are about using a prompt to guide the model through that scheduling judgment well.

Dynamic Workflows takes a different tack: replace the "model acting as coordinator" with a deterministic JS script. How many agents to fan out, how their results chain, when to converge — all written into the script's control flow, not decided by the model round by round. And the underlying primitives — how a subagent is spawned, how an async task reports back via `<task-notification>`, how parallel file writes are isolated with worktrees — are reused unchanged from Chapter 8 (see [Chapter 8](07-multi-agent.md), 8.2, 8.3).

Set against Chapter 17, the two are two different axes. `/loop` is serial continuation on the time axis: the same main model comes back to do another round after a while. A workflow is parallel fan-out on the space axis: one script plans out dozens to hundreds of agents, which the runtime then runs in parallel, in batches up to the concurrency cap. They share the same philosophy — the upper layer expresses "what to do" in a script or prompt, the runtime underneath handles "how to schedule and how to stay alive" — but they aren't the same thing, and aren't on the same axis.

This "script as coordinator" design also explains the boundary of reverse-engineering it: what you can get is the tool description that teaches the model to write scripts, plus each spawned agent's request; what you can't get is any one run's exact branch logic — because it lives in the script the model wrote on the fly that time, not in a fixed prompt.

## 19.2 What a workflow script looks like

What the Workflow tool does is have the main model write a JS script on the fly, handed to the runtime to execute deterministically. The script must open with a `meta` declaration — a pure literal, stating the name, a one-line description, and which phases it has (these show up in the permission dialog and the `/workflows` panel):

```javascript
export const meta = {
  name: 'review-changes',
  description: 'Review changed files across dimensions, verify each finding',
  phases: [{ title: 'Review' }, { title: 'Verify' }],
}
```

After `meta` comes the script body, orchestrated with a set of built-in hooks:

- `agent(prompt, opts?)` — spawn a subagent and return its result. Without a schema it returns the subagent's final text; with a `schema` it returns a validated object (next section). `opts` can set its label, which phase it belongs to, which model, how much reasoning effort, whether to isolate it in a worktree, and which agent type.
- `pipeline(items, stage1, stage2, …)` — run each item independently through a series of stages.
- `parallel(thunks)` — run a batch of tasks concurrently, awaiting all of them.
- `phase(title)` groups, `log(msg)` prints progress to the panel, `budget` checks the token budget, and `workflow(nameOrRef, args?)` runs another workflow inline — a saved workflow or a script path; the child shares this run's concurrency cap, agent counter, abort signal, and token budget, and nesting is one level only.

That `agentType`, by the way, is resolved from the same registry as the Agent tool and composes with `schema` — so you can spawn a custom-typed subagent and still force it to return a structured result. The script is plain JS, not TS (writing type annotations fails to parse), and `Date.now()`, `Math.random()`, and argless `new Date()` are all disabled — because they would make "resume from checkpoint" (19.6) compute different results. Those restrictions themselves reveal the design's stance: the script has to be replayable, so no nondeterminism allowed.

A real workflow looks like this — review a batch of changed files, and adversarially re-check each finding as it comes:

```javascript
const results = await pipeline(
  DIMENSIONS,
  d => agent(d.prompt, { phase: 'Review', schema: FINDINGS }),
  review => parallel(review.findings.map(f => () =>
    agent(`Adversarially verify: ${f.title}`, { phase: 'Verify', schema: VERDICT })
      .then(v => ({ ...f, verdict: v })))),
)
```

What exactly is a spawned subagent? Capturing a real workflow's network requests makes it plain. It's an independent client-side request, and the first line of its system prompt is "You are a subagent spawned by a workflow orchestration script," with a pointed note: your final text is returned verbatim as a string to the calling script — it's your return value, not a message to a human. It's also a leaf worker. The tools the main model has for further orchestration and scheduling — `Agent`, `Workflow`, `ScheduleWakeup`, plus `TaskOutput` — are stripped from a subagent's tool set (in the capture the subagent had 76 tools to the main model's 80, and those four are the difference). So it can only do the work, not fan out again. And to the main model, the whole Workflow call is a background task: it returns a task id immediately, and a `<task-notification>` arrives when the workflow completes.

## 19.3 Default to pipeline; parallel is a barrier — why

This is the finest bit of the whole design, and the part the tool description spends the most space on: when to use `pipeline` and when to use `parallel`.

`parallel` is a barrier: it runs a batch of tasks concurrently, then awaits all of them before returning. `pipeline` has no barrier: each item passes through all stages independently, and item A can already be at stage 3 while item B is still at stage 1. The difference in wall-clock time is very real — a `pipeline`'s total is roughly "the slowest single-item chain," not "the sum of the slowest in each stage." If 5 finders run in parallel and the slowest takes three times the fastest, a barrier wastes two-thirds of the fast finders' idle time.

So the tool description states it flatly: default to `pipeline`. There's only one case for a barrier — stage N genuinely needs "all of the previous stage's results together," such as deduping the full result set before an expensive downstream step, or "skip the whole round on zero findings," or a stage whose prompt references "all the other findings" for comparison. It even gives a test: if what you wrote is "run a batch concurrently → flatten/map/filter the results → run another batch concurrently," that middle step doesn't need a barrier at all; fold it into a pipeline stage.

A system that turns the programming model into a tool and then, in the tool description, teaches the model "don't set barriers carelessly, and when in doubt use pipeline" is really heading off a classic concurrency anti-pattern on the model's behalf.

One more thing on error semantics: if a task in `parallel` throws, that slot becomes null (so `.filter(Boolean)` before using the results); if a stage in `pipeline` throws, that item drops and skips its remaining stages. Failures are isolated to a single point rather than sinking the whole batch.

## 19.4 Getting subagents to return structured data

For multiple agents to chain into a pipeline, one's output has to feed the next. Plain text is awkward — you'd have to parse it. Workflow's answer is to give `agent()` a `schema` (a JSON Schema). With it, the runtime injects a tool called `StructuredOutput` into that subagent's tool set, with your schema set as that tool's input schema; its description says "You MUST call this tool exactly once at the end." The subagent has to call it once when done, its argument is validated against your schema on the spot (`required` and `additionalProperties: false` both enforced), and `agent()` gets a validated object, no parsing on your end. (This mechanism was confirmed by capturing a real schema workflow and seeing that `StructuredOutput` tool in the subagent's request.)

The key is that validation happens at the tool-call layer: on the success path, `agent()` gets a validated object directly; a wrong shape triggers a nudge and retry, but if the subagent never calls it or exceeds the structured-output retry cap, that `agent()` call fails — inside a `parallel`/`pipeline` slot the failure is isolated to null, while a direct `await agent(...)` throws. So chaining like "the previous agent emits a batch of `{file, line, severity}`, the next agent verifies each" is reliable on the success path — what flows between stages is structured data, not strings waiting to be regexed. In the 19.2 example, the two schemas `FINDINGS` and `VERDICT` are exactly how the review stage and the verify stage connect.

## 19.5 Three caps and a budget

Once a script can fan out hundreds of agents, there have to be guardrails, or one mistyped loop can blow up the machine and the bill together. The runtime imposes three hard caps, spelled out verbatim in the tool description:

- Concurrency: the `agent()` calls running at once are clamped to `min(16, cpu cores - 2)`; excess ones queue and run as slots free up.
- Per call: one `parallel()` or `pipeline()` takes at most 4096 items; more than that is an outright error, not a silent truncation.
- Total: one workflow spawns at most 1000 agents over its whole lifetime (in the runtime it's a constant, `1000`), which the description calls "a runaway-loop backstop, set far above any real workflow."

The fourth thing isn't a cap but a budget. If you set a token target for this turn (that `+500k`-style directive), `budget` turns it into a hard ceiling: `budget.total` is the target, `budget.spent()` is spent so far, `budget.remaining()` is what's left; once it hits the top, further `agent()` calls throw. This lets a script scale its depth to the budget dynamically — the description's pattern is `while (budget.total && budget.remaining() > 50_000) { … }`, spending while watching what's left to decide whether to dispatch another round. Hitting the cap or the budget is recorded as `tengu_workflow_agent_cap_exceeded` and `tengu_workflow_budget_cap_exceeded` respectively.

## 19.6 Isolation and resume-from-checkpoint

With hundreds of agents working in parallel, two things go wrong: they overwrite each other editing the same file; and if a job this big dies halfway, starting over is a waste. Workflow has a handle on each.

Isolation is via worktrees. Pass `isolation: 'worktree'` to `agent()` and the runtime gives that agent its own git worktree to edit in its own copy, auto-removing it if unchanged when done and handing back the path if changed. This is exactly Chapter 8's worktree isolation substrate (see 8.2), reused directly. The description also warns it's expensive — a few hundred milliseconds plus a disk copy per agent — so use it only when "parallel writes would conflict."

Resume is via a journal and a runId. Each run has an id (like `wf_000d0acb-383`), and every `agent()` writes two lines to the journal — a `started` and a `result` — both carrying a `v2:<hash>` key that is the content hash of that call (prompt plus opts). On a rerun with `resumeFromRunId`, the longest prefix of `agent()` calls whose key is unchanged hits last time's stored results from the journal, and only from the first call whose hash changed does it actually rerun. (All of this is visible in what a real workflow run leaves on disk: the script the model wrote is itself saved as an editable `.js` file, the run directory `<session>/subagents/workflows/wf_<runId>/` holds `journal.jsonl`, and each subagent's full transcript is saved alongside as `agent-<id>.jsonl`.) The binary shows the skeleton of this: the runtime registers and manages this `local_workflow` task by `workflowRunId`, and cache hits are served from the same runId's journal (related events `tengu_workflow_journal_started_hit_respawn`, `tengu_workflow_saved`). So a 100-agent migration that dies at agent 60 resumes without redoing the first 59. This also explains why 19.2 forbids `Date.now()`/`Math.random()` — the script must be replayable, or the cache hits won't line up.

## 19.7 Orchestrating reliability into the process

The genuinely interesting thing about Workflow isn't just "parallel is fast" — it's that it writes a set of confidence-seeking orchestration patterns into the tool description and encourages the model to build them. A few typical ones:

- Adversarial verify: for each finding, spawn N independent skeptics, each prompted to refute it; kill the finding if a majority's refutations hold. This guards against a "sounds plausible but doesn't hold up" finding slipping through.
- Perspective-diverse verify: when a finding can fail in more than one way, give each verifier a different lens (correctness, security, performance, does-it-reproduce) instead of N identical ones.
- Judge panel: generate N approaches independently from different angles, score them with parallel judges, synthesize from the winner while grafting the runners-up's best ideas.
- Loop-until-dry: for findings of unknown count (bugs, edge cases), keep spawning finders until K consecutive rounds turn up nothing new — a plain "run N rounds" misses the long tail.
- Completeness critic: at the end, spawn an agent whose whole job is to ask "what's missing — which search wasn't run, which claim wasn't verified, which source wasn't read"; what it finds becomes the next round of work.

Together these patterns make a plain claim: the value of multiple agents isn't speed but that structured redundancy can systematically correct for "a single agent easily overlooking things."

These patterns aren't hypothetical, either. Claude Code's built-in `/deep-research` is a workflow built on exactly them; the phase structure its metadata exposes in the binary reads: first Scope, decomposing the question into 5 search angles; then Search, 5 parallel WebSearch agents, one per angle; then Fetch, deduping URLs, fetching the top 15 sources, extracting falsifiable claims; then Verify, 3-vote adversarial verification per claim (2 of 3 refutes to kill); then Synthesize, merging semantic duplicates, ranking by confidence, adding citations. Count them off: the multi-path search is a multi-modal sweep, the 3-vote / 2-of-3 is adversarial verify, and the URL-dedup at the fetch step is exactly the "gather everything first" that needs a barrier — one workflow uses nearly all the patterns from the earlier sections.

## 19.8 ultracode, the panel, deep-research, and the off switch

You won't usually hand-write a workflow script. Two entry points involve the word `ultracode`, and they're worth telling apart. One is including it in a prompt — that opts this turn into the Workflow tool, so the model writes a workflow itself and runs it, breaking a big task apart and fanning it out, just for this turn. The other is the session-level `/effort ultracode`, which is what the official docs describe as "xhigh reasoning + standing automatic workflow orchestration," on for the whole session. The `/workflows` panel shows each workflow's live progress (the phases in `meta` and the messages from `log()` show up here). `/deep-research` is an officially bundled workflow — multi-path search, deep reading, adversarial verification, then synthesis into a cited report — itself a finished product of this orchestration capability. The whole feature can be turned off with `CLAUDE_CODE_DISABLE_WORKFLOWS=1`.

## 19.9 From ultraplan to dynamic workflows: an evolution you can reconcile

This capability, too, can be reconciled across two ends. The snapshot end (v2.1.88) already has its predecessor — the `utils/ultraplan/` directory: a `keyword.ts` that detects the trigger word `ultraplan` in your input (careful enough to skip false positives inside quotes, inside paths, after a question mark, or in a slash command, its shape aligned with the "extended thinking" trigger detection), and a `ccrSession.ts` handling the cloud-session side: `/ultraplan` teleports to a remote session at launch and sets it to plan mode, after which `ccrSession` polls the event stream every 3 seconds, waits for you to approve an `ExitPlanMode` in the browser, and extracts the plan text. In other words, back then the feature was called `ultraplan`, triggered by a keyword, and did "run plan mode in the cloud, wait for your approval, retrieve a plan."

The current end (2.1.201): the trigger word became `ultracode`, the telemetry went from a `tengu_ultraplan_*` family to a `tengu_workflow_*` family, and the capability grew from "produce a plan via the cloud" into "a deterministic JS script fanning out an agent fleet." The keyword-trigger machinery carried straight over, while the fan-out orchestration runtime is what grew after the snapshot. The trigger skeleton comes from the snapshot source, the finished orchestration from the current binary — another line showing "didn't leak with the snapshot" doesn't mean "can't be analyzed."

## 19.10 Appendix: how we know this (the reverse-engineering method, reproducible)

This chapter's evidence comes from a few reproducible routes.

First, read the tool description. Workflow is a tool injected into the main model, and its description is the full manual for "how to write a workflow script" — the programming model, the semantics of pipeline versus parallel, the three caps, the budget, isolation, resume, and that set of quality patterns, all in it. It's in the runtime's tool set, and its key sentences (`min(16, cpu cores - 2)`, `at most 4096 items`, `capped at 1000`) line up verbatim with the 2.1.201 binary — the two corroborate.

Second, pull strings and runtime constants from the binary. `grep` all the `tengu_workflow_*` and `tengu_ultraplan_*` event names for a feature map; the minified runtime code also yields the cap constant `1000` and the skeleton of the resume mechanism (`workflowRunId` / `resume` / the task registry).

Third, capture the fan-out. Stand up a plaintext reverse proxy (recipe in [Chapter 17](17-autonomy-goal-loop.md), 17.5) and run a very small workflow, and you'll see the subagent requests one script dispatches in parallel. This chapter actually ran two probes. The 2-agent parallel one captured the two subagents' request timestamps differing by only 0.01 seconds — confirming concurrent dispatch — along with the "You are a subagent spawned by a workflow orchestration script" system prompt. The schema one showed the `StructuredOutput` tool injected into the subagent's tool set, its input schema being the schema you gave. Mind the scale: a workflow spawns lots of agents and burns tokens, so keep the probe to two or three agents, read-only tasks, and stop when done.

Fourth, read the snapshot predecessor. `utils/ultraplan/` is in the v2.1.88 snapshot, used to reconcile the ultraplan → dynamic-workflows evolution.

An honest boundary: the tool description, the caps, the events, and the resume mechanism are all readable or directly confirmable; but any one workflow's exact convergence and branch logic lives in the script the model wrote on the fly, not a fixed prompt, and the internals of the runtime's concurrency scheduler, as well as the server side, are out of reach. Which model tier `ultracode` runs behind varies with rollout, so the text doesn't hard-code it.

---

## Appendix: the Workflow tool description (key excerpts)

The body quotes only the highlights. Below are excerpts, marked for which are verbatim and which are condensed: the caps and pipeline/parallel sections are verbatim (cross-checked sentence by sentence against the 2.1.201 binary); the programming-model skeleton and pattern list are faithfully condensed — original meaning and key phrases kept, full type signatures and details (like `agent()`'s null return and `workflow()`'s sharing and one-level nesting) dropped, since the body covers them. The source is the runtime-injected Workflow tool description — the original that teaches the main model to write workflows.

### Script skeleton and hooks

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

### pipeline vs parallel

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

### Caps and budget

```text
Concurrent agent() calls are capped at min(16, cpu cores - 2) per workflow —
excess calls queue and run as slots free up. … Total agent count across a
workflow's lifetime is capped at 1000 — a runaway-loop backstop set far above any
real workflow. A single parallel()/pipeline() call accepts at most 4096 items.

budget: {total, spent(), remaining()} — the turn's token target … a HARD ceiling:
once spent() reaches total, further agent() calls throw.
  while (budget.total && budget.remaining() > 50_000) { … }
```

### A set of quality-orchestration patterns (list; details elided)

```text
- Adversarial verify: spawn N independent skeptics per finding, each prompted to
  REFUTE. Kill if ≥majority refute.
- Perspective-diverse verify: give each verifier a distinct lens.
- Judge panel: generate N independent attempts, score with parallel judges,
  synthesize from the winner.
- Loop-until-dry: keep spawning finders until K consecutive rounds return nothing.
- Multi-modal sweep / Completeness critic: "what's missing?" becomes next round.
```

### The subagent system prompt and the StructuredOutput tool (verbatim, reverse-proxy capture, 2.1.202)

The head of a workflow-spawned subagent's system prompt:

```text
You are a subagent spawned by a workflow orchestration script. Use the tools available to complete the task.

CRITICAL: Your final text response is returned **verbatim** as a string to the calling script — it is your return value, not a message to a human. Output the literal result (data, JSON, text). Do NOT output confirmation…
```

When a `schema` is passed, the `StructuredOutput` tool injected into the subagent's tool set (its `input_schema` being the caller's schema):

```text
StructuredOutput — Use this tool to return your final response in the requested
structured format. You MUST call this tool exactly once at the end of your
response to provide the structured output.
```

> Provenance: the tool-description excerpts are from the runtime-injected Workflow tool; sentences like `min(16, cpu cores - 2)` / `at most 4096 items` / `capped at 1000` are cross-checked verbatim against the 2.1.201 binary, while the subagent system prompt and the `StructuredOutput` tool come from a plaintext reverse-proxy capture of a probe workflow actually run on the 2.1.202 client. Examples and pattern details are elided with `…`. Any one workflow's actual script, the internals of the runtime's concurrency scheduler, and the model tier behind `ultracode` are all out of view.
