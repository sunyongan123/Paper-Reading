# NeoHorse-1: Towards Recursive Self-Improvement via Agentic Post-Training with Routing Harness

> 一句话 TL;DR：把部署中的路由 harness 当作"能力边界观测器"，用路由信号组织三阶段课程 SFT + 路由引导在线策略蒸馏，让模型在"评估—选择—更新"闭环里越用越强。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | NeoHorse-1: Towards Recursive Self-Improvement via Agentic Post-Training with Routing Harness |
| **作者 / 机构** | NeoHorse Team（署名按姓氏字母序；通讯作者 Yu Wang、Yunhe Wang\*）。机构：TokenRhythm Technologies、Infinigence AI、清华大学、北京大学、香港中文大学、Visionplus Capital、WX Capital、阿里巴巴集团（中国/中国香港） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-08；arXiv preprint（技术报告，cs.CL），arXiv:2609.08183v1 |
| **arXiv 链接** | https://arxiv.org/abs/2609.08183 |
| **代码仓库** | ✅ GitHub：https://github.com/TokenRhythm/NeoHorse |
| **数据集地址** | ⚠️ 模型合集：https://hf.co/collections/TokenRhythm/neohorse-1 ；训练语料（10^5–10^6 条 harness 轨迹）**未公开** |
| **类型标签（论文类别）** | `General` `Planning` `Distillation` `SFT` |
| **训练方法标签** | `SFT`（三阶段路由课程）、`Distillation`（Routing-Guided On-Policy Distillation，反向 KL） |
| **关键词** | 递归自我改进(RSI)、路由 harness、在线策略蒸馏、课程学习、能力导向数据分配、Agent 后训练 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_NeoHorse-1_Towards_Recursive_Self-Improvement_via_Agentic_Post-Training_with_Routing_Harness.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：为"递归自我改进（RSI）"补上一个具体机制——系统如何**观测自身能力边界**，并把这套观测转化为下一轮学习内容。作者落脚在 agentic post-training 这一具体场景。
- **为什么重要**：RSI 的结构性吸引力在于，一旦"模型改进模型"部分自动化，每代模型都能为下一代贡献数据，训练不再被人工标注卡死。但缺一个机制，RSI 就只是口号。agent 天然是载体——每次交互都留下"推理—工具调用—观测—结果"的轨迹，恰好是能力短板的可观测证据。
- **现有方法有什么不足**：多数轨迹方法只把交互记录当**静态监督**（FireAct、AgentTuning 的模仿 SFT、拒采样、蒸馏），学完即止，数据分布固定；作者明确批评"用实际服务的模型身份当难度标签"这一常见做法（它受用户覆盖、服务可用性、部署策略污染）。真正缺口是**闭环**：系统学到什么，必须影响它接下来从什么数据里学。
- **Research Gap**：作者认为自己补的是"评估—选择—更新"回路的**工程化实现**——路由 harness 本已内含观测机制，把它的轨迹/路由信号/结果三条信息组织成训练混合物，闭环驱动数据分配（作者原文 claim）。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

用路由信号（能力需求估计）作为课程轴，把部署轨迹组织成"三阶段课程 SFT + 路由引导在线策略蒸馏"，再靠能力反馈重配下一轮数据，形成数据飞轮。

### 3.2 方法总览（Pipeline）

- **输入**：多样真实任务 → 路由 harness（异构模型池 C0~C3）执行产生的交互轨迹；**输出**：NeoHorse-1 模型（4B/9B）。
- **三大模块**：
  1. **数据侧**：轨迹切成 user turn（基本序列化单元），保留当前轮交织推理/工具调用/观测，丢历史轮推理；经"确定性结构校验 + 六维语义评估"准入。
  2. **路由信号侧**：记录每个 turn 的**原始预测—策略调整决策—实际服务层级**三者分离，路由分数 = 硬排序（层级索引）或软排序（分数加权均值）。
  3. **训练侧**：用路由分数排三阶段课程 SFT（每阶段约 1/3 样本，刻意后段保留低分样本）；同套课程推广到 on-policy 蒸馏（学生自 rollout、固定教师给 token 分布、反向 KL）。
- **连接**：路由分数同时驱动 SFT 课程与 OPD 的上下文调度；评估反馈（能力缺口画像）决定下一轮数据混合配比，模型回 harness 产生新轨迹，闭环。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：`预测—动作—结果`三者分离的路由记录（不用"实际服务模型"当难度标签，用"预测需求"排课程）；把同一路由课程**贯通 SFT 与 OPD** 的统一调度。
- **工程组合**：课程学习、轨迹 SFT、on-policy distillation、反向 KL 均为已有组件，本文价值在于把它们焊进 harness 数据飞轮。
- **对性能提升最关键的设计**：数据来源对照（表 3，+6.26）说明**harness 真实轨迹 > 公开合成数据**；课程保留低分样本防止末期被高难交互独占。这是两个被实验证明贡献最大的点。
- **证据不足 / 仅声称有效**：能力导向数据分配（3.5 节）只有机制描述，**无消融定量支撑**；"Recursive"仅跑单轮，跨代累积增益未验证；蒸馏的 top-K 取值、教师身份、rollout 刷新频率均未公开。

---

## 4. 具体技术细节

### 4.1 模型结构

- **Base model**：Qwen3.5-4B / Qwen3.5-9B（纯 LLM，文本接口，非多模态 MLLM）；参数量 4B、9B。评测声明"仅评估文本接口"。
- 全参微调（未提及 LoRA/冻结）；部署统一用 SGLang v0.5.17，thinking mode（`enable_thinking=true, force_nonempty_content=true`），采样 temperature=1.0、top-p=0.95、top-k=20、presence penalty=1.5。

### 4.2 训练流程（routing harness 是重点）

| 阶段 | 目标 | 训练能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| **数据构建** | 把部署轨迹转成可监督样本 | 交织推理/工具/回复 | 路由 harness 真实轨迹（10^5–10^6）+ 公开指令/推理/代码/工具数据 | trajectory→user turn→subscene 三级 | 结构校验（因果顺序、tool-call/result 配对闭包）+ 六维语义评估（PASS/WARN/FAIL/NOT_EVALUATED，证据缺失不折算为正面） |
| **Stage 1–3（课程 SFT）** | 按能力需求递增引入样本 | 从低难到高难的任务执行 | 同语料，按路由分数排序 | user turn 序列化，token 级二值 mask | 掩码 SFT：`-Σ mᵢ,ₜ log p(xᵢ,ₜ|xᵢ,<ₜ)/Σ mᵢ,ₜ`，只监督当前轮 assistant 目标段（推理/工具调用/回复/结束符），历史与工具结果 mask=0 |
| **Routing-Guided OPD** | 缩小"学记录回复"与"自生成前缀"的分布差 | on-policy 行为 | 记录中"assistant 回复前的上下文"作起点 | 学生自 rollout 的响应 | response-normalized 反向 KL：`Σ wᵣ/Lᵣ · Σₜ KL(P̂_θ‖Q̂_teacher)`，学生 top-K 候选 + 1 个余量 bin，教师固定、逐阶段刷新学生 rollout checkpoint |

> 关键设计：路由分数 `sᵢ = kᵢ`（硬）或 `Σ k·πᵢ,ₖ`（软），**只用于排序调度，不重加权 loss**；三阶段每阶段约 1/3 样本，且**刻意保留部分低分样本到后段**，全程同一 loss、不重置优化器/学习率。

### 4.3 推理流程

- 推理即标准 agentic 部署：模型回 harness，服务 C0（有界低风险）~C3（最强/最可靠）四个层级请求；单次交互内部是多步（推理→工具调用→观测→再推理），终止由任务完成或预算控制。
- 推理本身不是本文技术贡献点，重点是**推理产生的轨迹回流训练**，构成下一轮数据。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| QwenClawBench | General（OpenClaw） | 端到端 Agent 执行 | 未公开 | 文本（指令+环境） | 动作/工件 | 成功率 |
| WorkBuddy Bench | General（多域办公） | 端到端 Agent 执行 | 未公开 | 文本 | 动作/工件 | 成功率 |
| PinchBench | General（OpenClaw 标准化） | 端到端 Agent 执行 | 未公开 | 文本 | 动作/工件 | 成功率 |
| VitaBench | General（日常生活服务） | 多轮交互 | 未公开 | 文本 | 动作/答复 | 成功率（judge） |
| BFCL V4 | General | 函数调用/工具使用 | 未公开 | 文本 | 函数调用 | 准确率 |
| τ²-Bench | General（Airline/Retail/Telecom） | 多轮任务完成 | 未公开 | 文本 | 动作/答复 | 成功率 |
| HumanEval | —（代码） | 代码生成 | 164 | 文本 | 代码 | Pass@1 正确率 |
| LiveCodeBench v6 | —（代码） | 竞赛级代码 | 未公开 | 文本 | 代码 | 正确率 |
| IFEval | —（指令遵循） | 可验证指令合规 | 未公开 | 文本 | 答复 | 指令级准确率 |
| IFBench | —（指令遵循） | 未见约束泛化 | 未公开 | 文本 | 答复 | 准确率 |

> 注：QwenClawBench/WorkBuddy/τ²-Bench 各跑 3 次取均值，PinchBench/VitaBench 单次；VitaBench 的 user-simulator 与 judge 因原推荐模型下架改用 DeepSeek-V4-Flash。

### 5.2 实验结果分析

**主结果（宏观平均，10 benchmark）**：4B **58.94 → 64.87**；9B **65.60 → 69.04**。4B 在**每一个**双方都有结果的 benchmark 上超过 Qwen3.5-4B；9B 在多数 benchmark 上超 Qwen3.5-9B，但指令遵循基本持平甚至微降（IFEval 89.46→89.09，-0.37）。增益集中在"交互执行"而非"静态指令合规"。NeoHorse-1-4B 在多个 benchmark 上已追平/超过 Qwen3.5-9B，说明训练补偿部分规模差距。单点亮点：HumanEval 4B 96.95、9B 98.17；τ²-Bench 4B 88.46、9B 90.82。

**Ablation 1（数据来源，表 3）**：同课程配置下，路由 harness 数据 vs 公开合成数据（Toucan）：五 benchmark 平均 **70.57 vs 64.32（+6.26）**，其中 HumanEval **+8.54**、τ²-Bench **+11.31**、IF +5.00、LCB +4.00、BFCL +2.43。直接回答"自采数据是否更值"。

**Ablation 2（监督量缩放，图 7）**：严格嵌套子集上，五 benchmark 平均从基座 **69.31** 升到最大规模的 **71.45**，单调上升，支持"高质量监督可规模化获益"。

**轨迹定性**：4B 在 QwenClaw 排期任务漏读含依赖约束的邮件、产出无效排期；9B 检索补充证据、重算并验证。PinchBench 数据分析中，9B 在 pandas 不可用后放弃反复安装、改用标准库完成，相比 4B **模型请求数/执行时间/token 分别减少约 70.8%/76.7%/83.6%**。

---

## 6. 亮点与贡献（Why it matters）

1. **"部署即数据源"落成可操作闭环**：路由不再只是推理成本优化手段，而是能力需求与短板的观测器，对任何有线上流量的 Agent 团队极具现实意义。
2. **预测—动作—结果分离很干净**：规避"用实际服务模型当难度标签"的谬误，让课程信号与部署策略解耦。
3. **数据质量管线值得抄**：结构校验与六维语义评估分离、证据缺失不折算为正面、证据覆盖率单独存储，可直接作为轨迹准入标准。
4. **同预算数据来源对照最有说服力**：用数字回答"为什么不用公开合成数据"。

---

## 7. 局限与可改进点（个人点评）

- **RSI 只跑了一轮**：标题"Recursive"目前是设计蓝图而非实测结论，跨代累积增益未验证，这是全文最大的名不副实之处。
- **路由信号可信度是隐性假设**：课程全程依赖 router 的能力需求估计是否校准，但文中**未报告 router 自身准确率**，只在未来工作里提"训练 router 本身"；若估计有偏，课程排序系统性偏移。
- **能力导向分配无消融**：3.5 节的"评估反馈→下一轮数据配比"只给了机制，缺定量支撑，是整个闭环里最薄弱、最像宣传的一环。
- **验证范围偏窄**：集中在 Agentic 与编码，harness 实际服务的更多能力未评估；9B 指令遵循微降，提示该配方存在能力此消彼长。
- **口径与可复现性**：部分对比成绩来自官方博客/报告（标 ∗）与自测混排；VitaBench 换了 judge 影响可比性；PinchBench/VitaBench 单次跑方差未知；蒸馏 top-K/教师/刷新频率未公开。

---

## 8. 对我们的启示 / 可借鉴点

**对 GUI Agent 的可借鉴点**（本篇为 Agent 方法论域，非 GUI 落地，故单列）：

- **路由 harness → GUI 训练闭环**：GUI Agent 天然跑在 harness（截图—解析—点击—观测）里，每步都留"请求+动作+观测+结果"。可照搬"预测—动作—结果"三元记录，让轻量 router 预估每步难度（C0 单步点击、C1 表单、C2 跨页多步、C3 长程异常恢复），用预估组织课程，省去人工难度标注。
- **路由分数当课程轴 → 缓解 GUI 长尾**：GUI 数据长尾严重（大量简单点击淹没少量复杂跨 App 流程）；三阶段课程 + "后段保留低分样本"正好防止末期遗忘基础交互。
- **On-policy 蒸馏直接可迁移**：GUI 分布漂移更严重（点错一步，后续观测全偏离记录轨迹）；用"每步动作前上下文"作起点、学生自 rollout、教师给 token 级监督，比纯轨迹 SFT 更贴合部署分布。
- **六维语义评估 → GUI 轨迹准入**：把"工具使用"换成"控件定位/操作正确性"、"证据一致性"换成"截图证据与动作一致"，即得 GUI 轨迹质量门槛。
- **数据来源对照方法论可复用**：GUI 领域同样有大量公开合成轨迹，建议固定课程配置下与自家真实轨迹做同预算对照，用数字说话。

---

## 9. 延伸阅读

- **Agentic Routing**（数据飞轮前作，路由信号设计来源）；**On-Policy Distillation**（反向 KL 目标的理论基础）
- **Agent Lightning v1.0**、**Co-Harness**：把环境循环放进部署 harness、联合更新 harness 与模型
- **Self-Harness / Agentic Harness Engineering / Retrospective Harness Optimization**：用失败与轨迹更新 harness
- **Agent-FLAN、AgentBank、FireAct、AgentTuning**：轨迹 SFT 与数据组成

---

*解读生成时间：2026-09-10 ｜ 解读人：WorkBuddy（AI）*
