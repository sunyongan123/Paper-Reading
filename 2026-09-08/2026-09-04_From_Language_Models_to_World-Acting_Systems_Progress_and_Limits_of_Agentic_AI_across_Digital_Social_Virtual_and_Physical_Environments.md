# From Language Models to World-Acting Systems: Progress and Limits of Agentic AI across Digital, Social, Virtual, and Physical Environments（从语言模型到"作用于世界的系统"：Agentic AI 在数字、社会、虚拟与物理环境中的进展与边界）

> TL;DR 一篇以 2026-08-31 为文献截止线的批判性综述：把智能体视为"模型+harness+环境+委托者"的配置系统，沿"委托权威、时间持续性、环境耦合"三维取证，论证**行动接口的扩张远比"可验证的可靠自主"更有证据支撑**，并提出 justified delegation（有据委托）作为分析基准与研究议程。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | From Language Models to World-Acting Systems: Progress and Limits of Agentic AI across Digital, Social, Virtual, and Physical Environments |
| **arXiv ID / DOI** | arXiv:2609.04894v1 [cs.AI]（无 DOI） |
| **arXiv 链接** | https://arxiv.org/abs/2609.04894 |
| **发表出处（Venue）** | arXiv preprint（2026-09-04 预印本；critical review） |
| **发布时间** | 2026-09-04 |
| **作者** | Linsen Zhu；Mengqing Cai |
| **所属机构** | 待确认（正文首页未列出作者机构信息） |
| **开源情况 / 代码** | ❌ 未在文中声明开源（综述性质，无代码/artifact；文末"Use of generative AI"声明使用 AI 辅助文献发现与写作） |
| **类型标签** | `General` `Web` `Desktop`（跨域批判性综述，重点覆盖 computer-use/GUI 类数字动作证据） |
| **训练方法标签** | —（综述，无训练方法） |
| **关键词** | Agentic AI；计算机使用 Agent；委托式自主；World Model；多智能体；MCP/A2A；人类监督 |
| **来源渠道** | arxiv-api + listing |
| **PDF 存档** | 2026-09-04_From_Language_Models_to_World-Acting_Systems_Progress_and_Limits_of_Agentic_AI_across_Digital_Social_Virtual_and_Physical_Environments.pdf |

---

## 1. 研究背景与要解决的问题

语言模型最初只是文本生成器：输入文本、得到文本。如今一个输出 token 就可能触发数据库查询、编辑代码仓库、浏览网站、调度另一个智能体、询问一个人，甚至移动实验仪器——模型开始"改变外部世界状态"。作者认为核心问题已不再是"模型能否写出合理计划"，而是"配置好的系统能否在被委托的任务上长期执行，同时保住用户意图、机构约束与可追溯的记录"。

真正的概念陷阱在于 "more agentic" 一词混用了至少四种不同含义：更强的基座模型、更丰富的 harness（外围系统）、更长的任务、或更后果严重的环境——它们之间互不蕴含。模型可以推理很强却没有行动权限；弱模型可被授予宽泛凭据；协议能让工具互通却不代表安全。把这些差异统统排进一条"自主性阶梯"，恰好隐藏了决定效用与风险的关键变量。既有综述多按组件或评测方法组织，本文改用三个独立维度（委托权威、时间持续性、环境耦合），并始终把 model/harness/environment/delegator 分开，以暴露架构式或榜单式叙事所掩盖的"保证缺口"。

## 2. 核心方法 / 思路（分析框架与核心论点）

作为批判性综述，本文的"方法"即其分析框架。

**分析单位是配置系统** S=(M,H,E,U)：模型 M 提出行动；harness H 负责上下文构建、记忆、工具 schema、验证器、重试逻辑与凭据管理；环境 E 提供状态、动作与后果；委托者 U 授予目标与合法权限。受影响的第三方既不是 U 也不操作该系统，需独立的通知、申诉与救济渠道。这种拆解解释了为何"模型性能"常是系统属性——SWE-agent 证明换掉接口、权重不变结果也变。

**三个刻画维度**：委托权威（从"只建议"经"审批后行动"到"直接改状态"）；时间持续性（单次回复→多步可恢复任务→跨 episode 义务，且须按对象拆开——会话、情景记忆、技能、环境状态、授权持久化并不等价）；环境耦合（可重置模拟→有界在线服务→难逆转的共享社会/物理环境）。

**证据纪律（表 1）**：按来源给结论定级，同行评审基准与受控物理实验证据最强，preprint、协议规范、厂商 preview 只能支撑较弱主张。三条边界规则：单个成功轨迹只证明可行性而非可靠频率；"开放性"指动作环境而非画面丰富度；能力演示不能外推为安全部署。

在此基础上给出核心论点：**行动接口（action coverage）扩张的证据，比"验证、恢复、授权、治理"可靠化的证据更充分**。MCP/A2A 标准化了能力交换与任务传递，却不建立可信委托；多智能体带来专门化，也引入成本与相关性错误（等预算对比下可能不如单智能体）；持久化模拟与 world model 支撑训练和规划，但它们本身不是 agent；机器人/VLA 与自动实验室只确立"有界可行性"。因此全文提出 **justified delegation**（有据委托）：只应在证据支持溯源、有界授权、故障检测、安全恢复与可校准人类控制之处扩大动作范围，并据此给出七项研究议程。

## 3. 关键实验结果（综述汇总的证据）

综述不生产实验，其分量在精选的代表性对比数字：

- **GUI/桌面基线**：WebArena 原研究最强 GPT-4 agent 仅 **14.41%**（人类 78.24%）；OSWorld 原研究最佳配置 **<12.2%**（人类 72.4%），证明视觉感知、操作知识与跨应用状态追踪不同于流畅的语言生成。后续一手系统大幅跃升：OpenAI CUA 报 OSWorld **38.1%**、WebArena **58.1%**，但其 system card 自认 38.1% 不足以支撑系统自动化、须人类监督，并披露 prompt injection 等风险——本文据此强调一手报告不能替代独立评测。
- **harness 论证**：SWE-agent 在 SWE-bench pass@1 = 12.5%，表明同模型下 agent-computer 接口设计实质影响成绩。
- **可靠性**：τ-bench 的 passk 分析指出，单次成功率 p、错误独立时，连续 k 次成功仅 p^k——"有时能跑通"远不等于"持续可靠"。
- **多智能体**：2026 preprint 显示等推理预算下单智能体在多跳推理上持平或反超多智能体组织；EACL 2026 证据显示多智能体 debate 会随轮次漂移离题。
- **物理/实验域**：AFMBench（2025）发现领域问答能力不能可靠转化为实验胜任力（"sleepwalking"）；Anthropic 的 Model Hardware Standard（8 月 27 日 preview）与 Genie 3、Project Eden 等前沿一手资料只作架构方向引用，单独标注证据级别。

作者明言历史数字非当前榜单，作用是检验概念主张而非排名。

## 4. 亮点与贡献（Why it matters）

1. **把"进步"拆成三分**：策略胜任、动作覆盖、保证（assurance）三个可分离的主张，终结"什么都叫自主"的误导比较。
2. **证据分级可操作**：source-to-claim 绑定规则（表 1）可直接当作阅读 agentic 论文的方法论工具，识别哪些结论来自 peer review、哪些来自系统卡。
3. **justified delegation 取代"最大自主"**：研究议程七项具体可执行，且明确"更高 benchmark 分数不应自动解锁更大动作面"——与主流叙事相反。
4. **时效性好**：以 2026-08-31 为线，同步了 MCP 2026-07-28 修订、A2A 1.0、Agent2Agent、Magentic-UI、RentAHuman 等最新协议与产品动态，且不为厂商背书。

## 5. 局限与可改进点（个人点评）

- **非系统性综述的自限**：作者自承不能支撑 prevalence 推断，选择性与证据不对称（偏英文、软件/Web、机器人实验室），高风险的长期企业运营与"受影响的非用户"证据稀缺；读者易把其中数字误读为现状排名，需回原文核证据等级。
- **前沿证据脆弱**：Genie 3、Project Eden、MHS 等均为一手 preview，无档案论文与独立复现；综述结论随协议修订与发布快速过期，需要持续维护。
- **justified delegation 缺乏操作刻度**：它是规范性启发而非可测度量——"证据充分到何种阈值才可扩权"没有给出可判定的标准。
- **方法面单薄**：视角集中在行动/治理/评测，对"如何训练更强的行动策略"（RL 细节、grounding 方法）着墨少；作为方法论参考，其价值在评估面而非训练面，GUI 上的 RL 进展需另补。

## 6. 对我们的启示 / 对 GUI Agent 的可借鉴点

- **把 GUI Agent 也当配置系统**：截图分辨率、无障碍树、动作空间、重试预算等 harness 变量必须在对比中固定或分层报告，不能把成绩全记到模型头上；发布基准分时应附完整 harness 描述。
- **动作覆盖不要跑在验证前面**：新增 scroll/双击/type 等动作覆盖面时，须同步配 grounding 验证与终点状态断言（截图 diff、AT 树/DOM 谓词），并以"多次重试的一致成功率"（τ-bench passk 思路）取代单次 best-case。
- **混合 API 与纯像素**：能用结构化调用就用结构化调用，computer-use 只兜底，并让同一策略与权限层贯穿两路——否则兜底接口会变成绕过控制的通路。
- **注入攻击在 GUI 上是 action-security**：网页文字/截图文本可劫持点击流，需最小权限、隔离浏览器与独立于模型的安全恢复，而不只是"内容完整性"问题。
- **持久化逐对象报告**：技能库持久 ≠ 凭据持久；长跑 GUI agent 需要权限续期、撤销与审计。MCP 可作工具接入层复用，但"语义成功"须由外部证据验证，host 不能自证。

## 7. 延伸阅读

- **agentic 起点三件套**：ReAct（2023）、Toolformer（2023）、Reflexion（2023）；认知架构视角：Cognitive Architectures for Language Agents（TMLR 2024）。
- **harness 与计算机使用**：SWE-agent、WebArena、OSWorld、τ-bench（均 NeurIPS/ICLR 2024/2025）。
- **评测与安全**：AgentBench、AgentDojo、ToolEmu、Agent Security Bench。
- **协议与环境**：MCP 规范（2026-07-28 修订）、A2A 1.0、Magentic-UI（MSR TR）、Generative Agents、Voyager。

---
*解读生成时间：2026-09-08 ｜ 解读人：WorkBuddy（AI）*
