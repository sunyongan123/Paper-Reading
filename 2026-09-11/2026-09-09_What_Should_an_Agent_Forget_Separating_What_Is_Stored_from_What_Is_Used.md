# What Should an Agent Forget? Separating What Is Stored from What Is Used

> 一句话 TL;DR：RD-Forget 把 Agent 记忆的"存什么"和"用什么"拆开——源档案全量保留，用查询条件化的记忆视图 + 同槽替换抑制过期事实，training-free 地提升五类记忆任务。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | What Should an Agent Forget? Separating What Is Stored from What Is Used |
| **作者 / 机构** | Yuhang Li\*（北京航空航天大学，中国杭州）、Yuchen Li\*（华东师范大学，中国上海）；\* 同等贡献，未标注通讯作者 |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-09；arXiv preprint，arXiv:2609.10263v1 [cs.AI]（正文未标注投稿会议/期刊） |
| **arXiv 链接** | https://arxiv.org/abs/2609.10263 |
| **代码仓库** | ❌ 未开源（全文未提供代码或项目页，prompt 细节仅以正文文字描述） |
| **数据集地址** | 未公开自建数据，全部复用公开 benchmark：AMB-Text 来自 AgentMemoryBench（https://openreview.net/forum?id=DT7JyQC3MR ）；LME-KU 来自 LongMemEval（https://openreview.net/forum?id=pZiyCaVuti ）；MAB-FC 来自 MemoryAgentBench 的 Conflict_Resolution 切分；BEAM（https://openreview.net/forum?id=y59hf5lrMn ）；PersonaMem-v1（https://openreview.net/forum?id=6ox8XZGOqP ） |
| **类型标签（论文类别）** | `General` |
| **训练方法标签** | —（不训练；training-free 框架，curation/selection/answering 全靠冻结 LLM 的提示工程） |
| **关键词** | 持久记忆、选择性遗忘、查询条件化检索、语义槽替换、上下文预算 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-09_What_Should_an_Agent_Forget_Separating_What_Is_Stored_from_What_Is_Used.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：持久化 Agent 要同时决定"存什么"与"用什么"。二者在世界变化时分叉：旧雇主记录会污染"现在雇主是谁"，却是"上一份工作在哪"的正确答案。
- **为什么重要**：遗忘若作用在存储上（删或覆盖），历史永久丢失，历史类、演化类、多跳类问题无法回答；完全不遗忘则过期事实污染当前回答。Generative Agents、MemGPT、A-MEM、ACE、ReasoningBank 擅长积累经验，却没把"可用"与"该用"解耦。
- **现有方法有什么不足**：① recency 分不清"竞争值"与"互补关系"——新雇主取代旧雇主是替换，而"新雇主所在地"是独立新增关系，按新旧排序会误删；② MemAct（RL 学上下文增删）与 FadeMem（衰减 + 冲突消解）的遗忘仍作用在留存层面，删了就没了，且需训练；③ 通用紧凑记忆会丢掉"只有问题到来才知道有用"的关系。
- **Research Gap**：把遗忘重定义为**查询局部**——观测留在档案里，只对某次回答降低影响力；并在不训练前提下实现可逆选择性遗忘、历史可复活与互补关系保留。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

保留一份不删东西的源档案 H；每次提问用冻结 LLM curator 抽原子证据并按 `subject|relation|scope` 槽分组，用"同槽 + ACTIVE 替代者"判定取代关系，再按查询意图与 rescue 开关决定非活跃条目能否回到候选，最后在 token 预算内贪心打包出记忆视图 Z(q)。

### 3.2 方法总览（Pipeline）

- **输入 / 输出**：输入源历史 H（带 session/时间戳/序号）、问题 q、预算 B=2048、rescue 开关 r；输出 g_θ(q, Z(q))，Z(q) 只影响本次回答，不改 H。
- **① Curator（冻结 LLM）**：抽最小完整证据集，每条含原子陈述 + 槽 + 源证据串 + 有效期。槽为 `subject|relation|scope`，故换雇主只更新 employment 槽，不碰 (新雇主, location) 独立槽；同时保留多跳关系链并逐 hop 消解更新。
- **② Materializer + Eligibility（规则）**：旧条目若有同槽 ACTIVE 替代者则写为 SUPERSEDED 且 superseded_by 指向它（式 4），否则不进物化记忆但源观测仍在 H。候选池 E(q)=A ∪（r=1 时 D）∪（r=1 且 h(q)=1 时 S）；h(q) 用词面线索 + 任务类型标签识别历史/演化/矛盾消解问题，只有历史意图才让 superseded 回候选。
- **③ Select & Pack**：词面重叠排序（ACTIVE +0.4）→ 冻结 LLM selector 返回有序 id → 按 c(m)=Tok+8 贪心塞入 B；curation 为空时从 H 返回最多 20 行兜底。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：**存储与使用的显式解耦 + 可逆槽级替换**。用三段槽决定"谁取代谁"而非按时间戳排序，解决 recency 分不清"竞争值 vs 互补关系"的老问题；`superseded_by` 让"被遗忘"本身可检索，配合 h(q) + rescue 构成条件性复活。
- **工程组合**：冻结 LLM 抽取 + 词面排序 + LLM 选择器 + 贪心打包都是成熟组件拼接；ACTIVE +0.4、B=2048 等是手调常数；多跳链保留靠 prompt 而非机制。
- **对性能最关键的设计**：**−FORGET 与 −QC 贡献最大**。Luna 上 −FORGET 掉 23.26 / 33.33 / 23.00 点，−QC 掉 13.95 / 11.54 / 18.00 点，差距说明"按查询构建视图"比"做槽替换"更根本；−SLOT / −RESCUE / −CLOSURE 只掉 3~10 点。
- **证据不足 / 仅声称有效**：① **rate–distortion 基本是装饰**——既没估计也没优化 D_q(Z)；② **时间语义全靠 benchmark 元数据**（序号、oracle session、query date），未在缺失时间戳下验证；③ **同槽替换正确性全押在 curator 的槽划分上**；④ 20 行 fallback 贡献未单独测量。

---

## 4. 具体技术细节

### 4.1 模型结构

base model 全是**纯文本 LLM，不含视觉编码器**：Qwen3.5-flash、GPT-5.6-Luna、MiniMax-M2.5、Kimi-K2.5（参数量未披露）。**全部冻结，不微调任何参数**；curation、selection、answering 与评测判分共用同一模型家族，作者自认只能算"协议内评测"。

### 4.2 训练流程（若需要训练）

**不训练**——training-free 框架，无任何梯度更新。按模板字段列出运行期阶段：

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| 建档案 | 保留全量源观测 | — | benchmark 原始对话/事实历史 H | 带 session/时间戳/序号的文本条目 | 无 loss |
| Curation | 抽相关证据 + 槽 + 多跳关系 | 冻结 LLM 零样本抽取 | H + 问题 q（+ query date、任务类型标签） | 原子陈述 + slot(subject\|relation\|scope) + 源证据串 + 有效期 | 无 loss；输出上限 4096 token |
| Materialize & Eligibility | 生成 superseded 链接、定候选池 | 确定性规则 | Curation 的 active/replaced 条目 | 带 status ∈ {ACTIVE, DEPRECATED, SUPERSEDED} 与 superseded_by 的条目表 | 无 loss（式 4、式 5） |
| Select & Pack | 2048 token 预算内选视图 | 冻结 LLM 排序 | 候选池 E(q) | 有序 id → 打包后的 Z(q) | 无 loss；selector 单次 256 token |
| Answer & Judge | 产出答案并打分 | — | q + Z(q) | 自然语言答案；同模型判分 | 无 loss |

> 评测规模：主套件 264 题/方法；query-intent 扩展再加 BEAM 150 题与 PersonaMem 579 题。

### 4.3 推理流程（若不训练或重点在推理）

- **模型与输入输出**：同一冻结 LLM 分饰 curator、selector、answerer 三角色；输入"问题 + 打包后的记忆视图（带 id 与 status）"，输出自然语言答案。
- **单步还是多步**：**单轮（single-pass）流水线**——一次查询触发固定顺序调用链 Curate → Materialize → Eligible → Select → Pack → Answer，无迭代回溯、无反思、无工具调用；所谓"多跳"是在单次 curation 内保留关系链，由 answerer 一步推出。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| AMB-Text | General（长期对话记忆） | 多会话对话记忆 QA | 86 题 | 对话历史（截到 session 边界）+ 问题 | 自然语言答案 | 二元语义判定均值（LLM judge） |
| LME-KU | General（知识更新） | knowledge-update QA | 78 题 | oracle sessions + query date + 问题 | 自然语言答案 | 同上 |
| MAB-FC | General（事实整合） | 多跳 fact consolidation，沿关系路径选当前修订值 | 100 对 QA | 事实历史 + 问题 | 自然语言答案 | 规则准确率：归一化子串匹配 |
| BEAM | General（超长历史，100K） | 知识更新 / 矛盾消解 / 时序推理 / 偏好遵循 | 150 题 | 长对话 + 问题 | 自然语言答案 | rubric 0/0.5/1，题内平均 |
| PersonaMem-v1 | General（画像演化，32K） | 依演化画像作答，四选一 | 579 题 | 截断上下文 + 问题 + 4 选项 | 选项字母 | 官方 single-option 准确率 |

### 5.2 实验结果分析

**主结果（Table 1，准确率 %，Mean 为三 benchmark 等权平均）**：

| 模型 | 方法 | AMB-Text | LME-KU | MAB-FC | Mean |
|---|---|---|---|---|---|
| Qwen3.5-flash | ACE | 52.33 | 60.26 | 34.00 | 48.86 |
| | ReasoningBank | 54.65 | 60.26 | 23.00 | 45.97 |
| | **RD-Forget** | **74.42** | **74.36** | **51.00** | **66.59** |
| GPT-5.6-Luna | ACE | 67.44 | 83.33 | 51.00 | 67.26 |
| | ReasoningBank | 75.58 | 71.79 | 46.00 | 64.46 |
| | **RD-Forget** | **89.53** | **93.59** | **77.00** | **86.71** |
| MiniMax-M2.5 | ACE | 69.77 | 61.54 | 42.00 | 57.77 |
| | ReasoningBank | 62.79 | 51.28 | 46.00 | 53.36 |
| | **RD-Forget** | **87.21** | **87.18** | **57.00** | **77.13** |
| Kimi-K2.5 | ACE | 81.40 | 70.51 | 41.00 | 64.30 |
| | ReasoningBank | 80.23 | 62.82 | 45.00 | 62.68 |
| | **RD-Forget** | **82.56** | **76.92** | **59.00** | **72.83** |

- 12 个"模型 × benchmark"组合全部第一。相对较强基线：AMB-Text +1.16~19.77 点，LME-KU +6.41~25.64 点，MAB-FC +11.00~26.00 点。**提升最大的是 MAB-FC**（该任务正要求沿关系路径选当前修订值）；最小提升是 Kimi 的 AMB-Text（+1.16 点）。
- **query-intent 扩展（Table 3）**：BEAM 上四个骨干为 50.21 / 70.58 / 42.39 / 57.83，PersonaMem 上为 59.93 / 79.97 / 60.10 / 64.08，8 个组合全胜；增益 BEAM 8.47~15.67 点、PersonaMem 10.88~20.72 点。

**Ablation（Table 2，Luna，准确率 %）说明了什么**：

| 变体 | AMB-Text | LME-KU | MAB-FC | 相对 matched Full 缺口 |
|---|---|---|---|---|
| matched Full | 91.86 | 93.59 | 74.00 | — |
| −FORGET（旧修订保持 ACTIVE） | 68.60 | 60.26 | 51.00 | −23.26 / −33.33 / −23.00 |
| −QC（curator 看不到问题） | 77.91 | 82.05 | 56.00 | −13.95 / −11.54 / −18.00 |
| −RESCUE（非 ACTIVE 不进检索） | 88.37 | 91.03 | 65.00 | −3.49 / −2.56 / −9.00 |
| −SLOT（去掉同槽分组） | 86.05 | 89.74 | 64.00 | −5.81 / −3.85 / −10.00 |
| −CLOSURE（不展开多跳链） | 87.21 | 89.74 | 71.00 | −4.65 / −3.85 / −3.00 |

① **−FORGET 贡献最大**：三个 suite 都是最大缺口（LME-KU 掉 33.33 点），说明过期值留在候选里是当前状态问题的主要错误源。② **−QC 第二**：它是在"curation 额度更大 + 下游 selector 仍能看到问题"的有利条件下掉的，说明"见到问题前就把记忆压成通用摘要"本身不可逆。③ **−SLOT / −RESCUE / −CLOSURE 掉 3~10 点**，且与 Table 1/2 同配置的批次波动（AMB-Text 89.53 vs 91.86、MAB-FC 77.00 vs 74.00）同量级，结论不够稳固。

---

## 6. 亮点与贡献（Why it matters）

1. **把"遗忘"从删除操作变成检索策略**：存储层只增不减（审计友好、可回溯），影响力在回答时按查询裁剪，工程上可拆成"不可变 archive + 可插拔 view builder"。
2. **同槽替换是解决时效性失效的干净机制**：`subject|relation|scope` 让替换落在正确粒度，换雇主不会连带否掉雇主所在地。
3. **`superseded_by` 让"被遗忘"可检索**：被取代的事实带着"被谁取代"的指针留存，使历史问题可复活、演化问题可同时呈现两个状态。
4. **消融给出清晰的设计优先级**：预算有限时先做查询条件化 curation，再做同槽替换，最后优化槽分组与多跳闭包。
5. **training-free 带来可替换性**：curator/selector 只是 prompt 角色，换骨干即可迁移（4 个家族骨干全部第一）。
## 7. 局限与可改进点（个人点评）

- **判分与作答同模型**：AMB-Text/LME-KU 是 LLM 二元判定、BEAM 是 rubric 打分，对判分模型高度敏感；缺换 judge 的交叉验证。
- **baseline 只有两个且都是上下文工程类**：相关工作点名的 MemAct、FadeMem、A-MEM、MemGPT 一个都没比，"最高准确率"实际只是"在 ACE 与 ReasoningBank 之间最高"。
- **大量依赖 benchmark 元数据**：序号、oracle session、query date、session 边界都是评测集白送的；未做"元数据退化"实验，这是最该补的鲁棒性组。
- **规模小、无统计检验且批次不一致**：每方法 264 题、单 benchmark 仅 78~100 题，Kimi 优势仅 1.16 点却无标准差；Table 2 的 matched Full 与 Table 1 同名配置差 2~3 点，而 −CLOSURE / −RESCUE 缺口正好 3~5 点，可能一半来自波动。
- **消融是整体开关**：作者自认 −QC / −FORGET 会同时改变上游抽取与下游 eligibility，无法定位因果。
- **单轮流水线的能力天花板**：抽什么、谁取代谁、选哪些都在一次前向里拍定，没有"先检索、发现不够、再回查"的循环。
- **成本未被讨论**：每题至少 3 次 LLM 调用 + 1 次 judge，curation 上限 4096 token 而记忆预算仅 2048。

## 8. 对我们的启示 / 可借鉴点

**通用方法论**：记忆系统的正确抽象可能是"不可变日志 + 查询时视图"而非"可变记忆库"；用 LLM 做抽取/选择的系统都必须回答"元数据从哪来"。

### 对 GUI Agent 的可借鉴点

1. **GUI 记忆必须"存全、用窄"**。GUI 状态变化远快于对话（URL 跳转、DOM 重排、元素消失、弹窗遮挡），每步都可能让上一步观测过期。把历史截图/DOM 快照/动作序列作为不可变 archive，每次决策只构建"当前页面 + 当前目标"条件化视图，避免旧状态污染动作选择，同时保留"回到上一页/撤销"类历史意图。
2. **把语义槽换成 GUI 版 `app|页面|元素`**。同槽替换即"同一元素的新属性覆盖旧属性"（位置、开关状态、字段值），不同槽观测保留为互补证据。注意同一元素的新旧坐标是竞争，元素 A 与 B 的坐标是互补，槽错一个层级就会误删信息。
3. **显式 token 预算打包对 GUI 尤其关键**。GUI 上下文被截图 patch token 吃掉大半，c(m)=Tok+8 加贪心塞预算可直接复用到 observation 压缩：给每条记忆标 token 成本后贪心选择。
4. **把 rescue 改造成"失败动作黑名单 + 条件性重试"**。GUI Agent 常重复执行同一失败动作；可把"失败动作 + 当时页面状态"存成 SUPERSEDED/DEPRECATED，默认不进上下文，仅在显式信号（任务标签 h(q) 表示"上次这里报错"）出现时放回候选。
5. **评测要用 GUI 原生的状态更新子集，且必须换 judge**。挑"同一页面状态多次变化"的时序子集（多会话、跨页返回、A/B 布局）做消融，并用与 agent 不同的模型判分。
## 9. 延伸阅读

- Cover & Thomas, *Elements of Information Theory*——式 (1)(2) 的 rate–distortion 来源。
- Zhang et al., *Agentic Context Engineering (ACE)*（ICLR 2026）与 Ouyang et al., *ReasoningBank*（ICLR 2026）——两个直接 baseline。
- Zhang et al., *Memory as Action (MemAct)*（ACL 2026 Findings）——用 RL 学上下文增删，与本文 training-free 路线形成对照，也是最好的下一步基线。
- Wei et al., *FadeMem*（ICASSP 2026）、Hu et al., *MemoryAgentBench*（ICLR 2026）、Wu et al., *LongMemEval*（ICLR 2025）。
- 本地相关：`2026-09-09_Procedural_Memory_Under_Change_Reuse_and_Interference_in_Controlled_Web_Tasks`、`2026-09-07_MEMO_Multimodal_Evidence_Memory_Organization_for_Long-Horizon_LLM_Agents`。

---
*解读生成时间：2026-09-11 09:00 ｜ 解读人：WorkBuddy（AI）*
