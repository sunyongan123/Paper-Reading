# When Validation Stops Learning: Auditing Update Admission for Continual Embodied Agents

> 一句话 TL;DR：把"策略更新准入"从单一错误控制重新定义为"错误控制 + 在给定交互预算下保留有用更新机会"的联合审计问题，用 paired-binomial 区间替换 range-based 区间来修复"闸门合法地冻结了学习"，并给出 round-level missed-opportunity 指标。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | When Validation Stops Learning: Auditing Update Admission for Continual Embodied Agents |
| **作者 / 机构** | Qinzhen Ma（Rice University，qm18@rice.edu）、Ruihai Wu（University of California, Berkeley，ruihai@berkeley.edu）（美国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-09；arXiv preprint（cs.AI，正文标注 "Preprint"） |
| **arXiv 链接** | https://arxiv.org/abs/2609.10873（点击直达） |
| **代码仓库** | ❌ 未开源。原文明确：实验实现、配置文件、源码 hash、命令、原始决策记录与 pilot runs **均未包含在本预印本的 LaTeX 包中**；仅提到三个脚本名 `experiment.py`、`admission.py`、`summarize.py`，未给出仓库地址 |
| **数据集地址** | 未公开。全部证据来自合成标量仿真——论文明确声明无人类参与者、无专有数据、无预训练 VLA、无 LIBERO run、无学习到的视频模型、无实体机器人试验 |
| **类型标签（论文类别）** | `Online` `Reflection` `General` |
| **训练方法标签** | `—（理论/审计方法，不训练 LLM 权重）` |
| **关键词** | update admission、continual learning、paired binomial confidence、missed opportunity、embodied agent、safe policy improvement |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-09_When_Validation_Stops_Learning_Auditing_Update_Admission_for_Continual_Embodied_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **问题**：持续学习 Agent 需不断更新策略，独立评测是标准防线，但**"什么都不放行"的闸门能在"零次有害准入"下让学习器毫无进展**（"validation stops learning"）。
- **重要性**：安全门控几乎只报"违规率"、不报"准入率"，而**恒 REJECT 的规则在违规率上完美无缺**。
- **现有方法不足**：(1) **range-based gate 结构性失效**：`X ∈ {−1,0,1}`，`r_H = √(2 log(2m/α_t)/n)`；**即使每 pair 都 `X = 0`，除非 `r_H ≤ ε` 否则保留性过不了**——`m=3, t=1, δ=ε=0.05` 下要**每次 4,229 对、共 25,374 episodes**，**超 20,000 上限**。(2) **永久初始参照漏掉后续遗忘**：反例 `0.30 → 0.90 → 0.26` **始终落在初始策略 0.05 邻域内**。(3) **代理分数不能充当独立证据**：**换 agent 或新鲜 seed 不消除共享模型偏差**。(4) **审计单元选错**：以 candidate 为单位会把"4 个有效候选只采纳 1 个"算成 75% 浪费。
- **Research Gap**：补 **admission 流程的可审计性**（**规则特定可行性分析**、**32 seeds 可执行诊断**、**不把正常候选筛选误计为错误的 opportunity audit**）。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把一次更新尝试拆成"**候选生成（可用开发反馈）→ 承诺 → fresh paired checks 准入（须独立环境证据）→ 事后全量审计（对学习循环不可见）**"四阶段，用 **paired-binomial 区间**替代 range-based 区间，零分歧下把所需 pair 数从 4,229 降到 **117**。

### 3.2 方法总览（Pipeline）

- **输入/输出**：`π′`、`π`、`b_j`、`P_j`、`S`、`m`、`B` → **ACCEPT/REJECT/DEFER** + 审计报告（coverage、contract violation、**retention violation 单列**、增益、遗忘）。
- **七模块（Fig. 1）**：① 候选生成+开发代理（反馈**只排序**）；② 承诺（候选/参照/任务/样本数/`α_t` 检查前固定）；③ fresh paired checks（`n=⌊B/(2m)⌋` 对，**两侧 episode 都计入**，`X = S(π′) − S(b_j)`，**无可选停止/复用**）；④ 三值判决（`k±` 取 Clopper–Pearson 界，单尾 `β = α_t/(4m)`；**全部 `L_j ≥ q_j` → ACCEPT，任一 `U_j < q_j` → REJECT，否则 DEFER**）；⑤ 认证式参照提升（**仅 `L_j ≥ 0` 才提升**，上限 10 任务，**驱逐即终止保护**）；⑥ 事后全量审计（每任务 4,096 reset 穷尽评测，**结果不进学习循环**）；⑦ round-level missed-opportunity（`E_t` = 第 `t` 轮**至少一候选满足全部真实阈值**；指标 = "`E_t` 成立但无候选被采纳"轮数 ÷ `E_t` 轮数）。

### 3.3 真正的创新点（去伪存真）

- **真·创新**：(1) **admission 独立于 proposal/screening，且约束证据可见性**（承诺后用 fresh pairs，全候选审计**对学习循环不可见**）。(2) **round-level missed-opportunity 指标**：以**承诺候选池**为单位。(3) **规则特定可行性分析**：`k+=k−=0` 时 `n ≥ log β/log(1−ε)`，Hoeffding 需 `n ≥ 2 log(2m/α_t)/ε²`；`(3,1)` 下 **117 vs 4,229**，`(10,8)` 下 **222 vs 8,519**，等分配成本 **4,440 vs 170,380**。
- **工程组合**：paired binomial、Hoeffding、empirical Bernstein、ridge 候选、replay 缓冲——**全既有件**。
- **最关键设计**：决定**本文结论**的是**区间构造本身**——三个 guarded proxy **只差区间**，结果 **0/0/81 次准入（B=2,000）**；决定**最终成功率**的**不是准入规则而是无条件 replay**（frozen 31.4%、paired 59.6%、replay **100.0%**）。
- **证据不足**：(1) **参照提升无实证收益**。(2) **选择乐观性结论建立在玩具环境**（**单步标量** `x′ = a − c a²`、线性代理 32 探针）。(3) **零违规是"观测"非"零风险"**。(4) **"easy local candidate" 把最有利情形送给 paired**：有一个候选**保持先前任务动作不变**，而 low-disagreement 恰是 paired 最大优势区间；**31.6% 是对 paired 最有利设定下的结果**。

---

## 4. 具体技术细节

### 4.1 模型结构

- **不训练 LLM/VLA**：无任何权重训练，无预训练 VLA、无 LIBERO run、无实体机器人试验。
- **策略形态**：准静态标量模型——动作 `a = clip[(θ_shared + θ_j) d, 0, 3]`，终末位移 `g_j (1 + z) a`，成功判据**绝对目标误差 ≤ 0.06**；任务描述符是**已知整数 identity**。
- **环境与代理**：10 个 mobility 均匀排在 0.55–1.6；reset 抽 `d ∼ U(0.5,1)`、`z ∼ U(−0.04,0.04)`。每任务从 32 个探针拟合**线性**代理 `x̂′ = ĝ_j a`，**忽略 z 且被冻结**；320 个预训练探针**在规则间共享**。该模型**省略接触几何、惯性、感知与多步规划**；完整评测含 **1,536** 个设置、**12,288** 个决策、**1,024** 个搜索设置，共 **50.9 秒**。

### 4.2 训练流程

**本论文不训练任何模型权重**（无梯度更新、无 LLM/VLA 微调）；"训练阶段表"记录的是**机制与审计协议**：

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| **不训练**（权重层面） | — | — | — | — | — |

(A) **候选生成**：每任务 32 条解析演示动作，replay 缓冲 320 episodes；4 个候选为 `(θ_shared, θ_j)` 向量，候选 1/2 最小化当前任务模仿损失 + `λ‖θ − θ_parent‖²`（`λ ∈ {0.01, 0.1}`），候选 3 混合历史损失，候选 4 是 task-local 最小二乘更新。(B) **准入审计（Algorithm 1）**：每任务独立 **4,096-reset audit bank**，抽样只在承诺之后；`η = ε = δ = 0.05`，cap `B ∈ {200, 2000, 20000}`，`n = ⌊B/(2m)⌋`，`α_t = 6δ/(π²t²)`；候选与参照 episode **都计入** cap，**无重试、无可选停止**。(C) **事后全量审计**：额外 `368,640` 个 checkpoint outcome 算 final success 与 forgetting，**结果不进学习循环**。

- **两种运行模式**：**common-stream 诊断**（所有闸门针对**同一外部固定 contract stream** 评判）；**closed-loop 运行**（每个 learner 各自分支）。

### 4.3 推理流程（若不训练或重点在推理）

- **不适用（无 LLM 推理）**；重点是**决策流程**。
- **流程**：10 个 mobility 任务顺序到达，**前 2 个 warm-up**，随后 **8 次更新尝试**；每次走 Algorithm 1 六步（生成候选 → 承诺 → 资金不足则 DEFER，否则抽 fresh pairs → 三值判决 → 接受后 `L_j ≥ 0` 时提升旧参照 → 记录证据）。
- **多轮顺序决策循环**，不含 ReAct 式反思。
- **coverage**：准入次数 ÷ **8 次更新机会**。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| **Sequential pushing diagnostic**（论文自建的构造性仿真） | General（具身/机器人持续学习；**非 GUI、非接触丰富机器人**） | 持续策略更新的准入审计（一轮一个候选池，8 次更新机会） | 10 个 mobility 任务（2 warm-up + 8 更新）；每轮 4 个候选；每配置 **256** 个尝试轮；**32 个固定评测 seed**（与 4 个 pilot seed 不相交）；每任务 **4,096** reset 的审计 bank；cap `B ∈ {200, 2000, 20000}` | 标量期望位移 `d ∼ U(0.5,1)`、mobility `g_j ∈ [0.55, 1.6]`、扰动 `z ∼ U(−0.04,0.04)`；任务身份为已知整数 | 每次尝试的 **ACCEPT / REJECT / DEFER** | coverage（准入/8 机会）、contract violations per admission、**retention violations 单独报告**、new-task gain、final success、forgetting（前 9 个任务上"该任务到达后最大成功 − 最终成功"的均值）、**round-level missed opportunity**、全部资源成本；不确定性用 32 条 seed 轨迹的 percentile bootstrap（4,000 次重采样，seed 20260908） |
| **Factorial proxy-feedback stress test**（独立的单步环境） | General（世界模型偏差 vs 选择反馈误差） | 区分**模型偏差**（held-out proxy − true）与**选择偏差**（winning development score − held-out proxy） | 2×2×2×4 设计（random/adaptive × reused/fresh 反馈 × 2 种曲率 × 4 个查询预算 `K ∈ {4,16,64,256}`）× 32 seeds = **1,024** 次搜索 | 单步环境 `x′ = a − c a²`，`c ∈ {0, 0.35}`；动作 `a ∈ [0.2, 1.6]`；目标 `0.65 + u`，`u ∼ U(−0.08, 0.08)`；成功容差 0.06；线性代理从 32 个探针过原点拟合 | 选定动作的 proxy / true score | model gap、selection gap（均为 seed 间 bootstrap 区间，**不是**机器人不确定性） |

### 5.2 实验结果分析

**主结果一：common-stream 诊断（Fig. 2a）——同一候选流、同一契约，只有区间不同**

| 检查规则 | B=200 | B=2,000 | B=20,000 | 观测到的违规 |
|---|---|---|---|---|
| Fresh **Hoeffding**（range-based） | 0 | **0** | 0 | 0 |
| Fresh **empirical Bernstein** | 0 | 0 | **195/256** | 0 |
| Fresh **paired** | 0 | **81/256（31.6%）** | **229/256** | 0 |
| Paired, no proxy | 0 | 84/256 | 232/256 | 0 |

**区间选择直接决定更新机会存活**：paired `B=2,000` 准入 **81/256（31.6%）**、`B=20,000` **229/256**，零违规；**Hoeffding 任何 cap 零准入**；empirical Bernstein 仅 `B=20,000` 放到 195/256；**三者 `B=200` 全 defer**；missed-opportunity：**233 / 152 / 4**。

**主结果二：closed-loop 诊断（Table 1，B=2,000，32 seeds，百分比）**

| 规则 | Adopt | Violate | Miss | Final success | Forget (pp) |
|---|---|---|---|---|---|
| Frozen policy | 0.0 | – | 100.0 | 31.4 | 0.0 |
| Direct fine-tuning | 100.0 | **99.6** | 99.4 | 28.1 | **79.9** |
| Replay update | 100.0 | 5.9 | **0.0** | **100.0** | **0.0** |
| Proxy only | 87.1 | 2.7 | 10.0 | 98.4 | 0.5 |
| Fresh Hoeffding | **0.0** | – | **100.0** | 31.4 | 0.0 |
| Fresh emp. Bernstein | **0.0** | – | **100.0** | 31.4 | 0.0 |
| **Fresh paired** | **37.1** | **0.0** | **60.4** | 59.6 | 0.0 |
| Paired, no proxy | 37.5 | 0.0 | 58.4 | 59.8 | 0.0 |

**paired 准入 37.1%**（[36.3, 37.5]）、**违规 0/95、retention 0**、成功 59.6%、遗忘 0.0，但**漏掉 60.4% 机会**（miss 145/240）；**replay miss 0/241、成功 100.0%**。**Hoeffding/empirical Bernstein adopt 0.0、miss 100.0、成功 31.4%——与"完全冻结"一样**。**replay 的 5.9% 违规是"增益不足"而非保留损失**。**direct fine-tuning**：准入 100%、违规 99.6%、遗忘 **79.9pp**、成功 28.1%。**proxy only** 好（87.1/2.7/10.0/98.4/0.5）但**不用检查 cap**；**paired, no proxy 略升覆盖率**（37.5% vs 37.1%）。


**压力测试（Fig. 2b）**：`c = 0.35`、**256 次自适应查询**下，held-out proxy 成功**高于**真实 **40.8pp**（[25.5, 53.9]）与 **41.9pp**（[26.3, 55.1]）；`c = 0` 时仅 **0.15 / 0.32pp**；selection gap 在 `c = 0.35` 下为 **11.5** [8.2, 14.8] 与 **19.3** [15.8, 22.4]pp。**"换一批新鲜样本"不消除选择乐观性**。

---

## 6. 亮点与贡献（Why it matters）

1. **"missed opportunity" 应被普遍采用**：把"不学"变成可测量量（round-level）；**有安全门控的系统都应同时报告准入率**。
2. **"range-based gate 会冻结学习"变成可复核算术**：`n ≥ log β/log(1−ε)`（117）vs `n ≥ 2 log(2m/α_t)/ε²`（4,229），可**代入自己的 m、ε、δ** 自查。
3. **"审计证据"与"学习反馈"严格分离**：admission 必须用 fresh pairs，**全候选审计结果对学习循环不可见**（呼应 adaptive holdout，Dwork et al. 2015）。
4. **"反馈新鲜度"≠"证据独立性"**：`40.8 vs 41.9` 说明**重抽样不能把选择分数变成独立估计**。

## 7. 局限与可改进点（个人点评）

- **准入契约的价值边界未被正面回答**：Table 1 中最好的"准入策略"是"全接受 + replay 保护"（100% 准入、100.0% 成功、0.0 遗忘），而 paired 只准入 37.1%、成功 59.6%，还花 15,963 个 check episode 换 60.4% miss——**若 replay 已既 100% 成功又零遗忘，admission contract 何时才必要？**
- **覆盖性只有解析论证，无实验验证**：规则正确性依赖四尾 + 任务 + 尝试联合界（`Σ 6δ/(π²t²) = δ`）；**"zero observed violations 只是观测"意味着实际错误率未知**。
- **三个简化叠加 + 代码未发布**：任务 identity 完全可见、准静态标量环境、审计在有限总体上精确可算（"遗忘几乎可解析消除"，而遗忘正是持续学习最核心难点）；且论文称贡献之一是"an executable diagnostic"，但附录 B 明确实现、配置、hash、命令、原始决策与 pilot runs **均未包含在预印本 LaTeX 包中**。

## 8. 对我们的启示 / 可借鉴点

- **准入/门控类设计必须同时报告准入率**：只报"违规率"会漏掉"挡掉了多少本应通过的"——**恒拒绝的守卫在所有错误指标上最优**。
- **"独立证据"与"开发反馈"必须流程隔离，"换一批新鲜采样"也不等于独立证据**：候选可用开发反馈打磨，但一旦承诺就必须用全新环境交互验证且**结果不得回流学习循环**；用模拟器筛选时**换 seed/批次都不消除选择偏差**。
- **"预算不足时规则退化成冻结"可事先算**：`n ≥ 2 log(2m/α_t)/ε²` 自查，不够时应**降低保护任务数或放宽 ε**。

### 对 GUI Agent 的可借鉴点

1. **GUI 技能更新准入可整套搬用**：上线新技能前需在**历史任务**上验证没破坏旧技能；把 `S` 换成 GUI 成功率、pair 换成"同一任务上候选与现任的成败对照"。**paired 构造利用"两者同一次试验结果相同"**。
2. **"零违规"式守卫结论需重审**：必须同时报告**准入率**——挡住 100% 动作的守卫违规指标完美，但 Agent 什么也做不了。
3. **"fresh evidence 不可复用"**：GUI 常用固定任务反复验证会破坏有效性，要求**每次重试用全新数据**；折中是**固定任务集但每次重采样实例**。
4. **模拟器筛选的乐观偏差可预警**：`40.8 / 41.9` 个百分点提醒，**换一批采样不消除乐观性**；应保留**与筛选隔离的 held-out 真值通道**。
5. **事前预算可行性检查**：门控需保护 `m` 个历史任务时，用 `n ≥ 2 log(2m/α_t)/ε²` 估算交互次数，**避免"预算内必然全 defer"的管线**。
6. **把"机会损失"作为常规指标**：按 round-level 统计"这一轮有可用更新但没上线"的比例。

## 9. 延伸阅读

- **准入/安全策略改进**：High Confidence Policy Improvement（Thomas et al., ICML 2015）；Towards Safe Policy Improvement for Non-Stationary MDPs（Chandak et al., NeurIPS 2020，arXiv:2010.12645）。
- **持续学习与抗遗忘**：Experience Replay for Continual Learning（Rolnick et al., NeurIPS 2019，即 CLEAR，本文 replay 基线来源）；LIBERO（arXiv:2306.03310）。
- **世界模型评估**：WorldEval（arXiv:2505.19017，与本文"代理不可作为独立证据"形成直接张力）；Imperfect World Models Are Exploitable（arXiv:2605.15960）。
- **评测有效性方法论**：The Reusable Holdout（Dwork et al., Science 2015）；Selective Classification for Deep Neural Networks（Geifman & El-Yaniv, NeurIPS 2017）。
- **统计工具与自演化 Agent**：Clopper & Pearson（Biometrika 1934）；Hoeffding（JASA 1963）；Empirical Bernstein Bounds（arXiv:0907.3740）；Darwin Gödel Machine（arXiv:2505.22954）。

---
*解读生成时间：2026-09-12 ｜ 解读人：WorkBuddy（AI）*
