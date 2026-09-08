# RISE: Recursive Improvement via Self-Extrapolating Policy Distillation（RISE：经由自外推策略蒸馏的递归式自我提升）

> TL;DR 不依赖任何外部教师模型或特权信息，把模型自身 RLVR 训练中相邻 checkpoint 的"进步位移"外推成"未来的自己"当教师，做逐 token 蒸馏，让蒸馏从一次性压缩变成递归自我改进。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | RISE: Recursive Improvement via Self-Extrapolating Policy Distillation |
| **arXiv ID / DOI** | arXiv:2609.05295v1（无 DOI） |
| **arXiv 链接** | https://arxiv.org/abs/2609.05295 |
| **发表出处（Venue）** | arXiv preprint（Salesforce AI Research 预印本） |
| **发布时间** | 2026-09-04 |
| **作者** | Yang Li, Semih Yavuz, Shafiq Joty |
| **所属机构** | Salesforce AI Research（美国） |
| **开源情况 / 代码** | ❌ 未在文中声明开源 |
| **类型标签** | RL, Online, Distillation, General |
| **训练方法标签** | RLVR (GRPO) + On-policy Distillation（RLVR 与自外推 OPD 交替的两阶段训练；无外部教师、无特权条件） |
| **关键词** | 递归自我提升；on-policy 蒸馏；RLVR；checkpoint 外推；逐 token 信用分配；自蒸馏 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | 2026-09-04_RISE_Recursive_Improvement_via_Self-Extrapolating_Policy_Distillation.pdf |

---

## 1. 研究背景与要解决的问题

让大模型从自己的经验中递归变强，是 post-training 的圣杯。RLHF/RLVR、self-play、rejection sampling fine-tuning 是三条主流路径，但它们有个共同毛病：**学习信号是序列级的**——一个结果奖励、一个对/错标签或一个标量偏好，无法告诉模型"到底是哪几个 token 导致了成败"。On-policy distillation（OPD）能补上这层：教师在每个位置给出完整的 next-token 分布，相当于逐 token 告诉学生"下一步该怎么想"。

于是问题变成：**可靠的教师从哪来？** 作者把现有答案逐一否掉：外部强模型当教师，会因"prefix 分布不匹配"失效——学生探索出的推理路径教师没见过，教师在这种陌生上下文下的分布并不可靠；用同一基座模型 + 特权条件（把正确答案或环境反馈喂给教师）做 on-policy self-distillation（OPSD），则受限于模型的 ICL 容量，特权信息常常"喂了也白喂"，反而引入另一类分布失配。至于门控、KL 混合、DAgger 采样、轨迹改写等补救手段，都是在"容忍一个坏教师"。作者把矛头直指本质：**教师应该是谁？** 答案是一个自然的候选者——模型自己训练正在收敛去的那个"未来自我" π\*。

## 2. 核心方法 / 思路

RISE 的构造极其巧妙：未来自我 π\* 不可知，但 RLVR 训练轨迹揭示了"向它收敛的方向"。把策略映射到可做线性运算的向量空间 φ（参数或 logit），相邻 checkpoint 的位移 φ(θ′)−φ(θ) 就刻画出最近的进步方向，外推即得合成教师：

φ(π_future) = φ(θ_anchor) + β·(φ(θ′_{n+1}) − φ(θ_anchor))，β>1。

β=1 时教师等于当前 checkpoint（无蒸馏信号）；β>1 时放大了最近一次 RLVR 更新，"超前投影"出更强的自己。这并非玄学：近期分析表明 post-training 更新由低秩子空间主导、近似线性演化，作者自测三个方向即捕获约 87% 方差。RISE 据此给出两种实例化：**权重空间外推**（φ=参数，即 task arithmetic 式的外推系数版，需物化 θ_future 并做前向）与 **logit 空间外推**（φ=logπ，等价于几何混合 π_future∝π_anchor^(1−β)·π′^β，仅对缓存 top-K logits 做外推，零额外前向）——后者恰是前者的泰勒一阶近似。

整体是 RLVR 与 OPD 互补的双阶段循环（Algorithms 1/2）：① RLVR 阶段用 GRPO 做策略梯度得到 θ′_{n+1}，**outcome reward 为外推方向"定锚"**；② OPD 阶段把外推出的 π_future 以 top-K+tail（K=100，代码域 K=20）逐 token 蒸馏回学生 θ′_{n+1}（损失实际取有界且稳定的 JSD），得到 θ_{n+1}。每轮复用同一批 rollout，**无额外采样成本**。关键设计还有：β 采用衰减调度 β_n=1+(β0−1)(1−n/N)（默认 β0=1.2），因为越接近最优、安全外推区间越窄，固定激进 β 迟早会毁掉教师；anchor 可用 EMA（Qwen 系取 η=0.1）平滑位移方向。为什么不一口气把 θ_future 采纳为下一策略？外推点超出信任域，直接采纳会放大 RLVR 噪声甚至退化——OPD 在这里扮演 trust-region projection。

## 3. 关键实验结果

覆盖 4 类任务、1.7B–8B 多种模型：数学推理（Qwen3-8B/1.7B/1.7B-Base on DAPOMath，OLMo3-7B on OpenR1-Math）、数学+STEM 多域混合（Qwen3-4B）、代码生成（Qwen3-8B on Skywork-OR1-Code）、多轮 agentic（Qwen2.5-3B on ALFWorld/WebShop，走 GIGPO 流程）。基线为纯 GRPO 与三种特权自蒸馏变体 GRPO+SDPO/SDAR/RLSD。

- 数学：Qwen3-8B Math Avg 60.0→**62.7**（weight，+2.7），OOD 70.6→72.0 不掉反升；Qwen3-1.7B 45.4→**50.2**（logit，+4.8）；OLMo3-7B 47.6→**56.4**（logit，+8.8），AIME'24 30.2→46.9（+16.7）。三种子复现：8B 62.4±0.2 vs GRPO 60.1±0.2；1.7B 49.6±0.4 vs 45.1±0.4。
- 多域混合：Math Avg 40.2→44.8、STEM Avg 45.5→47.5，AIME'24 +5.0、TheoremQA +3.7——跨域位移不外推不互相稀释。
- 代码：终值相当但收敛快一倍（HumanEval+ 上 RISE 第 50 步 ≈ GRPO 第 90 步）。
- Agentic：ALFWorld 75.0→**84.4**（weight，+9.4）；WebShop Score 79.8→86.3、Acc 63.3→**74.2**（+10.9）。
- 机制消融证伪两手都要硬：去掉 RLVR 锚定，纯自蒸馏方向外推在 60 步内崩溃（MATH-500 跌至 2.4%、响应长度 2K 爆到 8K 上限）；去掉 OPD 直接采纳 θ_future，增益几乎归零（60.0→60.3 / 45.4→45.6）。β 安全区随训练收窄：第 50 步 β=2.0 尚可 +7.3 点，第 100 步同 β 灾难性 −15 点。与等梯度预算的 GRPO-2× 相比 RISE 仍领先 +2.2~+2.9，说明增益来自监督质量而非多算几步。墙钟开销仅 1.3–1.6×，无额外采样。

## 4. 亮点与贡献（Why it matters）

一，把 OPD 瓶颈定位为"教师质量"这一根因，并用自身训练轨迹造出无外部模型、无特权条件的教师。二，教师每轮随学生进步而刷新（非平稳），把 OPD 从"一次性压缩"升格为"递归改进"，消掉静态教师的天花板（直接对标 ExOPD）。三，φ 统一抽象下给出参数/logit 两种实例化，计算—统计权衡清晰可消融。四，实验横跨推理/STEM/代码/agent 四族，证据链完整：两相缺一不可、β 调度依据充分、覆盖增益真实（1.7B AIME'24 pass@16 +9.6 超过 avg@16 的 +6.3，说明在扩大可解集而非只会磨尖旧解）。五，理念普适：任何"稀疏结果奖励 + 逐 token 蒸馏"的训练管线都可套用这套脚手架，对做 GUI Agent post-training 的人尤其相关。

## 5. 局限与可改进点（个人点评）

外推的合法性建立在线性低秩假设上：RLVR 更新近似 rank-1、系数近线性。这在文本推理里站得住（三方向捕 87% 方差），但一旦策略面更崎岖（GUI 的长程稀疏奖励、多模态观测、结构化动作输出），"沿最近位移外推"的可靠性没有保障，作者也自认缺乏"何时外推开始失效"的原则性判据，只靠 β 衰减 + EMA 打补丁。其次，reward 可 hack 时外推会把伪方向一起放大——GUI 里奖励函数构造质量参差，这个风险比数学题更现实。再者，EMA anchor 参数取向跨模型不一致（Qwen 要 η=0.1、OLMo 要 η=1 且相反设置掉 6 分以上），本质是超参对训练噪声敏感，缺乏自适应准则。最后是验证面窄：只在文本输出域验证，未见视觉/多模态输入，logit 空间外推还隐含"同词表同架构"前提；代码域终值增益小，说明对强策略提效空间有限；且未开源，复现成本高。

## 6. 对 GUI Agent 的可借鉴点

GUI Agent post-training 正是"结果奖励稀疏、credit assignment 极难"的重灾区——整条点击/输入/滚动轨迹只拿到一个最终成败标量，GRPO/DAPO 只能给整条响应同一 advantage。RISE 给出**完全自举的稠密化方案**：用 GUI 策略自身相邻 checkpoint 的位移外推教师，对每一步动作做逐位置蒸馏，全程不需要更大的外部教师 VLM——既绕开"教师没见过学生探索出的 GUI 状态"的分布失配，也省掉昂贵的外部标注/推理。rollout 复用让环境交互（GUI 里最贵的资源）成本减半，两阶段共用一份采样。β 衰减 + OPD 作 trust-region 的思想也可移植：GUI 训练早期大步探索、后期保守收缩，防止把某次偶然的 reward 噪声外推成长期策略。落地时要注意两处改造：GUI 动作是结构化多模态输出（含坐标/元素 id），top-K+tail 的 token 级蒸馏需替换为动作分布上的蒸馏；以及在动手前先用少量轨迹做"线性度诊断"（特征值衰减、rank 占比），确认假设在该数据 regime 下成立。

## 7. 延伸阅读

- ExOPD（Yang et al., 2026d）：reward 外推的 OPD，其最优策略与 RISE logit 公式同构，但教师静态，是 RISE 最直接的对照系。
- RLSD（Yang et al., 2026a）、GRPO+SDPO（Hübottel et al., 2026）、SDAR（Lu et al., 2026）：privileged-conditioning 自蒸馏三件套，RISE 的基线。
- Rank-1 RLVR 轨迹（Wei et al., 2026）、"RLVR 训练的线性度"（Wang et al., 2026a）：支撑外推合理性的两条经验证据。
- Task Arithmetic（Ilharco et al., 2022）/ Model Soups（Wortsman et al., 2022）：权重空间的插值家族，RISE 是其外推化。
- GIGPO（Feng et al., 2025）：ALFWorld/WebShop 的 agentic RLVR 训练流程，RISE 的 agent 实验底座。
- OPD 综述（Song & Zheng, 2026）：on-policy distillation 全貌地图。

---
*解读生成时间：2026-09-08 09:35 ｜ 解读人：WorkBuddy（AI）*
