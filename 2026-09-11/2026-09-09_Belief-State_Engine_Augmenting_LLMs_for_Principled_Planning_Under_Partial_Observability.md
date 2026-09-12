# Belief-State Engine: Augmenting LLMs for Principled Planning Under Partial Observability

> 一句话 TL;DR：指出 LLM Agent 在部分可观测下失效的根因是"历史条件策略 ≠ 信念可测策略"，提出把贝叶斯滤波器外挂为 Belief-State Engine，只把后验信念喂给 LLM，并用四公理与六定理为这一分工给出健全性保证。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Belief-State Engine: Augmenting LLMs for Principled Planning Under Partial Observability |
| **作者 / 机构** | Arnab Chattopadhayay（第一作者，Independent Researcher，UCL Alumni，印度班加罗尔）；Debdipta Halder（Independent Researcher，IIT-Kharagpur Alumni，印度班加罗尔） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-09；arXiv preprint（cs.AI），页面标注 PREPRINT, APRIL 2026 |
| **arXiv 链接** | https://arxiv.org/abs/2609.10036 |
| **代码仓库** | ✅ 论文脚注声明：https://github.com/debdipta-h/bse-llm （称随预印本发布代码/环境规格/prompt 模板/paired-seed 日志；本次解读未独立验证可用性） |
| **数据集地址** | 未公开。Tiger POMDP 为经典公开基准（Kaelbling et al. 1998）；红队攻击图为作者自建，转移/观测表在论文附录 C，无公开下载地址 |
| **类型标签（论文类别）** | `Planning` `Online` `General` |
| **训练方法标签** | —（无训练；外挂贝叶斯滤波 + 提示工程，LLM 权重冻结） |
| **关键词** | 部分可观测；POMDP；信念状态；LLM Agent 失效模式；贝叶斯滤波；组合健全性 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | 2026-09-09_Belief-State_Engine_Augmenting_LLMs_for_Principled_Planning_Under_Partial_Observability.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：通用 LLM Agent 在部分可观测环境（延迟/含噪/带欺骗反馈）下的多步序贯决策。非 GUI，用 Tiger POMDP 与红队攻击图。
- **为什么重要**：卡住了 Agent 从"下一步能靠当前观测决定"走向需主动探测的长时程任务。三种症状：证据模糊就过早承诺、单条观测就把不确定性坍缩到错假设、history 变长后策略漂移。
- **现有方法有什么不足**：论文把 ReAct / Reflexion / ToT / Voyager / SWE-agent 统一写成 $\pi_{LLM}(a|h_t)$，指出两点：(1) **surface sensitivity**——不同 history 可有同一后验，但序列化后文本不同，动作分布随之不同，连信念可测性都不满足；(2) **无显式滤波器**——没有跨步携带并归一化 $S$ 上的分布，LLM 每次要从原始文本重猜后验。故 CoT、长上下文、反思循环累积的是**文本**而非**概率质量**。
- **Research Gap**：缺一个"LLM 外部、可审计、POMDP-grounded、闭式维护、可与任意 LLM 组合、且有组合性定理"的信念表示。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把维护信念的职责从 LLM 拿走，交给跑在 POMDP 模型 $(T,Z)$ 上的外部两步贝叶斯滤波器，每步只把归一化后验 $b_t\in\Delta(S)$ 作为 LLM 的**唯一**决策上下文。

### 3.2 方法总览（Pipeline）

- **输入/输出**：$(a_{t-1},o_t)$、$T,Z,\mu_0$、固定 prompt 与域 header → `ACTION: <name>`。原始 action-observation trace **永不进入 prompt**（Axiom A4 的实现约束，也是全部定理的生效前提）。
- **模块一：BSE**。两步滤波：预测 $ar b_{t+1}(s')=\sum_s T(s'|s,a_t)b_t(s)$，再用似然 $\ell(s')=Z(o_{t+1}|s',a_t)$ 校正归一化；log 空间计算防下溢；零似然观测按先验延拓并告警。$O(|S|^2)$ 时间、$O(|S|)$ 内存。
- **模块二：Serialiser $\sigma$**。tabular / top-$k$（余量折叠为 residual）/ full-support sorted 三种，均显式给支撑集且概率归一；另附与 $t$ 无关的域 header 声明 $S,A$ 与奖励结构。
- **模块三：LLM 策略槽 + parser**。LLM 每步**全新调用**（per-step reset）；parser 只做 exact-match，畸形输出先 $	au$=0 重采样，再失败落默认动作，fallback 率作为一等指标记录。

### 3.3 真正的创新点（去伪存真）

**真·方法创新**
1. 把"部分可观测下失效"升级为**可证伪的结构性命题**：$\pi_{LLM}$ 不是 $\sigma(b_t)$-可测而是 $\sigma(\phi(h_t))$-可测，而 $\phi$ 有损、对顺序敏感、对语义等价改写不不变。
2. **四公理刻画**（A1 递归可更新 / A2 预测充分性 / A3 概率内化 / A4 信念可测策略）：canonical posterior 是满足 A1–A3 的**最粗**表示（Thm 1–3）、贝叶斯更新是唯一相容算子（Thm 4）、策略与值等价（Thm 5–6）、双模拟下歧义保持（Thm 7–8）。
3. **Thm 9（组合健全性）**：LLM 只经 $b_t$ 条件化时组合体就是信念 MDP 上的 Markov 策略，并明确**失败模式**：暴露原始 history 即违反 A4、保证失效。

**工程组合**：贝叶斯滤波、QMDP、POMCP、序列化、parser 全是已有组件；把 LLM 放在"信念→动作"槽位是架构主张，组件层面无新算法。

**对性能最关键的设计**：消融指向**观测校正步**而非预测步。AB2（去校正）在 Tiger 上成功率掉到 1/25（4.0%），return −16.08，信念熵钉死在 $\ln 2$；AB1（去预测）几乎无影响，因为 `listen` 的转移核就是恒等。增益主要来自"把观测持续压进后验"，而非贝叶斯滤波的完整形式。

**证据不足 / 仅声称有效**
- 六 baseline 只跑三个，**QMDP、POMCP 完全没跑**——这两个恰是唯一能回答"LLM 是否真比经典 POMDP 规划器强"的对照。
- 十消融只跑三个，其中 **AB7（暴露原始 history）是验证 A4 的关键消融，没有做**。
- 攻击图的 $T$ **对所有动作都是恒等矩阵**（patch/exploit 动态未实现），使"攻击图压测信念维护"的设计意图落空。
- 消融的 "Standard BSE" 与主实验 BSE 行数字不一致（Tiger 84.0% vs 95.0%），作者归因于 gpt-4o 跨调用不确定——等于承认 LLM 侧结论不可复现。

---

## 4. 具体技术细节

### 4.1 模型结构

- Base model 是**纯文本 LLM：GPT-4o**（$	au$=0.3、top-p=0.95、max 256 tokens）。**无视觉编码器，不是 MLLM**；参数量未公开（闭源）。**完全冻结、零微调**，唯一适配手段是 system prompt 与序列化格式。
- 第二个"模型"是 BSE：**不是神经网络**，是 <400 行 Python 模块（`POMDPModel` / `BeliefFilter` / `Serialiser`）。环境无关：从 Tiger 换到攻击图只需替换 `POMDPModel` 实例与域 header。

### 4.2 训练流程（若需要训练）

**不训练。** 无权重更新、无 RL、无 SFT、无蒸馏：

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| — | 不训练（training-free） | 无 | 无 | 无 | 无 |

$T$ 与 $Z$ 由**环境规格直接给出**（有限状态下行随机矩阵；有结构时可传 callable），论文明确说"从数据学信念状态模型是另一条线"（section VIII）。唯一被"设定"的是 prompt 的决策规则与序列化器选择，二者**人工设定并作为实验轴扫描**。指标族、六 baseline、十消融在跑主 LLM **之前**已固定。

### 4.3 推理流程

- **输入**：$\sigma(b_t)$ token 串；**输出**：`ACTION: <name>`，经 parser 得 $a_t$。
- **多步推理**。每 episode 为 $T$ 步循环（Tiger 20，攻击图 30），每步一次 LLM 调用。机制不是 ReAct 式累积，而是**滤波-决策分离**：BSE 更新 $b_t$（无 LLM 参与），LLM 只做 $b_t	o a_t$。终止条件为 horizon 到达或（Tiger）正确开门。**LLM 调用无状态。**

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| Tiger POMDP（经典） | 非 GUI（通用 Agent / POMDP） | 部分可观测序贯决策（探测后承诺） | $S$=2，$A$=3，$O$=2；T=20，γ=0.95；$N$=40（消融 25） | 序列化信念 $b_t$ + 域 header | `ACTION: <name>` | 成功率、return、校准（Brier/NLL/熵）、成本、JSD |
| 红队攻击图（自建） | 非 GUI（安全运维决策） | 攻击图上的探测/利用/修补规划 | K=4（$|S|$=16）与 K=6（$|S|$=64）；$A$=3K+1；T=30，γ=0.95；FPR 0.1、FNR 0.15；$N$=40（消融 25） | 同上 | 同上 | return、coverage、利用成功率、≥1 次入侵率、首次入侵步数、校准、JSD |

> 六 baseline 与十消融的**完整设计**见 section VI，但实际只跑了三个 baseline（Reactive/BSE/NL-Tracker）与三个消融（AB1/AB2/AB9）。

### 5.2 实验结果分析

**主结果（Tiger，$N$=40，gpt-4o，τ=0.3）**

| 指标 | Reactive | BSE | NL-Tracker |
|---|---|---|---|
| 成功率 | 32/40（80.0%） | **38/40（95.0%）** | 32/40（80.0%） |
| 平均折扣 return [95% CI] | −12.00 [−25.75, 1.75] | **3.06 [−4.41, 8.24]** | −12.00 [−25.75, 1.75] |
| 平均 listen 次数 | 0.00 | **1.45** | 0.00 |
| Brier / NLL / 熵 | 0.325 / 0.509 / 0.423 | **0.234 / 0.367 / 0.365** | 0.500 / 0.693 / 0.693 |
| token / LLM 调用 per episode | 459 / 1.00 | 1,270 / 2.45 | 249 / 1.00 |

Reactive 与 NL-Tracker **逐项相同**：都在第一步就开门赌 85% 准确的读数，0 次 listen；BSE 先 listen 1.45 次再承诺，是三者中唯一 CI 不含 0 的。

**攻击图 K=6（$|S|$=64，$N$=40）**：三者 return CI 大幅重叠，点估计 NL-Tracker 7.87 [−5.2, 21.1] > BSE −2.80 [−16.0, 10.7] > Reactive −5.40 [−17.4, 7.0]，**统计不可区分**。BSE 的 coverage 最高（42.1% vs 32.5%/40.8%），首次入侵更晚（2.58 步 vs 1.41）。唯一明确分开 BSE 与 NL-Tracker 的是**决策一致性 JSD**：BSE median 0.0 [0.0, 0.229]，NL-Tracker 0.043 [0.0, 0.693]（$n$=24 pairs）。

**Ablation 说明了什么**

| 配置 | Tiger 成功率 | Tiger return | Tiger 熵 | 攻击图 return | 攻击图 coverage | 攻击图 熵 |
|---|---|---|---|---|---|---|
| Standard BSE | 21/25（84.0%） | −9.23 | 0.350 | 6.10 | 47.3% | 3.155 |
| AB1（去预测步） | 23/25（92.0%） | −0.19 | 0.368 | −0.01 | 44.0% | 3.199 |
| AB2（去观测校正） | 1/25（4.0%） | −16.08 | 0.693 | −6.07 | 26.7% | 4.159 |
| AB9（τ=1.0） | 21/25（84.0%） | −8.83 | 0.364 | 11.79 | 46.7% | 2.875 |

三条结论：(1) **观测校正步不可缺**——AB2 在两域都把后验钉死在均匀先验（熵 = ln|S|），是唯一灾难性退化；(2) **预测步可有可无**——Tiger 的 `listen` 与攻击图的 $T$ 都是恒等，AB1 近零效应，作者自认这在攻击图上"是环境缺陷而非实证发现"；(3) τ=1.0 未见显著变差（也可能功效不足）。Tiger 上决策一致性探针**零个合格 history 对**。

---

## 6. 亮点与贡献（Why it matters）

1. **把 Agent 失效从"提示工程问题"改写成"表示问题"**。$\pi_{LLM}$ 不是 $\sigma(b_t)$-可测这句话可直接诊断任何 memory/scratchpad 设计：只要两条后验等价的历史触发不同动作分布，该设计就违反信念可测性。
2. **给出可审计的中间量**：$b_t$ 每步都是完整可打印的概率向量，可监控、可在零似然事件上定位 $(T,Z)$ 规格错误。
3. **架构与骨干解耦 + 诚实的自我披露**：换环境只需换一个 `POMDPModel` 实例，任何 LLM 都能塞进策略槽；section VI 还明说 10⁵ 次 API 超预算、N=40 而非 900、开权重复现未跑。

## 7. 局限与可改进点（个人点评）

- **理论-实验落差明显**。最重的理论资产是 Thm 9，而它最关键的失效条件（违反 A4、暴露原始 history）对应的 AB7 **没跑**——"保证是必要的"只有证明、没有证据；撑起性能的 AB2 讲的只是"要持续做观测校正"，宣传重心与证据重心错位。
- **与经典规划器的对比缺失是硬伤**。QMDP/POMCP 未跑，无法回答"BSE+LLM 相对 1995 年的 QMDP 启发式赢在哪"；Tiger 这种 $|S|$=2 的域里最优有限时域策略本可轻松超过 95%，"95% 成功率"含金量大打折扣。
- **环境规模过小**。$|S|$=2 与 64 都在精确滤波可行的范围内；section VIII 列出 particle / amortised variational / factored POMDP 三条扩展路线，但**一条都没试**。
- **模型依赖是真实隐忧**。GUI/Web 场景里 $T,Z$ 几乎不可能手工写全；论文承认模型误规格时"所有保证失效"，给出的 Bayes-adaptive 与学习式转移模型路线只是文字讨论——而这恰是搬到 GUI Agent 上最大的工作量。
- **可复现性被 provider 非确定性侵蚀**。同 prompt 同温度下 gpt-4o 跨调用不一致，已导致内部数字对不上；作者的缓解手段在**实际执行的子集里一个都没完整落地**；框架也停留在单智能体、平稳设定。

## 8. 对我们的启示 / 可借鉴点

- **把"信念可测性"当作 memory 设计的验收指标**：构造语义与证据等价但顺序/措辞/长度不同的历史对，比较动作分布的 JSD。**无需标注**，可直接评估 scratchpad、记忆压缩、检索式记忆是否引入 surface sensitivity。
- **观测校正不可省，预测步可以省**：在线 Agent 优先保证"每步把新观测压进状态"；算力紧张时先砍状态预测——至少在转移接近恒等（GUI 的等待/滚动就是恒等）时成立。
- **fallback 率应作为一等指标上报**；**代价要提前算清**：显式信念等价于需要一个页面状态机/元素状态模型，降级路线（Bayes-adaptive、learned transition、particle/variational）才是要做的工程。

### 对 GUI Agent 的可借鉴点

1. **把"页面状态后验"外置，而不是让 LLM 从截图历史里自己猜**。GUI 观测天然部分可观测（遮挡、未加载完、异步渲染、模态浮层）。可维护离散状态后验（"当前页面属哪一类""目标元素是否可交互""上一步是否生效"），序列化成结构化数值喂给策略模型，而非把 N 张截图 + N 条动作全塞进 context。
2. **给"信息收集动作"留预算，并当作一等动作类型**。BSE 多花 1.45 次 listen 换到 95% 成功率，而 Reactive/NL-Tracker 第一步就开门、只有 80%。GUI 的对应物是先滚动/等待加载/截图确认再点击；"看到模糊 UI 就立刻点"与 premature commitment 同源。
3. **用等价历史对的 JSD 检测 grounding 漂移**。同一页面状态的不同到达路径、DOM 顺序、截图裁剪构成后验等价的历史对；若动作分布在这些对上发散，说明策略依赖 surface 特征而非页面语义——这是坐标幻觉与元素误指的早期信号。
4. **决策一致性可补足稀疏的成功率信号**。GUI 评测成功率方差大、样本昂贵；JSD 能在成功率还看不出差异时给出信号（本文攻击图即如此）。
5. **LLM 调用无状态化 + per-step reset**，杜绝 token 级状态跨步泄漏。

## 9. 延伸阅读

- Kaelbling, Littman & Cassandra, AIJ 1998 —— Tiger 规格与信念 MDP 的来源。
- Silver & Veness（POMCP）, NeurIPS 2010；Littman et al.（QMDP）, ICML 1995 —— 未跑但最该跑的对照。
- ReAct（ICLR 2023）、Reflexion（NeurIPS 2023）、ToT（NeurIPS 2023）—— 被本文批判为 history-conditioned policy 的代表。
- Sumers et al., *Cognitive Architectures for Language Agents*, TMLR 2024 —— LLM Agent 架构总览。

---
*解读生成时间：2026-09-11 09:00 ｜ 解读人：WorkBuddy（AI）*
