# Improving Proficiency and Efficiency of Android GUI Agents via Self-Generating Tool Actions

> 一句话 TL;DR：让 Android GUI Agent 通过 agentic 工作流自生成作用于应用状态的工具，经关系化真机验证后嵌入混合动作空间，成功率更高、步数更省。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Improving Proficiency and Efficiency of Android GUI Agents via Self-Generating Tool Actions |
| **作者 / 机构** | Juyong Lee\*；Woogyeol Jin\*；Kimin Lee（通讯）；韩国科学技术院 KAIST（韩国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-06；arXiv preprint（cs.AI） |
| **arXiv 链接** | https://arxiv.org/abs/2609.06792 |
| **代码仓库** | ❌ 未开源（文中未声明） |
| **数据集地址** | 未公开（评测环境 AndroidWorld / B-MoCA / MobileSafetyBench 均为既有公开基准） |
| **类型标签（论文类别）** | `Mobile` `GUI Grounding` |
| **训练方法标签** | `—（无训练；基于 MLLM Gemini-3.5-flash 的提示 + agentic 工具生成工作流）` |
| **关键词** | Android GUI Agent；工具生成；混合动作空间；应用状态；关系化验证；agentic workflow |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-06_Improving_Proficiency_and_Efficiency_of_Android_GUI_Agents_via_Self-Generating_Tool_Actions.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：在手机（Mobile）GUI Agent 场景中，如何以最小人工成本为 Agent 自动生成「直接操作应用状态」的工具动作（tool action），并让工具与 GUI 动作在同一动作空间里协同。
- **为什么重要**：纯 GUI 路线虽是通用接口，但简单任务也要十几步（导航、定位、逐字输入），既慢又脆。工具动作（经 API / 数据库直接改状态）能把「进便签→定位文字→替换」压成一次写入，是通往低成本、长任务手机 Agent 的关键杠杆；若无人解决「造工具太贵」，这条捷径就永远打不通。
- **现有方法有什么不足**：论文明确批评两类基线——① GUI+Skill（skill.md）仍停留在「照着点按流程走」，依旧是 GUI 动作，且常牺牲安全；② GUI+CLI 给通用 shell，命令易语法错、权限错（实测 CLI 命令成功率仅 83.77%，573/684）。二者都未提供「针对 App 状态、预先验证过的语义工具」。更深层的死结在**验证**：mock 单测在真机上常崩（假阴性），真机测又摆不好前置条件（测读要先有数据、测删要先造条目）。
- **Research Gap**：作者认为补上的缺口是——「如何以最少人力（仅每 App 一份 Developer Document）为 GUI Agent 自造并验证可靠的工具函数，且工具与 GUI 动作混合执行」。其 claim 是：工具验证必须**关系化**（让相关工具互为测试前置条件），才能既提覆盖率又不误杀好工具。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把「造工具」本身交给一个 LLM agentic 工作流，产出作用于应用状态的 Python 函数，经关系化真机验证后注册为 Agent 的可调用动作，与 GUI 动作混合使用。

### 3.2 方法总览（Pipeline）

- **输入**：人类为每个 App 写一份 Developer Document（描述状态实体、字段、访问路径、一致性约束，**不含源码**）；**输出**：一套经验证、注册为动作选项的工具函数（含元数据）。
- **模块**（五段）：① **提案与规格**——design agent 依文档按「实体生命周期」提工具，specification agent 转成结构化实现契约；② **实现**——implementation agent 写成 Python 模块（函数带 success 标志 + error 字符串），规则检查器验语法合规，不合规重生成；③ **测试生成与执行**——test agent 生成「入参 + 断言」测试，在真机模拟器执行；④ **修复与注册**——repair agent 分析失败、只改缺陷部分再跑旧测试，循环直到全过或达迭代上限；通过者注册。
- **连接方式**：数据流为 文档→提案→规格→代码→测试→修复，控制流上「实现—测试—修复」构成闭环；注册后的工具经函数调用进入 Agent 交互循环，结果写入历史上下文供后续决策。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：**关系化测试（relation-aware test）**——不为每个工具单测，而是把操作同一状态实体的工具按依赖排成调用序列（create→read→delete），让前面的工具为后面的工具自然造好前置条件；缺少前置条件则跳过且**跳过不算通过**。这同时解决真机测试摆前置条件的死结与 mock 假阴性问题。
- **工程组合**：design/specification/implementation/repair 各 agent 本身不新，是「微型软件开发」流水线的移植；作用于 SQLite / SharedPreferences / ContentProvider / 文件的 Python 工具 + ADB 执行也沿用 CLI 路线。
- **对性能提升最关键的设计**：关系化验证 → 修复闭环。它让注册工具从 82（Separate）提到 113，工具调用成功率从 74.86%（Unit）提到 90.20%，是主结果提升的直接来源。
- **证据不足 / 仅声称有效**：未对「工具改了状态但 UI 呈现错误」这类副作用错做 GUI 层验证，作者也承认 GUI 校验缺位；成本（API 消耗、修复轮数）未量化。

---

## 4. 具体技术细节

### 4.1 模型结构

- **Base model**：Gemini-3.5-flash（DeepMind，闭源商用）。Agent 每步接收**屏幕截图 o_t** 与历史上下文 h_t，故其决策主体是 **MLLM（含视觉编码器）**，非纯文本 LLM。工具生成各阶段 agent 同样用 Gemini-3.5-flash。
- **参数 / 微调**：参数未披露；**全程无训练、无 LoRA/微调**，靠提示 + agentic 工作流驱动。

### 4.2 训练流程

无训练阶段，故无 Stage 表。方法核心在「工具生成」与「验证修复」两个工程阶段（对应第 3.2 节），非模型权重更新。

### 4.3 推理流程

- **多步交互循环**：每步 Agent 看截图 + 历史 → 在混合动作空间 A = A_GUI ∪ A_tool 中选一个动作（触屏/滑/输入，或函数调用某工具）；工具返回结果进入 h_{t+1} 供后续决策，循环直到任务完成。终止条件由各 benchmark 环境的任务反馈判定。
- **工具调用**：Agent 依注册元数据（函数名、入参、输出 schema、简短描述）经 function calling 调工具；工具经 ADB 在设备上执行。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| AndroidWorld (AW) | Mobile | 端到端 App 操作 | 116 任务 | 截图 + 指令 | 混合动作（GUI/工具） | 任务成功率 SR、交互步数 |
| B-MoCA (BMC) | Mobile | 基本场景 + 应用设置配置 | 119 任务（剔除 Instagram/Walmart 远程服务类） | 截图 + 指令 | 同上 | SR、步数 |
| MobileSafetyBench (MSB) | Mobile | 安全相关日常任务 | 100 日常场景任务 | 截图 + 指令 | 同上 | 低风险：熟练分；高风险：安全分 |

> 三基准合计 335 任务、29 个 App，统一 Gemini-3.5-flash，三次运行取均值±标准差。

### 5.2 实验结果分析

- **主结果（Table 1）**：DroidTool 总体成功率 **79.20%**，比 GUI-only（74.73%）高 **4.47 个百分点**；分基准 AW 77.30 vs 69.83、BMC 82.35 vs 78.71、MSB 77.67 vs 75.67。平均交互步数从 **8.13 降到 6.50（约省 20.05%）**，B-MoCA 闹钟设置等复杂任务省 10 步以上。对照：GUI+Skill 71.44（反而更差）、GUI+CLI 77.01（CLI 命令成功率仅 83.77%）。
- **Ablation（Table 2/3，验证策略）**：从 121 个提案工具中，Relation 注册 113 个（Unit 101、Separate 82）；工具调用成功率 Relation **90.20%**（Unit 74.86%、Separate 78.76%、Unverified 73.41%）；基准成功率 79.20% vs 78.61%（Unit）/ 76.22%（Separate）/ 74.03%（Unverified）。
  - 结论一：**关系化验证 + 修复是主因**——既提覆盖率（注册最多）又提可靠性（调用成功率最高）。
  - 结论二：**验证设计不当反伤能力**——Unit/ Separate 未一致优于 Unverified，甚至更低；mock 曾把 Joplin 其实好用的 search_notes 误杀（假阴性），而 Relation 在真机上能区分「真缺陷」与「fixture/mock 不完备」。
  - 结论三：未验证工具已有部分收益（AW 70.98 vs 69.83、MSB 79.67 vs 75.67），说明工具本身有用、可靠性靠验证兜底。

---

## 6. 亮点与贡献（Why it matters）

1. **打通「自动造工具→验证→嵌入混合动作空间」全链路**：把人工成本从「逐工具手写」压到「每 App 一份状态文档」，是真正可规模化的工程步。
2. **关系化验证是方法论亮点**：让工具互为测试前置条件，一举解决真机测试摆前置条件的死结，同时提覆盖率、可靠性，并避免 mock 假阴性误杀好工具。
3. **实证扎实**：三个代表性基准、统一主模型、三种验证策略 + 未验证对照，量化了「验证设计不当反而更糟」这一反直觉结论。
4. **为效率型 GUI Agent 提供新杠杆**：混合动作空间下 Agent 自由决定「点屏 or 调工具」，是可复用的持久资产，成本随多任务摊薄。

---

## 7. 局限与可改进点（个人点评）

- **Developer Document 仍是人工瓶颈**：每个新 App 都要人写文档，版本升级后文档与状态会漂移；「无需源码」的宣称实际把成本转移给了文档维护。
- **工具静态、离线生成**：任务执行前一次生成完毕，运行中不能按需新增/裁剪/修复，对长尾、动态需求无力。
- **验证只覆盖「函数不报错」**：工具调用成功率高 ≠ 任务结果正确，改了状态但 UI 呈现错误这类副作用错未被 GUI 层测试，作者也承认缺位。
- **单一闭源模型耦合**：所有结论建立在 Gemini-3.5-flash 上，工具生成质量与 Agent 决策质量耦合，换基座未必同增益；且无成本对比（API 消耗、修复轮数）。
- **安全维度交代较粗**：MSB 上工具既用于「高效完成」也用于「核查敏感上下文」，但缺少 least-privilege、用户授权等滥用边界机制（正文仅一句 broader impact 带过）。

---

## 8. 对我们的启示 / 可借鉴点

- **「先写状态文档、再生成工具」的中间层思想可直接落地**：与其让 Agent 去猜数据库 schema，不如给一套紧凑、人可维护的「状态接口清单」，让 grounding 与工具调用都建立在这层可信接口上。
- **关系化验证值得移植到我们的评测/回归体系**：当项目存在互相依赖的能力模块（取元素→点击→断言状态变化）时，把它们串成前置条件链，比逐个单测更接近真实使用、更难造假。
- **为 RL 提供现成训练环境**：混合 GUI+工具动作空间正是「何时切捷径」的决策问题，可直接作为后续用 RL/GRPO 学习工具调用策略的环境，契合项目 RL 与 GUI Grounding 方向。

---

## 9. 延伸阅读

- **PhoneHarness（Li et al., 2026）**：混合 GUI/CLI/工具动作的手机 Agent，同赛道互补。
- **ToolCUA（Hu et al., 2026）／ UltraCUA（Yang et al., 2025a）**：GUI 与工具路径最优编排、混合动作基座模型。
- **MAS-Bench（Zhao et al., 2026）**：针对「含捷径的混合手机 GUI Agent」的统一评测，可作扩展基准。
- **MobileSafetyBench（Lee et al., 2026b）／ AndroidWorld（Rawles et al., 2025）**：本文主要评测环境。
- **skill.md 路线（Agrawal et al., 2026；Yang et al., 2026）**：GUI+Skill 基线来源，可对比「技能文档 vs 状态工具」两条自增强路线。

---
*解读生成时间：2026-09-10 ｜ 解读人：WorkBuddy（AI）*
