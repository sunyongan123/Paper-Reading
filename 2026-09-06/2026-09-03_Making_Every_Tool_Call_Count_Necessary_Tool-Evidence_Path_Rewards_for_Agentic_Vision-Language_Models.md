# Making Every Tool Call Count: Necessary Tool-Evidence Path Rewards for Agentic Vision-Language Models

> TL;DR：用"证据路径"级过程奖励逐次评估工具调用，抑制冗余与脱靶，训出更准更省的 agentic VLM。

---
## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | Making Every Tool Call Count: Necessary Tool-Evidence Path Rewards for Agentic Vision-Language Models |
| **arXiv ID / DOI** | arXiv:2609.03493v1（cs.AI） |
| **arXiv 链接** | https://arxiv.org/abs/2609.03493（点击直达） |
| **发表出处（Venue）** | arXiv preprint（预印本，未标注会议/期刊） |
| **发布时间** | 2026-09-03 |
| **作者** | Xingming Long、Yu Liu、Zhiwei Yang（并列一作）等 10 位；通讯作者 Pei Fu、Yu Liu（上下标：CAS-IEE ¹、CMU ²、Xiaomi ³，多数作者挂 Xiaomi） |
| **所属机构** | ①中国科学院信息工程研究所（Institute of Information Engineering, CAS）；②卡内基梅隆大学机器学习系（Dept. of Machine Learning, CMU）；③小米 MiLM Plus（MiLM Plus, Xiaomi Inc.） |
| **开源情况 / 代码** | 未在文中声明开源（正文与附录均无代码/GitHub/权重链接；仅 E 节声明评测基准底层图像数据仅供非商业学术使用） |
| **类型标签（论文类别）** | `RL` `Planning` `General`（Agent 方法论域，跨域参考，非 GUI 落地） |
| **训练方法标签** | `RL (GRPO + NTEP 过程奖励)`（token 级 group-relative advantage、无 critic；含两阶段教师蒸馏式 NTEP 标注） |
| **关键词** | Necessary tool-evidence path; Process reward; Agentic VLM tool use; Evidence acquisition; Tool-use efficiency; GRPO |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-03_Making_Every_Tool_Call_Count_Necessary_Tool-Evidence_Path_Rewards_for_Agentic_Vision-Language_Models.pdf |

---
## 1. 研究背景与要解决的问题

现代 VLM 已能回答不少图像问题，但复杂查询常依赖细粒度视觉细节、陌生实体或外部知识，于是 agentic VLM 学会主动调用工具——本文框架含图像裁剪/缩放、以图搜图、文本检索三类。问题出在训练信号：现有范式基本只按最终答案是否正确给奖赏，以结果为中心的监督无法回答"这次调用是否必要、返回证据是否被真正用上"。

作者据此点出两类失败：①**调用前错位**——发出脱靶或冗余的调用，没在追关键证据；②**获取后失败**——工具返回了有价值上下文，模型却不从中抽取必要信息。答案相同的轨迹可混着有用/冗余/脱靶调用，部分成功的轨迹也可能拿足证据却答错——末态正确性无从做过程信用分配。核心问题即 **evidence-level credit assignment**：不仅监督"工具被调用了"，还监督"调用意图确实必要、检索信息确实被利用"。

## 2. 核心方法 / 思路

整体分两阶段：先离线构建**样本专属的 NTEP 标注**，再在 GRPO 训练中用其驱动**路径级过程奖励（NTEP-R）**。

**NTEP 标注。** 每个答案关键步被建模为"工具介导的证据转移" `g_j —t_j→ e_j`：证据目标 g_j、预期工具 t_j、必须从返回观测抽取的信息 e_j，路径即这些转移的有序集合。刻意只定义证据里程碑而不锁死 query/坐标/措辞，且与工具无关。标注不靠教师直接示范，而是从**目标策略自身的 warm-up rollout** 提炼：教师模型（Gemini-3.1-Pro）在成功轨迹上删冗余/非关键步，在失败轨迹上补写缺失转移。7,774 例历史池含抽取 4,819 例、补全 2,955 例，来自 FVQA、DeepEyes-4K、Visual-Probe、VDR 等公开源；路径训练前冻结。

**NTEP-R 过程奖励。** 对每次工具交互，冻结判官（Qwen3-VL-Plus，temperature 0）给出两个**独立二值判定**：α＝Align（调用前意图与工具调用是否对准某必要证据目标）、β＝Acquire（调用后推理是否捕获必要信息）。据此沿路径统计：命中的目标数 H_goal、抽得的信息条数 H_info、匹配不到目标的脱靶数 M_goal、重复命中**已满足目标**的冗余数 D_dup，合成 R_NTEP＝1/N·[(1−λ_g)H_info＋λ_g·H_goal − λ_g·M_goal − λ_d·D_dup]。关键规则：**只有首次命中某目标的调用能拿对齐分**，再次对齐同一目标一律计入 D_dup 受罚。总奖励 R＝R_ans＋R_fmt＋λ·R_NTEP，答案奖励锚定任务成功；GRPO 的组内相对优势改按 token 级计算、不加 critic。

配套理论（附录 A.6）：脱靶/冗余调用带独立加价，但重试未决目标在成功率 p＞λ_d/(1−λ_g)＝1/7 时仍划算（罚纯重复、不罚必要重试）；目标全满足后最优策略立即作答；credit 绑定**证据状态而非工具身份**，价值随工具清单单调不减——为跨工具迁移奠基。

## 3. 关键实验结果

**评测设置。** 七个图像类基准：MMSearch、HR-MMSearch、InfoSeek、MAT-Search（搜索导向）与 V* Bench、HR-Bench 4K/8K（细粒度视觉）。NTEP-8B 从 Qwen3-VL-8B-Instruct 初始化；外部 RL-agent 检查点经通用 adapter 接入同一三工具后端，保证同口径。

**主结果（Table 1）。** NTEP-8B 平均 70.34，超同框架最强基线 SenseNova-MARS-8B（68.31）**2.03 点**：Search Avg 58.12→60.55、Visual Avg 81.90→83.40。搜索类最强无工具端到端 Gemini-2.5-Pro 仅 45.92。

**工具效率（Figure 3）。** 相对 MARS-8B，平均调用次数搜索类 2.54→1.89、视觉类 1.86→1.08，准确率反升：增益来自调用更精而非更勤。

**轨迹审计（Figure 4/5）。** necessary-and-used 调用率 78.8%（MARS-8B 69.0%）；"每次调用必要且最终答对"的 Case-5 从 39.0% 升到 58.7%；冗余从去掉非重复目标正则的 13.5% 压到 1.0%；wrong-tool 仍是最大残余失败（14.5%）。

**奖励消融（Figure 7 / Table 6）。** 只留调用前对齐的 goal-only 变体：Search Avg 崩至 34.58、每样本调用 5.36 次——"会调工具≠会用证据"；仅答案奖励 3.11 次、Overall 69.63；完整 NTEP-R 调用降至 **1.55 次**（较前两者约省 71%/50%），Search Avg 回到 59.44，Visual Avg 几乎不动，说明信息摄取项不压制必要视觉操作。（仅答案奖励行用较小 legacy 池 3,859 例，作者自注非严格同池消融。）

**跨工具 OOD（Figure 6 / Table 10）。** 仅用两工具做搜索训练、完全未见 crop/zoom，评测时仅靠接口新增该未见工具：Visual Avg 75.01→82.32（V* 增益最大 +9.95 pp），平均调用 1.679→1.294。预算扫描 B＝5~20 时准确率与调用几乎不变（69.45–69.69；1.545–1.549），调用需求由证据路径而非预算决定。判官仅在训练期使用，100 例盲审人机一致 97.0%（κ≈0.94）。

## 4. 亮点与贡献（Why it matters）

1. **把"证据获取"拆成调用前意图对齐＋调用后信息摄取两条独立通道**，以冻结判官做证据级信用分配，比"答对/答错"的二元结果监督细得多。
2. **非重复目标正则兼具实测与理论**：冗余 13.5%→1.0%，并给出"罚纯重复、放过 p>1/7 的必要重试"的量化门限，是效率—稳健权衡下少见的校准设计。
3. **credit 绑证据状态而非工具身份**，换来对未见工具的零训练迁移（OOD 准确率 +7.3 pp 且调用减少），说明学到的是可复用的取证纪律。
4. **把"过程监督在答案之上的增量"单独量化**：goal-only 与仅答案奖励两组诊断清楚展示"同分不同行"，是干净的对照实验范式。
5. **可信度可核**：统一 harness＋外部检查点 adapter＋判官盲审 97%/κ≈0.94。

## 5. 局限与可改进点（个人点评）

- **NTEP 质量上限被教师绑死**：completion 分支给失败轨迹补步比剪枝更难，残余路径噪声只靠答案奖励兜底；教师一旦把"必要证据"判错，奖励信号就系统偏差。
- **语义判官粒度偏粗**：Align/Acquire 都是冻结 VLM 的单次二值判断，附录自陈分歧全落在"近义边界"（如 contemporary vs modern）这类模糊处，且盲审样本仅 100 例。
- **wrong-tool 是最大残余却未被直接监督**（14.5%）：方法只在目标层奖励对齐，工具选择本身靠模型泛化——作者承认这是互补缺口而非已解问题。
- **消融与场景局限**：仅答案奖励行用不同规模历史池，只能看趋势；单次训练单次评测未报方差；三工具＋英语检索为主的环境仍偏窄。
- **效率侧重是双刃剑**：非重复目标正则把策略推向效率前沿的某一工作点，在检索源不稳、重试确有价值的任务上可能过早收手（λ_d 需按场景调）。
- **域外普适性未验证**：全部实验限于图像问答＋三类检索/缩放取证，泛化到 GUI、代码、数据库等动作域仍是开放问题。

## 6. 对 GUI Agent 的可借鉴点（跨域参考专节）

GUI Agent 与本文 agent 同患两种病：一是调用冗余/脱靶——反复全屏截图、无目的滚动、重复点击已确认元素；二是证据利用不足——a11y tree/DOM/截图已返回关键状态（弹窗文案、选中态等），agent 却视而不见。NTEP-R 的机制几乎可逐条平移：

- **观察动作也标成 NTEP 三元组**：g_j＝此刻需要的界面证据（如按钮可达性、某 tab 选中态），t_j＝取证通道（截屏/OCR/a11y tree/滚动/局部裁剪），e_j＝必须抽出的状态或字段值。不锁坐标与查询措辞，兼容不同 GUI 平台观测差异。
- **RL 用路径奖励替代纯结果奖励**：对每次界面操作做 α（调用前思考是否在追必要证据，而非泛泛"看一眼界面"）与 β（操作返回后下一句推理是否引用证据里的关键信息）双判定——直接惩罚"截了图却没读"这类 GUI 高发废步。
- **非重复目标正则＝防抖与防循环**：已满足目标再被"取证式"动作重复命中即计冗余受罚，从奖励端根治重复点击、同屏反复滚动；门限可调，正对应"失败该重试 vs 空转"的权衡。
- **三个迁移难点与应对**：①像素级坐标难语言化——证据目标宜抽象到"控件语义状态"而非坐标；②GUI 证据多为像素、判官成本高——可先对 a11y tree、OCR、URL 等文本化观测做 β 判定，像素判官作二期；③grounding 点错元素≈本文 wrong-tool（最大残余 14.5%），路径奖励之外还需显式的"操作/工具选择"监督。
- **最值得抄的两点**：一是"教师从**目标策略自家 rollout** 提炼＋补齐路径"的数据工厂思路——GUI grounding 标注昂贵，可借此低成本量产"界面证据路径"用于奖励监督；二是 **credit 绑证据目标而非交互原语**——本文实证据此可让未见工具零训练激活，GUI 侧新增手势、新组件 API 或可同理免训迁移。

## 7. 延伸阅读

- **文本域检索 RL**：Search-R1、R1-Searcher/R1-Searcher++、StepSearch、WebThinker——本文文本检索通道的直接先例。
- **多模态搜索/视觉操作 agent**：MMSearch-R1、SenseNova-MARS（同框架最强基线）、MM-DeepResearch、WebWatcher、DeepEyes/DeepEyesV2、Vision-R1、Visual-ARFT、PixelReasoner、Thyme-RL、Chain-of-Focus、V-Thinker。
- **过程/工具奖励方法**：ToolRL、RLTR（无正确结果也奖好过程）、Atom-Searcher、TA-MDP（LVLM 奖励分解）；GRPO 源头见 DeepSeekMath（Shao et al. 2024）。
- **评测基准**：MMSearch、InfoSeek、V* Bench、HR-Bench 4K/8K；同日同源 GRPO 机制研究（Headroom-Drift Replay、Spurious Advantage）可对照阅读。

---
*解读生成时间：2026-09-06 09:45 ｜ 解读人：WorkBuddy（AI）*
