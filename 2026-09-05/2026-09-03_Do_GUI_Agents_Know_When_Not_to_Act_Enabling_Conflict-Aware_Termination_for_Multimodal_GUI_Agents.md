# Do GUI Agents Know When Not to Act? Enabling Conflict-Aware Termination for Multimodal GUI Agents

> TL;DR：GUI Agent 不仅要会"做"，还要知道"不该做"——本文发布 CONFLICTGUI 基准（可执行指令内部冲突 / 指令与界面上下文冲突）并给出 CONFLICTGUARD 推理期干预框架，把冲突场景成功率从不足 10% 提到近六成。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | Do GUI Agents Know When Not to Act? Enabling Conflict-Aware Termination for Multimodal GUI Agents |
| **arXiv ID / DOI** | arXiv:2609.03438v1（cs.AI） |
| **arXiv 链接** | https://arxiv.org/abs/2609.03438（点击直达） |
| **发表出处（Venue）** | arXiv preprint（预印本，未标注会议/期刊） |
| **发布时间** | 2026-09-03 |
| **作者** | Zhaoyuan Huang、Tianjie Ju、Pengzhou Cheng、Zheng Wu、Yansi Li、Chuanbiao Song、Jun Lan、Huijia Zhu、Weiqiang Wang、Zhuosheng Zhang（‡ 通讯：Jun Lan、Zhuosheng Zhang） |
| **所属机构** | 上海交通大学计算机科学学院（中国）：Huang/Ju/Cheng/Wu/Li/Zhang；蚂蚁集团（中国）：Song/Lan/Zhu/Wang（一作 Huang 在蚂蚁集团实习期间完成部分工作） |
| **开源情况 / 代码** | ✅ 有开源：代码与数据见 https://github.com/serein356/ConflictGuard（摘要中声明 "Code and dataset are available at"） |
| **类型标签（论文类别）** | `GUI Grounding` `Benchmark` `Mobile` |
| **训练方法标签** | —（方法/评测：推理期"可行性验证提示 + 条件激活引导"，不训练/微调任何模型参数） |
| **关键词** | Conflict-aware termination; GUI agent reliability; Execution-biased over-compliance; Feasibility verification; Activation steering; CONFLICTGUI |
| **来源渠道** | arxiv-web-search（arXiv export API 当日不可达，走网页搜索 + 分类 listing 交叉验证回退） |
| **PDF 存档** | papers/2026-09-03_Do_GUI_Agents_Know_When_Not_to_Act_Enabling_Conflict-Aware_Termination_for_Multimodal_GUI_Agents.pdf |

---

## 1. 研究背景与要解决的问题

多模态大模型（MLLM）驱动的 GUI Agent 已能看懂截图、听懂自然语言指令，并完成点击、输入、滚动等操作。绝大多数评测（GUI Grounding、动作预测、多步任务完成）都默认一个前提：**用户指令是"合理且可执行"的**，成功率衡量的就是"agent 最终有没有把要求的事做出来"。

但真实用户会犯错——而且常是无心之失。比如对着音乐 App 说"帮我订个披萨"，或页面上明明显示洛杉矶的机票却让 agent"选这个去巴黎的航班"。此时一个可靠的 agent 应该**停下来、向用户说明冲突**，而不是照指令表面语义硬执行。盲目执行轻则白耗算力、陷入死循环，重则引发不可逆操作（如误删）甚至隐私泄露。

已有工作（如 VeriOS、VenusBench-GD 的 refusal grounding）只覆盖"证据缺失或目标模糊"时应拒绝/确认的情形，但**更普遍的冲突**——指令内部自相矛盾、指令与当前界面证据不符——仍少有人系统研究。本文的缺口正是：把"何时不该行动（conflict-aware termination）"显式形式化，证明主流 agent 在这类场景下病态"过度顺从"，并提出一个不训练模型即可大幅缓解的推理期方案。

## 2. 核心方法 / 思路

全文分两大部分：诊断问题的**基准 CONFLICTGUI** 与解决问题的**框架 CONFLICTGUARD**。

**（1）问题形式化。** 把 GUI 交互写成逐步决策：第 t 步 agent 观察界面 `g_t`（截图 I_t + 历史 H_t），给定指令 q 预测下一动作。动作空间除常规 GUI 动作外含任务级动作 `terminate`。指令可行性定义为两个条件合取：**L(q)** 指令内部逻辑自洽，**C(q, g_t)** 指令被当前界面证据支持，任一不成立即不可执行。据此分两类冲突：①**指令内部冲突（C1）**——只看指令文本就能识破（"点删除键来保存文件"）；②**指令-GUI 上下文冲突（C2）**——指令与当前截图矛盾（"点红色按钮"而界面全是蓝色）。两类情形期望动作都是 terminate + 报告冲突，而不是换个可执行动作凑合。

**（2）CONFLICTGUI 数据构造。** 从 AMEX、AndroidControl、AITZ 三个移动 GUI 数据集抽取截图、指令与参考动作，形成 2,364 条**可行指令**作对照；再用 Kimi-K2.5 与 Gemini-2.5 Pro 把可行样本改写为"只注入一个冲突"的变体，得 1,122 条 C1、1,174 条 C2，形成**可行-冲突配对**。全部样本两名标注员独立复核 + 第三名抽检 100 条/类，通过率 95%（C1）/98%（C2）。

**（3）CONFLICTGUARD：两段式设计。** 对 vanilla 的定性分析发现两种失败模式：**前提盲执行**（不验证指令合理性就照表面语义做）与**意识-动作错配**（推理里察觉冲突，输出仍是可执行动作）。据此分两步：
- **离线校准**：利用可行-冲突配对，在冻结 agent 的动作生成起始隐状态上，对每类冲突做"可行 vs 冲突"逐层激活差，用 PCA 第一主成分提取**冲突条件方向** d^c_l（两类各一条）；再对可行样本强制作答"终止型后缀"与"过度顺从型后缀"，提取**反过度顺从方向** v_l（把模型往"终止"而非"执行"推）。
- **推理期干预**：①动作生成前先加**可行性验证提示**（检查指令逻辑、界面证据、动作后果）；②当隐状态与冲突方向相似度超过阈值 θ_c（两类经 OR 门融合）时，向若干解码层注入 α·v_l 引导终止。门未触发则完全走原模型前向，避免给可行任务强加普遍拒答偏向。

类比：可行性验证像"安检员先问：这事合理吗、界面支持吗"；条件引导像"安检亮红灯时用语义方向把模型从'执行'推上'停手'轨道"。

## 3. 关键实验结果

**评测设置**：在 CONFLICTGUI 上评测 10 个模型——通用 MLLM（GPT-5、Claude Sonnet 4.6、GLM-4.5V、Qwen3-VL-4B/8B/32B）与 GUI 特化 agent（UI-Venus-1.5-8B、UI-TARS-1.5-7B、OS-Atlas-Base-7B、AgentCPM-GUI）。校准集 600 对（两类各 300），测试集与校准严格不重叠。主指标 SR（可行指令=动作匹配；冲突样本=必须 terminate 且以 failure 状态指出冲突）+ FEX（冲突下仍执行参考动作的比例）。对比三档：Vanilla / 仅 Feasibility Prompt / CONFLICTGUARD。

**发现一：vanilla 严重过度顺从。** 可行任务平均成功率普遍超 70%，但冲突 SR 平均**低于 10%**，平均 FEX 超 70%。即"执行强"与"会停手"完全不是一回事。

**发现二：纯提示不足。** Feasibility Prompt 对 Qwen3-VL、GPT-5 有效，但对 UI-Venus、AgentCPM-GUI 几乎无提升，OS-Atlas 甚至 Overall SR 下降——提示能暴露部分可行性信号，却压不过执行先验、难以把"意识到冲突"可靠转成"终止动作"。

**发现三：CONFLICTGUARD 显著缓解（表 1 核心数字）**：
- 5 个开源模型上平均 Conflict SR 从 **6.91% → 58.63%**，平均 FEX 从 **73.37% → 32.76%**；可行 SR 仅从 75.77% 微降至 73.15%，证明不是无差别拒答。
- 最优在 Qwen3-VL 系：8B 冲突 SR 11.39%→70.08%，32B 8.41%→77.79%；GUI 特化 agent 有提升但不均匀（疑因其后训练过度强化可执行动作预测）。

**消融（图 3）**：去掉可行性验证，Qwen3-VL-8B / UI-TARS Overall SR 各掉 12.1 / 11.3 点；去掉条件引导，两类冲突 SR 明显下滑；**去掉条件门、无条件加引导向量**则灾难——可行 SR 从 70.80% 掉到 46.20%（Qwen3-VL-8B）、76.80%→29.20%（UI-TARS）——"选择性触发"是保可行任务的关键。

**泛化**：源分离校准（只拿 AMEX 校准、在 AndroidControl 评测及反向）Qwen3-8B 两方向 Conflict SR 66.43%/65.00%（全量校准 70.08%），冲突方向不绑死单一数据源；外部基准 VenusBench-GD（refusal grounding）Qwen3-8B/32B、UI-TARS 分别 +53.95/+73.42/+15.24 点，GUIOdyssey（可行任务）几乎不变（±0.30 点内）——终止行为可迁移且不伤可行任务。

**效率与长程（表 4、图 5）**：Qwen3-VL-8B 单样本 5.76s→5.26s 且 token 更少，其余模型变化微小——无端到端延迟开销。长程初步实验（50 可行+50 冲突、冲突需几步后才显形）里 vanilla 冲突 SR 仅 2%，可行性提示 12%，CONFLICTGUARD 达 36%（平均 SR 26%→42%）。

## 4. 亮点与贡献（Why it matters）

1. **首次把"冲突感知终止"显式建基准**：可行-冲突配对设计让"过度顺从"第一次可量化——实证可行性 SR>70% 与冲突 SR<10% 的鸿沟。
2. **两个可复用的失败模式命名**：前提盲执行与意识-动作错配，点出"会执行≠会判断该不该执行"。
3. **轻量推理期干预范式**：提示级可行性验证 + 表征工程级条件引导，不训练参数、可插拔到任意开源权重 agent；条件闸门避免普遍拒答的思路值得借鉴。
4. **严谨的实验纪律**：校准/测试严格分离、源分离与外部基准双路泛化、FEX 直击过度顺从、报告延迟与 token 成本——方法学模板。
5. **呼应"可靠性优先"的行业需求**：落地时"会停手、会求助"是安全底线，本工作也明确了白盒假设的边界。

## 5. 局限与可改进点（个人点评）

- **评测以单步决策为主**：CONFLICTGUI 的冲突在"看当前截图"时即可判定；长程实验仅 100 条任务、冲突 SR 36%，"冲突在多步后浮现"仍是开放问题——恰是真实长任务最常见的形态。
- **依赖白盒隐状态**：激活引导须访问模型内部，**无法用于 GPT-5 / Claude / GLM-4.5V 等闭源 API**（表 1 中它们只能跑 Vanilla/Feasibility Prompt 两档）。与前日《Monitoring Web Agents Without Internal Signals》对照：监控可纯黑盒，干预目前仍需白盒。
- **开源模型间提升差异大**：GUI 特化 agent（UI-Venus/UI-TARS）提升远不如 Qwen3-VL，文中归因于后训练强化执行先验但无验证实验——"什么样的训练让 agent 更会停手"未被回答。
- **合成数据覆盖有限**：冲突变体由 VLM 生成，虽人工复核通过率高，但仅两类冲突、集中在移动端，网页/桌面端缺失；终止后如何澄清并接续执行也未展开。

## 6. 对我们的启示 / 可借鉴点

1. **给"该不该做"单独设一道闸**：做 RL/Grounding 时我们通常只优化"做得准不准"，本文提示应把"可执行性判断"作为独立能力评测与干预点——动作头前加可行性验证提示几乎零成本，可直接纳入我们的数据管线。
2. **"停止/拒绝"也应成为动作空间的头等公民**：CONFLICTGUI 把 terminate 当普通动作并要求带 failure 状态与理由，这种"动作+状态+理由"的结构化输出很适合我们训练数据的 schema 设计。
3. **表征工程的性价比**：只要离线有可行-冲突对照对，PCA 方向 + 阈值门 + 少量层注入即可在不训练前提下改行为——若我们有 grounding 正负样本，可低成本复刻这套"条件引导"。
4. **评测设计范本**：可行-冲突配对 + FEX 指标 + 校准/测试隔离，是做行为对齐类评测的现成模板；也可用 VenusBench-GD 验证"终止能力"的跨任务迁移。

## 7. 延伸阅读

- **VeriOS**（Wu et al., 2025）：不可靠场景请求人工确认的 GUI agent 框架，与"何时求助"互补。
- **VenusBench-GD**（Zhou et al., 2025）：refusal grounding 基准，本文用它做外部泛化验证。
- **CAST / Conditional Activation Steering**（Lee et al., 2025, ICLR）：本文条件引导的技术源头。
- **AMEX / AndroidControl / AITZ**：CONFLICTGUI 的三大来源数据集（移动 GUI）。
- **Faithful Mobile GUI Agents with Guided Advantage Estimator**（Hu et al., 2026，arXiv:2605.01208）：证据锚定的 GUI 执行，同一脉络。

---
*解读生成时间：2026-09-05 09:10 ｜ 解读人：WorkBuddy（AI）*
