# Act More, Decide Less: Skill-Guided Adaptive Action Chunking for Long-Horizon LLM Agents

> 一句话 TL;DR：SPACE 从成功轨迹归纳程序化技能，把 subskill 边界蒸馏成"动作块边界"监督，训练 LLM 每轮输出 1~6 个原始动作的变长块；成功率较最强基线提升 7.0%~31.3%，决策轮次最多降 78.9%。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Act More, Decide Less: Skill-Guided Adaptive Action Chunking for Long-Horizon LLM Agents |
| **作者 / 机构** | Yanting Yang\*、Can Jin\*（共同一作）、Jinman Zhao、Jiahao Wu、Yang Zhou、Zhepeng Wang、Zhendong Wang、Mu Zhou、Dimitris N. Metaxas；Rutgers University（美）、University of Toronto（加）、The Hong Kong Polytechnic University（中国香港）、Amazon、Microsoft |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-02（arXiv v1）；arXiv 页面标注 "EMNLP 2026 Camera Ready"（正文仅见 cs.LG 分类） |
| **arXiv 链接** | https://arxiv.org/abs/2609.02042 |
| **代码仓库** | ❌ 未开源（正文与 arXiv 页面均无代码/模型链接） |
| **数据集地址** | 使用公开 benchmark：ALFWorld（https://github.com/alfworld/alfworld）、ScienceWorld（https://github.com/allenai/ScienceWorld）；无新增私有数据 |
| **类型标签** | `RL` `Online` `Distillation` `Planning` |
| **训练方法标签** | `RLVR`（Hybrid On-/Off-Policy：on-policy 用 PPO-style clipped 目标，off-policy 用自模仿蒸馏 loss，配 chunk-aware 两层优势；基座 Qwen3-4B / Llama-3.1-8B-Instruct，无 thinking 模式） |
| **关键词** | Action Chunking、变长动作块、程序化技能蒸馏、chunk 边界学习、长程 LLM Agent、RLVR |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-02_Act_More_Decide_Less_Skill-Guided_Adaptive_Action_Chunking_for_Long-Horizon_LLM_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：长程交互任务中 LLM agent 的**动作粒度**。主流 ReAct 协议每轮 LLM 只推理并执行**一个**原始动作，拿到反馈再想下一步；本文主张 agent 应每轮输出**变长动作序列（chunk）**，即"一次决策、批量开环执行"。
- **为什么重要**：长程任务里大量轮次耗在"找→走→开→放"这类例行动作序列上，LLM 被反复唤醒却没在做新决策。细粒度交互已被指出会诱发短视（short-sighted）、放大复合误差（compounding errors）、使 agent 陷入重复循环或无效指令（Zhu et al. 2025；Xie et al. 2024）。
- **现有方法有什么不足**：① Prompting 类（ReAct/Reflexion）与 RL 类（RLOO/GRPO/GiGPO）**全在单动作范式内**优化，未触及时间粒度；② 直接扩展动作空间为变长、用标准 RL（GRPO）训练，会现两种典型失败——Qwen3-4B **塌缩回单动作**，Llama-3.1-8B **过度承诺**输出 5~6 动作/轮但成功率反降（Figure 1）。
- **Research Gap**：两病同源——**在只有稀疏终端奖励（成/败）的 RLVR 里，奖励信号不携带"chunk 边界该划在哪"的任何信息**。故核心难题不是"能不能输出多动作"，而是**学习 chunk 边界**。作者 claim：成功轨迹本就藏有边界结构（可分解为带参数的 programmatic skills），其 subskill 边界可作为免费的分块监督，把该结构蒸馏进策略即可补上缺口。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把"技能层级结构"当训练期老师，把 subskill 边界蒸馏成策略的"动作分块本能"，测试期完全剥离技能库。

### 3.2 方法总览（Pipeline）

- **输入 / 输出**：输入 = 任务描述 + 文本交互历史（含 admissible actions 列表）；输出 = 变长动作块 u=(a₁,…,a_L)，1≤L≤K=6，全为原始动作。
- **四个模块**：
  1. **两级技能归纳**：从成功轨迹 τ 提示 LLM 生成"复合技能（任务级主函数）+ 有序子技能调用（局部例程，如 find/heat object）"的可执行 Python；经语法/可编译性检查 + AST 去重入库。
  2. **混合 rollout**：交替两种模式——原始块模式（部署形态，构成 on-policy 数据 D_on）与技能增强模式（额外可调用检索技能，检索用"类别优先 + UCB 打分 + 语义相似度兜底"）。
  3. **skill-to-chunk 展开**：把技能调用展开回动作块——子技能→1 块；复合技能→沿 subskill 序列展开成 M 个带明确边界的块，构成 off-policy 数据 D_off。
  4. **chunk-aware 联合优化**：轨迹级优势（组内归一化）+ 步级优势（按 anchor 观测分组归一化，回合折扣 return 广播给块内每个动作），分别驱动 on-policy clipped loss 与 off-policy 自模仿蒸馏 loss。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：把 action chunking 从"输出格式问题"**重构为"边界监督问题"**，并用**两级程序化技能的 subskill 边界**作为唯一且免费的边界监督源——这是全文最核心的贡献点；同时把 GiGPO 的两层优势推广到 chunk 粒度（return 广播进块内动作）。
- **工程组合**：技能归纳（承接 ASI/CodeAct/Voyager 的代码化技能思想）、混合 rollout、skill-to-chunk 展开、on/off-policy 联合（clipped PPO + self-imitation learning）均是已有组件拼装，本身不新。
- **对性能提升最关键的设计**：Table 3 消融显示**技能蒸馏贡献最大**（去掉后 seen 96.1%→86.7%，掉 9.4pp），chunk-aware 优势次之（→90.6%，掉 5.5pp），二者互补、缺一不可。
- **证据不足 / 仅声称有效**：K=6 的取值、冷启动技能数量（3/5）的敏感性、检索策略（UCB/相似度）的独立收益均无定量消融；ρprim 比值只在 ScienceWorld 做了敏感性（Figure 4b），ALFWorld 未做。

---

## 4. 具体技术细节

### 4.1 模型结构

纯 **LLM（非 MLLM，无视觉编码器）**：基座为 Qwen3-4B 与 Llama-3.1-8B-Instruct（ALFWorld 二者皆用，ScienceWorld 仅 Llama）；RLVR 微调，采用 **no-thinking 模式**（关闭 chat template 的 `<think>` 块，沿用 LaMer 设定）。论文未声明 LoRA，按全参微调理解。

### 4.2 训练流程

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| Stage 1 技能归纳 | 构建/更新技能库 | 无（库更新，非梯度训练） | 当前迭代的成功轨迹 | 可执行 Python（复合技能+子技能） | 无 loss；语法/编译检查 + AST 去重，每 5 步更新一次 |
| Stage 2 混合 rollout | 采集两类数据 | on-policy 探索 + 高质量带边界轨迹 | 环境交互 | D_on（原始动作块）；D_off（技能展开后的块序列） | 无 loss；ρprim=0.5(ALFWorld)/0.75(ScienceWorld)，组规模 N=8 |
| Stage 3 chunk-aware 联合优化 | 学习变长块策略 | 分块 + 任务完成 | D_on + D_off | 统一块格式 (h,u) | L = L_on + λ_off·L_off；L_on 为 clipped surrogate，L_off 为 advantage-weighted 自模仿回归（λ_off=0.1/0.05） |

关键训练设定：lr=1e-6（AdamW）、batch 16 tasks、每 task 8 条 rollout、150 epochs、history length 5、最大块长 K=6、reward {0,10}（ALFWorld）/ [-100,100] 缩放后 clip 负值到 0（ScienceWorld）、非法动作罚 0.1、硬件 4×H200（Llama）/4×GH200（Qwen）。冷启动技能 ALFWorld 3 个、ScienceWorld 5 个；每类别复合技能上限 20，长期零成功率的定期剪枝。

### 4.3 推理流程（动作分块为重点）

- **模型与接口**：测试期只用学到的 primitive-chunk 策略，**无技能调用、无技能库依赖**、无额外标注。
- **单步 vs 多步**：属于"**单次 LLM 决策 + 多步开环执行**"的半马尔可夫范式。每轮 LLM 调用只做**一次**推理，输出 1~6 个逗号分隔的合法原始动作块，由 executor **开环顺序执行**，**期间不再暂停观察**；直到整块执行完或遇到非法动作才停下、回传新观测进入下一轮。这不同于 ReAct 的"每动作一观测"，也不同于规划-执行式的显式两阶段——它把"何时停下来重新观测"完全交给学到的**块边界**决定。
- **终止条件**：环境硬上限 ALFWorld 50 步 / ScienceWorld 30 轮，或任务成功/失败；块内遇非法动作则整轮拒绝、零动作执行（prompt 明示）。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| ALFWorld | General（文本家居） | 多步目标导航+物体操作 | 6 类任务 × train/seen/unseen 划分 | 文本观测+任务指令+admissible actions | 文本原始动作（go/open/take/heat/cool…） | 成功率 SR、每 episode LLM 决策轮次 |
| ScienceWorld | General（虚拟实验室） | 多步科学实验（测温度/连电路/混化学） | 15 类任务（AgentGym-RL 配置，剔除 oracle 超 100 步的） | 文本观测+指令+可用动作 | 文本原始动作 | SR、任务分、LLM 轮次 |

### 5.2 实验结果分析

- **主结果（vs ReAct/Reflexion/RLOO/GRPO/GiGPO/Multi-action GRPO）**：
  - ALFWorld（Table 1）：SPACE 较**每设置最强基线**提升 **+7.0~15.6pp**——Qwen3-4B seen **99.2%/3.7轮**（GiGPO 85.2%/15.9轮）、unseen **96.9%/4.4轮**（GiGPO 72.7%/21.7轮）；Llama seen **96.1%/5.0轮**（GiGPO 89.1%/14.5轮）、unseen **94.5%/5.2轮**（GiGPO 83.6%/18.8轮）。
  - ScienceWorld（Table 2，Llama）：**近乎翻倍**，seen **67.2%** vs GiGPO 35.9%（+31.3pp）、unseen **61.7%** vs 34.4%（+27.3pp），轮次约减半（5.2 vs 10.2、5.8 vs 10.1）。
  - **学出非退化且均衡的块**（Figure 3b/c）：Multi-action GRPO 在 Qwen 塌缩到约 1 动作/轮、在 Llama 膨胀到 5~6 动作/轮；SPACE 稳定在 **3~4 动作/轮**，且策略熵显著更高（探索更充分不塌缩）。
  - **训练更省**：ALFWorld（Llama）SPACE 匹配 GRPO 最终性能仅需 **43.60K rollout 轮次**；达 Multi-action GRPO 最终水平仅用约 **40/150 步（26.6%）**。
- **Ablation 说明了什么**（Table 3，ALFWorld Llama）：去技能 → 86.7%/88.3%（掉 ~9pp），去 chunk-aware 优势 → 90.6%/89.1%（掉 ~5.5pp），证明**技能蒸馏是主因、优势函数是必要补充**；"GRPO w/ Skills"（84.4%/82.0%）低于 SPACE，说明仅加技能不用 chunk-aware 优势仍不够。Figure 4b：ρprim=0.75 最佳，过大过小都差——技能引导应辅助而非主导。Table 4：chunk 策略 × Best-of-N(N=8) 的 SR 增益（52.1%→60.4%，+8.3）近乎单动作策略（+4.2）的两倍，且 TTS 后每 episode LLM 调用 48.3 vs 74.9，二者天然互补。

---

## 6. 亮点与贡献（Why it matters）

1. **问题定位反直觉且深刻**：把"多动作输出"（形式问题）重定位为"学习 chunk 边界"（监督信号问题），直指稀疏奖励下 RL 信号的信息论缺陷，这一 framing 是该方向后续工作的抓手。
2. **"训练期老师、测试期剥离"**：用程序化技能拿到免费边界监督（零人工标注），蒸馏进策略后部署不依赖技能库、无检索开销——把技能变成可泛化的"分块本能"。
3. **成功率 × 效率 × 样本数三赢**：同时提 SR、降轮次、省训练步，并展示了"块策略 × 测试时搜索"的叠加增益，为长程 agent 的推理成本优化提供了清晰杠杆。
4. **可迁移性设计**：方法对技能表示、rollout 模式、优势函数都给出模块化接口，便于在 GUI/Web 等文本动作域替换底座。

## 7. 局限与可改进点（个人点评）

- **环境面窄、假设强**：仅两个确定性较强的文本环境，动作是受限离散指令集（含 admissible actions），"每轮 3~4 动作"的结论在开放/连续/含视觉的 GUI 场景难以直接外推；开环批量执行在高度随机或安全关键环境有真实风险（作者亦承认）。
- **冷启动依赖成功轨迹**：初始成功率低或奖励极稀疏的任务域，技能库可能长期长不大、监督质量不足——这是"技能当老师"路线的系统性软肋。
- **复现门槛高**：未开源，RLVR 需 4×H200/GH200 + 150 epochs；部分设计（K、冷启动技能数、检索策略）缺敏感性分析，超参结论的鲁棒性存疑。
- **边际收益可疑**：Llama 基座 ALFWorld 上 SPACE 相对 Multi-action GRPO 的轮次优势已很有限（5.0 vs 5.4），暗示"回合已被压得很短"的场景分块增益递减；分块的真正价值或集中在长尾长程任务，需更多 per-task 粒度证据（Table 5 虽补充，仍偏粗）。

## 8. 对我们的启示 / 可借鉴点

**（本文属 Agent 方法论域，非 GUI 落地，故单列"对 GUI Agent 的可借鉴点"）**

- **动作分块 = GUI 的"宏操作"**：GUI/Web 长程任务（几十上百步的导航→输入→点击→校验）与本文同构。可把稳定子流程打包成一个 chunk，块间才暂停等截图/DOM 反馈，直接复刻"少决策多执行"，降低单步决策的 token 与延迟开销。
- **技能归纳迁移**：把"填表单/登录/翻页下载"归纳成带参数的 GUI 动作 API 序列（selector 点击、输入、键盘事件）而非自然语言步骤，其调用边界即分块边界，作为 GUI 场景的廉价边界监督。
- **混合训练 + 蒸馏防塌缩**：GUI RL 用稀疏成败奖励直接训多动作策略同样会塌缩/过度承诺（本文 Figure 1 两种失败极常见），可照搬"技能增强 rollout → 展开成 off-policy 蒸馏数据 + chunk-aware 优势"的配方，测试时剥离技能库。
- **务必保留块内可中断**：GUI 的 DOM/页面状态部分可观测且易变（弹窗、加载时序），随机性高于文本环境，建议加"元素出现/消失即停"的隐式中断条件；并可结合视觉相似度打分做 chunk 级 Best-of-N 搜索（本文已证增益翻倍）。

## 9. 延伸阅读

- **动作分块脉络**：Q-chunking（Li et al., 2025，连续控制 RL 分块）、ACT（Zhao et al., 2023，模仿学习分块）、SEAR（Nagy et al., 2026，更大 chunk）。
- **RLVR 与信用分配**：RLVR/DeepSeekMath（Shao et al., 2024）、GiGPO（Feng et al., 2025，本文两层优势的直接来源）、RLOO（Ahmadian et al., 2024）。
- **程序化技能**：ASI（Wang et al., 2025b）、CodeAct（Wang et al., 2024b）、Voyager（Wang et al., 2024a）、SkillAct（Liu et al., 2024）、Trace2Skill（Ni et al., 2026）。
- **长程 agent 训练框架**：ReAct（Yao et al., 2023）、Reflexion（Shinn et al., 2023）、AgentGym-RL（Xi et al., 2025，ScienceWorld 配置来源）、LaMer/Meta-RL（Jiang et al., 2025，no-thinking 设定来源）。

---
*解读生成时间：2026-09-07 ｜ 解读人：WorkBuddy（AI）*
