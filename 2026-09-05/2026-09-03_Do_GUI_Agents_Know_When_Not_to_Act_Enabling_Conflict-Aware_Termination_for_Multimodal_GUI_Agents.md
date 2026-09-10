# Do GUI Agents Know When Not to Act? Enabling Conflict-Aware Termination for Multimodal GUI Agents

> 一句话 TL;DR：本文首次把 GUI Agent 的"冲突感知终止"做成可行-冲突配对基准 CONFLICTGUI，并给出推理期框架 CONFLICTGUARD（可行性验证提示 + 条件激活引导），将冲突场景成功率从不足 10% 提到近六成。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Do GUI Agents Know When Not to Act? Enabling Conflict-Aware Termination for Multimodal GUI Agents |
| **作者 / 机构** | Zhaoyuan Huang（一作，上海交大在读，蚂蚁集团实习期间完成）、Tianjie Ju、Pengzhou Cheng、Zheng Wu、Yansi Li、Chuanbiao Song、Jun Lan（‡ 通讯）、Huijia Zhu、Weiqiang Wang、Zhuosheng Zhang（‡ 通讯）；上海交通大学计算机学院 + 蚂蚁集团（中国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-03；arXiv preprint（cs.AI，无会议标注） |
| **arXiv 链接** | https://arxiv.org/abs/2609.03438 |
| **代码仓库** | ✅ GitHub：https://github.com/serein356/ConflictGuard |
| **数据集地址** | CONFLICTGUI 随代码同仓库发布（未单列 HF），由 AMEX / AndroidControl / AITZ 三个移动数据集改写合成 |
| **类型标签（论文类别）** | `GUI Grounding` `Benchmark` `Mobile` |
| **训练方法标签** | `—（评测/工程：推理期激活引导 + 提示，不训练任何模型参数）` |
| **关键词** | conflict-aware termination；execution-biased over-compliance；feasibility verification；activation steering；CONFLICTGUI |
| **来源渠道** | arxiv-api / github-awesome |
| **PDF 存档** | papers/2026-09-03_Do_GUI_Agents_Know_When_Not_to_Act_Enabling_Conflict-Aware_Termination_for_Multimodal_GUI_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：多模态 GUI Agent 面对"不可执行指令"时，能否正确**停手并报告冲突**（conflict-aware termination），而非照指令表面语义硬执行。任务属 Mobile GUI 端到端动作决策。
- **为什么重要**：真实用户常因无心之失发出不可行指令（用 Spotify 订披萨、选"去巴黎"的航班而页面是洛杉矶）。盲目执行轻则白耗算力、陷入无意义循环，重则触发不可逆操作（误删）或隐私泄露。这是"可靠性"而非"能力"的底线问题。
- **现有方法有什么不足**：绝大多数 GUI 评测（grounding / 动作预测 / 多步完成）默认指令"合理可执行"，成功率只衡量"有没有做成"。VeriOS 只处理"不可靠场景求助"，VenusBench-GD 的 refusal grounding 只覆盖"证据缺失/目标模糊"时应拒绝；更普遍的**指令内部自相矛盾、指令与界面证据冲突**仍无系统研究。定性分析还暴露两种失败模式：**前提盲执行**（不验证就照做）与**意识-动作错配**（推理里发现了冲突、输出却仍是可执行动作）。
- **Research Gap**：作者认为缺口在于把"何时不该行动"显式形式化为 `V(q,g_t)=L(q)∧C(q,g_t)`（指令逻辑自洽 + 被当前界面证据支持），证明主流 agent 病态"过度顺从"，并给出一个**不训练即可大幅缓解**的推理期方案。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

用可行性验证提示先"暴露冲突证据"，再用离线校准出的语义方向做**条件门控的激活引导**，把模型的过度顺从输出"扳"向终止动作，且只在检测到冲突时才介入。

### 3.2 方法总览（Pipeline）

- **输入**：截图 `I_t` + 交互历史 `H_t` + 用户指令 `q`（拼接后追加可行性验证提示）。
- **输出**：动作 `a_t`，含常规 GUI 动作（click/scroll/type/press_button）或任务级 `terminate`（带 `status=failure`）。
- **模块**：
  1. **可行性验证提示**：动作生成前要求模型核查指令逻辑、界面证据、动作后果，自相矛盾或不匹配则终止。
  2. **冲突条件方向 `d^c_l`**（离线）：用可行-冲突配对在动作生成起始位做逐层激活差，PCA 第一主成分即"冲突方向"，两类冲突各一条。
  3. **反过度顺从方向 `v_l`**（离线）：对可行样本强制作答"终止型后缀"与"顺从执行型后缀"，二者隐态差的 PCA 主方向。
  4. **条件门控 + 引导**（推理）：当前隐态与 `d^c_l` 余弦相似度超阈值 θ_c 则触发（两类 OR 门），向若干解码层注入 `α·v_l`；未触发则走原前向。
- **连接**：离线校准产出方向与阈值 → 推理期"可行性提示改写输入 → 前向 → 条件检测 → 若命中则加引导向量，否则正常解码"。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：把"冲突感知终止"首次形式化并建基准；将 CAST 式条件引导扩展为**"冲突条件方向"与"反过度顺从方向"分离、双条件 OR 门融合**，解决了"何时介入"与"往哪介入"解耦的问题。
- **工程组合**：可行性验证提示是纯 prompt 工程；PCA 方向提取、激活引导直接继承 activation steering / CAST / ITI；数据构造是"VLM 改写 + 人工复核"标准流程。这些单看都不新。
- **对性能提升最关键的设计**：**条件门控**——去掉门、无条件加引导向量会导致可行 SR 灾难性下跌（Qwen3-8B 70.80%→46.20%、UI-TARS 76.80%→29.20%）；可行性验证与条件引导互补（各删其一 Overall SR 掉 11~12 点）。
- **证据不足 / 仅声称有效**：长程冲突仅 100 条、SR 仅 36%，且方向仍用单步校准；"GUI 特化 agent 提升不均匀源于后训练强化执行先验"纯属猜测，无对照实验。

---

## 4. 具体技术细节

### 4.1 模型结构

- **Base model 是 MLLM**（含视觉编码器）：本文在 Qwen3-VL-4B/8B/32B-Instruct、UI-Venus-1.5-8B、UI-TARS-1.5-7B 五个开源 VLM 上应用 CONFLICTGUARD。参数量 4B/8B/32B/7B 不等。
- **全部冻结，零微调**：只在动作生成起始位（assistant 首个 token 之前的最后输入 token）与解码 token 的隐态上做加法注入，扰动只在语言侧/多模态融合后的表示层，不碰图像与历史表示。
- 闭源 GPT-5、Claude Sonnet 4.6、GLM-4.5V 无法访问内部隐态，只能跑 Vanilla / Feasibility Prompt 两档——这是白盒假设的硬边界。

### 4.2 训练流程（本篇不训练，仅"离线校准"）

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| 校准①冲突条件方向 | 提取"可行→冲突"的激活位移方向 | 无（表征提取） | 300 C1 + 300 C2 可行-冲突配对（AMEX/AndroidControl/AITZ） | 可行样本与冲突变体的动作生成起始位逐层隐态 | 无 loss；PCA 第一主成分 `d^c_l` |
| 校准②反过度顺从方向 | 提取"执行→终止"的行为方向 | 无（表征提取） | 可行样本 + 强制后缀（终止型 y⁺ / 执行型 y⁻） | 后缀 token 逐层隐态均值差 | 无 loss；PCA 主成分 `v_l` |

> 超参 `l_c`（条件层，取可行-冲突 PCA 对比的高方差区）、`θ_c`、`α`、`L_b` 均在校准集上网格搜索，优化"冲突 SR vs 可行 SR"权衡。校准集与测试集严格 disjoint。

### 4.3 推理流程

- **以单步推理为主**：输入 = 截图 + 历史 + 指令 + 可行性验证提示；输出 = 单步动作（含 terminate）。核心是逐步决策 `a_t = π_θ(q, g_t)` 的"当步判定"。
- **多步/长程为初步实验**：长程任务中冲突在多步后才显形（如导航到某页才发现目标不存在），方向仍复用单步校准，未做在线重校准；终止条件即冲突门触发后输出 `terminate(status=failure)`。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| **CONFLICTGUI**（本文） | Mobile | 冲突感知终止 / 动作预测 | 2,364 可行 + 1,122 C1 + 1,174 C2（校准 600 对 + 测试 1,800/822/874） | 截图 + 指令 | 动作（含 terminate+status） | SR、FEX、Overall SR |
| VenusBench-GD（Refusal 子集） | Mobile/多平台 | refusal grounding（外部泛化） | 子集 | 截图 + 指令 | grounding/拒答 | 成功率 |
| GUIOdyssey | Mobile | 跨 App 导航（可行任务，外部泛化） | 基准集 | 截图 + 指令 | 动作序列 | 成功率 |

### 5.2 实验结果分析

**主结果（Table 1）**：
- **发现一（过度顺从）**：Vanilla 下所有模型可行任务 SR 普遍 >70%，但冲突 SR 平均 **<10%**、平均 FEX **>70%**——"会执行"与"会停手"是两回事。
- **发现二（提示不足）**：Feasibility Prompt 对 Qwen3-VL、GPT-5 有效，但 UI-Venus、AgentCPM-GUI 几乎无提升，OS-Atlas 甚至 Overall SR 下降——提示暴露了部分信号，却压不过执行先验。
- **发现三（CONFLICTGUARD 显著缓解）**：5 个开源模型平均冲突 SR **6.91%→58.63%**，FEX **73.37%→32.76%**，可行 SR 仅 **75.77%→73.15%**（证明不是无差别拒答）。最优在 Qwen3-VL 系：8B 冲突 SR 11.39%→70.08%，32B 8.41%→77.79%。

**Ablation（图 3）**：去掉可行性验证，Overall SR 掉 12.1（Qwen3-8B）/11.3（UI-TARS）；去掉条件引导，C1/C2 SR 明显下滑；**去掉条件门**导致可行 SR 灾难性下跌（见 3.3）——选择性触发是保可行任务的关键。

**泛化与效率**：源分离校准（AMEX↔AndroidControl）Qwen3-8B 冲突 SR 仍有 66.43%/65.00%；外部 VenusBench-GD 上 Qwen3-8B/32B/UI-TARS 分别 +53.95/+73.42/+15.24 点，GUIOdyssey 可行任务几乎不变（±0.30 内）；Qwen3-8B 单样本 5.76s→5.26s、token 更少，无端到端延迟开销。长程初步实验中冲突 SR：Vanilla 2% → Prompt 12% → CONFLICTGUARD 36%。

---

## 6. 亮点与贡献（Why it matters）

1. **首次把"何时不该行动"做成可量化基准**：可行-冲突配对设计让"过度顺从"第一次有了硬指标（可行 SR>70% vs 冲突 SR<10% 的鸿沟），并定义了 FEX 直击"假执行"。
2. **两个可复用的失败模式命名**：前提盲执行、意识-动作错配，精准点出"会执行 ≠ 会判断该不该执行"，为后续诊断提供语言。
3. **轻量、可插拔、零训练的干预范式**：提示 + 表征工程即可在不改参数前提下把冲突 SR 提到近六成，且条件门避免普遍拒答——工程性价比极高。
4. **实验纪律堪称模板**：校准/测试严格隔离、源分离 + 外部基准双路泛化、报告延迟与 token 成本。
5. **锚定"可靠性优先"的产业需求**：落地场景里"会停手、会求助"是安全底线，本文明确了这条能力线及其白盒边界。

---

## 7. 局限与可改进点（个人点评）

- **主评测是单步判定**：CONFLICTGUI 的冲突看当前截图即可识别，长程实验仅 100 条、冲突 SR 仅 36%，而"冲突在多步交互后浮现"恰是真实长任务最常见的形态——这才是真正难啃的骨头。
- **白盒假设限死适用面**：激活引导必须访问隐态，GPT-5/Claude/GLM 等闭源 API 完全无法受益；黑盒侧能否用"监控+外部终止器"替代仍是开放问题。
- **GUI 特化 agent 提升不均且归因悬空**：UI-TARS/UI-Venus 远不如 Qwen3-VL，"后训练强化了执行先验"只是推测，没有用训练数据消融去验证——这其实暗示了一个更本质的问题：**什么样的训练会让 agent 更愿意停手**，本文没回答。
- **覆盖范围窄**：冲突仅两类、集中在移动端、由 VLM 合成，网页/桌面端、终止后如何澄清并接续执行均未展开；`terminate` 的评估只要求 status=failure，对"理由质量"无细粒度打分。

---

## 8. 对我们的启示 / 可借鉴点

1. **把"该不该做"从"做得好不好"里拆出来单独设闸**：我们的 RL/Grounding 管线通常只优化"准不准"，应把"可执行性判断"作为独立能力评测与干预点——动作头前加一条可行性验证提示几乎零成本，可直接进数据管线。
2. **"停止/拒绝"应是动作空间的头等公民**：CONFLICTGUI 把 terminate 当普通动作并强制带 status+理由，这套"动作+状态+理由"结构化 schema 值得直接抄进训练数据的输出模板。
3. **表征工程的高性价比可低成本复刻**：只要有"正-负对照对"（grounding 正样本/反样本、可执行/不可执行），PCA 方向 + 阈值门 + 少量层注入就能在不训练前提下改行为；若我们要做"拒绝行为对齐"，这比再训一轮便宜得多。
4. **评测设计范本**：可行-冲突配对 + FEX + 校准/测试隔离，是做行为对齐类评测的现成模板；可用 VenusBench-GD 验证"终止能力"的跨任务迁移，用 GUIOdyssey 确认不伤可行任务。

---

## 9. 延伸阅读

- **CAST / Conditional Activation Steering**（Lee et al., 2025, ICLR）：本文条件引导的直接技术源头。
- **VeriOS**（Wu et al., 2025）：不可靠场景下主动寻求人工确认的 OS agent，与"何时求助"互补。
- **VenusBench-GD**（Zhou et al., 2025）：refusal grounding 基准，本文外部泛化验证对象。
- **Faithful Mobile GUI Agents with Guided Advantage Estimator**（Hu et al., 2026, arXiv:2605.01208）：证据锚定执行，同一"忠实性"脉络。
- **AMEX / AndroidControl / AITZ**：CONFLICTGUI 三大来源数据集。
- **Monitoring Web Agents Without Internal Signals**（同批论文）：黑盒监控与本文白盒干预互为镜像，可对照思考"无内部信号时如何判定该不该停"。

---
*解读生成时间：2026-09-05 ｜ 解读人：WorkBuddy（AI）*
