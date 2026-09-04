# CoAdapt-GUI：为"没见过"的 GUI 应用做联合工作流上下文与策略自适应

> TL;DR：让移动 GUI 智能体遇到训练外的新 App 时，只靠自身试错与奖励，同时升级"流程知识"与"操作策略"，在两个 unseen-app 评测上均达最优成功率。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | CoAdapt-GUI: Joint Workflow Context and Policy Adaptation for Unseen GUI Applications |
| **arXiv ID / DOI** | 2608.11588v1 |
| **arXiv 链接** | <https://arxiv.org/abs/2608.11588> |
| **发表出处（Venue）** | arXiv preprint（venue 未标注） |
| **发布时间** | 2026-08-12 |
| **作者** | Linqiang Guo, Li Gu, Zihuan Jiang, Zhixiang Chi, Siobhan Reid, Ziqiang Wang, Yuanhao Yu, Wei Liu, Yang Wang, Tse-Hsun (Peter) Chen |
| **所属机构** | Concordia University（康考迪亚大学，加拿大）；Mila – Québec AI Institute（加拿大）；University of Toronto（多伦多大学，加拿大）；McMaster University（麦克马斯特大学，加拿大） |
| **开源情况** | 未在文中声明开源（全文无代码/数据链接；复用了已发布的 AndroidWorld-Generalization 设定，并自建 AndroidWorld Plus） |
| **类型标签** | `RL` `Online` `Mobile` `Reflection` |
| **训练方法标签** | RL (GRPO 式 group-relative), Online RL, LoRA（测试时自适应） |
| **关键词** | 测试时自适应（TTA）、GUI 智能体、跨应用泛化、工作流上下文、LoRA、在线强化学习 |
| **来源渠道** | github-awesome-OSU（OSU-NLP-Group/GUI-Agents-Paper-List 收录） |
| **PDF 存档** | papers/2026-08-12_CoAdapt-GUI_Joint_Workflow_Context_and_Policy_Adaptation_for_Unseen_GUI_Applications.pdf |

---

## 1. 研究背景与要解决的问题

移动 GUI 智能体（帮你订餐、设闹钟、查运动的 agent）通常在固定一批 App 上训练与评测。一旦部署，用户打开的很可能是不在训练集里的新 App——好比一个只熟自家医院流程的实习生被派到陌生医院：既找不到挂号入口，更不知道"该先干什么、怎样才算办完"。

论文引用的现象点明了难点所在：在**同一种界面上做新任务**，源端在线 RL 能把 7B 策略提升 26.1 个百分点；但换到**全新 App** 时增益骤降到 8.3 个百分点（Gu et al. 2026）。"熟悉环境上的新题"与"陌生 App 里的老题"不是一回事。作者把失败拆成两类缺口：**界面缺口**——读得懂目标却找不到/不会点新界面的元素；**流程缺口**——单个动作会做，但缺完整流程、易错点与"何时算完成"的校验。

以往工作只修一半：AndroidWorld-Generalization 只更新策略、工作流上下文不动；UI-Mem 在源端联合学"经验记忆+策略"再整体迁移。本文补的洞是：**agent 撞上陌生 App 之后，能否仅凭短时间自主交互与奖励，把流程知识与策略一起改好，再在没碰过的留出任务上验收？**

## 2. 核心方法 / 思路

总框架一句话：把"会不会用这个 App"拆成两个独立又互相喂数据的可更新状态——**显式的工作流上下文**（一条条可读的知识）与**参数化的策略**（frozen 大模型上的 LoRA），在目标 App 上跑"取轨迹→拿奖励→双通道更新"的循环同时改进。

**状态一：可迁移约束的工作流上下文。** 每个可迁移条目是类型化四元组 `⟨c,P,F,V⟩`：适用条件、抽象步骤、失败/恢复条件、可观测的完成校验。关键设计是防负迁移的"出入证"：源 App 知识被切两类——**app-bound 状态**（屏跳转 FSM：界面、坐标、资源 id、导航路径，永不跨 App）与**可复用流程**（只讲达成什么、避哪些坑、怎么验收）。任何要跨界的条目都要过 schema 校验与 linter：夹带 App 名、包名、资源标识符、控件文案、坐标、任务实例值的通通拒绝；类别不匹配则检索返回空。类比：跳槽能带走"做事方法论"，不能带前东家的通讯录。源库固定不改，目标端另设独立上下文区从零累积，避免把源知识越改越坏。

**状态二：任务-上下文匹配的策略更新。** frozen VLM + LoRA，采用 group-relative 目标（源自 DeepSeekMath）：**只在"任务 + 上下文变体"都相同的轨迹组内**算相对优势（组内奖励减均值再归一化）。因为不同上下文会改变有效策略输入，奖励差可能只是上下文差异而非动作好坏；奖励无波动的组不给梯度。

**双通道协同。** 每轮选一个适配任务，用若干"已实体化"的上下文变体在可重置模拟器里跑受控轨迹：
- **上下文通道**：奖励喂 TrueSkill 给各变体打分；冻结的"反思器"对比成败轨迹，向高分父条目提类型化修订。修订出的"子代"不直接生效——过 schema/来源/lint 三关才进候选群体，且不继承产生它的本轮奖励，须下轮被采样实测拿好分才算数，杜绝自说自话式学习；
- **策略通道**：轨迹按（任务,上下文）分组，有非简并组就做一次 LoRA 更新，随后清空 buffer、轨迹不重用。

两通道共享受控 rollout 但节奏不同：上下文每轮打分/演化，策略攒够有效比较组才动。二者是"交互耦合"而非联合可微：上下文塑造策略要学的轨迹，策略又决定将来修订上下文所用的成败样例。预算耗尽后取最高分上下文 + 最终 LoRA，全部**冻结**再上留出评测。与已有方法差异：LearnAct/AdaptAgent 靠少量示范，本文只用自身带可执行奖励的交互；UI-Mem 把记忆内化进策略，本文坚持上下文与策略是独立、可单独验证的目标端状态。

## 3. 关键实验结果

**设定一：AndroidWorld-Generalization（新 App + 见过模板的新实例）。** 源端 12 App/62 模板/905 实例，基座 UI-TARS-7B（step-500）；目标 5 个不相交 App，8 个适配实例/App（共 40），48 个留出实例，50 步/App。成功率：Base 27.10%，Static Context Transfer 28.75±2.28，Policy-Only TTA 37.50%，Context-Only TTA 35.00±1.74，**CoAdapt-GUI 45.00±1.86（5 次运行）**——比 Policy-Only 高 7.5 点，比 Context-Only 高 10 点。

**设定二：AndroidWorld Plus（作者自建，测"同 App 内全新任务模板"）。** 并入 B-MoCA、AndroidLab 应用，共 25 App/191 模板，源 12 App（96 模板）与目标 13 App（95 模板）不相交；目标 App 内适配与评测模板也不相交（60 模板×5 实例为适配池，35 模板×3 种子共 105 个留出 episode）。基座 frozen Qwen3-VL-8B + 源端训过的 LoRA，20 轮/App（≤160 次 rollout）。总体成功率：Base 38.6% → Static 43.3% → Policy-Only 40.0% → Context-Only 48.1% → **CoAdapt-GUI 52.9%（净增 14.3 点）**。

按目标 App 类别能否命中源库再拆更有信息量：
- **Category-Shared（6 App）**：CoAdapt-GUI 70.4%，对比 Policy-Only 53.7%、Context-Only 63.9%；
- **Category-Novel（7 App，源检索为空，从零开始）**：Base=Static=29.4%，**Policy-Only 反跌至 25.5%（纯调策略负迁移）**，Context-Only 31.4%、CoAdapt-GUI 34.3%——没有源流程可抄时，目标端自己演化上下文仍能涨点。

一个漂亮的 case：Chrome 适配中前 11 轮所有变体奖励为 0，第 12 轮某个演化出的变体拿到 4 条轨迹平均 0.25 的奖励——上下文演化"炸出"了策略原本观察不到的成功信号，再反哺策略通道。成本：一次 20 轮单 App 自适应在 NVIDIA H200 上约 9–10 GPU 小时。

## 4. 亮点与贡献（Why it matters）

1. 把 TTA 明确拆成"上下文 + 策略"两个目标端状态，用受控对照证明它们是**互补通道**而非冗余表达。
2. 给出工程上可执行的**防负迁移口径**：app-bound/transferable 分离 + eligibility/linter + 按 Google Play 类别匹配检索，让"哪条知识能过界"可审计、可解释。
3. 只在"任务+上下文"同组内算相对优势，**剔除了上下文差异对策略归因的混淆**，比朴素 on-policy 更新更稳。
4. 证明显式流程演化在稀疏奖励下有**"信号放大"价值**（第 12 轮才见 reward 的 case），流程知识对策略学习有导引作用。
5. 自建 AndroidWorld Plus，把泛化评测从"同模板新实例"推到"同 App 内全新任务类型"，且全程严守泄漏控制（源/目标 App 不相交、适配与评测任务不相交、评测前冻结）。

## 5. 局限与可改进点（个人点评）

- **评测偏窄**：全部在 Android 模拟器、约 18 个目标 App，Category-Novel 仅 7 App、绝对成功率 34.3%；结论能否外推到更长尾分布及 Web/Desktop 尚未可知。
- **强依赖"可重置模拟器 + 程序化奖励"**：受控 reset、按种子复跑、可执行奖励是整套匹配 rollout 的前提；真实设备的不可逆操作与无奖励场景无法直接平移，论文也未谈需要的代理信号。
- **上下文演化效率成谜**：每个"子代"都要靠后续真实 rollout 打分才能上位，20 轮 ≤160 次/app 的预算能认真评估的候选有限；文中没报告"过校验多少、被奖励淘汰多少"的中间统计，无法判断反思-修订环节的性价比。
- **源库质量只拦结构不拦语义**：源流程来自审过轨迹 + 合成器，若反射环节带入系统性错误，linter 挡得住格式违规，挡不住"合规但做法不对"。
- **比较口径小瑕疵**：设定一中 Base/Policy-Only 直接引用他人报告值，自家方法报 5 次均值±std，未做显著性检验；Policy-Only 在 Category-Novel 比 Base 低 3.9 点这一明显失败模式仅一笔带过，没有分析"纯策略通道为何在无流程知识时崩溃"。
- **规模化开销未讨论**：单 App 9–10 GPU 小时尚可，覆盖海量长尾 App 的总成本与跨 App 复用问题都未涉及。

## 6. 对我们的启示 / 可借鉴点

- **"显式流程慢更新 + 参数快更新"双层框架**值得吸收：上下文按轮演化（可解释、可审计），策略攒够组才微调（frozen backbone + LoRA，成本低）。凡是能量化为 reward 的环节都能拆成两层，而非把记忆一把塞进权重。
- **同组相对优势的纪律**：算 reward 差分前先问"输入条件（任务难度、上下文版本）是否相同"——任何 online/RL 项目都可低成本照做，防混淆。
- **"先让流程看到信号，再动策略"的顺序感**：Category-Novel 上纯策略更新倒退、加流程演化才转正，提示稀疏奖励下先演化显式流程、再让策略跟学更稳。
- **"候选先验证后生效"的反自欺 gate**：反思出的修订不过夜生效，须过 lint + 下轮实测淘汰——可直接搬进含 Reflection 环节的流水线。
- **评测纪律本身即是贡献**：适配/评测任务不相交、冻结后再评测、固定 manifest、多轮重复取均值，这套防泄漏检查单建议写进我们评测框架的默认约束。

## 7. 延伸阅读

- **Gu et al. 2026**，*Generalization in Online Reinforcement Learning for Mobile Agents*（arXiv:2603.07432）：Policy-Only TTA 基线与 AndroidWorld-Generalization 设定出处。
- **Guo et al. 2026**，*Agent-SAMA: State-Aware Mobile Assistant*（AAAI 2026）：screen-transition FSM 与恢复路径的思想来源。
- **Rawles et al.**，*AndroidWorld*（arXiv:2405.14573）：主评测环境，AndroidWorld Plus 的底座。
- **Shao et al. 2024**，*DeepSeekMath*（arXiv:2402.03300）：group-relative 策略优化目标的出处。
- **Xiao et al. 2026**，UI-Mem：源端"经验记忆+策略"联合学习的对照。
- **Liu et al. 2025a（LearnAct）/ Verma et al. 2024（AdaptAgent）**：靠少量示范而非自主奖励的目标端自适应路线。
- **Sun et al. 2020**：Test-Time Training，测试时自适应范式源头。

---
*解读生成时间：2026-09-03 ｜ 解读人：WorkBuddy（AI）*
