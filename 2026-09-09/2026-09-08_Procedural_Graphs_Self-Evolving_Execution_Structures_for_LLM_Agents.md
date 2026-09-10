# Procedural Graphs: Self-Evolving Execution Structures for LLM Agents

> 一句话 TL;DR：把任务"该做什么"的程序性知识外置成 (流程,关系,流程) 有向属性图，在线按进度取 2 跳子图生成情境化引导，离线用验证门控+拒绝记忆自演化图，全程不改模型权重。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Procedural Graphs: Self-Evolving Execution Structures for LLM Agents |
| **作者 / 机构** | Yuxing Lu（Google，另挂 Georgia Tech 与 Peking University）；Yicheng Chen、Shanchan Wu、Sercan Ö. Arık（通讯作者 soarik@google.com）；Google（美国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-08；arXiv preprint（cs.AI），arXiv:2609.09153v1 |
| **arXiv 链接** | https://arxiv.org/abs/2609.09153 |
| **代码仓库** | ❌ 未开源（文中未声明） |
| **数据集地址** | 未公开（实验基于公开 benchmark：HotpotQA / MultiChallenge / GDPval / ALFWorld / τ-bench / BFCL v3 / EnterpriseArena） |
| **类型标签（论文类别）** | `Planning` `Online` `General` |
| **训练方法标签** | `—（方法框架，无模型权重更新）`：推理期图引导 + 离线 LLM 驱动的图自演化与验证门控 |
| **关键词** | Procedural Graph；self-evolution；procedural memory；long-horizon planning；LLM agents；graph guidance |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_Procedural_Graphs_Self-Evolving_Execution_Structures_for_LLM_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：长程 LLM Agent 的"程序性知识"（何时做什么、按什么顺序、满足什么条件才能做）如何被**结构化表示、随当前进度取用、并能从执行反馈中自我改进**——属于通用 Agent 规划场景，而非纯 GUI。
- **为什么重要**：主流 ReAct 式自由生成把程序一致性完全压在"变长的动作—观测历史"上，轨迹一长就出现目标漂移、乱序调用工具、重复无用动作等失败模式。不解决，Agent 的长程可靠性与 token 效率都上不去。
- **现有方法有什么不足**：① 文本记忆/反思类（Reflexion、ExpeL、MemoryBank、RAP）只存经验文本，solver 仍需自己重建"怎么用、前后怎么约束"；② 状态条件规则（AutoGuide）只给针对性提示，**不把相邻步骤显式连起来**；③ 工作流/状态机/流程图（FlowBench、KnowAgent、AWM）虽显式化步骤，但**大多依赖人工设计，且不随经验自动修订**（AFlow 虽能离线搜工作流，仍非随进度取用的引导）。
- **Research Gap**：作者认为缺一种**"既显式可编辑、又能按当前进度取用、还能从执行反馈自改进"三者兼具**的程序知识表示——于是提出 Procedural Graph（PG，程序图）。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

知识图谱用 (实体,关系,实体) 回答"是什么"，PG 用 (流程,关系,流程) 回答"下一步做什么"：把程序知识建成带属性边的有向图，在线"定位→取子图→生成引导"，离线"诊断→变异→门控→记拒绝"自演化，权重始终不动。

### 3.2 方法总览（Pipeline）

- **输入/输出**：输入 = 用户 query + 交错的动作—观测历史 T；输出 = solver 的下一步动作 aₜ。
- **数据结构**：有向带属性图 G=(V,R,E,Φ)，E⊆V×R×V。节点抽象工具动作/技能/推理步骤/任务状态；边 (u,r,v) 声明"u 之后在关系 r 下可走到 v"；关系词表仅 4 种（LEADS_TO / TRIGGERS / PROVIDES_INPUT_FOR / CONVERGES_TO）；每条边挂 3 个文本属性——**condition**（何时可用）、**guidance**（怎么做）、**pitfalls**（避什么坑）。
- **在线推理（图冻结）三操作**：① **locate**——用最近一次动作精确匹配当前活动节点 uₜ（匹配失败退回整图）；② **extract**——取以 uₜ 为中心向外 h=2 跳的局部子图（连邻接条件一起取，避免"只看到 submit、看不到前置 check"的上下文割裂）；③ **generate**——引导模型 Ψ 把子图 + 最近 w=3 步轨迹翻译成一句"情境化引导" gₜ，追加进 solver prompt。引导"只指方向不代做决定"，是软约束而非硬状态机。
- **离线自演化（每批训练任务后改图，不动权重）四步闭环**：① **诊断 rollout**——用当前图跑一批训练任务，记录轨迹与分数；② **反馈驱动变异**——LLM refiner 对比高/低分轨迹，产出结构化编辑集 ΔG（Add 节点/边、Delete 节点/边；改属性 = 删边重加），严格输出 JSON；③ **验证门控**——结构合法（端点存在、可达终止、遵循环策略）的候选图在**独立验证集**上重跑，仅当 S_val(candidate) ≥ S_val(当前图) 才提交，否则回滚；④ **拒绝记忆**——被拒候选连同轨迹与原因写入 H_rejected，下轮喂给 refiner 当负例防重复提案。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：把程序知识做成**带类型关系 + 文本属性边**的三元组图，并用"当前进度定位 + 局部拓扑取用 + 引导生成"来消费它；以及"验证门控 + 拒绝记忆 + 结构校验"组合出的自演化安全阀——这两点是同类自改进 Agent 里较完整的。
- **工程组合**：ReAct solver、LLM refiner、图检索、验证集 rollouts 都是现成组件，本身不新；创新在把它们接成一个闭环。
- **对性能提升最关键的设计**：ablation 证明**"局部子图 + 生成式引导"**贡献最大（见 5.2 效率消融）；**验证门控**则直接决定自演化是否收敛而非发散。
- **证据不足 / 仅声称有效**：跨 solver/工具接口的图迁移性仅在结论里作为 future work，无实验；引导文本是否隐性过拟合到训练所用的特定 LLM 措辞，未验证。

---

## 4. 具体技术细节

### 4.1 模型结构

- 纯 **LLM**（无视觉编码器），全程 **不微调、不冻结/更新任何权重**。
- 用 4 个异构底模：Claude Sonnet 4.6、Gemini 3.1 Pro、Gemini 3.5 Flash、Grok 4.1 Fast。**引导模型 Ψ 与离线 refiner 与 solver 永远同源**（同一 LLM）。
- 所有调用 greedy decoding（temperature=0）保证可复现。

### 4.2 训练流程（本文无权重训练，以下为"图优化"阶段）

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 目标 / 判定 |
|---|---|---|---|---|---|
| Stage 1 诊断 rollout | 收集执行反馈 | 暴露失败回路与成功捷径 | 当前图 G_{k-1} 跑训练批 B_k | (q, 轨迹 T, 分数 S∈[0,1]) | 无 loss，直接记录 |
| Stage 2 变异 | 提出图编辑 | refiner 总结"错在哪/怎么改" | 高/低分轨迹对比 + 拒绝记忆 H_rejected | 结构化 JSON 编辑集 ΔG（Add/Delete/改属性） | 提示驱动，无梯度 |
| Stage 3 验证门控 | 防过拟合编辑 | 泛化性能筛选 | 独立验证集 Dval | rollout 打分 | S_val(cand) ≥ S_val(当前) 才 commit |
| Stage 4 拒绝记忆 | 防重复提案 | 负例累积 | 被拒候选 + 轨迹 + 验证结果 | 超长则保留末尾 L_max tokens | 下轮注入 refiner |

### 4.3 推理流程

- 多步循环（沿用 ReAct）。solver 每步在 prompt = query + 历史 T + 引导 gₜ 条件下采样 aₜ。
- 引导生成：uₜ = Match(a_{t-1}, V)（首步固定 u₁=Start）；匹配成功取 N_h(uₜ)（h=2），失败退整图；gₜ = Ψ(Gₜ, q, T_{t-w:t})（w=3）。
- 终止条件由各 benchmark 环境定义；引导是追加文本，不改变 action 空间。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | 场景 | 任务类型 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|
| HotpotQA | General（搜索工具） | 多跳问答 | 纯文本+搜索工具 | 答案 | Acc / Ans EM / Ans F1 |
| MultiChallenge | General（多轮对话） | 指令保持 | 多轮文本 | 对话动作 | 记忆保持/可靠编辑/连贯/综合成功率 |
| GDPval | General（专业长文档） | 开放专业任务 | 长文本 | 结构化输出 | Rubric Score（专家评分） |
| ALFWorld | General（具身） | 家电任务（严格顺序） | 文本世界状态 | 动作序列 | Success |
| τ-bench | General（工具策略合规） | 实时用户交互工具使用 | 文本交互 | 工具调用 | Pass@1 |
| BFCL v3 | General（函数调用） | 多轮函数调用 | 文本 | 函数调用 | Acc |
| EnterpriseArena | General（金融长程决策） | 最长 132 个月 CFO 决策 | 月度状态（含 3 次未公开宏观危机） | 月度决策 | 生存率/平均寿命/平均企业分/融资额 |

### 5.2 实验结果分析

- **主结果（6 benchmark × 4 LLM，统一 ReAct solver，7 基线：Vanilla ReAct / MemoryBank / RAP / ExpeL / AutoGuide / AWM / KnowAgent）**：PG 在 24 个模型×基准设置中 **21 个第一或并列第一**；对最强基线 **19 胜 2 平 3 负**（符号检验 p=4.3×10⁻⁴）。最大领先：BFCL v3 × Gemini 3.5 Flash（67.00% vs 58.00%，**+9.00**）、GDPval × Gemini 3.1 Pro（78.78 vs 71.37，+7.41）、τ-bench × Gemini 3.1 Pro（80.00% vs 73.04%，+6.96）。**HotpotQA 增益最弱**（相对最强基线 −0.90 ~ +1.30 点），说明收益集中在结构性强的长程任务。
- **长程韧性（EnterpriseArena）**：存活率 Claude Sonnet 4.6 44.0%→58.0%、Gemini 3.1 Pro 6.0%→34.0%、Grok 4.1 Fast 26.0%→40.0%（平均企业分最优 $39.62M）；Flash 全配置 0% 存活，但平均寿命 33.58→40.62 个月、平均融资 $0→$9.39M（Grok PG 达 $30.11M）。核心行为是**提前融资**——资本 1~6 个月后才到账，引导让 agent 在平稳月就发起申请。
- **Ablation——构造策略（5 模式，HotpotQA/MultiChallenge，Flash）**：Mode 5（从零骨架 + 在线演化）在 HotpotQA 达 **78.79 Ans F1 / 66.30 EM**（较无图 +7.58/+7.50 点）；MultiChallenge 上 Mode 1 专家图反而 87.50%→58.93%、Mode 2 静态更新更差 53.57%，而 Mode 3（专家图+在线演化）回收到 **92.86%**（较专家初始 +33.93 点）——证明循环能**修复劣质先验**。
- **Ablation——自演化 10 轮（EnterpriseArena，Flash）**：验证集存活率 0%→45%（Round 1 发现"先查现金/预测 runway 再融资"主链）→80%（Round 2 加 recall_notes，工具调用 17.23→3.08 次/月）→90%（Round 8）；最终返回图测试存活率 **85% vs 基线 0%**（Fisher p=2.6×10⁻⁸，最佳单轮曾达 95% 但作者拒绝挑选）。
- **Ablation——效率（Flash）**："局部子图 + 生成式引导"在 MultiChallenge/GDPval/ALFWorld 全部最优（**89.31 / 63.99 / 81.53**），比各自次优 +2.0/+6.8/+9.0 点；相对整图引导 token 降 **70.9% / 18.1% / 14.8%**；求解步数 GDPval 28.20→18.57。代价是比无图基线总 token 仍高 **33.4% / 55.4%**（引导本身多一次 LLM 调用）。

---

## 6. 亮点与贡献（Why it matters）

- **外置、可编辑、可审计的程序知识表示**：把"何时做什么/怎么做/避什么坑"结构化放模型外，天然支持检查与迭代，无需重训即可修正流程。
- **"定位→局部取用→生成引导"消费方式**：解决零散规则检索丢上下文的问题，兼顾结构化约束与推理自由度（"引导但不独裁"），且显著省 token。
- **带安全阀的自演化闭环**：验证门控 + 拒绝记忆 + 结构校验构成防过拟合、防退化、防重复提案的完整机制，比多数自改进工作更严谨（尤其"拒绝在测试集上挑选最优轮"的纪律）。
- **跨 7 基准、4 个异构模型族一致受益**：从多跳 QA 到 132 个月 CFO 决策，收益稳定，说明方法与底模解耦。

---

## 7. 局限与可改进点（个人点评）

- **恒定 token/延迟税**：每步多一次 LLM 调用，即使局部化后总 token 仍比无图高 33%+；作者自己承认应跨步复用或选择性生成，在线产品里是现实瓶颈。
- **验证门控成本与小样本噪声**：每个候选都要独立验证集重跑；EnterpriseArena 每折仅 20 集，作者也承认单轮接受/拒绝常取决于一两集，应视为搜索轨迹而非显著性结论。
- **全提示驱动，编辑质量受 refiner 天花板限制**：refiner 幻觉会直接产生错误 JSON/错误边；拒绝记忆无限增长会稀释有效负例，未见压缩/剪枝策略。
- **迁移性未验证**：跨 solver/工具接口复用只在 future work；引导文本可能隐性过拟合到特定 LLM 措辞。
- **与 GUI 场景有距离**：全部为文本式工具/指令环境，无视觉、无截图、无 grounding，观测噪声问题未涉及。
- **收益不稳定**：HotpotQA 增益近噪声水平，个别格子（ALFWorld/Grok）PG 非最优，说明优势集中在强结构长程任务。

---

## 8. 对我们的启示 / 可借鉴点

- **"外置可编辑的结构先验"契合 GUI 长任务痛点**：GUI agent 的乱序操作、重复点击、跳过前置校验正是 PG 想治的病；把流程常识从每轮 prompt 隐式遵守中解放出来，是低成本高回报的落地形态。
- **软引导而非硬状态机**：GUI 页面状态多变，硬状态机易碎；PG 的"软引导 + solver 自由决策"是更稳的路线，可与 RL 并存先行落地。
- **自演化 + 验证门控 = 可复现的经验沉淀机制**：历史轨迹自举技能最怕"越改越差"，其三步保险（验证集不降分才提交 / 结构合法校验 / 被拒方案当负例喂回）可直接抄作防回归设计。
- **局部检索优于全库注入**：按"当前进度节点"取 2 跳子图而非全量注入，既降 token 又保步骤依赖，值得在 GUI 观测序列上复刻。
- **成本结构要有心理预期**：额外 LLM 调用与 token 增长是实打实的，落地需权衡"每步引导"vs"仅易错状态引导"或换更小引导模型。

**对 GUI Agent 的可借鉴点（本篇为 Agent 方法论域）：**

GUI Agent 每一步本质可抽象成图节点——open_app、login、navigate、fill_form、click、submit、verify、wait_for_element——页面级前置条件（按钮可点前需登录态、校验通过前不提交）正好编码成带 condition/pitfalls 的边。照搬 PG 可做三步：① 把某类 GUI 任务（订票、企业软件操作）归纳成程序图；② 在线执行时用"上一步已完成的 UI 操作"定位当前节点，只把其后 1~2 跳子图及注意事项翻译成一句引导塞进 prompt，即可不改权重、不动策略地压低乱序/漏步/重复点击；③ 用失败轨迹（报错弹窗、元素超时、提交被拒）与成功轨迹对比，让 refiner 自动增删节点/改写引导，再用小规模验证集门控防技能漂移。由于图与权重解耦，同一张"任务程序图"可复用于不同 GUI 基座模型，免重训。进阶可与 Online RL 结合：先由图保证流程合法性，再用 RL 优化点击目标/输入内容等交互细节，并让 RL 学到的成功模式反向驱动图更新。落地注意：GUI 的截图解析与元素定位偏差会抬高"活动节点匹配"失败率，需给匹配失败准备可靠的全图回退与重定位策略。

---

## 9. 延伸阅读

- KnowAgent（Zhu et al., 2025）：文本式动作转移规则先验，同方向最早一批。
- AutoGuide（Fu et al., 2024）：状态条件规则生成与选择，与 PG 最接近的基线。
- AWM（Wang et al., 2025b）与 FlowBench（Xiao et al., 2024）：线性工作流/流程图式过程先验。
- AFlow（Zhang et al., 2025）：用 MCTS 自动搜索工作流图。
- ExpeL（Zhao et al., 2024）、Reflexion（Shinn et al., 2023）、MemoryBank（Zhong et al., 2024）、RAP（Kagaya et al., 2024）：对比式经验提炼与文本记忆家族。
- MemP（Fang et al., 2025）、ToolNet（Liu et al., 2024a）、SkillGraph（Nie et al., 2026）：程序记忆生命周期、工具转移图、多智能体技能图。
- SE-Agent、SkillWeaver、Voyager：计算机/沙盒场景的技能自改进代表，可对照"经验外置"路线在 GUI 的可行性。

---
*解读生成时间：2026-09-09 ｜ 解读人：WorkBuddy（AI）*
