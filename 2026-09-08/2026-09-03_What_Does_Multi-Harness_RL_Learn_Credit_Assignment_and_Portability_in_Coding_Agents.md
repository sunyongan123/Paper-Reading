# What Does Multi-Harness RL Learn? Credit Assignment and Portability in Coding Agents

> 一句话 TL;DR：固定曝光与预算、只移动 GRPO 分组边界，Cross（harness 混池）与 Within（逐 harness 分组）在训练未见 harness 上无统计差异——评测 harness 才是主导变量。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | What Does Multi-Harness RL Learn? Credit Assignment and Portability in Coding Agents |
| **作者 / 机构** | Chenqian Le、Jiayi Cheng（并列一作，纽约大学）；Qijia He（华盛顿大学）；Runhao Li（南加州大学）；Yinghao Li（哥伦比亚大学）；Xupeng Chen（通讯，Dimension Gate） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-03；arXiv preprint（arXiv:2609.04518v1 [cs.AI]） |
| **arXiv 链接** | https://arxiv.org/abs/2609.04518 |
| **代码仓库** | ⚠️ 声明随文发布 artifacts（PLAINCROSS、weak-ReAct、可复现全部数字的脚本/JSON），但未给可点击 URL |
| **数据集地址** | 训练：SWE-Gym（repo 级 Python 任务池，未单独给链接）；评测：SWE-bench Verified（https://openai.com/index/introducing-swe-bench-verified/） |
| **类型标签（论文类别）** | `RL` `Online` `General` |
| **训练方法标签** | `SFT warm-start（成功轨迹蒸馏）` + `RL (GRPO)`；核心干预为 GRPO 相对优势的分组边界 |
| **关键词** | 信用分配、多 harness、GRPO、编程 Agent、可移植性、SWE-bench |
| **来源渠道** | arxiv-api |
| **PDF 存档** | 2026-09-03_What_Does_Multi-Harness_RL_Learn_Credit_Assignment_and_Portability_in_Coding_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：编程 Agent 的 RL 普遍跑在完整执行 harness（决定 prompt、工具、观测、上下文管理、重试与控制流的 scaffold）之上，前沿配方常把多个 harness 喂给同一个策略。所谓 "multi-harness RL" 其实混着两个独立选择——①把策略**暴露**给多个 harness（exposure）；②在同一个相对优势组内**比较**不同 harness 产出的奖励（credit assignment）。本文只把第②个旋钮单独拎出来做受控实验。

- **为什么重要**：业界两种做法并存且互相矛盾——HarnessX 把同一任务上跨 harness/版本的轨迹放进同一 GRPO 组直接竞争；ClawGym II 以 task–harness 对为归一化单元、奖励各归各组。由于这些系统在 exposure、数据生成、预算、评测上都不一致，端到端分数无法回答"分组边界本身带来什么"。不澄清这一点，源 harness 上的增益会冒充"模型内化能力"，误导排名与部署。

- **现有方法有什么不足**：作者逐条点名——OpenForgeRL 报告的三 harness 配方 held-out +9.5/+20.3 pp 混着 SFT 蒸馏与 RL rollout 双阶段、边际贡献未被隔离；Orchard 的 45.0% 未见 Kimi-CLI 是打包 SFT 配方的绝对分数而非 base 相对 RL 效应；Polar 的 GRPO 增益强烈 harness 索引化（+0.6~+22.6 pp）、每个 checkpoint 只在自己训练 harness 上打分。共同缺陷：**从不报告 advantage 归一化是否跨 harness 边界，也从不引入训练未见 harness 做可移植性探针**。

- **Research Gap**：作者声称补上了"在 repo 级 coding 上，同数据、同预算、同 oracle 条件下隔离 GRPO 分组边界"的缺口，并用一个训练中从未出现的评测 harness（weak-ReAct）把"源配置适配"与"可移植能力"分开测量。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

只动一个变量——从同一 SFT 起点回放同一份冻结轨迹 manifest，唯一差异是 GRPO 优势分组边界（Within 逐 harness 分组 vs Cross 同任务混池），并用训练未见 harness 度量可移植性。

### 3.2 方法总览（Pipeline）

- **输入**：SWE-Gym 任务池；**输出**：各 checkpoint 在源 harness 与 held-out harness 上的二元解决率。
- **模块**：
  1. **经验采集**：四个生产 harness（Aider / OpenHands / Qwen Code / SWE-agent）在 183 个"harness 间存在分歧"的任务上各采轨迹，冻结为同一 manifest（81216 条记录 / 5543 episode，reward、token、loss mask 全固定）。
  2. **SFT warm-start**：同一 Qwen3-8B，用 4 harness 在 SWE-Gym 186 任务上的 8841 条成功轨迹做蒸馏。
  3. **GRPO 训练（四臂）**：同预算（81200 步、单 epoch、lr 3×10⁻⁷、KL 0.01）回放同一数据，仅优势分组不同。
  4. **密封评测**：单一逐实例 oracle，每个 checkpoint 同时在四源 harness 和训练未见的最小 harness（weak-ReAct 单循环 ReAct）打分。
- **连接**：采集 → 冻结 manifest → SFT 起点 → 四臂并行 RL → 同一 oracle 双剖面（源 profile = 源适配 + 可移植；held-out profile 只隔离可移植成分）。六个 held-out 对比在 checkpoint 存在前冻结，逐任务配对 bootstrap。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：把 "multi-harness RL" 拆成 exposure 与 credit assignment 两个旋钮，首次在 repo 级 coding 上隔离分组边界；用"训练未见 harness"作可移植性探针；用 OOF 分类器直接度量 advantage 里携带的 harness 身份（这是把"信用污染"变成可测量的新诊断）。
- **工程组合**：GRPO、SWE-RL/SWE-Gym 的 outcome-reward recipe、四个生产 harness、密封 oracle、Coverage-matched/Residualized（后者是 Liu 2025b/Yu 2025 单 harness 偏差修正思路的迁移）本身都不新，新在"同数据同预算只动边界"的受控组合。
- **对性能提升最关键的设计**：结论是 null，因此最关键的"设计"是**评测协议本身**——密封 oracle + held-out harness + 任务级聚类 bootstrap + 故障隔离（infra failure 不计零）——它让"无差异"结论可信而非噪声。
- **证据不足 / 仅声称有效**：Residualized 臂基线在全 1008 任务上估计、训练只用 183 分歧任务，实际只消掉 9% 偏移，因此"残差化能去除 harness 偏移"并未真正被验证；on-policy 重采集只有单臂单 seed，结论弱。

---

## 4. 具体技术细节

### 4.1 模型结构

- **Base model**：Qwen3-8B（纯 **LLM**，无语视觉编码器；本任务输入为文本 issue + 代码上下文，不涉 MLLM）。参数量 8B。SFT 与 RL 均全参微调（非 LoRA/冻结）。
- 第二模型对比：Seed-Coder-8B（仅做未训练的 harness 对比，context 32768 tokens vs Qwen3-8B 的 40960）。

### 4.2 训练流程（credit assignment 是重点）

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| Stage 0（Base） | 锚点 | —（不更新） | — | — | 未训练 Qwen3-8B |
| Stage 1（SFT warm-start） | 成功轨迹蒸馏 | 基础 repo 级修复行为 | 4 harness 在 SWE-Gym **186** 任务上的成功轨迹 | **8841** 条记录，成功-only | 监督 loss（继承 RL 的 lr 3×10⁻⁷ 与预算，为"匹配超参消融"而非调优蒸馏） |
| Stage 2（RL，四臂同回放） | 隔离 credit assignment 边界 | 比较四臂差异 | 冻结 manifest：SWE-Gym **183** 个 harness 分歧任务 | **81216** 条记录 / 5543 episode，reward/token/loss mask 固定 | GRPO：L = E[min(ρ·Â, clip(ρ,1−ε,1+ε)·Â)] − β·D_KL(π_θ‖π_ref)，KL 0.01，单 epoch，**81200** 步 |

**四种 credit assignment 规则（唯一变量）**：

| 臂 | 分组边界 | 优势定义 | 训练信号审计 |
|---|---|---|---|
| **Within** | 每 task–harness 对一组 | A = (r − μ_{x,h})/(σ_{x,h}+ε)，组内共享 harness，harness 偏移构造性消掉 | 698 组，仅 44.8% 非零方差，**55.1% 组全对/全错、优势为 0** |
| **Cross（PLAINCROSS）** | 每 task 一组，全 harness 混池 | A = (r − μ_x)/(σ_x+ε)，组均值混入各 harness 自身 solve rate | 183 组，100% 非零方差，零优势 0% |
| **Coverage-matched** | Cross 的组 | 把 Cross 非零优势轨迹数下采样到与 Within 相等，其余置零 | 检验"梯度支撑宽度"而非"比较"是否起作用 |
| **Residualized** | Cross 的组 | 混池前先减 leave-one-task-out 逐 harness 基线 b_h^(−x) | 基线在全 1008 任务估计，仅消掉 **9%** 偏移 |

训练时四 harness 分歧任务的 harness solve rate：Aider 6.9%、SWE-agent 9.8%、Qwen Code 20.9%、OpenHands 27.0%（这正是 Cross 组均值里的偏移来源）。

### 4.3 推理流程

- 评测是**多步 agent rollout**：每个 checkpoint 在 SWE-bench Verified 500 任务上以不同尝试数打分——源 harness avg@2（每任务两次尝试，共 24000 次密封评测、99.1% 覆盖），held-out weak-ReAct 上 k=4 与 k=8。weak-ReAct 是单循环 ReAct（多步但不带重试/反思），训练中从未出现。
- 训练 replay 为**离线单 epoch**（无在线 rollback，优势预计算）；仅 re-collected 臂后半段改为 on-policy 重采集。推理终止由各 harness scaffold 的控制流决定，评测以密封逐实例 oracle 四态判据（infra 故障隔离、不计零）为准。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| SWE-Gym | General（repo 级 coding，训练） | 端到端 issue 修复 | 186（SFT）/183（RL manifest）任务 | issue 文本 + 仓库代码 | 补丁（patch） | 成功轨迹数 |
| SWE-bench Verified | General（repo 级 coding，评测） | 端到端 issue 修复 | 500 任务 | issue 文本 + 仓库代码 | 补丁（patch） | 二元 resolved rate（密封 oracle），avg@2/4/8 |

> 两个 benchmark 家族构造性不相交，训练与评测无泄漏。

### 5.2 实验结果分析

**主结果（三个层次）**：

1. **评测 harness 是最大变量**：同一 24000 条密封评测上，harness 列均值从 Aider 2.14% 到 OpenHands 9.27%，跨度 7.13pp（约 4.3 倍）；训练配方行均值仅 5.55%~6.46%（0.91pp，约 1.16 倍）。harness 强弱非模型无关标量：对 Seed-Coder-8B，Aider/SWE-agent 相对其最小 harness 各折损约 4.4pp，OpenHands 与 Qwen Code 完全跑不起来。
2. **分组规则在 held-out 上无差异**：weak-ReAct 上 k=8 时 Cross−Within = +0.25pp [−0.48, +1.02]；相对 SFT 为 −0.03 / +0.13，区间全含零。k=4 六配方跨度 4.01%~4.73%，小于最小可检出差（1.19~1.42pp）。预算翻倍后 Cross−Within 从 +0.58 收敛到 +0.25；3 训练 seed 汇总 +0.16 [−0.41, +0.72]，逐 seed +0.62/+0.05/−0.20（符号翻转），且每臂自身 seed 波动 0.42~0.45pp 大于臂间差。on-policy 重采集一半（re-collected 相对 Cross +0.13 [−0.85, +1.10]）也不改结论。
3. **增益都落在源 harness 且集中于同一 harness**：源 harness 上 Cross 相对 SFT +0.77pp [0.03, 1.52]（Bonferroni 99% [−0.21, +1.75]），Within +0.48pp；二者最大单列增益都在 OpenHands（Cross +1.81 vs Within +1.71）。作者称之为 "configuration adaptation without portable capability"。

**Ablation / 诊断说明了什么**：
- **advantage 身份诊断**：OOF 分类器（chance 25%）从优势反推 harness——Within +0.02pp（构造性无），Cross +4.48pp [+3.22, +5.83]（斜率 2.38）；Coverage-matched 保留约 1/4（+1.17），Residualized 几乎全保留（+4.54）。说明 Cross 的组均值污染真实可测。
- **行为侧 JSD**：同 harness 内 SFT vs Cross 动作分布 JSD 仅 0.0003~0.0029 bit，而换 harness 达 0.17~0.35 bit（差 ≥59 倍）——**动作库由 harness 设定，credit 规则改不了它**。
- **无下游后果**：删除污染（Residualized/Coverage-matched）也不带来 held-out 增益，故问题不在标量偏移层面。

---

## 6. 亮点与贡献（Why it matters）

- 把 "multi-harness RL" 拆成 exposure 与 credit assignment 两个旋钮，给出 repo 级 coding 上第一个隔离分组边界的同数据/同预算/同 oracle 对照实验。
- 用"训练未见 harness"作可移植性探针，正面演示源 harness 增益如何伪装成模型能力（逐条点评 OpenForgeRL / Orchard / Polar 的报告数字）。
- 方法学严谨：密封 oracle、故障隔离、6 个预注册对比、逐任务聚类 bootstrap、seed 与预算灵敏度齐备，是"把 harness 当一等变量报告"的示范。
- 结论直指社区：harness 移动分数的能力（×4.3）远超训练配方（×1.16），agent 评测不公开 harness 版本就没有可比性。
- 对 RL 实践者的可操作警示：Cross 的组均值污染真实可测，但删除它也不带来 held-out 增益，说明修标量偏移不够。

---

## 7. 局限与可改进点（个人点评）

- 覆盖窄：单模型族（Qwen3-8B，Seed-Coder-8B 只做未训练对比）、单预算、单域（Python SWE）、二元终端奖励、绝对 solve rate 仅 1~10%。分辨力约 1pp，正确表述是"无证据有差异"而非"等价"——作者自认 null 只是上界。
- 主结论多基于单 checkpoint（每臂一个 RL seed），Residualized/Coverage-matched 无 seed 波动估计。
- held-out harness 选的是最弱的 minimal ReAct，其分数与未训练 base 几乎齐平、全部配方都不超 base——留出接口难度选择可能低估"可移植能力"存在的可能，换更强的留出 harness 或可测出差异。
- SFT 臂按 RL 的 lr 与算力训练，是"匹配超参消融"而非调优蒸馏配方，不宜推出"RL 无用"的强结论。
- 未见 harness 评测需额外 rollout+grading 算力，规模化有成本门槛；后续若引入 per-step value、分层/hindsight credit 或 process supervision，可能触及标量偏移碰不到的信道。

---

## 8. 对我们的启示 / 可借鉴点

本文属【Agent 方法论域】（非 GUI 落地），对 GUI Agent 的可借鉴点如下：

- **报告惯例**：任何 multi-environment GUI RL 论文都应写清分组边界——是每 (任务, 环境) 一组（Within 式）还是同任务多环境混池（Cross 式），并补一个训练未见环境/harness 的评测。GUI 的 harness 效应很可能比 coding 更极端（obs 格式、动作空间、屏幕分辨率都是变量），评测环境的选择本身就可能移动分数数倍。
- **credit assignment 建议**：多环境数据训练 GUI policy 时，首选每环境分组，或先减 leave-one-task-out 逐环境基线再混池（对应 Resid），防止强环境抬高 baseline、把"环境帮的忙"记到模型头上；跨环境混池需警惕 55% 全对/全错退化任务被 pool 救活。
- **行为侧启示**：本文 JSD 证据表明，想改变 GUI 策略的"操作风格"（点 vs 键盘、截图频率）应改 harness/观测接口，而不是换 credit 规则。
- **对库内 GUI 评测**：既然 harness 列均值可差 4.3 倍，我们库内 benchmark 解读应把 scaffold 版本、grader、rollout 次数列为必填项（正如此文 Table 6 逐一 pin 版本）。

---

## 9. 延伸阅读

- 多 harness 训练：HarnessX、ClawGym II、OpenForgeRL、Polar、Tmax、Kimi K3、Orchard；数据与评测根基 SWE-RL / SWE-Gym / SWE-bench。
- GRPO 分组与归一化：DAPO、DeepSeekMath；单 harness 分组偏差修正 Liu et al. 2025b、Yu et al. 2025。
- harness 作为评测变量：Harness-Bench、Kapoor et al.（AI agents that matter / Holistic Agent Leaderboard）。
- GUI/环境 RL 相关：WebRL、DigiRL、RAGEN。
- 本库关联解读：2026-09-03《Spurious Advantage Hidden in GRPO》、2026-09-03《Headroom-Drift Replay》、2026-09-02《Tail-Likelihood Reinforcement Learning》。

---
*解读生成时间：2026-09-10 ｜ 解读人：WorkBuddy（AI）*
