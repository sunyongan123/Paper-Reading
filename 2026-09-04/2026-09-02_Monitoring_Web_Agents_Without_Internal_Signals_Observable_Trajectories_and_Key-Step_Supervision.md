# Monitoring Web Agents Without Internal Signals: Observable Trajectories and Key-Step Supervision

> 一句话 TL;DR：不碰模型内部信号，仅凭可观测轨迹前缀 + 关键失败步监督，即可在线预测 Web agent 的失败风险。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Monitoring Web Agents Without Internal Signals: Observable Trajectories and Key-Step Supervision |
| **作者 / 机构** | Sitong Pan、Yilin Lu、Caiwen Ding、Qianwen Wang（明尼苏达大学，美国）；Yipeng Shen（普渡大学，美国）；Lu Cheng（宾夕法尼亚州立大学，美国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-02；arXiv preprint（arXiv:2609.02057v1，cs.AI，未标注会议/期刊） |
| **arXiv 链接** | https://arxiv.org/abs/2609.02057 |
| **代码仓库** | ❌ 未开源（正文及附录均未声明 GitHub / 项目主页） |
| **数据集地址** | 未发布自有数据集；评测沿用公开基准 WebArena-Lite 与 Online Mind2Web，无独立下载链接 |
| **类型标签（论文类别）** | `Web` `Online` `Benchmark` `General` |
| **训练方法标签** | —（评测/监控方法研究：黑盒 Web agent 前缀级失败风险预测，非模型训练论文） |
| **关键词** | Prefix-level failure prediction；Observable trajectory signals；Key-step supervision；Black-box uncertainty；Web agent monitoring；Early intervention |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-02_Monitoring_Web_Agents_Without_Internal_Signals_Observable_Trajectories_and_Key-Step_Supervision.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：在**通用 Agent 场景下的 Web 导航**任务中，给定一条仍在进行中的执行前缀 $\tau_{\le k}$，实时估计"本次执行仍在正轨上（on track）还是正滑向失败（tending toward failure）"，即前缀级失败风险预测 $r_{i,k}=P(y_{i,k}=1\mid\tau_{i,\le k})$。

- **为什么重要**：Web agent 走的是"看页面→推理→执行动作"的多步循环，一旦走上注定失败的路，继续推理和操作只是白白烧 token 和时间；而等到任务终结才发现失败，又错过了挽回/接管时机。**提前感知失败倾向**是生产环境做人工接管、成本控制、选择性重试的直接前提——这本质是 Agent 部署的"监控哨兵"问题。

- **现有方法有什么不足**：论文把缺陷归结为两个结构性困难。
  1. **输入可观测性**：主流 UQ 方法依赖 token logits、hidden states（HTC、MARS 等），闭源 agent（如 Claude）根本不暴露；而黑盒方法（verbalized confidence、采样一致性）只评估**孤立的单步输出**，不做"随时间演化的轨迹序列"的累积判断。
  2. **前缀监督**：任务只给一个稀疏的最终标签。若把失败结局直接传播到所有前缀，会把失败轨迹开头的有效行为误标为"失败"——而同样的早期行为也可能出现在成功执行里，造成**相互矛盾的监督**。

- **Research Gap**：作者主张补上的是"**在完全黑盒、只看可观测轨迹的前提下，做时间对齐的前缀级失败风险预测**"这一空白——此前最接近的 PrefixGuard（outcome 式监督）和 HTC-Full（内部信号）都未能同时解决"黑盒输入"与"时间对齐监督"两个问题。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把监控拆成"**两类可观测轨迹特征 + 一个轻量风险预测器**"，并用"关键失败步"把稀疏的失败标签对齐到前缀时间轴上，全程不触碰模型内部。

### 3.2 方法总览（Pipeline）

- **输入**：变长轨迹前缀 $\tau_{\le k}$（任务指令 $q$、历史决策文本 $R$、执行动作 $A$、环境反馈 $O$）。
- **输出**：前缀失败风险标量 $\hat r_{i,k}\in[0,1]$。
- **模块**：
  1. **Macro 特征**（31 维）：直接"读已发生的日志"，统计跨步行为模式——动作重复/单双循环、动作使用次数、grounding/parse 错误、思考文本自报的"动作无效/错上下文/环境出错"等累积症状。
  2. **Micro 特征**（18 维）：每步用结构化提示反复探测 agent（N=10 次采样），让它输出"当前意图/可执行动作/预期状态变化"，用 Qwen-0.6B 编码器做语义聚类，从经验分布里估出 6 个代数上不冗余的不确定性指标（意图熵、grounding 熵、条件状态熵、意图贡献、残差动作熵等），再以 mean/max/current 聚合。
  3. **预测器** $f_\theta$：ℓ2 正则 logistic 回归，把信号 $z$ 映射为风险分。
  4. **关键步标注器**（离线）：两阶段 LLM-as-judge（Gemini-3.5-Flash）+ 人工提炼的 codebook，为每条失败轨迹标出关键失败步 $k^\star$。
- **连接**：Macro/Micro 特征拼接 → 前缀长度归一化 + 标准化 → 喂给 logistic 预测器；$k^\star$ 只用于构造监督标签 $y_{i,k}$（$k\ge k^\star$ 且最终失败则为 1），不参与线上推理。数据流上，Micro 需要额外的重复采样与聚类（推理期开销），Macro 无需额外查询。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：① 系统化定义并拆解"可观测轨迹信号"两族（Macro 行为反馈 + Micro 决策一致性），证明它们在**黑盒**条件下可媲美内部信号——这是首次在 web agent 前缀监控上的完整验证；② **Key-step-aware 前缀监督**：把失败标签从"整条一刀切"改为"只在关键失败步之后传播"，本质是一种 step-level credit assignment 的标注配方。
- **工程组合**：logistic 回归是极简单的现成分类器；Macro 里的重复/循环/错误计数、Micro 里的语义熵（semantic entropy, Kuhn et al.）都是已有概念，本文把它们**组合进"轨迹前缀监控"这个新场景**并统一成固定维特征向量。
- **对性能提升最关键的设计**：消融明确指向**监督方式**（关键步 vs 最终结局）是最大单一贡献——γ=0.5 时把成功轨迹的误切率从 87.3–93.0% 压到 27.6–28.6%；其次 Macro 单独即强、Micro 是 benchmark 依赖的增益。
- **证据不足 / 仅声称有效**：论文声称"可观测信号足以监控"，但 Stacked L2 对照组特征维度远小于 49 维（作者自己在 Limitations 承认维度/容量未匹配），因此"监督收益 vs 特征维度收益"并未完全隔离；Micro 在 WebArena-Lite 上无一致增益，说明其价值高度依赖环境特性。

---

## 4. 具体技术细节

### 4.1 模型结构

本篇是**监控/评测方法研究，不训练 agent 本身**，涉及四类"模型"角色，需区分清楚：

| 角色 | 模型 | 类型 | 作用 |
|---|---|---|---|
| 被监控的 agent backbone | Qwen3-VL-30B、Kimi 2.5（开源）；GPT-5.2、GPT-5.4-nano、Claude Sonnet 4.6（闭源） | **MLLM（VLM，含视觉编码器）** | 执行 ReAct 式 web 导航，产出轨迹 |
| Micro 语义编码器 | Qwen-0.6B（Qwen3 系列 sentence encoder） | LLM（纯文本） | 把意图/状态变化描述嵌入后做 agglomerative 聚类（cosine 阈值 0.3） |
| 风险预测器 | ℓ2 正则 logistic 回归 | 线性分类器 | $f_\theta(z)=\sigma(w^\top z+b)$ |
| 关键步标注 judge | Gemini-3.5-Flash | LLM | 离线两阶段标注 $k^\star$ |

### 4.2 训练流程（若需要训练）

**本篇不训练 agent**——它研究的是"黑盒 agent 的失败风险监控"，因此没有 SFT/RL 阶段。唯一的"有监督学习"是**拟合 logistic 预测器**，机制如下：

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| 预测器拟合 | 学前缀信号→失败倾向映射 | 风险标定 | 2,183 条轨迹的每一步前缀 | 信号向量 $z_{i,k}$（31/18/49 维）+ 前缀标签 $y_{i,k}$ | 正则化二分类交叉熵（ℓ2）；5 折按任务切分 CV，长度归一化 + 训练折统计标准化 |

关键点：**监督标签 $y_{i,k}$ 由关键失败步 $k^\star$ 构造**（最终失败且 $k\ge k^\star$ 才为 1），而非直接把最终结局贴到所有前缀。$k^\star$ 的标注可靠性有 150 条轨迹背书：80.0% 关键步落在 ±1 步内，人机前缀标签一致率 89.8%，三次独立运行两两一致 90.9%、单次与多数一致 95.5%。

### 4.3 推理流程（若不训练或重点在推理）

- 推理输入：前缀 $\tau_{\le k}$；输出：风险分 $\hat r_{i,k}$。
- **Micro 是多步/重采样推理**：每步固定上下文采样 N=10 次 → 解析三字段 → 语义聚类 → 估 6 个熵指标 → mean/max/current 聚合成 18 维；Macro 则是纯前缀统计，无额外查询。
- **线上干预机制（early-cut policy）**：给定阈值 $\gamma$，在第一个 $k_\gamma<T$ 使风险超过 $\gamma$ 的步截断轨迹；失败轨迹只有在 $k_\gamma\in[k^\star,T)$ 时才计为"检出"，成功轨迹被截断则计为"误切"（FCR）。这比 E-AURC 的独立前缀排序更贴近真实部署的"状态式"决策。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| WebArena-Lite | Web（可复现 Docker） | 端到端导航 | 165 任务 / 774 轨迹 | 截图 + 指令 + 历史 | 动作 + 环境反馈 | Brier↓ / E-AURC↓ / AUROC↑ |
| Online Mind2Web | Web（136 个真实在线网站） | 端到端导航 | 300 任务 / 1,409 轨迹 | 截图 + 指令 + 历史 | 动作 + 环境反馈 | Brier↓ / E-AURC↓ / AUROC↑ |

总计 2,183 条轨迹，5 个 backbone（开源 Qwen3-VL-30B、Kimi 2.5；闭源 GPT-5.2、GPT-5.4-nano、Claude Sonnet 4.6），ReAct 框架。失败原因分布（Table 4）：Action 29.8%/30.8%、Planning 37.1%/32.8%、Reflection 26.4%/27.6%、Memory 3.9%/3.2%、System 2.8%/5.6%（WebArena-Lite / Mind2Web）。

### 5.2 实验结果分析

**主结果**（对比 HTC-Full 这一最强内部信号基线，及 verbalized confidence 等标量信号）：
- 可观测信号在 Mind2Web 的 **15/15** backbone×指标组合上持平或超过最强 UQ 基线，WebArena-Lite 为 **9/15**。
- Claude（内部信号不可用）：M&M 的 AUROC 达 **0.748 / 0.777**（WebArena / Mind2Web），远高于 verbalized confidence 的 0.645/0.693；GPT-5.4-nano 在 Mind2Web 以 **0.807** 胜过 HTC-Full 的 0.736。
- 但开源 Qwen3-VL-30B 上 HTC-Full 达 0.858，仍高于 M&M 的 0.838——**内部信号在开源模型上依然是最强基线**，这是作者没藏着的反例。
- Stacked L2（把同一 logistic 预测器套在标量 UQ 信号上）在 E-AURC/AUROC 上十个设置全输，说明优势**不来自"套了有监督分类器"本身**。

**Ablation 说明**：
- **信号分工**：Macro 单独更强更稳；Micro 并入后对 Mind2Web 五个 backbone 三指标全提升（配对 t 检验 p=0.023/0.032/0.012），但 WebArena-Lite 无一致增益——Micro 的价值是 benchmark 依赖的。
- **监督方式**（核心消融）：关键步监督把成功轨迹（5,299 前缀）误切率从 87.3–93.0% 压到 27.6–28.6%（γ=0.5）。
- **固定误切预算的干预**：8 个非 Claude 设置、1,576 轨迹交集上，可观测配置在 8 个工作点里 7 个超过 HTC-Full；20% FCR 预算下检测率 44.3%/44.5%，高于 HTC-Full 的 41.9%/40.4%。
- **跨类别泛化**：5 类网站留一类、size-matched 子采样后，M&M 与域内训练差距仅 AUROC −0.0005 / E-AURC −0.0039 / Brier +0.0030，远小于类别间波动（≈0.013），平均 Brier 0.191 优于 HTC-Full 0.204（但 E-Commerce 类 HTC-Full 全胜）。
- **采样预算**：Micro 在 N=5~8 即达平台期（N=5 保留 N=10 一半 decode 成本、53.8% 串行延迟，AUROC 增益 +0.0148），N>8 无提升。
- **前缀时间**（Appendix H）：内部信号在轨迹早期更强（HTC-Full early AUROC 0.723/0.718），可观测信号在中后期反超（Macro late 0.778，M&M late 0.779）——即**"早期看置信度、后期看行为历史"**的时序分工。

---

## 6. 亮点与贡献（Why it matters）

1. **打通了黑盒 agent 监控的可行性**：首次系统验证"纯可观测轨迹信号"足以支撑前缀级失败预测，让不开放 logits/hidden states 的闭源 agent 也能被监控，这是 Agent 落地绕不开的痛点。
2. **信号拆解清晰且有理论落点**：Macro（读行为反馈）与 Micro（探决策一致性）从"做了什么"和"想得是否一致"两个正交角度切入，并用案例说明它们对同一失败在不同时刻报警（Macro 在重复点击即报警，Micro 在十次采样高度分歧时才报警）。
3. **Key-step 监督是通用的 step-level credit assignment 配方**：避免把失败标签无脑回灌早期有效行为，且有三层可靠性统计背书，可直接迁移到 SFT 数据筛选 / RL 奖励标注。
4. **实验纪律出色**：与 HTC-Full 同口径对比、Stacked L2 控制监督收益、size-matched 跨类别评测、固定 FCR 预算报告操作点、聚类鲁棒性（3 编码器 × 3 阈值）与采样成本曲线一并给出，结论可信度高。
5. **面向生产的决策框架**：在固定"误切预算"下提前切断/接管正在失败的任务，直接对接人工接管与成本控制诉求。

---

## 7. 局限与可改进点（个人点评）

- **架构覆盖窄，泛化存疑**：只在 ReAct 式单 agent 上验证，预测器仅是线性 logistic 回归；多 agent、规划-执行、反思式框架以及更强序列模型（如直接对前缀做序列建模）的外推性完全未知。
- **"可观测"仍有一层隐藏假设**：Macro 依赖 agent 暴露的决策/思考文本，Micro 依赖结构化输出；只暴露"动作+环境反馈"的商用 agent 连这些都没有（作者也列为 future work），离"真正的极黑盒"还有距离。
- **监督的因果性没闭环**：$k^\star$ 由 LLM judge 事后看完整轨迹标出，是"观测到未被纠正的错误"，不是客观的不可恢复状态；离线的"关键步"与线上实时可用的风险之间隔着一条较长的标注链，最坏情况下是"用事后信息训了一个看似可用的线上预测器"。
- **对照组未完全公平**：Stacked L2 的特征维度远小于 49 维，作者自己承认"监督收益 vs 特征维度收益"未隔离——这是结论"优势来自特征内容而非监督拟合"的一个软肋。
- **评测域有限**：仅 Web 域，移动/桌面 GUI 未测；跨类别实验共享同批 backbone 与站点池，模型换代、环境非平稳、分布漂移后是否需重训均未讨论。
- **成本是隐性约束**：Micro 每步 N=10 次采样 + 语义聚类开销不小，虽有 N=5~8 的平台期结论，但部署时"每步多花几倍推理量换一个风险分"是否划算，取决于具体 token 价格。

---

## 8. 对我们的启示 / 可借鉴点

**对 GUI Agent 的可借鉴点**（本篇属 Agent 方法论域的"监控"方向，特单列）：
- **监控哨兵模板可直接移植**：不侵入模型、无需 logits，仅凭"页面观测 + 动作 + 环境反馈"喂给轻量分类器即出风险分——与我们做失败预警/人工接管的诉求高度契合，可照搬"固定误切预算下调阈值"的决策框架。
- **Macro 行为信号接近领域无关**：重复点击、双动作循环、连续 grounding/parse 错误、思考文本自报"动作无效/换页"等运行时症状，做移动端/桌面端可先套用再增量扩展，成本几乎为零。
- **关键步监督是通用标注配方**：凡用失败轨迹做训练数据，都应先标"何时走偏"再给标签，避免把失败标签无脑回灌早期步——这本质是 step-level credit assignment，能显著减少对有效早期行为的误伤，值得引入我们的 RL 奖励 / SFT 数据管线。
- **"早期置信度 + 后期行为历史"的时序分工**（Appendix H）是一个可落地的组合思路：若我们的 agent 有内部信号，可在早期用 logits、后期用轨迹特征，两者互补。
- **成本—价值量化意识**：采样预算实验把 decode 成本、延迟模型与收益拐点一起给出，提示凡引入运行时重复查询都应附上类似成本曲线，方便部署取舍。

---

## 9. 延伸阅读

- **PrefixGuard**（Huang et al., 2026, arXiv:2605.06455）：同样做轨迹前缀失败监控，但用 outcome 式监督，是理解本文"关键步监督"改进点的直接对照。
- **Agentic Confidence Calibration / HTC**（Zhang et al., 2026d, arXiv:2601.15778）：主基线 HTC-Full 出处，代表"把 token 级置信度统计映射为任务失败概率"的内部信号路线。
- **AgentRx**（Barke et al., 2026, arXiv:2602.02475）：从事后执行轨迹诊断并定位关键错误步，"关键失败步"概念与两阶段标注协议由此而来。
- **Web-Shepherd**（Chae et al., 2026, NeurIPS 2025）：用过程奖励模型做步级网页导航评估，是"逐步监督导航轨迹"的另一互补路线；向上可溯至 **Where LLM Agents Fail**（Zhu et al., 2025）的 AgentErrorTaxonomy 失败分类法。

---
*解读生成时间：2026-09-04 ｜ 解读人：WorkBuddy（AI）*
