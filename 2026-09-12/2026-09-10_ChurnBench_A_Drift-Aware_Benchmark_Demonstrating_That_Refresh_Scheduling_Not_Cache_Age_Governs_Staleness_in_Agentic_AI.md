# ChurnBench: A Drift-Aware Benchmark Demonstrating That Refresh Scheduling, Not Cache Age, Governs Staleness in Agentic AI

> 一句话 TL;DR：把企业数据世界生成为一条"时间线"而非快照、用 append-only ledger 当唯一真值，从而把"检索时对、评测时已错"的 freshness error 单独检出，并据此证明 staleness 由 TTL 刷新调度而非 cache age 决定。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | ChurnBench: A Drift-Aware Benchmark Demonstrating That Refresh Scheduling, Not Cache Age, Governs Staleness in Agentic AI |
| **作者 / 机构** | Vivek Kumar Singh（ORCID 0009-0002-9350-3207）、Preeti Priyam（ORCID 0000-0000-9423-741X）；Independent Researcher, McKinney, TX（美国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-10；arXiv preprint（IEEE 格式预印本） |
| **arXiv 链接** | https://arxiv.org/abs/2609.11515（点击直达） |
| **代码仓库** | ✅ GitHub：https://github.com/vsingh45/churnbench（本体、评测 harness、全部逐错误数据均开源，另附 ARCHITECTURE.md） |
| **数据集地址** | 与代码同仓：https://github.com/vsingh45/churnbench —— 报告的 run 所用 ledger 已 commit，并附生成命令与 seed；相同 seed 产生相同 ledger |
| **类型标签（论文类别）** | `Benchmark` `General` |
| **训练方法标签** | `—（评测/工程，不训练）` |
| **关键词** | data freshness、staleness、TTL、RAG grounding、enterprise agent benchmark、temporal drift |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-10_ChurnBench_A_Drift-Aware_Benchmark_Demonstrating_That_Refresh_Scheduling_Not_Cache_Age_Governs_Staleness_in_Agentic_AI.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：企业 Agent 的第二个职责被忽视——**确保推理之下的数据来自正确的源、被正确 join、并且仍然是真的**。
- **为什么重要**：这类失败**在既有评测里不可见且设计上不可测**——基准都冻结语料，**冻结语料不可能产生过期答案**；staleness 住在**缓存、物化聚合与向量索引**里，supersede 规则**对物化聚合无效**。
- **现有方法不足**：检索基准（BEIR、MS MARCO、MuSiQue、WixQA）各为固定快照；企业 Agent 基准（AgentArch 等）**无 drift 注入、无 freshness 测量**；MemStrata 管的是 Agent 记忆里的 stale fact，本文在 grounding 层。
- **Research Gap**：没有基准落在"**多源结构化 + 非结构化企业数据、受控 drift、基于执行的评分、per-task 成本**"这个交叉点。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

不训练任何模型，而是造一台**测量仪器**：模拟器把企业活动推演成**时间线**、每次变更写入 **append-only ledger**，live store 只是 ledger 在某时间戳上的 **fold**，gold 由**非模型 resolver** 计算；"检索时对、评测时已错"即 **freshness error**。

### 3.2 方法总览（Pipeline）

- **输入 / 输出**：180 个参数化模板任务（intent：spend visibility / savings opportunity / criticality / utilization；难度单源 81、跨源 63、跨模态 36）；世界在 **T** 冻结、缓存与索引在 **T′** 构建，差值即 **cache age**；输出 **accuracy**、**美元成本**、**latency**、**freshness error 标记**。
- **模块**：(1) **模拟器 + ledger**——逐日 Poisson 流抽招聘/离职/重分配/价格变更/续签/成本中心迁移，同 seed 同 ledger。(2) **四源 fabric**——PostgreSQL 16、MongoDB 7、mock REST（**350ms、限 60 请求/分钟**）、合同文档。(3) **语义注册表**——每实体类记 **tier**、**TTL**、**上次刷新时间**（hot 1 天 / warm 7 天 / cold 30 天）。(4) **规则式路由**，连同**每个被读实体的 last-refresh** 记入 trace。(5) **分层刷新**——逐日刷新 TTL 已过期实体，**受控对比中唯一被改动的变量**。(6) **gold resolver** 由 ledger 算、**从不从 live store 算**。
- **关键洞察**：判定规则是"**在 T 上错、但在 T_eff 上对**"；T_eff 由 trace 推导——live 路径取 T，staged 路径取实际读到实体的 last-refresh，多实体取最小值。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：(1) **"freshness error" 的可验证定义**（`在 T 错 ∧ 在 T_eff 对`）——价值在**归因**：在更早世界为真的答案可用刷新策略修；论文对**每一个** freshness error 都在两个时间戳上解析了 ledger。(2) **ledger 作唯一真值 + live store 只是 fold**——gold 路径与执行路径**完全解耦**。(3) **被明确报告的负面结果 + 完全匹配的受控消融**；**测量陷阱**：TTL lapse 计数在两种相反配置下都读 0，可靠仪器是 **per-entity refresh timestamps**。(4) **per-task 成本归因**。
- **工程组合**：四源 fabric、registry + 规则路由、tiered refresh、LangGraph 等**全是成熟工程件**；被设计的是**它们如何被组织成可审计的仪器**。
- **对性能最关键的设计**：决定**核心结论**的是 **tiered refresh 的开与关**（唯一变量，28 天窗口 4 → 45、1 天窗口 7 → 7）；决定 **accuracy** 的**不是架构而是注册表覆盖率**——180 题触达 22 个 measure，**9 个注册、13 个没有**；**109/180** 题退化为**模型生成 SQL**，51–69 个 reasoning error 里 **49–53 个**来自这些题。
- **证据不足 / 仅声称有效**：(1) **"staleness 暴露 = tier 宽度 × 实体变化率的乘积"证据很薄**——仅一张 4 行 matched-pair 表与 4 / 45，**无拟合支撑**。(2) **没有基线对比**——三个 arm 在代码里但**没在最终 180 题集上评测**。(3) **三窗口 sweep 配置不匹配**（14 天那格开着 reasoning mode）。(4) **标题主张的那个变量本文自己没 sweep**。

---

## 4. 具体技术细节

### 4.1 模型结构

- **本文不训练任何模型**，它是 benchmark + 测量协议 + 参考实现。
- **被测系统配置**：Table I 的 run 用**单一模型** `nvidia/nemotron-3-ultra-550b-a55b`（NVIDIA NIM，`temperature = 0`，**关闭 reasoning mode**），framework 为 **LangGraph**；Table III 的三窗口 sweep 更早执行、**开着 reasoning mode**。约 **7,500 行 Python + 316 个测试**。
- **成本与规模**：一次 180 题 run 约 **1.50 美元**（reasoning 开）/ **0.50 美元**（关），每任务 **0.008 / 0.003 美元**；ledger 含 **56,370** 个事件，其中 **54,000** 是消费事件，另有 717 招聘、605 重分配、237 离职、77 价格变更、**仅 27 合同续签**。

### 4.2 训练流程

**本论文不训练任何模型**（既不训练 LLM 也不训练评分器；gold 由非模型 resolver 从 ledger 计算）：

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| **不训练** | — | — | — | — | — |

机制分三段，均**无 loss**：(A) **生成器**——确定性模拟，逐日事件写入 ledger，live store 由 ledger 在指定时间戳 fold 得到；(B) **刷新协议**——按 tier 与 TTL 逐日刷新，**受控对比中唯一被改动的开关**；(C) **判定协议**——由 trace 算 T_eff 后在两个时间戳上各解析一次 ledger。

- **可复现性**：result 文件记录 commit hash、model string、T′、T、seed、task-set hash。
- **重试的诚实披露**：**四次 run 遭遇接口不稳定**（含产生最大效应的那次 ablation run，4 个失败是**事后审计已提交 result 文件时才发现**）；重试中 freshness-error 计数**均未改变**——判定要求存在"匹配更早世界"的实质性答案，**stub 不可能匹配任何世界**。

### 4.3 推理流程（若不训练或重点在推理）

- **使用什么模型**：同 4.1（temperature 0，关闭 reasoning mode，LangGraph）。
- **输入 / 输出**：企业问题 + 四源数据；输出答案 + 一条**必须记录的 trace**（每个 fact 由哪个源提供、何时被最后刷新）。
- **单步还是多步**：**不是反思/规划式多步循环**，核心是**规则式路由 + 一次模型调用**；没有 trace 就无法推导 T_eff。
- **保留的"不可回答"问题**：**事件历史类**与**逐许可证成本合计**，理由是**只有可回答问题构成的 benchmark 会高估系统能力**。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| **ChurnBench**（本文提出的基准 + 参考实现） | General（企业软件资产管理；非 GUI 截图场景，四源异构数据 fabric） | 在漂移数据上的企业问答（跨源 join + 跨模态结合合同散文）；核心测量对象是**数据时效性失败** | 180 个任务（tier 1/2/3 = 81/63/36；intent 分布 50/38/48/44）；底层 ledger **56,370** 个事件；单次 run 触达 22 个 measure（9 个已注册、13 个未注册） | 自然语言问题 + 四源数据（PostgreSQL 16 / MongoDB 7 / 限流 mock REST / 渲染的合同文档）；世界在 T 冻结、缓存与索引在 T′ 构建 | 答案 + trace（每个 fact 的来源与 last-refresh 时间戳） | accuracy、**freshness error**、美元成本（精确 token 归因）、端到端 latency |

**对照设置**：matched pair（Table I）= 2×2（tiered refresh {enabled, disabled} × cache age {1 天, 28 天}，180 题/格）；cache-age sweep（Table III）= tiered 开启下扫 1/14/28 天；per-entity 归因（Table II/IV）；accuracy by tier（Table V）。

### 5.2 实验结果分析

**主结果（Table I，matched pair，freshness error 计数）**：

| Refresh 配置 | 1 天 cache | 28 天 cache |
|---|---|---|
| Tiered refresh **enabled** | **7** | **4** |
| Tiered refresh **disabled** | **7** | **45** |

关闭分层刷新在 28 天 cache age 下把 freshness error 抬高十一倍，1 天下完全不变（都恰好 7 个、来自**同样 7 个任务与实体类**）——**不对称性就是发现本身**，是 **TTL 界定的机制**而非 cache age 驱动。

**机制的直接证据（Table II，28 天 cache age 下各实体类的实际 cache age，单位天）**：

| 实体类 | TTL (d) | Tiered (d) | Untiered (d) |
|---|---|---|---|
| assignments | 1 | **1** | **29** |
| user status | 1 | **1** | **29** |
| prices | 7 | **5** | **29** |
| contract terms | 30 | **29** | **29** |

分层开启时 1 天 TTL 实体始终在 1 天以内，关闭后**每个实体的 last-refresh 都等于缓存构建时间、29 天陈旧**；合同条款两配置相同，因为 **30 天 TTL 在 28 天窗口内永不过期**。

**cache-age sweep（Table III）**：1/14/28 天对应 **7 / 4 / 4**、accuracy **67.2% / 69.4% / 65.0%**、**TTL lapse 全为 0**；孤立看是无信息的 null，配上 Table II 就是机制——**任何窗口下都没有观测到 TTL 在查询时过期**，**被扫的 cache age 对 TTL 短于窗口的实体根本没到达检索层**；修法是改扫 **TTL 配置本身**。

**归因（Table IV）**：tiered 下 user status 0 / assignments 0 / prices **4** / contract terms 0 = **4**；untiered 下 **15 / 10 / 20 / 0 = 45**。分层开启时 15 个中 **14 个来自 prices**；**1 天 tier 与 30 天 tier 零错误**（后者因 **56,370 个事件里只有 27 次续签**）。**staleness 暴露由"tier 宽度 × 实体变化率"的乘积决定**；**tier 应依据实测变化率分配**。

**accuracy 披露（Table V）**：单源 **80.2% / 42.0%**、跨源 **47.6% / 17.5%**、跨模态 **55.6% / 55.6%**、全部 **65.0% / 36.7%**（tiered / untiered）；Tier 3 两设置相同（都在 T′ 构建文档索引）。该数字**主要测量 registry coverage**（9/22 已注册、109/180 题退化、51–69 个 reasoning error 里 49–53 个来自这些题），**不应读作架构结论**。

**消融**：唯一变量的 matched pair 证明决定 staleness 的是刷新调度而非 cache age，效应**在短窗口下恰好为零**；per-entity 归因把机制变成**可直接观察的年龄曲线**并揭示 lapse count 这个**测量陷阱**。

---

## 6. 亮点与贡献（Why it matters）

1. **"世界是时间线而不是快照"是可复用的基准设计原则**：兼顾 **gold 的可计算性**与**漂移的可控性**。
2. **freshness error 的定义可证伪**：**恰好匹配某个更早世界状态的错误答案满足第一个条件但不满足第二个**，被正确排除。
3. **报告负面结果的勇气**：把平坦的 sweep 与 per-entity 年龄曲线、零 lapse 计数、4 vs 45 的消融绑在一起，**把 null 变成了机制**。
4. **主动披露"accuracy 主要测量 registry coverage"**。

## 7. 局限与可改进点（个人点评）

- **支撑标题主张的那张表配置不匹配**：Table III 的 **14 天那格开着 reasoning mode**，且它恰是唯一出现 4 而非 7 的窗口。
- **统计功效薄**：计数只有 4–7，乘积律仅由 **4 个数据点**定性概括，**无网格扫描或拟合**。
- **没有任何基线**：Table V 的 accuracy 没有外部参照点。
- **单域 + 合成数据 + 两路未标定变化率**，结论不脆弱但未必可迁移。
- **frozen-at-T 放弃了 run 中途的漂移**，"staleness vs 成本"未被受控测量。

## 8. 对我们的启示 / 可借鉴点

- **"在变化的数据上评测 Agent"应当成为一等评测类别**：我们的冻结环境同样看不见"世界已变"这类失败。
- **gold 必须来自与执行路径解耦的真值源**：用 LLM judge 打分的评测应**让判定路径尽量不含模型**。
- **"把失败按归因分类"比"报一个总分"更有用**：拆成"可被系统修"与"只能被模型修"。

### 对 GUI Agent 的可借鉴点

1. **GUI Agent 的"界面时效性"是同一类问题，且几乎没人测**：界面状态同样会变，而 GUI 评测几乎全在冻结环境下进行，**"基于过期观测做了正确推理但动作已失效"完全不可见**；可把"页面状态"做成时间线（每个 UI 元素带 last-observed 时间戳）。
2. **per-entity refresh timestamp 可迁移为 per-observation provenance**：每个 a11y node / 截图都应带上"这个 UI 快照是什么时候取的"，**记录时间戳比记录事件计数有诊断力**。
3. **"TTL 配置 × 变化率"可迁移为"缓存策略 × 页面变化率"**：实时变化区域用极短 TTL，稳定区域用长 TTL；准则是**依据实测变化率定级**。
4. **可复现性范式值得直接抄**：确定性 timeline + ledger + 非模型 resolver + 结果文件记录 commit/model/seed。

## 9. 延伸阅读

- **时序漂移**：RAG or Learning? arXiv:2604.05096（本文动机来源）；MemStrata，arXiv:2606.26511（staleness 在 Agent 记忆而非 grounding 层，互补）。
- **检索基准**：BEIR（arXiv:2104.08663）、MS MARCO（arXiv:1611.09268）、MuSiQue（arXiv:2108.00573）。
- **基于执行的评分**：BIRD，arXiv:2305.03111。
- **框架**：LangGraph、JOLTS、Protocol-H；代码 https://github.com/vsingh45/churnbench。

---
*解读生成时间：2026-09-12 ｜ 解读人：WorkBuddy（AI）*
