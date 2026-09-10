# From Interaction Traces to Persistent Skills: Online Evolution for Computer-Use Agents（从交互轨迹到持久技能：计算机使用智能体的在线演化）

> 一句话 TL;DR：把 agent 的交互轨迹与评估反馈，在线改写成带版本、可溯源的持久技能库，参数全冻结，仅靠外部程序性记忆让固定骨干越用越强。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | From Interaction Traces to Persistent Skills: Online Evolution for Computer-Use Agents |
| **作者 / 机构** | Longtao Hu（第一作者，电子科技大学）；Xiao Liang（浙江大学）；Linchao Zhu\*（通讯作者，浙江大学）；中国 |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-04；arXiv preprint（arXiv:2609.04869v1 [cs.AI]，未标注会议） |
| **arXiv 链接** | https://arxiv.org/abs/2609.04869 |
| **代码仓库** | ✅ GitHub：https://github.com/LongtaoHu/Skill-Evo4GUI.git |
| **数据集地址** | 未单独公开数据集；基于 OSWorld（v0.1.16-934-g8f41f80a + happysixd/osworld-docker:latest）四个应用域 |
| **类型标签（论文类别）** | `Online` `Reflection` `Desktop` |
| **训练方法标签** | `—（框架方法：所有模型参数冻结，仅通过外部技能库的创建/编辑/删除在线演化）` |
| **关键词** | 在线技能演化；程序性记忆；技能库；GUI 自动化；可追溯性（provenance） |
| **来源渠道** | arxiv-api + listing |
| **PDF 存档** | papers/2026-09-04_From_Interaction_Traces_to_Persistent_Skills_Online_Evolution_for_Computer-Use_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：在真实桌面 GUI（OSWorld）中，computer-use agent 把自然语言指令 + 屏幕观测转化为键鼠操作，如何让它的"程序性经验"被**系统性地保留、精炼、复用**。
- **为什么重要**：一次 rollout 跑通的流程、踩过的坑，任务结束即丢。后果是 agent 会在后续任务里**重新发现同一流程、复现同一失败**——这正是生产部署中最昂贵的浪费。
- **现有方法有什么不足**：已有技能库范式（CUA-Skill、MMSkills、MMG2Skill、EvoSkill、SkillOS 等）提供外部程序知识，但两个关键问题**始终未被刻画**：① 相比同一套系统"不带技能"，会自我演化的技能库到底多出多少增量价值？② 技能库的纵向动态——技能是否真的被"跨任务起源边界"复用，还是只是各任务的私房笔记？反复修改某个技能能否救回它的起源任务？作者指出"最终库 + 聚合端点分数"无法回答这些问题（第 2 页）。
- **Research Gap**：作者主张补上的缺口是——在**固定 Executor–Grounding 骨干**下，用**配置匹配的空库对照 + 纵向溯源分析**，刻画"在线演化技能库"的增量价值与长期动态，而非再堆一个通用技能 benchmark。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

在参数完全冻结的 GUI agent 外挂一层 Git 版本化的技能库，通过"冻结快照执行—证据引导演化"的迭代闭环，把交互轨迹变成可复用、可审计的共享程序性记忆。

### 3.2 方法总览（Pipeline）

- **输入**：任务指令 q + 当前屏幕截图 + 冻结技能快照 S_t；**输出**：pyautogui 键鼠动作，以及下一轮快照 S_{t+1}。
- **四大模块**（图 1）：
  1. **运行时执行（Runtime Execution）**：固定 Executor 选工具、固定 Grounding 映射坐标、pyautogui 落盘执行；技能目录只渲染 `name + description`，Executor 显式调 `get_skill(name, reasoning)` 才取回完整 SKILL.md。
  2. **轨迹抽象（Trace Abstraction）**：Extractor 把原始轨迹 + 官方评估器得分 + 技能调用遥测转成结构化事实 F_t。
  3. **在线技能演化（Online Skill Evolution）**：Proposer 诊断 → Coordinator 校验 → Skill Builder 物化，产出 create/edit/delete/no-op/unresolved 变更。
  4. **持久技能库（Persistent Library）**：每次变更一次 Git commit，形成下一轮快照。
- **连接方式**：**读执行、写演化彻底分离**——本轮所有任务都在同一个冻结快照下跑完，演化产生的变更只在**下一轮**快照才可见。前 5 轮空库预热（t=0–4），t≥5 才出现第一个非空快照。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新（本文无新损失/新结构，创新在研究设计与可审计性）**：① **iteration-frozen 快照的读写分离**，把"评估看到的状态"与"更新中的状态"隔离，避免技能演化与模型微调/轮内自适应混淆；② **provenance-aware 溯源**——每个技能固定"起源任务、修订历史、消费任务、下游结局"四元留痕；③ **Coordinator 校验门**（创建需 5 次观察、尺寸门降级、域前缀、unresolved 需至少 2 次失败）。
- **工程组合**：Extractor/Proposer/Builder 三角色全用 Kimi K2.5 编排；渐进检索（catalog 只放元数据）；EvoCUA-32B + MVP grounding + pyautogui；Git 版本化。均为现成组件拼接。
- **对性能提升最关键的设计**：**无法由消融定位**——本文没有任何组件级消融（作者把 "matched component ablations" 明列为 immediate priority）。从研究设计看，最关键是"iteration-frozen + 空库对照"，它让增量价值**可归因**，而非某个性能组件本身。
- **证据不足 / 仅声称有效**：43.3% 的跨来源复用是**观察性证据**（遥测只记"取了哪个技能"，不测"是否照做、是否受益"）；Full−Empty 的差受预热期混淆（Writer 空库期已 +9.1）；每条件-域仅 1 条纵贯轨迹，无显著性检验。

---

## 4. 具体技术细节

### 4.1 模型结构

- **Executor**：EvoCUA-32B（视觉 agent 策略，MLLM），负责工具选择与自然语言元素描述。
- **Grounding**：MVP（Multiple View Prediction，训练自由的 multi-view 推理），把描述 + 截图映射为绝对像素坐标。
- **Extractor / Proposer / Skill Builder**：Kimi K2.5（视觉 agent 骨干）。
- **关键设定**：以上**所有参数冻结**，不微调、不 RL、不 LoRA。

### 4.2 训练流程（本文无参数训练，重点是 online evolution 循环）

| 阶段 | 目标 | 处理什么 | 数据来源 | 数据形态 | 关键约束/机制 |
|---|---|---|---|---|---|
| 执行（Execute） | 生成轨迹 | 任务在冻结快照 S_t 下跑完整 sweep | 固定任务集 Q_d | 轨迹 τ + 遥测 | 本轮内看不到任何技能修改 |
| 抽象（Extract） | 轨迹→结构化事实 | 抽取结果、状态-动作-效果、重复模式、技能调用观察 | 轨迹 + 官方评估器分 + 后置检查 | 结构化事实 F_t | 官方分数**覆盖**模型自述；遥测校正（幻觉调用删除、遗漏调用恢复） |
| 诊断（Propose） | 产出变更提案 | 同一任务最近至多 5 轮事实历史 + 当前库元数据 | F_t 历史 | create/edit/delete/no-op/unresolved | create/edit 只给高层意图，非完整文档 |
| 校验+物化（Coordinate+Build） | 落地技能变更 | Coordinator 校验 → Builder 生成完整 SKILL.md（edit=完整替换） | 校验后动作 | 可替换的 SKILL.md | 创建需 5 次观察；重复名/超尺寸降级 no-op；一次变更一次 Git commit |

> 演化采用**串行决策模式**（serial decision mode）：任务按固定 ID 顺序进入演化阶段，前一任务的 commit 完成后才处理下一个，因此演化**顺序依赖**；但所有 rollout 已在本轮快照下跑完，本轮 Executor 永远看不到本轮 commit。

### 4.3 推理流程

- **多步推理**：固定 Executor 依据指令 + 当前截图选工具 → 坐标类动作由 Grounding 映射 → pyautogui 执行 → 新截图成为下一观测，循环直至 finish/refine。
- **渐进检索**：catalog（仅 name+description）渲染进 prompt；Executor 判定相关时调 `get_skill(name, reasoning)` 取回完整技能体作为工具结果。catalog 与 get_skill 均**只读**，不产生副作用。技能**不直接执行 GUI 动作**——技能条件化决策仍走同一 Executor 工具接口，坐标类动作仍需固定 Grounding。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| OSWorld — GIMP | Desktop（Linux） | 端到端图像操作 | 26 任务 × 35 轮 | 截图 + 指令 | 键鼠动作 | 官方 evaluator 分 [0,1]（含 partial credit）→ 域均值% |
| OSWorld — VLC | Desktop | 端到端媒体操作 | 17 × 20 | 截图 + 指令 | 键鼠动作 | 同上 |
| OSWorld — LibreOffice Writer | Desktop | 端到端文档操作 | 23 × 20 | 截图 + 指令 | 键鼠动作 | 同上 |
| OSWorld — Thunderbird | Desktop | 端到端邮件操作 | 15 × 40 | 截图 + 指令 | 键鼠动作 | 同上 |

> 外部 OSWorld 参考分（非匹配基线）：GIMP 76、VLC 49、Writer 69、Thunderbird 80（百分制），用于标定操作区间。

### 5.2 实验结果分析

- **主结果（Full vs Empty 配置匹配对照，预热后均值 t≥5）**：

| 域 | Full 预热后 | Empty 预热后 | 预热后差 | 预热期差(t=0–4) |
|---|---|---|---|---|
| GIMP | 80.2 | 74.4 | **+5.7** | −4.0 |
| VLC | 64.0 | 45.4 | **+18.6** | +4.7 |
| Writer | 63.9 | 52.0 | **+11.8** | +9.1 |
| Thunderbird | 74.7 | 62.3 | **+12.4** | −6.7 |

  四个域 Full 预热后均值**全面领先**（差 5.7–18.6 pp），但**既不均匀也不单调**：VLC 差最大且分离稳定；GIMP 差最小、多次交叉；GIMP（−4.0）与 Thunderbird（−6.7）预热期为负、技能启用后由负转正；而 Writer 空库期就已 +9.1，说明其大部分差距**不能归因于技能**。
- **溯源分析（仅 GIMP）**：35 轮后库内 **27 个技能**；82.4% 有效 rollout 至少调用一次 get_skill；全部调用中 56.7% 取回当前任务起源的技能，**43.3% 跨起源取回**——共享程序性记忆的直接证据（但取回≠遵循）。
- **Ablation 说明了什么**：本文**无组件消融**，最接近消融的是 Empty 空库对照 + 预热期差分解耦。它证明"库是否可演化"整体有效，但**无法定位**贡献来自检索、演化还是抽象；"修订空转"案例（gimp-add-alpha-channel 反复被接受编辑，其起源任务 35 轮仅成功 2 次 = 5.7%）反向提示——**光靠失败驱动改库可能空转**，瓶颈在固定 Executor–Grounding 栈本身。

---

## 6. 亮点与贡献（Why it matters）

1. **罕见的诚实对照**：配置匹配的空库对照 + 纵向轨迹，明确区分"哪些提升来自技能、哪些预热期就存在"，不搞"挂库即涨分"的营销式结论。
2. **不碰参数的在线自适应范式**：演化完全发生在外部技能库，对不能随意重训、部署受限的产品场景有直接参考价值。
3. **把技能库变成可审计对象**：Git 式提交 + 快照冻结 + 起源/修订/消费/结局四元留痕，为后续技能质量治理打地基。
4. **把负面结论讲透**：修订空转、域相关收益不稳定、检索≠遵循——比涨分更有信息量，能防止同行盲目堆技能库。
5. **面向研究问题的评测粒度**：用溯源数据回答"跨任务复用是否存在"，而非只看端点分数，方法论可迁移。

---

## 7. 局限与可改进点（个人点评）

- **证据强度偏弱**：每条件-域仅 1 条独立纵贯轨迹，随机波动无法估计；预热期不平衡（Writer 空库已 +9.1）使"预热后差 = 技能贡献"的因果解读不成立，作者也承认。各复跑 3–5 条才谈得上可信度。
- **任务集重复固定**：衡量的是"背下这套题"的域内适应，对**未见任务零评测**——而未见任务才是技能库泛化价值的真正所在，实际落地收益可能被高估。
- **检索-遵循脱节未闭环**：遥测只记"调用了哪个技能"，不测"是否照做、是否受益"，43.3% 跨来源复用因此只是观察性证据，缺最后一环。
- **规模与覆盖面小**：仅 GIMP 有技能级剖析；VLC 仅 17 任务；串行演化顺序依赖、无自动回滚/合并，轮次一长库内可能累积矛盾修订。
- **演化质量押在 LLM 评审上**：Extractor/Proposer/Builder 的噪声无人抽检，错误提案会随轮次滚雪球。

---

## 8. 对我们的启示 / 可借鉴点

1. **"冻结快照、下一轮生效"是应抄的工程纪律**：把"评估看到的状态"与"更新中的状态"隔离，直接消除自我改进型 agent 评测里最常见的脏状态问题。
2. **外挂技能是重训之外的低成本演进通道**：对几十 B 级模型，"技能库 + 固定骨干"能买到可观增量（本文最多 +18.6 pp），产品侧可先上这个再谈重训。
3. **给技能加"修订刹车"**：针对修订空转，可给演化策略加轻量强化信号（下游成功率升了才奖励修订），或设"连续 N 次修订未救回任务即回滚/冻结该技能"护栏。
4. **复用 CUA-Universe 混合思路升级技能载体**：从纯文本 SKILL.md 升级为"CLI 工具面 + GUI 操作 + when_to_use/not_when"的结构化条目，命中率与遵循率大概率高于纯文本。
5. **溯源打点成本低、收益高**：任何轨迹/技能数据管理都应记录"出生任务、消费任务、修订历史"，这是回答"数据到底有没有被跨场景复用"的唯一办法。

---

## 9. 延伸阅读

- 经验/反思外存：Reflexion、ExpeL、Voyager、Agent Workflow Memory
- 面向 GUI 的技能表示：CUA-Skill、MMSkills、MMG2Skill、Skill-Use、SkillsBench
- 技能库自我演进：EvoSkill、SkillOS、XSkill、Mem2Evolve、MUSE-Autoskill、SAGE
- 底层评测与基座：OSWorld、WebArena、SeeClick、MVP、EvoCUA、Kimi K2.5、Qwen3

---
*解读生成时间：2026-09-10 ｜ 解读人：WorkBuddy（AI）*
