# TRACE: Training Reasoning Agents for Causal Exploration with Synthesized Rewards

> 一句话 TL;DR：把"验证不对称性"工程化——先用可控模拟器注入隐藏干预、再生成观测，干预本身即 oracle 标签，从而在缺乏自然 verifier 的归因诊断任务上做 RL，让 35B 模型超过 Claude Opus 5。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | TRACE: Training Reasoning Agents for Causal Exploration with Synthesized Rewards |
| **作者 / 机构** | Rui Sun、Zhan Shi、Bing He（均标注 Independent Researchers，未给出机构与国家；无通讯作者标注） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-09；arXiv preprint（正文标注 "Preprint. Under review."），arXiv:2609.10315v1 [cs.AI] |
| **arXiv 链接** | https://arxiv.org/abs/2609.10315 |
| **代码仓库** | ❌ 未开源（全文未提供代码、环境或项目页，模拟器与 oracle 均未发布） |
| **数据集地址** | 未公开。TRACE 模拟器自建：RL 数据集 5,000 个 oracle 已验证 episode（4,472 train / 528 val），评测集 235 个 held-out episode（在 episode、campaign、intervention 三层与训练/验证集不重叠） |
| **类型标签（论文类别）** | `RL` `SFT` `Benchmark` `General` |
| **训练方法标签** | `SFT`（对 Claude Opus 4.8 教师轨迹做拒绝采样，1,200 条）+ `RL (GRPO)`（含 DAPO 类稳定化改进） |
| **关键词** | RLVR、合成奖励、模拟器–oracle、根因归因、工具使用 Agent、数字广告诊断 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-09_TRACE_Training_Reasoning_Agents_for_Causal_Exploration_with_Synthesized_Rewards.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：在无自然可验证答案的**诊断式推理**上做 RLVR。任务为数字广告根因归因：给定 campaign 绩效异常与多表数据库，Agent 用 Python/SQL 反复取数、提假设、验证据，给出 12 类根因之一，segment-specific 类还需给出受影响切片。
- **为什么重要**：RLVR 依赖"验证比求解容易"，而确定真因往往需昂贵人工排查，事后仍模糊（多变更并发、观测噪声）。缺客观 ground truth，此类任务便被排除在 RL 之外。
- **现有方法不足**：① LLM-as-judge——目标模糊时救不了，还会被 gaming；② 合成数据——只给"可模仿样本"不给"可验证标签"；③ 交互式模拟环境（ScienceWorld、WebArena 等）——用交互**测试**能力，不产训练奖励；④ Text-to-SQL / 数据科学 benchmark——评查询正确性，非受控混淆下的归因；⑤ 因果 benchmark（CLadder 等）——静态问答或符号图，非交互式。
- **Research Gap**：把"验证不对称性"从"等待自然出现"变成"主动制造"——先采样干预、注入可控模拟器、生成观测，干预本身留作 oracle 标签，即 **RL with synthesized rewards**。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

用可控模拟器按"先定因果、再生成数据"批量造诊断题，隐藏干预即答案，归因可被确定性比对打分，把模糊诊断变成可做 GRPO 的 RLVR。

### 3.2 方法总览（Pipeline）

- **输入 / 输出**：campaign 异常描述 + 多表事实库（仅经 Python/SQL 访问）→ 结构化 decision（root_cause + driver_segments + evidence + 解释）。
- **① 模拟器**：建 campaign 总体（品类、竞价、预算与曝光基线、活跃 segment、quality multiplier）→ 采样干预（原因类别、driver slice 1~2 属性、signal strength、onset）→ 按 Poisson 抽总曝光、Dirichlet-Multinomial 分 segment，各 segment 的 CTR/CVR/CPC = 基线率 × multiplier × 干预乘子，经总曝光量、segment 分配、segment 级速率三通道生效。
- **② Oracle 验证器**：只读 Agent 可见事实表，检查特征层级、延迟/渐变窗口、segment 级 SNR ≥ 1，并排除替代原因；只接受"可恢复 + 强于最强混淆 + 可区分于竞争原因"的 episode。
- **③ RL 奖励**：r = 0.65·r_attr + 0.30·r_full + 0.05·r_fmt。r_attr = 1{ĉ=c\*}·r_slice（**原因错则归因分为 0**）；r_slice = α+(1−α)·J(Ẑ,Z\*)（α=0.5，J 为切片 Jaccard），不需切片则 r_slice = 1；r_full = 1{r_attr = 1}；r_fmt 为格式奖励，解析失败则 r_attr = 0。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：**"先采样干预、再生成观测、干预即标签"的 simulator–oracle–RL 范式**。它与"合成数据"（生成要模仿的轨迹）、"交互式模拟环境"（用交互测能力）都不同：本文生成带客观答案的题目、用模拟造奖励。第二点是 **oracle 把"可解性"与"生成"解耦**：模拟器负责多样性，验证器保证可检测、强于最强混淆、与竞争原因可区分。
- **工程组合**：GRPO + DAPO 稳定化、Megatron-LM + SGLang + Ray、拒绝采样 SFT 数据均为现成配方；Jaccard 部分分 + 二值满分亦属常见套路。
- **对性能最关键的设计**：**`r_full` 二值完整归因奖励**。只用分级奖励时 FullAttr@1 仅 0.596、二维精确匹配 0.00；加 r_full 后整体 0.757，一维 0.67→0.92、二维 0.00→0.27。次要关键是 **SFT 热启动**：从 base 直接 RL 仅 0.434，从 SFT 起步 0.757，解析率 0.68→1.00。
- **证据不足 / 仅声称有效**：① **"judge 不可用"是论证而非实验**——无 judge-RL 基线；② **KL 结论过强**——只测系数 0.005；③ **"可扩展、可做课程、可跨域迁移"全是展望**；④ **no-signal 退化的归因无实验支撑**。

---

## 4. 具体技术细节

### 4.1 模型结构

base model 是**纯 LLM，不含视觉编码器**：Qwen3.5-35B-A3B，**MoE 开放权重，35B 总参数 / 3B 激活**；任务是文本 + 表结构 + SQL/Python 工具，不涉及截图。**全参数后训练，不冻结**；bfloat16 参数、float32 梯度累积/softmax/router，Megatron-LM + SGLang + Ray。基线含同族 Qwen3.5-122B-A10B 与 Claude Opus 5 / Sonnet 5 / GPT-5.5 / GPT-5.6 Sol；**SFT 教师是 Claude Opus 4.8**（与评测基线 Opus 5 非同一版本）。

### 4.2 训练流程（若需要训练）

| 阶段 | 目标 | 训练能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| SFT 热启动 | 教输出格式与多轮工具使用 | 结构化答案、Python/SQL 调用 | 1,200 条拒绝采样教师轨迹（教师 = Claude Opus 4.8；困难 episode 给教师"根因特征"轻提示，**不入保留 prompt**） | prompt + tool calls + tool outputs + 结构化答案 | LM loss，**只作用于 assistant turn**，3 epochs，lr 1e-5 余弦衰减 |
| RL（GRPO） | 合成奖励下提升完整归因 | 多轮证据探查、因果归因、切片定位、何时停止 | 4,472 条 oracle 已验证 episode（val 528，按原因/signal level/1D-2D 分层） | 每 prompt 8 条多轮轨迹（≤30 turn，上下文 32,768 token，单 turn ≤2,048） | GRPO + 3.2 奖励；AdamW (0.9,0.98)，lr 1e-6 常数，wd 0.1，clip [0.8,1.28]，grad clip 1.0，entropy 0，温度 1.0；32 prompts × 8 = 256 轨迹/step，共 300 步 |
| 消融对照 | 隔离各设计作用 | — | 同划分同超参 | 同上 | ① 只用分级奖励；② 从 base 直接 RL；③ 加 KL（系数 0.005，k3 估计器，参考策略为 SFT 模型） |

> RL 与评测**共用同一套多轮工具接口**，无分布错配。奖励只解析最终 decision 字段；evidence 用于错误分析但**不进奖励**。

### 4.3 推理流程

- **模型**：后训练后的 Qwen3.5-35B-A3B；输入为系统 prompt（角色、12 类候选原因、可用表、答案格式）+ 用户 prompt（campaign 绩效变化）。
- **单步还是多步**：**多步**工具循环——模型生成 Python（内含 SQL），在沙箱 DuckDB 执行，**状态跨调用持久化**，结果回灌后修正假设、继续取数直至给出答案；终止于自行输出答案或 backstop（30/60 turn 上限、工具 60 秒超时、输出截断 8,000 字符）。**无显式规划器或反思模块**，规划隐含在"提假设→取数→修正"循环里。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| TRACE（自建，held-out） | General（数字广告数据诊断，非 GUI） | 交互式根因归因：识别 12 类根因之一，segment-specific 类还需给出 driver slice（1D/2D） | 235 episode，**刻意富集 segment-specific**：campaign-wide 30 / segment-mix 21 / 1D 115 / 2D 49 / no-signal 20 | 异常描述 + 多表事实库（daily_campaign / segment_daily / budget_log / inventory），仅经 Python/SQL 访问 | 结构化 decision：root_cause + driver_segments + evidence + 解释 | Cause@1、FullAttr@1、FullAttr@5、Jaccard、No-Signal、Parsed、平均 tool calls |
| TRACE RL 训练集（非评测） | 同上 | 同上 | 5,000 episode → 4,472 train / 528 val，按原因、signal level、1D/2D 复杂度分层 | 同上 | 同上 | 训练用合成奖励（与评测同源的 cause + slice 判据） |

12 类原因分 4 组：Campaign-wide（BID_INCREASE、BUDGET_CAP、PAGE_DEGRADATION、OUT_OF_STOCK）、Segment-mix（PLACEMENT_SHIFT、TARGETING_BROADENING、TARGETING_NARROWING）、Segment-specific（CREATIVE_FATIGUE、COMPETITIVE_PRESSURE、AD_QUALITY_DROP、AUDIENCE_SATURATION）、No signal（NO_SIGNAL）。难度由 dimensional（1 属性 vs 2 属性交集）与 temporal（立即/延迟/渐变）两轴控制。

### 5.2 实验结果分析

**主结果（Table 2，235 题 held-out）**：

| 模型 | Cause@1 | Jaccard | FullAttr@1 | FullAttr@5 | No-Signal | Parsed |
|---|---|---|---|---|---|---|
| Qwen3.5-35B（base） | 0.184 | 0.596 | 0.159 | 0.370 | 0.41 | 0.53 |
| + SFT | 0.685 | 0.958 | 0.637 | 0.762 | 0.76 | 1.00 |
| + RL（从 base 起步） | 0.471 | 0.913 | 0.434 | 0.711 | 0.28 | 0.68 |
| **+ SFT→RL** | **0.823** | 0.928 | **0.757** | **0.851** | 0.49 | 1.00 |
| Qwen3.5-122B-A10B | 0.296 | 0.842 | 0.283 | 0.434 | 0.78 | 0.88 |
| Claude Opus 5 | 0.764 | 0.913 | 0.686 | 0.809 | **0.87** | 0.99 |
| GPT-5.6 Sol | 0.635 | 0.852 | 0.565 | 0.719 | 0.30 | 0.99 |
| GPT-5.5 | 0.581 | 0.893 | 0.524 | 0.643 | 0.61 | 0.99 |
| Claude Sonnet 5 | 0.472 | 0.899 | 0.438 | 0.607 | 0.85 | 1.00 |

三条主线：① **任务未饱和**——最强 prompt 基线 Opus 5 仅 0.686 FullAttr@1；② **RL 对 SFT 是加性的**——SFT 0.637 → SFT→RL 0.757（+12.0 点），超过所有 prompt 基线，FullAttr@5 从 0.762 升到 0.851；③ **后训练可压过 prompt 侧规模优势**——同族 35B 后训练版（0.757）远胜 122B prompt 版（0.283），Cause@1 从 0.296 拉到 0.823。**反面数字必须同看**：SFT→RL 的 **No-Signal 仅 0.49**，低于 SFT 的 0.76、远低于 Opus 5 的 0.87；从 base 直接 RL 更掉到 0.28——"整体归因最强"者在"不该归因时"最易乱归因。

**分类型结果（Table 4，FullAttr@1，顺序 base / SFT / SFT→RL / Opus 5）**：Campaign-wide（30）0.46 / 0.99 / 0.95 / 0.95；Segment-mix（21）0.48 / 0.98 / **1.00** / 0.98；1D（115）0.04 / 0.71 / **0.92** / 0.68；2D（49）0.01 / 0.04 / 0.27 / **0.33**；No-signal（20）0.41 / 0.76 / 0.49 / **0.87**。SFT→RL 在 1D 上碾压 Opus 5，但**最难的 2D 切片仍落后**（0.27 vs 0.33），no-signal 差距更大；含 device 维度的 2D 错误中它找回正确值 66/94。

**Ablation（Table 3，FullAttr@1）说明了什么**：

| 训练条件 | Overall | 1D Slice | 2D Slice | No Signal |
|---|---|---|---|---|
| SFT only | 0.637 | 0.71 | 0.04 | 0.76 |
| SFT→RL，仅分级奖励 | 0.596 | 0.67 | **0.00** | 0.64 |
| SFT→RL，+ 完整归因奖励 | **0.757** | 0.92 | **0.27** | 0.49 |
| SFT→RL，+ 完整归因奖励 + KL | 0.724 | 0.92 | 0.16 | 0.30 |
| RL from base，+ 完整归因奖励 | 0.434 | 0.51 | 0.03 | 0.28 |

① **r_full 必需**：无它，RL 反把 SFT 的 0.637 拉低到 0.596，二维精确匹配归零——部分分奖励会把策略锁死在"差不多对"。② **SFT 热启动不可省**：同奖励下从 base 起步仅 0.434、解析率 0.68（SFT 版 1.00），说明 SFT 还承担"把输出变成可打分"的基础设施作用。③ **KL 正则有害**：0.757→0.724，2D 0.27→0.16，no-signal 0.49→0.30。SFT 另把平均 tool calls 从 22.05 降到 10.75（SFT→RL 为 11.73），排除了"涨分靠多查几次"。

---

## 6. 亮点与贡献

1. **"干预即标签"是可迁移的通用配方**：任何过程可控、故障可注入的领域（软件运维、数据质量、推荐系统）都能"先定因、再生果"免费拿到 oracle 标签，把 RLVR 从"答案易验证"扩展到"过程可控"。
2. **生成–验收分离解决 solvable / non-trivial / realistic 三难**：oracle 只读 Agent 可见数据、逐条排除替代原因、要求 SNR ≥ 1，使合成题既有真实噪声与混淆又有客观标签——最值得抄的机制。
3. **奖励设计给出可直接复用结论**：多字段结构化输出上，"分级部分分 + 二值满分"必须同时存在；只有部分分会锁死在次优解（2D 精确匹配 0.00 是极强证据）。
4. **提供 SFT→RL 可比配方**：35B MoE、300 步 GRPO、8 samples/prompt、clip [0.8,1.28]、lr 1e-6、KL 反而有害。
5. **同族 35B vs 122B 对照把"后训练信号 > 模型规模"讲得很硬**（0.757 vs 0.283）。

## 7. 局限与可改进点（个人点评）

- **no-signal 退化是硬伤且未解决**。诊断系统的价值很大程度取决于"何时不下结论"，而 SFT→RL 把 no-signal 从 0.76 降到 0.49，低于 Sonnet 5（0.85）与 Opus 5（0.87）。奖励无 abstain 项，策略必然学会"宁可猜一个像样的原因"；作者称其为 trade-off，却未尝试加 abstain 奖励打破它。
- **"超越所有 prompt 基线"在关键子集上不成立**：整体 0.757 > 0.686，但 2D 切片 0.27 < 0.33、no-signal 0.49 < 0.87，而 2D 正是论文自称"最难"的部分。
- **单域单环境，无迁移证据**：结论全基于自建数字广告模拟器（12 类原因、固定 schema）；所列软件运维、数据质量等迁移方向均未验证，模拟器又不公开。
- **SFT 数据来自单一闭源教师且对难题给了提示**：1,200 条全来自 Claude Opus 4.8，困难 episode 还喂了"根因特征"提示；作者称提示未进保留 prompt，但**教师上限即策略初始上限**，且相对 4,472 条 RL prompt 覆盖偏小。
- **实验与评测口径留有缺口**：RL 只跑 300 步且未报训练曲线/收敛性，KL 只测系数 0.005 就断言无效；235 题中 164 题（约 70%）为 segment-specific、49 题为 2D 交集，不能推断自然分布准确率；Parsed 率混进主表，RL-from-base 仅 0.68 会模糊 0.434 含义。
- **缺少最关键的对照基线**：用 LLM-as-judge 训练的 RL。核心动机是"judge 不行、合成奖励行"，就应给出"强模型当 judge 打同一套 GRPO"的对照。

## 8. 对我们的启示 / 可借鉴点

**通用方法论**：想拿可验证奖励，先问"过程能不能被控制"而非"答案能不能被检查"；多字段输出必须配"全对才给分"的二值项；消融要按子集拆开看——本文总分 +12 点掩盖了 no-signal 的 −27 点。

### 对 GUI Agent 的可借鉴点

1. **把"注入式合成"搬到 GUI**。GUI 天然可编程，可注入布局位移、元素失效、权限降级、接口变慢等隐藏变更，让 Agent 通过截图与操作**诊断"这一步为何失败"**——变更即标签，无需人工标注或 judge；此类"GUI 故障归因"（自动化测试定位、客服排障）仍是空白。
2. **多字段动作输出照搬"Jaccard 部分分 + 二值满分"**。GUI 动作含类型 + 元素/坐标 + 文本，失败常是"类型对但元素错"或"元素对但参数错"；用 α+(1−α)·Jaccard 给部分分，再加"全字段精确匹配"二值项，避免只学会"大致点对位置"。
3. **必须显式设计 abstain 奖励**。TRACE 的 no-signal 从 0.76 掉到 0.49 是反面教材：GUI 里"不该操作"的情形很多（页面未加载完、需授权、元素不存在），只奖励"完成任务"必然乱点乱填。建议把 NO_OP / 请求确认做成独立动作并给正奖励。
4. **SFT 热启动 + 短步数 RL 可复用，但须重视步数预算**。"22.05 → 10.75 tool calls"提示：**GUI Agent 步数预算紧，SFT 教"何时停止"的收益可能大于 RL 教"怎么做得更好"**。

## 9. 延伸阅读

- Shao et al., *DeepSeekMath*（arXiv:2402.03300）——GRPO 原始出处。
- Yu et al., *DAPO*（NeurIPS 2026）——本文"稳定化改进"来源。
- Wei, *Asymmetry of Verification and Verifier's Law*（blog, 2025）——核心理论动机。
- Wei et al., *SWE-RL*（NeurIPS 2026）、Luo et al., *DeepSWE*——代码 Agent 的 RLVR 路线，形成"自然 verifier vs 合成 verifier"对照。
- Wang et al., *ScienceWorld*（EMNLP 2022）、Zhou et al., *WebArena*（ICLR 2024）、Huang et al., *MLAgentBench*（ICML 2024）——被本文区分开的"交互式模拟环境"一族。
- 本地相关：`2026-09-03_TIGPO_Temporal_Instance-Graph_Policy_Optimization_for_Long-Horizon_LLM_Agents`、`2026-09-03_Headroom-Drift_Replay_A_Primitive_for_Principled_Replay_Control_in_GRPO`、`2026-09-06_The_Double_Measurement_Confound_in_Agent_Benchmarks`。

---
*解读生成时间：2026-09-11 09:00 ｜ 解读人：WorkBuddy（AI）*
