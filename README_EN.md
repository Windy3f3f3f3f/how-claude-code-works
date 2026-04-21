# How Claude Code Works

[![GitHub stars](https://img.shields.io/github/stars/Windy3f3f3f3f/how-claude-code-works?style=social)](https://github.com/Windy3f3f3f3f/how-claude-code-works)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
[![TypeScript](https://img.shields.io/badge/Source-TypeScript-3178C6?logo=typescript&logoColor=white)](https://github.com/anthropics/claude-code)

> A deep dive into the source code architecture of the most successful AI coding agent

<p align="center">
  <a href="https://windy3f3f3f3f.github.io/how-claude-code-works/#/en/"><strong>📘 Read Online</strong></a>
  &nbsp;&nbsp;|&nbsp;&nbsp;
  <a href="./README.md">中文</a>
</p>

> 🛠️ **Want to build one yourself?** Companion project **[Claude Code From Scratch](https://github.com/Windy3f3f3f3f/claude-code-from-scratch)** — 1300 lines of TypeScript, 8-chapter step-by-step tutorial, build your own Claude Code from zero

---

Claude Code is the most widely used AI coding agent today. It understands entire codebases, autonomously executes multi-step programming tasks, and safely runs commands — all powered by engineering wisdom distilled into **500K+ lines of TypeScript source code**.

Anthropic open-sourced this codebase. **But where do you even start with 500K lines of code?**

This project is the answer. We've distilled **14 topic-specific documents** (338K characters total) covering every critical design decision, from the core agent loop to the security architecture. Whether you want to build your own AI agent or deeply understand how Claude Code works, this is the shortest path.

## System Architecture

<img alt="Architecture Overview" src="./assets/architecture.png" width="800" />

```mermaid
graph TB
    User[User Input] --> QE[QueryEngine Session Manager]
    QE --> Query[query Main Loop]
    Query --> API[Claude API Call]
    API --> Parse{Parse Response}
    Parse -->|Text| Output[Streaming Output]
    Parse -->|Tool Call| Tools[Tool Execution Engine]
    Tools --> ReadTool[File Read]
    Tools --> EditTool[File Edit]
    Tools --> ShellTool[Shell Exec]
    Tools --> SearchTool[Search Tools]
    Tools --> MCPTool[MCP Tools]
    Tools -->|Results| Query

    Context[Context Engineering] --> Query
    Context --> SysPrompt[System Prompt]
    Context --> GitStatus[Git Status]
    Context --> ClaudeMD[CLAUDE.md]
    Context --> Compact[Compression Pipeline]

    Perm[Permission System] --> Tools
    Perm --> Rules[Rule Layer]
    Perm --> AST[Bash AST Analysis]
    Perm --> Confirm[User Confirmation]
```

## Why is this source code worth studying?

Most AI agent frameworks are "demo-grade" — they work for one scenario and call it done. Claude Code is different. It's a **production system used daily by millions of developers**, tackling problems far more complex than any demo:

- Conversations grow to tens of thousands of tokens — what happens when the context window runs out?
- A user asks the AI to run `rm -rf /` — how do you stop it?
- 66 built-in tools coexist — how do you coordinate them?
- Network drops, API overloads, token limits hit — how do you avoid crashing?
- How do you make it *feel* fast when model inference alone takes tens of seconds?

The answers are all in the source code.

## Key Designs from the Source Code

> Everything below comes from actual source code analysis, not speculation.

### Why does Claude Code feel so fast?

It does three clever things:

1. **End-to-end streaming** — Instead of waiting for the model to finish thinking, every token is displayed the instant it's generated. The entire pipeline from API call to terminal rendering is streaming.
2. **Tool pre-execution** — When the model says "I need to read this file," that file is already being read. The system parses and executes tool calls while the model is still generating output, hiding ~1s of tool latency within the 5-30s model generation window.
3. **9-phase parallel startup** — Independent initialization tasks run in parallel, compressing the critical path to ~235ms.

### What happens when things go wrong? — Silent recovery

Most programs show errors to users. Claude Code's strategy: **if an error is recoverable, the user never sees it.**

When a conversation exceeds the context window, it doesn't pop up an error dialog — it silently compresses the context and retries. Hit the output token limit? It automatically escalates from 4K to 64K and tries again. The agent loop has 7 different "continue" strategies, each handling a different failure recovery path.

This is why you rarely see errors in Claude Code — not because there aren't any, but because most are handled internally.

### What about long conversations? — 4-level progressive compression

One of the most elegant designs in the entire system. When context approaches its limit, instead of a blunt compression pass, it goes through 4 graduated levels:

1. **Snip** — Truncate large content blocks (old tool outputs) from history
2. **Deduplicate** — Remove duplicate content at near-zero cost
3. **Collapse** — Fold inactive conversation segments without modifying originals (reversible)
4. **Summarize** — Last resort: spawn a child agent to summarize the entire conversation

Each level may free enough space that subsequent levels don't need to run. After compression, the system **automatically restores the 5 most recently edited files**, preventing the model from forgetting what it was just working on.

### How do you prevent AI from executing dangerous operations? — 5 layers of defense

Claude Code runs commands directly on your machine — security has to be rock-solid. It doesn't rely on a single "are you sure?" dialog. Instead, it builds 5 layers of defense:

1. **Permission modes** — Different trust levels restricting what operations can run
2. **Rule matching** — Pattern-based allowlists and denylists
3. **Deep Bash analysis** — The most hardcore layer: uses syntax tree analysis (not regex) to dissect the true intent of shell commands, with 23 security checks covering command injection, environment variable leaks, special character attacks, and more
4. **User confirmation** — Dangerous operations trigger a confirmation dialog with 200ms debounce protection against accidental key presses
5. **Hook validation** — Users can define custom security rules that even modify tool inputs on the fly (e.g., automatically adding `--dry-run` to `rm` commands)

If any single layer blocks the action, it doesn't execute. Defense in depth.

### How do 66 tools work together?

All tools — file reading, file writing, shell commands, search, even third-party MCP tools — follow **the same interface specification**. This means:

- Third-party tools go through the exact same execution pipeline as built-in tools, getting identical security checks and permission controls
- Read-only tools automatically run in parallel; write operations are serialized — no manual concurrency management needed
- When tool output exceeds 100K characters, it's automatically saved to disk; the model gets a summary and file path, reading the full content on demand

### How do multiple agents collaborate?

Claude Code supports three multi-agent modes:

- **Sub-agent** — The main agent dispatches tasks to child agents and waits for results
- **Coordinator** — Pure commander mode: the coordinator can only assign tasks, **it cannot read files or write code itself**, enforcing division of labor
- **Swarm** — Named agents communicate peer-to-peer, each working independently

To prevent conflicts from multiple agents editing the same files, the system uses Git Worktrees to give each agent its own isolated copy of the codebase.

## Documentation

### Quick Start
- **[Understand Claude Code in 10 Minutes](./en/docs/quick-start.md)** — Condensed overview of everything

### Deep Dives

| # | Document | What you'll learn |
|---|----------|-------------------|
| 1 | [Overview](./en/docs/01-overview.md) | What problem Claude Code solves, the thinking behind tech choices, overall architecture |
| 2 | [Agent Loop](./en/docs/02-agent-loop.md) | How the agent "think-act-observe" loop works, how it handles interruption and recovery |
| 3 | [Context Engineering](./en/docs/03-context-engineering.md) | How to fit the most useful information into a limited context window, full compression strategy details |
| 4 | [Tool System](./en/docs/04-tool-system.md) | How 66 tools are registered, dispatched, and concurrency-controlled; how to integrate third-party tools |
| 5 | [Skills System](./en/docs/09-skills-system.md) | 6 skill sources, lazy loading, inline/fork execution, permission model, post-compaction preservation |
| 6 | [Memory System](./en/docs/08-memory-system.md) | 4 memory types, Sonnet semantic recall, background extraction agent, drift defense |
| 7 | [Hooks & Extensibility](./en/docs/06-hooks-extensibility.md) | 23 hook events, how to customize Claude Code's behavior without modifying source code |
| 8 | [Multi-Agent Architecture](./en/docs/07-multi-agent.md) | Sub-agent, Coordinator, and Swarm — design tradeoffs of three multi-agent modes |
| 9 | [Plan Mode](./en/docs/10-plan-mode.md) | Two entry paths, 5-phase workflow, attachment throttling, Phase 4 A/B experiments, plan file management, approval and permission restoration |
| 10 | [Code Editing Strategy](./en/docs/05-code-editing-strategy.md) | Why "search-and-replace" over "full file rewrite," how to ensure edit safety |
| 11 | [Task Management System](./en/docs/15-task-system.md) | File-level storage with concurrency locking, 3-layer change detection, dependency tracking and atomic claiming, multi-agent task coordination, verification nudge |
| 12 | [Permission & Security](./en/docs/11-permission-security.md) | The complete 5-layer security system, 23 Bash security checks |
| 13 | [System Prompt Design](./en/docs/14-system-prompt-design.md) | 7-layer progressive prompt architecture, anti-pattern inoculation, blast radius risk framework, 7 agent prompt design principles |
| 14 | [User Experience](./en/docs/12-user-experience.md) | Why React for terminal UI, streaming output implementation, terminal interaction details |
| 15 | [Minimal Components](./en/docs/13-minimal-components.md) | The minimum modules needed for a coding agent, the evolution path from 500 lines to 500K |

## Who should read this?

| You are | What you'll get |
|---------|----------------|
| A developer building AI agent products | A battle-tested architecture reference validated by millions of users |
| A Claude Code user | Understanding of why it works the way it does, and how to deeply customize it with Hooks and CLAUDE.md |
| Someone interested in AI safety | Production-grade AI security design in practice, not just theory from papers |
| A student or AI researcher | First-hand material on large-scale engineering practice, more real than any textbook |

## Key Stats

| Metric | Value |
|--------|-------|
| Source lines | 512,000+ |
| TypeScript files | 1,884 |
| Built-in tools | 66+ |
| Compression levels | 4 |
| Security layers | 5 |

## Reading Recommendations

**Only have 10 minutes?**
→ Read [Quick Start](./en/docs/quick-start.md)

**Want to understand core principles?**
→ Read in order: [Agent Loop](./en/docs/02-agent-loop.md) → [Context Engineering](./en/docs/03-context-engineering.md) → [Tool System](./en/docs/04-tool-system.md)

**Want to build your own AI agent?**
→ Start with [Minimal Components](./en/docs/13-minimal-components.md), then follow **[claude-code-from-scratch](https://github.com/Windy3f3f3f3f/claude-code-from-scratch)** — 8-chapter hands-on tutorial, 1300 lines of code, every step mapped to the real source

**Want to customize Claude Code?**
→ Read [Hooks & Extensibility](./en/docs/06-hooks-extensibility.md) + [Memory System](./en/docs/08-memory-system.md) + [Skills System](./en/docs/09-skills-system.md)

**Care about security?**
→ Read [Permission & Security](./en/docs/11-permission-security.md) + [Code Editing Strategy](./en/docs/05-code-editing-strategy.md)

## Contributing

Issues and PRs welcome! If you find an error in the analysis or have a better perspective, we'd love to discuss.

## Changelog

| Date | Changes |
|------|---------|
| 2026-04-09 | Comprehensive review and fix of all 13 chapters: corrected inaccurate numbers/references (line counts, percentages, event counts, chapter numbering), added high-level overviews to chapters that lacked them, restructured sections for better readability (ch05 split/swap, ch08 reorder/merge), synchronized Chinese and English versions |
| 2026-04-03 | Added Chapter 14: System Prompt Design Philosophy, in-depth analysis of prompt content design principles and engineering practices |
| 2026-04-03 | Added dark mode, reading progress bar, back-to-top button, context-aware language switching, and other UI improvements |
| 2026-04-03 | Completed English translations for all 13 documents, supporting bilingual Chinese-English switching |
| 2026-04-01 | Split Memory & Skills into separate chapters (11→12 articles), renumbered 01-12 by sidebar grouping |
| 2026-04-01 | Major expansion of all 12 chapters (doubled in length), added source-level implementation details, Mermaid architecture diagrams, and code examples |
| 2026-03-31 | Added 3 chapters: Hooks & Extensibility, Multi-Agent Architecture, Memory & Skills System |
| 2026-03-31 | Launched Docsify documentation site with search, Mermaid rendering, and chapter navigation |
| 2026-03-31 | Initial release: 8 core architecture analysis documents |

## License

MIT
