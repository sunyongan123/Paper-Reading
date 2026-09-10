# OmegaUse-SOP: SOP Engineering for Professional Computer Use from Human Demonstrations

> 一句话 TL;DR：把专家演示工程化为可复用 SOP 技能（录制→语义抽象→注入规则→逐步校验），使 GUI Agent 在 PVsyst 专业流程上从 1~3/5 提升到 5/5。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | OmegaUse-SOP: SOP Engineering for Professional Computer Use from Human Demonstrations |
| **作者 / 机构** | Yixiong Xiao 等；Jingbo Zhou（通讯 / Project lead）；Baidu, Inc.（中国北京）+ Ningxia Electric Power Engineering Co., Ltd.（宁夏电力工程有限公司，中国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-02；arXiv preprint（cs.HC，v1，未标注会议） |
| **arXiv 链接** | https://arxiv.org/abs/2609.02149 |
| **代码仓库** | ⚠️ 部分开源：可执行演示包 https://github.com/baidu-frontier-research/omegause-sop ；人在回路开源实现 https://github.com/ethanyxx/co-work ；完整系统与数据未开源（HITL 完整实现部署在客户环境） |
| **数据集地址** | 未公开（本文为 case study，无标准数据集，仅有 5 个来自客户真实工作流的 PVsyst 任务） |
| **类型标签（论文类别）** | `Desktop` `GUI Grounding` `Planning` |
| **训练方法标签** | —（工程系统：Human Demonstration → SOP 技能工程，无模型训练） |
| **关键词** | SOP Engineering、GUI Agent、Human Demonstration、技能复用、逐步校验、PVsyst |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-02_OmegaUse-SOP_SOP_Engineering_for_Professional_Computer_Use_from_Human_Demonstrations.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：GUI Agent 在**专业桌面软件里执行"规范操作流程（SOP）"**——即光伏仿真软件 PVsyst 7.2 中的真实业务工作流（气象数据导入、平面朝向设置、并网系统设置、详细损耗设置、仿真执行）。

- **为什么重要**：专业软件里的成败往往不取决于"会不会点鼠标"，而取决于模型是否掌握屏幕上看不出来的隐含领域知识、软件自身约定（如旋钮上下箭头各代表什么）、校验习惯，以及每次任务都不同的参数。若 GUI Agent 只具备通用感知/动作能力，在这些场景会稳定翻车，无法落地真实行业。

- **现有方法有什么不足**：①通用 benchmark（OSWorld）只测"会用电脑"，测不出"会按行业规范办业务"；②GUI 专门模型（UI-TARS、GUI-Owl、OmegaUse）与 Agent 框架（Agent S2、UFO2）都押注更强的感知/动作/规划，但都**没有"捕获并复用专家流程知识"的机制**；③SOP-Bench、Workflow-GYM 已实证 LLM Agent 在分支逻辑、阶段遗漏、错误传播、目标漂移上大量失败；④结构上，坐标回放跨界面即失效、上百步 SOP 全量塞上下文会撑爆窗口、黑盒策略不可审查。

- **Research Gap**：作者认为缺的不是更强的模型，而是"把专家的程序性知识捕获、语义化、参数化、可校验地复用"这一机制——由此提出 **SOP Engineering**，类比成 Prompt Engineering 式的迭代打磨过程。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

SOP Engineering：像 Prompt Engineering 一样，对着真实环境反复迭代打磨"演示 + 执行规则 + 领域知识 + 任务参数"，把专家的专业操作演示工程化为可复用、可校验、可参数化的 GUI Agent 技能。

### 3.2 方法总览（Pipeline）

- **输入**：专家在目标专业软件里的一次人工演示（鼠标/键盘事件 + 截图），以及专家事后补充的领域规则与任务参数。
- **输出**：一个"配置好的 SOP 技能"——绑定到具体任务实例、能在真实 GUI 环境逐步执行并校验的流程描述。
- **四个模块（数据流按序）**：
  1. **Observe（录制）**：连续监听鼠标键盘并截屏；每次操作前先存一帧"动作前截图"以保留"先看到、再操作"顺序。坐标类动作（左键/双击/右键）用 OmniParser + PaddleOCRv5 解析点击处 UI 元素并裁出局部图；非坐标动作（Type/Hotkey）按功能聚合成语义事件（连续字符合成一次输入、Enter/Ctrl+C 单记）。产出多模态 SOP trace。
  2. **Reason（语义抽象）**：VLM 结合动作前截图 + 点击区放大图 + 动作后截图，把每步写成语义指令（"在项目名输入框输入项目名""配置完成后保存项目"）。只做 grounding 式补全，**不做新规划**。
  3. **Configure（注入知识）**：两类可编辑上下文——domain SOP guidance（执行规则，如"调串联模块数时点下箭头减、上箭头加""用户指定倾角 35° 优先于历史值"）+ task-specific parameters（把演示中会变的值标记为变量）。
  4. **Execute（执行 + 校验）**：仿 Claude Code 技能调用，按当前步渐进检索该步相关信息喂给模型生成动作；每步执行后对比当前屏与演示期望后屏做结果校验（容忍光标/时间戳等无害差异），偏离则触发人在回路（continue/retry/stop）。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：**SOP Engineering 这一概念框架本身**——把"演示转技能"抽象为 Observe→Reason→Configure→Execute 四阶段 + 迭代闭环。这是贡献主标签，但本质是方法论/系统级创新，非新机制、新 loss、新结构。
- **工程组合**：Observe 的 UI 解析（OmniParser + PaddleOCRv5）、Reason 的 VLM 语义补全、Execute 的渐进式检索（仿编码 Agent 技能调用）与逐步校验，都是成熟组件的组合，单独看无新意，但拼装成端到端闭环后有效。
- **对性能提升最关键的设计**：**Reason 模块**——全文唯一做了消融的组件，去掉后 5/5 → 2/5，证明"语义步骤表示"是系统成立的核心。
- **证据不足 / 仅声称有效**：Configure 的两类知识注入、Execute 的逐步校验、人在回路、Observe 的 UI 解析质量，**均无消融或定量支撑**；系统整体增益是否被 Reason 一己之力解释，未做归因。

---

## 4. 具体技术细节

### 4.1 模型结构

系统本身**模型无关（model-agnostic）**，不训练任何模型。评测驱动 3 个 VLM（均含视觉编码器、全部冻结）：开源权重的 **Qwen3-VL-235B-A22B-Instruct**（MoE，235B 总参 / 22B 激活），以及闭源的 **GPT-5.5** 与 **Opus-4.7**。Reason 模块内部同样调用 VLM 做 grounding（未指明具体型号）。

### 4.2 训练/构建流程（SOP 技能工程）

本文无模型训练，核心是把人类演示转化为技能并验证。按阶段：

| 阶段 | 目标 | 构建什么 | 数据来源 | 数据形态 | 验证 / Loss |
|---|---|---|---|---|---|
| Observe | 记录专家操作 | 多模态 SOP trace | 专家在 PVsyst 的人工演示 | 动作前截图 + UI 元素裁图 + 鼠标/键盘事件 + 时间戳 | 无（人工审核） |
| Reason | 低层事件→语义步骤 | 语义 SOP 表示 | Observe 的 trace | 自然语言步骤指令（对象+位置+意图） | 无独立评测，靠下游成功率间接验证 |
| Configure | 注入领域知识与参数 | 可编辑技能规格 | 专家经验 / 试错后分析 | domain SOP guidance 文本 + 参数变量绑定 | 无（迭代打磨） |
| Execute | 真实环境逐步执行 | 可复用技能闭环 | 前三个模块输出 | 逐步动作 + 结果校验 + 人在回路干预 | 逐步校验（当前屏 vs 期望后屏）+ 专家判定成败 |

### 4.3 推理流程

Execute 是**多步推理**，循环机制为"检索—执行—校验"：每步先截取当前屏、只检索该步相关的 trace/语义/规则/参数（避免上百步撑爆上下文），模型生成 computer-use 动作（点击/输入/热键/滚动/等待/终止），经底层鼠标键盘执行；随后拿执行后屏与演示期望后屏对比，让模型判断是否继续；校验不通过则进入人在回路（continue/retry/stop）。终止条件：全部步骤完成，或校验失败后用户选择终止。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

（注：本文是 case study，非标准 benchmark，下表按任务拆解）

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| PVsyst 7.2 客户工作流 | Desktop（专业光伏仿真软件） | 端到端 SOP 流程执行 | 5 个任务 | 截图 + 高层用户指令 | 鼠标键盘动作序列 | 任务成功率（专家人工判定，best-of-3） |

5 个任务：Meteorological Data Importation（气象数据导入）、Plane Orientation Setting（平面朝向设置）、Grid Connected System Setting（并网系统设置）、Detailed Losses Setting（详细损耗设置，需跨多个子面板配热行为/欧姆损耗/老化等参数，最吃领域知识）、Simulation Execution（仿真执行）。

### 5.2 实验结果分析

- **主结果**（Table 2，每任务试 3 次取最优、领域专家判定）：基线直接执行下 Qwen3-VL 1/5、GPT-5.5 3/5、Opus-4.7 2/5，失败几乎都集中在需专业流程知识的任务（如 Detailed Losses Setting 三模型全败）；加 OmegaUse-SOP 后**三模型全部 5/5**，提升对所有模型一致——说明性能差距的主因是流程知识的表示形式，而非模型强弱。
- **Ablation**（Table 3，Qwen3-VL）：去掉 Reason、直接拿低层 trace 执行，成功率 5/5 → **2/5**（只剩 Plane Orientation Setting 与 Simulation Execution 能过），证明"把演示翻译成语义步骤"是系统成立的关键一环。

---

## 6. 亮点与贡献（Why it matters）

1. 提出 **SOP Engineering** 方法论，把"专家演示→可复用技能"抽象成类 Prompt Engineering 的迭代流程，概念清晰、易被社区接受和复用。
2. 端到端闭环 Observe→Reason→Configure→Execute，覆盖采集、语义抽象、知识注入、带校验执行全链路，天然支持人在回路。
3. 渐进式信息披露（按步检索）解决长流程 SOP 的上下文溢出问题，是做长任务 Agent 的通用技巧。
4. 真实行业验证：与电力客户合作、任务来自真实工作流，且对开源/闭源模型通用，说服力强于玩具 demo。
5. 消融给出可迁移经验：坐标级行为克隆跨界面脆弱（2/5），语义步骤表示有效（5/5），对做演示学习/模仿数据的人有直接指向意义。

---

## 7. 局限与可改进点（个人点评）

- **评测规模与统计效力偏弱**：单软件、单领域、5 任务、3 模型，best-of-3 只报最优、专家人工判定——无法回答稳定性/方差，也不排除过拟合到这几个界面。至少应报多次运行的分布而非仅最优值。
- **"人在回路"成本未量化**：演示要录、规则要写、参数要标、失败要修，本质是把教模型的成本转移给了人；论文没给"一条 SOP 技能平均耗时与 ROI"，而这正是企业最关心的。
- **Reason 质量依赖所调 VLM**，语义步骤粒度与正确性无独立评测；若某步"翻译歪了"，错误会沿技能链传播，文中未见失败模式分析。
- **界面/版本迁移鲁棒性未知**：PVsyst 7.2 内布局变化算被覆盖，换版本、换皮肤、跨软件复用同一 SOP 是否仍 5/5，无证据。
- **只评到"任务完成"**，未核对产出质量（如仿真出来的发电量数字与专家结果是否一致）；逐步校验只比屏幕外观，语义级正确性难以保证。
- 作为 10 页 preprint，缺轨迹规模、Token/时间/成本等工程细节，复现更多靠仓库而非论文。

---

## 8. 对我们的启示 / 可借鉴点

- **流程显式化可能是专业 Agent 的第一性短板**：与其继续押注更强的 grounding/RL，不如先把"怎么做才算对"的专家知识结构化。对走 RL/Grounding 的路线，本文提示：数据层面先做"语义动作单元 + 专家规则"标注再谈训练，效果可能更直接。
- **Reason 消融（2/5 vs 5/5）是重要信号**：纯坐标/GUI 框行为克隆跨界面脆弱。做演示采集或模仿数据管线时，可参考"动作前截图 + UI 解析交互目标 + VLM 补语义描述"流水线，把目标表述成"对象+位置+意图"，可迁移性会显著更好。
- **Execute 的"逐步校验 + 人在回路"适合吸收为 Online 执行范式**：校验提示"容忍无害差异、揪出语义偏差"，配合 continue/retry/stop 三元干预，比让模型一口气跑完长流程稳健得多，且实现成本低。
- **渐进式信息注入**（按步检索而非全量塞上下文）是做长流程 Agent 的实用工程技巧，可直接借鉴。
- 若团队要自建"技能库"，Configure 的"领域规则 vs 任务参数"二分法是一个简洁好用的技能描述 schema 起点。

---

## 9. 延伸阅读

- **OmegaUse**（arXiv:2601.20380）：同团队（百度）通用 GUI Agent，本文是其专业流程能力延伸，先读它理解底座。
- **SOP-Bench**（arXiv:2506.08119）、**Workflow-GYM**（arXiv:2606.11042）：工业/专业工作流基准，实证 LLM Agent 失败模式，本文问题动机来源。
- **OSWorld**（arXiv:2404.07972）：桌面通用 GUI 基准，理解"通用 vs 专业"差距的对照系。
- **UI-TARS-2**（arXiv:2509.02544）、**Agent S2**（arXiv:2504.00906）、**UFO2**（arXiv:2504.14603）、**Mobile-Agent-v3.5**（arXiv:2602.16855）：GUI 模型训练与 Agent 框架代表工作。
- **OmniParser**（arXiv:2408.00203）、**PP-OCRv5**（arXiv:2603.24373）：Observe 模块用的 UI 解析/OCR 底座。
- **Claude Code / Codex computer use** 产品文档：Execute 模块"技能调用 + 控制电脑"思路来源。

---

*解读生成时间：2026-09-10 ｜ 解读人：WorkBuddy（AI）*
