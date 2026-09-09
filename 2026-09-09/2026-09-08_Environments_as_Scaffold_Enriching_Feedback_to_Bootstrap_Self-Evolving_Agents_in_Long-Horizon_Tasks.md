# Environments as Scaffold: Enriching Feedback to Bootstrap Self-Evolving Agents in Long-Horizon Tasks（环境即脚手架：富化反馈以引导长程任务智能体自进化）

> **TL;DR**：与其把 RL 训练卡在奖励稀疏的问题上反复调 agent 初始化，不如直接改造环境——按"早期注入动作指引、后期切到观测富化"的时序给环境反馈"加料"（FEE），跨 Qwen3-4B/8B 与 GRPO/DAPO/GSPO 平均提升 2.82%，训练更稳、探索更主动。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | Environments as Scaffold: Enriching Feedback to Bootstrap Self-Evolving Agents in Long-Horizon Tasks |
| **arXiv ID / DOI** | arXiv:2609.08404v1（cs.LG） |
| **arXiv 链接** | https://arxiv.org/abs/2609.08404（点击直达） |
| **发表出处（Venue）** | arXiv 预印本（arXiv preprint，21 页） |
| **发布时间** | 2026-09-08 |
| **作者** | Hongbang Yuan（复旦）、Zhuoran Jin（CASIA）、Yixin Cao（复旦 / 上海创智学院，†通讯作者） |
| **所属机构** | 复旦大学、中国科学院自动化研究所（CASIA）、上海创智学院（中国） |
| **开源情况 / 代码** | ✅ 有代码：https://github.com/HongbangYuan/EnvAsScaffold |
| **类型标签（论文类别）** | `RL`、`Online`、`General` |
| **训练方法标签** | `RL（GRPO / DAPO / GSPO）`——在 Qwen3-4B/8B 上做标准 agentic RL，改动全在环境侧反馈；未用 SFT warm-up |
| **关键词** | 奖励稀疏（reward sparsity）、长程任务、Feedback-Enriched Environments、动作指引、观测富化、agentic RL |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_Environments_as_Scaffold_Enriching_Feedback_to_Bootstrap_Self-Evolving_Agents_in_Long-Horizon_Tasks.pdf |

---

## 1. 研究背景与要解决的问题

LLM 在数学、代码这类"单轮静态推理"上已经很强，但把它训练成能在网页、终端里连续操作几十步的长程 agent，主流路线是 RL。长程 RL 的头号障碍是**奖励稀疏**：任务成功是轨迹末尾的一次 0/1 信号，agent 在早期几乎采不到正样本，梯度近乎为零、困在零奖励轨迹里打转。业界常规解法是"agent 侧热身"——先用专家轨迹 SFT 把初始策略拔高。但 SFT 有两个硬伤：专家轨迹采集贵、难规模化；而且把 SFT 目标优化过头反而把策略"焊死"在示范行为上，压缩了 RL 阶段需要的探索空间。

本文提出一个视角切换：**问题不一定出在 agent 不够好，而是环境给的学习信号太差**。与其换个更好的初始化，不如在训练期改造环境反馈（observation 层面的干预，而非改奖励函数），让 agent 在稀疏奖励的长程任务里也能学到东西。

## 2. 核心方法 / 思路

形式化上，环境是一个目标条件下的 POMDP，agent 每步按"目标+观测+历史"生成动作，拿到的是稀疏的轨迹级成败奖励。作者定义的**富化反馈**是对观测空间的干预：把原始观测 o_t 经变换 ℰ(o_t, h_t) 变成信息更足的 o_t⁺，相当于在不动奖励函数的前提下重塑了策略可感知的状态，隐式调节任务难度、鼓励轨迹多样性。

关键问题是两个维度的设计：**给什么信息（what）+ 什么时候给（when）**。

- **给什么**：①动作指引（Action Guidance, AG）——把"下一步该做什么"的提示直接写进观测，像先给一句"进入实验室"，剪掉组合爆炸式的无效探索；②观测富化（Observation Enrichment, OE）——补充原观测缺的语义信息，例如电路任务里告诉 agent"负极仍没通电"，针对的是 POMDP 的部分可观测性。
- **什么时候**：分两条时间轴——**episode 内**（单条轨迹的前几步还是后几步给）与**训练周期内**（模型还很弱的前期还是已变强的后期给）。

通过 SciWorld 上的 pilot 实验（Qwen3-4B + GRPO，200 步，0.5 概率注入），作者发现一个清晰规律："**富什么决定何时富**"——AG 全程优于 OE（显式剪枝比让模型自己从语义里因果推理更快见效），且 **AG-Early 前期更好、OE-Late 后期上扬**；把两者串起来（前 100 步 AG-Early、后 100 步切 OE-Late）效果全场最高。整体训练在**富化环境**里做、在**标准环境**上用不相交任务评测，保证提升是"学会的本事"而非"见了提示才会做"。

## 3. 关键实验结果

- **主结果**：在 SciWorld 与 BFCL-V3 Multi-Turn 上，Qwen3-4B/8B 配 GRPO/DAPO/GSPO 六组配置，FEE 全部持平或超过标准环境，平均提升 **2.82%**。亮点：4B+DAPO 平均 43.88→47.81（+3.93）、8B+DAPO 45.80→48.11（+2.31）；单点最大增益在 BFCL-Base（4B+GRPO 58→68、4B+DAPO 61→71，各 +10 个百分点）与 SciWorld（8B+GSPO 53.91→60.94，+7.03）。
- **逼近闭源上限**：RL 后 Qwen3-4B/8B 平均分最高 47.81/48.11，远超 Qwen3-235B-Thinking（35.09），逼近 GPT-5.4（52.07）与 Kimi-K2-Thinking（53.51）——小模型靠 RL 就能打平大模型。
- **训练稳定性**：标准环境下 4B 策略熵在约第 250 步塌缩到 0，FEE 全程 300 步不崩；加了熵正则后，标准环境约 130 步就崩，FEE 仍能稳近 200 步。
- **探索与内化**：按难度分层评测，FEE 训练出的模型在 hardest 档比基线高 **4.3%**；BFCL 的探测实验显示它把"富化信息"内化进了策略权重而非当成推理时的临时提示。
- **负例与边界**：训练集里同组反馈必须一致，组内随机给不同反馈会让成功率剧烈震荡；FEE 对 BFCL 的 Miss Param/Miss Func（信息缺失需反问的用例）反而有下滑（如 4B+DAPO 的 Miss Param 41→34）。

## 4. 亮点与贡献（Why it matters）

1. **提出"环境侧适配"范式**：把 agentic RL 难训归因到 agent 之外，为稀疏奖励问题增加了一个与任何 agent 侧算法正交、可叠加的抓手。
2. **提炼出可迁移的设计准则**："AG 早、OE 晚"，并在 intra-episode 与 inter-episode 两条时间轴上同时成立，简洁、可操作。
3. **证据链完整**：不只报分数，还用熵动态、难度分层、权重探测、组内一致性四组分析把"为什么有效"讲透。
4. **组内反馈一致性这个发现对 GRPO 系算法有普适警示**：任何往 rollout 里注入随机性的方法都要先保证组内一致，否则组相对 advantage 被污染。
5. **门槛低、全开源**：不需要额外专家轨迹与奖励工程，代码已放出，容易在现有 RL pipeline 上复现。

## 5. 局限与可改进点（个人点评）

- **阶段边界仍是手调超参**："前 100 步 AG / 后 100 步 OE"、0.5 注入概率都靠人拍板，正文没有给出敏感性分析——这套时序在别的大小的模型/任务上未必最优，论文自己也承认 early/late 定义是手设的。
- **通用性有限**：作者明确披露在 AppWorld（457 个真实 App API 的长程任务）上，4B/8B 加富化反馈依然困在零奖励轨迹里。也就是说 FEE 能救"中等稀疏"，救不了"极稀疏+高难度"；且富化内容（参考轨迹 hint、任务进度检查）依赖环境内部逻辑，很多场景根本没有现成的 ground-truth 中间信号可抄。
- **副作用未被充分讨论**：FEE 让 agent 更"听话"却更不会在信息不足时反问澄清（Miss Param/Miss Func 下滑），这提示富化会侵蚀 agent 对输入正确性的怀疑能力——在真实部署里这是要命的缺陷，值得专门研究如何保留"质疑"能力。
- **评测口径偏窄**：只有两个文本/API 类 benchmark，没覆盖视觉 GUI/网页环境；主表取的是训练 200 步内的峰值而非稳定终值，泛化结论要打折扣。

## 6. 对我们的启示 / 可借鉴点

做长程 agent RL（尤其自带环境仿真的 GUI/Web 训练管线）时，本文最实用的启示是：**遇到"0 奖励不收敛"，先别急着加 SFT 数据或堆算力，先看能不能把环境反馈改得更好学**。它完全不碰奖励函数（规避了 reward hacking 风险），只是以文本形式改观测，工程上改动小，且能跟我们的 GRPO/DAPO、经验回放、curriculum 等方案正交叠加。其次是"组内反馈一致性"这条红线可以直接写进我们的采样器：任何在 rollout 里加噪声/提示/难度变换的地方，保证同 batch 组内一致，否则 GRPO 的组内 advantage 会被打乱。

**对 GUI Agent 的可借鉴点：** GUI 环境正是典型的部分可观测、稀疏二元奖励、动作空间巨大（点哪、输什么、滑多少）的长程场景，早期 RL 经常整段整段采不到正奖励。可把 FEE 迁移过来缓解稀疏奖励、引导 RL：①**早期用"动作指引"**——把参考轨迹或规划器产出的下一步微操作 hint（如"点击右上角 Add to Cart"）以系统消息形式注入观测，先帮策略剪掉无效探索、把成功率从 0 抬起来；②**中后期切"观测富化"**——GUI 原生观测（截图/无障碍树）常缺任务级状态，可叠加"当前子任务进度（已完成 2/5 步）、表单字段是否填齐、页面是否已跳转"等语义标注，对应本文 OE 在后期发力、帮 agent 把长程状态追踪内化进权重；③两条时间轴都要注意——先 hint 后状态摘要，切换点可按我们自己的收敛曲线定，并把"AG 早、OE 晚"作为默认策略。尤其可借鉴其 SciWorld 版 OE：很多 GUI benchmark（如 AndroidWorld）自带 checker/进度判定逻辑，能低成本抽出中间子任务完成信号；但对信息缺失类 GUI 请求（用户指令本身缺参），需警惕 FEE 式富化会让 agent 变得只知执行、不知反问，可在训练分布里保留需要澄清的任务来对冲。

## 7. 延伸阅读

- **RLVMR**（Zhang et al., 2025，其实现所基于的代码库）：verifiable meta-reasoning reward 训长程 agent，FEE 可与其奖励改造叠加。
- **verl-agent / verl**：本文实现所基于的开源 agentic RL 框架，也是我们改 rollout 环路的参考。
- **UI-S1**（Lu et al., 2025）：GUI 自动化上的 semi-online RL，同属"GUI + RL 训练"方向，可对比环境侧 vs 数据侧改造。
- **WebRL / AgentGym-RL（ScalingInter-RL）**：自我演化在线 curriculum RL 与长程 agent RL 的另一主流路线，代表"agent/任务侧"解法，与本篇"环境侧"互补。
- **AdHint / StepHint**：在数学/推理 RL 里注入自适应 stepwise hint，本质也是"给学习信号加脚手架"，印证 hint 注入是通用手段。
- **AppWorld**（Trivedi et al., 2024）：本文披露 FEE 失效的极稀疏长程 benchmark，挑战更大。

---
*解读生成时间：2026-09-09 ｜ 解读人：WorkBuddy（AI）*
