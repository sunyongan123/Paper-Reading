# SkillAdam: Stable and Efficient Skill Evolution for Agents

> 把 Adam 优化器的一阶/二阶动量思想搬到离散技能文档上：用"问题记忆"稳住修订方向、用"波动率驱动的编辑预算"自适应控制改多少，比启发式自演化更稳、更省、更出活。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | SkillAdam: Stable and Efficient Skill Evolution for Agents |
| **arXiv ID / DOI** | 2609.08944v1 [cs.AI] |
| **arXiv 链接** | https://arxiv.org/abs/2609.08944（点击直达） |
| **发表出处（Venue）** | arXiv preprint（17 页） |
| **发布时间** | 2026-09-08 |
| **作者** | Gaoyuan Li；Meihao Fan；Yizhe Liu；Shaolei Zhang（通讯作者）；Ju Fan；Siyi Wang；Jiaheng Hou；Xudong Weng；Honghan Tian；Zang Li（共 10 人） |
| **所属机构** | 中国人民大学 与 腾讯（中国） |
| **开源情况 / 代码** | ✅ 有代码：https://github.com/ruc-datalab/SkillAdam（点击直达） |
| **类型标签（论文类别）** | `General` `Online` |
| **训练方法标签** | `—`（无模型参数训练：目标 LLM 全程冻结，Adam 式外部技能文档迭代自演化，属离散文本空间优化而非 SFT/RL） |
| **关键词** | Agent Skills；Skill Self-Evolution；Adam-inspired Optimization；Optimization Memory；Volatility-Driven Edit Budget |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_SkillAdam_Stable_and_Efficient_Skill_Evolution_for_Agents.pdf |

---

## 1. 研究背景与要解决的问题

按 Anthropic 的定义，Agent Skill 是可分发的"指令 + 支撑资源"模块，让冻结参数的 LLM agent 不重训也能获得领域专长。问题出在**高质量技能既关键又难得**：SkillsBench 表明精心策展的技能能大幅提升 agent 表现；SkillAxe 又报告 LLM 直接写出的技能不精修往往无效。专家手写耗时费力，自动生成又依赖人设 prompt 与任务特化启发式，难以跨域泛化。

近几年兴起的"技能自演化"（skill self-evolution）用执行反馈迭代修订，自动化程度高，却有两个致命毛病：**方向不稳定**——每轮只见少量 case 的局部反馈，前后修订互相覆盖、互相撤销（一轮让 agent 压预算，下轮又加内容把它破坏）；**更新不自适应**——该大改还是小修，本应看近期改动对 case 影响的一致性，现有方法却无此量纲控制。

作者的洞察很漂亮：这跟随机优化里的困境同构。Adam 优化器正是靠两个互补原则解决同类问题——用一阶动量（梯度指数滑动平均）稳定更新方向，用二阶动量（梯度平方的滑动平均）重缩放有效步长。SkillAdam 的立意就是把这两条原则**在离散、不可微的技能文档空间里做功能化类比**，而不是数值套用。

## 2. 核心方法 / 思路

SkillAdam 视一份技能为 Markdown 自然语言指令文档，在"采样 mini-batch → 执行 → 反馈 → 修订"的循环中优化它，全程保持目标 LLM 冻结。循环里有三块：

1. **Trajectory-informed 初始化**：不从一个随机技能起步，而是先收集一批执行轨迹与评测反馈，让 LLM 归纳出一份初版技能 S₀——相当于从数据里拿到一个好的"起始参数"。
2. **Evolving Issue Tracker（演化式问题跟踪器）≈ Adam 的一阶动量 m**：维护一组结构化条目 Iⱼ=(pⱼ, zⱼ, Aⱼ)，即"错误模式 + 当前状态 + 历史解决尝试及结果"。每轮修订评测后，用 LLM 把本轮失败**归并到既有问题**（能对得上就不新建）、记录每次尝试与结局、同一失败复发就重开问题。它的作用正如一阶矩：把当前信号与跨轮累积的证据整合，**让有效修正被沉淀而非被覆盖**——决定了"该修什么"。
3. **Volatility-driven Edit Budget（波动率驱动的编辑预算）≈ Adam 的二阶动量 v / 自适应步长**：对 mini-batch 里每个 case 算改进量 δ，统计其方差（跨 case 的一致性），再做指数滑动平均 V_t=β₂V_{t-1}+(1−β₂)V̂_t；下一轮编辑预算 σ_{t+1}=max(b_min, b_base·(1−clip(V_t/V_max)))——**波动高=效果忽好忽坏=只许小修，波动低=收益一致=允许大改**——决定了"能改多少"。

补丁生成器（≈梯度）在上述约束下产出候选修订，并在**同一 mini-batch** 上与当前技能对照评测，由多指标接受门控（主指标达标 + 受保护指标不越界回退）决定是否收下。作者在表1给出 Adam↔SkillAdam 的逐项功能对应，并强调这是**功能类比而非数值模拟**——技能空间没有真梯度，但优化角色一致。

## 3. 关键实验结果

七个 benchmark、十个评测切片：短程五个（SearchQA、SpreadsheetBench、OfficeQA、DocVQA、LiveMathematicianBench，平均 <10 次工具调用）加长程两个（ALFWorld、DeepPlanning 的 Shopping L1–L3 + Travel EN）。目标模型 GPT-5.5（DeepPlanning 用 Claude Sonnet 4.5，温度 0），全部冻结。对照七个基线：NoSkill / HumanSkill / LLMSkill / Trace2Skill / TextGrad / GEPA / SkillOpt，其中 SkillAdam 与 SkillOpt 共享同一份轨迹初始化技能。核心数字：

- **短程任务**：SearchQA 87.5 vs SkillOpt 87.3；Spreadsheet 81.1 vs 80.7；OfficeQA 72.1（平）；DocVQA 92.3 vs 91.2；LiveMath 67.7 vs 66.9——四胜一平。相对 HumanSkill 平均 +14.45%，相对 LLMSkill 平均 +31.20%。
- **长程任务**（最出彩）：ALFWorld 89.6 vs SkillOpt 87.3；DP-Travel 上 SkillOpt 只有 1.7、NoSkill 是 0.0，**SkillAdam 干到 11.7**；DP-Avg 从 21.7 提到 28.3（+6.7pp）。
- **消融**（DeepPlanning，累积式）：全配置 DP-Avg 28.3 → 去掉预算 21.7 → 再去掉记忆 19.2（NoSkill 15.8）。预算机制贡献 +6.7pp（最大增益在 DP-Travel），记忆贡献约 +2.5pp。
- **跨模型迁移**（GPT-5.5 优化、原样部署到 GPT-5.4-mini）：平均目标分数 67.8 vs SkillOpt 63.1；平均保留率 81.3% vs 76.2%（ALFWorld 保留率 92.4% vs 76.1%）。
- **技能质量评估**（GPT-5.5 打分 1–5，六维中位数平均）：SkillAdam 3.88 vs SkillOpt 3.30，十个切片全部占优。
- **成本（DeepPlanning 四切片）**：总 token 226.6M → 74.0M（**−67.3%**），API 请求 9071 → 2830（**−68.8%**），同时 DP-Avg 反而从 21.7 升到 28.3，按每百万 token 产出计约 **4 倍效率**。
- **稳定性**（Shopping L1 单次运行）：SkillAdam 首轮即找到强修订且此后被接受的技能都高于起点；达到 56% 测试精度约耗 4M token，而 SkillOpt 花约 20M token 产出的候选还不被接受（约 1/5 成本）。

## 4. 亮点与贡献（Why it matters）

1. **"把 Adam 搬到离散技能空间"的建模干净且有解释力**：EMA、一阶/二阶矩、自适应步长逐项对应到问题跟踪器与编辑预算，让"每轮改什么、改多大"第一次有了可解释的优化论据。
2. **跨迭代持久状态的稀缺性**：多数自演化方法每轮"从零看反馈"，SkillAdam 的记忆让有效修正跨轮累积，直击"前后修订互相踩踏"这一行业痛点。
3. **实打实的降本增效**：−67% token、−69% API 请求还换来更高性能，对依赖商用 LLM API 的技能优化管线是显著的经济账。
4. **技能可迁移性被量化**：跨模型保留率 81.3%、质量分十切片全胜，说明优化产物是稳定、可复用、模型无关的"程序性资产"。
5. **长程规划任务上的突破信号**：DP-Travel 从近乎失败（1.7）到 11.7，说明受控的迭代能啃下多约束长程规划这种 hard case。

## 5. 局限与可改进点（个人点评）

- **评测生态偏"自产自销"**：DeepPlanning、SkillsBench、DeepPrep 等多出自人大团队；短程基线的 SkillOpt 数字直接引自原论文（GPT-5.5 no-harness 设定），仅 ALFWorld/DeepPlanning 为本地复现，需第三方中立背书。
- **自评偏置风险**：技能由 GPT-5.5 优化、亦由 GPT-5.5 打分，模型同源或高估质量分，宜加异构 judge 交叉验证。
- **消融为累积式**，未独立隔离记忆与预算的交互；图4 稳定性证据仅来自单次 run，泛化性存疑。
- **技能仍只是单文档**：未覆盖多文件 package、代码化技能、多 agent 共享技能库等形态；且文档经多轮 LLM 补丁拼接后可能藏有隐藏的前后矛盾。
- **接受门控依赖各 benchmark 自定义指标与阈值**，跨域部署需重设 evaluator，这笔 setup 成本不在论文统计内。
- **相对降幅大、绝对成本仍高**：−67% 后的 74M token 并非小数，每轮补丁生成与对照评测都要 LLM 调用；且 DP-Avg 28.3、DP-Travel 11.7 的绝对值仍很低——方法显著，任务远未解决。

## 6. 对我们的启示 / 可借鉴点

对本项目主要有四点启示：① **把"外部技能"当作可优化参数**——训练信号就是 agent 回放轨迹 + 评测反馈，不碰模型权重也能迭代变强，契合"冻结骨干、快速适配领域"的工程约束；② **引入跨轮记忆与门控**可显著提高迭代稳定性，避免"周一修好、周二改坏"的文档回退；③ **用"改进一致性（波动率）"决定改动幅度**：改动在部分 case 上引发回归时，先收窄范围而不是推倒重写；④ **把跨模型保留率纳入验收指标**——换 backbone 后还能保住多少效果，比单模型测试分数更能反映技能真质量。

**对 GUI Agent 的可借鉴点：** GUI 操作技能天然是"步骤 + 元素定位 + 界面断言"的文本指令，恰好落入 SkillAdam 的单文档迭代框架：把每条 GUI 技能当待优化"参数"，截图操作日志当 trajectory、UI 验收结果当反馈。三个机制可直接移植：**其一**，问题跟踪器专门记录 GUI 高频病根（"选择器对动态 DOM 失效""支付弹窗拦截""等待时序竞态""无障碍标签缺失"），同类问题归并去重并留存每次修复方案的结局，防止今天为 Web 端修的选择器明天被移动端改动推翻；**其二**，波动率预算对 GUI 尤其关键——一次改动若部分用例通过、部分反而挂掉，下一轮就应收敛成小修而不是继续大改；**其三**，把"技能跨模型保留率"当硬指标——GUI agent 的视觉/操作骨干常需切换（不同 VLM 元素定位能力差异大），SkillAdam 式优化产出的技能更"模型无关"、换骨干少返工。工程上可让每个环境（Web/Mobile/Desktop）各维护技能库的"波动率状态"，隐式建模哪类环境技能改动风险高，自演化时优先小步、稳迭代。

## 7. 延伸阅读

- Adam 原论文：Kingma & Ba, "Adam: A Method for Stochastic Optimization"（本文方法论的灵感来源）。
- 技能自演化直接相关：SkillAxe（评测驱动打磨）、SkillOpt（受限编辑 + 验证接受，本文最强基线）、EvoSkill、CoEvoSkills、SkillOS、AutoManual、Trace2Skill。
- Prompt/文本优化谱系：OPRO（LLM as Optimizer）、TextGrad（文本"反向传播"）、ProTeGi（文本梯度）、GEPA（反思式演化）、ERM——SkillAdam 把同一谱系从"prompt"推进到"可复用技能文档"。
- Agent 技能综述与评测：SkillsBench、SoK: Agentic Skills、Agent Workflow Memory、ExpeL。

---
*解读生成时间：2026-09-09 ｜ 解读人：WorkBuddy（AI）*
