# RISE: Recursive Improvement via Self-Extrapolating Policy Distillation

> 一句话 TL;DR：把模型 RLVR 训练轨迹中"相邻 checkpoint 的进步位移"沿原方向外推成"未来的自己"当教师，做逐 token 蒸馏，让 on-policy 蒸馏从一次性压缩升级为递归自改进，全程无需外部教师或特权信息。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | RISE: Recursive Improvement via Self-Extrapolating Policy Distillation |
| **作者 / 机构** | Yang Li, Semih Yavuz, Shafiq Joty（Salesforce AI Research，美国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-04；arXiv preprint（arXiv:2609.05295v1，cs.AI） |
| **arXiv 链接** | https://arxiv.org/abs/2609.05295 |
| **代码仓库** | ❌ 未开源（文中未声明代码/权重，亦未给项目主页） |
| **数据集地址** | 未发布新数据集；训练/评测用公开基准：DAPOMath、OpenR1-Math-46K、Skywork-OR1-Code、ALFWorld、WebShop、GPQA-Diamond、IFEval、MMLU-Pro 等 |
| **类型标签（论文类别）** | `RL` `Online` `Distillation` `General` |
| **训练方法标签** | `RL (GRPO)` + `Distillation`（on-policy 自外推蒸馏；无外部教师、无特权条件） |
| **关键词** | 递归自改进；on-policy 蒸馏；RLVR；checkpoint 外推；逐 token 信用分配；自蒸馏 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-04_RISE_Recursive_Improvement_via_Self-Extrapolating_Policy_Distillation.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：通用 Agent/LLM post-training 里的"递归自改进"——让模型仅凭自身经验、无需外部监督地持续变强。更聚焦地说，是 on-policy distillation（OPD）这一给每 token 稠密监督的技术路线里，**"可靠教师从哪来"** 这一根因问题。

- **为什么重要**：RLHF/RLVR、self-play、rejection sampling 的共同缺陷是学习信号是**序列级**的——一个 outcome reward、一个对错标签或一个标量偏好，无法告诉模型"到底是哪几个 token 导致成败"（credit assignment 瓶颈）。OPD 能补上这层，但 OPD 的上限完全卡在教师质量上：教师不行，稠密信号就是有害噪声。

- **现有方法有什么不足**（论文逐条批评）：① 外部强模型当教师 → **prefix 分布失配**：学生探索出的推理路径教师没见过，教师在这种陌生前缀下的 next-token 分布不可靠，恰恰在学生最需要指导的困难步骤失效；② 同一基座 + 特权条件（喂正确答案/环境反馈）做 on-policy self-distillation（OPSD）→ 受限于模型 ICL 容量，特权信息"喂了也吸收不了"，且师生看不同输入，引入另一类失配；③ token 门控、KL 混合、DAgger 采样、轨迹改写等补救手段 → 都是在"容忍一个坏教师"，治标不治本。

- **Research Gap**：作者把矛头指向本质——**"教师应该是谁？"** 答案应是策略正在收敛去的那个"未来自我 π\*"。π\* 虽未知，但 RLVR 轨迹揭示了收敛方向，把相邻 checkpoint 的位移外推即可合成教师。这是论文自认补上的缺口：**不引入外部模型、不依赖特权条件，用模型自身训练轨迹造出高质量教师**，并把蒸馏升格为递归改进而非一次性压缩。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

用 RLVR 更新"定锚"出可靠进步方向，再把这个方向的位移外推放大，合成一个"更未来的自己"作为逐 token 教师，蒸馏回当前策略，两阶段交替构成递归循环。

### 3.2 方法总览（Pipeline）

- **输入/输出**：输入是 prompt x 与策略 π_θn 采样的 rollout；输出是更新后的策略 π_θn+1。
- **核心模块与连接**：每轮迭代两个阶段（Algorithm 1/2）：
  1. **RLVR 阶段**：当前策略采样 rollout，算 outcome reward，用 GRPO 做策略梯度得 θ′_{n+1}——这一步给外推方向"定锚"，保证方向指向真实改进；
  2. **OPD 阶段**：构造外推教师 π_future（见 3.3 的两种 φ 实例化），逐 token 蒸馏回学生 θ′_{n+1}，得 θ_{n+1}。
- **关键省成本设计**：两个阶段**复用同一批 rollout**，OPD 零额外采样；logit 空间版本只需缓存 top-K logits，OPD 阶段无教师前向。
- **配套机制**：β 衰减调度 β_n = 1 + (β0−1)(1−n/N)（默认 β0=1.2），因为越接近最优、安全外推区间越窄；anchor 可选 EMA 平滑位移方向。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：① 把"教师"重新定义为**自身轨迹外推的未来自我**，用 φ 统一抽象（logit 空间 = 几何混合 π_future ∝ π_anchor^(1−β)·π′^β；权重空间 = 带外推系数的 task arithmetic），两者互为泰勒一阶近似；② 两阶段互补循环——RLVR 定方向、OPD 做 trust-region projection，把稀疏 outcome 信号转成稠密 token 目标，这一"外推 + 蒸馏而非直接采纳"的组合是核心新意。
- **工程组合**：top-K+tail（K=100，代码域 K=20）截断、JSD 替代 KL、EMA anchor、β 衰减，均为已知组件，被组合进新框架。
- **对性能提升最关键的设计**：消融显示**两个阶段缺一不可**——去 RLVR 则 60 步内崩溃，去 OPD（直接采纳 θ_future）则增益归零（见 5.2）；其次 β 衰减调度是防坍塌的关键。
- **证据不足 / 仅声称有效**：外推的合法性依赖"轨迹低秩近线性"假设，论文只给了三方向捕约 87% 方差的间接佐证；"何时外推开始失效"缺乏原则性判据，只靠 β 衰减 + EMA 打补丁；代码域终值增益微弱，方法对强策略的边际价值证据偏弱。

---

## 4. 具体技术细节

### 4.1 模型结构

纯 **LLM（非 MLLM，无视觉编码器）**，全部实验都在文本域。规模 1.7B–8B：Qwen3-8B / Qwen3-1.7B / Qwen3-1.7B-Base（DAPOMath）、OLMo3-7B-Instruct-SFT（OpenR1）、Qwen3-4B-Base（混合 math+STEM）、Qwen3-8B-Base（代码）、Qwen2.5-3B-Instruct（agentic）。全量微调策略参数（外推直接在参数/logit 上做，未提 LoRA/冻结）。

### 4.2 训练流程（self-extrapolating distillation 是重点）

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| RLVR 阶段 | 产出定锚的进步方向 | 结果驱动的策略优化 | 当前策略自采样 rollout | prompt + 响应 + outcome reward ∈[0,1] | GRPO clipped surrogate（式 2），token 内共享 advantage |
| OPD 阶段 | 把外推教师稠密地蒸馏回学生 | 逐 token 的细粒度信用分配 | 复用 RLVR 同一批 rollout | 每个位置的 top-K+tail 分布 | L_RISE = Σ_t D_KL(π_θ ∥ sg[π_future])，实际用有界 JSD（式 9） |

- **外推公式**：φ(π_future) = φ(θ_anchor) + β·(φ(θ′_{n+1}) − φ(θ_anchor))，β>1。β=1 退化为当前 checkpoint（无蒸馏信号），β>1 沿最近一次 RLVR 位移"超前投影"。
- **两种实例化**：权重空间 φ=θ（θ_future = θ_anchor + β·∆θ，需物化参数并做教师前向）；logit 空间 φ=log π（几何混合，仅缓存 top-K logits 做外推，零额外前向），后者是前者的泰勒一阶近似。
- **loss 细节**：logit 空间在 S=TopK(π′_{n+1}) 上把两个分布投影到同一 (K+1)-simplex（含 tail 桶）再外推；π_future 全程 stop-gradient。用 JSD 因有界于 log 2，避免 reverse KL 数值不稳定，且与 KL 同唯一极小点，trust-region 保证保守成立。
- **anchor 动力学**：默认 η=1（anchor=上一 checkpoint）；Qwen 系用 EMA η=0.1 平滑噪声方向，OLMo 用 η=1（其 GRPO 步已大且稳）。

### 4.3 推理流程

方法只改训练，**推理不做任何改变**：标准的自回归单次生成（single-pass），无 ReAct/规划/反思循环。评测用采样多次取平均 avg@N 与 best-of-N 的 pass@N（如 AIME'24 avg@16/pass@16），用于区分"磨尖均值"与"扩大可解集"。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| MATH-500 / AIME'24/25 / AMC'23 / Minerva / OlympiadBench | General（非 GUI） | 数学推理 | 各数百~数千题 | 纯文本题面 | 逐步推理 + 答案 | Accuracy / avg@N / pass@N |
| GPQA-Diamond / IFEval / MMLU-Pro | General | OOD 泛化检查 | 数百~数千 | 纯文本 | 答案 | Accuracy |
| TheoremQA / SuperGPQA / MMLU | General | STEM 问答 | 数百~数千 | 纯文本 | 答案 | Accuracy |
| HumanEval+ / MBPP+ | General | 代码生成 | 各 ~200 题 | 纯文本 | 代码 | pass@k / avg@k |
| ALFWorld | General（text-embodied） | 多轮 agentic | ~140 局 | 文本状态+指令 | 动作序列 | Success Rate |
| WebShop | General（text-embodied） | 多轮 agentic | 数千 | 文本商品+指令 | 点击/购买序列 | Score / Accuracy |

### 5.2 实验结果分析

- **主结果**（基线：GRPO、GRPO+SDPO、SDAR、RLSD）：
  - 数学：Qwen3-8B Math Avg 60.0→**62.7**（weight，+2.7），OOD 70.6→72.0 不掉反升；Qwen3-1.7B 45.4→**50.2**（logit，+4.8）；OLMo3-7B 47.6→**56.4**（logit，+8.8），AIME'24 30.2→46.9（+16.7）。三种子复现 8B 62.4±0.2 vs 60.1±0.2、1.7B 49.6±0.4 vs 45.1±0.4。特权基线整体偏弱，GRPO+SDPO 甚至低于 GRPO（55.9 vs 60.0）。
  - 多域（Qwen3-4B）：Math Avg 40.2→44.8、STEM Avg 45.5→47.5，AIME'24 +5.0、TheoremQA +3.7，跨域位移互不稀释。
  - 代码：终值相当，但收敛快一倍（RISE logit 第 50 步 ≈ GRPO 第 90 步的 HumanEval+）。
  - Agentic：ALFWorld 75.0→**84.4**（+9.4）；WebShop Score 79.8→86.3、Acc 63.3→**74.2**（+10.9）。
- **Ablation**：① 去 RLVR（纯自蒸馏方向外推）60 步内崩溃，MATH-500 跌至 2.4%、响应长度 2K 爆到 8K 上限、reward 归零——外推必须被 outcome reward 定锚；② 去 OPD（直接采纳 θ_future）增益几乎归零（8B 60.0→60.3、1.7B 45.4→45.6），证明增益来自蒸馏的 trust-region 投影而非"多走一步"；③ β 安全区随训练收窄：第 50 步 β=2.0 尚 +7.3 点，第 100 步同 β 灾难性 −15 点，第 200 步 β=1.5 已 −7.7 点；④ β0∈[1.2,1.5] 稳健（62.5/62.3/61.9），β0=2.0 发散，任意衰减优于固定 β（cosine 63.0 > fixed 61.8）；⑤ anchor：Qwen 越低 η 越好（logit 46.2→48.4→50.2），OLMo 相反（η=1 的 56.4 > η=0.1 的 50.1）；⑥ 重采样 vs rollout 复用结果相当（8B 62.5），复用默认更省；⑦ 等梯度预算的 GRPO-2× 仅 +0.5~+1.9，RISE 仍领先 +2.2~+2.9，增益来自监督质量非算力；⑧ 墙钟开销仅 1.3–1.6×，零额外采样。

---

## 6. 亮点与贡献（Why it matters）

1. **重新定位 OPD 瓶颈**：把问题从"如何整合教师信号"上溯到"教师该是谁"，并用自身轨迹外推给出无需外部模型/特权条件的自举教师，直接消掉外部教师失配与特权条件失配两大病灶。
2. **把蒸馏从一次性压缩升级为递归改进**：教师每轮随学生刷新（非平稳），消除 ExOPD 式静态教师的天花板。
3. **统一抽象 + 可消融的双实例化**：logit/权重空间互为泰勒近似，计算—统计权衡清晰。
4. **证据链完整**：四族任务、1.7B–8B 跨规模跨架构，两阶段缺一不可、β 调度依据充分；且证明覆盖增益真实（1.7B AIME'24 pass@16 +9.6 > avg@16 +6.3，说明在扩大可解集而非只磨尖旧解）。

---

## 7. 局限与可改进点（个人点评）

1. **外推合法性建立在"低秩近线性"假设上**：文本推理里站得住（三方向捕 87% 方差），但策略面一旦崎岖——GUI 的长程稀疏奖励、多模态观测、结构化动作输出——"沿最近位移外推"可靠性无保障，且论文自认缺乏"外推何时失效"的原则判据，只靠 β 衰减 + EMA 打补丁。
2. **reward 可 hack 时风险被放大**：外推方向不辨真伪，若奖励函数有漏洞，会把伪方向一起外推，这在 GUI 奖励构造质量参差的环境里比数学题更现实。
3. **超参对训练噪声敏感**：EMA anchor 取向跨模型相反（Qwen 要 η=0.1、OLMo 要 η=1 且反设置掉 6 分以上），缺自适应准则，落地需逐模型调参。
4. **验证面窄**：仅在文本输出域，未见视觉/多模态；logit 空间外推隐含"同词表同架构"前提；代码域终值增益小，对强策略提效有限。
5. **未开源**：复现成本高，外推两实例化的实现细节（如 tail 桶重归一化、top-K 缓存）只能从伪代码推断。

---

## 8. 对我们的启示 / 可借鉴点

**对 GUI Agent 的可借鉴点**（本论文属 Agent 方法论域，非 GUI 落地）：GUI Agent 的 post-training 正是"结果奖励稀疏、credit assignment 极难"的重灾区——整条点击/输入/滚动轨迹只拿到一个最终成败标量，GRPO/DAPO 只能给整条响应同一 advantage。RISE 给出**完全自举的稠密化方案**：用 GUI 策略自身相邻 checkpoint 的位移外推教师，对每步动作做逐位置蒸馏，全程无需更大的外部教师 VLM——既绕开"教师没见过学生探索出的 GUI 状态"的分布失配，也省掉昂贵的外部标注/推理；rollout 复用让环境交互（GUI 里最贵的资源）成本减半。β 衰减 + OPD 作 trust-region 的思想可移植为"训练早期大步探索、后期保守收缩"，防止把某次偶然 reward 噪声外推成长期策略。落地需两处改造：① GUI 动作是结构化多模态输出（含坐标/元素 id），top-K+tail 的 token 级蒸馏应替换为动作分布上的蒸馏；② 动手前先用少量轨迹做"线性度诊断"（特征值衰减、rank 占比），确认低秩近线性假设在该数据 regime 下成立。

---

## 9. 延伸阅读

- **ExOPD**（Yang et al., 2026d）：reward 外推 OPD，其最优策略与 RISE logit 公式同构但教师静态，是 RISE 最直接对照系。
- **GRPO+SDPO**（Hübottel et al., 2026）、**SDAR**（Lu et al., 2026）、**RLSD**（Yang et al., 2026a）：privileged-conditioning 自蒸馏三件套，本文基线。
- **Rank-1 RLVR 轨迹**（Wei et al., 2026）、**RLVR 训练线性度**（Wang et al., 2026a）：支撑外推合理性的两条经验证据。
- **Task Arithmetic**（Ilharco et al., 2022）/ **Model Soups**（Wortsman et al., 2022）：权重空间插值家族，RISE 是其外推化。
- **GIGPO**（Feng et al., 2025）：ALFWorld/WebShop 的 agentic RLVR 训练流程，RISE 的 agent 实验底座。
- **OPD 综述**（Song & Zheng, 2026）：on-policy distillation 全貌地图。

---
*解读生成时间：2026-09-10 ｜ 解读人：WorkBuddy（AI）*
