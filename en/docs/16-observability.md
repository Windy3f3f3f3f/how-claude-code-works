# Chapter 16: Observability — Making a Whole Task Traceable

> A distributed database can run EXPLAIN on a single SQL query and break it into operators: which steps it took, how many rows each read, how long each took, whether there was skew. Claude Code does the same thing to itself. You hand it a task; it completes it across several turns; and which APIs each turn called, which tools it ran, how many tokens and dollars it spent, where it waited for you to approve a permission, whether it hit an error and retried — all of it is recorded, and can be stitched back into one chain and replayed. This chapter is about how it does that, and why it's built this way.

## 16.1 Three kinds of question, four kinds of record

Start with the problem. When you ask Claude Code to fix a bug, it doesn't just "call the model once" — it runs a string of turns: read files, locate, edit, wait for your approval, execute, run tests, summarize. Afterward you might want to ask three very different things:

- How much did the team spend this month, in dollars and tokens? You want numbers you can add up, and you don't care which prompt produced them.
- Who approved that `git push` last week? You want discrete records you can filter by field.
- Of the seven or eight steps in this bug fix, which was slowest, and where did it stall? You want a tree with parent-child relationships and durations — the operator tree from EXPLAIN.

Three questions, three completely different data structures: an aggregatable counter, a queryable event stream, a nested trace tree. Cram them all into one big log and you can't answer any of them well. So Claude Code splits observability into layers, each doing one job.

There are four. The first three are the standard OpenTelemetry trio — metrics, events, traces — and the fourth is a session record it writes to disk on every run.

- Metrics answer the "grand total" questions: a handful of monotonic counters — session count, tokens, cost, lines of code changed. To be aggregated efficiently by a time-series database, they deliberately carry only coarse dimensions (model name, token type) and never the ID of an individual prompt: that would blow up cardinality and drag the database down. This "the aggregation layer refuses high-cardinality IDs" line recurs throughout.
- Events answer the "which specific thing" questions. Every API call, every tool invocation, every permission decision leaves one discrete, timestamped record you can filter by prompt, by tool, by outcome.
- Traces answer the "whole chain" questions. A task becomes a tree: one interaction is the root, and under it hang each turn's model call, each tool execution, and the stretch spent "blocked waiting for your approval," every node carrying its duration. This is the layer closest to EXPLAIN — but it's off by default and still beta, for reasons below.
- The transcript is the trace that's "always there, and everyone has it." The first three layers need you to turn telemetry on and point it at a backend; the transcript is written on every run to a JSONL file under `~/.claude/projects/`, capturing every message, every token usage, and the parent-child links between messages. `/resume` reads it; so do nearly all community analysis tools. For most people, this is the trace they can actually use.

### Stitching one prompt's data back into a chain across four layers

Each layer records its own thing, but when you're debugging you want "everything about this one prompt." Aligning all four to the same prompt relies on one correlation anchor: a UUID.

When your input arrives, CC generates this UUID (the `prompt.id`), stores it in global state, and then, in the same spot in the code, does two things — opens the root of a trace tree, and emits a `user_prompt` event. So the trace and the events are hung on the same prompt from the very start:

```typescript
// utils/processUserInput/processTextPrompt.ts:31
const promptId = randomUUID()
setPromptId(promptId)
startInteractionSpan(userPromptText)          // open the root of the trace tree
void logOTelEvent('user_prompt', { /* … */ 'prompt.id': promptId })
```

But the four layers use this ID differently, and this is worth stating clearly, or you'll assume it's the same field spread flat across all four:

- Events: every event automatically carries `prompt.id` (`events.ts:49`). This is its real correlation key — all of one prompt's events are gathered by it.
- Transcript: `prompt.id` is stamped only on the "user message" rows (the input row, and the tool-result rows fed back in — both count internally as `user` type); assistant rows don't carry it. So it's an anchor for locating this prompt's starting point in the local record, not a grouping key that sweeps up every related message.
- Trace: spans don't carry `prompt.id` at all. Parent-child structure inside an interaction isn't expressed by an ID but by "which span is whose parent" — a tree, all hanging under the `claude_code.interaction` root (see 16.4). It refers to the same prompt as the events and the record purely because all three are opened at the same spot in the code.
- Metrics: deliberately carry no prompt-level ID at all (the `events.ts:49` comment says outright: otherwise, unbounded cardinality).

Discrete layers correlate by ID, the causal layer by tree membership, the aggregation layer refuses IDs — this dividing line is the first principle of the whole design. Hold onto it; every layer below is applying it.

The next four sections take the four layers one at a time.

---

## 16.2 Metrics: one switch, two pipelines

Metrics have a detail that's easy to miss: they feed two unrelated pipelines at once, each controlled by a different switch.

One is for you. Set `CLAUDE_CODE_ENABLE_TELEMETRY` and CC sends metrics over standard OTLP to your own Prometheus or collector (`instrumentation.ts:324`), which enterprises use to build usage dashboards for their teams. The other is for Anthropic: for API customers, enterprise, and team plans, metrics go into Anthropic's own BigQuery (`isBigQueryMetricsEnabled`, `instrumentation.ts:336`) for product analysis.

The two are independent. Even if you never set `CLAUDE_CODE_ENABLE_TELEMETRY`, the BigQuery pipeline may still be running — it's controlled separately by an organization-level opt-out. Grasp this, and you won't mistake "I didn't turn on telemetry" for "nothing was reported."

### Don't let reporting drag down the main flow

Before each export, the BigQuery pipeline has to confirm "is this organization allowed to report?" The naive version calls an API on every export — but a one-shot invocation like `claude -p` can run hundreds of times a day, and neither the startup path nor the network can take that. So CC uses two levels of cache to compress it to "roughly one API call a day": an in-process, one-hour-TTL memory cache to dedupe within a process, plus a 24-hour-TTL disk cache to survive across processes (`metricsOptOut.ts:22`; the comment says outright that this cache collapses N `claude -p` invocations into ~1 API call/day). A fresh cache returns with zero network; only a stale one triggers a background async refresh. On top of that sits a circuit breaker: the moment "non-essential traffic is globally disabled," it returns not-enabled and stops the export right at the consumer (`metricsOptOut.ts:57`).

Behind this is a generalizable engineering judgment: observability itself must not become a burden. An observability component has to be cheap to turn off, tolerant of reading a stale value, and non-blocking by default.

### Eight metrics, a stable foundation

CC registers exactly eight internal metrics, all monotonic counters (`bootstrap/state.ts:955`): sessions started, lines of code changed, PRs and commits created, session cost (USD), token count, the approve/reject count for code-editing tools, and active seconds.

Two details are worth noting. First, these eight line up one-for-one with the official [monitoring docs](https://code.claude.com/docs/en/monitoring-usage) — a rare "the snapshot is the present" stable point; the next section shows the event list is the opposite, ballooning after the snapshot. Second, they're all counters rather than instantaneous gauges: a counter is more robust to sampling loss, process restarts, and out-of-order arrival, and aggregating it only takes a difference — CC even defaults its temporality to delta (`instrumentation.ts:113`, whose comment calls this the saner default).

Note too that metrics carry "shape," not content. The token counter carries `type` (input/output/cacheRead/cacheCreation) and model name, but never prompt text; the lines-of-code counter carries added/removed, but never file paths or code. This isn't an oversight — it's deliberate; see 16.8.

### Cardinality as a knob

Which common attributes each metric should carry is itself configurable — because "carry one more attribute" often equals "the time-series database splits into one more dimension." CC makes the most dangerous ones into switches whose very defaults show restraint (`telemetryAttributes.ts:10`): `session.id` on by default, `app.version` off by default.

The reason `app.version` is off is the same reason metrics refuse `prompt.id`: CC ships weekly, and if every version number became a new dimension the time-series database would be shredded by version. Handing these to env switches gives operators the trade-off between "how finely I want to attribute" and "how much cardinality I can bear." This trade-off comes up again and again in observability design.

(One small but instructive detail: the three OTLP protocol exporters total about 1.2MB, and rather than statically import them all at startup, CC does an on-demand `await import()` inside the protocol switch — keeping a heavy dependency that's "only needed if telemetry is on and using a given protocol" off the cold-start critical path, `instrumentation.ts:3`. A CLI is sensitive to startup latency, and this kind of discipline is everywhere.)

---

## 16.3 Events: one call site, two event streams

The second key to understanding CC's telemetry is that "internal analytics" and "customer observability" are two parallel event streams — and they're often emitted from the same spot in the code. The instant an API succeeds, that spot fires three things at once: an internal analytics event (bound for Anthropic's Statsig/Datadog/BigQuery), a customer OTel event (bound for your own OTLP backend), and the end of the trace span for this request.

```typescript
// services/api/logging.ts (on API success)
logEvent('tengu_api_success', { model, inputTokens, outputTokens, costUSD, … })  // internal analytics
void logOTelEvent('api_request', { model, input_tokens, output_tokens, … })       // customer observability
endLLMRequestSpan(llmSpan, { success: true, inputTokens, … })                     // end the trace span
```

The two streams divide cleanly. The internal one (names all prefixed `tengu_`) goes to Anthropic for product analysis, A/B tests, and behavioral regression, and numbers in the several hundreds of event names, gated by feature flags plus sampling. The customer one (`logOTelEvent`) goes to your own backend for usage and health dashboards — just the dozen-or-two documented kinds, gated by `CLAUDE_CODE_ENABLE_TELEMETRY`.

The internal stream has an interesting type-level guardrail: `logEvent` won't take a bare string. Any string you want to pass must be explicitly asserted into a very long-named type — `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` (`services/analytics/index.ts:19`). That turns "don't accidentally send code or filenames to the analytics backend" into a compile-time constraint that won't get past CI. To send actual PII you have to go through a `_PROTO_` prefix, which routes it into an access-controlled BigQuery column and gets stripped before fan-out to Datadog (`index.ts:45`).

### The skeleton of each customer event

Customer events all go through `logOTelEvent` (`events.ts:21`), which stamps three envelope fields on each: the event name, a timestamp, and a session-monotonic `event.sequence`.

`event.sequence` deserves a sentence, because it's easy to misread: it's an emission counter, incremented by one per event, whose job is to let the backend reorder one prompt's events by emission order after batching and the network have scrambled arrival order. It is not wall-clock time — which of two concurrent or overlapping API/tool activities actually happened first, and how long each lasted, comes from the timestamp and the span durations in the next section. `event.sequence` guarantees exactly one thing: that the emission order of discrete events is recoverable.

### Which events are in the snapshot, and how many more there are now

In the v2.1.88 snapshot, the events actually emitted are: `user_prompt`, `api_request`, `api_error`, `tool_result`, `tool_decision`, plus `system_prompt` and `tool` under beta detailed tracing, and hook start/finish plus a survey event.

The current official list has clearly expanded, adding `assistant_response`, `compaction`, `skill_activated`, `permission_mode_changed`, `api_request_body`/`api_response_body`, `mcp_server_connection`, `plugin_loaded`, and more.

> A note on provenance: this chapter's metric list (8) is stable between the snapshot and the official docs, but the event list has grown considerably since v2.1.88. Any identifier not grep-able in the snapshot (`assistant_response`, `OTEL_LOG_RAW_API_BODIES`, and so on) follows the official [monitoring docs](https://code.claude.com/docs/en/monitoring-usage), tagged "per official docs (beyond the snapshot)," and is not treated as source-level fact.

### Reading one call through api_request

Take `api_request`, the one that best answers the reader's question (`logging.ts:718`). It fires once after each API success, carrying the model name, how many input/output/cache-read/cache-write tokens, how much it cost, how long it took, and the speed tier. Pull all the `api_request` events sharing one `prompt.id`, sort them by `event.sequence`, and you have a ledger of this prompt's model calls: how many API calls total, how many tokens and dollars each, how much cache it hit, how long. `tool_decision` and `tool_result` then fill in "how the tool was approved or blocked" and "how long the tool ran, whether it succeeded, how big the result." Stitch the three together and you have a full ledger of this prompt's outward actions.

But one boundary has to be stated honestly. In the v2.1.88 snapshot, these events are correlated by prompt, not by turn: `api_request` carries no turn ordinal, and `tool_decision` and `tool_result` carry no `tool_use_id` either (the current version adds it). So precise causality — "which tool_result corresponds to which api_request, in which turn" — can't be reconstructed from event fields alone in the snapshot; that's the job of the next section's span tree and the transcript's parent edges. The event layer gives you an ordered ledger of this prompt's actions; the causal tree is left to the trace layer.

---

## 16.4 Trace: the operator tree of one interaction, plus a record everyone has

This is the heart of the chapter, and the layer closest to "running EXPLAIN on a task." It's carried by two things together: OpenTelemetry's span tree (precise causality, but off by default and still beta), and the local session record (on by default, and everyone has it).

### (A) The span tree: building one interaction into an operator tree

CC defines six kinds of span whose parent-child relationships, laid out, are exactly an EXPLAIN operator tree:

```
claude_code.interaction              root: one full user prompt → reply cycle
  ├─ claude_code.llm_request         one model call per turn
  ├─ claude_code.tool                one tool invocation
  │    ├─ …tool.blocked_on_user      wall-clock time stuck "waiting for your approval"
  │    └─ …tool.execution            the tool's actual execution time
  └─ claude_code.hook                one hook execution (beta detailed tracing only)
```

(Type definitions at `sessionTracing.ts:49`.) The root is the `startInteractionSpan` opened back in 16.1 when the prompt arrives, closed with the full duration. How is parentage established? Via `AsyncLocalStorage`: the interaction span is stored there, and when a child span opens it pulls the current parent back out — so even across several layers of async awaits, the child span still hangs correctly under the current interaction, without manually threading a handle everywhere.

Three design details show how easy it is to get tracing wrong.

First, timing "waiting for your approval" separately. Why give this stretch its own `tool.blocked_on_user` span? Because a slow Edit might be a slow model, a slow disk — or just you going to pour a coffee. Peel "waiting on a human" out of "executing," and the trace shows at a glance where the wall-clock time actually went, which directly decides whether you should optimize the model, the IO, or the workflow. This span also records whether it was approved or denied, and by whom, with data from the permission layer (see 16.6).

Second, concurrent requests must pass their span explicitly. One interaction often has several model requests in flight at once (main thread, warmup, topic classifier, path extractor). A comment warns outright: when filling in a response, if you don't carry the original span explicitly, it may hang on "whichever span happened to open last," causing a response mismatch (`sessionTracing.ts:342`). This is the classic concurrent-tracing pitfall, and CC cures it with "pass the handle explicitly" rather than "implicitly take the most recent."

Third, a backstop for span leaks. The normal path ends and deletes spans promptly, but when a stream is aborted or a turn throws midway, a span may never end. CC keeps live spans behind weak references and hangs a half-hour background sweep that force-reclaims any that are unreferenced or timed out — an observability facility must not become a memory leak of its own.

Every span carries its duration on close, and model spans also carry token detail. So the tree is, by nature, an EXPLAIN annotated with time and consumption: which turn, which tool, which wait is most expensive, at a glance.

So why is it off by default and still beta? Because it's both expensive and sensitive. Turning it on requires both "telemetry is on" and "enhanced tracing is on," and the latter also has to be compiled in and then allowed at runtime. The more detailed beta path even writes the system prompt and model output into spans, with a higher bar and its own redaction. Span count grows linearly with tool calls, and each carries a fair number of attributes, so leaving it on by default would charge both a performance and a privacy bill.

### (B) The session record: the trace that's already there

If the span tree is the trace you "turn on specially, for a backend to look at," the session record is the one that's "there on day one, on everyone's machine." The former needs you to enable telemetry and configure a backend; the latter is written automatically on every run to `~/.claude/projects/<escaped working dir>/<session id>.jsonl`. `/resume` reads it, and so do nearly all community analysis tools. For most people, this is the trace they can actually use. Subagents write their own files, so each delegation chain's trace stays separate.

It's fundamentally a linked list joined by `parentUuid`: each message written is stamped with a batch of fields, one of which — `parentUuid` — points to the previous. Three subtleties:

- A tool result's parent pointer points to the assistant message that issued it, not simply the "physically previous" one. The code handles this specifically (`sessionStorage.ts:1031`). So a given tool result can be traced back precisely to "which tool_use, in which assistant turn, requested it" — causality is an explicit edge, not a guess from time order.
- A compaction boundary breaks the physical chain but keeps the logical one. Context compaction swaps a stretch of history for a summary, and at that point `parentUuid` is nulled (a physical break, to keep the prompt cache stable), but a separate `logicalParentUuid` preserves the original parent pointer, bridged again on read. Compaction doesn't break the trace's logical continuity.
- `promptId` is stamped only on user messages — the on-disk landing spot for that anchor from 16.1.

So how do you reconstruct "one prompt's several turns" from this record? Mostly by walking `parentUuid` edges: trace any message back to the root, or expand down from a starting point along child edges, and you connect the assistant turns, the tool calls they issued, and the results that came back. `promptId` here is a locating anchor, not a grouping key — it's only on user rows, so you use it to find a prompt's starting point and then walk `parentUuid` to trace out that prompt's assistant and tool causality; grouping "by promptId" alone would miss the assistant rows that don't carry it, and never collect the whole set. This is the transcript's version of EXPLAIN — and it's already sitting on your disk, with no telemetry turned on at all.

You don't have to parse the JSONL by hand, either: `/cost` sums this session's tokens, cost, and time (see 16.5), `/status` and `/context` show other dimensions, and `--debug` opens the debug log. The community tools that count usage or capture requests are all built on this record or its equivalent.

### A live example: how `/goal` and `/loop` are traced

With the abstractions covered, here are two real features to ground it. The following comes from reverse-engineering the 2.1.201 client (static string extraction plus traffic capture); it postdates the snapshot, is tagged separately, and is not mixed with v2.1.88 source facts.

`/goal` (keep Claude working toward a goal until it's met) carries its own trace fields. Set a goal and the session record gets a `goal_status` entry: the condition, how many rounds it iterated, duration, how many tokens it burned, whether it was met — every round's progress accumulates in. The mechanism is a Stop hook that fires at the end of each round, dispatching a separate small model to judge "met or not," with the verdict and counters landing in this record.

`/loop` (rerun a prompt on an interval or a self-chosen cadence) tracks each self-scheduled tick through a string of semantic events: next wakeup scheduled, this wakeup ended a turn, aged out from inactivity, keepalive fired, loop ended — plus a family of events from the background scheduling daemon. Every loop tick, replaying the same prompt, leaves a mark in this stream.

Both examples make the same point: CC's "one task across several turns" is not a black box — it lands either as a structured field in the session record (goal) or as a string of semantic events (loop), and both let you reconstruct "how many rounds this ran, and the state of each." The full reverse engineering of these two is in [Chapter 17](17-autonomy-goal-loop.md).

---

## 16.5 Cost: not a total, but a running sum

Behind the `/cost` summary is an account tallied the instant each API succeeds. On each success, this call's usage accumulates into a per-model total and is fed to two counters at once — one for cost, one for tokens (`cost-tracker.ts:291`).

Tokens are split four ways — input, output, cache-read, cache-write — and counted separately, because the four have different unit prices: a cache read is far cheaper than fresh input, and without splitting you can't cost it right. CC even bills recursive advisor sub-calls separately and emits a dedicated event, so a derived call's cost isn't lost.

This account also carries across sessions: cost state is saved into the project config and restored by session ID on `/resume`, so a resumed session doesn't start `/cost` from zero. On exit it prints the grand total and saves it — that "total cost / API and wall-clock duration / code changes / per-model usage" block you see in the terminal.

So this section answers the "how much does each stage consume" part of EXPLAIN: tokens and dollars aren't an after-the-fact total but a running quantity, accumulated on every call, dimensioned by model and type, and carried across resume.

---

## 16.6 Permission decisions: recording "why it was allowed," too

EXPLAIN has one more link — whether each action is normal, and whether it was blocked. CC records every tool permission decision as observable data. At the core is a function, `logPermissionDecision`, the single exit for every approve/reject, fanning one decision out to four places (`permissionLogging.ts:181`): an internal analytics event whose name varies by source (for an approval funnel), a `+1` to the code-editing-tool counter, a write-back of the decision into the tool context for the trace layer to read, and a customer OTel event.

What's observationally valuable is that it structures "who approved it" into a `source` field: auto-allowed by an allowlist in config, allowed by a hook, allowed because you clicked "always" or "this time," allowed by a classifier, rejected by you, aborted by you. In a security post-mortem you can tell at a glance whether a dangerous operation was released by hand or automatically by a rule — often the decisive distinction. It also quantifies "waiting on a human" into a millisecond count, recorded only when a confirmation was actually shown (auto-approvals don't count); this number and the trace layer's `tool.blocked_on_user` span are the same thing shown two ways. And because the decision is written back into the tool context, that span can be tagged on close with the exact approve/reject and source — the three layers are wired together at this one point.

The permission model itself (`ask`/`allow`/`deny`, the classifier, hook decisions) is covered in [Chapter 12](11-permission-security.md); here we only care how the decision gets observed.

---

## 16.7 First-party / internal-only facilities: an honest boundary

Not every observability facility is for external users. Spelling out which you can use, and which are eliminated at compile time or exist only inside Anthropic, keeps you from misjudging.

`ant-trace` is an empty shell in external builds. Its implementation in the snapshot is a one-line placeholder that always returns "not enabled." This is a live specimen of compile-time dead-code elimination — the external build physically has no real implementation, only a stub that's permanently off. A methodological aside: not finding a feature's full logic in the public artifact doesn't mean it doesn't exist; it may just have been cut at compile time.

Perfetto local tracing is the same — Ant-only, compile-time gated. It can export spans as a flame graph in the Perfetto UI, which sounds tempting, but the source header says outright "Ant-only" and "eliminated from external builds," with the whole initialization wrapped in a compile-time feature switch (`perfettoTracing.ts:2`, `:7`, `:260`). That is, `CLAUDE_CODE_PERFETTO_TRACE=1` only matters when that feature is compiled into the build; in an external build the code is already cut, and setting the env var does nothing. Don't take it for something an external user can casually enable.

Two more are worth just a mention: the pipeline to Anthropic's BigQuery (covered in 16.2, gated by the org opt-out, external users don't touch it); and an FPS tracker that folds terminal render frame rate into self-observation — a TUI recording even how smoothly it draws itself, a nice touch of UX engineering (renderer detail in [Chapter 14](12-user-experience.md)). As for beta detailed tracing, it can write the system prompt and model output into spans too, but a strict visibility matrix governs it: the model's raw reasoning is Ant-only, unreachable by external users even at full detail; the full system prompt appears only on this path, and only once per unique hash, to avoid re-reporting large text.

> The instruments CC turns on to observe and defend against reverse engineers (fake-tool injection, steganographic fingerprints, and the like) are a separate topic this book won't go into.

---

## 16.8 The privacy boundary: you see "shape," not "content"

Observability and privacy are inherently in tension: record more and you can debug more, but it's also more dangerous. CC's overriding principle, in one line: export only "shape" by default; content always requires an explicit opt-in.

"Shape" means metadata like length, token count, duration, model name, tool name, decision type; "content" means the prompt text, the model reply, a tool's input and output, code and files. The default is the former; the latter must be turned on item by item:

| Content | Switch (env) | Default |
|---|---|---|
| User prompt text | `OTEL_LOG_USER_PROMPTS` | redacted, length only |
| Assistant reply text | `OTEL_LOG_ASSISTANT_RESPONSES` | redacted, length only |
| Tool args / commands | `OTEL_LOG_TOOL_DETAILS` | omitted / redacted |
| Tool input/output | `OTEL_LOG_TOOL_CONTENT` | omitted |
| Raw API request/response bodies | `OTEL_LOG_RAW_API_BODIES` | not exported |
| Full system prompt | beta detailed tracing only | not in normal export |
| Model reasoning text | internal only | never exported externally |

(Rows 1, 3, and 4 — user prompt / tool arguments and commands / tool input-output — have corresponding redaction code in the snapshot; row 2 "assistant reply" (`OTEL_LOG_ASSISTANT_RESPONSES`) and row 5 `OTEL_LOG_RAW_API_BODIES` both postdate the snapshot, per official docs.)

A few design intentions are worth naming.

Code and file contents never enter metrics or events. The docs say plainly that "raw file contents and code snippets are not included in metrics or events," and the source side turns it into a compile-time constraint via that long-named type from 16.3. In the default observability surface, you can see "changed N lines of TypeScript," but not what was changed.

Privacy gating and cardinality control are often two sides of the same knob. `prompt.id` goes into events but not metrics; `app.version` is off by default — one fear is leaking attributable information, the other is cardinality explosion, but they turn the same set of switches. That's no coincidence: putting less information into the aggregation backend both saves cost and protects privacy, and the two goals often point to the same decision.

Subprocesses have a point that's easy to get backwards: by default (an ordinary local session) CC passes its entire environment through to the subprocesses it spawns, `OTEL_*` included; only when claude-code-action sets `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` — i.e. a CI scenario where the session is exposed to untrusted content (with `allowed_non_write_users` configured) — does CC strip the four OTLP header vars that carry `Authorization=Bearer` tokens (the `OTEL_EXPORTER_OTLP_HEADERS` family) from the subprocess environment, lest credentials leak via shell expansion to a third party; the backend address `OTEL_EXPORTER_OTLP_ENDPOINT` is never stripped under any circumstance. But per the current docs, when tracing is active it does inject the "which trace is this" correlation info (W3C's `TRACEPARENT`) into Bash/PowerShell subprocesses, so their spans hang on the same trace. What's passed is correlation info, not export credentials — don't conflate the two. (Both env variables are un-grep-able in the snapshot, postdate it, and follow the official docs.)

---

## Summary

Claude Code's observability isn't "dump a pile of logs" but a layered, purpose-tailored, privacy-first system of self-inspection. It answers "run EXPLAIN on a task" head-on: a prompt's several turns, each turn's API and tool actions, their durations and consumption, where it waited on a human, whose decision released each action, whether it erred. The four layers each do their job — events gathered by `prompt.id`, traces strung by span parentage, the session record rebuilt from `parentUuid` edges, and metrics refusing all these high-cardinality IDs so they stay aggregatable.

A few recurring trade-offs worth remembering on their own:

| Trade-off | Why |
|---|---|
| Four separate layers, not one big log | Aggregate, filter, trace causality, and full replay are four kinds of problem, each needing its own data structure |
| The aggregation layer refuses `prompt.id` / omits the version by default | High-cardinality dimensions blow up the time-series database |
| Metrics are all counters plus delta | Robust to loss, restart, and reordering; aggregating is just a difference |
| Internal `tengu_` and customer OTel run as parallel streams | Anthropic's product analysis and your own observability don't interfere, emitted concurrently from one spot |
| Time "waiting for your approval" separately | Otherwise "waiting on a human" and "executing" blur together in the wall-clock and can't be attributed |
| Pass the span handle explicitly on concurrent requests | Cures the response mismatch in concurrent tracing |
| Weak references plus a timed sweep for spans | The observability facility must not become a memory leak |
| The session record uses parent edges; a result points to the call that issued it | Causality is an explicit edge, not a guess from time order |
| Two-level cache plus a circuit breaker on opt-out | Observability must not drag down the main flow: cheap to turn off, tolerant of stale reads |
| Export "shape" by default, content item by item | Resolving the observability-vs-privacy tension with "redact by default, enable explicitly" |

> Version note: this chapter is a walkthrough of the v2.1.88 leaked-source snapshot, cross-checked against the official [monitoring docs](https://code.claude.com/docs/en/monitoring-usage). The eight metrics are stable between the snapshot and the current docs; the event list expanded considerably after the snapshot, and any identifier beyond the snapshot is tagged "per official docs." The `/goal` and `/loop` trace details come from reverse-engineering the 2.1.201 client (static extraction plus capture), tagged separately and not mixed with snapshot facts.
