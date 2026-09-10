# Headroom-Drift Replay: A Primitive for Principled Replay Control in GRPO

> 一句话 TL;DR：把 GRPO 的回放拆成「Headroom 学习价值排序 + Drift 策略漂移门控」两个正交控制轴，不新增任何生成即让旧轨迹复用提质降本。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Headroom-Drift Replay: A Primitive for Principled Replay Control in GRPO |
| **作者 / 机构** | Hyun Bin Park（一作）、Du-Seong Chang；韩国西江大学（Sogang University）人工智能系，首尔 |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-03；COLM 2026（Conference on Language Modeling）；arXiv:2609.03941v1（cs.LG） |
| **arXiv 链接** | https://arxiv.org/abs/2609.03941 |
| **代码仓库** | ❌ 未开源（正文与附录均未提供代码或模型链接） |
| **数据集地址** | 未公开自建数据；使用公开 benchmark 集合（AIME24/AMC23/MATH500/Minerva/OlympiadBench/NQ/TriviaQA/PopQA/HotpotQA/2WikiMultiHopQA/Musique/Bamboogle/Geometry3K/MathVista/MathVision） |
| **类型标签（论文类别）** | `RL` `Online` `General`（通用推理 + Agentic Search，非 GUI 落地） |
| **训练方法标签** | `RL (GRPO)` `Online RL`（组级回放增广的在线 RL 后训练，GRPO 主更新式不变） |
| **关键词** | Replay control; GRPO; Off-policy reuse; Sample efficiency; Policy drift; Agentic RL |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-03_Headroom-Drift_Replay_A_Primitive_for_Principled_Replay_Control_in_GRPO.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：GRPO 风格推理后训练被「反复生成全新 rollout」的成本卡死，agentic 场景尤甚——一次 rollout 要模型与环境（检索/工具）多轮真实交互，墙钟时间才是主导开销。论文要回答：能不能靠「有原则地复用旧轨迹」本身，就省下这笔钱且不伤质量。
- **为什么重要**：回放若控制不好会学不稳，直接限制了大模型推理 RL 在大规模 agentic 场景的可扩展性——不解决，就只能在「贵」和「飘」之间二选一。
- **现有方法有什么不足**：一条存储轨迹组会因两种截然不同的原因失效——仍有学习信号却已落后于当前策略（stale），或贴近当前策略但已无可学。已有回放工作（RePO、EFRame、ExGRPO、BAPO 等）都把回放嵌进探索、过滤、混合策略优化等更大流水线里，**回放自身的贡献无法被隔离研究**。
- **Research Gap**：作者认为缺的是一个「可独立开关、可单独评估的回放侧控制原语」——把「还值不值得学」和「现在还合不合身」两个判断显式分离并各自可消融，从而回答「有原则的回放选择单独能走多远」。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

新策略的 on-policy 流一行不改，唯一改动是把按规则筛出的旧组并入混合 batch，让一切训练差异都追溯到回放侧控制。

### 3.2 方法总览（Pipeline）

- **输入**：当前策略 π_θt、prompt batch、回放缓冲区 B_t、回放预算 K_rep、漂移阈值 τ。
- **输出**：混合 actor batch → 标准 GRPO 更新。
- **模块与数据流**（Algorithm 1，每步五阶段）：A. 照常生成新鲜组并算组内优势，把「组内结果混合（非全对非全错）」的组标为回放入口候选——全同奖励组优势为零、无学习信号；B. 缓冲区旧组按 Headroom 降序排列；C. 依序做 teacher-forced 前向重估（固定序列、无需自回归生成与环境交互），漂移不超 τ 才接纳，直到 K_rep 填满即停；D. 接纳组与新鲜组合成混合 batch 跑标准 GRPO；E. 更新后才把入口候选追加进 FIFO 缓冲，杜绝同一步自回放。同一前向 pass 的概率既做门控、又顺手刷新 Headroom 缓存。
- 复用最小单位是**整组而非单条回答**，保留 GRPO 最核心的组内相对优势比较结构。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：① Headroom 的 token 级定义——对 A_i>0 的回答记 `1−π`、对 A_i<0 的记 `π`（正向 token 还能抬多少、负向 token 还能压多少），组内取平均；② Policy Drift 用 L2 平方聚合逐 token log-prob 漂移（`(log π_θ − log π_gen)²`），避免正负抵消；③ 二者形式化挂钩：通过门控的组，其当前 Headroom 相对存储排序最多偏差 √τ（Proposition 2）。
- **工程组合**：整组复用、FIFO 缓冲、teacher-forced 同 pass 刷新缓存、replay-prefix 批布局（回放组占前 mini-batch）、序列长度均衡分片（跨 GPU 最大最小差 73K→5K tokens）——本身不新，但组合有效。
- **对性能提升最关键的设计**：MATH-500 消融显示 Headroom 是主引擎——完整版 Mean@32=0.7215，仅 Headroom 0.7140，仅 Drift 0.6794；Drift 门控是稳定化加成。L2 门控「靠选择质量而非回放量」赢（L2 回放 KL 比 L1 小约 9×）。
- **证据不足 / 仅声称有效**：agentic 主设定下 vs naive replay 的 Avg Mean@32 只差 0.0029（作者自认噪声内）；7B 复核是「每方法单次运行」无方差；CISPO 迁移只有 training-score 初步证据、无 held-out 结果；L2 vs L1 取舍仅附录定性说明。

---

## 4. 具体技术细节

### 4.1 模型结构

- **纯文本 LLM，无视觉编码器**：数学/多模态用 **Qwen2.5-Math-1.5B**，agentic 用 **Qwen2.5-3B-Instruct / Qwen2.5-7B-Instruct**（8×H100；7B 用 8-bit optimizer）。全文未见 image/encoder/MLLM 描述——所谓「multimodal reasoning」实为文本化几何/数学题，未用 MLLM，这是值得留意的命名问题。

### 4.2 训练流程

无独立 SFT 阶段，直接以 instruct 模型为起点做**单阶段在线 GRPO 后训练**，每步内嵌回放选择：

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| 在线 GRPO + 回放控制（逐 step 循环，非分阶段） | 以最小干预复用旧轨迹、提质降本 | 数学/几何推理、agentic 多轮检索推理 | 单轮：AIME24/AMC23/MATH500/Minerva/OlympiadBench/Geo3K/MathVista/MathVision；多轮：NQ/TriviaQA/PopQA/HotpotQA/2WikiMultiHopQA/Musique/Bamboogle | 组级 rollout（同 prompt 采样多条回答成组，G=8）；固定可验证奖励 | 标准 GRPO group-clipped objective，只是 batch 混合了回放组；回放组唯一区别是分母 reference policy 用 π_gen(g)（生成时策略）而非 rollout 时 old policy，`r_i,j = π_θ/π_gen(g)` |

**Loss / 奖励设计是本篇重点**：混合 batch 目标 `L_mix = (1/|A_t|) Σ ℓ_GRPO(g; θ, π_gen(g))`，可精确分解为 `α·L_on + (1−α)·L_rep`——回放「增广而非替换」on-policy 流。缓冲区维护不可变的 per-group 参考状态（存储动作、生成时 log-prob、response-level advantage），把每条旧轨迹锚定到它被采样的那一刻。

### 4.3 推理流程

训练阶段 rollout 分两类：数学/多模态为**单轮推理**（一次性生成答案），Agentic Search 为**多轮推理**（Search-R1 风格，模型与检索环境多轮交互取证据）。评估用 Best@n/Mean@n（每输入采样 n 次，主实验 n=32、7B 因算力 n=4），reward 均为 0–1 可验证标量。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| AIME24 / AMC23 / MATH500 / Minerva / OlympiadBench | General（数学） | 单轮数学推理 | 论文未逐项公开（报 fresh responses 总量） | 文本题目 | 答案 | Best@32 / Mean@32 |
| NQ / TriviaQA / PopQA / HotpotQA / 2WikiMultiHopQA / Musique / Bamboogle | Web（检索） | 多轮 Agentic Search | 同上 | query + 检索环境 | 检索答案 | Best@32 / Mean@32 |
| Geometry3K / MathVista / MathVision | General（几何/多模态） | 单轮视觉-数学 | 同上 | 文本化几何题 | 答案 | Best@32 / Mean@32 |

### 5.2 实验结果分析

- **数学（基线最全，Table 3）**：Headroom-Drift 的 Avg Mean@32 = **0.3533**（Top-1），超过 BAPO 0.3469、DAPO 0.3409、ExGRPO 0.3139、on-policy matched 0.3356 / larger 0.3344、naive replay 0.3117/0.3053；且 fresh responses 1,843,200 与 matched 持平、明显少于 larger 的 2,764,800（省 1/3 新鲜预算），排除「纯靠更多新数据」。注意 Best@32=0.5565 反而输给 ExGRPO/BAPO/DAPO——它在**平均质量**而非**上尾**上赢。
- **Agentic Search（Table 1+5）**：Avg Mean@32=0.3577 仅比 naive replay matched 0.3548 高 0.0029（噪声内），但 Avg Best@32=0.4879 vs 0.4623 差距明显；相对 on-policy larger 是严格帕累托改进——0.3577 @ 166.3s/步 vs 0.3212 @ 197.2s/步。7B 复核：Avg Mean@4 0.3737→0.3955、Weighted Mean@4 0.4158→0.4298，7 项赢 6 项。
- **多模态（Table 4）**：Avg Mean@32=0.4137 > matched 0.3986、larger 0.4005、DAPO 0.4056。
- **Ablation**：MATH-500 组件消融 Mean@32 完整 0.7215 > 仅 Headroom 0.7140 > 仅 Drift 0.6794；τ 三态（acceptance 6.4%/40.2%/94.4%）显示 MODERATE（τ=0.01）选择性回放且 replay KL 稳定在 ~0.002；sign-only Headroom 平均训练分 0.3288 显著优于 advantage-weighted 0.3022；回放相对 on-policy larger 更晚进入低熵区（237 vs 207 步）且停留更久（261 vs 180）。

---

## 6. 亮点与贡献（Why it matters）

1. **把回放重构为可独立评估的最小原语**：学习价值与策略兼容两条轴显式分离、各自可消融，回放第一次被当作「单独的控制问题」研究。
2. **零额外生成、零新增训练机制**：选择只靠 teacher-forced 前向计算，on-policy 流与 GRPO 更新式完全不动，归因干净，可与 EFRame/ExGRPO/BAPO 等组合成可插拔控制层。
3. **agentic 场景给出墙钟意义上的帕累托改进**：用「重估旧轨迹」置换「昂贵的新环境交互」，正是回放价值最稀缺的用例。
4. **提供了回放与熵坍缩关系的新观察**：有原则的复用本身是稳定化机制，延迟熵坍缩。
5. **工程可落地**：τ 是粗校准（对数间隔三态即判可行域）而非细调，且跨规模复用（3B→7B 直接沿用）。

---

## 7. 局限与可改进点（个人点评）

- **最强卖点发生在最弱的统计对比上**：agentic 主设定 vs naive replay 的 Avg Mean@32 仅 +0.0029（作者自认噪声内），且 naive replay 每步 154.4s 其实比 Headroom-Drift 166.3s 更便宜——「省时」只对 on-policy scaling 成立，对朴素回放不成立。
- **规模与广度有限**：主实验 1.5B/3B/7B，30B+ 级模型上生成贵 vs 环境贵的代价画像会变，结论未必平移；7B 是单次运行无方差，0.0029 量级差值需更多重复。
- **入口规则偏窄**：只收「组内结果混合」的组，稀疏奖励长程任务（全组失败常见）缓冲会很快饿死；Geo3K 多奖励分析只在「如何定义成功」上打补丁，未解决稀疏奖励下无混合组可入的问题。
- **门控只测策略漂移、不测环境漂移**：环境本身在变的任务，旧轨迹失效原因与策略无关，L2 门控对此无感（对 GUI 场景尤其致命）。
- **「multimodal」名不副实**：全文无视觉编码器，用的是文本化几何题，跨界到真视觉 MLLM 的结论存疑。

---

## 8. 对我们的启示 / 可借鉴点

GUI Agent 的 RL 训练与本文痛点高度同构（每回合真实驱动浏览器/桌面、截图+辅助树解析，环境交互慢、并行受限），本文方案可逐条迁移：

- **「组」的切法**：GUI 里可用「同一任务/同一起始状态采样的多条动作轨迹」为组，组内相对优势天然成立；Headroom 的 token 级定义直接套到动作 token 上，语义不变。
- **Drift 门控必须扩展为「双漂移」**：叠加「屏幕/无障碍树相似度」作环境侧门控，否则 L2 门控会放过「策略没变但 UI 已变」的陈旧数据。
- **入口规则按稀疏奖励改造**：多步任务大多整条失败，需先拆奖励源（点击有效元素/达成子目标/最终成功），把「部分成功」定义成可用混合信号，否则回放缓冲早期就干涸。
- **省时逻辑直接成立**：回放组只需在存储序列上 teacher-forced 前向，GPU 并行即可、不占浏览器并发——「重估换交互」正是 GUI 训练最需要的收益形态。
- **可复用体检工具**：τ 三态诊断法（回放 KL 失衡即出局）可作 GUI 回放上线前的快速标定手段。

## 9. 延伸阅读

- **GRPO / DeepSeekMath（Shao et al., 2024）与 DeepSeek-R1（Guo et al., 2025, Nature）**：被增广的 RL 后训练基础。
- **RePO（Li et al., 2025, arXiv:2506.09340）**：把回放直接织入 GRPO 循环的直系前作。
- **EFRame（Wang et al., 2025）、ExGRPO（Zhan et al., ICLR 2026）、BAPO（Xi et al., ICLR 2026）**：回放+探索/过滤/缓冲优化对照系，本文原语可嵌入其中。
- **Prioritized Experience Replay（Schaul et al., 2016）与 RETRACE/IMPALA（Munos et al., 2016; Espeholt et al., 2018）**：Headroom 与 Drift 两轴的经典 RL 出处。
- **DAPO（Yu et al., 2025）**：最强非回放基线；**熵动力学（Wang et al., 2026b）**：熵坍缩分析参照。

---
*解读生成时间：2026-09-06 ｜ 解读人：WorkBuddy（AI）*
