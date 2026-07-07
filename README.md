<div align="center">

# How Claude Code Works

**Claude Code 架构学习笔记 — Agentic Coding System 深度分析**

[![GitHub stars](https://img.shields.io/github/stars/Windy3f3f3f3f/how-claude-code-works?style=flat-square&logo=github)](https://github.com/Windy3f3f3f3f/how-claude-code-works)
[![GitHub forks](https://img.shields.io/github/forks/Windy3f3f3f3f/how-claude-code-works?style=flat-square&logo=github)](https://github.com/Windy3f3f3f3f/how-claude-code-works/fork)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](./LICENSE)
[![TypeScript](https://img.shields.io/badge/Source-TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)](https://github.com/anthropics/claude-code)
[![Docs](https://img.shields.io/badge/Docs-19_chapters-orange?style=flat-square)](#专题深入)

<br/>

[**📘 在线阅读文档**](https://windy3f3f3f3f.github.io/how-claude-code-works/#/)
&nbsp;&nbsp;|&nbsp;&nbsp;
[English](./README_EN.md)

<br/>

> 🛠️ **想动手造一个？** 配套项目 **[Claude Code From Scratch](https://github.com/Windy3f3f3f3f/claude-code-from-scratch)** — ~4300 行 TypeScript / Python 两个版本，13 章分步教程，clean-room 教育实现，从零造一个受 Claude Code 启发的 Coding Agent

</div>

---

> ⚖️ **声明 / Disclaimer**：本项目是面向开发者的教育性架构分析。所有内容均为独立研究与推理，**不代表 Anthropic 官方设计**，也不保证与 Claude Code 真实内部实现一致。"Claude Code" 是 Anthropic 的商标，本项目与 Anthropic 无任何关联，亦不对外分发任何来自 Anthropic 的源码。

> ⚖️ **写在前面**：2026.7.4: 距离3月底的源码泄露事件，到现在claude code又更新了许多新的功能，尤其是/goal /loop组成的 loop engineering, 以及dynamic workflow为代表的新的agent编排、观测和干预方法，这些新的功能的原理分析其实对于我们使用claude code帮助很大，因此我们近期也在持续逆向解析这些新的功能，不断更新该repo

Claude Code 是目前使用最广泛的 AI 编程 Agent，也是我们认为最好用的 AI 编程工具。它能理解整个代码仓库、自主执行多步编程任务、安全地运行命令——而这一切背后是 **50 万行 TypeScript 源码**中沉淀的工程智慧。

这份 50 万行 TypeScript 的源码以快照形式在社区流出后，第一次让我们能从一个真实的、跑在百万开发者机器上的生产级 Coding Agent 里，亲眼看到它到底长什么样。**但 50 万行——从哪里开始读？**

这也是我们创建这个项目的初衷——我们都遇到了没有办法阅读这么庞大代码项目的问题，解决方案是和 Claude Code 一起读，让它写文档配合我们理解源代码。在此同时，我们想把这个过程文档化，就形成了这个项目。

我们和 Claude Code 加班从源码中提炼出 **16 篇专题文档**，覆盖了从核心循环到安全防护的每一个关键设计决策。不管你是想造自己的 AI Agent，还是想更深入地理解和使用 Claude Code，这里都是最短路径（应该？就算不是最短的，我们也会不断更新这个项目）。

<div align="center">
  <img alt="在线阅读文档网站截图" src="./assets/architecture.png" width="800" />
</div>

## 🏗️ 系统架构

```mermaid
graph TB
    User[用户输入] --> QE[QueryEngine 会话管理]
    QE --> Query[query 主循环]
    Query --> API[Claude API 调用]
    API --> Parse{解析响应}
    Parse -->|文本| Output[流式输出]
    Parse -->|工具调用| Tools[工具执行引擎]
    Tools --> ReadTool[读文件]
    Tools --> EditTool[编辑文件]
    Tools --> ShellTool[Shell 执行]
    Tools --> SearchTool[搜索工具]
    Tools --> MCPTool[MCP 工具]
    Tools -->|结果回注| Query

    Context[上下文工程] --> Query
    Context --> SysPrompt[系统提示词]
    Context --> GitStatus[Git 状态]
    Context --> ClaudeMD[CLAUDE.md]
    Context --> Compact[压缩流水线]

    Perm[权限系统] --> Tools
    Perm --> Rules[规则层]
    Perm --> AST[Bash AST 分析]
    Perm --> Confirm[用户确认]
```

## 🤔 这份源码为什么值得深入研究？

大多数 AI Agent 框架都是"demo 级别"——跑通一个场景就宣布成功。Claude Code 不同，它是一个**日活百万级开发者实际使用的生产系统**，需要处理的问题远比 demo 复杂：

- 对话动辄上百万 token，上下文窗口不够用怎么办？（记忆管理、压缩方案是如何设计的超重要）
- 数十个内置工具同时存在，怎么协调？（如果所有工具上下文都给AI，那将直接爆炸）
- 怎么让用户感觉"快"，哪怕模型推理本身就要几十秒？（如何实现流水线设计）
- 用户让 AI 执行 `rm -rf /`，怎么拦住？（安全护栏很重要）

这些问题的解法，就藏在源码里。

## 🔍 从源码中发现的关键设计

> 以下内容均来自对源码的实际分析，不是猜测。

### 为什么 Claude Code 用起来感觉那么快？

它其实做了三件聪明的事：

1. **全链路流式输出** — 不是等模型全部想完再显示，而是每生成一个 token 就立刻展示。从 API 调用到终端渲染，整条链路都是流式的。
2. **工具预执行** — 模型说"我要读某个文件"的时候，这个文件其实已经在读了。系统在模型还在输出的同时就开始解析和执行工具调用，利用模型生成的 5-30 秒窗口，把约 1 秒的工具延迟藏起来了。
3. **9 阶段并行启动** — 启动时把不相关的初始化任务并行执行，关键路径压到约 235ms。

### 出错了怎么办？—— 静默恢复

普通程序遇到错误会报错给用户。Claude Code 的策略是：**能恢复的错误，用户根本看不到。**

比如对话太长超出了上下文窗口，它不会弹个错误框让你手动处理，而是悄悄压缩上下文、自动重试。token 输出达到上限？自动从 8K 升级到 64K 再重试。整个 Agent 循环有 7 种不同的"继续"策略，每种对应一种故障恢复路径。

这就是为什么用 Claude Code 的时候很少遇到报错——不是没有错误，而是大部分都被内部消化了。

### 对话太长怎么办？—— 4 级渐进式压缩

这是整个系统中最精妙的设计之一。当上下文快要超限时，不是一刀切地压缩，而是分 4 个级别逐步处理：

1. **裁剪** — 先把历史消息中的大块内容（旧的工具输出）截断
2. **去重** — 几乎零成本地去除重复内容
3. **折叠** — 把不活跃的对话段落折叠起来，但不修改原始内容（可以展开恢复）
4. **摘要** — 最后手段，启动一个子 Agent 对整个对话做摘要

每一级都可能释放足够的空间，让后面的级别不需要执行。而且压缩后系统会**自动恢复最近编辑的 5 个文件内容**，防止模型忘记刚刚在干什么。

### 怎么防止 AI 执行危险操作？—— 7 层纵深防御

Claude Code 让 AI 直接在你电脑上跑命令，安全设计必须过硬。它不是靠一个"你确定吗？"对话框，而是搭建了 7 层防御体系：

1. **工作区信任** — 首次进入一个目录先确认信任；不信任就禁用该项目的所有自定义 Hook，挡住恶意仓库预埋脚本
2. **权限模式** — 不同信任级别，限制可执行的操作范围
3. **规则匹配** — 基于命令模式的 allow/deny/ask 列表
4. **Bash 命令深度分析** — 这里最硬核：用语法树分析（不是正则匹配）拆解 Shell 命令的真实意图，包含 23 项静态安全检查，覆盖命令注入、环境变量泄露、特殊字符攻击等
5. **工具级安全** — 每个工具自带输入校验与专属安全逻辑（如文件编辑工具拦截危险路径）
6. **沙箱与隔离** — 进程级沙箱（macOS Seatbelt / Linux 命名空间）+ Git Worktree 文件隔离
7. **用户确认** — 危险操作弹确认框，与 Hook、LLM 分类器竞速；有 200ms 防抖保护，且用户一旦操作，人类意图永远优先

这 7 层任何一层拦住就不会执行，纵深防御。

### 数十个工具如何协同工作？

所有工具——读文件、写文件、跑命令、搜索、甚至第三方 MCP 工具——都遵循**同一套接口规范**。这意味着：

- 第三方工具和内置工具走完全相同的执行流水线，享受同样的安全检查和权限控制
- 只读工具自动并行执行，写操作自动串行，不需要手动管理并发
- 工具输出超过 100K 字符时自动落盘，模型只拿到摘要和文件路径，需要时再读取全文

### 多个 Agent 如何协作？

Claude Code 支持三种多 Agent 模式：

- **子 Agent** — 主 Agent 分派任务给子 Agent，等结果返回
- **协调器** — 纯指挥官模式，协调器只能分配任务，**不能自己读文件、写代码**，强制分工
- **Swarm** — 多个命名 Agent 之间点对点通信，各自独立工作

为了防止多个 Agent 同时改同一个文件产生冲突，系统用 Git Worktree 给每个 Agent 一份独立的代码副本。

## 📚 专题深入

| # | 文档 | 你会了解到 |
|---|------|-----------|
| 1 | [概述](./docs/01-overview.md) | 技术选型背后的思考（为什么 Bun/React/Zod）、6 条核心设计原则、9 阶段 235ms 启动流程、数据流全景 |
| 2 | [系统主循环](./docs/02-agent-loop.md) | Agent 循环的双层架构、7 种 Continue Sites 故障恢复、工具预执行、StreamingToolExecutor 并发机制 |
| 3 | [上下文工程](./docs/03-context-engineering.md) | 4 级压缩流水线完整细节、压缩后自动恢复机制（5 文件 + 技能重激活）、提示词缓存策略与缓存断裂检测 |
| 4 | [工具系统](./docs/04-tool-system.md) | 数十个工具的注册与并发控制、MCP 6 种传输详解、连接状态机、OAuth 2.0 + PKCE 认证流程 |
| 5 | [技能系统](./docs/09-skills-system.md) | 6 层技能来源与优先级、懒加载与 Token 预算分配、Inline/Fork 双执行模式、白名单权限模型、压缩后技能保留 |
| 6 | [记忆系统](./docs/08-memory-system.md) | 4 种记忆类型与封闭分类法、Sonnet 语义召回与异步预取、后台记忆提取 Agent、记忆漂移防御、团队记忆 |
| 7 | [Hooks 与可扩展性](./docs/06-hooks-extensibility.md) | 23+ Hook 事件全景、5 种 Hook 类型、6 阶段执行管道、PermissionRequest 4 种能力、信任模型与安全 |
| 8 | [多 Agent 架构](./docs/07-multi-agent.md) | 子 Agent 4 种执行模式与 Worktree 隔离、协调器纯编排设计、Swarm 3 种执行后端与信箱通信 |
| 9 | [Plan 模式](./docs/10-plan-mode.md) | 两条进入路径、5 阶段与迭代双工作流、附件节流机制、Phase 4 四种实验变体、计划文件管理与恢复、审批与权限恢复 |
| 10 | [代码编辑策略](./docs/05-code-editing-strategy.md) | search-and-replace 为什么比整文件重写更好、唯一性约束与抗幻觉设计、编辑前强制读取的代码级实现 |
| 11 | [任务管理系统](./docs/15-task-system.md) | 文件级存储与并发锁设计、三层变更检测、依赖追踪与原子认领、多 Agent 任务协调、验证提醒机制 |
| 12 | [权限与安全](./docs/11-permission-security.md) | 7 层纵深防御体系、tree-sitter AST 分析 + 23 项安全检查、竞速确认机制与 200ms 防误触 |
| 13 | [系统提示词设计](./docs/14-system-prompt-design.md) | 7 层递进式提示词架构、反模式接种与负面清单设计、爆炸半径风险框架、内外分层变体、7 条 Agent 提示词设计原则 |
| 14 | [用户体验设计](./docs/12-user-experience.md) | 自研 Ink 渲染器架构、Yoga Flexbox 布局、虚拟滚动与对象池优化、Vim 模式 |
| 15 | [最小必要组件](./docs/13-minimal-components.md) | 7 个最小必要组件框架、最小实现 vs 生产级实现的逐项对照、从 500 行到 50 万行的演进路线 |
| 16 | [可观测性：Metrics 与 Trace](./docs/16-observability.md) | 一次 prompt 的 EXPLAIN、三观测平面 + transcript 持久层、prompt.id 关联键、OTel metric/event/span 走读、成本核算、权限决策日志、隐私边界 |
| 17 🔍 | [自治与续跑：`/goal` 与 `/loop`](./docs/17-autonomy-goal-loop.md) | **快照之后·黑盒逆向**：自治的两种范式（守门评估器 vs 自排程闹钟）、`/goal` 评估器的完整系统提示词与 impossible 死循环刹车、`/loop` 解析规则与 cron / ScheduleWakeup 三条执行路径、文末附可复现的逆向方法（静态串 + 明文反代抓包） |
| 18 🔍 | [Auto Mode：权限进入分类器时代](./docs/18-auto-mode.md) | **快照之后·源码+抓包**：权限从"规则 + 确认框"进化到 ML 分类器逐动作裁决、四个自然语言规则桶、两段式（粗筛→细判）分类器、reasoning-blind"被审者无从辩护"、它怎么听懂"别 push"、刹车与降级；文末附分类器完整系统提示词原文 + 可复现逆向方法 |
| 19 🔍 | [Dynamic Workflows：确定性脚本编排 agent 舰队](./docs/19-dynamic-workflows.md) | **快照之后·工具描述+strings+抓包**：把第 8 章"模型当协调器"换成确定性 JS 脚本 fan-out 几十上百 subagent；pipeline 无 barrier 的设计、schema 结构化输出、三道上限 + token 预算、worktree 隔离与断点续跑、把可靠性编排进流程的质量范式、ultraplan→ultracode 演进；文末附 Workflow 工具描述逐字节选 |

## 🎯 谁应该读这个？

| 你是 | 你能获得 |
|------|---------|
| 想做 AI Agent 产品的开发者 | 一个经过百万用户验证的架构参考，少走弯路 |
| Claude Code 用户 | 理解它为什么这样工作，学会用 Hooks 和 CLAUDE.md 深度定制 |
| 对 AI 安全感兴趣的人 | 生产级 AI 系统的安全设计实战，不是论文里的理论 |
| 学生或 AI 研究者 | 大规模工程实践的第一手材料，比任何教科书都真实 |

## 📊 关键数据

| 指标 | 数值 |
|------|------|
| 源码总行数 | 512,000+ |
| TypeScript 文件 | 1,884 |
| 内置工具 | ~55（注册表 tools.ts，含 feature-gated） |
| 压缩流水线级数 | 4 级 |
| 权限防御层数 | 7 层 |

## 🗺️ 阅读建议

**只有 10 分钟？**
→ 读 [快速入门](./docs/quick-start.md)

**想理解核心原理？**
→ 按顺序读 [主循环](./docs/02-agent-loop.md) → [上下文工程](./docs/03-context-engineering.md) → [工具系统](./docs/04-tool-system.md)

**想自己造一个 AI Agent？**
→ 先读 [最小必要组件](./docs/13-minimal-components.md)，然后跟着 **[claude-code-from-scratch](https://github.com/Windy3f3f3f3f/claude-code-from-scratch)** 的 13 章教程动手实现——~4300 行代码，每一步都对照源码讲解

**想定制 Claude Code？**
→ 读 [Hooks 与可扩展性](./docs/06-hooks-extensibility.md) + [记忆系统](./docs/08-memory-system.md) + [技能系统](./docs/09-skills-system.md)

**关注安全？**
→ 读 [权限与安全](./docs/11-permission-security.md) + [代码编辑策略](./docs/05-code-editing-strategy.md)

## 🧭 路线图 / TODO

本仓库的分析基于约 **v2.1.6x（2026 年 3 月底）** 的源码快照。此后 Claude Code 又发布了 ~130 个版本，出现了一批快照里没有的重要新能力。这些内容**没有源码可走读**，我们计划换一种方法继续分析——**黑盒实测 + 开源情报**（官方 changelog、官方文档、公开博客）：安装最新版真实使用，观察行为、边界与失败模式，再对照官方资料交叉验证，推理出"它大概是怎么做的"（推理与实锤会明确区分）。

计划新增的专题：

- [x] **可观测性：Metrics 与 Trace**（[#10](https://github.com/Windy3f3f3f3f/how-claude-code-works/issues/10) 提议）—— Claude Code 自身如何打点：OpenTelemetry 指标/事件导出、成本核算、会话 transcript 作为 turn 级 trace。**已完成 → [第 16 章：可观测性](./docs/16-observability.md)**
- [x] **自治闭环：`/goal`、`/loop` 与 cron 调度**（v2.1.71 / v2.1.139）—— 设定完成条件后跨 turn 自主工作直到达成；定时/模型自定节奏的循环任务。**已完成 → [第 17 章：自治与续跑](./docs/17-autonomy-goal-loop.md)**（含 `/goal` 评估器与 `/loop` 命令的完整提示词原文 + 可复现逆向方法）
- [x] **Dynamic Workflows（触发词 "ultracode"）**（v2.1.154 起）—— 用一段编排脚本在后台指挥几十到几百个 agent，带 token 预算控制、断点续跑与 `/workflows` 监控面板。**已完成 → [第 19 章：Dynamic Workflows](./docs/19-dynamic-workflows.md)**（含 Workflow 工具描述逐字节选 + 可复现逆向方法）
- [x] **Auto Mode：权限进入分类器时代**（v2.1.152 起免 opt-in）—— 从"规则 + 确认框"到 LLM 分类器实时判断每个动作放行/拦截，并能理解口头边界（如"别 push"）。**已完成 → [第 18 章：Auto Mode](./docs/18-auto-mode.md)**（含分类器完整系统提示词原文 + 四规则桶 + 可复现逆向方法）
- [ ] **后台 Agent 舰队**（v2.1.139–198）—— `/bg` 后台化、常驻 daemon、`claude agents` 全局面板、会话睡醒机制、完工自动 commit+push+draft PR、子 agent 默认后台运行
- [ ] **云端多 Agent 审查**（v2.1.111–147）—— `/ultrareview` → `/code-review`：并行多 agent 分析 + 对抗式批判，带档位（low→ultra）与 CI 集成
- [ ] **Agent Teams 与跨会话安全**（v2.1.166–178）—— `SendMessage` 组队协作；"跨会话消息不携带用户权威"的防提示注入设计

> 欢迎在 [issues](https://github.com/Windy3f3f3f3f/how-claude-code-works/issues) 里投票或补充你最想看的主题。

## 🤝 贡献者

| <img src="https://github.com/Windy3f3f3f3f.png" width="60" /> | <img src="https://github.com/davidweidawang.png" width="60" /> | <img src="./assets/kaibo.jpg" width="60" /> | <img src="https://github.com/longx24.png" width="60" /> |
|:---:|:---:|:---:|:---:|
| [@Windy3f3f3f3f](https://github.com/Windy3f3f3f3f) | [@davidweidawang](https://github.com/davidweidawang) | [Kaibo Huang](https://scholar.google.com/citations?user=C7B5X5IAAAAJ&hl=zh-CN) | [@longx24](https://github.com/longx24) |

欢迎提 Issue 和 PR！如果你发现分析有误或有更好的理解角度，非常欢迎讨论。

## 🙏 致谢

感谢 [LINUX DO](https://linux.do/) 社区的支持与讨论。

## 💬 更多交流

<div align="center">

**加入 AI Agent 工坊 交流群**

<img src="./assets/qq.jpg" width="280" alt="QQ 群二维码" />

QQ 群号：**1090526244**

</div>

## 📈 Star History

<div align="center">
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=Windy3f3f3f3f/how-claude-code-works&type=Date&theme=dark" />
  <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=Windy3f3f3f3f/how-claude-code-works&type=Date" />
  <img alt="Star History Chart" src="https://api.star-history.com/svg?repos=Windy3f3f3f3f/how-claude-code-works&type=Date" width="600" />
</picture>
</div>

## 📝 更新记录

| 日期 | 更新内容 |
|------|---------|
| 2026-07-07 | 新增第 19 章：Dynamic Workflows（触发词 ultracode）——用确定性 JS 脚本编排几十上百个 subagent，讲 pipeline 无 barrier 的设计、并发/agent/预算三道上限、worktree 隔离与断点续跑，附 Workflow 工具描述节选 |
| 2026-07-06 | 新增第 18 章：Auto Mode——权限从"规则 + 确认框"进入 ML 分类器时代，含分类器完整系统提示词、四个自然语言规则桶、两段式分类与"别 push"的语义边界 |
| 2026-07-06 | 新增第 17 章：自治与续跑（/goal 与 /loop）——含 /goal 评估器与 /loop 命令的完整提示词原文，以及可复现的逆向方法 |
| 2026-07-06 | 第 16 章可观测性重写为清晰技术写作并去 AI 味；全书对 v2.1.88 源码快照做对抗式事实核查并修正；纵深防御口径统一为 7 层 |
| 2026-07-04 | 新增第 16 章：可观测性（Metrics 与 Trace，呼应 #10），开启"泄露快照之后的新功能"模块——对 goal/loop、Auto Mode、Dynamic Workflows 等快照后新能力做黑盒逆向 + 开源情报分析 |
| 2026-07-03 | 补充路线图 / TODO：规划快照（≈v2.1.6x）之后新功能的分析计划 |
| 2026-07-02 | 项目定位措辞重构，补充免责声明（教育性架构分析、非官方、不分发源码） |
| 2026-05-05 | 完善工具系统文档 |
| 2026-04-21 | 重组文档结构、更新 favicon，新增贡献者 |
| 2026-04-10 | 修复过宽的 Mermaid 图，改为纵向布局避免文字挤压 |
| 2026-04-09 | 全面 Review 并修正全部 13 章：修复数字/引用错误（行数、百分比、事件数量、章节编号），为缺少概述的章节补充 high-level 导语，优化章节内部结构（ch05 拆分/交换、ch08 重组），中英文同步更新 |
| 2026-04-03 | 新增第 14 章：系统提示词设计哲学，深入分析提示词内容的设计原理与工程实践 |
| 2026-04-03 | 新增暗色模式、阅读进度条、回到顶部按钮、上下文感知语言切换等 UI 优化 |
| 2026-04-03 | 完成全部 13 篇文档的英文翻译，支持中英双语切换 |
| 2026-04-01 | 拆分记忆与技能为独立章节（11→12 篇），按侧边栏分组重新编号 01-12 |
| 2026-04-01 | 全部 12 章大幅扩充（篇幅翻倍），补充源码级实现细节、Mermaid 架构图、代码示例 |
| 2026-04-01 | 同步更新 quick-start 总览页、README 文档目录描述，匹配各章节新增内容 |
| 2026-03-31 | 新增 3 章：Hooks 与可扩展性、多 Agent 架构、记忆与技能系统 |
| 2026-03-31 | 充实原有 8 章内容，补充启动流程、MCP 集成、压缩恢复机制等专题 |
| 2026-03-31 | 上线 Docsify 文档站点，支持搜索、Mermaid 渲染、章节导航 |
| 2026-03-31 | 初始发布：8 篇核心架构分析文档 |

## 📄 License

MIT
