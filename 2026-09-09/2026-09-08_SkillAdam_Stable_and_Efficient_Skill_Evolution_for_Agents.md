# SkillAdam: Stable and Efficient Skill Evolution for Agents

> 一句话 TL;DR：把 Adam 优化器的一阶/二阶动量思想**功能化类比**搬到离散技能文档空间——用"问题记忆"稳住修订方向、用"波动率驱动的编辑预算"控制改多少，比启发式自演化更稳、更省、更出活。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | SkillAdam: Stable and Efficient Skill Evolution for Agents |
| **作者 / 机构** | Gaoyuan Li、Meihao Fan、Yizhe Liu、Shaolei Zhang（通讯）、Ju Fan、Siyi Wang、Jiaheng Hou、Xudong Weng、Honghan Tian、Zang Li（共 10 人）；中国人民大学 + 腾讯（中国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-08；arXiv preprint（17 页，cs.AI） |
| **arXiv 链接** | https://arxiv.org/abs/2609.08944 |
| **代码仓库** | ✅ GitHub：https://github.com/ruc-datalab/SkillAdam |
| **数据集地址** | ❌ 无独立数据集（所用 7 个 benchmark 均为公开第三方基准，见第 5 节） |
| **类型标签（论文类别）** | `General` `Online` |
| **训练方法标签** | `—`（无模型参数训练：目标 LLM 全程冻结，属离散文本空间的迭代自演化，非 SFT/RL/蒸馏） |
| **关键词** | Agent Skills；Skill Self-Evolution；Adam-inspired Optimization；Optimization Memory；Volatility-Driven Edit Budget |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_SkillAdam_Stable_and_Efficient_Skill_Evolution_for_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：Agent Skill（按 Anthropic 定义，指"指令 + 支撑资源"的可分发模块，能让**冻结参数**的 LLM agent 获得领域专长）的**自动化高质量获取**。本文关注技能主体为一份 Markdown 自然语言指令文档、在任务评测下的**迭代优化**，属通用 Agent 域（知识工作 + 交互式规划），非 GUI 专属。
- **为什么重要**：高质量技能"既关键又难得"。SkillsBench 表明精心策展的技能能大幅提升 agent 表现；SkillAxe 却报告 LLM 直接写出的技能不精修往往无效。专家手写耗时费力；自动生成又依赖人设 prompt 与任务特化启发式，难跨域泛化。卡住的是"冻结骨干、低成本适配领域"这条工程路径。
- **现有方法有什么不足**：技能自演化（Trace2Skill/SkillAxe/EvoSkill/CoEvoSkills/SkillOS/SkillOpt 等）虽用执行反馈迭代修订，但有两个结构性缺陷——**方向不稳定**：每轮只观察少量 case 的局部反馈，前后修订互相覆盖、互相撤销（论文原例：一轮让 agent 选最便宜可行路线压预算，下一轮又加景点破坏该预算约束）；**更新不自适应**：该大改还是小修本应取决于近期改动对 case 影响的一致性，现有方法却无此量纲控制，等价于"SGD 式"优化——每轮决定"改什么、改多少"都靠预设规则，缺乏可靠的跨轮状态。
- **Research Gap**：作者 claim 补上"方向稳定性 + 更新自适应性"两个缺口——在离散、不可微的技能空间里，把 Adam 的一阶矩（聚合历史信号稳定方向）与二阶矩（用信号幅值重缩放有效步长）两条原则做**功能类比**，首次为技能演化引入**持久的优化器状态**（记忆 + 波动率）。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把技能文档当作"待优化参数"，用 Adam 式的一阶动量（跨轮问题记忆）定方向、二阶动量（改进波动率→编辑预算）定步长，在采样-执行-反馈-修订循环中演化技能，全程不碰模型权重。

### 3.2 方法总览（Pipeline）

- **输入**：任务数据集 D、领域 evaluator E（返回结构化 case 级反馈 `E(S,d)` 的 m 维指标向量 + 诊断信息 `C(S,d)`）；**输出**：优化后的技能文档 `S_Tmax`。
- **循环（每轮 t）**：① **Rollout**——采样 mini-batch `B_t`，用当前技能 `S_{t-1}` 执行得轨迹 `T_t` 与反馈 `F_t^roll`；② **Moment Estimation**——维护两个跨轮持久状态：`Evolving Issue Tracker M_t`（≈一阶矩）与历史波动率 `V_t`（≈二阶矩）；③ **Skill Update**——补丁生成器 `g_t = G_LLM(S_{t-1}, T_t, F_t^roll, M_{t-1}, σ_t)` 产出候选 `S̃_t`，在同一 `B_t` 上对照评测得 `F_t^val`，由多指标接受门控 `a_t = G_θ` 决定是否收下。
- **连接**：本轮 `M_t`、`V_t` 由本轮评测结果刷新后，指导**下一轮**的"改什么"（问题集）与"改多少"（预算 `σ_{t+1}`）。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：① **Evolving Issue Tracker**——结构化条目 `I_j=(p_j, z_j, A_j)`（错误模式+状态+历史解决尝试及结局），把失败归并去重、记录每次尝试、失败复发即重开问题，让有效修正跨轮累积而非被覆盖；② **Volatility-driven Edit Budget**——对 case 级改进量 `δ_{t,i}` 取方差 `V̂_t`，做 EMA `V_t=β₂V_{t-1}+(1−β₂)V̂_t`，下一轮预算 `σ_{t+1}=max(b_min, b_base·(1−clip(V_t/V_max,0,1)))`，波动高→只许小修、波动低→允许大改。两者分别对应一阶/二阶矩，是首次在技能空间落地的持久优化状态。
- **工程组合**：trajectory-informed 初始化（从执行轨迹归纳初始技能）、补丁生成器、多指标接受门控——均为已有组件的组合，本身不新但有效。
- **对性能提升最关键的设计**：**波动率预算**。DeepPlanning 上固定记忆时加预算使 DP-Avg +6.7pp（最大增益在 DP-Travel）；记忆贡献约 +2.5pp。长程规划场景尤其受益。
- **证据不足 / 仅声称有效**：消融为**累积式**，未独立隔离记忆与预算的交互；稳定性/效率证据来自单次 run（图4）；跨模型迁移增益未做组件归因（论文自承）。

---

## 4. 具体技术细节

### 4.1 模型结构

纯文本 skill 文档 + 冻结目标 LLM。**无视觉编码器、无训练**。6 个 benchmark（SearchQA/Spreadsheet/OfficeQA/DocVQA/LiveMath/ALFWorld）用 **GPT-5.5**（medium reasoning、温度 1.0、最大输出 16384 token）；**DeepPlanning** 用 **Claude Sonnet 4.5**（温度 0.0）。全程 no-harness 直连聊天、技能以自然语言指令注入，**目标模型冻结**，优化只发生在技能文档上。补丁生成、问题归并、接受判断均由 LLM 完成。

### 4.2 训练流程（方法机制重点）

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| 初始化 | 拿到好"起始参数" `S_0` | 归纳领域初步技能 | 固定基线执行轨迹 + 评测反馈 | 轨迹文本 + 结构化反馈 | 无显式 loss，LLM 归纳 |
| 迭代演化（核心） | 稳定且自适应地修订技能 | 方向稳定（记忆）+ 步长自适应（预算） | mini-batch `B_t` 的 rollout + 候选对照评测 | case 级指标向量 `E` + 诊断信息 `C` | 无梯度 loss；"梯度"由补丁 `g_t` 扮演，"loss"由反馈 `F_t^roll` 扮演 |

关键机制（Adam↔SkillAdam 功能对应，见表 1）：
- **一阶矩** `m^Adam ↔ M_t`：`U_EIT` 对比 rollout 与验证结局，把本轮失败**归并到既有 issue**（能对上不新建）、记录每次尝试及结局、同一失败复发就重开——整合当前信号与历史证据，稳定方向。
- **二阶矩** `v^Adam ↔ V_t`：case 级改进 `δ_{t,i}=s(E^{val}_{t,i})−s(E^{roll}_{t,i})`，方差 `V̂_t` 经 `V_t=β₂V_{t-1}+(1−β₂)V̂_t` 平滑，映射为下一轮编辑预算 `σ_{t+1}`。
- **有效步长** `α^eff ↔ σ_{t+1}`：预算封顶封底（`b_base`/`b_min`），直接限制补丁可改动范围。
- **接受门控**：候选在**同一 mini-batch** 上对照评测，主指标达阈值 + 所有 protected 指标不越界回退才接受；`S_t = S̃_t` 或 `S_{t-1}`。作者强调这是**功能类比而非数值模拟**——技能空间没有真梯度。

### 4.3 推理流程

推理阶段无额外训练、无多步反思循环：把演化得到的技能文档**原样注入**目标 agent，agent 按该指令在原生任务环境上单轮执行。优化期每轮 = 一次 mini-batch 级"修改尝试"，终止条件为 benchmark 特定的停止规则（六基准单遍扫描优化池；DeepPlanning 随机采样 + 任务特定停止）。跨模型部署时技能 verbatim 迁移，不再优化。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| SearchQA | General | 开放域 QA（噪声检索上下文） | 未披露 | 文本（问题+检索上下文） | 答案 | Exact Match |
| SpreadsheetBench | General | 电子表格编辑 | 未披露 | 表格+自然语言指令 | 编辑后表格 | 硬成功率 |
| OfficeQA | General | 企业文档 QA（美财政部公报） | 未披露 | 文本 | 答案 | Exact Match |
| DocVQA | General | 文档图像 VQA | 未披露 | 文档图像+问题 | 答案 | ANLS-hard |
| LiveMathematicianBench | General | 数学定理/证明推理 | 未披露 | 文本（定理+证明草图） | 答案 | Exact Match |
| ALFWorld | General | 文本具身家务（长程） | 未披露 | 文本（具身观察） | 动作序列 | 目标完成率 |
| DeepPlanning（Shopping L1–L3 / Travel EN） | General | 多步购物/旅行规划（长程，可验证约束） | Shopping 各 25/25/10、Travel 60（test case） | 文本（约束） | 结构化规划 | Case Accuracy |

> 注：除 DeepPlanning 外，其余 benchmark 论文未披露评测切片规模。短程 5 个平均 <10 次工具调用，ALFWorld/DeepPlanning 平均 ≥10 次。

### 5.2 实验结果分析

**主结果（短程，GPT-5.5）**：SkillAdam 五基准四胜一平。SearchQA 87.5（SkillOpt 87.3）、Spreadsheet 81.1（80.7）、OfficeQA 72.1（平）、DocVQA 92.3（91.2，+1.21%）、LiveMath 67.7（66.9，+1.20%）。相对 HumanSkill 平均 +14.45%、相对 LLMSkill 平均 +31.20%。

**主结果（长程）**：ALFWorld 89.6（SkillOpt 87.3）；DP-Travel 上 NoSkill 0.0、SkillOpt 1.7，**SkillAdam 干到 11.7**；DP-Avg 21.7 → 28.3（**+6.7pp**）。

**Ablation（DeepPlanning 累积式）**：全配置 DP-Avg 28.3 → 去预算 21.7 → 再去记忆 19.2（NoSkill 15.8）。预算贡献 +6.7pp（最大在 Travel），记忆贡献约 +2.5pp（增益集中在更难 L2/L3，L1 反降）。因是累积式，无法独立分离两组件交互。

**跨模型迁移（GPT-5.5→GPT-5.4-mini）**：平均目标分 67.8 vs 63.1，平均保留率 81.3% vs 76.2%（ALFWorld 保留率 92.4% vs 76.1%）。**技能质量**（GPT-5.5 judge 1–5，六维中位数均值）3.88 vs 3.30，十切片全胜。**成本（DeepPlanning 四切片）**：总 token 226.6M→74.0M（−67.3%）、API 请求 9071→2830（−68.8%），DP-Avg 反升；按每百万 token 产出约 4 倍效率。

---

## 6. 亮点与贡献（Why it matters）

1. **"Adam 离散化"的建模干净且可解释**：EMA、一阶/二阶矩、自适应步长逐项映射到问题跟踪器与编辑预算，"每轮改什么、改多大"第一次有了优化论据而非黑箱启发式。
2. **跨迭代持久状态稀缺且直击痛点**：多数自演化每轮"从零看反馈"，SkillAdam 的记忆让有效修正跨轮沉淀，解决"前后修订互相踩踏"这一行业通病。
3. **实打实降本增效**：−67% token、−69% 请求还换来更高性能，对依赖商用 LLM API 的技能优化管线是显著经济账。
4. **技能可迁移性被量化**：保留率 81.3%、质量分十切片全胜，说明产物是稳定、可复用、模型无关的程序性资产。
5. **长程规划突破信号**：DP-Travel 从 1.7→11.7，受控迭代能啃下多约束长程规划 hard case。

## 7. 局限与可改进点（个人点评）

- **评测生态偏"自产自销"**：DeepPlanning/SkillsBench 等多出自人大团队，短程 SkillOpt 数字直接引自原论文（仅 ALFWorld/DeepPlanning 本地复现），缺第三方中立背书。
- **自评偏置**：技能由 GPT-5.5 优化、亦由 GPT-5.5 打分，同源 judge 或高估质量分，宜加异构 judge 交叉验证。
- **消融为累积式**，未独立隔离记忆与预算交互；图 4 稳定性仅单次 run，泛化性存疑。
- **技能仍只是单文档**：未覆盖多文件 package、代码化技能、多 agent 技能库；多轮 LLM 补丁拼接后可能藏前后矛盾。
- **接受门控依赖各 benchmark 自定义指标与阈值**，跨域部署需重设 evaluator，setup 成本未计入统计。
- **绝对成本仍高、绝对性能仍低**：−67% 后 74M token 非小数；DP-Avg 28.3、DP-Travel 11.7 绝对值很低——方法显著，任务远未解决。

## 8. 对我们的启示 / 可借鉴点

① **把"外部技能"当可优化参数**——训练信号就是 agent 回放轨迹 + 评测反馈，不碰模型权重也能迭代变强，契合"冻结骨干、快速适配领域"的工程约束。② **跨轮记忆 + 门控**可显著提高迭代稳定性，避免"周一修好、周二改坏"的文档回退。③ **用改进一致性（波动率）决定改动幅度**：改动在部分 case 上引发回归时，先收窄范围而不是推倒重写。④ **把跨模型保留率纳入验收**——换 backbone 还能保住多少效果，比单模型分数更反映技能真质量。

**对 GUI Agent 的可借鉴点**：GUI 操作技能天然是"步骤 + 元素定位 + 界面断言"的文本指令，正好落入单文档迭代框架：把每条 GUI 技能当待优化"参数"，截图操作日志当 trajectory、UI 验收结果当反馈。三个机制可直接移植：**其一**，问题跟踪器专记 GUI 高频病根（"选择器对动态 DOM 失效""弹窗拦截""等待时序竞态""无障碍标签缺失"），同类归并去重、留存每次修复结局，防止今天为 Web 修的选择器明天被移动端改动推翻；**其二**，波动率预算对 GUI 尤关键——一次改动若部分用例过、部分反而挂，下一轮就收敛成小修而非继续大改；**其三**，把"技能跨模型保留率"当硬指标——GUI 骨干（VLM 元素定位能力差异大）常需切换，SkillAdam 式优化产物更"模型无关"、换骨干少返工。工程上可让每个环境（Web/Mobile/Desktop）各维护技能库的"波动率状态"，隐式建模哪类环境改动风险高，自演化时优先小步稳迭代。

## 9. 延伸阅读

- Adam 原论文：Kingma & Ba, "Adam: A Method for Stochastic Optimization"（本文灵感来源）。
- 技能自演化：SkillOpt（受限编辑+验证接受，本文最强基线）、SkillAxe、EvoSkill、CoEvoSkills、SkillOS、AutoManual、Trace2Skill。
- 文本/提示优化谱系：OPRO（LLM as Optimizer）、TextGrad（文本反向传播）、ProTeGi、GEPA、ERM——SkillAdam 把该谱系从"prompt"推进到"可复用技能文档"。
- 技能评测与综述：SkillsBench、SoK: Agentic Skills、Agent Workflow Memory、ExpeL。

---
*解读生成时间：2026-09-09 ｜ 解读人：WorkBuddy（AI）*
