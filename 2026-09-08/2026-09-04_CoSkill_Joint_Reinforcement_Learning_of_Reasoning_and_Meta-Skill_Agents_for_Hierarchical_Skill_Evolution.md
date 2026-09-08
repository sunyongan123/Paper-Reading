# CoSkill: Joint Reinforcement Learning of Reasoning and Meta-Skill Agents for Hierarchical Skill Evolution（CoSkill：推理智能体与元技能智能体的联合强化学习，实现分层技能的协同进化）

> TL;DR 现有技能库范式要么把技能进化与策略优化解耦、要么把 meta-skill 做成固定 workflow。CoSkill 把静态 meta-skill workflow 重铸为一个**可学习的 Meta-Skill Agent**，与 Reasoning Agent **共享同一 backbone、联合 RL 训练**：推理策略学会利用演进的技能，元技能智能体学会按下游执行反馈精修技能，实现端到端共适应。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | CoSkill: Joint Reinforcement Learning of Reasoning and Meta-Skill Agents for Hierarchical Skill Evolution |
| **arXiv ID / DOI** | arXiv:2609.04865v1 [cs.AI]（无 DOI） |
| **arXiv 链接** | https://arxiv.org/abs/2609.04865 |
| **发表出处（Venue）** | arXiv preprint（2026-09-04 预印本） |
| **发布时间** | 2026-09-04 |
| **作者** | Jinyuan Feng¹、Dongmin Li¹、Yiqun Chen²、Yang Gao³、Xing Chen³、Huimu Wang¹、Zhiqiang Pu¹ |
| **所属机构** | ¹中国科学院自动化研究所（Institute of Automation, CAS，中国）；²中国人民大学（中国）；³字节跳动（ByteDance，中国） |
| **开源情况 / 代码** | ✅ 有代码：https://github.com/jinyuan-cookie/CoSkill（基于 verl 与 verl-agent 实现，另有可复现性声明） |
| **类型标签** | `RL` `Planning` `General` |
| **训练方法标签** | `RL (GiGPO)`；online 多智能体联合训练，双角色共享单 backbone（Qwen2.5-7B-Instruct），含 post-edit verification reward |
| **关键词** | 分层技能库；Meta-Skill Agent；技能-策略共适应；GiGPO；ALFWorld；WebShop |
| **来源渠道** | arxiv-api + listing |
| **PDF 存档** | 2026-09-04_CoSkill_Joint_Reinforcement_Learning_of_Reasoning_and_Meta-Skill_Agents_for_Hierarchical_Skill_Evolution.pdf |

---

## 1. 研究背景与要解决的问题

长程 agentic RL（让 LLM 智能体在环境中多步交互学策略）的痛点是样本效率。技能库让智能体跨任务复用程序性知识，能显著改善这一点，但作者归纳出三代范式的共同缺陷——它们都**把技能当作被动管理的对象**：

- **(a) 外部编排式技能进化**（SkillRL、D2Skill、ReSkill、Trace2Skill）：把技能库当独立知识库，技能生成/修订交给外部 LLM 或手写规则。技能进化游离于策略学习目标之外，库越积越陈旧、冗余，与正在演进的策略失配。
- **(b) RL 优化的库管理**（SAGE、ARISE、Skill1）：把库生命周期决策放进 RL 循环，用下游奖励决定增删。但粒度粗——技能被当作原子条目，内部流程本身不被优化，"有用但不完美"的技能可能被弃用/删除而非被精修。
- **(c) meta-skill 驱动的技能优化**（SkillOpt、EvoSkill、SkillEvolver、MetaSkill-Evolve）：用执行反馈直接改写单个技能，但依赖预定义的 meta-skill 程序（rubric 提示词、SKILL.md、多智能体 workflow），且通常冻结推理 agent——固定更新规则无法与演进中的策略共适应。

核心缺口：**meta-skill 本身能否变成 RL 可学习对象，并与推理策略一起进化？** CoSkill 正面回答这个问题。

## 2. 核心方法 / 思路

CoSkill 是一个统一多智能体 RL 框架，核心设计洞见是：把 meta-skill 建模成**可训练 agent 而非固定 workflow**，其更新策略就能与推理策略共适应。技术要点分四块：

**① 分层技能库与两级检索。** 库 B={B_k} 由 K 个"按任务索引的技能束"组成，每束 = 一个任务技能（task skill，episode 级指引）+ 它的子技能集合（step skills，针对中间观测的局部流程）。给定任务指令先全局检索任务技能，**再只在该任务的子树内**按当前观测检索 step skill。任务技能提供稳定全局指引，step skill 提供观测自适应的局部决策，子树限定减少无关技能干扰。初始库由离线经验预置（offline-to-online）。

**② 共享参数的半马氏多智能体过程（MSMDP）。** 两个角色共享同一策略 π_θ 与 token 空间，仅靠不同 role prompt 牵引出异构语义动作空间：Reasoning Agent 输出环境动作 a_t；每个推理 transition 后，Meta-Skill Agent 观察"推理上下文+动作+环境反馈"，输出结构化编辑动作 INSERT/UPDATE/DELETE/KEEP。候选编辑先 staging 到 episode 结束，再作用到私有库拷贝上——保证基线轨迹无损、可验证。

**③ Post-Edit 技能验证奖励。** 编辑序列的价值只能由下游任务执行来定：基线回合得 R⁰，随后重置同任务、用私有编辑库跑 M 次验证回合，得 Δ_skill = mean(验证收益) − R⁰。基线相减控制任务难度，多次验证降方差。只有"有效 + 非平凡 + 验证确实改善"的候选才被写入持久库，且每组只晋升 top-K_prom 版本、保留原技能血缘——有纪律地约束库膨胀。

**④ GiGPO 联合优化。** 双角色的轨迹按 rollout 配对；每个角色各自算 episode 级相对优势 + step 级相对优势（推理以 (x,o_t) 为 anchor，编辑以 (x,o_t,当前 step skill) 为 anchor），按角色**分别归一化**，避免任务执行与技能验证两套奖励尺度互相泄漏。联合损失 L = L_R + L_S 更新共享 actor——一次策略更新同时推进推理与技能编辑，实现端到端共适应。

## 3. 关键实验结果

在 **ALFWorld**（文本家务）与 **WebShop**（模拟电商）上，backbone 为 Qwen2.5-7B-Instruct，训练栈 verl/verl-agent。关键数字：

- **主结果**：ALFWorld 平均成功率 **98.4%**（6 类中 5 类满分，+3.5 pp 胜过此前最强 RetroAgent 94.9）；WebShop score **95.9**（+4.3）、成功率 **90.6%**（+6.2）。
- **增益不来自优化器本身**：相比无技能 GiGPO 基线 +7.6 pp（ALFWorld）/ +17.8 pp（WebShop）。
- **与外部编辑器对比最具说服力**：超过 SkillRL +8.5/+17.9 pp；超过最强 D2Skill 变体 +7.8/+6.2 pp——而 D2Skill 用 Gemini-3-Flash/O3 当外部技能编辑器，CoSkill 仅凭共享 7B backbone 胜出，说明**与环境奖励对齐比编辑器规模更重要**。
- **细粒度优于原子级管理**：比 Skill1 +4.7 pp（ALFWorld）/ +15.6 pp（WebShop），其中 Heat/Cool/Pick2 分别 +12.5/+23.4/+7.7 pp。
- **样本效率与消融**：step 20 已达 70.31%（高出所有消融 7.81–14.06 pp），step 100 收敛到 95.31%。去掉 Meta-Skill RL：step 20 掉 12.50 pp、终值仅 92.19；去掉分层库：step 20 掉 14.06 pp；把联合更新改成每 10/20 步交替：step 60 落后 25.00/18.75 pp，step 120 只有 78.12/81.25——**高频紧耦合联合更新必不可少**。
- **进化动力学**：有 RL 时编辑动作收敛到 97–99% UPDATE、DELETE 近乎消失；无 RL 时 UPDATE 从 73% 降到 54%、DELETE 从 13% 涨到 36%（高熵但未被环境校准）。RL 使累计晋升技能束减少 **47.6%**（387 vs 739）反而更强；step-skill 检索命中率恢复到约 100%；空技能束率从 12.4% 降到 0.2%。

## 4. 亮点与贡献（Why it matters）

1. **首次把 meta-skill 从 workflow/提示词变成 agent**，技能进化由"外部编排/启发式"变为可学习的、与策略同一目标的优化环节——补齐三代范式各自的结构性缺口。
2. **单 backbone 共享参数**：不引入第二个模型即可获得"双 agent"协作，训练、部署开销都很轻，且天然端到端。
3. **验证后写回 + 基线相减 + top-K 晋升**：把库增长做成有纪律、可审计的过程，直接呼应"技能库会膨胀/退化"这一工程痛点。
4. **实验叙事完整**：从终点性能、样本效率、消融到编辑动力学多层证据，证明"与环境奖励对齐 > 编辑器规模"，对当下"用最强闭源模型做外部技能编辑"的主流做法是有力反驳。

## 5. 局限与可改进点（个人点评）

- **评测面窄**：两个环境都是文本观测、终局奖励明确可判定的模拟；没有视觉/GUI/真实网页，未验证截图类观测下技能条件与检索是否成立。在"结果难验证"的现实任务里，post-edit 验证奖励的信用会弱很多。
- **角色分离的脆弱性未研究**：双角色靠 role prompt 区分，若提示漂移或 RL 更新偏向某角色，共适应可能退化成互扰；无针对该风险的消融与长期稳定性分析。
- **训练成本被低估**：M 次验证 rollouts 的开销未单列；wall-clock 胜过基线不等于绝对成本低，迁移到重置代价高的真实 GUI 环境时，验证回合几乎不可能照搬。
- **依赖结构先验**：任务技能/子技能的人工 bundle 划分与离线预置库质量直接影响结果；未测完全冷启动或跨任务、未见任务类型上的泛化，也未给出库规模上界与技能过时的度量。
- **基线数值部分引用历史表**（如 D2Skill 无标注配置），跨论文比较的公平性（提示词、rollout 预算）未完全自控。

## 6. 对我们的启示 / 对 GUI Agent 的可借鉴点

- **GUI 天然适合分层技能**：任务技能如"填写报销单"、step skill 条件于界面状态如"日期弹层出现后点今天"。两级检索限定子树，可显著减少跨 App/跨页面技能的互相干扰。
- **最值得迁移的洞见是"技能管理本身可训练"**：与其用闭源大模型定期改写 skill prompt（昂贵、失配）或用规则管理库，不如把技能编辑做成**与推理策略共享 backbone 的第二角色/动作头**，用下游 GUI 奖励端到端校准——这正是我们 Online/RL 方向的落点。
- **Post-Edit 验证可变成 GUI 版"写回门禁"**：候选技能先过可执行验证（截图 diff、无障碍树/DOM 断言、终点状态谓词）再入库，与确定性 guardrail 思路天然契合，防止"语言上漂亮但操作有害"的技能沉淀。
- **信用分配可借鉴 GiGPO 的分组思想**：GUI 长轨迹动作稀疏、reward 延迟，按 anchor state（观测,活动技能）构造 step 级相对优势、且把"技能改写奖励"与"动作奖励"分开归一化，能缓解奖励尺度互相污染。
- **迁移的坑**：GUI 观测是像素/AT 树而非文本，技能条件、检索与编辑都要多模态 grounding 编码支撑；真实界面重置昂贵，M 次验证 rollouts 需改为录屏回放、合成页面或弱验证器；同时要显式建模"何时不该复用/该删除技能"，避免技能改写破坏当前已正确的行为。

## 7. 延伸阅读

- **三代范式对照**：SkillRL（Xia et al. 2026）、D2Skill（Tu et al. 2026）、Skill1（Shi et al. 2026）、ReSkill（He et al. 2026）、Trace2Skill（Ni et al. 2026）；更早的可迁移技能 RL：Skill Set Optimization（Nottingham et al. ICML 2024）。
- **优化器谱系**：PPO/RLOO/GRPO 与本文采用的 GiGPO（Feng et al. NeurIPS 2025，veRL 系），同仓对比方便。
- **分层/技能先例**：SkillAct（ICML 2024）、MetaFlows（NeurIPS 2025）；开放世界技能库：Voyager。
- **评测环境**：ALFWorld（ICLR 2021）、WebShop（ICLR 2022）。

---
*解读生成时间：2026-09-08 ｜ 解读人：WorkBuddy（AI）*
