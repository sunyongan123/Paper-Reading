# Efficient GUI Agents: A Systems Survey of Observation, Memory, Action, and Runtime Optimization

> TL;DR：把"能不能完成任务"和"完成任务花了多少代价"分开，从观察、记忆、动作、系统四层系统性梳理 GUI 智能体如何做得更快更省的一篇综述。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | Efficient GUI Agents: A Systems Survey of Observation, Memory, Action, and Runtime Optimization |
| **arXiv ID / DOI** | arXiv:2609.02309v1（cs.CL） |
| **arXiv 链接** | <https://arxiv.org/abs/2609.02309> |
| **发表出处（Venue）** | arXiv preprint；arXiv comment 标注 "Accept at Grounding Language Models: Learning Faithfully and Efficiently @ EMNLP 2026"（PDF 首页与正文未印 venue，出处取自 arXiv 元数据） |
| **发布时间** | 2026-09-02 |
| **作者** | Bizhe Bai†, Jiakang Yuan†, Hongming Wu, Xinyue Wang, Jie Ren, Siyao Chen, Yuchen Ya, Fan Bai, Pai Peng, Huafeng Qin, Tao Chen（† 为共同一作，据首页脚注标记；未标注通讯作者） |
| **所属机构** | 复旦大学 未来信息技术学院（上海）、上海创新研究院（上海）为主；Fan Bai、Pai Peng 标注为独立研究者；Huafeng Qin 属重庆工商大学 |
| **开源情况** | 未在文中声明开源（正文无代码/数据发布声明；附录表格中的 "GitHub" 列指被引论文的仓库，非本文资源） |
| **类型标签** | `General` `Web` `Mobile` `Desktop`（跨平台效率综述，非单一方法论文） |
| **训练方法标签** | —（系统综述：效率分类学，非训练方法论文） |
| **关键词** | GUI 智能体、系统效率、观察压缩、记忆管理、动作抽象、综述 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-02_Efficient_GUI_Agents_A_Systems_Survey_of_Observation_Memory_Action_and_Runtime_Optimization.pdf |

---

## 1. 研究背景与要解决的问题

近两年 LLM/VLM 把 GUI 自动化推向了"开放式计算机使用"：智能体能读自然语言指令、看实时界面、跨网页/手机 App/桌面系统执行多步任务，Mind2Web、WebArena、VisualWebArena、AndroidWorld、OSWorld、Windows Agent Arena 等基准不断刷新成功率。

但本篇作者认为：**"能做"不等于"能用"**。一个智能体可能最终完成任务，却要跑几十轮"观察→推理→动作"循环、调用无数次模型、在每步之间让用户干等。做个类比：两个厨师都能把菜做出来，一个手忙脚乱、反复翻菜谱、在灶台前耗半小时还打翻两个碗，另一个熟门熟路十分钟收工——如果只问"菜熟没熟"，两人评分一样，这显然不公平。OSWorld-Human 的数据证实这不是杞人忧天：领先的 computer-use 智能体完成任务所需的步数比人类轨迹多 2.7~4.3 倍，且规划、判断、反思占了端到端延迟的大头（反思单独就占 76%~96%）。

GUI 智能体与纯文本工具智能体不同：感知靠截图/DOM/无障碍树（可观测性差、视觉密集、小目标多）、任务是长程且错误敏感的，每次交互都暴露在用户屏幕上。效率因此不只是省钱问题，还直接决定可部署性、隐私（减少外传 UI 内容）和用户体验。本文要补的缺口是：**已有综述按"能力/架构"组织文献，本文第一次把"效率"作为一等公民，用端到端系统视角回答"GUI 智能体的开销到底花在哪、业界有哪些手段在省、省完又把成本转移到哪里"。**

## 2. 核心方法 / 思路

本文不提出新模型，而是给出一个**效率导向的分类学（taxonomy）**。它先把 GUI 智能体抽象成一个闭环：`a_t ~ π(g, o_t, m_t)`——拿着目标 g，看一眼当前界面观察 o_t，结合累积的记忆 m_t，决定下一步动作 a_t。围绕这条链，效率被切成四个轴，正文各用一章展开：

- **观察效率（Observation）**：当前界面"怎么表示"最省。原始 DOM 可能长达 80 万 token（Agent-E 实测），截图则充满装饰性背景和密集小控件。手段分三派：①文本缩减——把 DOM/无障碍树当检索与剪枝对象而非全文灌入，如 FocusAgent 平均砍掉一半以上 AxTree、Prune4Web 把 >500 个候选元素压到 <20 个；②区域聚焦视觉——"全局扫一眼、局部放大看"，Ferret-UI 把屏幕分块、RegionFocus 渐进缩放可疑区域、ShowUI 直接在 token 层剪掉 33% 冗余视觉 token；③解析增强与混合观察——给"无结构截图"补结构，如 OmniParser、Set-of-Mark，或反之让截图优先模型（Aria-UI、UGround、Aguvis、UI-TARS）摆脱对 DOM 的依赖，Aguvis 每步输入从 4k~6k token 降到 1196（约省 70%）。
- **上下文与记忆效率（Context & Memory）**：历史信息"怎么存、怎么取"。一条主线是用可恢复的精简记忆取代"逐帧回放原史"：Agent-S、GUI-Rise、PAL-UI 做摘要压缩与选择性回看，ColorBrowserAgent 干脆把交互上限钉在 30 步。另一条更底层的主线是运行时级压缩：GUI 轨迹的截图在 KV cache 里极其昂贵——5 张截图就能吃超 80GB 显存，GUI-KV、ST-Lite 专门做面向 GUI 注意力模式的缓存剪枝（用 5%~20% 缓存预算换 38.9% 更少的每 token 计算量、2.45× 解码加速）；Continuous Memory 更极端，把 >15k token 的轨迹压成 8 个稠密向量（只微调 1.2% 参数）。
- **动作效率（Action）**：目标是用更少的步数和更少的不可逆错误抵达终点。三条路：①动作抽象——把反复出现的原始操作提成可复用技能/程序，ActionEngine 把频繁交互"编译"成状态机记忆，成本从 $0.71 降到 $0.06、模型调用从 10.2 次降到 1.8 次；②候选剪枝/验证/恢复——执行前先给动作打分或验证（V-Droid 每步决策仅 0.7 秒），出错了能回溯（BacktrackAgent、LongHorizonUI）；③探索控制——让探索被意图或学习到的转移逻辑引导，而不是盲目试错（Auto-Intent、WebOperator），MobileUse 只在冷启动时才全量探索。
- **规划器侧与系统效率（Planner & System）**：这是本文最尖锐的一刀——**反思/验证往往比"下一步预测"本身还贵**（占任务延迟 76%~96%），所以出现了按难度选择性调用"慢思考"的做法（AdaGUI-R1 省 40% 推理 token；Think Twice, Click Once 给难例才开慢速模式），以及层级化、按需触发的反思（MobileUse 把反射开销压到 10%）。系统层则把本地/云端分工、隐私过滤（CORE、GUIGuard）、以及"GUI 操作 vs 写代码执行"的混合运行时（CoAct-1，把子任务路由给代码后端后平均步数从 15.22 降到 10.15）纳入效率设计。

写作方法上，本文从一组种子文献出发，用定向检索加前/后向引用链扩张，再按上述小节省并成"证据台账"（见附录 Table 2–5）；每个机制都同时记录**报告的效率信号**和**它新引入的开销**——这是它区别于一般综述的诚实之处。文末把全文收敛为五个反复出现的思想：选择性阅读而非全量灌入、全局到局部的视觉分配、可恢复记忆而非原始历史回放、验证感知的控制、GUI 与非 GUI 可切换的混合运行时。

## 3. 关键实验结果

综述没有自己的实验，但它的价值恰恰在于把散落各处的数字摆到一张桌上，暴露"局部好看、全局没账"的问题。几条最值得记住的观察：

- **原生界面很重**：原始 DOM 可达 80 万 token（Agent-E）；WorkArena 单页 HTML 在 4~9 步回看设置下就有 4 万~50 万 token；5 张 GUI 截图在 KV cache 中可超 80GB 显存（GUI-KV）。这说明观察与记忆层的压缩不是锦上添花，而是能不能跑长任务的前提。
- **省 token 不等于省时间**：RegionFocus 采用渐进缩放后平均轨迹开销反而 +66.8%、步数 +19.7%；R-VLM 区域提议的推理延迟最高到 2×；DiMo-GUI 一次任务最多迭代 7 次 zoom。剪枝/检索/解析器本身都是新的成本项——这正是全文反复强调的"成本转移"。
- **动作抽象收益极大但常被忽视**：ActionEngine 把单任务成本从 $0.71 压到 $0.06、时延从 237.5 秒降到 118.3 秒；CoAct-1 用"GUI+代码"混合执行把平均步数 15.22→10.15；V-Droid 做到每步 4.3 秒，而典型移动智能体每步常超过 20 秒——数量级的差距。
- **效率基准已在成形**：OSWorld-Human 报告领先智能体步数为人类的 2.7~4.3 倍、反思占延迟 76%~96%；MMBench-GUI 用 50 步预算统计出 7~8 的冗余步成本；UI-R1 只用 136 条样本 + 8 块 4090 约 8 小时就改善了 GUI 动作预测——说明"数据/算力效率"与"运行效率"同样被纳入议程。

## 4. 亮点与贡献（Why it matters）

1. **第一个以效率为中心组织文献的 GUI 智能体综述**，四轴分类学（观察/记忆/动作/规划与系统）让新读者能快速定位"我的开销卡在哪一环"。
2. **坚持全局核算的批评立场**：每个机制都追问"省下的钱花到哪里去了"，明确点出 parser、verifier、retriever、编排这些"隐性第二成本"，比单纯罗列 token 节省更有参考价值。
3. **把反思/验证成本高的问题推到台前**：76%~96% 延迟占比这个数字对整个社区是强提醒——优化 GUI 智能体不应只盯着骨干模型推理。
4. **给出效率评测的落地清单**：呼吁未来基准上报峰值显存、prefill/decode 延迟、每 token MFLOPs、训练 GPU 小时数与"成功率归一化的 GPU 成本"，为统一比较提供了可操作标准。
5. 覆盖面新（大量 2025–2026 工作）且区分了成熟与不稳定证据，附录按小节省并的"证据台账"含每家论文的 NR（未报告可比指标）标注，便于顺藤摸瓜。

## 5. 局限与可改进点（个人点评）

- **数字可信度天花板**：作者自己把附录表定性为 "evidence ledger" 而非 meta-analysis——所有收益数字都是各论文自报、基线各异的，跨论文不可比。综述没有（也无法）用统一实验校准，读者若想据此做技术选型仍需回到原文。
- **"全局核算"只到叙事层面**：虽然反复强调成本转移，但几乎没有给出任何跨机制的量化权衡（比如"剪枝到 20 个候选的检索器开销"vs"少算 80% DOM"到底净赚多少），这使其核心论点停留在定性呼吁。
- **领域时效性风险**：大量引文是 arXiv 预印本，附录不得不用 "normalized/emerging/unstable" 来标注证据成熟度，说明领域仍在快速变化，成稿即局部过时；个别机制（如 UltraCUA）因状态不稳只留在挑战章，覆盖不够完整。
- **工业落地视角缺失**：评测与例子几乎全来自研究 benchmark，没有真实部署中的并发用户、多任务调度、上下文缓存复用的系统级度量，而这恰是"runtime optimization"最有说服力的场景。
- **效率指标的边界模糊**：观察/记忆/动作三章有重复归类嫌疑（如 Prune4Web 同时出现在观察缩减与动作剪枝），四轴之间缺少形式化的开销归因框架；另外未讨论数据泄漏/过拟合对效率评测的干扰。

## 6. 对我们的启示 / 可借鉴点

结合项目关注的 RL / GUI Grounding / Online / Planning 方向，几条可以落地的启发：

- **训练目标里应显式惩罚低效行为**：本文最大的警钟是"成功率会掩盖低效"。做 RL 训练时，若 reward 只看任务完成，模型学会的可能是 30 步的冗余轨迹。可参考 MMBench-GUI 的冗余步成本与 OSWorld-Human 的步数差距，把步数、重复动作、反射调用次数做成约束项或 shaped reward。
- **verifier/反思的成本必须进账本**：项目若采用 RLHF/过程奖励（类似 GUI-Shepherd 的思路），要留意"验证器调用"本身是新的大头开销；把验证器做成轻量、可复用、按需触发的信号，而非每一步都请大模型裁判。
- **记忆表示直接决定长程 RL 可行性**：GUI 历史很贵（5 张截图 80GB 显存）。做 long-horizon 训练/部署时，KV 预算与摘要记忆（或 Continuous Memory 式稠密记忆）应作为状态表示的一部分来设计，而不是事后压缩。
- **难度自适应的慢思考值得借鉴**：Think Twice, Click Once 与 AdaGUI-R1 告诉我们——不必对每个 GUI 步骤都开满推理；把"要不要深想"做成可学习的决策，与我们的 difficulty-aware 策略复用方向天然契合。
- **混合运行时拓宽动作空间**：CoAct-1 证明"能用代码/API 就别用 GUI 点点点"能显著降步数。构建 agent 动作空间时，可以显式加入非 GUI 执行通道作为高层动作，尤其在文件操作、数据处理类子任务上。
- **Grounding 训练可借鉴"全局到局部"**：Aguvis/UGround/RegionFocus 的截图优先 + 区域放大思路，提示 GUI grounding 的数据与模型设计可用多尺度 crop 让模型学会"先定位再细看"，并显式惩罚无意义的过度放大。

## 7. 延伸阅读

- 本文对齐的三篇基础综述：Yang et al., 2026b（efficient agents）、Sager et al., 2025（computer-use agents）、Nguyen et al., 2024 *GUI Agents: A Survey*（arXiv:2412.13501）。
- 效率评测实证：Abhyankar et al. *OSWorld-Human*（arXiv:2506.16042）、Wang et al. *MMBench-GUI*。
- 与项目方向最相关的代表工作：UI-R1（RL 高效动作预测，arXiv:2503.21620）、MobileWizard（数据高效的渐进式 RL）、CoAct-1（GUI+代码混合运行时）、ActionEngine（状态机/程序化动作编译）、GUI-Shepherd（过程奖励作验证信号）。

---
*解读生成时间：2026-09-03 ｜ 解读人：WorkBuddy（AI）*
