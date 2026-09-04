# Monitoring Web Agents Without Internal Signals: Observable Trajectories and Key-Step Supervision

> TL;DR：不用模型内部信号，仅凭可观测轨迹即可在线预测 Web agent 失败风险。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | Monitoring Web Agents Without Internal Signals: Observable Trajectories and Key-Step Supervision |
| **arXiv ID / DOI** | arXiv:2609.02057v1（cs.AI） |
| **arXiv 链接** | <https://arxiv.org/abs/2609.02057> |
| **发表出处（Venue）** | arXiv preprint（预印本，comments 字段为 "preprint"，未标注会议/期刊） |
| **发布时间** | 2026-09-02 |
| **作者** | Sitong Pan、Yipeng Shen、Yilin Lu、Caiwen Ding、Lu Cheng、Qianwen Wang（按署名序） |
| **所属机构** | 明尼苏达大学（美国）：S. Pan、Y. Lu、C. Ding、Q. Wang；普渡大学（美国）：Y. Shen；宾夕法尼亚州立大学（美国）：L. Cheng |
| **开源情况** | ❌ 未在文中声明开源（正文未检索到 GitHub/代码/数据链接声明） |
| **类型标签** | `Web` `Online` `Benchmark` |
| **训练方法标签** | —（监控/评测方法研究：黑盒 Web agent 失败风险预测，非模型训练论文） |
| **关键词** | Prefix-level failure prediction; Web agent monitoring; Black-box uncertainty; Observable trajectory signals; Key-step supervision |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-02_Monitoring_Web_Agents_Without_Internal_Signals_Observable_Trajectories_and_Key-Step_Supervision.pdf |

---

## 1. 研究背景与要解决的问题

网页 agent 以"看页面→推理→执行动作"的多步循环完成任务，难免中途出错。一旦走上注定失败的路，继续推理与操作只会白白消耗时间和 token；等到任务终结才察觉失败，也错过了挽回时机。因此需要**在任务完成前，仅凭已走过的轨迹前缀估计本次执行是仍在正轨上、还是正在滑向失败**。

本文指出两大难点。其一是**输入可观测性**：大量风险/不确定性方法依赖 token logits、hidden states 等模型内部信号，闭源 agent（如 Claude）并不暴露；而黑盒手段（口头置信度、采样一致性）通常只评估孤立单步输出。现实中监控方只能看到 agent 输出的文本、执行的动作与环境反馈——这些"可观测轨迹信号"够不够用，尚无答案。其二是**前缀监督**：任务只给一个稀疏的最终标签，把它简单传播到所有前缀在时间上不准确——失败轨迹的开头几步常常有效，这些早期行为也可能出现在成功执行中，一刀切打上"失败"会产生矛盾的监督。

## 2. 核心方法 / 思路

本文把监控做成"**特征提取 + 轻量风险预测器**"，全程不触碰模型内部：

**（1）两类可观测轨迹信号。** Macro 特征（宏观，31 维）直接"读已发生的日志"：统计前缀里跨步的动作重复与循环、动作使用次数、执行/grounding 错误、思考文本中自报的"动作无效/环境出错"等，捕捉单步看不出、需累积才显现的失败症状。Micro 特征（微观，18 维）则"反复探测 agent"：每一步让 agent 按结构化提示输出"当前意图 / 可执行动作 / 预期状态变化"，采样 N=10 次后用 Qwen-0.6B 编码器做语义聚类，从经验分布中估计 6 个代数上不冗余的不确定性指标（意图熵、grounding 熵、条件状态熵等），再以均值/最大/当前值聚合成前缀信号。类比：Macro 像看行为录像判断"这人是否在原地打转、反复撞墙"，Micro 像反复追问"你到底要去哪、怎么做"，看他答得是否一致。

**（2）Key-step 感知的前缀标注。** 对每条失败轨迹标注"关键失败步"k\*：**后续执行中第一个未被纠正、且与最终失败相关联的关键错误**（沿用 AgentRx 的定义）。k\* 之前的有效前缀与成功轨迹标为 on track，k\* 起才标为 tending toward failure，避免误伤失败轨迹开头的好行为。标注用 LLM-as-judge（Gemini-3.5-Flash）+ 人工审阅提炼的 codebook 两阶段完成，并用 150 条轨迹做人机对比与三次独立运行验证：80.0% 关键步落在 ±1 步内，前缀标签人机一致率达 89.8%，运行间两两一致 90.9%，单次运行与三次多数一致 95.5%。

**（3）预测器。** 将信号 z 配标签 y，用 ℓ2 正则 logistic 回归拟合前缀失败风险，任务级 5 折交叉验证。与最接近的 PrefixGuard、HTC-Full 相比，关键差异是只用可观测信号、且监督"时间对齐"（失败只从关键步起传播）而非整条失败轨迹一刀切。

## 3. 关键实验结果

评测基准为 WebArena-Lite（165 个可复现 Docker 任务）与 Online Mind2Web（300 任务、136 个真实在线网站），ReAct 框架配 5 个 backbone（开源 Qwen3-VL-30B、Kimi 2.5；闭源 GPT-5.2、GPT-5.4-nano、Claude Sonnet 4.6），共 2,183 条轨迹，指标 Brier/E-AURC/AUROC。

- **对内部信号基线**：可观测信号在 Mind2Web 的 15 个 backbone×指标组合上全部持平或超过最强 UQ 基线（含需内部信号的 HTC-Full），WebArena-Lite 为 15 中胜 9；监督拟合对照组 Stacked L2 在十个设置上 E-AURC/AUROC 全输，说明优势不来自"套了同样的有监督分类器"。代表数字（AUROC）：Claude 上 HTC-Full 不可用，M&M 仍达 0.748（WebArena）/0.777（Mind2Web），远高于口头置信度 0.645/0.693；GPT-5.4-nano 在 Mind2Web 以 0.807 胜过 HTC-Full 0.736。但开源 Qwen3-VL-30B 上 HTC-Full 达 0.858，高于 M&M 的 0.838——内部信号在开源模型上仍是强基线。
- **信号分工**：Macro 单独更强更稳；Micro 单独略弱，并入 Macro 后对 Mind2Web 五个 backbone 的三个指标全部提升（配对 t 检验 p=0.012~0.032），WebArena-Lite 上无一致增益。
- **监督方式**：γ=0.5 下，关键步监督把成功轨迹（5,299 前缀）的误切率从 87.3–93.0% 压到 27.6–28.6%。
- **固定误切预算的干预**：在 8 个非 Claude 设置、1,576 轨迹交集中，可观测配置 8 个工作点里 7 个超过 HTC-Full；20% FCR 预算下检测率达 44.3%（WebArena-Lite）与 44.5%（Mind2Web），高于 HTC-Full 的 41.9%/40.4%。
- **跨站点泛化**：5 类网站留一类、size-matched 子采样后，M&M 与域内训练的 AUROC/E-AURC/Brier 差距仅 −0.0005/−0.0039/+0.0030，远小于类别间波动（≈0.013），平均 Brier 0.191 优于 HTC-Full 0.204（但 HTC-Full 在 E-Commerce 类全胜）。
- **案例**：设"截止日 2030 年 1 月"的任务（k\*=8，错点 Previous Month 并重复）中，Macro 在重复点击的 step 9 峰值 0.82，Micro 到 step 11 十次采样动作高度分歧时才跳至 0.96；另一条 Mind2Web 轨迹上内部信号全程不过阈值，Macro 因动作重复率在 step 6 报警，比环境重复守卫早 10 步。

## 4. 亮点与贡献（Why it matters）

1. 首次系统验证"纯可观测轨迹信号"足以支撑前缀级失败风险预测，让不开放 logits/hidden states 的闭源 agent 也能被监控。
2. 提出并拆解两类互补信号（Macro 读行为反馈、Micro 探决策一致性），用案例说明它们对同一失败的不同反应时机。
3. "关键失败步"式时间对齐监督是通用标注思想：避免把失败轨迹早期的有效行为误当坏样本，并有 150 条轨迹的三层可靠性统计背书。
4. 实验纪律好：与 HTC-Full 同口径对比、Stacked L2 控制监督收益、size-matched 跨类别评测、固定 FCR 预算报告操作点，结论可信。
5. 直接服务落地：可在固定"误切预算"下提前切断/接管正在失败的任务，对接生产环境的人工接管与成本控制诉求。

## 5. 局限与可改进点（个人点评）

- **架构覆盖窄**：只在 ReAct 式单 agent 上评测，预测器也仅是线性 logistic 回归，更复杂架构与序列模型的外推性未知。
- **"可观测"仍有假设**：Macro/Micro 都依赖 agent 暴露的决策/思考文本，只暴露"动作+反馈"的商用 agent 连这些都没有（作者也将其列为未来工作）。
- **监督因果性存疑**：关键步由 LLM judge 事后看完整轨迹标出，是"观测到未被纠正的错误"，并非客观不可恢复状态；离线的关键步与"线上实时可用风险"之间隔着一条较长的标注链。
- **评测维度有限**：只覆盖 Web 域，移动/桌面 GUI 未测；跨类别实验共享同批 backbone 与站点池，新站点/新模型/非平稳环境下的漂移及"换代后是否需重训"均未讨论。
- **可复现性与成本**：全文未声明开源；Micro 每步 N=10 次查询的采样与语义聚类开销不小，作者虽给出 N=5~8 的性价比平台，但这本身是部署前必须权衡的隐性约束。

## 6. 对我们的启示 / 可借鉴点

- **GUI Agent 部署的"监控哨兵"模板**：不必侵入模型、无需 logits，仅凭"页面观测+动作+环境反馈"喂给轻量分类器即可输出风险分——与我们做失败预警/人工接管的目标高度契合，可照搬"固定误切预算下调阈值"的决策框架来决定何时上人工。
- **行为信号基本可移植**：重复点击、双动作循环、连续 grounding/parse 错误、思考文本中的无效/换页自报等 Macro 清单接近领域无关的运行时症状，做移动端/桌面端可先套用再增量扩展。
- **关键步监督是通用标注配方**：凡用失败轨迹做训练数据（SFT 数据筛选、RL 奖励标注）都应避免把失败标签无脑回灌早期步；先标"何时走偏"再给标签，本质是一种 step-level credit assignment，能显著减少对有效早期行为的误伤。
- **信号分组报告的诊断价值**：分开报 Macro/Micro 再合并、并做留组消融（预测力分散在相关特征组而非个别特征），这种可解释的消融报告方式值得在自研特征中沿用。
- **成本—价值量化意识**：采样预算实验把解码开销、延迟建模与收益拐点一起给出，提示凡引入运行时重复查询都应附上类似的成本曲线，方便部署取舍。

## 7. 延伸阅读

- **PrefixGuard**（Huang et al., 2026, arXiv:2605.06455）：同样预测轨迹前缀失败、但用 outcome 式监督的在线监控器，是理解本文"关键步监督"改进点的直接对照。
- **Agentic Confidence Calibration / HTC**（Zhang et al., 2026d, arXiv:2601.15778）：主基线 HTC-Full 出处，代表"把 token 级置信度统计映射为任务失败概率"的内部信号路线。
- **AgentRx**（Barke et al., 2026, arXiv:2602.02475）：从事后执行轨迹诊断并定位关键错误步，"关键失败步"概念与两阶段标注协议由此而来。
- **Web-Shepherd**（Chae et al., 2026, NeurIPS 2025）：用过程奖励模型做步级网页导航评估，是"逐步监督导航轨迹"的另一互补路线；向上可溯至 **Where LLM Agents Fail**（Zhu et al., 2025）的 AgentErrorTaxonomy 失败分类法。

---
*解读生成时间：2026-09-04 09:02 ｜ 解读人：WorkBuddy（AI）*
