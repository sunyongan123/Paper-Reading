# SearchAtlas: Analyzing Agentic Search Strategies via Evidential Query Graphs

> 一句话 TL;DR：把每条搜索轨迹转成"查询→查询"的证据依赖 DAG，边由可归因的检索证据门控，再由图结构导出三个按问题约束类型条件化的过程诊断指标。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | SearchAtlas: Analyzing Agentic Search Strategies via Evidential Query Graphs |
| **作者 / 机构** | Jiacheng Sang\*、Mengyuan Li\*、Sanxing Chen、Yukun Huang、Yu Feng、Bhuwan Dhingra（\*同等贡献）。机构：Duke University、University of Pennsylvania（美国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-09；Findings of EMNLP 2026（原文 txt 未打印 venue 行，按任务元数据） |
| **arXiv 链接** | https://arxiv.org/abs/2609.10901 |
| **代码仓库** | ⚠️ 论文脚注 1 声明 "Code and data are available at https://github.com/DukeNLP/SearchAtlas"（经 PDF 链接注解核实确为原文地址），但**该地址截至 2026-09-12 返回 404**，疑为尚未公开 |
| **数据集地址** | ⚠️ 论文称代码与数据同仓库（https://github.com/DukeNLP/SearchAtlas，当前 404）；实验使用公开 benchmark：BrowseComp、WebWalker-Hard-English、DeepSearchQA（含题目 ID 与筛选细节见附录 A.9） |
| **类型标签（论文类别）** | `Planning` `Reflection` `Benchmark` `General` |
| **训练方法标签** | `—（分析框架/评测，不训练）` |
| **关键词** | search agents；evidence dependency DAG；process diagnostics；trajectory interpretability；constraint grounding；prior-knowledge reliance |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-09_SearchAtlas_Analyzing_Agentic_Search_Strategies_via_Evidential_Query_Graphs.pdf |

---

## 2. 论文要解决的核心问题

- **问题与重要性**：搜索 agent 主要按**最终答案正确率**评测，过程被忽略；而两个 agent 可能用截然不同的方法得到同一答案。论文要把压平成时间序的轨迹转成显式暴露**证据依赖**的结构表示。必要性有实证：三个诊断与正确性的关联（macro AUC 0.840–0.856）**强于** LLM judge 看完整日志（0.738）或有序 query 列表（0.677）——**结构化证据依赖携带了非结构化轨迹所不含的信息**。
- **现有方法不足**：评测工作绝大多数只看最终产出；少数考察过程的（DeSA、Agent-RRM、PPR、RE-TRAC）**把过程当训练信号或步级质量判断，却不分析检索到的证据在一次运行中如何流动**。图表示工作则或是**规范性**的（ToT、GoT、GPTSwarm），或是**事后恢复**的（ReasoningFlow、Graph of Verification、WebGraphEval——最接近，但**把多次运行聚合成 consensus graph**）。
- **Research Gap**：SearchAtlas **一次只处理一条轨迹**，边由**可归因的检索证据**门控（复用片段、访问过的页面、显式失败）。贡献：DAG 表示 + 经人工标注验证的构建流水线；5 agent × 3 benchmark 的规模刻画与三个**按约束类型条件化**的诊断。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把一条轨迹的**可观察证据流**表示成带类型的 DAG（节点：原问题 q₀、各次查询 qᵢ、未归因先验知识源 PK、最终答案 A；边仅在"u 处内容可观察地贡献于 v"时建立），并由此导出三个诊断指标。

### 3.2 方法总览（Pipeline）

- **四类边**：**constraint-use（q₀→qᵢ）** 查询针对原问题的某个要求；**evidence-use（qᵢ→v）** 后续查询或答案依赖 qᵢ 检索到的事实；**failure-response（qᵢ→qⱼ）** qⱼ 响应早期检索的**显式失败或不足**（硬失败如零结果，软失败如结果不足）；**prior-knowledge（PK→v）** v 的内容既不来自 q₀ 也不来自任何早期检索证据。
- **归因规则与流水线**：答案侧先拆成**事实单元**（人名/日期/数字/缩写），逐单元找最小支撑查询集并连 `qᵢ→A`，无支撑者连 `PK→A`（通往 A 路径上的边称 **answer-reaching**）；重叠支撑只保留**最小父集**。构建分两阶段：**确定性预处理**（抽节点与检索结果、检测失败）与 **LLM 归因**（逐查询给 LLM 原问题 + 当前推理块 + 早期查询及其证据，识别 evidence-use 边）——唯一非确定性来源。

### 3.3 去伪存真

- **真·方法创新**：边由"可归因的检索证据"门控，只连"可见贡献"而非"可能相关"；**按约束类型条件化诊断**——sequential（约束成依赖链）与 parallel（多个可独立验证的约束）的**理想证据结构不同**，同一指标期望值相反；**"约束是否落在 answer-reaching backbone 上"的局部化视角**。
- **工程组合**：DAG 表示、LLM 归因、ROC-AUC、与 LLM judge 对比、等权 z-score 相加。
- **对结论最关键的设计**：**backbone 局部化**（BrowseComp 上 trajectory-wide 0.459 vs DAG-localized **0.764**；两列用**相同约束单元**，差异完全来自局部化）；**PK 的位置**（0.840→**0.777**）；**两阶段分工**（四模型 edge F1 0.814–0.860）。
- **证据不足/仅声称**：sequential/parallel **分类器未被评估**；**未报告人类标注者间一致性**，0.860 是"模型 vs 裁定后图"而非"人类共识"；与 LLM judge 的对比**不是同条件对照**；AUC 为 **within-agent**；验证集仅 **100 条**。

---

## 4. 具体技术细节

### 4.1 模型结构

- **不训练任何模型**：系统 = 确定性解析器 + LLM 归因器 + 图分析代码。
- 归因模型默认 **GPT-5.2**（敏感性覆盖 GLM-5.2、Gemini-3.5-Flash、Claude-4.5-Haiku）；对比基线为 GPT-5.2 judge。
- 被分析的五个搜索 agent：**WebSailor**（32B，agentic RL/DUPO 后训练）、**MiroThinker**（30B，256K 内维持数百次 tool call）、**Tongyi DeepResearch（TYDP）**（30B MoE，mid-training + GRPO）、**TYDP-GPT5**、**TYDP-Qwen3**；后两者共用 scaffold 只换 backbone，用于**分离 harness 与模型贡献**。

### 4.2 训练流程（若需要训练）

**本篇不训练任何模型权重。** 下表说明其"数据/模型如何构造或调用"。

| 阶段 | 目标 | 构造/调用 | 数据来源 | 数据形态 | 判定方式 |
|---|---|---|---|---|---|
| ① 轨迹采集 | 得到待分析样本 | 五个 agent 配置 | BrowseComp（150 题）、WebWalker-Hard-English（70 题）、DeepSearchQA（50 题） | **1,350 条轨迹**（5×270），含 thoughts、tool calls、检索结果、访问页面、答案 | 不训练；轨迹是被分析对象 |
| ② 确定性解析 | 抽出可复现的图部分 | Python 程序 | ①的原始轨迹 | 查询节点 + 检索结果 + 失败标记 + `q₀→qᵢ` 边 | 无 loss；规则可复现 |
| ③ LLM 归因 | 解析需语义解释的依赖 | 归因 LLM（默认 GPT-5.2） | 原问题 + 当前推理块 + 早期查询及其证据；以及答案 A | 逐查询 evidence-use 边 + 剪枝；A 逐单元归因（无支撑者连 PK） | 无 loss；prompt 式判定 |
| ④ 人工标注验证 | 建立 canonical 参考图 | 双人独立标注 + 五人审计 | **100 条**真实轨迹（非 gold answer） | HTML 工具逐 query 审阅 | macro edge F1（**0.860**） |

### 4.3 推理流程（若不训练或重点在推理）

这是**离线的一次性解析流水线，不是交互式 agent 循环**：不 ReAct、不规划，而是对已跑完的轨迹做后验归因——**SearchAtlas 不是 agent，而是 agent 的审计工具**。输出为 DAG + 图统计 + 三个诊断指标 + 综合分数。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| BrowseComp | Web（深度搜索） | **sequential-constraint** 深度检索问答 | 150 题 | 问题 + agent 轨迹日志 | 短答案 + 证据依赖 DAG | edge F1 / macro AUC / F1 |
| WebWalker-Hard-English | Web | **parallel-constraint** 检索问答 | 70 题 | 同上 | 同上 | 同上 |
| DeepSearchQA | Web（压力测试） | 两种 regime 各 25 题（共 50） | 50 题 | 同上 | 同上 | 同上 |

> 分析规模：**5 agents × 270 题 = 1,350 条轨迹**；另有 **100 条 canonical DAG** 作人工标注验证集（跨四模型 macro edge F1 0.814–0.860）。

### 5.2 实验结果分析

**① 流水线质量**：对 100 条人工标注 DAG，**macro edge F1 = 0.860**（摘要口径 86.0%），三 benchmark 相近。换归因模型：GPT-5.2 **0.860**、Claude-4.5-Haiku **0.842**、Gemini-3.5-Flash **0.814**；全量重跑时 GLM-5.2 与 Gemini-3.5-Flash 达 **0.866 / 0.870**。

**② 搜索行为差异（Table 1）**：MiroThinker 搜索最广（BrowseComp 节点 **103**、深度 **26**），而 **TYDP-Qwen3（接受最少搜索训练）三 benchmark 一致最短**（仅 6 节点）；**depth 是 agent 相对稳定的签名，width 更受问题类型影响**。

**③ 诊断与正确性的关联**：仅拓扑已是强排序信号（0.714–0.771），加 grounding 后每个 benchmark 都提升（最高 **+0.106**），再加 PK 依赖达 **0.840–0.856** macro AUC——**grounding 增益最大**；留出阈值 sequential **0.705**、parallel **0.780**。

**④ 与非 DAG 基线对比（Table 3，pooled over 1,350 轨迹）**：Ordered-query-list judge 0.677、Full-trajectory judge 0.738、**SearchAtlas 0.776**。对照"只用原始日志统计量"的分类器，SearchAtlas **大幅领先**——搜索统计量不能替代"证据如何被组织与使用"的建模。

**⑤ 局部化消融（Table 4）**：BrowseComp 上 trajectory-wide 0.459 vs DAG-localized **0.764**；WebWalker-Hard 上 0.568 vs **0.680**。BrowseComp 上 trajectory-wide **不比随机好**——**约束被提到过几乎不携带信号，重要的是它是否贡献到答案**。

**⑥ 分歧分析（Table 5）**：共审计 **100 例**（**68 例高分错误 / 32 例低分正确**），典型分歧包括错误目标上的过早绑定、漏掉正确证据、过度搜索的成功、捷径成功（靠先验知识）。结论：**结构组织是成功推理的必要条件而非充分条件**。

---

## 6. 亮点与贡献（Why it matters）

1. **把"过程可解释性"做成可复算的结构对象**：四类边对应四类可观察证据关系，过程诊断从形容词变成数字。
2. **edge F1 86% 证明自动构建可行**：0.814–0.860 的跨模型稳定区间说明两阶段分工可规模化。
3. **"约束落地的 backbone 局部化"是最有价值的方法论贡献**："某约束被提到过"几乎无信号（0.459，不如随机），"贡献到答案"才有信号（0.764）。
4. **divergence 分析比主结果更有洞察力**：高分错误与低分成功并存，说明**过程诊断不能替代答案正确性**。

---

## 7. 局限与可改进点（个人点评）

1. **sequential/parallel 硬路由是最脆弱环节**：所有下游指标都依赖"问题已被正确路由"，但论文既没给分类器、也没给准确率或人类一致性，而真实问题常是混合约束。**这是最该优先补的实验。**
2. **"稳定"不等于"正确"**：四模型一致只排除模型间随机差异，不排除共享的系统性偏见（作者自承）；**未报告人类标注者间原始一致性**，故 0.860 是"模型 vs 裁定后图"而非"人类共识"。
3. **与 LLM judge 的对比不是同条件对照**：judge 基线只拿到原始日志或有序 query 列表，未被给予事实单元拆分、四类边定义或任何 scaffold，故"更准"混合了"结构化表示的价值"与"额外分析步骤"。
4. **只在 closed-answer、英文、纯搜索工具设定下验证**：开放式、非英语、多模态未测，**无法直接用于报告型 deep research 的过程评估**——而那恰是搜索 agent 最主流的商业形态。
5. **构建成本限制在线使用**：全量重建约 $1,270–$2,540，作离线审计可接受，作在线 rollout 打分器（如 RL 过程 reward）太贵。

---

## 8. 对我们的启示 / 可借鉴点

- **评测要从"结果"扩展到"证据结构"**：过程信号与结果正确性**互补**（68 例高分错误 + 32 例低分成功），只看成功率会漏掉这两类样本。

### 对 GUI Agent 的可借鉴点

把 DAG 换成 GUI 版：节点 q₀→原任务指令、qᵢ→第 i 个 step（观察+动作）、PK→未经观察验证的记忆、A→最终提交状态；四类边对应 constraint-use（针对哪个显式要求）、evidence-use（界面状态支撑后续哪步决策）、failure-response（动作失败后是否响应）、prior-knowledge（未观察就采取的动作）。它直接暴露 GUI agent 三类最常见过程失败：**约束未落地、盲目重试、不看页面就操作**。

1. **约束必须落在 answer-reaching path 上**：GUI 约束很具体（目标元素、目标状态、权限边界如"只删除 X 不动 Y"），**在某个被放弃的分支里提到过"不要删除 Y"并不等于遵守了约束**。
2. **failure→retry 边价值更高**：点击无响应、渲染延迟、元素位移、弹窗遮挡使重试成为常态，显式建模可量化"重试是否带来新信息"，从而区分"有策略的恢复"与"死循环"。
3. **PK reliance 可直接作惩罚项**：统计**直接支撑最终提交的动作**中有多少基于未经当前观察验证的信息（按记忆点坐标、直接构造 URL），比"总步数"精确。
4. **顺序 vs 并行约束可映射为 GUI 任务类型**：sequential（"先登录再搜索再导出"）应期望聚焦的动作-证据链，parallel（"确认三个字段都正确"）应期望扁平支撑——**用同一套指标评价两类任务会得出错误结论**：**先按任务结构条件化，再做过程诊断**。

---

## 9. 延伸阅读

- **最接近的前作**：WebGraphEval（Qian et al. 2025）——同样处理 web agent 轨迹，但把多次运行聚合成 consensus graph。
- **事后图恢复**：ReasoningFlow（Lee et al. 2025）；Graph of Verification（AAAI 2026）；DAG-Math（Dziri et al. 2023）。
- **规范性图表示（对照）**：Tree-of-Thoughts；Graph-of-Thoughts（AAAI 2024）；GPTSwarm；S-DAG（AAAI 2026）。
- **过程评测与过程奖励**：AgentProcessBench（arXiv:2603.14465）；Agent-RRM（arXiv:2601.22154）；PPR（Xu et al. 2025）；RE-TRAC（Zhu et al. 2026）；DeSA（Wang et al. 2025）。
- **评测基准**：BrowseComp（Wei et al. 2025）；WebWalker-Hard-English（Wu et al. 2025）；DeepSearchQA（Gupta et al. 2026）。

---
*解读生成时间：2026-09-12 ｜ 解读人：WorkBuddy（AI）*
