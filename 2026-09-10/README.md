# 📖 当日论文速览 · 2026-09-10

> 处理日：2026-09-10（周四）｜ **arXiv 未发布新公告组**——6 分类（cs.HC/cs.AI/cs.LG/cs.CL/cs.SE/cs.CR）listing 最新组仍为 **Wed, 9 Sep 2026**（昨日已处理组）。
> 故本日入选的 **15 篇为 Wed, 9 Sep 组内昨日 15 篇上限未覆盖的存量论文（backlog）**，按「GUI 域优先 → 日期新 → 主题贴合（RL/Grounding/Online/Planning/Reflection/Distillation）」补足；剩余 29 篇（多为安全/渗透测试/领域应用类）记入 `logs/2026-09-10_backlog.json`。
> 主题一句话：**GUI 域回到"持续学习 + 知识资产"主线**——SKC 用神经元级梯度手术让 GUI Agent 在应用流上不遗忘地持续进化，Typed Federated Artifacts 探索冻结异构 agent 间的工具路由知识共享，SafeMem 用长期图记忆补上"离开视野的危险物"这一安全盲区，SciFigure2Code 把科研图逆向重建为可编辑代码评测；**Agent 方法论域则集中在"把经验/结构变成可复用资产"与"多智能体规模化"两条线**——技能图自演化（SE-GoS）、递归自改进后训练（NeoHorse-1）、场景化记忆（CreaMem）、轨迹数据库愿景（TrajectoryDB）、长时程世界模型闭环（Hi-FLoop）、多智能体前景状态传播与子队分解（PspMAS/RCSD）、长时程决策再验证（ATR）、长上下文并行阅读（PARSER）与忠实引用（ReCite）。
> 本日入选 **15 篇**（GUI 5 + Agent 方法论 10）并完成精读解读；GitHub awesome 两列表近 7 天无新增（ZJU 停 07-28、OSU 停 08-12）。
> 标签图例：`类型标签`（论文类别）｜ `训练方法标签`（方法级）｜ 域标注：GUI / Agent 方法论·跨域参考

---

## GUI 域（本领域，5 篇）

### SKC：用神经元级梯度手术让 GUI Agent 在应用流上"学新不忘旧"

- 标签：`RL` `Online` `Desktop` ｜ 训练方法：`Online RL`、`RL (GRPO)`、神经元级梯度手术（梯度截断 + 正交子空间投影）
- 域标注：**GUI**
- 一句话结论：把持续学习中的历史知识切成"该保护的专属知识"与"该继续演化的共享知识"，对前者做梯度截断、对后者做正交投影，在 OSWorld 八阶段应用流上把整体成功率平均抬高 6 个百分点以上，且不显著牺牲旧应用表现。
- [arXiv](https://arxiv.org/abs/2609.06530) ｜ 代码：⚠️ 项目主页 [SKC](https://shzirui.github.io/SKC/)（未声明代码仓库；训练基于开源 DART-GUI 基础设施）
- **[阅读完整解读](./2026-09-06_Selective_Knowledge_Control_for_Continual_GUI_Agent_Learning_over_Application_Streams.md)**

### Typed Federated Artifacts：让冻结异构 Agent 共享"该调哪个工具"的知识

- 标签：`Web` `Benchmark` ｜ 训练方法：`—（工程/评测，无模型训练）`；对比基线含 TF-IDF+SVM 分类器，方法本身推理时用冻结 LLM 打分
- 域标注：**GUI**
- 一句话结论：提出"类型化联邦构件"作为 agent 网络交换单元，让冻结且异构的 LLM Agent 共享工具路由经验；消融显示收益主要来自**类型化格式**而非联邦聚合本身，并给出两条关于 Agent 评测的负面发现（直接点名"评测协议设计"是当前薄弱环节）。
- [arXiv](https://arxiv.org/abs/2609.06815) ｜ 代码：⚠️ 部分开源（匿名评审仓库）[FederatedRAG](https://anonymous.4open.science/r/FederatedRAG-EE50/README.md)
- **[阅读完整解读](./2026-09-06_Typed_Federated_Artifacts_for_the_Agentic_WebSharing_Tool-Routing_Knowledge_Across_FrozenHeterogeneous_LLM_Agents.md)**

### Agentic Visual Generation：用"因果影响范围"给视觉生成智能体分级

- 标签：`General` `Planning` `Reflection` `Benchmark` ｜ 训练方法：`—（综述/评测/工程）`
- 域标注：**GUI**
- 一句话结论：综述性工作——以"控制器能改变多晚的生成决策"为唯一判据提出 L0–L4 层级框架，把视觉生成智能体按因果影响范围分类，并配套层级条件化评估协议与训练四阶段分析；对"多模态生成工具如何接入 GUI/Agent 工作流"提供了统一的定位语言。
- [arXiv](https://arxiv.org/abs/2609.06758) ｜ 代码：✅ [Awesome-agentic-visual-generation-model](https://github.com/YinmingHuang/Awesome-agentic-visual-generation-model)（含结构化语料 CSV/JSON）
- **[阅读完整解读](./2026-09-06_Agentic_Visual_Generation_From_Generative_Models_to_Agentic_Control.md)**

### SafeMem：用长期图记忆补上"离开视野的危险物"盲区

- 标签：`Planning` `Reflection` `General` ｜ 训练方法：`—（无模型训练，基于 GPT-4o 提示 + 图记忆推理）`
- 域标注：**GUI**
- 一句话结论：把"见过的危险物/约束"写成持续更新的语义图记忆，让具身智能体在部分可观测条件下也能规避已移出视野的隐患，安全成功率显著提升——对 GUI Agent 的"不可逆操作前检查历史上下文"有直接类比价值。
- [arXiv](https://arxiv.org/abs/2609.08444) ｜ 代码：❌ 未在文中声明开源代码（项目主页 [SafeMem](https://sites.google.com/view/safemem)）
- **[阅读完整解读](./2026-09-08_Safe_Task_Planning_with_Long-Term_Graph_Memory_for_Embodied_Agents.md)**

### SciFigure2Code：把已发表科研图"逆向重建"成可编辑代码的评测

- 标签：`Benchmark` `General` ｜ 训练方法：`—（综述/评测/工程：基准构建与零样本评测，无模型训练）`
- 域标注：**GUI**
- 一句话结论：用 Codex 智能体把已发表论文里的科研图逆向重建为可编辑 Python 程序，构建 6740 篇图集 + 337 图测试基准，系统评测 14 个模型的"图表呈现复现"能力——属于"以 Agent 造评测数据"这一正在升温的范式。
- [arXiv](https://arxiv.org/abs/2609.08155) ｜ 代码：❌ 未在文中声明开源
- **[阅读完整解读](./2026-09-08_SciFigure2Code_An_AI-Reconstructed_Benchmark_for_Scientific_Figure-to-Code.md)**

---

## Agent 方法论域（跨域参考，10 篇）

### SE-GoS：不改技能内容，只"重连"技能图就把检索奖励从 52.4% 提到 59.4%

- 标签：`Planning` `Reflection` `General` ｜ 训练方法：`—（无训练，training-free：离线图结构演化 + 执行反馈）`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：不训模型、不改技能文本，仅用执行轨迹离线"重连"技能之间的图结构，一次演化即让技能检索奖励从 52.4% 升到 59.4%、token 省约三分之一——与 GUI 技能库路线天然契合。
- [arXiv](https://arxiv.org/abs/2609.08228) ｜ 代码：❌ 未在文中声明开源
- **[阅读完整解读](./2026-09-08_SE-GoS_Self-Evolving_Graph-of-Skills_for_Skill_Library_at_Scale.md)**

### NeoHorse-1：把"路由 harness"当自我改进的传感器

- 标签：`SFT` `Distillation` `Planning` `General` ｜ 训练方法：`SFT`（三阶段路由课程）、`Distillation`（Routing-Guided On-Policy Distillation，反向 KL）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：用路由信号组织"三阶段课程 SFT + 在线策略蒸馏"，让模型在反复使用中递归变强；把"该走哪条推理路径"的失败信号显式转化为训练数据，对 GUI Agent 的"工具/动作路由 + 后训练"很有借鉴价值。
- [arXiv](https://arxiv.org/abs/2609.08183) ｜ 代码：✅ [NeoHorse](https://github.com/TokenRhythm/NeoHorse)（模型合集 [HF](https://hf.co/collections/TokenRhythm/neohorse-1)）
- **[阅读完整解读](./2026-09-08_NeoHorse-1_Towards_Recursive_Self-Improvement_via_Agentic_Post-Training_with_Routing_Harness.md)**

### CreaMem：按"生活场景"切分记忆，时间线与特质互补

- 标签：`General` `Reflection` `Benchmark` ｜ 训练方法：`—（工程/系统架构，无模型训练）`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：把同一段经历同时存成"时间线事件"与"场景特质"两条通道，检索时互补，多跳推理提升明显——对应到 GUI Agent 就是"跨 App 任务历史 + 用户操作习惯画像"的双通道记忆。
- [arXiv](https://arxiv.org/abs/2609.08550) ｜ 代码：⚠️ 文中声明已开源但未给出 URL（待确认）
- **[阅读完整解读](./2026-09-08_CreaMem_A_Scene-Aware_Memory_Architecture_for_Personalized_Agents.md)**

### TrajectoryDB：把"Agent 轨迹"当成一种独立数据类型

- 标签：`General`（数据库/系统工程愿景）｜ 训练方法：`—（综述/评测/工程）`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：提出从摄取、存储到查询三层协同设计"轨迹数据库"，而不是继续把轨迹散落在 JSONL、向量库和日志里——对 GUI Agent 的海量截图-动作轨迹管理与训练数据治理有直接基础设施意义。
- [arXiv](https://arxiv.org/abs/2609.07782) ｜ 代码：❌ 未在文中声明开源（artifact 栏为占位符）
- **[阅读完整解读](./2026-09-07_TrajectoryDB_A_New_Database_for_Agent_Trajectories.md)**

### Hi-FLoop：多时间尺度状态反馈闭环的世界模型

- 标签：`Planning` `Online` `General` ｜ 训练方法：监督式闭环生成训练（生成前缀写回历史 + TBPTT 截断反传；正文明确不使用 RL）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：用"8 秒世界分支 + 2 秒交互预演 + 1 秒物理控制"的三层状态反馈闭环，让多智能体仿真在长时程生成中保持跨尺度一致——迁移到 GUI 场景即"粗粒度页面级预测 + 细粒度控件级反馈"的分层环境模型。
- [arXiv](https://arxiv.org/abs/2609.08796) ｜ 代码：❌ 未在文中声明开源
- **[阅读完整解读](./2026-09-08_Hi-FLoop_Hierarchical_State-Feedback_Loops_for_Multi-Timescale_World_Modeling.md)**

### PspMAS：多智能体仿真中"动作趋同"的成因与解法

- 标签：`General` `Reflection` ｜ 训练方法：`—（无模型训练：Qwen3 系列推理 + 确定性规则传播）`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：揭示 LLM 多智能体仿真中"周期性语义压缩导致异质性衰减（动作趋同）"的现象，用"前景状态 + 语义状态"双分支解耦保住异质性并实现可扩展——对多 GUI Agent 协作时的"角色分工塌缩"问题有预警意义。
- [arXiv](https://arxiv.org/abs/2609.08033) ｜ 代码：❌ 未在文中声明开源
- **[阅读完整解读](./2026-09-07_Scaling_Multi-Agent_Systems_with_Prospect-State_Propagation.md)**

### ATR：区分"版本冲突"与"决策冲突"，只重验受影响的决策前提

- 标签：`General` `Planning` ｜ 训练方法：`—（工程原型，无模型训练；评测为确定性可执行规则）`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：让长时运行 Agent 在状态变化后**只**重验真正受影响的决策前提，既保住无害决策又拦住失效动作——GUI 长程任务中"页面/数据变了，之前定的操作还成立吗"正是同一问题。
- [arXiv](https://arxiv.org/abs/2609.08015) ｜ 代码：✅ [atr-decision-validation](https://github.com/ezreal13/atr-decision-validation)
- **[阅读完整解读](./2026-09-07_From_Version_Conflicts_to_Decision_Conflicts_Selective_Revalidation_for_Long-Running_AI_Agents.md)**

### RCSD：规划前先用"可达性证书"把多智能体拆成子队

- 标签：`RL` `Planning` `General` ｜ 训练方法：`—（理论方法／精确动态规划，无训练）`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：在规划之前就按通信容量把多智能体拆成固定子队，并给出"跨队奖励被删除后价值损失的上界"——为多 GUI Agent 协作提供了可证明的分工边界判据。
- [arXiv](https://arxiv.org/abs/2609.08366) ｜ 代码：未在文中声明开源
- **[阅读完整解读](./2026-09-08_Reachability-Certified_Subteam_Decomposition_for_Locally_Interacting_Multi-Agent_MDPs.md)**

### ReCite：把"引用推荐"重构成主张级推理任务

- 标签：`Reflection` `Planning` `SFT` ｜ 训练方法：`SFT`、`RL (GRPO)`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：用"定位—意图规划—反思校验"的闭环 Agent，解决"引用真实论文但逻辑上不支持论点"的错配——其"主张级证据校验"思路可直接迁移为 GUI Agent 的操作证据链与反思校验。
- [arXiv](https://arxiv.org/abs/2609.09156) ｜ 代码：⚠️ 项目主页 [ReCite](https://hyy279.github.io/ReCite/)（未声明 GitHub 地址）
- **[阅读完整解读](./2026-09-08_ReCite_Agentic_Reasoning_for_Faithful_Citation.md)**

### PARSER：把"顺序读、边读边压"改成"并行读、按需深挖"

- 标签：`RL` `Planning` `General` ｜ 训练方法：`RL (GRPO)`、`Online RL`（RLVR，仅训练 lead agent，子 Agent 冻结）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：一群轻量子 Agent 并行扫完全文，一个主 Agent 用多轮 scatter–gather 逐步深挖，只训练主 Agent——对 GUI Agent 处理长截图/长历史（先并行粗读、再按需放大细看）是极自然的架构借鉴。
- [arXiv](https://arxiv.org/abs/2609.06702) ｜ 代码：⚠️ 项目主页 [PARSER](https://cuhk-parser.github.io/)（未声明具体仓库）
- **[阅读完整解读](./2026-09-06_PARSER_Read_in_Parallel_Reason_in_Depth_for_Long-Context_LLM_Agents.md)**

---

## 本日趋势归纳

**按标签统计**（每篇 1~4 标签）：`General`×12 ｜ `Planning`×9 ｜ `Reflection`×6 ｜ `Benchmark`×4 ｜ `RL`×3 ｜ `Online`×2 ｜ `SFT`×2 ｜ `Distillation`×1 ｜ `Web`×1 ｜ `Desktop`×1 ｜ `Mobile`×0 ｜ `GUI Grounding`×0
**按域分布**：GUI 5 篇（持续学习 1、联邦工具路由 1、生成智能体综述 1、图记忆安全规划 1、评测基准 1）｜ Agent 方法论 10 篇（技能/记忆资产化 4、多智能体规模化与安全 3、长时程与世界模型 2、长上下文 1）

**几个值得注意的方向信号**：
1. **今日无新公告组，处理的是 Wed 9 Sep 组的存量 backlog**：arXiv 6 分类 listing 最新组仍为 Wed, 9 Sep 2026，API `--days 3/4` 命中的 published 也仅到 09-08。说明"当日新增"必须以 listing 公告组为准；同时暴露流水线在"单组体量 > 15 篇"时会产生跨日 backlog，本日已按规则优先消化 GUI 域。
2. **GUI 域主线回到"持续学习与知识保护"**：SKC 用神经元级梯度手术区分"该保护/该演化"的知识，直击 GUI Agent 多应用流上的灾难性遗忘——这与此前 GUI 域"评测自动化"热潮形成互补，说明该领域正从"造环境/造评测"走向"训练方法本身"。
3. **"经验资产化"仍是 Agent 域最热主题**：SE-GoS（技能图自演化）、NeoHorse-1（路由信号驱动的自改进后训练）、CreaMem（场景化记忆）、TrajectoryDB（轨迹数据库）四篇同向——共同指向"把轨迹/技能/记忆当可优化对象、甚至先冻结权重"的范式，成本低、可跨模型迁移，与 GUI 技能库路线高度契合。
4. **多智能体"规模化"开始被拆成可分析的问题**：PspMAS 指出语义压缩导致动作趋同（异质性衰减），RCSD 用可达性证书给出子队分解的价值上界——一个讲现象、一个给理论边界，对"多 GUI Agent 分工协作"是少见的组合。
5. **长时程状态管理成为跨域公共难题**：ATR（决策前提再验证）、Hi-FLoop（多尺度状态反馈）、PARSER（并行读取 + 按需深挖）分别从"状态失效检测""环境动态预测""上下文经济"三个角度切入，恰好对应 GUI 长程任务的三个痛点。
6. **开源率偏低**：15 篇中仅 3 篇给出明确代码仓库（NeoHorse-1、ATR、Agentic Visual Generation），4 篇仅有项目主页/匿名仓库，8 篇未声明——复现门槛需注意。
