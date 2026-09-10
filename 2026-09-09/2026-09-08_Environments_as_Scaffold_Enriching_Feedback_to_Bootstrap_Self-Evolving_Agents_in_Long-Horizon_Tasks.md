# Environments as Scaffold: Enriching Feedback to Bootstrap Self-Evolving Agents in Long-Horizon Tasks（环境即脚手架：富化反馈以引导长程任务智能体自进化）

> 一句话 TL;DR：不再靠 SFT 热启动或改奖励来对抗长程 RL 的奖励稀疏，而是按"先动作指引、后观测富化"的时序改造环境观测，跨 Qwen3-4B/8B 与 GRPO/DAPO/GSPO 平均提升 2.82%。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Environments as Scaffold: Enriching Feedback to Bootstrap Self-Evolving Agents in Long-Horizon Tasks |
| **作者 / 机构** | Hongbang Yuan（复旦）、Zhuoran Jin（中科院自动化所 CASIA）、Yixin Cao（复旦 / 上海创智学院，†通讯作者） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-08；arXiv preprint（cs.LG） |
| **arXiv 链接** | https://arxiv.org/abs/2609.08404 |
| **代码仓库** | ✅ GitHub：https://github.com/HongbangYuan/EnvAsScaffold |
| **数据集地址** | SciWorld（Wang et al., 2022）、BFCL-v3 Multi-Turn（Patil et al., 2025），均为公开 benchmark；无私有数据集 |
| **类型标签（论文类别）** | `RL`、`Online`、`General` |
| **训练方法标签** | `RL (GRPO)`、`RL (DAPO)`、`RL (GSPO)`——在 Qwen3-4B/8B 上做标准 agentic RL，无 SFT warm-up，改动全在环境侧反馈 |
| **关键词** | 奖励稀疏、长程任务、Feedback-Enriched Environments、动作指引、观测富化、agentic RL |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_Environments_as_Scaffold_Enriching_Feedback_to_Bootstrap_Self-Evolving_Agents_in_Long-Horizon_Tasks.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：把 LLM 训练成在网页、终端里连续决策几十步的**长程 agent**，核心难点是**奖励稀疏**——任务成功只是轨迹末尾一次 0/1 信号，早期几乎采不到正样本，梯度趋零，agent 困在零奖励轨迹里打转。
- **为什么重要**：这直接卡住了"从单轮静态推理转向动态环境自主执行"这条 agentic RL 主线。不解决，模型就只会背题、不会在长链条里探索。
- **现有方法有什么不足**：主流是"agent 侧热启动"——先 SFT 专家轨迹。作者指出两条结构性缺陷：(1) 专家轨迹采集昂贵、难规模化；(2) 把 SFT 目标优化过头会把策略"焊死"在示范行为上，反而压缩 RL 阶段所需的探索空间（引 Kang et al. 的 SFT-RL 困境）。
- **Research Gap**：作者主张**范式切换**——问题未必出在 agent 不够好，而是**环境给的学习信号太差**。前人（Scaf-GRPO、AdHint、StepHint）都在 agent 侧或数据侧加脚手架，而"如何系统重构多轮动态环境来促进 agent 进化"这一**环境侧**视角仍是空白。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

在训练期把环境的原始观测 `o_t` 经变换 `ℰ(o_t, h_t)` 富化为信息更足的 `o_t⁺`，**不动奖励函数**，仅靠重塑"策略可感知的状态"来隐式调节难度、鼓励探索。

### 3.2 方法总览（Pipeline）

- **输入**：目标指令 + 部分观测 + 交互历史；**输出**：文本动作。环境建模为 goal-conditioned POMDP `(G,S,A,O,T,r)`。
- **两个设计轴**：①**给什么（what）**——动作指引（Action Guidance, AG，把"下一步做什么"写进观测，剪掉组合爆炸式无效探索）与观测富化（Observation Enrichment, OE，补 POMDP 缺失的隐状态语义，如电路任务提示"负极仍未通电"）；②**何时给（when）**——分**episode 内**（单条轨迹前/后段）与**训练周期内**（模型弱的前期/强的后期）两条时间轴。
- **数据流**：RL rollout 在富化环境里采样（0.5 概率注入反馈），轨迹级 advantage 均匀赋给 agent token、**环境 token 被 mask**；评测则在标准环境、不相交任务上进行，保证提升来自"学会的本事"而非"见了提示才会"。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：①"环境侧适配"范式本身（改 observation 而非 reward/初始化，规避 reward hacking）；②提炼出可迁移准则 **"AG 早、OE 晚"** 且两条时间轴同时成立；③发现 **intra-group feedback consistency** 是 GRPO 系稳定优化的前提。
- **工程组合**：AG/OE 本质是 AdHint/StepHint 的 stepwise hint 思路在环境观测上的迁移；GRPO/DAPO/GSPO、Qwen3 全为现成组件，本身不新。
- **对性能提升最关键的设计**：AG-Early 与 OE-Late 的**时序切换**（pilot 里前 100 步 AG、后 100 步切 OE 全场最高）；以及组内反馈一致性（不一致会致成功率剧烈震荡）。
- **证据不足 / 仅声称有效**："early/late"阶段边界与 0.5 注入概率均为手设超参，正文无敏感性分析；作者自承通用性未充分验证。

---

## 4. 具体技术细节

### 4.1 模型结构

base model 为 **Qwen3-4B / Qwen3-8B**（纯 LLM，无视觉编码器，对应 SciWorld/BFCL 两个文本/API 环境）。不冻结、直接做 agentic RL；无 SFT 预热，仅对比 GPT-5.4、Kimi-K2-Thinking、Qwen3-235B-Thinking 作为性能天花板。

### 4.2 训练流程

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| Stage 1（唯一阶段） | 在 FEE 里自演化出长程决策策略 | 探索 + 长程状态追踪 | SciWorld、BFCL-v3 Multi-Turn | 目标指令 + 富化观测（AG/OE 文本注入）+ 文本动作 | trajectory-level advantage 均匀赋给 agent token，环境 token mask；GRPO/DAPO/GSPO |

- 学习率 1e-6，每 5 步评测一次，Table 1 记录训练峰值（非终值）。
- 无 Stage 2，无奖励函数改造，唯一干预是观测空间的文本注入。

### 4.3 推理流程

多步推理：agent 依 `a_t ~ π_θ(a_t | g, o_t, h_t)` 在有限 horizon T 内循环交互，环境返回富化观测 `o_t⁺`。**训练在富化环境、评测切回标准环境**；trajectory-level reward 是二元成功/失败信号。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| SciWorld | General（文本游戏） | 小学科学实验端到端执行 | 30 任务类型 × 多变体 | 目标指令 + 文本观测 | 文本动作 | 成功率 % |
| BFCL-v3 Multi-Turn | General（API 多轮） | 函数调用 / 文件系统 / 长上下文 | 400 环境（按难度分层） | 用户指令 + API 观测 | API 调用 | 成功率 %（Base/Long Context/Miss Func/Miss Param 四子集） |

### 5.2 实验结果分析

- **主结果**：六组配置（4B/8B × GRPO/DAPO/GSPO）全部持平或超过标准环境，**平均 +2.82%**。峰值：4B+DAPO 43.88→47.81（+3.93）、8B+DAPO 45.80→48.11（+2.31）；单点最大增益在 BFCL-Base（4B+GRPO 58→68、4B+DAPO 61→71，各 +10pp）与 SciWorld（8B+GSPO 53.91→60.94，+7.03）。
- **逼近闭源上限**：RL 后 4B/8B 平均 47.81/48.11，远超 Qwen3-235B-Thinking（35.09），逼近 GPT-5.4（52.07）与 Kimi-K2-Thinking（53.51）。
- **Ablation / 机制分析（四项）**：①**训练稳定性**——标准环境 4B 策略熵约 250 步塌缩到 0，FEE 全程 300 步不崩；加熵正则后标准环境 130 步崩、FEE 稳近 200 步。②**状态探索**——400 环境按标准模型成功率分 easy/medium/hard 三档，FEE 训练模型在 **hard 档 +4.3%**（6.5→10.9）。③**反馈内化**——BFCL 文件系统探针显示 FEE 训练模型把富化信息写进权重而非当推理时提示。④**组内一致性**——组内随机给不同反馈致成功率剧烈震荡，必须组内一致。

---

## 6. 亮点与贡献（Why it matters）

1. **范式正交**：给稀疏奖励难题增加一个与所有 agent 侧算法正交、可叠加的"环境侧"抓手，不改奖励、无 reward hacking。
2. **可迁移准则**："AG 早、OE 晚"在 intra/episode 与 inter/episode 两条时间轴同时成立，简洁可操作。
3. **证据链完整**：不止报分，用熵动态、难度分层、权重探针、组内一致性四组分析把"为什么有效"讲透。
4. **普适警示**：组内反馈一致性对 GRPO 系是条红线——任何在 rollout 注入随机性的方法都要先保证组内一致，否则污染组相对 advantage。
5. **门槛低全开源**：不需额外专家轨迹与奖励工程，代码已放出，易在现有 RL pipeline 复现。

## 7. 局限与可改进点（个人点评）

- **阶段边界手调**：前 100 步 AG / 后 100 步 OE、0.5 注入概率全靠人拍，无敏感性分析，跨模型/任务未必最优。
- **通用性有限**：作者自曝在 AppWorld（457 个真实 App API 长程任务）上 4B/8B 加富化仍困在零奖励轨迹——FEE 能救"中等稀疏"，救不了"极稀疏 + 高难度"；且富化内容（参考轨迹 hint、进度检查）依赖环境内部逻辑，很多场景没有现成 ground-truth 中间信号可抄。
- **副作用讨论不足**：FEE 让 agent 更"听话"却更不会在信息不足时反问澄清（Miss Param 4B+DAPO 41→34、Miss Func 下滑），提示富化会侵蚀对输入正确性的质疑能力，真实部署里是致命缺陷。
- **评测口径窄**：仅两个文本/API 环境，未覆盖视觉 GUI/网页；主表取训练峰值而非稳定终值，泛化结论要打折扣。

## 8. 对我们的启示 / 可借鉴点

做长程 agent RL（尤其自带环境仿真的训练管线）时最实用的启示是：**遇到"0 奖励不收敛"，先别急着加 SFT 数据或堆算力，先看能否把环境反馈改得更好学**。它完全不碰奖励函数，只以文本形式改观测，工程改动小，能与 GRPO/DAPO、经验回放、curriculum 正交叠加。"组内反馈一致性"这条红线应直接写进采样器：任何在 rollout 里加噪声/提示/难度变换的地方，必须保证同 batch 组内一致。

**对 GUI Agent 的可借鉴点**：GUI 环境正是典型的部分可观测、稀疏二元奖励、动作空间巨大（点哪、输什么、滑多少）的长程场景，早期 RL 常整段采不到正奖励。可迁移三点：①**早期动作指引**——把参考轨迹或规划器产出的下一步微操作 hint（如"点击右上角 Add to Cart"）以系统消息注入观测，先帮策略把成功率从 0 抬起来；②**中后期观测富化**——GUI 原生观测（截图/无障碍树）缺任务级状态，可叠加"当前子任务进度（已完成 2/5 步）、表单字段是否填齐、页面是否已跳转"等语义标注，对应 OE 后期发力；③**两条时间轴都要注意**——先 hint 后状态摘要，切换点按收敛曲线定。尤其可借鉴其 SciWorld 版 OE：很多 GUI benchmark（如 AndroidWorld）自带 checker/进度判定逻辑，能低成本抽中间子任务完成信号。但需警惕：对信息缺失类 GUI 请求（指令本身缺参），富化会让 agent 只知执行不知反问，应在训练分布里保留需澄清的任务来对冲。

## 9. 延伸阅读

- **RLVMR**（Zhang et al., 2025）：verifiable meta-reasoning reward 训长程 agent，与本文奖励改造可叠加。
- **verl-agent / verl**：本文实现所基于的开源 agentic RL 框架。
- **UI-S1**（Lu et al., 2025）：GUI 自动化 semi-online RL，同属"GUI + RL"，可对比环境侧 vs 数据侧改造。
- **WebRL / AgentGym-RL（ScalingInter-RL）**：自我演化在线 curriculum RL，代表"agent/任务侧"解法，与本篇"环境侧"互补。
- **AdHint / StepHint**：数学/推理 RL 里注入自适应 stepwise hint，印证 hint 注入是通用脚手架手段。
- **AppWorld**（Trivedi et al., 2024）：本文披露 FEE 失效的极稀疏长程 benchmark，挑战更大。

---
*解读生成时间：2026-09-09 ｜ 解读人：WorkBuddy（AI）*
