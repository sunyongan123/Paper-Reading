# NeoHorse-1: Towards Recursive Self-Improvement via Agentic Post-Training with Routing Harness

> 把"路由 harness"本身当作自我改进的传感器：用路由信号组织三阶段课程 SFT 与在线策略蒸馏，让模型越用越强。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | NeoHorse-1: Towards Recursive Self-Improvement via Agentic Post-Training with Routing Harness |
| **arXiv ID / DOI** | 2609.08183 |
| **arXiv 链接** | https://arxiv.org/abs/2609.08183（点击直达） |
| **发表出处（Venue）** | arXiv preprint（技术报告，cs.CL） |
| **发布时间** | 2026-09-08 |
| **作者** | NeoHorse Team（署名按姓氏字母序；通讯作者：Yu Wang、Yunhe Wang）。核心贡献者含 Guoliang Cao、Guohao Dai、Tianyu Guo、Kai Han、Hailin Hu、Zihan Jiang、Xiang Kuang、Boxun Li、Yulong Li、Zehua Pei、Yuchuan Tian、Jiamin Wang、Yihong Wu、Haiyang Xu、Shuo Zhang、Hang Zhou 等 |
| **所属机构** | TokenRhythm Technologies、Infinigence AI、清华大学、北京大学、香港中文大学、Visionplus Capital、WX Capital、阿里巴巴集团（中国/中国香港） |
| **开源情况 / 代码** | ✅ 有代码：https://github.com/TokenRhythm/NeoHorse ；模型合集：https://hf.co/collections/TokenRhythm/neohorse-1 |
| **类型标签（论文类别）** | `SFT` `Distillation` `Planning` `General` |
| **训练方法标签** | `SFT`（三阶段路由课程）、`Distillation`（Routing-Guided On-Policy Distillation，反向 KL） |
| **关键词** | 递归自我改进(RSI)、路由 harness、在线策略蒸馏、课程学习、能力导向数据分配、Agent 后训练 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_NeoHorse-1_Towards_Recursive_Self-Improvement_via_Agentic_Post-Training_with_Routing_Harness.pdf |

---

## 1. 研究背景与要解决的问题

递归自我改进（RSI）的吸引力是结构性的：一旦"模型改进模型"被部分自动化，每代模型都能为下一代贡献产出，训练就不再完全受限于人工标注数据。但作者指出，RSI 需要一个**具体机制**——系统能观测自己的能力边界，并把观测转化为下一轮学习内容。

本文的切入很巧妙：**部署中的路由 harness 本身就已经包含这个机制**。harness 是管理 Agent 上下文、工具调用与环境交互的执行层；叠加路由模块后可按请求与交互状态选择不同模型（C0~C3 四个服务层级）。于是每次交互不仅产出任务结果，还留下三样可复用的东西：执行轨迹（把训练锚定在真实交互上）、路由信号（刻画每次请求的能力需求）、记录下来的结果（暴露模型短板）。

现有做法的缺口在于：多数工作只把轨迹当作**静态监督**——蒸馏、模仿、拒采样微调，学完即止，数据分布固定。而 RSI 要求闭环：**系统学到什么，会影响它接下来从什么数据里学**。本文要补的就是这条"评估—选择—更新"回路的工程化实现。

## 2. 核心方法 / 思路

整体是一条数据飞轮：多样任务 → 路由 harness（异构模型池）产生交互经验 → 经验被组织成训练混合物 → 训练出 NeoHorse-1 → 更新后的模型回到 harness → 产生新轨迹与能力反馈 → 指导下一轮数据分配。方法侧有三个部件。

**（一）数据侧：把轨迹切成"用户轮"训练样本。** 数据分三个粒度：trajectory（完整执行历史）、user turn（从一个用户请求到下一个，是基本序列化单元）、subscene（若干共享局部目标的相邻 user turn，是语义标注单元）。每个 user turn 保留当前请求、交织的推理、工具调用与观测，形成"推理—行动—反馈"链；**历史轮次的推理被丢弃**，可见回复与工具交互保留为上下文。质量门槛分两道：先做确定性结构校验（事件因果顺序、tool-call/result 配对闭包），再做**六维语义评估**（目标达成、指令遵循、工具使用、证据一致性、错误恢复、终止），每维给 PASS/WARN/FAIL/NOT_EVALUATED，绝不把"缺证据"折算成正面结论。

**（二）路由信号：预测—动作—结果三者分离。** 路由器在 user-turn 级根据当前请求、近期对话、历史路由决策与执行状态估计能力需求，分到 C0~C3。语料同时保留**原始预测、策略调整后的决策、实际服务的层级**。作者强调：不能把"实际服务的模型身份"当难度标签，因为它还受用户覆盖、服务可用性与部署策略影响。

**（三）训练侧：课程 SFT + 路由引导的在线策略蒸馏。** SFT 用 token 级二值 loss mask，只监督当前轮 assistant 的保留目标段（推理、工具调用与参数、可见回复、结束符）。路由分数 𝑠ᵢ 取硬排序或软排序（层级索引的分数加权均值），据此把样本排成**三阶段课程**：逐阶段引入更高分样本，但**刻意保留部分低分样本到后段**，避免训练末期被高难度交互独占；全程用同一 loss。由于 SFT 学的是记录下来的回复，而部署时学生在**自己生成的 prefix** 上解码，作者把同一套课程推广到 on-policy distillation：用记录中"assistant 回复前的上下文"作起点，学生自己 rollout，固定教师在每个位置给下一 token 分布，用 **response-normalized 反向 KL** 更新学生。

## 3. 关键实验结果

评测覆盖 10 个 benchmark：Agentic（QwenClawBench、WorkBuddy Bench、PinchBench、VitaBench、BFCL V4、τ²-Bench）、Coding（HumanEval、LiveCodeBench v6）、Instruction Following（IFEval、IFBench）。基座为 Qwen3.5-4B / 9B，统一使用各 benchmark 的 harness、工具接口、上下文上限与交互预算，用 SGLang v0.5.17 部署。

**主结果**：后训练把宏观平均分从 **58.94 提升到 64.87（4B）**、从 **65.60 提升到 69.04（9B）**。4B 在每一个双方都有结果的 benchmark 上超过 Qwen3.5-4B；9B 在多数 benchmark 上超过 Qwen3.5-9B，但指令遵循基本持平甚至有一项微降，说明增益集中在"交互执行"而非"静态指令合规"。NeoHorse-1-4B 在若干 benchmark 上已追平或超过 Qwen3.5-9B，说明训练能补偿一部分规模差距。

**数据来源对照（表 3）**：完全相同的课程配置下，路由 harness 数据显著优于公开合成工具 Agent 数据（Toucan），五 benchmark 非加权平均 **70.57 vs 64.32（+6.26）**，其中 HumanEval +8.54、τ²-Bench +11.31。

**监督量缩放（图 7）**：在严格嵌套子集上，五 benchmark 平均从基座 69.31 稳步升到最大数据规模时的 **71.45**。

**轨迹定性分析**：4B 在 QwenClaw 排期任务中漏读了含更新依赖约束的邮件，基于过时信息产出无效排期；9B 能检索补充证据、重算并验证。在 PinchBench 数据分析任务中，9B 发现 pandas 不可用后放弃反复安装，改用标准库完成分析与报告，相比 4B 轨迹**模型请求数、执行时间、token 用量分别减少约 70.8%、76.7%、83.6%**。

## 4. 亮点与贡献（Why it matters）

1. **把"部署即数据源"讲成了可操作的闭环**：路由不再只是推理成本优化手段，而是能力需求与短板的观测器，对任何有线上流量的 Agent 团队都极具现实意义。
2. **预测—动作—结果分离的设计很干净**：避免了"用服务的模型当难度标签"这一常见谬误，课程信号与部署策略解耦。
3. **数据质量管线值得抄**：结构校验与六维语义评估分离、不确定不折算为正面、证据覆盖率单独存储，可直接借鉴为轨迹准入标准。
4. **同预算下的数据来源对照**是全文最有说服力的实验之一，直接回答了"为什么不用公开数据"。

## 5. 局限与可改进点（个人点评）

- **RSI 只跑了一轮**。作者自己也承认这只是单次"评估—选择—更新"的初步验证，能否跨代累积增益尚未测试；标题里的 "Recursive" 目前更多是设计蓝图而非实测结论。
- **路由信号的可信度是隐性假设**。整条课程依赖 router 的能力需求估计是否校准，而文中未报告 router 自身准确率，未来工作才提出"训练 router 本身"。若路由估计有偏，课程排序会系统性偏移。
- **验证范围偏窄**。评测集中在 Agentic 与编码，harness 实际服务的更多能力未评估；9B 在指令遵循上出现微降，说明该配方可能存在能力此消彼长。
- **对齐口径不够透明**。部分对比模型成绩来自官方博客/技术报告（表中标 ∗），与自测结果混排；VitaBench 的 user-simulator 与 judge 因原推荐模型下架而换成 DeepSeek-V4-Flash，可比性受影响；PinchBench、VitaBench 只跑单次，方差未知。
- **蒸馏细节未公开**。top-K 的 K 取值、教师模型身份、rollout 刷新频率等关键超参缺失，复现门槛偏高。

## 6. 对我们的启示 / 可借鉴点

**对 GUI Agent 的可借鉴点**：

- **自改进后训练 / 路由 harness → GUI Agent 训练闭环**。GUI Agent 天然跑在 harness（截图—解析—点击—观测）里，每步都留下"请求 + 动作 + 观测 + 结果"。可照搬其"预测—动作—结果"三元记录：让轻量 router 在每步预估难度（C0 单步点击、C1 常规表单、C2 跨页多步、C3 长程异常恢复），再用该预估组织 GUI 训练样本的课程，而不靠人工标注难度。
- **路由分数当课程轴 → 缓解 GUI 长尾**。GUI 数据极度长尾：大量简单点击淹没少量复杂跨 App 流程。三阶段课程 + "保留部分低分样本到后段"正好防止模型在训练末期只见高难样本而遗忘基础交互。
- **On-policy 蒸馏直接可迁移**。GUI 的分布漂移更严重（一旦点错，后续观测完全偏离记录轨迹）。用"记录中每步动作前的上下文"作起点、学生自己 rollout、教师给 token 级监督，能有效纠正累积误差，比纯轨迹 SFT 更贴合部署分布。
- **六维语义评估 → GUI 轨迹准入**。把"工具使用"换成"控件定位/操作正确性"、"证据一致性"换成"截图证据与动作一致"，即得一套 GUI 轨迹质量门槛；"不确定不折算为正面"对 GUI 这种标注噪声大的场景尤其关键。
- **数据来源对照方法论值得复用**：GUI 领域同样有大量公开合成轨迹，建议在固定课程配置下与自家真实轨迹做同预算对照，用数字回答"自采数据是否更值"。

## 7. 延伸阅读

- Agentic Routing（本文数据飞轮的前作，路由信号设计来源）；On-Policy Distillation（本文蒸馏目标的理论基础）
- Agent Lightning v1.0、Co-Harness：把环境循环放进部署 harness、联合更新 harness 与模型
- Self-Harness / Agentic Harness Engineering / Retrospective Harness Optimization：用失败与轨迹更新 harness
- Agent-FLAN、AgentBank、FireAct、AgentTuning：轨迹 SFT 与数据组成

---
*解读生成时间：2026-09-10 08:39 ｜ 解读人：WorkBuddy（AI）*
