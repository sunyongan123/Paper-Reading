# The Menu Is an Execution Prior: State-Path Tool Menus for Online Agents

> 一句话 TL;DR：把"执行前给 agent 看的短工具菜单"当作执行先验，用状态路径（entry→bridge→target→terminal）学出"选谁进菜单 + 谁排前面"，在不改 agent 的前提下把 ToolBench 在线成功率从 0.737 提到 0.898。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | The Menu Is an Execution Prior: State-Path Tool Menus for Online Agents |
| **作者 / 机构** | Bo Yan（University of Central Florida，计算机系与人工智能研究所，美国）、Weikai Lin（University of Rochester，计算机系，美国）、Song Wang（UCF，通讯作者 song.wang@ucf.edu） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-08；arXiv preprint（arXiv:2609.09395v1，cs.AI） |
| **arXiv 链接** | https://arxiv.org/abs/2609.09395（点击直达） |
| **代码仓库** | ❌ 未开源。论文未给出任何仓库、项目主页或代码可用性声明 |
| **数据集地址** | ✅ 使用公开基准：ToolBench（官方仓库 https://github.com/OpenBMB/ToolBench ）、AppWorld、TRAJECT-Bench、UniToolCall、ToolHop、AssistantBench。论文未提供统一的处理后数据或菜单构造结果下载 |
| **类型标签（论文类别）** | `Online` `Planning` `Web` `General` |
| **训练方法标签** | 监督训练（非 LLM 微调）：训练 retriever（0.416M）与 reranker（0.449M）两个小 Transformer，文本编码器 BGE-large-en-v1.5 冻结 |
| **关键词** | tool menu、state path、execution prior、tool retrieval、producer-before-consumer ordering、ToolBench |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_The_Menu_Is_an_Execution_Prior_State-Path_Tool_Menus_for_Online_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：在线工具使用场景下的"动作空间构造"。工具库大到 ToolBench 的 16,000+ API 时，系统只能执行前检索小候选集。论文把这个"请求特定的有序接口子集"命名为 **tool menu**：agent 只能调用菜单内工具，菜单执行期固定，循环与预算不变。
- **为什么重要**：多步任务成功需要"最终动作"和"生产其输入的前置工具"同时在场且顺序正确。菜单里少一个安静的 bridge，或把 consumer 排在 producer 之前，agent 第一步就拿到不可执行的动作——而它的推理与预算都没问题。**菜单构造因此是纯接口层的杠杆**。
- **现有方法不足**：主流构造器（ToolRet、ToolRerank、SkillRouter、ToolGen、官方检索器）都按**请求相关度**逐工具打分，无法表达"这组调用能否构成完整路线"。反例：只可见 order_id 时，相关度菜单把 `SENDEMAILRECEIPT` 排第一，而它需要的 email 与 receipt_id 尚不存在。**COLT** 最接近，追求覆盖但不排序；**AutoTool / ToolTree** 在执行中更新选择，仍解决不了"第一个动作不可执行"。
- **Research Gap**：把有界有序菜单形式化为 **execution prior**，并补上缺失目标——**state-path completeness**：每个前置工具必须能从可观测初始状态出发执行，且排在它的消费者之前。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

用 encoder 学出三种方向信号（工具现在能不能跑、能否生产后续字段、哪些顺序在轨迹里反复出现），retriever 按路径角色边际覆盖选出 32 工具，reranker 把前 8 位排成可执行前缀，一次性输出有序菜单。

### 3.2 方法总览（Pipeline）

- **输入 / 输出**：请求 q、可观测初始状态 s0(q)、工具库 T（文档 + I/O schema）、训练轨迹冻结的路径统计 M → 32 工具有序菜单，执行前一次交付、之后不再改变。
- **Encoder**：三个方向信号——**状态兼容 χ_i**（工具输入有多少已在可见状态中）、**schema flow κ_ij**（u_i 输出覆盖 u_j 输入的比例，名字不相似也能发现 producer→consumer）、**路径优先 ρ(i,j)=log((c(i≺j)+1)/(c(j≺i)+1))**；节点特征 + 路径摘要 token 送入关系感知 Transformer，工具对关系类型作 attention bias。
- **Retriever**：从 128 工具宽前沿逐槽边际解码 `s_i = M_i + λC_i − ηD_i`；每选一个工具就更新未覆盖角色与字段，使 entry 入列后其输入的生产者价值升高。
- **Reranker**：只接收选出的 32 工具，优化**长度 8 的可执行前缀**（entry 分只对第 1 位生效、slot 分、precedence 分、把已选工具输出并入前缀状态再评估下一候选）；后 24 位保留检索顺序。三阶段串行、全在执行前完成。
### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：① **state path 形式化**：`s0(q) → ENTRY → BRIDGE* → TARGET [→ TERMINAL]`，每个工具指派四种路径角色，把"能否跑通"变成"假设路线是否完整"；② **三个方向信号 + 关系类型注入 attention bias**，κ 在工具名无重叠时也能识别 producer-consumer；③ **把"顺序"作为独立可学目标**。
- **工程组合**：encoder–retriever–reranker、关系感知 Transformer、MMR 式边际覆盖均为既有件；贡献不在架构新颖性，而在**把对象从"工具集合"换成"可执行路线"**。
- **对性能最关键的设计**：消融给出干净三级排序——只改官方 32 工具的**顺序** 0.737 → **0.773**；固定路径模板也是 **0.773**；**学习成员选择 + 学习顺序** 到 **0.898**（较任一局部变体 +12.5pt）。**"选谁进菜单"比"怎么排"贡献更大，但两者必须一起学**；路径统计提供的是**"重复证据"而非精确路径**，对局部噪声鲁棒。
- **证据不足 / 仅声称有效**：① **跨 executor 证据太薄**，只有三个点（+16.1/+12.8/+6.9），无分解；② **12 个"answer rejection 后成功"案例机制未解释**，却占新增收益 24%；③ **固定模板已拿 0.773**，学习净增量有多少来自开发集调优无法分离；④ **跨库迁移与冷启动证据不足**：first tool 仅 +2.2pt、chain 仅 +0.7pt，M 依赖有历史轨迹的库。

---

## 4. 具体技术细节

### 4.1 模型结构

被服务的在线 agent 是 **LLM，不是 MLLM**：**Qwen2.5-72B-AWQ**，纯文本，**无视觉编码器**，确定性解码，**完全冻结、不微调**，8 次工具调用预算。构造器是两个很小的 Transformer：**retriever 0.416M、reranker 0.449M**，文本编码器为**冻结的 BGE-large-en-v1.5**（取前 64 维并 ℓ2 归一化），hidden 128、2 layers，前缀长度 8。**本篇是纯文本工具接口层论文**：优化的是"agent 看见什么"，而非"怎么看屏幕"。

### 4.2 训练流程

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| S1 Encoder+Retriever | 学路线成员、入口与槽位 | 状态兼容 + schema 流向 + 历史顺序 | ToolBench 训练**成功轨迹**（目标任务排除，out-of-fold） | 工具节点（12 维）+ 5 类关系类型 + 路径摘要 token | membership / entry / path-slot 监督；负采样 hard / in-frontier / boundary；lr 8×10⁻⁴，2 epochs |
| S2 Reranker | 把 32 工具排成可执行前缀 | producer 先于 consumer、首动作可执行 | 同批轨迹 slot / precedence 标签；菜单由 out-of-fold 检索器生成 | 32 节点（14 维）+ 16 维成对 attention bias | `L_O = λ_entry L_entry + λ_slot L_slot + λ_prec L_prec + λ_member L_member + λ_path L_path`，权重 0.55/0.55/0.75/0.30/1.15；lr 1×10⁻⁴，4 epochs |
| S3 路径统计 M | 提供 ρ(i,j) 顺序先验 | 跨库可迁移的顺序偏置 | 训练轨迹共现计数 | 有序对计数表 c(i≺j) | 非梯度更新：ρ = log((c(i≺j)+1)/(c(j≺i)+1)) |
| 在线 agent（冻结） | 无 | 不训练 | — | — | 不改 prompt / 循环 / 预算，仅接收最终 32 工具有序菜单 |

> 关键约束：**构建最终菜单时，目标任务的参考链、评测器决定、执行后 observation 与失败标签全部不可见**。

### 4.3 推理流程

**两段式：菜单构造是单次前向，agent 执行是多步循环。** 菜单构造**一次**完成、不随观测更新；开销 median 406.41 ms / P95 552.85 ms，两个神经通路合计仅约 6 ms，**大部分时间花在显式路径特征构造与约束解码**，菜单只建一次、被后续调用共享。agent 执行是**多步**：每步要么调一个菜单内工具并给出合法 JSON 参数，要么输出最终答案；**无效与有效调用消耗同样 8 次预算**，缺工具、参数非法、异常与空响应统一转成标准化 observation。终止条件 = 请求被满足或预算耗尽，**无反思与重规划**。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| ToolBench（主评测） | General（API 工具库） | 端到端多步工具调用 | 304 个可执行任务（分 107/134/63 三组） | 任务文本 + 32 工具有序菜单 | JSON 工具调用 + 最终答案 | 官方 ToolEval 二值成功率；另报 chain coverage |
| AppWorld / TRAJECT-Bench / UniToolCall / ToolHop | General | 有状态执行 / 轨迹感知 / 有序前缀 / 首动作 | TRAJECT-Bench 跨库迁移 2,000 个 hard 任务；其余未给规模 | 同上 | 同上 | Next tool / Full chain / Ordered prefix / First action |
| AssistantBench（附录诊断） | Web | 网页证据路由 | 33 个 public validation | 同一 web cache + top-10 页面预算 | 答案 | 官方 BrowserGym scorer：hit / accuracy |

### 5.2 实验结果分析

**主结果（所有条件都只把 32 工具有序菜单交给同一个 agent）**：

| Menu constructor | ToolBench 成功率 | AppWorld next tool | TRAJECT-Bench full chain | UniToolCall ordered prefix | ToolHop first action |
|---|---|---|---|---|---|
| ToolBench official | **0.737** | – | – | – | – |
| COLT | 0.582 | 0.190 | 0.230 | 0.215 | 0.238 |
| ToolRet | 0.632 | 0.114 | 0.188 | 0.134 | 0.196 |
| Tool-REX family | 0.661 | 0.180 | 0.350 | 0.262 | 0.597 |
| SkillRouter | 0.707 | 0.123 | 0.167 | 0.169 | 0.467 |
| ToolGen | 0.730 | 0.049 | 0.708 | 0.215 | 0.265 |
| **State-Path (ours)** | **0.898** | **0.465** | **0.732** | **0.635** | **0.683** |

ToolBench 上 State-Path 解出 **273/304**，官方菜单 224/304；51 个不一致任务里**赢 50 输 1**。**增益拆解**：保留 223 个官方成功、丢 1、新增 50——27 个（54%）"官方菜单缺完整链"、11 个（22%）"入口缺失或过晚"、12 个（24%）"答案被拒后成功"。路径诊断：完整链 0.510 → 0.704；**官方 128 工具 0.691 vs 本文 32 工具 0.704**。评测器稳定性：三个独立评测器分别 0.70 / 0.72 / 0.34，均把 State-Path 排第一。

**Ablation 说明了什么**：① 只重排官方候选就 +3.6pt（0.737→0.773），说明**菜单顺序是接口的一部分**；② **固定模板也拿 0.773**，说明相当一部分收益来自"entry→bridge→target"这一通用路线形状先验；③ **成员选择 + 顺序必须联合学习**，完整方法比两个局部变体各高 **12.5pt**。补充证据：25%/50% 路径各破坏一步后覆盖率仅 0.701/0.694，顺序先验对噪声鲁棒；仅工具名时 0.628 边准确率 / 0.105 replay，加 I/O 字段后 0.676 / 0.155，**完整描述与结构化字段打平**。附录 ToolTree / AutoTool / Dynamic 完整链覆盖 0.609 / 0.592 / 0.388，均低于 0.704。

---

## 6. 亮点与贡献（Why it matters）

1. **把"工具菜单"提升为研究对象，并给出可证伪的机制解释**：用 chain coverage / entry-in-top-5 / executable prefix 三个诊断量把增益归因到"路线可见性"，证明优势随 bridge 数量增长。
2. **"不改 agent，只改输入接口"是极强的实验设计**：同一 executor、8 步预算、prompt 与评测器，唯一变量是菜单；**32 个选得好的工具还胜过 128 个官方列表**（chain 0.704 vs 0.691），问题不是"给更多工具"，而是"给一条能跑的路线"。
## 7. 局限与可改进点（个人点评）

1. **菜单一次性冻结是最大的结构性限制，跨 executor 证据也太单薄**。真实在线 agent 会观察、会失败、会重规划；把菜单冻结在执行前，等于假设"请求文本 + 工具 schema 足以确定路线"。论文承认了，却没测量其在长视野任务上的代价——8 步预算偏短，掩盖了问题。跨 executor 只有主表一个执行者与 Figure 4 三个点，无置信区间与 per-task 分解，支撑明显弱于主结果。
2. **24% 的新增收益机制不明，增益归因也无法完全分离**。12 个 answer-rejection 案例只被解释为"答案取决于它看到的路线"，更像观察而非机制，无 trace 分析就是黑箱；固定模板已 0.773、学习版 0.898，中间 12.5pt 需同时归因于"更好的成员集合"与"请求特定顺序"，但消融没做交叉变体。**冷启动覆盖也不足**：只有工具名时 replay success 仅 0.105，无路径记忆时 chain 只有 0.510，新库、私有库、文档缺失库能否启动只列为 future work。

## 8. 对我们的启示 / 可借鉴点
**对 GUI Agent 的可借鉴点**：

1. **把"可操作性先验"搬到 GUI 动作空间**。只按语义相关度把"提交订单"排给 agent，而"填写收货地址"在当前界面不可见，第一步就会失败。可照搬四分法组织 GUI 技能菜单：entry = 当前 DOM/截图中可立即操作的元件，bridge = 能生产后续动作所需字段的动作（展开面板、进详情页），target = 完成主请求，terminal = 提交/确认。
2. **"顺序即接口"值得当一等目标来学**。仅重排已有候选就有 +3.6pt，固定模板也有效；**不做任何训练，把候选动作按"可执行优先、生产者在前"重排就是白捡的收益**。菜单数量也不是瓶颈，路线完整性才是（32 选得好 > 128 官方）——与其把整页可交互元素全塞给模型，不如先算出一条"从当前状态到目标的可执行路径"，只暴露其上的动作；顺序先验还能从脏轨迹里挖（50% 路径损坏只掉 1.0pt）。**但要注意一次性菜单的风险**：GUI 任务常需在线重规划（弹窗、加载失败、意外跳转），合理组合是先用 state-path 先验保证第一步可执行，再保留在线更新应对观测变化。

## 9. 延伸阅读

- **工具检索与排序（主要对手）**：ToolRet（ACL 2025 Findings）、ToolRerank（COLING 2024）、COLT（CIKM 2024，追求覆盖但不排序）、Tool-REX（arXiv:2510.22670）、SkillRouter（CoLM 2026）、ToolGen（ICLR 2025）。
- **依赖感知的在线规划**：Dynamic Tool Dependency Retrieval（ACL 2026 Findings）、AutoTool（AAAI 2026）、ToolTree（ICLR 2026）。
- **基准与环境**：ToolBench / ToolLLM（ICLR 2024）、AppWorld（ACL 2024）、τ-bench（ICLR 2025）、UniToolCall（arXiv:2604.11557）、TRAJECT-Bench（ICLR 2026）、AssistantBench（EMNLP 2024）。

---
*解读生成时间：2026-09-11 09:00 ｜ 解读人：WorkBuddy（AI）*