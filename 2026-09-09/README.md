# 📖 当日论文速览 · 2026-09-09

> 处理日：2026-09-09（周三）｜ 对应 arXiv 最新公告组：**Wed, 9 Sep 2026**（介于上一组 Mon, 7 Sep 2026 与本次之间无 Tue 组；listing 6 分类该组合计约 1449 条）
> 主题一句话：**"评测自动化"信号明确**——GUI 域 3 篇中 2 篇直击评测可复现/自动构建（APPSim-Bench 把真实 App 搬进可控沙盒、FinCUABuild 让 agent 自主造金融 CUA 评测任务），1 篇用自生成工具动作增强 GUI 执行（DroidTool）；Agent 方法论域双线并行——RL 训练"配方级"实证（SRPO 集合动作 RL、Elastic Horizon 自适应交互预算、Env-Scaffold 环境反馈加料、SVRL RL 自验证）与"经验→可复用资产"的自演化/对齐（SkillAdam、Procedural Graphs、Co-Evolving Harnesses、SkillAlign），另收长轨迹上下文工程（AttnCompress/MEMO）、风险感知评测（AURA-Eval）与 test-time 世界模型适应（WorldAgen）。
> 本日入选 **15 篇**（GUI 3 + Agent 方法论 12）并完成精读解读；GitHub awesome 两列表近 7 天无新增。
> 标签图例：`类型标签`（论文类别）｜ `训练方法标签`（方法级）｜ 域标注：GUI / Agent 方法论·跨域参考

---

## GUI 域（本领域，3 篇）

### APPSim-Bench：弥合"真实 vs 可复现"的移动 GUI 智能体评测

- 标签：`Benchmark` `Mobile` ｜ 训练方法：—（评测/Benchmark，未训练模型）
- 域标注：**GUI**
- 一句话结论：以"人给规格 + 编码 Agent 生成 + 人工校验"流水线把 557 个真实高频任务搬进 17 个可控模拟 App（中英生态），用确定性结果验证替代路径比对，19 个 GUI agent 横评天花板仅 50.27%（Claude-Opus-4.7），28.55% 任务（159/557）无任何模型解出。
- [arXiv](https://arxiv.org/abs/2609.07712) ｜ 代码：✅ [AppSim](https://github.com/Acrab-Agentic-Labs/AppSim)
- **[阅读完整解读](./2026-09-07_APPSim-Bench_Bridging_Real-world_Apps_and_Reproducible_Evaluation_for_Mobile_GUI_Agents.md)**

### FinCUABuild：让 Agent 自动构建动态金融 Computer-Use 评测任务

- 标签：`Benchmark` `Web` `Desktop` ｜ 训练方法：—（评测/Benchmark；FinCUABuildAgent 为基于现有 LLM 的多智能体系统）
- 域标注：**GUI**
- 一句话结论：首次为"AI 能否自主构造多样化金融 CUA 评测任务"建基准——FinCUABuildBench 含 576 个构造请求/24 类金融工作流 × 3 类运行时变化，配执行测试 + 质量检查的任务认证机制；三模块 FinCUABuildAgent 同骨干下任务质量 Qual 达 31.3%（通用基线仅 1.3–8.3%），换骨干最高 41.9%。
- [arXiv](https://arxiv.org/abs/2609.07603) ｜ 代码：✅ [FinCUABuild](https://github.com/FengxianJi/FinCUABuild)
- **[阅读完整解读](./2026-09-07_FinCUABuild_Can_Agents_Build_Reliable_Benchmarks_for_Dynamic_Financial_Computer_Use.md)**

### DroidTool：Android GUI Agent 的自生成工具动作（混合动作空间）

- 标签：`Mobile` `GUI Grounding` ｜ 训练方法：—（未训练；商用模型 Gemini-3.5-flash 提示 + agentic 工具生成工作流）
- 域标注：**GUI**
- 一句话结论：让 Android agent 用"提案→实现→测试→修复"工作流自主生成直接读写 App 状态的 Python 工具（含跨工具关系化测试验证），把 GUI 点击与工具动作拼成混合动作空间；AndroidWorld/B-MoCA/MobileSafetyBench 平均成功率 +4.47pp（79.20 vs 74.73）、交互步数 −20.05%（KAIST）。
- [arXiv](https://arxiv.org/abs/2609.06792) ｜ 代码：未开源
- **[阅读完整解读](./2026-09-06_Improving_Proficiency_and_Efficiency_of_Android_GUI_Agents_via_Self-Generating_Tool_Actions.md)**

---

## Agent 方法论域（跨域参考，12 篇）

### SRPO：把"共同引发一次迁移的多个输出"当集合动作做 RL

- 标签：`RL` `General` ｜ 训练方法：`RL (GRPO 系，setwise 集合式相对策略优化)`（组间相对优势 + 事件级裁剪，在线 RL 后训练）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：针对多智能体 LLM 中"多个响应共同造成一次状态迁移、却各自独立优化"的错位，把 active set 当单个集合动作做基数归一化 + 单次裁剪的相对策略优化，一套训练接口统一分工/共演/动态路由（东南大学+快手）；Math macro Avg@16 61.3/62.5（4B/8B）、多轮搜索 Pass@16 56.1/61.6（3B/7B，较最强基线 +3.1/+3.3）。
- [arXiv](https://arxiv.org/abs/2609.08452) ｜ 代码：未在文中声明开源
- **[阅读完整解读](./2026-09-08_SRPO_Setwise_Relative_Policy_Optimization_for_Multi-Agent_LLMs.md)**

### Procedural Graphs：把"下一步该做什么"做成可自演化的执行结构图

- 标签：`Planning` `Online` ｜ 训练方法：—（工程/方法框架，无模型权重更新；推理期图引导 + 离线 LLM 图自演化与验证门控）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：仿知识图谱把程序性知识组织成 (procedure, relation, procedure) 有向属性图，在线按活跃节点取局部子图生成情境化引导、离线对比成败轨迹自演化拓扑与属性并"拒绝记忆"坏编辑；24 模型×基准组合中 21 个取得最优，EnterpriseArena 长程存活率 6%→34%/0%→85%，工具调用 17.23→3.08 次/月（−81.8%）。
- [arXiv](https://arxiv.org/abs/2609.09153) ｜ 代码：未在文中声明开源
- **[阅读完整解读](./2026-09-08_Procedural_Graphs_Self-Evolving_Execution_Structures_for_LLM_Agents.md)**

### Co-Evolving Harnesses and Models：整轨模仿为何让弱模型在演化 harness 下全面倒退

- 标签：`Online` `SFT` `Reflection` ｜ 训练方法：`LoRA-SFT`（专家轨迹模仿 + on-policy 定点纠错两类数据）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：harness（提示词/工具集/脚手架）演化 + 模型微调需协同——把专家在演化 harness 下的完整轨迹整轨模仿给弱模型，7 个企业任务全部倒退（均值 −14.9 分，规划类失败 1.1%→14.6%），破坏"模型—harness 契合"；改用 on-policy 专家定点纠错保住契合并继续提升（+1.7，Salesforce AI）。
- [arXiv](https://arxiv.org/abs/2609.09134) ｜ 代码：未在文中声明开源
- **[阅读完整解读](./2026-09-08_Co-Evolving_Harnesses_and_Models_On-Policy_Correction_Helps_Weaker_Models_Catch_Up_Where_Imitation_Fails.md)**

### SkillAdam：把 Adam 的一、二阶动量搬进技能文档自演化

- 标签：`General` `Online` ｜ 训练方法：—（无模型参数训练：目标 LLM 冻结，Adam 式外部技能文档迭代自演化）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：用"问题记忆"（Adam 一阶矩类比）稳定修订方向、用"波动率驱动的编辑预算"（二阶矩类比）自适应控制改多少，让技能自演化更稳更省（中国人民大学+腾讯）；DP-Avg 28.3 vs SkillOpt 21.7，总 token −67.3%、API 请求 −68.8%，跨模型保留率 81.3% vs 76.2%。
- [arXiv](https://arxiv.org/abs/2609.08944) ｜ 代码：✅ [SkillAdam](https://github.com/ruc-datalab/SkillAdam)
- **[阅读完整解读](./2026-09-08_SkillAdam_Stable_and_Efficient_Skill_Evolution_for_Agents.md)**

### Environments as Scaffold：与其热身 agent，不如给环境反馈"加料"

- 标签：`RL` `Online` `General` ｜ 训练方法：`RL (GRPO / DAPO / GSPO)`（Qwen3-4B/8B 标准 agentic RL，未用 SFT warm-up）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：把缓解稀疏奖励的重心从 agent 侧 SFT 热身移到环境侧——按"早期注入动作指引（AG）、后期切到观测富化（OE）"的时序构造 Feedback-Enriched Environment，跨 SciWorld/BFCL 与三种 RL 算法一致提升（平均 +2.82%，BFCL-Base 单点 +10pp，8B+GSPO SciWorld 53.91→60.94）。
- [arXiv](https://arxiv.org/abs/2609.08404) ｜ 代码：✅ [EnvAsScaffold](https://github.com/HongbangYuan/EnvAsScaffold)
- **[阅读完整解读](./2026-09-08_Environments_as_Scaffold_Enriching_Feedback_to_Bootstrap_Self-Evolving_Agents_in_Long-Horizon_Tasks.md)**

### AttnCompress：注意力引导的软件工程 Agent 长轨迹动态压缩

- 标签：`General` ｜ 训练方法：—（免训练中间件/工程方法）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：以 PPL 尖峰做结构感知分块、以代理注意力打分衡量历史块与当前任务的关联、滚动保留"保近端 + 压远端"压缩长轨迹（北京大学，ISSTA 2026）；平均 Pass 53.17% 超 SOTA AgentDiet 51.17%，输入 token −21.6%、单实例成本 $0.0949（−33.6% vs AgentDiet）。
- [arXiv](https://arxiv.org/abs/2609.08318) ｜ 代码：✅ [AttnCompress](https://github.com/ZZR0/AttnCompress)
- **[阅读完整解读](./2026-09-08_AttnCompress_Dynamic_Attention-Guided_Trajectory_Compression_for_Software_Engineering_Agents.md)**

### WorldAgen：世界模型 + 动作预测的共享骨干与 Test-Time 在线适应

- 标签：`Online` `General` ｜ 训练方法：`SFT`（联合 state/action 预训练）＋ `LoRA`（Test-Time Training，在线自监督微调）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：VLA 用共享 Transformer + 混合单向注意力掩码同时学"世界模型头（预测未来状态）"与"动作头"，部署后用少量探索轨迹做仅观测损失驱动的 LoRA-TTT 就地适应新环境（Northwestern，AAAI 2026）；CALVIN ASL 3.87→3.93（超 Seer-Large 3.83）、LIBERO 75.5%→79.0%，TTT 单张 4090 仅 2–8 分钟。
- [arXiv](https://arxiv.org/abs/2609.08162) ｜ 代码：未声明开源（项目主页 [worldagen.github.io](https://worldagen.github.io)）
- **[阅读完整解读](./2026-09-08_WorldAgen_Unified_State-Action_Prediction_with_Test-Time_World_Model_Training.md)**

### SVRL：用 RL 在多模态 agent 的轨迹内"教"出证据自验证

- 标签：`RL` `Reflection` ｜ 训练方法：`RL (GRPO)`（Dr.GRPO 变体做 on-policy 更新）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：纯 RL 微调让多模态 agent 学会在自身推理轨迹内验证并过滤检索证据（替代外部 verifier），配 search-aware 惩罚抑制多余搜索与 query-diversity 奖励（UW+Amazon，ECCV 2026）；Qwen2.5-VL-7B 仅 5000 样本，FVQA 65.3% vs 最强基线 56.4%，搜索率 61.4% vs 80.3%，逼近"始终检索"的 GPT-4o（66.0%）。
- [arXiv](https://arxiv.org/abs/2609.08025) ｜ 代码：未声明开源（项目主页 [svrl](https://vishwassathish.github.io/projects/svrl/)）
- **[阅读完整解读](./2026-09-07_Eliciting_Self-Verification_in_Multimodal_Reasoning_Agents_with_Reinforcement_Learning.md)**

### SkillAlign：技能的"暴露接口"与任务成败强相关

- 标签：`General` `Planning` ｜ 训练方法：`SFT + DPO (LoRA)`（仅用于探索性的暴露接口选择器，非主体方法）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：提出"技能接口对齐"——同一批技能以全文档/提示/压缩摘要/工作流/不暴露等接口呈现，agent 表现差异巨大（ALFWorld 同候选集仅换接口 47.9→72.1），紧凑 top-k 暴露常优于全量注入（全量注入 1000 条成本约 900K token 且低于无技能基线；中科院自动化所等，EMNLP 2026）。
- [arXiv](https://arxiv.org/abs/2609.07255) ｜ 代码：未开源
- **[阅读完整解读](./2026-09-07_SkillAlign_Aligning_Skill_Interfaces_for_LLM-based_Agents.md)**

### MEMO：多模态证据记忆组织——给长程 agent 造"查询定制"的工作记忆

- 标签：`General` ｜ 训练方法：`SFT`（证据提取器与记忆管理器基于 Qwen2.5-1.5B-Instruct 监督微调）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：把历史记忆切成带出处/跨度的"证据单元"，由训练过的管理器在统一 token 预算下为每单元分配文本/图像/双通道载体并排版（上交/清华/自动化所）；128-token 严格预算下多 reader 全面第一，Qwen3-VL-32B 上 2Wiki F1 73.91（纯文本 56.26/纯图像 35.89），EM 超最强基线 +2.35。
- [arXiv](https://arxiv.org/abs/2609.07471) ｜ 代码：未开源
- **[阅读完整解读](./2026-09-07_MEMO_Multimodal_Evidence_Memory_Organization_for_Long-Horizon_LLM_Agents.md)**

### Elastic Horizon：闭环发现 agentic RL 的"有效交互前沿"

- 标签：`RL` `Online` `General` ｜ 训练方法：`RL (GRPO)`（Qwen2.5-7B/14B，KL=0，创新点是训练超参——每集交互预算的闭环调度）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：提出"有效交互前沿"假设——预算超过边界后收益递减而成本线性增长，用成功轨迹长度的 P90 做闭环控制器自动扩张/收缩每集交互预算（阿里云+中科大，EMNLP 2026）；从欠配/过配双初始化都能收敛进饱和区，14B AppWorld mean@8 77.85%（best@8 90.42%）全场最优，每步 token 最多省 25%。
- [arXiv](https://arxiv.org/abs/2609.07247) ｜ 代码：✅ [ElasticHorizon](https://github.com/junjie-meng/ElasticHorizon)
- **[阅读完整解读](./2026-09-07_Elastic_Horizon_Discovering_the_Effective_Interaction_Frontier_in_Agentic_Reinforcement_Learning.md)**

### AURA-Eval：受控增广 + 过程级诊断的 agent 风险意识评测

- 标签：`Benchmark` `Reflection` ｜ 训练方法：—（评测/框架）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：把 157 条真实工具轨迹按 6 种风险机制/5 种难度改写为 1,249 条"有（SP）/无（NSP）安全完成路径"成对评测项，分轴诊断"是否察觉风险/采取何种行动/行动是否安全"；结果显示识别≠行动——NSP 下前沿不安全执行率 26.8–40.0%，开源模型飙到 57.9–82.6%。
- [arXiv](https://arxiv.org/abs/2609.06783) ｜ 代码：未开源
- **[阅读完整解读](./2026-09-06_AURA-Eval_Evaluation_Framework_for_Acting_Under_Risk_Awareness_in_LLM_Agent_Trajectories.md)**

---

## 本日趋势归纳

**按标签统计**（每篇 1~4 标签）：`General`×8 ｜ `Online`×6 ｜ `RL`×4 ｜ `Benchmark`×3 ｜ `Reflection`×3 ｜ `Planning`×2 ｜ `Mobile`×2 ｜ `SFT`×1 ｜ `GUI Grounding`×1 ｜ `Web`×1 ｜ `Desktop`×1 ｜ `Distillation`×0
**按域分布**：GUI 3 篇（2 Benchmark/评测 + 1 Mobile 混合动作工具增强）｜ Agent 方法论 12 篇（RL 训练实证 4、经验资产化 4、上下文/记忆工程 2、评测 1、test-time 世界模型 1）

**几个值得注意的方向信号**：
1. **"评测自身的自动化与可复现"成为 GUI 域当日主调**：APPSim-Bench 用模拟 App 沙盒把真实商业 App 级任务变成确定性可复现评测；FinCUABuild 更进一步——让 agent 自己构造动态 CUA 评测任务并做质量认证。配合此前 CUA-Universe（环境→数据流水线）与 ElderBench（真实用户表达），"造环境/造评测"正在成为 GUI Agent 研究的基础设施层。
2. **RL 训练进入"配方级"实证阶段**：SRPO（把 GRPO 相对优势推广到集合动作/多智能体）、Elastic Horizon（把"每集交互预算"变成可调度超参）、Env-Scaffold（把稀疏奖励改造移到环境侧）都在回答"同一套 RL 下哪些训练细节决定成败"，对 GUI RL 的环境/预算/奖励设计直接可借鉴。
3. **"经验资产化"持续发烧且转向"文档级优化"**：SkillAdam（Adam 式技能文档自演化）、Procedural Graphs（程序图自演化）、SkillAlign（技能暴露接口对齐）、Co-Evolving（harness 与权重协同）——共同指向"把轨迹/技能当可优化对象、模型权重反而冻结"的范式，成本低、可跨模型迁移，与 GUI 技能库路线天然契合。
4. **长轨迹的 token/记忆经济**：AttnCompress（轨迹压缩）与 MEMO（预算约束下的多模态记忆组织）双管齐下控上下文成本——GUI agent 轨迹长、截图多，这两类工程最贴近落地。
5. **安全/风险感知评测入场**：AURA-Eval 证明"能识别风险"与"会采取安全行动"是两回事（无安全路径时开源模型不安全率高达 82.6%）——GUI agent 面对支付、删除等不可逆操作时同样需要"干预型安全"而非只是"识别型安全"。
