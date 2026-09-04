# Discriminative World Models for Web Agents

> 判别式状态匹配：训练 Web 世界模型区分动作后果，而非复述固定格式。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | Discriminative World Models for Web Agents |
| **arXiv ID / DOI** | arXiv:2609.02885v1（abs 页 <https://arxiv.org/abs/2609.02885>） |
| **arXiv 链接** | <https://arxiv.org/abs/2609.02885> |
| **发表出处（Venue）** | arXiv preprint（comments 字段为空，PDF 中无会议/投稿线索） |
| **发布时间** | 2026-09-02 |
| **作者** | Kelvin Li\*, Dhruv Pendharkar\*, Anish Pahilajani, Chuyi Shang, Leon Oks, Leonid Karlinsky, Rogerio Feris, Trevor Darrell, Roei Herzig（\* 共同一作） |
| **所属机构** | University of California, Berkeley（美国）；MIT-IBM Watson AI Lab（美国）；Cal Poly San Luis Obispo（美国）；Xero |
| **开源情况** | ⚠️ 部分开源：正文仅声明项目主页 <https://dhruvpendharkar.github.io/dwm/>；第 7 节伦理声明提到将按原工件许可条款发布衍生数据/代码/模型或处理脚本，但未给出具体仓库链接 |
| **类型标签** | `Planning` `RL` `Web` `Benchmark` |
| **训练方法标签** | RL (GRPO)（世界模型判别式目标 + 分组相对策略优化，Qwen3-8B 底座、每组采样 8 条） |
| **关键词** | Web agents; World models; Predicted-state matching; Process reward models (PRM); Test-time action selection; Discriminative representation |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-02_Discriminative_World_Models_for_Web_Agents.pdf |

---

## 1. 研究背景与要解决的问题

大语言模型让"智能体通过浏览器完成网页任务"成为可能，但网页导航是**部分可观测的多步决策问题**：每一步点哪、填哪，既要看当前页面，也要想这一步对后续的影响。多数既有 agent 走"反应式 next-action 预测"路线——直接由当前观测生成下一个动作，不显式比较同一状态下各候选动作的优劣。近年兴起的**测试时动作选择**则先让策略提出多个候选动作，再由打分器（ranker）或过程奖励模型（PRM）选一个执行；**基于模型的规划**再进一步，用世界模型预测每个候选动作执行后的页面状态，让打分器在"预期后果"间比较。

这带来一个基础问题：**预测出的下一状态该表示成什么**？现有世界模型普遍用**监督式 next-state prediction** 训练，逼模型生成固定格式——文本摘要（如 WebDreamer）或完整 HTML/AXTree 快照（如 WebWorld）。作者认为该目标与下游排序**错配**：摘要方法可能恰好漏掉区分竞争动作的那处变化，完整结构化表示里相关变化又淹没在大量未变结构中（Figure 1 的 Reddit 决策点即为例证）。ranker 真正需要的是**预测表示在候选动作之间有区分度**。本文提出的 `predicted-state matching` 正为此而来。

## 2. 核心方法 / 思路

方法分三块：分支数据集、判别式训练目标、下游接入。

**（1）分支决策点数据集。** 原数据 Go-Browse 是 WebArena 中的线性探索轨迹，看不到"同一状态走其它动作会怎样"。作者把各轨迹中**重复出现的状态合并成状态-动作图**：节点为状态，边为该状态上被真实执行过的动作及其观测到的结果。共享状态带多条出边，便形成局部决策点 Dₜ = {(aᵢ, sᵢ₊₁)}，再展开成两两样本：给定"被查询动作、其真实结果状态、另一动作的备选结果状态"，模型要能分辨。最终得到 7,730 个决策点（源自 2,839 条轨迹）、30,920 个两两样本，按 WebArena 域分层切分。

**（2）predicted-state matching 训练目标。** 输入任务、历史、当前状态与被查询动作，世界模型（Qwen3-8B 底座）生成一段自由文本的下一状态表示 ẑ——不绑定任何格式，由模型自适应详略。关键在**匹配裁判**：训练期用固定的 Qwen3-32B，它只收到 `(ẑ, 真实结果, 备选结果)` 且顺序随机，要选出与预测最匹配者；裁判**看不到任务、历史、当前状态与动作**，表示必须自带足够信息。奖励 R = R_match + λ_fmt·R_fmt（格式奖励鼓励合法 `<predicted_state>` 非空输出，λ_fmt=0.4），用 **GRPO** 优化（每组采样 8 条）。该目标对表示格式完全无关（representation-agnostic）：真值状态只是匹配候选项，模型学的不是复述目标串，而是**把被查询动作引向的状态与其它动作引向的状态分开**。

**（3）接入动作排序。** 推理时世界模型为每个候选动作各生成表示 ẑᵢ，将 `(动作, ẑᵢ)` 拼进上下文交给 PRM/ranker 判偏好。三组对照由此拉开：纯动作、WebWorld-8B 固定格式预测、本文表示——打分器骨架、数据、监督方式全锁定，只换"下一状态信息来源"。

## 3. 关键实验结果

三个层级的评测（模型均冻结、greedy 解码）：

**（1）留出集 predicted-state matching 准确率**（主裁判 Qwen3-32B，覆盖 Shopping/CMS/Reddit/GitLab/Map 五域）。本文总体 **80.80%**，超过 WebDreamer-7B（74.51%）与 WebWorld-8B（70.17%），更远超 GPT-4o 直接生成（49.40%）。最关键的是对**数据匹配的监督式 SFT 基线**（同用 Qwen3-8B、同喂分支数据，但目标改为生成完整 AXTree）——其仅 **47.77%**，差距约 33 个点，证明增益来自训练目标而非数据。换裁判验证：本文在 GPT-4o、Llama-3.1-70B-Instruct 下分别得 81.26% 与 79.31%，仍领先，说明表示未过拟合训练裁判。附录还显示本文预测平均 **91.6 token/状态**，仅 WebWorld（412.7 token）的约四分之一——更短却更准。

**（2）WebPRMBench 动作排序。** 训练打分器（Qwen2.5-7B、answer-only SFT，受控对照末三行）：平均 Best-of-N 从无状态 55.80%（Pairwise 82.02）→ 接入 WebWorld-8B 状态 67.63% → 本文 **72.70%**（Pairwise 89.36）；平均 Pairwise 略超 WebArbiter-7B 的 89.19%，BoN 逼近其 74.60%，并高于 GPT-4o 等 LLM-as-judge 与 WebShepherd-8B（BoN 43.28）。冻结打分器（零训练）下同样成立：Qwen2.5-7B 的 BoN 从 42.78% → +WebWorld 51.53% → 本文 **54.65%**；3B 版由 26.76% 提到 42.96%。

**（3）WebArena-Lite 端到端成功率**（policy 为 GPT-4o）。ReAct 式单动作选择 **13.94%**（与 WebRL 的 GPT-4o 13.9% 吻合）；Best-of-5 采样排序到 **21.82%**；加本文世界模型状态后达 **28.48%**。

训练开销上：本文仅 30,920 样本、390 A100-GPU 时，而 WebDreamer 用 310 万+合成交互、WebWorld 用 106 万条轨迹与 1,568 GPU 时。结论：目标对齐比单纯扩大监督式 next-state prediction 规模更有效。

## 4. 亮点与贡献（Why it matters）

1. **重定义"好世界模型"的评价标准**：以"能否帮下游区分竞争动作"取代"能否复现既定格式"，训练与使用目标对齐，概念干净普适。
2. **配套的分支数据构造法**：把现成线性轨迹按状态复用重组为 state-action 图，零额外标注即可产生"动作后果对比"监督并支撑留出评测。
3. **三层次验证闭环**：表示质量（matching）→ 效用（ranking，训练/冻结两类打分器）→ 端到端收益，且受控对照把"目标"与"数据/骨架"的贡献拆得干净。
4. **表示简洁高效**：约 WebWorld 1/4 输出长度拿更高分数，对 token 预算敏感的落地场景友好。
5. **跨裁判、跨打分器泛化**：表明学到的是状态本身的判别信息，而非对某 judge 的投机拟合。

## 5. 局限与可改进点（个人点评）

- **评测域偏单源**：分支数据全部出自 WebArena 系（Go-Browse 轨迹），端到端只在 WebArena-Lite、policy 仅 GPT-4o；真实站点多样性、视觉依赖页面、桌面/移动 GUI 均未覆盖。
- **匹配正确性仍是"模型代理"**：judge 是 LLM，其偏好（爱详细或爱概括）会直接变成训练偏置；作者验证了三家裁判一致性，但缺人工判定对标，也未讨论 judge 被高置信措辞欺骗的可能。
- **分支覆盖不完整**：备选动作只来自其它轨迹真实执行过的动作，"可能但没人试过"的动作与无轨迹交汇的状态都不存在，数据是"观察到的分支"而非"全空间树"。
- **RL 消融偏薄**：仅给格式权重 λ_fmt（0.2–1.0 间 78.9%–80.8%），缺 GRPO 对直接 SFT/DPO 的对比，也未评测候选数大于 2 的情形。
- **可复现性受限**：只有项目主页，无代码/数据/权重链接，分支数据复现依赖未放出的脚本。

## 6. 对我们的启示 / 可借鉴点

- **"预测要为决策服务"是通用原则**：网页乃至 GUI/桌面 agent 的表示学习都应锚定下游任务（排序、打分、规划），而非还原观测格式。可直接指导我们 RL/Grounding 管线的 reward/critic 设计——问"该信号能否让两个候选动作可区分"比"它像不像真值"更有价值。
- **LLM-as-judge 奖励 + GRPO 对齐小模型**是低成本配方：Qwen3-8B 规模、390 GPU 时即可训练能预测动作后果的模块，判别式对比监督的数据效率远超逐 token 生成式自监督，对数据稀缺的垂直 GUI 场景尤其值得借鉴。
- **表示是"免重训"的通用增强接口**：冻结 ranker 仅靠拼接预测状态就涨点（Qwen2.5-7B BoN 42.78%→54.65%），意味着世界模型与 PRM 可解耦升级；我们做 agent 时可预留"动作后果预测"作为向后兼容的可选输入。
- **数据工程技巧可直接搬运**：多条轨迹按状态去重合并成图、从共享状态长出分支，是零标注制造对比监督的通用方法，可用于我们自有的交互日志。
- 同时需前置规避 judge 偏差传导与单一环境评测风险（多 judge 交叉 + 抽样人工复核）。

## 7. 延伸阅读

- **WebDreamer**（Gu et al., arXiv:2411.06559）：文本摘要式世界模型 + 模型基规划，本文"监督式 next-state prediction"的主要反例。
- **WebWorld**（Xiao et al., arXiv:2602.14721）：预测完整 AXTree 的大规模结构化世界模型，本文最强的固定格式基线。
- **WebArbiter**（Zhang et al., arXiv:2601.21872）：原则引导推理的 WebPRM，提供 WebPRMBench/Collection 及排序评测脚手架。
- **Web-Shepherd**（Chae et al., arXiv:2505.15277）：清单引导式 WebPRM，与 WebArbiter 同属"动作排序"工作线。
- 分支数据源头可再追 **Go-Browse**（Gandhi & Neubig, arXiv:2506.03533）。

---
*解读生成时间：2026-09-04 09:06 ｜ 解读人：WorkBuddy（AI）*
