# 📖 当日论文速览 · 2026-09-08

> 处理日：2026-09-08（周二）｜ 对应 arXiv 公告组：**Mon, 7 Sep 2026**（本周首个新公告组，6 分类合计 521 条，去重 404 条）
> 主题一句话：**"技能沉淀与混合界面"爆发日**——GUI 域迎来混合 GUI+CLI 环境全流程配方（CUA-Universe）、computer-use 在线技能演化（Interaction Traces）、老年用户真实表达评测（ElderBench）与车载 GUI 自动测试（ARIA）；Agent 方法论域 8 篇围绕"轨迹→技能/图→策略协同进化"（CoSkill/Trace2Tower/TROVE）、蒸馏防漂移（RISE/Persistent Teacher）与 RL 训练配方实证（SiLR/Multi-Harness RL）展开，另收一篇 Agentic AI 全景综述（World-Acting Systems）。
> 本日入选 **12 篇**（GUI 4 + Agent 方法论 8）并完成精读解读；GitHub awesome 两列表近 7 天无新增。
> 标签图例：`类型标签`（论文类别）｜ `训练方法标签`（方法级）｜ 域标注：GUI / Agent 方法论·跨域参考

---

## GUI 域（本领域，4 篇）

### CUA-Universe：混合 GUI+CLI 智能体的可扩展动态环境

- 标签：`Benchmark` `GUI Grounding` `Desktop` ｜ 训练方法：`Distillation + SFT (LoRA)`
- 域标注：**GUI**
- 一句话结论：把真实桌面软件改造成共享同一应用状态的 GUI+CLI 混合环境，端到端批量产出混合任务与高质量轨迹，9B 开源模型 OSWorld SR 40.2%（vs 纯 GUI 23.4%）、CUA-Verse Score 0.582 超过教师 Kimi K2.5。
- [arXiv](https://arxiv.org/abs/2609.05374) ｜ 代码：⚠️ 承诺开源（文中声明将 release，未给 URL）
- **[阅读完整解读](./2026-09-04_CUA-Universe_A_Scalable_and_Dynamic_Environment_for_Hybrid_GUICLI_Agents.md)**

### From Interaction Traces to Persistent Skills：计算机使用智能体的在线技能演化

- 标签：`Online` `Reflection` `Desktop` ｜ 训练方法：—（方法/框架，不更新模型参数）
- 域标注：**GUI**
- 一句话结论：把 agent 每次交互的轨迹与评估反馈改写成带版本、可追溯的持久技能库（冻结模型参数），用冻结技能快照迭代执行，让 computer-use agent 越用越聪明（多个 GUI 任务 +5.7~18.6pp）。
- [arXiv](https://arxiv.org/abs/2609.04869) ｜ 代码：✅ [Skill-Evo4GUI](https://github.com/LongtaoHu/Skill-Evo4GUI.git)
- **[阅读完整解读](./2026-09-04_From_Interaction_Traces_to_Persistent_Skills_Online_Evolution_for_Computer-Use_Agents.md)**

### ElderBench：面向老年用户的移动 GUI Agent 评测基准

- 标签：`Benchmark` `Mobile` ｜ 训练方法：—（评测）
- 域标注：**GUI**
- 一句话结论：首个以老年用户真实表达为基准的移动 GUI 评测集——249 条自然诱发任务/20 个应用，量化老年指令（间接表述、指代歧义、低指定度）与现有基准指令的语法/语义/语用落差，揭示主流 agent 在该人群上的显著退化。
- [arXiv](https://arxiv.org/abs/2609.04850) ｜ 代码：未开源
- **[阅读完整解读](./2026-09-04_ElderBench_Benchmarking_Autonomous_Mobile_Agents_for_Older_Adults.md)**

### ARIA：车载信息娱乐系统自主测试的多智能体框架

- 标签：`General` `Benchmark` ｜ 训练方法：—（工程/评测）
- 域标注：**GUI**
- 一句话结论：面向 Android 车载 infotainment 的四 Agent 闭路端到端 GUI 测试框架，工业环境 30 场景跑通 93.3%、5 个已知缺陷全部检出（0 漏报），但误报偏高仍需人工复核。
- [arXiv](https://arxiv.org/abs/2609.04913) ｜ 代码：未开源（受 NDA 约束）
- **[阅读完整解读](./2026-09-04_ARIA_-_An_Agentic_Framework_for_Autonomous_Testing_of_Infotainment_Systems.md)**

---

## Agent 方法论域（跨域参考，8 篇）

### From Language Models to World-Acting Systems：Agentic AI 全景批判性综述

- 标签：`General` `Web` `Desktop` ｜ 训练方法：—（综述）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：以 2026-08-31 为文献截止线的批判性综述，沿"委托权威 / 时间持续性 / 环境耦合"三维取证并区分 model/harness/environment，论证"行动接口扩张远比可验证的可靠自主更有证据支撑"，提出 justified delegation 作为分析基准与研究议程（覆盖 computer-use/MCP/GUI agent 现状）。
- [arXiv](https://arxiv.org/abs/2609.04894) ｜ 代码：未开源（综述）
- **[阅读完整解读](./2026-09-04_From_Language_Models_to_World-Acting_Systems_Progress_and_Limits_of_Agentic_AI_across_Digital_Social_Virtual_and_Physical_Environments.md)**

### CoSkill：推理智能体与元技能智能体的联合强化学习

- 标签：`RL` `Planning` `General` ｜ 训练方法：`RL (GiGPO)`（Qwen2.5-7B 共享 backbone，online 联合训练）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：把静态 meta-skill workflow 重铸为可学习的 Meta-Skill Agent，与 Reasoning Agent 共享单 backbone 做端到端联合 RL，使技能库随策略共适应而非被动管理，在 ALFWorld/WebShop 等任务上验证分层技能进化。
- [arXiv](https://arxiv.org/abs/2609.04865) ｜ 代码：✅ [CoSkill](https://github.com/jinyuan-cookie/CoSkill)
- **[阅读完整解读](./2026-09-04_CoSkill_Joint_Reinforcement_Learning_of_Reasoning_and_Meta-Skill_Agents_for_Hierarchical_Skill_Evolution.md)**

### RISE：经由自外推策略蒸馏的递归式自我提升

- 标签：`RL` `Online` `Distillation` `General` ｜ 训练方法：`RLVR (GRPO) + On-policy Distillation`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：不依赖外部教师或特权信息，把模型自身 RLVR 训练中相邻 checkpoint 的"进步位移"外推成"未来的自己"当教师做逐 token 蒸馏，让蒸馏从一次性压缩变成递归自我改进（Salesforce AI Research）。
- [arXiv](https://arxiv.org/abs/2609.05295) ｜ 代码：未在文中声明开源
- **[阅读完整解读](./2026-09-04_RISE_Recursive_Improvement_via_Self-Extrapolating_Policy_Distillation.md)**

### Trace2Tower：面向转移的 EigenTrace 多层技能归纳

- 标签：`General` `Planning` `Web` ｜ 训练方法：—（无模型权重训练：对比谱分解 + 图编辑）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：把 LLM Agent 原始成败轨迹抽象成规范事件，用"语义+转移+结果"三类证据构图，经对比谱分解剔除失败捷径，归纳出动作—程序—策略三层技能塔，提供紧凑可复用的层级经验。
- [arXiv](https://arxiv.org/abs/2609.05261) ｜ 代码：✅ [Trace2Tower](https://github.com/FudanSELab/Trace2Tower)
- **[阅读完整解读](./2026-09-04_Trace2Tower_Transition-Aware_EigenTrace_Induction_of_Multi-Level_Skills_for_LLM_Agents.md)**

### TROVE：基于轨迹证据的智能体技能编排（执行期路线校验与编辑）

- 标签：`Planning` `Online` `General` ｜ 训练方法：—（无模型参数训练：工作流搜索 + 轨迹蒸馏 + 启发式/LLM 在线控制）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：针对"执行前锁定路线"的编排瓶颈，离线把评估过的工作流搜索轨迹蒸馏成原子/组合技能与 outcome-conditioned 转移图，运行时只修订被证据作废的后缀，兼顾质量与效率。
- [arXiv](https://arxiv.org/abs/2609.05019) ｜ 代码：未在文中声明开源
- **[阅读完整解读](./2026-09-04_TROVE_Adaptive_Agent_Skill_Orchestration_via_Trace-Grounded_Route_Validation_and_Editing.md)**

### Persistent Teacher Anchoring：面向工具使用智能体的教师锚定蒸馏

- 标签：`Distillation` `RL` `General` ｜ 训练方法：`Distillation`（on-policy forward-KL, top-256/K=3）+ 下游 `RL`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：学生逐块提议、老师逐块校验，只有老师放行的整轮才真正执行工具，缓解 on-policy 蒸馏中"学生越走越偏 → teacher-student 分布 gap 累积"的漂移问题（西江大学+密歇根大学）。
- [arXiv](https://arxiv.org/abs/2609.04773) ｜ 代码：未在文中声明开源
- **[阅读完整解读](./2026-09-04_Persistent_Teacher_Anchoring_for_Tool-Using_Agents.md)**

### SiLR：保持结构的准入与过程奖励（LLM 工具智能体运行时门禁）

- 标签：`RL` `Online` `General` ｜ 训练方法：`RL (GRPO) + LoRA`（策略侧；门禁为确定性 verifier + shadow 模拟）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：把 LLM 工具 agent 的运行时门禁从"标量过滤器"重铸为"保留违规几何结构的 product order 准入"，并同步用作 GRPO 过程奖励，破解聚合分数门禁的"投影陷阱"（21/21 违规族全拦截、性能不降）。
- [arXiv](https://arxiv.org/abs/2609.04629) ｜ 代码：未在文中声明开源
- **[阅读完整解读](./2026-09-04_SiLR_Structure-Preserving_Admission_and_Process_Reward_for_LLM_Tool_Agents.md)**

### What Does Multi-Harness RL Learn?（编程 Agent 的信用分配与可移植性）

- 标签：`RL` `Online` `General` ｜ 训练方法：`SFT warm-start + RL (GRPO)`（干预变量：分组边界 Within vs Cross）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：在 repo 级 coding 上隔离"多 harness 配方"中的 credit assignment 选择——固定曝光与预算、只移动 GRPO 相对优势的分组边界（逐 harness 分组 Within vs 混池 Cross），结论是二者在未见 harness 上无统计差异，**评测 harness 本身才是主导变量**。
- [arXiv](https://arxiv.org/abs/2609.04518) ｜ 代码：⚠️ 声明随文发布 artifacts（未给 URL）
- **[阅读完整解读](./2026-09-03_What_Does_Multi-Harness_RL_Learn_Credit_Assignment_and_Portability_in_Coding_Agents.md)**

---

## 本日趋势归纳

**按标签统计**（每篇 1~4 标签）：`RL`×6 ｜ `General`×8 ｜ `Online`×6 ｜ `Planning`×3 ｜ `Distillation`×2 ｜ `Benchmark`×3 ｜ `GUI Grounding`×1 ｜ `Reflection`×1 ｜ `Desktop`×2 ｜ `Mobile`×1 ｜ `Web`×2
**按域分布**：GUI 4 篇（2 Benchmark + 1 混合 GUI+CLI 训练 + 1 在线技能）｜ Agent 方法论 8 篇（RL/蒸馏/技能编排/综述）

**几个值得注意的方向信号**：
1. **GUI 域进入"混合界面"时代**：CUA-Universe 把 GUI+CLI 共享状态的混合操作做成可扩展环境与数据流水线，且 OSWorld 跨域迁移 +16.8pp、训练数据一台 CPU 机器即可产出——"环境合成→数据合成→后训练"成为 GUI agent 的主流配方。
2. **在线技能演化是当日最大共性主题**：Interaction Traces（GUI 落地）、CoSkill、Trace2Tower、TROVE 四篇不约而同处理"把轨迹沉淀成可复用技能/图，并在运行时让技能与策略协同进化"，方法与 GUI 场景（界面轨迹天然是结构化事件序列）高度契合。
3. **蒸馏质量与漂移问题升温**：RISE（自外推教师）、Persistent Teacher（proposer-verifier 防漂移）、CUA-Universe（蒸馏式 SFT）三篇从不同角度回答"教师从哪来、如何防止学生走偏"。
4. **RL 训练配方的细粒度实证**：Multi-Harness RL 提醒我们 GRPO 分组边界等"配方细节"的收益可能被评测 harness 差异淹没，做 GUI RL 对比实验时须警惕类似混杂。
5. **评测关注"真实用户"与"安全/违约"**：ElderBench（老年自然表达）、ARIA（工业 infotainment 缺陷检出）、SiLR（违规动作准入）把 GUI/agent 评测从理想指令拉向真实使用人群与违约恢复场景。
