# Act More, Decide Less: Skill-Guided Adaptive Action Chunking for Long-Horizon LLM Agents（少决策、多执行：面向长程 LLM Agent 的技能引导自适应动作分块）

> TL;DR 本文提出 SPACE 框架：从成功轨迹中归纳"程序化技能"，把技能的层级边界蒸馏成"动作分块边界"的监督信号，训练 LLM agent 学会每轮输出 1~6 个原始动作的变长动作块。在 ALFWorld 与 ScienceWorld 上相对最强基线成功率提升 7.0%~31.3%，同时每轮 LLM 决策最高减少 78.9%。

---

## 论文档案（Metadata）

| 字段 | 内容 |
| --- | --- |
| 论文标题 | Act More, Decide Less: Skill-Guided Adaptive Action Chunking for Long-Horizon LLM Agents |
| arXiv ID / DOI | 2609.02042 / 10.48550/arXiv.2609.02042 |
| arXiv 链接 | https://arxiv.org/abs/2609.02042 |
| 发表出处（Venue） | EMNLP 2026（Camera Ready；2026-09-02 以 arXiv v1 预印本形式公开，arXiv 页面 Comments 标注 "EMNLP 2026 Camera Ready"） |
| 发布时间 | 2026-09-02 |
| 作者 | Yanting Yang\*、Can Jin\*、Jinman Zhao、Jiahao Wu、Yang Zhou、Zhepeng Wang、Zhendong Wang、Mu Zhou、Dimitris N. Metaxas（\*共同一作） |
| 所属机构（含国家） | 罗格斯大学 Rutgers University（美国）、多伦多大学 University of Toronto（加拿大）、香港理工大学 The Hong Kong Polytechnic University（中国香港）、Amazon（美国）、Microsoft（美国） |
| 开源情况 / 代码 | ❌ 未在文中声明开源（arXiv 页面亦无代码/模型链接） |
| 类型标签 | RL、Online、Distillation、Planning |
| 训练方法标签 | Hybrid On-/Off-Policy RL（RLVR 框架；on-policy 采用 PPO 风格 clipped 目标，off-policy 采用自模仿蒸馏损失，配合同组归一化的两层 chunk-aware advantage 信用分配；基座 Qwen3-4B / Llama-3.1-8B-Instruct，无 thinking 模式） |
| 关键词 | Action Chunking、变长动作块、程序化技能蒸馏、chunk 边界学习、长程 LLM Agent、RLVR |
| 来源渠道 | arxiv-api |
| PDF 存档 | 2026-09-02_Act_More_Decide_Less_Skill-Guided_Adaptive_Action_Chunking_for_Long-Horizon_LLM_Agents.pdf |

---

## 1. 研究背景与要解决的问题

主流 LLM agent 遵循 ReAct 式协议：每轮 LLM 调用只推理并执行**一个**原始动作，拿到环境反馈后再想下一步。这种"一步一决策"的好处是频繁重规划、稳健，但在长程任务里代价高昂：大量轮次耗费在"找番茄→走到冰箱→开门→放进去"这类例行动作序列上，LLM 其实并没有在做新决策，只是被反复唤醒。已有研究也指出，细粒度交互会带来短视行为并放大复合误差。

一个自然的改进是让 agent 一次发出**变长动作序列（action chunk）**，即"一次决策、批量执行"。作者用标准 RL 目标（如 GRPO）直接训练这种变长策略，结果出现两种典型失败：要么策略"塌缩"回单动作行为，要么"过度承诺"输出过长的块反而损害成功率。两病同源——**在只有稀疏终端奖励（成/败二值或任务分）的环境里，RL 信号不携带"chunk 边界应该划在哪"的任何信息**。核心难题因此不是"能不能输出多个动作"，而是**学习 chunk 边界**。

## 2. 核心方法 / 思路

作者的洞察是：成功轨迹里本就藏着边界结构——复杂 agent 行为可分解为带参数的**程序化技能**（可执行 Python 代码，比自由文本更结构化、可复用、可校验）。SPACE 的核心是"让技能当训练老师、边界被蒸馏成策略能力"。

四步流程（见图 2）：

1. **两级技能归纳**。从成功轨迹 τ 出发，提示 LLM 把轨迹切分成子任务阶段，生成一个"复合技能"（composite skill，任务级主函数）+ 一串有序"子技能"调用（subskill，局部例程，如 find object、heat object）。子技能边界即天然的动作块边界。入库前经语法/可编译性检查，并按 AST 规范化去重。冷启动只需少量人工技能（ALFWorld 3 个、ScienceWorld 5 个）。

2. **混合 rollout**。训练时两种模式交替（原始块 rollout 与技能增强 rollout）：原始块模式直接按部署形态输出动作块构成 on-policy 数据 D_on；技能增强模式额外允许调用检索到的技能（检索采用"类别优先、UCB 打分 + 语义相似度兜底"）。

3. **技能到块展开（skill-to-chunk expansion）**。技能增强轨迹不直接进入训练数据，而是把技能调用展开回动作块：调子技能→展开成 1 个块；调复合技能→沿其子技能序列展开成 M 个带明确边界的块，构成 off-policy 数据 D_off。这样测试时策略完全不依赖技能库。

4. **块感知的 on-/off-policy 联合优化**。借鉴 GiGPO 的"两层优势"：轨迹级优势（组内归一化）+ 步级优势（按 anchor 观测分组归一化，回合级折扣 return 广播给块内每个动作），两者相加后分别用于 on-policy clipped 损失与 off-policy 自模仿蒸馏损失，完成 RLVR 微调。

测试时每轮输出 1~6 个逗号分隔的合法原始动作（最大块长 K=6），无技能调用、无额外标注。

## 3. 关键实验结果

实验环境为 ALFWorld 与 ScienceWorld 的 seen/unseen 两个划分，基座 Qwen3-4B、Llama-3.1-8B-Instruct，基线含 ReAct/Reflexion/RLOO/GRPO/GiGPO 及"多动作 GRPO"变体。

- **成功率与决策轮次**。ALFWorld 上相对最强基线提升 7.0~15.6 个百分点，每 episode LLM 轮次降至 3.7~5.2 轮，如 Qwen3-4B 在 seen 达 99.2% SR / 3.7 轮、unseen 达 96.9% / 4.4 轮，远优于最强 RL 基线 GiGPO（85.2% / 15.9 轮、72.7% / 21.7 轮）；ScienceWorld 上近乎翻倍（+27.3~31.3 个百分点，seen 67.2% vs GiGPO 35.9%），轮次约减半。
- **学出了"非退化且均衡"的动作块**。多动作 GRPO 在 Qwen3-4B 上塌缩到每轮约 1 个动作，在 Llama 上却过度膨胀到每轮 5~6 个动作且成功率大降；SPACE 稳定在每轮 3~4 个动作，策略熵也显著更高（探索更充分而不塌缩）。
- **训练更省更稳**。ALFWorld（Llama）上 SPACE 达多动作 GRPO 最终水平仅需约 40/150 训练步（26.6%），匹配 GRPO 最终性能只需 43.60K rollout 轮次；消融（Table 3）显示去技能 / 去块感知优势后成功率从 96.1% 分别掉到 86.7% / 90.6%（ALFWorld seen），印证两组件互补。
- **测试时搜索增效**。Best-of-N（N=8）在 chunk 策略上 SR 增益（52.1%→60.4%，+8.3）几乎是单动作策略（33.3%→37.5%，+4.2）的两倍，且 TTS 后每 episode LLM 调用仅 48.3 次 vs 74.9 次，说明"块策略 × 测试时缩放"天然互补。附录示例中同一任务 SPACE 仅用 3 轮 LLM 调用完成，而多动作 GRPO 用了 32 轮并出现两次"不在冰箱旁却反复 cool pan"的自循环。

## 4. 亮点与贡献（Why it matters）

第一，问题定位清晰且反直觉：多动作输出本身并不难，难的是稀疏奖励下学不会"边界在哪"，把 action chunking 从"输出格式问题"重构成"边界监督问题"。第二，用轨迹归纳的程序化技能充当**训练期老师**、测试期完全剥离——既拿到免费的子技能边界监督（省去人工标注），又避免部署时对技能库的依赖和检索开销，把技能蒸馏成了可泛化的"分块本能"。第三，方法在"成功率 × 效率 × 训练样本数"三个维度同时改善，并展示了与测试时搜索的可组合增益。对长程文本交互 agent 而言，"少决策、多执行"被证明不仅更省钱，还更准。

## 5. 局限与可改进点（个人点评，原创判断）

- **环境面窄**：只验证了文本家居与虚拟实验室两个确定性较强的文本环境，原始动作是受限的离散指令集（含 admissible actions 列表），收敛于每轮 3~4 个动作与真实开放场景差距尚远；作者自己在 Limitations 中也承认高度随机/安全关键环境里"开环批量执行"有风险——该风险恰是后续可用"边界置信度/中断条件"去缓解的方向。
- **成功轨迹依赖**：技能归纳依赖能拿到成功轨迹；初始成功率低或奖励极稀疏的任务域，技能库可能长期长不大、监督质量不足（论文也提到可能需更强探索/检索/校验机制）。
- **未声明开源**：无代码与模型权重，RLVR 复现成本高（需 4×H200/GH200、150 epochs 等），表 1 里 Llama 基座 ALFWorld seen 上 SPACE 相对 GiGPO 轮次优势不明显，暗示在"回合已被压得很短"的场景增益有限，泛化边界值得拷问。
- 可改进：把技能归纳的 LLM 调用改成小模型/规则，或引入"块级奖励塑形"与失败轨迹的负边界信号；chunk 数量与长度同 LLM 输入 cost 的关系也可做更精细的经济性建模。

## 6. 对 GUI Agent 的可借鉴点（Agent 方法论论文专节）

GUI/Web 操作 agent（计算机使用、网页点击流）的长程任务与本文设定高度同构：动辄几十上百步的"导航→输入→点击→校验"例行序列，正是 LLM 每轮只发一个 GUI 原语动作、被反复唤醒的低效来源。可迁移点有四：

1. **变长动作块直接对应 GUI 的"宏操作"**。把"打开设置→搜索→点击条目→返回"这类稳定子流程打包成一个 chunk，每轮 LLM 决策只发一个块；块与块之间才暂停等截图/DOM 反馈，天然复刻本文"少决策多执行"范式。
2. **技能归纳的迁移**：GUI 任务可把"填表单/登录/翻页下载"归纳成带参数的可执行子流程（对应本文 subskill），其调用边界即分块边界，作为 GUI 场景廉价的分块监督；建议把「文本技能代码」换成 GUI 动作 API 序列（点击 selector、输入、键盘事件）而非自然语言步骤描述。
3. **混合训练 + 蒸馏**：RLVR 稀疏成败奖励下直接训多动作 GUI 策略同样会塌缩/过度承诺（本文 Figure 1 的两种失败模式在 GUI RL 训练里非常常见），可照搬"技能增强 rollout 展开成 off-policy 蒸馏数据 + chunk-aware 优势"的做法，测试时剥离技能库、只发原始 GUI 块。
4. **务必保留"每步可中断"校验**：GUI 的 DOM/页面状态部分可观测且易变（弹窗、加载时序），随机性比文本环境更高，可扩展为"块内加隐式中断条件（元素消失/出现即停）"。另外表 4 表明"块 + Best-of-N 搜索"增益翻倍，GUI 场景可结合视觉相似度打分做候选块搜索，缓解逐步 lookahead 昂贵的问题。

## 7. 延伸阅读

- **Q-chunking（Li et al., 2025）**：在连续控制/机器人 RL 中引入动作分块，是本文"分块提高探索效率"思想的上游；**ACT（Zhao et al., 2023）**：模仿学习里的动作分块代表作，两篇共同构成 action chunking 脉络。
- **GiGPO（Feng et al., 2025）**：本文两层优势/信用分配直接借鉴的组内组外策略优化；**RLVR（DeepSeekMath, Shao et al., 2024）**：可验证奖励 RL 范式的源头。
- **ASI（Wang et al., 2025b）/ Voyager / CodeAct（Wang et al., 2024）**：把 agent 技能程序化表示/归纳为可执行代码的系列工作，是本文"程序化技能教师"概念的出处；**SkillAct（Liu et al., 2024）** 文本技能提示路线。
- **ReAct（Yao et al., 2023）**：本文"每轮单动作"基线协议；**AgentGym-RL（Xi et al., 2025）**：ScienceWorld 任务配置与多轮 RL 训练框架来源。
- 领域拓展对比：**LaMer / Meta-RL（Jiang et al., 2025）**（跨 episode 训练、探索）、**Reflexion（Shinn et al., 2023）**（反思路线）——与"分块"互补而非竞争。

---
*解读生成时间：2026-09-07 ｜ 解读人：WorkBuddy（AI）*
