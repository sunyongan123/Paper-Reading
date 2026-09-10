# Co-Evolving Harnesses and Models: On-Policy Correction Helps Weaker Models Catch Up Where Imitation Fails

> 一句话 TL;DR：在已演化的 harness 下整轨模仿专家会让弱模型全面倒退，改用 on-policy 专家定点纠错才能保住「模型—harness 契合」并叠加收益。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Co-Evolving Harnesses and Models: On-Policy Correction Helps Weaker Models Catch Up Where Imitation Fails |
| **作者 / 机构** | Zhou Yu\*（通讯作者，zhou.yu@salesforce.com）、Bin Bi、Shiva Kumar Pentyala、Shubham Mehrotra、Sougata Chaudhuri、Shilpa Bhagavath、Zeyuan Chen、Ran Xu、Phil Mui、James Zhu、Sitaram Asur（共 11 人）；Salesforce AI（美国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-08；arXiv preprint（arXiv:2609.09134v1 [cs.AI]） |
| **arXiv 链接** | https://arxiv.org/abs/2609.09134 |
| **代码仓库** | ❌ 未开源（正文无 GitHub 链接或代码可用性声明；实验复用 Yang et al. 2026 的 harness 优化代码库） |
| **数据集地址** | ⚠️ 未公开（任务套件/环境沿用 Yang et al. 2026「Better Harnesses, Smaller Models」，7 个企业 agent 任务，正文未给独立数据地址） |
| **类型标签（论文类别）** | `Online` `SFT` `Reflection` |
| **训练方法标签** | `LoRA-SFT`（两类数据：专家整轨模仿 SFT-imit、on-policy 定点纠错 SFT-corr） |
| **关键词** | Agent Harness；Harness Evolution；Co-Evolution；On-Policy Correction；LoRA；Enterprise Agent |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_Co-Evolving_Harnesses_and_Models_On-Policy_Correction_Helps_Weaker_Models_Catch_Up_Where_Imitation_Fails.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：一个 agent 的能力由「模型权重」和包在外面的「harness」（系统提示词、工具集、执行钩子、上下文管理脚手架）共同决定。harness 自动演化已能把便宜小模型抬到接近前沿水平，但 harness 只是两根可调杠杆之一，另一根是权重微调（LoRA）。本文要回答：在低成本约束下，harness 演化与轻量微调这两根杠杆该如何组合、能否「协同演化」而不互相干扰。
- **为什么重要**：harness 一旦为某模型调优，就会与该模型的规划/执行行为「耦合」。若后续权重更新破坏了这种耦合，harness 的收益会被抹掉。不解决这一点，企业级低成本 agent 的「先演化、再微调」流水线会系统性倒退，两根杠杆无法叠加。
- **现有方法有什么不足**：作者明确批评的基线范式是「整轨专家轨迹模仿」（SFT on expert trajectories）。它有两个结构性缺陷：(1) 数据取自专家分布，而非学生自己访问过的状态（off-policy）；(2) 它把专家整套「规划风格」一并迁移——弱模型学到了知识却学不来执行它的能力，于是与「围绕其原生规划风格打磨出的 harness」失配。同期协同演化工作（SIA、Co-Harness、HarnessForge、HarnessX）虽各自报涨，但都未回答「一旦 harness 已专门化为某模型，后续权重更新在什么条件下才会叠加而非破坏契合」。
- **Research Gap**：本文自认补上的缺口是——把「on-policy 教师纠错」这一支线（Lauffer 2025、Ye 2026）与「harness 演化」这一支线首次对接：证明整轨模仿会破坏模型—harness 契合，而「只在学生访问过的状态上做局部专家改写」能保住契合，使两根杠杆真正可叠加、可进入迭代协同演化循环。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

> 先为弱模型演化 harness，再从小模型**自己**在该 harness 下的 rollout 出发，由元级 MLE agent 定位失败回合、让专家只改写那一个回合，用这份最小编辑数据做 LoRA-SFT——既注入专家知识、又不扰动学生原生规划节奏。

### 3.2 方法总览（Pipeline）

- **输入**：7 个企业任务（工资审计、预算审批、库存告警、IoT 异常检测、浏览器自动化、网站管理、代码重构）的环境 + 一个弱模型 + 一个强专家模型。
- **输出**：一套演化后的 harness `h*` 和一个在其上进一步涨点的弱模型 `Mt+1`。
- **模块**：
  1. **Harness 演化**（GEPA 式搜索）：以 `gemini-3.1-pro-preview` 为反思元智能体，针对失败提出 harness 编辑（改系统提示、增删工具/钩子），只保留验证集确有提升的候选；弱模型为执行器。每任务 3 个随机种子、预算 $20、单 rollout 600s 限时。
  2. **向上迁移观测**：把专家放进为弱模型演化的 harness，确认其用得更好（暴露可榨取的能力缺口）。
  3. **失败归因**：用 Yang et al. 的六类失败本体（LLM-as-judge）把失败拆成「知识类 vs 规划类」。
  4. **On-policy 定点纠错**（核心）：从弱模型在 `h*` 下的失败 rollout 出发 → 元级 MLE agent 定位出错的那一回合 → 专家只就地改写该回合（一句纠错策略 + 修正后的工具调用）→ N=3 择优 → 与约 50 条自身成功样本混合（约 500 行）→ LoRA-SFT。
- **连接**：harness 演化与微调是**串行**的（先演化、后纠错微调），但作者主张把纠错微调后的模型重新送入下一轮 harness 演化，构成迭代闭环（Figure 1）。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：把「harness 已专门化」这一前提引入教师信号设计——发现并命名「模型—harness 契合」（model–harness fit）破坏这一失败模式；用「失败回合定位 + 专家单回合就地改写」在 harness 演化场景下首次让两杠杆叠加。
- **工程组合**：harness 演化（GEPA）、LoRA-SFT、on-policy 纠错、LLM-as-judge 失败分类，均为已有组件；本文贡献在于把它们拼进「协同演化」并给出失效条件。
- **对性能提升最关键的设计**：失败归因（知识类 46.2%→43.2% 下降、规划类 1.1%→1.8% 几乎不动）是纠错有效的直接证据；「只改写一个回合」是保住规划分布的关键动作。
- **证据不足 / 仅声称有效**：「可迭代协同演化循环」只在 Figure 1 示意，未给出多轮交替的实测；纠错收益量级小（+1.7），且未与 RL 或更大规模微调对比。

---

## 4. 具体技术细节

### 4.1 模型结构

- 弱模型（学生）：`Qwen3-Coder-30B-A3B-Instruct`（总 30B / 激活 3B，MoE）为主实验；`gemma-4-26b-a4b-it`（26B/4B）做跨家族复现。
- 专家（教师）：`gemini-3.1-pro-preview`（闭源 API）。
- 均为纯 **LLM**（工具调用/文本，无视觉编码器），非 MLLM；harness 演化阶段弱模型冻结，微调阶段用 **LoRA** 只训 adapter（rank 16 或 64 取优）。

### 4.2 训练流程（on-policy correction 是重点）

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| Stage 0：Harness 演化 | 提升弱模型在任务上的成功率 | 不改权重，改 harness | GEPA 反思搜索，gemini 提出、弱模型执行 | harness 编辑（提示词/工具/钩子），只保留验证集涨点候选 | 无（验证集成功率作为筛选信号） |
| Stage 1：SFT-imit（失败对照臂） | 验证整轨模仿 | 领域知识 + 专家规划风格 | 专家在 `h*` 下的成功轨迹（score 1.0）+ 弱模型自身成功轨迹 | 转成 Qwen chat/tool-call 格式的完整轨迹 | 标准 SFT（next-token / 工具调用），LoRA |
| Stage 2：SFT-corr（主方法） | 在不破坏契合下注入专家知识 | 领域知识（定点），保留原生规划 | 弱模型在 `h*` 下的**失败 rollout**（约 400 条，专家单回合改写）+ 自身成功样本（约 50 条） | 最小编辑轨迹：其余回合原样、仅错回合被专家改写（N=3 择优） | LoRA-SFT |

> 具体设定（附录 B）：LoRA rank 16/64、2 epochs、lr 1e-4、bf16、有效 batch 8、序列长 49152（若 >5% 数据截断则加训 98304 长上下文）；adapter 合并成 dense checkpoint（约 57–61GB），双 H200 张量并行，**单次训练 <1 小时**。

### 4.3 推理流程

- **多步推理**：agent 在 harness 内循环——观察环境/上下文 → 生成工具调用 → 环境返回结果 → 直到命中 `finish` 工具或超时/超步上限；评分是**二元**（0/1），成功轨迹 score=1.0。
- 输入：任务指令 + 环境状态 + 可用工具；输出：逐步工具调用序列，最终提交结构化结果。演化 harness 通过外化领域配方（如工资审计的「先按员工聚合→部门均值→封顶后乘→固定舍入」）和「算完核对一次再收尾」的节奏来拟合弱模型原生规划。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| Payroll Auditing（工资审计） | General/企业 | 长程数据审计 | 未披露（沿用 Yang et al. 套件） | 文本 + 内部 API/MCP 工具 | 工具调用序列 + 提交 | 任务成功率（binary，3 runs 均值±SEM） |
| Budget Approval（预算审批） | General/企业 | 审批流程 | 未披露 | 文本 + 工具 | 同上 | 同上 |
| Stock Alerting（库存告警） | General/企业 | 枚举+告警 | 未披露 | 文本 + 工具 | 同上 | 同上 |
| IoT Anomaly Detection（异常检测） | General/企业 | 查询-检测-上报 | 未披露 | 文本 + 工具 | 同上 | 同上 |
| Browser Automation（浏览器自动化） | Web | 端到端浏览器操作（Playwright） | 未披露 | 文本 + 浏览器工具 | 同上 | 同上 |
| Website Management（网站管理） | Web | 管理后台操作 | 未披露 | 文本 + 工具 | 同上 | 同上 |
| Code Refactoring（代码重构） | General/工程 | 重构 | 未披露 | 文本 + 工具 | 同上 | 同上 |

### 5.2 实验结果分析

- **Harness 演化主结果**：弱模型 Qwen 平均成功率 29.2% → 78.0%（**+48.8**，7 任务全涨）。
- **向上迁移**：专家在 `h*` 下 93.6%（自身基础 harness 84.4%，**+9.2**），且比弱模型高 **+15.6**；专家真实触发各演化组件（93.6%–100% 的 rollout 触发，领域计算配方 94.2% vs 弱模型仅 30.8%）。
- **整轨模仿倒退（SFT-imit）**：78.0% → 63.1%（**−14.9**），**7 任务全部倒退**（−4.2 ~ −29.9；最大跌幅工资审计 −29.9、网站管理 −20.3、浏览器自动化 −16.0）。同配方在**基础 harness** 下反而 +6.3（29.2%→35.5%）。换 Gemma 在 Webarena 复现：harness 演化 46.7%→55.6%，模仿后跌到 41.1%（比演化基线低 14.5、比默认基线还低 5.6）。
- **Ablation / 归因说明什么**：
  - Ablation I（基础 harness 对照）证明：倒退不是「模仿本身」的锅，而是「模仿 × 演化 harness」的交互。
  - 失败构成（表 3）：`h*` 下模仿使规划类失败 1.1%→14.6%（+13.5），知识类 46.2%→44.5%（−1.7）——**知识在涨、规划在崩**；领域配方使用率 30.8%→76.1%，说明模型确实学会了 harness 组件，但学走了专家规划风格却无力执行。个案：78 步里重复查同一视图 29 次、始终不 finish，答案在手仍得 0 分。
  - on-policy 纠错（SFT-corr）：78.0% → 79.7%（**+1.7**），5/7 任务上涨（网站 +5.6、库存 +2.2、重构 +2.2、异常 +1.7、浏览器 +1.2），2 个饱和任务持平（预算 −0.6、工资 −0.4）；规划类仅 1.1%→1.8%（+0.7），知识类降至 43.2%（−3.0），总失败率 28.9%→26.8%。

---

## 6. 亮点与贡献（Why it matters）

1. **实证一个反直觉现象**：在已贴合小模型的演化 harness 下，用更强专家轨迹做标准 SFT 会系统性倒退（7 任务全跌 −4~−30），且跨 Qwen/Gemma 两个模型家族复现——给「先蒸馏后微调」类流水线敲了警钟。
2. **归因清晰、可迁移**：失败是「规划风格漂移破坏模型—harness 契合」而非没学到知识；「知识类 vs 规划类失败占比」是可落地的诊断指标。
3. **低门槛配方**：失败回合定位 + 专家单回合改写 + LoRA，约 500 行数据、训练 <1 小时、双 H200 即可。
4. **提出设计原则**：harness 一旦专门化，后续权重更新应「保持契合」而非无意破坏；教师信号必须落在学生访问过的状态上（on-policy），并支持迭代协同演化。

---

## 7. 局限与可改进点（个人点评）

- **评测面窄**：仅 7 个企业任务（浏览器/Web 类只有 2 个），数据规模未披露，结论外推需谨慎；绝对数字因环境改动与 Yang et al. 不完全可比。
- **强弱配对单一**：弱模型限 Qwen/Gemma、教师固定 Gemini，未验证开源强模型教师是否改变结论；Gemma 复现仅 Webarena 一个任务。
- **收益量级小、上限仍在**：纠错后 79.7% 距专家 93.6% 仍差近 14 点，作者自认轻量 LoRA 容量受限，且**未与 RL（GRPO/PPO）或更大规模微调对比**——「能否继续逼近专家」悬而未决。
- **流程串行 + 依赖 LLM-as-judge**：harness 演化与微调分步而非联合优化；失败定位、N=3 择优都靠 LLM 判断，噪声未量化；「可迭代协同演化闭环」只有示意图、无多轮实测。
- **可复现性差**：未开源；只报 3 次运行均值±SEM，无显著性检验。

---

## 8. 对我们的启示 / 可借鉴点

最有价值的是一条纪律而非具体数字：**教师信号应落在学生自己访问过的状态上，并持续监控「规划 vs 知识」两类失败的此消彼长**。映射到 RL/Post-training 流程：「整轨模仿」对应把奖励/轨迹全部取自离线专家、忽视学生自身分布的做法；on-policy 定点纠错则与「带局部专家修正的在线训练」同源，数据效率极高，适合作为正式 RL 前的预热。「失败回合定位 + 单步改写」这套由元 agent 自动化的数据合成管线，比整轨清洗便宜得多，还能保留学生自身风格。

**对 GUI Agent 的可借鉴点**：GUI Agent 的 harness 通常由系统提示、屏幕/无障碍树读取工具、点击前校验 hook、上下文压缩策略组成。若你的方案是「小模型 + 为它定制的 GUI harness + 用 GPT-4o/Gemini 完整操作轨迹做 LoRA」，本文是明确警告：大模型轨迹里藏着的「操作节奏」（何时截图确认、滚动粒度、是否校验元素可见性、何时收尾提交）未必是小模型 + 该 harness 承受得起的——知识学来了，行为却水土不服，典型症状是「知道答案却永远不点提交」或跳过多步校验。更稳的做法是把 GUI 场景也跑成协同演化闭环：让 agent 在目标站点自我 rollout，用失败反思演化 harness（补工具、加「点击前校验可点击」hook、外化站点约定）；只在它自身轨迹出错的那个回合（表单校验失败、元素找不到后死循环重试、漏关弹窗）让强模型就地改写，再做 LoRA，多轮交替。本文的 planning/knowledge 失败归因也可作为 GUI 微调后掉点的诊断手段。

---

## 9. 延伸阅读

- Yang et al., 2026. *Better Harnesses, Smaller Models*（arXiv:2607.08938）——任务套件与 harness 演化框架来源。
- Agrawal et al., 2025. *GEPA: Reflective Prompt Evolution*（arXiv:2507.19457）——反思式 harness 演化算法。
- Lauffer et al., 2025（arXiv:2512.14895）、Ye et al., 2026（arXiv:2607.04574）——on-policy 教师纠错。
- Xu et al., 2026. *Adapting the Interface, Not the Model*（arXiv:2605.22166）——harness 向上迁移。
- Co-Harness（2607.22688）、SIA（2605.27276）、HarnessForge（2606.01779）、HarnessX（2606.14249）——协同演化同期工作。

---
*解读生成时间：2026-09-09 ｜ 解读人：WorkBuddy（AI）*
