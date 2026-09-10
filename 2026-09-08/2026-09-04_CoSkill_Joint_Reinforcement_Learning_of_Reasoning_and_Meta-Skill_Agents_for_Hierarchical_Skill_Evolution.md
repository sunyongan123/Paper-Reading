# CoSkill: Joint Reinforcement Learning of Reasoning and Meta-Skill Agents for Hierarchical Skill Evolution

> 一句话 TL;DR：把静态 meta-skill workflow 重铸为可学习的 Meta-Skill Agent，与推理智能体共享 backbone 联合 RL 训练，实现分层技能的端到端共适应进化。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | CoSkill: Joint Reinforcement Learning of Reasoning and Meta-Skill Agents for Hierarchical Skill Evolution |
| **作者 / 机构** | Jinyuan Feng、Dongmin Li、Huimu Wang、Zhiqiang Pu（中国科学院自动化研究所）；Yiqun Chen（中国人民大学）；Yang Gao、Xing Chen（字节跳动） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-04；arXiv preprint（arXiv:2609.04865v1 [cs.AI]） |
| **arXiv 链接** | https://arxiv.org/abs/2609.04865 |
| **代码仓库** | ✅ GitHub：https://github.com/jinyuan-cookie/CoSkill（基于 verl 与 verl-agent） |
| **数据集地址** | 未单独提供下载链接；使用公开 benchmark——ALFWorld（https://github.com/alfworld/alfworld）、WebShop（https://github.com/princeton-nlp/webshop） |
| **类型标签（论文类别）** | `RL` `Planning` `General` |
| **训练方法标签** | `RL (GiGPO)`；双角色共享单 backbone 的 online 多智能体联合训练，含 post-edit verification reward |
| **关键词** | 分层技能库；Meta-Skill Agent；技能-策略共适应；GiGPO；技能进化动力学 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-04_CoSkill_Joint_Reinforcement_Learning_of_Reasoning_and_Meta-Skill_Agents_for_Hierarchical_Skill_Evolution.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：长程 agentic RL 中，如何让"技能库的进化"与"推理策略的优化"不再割裂，而是同一目标下端到端共适应。任务场景是通用 agent（文本交互式决策环境 ALFWorld / WebShop），非 GUI 视觉落地。
- **为什么重要**：技能库是提升 agentic RL 样本效率的核心手段，但技能若不能随策略一起进化，会越积越陈旧、冗余，甚至与正在演进的策略失配，直接卡住样本效率和最终性能。
- **现有方法有什么不足**：作者把前人归为三代范式并逐一批评——(a) **外部编排式进化**（SkillRL/D2Skill/ReSkill/Trace2Skill）把技能生成修订交给外部 LLM 或手写规则，进化游离于策略学习目标之外；(b) **RL 优化的库管理**（SAGE/ARISE/Skill1）把技能生命周期决策放进 RL，但粒度粗、技能被当原子条目，内部流程不优化，"有用但不完美"的技能被弃用而非精修；(c) **meta-skill 驱动优化**（SkillOpt/EvoSkill/SkillEvolver/MetaSkill-Evolve）用执行反馈改写单个技能，但依赖预定义 meta-skill 程序（rubric/SKILL.md/多智能体 workflow）且冻结推理 agent，固定更新规则无法共适应。
- **Research Gap**：三代的共同缺陷是**把技能当作被动管理对象**。作者 claim 的缺口是——meta-skill 本身能否变成 RL 可学习对象，并与推理策略一起进化。CoSkill 正面补上这一缺口。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把 meta-skill 建模为可训练 agent（而非固定 workflow），与推理 agent 共享同一策略参数，用分层技能库 + 验证后写回 + GiGPO 联合优化，让技能编辑由下游执行反馈直接校准。

### 3.2 方法总览（Pipeline）

- **输入**：任务指令 x 与当前观测 o_t；**输出**：环境动作 a_t（推理角色）+ 结构化技能编辑 z_t（元技能角色）。
- **模块**：(1) **分层技能库** B={B_k}，每个技能束 = 一个任务技能（episode 级指引）+ 子技能集合（step skills，观测自适应的局部流程）；先全局检索任务技能，再只在子树内检索 step skill。(2) **共享参数 MSMDP**：双角色共享 π_θ 与 token 空间，靠 role prompt 牵引出异构语义动作空间，推理输出环境动作，元技能输出 INSERT/UPDATE/DELETE/KEEP 四类结构化编辑。(3) **Post-Edit 验证奖励**：编辑先 staging 到 episode 结束，作用到私有库拷贝，重置同任务跑 M 次验证回合，得 Δ_skill = mean(验证收益) − R⁰。(4) **GiGPO 联合优化**：双角色轨迹按 rollout 配对，各自算 episode/step 级相对优势并按角色分别归一化，联合损失 L = L_R + L_S 更新共享 actor。
- **连接**：每个推理 transition 之后，元技能 agent 观察"推理上下文 + 动作 + 环境反馈"提出编辑 → 基线回合结束后应用到私有库 → 验证 → 有效才晋升持久库 → 双角色批次一起更新策略。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：① 把 meta-skill 从 workflow/提示词升格为可学习 agent，用 MSMDP 建模推理/编辑的时标差异；② post-edit 验证奖励（基线相减控难度 + 多次验证降方差 + 写回前验证）把技能价值绑定到下游任务收益；③ 角色分离的 GiGPO 优势归一化，避免执行奖励与技能验证奖励尺度互相泄漏。
- **工程组合**：共享 backbone 双角色靠 prompt 区分的"零新增参数"多智能体（复用 PPO/GRPO/GiGPO 现有优化器）；offline-to-online 预置库（离线轨迹蒸馏 + grounding 检查）；veRL/verl-agent 训练栈。
- **对性能提升最关键的设计**：消融证明**Meta-Skill RL**（去掉后 step20 掉 12.50 pp）与**分层库**（去掉后 step20 掉 14.06 pp）是早期样本效率的关键；**高频紧耦合联合更新**（改成每 10/20 步交替，step60 掉 25.00/18.75 pp）是收敛稳定性的关键。
- **证据不足 / 仅声称有效**：M 次验证"降方差"的理论收益未消融（主实验实际取 M=1，多次验证的贡献只是陈述）；角色靠 prompt 分离的鲁棒性（提示漂移、RL 偏向某角色）无消融；部分基线数值引用历史表（如 D2Skill 无标注配置），跨论文公平性未完全自控。

---

## 4. 具体技术细节

### 4.1 模型结构

- **Base model**：Qwen2.5-7B-Instruct，**纯 LLM（无视觉编码器）**，双角色共享同一 backbone 与 token 空间，参数量 7B。技能检索用**冻结**的文本编码器 Qwen3-Embedding-0.6B（计算相似度，不参与 RL 更新）。
- 双角色异构动作空间仅由 role prompt p_R / p_S 区分，元技能动作空间被约束为四类结构化 JSON 编辑。

### 4.2 训练流程

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| Stage 1（离线初始化） | 构建分层技能库 B0 | 不训练，外部 LLM 生成 + grounding 检查 | 离线经验轨迹（N0=8 rollouts/任务组） | task skill + 若干 grounded step skills（K=300 束） | 无（LLM 生成 + 规则过滤） |
| Stage 2（online 联合 RL） | 推理与技能编辑端到端共适应 | 共享 π_θ 的两个角色：环境动作 + 技能编辑 | 环境 rollout（16 任务组 × 8 rollouts）；M=1 次验证 | 推理轨迹 D_R（动作-观测-奖励）+ 编辑序列 D_S（结构化编辑提案） | 联合 GiGPO：L = L_R + L_S，clipped PPO + KL 正则（β=0.01 low-variance KL，clip 0.2，lr 1e-6，γ 0.95） |

> 数据流关键：基线回合产出 D_R 与 staged 编辑；编辑应用到私有库后重跑同任务做验证（M=1），验证收益与基线收益之差 Δ_skill 作为编辑序列的延迟 episode 奖励；每组只晋升 top-K_prom(=2) 版本、保留原技能血缘，容量 300、每 5 步按效用剪枝。

### 4.3 推理流程

- 使用共享 backbone π_θ 按角色 prompt 分派；推理输入为 (x, o_t, h_t, Z_task, Z_step_t)，输出环境动作；元技能输入 u_S_t = (o_t, Z_step_t, a_t, r_t, e_t, o_{t+1}, h_t)，输出单个结构化 JSON 编辑。
- **多步循环**：ReAct 式逐步执行，每个 transition 后元技能 agent 提出编辑；episode 终止条件为环境终止或步数上限（ALFWorld 30 步 / WebShop 15 步）。编辑提案只在 episode 结束后才应用（staging 保证基线轨迹无损）。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| ALFWorld | General（文本家务） | 长程 embodied 决策（Pick/Look/Clean/Heat/Cool/Pick2 六类） | 训练 AlfredTWEnv；验证 64 任务/每 5 步 | 纯文本指令 + 文本观测 | 文本动作 | 分类成功率 + macro 平均 |
| WebShop | General（模拟电商） | 搜索/导航/购买满足指令的商品 | Small 子集（1K 商品，非人类目标） | 纯文本指令 + 页面文本 | 文本动作 | task score + 成功率 |

### 5.2 实验结果分析

**主结果**（Table 1，backbone 均 Qwen2.5-7B）：

- ALFWorld 平均成功率 **98.4%**（6 类中 5 类满分，仅 Cool 90.0），+3.5 pp 超此前最强 RetroAgent（94.9）；WebShop score **95.9**（+4.3）、成功率 **90.6%**（+6.2）。
- 相较无技能 GiGPO 基线：ALFWorld +7.6 pp、WebShop 成功率 +17.8 pp——增益不来自优化器本身。
- 相较外部编辑器（最有力证据）：超 SkillRL +8.5/+17.9 pp；超最强 D2Skill 变体（Gemini-3-Flash/O3 当编辑器）+7.8/+6.2 pp——**CoSkill 仅凭共享 7B backbone 胜出，说明与环境奖励对齐比编辑器规模更重要**。
- 相较原子级管理 Skill1：+4.7 pp（ALFWorld）/+15.6 pp（WebShop），其中 Heat/Cool/Pick2 分别 +12.5/+23.4/+7.7 pp。

**Ablation**（Table 2，ALFWorld 跨训练步）：

- 去掉 **Meta-Skill RL**：step20 从 70.31% 掉到 57.81%（−12.50 pp），终值 92.19%（−3.12 pp）——RL 对齐技能编辑与下游反馈，加速早期学习。
- 去掉 **分层库**：step20 掉 14.06 pp，终值 93.75%（−1.56 pp）——层级减少无关技能干扰、稳定长程学习轨迹。
- **交替更新 =10/20 步**：step60 落后 25.00/18.75 pp，step120 仅 78.12/81.25%——高频紧耦合联合更新不可少（stale 跨角色反馈所致，非容量不足）。

**进化动力学**（Figure 5）：有 RL 时编辑动作收敛到 97–99% UPDATE、DELETE 近乎消失；无 RL 时 UPDATE 从 73% 降到 54%、DELETE 从 13% 涨到 36%（高熵但未校准）。RL 使累计晋升技能束减少 **47.6%**（387 vs 739）反而更强；step-skill 检索命中率恢复约 100%；空技能束率从 12.4% 降到 0.2%。

---

## 6. 亮点与贡献（Why it matters）

1. **首次把 meta-skill 从 workflow 变成可学习 agent**，技能进化由"外部编排/启发式"变为与策略同目标的优化环节，一次性补齐三代范式的结构性缺口。
2. **单 backbone 共享参数的零新增模型多智能体**：不引入第二个模型即可获得"推理 + 编辑"协作，训练与部署开销轻、天然端到端。
3. **验证后写回 + 基线相减 + top-K 晋升 + 血缘保留**：把库增长做成有纪律、可审计的过程，直接回应"技能库膨胀/退化"这一工程痛点。
4. **完整的多层证据链**：从终点性能、样本效率、消融到编辑动力学 + 案例研究，有力反驳"用最强闭源模型做外部技能编辑"的主流做法——对齐环境奖励 > 编辑器规模。
5. 训练成本可量化（ALFWorld 160 步约 66 wall-clock hours / 526 GPU-hours），可复现性好。

## 7. 局限与可改进点（个人点评）

- **评测面窄**：两个环境均为文本观测、终局奖励明确可判定的模拟任务；无视觉/GUI/真实网页，技能条件与检索在截图/像素观测下是否成立未验证。在"结果难验证"的现实任务里 post-edit 验证奖励的信用会大幅削弱。
- **M=1 使"多次验证降方差"名不副实**：公式写 mean over M 次，主实验 M=1，降方差只停留在理论陈述，未见 M>1 的消融或成本-收益分析。
- **角色分离脆弱性未研究**：双角色仅靠 prompt 区分，若提示漂移或 RL 更新偏向某角色，共适应可能退化为互扰；缺少对应消融与长期稳定性分析。
- **依赖结构先验**：任务/子技能的 bundle 划分与离线预置库质量直接影响结果；未测冷启动、跨任务/未见任务泛化，也未给库规模上界与技能过时度量。
- **迁移成本**：M 次验证 rollouts 需重置同任务，重置代价高的真实 GUI 环境几乎无法照搬；wall-clock 优于基线不等于绝对成本低。

## 8. 对我们的启示 / 可借鉴点

- **最值得迁移的洞见是"技能管理本身可训练"**：与其用闭源大模型定期改写 skill prompt（昂贵且失配），不如把技能编辑做成与推理策略共享 backbone 的第二角色/动作头，用下游奖励端到端校准——正是我们 Online/RL 方向的落点。
- **分层技能对 GUI 天然契合**：任务技能如"填写报销单"、step skill 条件于界面状态如"日期弹层出现后点今天"；两级检索限定子树，可显著减少跨 App/跨页面技能互相干扰。
- **Post-Edit 验证可变成 GUI 版"写回门禁"**：候选技能先过可执行验证（截图 diff、无障碍树/DOM 断言、终点状态谓词）再入库，与确定性 guardrail 思路契合，防止"语言上漂亮但操作有害"的技能沉淀。
- **信用分配可借鉴 GiGPO 的分组思想**：GUI 长轨迹动作稀疏、奖励延迟，按 anchor state（观测,活动技能）构造 step 级相对优势，并把"技能改写奖励"与"动作奖励"分开归一化，能缓解奖励尺度互相污染。
- **迁移的坑**：GUI 观测是像素/AT 树而非文本，技能条件、检索与编辑都需多模态 grounding 编码支撑；真实界面重置昂贵，M 次验证需改为录屏回放、合成页面或弱验证器；同时要显式建模"何时该删除/不复用技能"，避免技能改写破坏当前已正确的行为。

## 9. 延伸阅读

- **三代范式对照**：SkillRL（Xia et al. 2026）、D2Skill（Tu et al. 2026）、Skill1（Shi et al. 2026）、ReSkill（He et al. 2026）、Trace2Skill（Ni et al. 2026）；可迁移技能 RL：Skill Set Optimization（Nottingham et al., ICML 2024）。
- **优化器谱系**：PPO / RLOO / GRPO 与本文 GiGPO（Feng et al., NeurIPS 2025，veRL 系）。
- **分层/技能先例**：SkillAct（ICML 2024）、MetaFlows（NeurIPS 2025）；开放世界技能库：Voyager。
- **评测环境**：ALFWorld（ICLR 2021）、WebShop（ICLR 2022）。

---
*解读生成时间：2026-09-08 ｜ 解读人：WorkBuddy（AI）*
