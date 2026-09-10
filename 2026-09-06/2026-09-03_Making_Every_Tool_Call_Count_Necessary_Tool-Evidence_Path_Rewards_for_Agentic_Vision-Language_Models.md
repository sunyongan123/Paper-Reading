# Making Every Tool Call Count: Necessary Tool-Evidence Path Rewards for Agentic Vision-Language Models

> 一句话 TL;DR：用"必要工具-证据路径"把过程奖励拆成调用前意图对齐＋调用后信息摄取双通道，抑制冗余/脱靶调用，训出更准更省的 agentic VLM。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Making Every Tool Call Count: Necessary Tool-Evidence Path Rewards for Agentic Vision-Language Models |
| **作者 / 机构** | Xingming Long、Yu Liu、Zhiwei Yang（并列一作 \*），Hanqi Feng、Shaojie Zhang 等共 10 人；通讯作者 Pei Fu†、Yu Liu†。机构：① 中科院信息工程研究所（IIE-CAS）；② 卡内基梅隆大学机器学习系（CMU）；③ 小米 MiLM Plus（多数作者挂小米） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-03；arXiv preprint（cs.AI，v1，未标注会议/期刊） |
| **arXiv 链接** | https://arxiv.org/abs/2609.03493 |
| **代码仓库** | ❌ 未开源（正文与附录均无 GitHub/权重链接） |
| **数据集地址** | ⚠️ 未公开 NTEP 标注集；底层训练池来自公开源（FVQA、DeepEyes-4K、Visual-Probe、VDR、Search-R1 文本语料），仅 E 节声明评测基准底层图像数据仅供非商业学术使用 |
| **类型标签（论文类别）** | `RL` `Planning` `General`（Agent 方法论域，非 GUI 落地） |
| **训练方法标签** | `RL (GRPO)` `Distillation`（GRPO 无 critic；教师蒸馏式 NTEP 标注，见 §4.2） |
| **关键词** | Necessary tool-evidence path；Process reward；Evidence-level credit assignment；Agentic VLM tool use；Tool-use efficiency；GRPO |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-03_Making_Every_Tool_Call_Count_Necessary_Tool-Evidence_Path_Rewards_for_Agentic_Vision-Language_Models.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：agentic VLM 通过调用工具（图像裁剪/缩放、以图搜图、文本检索）来补足复杂图像问答所需的细粒度视觉细节或外部知识，训练时如何对**每一次工具调用**做细粒度过程监督。
- **为什么重要**：现有训练范式几乎只按最终答案正确性（或"是否调用了工具"）给奖赏，是一个**不准确的代理信号**——同一条正确轨迹里可能混着有用/冗余/脱靶调用，部分成功的轨迹可能证据拿足却答错。不解决就无法判断"这次调用是否必要、返回证据是否被真正用上"。
- **现有方法有什么不足**：论文点出结果中心监督导致两类结构性缺陷——①**调用前错位**（pre-call misalignment）：发出脱靶或冗余调用，没在追关键证据；②**获取后失败**（post-acquisition failure）：工具返回了有价值上下文，模型却不从中抽取必要信息。搜索域先例（Search-R1、R1-Searcher、StepSearch）与多模态管线（MMSearch-R1、SenseNova-MARS、DeepEyes 等）只研究"能不能完成任务/学不学得到工具行为"，不显式规定每次调用的**证据角色**。
- **Research Gap**：作者认为自己补上的是 **evidence-level credit assignment（证据级信用分配）**——不仅监督"工具被调用了"，还监督"调用意图确实必要、检索信息确实被利用"；并刻意把奖励绑在**证据状态**而非工具身份上，以获得对未见工具的迁移。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

先离线把每个样本的"答案关键步"标注成工具介导的证据转移路径（NTEP），再在 GRPO 训练中用这条路径驱动一个双相过程奖励 NTEP-R：**调用前**奖励意图对齐必要证据，**调用后**奖励从观测中抽到必要信息，同时惩罚重复命中已满足目标的冗余调用。

### 3.2 方法总览（Pipeline）

- **输入**：图像 + 问题 `x = (v, q)`；**输出**：答案 `y`，中间可产生若干工具调用 `c_i = (t_i, a_i)` 与观测 `o_i`。
- **Stage 1（离线，不训模型）**：用目标策略自身 warm-up rollout 作原料，教师模型（Gemini-3.1-Pro）在成功轨迹上删冗余/非关键步、在失败轨迹上补写缺失步，产出样本专属的 `(g_j, t_j, e_j)` 三元组序列（证据目标 → 预期工具 → 必抽信息），路径在 GRPO 前**冻结**。
- **Stage 2（GRPO RL）**：当前策略 rollout 得到轨迹，冻结判官（Qwen3-VL-Plus，temperature 0）对每次交互给两个独立二值判定 α=Align（调用前意图是否对准必要目标）、β=Acquire（调用后推理是否抽出必要信息），沿路径统计命中/脱靶/冗余合成 `R_NTEP`，与答案奖励、格式奖励叠加后按 token 级 group-relative advantage 更新策略（无 critic）。
- **连接**：数据流为"rollout → 判官对 NTEP 打分 → 复合奖励 → GRPO 更新"，NTEP/判官/教师**仅在训练期使用**，推理时策略只见图像、问题与工具接口。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：① NTEP 标注方案本身——只定义证据里程碑、不锁 query/坐标/措辞、且工具无关；② NTEP-R 的**双相独立判定**（Align/Acquire 分开计分，把 credit 定位到"取证意图"与"证据摄取"而非塌缩到末态答案）；③ **非重复目标正则** `D_dup`——只有首次对齐某目标的调用拿对齐分，重复对齐一律受罚；④ token 级 group-relative advantage（无 critic）。
- **工程组合**：GRPO 是现成算法（DeepSeekMath）；冻结判官 + 教师蒸馏标注 + XML 交互协议（`<thinking>/<tool_call>/<tool_response>/<answer>`）均为工程手段，本身不新。
- **对性能提升最关键的设计**：消融证明**信息摄取项（Acquire）**最不可少——去掉它 Search Avg 从 59.44 崩到 34.58、调用从 1.55 涨到 5.36；**非重复目标正则**把冗余从 13.5% 压到 1.0%。
- **证据不足 / 仅声称有效**：wrong-tool 是最大残余失败（14.5%）却未被直接监督（方法只在目标层奖励对齐，工具选择靠泛化）；Answer Reward Only 用 3,859 例 legacy 池、非严格同池消融；单次训练单次评测未报方差；GUI/代码等域外普适性未验证。

---

## 4. 具体技术细节

### 4.1 模型结构

- **base model**：Qwen3-VL-8B-Instruct，是 **MLLM**（多模态大语言模型，含 ViT 视觉编码器 + 8B LLM 主干），非纯 LLM。参数量 8B。训练中 **vision tower 冻结**，仅微调语言/策略部分。判官为 Qwen3-VL-Plus（冻结，仅训练期算奖励），路径教师为 Gemini-3.1-Pro。

### 4.2 训练流程

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| Stage 1：NTEP 标注构建 | 为每样本产出必要证据路径 | 不训练模型（离线标注） | 目标策略自身 warm-up rollout + 教师 Gemini-3.1-Pro | `(g_j, t_j, e_j)` 有序三元组，N 条/样本 | 非梯度：成功轨迹删冗余、失败轨迹补写缺失（extracted 4,819 / completed 2,955 例，历史池 7,774 例；选定 checkpoint 用 search-only 4,855 例） |
| Stage 2：GRPO + NTEP-R | 学会必要取证、抑制冗余/脱靶 | 证据级信用分配（调用前意图 + 调用后摄取） | 同 Stage 1 池；5 源：FVQA/DeepEyes-4K/Visual-Probe/VDR/Search-R1 | 图像+问题+采样轨迹（G=8，batch 128 prompts） | 复合奖励 `R = R_ans + R_fmt + λ·R_NTEP`；`R_NTEP = (1/N)[(1−λ_g)H_info + λ_g·H_goal − λ_g·M_goal − λ_d·D_dup]`，token 级 group-relative advantage、无 critic |

> **tool-evidence path reward 设计是重点**：`H_goal`（去重命中目标数）、`H_info`（抽出信息条数）给正向 credit，`M_goal`（匹配不到任何目标的脱靶数）、`D_dup`（重复命中已满足目标数）给惩罚。关键规则——**只有首次对齐某目标的调用拿对齐分**，再次对齐同一目标一律计入 `D_dup` 受罚。数值记账（Table 5）：info hit +0.7、goal hit +0.3、goal miss −0.3、duplicate −0.1、process scale 0.5，对应 `λ_g=0.3、λ_d=0.1、λ=0.5`。超参：AdamW lr 1e-5、20 epochs、asymmetric clipping (0.20, 0.28)、无 KL 项、entropy 0、32K context/16K response、单样本至多 10 轮交互、每工具响应 8K token 截断。

### 4.3 推理流程

- 使用训练好的 NTEP-8B 策略，**多步循环**（类 ReAct）：`<thinking>` 给意图 → `<tool_call>` 发 JSON 调用 → 环境注入 `<tool_response>` → 下一段 `<thinking>` 整合信息……直到 `<answer>` 给出非空预测即终止；全局预算至多 10 轮、无 per-tool 上限。推理期**不出现** NTEP、教师解释、判官反馈。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模（主协议 matched） | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| MMSearch | General（多模态搜索） | 搜索导向 QA | 171 | 图像+问题 | 自由文本 | Acc（judge 评分） |
| HR-MMSearch | General | 高分辨率搜索 QA | 305 | 图像+问题 | 自由文本 | Acc（judge） |
| InfoSeek | General | 实体信息检索 QA | 500 | 图像+问题 | 自由文本 | Acc（judge） |
| MAT-Search | General | 多跳检索 QA | 150 | 图像+问题 | 自由文本 | Acc（judge） |
| V* Bench | General（细粒度视觉） | 视觉定位/感知 | 191 | 图像+问题 | 多选 | Acc（exact match） |
| HR-Bench 4K | General | 高分辨率视觉感知 | 500 | 图像+问题 | 多选 | Acc（exact match） |
| HR-Bench 8K | General | 高分辨率视觉感知 | 500 | 图像+问题 | 多选 | Acc（exact match） |

> 主对比用 matched 2,317 例；消融与预算扫描用全部 4,417 例。搜索类由共享 VLM 判官评开放答案，视觉多选走确定性选项精确匹配（避免判官污染）。所有 RL-agent 基线经通用 adapter 接入同一三工具后端，保证同口径。

### 5.2 实验结果分析

- **主结果（Table 1）**：NTEP-8B 整体平均 **70.34**，超同框架最强基线 SenseNova-MARS-8B（68.31）**2.03 点**；Search Avg 58.12→60.55、Visual Avg 81.90→83.40。逐项（NTEP-8B vs MARS-8B）：MMSearch 68.4 vs 62.0、HR-MMSearch 35.7 vs 34.1、InfoSeek 56.8 vs 54.4、MAT-Search 81.3 vs 82.0、V* 90.6 vs 92.7、HR-4K 81.2 vs 78.2、HR-8K 78.4 vs 74.8。无工具最强端到端 Gemini-2.5-Pro 搜索类仅 45.92，远低于 NTEP-8B 的 60.55，证明搜索任务必须主动取证。
- **效率（Figure 3）**：相对 MARS-8B，搜索类平均调用 2.54→1.89、视觉类 1.86→1.08，准确率反升——增益来自调用**更精**而非更勤。
- **轨迹/失败模式（Figure 4/5）**：necessary-and-used 调用率 78.8%（MARS 69.0%）；Case-5（每次调用都必要且答对）39.0%→58.7%；冗余 13.5%（w/o reg）→1.0%；wrong-tool 是最大残余（14.5%），提示工具选择监督是互补缺口。
- **Ablation（Figure 7 / Table 6）**：① 只留调用前对齐的 goal-only 变体——Search Avg 崩到 **34.580**、调用 **5.36** 次（image search 高达 3.178），证明"会调工具≠会用证据"；② 仅答案奖励——Overall 69.634 看似接近，但调用 **3.107** 次，隐藏了低效行为；③ 完整 NTEP-R——Search Avg 回 59.438、Overall 69.690，调用压到 **1.549** 次（较前者省约 71%/50%），Visual Avg 基本不动（83.359 vs 83.374），说明信息摄取项不压制必要视觉操作。
- **跨工具 OOD（Figure 6 / Table 10）**：仅用两工具做搜索训练、未见 crop/zoom，评测仅靠接口新增 zoom：Visual Avg **75.01→82.32**（+7.32 pp，V* 增益最大 +9.95 pp），平均调用 1.679→1.294。预算扫描 B∈{5,10,15,20} 时准确率 69.45–69.69、调用 1.545–1.549 几乎不变——调用需求由证据路径而非预算决定。判官仅训练期用，100 例盲审人机一致 **97.0%**（κ=0.936）。

---

## 6. 亮点与贡献（Why it matters）

1. 把"证据获取"拆成**调用前意图对齐 + 调用后信息摄取**两条独立通道，用冻结判官做证据级信用分配，比"答对/答错"的二元结果监督细得多，且视觉多选走 exact match 保证评测独立。
2. **非重复目标正则兼具实测与理论**：冗余 13.5%→1.0%，并给出"罚纯重复、放过 p>1/7 的必要重试"的量化门限，是效率—稳健权衡下少见的校准设计（附录 A.6 三条命题）。
3. **credit 绑证据状态而非工具身份**，换来对未见工具的零训练迁移（OOD 准确率 +7.32 pp 且调用减少），说明学到的是可复用的"取证纪律"而非死记工具清单。
4. 用 goal-only / 仅答案奖励两组对照，把"过程监督在答案之上的增量"单独量化，清晰展示"同分不同行"。
5. 可信度可核：统一 harness + 外部检查点 adapter + 判官盲审 97%/κ≈0.94，训练动态（Figure 9/10）与理论预测自洽。

## 7. 局限与可改进点（个人点评）

- **NTEP 质量上限被教师绑死**：completion 分支给失败轨迹补步比剪枝更难（VDR 82.0% 路径来自补全分支），残余路径噪声只能靠答案奖励兜底；教师一旦把"必要证据"判错，奖励就系统偏差。
- **语义判官粒度偏粗**：Align/Acquire 都是冻结 VLM 的单次二值判断，附录自陈全部分歧落在"近义边界"（contemporary vs modern）这类模糊处，盲审仅 100 例、3 处不一致。
- **wrong-tool 未被直接监督**（14.5% 残余）：方法只在目标层奖励对齐，工具选择本身靠 base model 泛化，作者自认是互补缺口而非已解问题。
- **消融与实验口径瑕疵**：仅答案奖励行用 3,859 例 legacy 池（非严格同池）；单次训练单次评测未报方差；三工具 + 英语检索环境仍偏窄。
- **效率侧重是双刃剑**：非重复目标正则把策略推到效率前沿某一工作点，在检索源不稳、重试确有价值的任务上可能过早收手（λ_d 需按场景调）。

## 8. 对我们的启示 / 可借鉴点

本篇属 Agent 方法论域（非 GUI 落地），其机制对 GUI Agent 高度可迁移，故单列如下：

**对 GUI Agent 的可借鉴点**：GUI Agent 与本文 agent 同患两种病——调用冗余/脱靶（反复全屏截图、无目的滚动、重复点击已确认元素）与证据利用不足（a11y tree/DOM/截图已返回关键状态却视而不见）。
- **观察动作标成 NTEP 三元组**：`g_j`=此刻需要的界面证据（按钮可达性、tab 选中态），`t_j`=取证通道（截屏/OCR/a11y tree/滚动/局部裁剪），`e_j`=必抽的状态或字段值，不锁坐标与查询措辞，兼容多平台观测差异。
- **用路径奖励替代纯结果奖励**：对每次界面操作做 α（调用前是否在追必要证据而非泛泛"看一眼"）与 β（返回后是否引用证据关键信息）双判定，直接惩罚"截了图却没读"这类 GUI 高发废步。
- **非重复目标正则＝防抖防循环**：已满足目标再被"取证式"动作命中即计冗余受罚，从奖励端根治重复点击/同屏反复滚动；门限可调对应"该重试 vs 空转"。
- **三个迁移难点**：①像素坐标难语言化——证据目标抽象到"控件语义状态"而非坐标；②GUI 证据多为像素、判官成本高——先对 a11y tree/OCR/URL 等文本化观测做 β 判定，像素判官作二期；③grounding 点错元素≈本文 wrong-tool，路径奖励之外还需显式"操作/工具选择"监督。
- **最值得抄的两点**：一是"教师从**目标策略自家 rollout**提炼＋补齐路径"的数据工厂思路，可低成本量产 GUI 证据路径做奖励监督；二是 **credit 绑证据目标而非交互原语**，本文实证据此让未见工具零训练激活，GUI 侧新增手势、新组件 API 或可同理免训迁移。

## 9. 延伸阅读

- **文本域检索 RL**：Search-R1、R1-Searcher/R1-Searcher++、StepSearch、WebThinker。
- **多模态搜索/视觉操作 agent**：MMSearch-R1、SenseNova-MARS（同框架最强基线）、MM-DeepResearch、WebWatcher、DeepEyes/DeepEyesV2、Vision-R1、Visual-ARFT、PixelReasoner、Thyme-RL、Chain-of-Focus、V-Thinker。
- **过程/工具奖励方法**：ToolRL、RLTR（无正确结果也奖好过程）、Atom-Searcher、TA-MDP（LVLM 奖励分解）；GRPO 源头见 DeepSeekMath（Shao et al. 2024）。
- **评测基准**：MMSearch、InfoSeek、V* Bench、HR-Bench 4K/8K；同日同源 GRPO 机制研究（Headroom-Drift Replay、Spurious Advantage）可对照阅读。

---
*解读生成时间：2026-09-10 13:10 ｜ 解读人：WorkBuddy（AI）*
