# What Does Multi-Harness RL Learn? Credit Assignment and Portability in Coding Agents

> 固定曝光与预算、只移动 GRPO 分组边界：Cross（harness 混池）与 Within（逐 harness 分组）在未见过 harness 上无统计差异，评测 harness 才是主导变量。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | What Does Multi-Harness RL Learn? Credit Assignment and Portability in Coding Agents |
| **arXiv ID / DOI** | 2609.04518v1 [cs.AI] |
| **arXiv 链接** | https://arxiv.org/abs/2609.04518（点击直达） |
| **发表出处（Venue）** | arXiv preprint |
| **发布时间** | 2026-09-03 |
| **作者** | Chenqian Le、Jiayi Cheng（并列一作）；Qijia He；Runhao Li；Yinghao Li；Xupeng Chen（通讯，Dimension Gate） |
| **所属机构** | 纽约大学（美国）；华盛顿大学（美国）；南加州大学（美国）；哥伦比亚大学（美国）；Dimension Gate（通讯作者单位） |
| **开源情况 / 代码** | ⚠️ 文中声明随文发布 artifacts（PLAINCROSS、weak-ReAct 及可重现全部数字的脚本/JSON），但未给出可点击 URL |
| **类型标签（论文类别）** | `RL` `Online` `General` |
| **训练方法标签** | `SFT warm-start（成功轨迹蒸馏）+ RL (GRPO)`；核心干预为 GRPO 相对优势的分组边界（Within vs Cross） |
| **关键词** | 信用分配、多 harness、GRPO、编程 Agent、可移植性、SWE-bench |
| **来源渠道** | arxiv-api + listing |
| **PDF 存档** | 2026-09-03_What_Does_Multi-Harness_RL_Learn_Credit_Assignment_and_Portability_in_Coding_Agents.pdf |

---

## 1. 研究背景与要解决的问题

编程 Agent 的强化学习如今普遍跑在完整执行 harness（决定 prompt、工具、观测、上下文管理、重试与控制流的 scaffold）之上，前沿配方还常把多个 harness 或 harness 配置喂给同一个策略。所谓 multi-harness RL 其实混着两个独立选择：一是把策略**暴露**给多个 harness（exposure），二是在同一个相对优势组内**比较**不同 harness 产出的奖励（credit assignment）。业界两种做法并存且互相矛盾：HarnessX 把同一任务上跨 harness/版本的轨迹放进同一个 GRPO 组直接竞争；ClawGym II 则以 task–harness 对为归一化单元，共享策略与 minibatch 但奖励各归各组。由于这些系统在 exposure、数据生成、训练预算与评测上都不一致，端到端结果无法回答"分组边界本身带来什么"。本文在 repo 级 coding 上把 credit assignment 这一个旋钮单独拎出来做受控实验，并引入一个训练中从未见过的评测 harness，用来把"源配置适配"与"可移植能力"分开测量。

## 2. 核心方法 / 思路

设计追求"只动一个变量"。先从同一 Qwen3-8B SFT 起点出发（SFT 语料：4 个 harness 在 SWE-Gym 的 186 任务上成功解决轨迹 8841 条），在 183 个"harness 之间存在分歧"的任务上由 Aider、OpenHands、Qwen Code、SWE-agent 四个生产 harness 各采集轨迹，冻结成同一份 manifest（81216 条记录 / 5543 条 episode，reward、token、loss mask 全部固定）。两支主臂以相同更新预算（81200 步、单 epoch、lr 3×10⁻⁷、KL 0.01）回放同一数据，**唯一差异是 GRPO 优势分组边界**：

- **Within**：每个 task–harness 对一组，优势在组内标准化。因组内共享同一 harness，任何 harness 级偏移被构造性地消掉（代价是 55% 的组陷入全对/全错、优势为 0）。
- **Cross**：每个 task 一组，全部 harness 混池标准化。组均值里混入了各 harness 自身的 solve rate，即"harness 身份"进入优势信号。

另有三个受控变体：Coverage-matched（把 Cross 携带非零优势的轨迹数下采样到与 Within 相等）、Residualized（混池前先减一个 leave-one-task-out 的逐 harness 基线）、以及一个重采集一半训练数据 on-policy 的 Cross 臂。评测：SWE-bench Verified 500 任务 + 单一密封逐实例 oracle，每个 checkpoint 同时在四个源 harness 和训练中从未出现的最小 harness（weak-ReAct 单循环 ReAct）上打分；6 个 held-out 对比在 checkpoint 存在前冻结，逐任务配对 bootstrap，约 6% 的 episode 重跑会翻转结果（连续批处理不确定性），故统一用任务级聚类推断。

## 3. 关键实验结果

**评测 harness 是最大变量。** 24000 条密封评测（99.1% 覆盖）中，harness 列均值从 Aider 的 2.14% 到 OpenHands 的 9.27%（跨度 7.13pp，约 4.3 倍），而训练配方行均值仅从 5.55% 到 6.46%（约 1.16 倍）。harness 强弱不是模型无关的标量：对 Seed-Coder-8B，Aider 与 SWE-agent 相对其自身最小 harness 各折损约 4pp，OpenHands 与 Qwen Code 甚至完全跑不起来。

**分组规则在 held-out 上无差异。** 在未见过的 weak-ReAct 上，k=8 时 Cross−Within = +0.25pp，95%CI [−0.48,+1.02]；相对 SFT，Within −0.03、Cross +0.13，所有区间含零。k=4 单 seed 六个对比的跨度 4.01–4.73% 小于任何对比可检出差（1.19–1.42pp）；预算翻倍后 Cross−Within 从 +0.58 收敛到 +0.25。3 个训练 seed 汇总：+0.16 [−0.41,+0.72]，逐 seed 为 +0.62/+0.05/−0.20——符号在 seed 间翻转，且每臂自身的 seed 波动（0.42–0.45pp）大于臂间差。on-policy 重采集一半数据也不改变结论（re-collected 相对 Cross +0.13 [−0.85,+1.10]）。

**两臂的增益都落在源 harness 上，且集中于同一 harness。** 在四个训练 harness 上，Cross 相对 SFT +0.77pp（95%CI [+0.03,+1.52]，Bonferroni 99% [−0.21,+1.75]），Within +0.48pp；二者最大单列增益都在 OpenHands（Cross +1.81 vs Within +1.71pp）。作者称之为"configuration adaptation without portable capability"——只换评测 harness 就能把这种增益与真正的内化能力区分开。

**Cross 的优势值确实携带 harness 身份，但无下游后果。** 从优势值反推产生它的 harness（OOF 分类器，chance 25%）：Within 高出自身置换空模型仅 +0.02pp（构造性无），Cross +4.48pp [+3.22,+5.83]（斜率 2.38）；Coverage-matched 保留约四分之一（+1.17）；Residualized 因基线在全体 1008 个任务上估计、只消掉 9% 偏移，几乎全保留（+4.54）。然而这没有反映到任何行为上：held-out 分数相同；同一 harness 内 SFT 与 Cross 的动作分布在各 harness 中 JSD 仅 0.0003–0.0029 bit，而把同一 checkpoint 换一个 harness 会改变 0.17–0.35 bit（相差至少 59 倍）——动作库由 harness 设定，credit 规则改不了它。

## 4. 亮点与贡献（Why it matters）

- 把"多 harness RL"这个模糊配方拆成 exposure 与 credit assignment 两个旋钮，并给出第一个在 repo 级 coding 上隔离分组边界的同数据、同预算、同 oracle 对照实验。
- 用"训练中未见过的 harness"作为可移植性探针，正面演示了源 harness 增益如何伪装成模型能力（Polar、OpenForgeRL 等报告的 gains 多为 harness 索引化的，本文附录逐条点评）。
- 方法学严谨：密封 oracle、故障隔离而非计零、6 个预注册对比、逐任务聚类、seed 与预算灵敏度齐备——是"报告 harness 为一等变量"的示范。
- 结论直指社区：评价 harness 移动分数的能力（×4.3）远超训练配方（×1.16），agent 评测不公开 harness 就没有可比性。
- 对 RL 实践者的警示：Cross 的组均值污染（harness 身份进入 advantage）真实可测，但删除它（Residualized/Coverage-matched）也不带来 held-out 增益，说明问题不在这个层面。

## 5. 局限与可改进点（个人点评）

- 覆盖窄：单模型族（Qwen3-8B，Seed-Coder-8B 只做未训练的 harness 对比）、单预算、单域（Python SWE）、二元终端奖励、绝对 solve rate 仅 1–10%。分辨力约 1pp，因此正确表述是"无证据有差异"而非"等价"——作者自己也承认 null 只是上界。
- 主结论大多基于单 checkpoint（每臂一个 RL seed），只有 Within/Cross 补了额外 seed；Residualized、Coverage-matched 无 seed 波动估计。
- held-out harness 选的是最弱的 minimal ReAct，其分数与 untrained base 几乎齐平，全部配方都不超 base——留出接口的难度选择可能低估了"可移植能力"存在的可能（换更强的留出 harness 或可测出差异）。
- SFT 臂按 RL 的 lr 与算力训练，是"匹配超参的消融"而非调优过的蒸馏配方，不宜推出"RL 无用"的强结论。
- 未见 harness 评测需额外 rollout+grading 算力，规模化采用有成本门槛；后续若加入 per-step value、分层/hindsight credit 或 process supervision，可能触及标量偏移碰不到的信道。

## 6. 对 GUI Agent 的可借鉴点（跨域参考）

GUI Agent 训练正处在同样的处境：大家常在多个环境/scaffold（WebArena、OSWorld、不同 observation encoder、不同动作空间的 harness）上采样做 RL，却几乎不报告"奖励归一化是否跨环境"。本文的两个动作可以直接平移：

其一，报告惯例。任何 multi-environment GUI RL 论文都应写清分组边界——是每 (任务,环境) 一组（Within 式）还是同任务多环境混池（Cross 式）——并补一个训练时未见过的环境/harness 的评测，否则源环境增益会冒充泛化能力。GUI 的 harness 效应很可能比 coding 更极端（obs 格式、动作空间、屏幕分辨率都是变量），测评环境的选择本身就可能移动分数数倍。

其二，credit assignment 建议。若用多环境数据训练 GUI policy，首选每环境分组或先减 leave-one-task-out 的逐环境基线再混池（对应本文 Resid），防止强环境抬高 baseline、把"环境帮的忙"记到模型头上；跨环境混池时需警惕 55% 全对/全错退化的任务被 pool 救活。行为层面本文的 JSD 证据提醒：想改变 GUI 策略的"操作风格"（点 vs 键盘、截图频率等），应该改 harness/观测接口，而不是换 credit 规则。

对 GUI 评测自身的启示：既然 harness 列均值可差 4.3 倍，我们库内 GUI benchmark 解读应把 scaffold 版本、grader 与 rollout 次数列为必填项（正如此文 Table 6 逐一 pin 版本的做法）。

## 7. 延伸阅读

- 多 harness 训练：HarnessX（跨版本混组）、ClawGym II（逐 harness 归一）、OpenForgeRL、Polar、Tmax；SWE-RL / SWE-Gym / SWE-bench（数据与评测根基）。
- GRPO 分组与归一化：DAPO、DeepSeekMath；单 harness 分组偏差修正见 Liu et al. 2025b、Yu et al. 2025。
- harness 作为评测变量：Harness-Bench、Kapoor et al.（AI agents that matter / Holistic Agent Leaderboard）。
- GUI/环境 RL 相关（本库可顺藤摸瓜）：WebRL、DigiRL、RAGEN。
- 本库关联解读：2026-09-03《Spurious Advantage Hidden in GRPO》、2026-09-03《Headroom-Drift Replay》（GRPO 重放控制）、2026-09-02《Tail-Likelihood Reinforcement Learning》。

---
*解读生成时间：2026-09-08 ｜ 解读人：WorkBuddy（AI）*
