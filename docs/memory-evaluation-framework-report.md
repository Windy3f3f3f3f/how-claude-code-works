# 记忆评测框架调研分析报告
## 面向 OpenClaw + Co-Sight Memory（cosight-memory）插件的 LoCoMo 与代码域评测选型

> 版本：v1.0（初版 + 深度校验已合并）
> 日期：2026-06-18
> 调研方法：基于书签所列开源仓库与论文，逐一进入 GitHub 仓库阅读源码、复核 issue 争议、核算论文表格数据；对关键结论（尤其 LoCoMo 可靠性）做了原始数据文件级的独立复算（详见第 4.2 节与附录 C）。报告中**每条关键结论均标注论文章节/表号或仓库文件路径**，无法核实者明确标注 `[未核实]`。

---

## 0. 执行摘要（给决策者）

**我们的主线：** ①基于一个权威框架在 **LoCoMo** 上评测 `OpenClaw + cosight-memory` 并调优 → ②在**最新最权威的代码域数据集**上、用同一或新的权威框架扩展代码域评测 → ③两类结果都要**能直接和别人对比** → ④之后自建代码域私有数据集。

**核心结论与建议（按主线给出）：**

| 阶段 | 推荐选择 | 一句话理由 | 依据 |
|---|---|---|---|
| ① 对话域主战场 | **LoCoMo + mem0 标准评测协议**（4 类问题 / LLM-as-Judge `J`） | 这是全行业事实上的"可比"协议，所有竞品（mem0、Zep、MemOS、OpenViking）都报这条线 | mem0 论文 §3.2；OpenViking README 评测节 |
| ① 可靠性护栏 | **强制套用 locomo-audit 的修正与报告规范** | LoCoMo 有 6.4% 错误标准答案、judge 62.81% 误判率、可复现性差，裸分不可信 | locomo-audit `AUDIT_REPORT.md`（已独立复算） |
| ① 学术级补充 | **LongMemEval-S** + **MemoryAgentBench** | 弥补 LoCoMo 在"知识更新/弃权"与"冲突消解"上的盲区，提升报告可信度 | arXiv 2410.10813；MemoryAgentBench README |
| ② 代码域首选 | **SWEContextBench**（arXiv 2602.08316）作"经验复用"主基准 + **LoCoBench-Agent**（arXiv 2511.13998）作"长上下文记忆保持"权威补充 | 前者最贴近"记忆=经验复用"语义、基于 SWE-Bench 可比；后者是 Salesforce 出品、含显式"多会话记忆保持"指标 | 两篇论文 + 仓库 |
| ② 同时强烈建议接入 | **EvoAgentBench** | 它**已经直接评测 `OpenClaw·27B/397B` + 记忆方法（EverOS/Memento/ReasoningBank/GEPA）在 SWE-Bench/LiveCodeBench 上的 Δ 提升**——我们能"对号入座" | EvoAgentBench 官网评测表 |
| ④ 私有数据集 | **复用 SWE-Bench 数据规范 + SWEContextBench 的"关系对"构造法**自建 | 保证私有集与公开集同构、可横向迁移评测脚本 | SWEContextBench 数据卡 |

**一句话给领导：** 对话域走"LoCoMo（mem0 协议）+ 审计护栏 + LongMemEval/MemoryAgentBench 交叉验证"；代码域走"SWEContextBench（经验复用）+ LoCoBench-Agent（长上下文记忆）"，并借 EvoAgentBench 直接挂到含 OpenClaw 的公开榜单上；私有集按 SWE-Bench 规范自建。**切忌只报 LoCoMo 单一裸分**——该基准已被公开审计证实存在系统性缺陷，单独引用会带来对外公信力风险。

---

## 1. 需求拆解与被测系统界定

### 1.1 主线需求（原文复述，避免跑偏）
1. **权威框架 + LoCoMo**：评测 `OpenClaw + cosight-memory` 记忆插件，并调优。
2. **代码域扩展**：用最新最权威的代码域数据集，在同一框架或新权威框架上，评测 `OpenClaw/Claude-Code + cosight-memory` 的代码域表现。
3. **可对比性**：LoCoMo 结果与代码域结果都能**直接和别人对比**。
4. **私有集**：之后自建代码域私有数据集评测。

### 1.2 被测系统（SUT）说明
- **OpenClaw**：在本次调研覆盖的多个公开评测中，"OpenClaw" 是一个被当作**被测编程/通用 Agent 基座**出现的名字：
  - EvoAgentBench 官网将其与 Nanobot 并列为两类被测 Agent，分 `OpenClaw·27B` 与 `OpenClaw·397B` 两种规模（EvoAgentBench 官网评测表）。
  - OpenViking README 评测节将 `OpenClaw` 作为对话记忆基线之一（OpenViking `README_CN.md` 📊 节）。
  - 因此可把 OpenClaw 理解为与 Claude Code 同类的 Agent 基座；`cosight-memory` 是我们在其上挂载的记忆插件，对标 OpenViking、mem0、EverOS 等"记忆层"。
- **cosight-memory / Claude Code 原生记忆**：本仓库 `docs/08-memory-system.md` 详述了 Claude Code 原生记忆的设计，cosight-memory 大概率是在此之上的增强：
  - 核心约束："只记忆不可从当前项目状态推导的信息"，刻意**排除**代码模式/架构/文件路径（`docs/08-memory-system.md` §6.1–6.2）。
  - 召回是"mtime 排序 + Sonnet 侧查询语义选 ≤5 条"（§6.5），而非稠密向量检索。
  - **这点对评测框架选型至关重要**（见 1.3）。

### 1.3 关键判断：原生记忆 ≠ LoCoMo 所测能力
LoCoMo 测的是"从海量对话历史中稠密召回事实"的能力，而 Claude Code 原生记忆**刻意不存可推导信息、用 mtime+LLM 选取做轻量召回**（`docs/08-memory-system.md` §6.5–6.6）。这意味着：
- 直接拿原生记忆裸跑 LoCoMo，分数会偏低——这不是"差"，是设计取向不同。
- `cosight-memory` 若要在 LoCoMo 上拿高分，需补齐**稠密事实抽取 + 向量/图检索**这条线（对标 mem0/OpenViking）。**报告必须显式说明这一"能力错位"，否则会误导对自身水平的判断。**

---

## 2. 评测框架全景与分类

按"评测域"划分本次调研覆盖的全部框架/数据集：

```
记忆评测
├── 对话域（Conversational / Personalization）
│   ├── LoCoMo ........... 事实上的行业基准（缺陷多，见 §4）
│   ├── LongMemEval ...... 更严谨的长程对话基准（ICLR'25）
│   ├── MemoryAgentBench . 多能力、可插拔记忆系统的统一框架（ICLR'26）
│   ├── SubtleMemory ..... 细粒度关系记忆辨别（更难的后继）
│   ├── STATE-Bench ...... 微软，企业有状态多轮任务
│   ├── OpenViking ....... 火山引擎，含 LoCoMo + OpenClaw 基线
│   └── EverMemOS/EverOS . 自称 SOTA 但可复现性遭质疑（见 §4.3）
└── 代码域 / 多域（Code / Multi-domain）
    ├── SWEContextBench .. 编码"经验复用/上下文学习"（基于 SWE-Bench）
    ├── LoCoBench-Agent .. Salesforce，长上下文软件工程 Agent（含记忆保持指标）
    ├── EvoAgentBench .... 多域自进化评测（已含 OpenClaw + 记忆方法）
    ├── AgentMemoryBench . 5 评测模式 × 6 交互任务
    ├── MemGym ........... 长程记忆环境
    ├── Vectorize AMB .... 工业界、成本感知、强调可复现
    └── TencentDB-Agent-Memory . 产品/存储，附带基准数字
```

**对应"权威性来源"分层（给领导判断分量用）：**
- **学术顶会/知名实验室**：LoCoMo（ACL'24, Snap）、LongMemEval（ICLR'25）、MemoryAgentBench（ICLR'26, HUST）、LoCoBench-Agent（Salesforce AI Research）。
- **大厂工程**：OpenViking（火山引擎）、STATE-Bench（微软）、TencentDB（腾讯云）、Vectorize AMB（Vectorize.io）。
- **新兴/单团队（需交叉验证、注意利益相关）**：EverMemOS/EverOS、EvoAgentBench（同属 EverMind-AI）、SWEContextBench、SubtleMemory、AgentMemoryBench、MemGym。

---

## 3. 对话域主战场：LoCoMo 与"可比协议"

### 3.1 LoCoMo 是什么、为什么是事实标准
- 出处：Maharana et al., *"Evaluating Very Long-Term Conversational Memory of LLM Agents"*，arXiv **2402.17753**，ACL 2024（Snap Research）。
- 规模：多会话超长对话（项目页：平均 ~300 轮 / ~9K token，最多 35 个 session）；问答分 **5 类**：单跳(single-hop)、多跳(multi-hop)、时间推理(temporal)、开放域/常识(open-domain)、对抗(adversarial/不可答)。
- 原始指标：QA 用 **F1**（项目页）。**LLM-as-a-Judge 是 mem0 等后续工作引入的下游惯例**，并非原论文主指标——这条"血缘"必须讲清，否则跨论文分数不可比。
- 为什么成事实标准：首个"周/月级"真实多会话、带类别标签、公开数据的记忆基准，便于一键 drop-in 对比。

### 3.2 mem0 协议 = 行业"可比"的最大公约数（我们必须照此跑）
所有主流竞品都按 mem0 的 LoCoMo 评测口径报数，要"能直接和别人比"，我们就必须复刻其口径：

- **流水线**：`Ingest（抽取+embedding入库）→ Search（按问题检索 top-k）→ Evaluate（answerer LLM 生成 + judge LLM 打分）`（mem0 memory-benchmarks 仓库）。
- **判定指标**：`F1 / BLEU-1 / LLM-as-Judge J`，**J 是头条指标**；judge 默认 GPT-4o，可 `--judge-model` 配置（mem0 论文 §3.2 + 仓库）。
- **judge 提示词刻意宽松**："be generous… 只要触及与金标同一主题即记为 CORRECT"（mem0 论文附录 A）。
- **丢弃对抗类**：mem0 因"对抗类无可用金标"而**剔除 adversarial**，只评 **4 类**（single/multi/temporal/open-domain，共 **1,540** 题）（mem0 论文 §3.1）。
- **多次运行报方差**：论文报 **10 次运行 mean ± 1σ**（§3.2）。

> **要可比，cosight-memory 的 LoCoMo 评测必须：4 类子集 + J(LLM-judge，附录 A 宽松提示) + 报 ≥10 次均值±σ + 披露检索 top-k 与 judge 模型。** 任一不一致，分数都不可与他人横比。

### 3.3 关键参照分数（这是我们要对标/超越的标尺）

**mem0 论文 Table 1（J，分类别）——经同行评审，最可信的"学术基线"：**

| 类别 | Mem0(向量) | Mem0ᵍ(图) | 最强基线 |
|---|---|---|---|
| 单跳 | 67.13 | 65.71 | OpenAI 63.79 |
| 多跳 | 51.15 | 47.19 | LangMem 47.92 |
| 开放域 | 72.93 | 75.71 | **Zep 76.60** |
| 时间 | 55.51 | **58.13** | A-Mem 49.91 |
| **总体 J** | **66.88 ±0.15** | **68.44 ±0.17** | OpenAI Memory 基线 +26% 相对提升 |

- 成本/时延（mem0 论文 Table 2）：mem0 检索 p50 0.148s、总 p50 0.708s、~7K token；**全上下文(full-context)基线 J=72.90 但 ~26K token、总时延 9.87s**。
- **重要细节（nuance）：全上下文裸喂仍比 mem0 的 J 更高**——mem0 的卖点是"时延/token 省 90%+"，不是峰值精度（mem0 论文 §4.3）。我们汇报时应同时给 **J + token + 时延**三元组。

**厂商自报高分（务必与论文分开、标注 `[厂商自报/未独立复现]`）：**
- mem0 仓库"新记忆算法"（2026-04）：LoCoMo **91.6**、p50 0.88s、7.0K token（mem0 仓库 README）`[厂商自报]`。
- OpenViking README 📊 节：`OpenClaw 24.20% → 82.08%（+3.39×，token −91%）`、`Claude Code 57.21% → 80.32%（token −63.2%）`、`mem0 56.62%`、`SuperMemory 42.99%`（OpenViking `README_CN.md`）`[厂商自报]`。
  - **这条对我们极有价值**：OpenViking 给出了 `OpenClaw` 与 `Claude Code` 作记忆基线的 LoCoMo 数字，cosight-memory 可与之**同口径对号入座**。

---

## 4. LoCoMo 的可靠性风险（必须读，决定汇报口径）

> 结论先行：**LoCoMo 裸分不可单独对外引用。** 我们对 `dial481/locomo-audit` 做了原始数据文件级独立复算，下列数字均已逐一核对一致（附录 C）。

### 4.1 审计来源
`github.com/dial481/locomo-audit`（"Full audit of the LoCoMo benchmark"）。我们克隆仓库、解析 `errors.json / data/locomo10.json / 各 AP 评分 JSON / per_category_breakdown.json`，并重算 ceiling/Wilson/token，**所有头条数字精确吻合**。

### 4.2 七大缺陷（含数字，均已复算）
1. **错误金标 6.4%**：1,540 题中 99 题金标错误（33 幻觉 / 26 时间错误 / 24 归因错误 / 13 歧义 / 3 不完整）；理论满分上限仅 **93.57%**（`AUDIT_REPORT.md`）。
2. **上限被"越界"**：EverMemOS 单跳 96.08%(808/841) > 单跳合法上限 95.72%(805/841)——说明≥3 个"正确"落在被污染的题上（`per_category_breakdown.json`，已复算）。
3. **Judge 过宽**：对"模糊但切题"的故意错误答案，judge 误判为正确率 **62.81%**；对"具体但错"的为 10.61%（**6× 宽松度**）；在 99 道真污染题上原 judge 仍判对 **242/495 = 48.9%**（`ap-baseline/README.md`，已复算 0.62814/0.10606）。
4. **Token 成本失真**：README 称 EverMemOS 2,298 token/题，论文自身 Table 8 实为 **6,669 / 6,045**（2.6–2.9×）；真实相对节省 67–70% 而非声称的 89%；EverMemOS 实为 **2.31 次 LLM 调用/题**，是最贵而非最省（`methodology/token_efficiency.md`）。
5. **可复现性差**：第三方 `EverMind-AI/EverMemOS#73` 复现 **38.38% vs 声称 92.32%**（53.94 点差）。**但审计如实标注**：该 38.38% 仅在 conv-26 单会话(152 题)、不同技术栈下测得，是"红旗"而非严格对照（`methodology/reproducibility.md`）。
6. **类别样本极不均衡**：评测类别 96→841（8.8×），且 **446 道对抗题(22.5%)被整体排除**（`STATISTICAL_VALIDITY.md`，已对 `locomo10.json` 复算 {1:282,2:321,3:96,4:841,5:446}）。
7. **是"提示词"而非"记忆系统"在拉分**：审计实测全上下文基线用 CoT 提示得 **92.62%**，反超 EverMemOS 92.32%；换成 5-6 词短提示降到 81.95%——**单提示词差异即造成 10.67 点波动**（`fc-baseline/README.md`）。

### 4.3 EverMemOS/EverOS 争议（"质疑证据"书签的落点）
- arXiv **2601.02163** 声称 LoCoMo 92.32% 为 SOTA；但 `EverMind-AI/EverOS#73` 多名第三方无法复现（维护者 cyfyifanchen 回应"本地可跑"并引流至 Discord/微信，issue 仍 **open**）；`#41` 提出"评测脚本应标准化"。
- 结论：**EverMemOS 的 SOTA 宣称应保留质疑**，不可作为我们对标的硬标尺；其评测方法（2-3 次 LLM 调用 + 729-token CoT + agentic 检索）与竞品单次调用不同口径，却同表对比——这正是"不可比"的反面教材。
- **同时提醒利益相关**：EvoAgentBench 与 EverOS 同属 **EverMind-AI**，引用其结论时应交叉验证。

### 4.4 审计给出的"公平评测"规范（我们直接采纳为护栏）
应用 `errors.json` 修正后再评；按类别报 **Wilson 95% CI**，CI 不重叠才声称差异；报**总** token（提示+检索+生成+调用次数）；用**同一提示词、同一模型**的全上下文基线做对照；用确定性/多选式打分修正 judge 宽松；真正评测对抗类(弃权)。

---

## 5. 对话域备选/补充框架对比

| 框架 | 出处/权威性 | 测什么 | 是否含 LoCoMo | 是否可插拔自有记忆 | 对我们的价值 |
|---|---|---|---|---|---|
| **LongMemEval** | arXiv 2410.10813, ICLR'25 | 5 能力：信息抽取/多会话推理/时间推理/**知识更新KU**/**弃权ABS**；500 题；S(~115K)/M(~1.5M token) | 否（独立集） | 是 | **首选学术补充**：KU/ABS 正好补 LoCoMo 盲区，且契合 Claude Code"新鲜度/验证"设计 |
| **MemoryAgentBench** | ICLR'26, HUST-AI-HYZ | 4 能力：精确检索AR/测试时学习TTL/长程理解LRU/**冲突消解CR**；"注入一次多次查询" | 否 | **是，内置 Mem0/Letta/Cognee/HippoRAG 适配器** | **首选工程框架**：可直接把 cosight-memory 接成新 adapter，复用其多基线对比 |
| **SubtleMemory** | arXiv 2606.05761 | 细粒度**关系记忆辨别**、长程 | 否 | 是 | LoCoMo 的更难后继，适合做"区分度"压力测试 |
| **STATE-Bench** | 微软, 2026-05 | 450 任务（客服/出行/购物），程序化+有状态+以用户为中心；pass⁵、用户模拟器 | 否 | 是 | 企业有状态任务，**纯对话无代码**；可作"落地场景"补充 |
| **OpenViking** | 火山引擎 | LoCoMo + ClawWork(经济模拟) + HotpotQA | **是** | 它本身是记忆层 | **直接提供 OpenClaw/Claude Code LoCoMo 基线数字**，强对标价值 `[厂商自报]` |
| **Vectorize AMB** | Vectorize.io（工业） | LoCoMo+LongMemEval+**agentic 任务**；分别计 ingest/retrieve/generate 的精度/速度/成本 | 是 | 是 | **成本感知**理念值得借鉴（"90%精度但$10/天 不如 82%且$0.1/天"） |

---

## 6. 代码域：候选数据集与框架对比（本次需求的难点与重点）

> 代码域"记忆"有两种语义：(A) **跨任务经验复用**（解过的相似问题能否帮新问题）——最贴近"记忆插件"价值；(B) **长上下文/多轮记忆保持**（单个大任务内跨轮不丢信息）。两者都要覆盖。

### 6.1 SWEContextBench —— 代码域"经验复用"首选（语义 A）
- 出处：arXiv **2602.08316** *"SWE Context Bench: A Benchmark for Context Learning in Coding"*；数据集 `huggingface.co/datasets/jiayuanz3/SWEContextBench`。
- 设计：**1,100 base 经验任务 + 376 related 任务**，源自 GitHub issue/PR 的真实依赖与引用关系；**51 个仓库 / 9 种语言**；构建在 **SWE-Bench（Lite/Verified/Multilingual）** 之上。
- 数据结构：`Experience.parquet`（经验池）+ `Related.parquet`（新问题）+ `Relationship.parquet`（关系映射），字段沿用 SWE-Bench（instance_id/patch/repo/problem_statement/test/commit）；附 Docker 镜像与公开评测代码。
- 评测口径：在"不同上下文复用设置与检索策略"下，看**正确摘要+检索的经验能否提升解决率、降低运行时与 token**；论文结论：**好经验显著提升（尤其难题），错/未过滤经验反而负收益**（arXiv 2602.08316 摘要）。
- **为什么首选**：① 语义最贴"记忆=经验复用"；② 基于 SWE-Bench → 与海量 SWE 工作**天然可比**；③ 有 Docker + 公开评测脚本 → 可复现；④ 2026-02 新近、覆盖 9 语言。
- 注意：`[需核实]` 论文摘要未列公开排行榜与具体基线表数字，落地前需跑通官方 Docker 评测确认基线。

### 6.2 LoCoBench-Agent —— 长上下文软件工程 Agent 权威补充（语义 B）
- 出处：arXiv **2511.13998**，**Salesforce AI Research**；"首个面向软件工程的长上下文 LLM Agent 基准"。
- 规模：**8,000 个交互式场景 / 1,000 个项目 / 10 种语言**；上下文 **10K–1M token**、4 个难度；**最多 50 轮**；**8 个工具**（读/写/搜替/列目录 + grep/glob/codebase_search + 代码分析）。
- 指标：**9 个指标两维度**——理解维(LCBA-Comprehension)含 **"多会话记忆保持(Multi-Session Memory Retention)"**、跨文件一致性、依赖遍历等；效率维(LCBA-Efficiency)含运行时/空间复杂度/信息覆盖/长程依赖解析；分数重标定到 [0.40,0.90] 去天花板效应。
- 现状：论文报 6 个模型（GPT-5/GPT-4.1/GPT-4o/Claude Sonnet 4.5/Sonnet 4/Gemini 2.5 Pro），理解 0.71–0.74、效率 0.60–0.63；**未提公开排行榜**（arXiv 2511.13998 v1）。
- **为什么入选**：① 大厂出品、权威性高；② **显式含跨轮记忆保持指标**，直测语义 B；③ 交互式多轮，贴合 Agent 真实工作流。

### 6.3 EvoAgentBench —— 直接含 OpenClaw + 记忆方法，强烈建议接入
- 出处：`evermind-ai.github.io/EvoAgentBench/`（**EverMind-AI**，与 EverOS 同团队——注意利益相关）。
- 设计：5 域——信息检索(BrowseCompPlus)、推理(OmniMath)、**软件工程(SWE-Bench, 101/26)**、**代码实现(LiveCodeBench, 97/39)**、知识工作(GDPVal)。
- **关键**：被测 Agent 直接含 **`OpenClaw`（27B/397B）** 与 Nanobot；评测记忆/自进化方法 **EverOS / Memento / ReasoningBank / GEPA** 带来的 **Δ 提升**（如 ReasoningBank 在 SWE 上 OpenClaw·397B **+40**）。
- **为什么接入**：这是**唯一一个已把 OpenClaw + 记忆方法挂在代码域榜上**的框架——cosight-memory 可作为"又一种记忆方法"直接对号入座、与 EverOS/ReasoningBank 同台。但因团队利益相关，需交叉验证、并辅以 SWEContextBench/LoCoBench-Agent。

### 6.4 其余代码/多域框架（备选，证据较弱）
| 框架 | 测什么 | 代码域? | 对我们价值 | 备注 |
|---|---|---|---|---|
| **AgentMemoryBench** (solomoon313) | 5 模式(Online/Offline/Replay/**Transfer**/**Repair**) × 6 交互任务，系统+个人记忆 | 部分 | Transfer/Repair 模式有特色 | 单团队，`[需核实]`基线 |
| **MemGym** (WujiangXu) | 长程记忆环境 | 多域 | 环境式压力测试 | 单团队，`[需核实]` |
| **Vectorize AMB** | agentic 任务 + 成本核算 | 含工具调用 | 成本感知方法论 | 工业、强调可复现 |
| **TencentDB-Agent-Memory** | 产品+基准数字 | 存储侧 | 工程对标 | 产品 README，非中立基准 |

---

## 7. 横向总表（决策一图流）

| 维度 | LoCoMo(+mem0协议) | LongMemEval | MemoryAgentBench | SWEContextBench | LoCoBench-Agent | EvoAgentBench |
|---|---|---|---|---|---|---|
| 域 | 对话 | 对话 | 对话(多能力) | **代码(经验复用)** | **代码(长上下文)** | **多域(含代码)** |
| 权威性 | 事实标准(但有缺陷) | ICLR'25 高 | ICLR'26 高 | 新近 2026-02 | Salesforce 高 | 单团队(利益相关) |
| 可比性/榜单 | **全行业都报** | 论文可比 | 多基线内置 | 基于 SWE-Bench 可比 | 论文可比、无公开榜 | **已含 OpenClaw 对号** |
| 可插自有记忆 | 是(照 mem0 口径) | 是 | **内置 adapter** | 是(Docker) | 是 | 是(作记忆方法) |
| 测"记忆"语义 | 稠密事实召回 | IE/MR/TR/KU/ABS | AR/TTL/LRU/CR | 跨任务经验复用 | 多会话记忆保持 | 自进化/技能积累 |
| 对我们结论 | **①主战场** | ①学术补充 | ①工程框架 | **②首选** | **②权威补充** | ②对号入座 |

---

## 8. 可比性策略（"能直接和别人比"的硬保障）

1. **协议钉死（Protocol Pinning）**：对话域固定 `mem0 LoCoMo 口径`（4 类 / J / 附录 A judge 提示 / GPT-4o judge / top-k 披露 / 10 次 ±σ）；代码域固定 `SWE-Bench 解决率(resolved %)` + `token/runtime` 作主指标。
2. **同口径基线自跑**：不要直接引用他人表里的数字横比（judge 模型/提示/类别子集常不同）——按审计建议，**自己用同提示同模型把 mem0、全上下文基线、OpenClaw 裸基线重跑一遍**，再放进同一张表。
3. **三元组汇报**：每个分数都带 `精度 J / token / 时延`，对齐 mem0 与 Vectorize AMB 的成本感知范式。
4. **置信区间**：按类别报 Wilson 95% CI，CI 不重叠才宣称领先（locomo-audit 规范）。
5. **代码域挂公开锚点**：通过 EvoAgentBench 把 `OpenClaw + cosight-memory` 与 EverOS/ReasoningBank 等同台；通过 SWEContextBench/LoCoBench-Agent 与 SWE-Bench 生态对齐。

---

## 9. 自建代码域私有数据集方案（第④步）

为保证私有集"与公开集同构、脚本可迁移、结果可横比"：
1. **数据规范沿用 SWE-Bench**：字段 `instance_id/repo/base_commit/patch/test_patch/problem_statement/FAIL_TO_PASS`，以便直接复用 SWEContextBench / LoCoBench-Agent 的 Docker 评测与解决率口径。
2. **关系对构造法沿用 SWEContextBench**：从我司内部 issue/PR 的**依赖与引用关系**抽取 `base 经验任务 → related 新任务` 对，构造 `Experience/Related/Relationship` 三件套，从而能评"经验复用"而非孤立解题。
3. **难度与语言分层**：覆盖我司主力语言与历史难修复缺陷，按 LoCoBench-Agent 的难度/上下文长度分桶，支持长上下文记忆保持评测。
4. **双指标**：`解决率提升 Δ`（vs 无记忆基线）+ `token/runtime 下降`，与公开集完全一致。
5. **去污染**：私有集需做训练数据泄漏检查（commit 时间、是否进入公开模型训练语料），避免高估。

---

## 10. 推荐路线图（落地节奏）

| 阶段 | 动作 | 产出 | 关键护栏 |
|---|---|---|---|
| P0（1-2 周） | 搭 mem0 LoCoMo 协议 + 套 locomo-audit 修正；接 cosight-memory | `OpenClaw+cosight` vs mem0/全上下文/OpenClaw裸基线 同口径表 | 4 类/J/10次±σ/Wilson CI |
| P1（并行） | 调优 cosight-memory（补稠密召回）；加 LongMemEval-S 的 KU/ABS | 调优曲线 + 学术级交叉验证 | 显式标注"能力错位"已补齐 |
| P2（2-4 周） | 接 SWEContextBench（Docker）+ LoCoBench-Agent；并在 EvoAgentBench 对号 | 代码域经验复用 + 长上下文记忆保持 + OpenClaw 同台 | SWE-Bench 解决率 + token/runtime |
| P3 | 按 SWE-Bench 规范自建私有代码集 | 私有榜，脚本与公开集复用 | 去污染 + 双指标 |

---

## 11. 风险与注意事项（给领导的"避坑清单"）
1. **LoCoMo 裸分对外有公信力风险**——必须随附审计修正与 CI，绝不单独引用 92% 这类数字。
2. **厂商自报分数（mem0 91.6 / OpenViking 82% / EverMemOS 92.32%）一律标 `[未独立复现]`**，不可当硬标尺；EverMemOS 已被公开质疑无法复现。
3. **EvoAgentBench 与 EverOS 同团队**，引用其代码域结论需交叉验证。
4. **能力错位**：原生 Claude Code 记忆不为 LoCoMo 而设，低分≠系统差；调优方向是"补稠密召回线"。
5. **可比性脆弱**：judge 模型/提示/类别子集任一不同，分数即不可比——坚持"同口径自跑全部基线"。
6. **部分单团队基准（SWEContextBench/SubtleMemory/AgentMemoryBench/MemGym）的具体基线表尚需跑通官方脚本核实**，本报告对其结论标注了 `[需核实]`。

---

## 附录 A：被测系统能力对照（Claude Code 原生 vs 对话记忆层）
| 能力 | Claude Code 原生（`docs/08-memory-system.md`） | LoCoMo 所需 | 差距 |
|---|---|---|---|
| 存储取向 | 只存不可推导信息，排除代码/路径(§6.1-6.2) | 稠密对话事实 | 需补事实抽取 |
| 召回 | mtime 排序 + Sonnet 选≤5(§6.5) | 向量/图 top-k 检索 | 需补稠密检索 |
| 抗漂移 | 新鲜度告警 + 引用前验证(§6.6) | 弱 | **优势，LongMemEval ABS/KU 可体现** |

## 附录 B：核心引用清单（链接）
- LoCoMo: arXiv 2402.17753；data `github.com/snap-research/locomo`
- LoCoMo 审计: `github.com/dial481/locomo-audit`
- mem0: arXiv 2504.19413；`github.com/mem0ai/mem0`、`mem0ai/memory-benchmarks`
- LongMemEval: arXiv 2410.10813（ICLR'25）
- MemoryAgentBench: `github.com/HUST-AI-HYZ/MemoryAgentBench`（ICLR'26）
- SubtleMemory: arXiv 2606.05761；`github.com/Yummytanmo/SubtleMemory`
- STATE-Bench: `opensource.microsoft.com/blog/2026/05/19/...`
- OpenViking: `github.com/volcengine/OpenViking`（`README_CN.md` 评测节、`/benchmark`）；`github.com/ZaynJarvis/openclaw-eval`
- EverMemOS/EverOS: arXiv 2601.02163；`github.com/EverMind-AI/EverOS`（issues #73/#41）、`github.com/lipps/EverMemOS`
- SWEContextBench: arXiv 2602.08316；`huggingface.co/datasets/jiayuanz3/SWEContextBench`
- LoCoBench-Agent: arXiv 2511.13998；`github.com/SalesforceAIResearch/LoCoBench-Agent`
- EvoAgentBench: `evermind-ai.github.io/EvoAgentBench/`
- AgentMemoryBench: `github.com/solomoon313/AgentMemoryBench`
- MemGym: `github.com/WujiangXu/MemGym`
- Vectorize AMB: `github.com/vectorize-io/agent-memory-benchmark`、`agentmemorybenchmark.ai`
- TencentDB-Agent-Memory: `github.com/TencentCloud/TencentDB-Agent-Memory`
- PowerMem: `powermem.ai/docs/architecture/overview`
- Awesome 列表: `github.com/TsinghuaC3I/Awesome-Memory-for-Agents`、`github.com/TeleAI-UAGI/Awesome-Agent-Memory`

## 附录 C：校验说明
- LoCoMo 审计全部头条数字（6.4% 错误率、93.57% 上限、62.81%/10.61% judge 误判、242/495、类别 {1:282,2:321,3:96,4:841,5:446}、Wilson CI、`data/locomo10.json` 与 `prompts.yaml` 的 SHA256）均经独立复算一致。
- 标注 `[厂商自报]/[未核实]/[需核实]` 者，表示尚未独立复现，落地前应跑官方脚本确认。
- 注意：个别 GitHub 文件树经 WebFetch 读取时存在幻觉路径；凡涉及具体文件路径的结论均以"克隆仓库后实读"为准。
