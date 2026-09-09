# FinCUABuild: Can Agents Build Reliable Benchmarks for Dynamic Financial Computer Use?

> 一句话说清这篇论文做了什么、解决了什么问题：为"AI 能否自动构造动态金融 CUA 评测任务"建基准（FinCUABuildBench）并给出多智能体构造方案（FinCUABuildAgent）。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | FinCUABuild: Can Agents Build Reliable Benchmarks for Dynamic Financial Computer Use? |
| **arXiv ID / DOI** | 2609.07603 (v1) |
| **arXiv 链接** | https://arxiv.org/abs/2609.07603（点击直达） |
| **发表出处（Venue）** | arXiv preprint |
| **发布时间** | 2026-09-07 |
| **作者** | Jingpu Yang、Fengxian Ji（共同一作）、Jinri Guo、Tianhao Li、Qian Jiang、Fan Zhang、Min Peng、Qianqian Xie（通讯作者）、Preslav Nakov、Zhuohan Xie（通讯作者）（共 10 人） |
| **所属机构** | 武汉大学（中国）；MBZUAI（阿联酋）；Northeastern University（美国）；Zhongguancun Academy（中国） |
| **开源情况 / 代码** | ✅ 有代码：https://github.com/FengxianJi/FinCUABuild |
| **类型标签（论文类别）** | `Benchmark` `Web` `Desktop` |
| **训练方法标签** | —（评测/Benchmark + 基于现有 LLM 的多智能体系统，未训练模型） |
| **关键词** | Computer-Use Agent (CUA)；金融评测任务自动构造；动态事件；基准生成；多智能体；验证门限 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-07_FinCUABuild_Can_Agents_Build_Reliable_Benchmarks_for_Dynamic_Financial_Computer_Use.pdf |

---

## 1. 研究背景与要解决的问题

Computer-Use Agent（CUA）泛指能操作浏览器、电子表格与桌面办公软件完成跨工具任务的智能体，金融是典型重场景：常要跨文件取数、做表格运算、按流程出报告，并覆盖不同的数据条件、工具组合与合规策略。现有评测（SpreadsheetBench、Finch 等）几乎全靠人工构造任务，代价极高——论文点名 Finch 即便用 LLM 辅助任务挖掘，**仍花 700+ 小时专家标注**，堵死了评测覆盖的规模化。于是作者反问：**能不能让 Agent 自己造出多样、动态的金融评测任务？** 这面临三难：(1) 构造请求须落在真实多样的工作流与运行时变化上，而非改写措辞堆"伪多样性"；(2) 方法间输入、工具、预算不一致会令比较失真，须标准化；(3) 任务说明"合理"不等于环境能跑、事件合法、判题正确，须对整包质检。文献既无统一基准量化"构造能力"，也无专用构造 Agent——这正是本文要补的两个洞。

## 2. 核心方法 / 思路

主线两条：怎么评"构造能力"，以及怎么把任务造好。

**FinCUABuildBench（评构造的基准）**：任务空间 = 24 个金融工作流类别（财报建模、对账、FP&A 差异调查、AP 例外处理、KYC/AML、监管报告等）× 每类 8 个语义互异的 WorkflowCore × 3 类运行时动态 profile：P1 状态更新（财报修正、市值刷新）、P2 工具故障（超时、会话过期）、P3 策略变更（审批拒绝、权限收紧），共 **576 个预注册槽位**。每个槽位要求产出完整 episode 包：指令、版本化证据、可重置初态、目标谓词、工具图、可回放的动态事件、隐藏 verifier、重置清单与权限。合格 = 一组硬门限的合取：人审接受、证据可溯源、可执行可回放、verifier 通过、动态事件有效、能力必需、去重。评测执行用 **automation-point DAG** 语义打分：把任务拆成语义检查点（验证财务事实、更新工件、跨工具传数、响应事件），静态/动态两类点集互补，查语义产物而非界面动作、允许多路径，并罚越权副作用。

**FinCUABuildAgent（能构造的系统）**：Factory Controller 统筹 11 个专职 Agent，走 **generate–audit–repair** 循环，以类型化编排图 + 每阶段工具白名单约束流转，失败由 Controller 定位到责任 Agent 定向修复。三段：① Discovery & Grounding——挖工作流候选、核对主体/报告期/币种与财报来源、分类去重；② 目标向后编译 + 环境构建——把"目标能力"**向后编译**成证据依赖与语义操作（Dependency Compiler）→ Episode Designer 出任务单 → Tool Architect 生成带 fallback 的工具图 → Environment Composer 物化工作区与事件；③ Execution & Assurance——**Independent Oracle** 在任何候选解之前先从证据推导隐藏目标与验证条件，**Reference Solver** 用与被测 agent 相同界面走通任务证明可解，**Red-Team** 注入错误数值、过期证据、越权副作用与捷径测 verifier。要点：工具图规格与环境物化分离、oracle 先行提交，从机制上防构造过程与判题器串谋。

## 3. 关键实验结果

- **构造质量（同骨干 MiniMax-M3）**：FinCUABuildAgent 严格合格率 Qual **31.3%**，远超工具增强提示（10.8–17.5%）与通用构造基线（AutoBencher 8.3%、Benchmark Agent 4.5%、BENCHAGENTS 1.3%）。换 Qwen3.5-397B-A17B 骨干达最高 **41.9%**（Kimi-K3 40.3%）。相对工具增强提示，Dyn 提升 29.0–48.2 个百分点、Dup 降 24.9–34.6、Qual 提升 13.8–29.4。
- **下游诊断（144 个合格任务 × 3 次运行）**：参考求解器全过、任务可行；通用 agent 框架 Static Completion 0.622–0.660、Financial Grounding 近 1.0，但 **Dynamic Completion 仅 0.004–0.043**（会查证不会应变）；外部结构化配置最高 0.676；17 个配置中 16 个 dynamic < static。
- **留出泛化（90 请求，3 个未见场景）**：严格 Qual 仅 **13.3%**，V-Acc 每格仍 ≥86.3%；P3 最难（Exec 46.7% 但仅 6.7% 合格）。
- **消融**：stage 化协调把 Exec 从单控 18.8% 拉到 73.8%；去掉 Grounding & Provenance 后 V-Acc 88.5% 但 Exec 崩至 10.4%；去掉 Independent Oracle 使 V-Acc 72.3→52.0、L-Acc 55.7→8.9、Dup 升至 43.2%；去掉 Red-Team 使 L-Acc 跌至 0.5%。

## 4. 亮点与贡献（Why it matters）

- **"Benchmark 的 Benchmark"**：首个量化"自动构造金融 CUA 评测任务"这一元能力的基准，质检口径含 groundedness、可回放、verifier 可靠性、动态性、能力必需、安全与去重。
- 场景结构化：576 槽 = 24 类工作流 × 8 WorkflowCore × 3 类动态，事件确定性可回放，同源槽位同 split 防泄漏。
- 多智能体构造范式：能力→依赖向后编译 + 独立 oracle 先行 + 红队/参考求解双审计，系统压制"任务与判题器共谋"和捷径污染。
- 证明自动构造有评测价值：合格任务能把"金融 grounding 强、动态适应弱"的现有 CUA 显著拉开（0.004 vs 0.676）。
- 现实意义：对照 Finch 的 700+ 人工小时，给出规模化扩展金融评测覆盖的路径，把"评测成本"变成可优化、可评估的对象。

## 5. 局限与可改进点（个人点评）

- **合格率仍是瓶颈**：MiniMax-M3 下 Qual 仅 31.3%、留出 13.3%，多数槽位造不出来——自动化只是降本而非免人工，仍需人审/LLM 审计兜底。
- **能力与骨干强耦合**：Qual 随骨干 31.3%→41.9% 漂移，开源/小模型能否支撑未验证，指标会随模型换代水涨船高。
- 企业级场景用合成 fixture + 一致性检查代替真实数据，可能与真实分布有偏差；公开源还牵涉版本化、许可与 verifier 维护成本。
- 评测口径复杂：Dynamic/Static 点集刻意不嵌套，导致"外部配置 dynamic 0.676 > static 0.524"这类反直觉读数；严格门限可能误杀"有用但非完美"任务，低估方法价值。
- 泛化声明有限：留出仅覆盖 taxonomy 内未见 WorkflowCore（13.3%），不构成对新类别的外推；144 任务为一次性固定抽样。

## 6. 对我们的启示 / 可借鉴点

- **"防串谋"工程处方可移植**：Independent Oracle 先于答案提交、工具图与环境物化分离、红队主动测捷径——这套 generate–audit–repair 质检流水线，正是我们在评测/数据合成或 RL 训练中防 reward hacking、防泄漏所需的机制样板。
- **动态评测三类模板**：P1 状态更新/P2 工具故障/P3 策略变更均"确定性可回放"，可作 Online/自适应 agent 评测与训练环境的现成扰动注入模板；"动态完成度≈0"的发现说明该方向研究缺口极大。
- **细粒度优于二元**：automation-point DAG 的语义验证 + 越权副作用惩罚，比"最终成败"给出更多可归因信号，可借鉴到 GUI/网页 agent 评分设计。
- **消融的工程规律**：强验证器（V-Acc 88.5%）救不了不可执行包（Exec 10.4%）——做 agent 系统要保证"证据—状态—工具—验证"端到端跑通；typed handoff 与逐阶段校验值得落地。
- **方法论互补**：本文用 LLM 自动构造任务包的思路，与 AppSim-Bench 式"编码 Agent 造模拟环境"互为表里（重判题可靠 vs 重界面真实），可组合出自动造高质量移动/桌面评测的完整流水线。

## 7. 延伸阅读

- 金融工作流评测：SpreadsheetBench、Finch、FinQA/TAT-QA、FinBen、FinDABench、SheetAgent、FinMCP-Bench、FinVault。
- 自动基准构造：AutoBencher、BENCHAGENTS、Benchmark Agent（Benchmark Everything Everywhere）、AgentSynth、EnvScaler、APIGen-MT、Dynabench。
- CUA 与可控环境：WebArena、OSWorld、AppWorld、AndroidWorld、τ-bench、ToolSandbox。

---
*解读生成时间：2026-09-09 ｜ 解读人：WorkBuddy（AI）*
