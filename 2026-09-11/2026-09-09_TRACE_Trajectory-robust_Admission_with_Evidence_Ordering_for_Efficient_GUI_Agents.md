# TRACE: Trajectory-robust Admission with Evidence Ordering for Efficient GUI Agents

> 一句话 TL;DR：把 cache 复用下的视觉 token 剪枝重述为「写入时不可逆准入」，用布局先验+嵌套排序+原生 token 补覆盖+单调 KV 收缩，在 5% 预算下把 GUI grounding 精度保住 55.19%（对比基线 34.20%）。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | TRACE: Trajectory-robust Admission with Evidence Ordering for Efficient GUI Agents |
| **作者 / 机构** | Yuhao Wang、Mu Qiao、Xindong Zhang、Yunzhi Zhuge、Lei Zhang、Huchuan Lu；大连理工大学（中国）、OPPO 研究院（中国）、香港理工大学（中国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-09；arXiv preprint（arXiv:2609.10297v1，cs.CV），标注 Technical Report，未见 Venue |
| **arXiv 链接** | https://arxiv.org/abs/2609.10297 |
| **代码仓库** | ❌ 未开源（仅声明 "The source code will be released"，未提供 URL） |
| **数据集地址** | ❌ 未公开（不新建数据集；全部用公开 benchmark：ScreenSpot-v2 / ScreenSpot-Pro / MMBench-GUI L2 / OmniGUI / Mind2Web / AndroidControl） |
| **类型标签（论文类别）** | `GUI Grounding` `Mobile` `Desktop` `Web` |
| **训练方法标签** | —（综述/评测/工程；training-free，全程不训练） |
| **关键词** | Visual Token Pruning；KV Cache Reuse；GUI Agent Efficiency；Nested Ordering；Spatial Coverage |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-09_TRACE_Trajectory-robust_Admission_with_Evidence_Ordering_for_Efficient_GUI_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：GUI Agent 多步交互累积高分辨率截图，视觉 token 主导推理成本。复用 KV cache 只算新帧，但复用只在**帧粒度**省算力，帧内空间冗余仍在；而 token 一旦被剪掉，后续任何一步都无法恢复。
- **现有方法有什么不足**：①按帧复用 cache 但帧内冗余没动；②cache-side 压缩（GUI-KV、ST-Lite）保留原生 token 也免重编码，但**每步重新打分**，keep 不嵌套；③FastV 的分数只有 prefill 时才可得（违反「准入须在 prefill 前」），merge 系的合成 token 无法作为 cache 行被删，而**所有分数驱动方法都会把预算塌缩到少数显著区域**，丢失密集界面所需的空间覆盖。
- **Research Gap**：作者首次把该问题形式化为 **write-time visual evidence commitment under trajectory uncertainty**，指出两个挑战：(1) **轨迹不确定性**——目标已知但沿轨迹的细粒度目标未知，须在目标出现前提交证据；(2) **紧预算下的空间覆盖**——偏置剪枝会让整个可操作区域零支持，随预算收紧甚至**低于均匀采样**。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

在 LLM prefill 前把布局交互先验、指令相关性与特征新颖性融合成**一个嵌套排序**，再用原生 token 补回空间覆盖，最后让每帧按该排序的**前缀**单调收缩进可复用 session state。

### 3.2 方法总览（Pipeline）

- **输入**：当前帧视觉 token、非视觉前缀、已准入历史帧、指令。**输出**：嵌套 keep 序 `π`（best→worst）与无重编码的可复用 KV state。
- **三条约束（Eq.1）**：`S_t^cur = σ(A_t)`（只用 prefill 前信息）；`S_t^hist ⊆ S_t^cur`（退役单调嵌套）；两者 ⊆ `{1..N}`（须保留**原生** token，合成 merge 无法作为 cache 行删除）。
- **四模块**：①**LIP**——OmniParser-v2 的 `icon_detect` 取框后按框内四项能量算密度并散射到 token 网格，转成软质量 `m_j = 1 + αN·p_j`，基线 1 保证每个 token 仍可入选。②**NEO**——融合指令相关性与先验重加权特征 `ψ_j = √m_j·z_j`，做贪心正交残差排序 `j* = argmax(log a_j + log d_j²(S))`，天然产出嵌套序，**更紧预算只是取前缀**。③**NCR**——当前帧保护前缀后**跨步采样**补尾部；历史帧改用**中位数 token**。④**MKC**——新截图到达时**先收缩**（把已存嵌入裁到 `π1` 前 `k_h` 项，纯索引零算力），再与新帧、指令在一次合并 forward 中重放。历史帧**只跑一次检测器**。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：①**把 cache 复用下的剪枝重述为「写入时不可逆准入」**并给出三条约束——它把「精度掉了」变成「违反了哪条约束」，并解释 merge 系与 attention 系为何在复用路径上结构性不可用（Table 8 对照九个方法：FastV 违反 (i)(ii)，merge 三兄弟违反 (iii)）。②**一个序服务所有预算的嵌套排序**——「更紧预算 = 取前缀」是让 reuse 可实现的必要机制。③**NCR 当前帧跨步采样、历史帧用中位数**，由 Table 7 直接支撑（当前帧 stride 67.92 vs medoid 58.65）。
- **对性能提升最关键的设计**：①**NCR 是精度上贡献最大的单点**——温和预算 SS-v2 从 56.45%（LIP+NEO）跳到 **73.90%**（+17.45）。②**LIP 不可或缺**——去掉先验在 SS-v2 紧预算上 **−23.74%**，远超 −diversity（−5.35）与 −instruction（−15.57）。③**MKC 是效率上贡献最大的单点**——单加 MKC 即把 TTFT 从 1116.8ms 降到 487.4ms（精度不变）；**大头省在复用路径，剪枝只是第二重收益 + 精度来源**。
- **证据不足 / 仅声称有效**：①**「嵌套不损失精度」（Fig 1(b) 称 99.5%/99.1%）只在小规模探针上验证**，未说明规模与任务构成，而这一结论恰是全文前提。②**α 与剂量逐 benchmark、逐预算从网格里挑**——作者承认 Mind2Web 在 c=25% 差 3.0%、AndroidControl 紧预算差 4.1%，主表数字**部分是调参收益**。③**先验收益与检测器质量耦合**：检测器平均只覆盖网格 42.6%，先验转移实验最好仍只有 34.36%，**无法排除换更强检测器 TRACE 会更好**。④**长历史收益被作者显式承认未验证**：benchmark 最多保留 2~3 帧历史，`O(k_c + H·k_h)` 的优势**基本未被激活**。

---

## 4. 具体技术细节

### 4.1 模型结构

- **base model 全部是 MLLM（含视觉编码器）**，且是 GUI 原生 Agent 模型（直接在原始截图上 grounding 并输出动作）：主力 backbone 为 **GUI-Owl-1.5-8B** 与 **GUI-Owl-1.5-2B**，跨家族验证用 **UI-TARS-1.5-7B**。
- **是否冻结/微调：完全冻结、零训练**。论文明确与 CogAgent（高分辨率 cross-attention）、ReVision（学习削减时序冗余）这类 training-based 设计对立，理由是需额外训练算力且效率无法跨架构迁移。
- **额外组件**：OmniParser-v2 的 `icon_detect` 分支，**读像素而非编码器特征**，可并行于 vision encode，选择器总开销 47.0~67.4ms。

### 4.2 训练流程（若需要训练）

**不训练。** 本文不含任何训练阶段——无 SFT、无 RL、无蒸馏、无 LoRA，也不对检测器做 GUI 域微调（直接用现成权重）：

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| — | 不训练 | — | — | — | — |

论文中的「逐数据集选参」属**推理配置**而非训练：

| 配置项 | 作用 | 取值范围 | 选择依据 |
|---|---|---|---|
| 先验强度 `α` | 布局先验的偏置力度 | `{1,2,4,6}`，实用 `min(ᾱ, α*)` | 有效支持规则 `N_eff(α) ≥ ζk`（ζ=1） |
| 剂量 `ρ_cur` / `ρ_hist` | 补覆盖 token 比例 | `{0,0.2,0.3,0.5}` / `{0.05,0.10,0.20}` | 逐数据集/预算从公开网格选；`ρ_hist` 仅在 `k_h<k_c` 启用 |
| 预算 `r` / `(c,h)` | 单步保留比例 / 多步当前与历史预算 | mild: r=10% 或 c=50%,h=10%；tight: r=5% 或 c=25%,h=5% | 与所有 baseline 严格对齐 |

### 4.3 推理流程（若不训练或重点在推理）

- **推理模型**：冻结的 GUI-Owl-1.5（8B/2B）或 UI-TARS-1.5-7B，外挂 OmniParser-v2 icon_detect。
- **输入 / 输出**：输入为当前帧视觉 token、非视觉前缀、已准入历史帧与指令；输出为保留 token 索引集合（嵌套序 `π` 的前缀）与更新后的可复用 KV session state，供 LLM 解码 grounding 坐标或动作。
- **单步还是多步：两者都覆盖，多步是重点。** 单步是「编码一次、按 `r` 取 `π` 前缀」；多步是 **Prepare → Retire → Append 三阶段单调收缩循环**——每步先把上一当前帧收缩到 `k_h`，再与新帧、指令合并做一次 forward。终止由 episode 结束决定；**无 ReAct/反思式循环控制**，TRACE 是服务层机制，不改变 Agent 决策逻辑。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| ScreenSpot-v2 (SS-v2) | 跨 Web/Mobile/Desktop | 单步 grounding | 官方全集 | 截图 + 指令 | 点击坐标 | Accuracy |
| ScreenSpot-Pro (SS-Pro) | Desktop（专业高分辨率） | 单步 grounding | 官方全集（4K icon-dense） | 高分辨率截图 + 指令 | 点击坐标 | Accuracy |
| MMBench-GUI L2 | 跨平台 | 单步 GUI 理解 | 官方 L2（Basic/Adv 分列） | 截图 + 问题 | 答案 | Accuracy |
| OmniGUI | Mobile（全模态手机环境） | 多步动作预测 | 官方全集 | 多步截图序列 | 动作 | Step SR |
| Mind2Web | Web | 多步动作预测 | 官方全集 | 多步截图序列 | 动作 + 元素 | Step SR |
| AndroidControl | Mobile | 多步动作预测 | 官方全集 | 多步截图序列 | 动作 + 参数 | Step SR |

**统一口径**：Step SR 仅在**动作类型与目标/参数同时正确**时通过；点击/长按预测点须在归一化欧氏距离 0.04 以内。所有方法对齐同一 token 预算，并通过 5 项机械检查（no-op、单元测试、精确预算 `|kept−⌈rN⌉|≤1`、位置一致性、多步账本）。

### 5.2 实验结果分析

**主结果（Table 1，backbone = GUI-Owl-1.5-8B，上界为 100% token）**

| 方法 | SS-v2 | SS-Pro | MMBench-GUI | OmniGUI | Mind2Web | AndroidControl | Avg.(%) |
|---|---|---|---|---|---|---|---|
| **Dense 上界** | 93.79 | 70.34 | 82.64 | 52.45 | 52.90 | 60.41 | 100.0% |
| Uniform (mild) | 43.79 | 10.69 | 28.27 | 46.11 | 39.65 | 59.08 | 59.5% |
| VisionTrim (mild) | 58.65 | **38.52** | 40.71 | 45.72 | 43.91 | 59.10 | 72.4% |
| PruneSID (mild) | **58.96** | 15.69 | 40.65 | 47.40 | 43.28 | 59.52 | 67.5% |
| **TRACE (mild)** | **73.90** | 37.63 | **50.47** | **48.91** | **45.52** | **60.20** | **78.7%** |
| Random (tight) | 15.49 | 1.27 | 9.29 | 34.25 | 18.43 | 52.07 | 36.0% |
| PruneSID (tight) | 34.20 | 4.55 | 19.48 | 40.09 | 27.57 | 55.46 | 47.8% |
| VisionTrim (tight) | 25.86 | 18.22 | 16.86 | 38.26 | 30.20 | 55.16 | 49.2% |
| **TRACE (tight)** | **55.19** | **20.30** | **30.88** | **43.08** | **34.20** | **57.06** | **61.1%** |

- **预算越紧优势越大**：mild 下领先 PruneSID **+14.94**（SS-v2）、领先 VisPruner **+8.12**（MMBench-GUI），但 SS-Pro 落后 VisionTrim **0.89**；到 r=5% 时对 PruneSID 优势扩大到 **+20.99**，对 VisionTrim 在 SS-Pro 上翻转为 **+2.08**。
- **多步与跨规模**：多步领先从 mild 的 0.54~0.74 分扩到 tight 的 1.30~3.53 分（最大差距在 Mind2Web：TRACE 34.20 vs VisPruner 30.67）。GUI-Owl-1.5-**2B** 紧预算下保住 dense 的 **52.9%**，PruneSID 只保住 43.5%。UI-TARS-1.5-7B 上 tight 领先 DivPrune 0.79~4.92 分（tight Mind2Web：17.92 vs 13.00）。
- **服务效率（OmniGUI, mild）**：dense 4892 token / TTFT 1102.6ms / E2E 2958.2ms / KV 697MB / Step SR 52.45 → TRACE 2011 token / 选择开销 67.1ms / **TTFT 453.9ms** / **E2E 2295.8ms** / **KV 292MB** / Step SR 48.91（保住 93.3%）。**TTFT −59%，视觉 KV −58%（紧预算 −68.1%），比 dense re-prefill 快 2.4×**。反例：PruneSID 选择开销 740.0ms 把 TTFT 抬到 1506.9ms——**选择成本高于 prefill 收益时剪枝反而变慢**。

**Ablation 说明了什么**

- **模块消融（Table 2）**：温和预算 SS-v2 上 LIP-only 32.63 → NEO-only 36.48 → LIP+NEO **56.45** → 加 NCR **73.90**；紧预算 Mind2Web 27.79 → **34.20**。结论：LIP 与 NEO **必须组合**（合用比任一单用高 19.97 分），**NCR 是不可省的最后一跳**且绝对分差最大。
- **失败归因（Appendix C.1）**：r=5% 时漏掉 503 个 dense 能解的例子，其中 **93.0% 仍有保留 token 与目标重叠**，「目标空间证据被删除」只解释 0.8% 的差距；剩余失败集中在**薄/低对比度控件**（目标仅占屏 0.15%~0.24%）。**瓶颈是精度而非「没找到区域」**。

---

## 6. 亮点与贡献（Why it matters）

1. **重新定义了问题边界而非只刷指标**。三条约束把一大类方法的适用性讲清；Table 8 用这三条逐条判定九个已有方法，本身就是一份可复用的方法学清单。
2. **嵌套序是「一份预算、多处复用」的正确抽象**，让服务端在同一份已准入证据上实现任意预算。
3. **覆盖修补揭示了一个被忽视的失效模式**：分数驱动剪枝都会塌缩到少数显著区域，在密集 GUI 上表现为「整个按钮区域零保留」；NCR 用极小预算换来 SS-v2 +17.45 分且不破坏嵌套性。

## 7. 局限与可改进点（个人点评）

- **最根本的局限是长历史收益未被验证**。全文动机建立在 `O(k_c + H·k_h)` 上，但现有 benchmark 最多保留 2~3 帧历史——**TRACE 最该赢的场景几乎没被测**。要证明 MKC 价值，需构造「证据在第 5 帧、查询在第 15 帧」的轨迹。
- **先验质量与方法质量未正交分离**：检测器平均只覆盖 42.6% 网格；「先验转移」改变的是方法而非检测器，无法回答「换更强检测器 TRACE 还能提升多少」。理想做法是固定 TRACE、扫检测器质量做上界分析。
- **SS-Pro 是明确短板**（mild 落后 VisionTrim 0.89%、r=50% 落后 VisPruner 1.90%），而它最接近真实桌面高分辨率场景；这与「残差失败集中在占屏 0.15%~0.24% 的薄控件」是同一问题的两面。

## 8. 对我们的启示 / 可借鉴点

**对 GUI Agent 的落地启示（本篇为 GUI 域效率论文，覆盖 Web/Mobile/Desktop）：**

1. **服务端 Agent 的正确抽象是「准入一次 + 单调收缩」**。应把「截图编码」与「帧内 token 选择」解耦成一次性准入，并把保留集合设计成**嵌套前缀**，使同一份证据可在不同预算/延迟档位复用。这个改造的收益（−59% TTFT、−58% KV）远大于在决策侧做优化。
2. **覆盖修补是低成本高回报的必备件**：在密集 GUI 上用极少量预算（5%~20%）做空间铺底（当前帧跨步采样、历史帧中位数代表）即可避免「整个可操作区域零保留」，且不破坏嵌套性。
3. **评测要覆盖紧预算档，失败分析要区分「没找到区域」与「找到了但定位不准」**——93% 的失败中保留 token 已覆盖目标，仅 0.8% 差距源于证据被删。薄/低对比度小控件在 patch 粒度上难与背景分离，**提高输入分辨率或引入 sub-patch 可能比改进剪枝更有效**。

## 9. 延伸阅读

- **An Image is Worth 1/2 Tokens After Layer 2 (FastV)**（Chen et al., ECCV 2024）——attention-based 剪枝代表，也是「违反 pre-prefill 与 nested 约束」的典型反例。
- **DivPrune: Diversity-based Visual Token Pruning for Large Multimodal Models**（Alvar et al., CVPR 2025）——特征多样性路线，NEO 正交残差贪心的直接对照。
- **GUI-KV: Efficient GUI Agents via KV Cache with Spatio-Temporal Awareness**（Huang et al., 2025）——最接近本文设定的 cache-side 方法，每步重打分、keep 不嵌套，建议作为后续必做对比。

---
*解读生成时间：2026-09-11 09:00 ｜ 解读人：WorkBuddy（AI）*
