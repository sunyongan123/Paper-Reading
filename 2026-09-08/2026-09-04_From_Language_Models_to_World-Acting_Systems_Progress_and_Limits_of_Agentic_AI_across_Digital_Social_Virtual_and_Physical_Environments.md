# From Language Models to World-Acting Systems: Progress and Limits of Agentic AI across Digital, Social, Virtual, and Physical Environments

> TL;DR：批判性综述，以 2026-08-31 为截止线，把智能体重定义为"模型+harness+环境+委托者"的配置系统，论证行动接口扩张的证据强于"可验证可靠自主"。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | From Language Models to World-Acting Systems: Progress and Limits of Agentic AI across Digital, Social, Virtual, and Physical Environments |
| **作者 / 机构** | Linsen Zhu、Mengqing Cai（正文未标注机构） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-04；arXiv preprint（critical review，[cs.AI]），文献截止 2026-08-31 |
| **arXiv 链接** | https://arxiv.org/abs/2609.04894 |
| **代码仓库** | ❌ 未开源（综述性质，无代码/artifact） |
| **数据集地址** | 未公开（汇总既有公开基准，无自产数据） |
| **类型标签（论文类别）** | `General` `Web` `Desktop` |
| **训练方法标签** | `—（综述）` |
| **关键词** | Agentic AI；computer use；委托式自主；world model；多智能体；人类监督；AI 安全 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | 2026-09-04_From_Language_Models_to_World-Acting_Systems_Progress_and_Limits_of_Agentic_AI_across_Digital_Social_Virtual_and_Physical_Environments.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：跨数字、社会、虚拟、物理四类环境，系统性梳理"语言模型→作用于世界的系统"这一转变中，哪些能力进步有证据支撑、哪些只是叙事。
- **为什么重要**：一个输出 token 现在可能触发数据库写、改代码仓库、调度子智能体、移动实验仪器。若把"更强模型/更丰富 harness/更长任务/更高后果环境"混为一谈，会系统性高估真实自主度，导致错误授权。
- **现有方法有什么不足**：作者点名两类不足——(1) 既有综述按"组件/应用"或"评测方法"组织，掩盖了 assurance 缺口；(2) 主流"自主性阶梯"叙事把四种互不蕴含的变化排成单一路线。SWE-agent 证明：换掉接口、权重不变结果也变，说明"模型性能"常是系统属性。
- **Research Gap**：作者认为缺的是一套"把 model/harness/environment/delegator 分开、沿委托权威×时间持续性×环境耦合三维取证、并按来源给结论定级"的证据纪律，据此提出 justified delegation（有据委托）作为分析基准与研究议程。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

以配置系统 S=(M,H,E,U) 为分析单位，沿三个独立维度给跨域证据定级，暴露"行动接口扩张"与"可靠自主验证"之间的不对称。

### 3.2 方法总览（分类学即方法）

- **分析单位**：S=(M,H,E,U)。模型 M 提行动；harness H 管上下文、记忆、工具 schema、验证器、重试、凭据；环境 E 供状态/动作/后果；委托者 U 授目标与合法权限。受影响的第三方在系统边界外，需独立的通知/申诉/救济渠道。
- **三个刻画维度**：委托权威（只建议→审批后行动→直接改状态）；时间持续性（单次回复→多步可恢复→跨 episode 义务，且按会话/情景记忆/技能/环境状态/授权持久化分对象拆开）；环境耦合（可重置模拟→有界在线服务→难逆转的共享社会/物理环境）。
- **证据分级（表 1）**：同行评审基准/受控物理实验最强；preprint、协议规范、厂商 preview 只能支撑较弱主张。三条边界规则：单次成功轨迹只证可行性不证频率；"开放性"指动作环境而非画面丰富度；能力演示不可外推为安全部署。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：把"进步"拆成策略胜任/动作覆盖/保证（assurance）三个可分离主张，并给出 source-to-claim 定级规则，是可复用的阅读方法论。
- **工程组合**：配置系统 S=(M,H,E,U)、三维度、证据分级均非全新概念，是对既有认知架构文献与评测文献的整合。
- **对论证最关键的设计**：S 拆解 + 证据分级，二者共同使"保证缺口"显性化。
- **证据不足 / 仅声称有效**：justified delegation 是规范性启发而非可测度量，未给出"证据充分到何阈值才可扩权"的可判定标准。

---

## 4. 具体技术细节

### 4.1 模型结构

不涉及。本文是批判性综述，无自有 base model、无视觉编码器、无微调。

### 4.2 训练流程

**综述不训练**。无任何训练阶段、数据、loss。对涉及训练的证据（RT-2、OpenVLA、V-JEPA 2 等）只作结论性引用，不展开训练细节。

### 4.3 推理流程

不适用。本文"推理"即其分析推理：沿三维度×证据分级，对 WebArena/OSWorld/SWE-agent/τ-bench/MCP/A2A/Genie 3/Project Eden/MHS/Coscientist/AFMBench 等逐一取证，最终收敛到七项研究议程。

---

## 5. Benchmark 与实验设置

### 5.1 覆盖环境与 Benchmark 一览

| 环境域 | 代表性 Benchmark/系统 | 任务类型 | 关键数字（核实） |
|---|---|---|---|
| 数字·工具/API | Toolformer、AgentBench、τ-bench | 学用 API、服务任务 | τ-bench passk：连续 k 次成功≈p^k |
| 数字·Web/桌面 | WebArena、OSWorld、SWE-agent、CUA | 端到端网页/操作系统操作 | WebArena GPT-4 **14.41%** vs 人类 78.24%；OSWorld 369 任务 **<12.2%** vs 72.4% |
| 数字·编码 | SWE-bench | 修真实 GitHub issue | SWE-agent **12.5%** pass@1 |
| 协议/多智能体 | MCP（2026-07-28 修订）、A2A 1.0、AutoGen、Magentic-One | 工具互通、任务传递 | 协议只标准交换、不立可信委托 |
| 虚拟/世界模型 | Generative Agents（25 智能体）、Voyager、Genie 3、Project Eden、V-JEPA 2 | 持久模拟、世界预测 | Genie 3 24fps/720p、视觉记忆约 1 分钟 |
| 物理/机器人 | RT-2、OpenVLA、Gemini Robotics、MHS | VLA、设备驱动 | RT-2 6000 次评测；OpenVLA 7B/97 万 episode |
| 物理/实验 | Coscientist、ChemCrow、AFMBench、A-Lab | 闭环实验 | Coscientist 6 任务；ChemCrow 18 工具；AFMBench"sleepwalking" |

### 5.2 实验结果分析

综述不生产实验，分量在精选对比数字：OpenAI CUA 报 OSWorld **38.1%**、WebArena **58.1%**，但其 system card 自认 38.1% 不足以支撑系统自动化、须人类监督，并披露 prompt injection 风险——印证"一手报告不能替代独立评测"。多智能体端：2026 preprint 显示等推理预算下单智能体持平或反超；EACL 2026 显示 debate 随轮次离题。物理端：AFMBench 证明领域问答能力不能可靠转化为实验胜任力。作者明言这些是历史数字，用于检验概念主张而非排名。

---

## 6. 亮点与贡献（Why it matters）

1. **终结"什么都叫自主"**：把进步拆成策略胜任/动作覆盖/保证三项，纠正误导性比较。
2. **证据分级可操作**：source-to-claim 绑定规则可直接当阅读 agentic 论文的方法论工具。
3. **justified delegation 反主流**：明确"更高 benchmark 分数不应自动解锁更大动作面"，七项议程可执行。
4. **时效性好**：同步 MCP 2026-07-28 修订、A2A 1.0、Magentic-UI、RentAHuman、MHS（8 月 27 日）等最新动态，且不为厂商背书。

---

## 7. 局限与可改进点（个人点评）

- **非系统综述的自限**：选择性取样，偏英文/软件/Web/机器人实验室，不能支撑 prevalence 推断；读者易把历史数字误读为现状排名。
- **前沿证据脆弱**：Genie 3、Project Eden、MHS 均为一手 preview，无档案论文与独立复现，结论随协议修订快速过期。
- **缺操作刻度**：justified delegation 没有可判定阈值，"证据多强才可扩权"悬而未决。
- **训练面单薄**：聚焦行动/治理/评测，对 RL 细节、grounding 方法着墨极少，GUI 上的 RL 进展需另补。

---

## 8. 对我们的启示 / 可借鉴点

本篇属【Agent 方法论域】（非 GUI 落地），单独列对 GUI Agent 的可借鉴点：

- **把 GUI Agent 也当配置系统**：截图分辨率、无障碍树、动作空间、重试预算等 harness 变量必须在对比中固定或分层报告，成绩不能全记到模型头上；发 benchmark 分须附完整 harness 描述。
- **动作覆盖别跑在验证前面**：新增 scroll/双击/type 等覆盖面时，须同步配 grounding 验证与终点状态断言（截图 diff、AT 树/DOM 谓词），并用"多次重试一致成功率"（τ-bench passk 思路）取代单次 best-case。
- **混合 API 与纯像素**：能用结构化调用就用结构化调用，computer-use 只兜底，且同一策略与权限层贯穿两路，否则兜底接口会变成绕过控制的通路。
- **注入攻击是 action-security**：网页文字/截图文本可劫持点击流，需最小权限、隔离浏览器与独立于模型的安全恢复。
- **持久化逐对象报告**：技能库持久 ≠ 凭据持久；长跑 GUI agent 需要权限续期、撤销与审计；MCP 可作工具接入层复用，但"语义成功"须由外部证据验证，host 不能自证。

---

## 9. 延伸阅读

- **Agentic 起点三件套**：ReAct（ICLR 2023）、Toolformer（NeurIPS 2023）、Reflexion（NeurIPS 2023）；认知架构：Cognitive Architectures for Language Agents（TMLR 2024）。
- **harness 与计算机使用**：SWE-agent、WebArena、OSWorld、τ-bench（NeurIPS/ICLR 2024/2025）。
- **评测与安全**：AgentBench、AgentDojo、ToolEmu、Agent Security Bench。
- **协议与环境**：MCP 规范（2026-07-28 修订）、A2A 1.0、Magentic-UI（MSR TR）、Generative Agents、Voyager。

---
*解读生成时间：2026-09-08 ｜ 解读人：WorkBuddy（AI）*
