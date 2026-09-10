# Efficient GUI Agents: A Systems Survey of Observation, Memory, Action, and Runtime Optimization

> 一句话 TL;DR：把「能不能完成任务」和「完成任务花了多少代价」拆开，用观察/记忆/动作/运行时四轴对 GUI Agent 效率优化做端到端系统化梳理的一篇综述。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Efficient GUI Agents: A Systems Survey of Observation, Memory, Action, and Runtime Optimization |
| **作者 / 机构** | Bizhe Bai†、Jiakang Yuan†（共同一作）、Hongming Wu、Xinyue Wang、Jie Ren、Siyao Chen、Yuchen Ya、Fan Bai、Pai Peng、Huafeng Qin、Tao Chen。复旦大学未来信息技术学院 + 上海创新研究院（上海）为主；Fan Bai、Pai Peng 为独立研究者；Huafeng Qin 属重庆工商大学。未标注通讯作者 |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-02；arXiv preprint（arXiv comment 标注 "Accept at Grounding Language Models: Learning Faithfully and Efficiently @ EMNLP 2026"） |
| **arXiv 链接** | https://arxiv.org/abs/2609.02309 |
| **代码仓库** | ❌ 未开源（正文无代码/数据发布声明；附录表格中的 "GitHub" 列均指被引论文的仓库，非本文资源） |
| **数据集地址** | 未公开（本文为综述，不发布数据） |
| **类型标签（论文类别）** | `General` `Web` `Mobile` `Desktop` |
| **训练方法标签** | —（系统综述） |
| **关键词** | GUI Agent、系统效率、观察压缩、记忆管理、动作抽象、运行时优化 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-02_Efficient_GUI_Agents_A_Systems_Survey_of_Observation_Memory_Action_and_Runtime_Optimization.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：GUI Agent 从「能做」走向「能用」的过程中，**效率成为部署瓶颈**，但领域仍主要用 task success 报进展，缺少一张「开销到底花在哪、业界怎么省」的系统地图。这篇综述要补的缺口，就是**GUI Agent 效率优化的系统化梳理**。

- **为什么重要**：成功率是单指标，会掩盖低效——一个 Agent 最终完成任务，却可能跑几十轮「观察→推理→动作」循环、反复调用模型、让用户在每步之间干等。OSWorld-Human 实测显示，领先的 computer-use Agent 完成任务所需步数是人类轨迹的 **2.7×–4.3×**，且规划、判断、反思占了端到端延迟的大头（**反思单独就占 76%–96%**）。GUI 交互天然多模态、观察密集（长 DOM/无障碍树、视觉密集小目标），效率直接决定可部署性、隐私（减少外传 UI 内容）与用户体验——「做对了但太慢/太贵/上下文太重」的 Agent 照样上不了生产。

- **现有方法有什么不足**：已有三篇综述（Yang et al., 2026b 高效 Agent；Sager et al., 2025 computer-use Agent；Nguyen et al., 2024 GUI Agent）都是**按「能力 / 架构」组织文献**，效率只是附带提及；而零散的效率工作各自为政、基线各异、数字不可比，缺乏统一的核算框架。

- **Research Gap**：作者声称自己是**第一个把「效率」当作一等公民**、用端到端系统视角组织的 GUI Agent 综述——不仅分类，还追问每个机制「省下的钱转移到哪里去了」（parser / verifier / retriever / orchestration 这些隐性第二成本），这是与既往「罗列 token 节省」式综述的本质区别。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

不训练任何模型，而是把 GUI Agent 抽象成一个闭环 `a_t ~ π(g, o_t, m_t)`，沿观察、记忆、动作、规划四环建立**效率分类学（taxonomy）**，并逐条追踪每个机制的「收益信号 + 新增开销」。

### 3.2 方法总览（Taxonomy）

- **输入/输出**：输入是「目标 g + 当前界面观察 o_t + 累积记忆 m_t」，输出是下一步界面级动作 a_t（点击/输入/滚动/拖动/快捷键/切换应用）。综述把一次完整 episode 抽象为系统级轨迹 `τ = (s_0, o_0, m_0, a_0, v_0, ...)`，其中 Ω（感知）、Um（记忆更新）、πθ（规划/grounding）、T（环境转移）、V（验证/反思）五模块串起数据流。
- **四大效率轴 + 代表方法**（对应正文 Section 3–6）：

| 效率维度 | 子类 | 代表方法 |
|---|---|---|
| **Observation**（观察效率，Sec.3） | 文本缩减 | Agent-E、Beyond Pixels、LineRetriever、FocusAgent、Prune4Web |
| | 区域聚焦视觉 | SeeClick、Ferret-UI、R-VLM、RegionFocus、DiMo-GUI、ShowUI、ScreenSpot-Pro、SimpAgent |
| | 解析增强/混合观察 | Set-of-Mark、ScreenAI、OmniParser、Tree-of-Lens、GUI-Actor、Agent-S、UGround、Aria-UI、Aguvis、UI-TARS |
| **Context & Memory**（上下文/记忆，Sec.4） | 摘要压缩/选择性回看 | Agent-S、ColorBrowserAgent、GUI-Rise、PAL-UI、HiconAgent、SimpAgent、Read More Think More |
| | 运行时表示压缩 | GUI-KV、ST-Lite、Continuous Memory、SecAgent |
| **Action**（动作，Sec.5） | 动作抽象 | SkillWeaver、PolySkill、Mobile-Agent-E、ActionEngine、CoAct-1 |
| | 剪枝/验证/恢复 | Prune4Web、V-Droid、VeriSafe Agent、GUI-Shepherd、SenseAct、BacktrackAgent、LongHorizonUI |
| | 探索控制 | LASER、Auto-Intent、OpenWebVoyager、GUI-explorer、WebOperator、MobileUse |
| **Planner & System**（规划/系统，Sec.6） | 规划器侧 | AndroidWorld、MMBench-GUI、OSWorld-Human、UI-R1、Think Twice Click Once、GUI-G1、AdaGUI-R1、MobileWizard、AgentCPM-GUI、Agent S2、InfiGUIAgent |
| | 系统侧 | ActionEngine、CORE、GUIGuard、CoAct-1、IntentCUA、OS-Symphony、LongHorizonUI、UltraCUA |

- **写作方法**：从种子文献出发，逐小节用定向检索（arXiv / ACL Anthology / OpenReview）+ 前向/后向引用链扩张，把通过筛选的论文并入「证据台账」（附录 Table 2–5），每篇同时记录「报告的效率信号」与「未报告/转移的成本（NR）」。

### 3.3 综述视角的洞察（去伪存真）

按「哪些最有效 / 哪些被高估 / 哪些被忽视」诚实标注：

- **收益最扎实的维度：动作抽象 + 混合运行时**。ActionEngine 把频繁交互「编译」成状态机/程序后，成本 **$0.71→$0.06（11.83×）**、时延 237.5→118.3s、输入 token 62.3k→8.1k、模型调用 10.2→1.8 次；CoAct-1 用「GUI + 写代码」双后端把平均步数从 GTA-1 的 15.22 / UI-TARS 的 14.90 压到 **10.15**。这类收益是数量级、且直接打中「步数多」这一真实痛点。
- **被高估 / 需警惕的维度：区域聚焦视觉**。作者反复强调「省 token ≠ 省时间」：RegionFocus 渐进缩放后平均轨迹开销反而 **+66.8%**、步数 **+19.74%**；R-VLM 区域提议的推理延迟最高到 **2×**；DiMo-GUI 一次任务最多 **7 次 zoom**。检索/裁剪/parser 本身都是新成本项，省下的 token 常被额外推理吃掉。
- **最尖锐、最被忽视的发现：反思/验证比 next-action 预测本身还贵**。OSWorld-Human 报告反思占任务延迟 **76%–96%**，这是对「优化骨干模型推理」路径的强力纠偏——优化 GUI Agent 不应只盯 backbone，验证器的成本必须进账本。
- **证据不足 / 仅声称有效的部分**：附录大量标注 **NR（未报告可比指标）**，如 Set-of-Mark、Tree-of-Lens、SkillWeaver、Agent-S 等几乎没有 token/时延/显存层面的定量信号，作者也明确声明这是「evidence ledger」而非 meta-analysis，跨论文数字不可直接比较。

---

## 4. 具体技术细节

### 4.1 模型结构

本文是综述，**不持有任何单一 base model**。它覆盖的被引骨干横跨纯 LLM、VLM 与专用 grounding 头，可作为理解「效率手段作用在哪种模型上」的参照：ScreenAI（**670M/2B/5B**，输入分辨率最高 **8122**）、Aria-UI（**3.9B** 激活参数）、UI-TARS（**2B/7B/72B**）、GUI-Actor（在 2B 上加 **20M**、7B 上加 **100M** 参数的 action head，实现 coordinate-free 单次前向出候选）。

### 4.2 训练流程

**本文是系统综述，不训练任何模型、也无自己的实验。** 此处按模板如实标注「—（系统综述）」。它引用的方法覆盖多种训练范式（如 UI-R1 的规则奖励 RL、MobileWizard 的渐进式 RL、ScreenAI/UI-TARS 的预训练），但均只是被分类的对象，而非本文的贡献。

### 4.3 推理流程与关键定量发现

综述不提出新的推理流程，而是把主流 GUI Agent 的通用闭环固化下来：**observe（感知）→ reason & plan（规划）→ act（执行）→ reflect & verify（反思/验证）**，多步循环直至成功或失败。它真正有价值的是把散落各处的数字并排，暴露「局部好看、全局没账」的问题：

- **原生界面很重**：原始 DOM 可达 **800k token**（Agent-E），典型任务需 150–220s、约 25 次 LLM 调用；WorkArena 单页 HTML 在 4/9 步回看下 40k–500k token；**5 张 GUI 截图在 KV cache 中可超 80GB 显存**（GUI-KV）。观察与记忆压缩不是锦上添花，而是跑长任务的前提。
- **输入 token 可大幅压缩**：Aguvis 把每步输入从 4k–6k token 压到 **1196（省 70%）**；FocusAgent 平均砍掉 >50% AxTree（常 >80%）；Prune4Web 把 >500 个 DOM 候选压到 <20。
- **运行时级 KV 压缩有硬收益**：GUI-KV 用 5%–20% cache 预算换 **38.9%** 更少 MFLOPs/decoded token；ST-Lite 在 10%–20% 预算下拿到 **2.45×** 解码加速、1.40× 端到端加速（prefill 约 1.0×，收益集中在 decode 侧）。
- **每步延迟的行业基线**：V-Droid 做到 **0.7s/决策、4.3s/步**，而典型移动 Agent 常 >20s/步——数量级差距。
- **效率评测已在成形**：MMBench-GUI 用 50 步预算统计出冗余步成本 **7–8**、隐私噪声 **40%**、专家成本 **16%**；UI-R1 仅用 **136 条样本 + 8×RTX 4090 约 8 小时**即改善 GUI 动作预测。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

综述覆盖的评测系统如下（数据规模/指标以 PDF 核实为准，未明确给出者标 —）：

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 / 效率信号 |
|---|---|---|---|---|---|---|
| Mind2Web | Web | 网页导航 grounding | — | HTML+指令 | 动作 | 成功率 |
| WebArena / VisualWebArena | Web | 端到端网页任务 | — | DOM/截图+指令 | 动作 | 成功率 |
| BrowserGym | Web | 浏览器交互 | — | 截图+DOM | 动作 | 成功率 |
| AndroidWorld | Mobile | 移动 App 操作 | **116 任务 / 20 应用** | 截图+无障碍树 | 动作 | 成功率；内存约 2GB、磁盘 8GB |
| OSWorld | Desktop/OS | 开放式 OS 任务 | — | 截图+指令 | 动作 | 成功率 |
| OSWorld-Human | Desktop/OS | **人类效率剖绘** | — | 人类轨迹 | — | **步数 2.7×–4.3×、反思占延迟 76%–96%** |
| Windows Agent Arena | Desktop/OS | 多模态 OS 任务 | — | 截图 | 动作 | 成功率 |
| MMBench-GUI | 跨平台 | **效率感知评测** | 50 步预算 | 截图 | 动作 | 冗余步成本 7–8、隐私噪声 40%、专家成本 16% |
| ScreenSpot-Pro | GUI/high-res | 高分辨率 grounding | 分辨率 >3k×2k | 截图 | 坐标 | grounding 准确率 |
| WorkArena | Web | 网页任务（回看分析） | HTML 40k–500k token | HTML | 动作 | 上下文开销 |

### 5.2 实验结果分析

综述本身没有 ablation，其「证据台账」的核心结论可归纳为三条跨工作规律：**(1)** 观察层压缩普遍有效，但裁剪/检索/parser 的**隐性成本**几乎全部 NR（未报告），无法做净收益核算；**(2)** 动作抽象与混合运行时的收益最实、且常被社区低估；**(3)** 效率指标高度碎片化——各家自报 token/步数/时延、基线不同，无法跨论文比较，这正是作者呼吁「成功归一化的 GPU 成本」等统一指标的原因。

---

## 6. 亮点与贡献（Why it matters）

1. **第一个以效率为中心组织文献的 GUI Agent 综述**，四轴分类学让做 GUI Agent 的人能快速定位「我的开销卡在哪一环」，且覆盖面新（大量 2025–2026 工作）。
2. **坚持全局核算的批评立场**：每个机制都追问「省下的钱花到哪里去了」，明确点出 parser / verifier / retriever / orchestration 这些隐性第二成本，比单纯罗列 token 节省更有参考价值。
3. **把反思/验证成本高的问题推到台前**：76%–96% 延迟占比对整个社区是强提醒——优化不该只盯着 backbone 推理。
4. **给出可操作的效率评测清单**：呼吁未来 benchmark 上报峰值显存、prefill/decode 延迟、每 token MFLOPs、训练 GPU 小时数与成功归一化 GPU 成本。
5. **附录「证据台账」的 NR 标注体系**：诚实区分成熟证据与不稳定证据，便于顺藤摸瓜、也防止读者被自报数字误导。

---

## 7. 局限与可改进点（个人点评）

- **数字可信度天花板**：作者自认附录是 evidence ledger 而非 meta-analysis，所有收益均为各家自报、基线各异，跨论文不可比；想据此做技术选型仍须回原文逐一核对。
- **「全局核算」止步于叙事**：全文反复强调成本转移，却几乎没给出任何**跨机制的量化权衡**（如「检索器开销」vs「少算 80% DOM」到底净赚多少），核心论点停在定性呼吁，未能落成形式化的开销归因模型。
- **时效性风险**：大量引文是 arXiv 预印本，附录不得不用 normalized/emerging/unstable 标注成熟度；个别机制（如 UltraCUA）因状态不稳只留在挑战章，覆盖不完整。
- **工业落地视角缺失**：评测几乎全来自研究 benchmark，缺真实部署中的并发用户、多任务调度、上下文缓存复用等系统级度量——而这恰是 runtime optimization 最有说服力的场景。
- **归类边界模糊**：观察/记忆/动作三章有重复归类（如 Prune4Web 同时出现在观察缩减与动作剪枝），四轴间缺少形式化的开销归因框架；也未讨论数据泄漏/过拟合对效率评测的干扰。

---

## 8. 对我们的启示 / 可借鉴点

结合项目关注的 RL / GUI Grounding / Online / Planning 方向，几条可落地启发：

- **训练目标里显式惩罚低效行为**：本文最大的警钟是「成功率会掩盖低效」。做 RL 时若 reward 只看任务完成，模型可能学会 30 步的冗余轨迹。可参考 MMBench-GUI 的冗余步成本、OSWorld-Human 的步数差距，把步数、重复动作、反射调用次数做成约束项或 shaped reward。
- **verifier/反思的成本必须进账本**：项目若用过程奖励（类 GUI-Shepherd 思路），要留意「验证器调用」本身是新的大头开销；把验证器做成轻量、可复用、按需触发的信号，而非每步都请大模型裁判。
- **记忆表示直接决定长程 RL 可行性**：GUI 历史很贵（5 张截图 80GB 显存）。做 long-horizon 训练/部署时，KV 预算与摘要记忆（或 Continuous Memory 式稠密记忆）应作为状态表示的一部分来设计，而非事后压缩。
- **难度自适应的慢思考值得借鉴**：Think Twice Click Once、AdaGUI-R1（省 40% 推理 token）说明不必对每步开满推理；把「要不要深想」做成可学习的决策，与 difficulty-aware 策略天然契合。
- **混合运行时拓宽动作空间**：CoAct-1 证明「能用代码/API 就别用 GUI 点点点」能显著降步数。构建动作空间时可显式加入非 GUI 执行通道作高层动作，尤其在文件操作、数据处理类子任务上。
- **Grounding 训练可借鉴「全局到局部」**：Aguvis/UGround/RegionFocus 的截图优先 + 区域放大思路，提示 GUI grounding 的数据与模型设计可用多尺度 crop 让模型「先定位再细看」，并显式惩罚无意义的过度放大。

---

## 9. 延伸阅读

- 本文对齐的三篇基础综述：Yang et al., 2026b（efficient agents）、Sager et al., 2025（computer-use agents）、Nguyen et al., 2024 *GUI Agents: A Survey*（arXiv:2412.13501）。
- 效率评测实证：Abhyankar et al. *OSWorld-Human*（arXiv:2506.16042）、Wang et al. *MMBench-GUI*（arXiv:2507.19478）。
- 与项目方向最相关的代表工作：UI-R1（RL 高效动作预测，arXiv:2503.21620）、MobileWizard（数据高效渐进式 RL）、CoAct-1（GUI+代码混合运行时，ICLR 2026）、ActionEngine（状态机/程序化动作编译，arXiv:2602.20502）、GUI-Shepherd（过程奖励作验证信号）、GUI-KV（GUI 时空感知 KV 压缩，arXiv:2510.00536）。

---
*解读生成时间：2026-09-10 ｜ 解读人：WorkBuddy（AI）*
