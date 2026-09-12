# Building the Harness Automatically: Self-Play in Code Distills a Text Harness for Black-Box Optimization

> 一句话 TL;DR：让 Agent 在代码沙箱里反复写优化器并配对评测，把"如何花预算"的搜索纪律一次性蒸馏成 197 词的文本 harness，密封冻结后在未见模型与目标上把 Gemini Flash 的 regret 降 48%。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Building the Harness Automatically: Self-Play in Code Distills a Text Harness for Black-Box Optimization |
| **作者 / 机构** | Yi Wu\*、Zheng Ren\*（\*同等贡献）、Zhiyu Hu、Haochen Wang、Daryl Chang、Li Wei、Ting Wang、Zhen Li、Pooja Gupta、Nitin Jindal、Lukasz Heldt；**Google**（Mountain View, USA），全部作者 @google.com |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-08；arXiv preprint（cs.LG） |
| **arXiv 链接** | https://arxiv.org/abs/2609.09468 |
| **代码仓库** | ✅ 论文脚注给出**匿名仓库**：https://anonymous.4open.science/r/llm-opt-sandbox-E5B3/README.md （随附 Harness A 文本、cache 与 manifest；含 code-only 蒸馏文本但不含全部 per-instance 轨迹） |
| **数据集地址** | 未公开。练习族（随机平移正定二次函数）与 held-out BBOB 变换版为作者自建；内部 YouTube 奖励调优基准为**保密历史数据集**（per-replicate 数据与代码均不公开）；BBOB 本身为公开基准（COCO platform，Hansen et al. 2021） |
| **类型标签（论文类别）** | `Distillation` `General` |
| **训练方法标签** | —（无权重训练；代码级自博弈实践 + 一次性文本蒸馏、密封冻结后零样本部署） |
| **关键词** | 黑箱优化；文本 harness；自博弈；一次性蒸馏；评测密封；跨执行器迁移 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | 2026-09-08_Building_the_Harness_Automatically_Self-Play_in_Code_Distills_a_Text_Harness_for_Black-Box_Optimization.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：低预算黑箱优化——极少查询预算下优化昂贵、无梯度目标。作者要的不是"让 LLM 更会算"，而是**学到开发后依然可靠的搜索行为**，并变成可移植、可审计的产物。
- **为什么重要**：LLM 能读懂优化历史，却因缺少结构支撑而是糟糕的数值优化器；若能自己练出搜索纪律，同一范式可迁移到任何"有可执行验证器但无梯度"的场景。
- **现有方法的不足**：**从测试结果反向修改指令会泄漏评测信息**——OPRO、GEPA、Meta-Harness 都在评测中持续修订 artifact；FunSearch / AlphaEvolve 交付的则是绑定实现的**程序**。
- **Research Gap**：需要"开发目标上练一次 → 门控 → 蒸馏 → 冻结"的搜索行为，且 artifact 一字不改即可跨执行器迁移。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

用固定 Agent 在代码沙箱做"挑战者 vs 在位者"配对自博弈（practice），开发集门控（gate），把冠军程序与实践记录**一次性**蒸馏成短文本策略（distill），再密封冻结（freeze），此后只作 prompt 前缀零样本部署。

### 3.2 方法总览（Pipeline）

- **输入 / 输出**：输入初代在位程序 $a_0$、开发目标族与预算 $B_{dev}$；输出冻结文本 harness $A_F$（197 词）与密封配置 $\theta$。
- **Practice**：第 $r$ 轮读记录 $\mathcal{L}_{r-1}$ 写挑战者 $\tilde a_r$（能执行才算 valid），与在位者同批等预算比较，更优才替换；记录含源代码、轨迹、分数、失败原因与选择决策。
- **Gate**：$R$ 轮后在新种子上评估 $a_R$ 得 $G\in\{0,1\}$；$G$=0 只作审计记录，不进评测。
- **One-shot Distill + Seal**：只取一次 $(a_R,\mathcal{L}_R)$ 合成 $s_{dist}$，再 $A_F=\mathrm{Seal}(s_{dist},\theta)$；**不搜索备选文本**，否则等于重新引入测试反馈通道。

### 3.3 真正的创新点（去伪存真）

**真·方法创新**

1. **流程本身**，尤其"一次性蒸馏 + 密封"：在**制品层面**切断开发与评测，并配可审计 manifest（模型版本、推理设置、prompt、种子、hash）。
2. **可复现性标准重定义**：不求复现同一段文字或程序，而求"独立 run 复现同一性能层级"，用 Harness B（205 词，与 A 无一句相同）验证。
3. **失败 run 公开**：Candidate C 未过门控（validation 24.78）仍作诊断记录；作者承认只有 2 个被接受 run，门控–结果关联仅 diagnostic。

**工程组合**：LLM 写优化器代码、配对选择、开发 / 测试分离、prompt 蒸馏、Wilcoxon 检验均已有。

**最关键设计**：**实践记录而非最终程序**。code-only 蒸馏在 Gemini Pro 上从 21.4 退化到 32.7（$p$=.013）、在 GPT-5+sandbox 崩塌（65.3 vs 19.0，$p$=.010）。

**证据不足 / 仅声称有效**

- code-only 关键对比**缺轨迹**，作者自标其 "motivate, but do not by themselves establish a mechanism"。
- 消融**执行器依赖**：压缩 / 截断 / 替代蒸馏在 Flash 上甚至优于完整文本（28.0 / 25.3 / 21.8 vs 32.9），只在 GPT-5 退化。
- 多数单元只有 $B$=20、$D$=8、$N$=10，中心效应靠独立 $N$=30 撑；次要比较多未校正。
- BBOB 上仅"进入 BO 层级"（只有 Gallagher 过 Holm，$p_{Holm}$=.018）；生产基准只有点估计（30 个 bootstrap replicate 为同份数据有放回重采样）。

---

## 4. 具体技术细节

### 4.1 模型结构

- Base model 是**纯文本 LLM，非 MLLM**（无视觉编码器），输入为"已执行点有序历史 + 值 + 剩余预算"。
- 执行器：**Gemini Flash（主）**、Flash-Lite、Gemini Pro、Claude Sonnet（claude-sonnet-5，Vertex endpoint）、原始 runtime 的 GPT-5（含 sandbox）；参数量均未公开。
- **全部冻结、零微调**：同执行器下 Base 与 Harness 共享相同的 formatter、parser、bounds 与推理配置，唯一差别是那段蒸馏文本（A100 节点，开源模型 vLLM 本地部署）。

### 4.2 训练流程（若需要训练）

**不训练任何模型权重**——无 RL / SFT / LoRA，也未对 LLM 做蒸馏；文中 "distill" 指把实践经验蒸馏成**文本**。"训练"只发生在 harness 生成阶段，不改参数：

| 阶段 | 目标 | 学到 / 产出什么 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| Stage 0 | 提供可执行起点 $a_0$ | — | 人工 / 既有实现 | 优化器源代码 | 无（纯执行） |
| Stage 1：Practice（$R$=10） | 把口头假设变成可证伪行为 | 搜索策略（在程序与记录里，不在权重里） | 随机平移正定二次函数；每轮 16 个 fresh paired instances，预算 20 evals；champion 在 40 个 held-out seeds 复评 | 源代码 + 轨迹 + 分数 + 失败原因 + 选择决策 | 无梯度损失；配对比较：$\tilde a_r$ valid 且更优才替换 |
| Stage 2：Gate | 拦截未达标 run | — | 新开发种子上的验证批 $V$ | 程序级验证分数 | $G(a_R;V)\in\{0,1\}$；$G$=0 仅作审计记录（如 Candidate C，24.78） |
| Stage 3：One-shot Distill | 把经验压缩成可移植文本 | $s_{dist}$（A=197 词 / B=205 / C=212） | **仅一次**：$(a_R,\mathcal{L}_R)$ | 先做"行为→原则"映射表，再改写为第二人称策略文本 | 无 loss；**不做备选文本搜索** |
| Stage 4：Seal | 冻结部署配置 | — | $s_{dist}$ 与 $\theta$ | manifest：模型版本、推理设置、prompt、种子、结果 hash | $A_F=\mathrm{Seal}(s_{dist},\theta)$ |

> 纪律：practice 程序只允许"对观测历史做初等算术"（禁拟合、核方法与优化器库），故 champion 的决策都能一句话说清——这是蒸馏可行的前提。

### 4.3 推理流程

- **输入**：完整**有序**已执行点历史 $D_t=((x_i,f(x_i)))_{i=1}^{t}$ + 剩余预算 $B-t$；每次调用都是 **stateless**（历史完整重放）。**输出**：下一批候选点（JSON）。
- **多步推理**：$B$=20 次顺序评估的循环，每轮一次调用，终止即预算耗尽；不是 ReAct 式工具循环，而是"读历史 → 选点 → 执行 → 追加"的固定长度序列决策。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| Practice family（随机平移正定二次函数） | 非 GUI | 低预算黑箱优化（开发） | 每轮 16 个 fresh paired instances；主研究 $N$=10，中心对比 $N$=30 | 有序已执行点历史 + 值 + 剩余预算（纯文本） | 下一批候选点（JSON，$D$=8） | Final simple regret，均值 ± 95% CI，paired Wilcoxon |
| Held-out BBOB（Bent Cigar / Gallagher-101 / Rastrigin f15） | 非 GUI | 同上（测试，练习时不可见） | $N$=10 paired | 同上 | 同上 | 同上（Bent Cigar 量级 10⁶） |
| 内部 YouTube 奖励调优（保密） | 非 GUI | 生产黑箱优化（7 维） | 1 份密封历史数据集 + 30 个 paired bootstrap replicates（**非独立试验**） | 7 维 reward 向量 + 密封候选列表 | 候选点（经投影） | Best-so-far regret 点估计（5/10/15/20 evals） |

> 通用设定：域 $[-5,5]^8$，$D$=8，$B$=20 次顺序评估。

### 5.2 实验结果分析

**主结果（练习族二次函数，regret@20，越低越好）**：Retained program 17.1±4.7；GP-BO strengthened 28.7±18.0；**Harness A（ours）32.9±16.4**；GP-BO plain 39.5±15.9；Base LLM（Gemini Flash）64.2±30.2；CMA-ES 68.0±32.9；Random search 79.4±22.0。Base 与随机搜索无差异（$p$=.375）。Harness A 把 Flash 从 64.2 降到 32.9（$N$=10 时仅方向性），独立 $N$=30 研究复现同样降幅：**69.5→36.0，降 48%，$p<.001$**，配对平均改善 33.5（95% 区间 [19.0, 49.7]），**27/30 个实例改善**。Harness 均值落在两种 GP-BO 之间且与之无差异（$p$=.70 / .32），源程序（17.1）显著更优（$p$=.049）——**文本把 LLM 推进 BO 区间，但没超过 BO**。

**跨执行器与跨 landscape（文本一字不改，$N$=30）**：Flash 69.5→36.0†（Rastrigin 77.4→65.3）；Flash-Lite 116.2→48.5†（112.2→93.1†）；Claude Sonnet 83.5→47.3†（106.6→54.0†）（† = $p\le.005$）；Gemini Pro 77.5→21.4（$p$=.010）。六单元均值全降。**算力不是解释**：单用 thinking（54.3）或 sandbox+thinking（54.2）都远差于 harness 的 32.9，而两者可**叠加**达 **16.1±4.3**（$p$=.014）。

**Held-out BBOB（$N$=10，依次 Bent Cigar / Gallagher / Rastrigin）**：Base 28.3M / 69.0 / 183.7；Harness A 15.7M / 52.4 / 149.7；Retained program 13.7M / 53.0 / 112.1；GP-BO strong 46.4M / 69.4 / 156.7；GP-BO plain 36.6M / 62.7 / 146.3。三个 landscape 都降 Base 的 regret，仅 Gallagher 过 Holm（$p_{Holm}$=.018）。

**独立复现与生产**：独立复现得到的 champion 是**有限差分坐标搜索**，蒸馏出 **Harness B（205 词，与 A 无一句相同）**；四个新 landscape 上（$N$=30）两个被接受的 harness 都优于各自 Base 与 plain GP-BO（Holm 后 $p\le.007$）。生产侧 YouTube 7 维调优上 Harness B 各检查点都最低（20 evals 时 **0.0495** vs Vizier 0.0894），但只报点估计。

**Ablation 说明了什么**（regret，三列依次 GPT-5+sandbox / Gemini Pro / Gemini Flash）：Full harness A 20.1±10.4 / 21.4±4.5 / 32.9±16.4；Code only 29.3±8.7 / 32.7±7.0 / 36.0±16.9；One-sentence 31.9±10.2 / 24.3±12.5 / 28.0±7.5；First half 48.4±24.4 / 21.2±6.4 / 25.3±10.3；Alt distill 85.7±47.6 / 22.4±2.9 / 21.8±2.8。结论：(1) **实践记录确实贡献额外信息**——code-only 在 Pro 显著退化、在 GPT-5 崩塌；(2) **"哪部分文本必不可少"无统一答案**——压缩 / 截断 / 替代在 Gemini 两执行器上都在噪声内甚至更好，只在 GPT-5 退化。行为签名（Table 7）：步长 2.17→1.90、变异系数 1.04→0.68、落在 incumbent 半径 2 内的提议 69%→79%、改善事件 9.3→10.9——序列**更局部、更稳定、更常产生改善**。

---

## 6. 亮点与贡献（Why it matters）

1. **"密封"值得所有做 Agent 迭代的人抄走**：多数 prompt / harness 优化都在测试集上反复迭代，报出的数字已含测试反馈；评测前冻结 artifact 并记录 manifest 与 hash，是让"提升 X%"可解释的最低要求。
2. **"蒸馏的是策略而非实现"有实证**：同一冻结文本跨四个执行器全部有效，独立复现得到完全不同的程序却复现同一性能层级。
3. **可复现性从"同一 artifact"放宽到"同一性能层级"**，并用 A/B 两段无重叠句子的文本证明可行；同时诚实报告失败 run 与门控局限。
4. **行为签名**提供"文本→行为→性能"的中间层证据。

## 7. 局限与可改进点（个人点评）

- **"自博弈"用得过重**：正文没有 multi-agent 对抗，Stage 1 实质是**单 Agent 迭代式程序改进 + 配对选择**。
- **实验规模是最大短板**：$B$=20、$D$=8、多数 $N$=10，恰是"坐标探测 + 局部移动"这类简单策略的甜区；换成 $D$=50 或 $B$=200 是否有效未知。
- **没有超越经典优化器**：练习族上 harness 不如保留程序，BBOB 上只赢 plain GP-BO 一项；标题易让人高估其定位。
- **门控阈值缺前瞻验证**：Gate 是唯一决定哪个 run 进评测的环节，却只有 2 接受 + 1 拒绝；one-shot 蒸馏也是"可审计性优先于适应性"，一旦允许多次蒸馏就退回 GEPA 那条线的泄漏风险。

## 8. 对我们的启示 / 可借鉴点

- **把"实践–门控–蒸馏–冻结"当作 harness 级优化的标准流程**：正式评测前冻结 prompt / 工具接口 / parser / 推理设置并记录 manifest，评测中只允许输入变化。
- **蒸馏原料应是"完整实践记录"而非"最终产物"**：失败原因、选择决策、轨迹承载了比最终代码更多可迁移信息，轨迹挖掘不应只留成功样例。
- **行为签名是可用的中间层诊断指标**：成功率样本稀疏时，用动作粒度、与当前最优状态的距离、探索-利用比、改善事件频率判断策略是否真的变了。
- **可复现性可放宽为"性能层级"**；harness 必须**按 backbone 校准**，不能跨模型直接搬。

### 对 GUI Agent 的可借鉴点

1. **GUI harness 可自动构建，且不必依赖测试集**：让 Agent 在沙箱里反复写"GUI 操作策略程序"，用**自建开发任务**做配对评测与筛选，再把冠军策略与实践记录蒸馏成短系统提示，冻结后在真实 GUI 评测集上零样本使用——绕开"在 WebArena / OSWorld 上反复调 prompt"的泄漏。
2. **"密封 + manifest"对 GUI 评测尤其关键**：页面模板与评测脚本极易被调优隐性吸收；定稿后应冻结 prompt、动作空间、坐标归一化与截图分辨率并记录 hash。
3. **"如何花预算"在 GUI 上即"如何花步数 / 截图调用"**：六条原则可映射为 GUI 纪律——先截图确认页面结构 → 每次只验证一个假设 → 复用已知入口 → 步数少时直接执行 → 忽略小于渲染噪声的差异。GUI harness 应是**元策略**而非规则表。
4. **用行为签名与门控做筛选**：把步长 / 距离统计迁移为点击坐标距离分布、落在目标元素邻域内的动作比例、截图间状态变化率与无效动作比例；候选 harness 须先在少量自建 GUI 任务上过门。

## 9. 延伸阅读

- OPRO（ICLR 2024）—— 文本轨迹提候选解，本文批评其在评测中继续修订。
- FunSearch（Nature 2024）、AlphaEvolve（2025）、LLaMEA（IEEE TEVC 2024）—— "LLM 写优化器"这条线。
- GEPA（ICLR 2026）、Meta-Harness（arXiv:2603.28052）—— 反射式 harness 演化，与本文"一次性蒸馏 + 密封"对立。
- LLAMBO（ICLR 2024）、BOPRO（ICLR 2025）、COCO（OMS 2021）—— 把 LLM 嵌入 BO 与 held-out landscape 来源。

---

*解读生成时间：2026-09-11 09:00 ｜ 解读人：WorkBuddy（AI）*
