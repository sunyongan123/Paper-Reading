# CoAdapt-GUI: Joint Workflow Context and Policy Adaptation for Unseen GUI Applications

> 一句话 TL;DR：把移动 GUI agent 部署到训练外 App 的"流程知识"和"操作策略"两个状态，用测试时自主 rollout+奖励联合自适应，在两个 unseen-app 评测上分别达 45.0% / 52.9%。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | CoAdapt-GUI: Joint Workflow Context and Policy Adaptation for Unseen GUI Applications |
| **作者 / 机构** | Linqiang Guo、Li Gu 等（第一作者 Guo Linqiang）；Concordia University、Mila – Québec AI Institute、University of Toronto、McMaster University（加拿大） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-08-12；arXiv preprint（cs.AI，v1） |
| **arXiv 链接** | https://arxiv.org/abs/2608.11588 |
| **代码仓库** | ❌ 未开源（全文无代码链接） |
| **数据集地址** | 复用已发布的 AndroidWorld-Generalization（Gu et al. 2026）；自建 AndroidWorld Plus **未公开** |
| **类型标签（论文类别）** | `RL` `Online` `Mobile` `Reflection` |
| **训练方法标签** | `RL (GRPO)`（group-relative 目标，源自 DeepSeekMath）`Online RL` `LoRA`（frozen VLM 上测试时自适应） |
| **关键词** | 测试时自适应（TTA）、GUI 智能体、跨应用泛化、工作流上下文、LoRA、在线强化学习 |
| **来源渠道** | github-awesome-OSU（OSU-NLP-Group/GUI-Agents-Paper-List） |
| **PDF 存档** | papers/2026-08-12_CoAdapt-GUI_Joint_Workflow_Context_and_Policy_Adaptation_for_Unseen_GUI_Applications.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：移动 GUI agent 部署到**训练集之外的新 App** 时，如何仅凭受限的自主交互预算（无目标端示范）在评测前自适应。属于 Mobile + 通用 Agent 的 test-time adaptation（TTA）场景。

- **为什么重要**：现有 GUI agent 的训练/评测都假设"App 集合预先固定"；真实部署中用户打开的多半是不在训练集里的 App。agent 撞上新 App 会同时暴露**两类缺口**——① 界面缺口（读懂目标却无法把动作 grounding 到陌生界面元素）；② 流程缺口（单个动作会做，但缺完整流程、易错点与"何时算完成"的校验）。只补其一，另一类失败源仍在。

- **现有方法有什么不足**：论文明确点名三类结构性缺陷——① **AndroidWorld-Generalization（Gu et al. 2026）** 只从目标端 rollout 更新策略，工作流上下文冻结不动；② **UI-Mem（Xiao et al. 2026）** 在源端联合学"经验记忆+策略"再整体迁移，未回答"撞上新 App 之后"能否继续自适应；③ **LearnAct / AdaptAgent** 依赖少量示范而非 agent 自身带奖励的交互。最有力的论据来自 Gu et al. 2026 的数据：源端 online RL 在同界面新任务上能给 7B 策略 +26.1 点，但换到全新 App 时增益骤降到 +8.3 点——"熟悉环境上的新题"与"陌生 App 里的老题"不是一回事。

- **Research Gap**（作者视角，忠实转述）：agent 遭遇 unseen App 后，能否只用它自己收集的交互与任务级奖励，**同时**把显式工作流上下文和策略两个状态改好，再在没碰过的 held-out 任务上验收？此前无人同时自适应这两者。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把"会不会用这个 App"拆成两个独立、可单独验证、又互相喂数据的目标端状态——**显式工作流上下文**（可读的类型化知识）+ **参数化策略**（frozen VLM 上的 LoRA），在目标 App 上跑"取轨迹 → 拿奖励 → 双通道更新"的循环同时改进。

### 3.2 方法总览（Pipeline）

- **输入**：任务指令 q、当前截图观测 o_k、交互历史 h_k、渲染后的工作流上下文 C_t(q)；以及可重置模拟器给出的可执行任务奖励 r(τ)。
- **输出**：评测前冻结的最优工作流状态 M* 与 LoRA 参数 θ*，在 held-out 任务上只做推理。
- **模块与连接**（见 Algorithm 1）：
  1. **源端初始化（离线）**：冻结的 synthesizer 从审过轨迹构建每个源 App 的 app-bound 屏跳转 FSM + transferable 工作流，前者永不跨 App；后者过 eligibility/linter 后按 Google Play 类别归并成不可变源库 L_src。另在源轨迹上训一个共享 LoRA（上下文固定）。
  2. **目标端检索**：按类别从 L_src 检索有界先验 M0(q)；类别无命中则返回空先验（不强制塞不相关知识）。
  3. **适配循环（核心）**：每轮选一个适配任务 q 与若干"已实体化"上下文变体，在受控 reset 下采 matched rollouts D_t（记录 (任务,上下文,reset) 三元身份）。
  4. **上下文通道**：reward 喂 TrueSkill 给变体打分；冻结的 reflector 对比成败轨迹、向高分父条目提类型化修订（add/modify/remove），新子代过 schema/来源/lint 三关才进候选群体，且**不继承本轮奖励**，须下轮实测拿分才上位。
  5. **策略通道**：轨迹按 (任务,上下文) 分组，组内算 group-relative advantage，有非简并组就做一次 LoRA 更新，随后清空 buffer、轨迹不重用。
  6. **两通道交互耦合而非联合可微**：当前上下文决定策略要学的轨迹，当前策略又决定未来修订上下文所用的成败样例。预算耗尽取最高分上下文+最终 LoRA，**冻结**后再评测。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：① 把 TTA 明确拆成"显式工作流上下文 + 策略"两个独立目标端状态并**从同一批 rollout 联合更新**；② **transfer-constrained 上下文**——app-bound/transferable 分离 + eligibility/linter 硬校验，是前人未做的"知识跨界治理"机制；③ **task–context-matched group-relative 目标**——只在（任务,上下文）都相同的组内算相对优势，显式排除"上下文差异"对策略归因的混淆。
- **工程组合**：group-relative 目标（DeepSeekMath/GRPO）、LoRA（Hu et al.）、TrueSkill（Herbrich 2006）、屏跳转 FSM（Agent-SAMA）、frozen synthesizer/reflector = Claude Opus 4.7——这些组件本身都不新，新在把它们按"验证-淘汰-冻结"的时序拼成闭环。
- **对性能提升最关键的设计**：**上下文通道是绝对主力**。Setting 2 增量分解（Table 12）：Static−Base 仅 +4.7，Policy-Only−Base 只有 +1.4（且在 Category-Novel 上 −3.9 负迁移），而 Context-Only−Static +4.8、CoAdapt−Context-Only 再 +4.8。策略只在"有演化上下文提供信息量"时才有价值——这是全文最重要的经验结论。
- **证据不足 / 仅声称有效**：① eligibility/linter/transfer-constraint 被反复强调防负迁移，但**没有"去掉 linter"的消融**，其定量贡献未被验证；② Policy-Only 在 Category-Novel 上比 Base 低 3.9 点这一明显失败模式只一笔带过，未解释"纯策略通道为何在无流程知识时崩溃"；③ Setting 1 的 Base/Policy-Only 直接引用 Gu et al. 报告值，与自家 5 次均值±std 混排，且未做显著性检验。

---

## 4. 具体技术细节

### 4.1 模型结构

- **base model 是 MLLM（含视觉编码器）**，两个设定不同：Setting 1 用 **UI-TARS-7B**（step-500 checkpoint）；Setting 2 用 **Qwen3-VL-8B-Instruct**（bfloat16）。
- **冻结 backbone + LoRA 微调**：VLM 全程冻结，只训 **rank-16 LoRA**（目标模块 `q_proj`、`v_proj`）。源端训一个共享 LoRA 作为初始化与策略锚点（anchor），目标端只在这个 adapter 上继续在线更新。

### 4.2 训练流程（含测试时自适应）

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| 源端初始化（离线） | 得到可跨界的流程知识 + 初始策略 | 流程归纳、策略基座 | 源 App 审过轨迹（Setting 1 用 12 App/62 模板/905 实例；Setting 2 用 12 App/96 模板/480 episode） | 轨迹 + 任务奖励；synthesizer 产出 ⟨c,P,F,V⟩ 类型化条目 | 无梯度损失；LoRA 用源轨迹做策略训练（上下文固定） |
| 目标端 TTA（在线） | 同时改好流程与策略 | 目标 App 的流程纠偏 + 动作策略 | agent 自身在 Q_adapt 上的 rollout + 可执行奖励 | 受控 reset 的 (轨迹,奖励,上下文变体,reset 身份) 四元组 | **上下文通道**：reward→TrueSkill 打分，无梯度；**策略通道**：L_policy = −(1/|B_act|)Σ A_j·ℓ_j(θ) + β·R(θ;π_anchor)，其中 A_j=(r_j−mean_G)/s_G 仅在 (task,context) 组内算，singleton/常数奖励组不给梯度 |

关键设定写实：advantage 用组内**均值中心化**（s_G=1）；优化器 AdamW lr=3e-4，梯度范数裁剪 1.0，log-ratio 裁剪 ±10，anchor 系数 β=0.05；至少 3 条活跃轨迹才更新；dropout 关闭（用采样种子而非 dropout 制造组内多样性）；策略 buffer 是**瞬时 on-policy 缓冲**，每次更新尝试后无论有无活跃组都清空，轨迹绝不重用。上下文侧：TrueSkill 初始均值 25.0/标准差 8.33，种群窗口 15，采样 softmax 温度 T=1；每轮选 ≤2 个变体、每个 (task,context) 采 N=4 条 rollout，20 轮/App ≤160 次 rollout。

### 4.3 推理流程（评测阶段）

- **多步交互循环**：每步 agent 依据（任务 q、当前截图 o_k、历史 h_k、渲染上下文 C_t(q)）产出动作 u_k，执行后观测更新，直到满足上下文里的完成校验 V 或达步数上限（Setting 1 名义 50 步/app）终止。本质是带"显式流程知识注入 + 完成校验"的 grounding 执行循环，非纯 ReAct。
- **评测纪律**：adaptation 与 evaluation 任务/种子不相交，适配结束后 M* 与 θ* 冻结，reflector/种群控制器/优化器全部关闭，只用 held-out manifest 跑推理；程序化评估器判定成功率（SR）。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| AndroidWorld-Generalization（Setting 1） | Mobile | 端到端多步操作 | 源 12 App/62 模板/905 实例；目标 5 不相交 App，8 适配实例/App（共 40）+ 48 held-out 实例，16 模板 | 截图 + 自然语言指令 + 渲染上下文 | 动作序列（tap/type 等） | 成功率 SR（%） |
| AndroidWorld Plus（Setting 2，自建） | Mobile | 端到端、跨任务模板泛化 | 25 App/191 模板；源 12 App(96 模板)/目标 13 App(95 模板) 不相交；60 适配模板×5 种子=300 池，35 留出模板×3 种子=105 episode | 截图 + 指令 + 渲染上下文 | 动作序列 | 成功率 SR（%），按 Category-Shared/Novel 拆 |

### 5.2 实验结果分析

**主结果（成功率）**：

- **Setting 1（新 App + 见过模板的新实例）**：Base 27.10% → Static 28.75±2.28 → Policy-Only TTA 37.50%（引用）→ Context-Only TTA 35.00±1.74 → **CoAdapt-GUI 45.00±1.86（5 次运行）**。比 Policy-Only 高 7.5 点，比 Context-Only 高 10.0 点。

- **Setting 2（同 App 内全新任务模板）**：Base 38.6% → Static 43.3% → Policy-Only 40.0% → Context-Only 48.1% → **CoAdapt-GUI 52.9%（净增 14.3 点）**。按类别再拆：**Category-Shared（6 App）** CoAdapt 70.4% vs Policy-Only 53.7% / Context-Only 63.9%；**Category-Novel（7 App，源检索为空）** Base=Static 29.4%，**Policy-Only 反跌至 25.5%**，Context-Only 31.4%、CoAdapt 34.3%。

- **Ablation/增量分解说明了什么**（Table 12，Setting 2 增量）：
  - Static−Base：+9.3(Shared)/0.0(Novel)/+4.7 —— 静态上下文只在类别命中时有效，符合"按类别检索"预期；
  - Policy−Base：+6.5/−3.9/+1.4 —— **纯策略 TTA 在没有流程知识时反而负迁移**，是全篇最反直觉也最关键的信号；
  - Context−Static：+7.4/+2.0/+4.8 —— 目标端从零演化上下文，对两类都有效；
  - CoAdapt−Context：+6.5/+2.9/+4.8 —— 策略只在叠加于演化上下文之上时才稳定增益。
  - 一个定性 case：Chrome 适配前 11 轮所有变体奖励为 0，第 12 轮演化出的变体拿到 4 条轨迹平均 0.25 的奖励（root 仍为 0）——上下文演化"炸出"了策略原本观察不到的成功信号。
  - 成本：单 App 20 轮自适应约 9–10 GPU-hours（NVIDIA H200 141GB）。

---

## 6. 亮点与贡献（Why it matters）

1. 用**受控对照（五种配置拆通道）**证明"上下文 + 策略"是**互补而非冗余**：Context-Only 已在两设定全面超越 Policy-Only，联合又再进一步。
2. 给出工程可落地的**防负迁移口径**：app-bound/transferable 分离 + eligibility/linter + 按 Google Play 类别匹配检索，让"哪条知识能过界"可审计、可解释。
3. **task–context-matched group-relative** 目标，把"上下文差异"从策略奖励归因中剥离，比朴素 on-policy 更新更稳——对做 online RL 的人是可直接复用的纪律。
4. 证明**显式流程演化在稀疏奖励下的"信号放大"价值**（Chrome 第 12 轮才见 reward 的 case）。
5. 自建 AndroidWorld Plus 把泛化评测从"同模板新实例"推到"同 App 内全新任务类型"，并严守泄漏控制（源/目标 App 不相交、适配与评测模板/种子不相交、评测前冻结）。

---

## 7. 局限与可改进点（个人点评）

- **评测范围偏窄**：全在 Android 模拟器、约 18 个目标 App，Category-Novel 仅 7 App、绝对成功率 34.3%；结论能否外推到 Web/Desktop 及更长尾分布未知。
- **强依赖"可重置模拟器 + 程序化奖励"**：受控 reset、按种子复跑、可执行奖励是整套 matched rollout 的前提；真实设备的不可逆操作与无奖励场景无法直接平移，论文也未谈替代的代理信号。
- **上下文演化的"候选-淘汰"效率成谜**：每个子代都要靠后续真实 rollout 打分才能上位，20 轮 ≤160 次/app 的预算能认真评估的候选有限；文中没报告"过校验多少、被奖励淘汰多少"的中间统计，反思-修订环节的性价比无法判断。
- **源库质量只拦结构不拦语义**：linter 挡得住格式违规（App 名/坐标/资源 id），挡不住"合规但做法错误"的系统性错误被反射环节带入。
- **比较口径小瑕疵**：Setting 1 的 Base/Policy-Only 是他人报告值，与自家 5 次均值±std 混排，未做显著性检验；Policy-Only 在 Category-Novel 的负迁移这一关键失败模式被轻描淡写，未剖析根因。
- **规模化开销未讨论**：单 App 9–10 GPU-hours 尚可，覆盖海量长尾 App 的总成本与跨 App 复用未涉及。

---

## 8. 对我们的启示 / 可借鉴点

- **"显式流程慢更新 + 参数快更新"双层框架**值得吸收：上下文按轮演化（可解释、可审计），策略攒够有效组才微调（frozen backbone + LoRA，成本低）。凡能量化为 reward 的环节都可拆成两层，而非把记忆一把塞进权重。
- **同组相对优势的纪律**：算 reward 差分前先问"输入条件（任务难度、上下文版本）是否一致"，任何 online/RL 项目可低成本照做，防混淆。
- **"先让流程看到信号，再动策略"的顺序感**：Category-Novel 上纯策略更新倒退、加流程演化才转正，提示稀疏奖励下**先演化显式流程、再让策略跟学**更稳。
- **"候选先验证后生效"的反自欺 gate**：反思出的修订不过夜生效，须过 lint + 下轮实测淘汰，可直接搬进含 Reflection 环节的流水线。
- **评测纪律本身即是贡献**：适配/评测任务不相交、冻结后再评测、固定 manifest、多轮重复取均值——这套防泄漏检查单建议写进我们评测框架的默认约束。

---

## 9. 延伸阅读

- **Gu et al. 2026**，*Generalization in Online Reinforcement Learning for Mobile Agents*（arXiv:2603.07432）：Policy-Only TTA 基线与 AndroidWorld-Generalization 设定出处。
- **Guo et al. 2026**，*Agent-SAMA: State-Aware Mobile Assistant*（AAAI 2026）：屏跳转 FSM 与恢复路径的思想来源。
- **Rawles et al. 2025**，*AndroidWorld*（arXiv:2405.14573）：主评测环境，AndroidWorld Plus 的底座。
- **Shao et al. 2024**，*DeepSeekMath*（arXiv:2402.03300）：group-relative 策略优化目标出处。
- **Xiao et al. 2026**，*UI-Mem*（arXiv:2602.05832）：源端"经验记忆+策略"联合学习的对照。
- **Liu et al. 2025（LearnAct，arXiv:2504.13805）/ Verma et al. 2024（AdaptAgent，arXiv:2411.13451）**：靠少量示范而非自主奖励的目标端自适应路线。
- **Sun et al. 2020**，*Test-Time Training*（ICML 2020）：测试时自适应范式源头。

---
*解读生成时间：2026-09-10 ｜ 解读人：WorkBuddy（AI）*
