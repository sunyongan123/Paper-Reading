# 📚 GUI 论文每日速览与精读

> 每日自动检索 **GUI Agent / GUI Grounding / Computer Use** 方向新论文 → 精读解读 → 汇总速览，归档于此。
> 每篇解读均附：**arXiv 链接**、**开源/代码链接**（如有）、**类型标签** 与 **训练方法标签**。

## 📅 历史速览索引

| 日期 | 速览 | 论文数 | 主题 |
|---|---|---|---|
| 2026-09-03 | [📖 当日速览](2026-09-03/) | 5 篇 | 主题一句话：**效率（系统综述）+ 专业流程技能（SOP）+ 蒸馏落地（CA-OPD）+ 人因评测（盲人可访问性）+ 测试时自适应（CoAdapt-GUI）** |
| 2026-09-04 | [📖 当日速览](2026-09-04/) | 2 篇 | 主题一句话：**Web Agent 走出"下一步动作"范式** —— 一篇用判别式世界模型服务测试时规划（Planning/RL），一篇用纯可观测轨迹信号做在线失败风险监控（Online/监控） |
| 2026-09-05 | [📖 当日速览](2026-09-05/) | 3 篇 | 主题一句话：**GUI Agent 的"停手能力" + Agent 方法论双视角** —— 一篇把可靠性从"做得到"延伸到"该不该做"（冲突识别停手）；另两篇来自通用 Agent 域（贝叶斯世界模型愿景、主动服务决策框架），为 GUI Agent 提供跨域思路参照 |
| 2026-09-06 | [📖 当日速览](2026-09-06/) | 3 篇 | 主题一句话：**"RL 信号去噪"专题** —— 三篇全部指向让 LLM/Agent 的强化学习训练更干净、更省、更准：拆穿 GRPO 的"伪优势"、为 GRPO 旧轨迹复用立规矩（Headroom-Drift Replay）、用"必要工具证据路径"奖励根治 Agentic VLM 的乱调用工具（NTEP） |
| 2026-09-07 | [📖 当日速览](2026-09-07/) | 4 篇 | 主题一句话：**"RL 方法论补漏"专题（周一无新公告，二次补漏）**——把 RL 训练目标改成"稀有高分导向"（TailRL）、让 agent 学会"少决策多执行"的动作分块（SPACE）、让图式信用分配跨策略更新累积经验（TIGPO）、以及给自改进 agent 上"确定性护栏"防 LLM 裁判被优化器钻空子（PROCTOR）。前三篇延续 09-06 GRPO 系列，都是 RL 训练侧的"手术刀式"改进；第四篇是评估护栏视角。 |
| 2026-09-08 | [📖 当日速览](2026-09-08/) | 12 篇 | 主题一句话：**"技能沉淀与混合界面"爆发日**——GUI 域迎来混合 GUI+CLI 环境全流程配方（CUA-Universe）、computer-use 在线技能演化（Interaction Traces）、老年用户真实表达评测（ElderBench）与车载 GUI 自动测试（ARIA）；Agent 方法论域 8 篇围绕"轨迹→技能/图→策略协同进化"（CoSkill/Trace2Tower/TROVE）、蒸馏防漂移（RISE/Persistent Teacher）与 RL 训练配方实证（SiLR/Multi-Harness RL）展开，另收一篇 Agentic AI 全景综述（World-Acting Systems）。 |
| 2026-09-09 | [📖 当日速览](2026-09-09/) | 15 篇 | 主题一句话：**"评测自动化"信号明确**——GUI 域 3 篇中 2 篇直击评测可复现/自动构建（APPSim-Bench 把真实 App 搬进可控沙盒、FinCUABuild 让 agent 自主造金融 CUA 评测任务），1 篇用自生成工具动作增强 GUI 执行（DroidTool）；Agent 方法论域双线并行——RL 训练"配方级"实证（SRPO 集合动作 RL、Elastic Horizon 自适应交互预算、Env-Scaffold 环境反馈加料、SVRL RL 自验证）与"经验→可复用资产"的自演化/对齐（SkillAdam、Procedural Graphs、Co-Evolving Harnesses、SkillAlign），另收长轨迹上下文工程（AttnCompress/MEMO）、风险感知评测（AURA-Eval）与 test-time 世界模型适应（WorldAgen）。 |
| 2026-09-10 | [📖 当日速览](2026-09-10/) | 15 篇 | 主题一句话：**GUI 域回到"持续学习 + 知识资产"主线**——SKC 用神经元级梯度手术让 GUI Agent 在应用流上不遗忘地持续进化，Typed Federated Artifacts 探索冻结异构 agent 间的工具路由知识共享，SafeMem 用长期图记忆补上"离开视野的危险物"这一安全盲区，SciFigure2Code 把科研图逆向重建为可编辑代码评测；**Agent 方法论域则集中在"把经验/结构变成可复用资产"与"多智能体规模化"两条线**——技能图自演化（SE-GoS）、递归自改进后训练（NeoHorse-1）、场景化记忆（CreaMem）、轨迹数据库愿景（TrajectoryDB）、长时程世界模型闭环（Hi-FLoop）、多智能体前景状态传播与子队分解（PspMAS/RCSD）、长时程决策再验证（ATR）、长上下文并行阅读（PARSER）与忠实引用（ReCite）。 |

---

## 🏷 标签体系

- **类型标签（论文类别）**：`RL`（强化学习）· `GUI Grounding`（界面定位）· `Online`（在线交互学习）· `Planning`（任务规划）· `Reflection`（反思/自纠）· `Distillation`（蒸馏）· `SFT`（监督微调）· `Benchmark`（数据集/评测）· `Web` · `Mobile` · `Desktop` · `General`（综述/其他）
- **训练方法标签（方法级）**：如 `RL (GRPO)`、`RL (PPO)`、`Online RL`、`SFT`、`Distillation`、`LoRA`；综述/评测/工程类标注"—（综述/评测/工程）"

## ℹ️ 说明

- 数据来源：arXiv API + GitHub [Awesome-GUI-Agents](https://github.com/ZJU-REAL/Awesome-GUI-Agents) 等精选列表
- 解读为 AI 辅助精读（以原文为准，涉及数字请回看 arXiv 原文）
- 由 WorkBuddy 自动化工作流每天 08:30 生成并同步
