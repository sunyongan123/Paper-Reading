# AttnCompress: Dynamic Attention-Guided Trajectory Compression for Software Engineering Agents

> 按语义分块、按代理注意力打分、滚动动态维护，免训练压缩智能体长轨迹——省 token、降成本、不伤成功率。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | AttnCompress: Dynamic Attention-Guided Trajectory Compression for Software Engineering Agents |
| **arXiv ID / DOI** | 2609.08318（v1）；DOI: 10.1145/3832149 |
| **arXiv 链接** | https://arxiv.org/abs/2609.08318（点击直达） |
| **发表出处（Venue）** | ISSTA 2026（Proc. ACM Softw. Eng., Vol. 3, No. ISSTA, Article ISSTA058, 23 页） |
| **发布时间** | 2026-09-08（v1） |
| **作者** | Zhengran Zeng、Yixin Li（共同一作）；Rui Xie、Wei Ye、Shikun Zhang（通讯作者） |
| **所属机构** | 北京大学（中国） |
| **开源情况 / 代码** | ✅ 有代码：https://github.com/ZZR0/AttnCompress |
| **类型标签（论文类别）** | `General` |
| **训练方法标签** | —（工程/方法） |
| **关键词** | Software Engineering Agents；Context Compression；Attention；Trajectory Management；Long Context |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_AttnCompress_Dynamic_Attention-Guided_Trajectory_Compression_for_Software_Engineering_Agents.pdf |

---

## 1. 研究背景与要解决的问题

自动软件工程（ASE）智能体以 ReAct 循环（思考→调工具→看反馈）在真实代码仓库里解决 GitHub issue。它很强，但"试错式"工作流会让轨迹很长：一次 `cat` 一个文件、一次 `pytest` 都常吐回成百上千行输出。作者统计发现，轨迹里 **Observation（环境输出）占 token 的 62.6%**（Action 24.8%、Thought 8.6%），上下文膨胀的主犯就是冗长反馈；长上下文带来成本高、延迟大、超窗口上限，还触发 "Lost-in-the-Middle" 使模型在噪声中变笨。

现有压缩分三类，各有硬伤：①启发式静态剪枝（ObsMask 直接扔旧回合输出），假设"旧内容不重要"，但调试中智能体常"焦点漂移"——前 20 轮查 A 文件，第 21 轮反馈却把根因指向第 1 轮看过的 B 文件，而 B 已被永久删除（审计 100 条轨迹，40 条存在该问题）；②LLM 摘要式（AgentDiet 每步叫大模型重写上一步），丢精确行号/变量名、易幻觉，且每步都贵；③token 级选择（LLMLingua-2），删括号缩进会破坏代码 AST/JSON。加上"粒度失配"：一个工具输出常只有十几行有用（django-11133 例：800 行文件真正要看的 `make_bytes` 约 15 行）。

本文补的缺口是：一个**介于消息与 token 之间、按语义块取舍、且能动态召回**的免训练压缩机制。

## 2. 核心方法 / 思路

AttnCompress 是**免训练、即插即用**的中间件，插在宿主智能体（Trae-Agent）的轨迹管理器与后端 LLM 之间；"管家"是 4B 的 Qwen3-4B-Instruct，只做前向推理、不参与决策。分三个阶段：

**Phase 1：PPL 尖峰驱动的结构感知分段。** 用代理小模型逐行算困惑度 PPL（可理解为对每行的"惊讶度"）；语义切换处——函数结束到新函数、日志到报错堆栈——惊讶度突增成尖峰。对相邻行 PPL 差做自适应阈值（均值 μ+h·σ）切分，输出切成若干"语义块"（单行碎块并入前块）。压缩单位是一段完整函数、一整段日志，而非拦腰截断的 token。

**Phase 2：代理注意力估算重要性。** 把"已压缩历史 + 新 observation + 一个特殊 query token"拼好喂给代理模型；该 token 取聊天模板中"即将开始生成回复"的起始符（`<|im_start|>`），因果模型生成下一响应前必须 attend 前面全部内容，其注意力分布即表达"当前推理最依赖哪些历史"。块得分 = 该 token 对块内 token 的注意力均值，按得分贪心填满预算 ρ·总长（默认 ρ=0.2，留 20%）。类比：学生答题前扫一眼笔记，目光停留处即关键。Phase 1 的 PPL logits 与 Phase 2 的注意力图在**同一次前向**取到，额外开销极小。

**Phase 3：三层滚动窗口动态维护。** 轨迹分三区：**Immediate**（最近 k=2 轮）原样保留保感知连贯；**Short-term**（中间区）每轮用最新 query 重新打分重压缩——被压掉的内容重新相关就能"复活"；**Long-term**（远端）平时冻结以命中主 LLM 的 prefix caching，仅当 Short-term 攒满 M=10 轮触发一次 **Global Refresh**，把归档与短期合并全局重选、捞回深历史中"过时却关键"的块。

对比旧法一句话：别人"看一眼当下就永久决定删什么"，AttnCompress 是"近端每轮复查、远端低频大整理"，让压缩可逆、可召回。

## 3. 关键实验结果

主基准 SWE-Bench-Verified 200 例测试集（沿 AgentDiet 协议），配 Gemini-3-Flash、Qwen3-235B、Qwen3-Coder-30B 三个后端 LLM，对照 7 个基线（统一 tail=2）。

- **RQ1 性价比**：平均 Pass **53.17%**（Gemini 71.50 / Qwen3-235B 43.50 / Coder-30B 44.50），三个后端均单独超全部基线；高于 SOTA AgentDiet 51.17%（相对 +3.9%）及 ObsMask 47.17%、Lingua 48.67%、SlidingWindow 49.83%、LLMSummary 50.83%（完整上下文 Original 55.17%）。输入 token 均值 644.66k，较 AgentDiet 减 **21.6%**、较 Original 减 42.3%；单实例总成本 **$0.0949，较 AgentDiet（$0.1429）降 33.6%**、较 Original（$0.1189）降 20.2%；端到端 366.7s（压缩仅 66.6s），AgentDiet 498.5s（−26.4%）。Coder-30B 重复 5 次：43.70% vs AgentDiet 41.80%，非单次运气。
- **RQ2 消融**（验证集，Coder-30B；完整 42.0%）：注意力换随机选 →35.0%（**−7.0**，贡献最大）；去滚动窗口做固定压缩 →37.0%（**−5.0**）；PPL 换 token 级剪枝 →39.0%（**−3.0**）。tail=2、ρ=0.2、M=10 为默认；参数主要调"成本-性能"天平而非脆弱开关（tail=10 可到 44.0% 但成本近翻倍）。
- **RQ3 泛化**：换 5 只代理（Qwen3 1.7B/4B/8B、Gemma3-4B、Llama3.2-3B），Pass 在 43.5%~46.5%，与 30B 主模型的块排序 Spearman ρ=0.56~0.64、top-10% 重叠 0.55~0.64 → 信号跨模型家族通用。Multi-SWE-Bench-Flash（300 例、7 语言）Pass **19.67%** vs Original 20.33%（约 96.7% 相对性能），压缩法最高（AgentDiet 18.33%、ObsMask 15.72%）；Java 17=17 打平、C 5>4 反超。

## 4. 亮点与贡献（Why it matters）

- **免训练即插即用**：无需微调或蒸馏，任何 ReAct agent 可挂载，把上下文压缩降级为一次 4B 前向的廉价服务。
- **"语义块"是恰到好处的粒度**：PPL 尖峰自动切块，化解消息级太粗、token 级破坏语法的两难。
- **注意力是免费信号**：与分段共用一次前向，近乎零成本量化"历史对当前推理的相关性"。
- **让压缩可逆**：三层滚动 + 低频全局刷新，正面解决 40% 轨迹都存在的焦点漂移。
- **评测口径扎实**：pass/token/成本/延迟/步数全报，做 5 次重复统计并验证跨模型、跨语言。

## 5. 局限与可改进点（个人点评）

- 只在 Trae-Agent 单宿主上验证（作者自认）；Action/Thought 合计 33% 的压缩空间基本未动。
- "相关性"只是近似：小/大模型注意力一致性仅 Spearman≈0.63，仍会误删"后来才重要"的证据——失败主因恰是早期文件路径、报错被丢后无法恢复，说明注意力信号不能保证关键证据零丢失。
- 与无损仍有差距：Pass 53.17% 低于 Original 55.17% 约 2 个百分点；所谓安全压缩只是概率意义上的。
- Multi-SWE-Bench-Flash 绝对分仅 19.67%，跨语言统计力偏弱；100 例调参 + 100 条人工抽查样本小，ρ、h 等对陌生领域缺自动适配策略。

## 6. 对我们的启示 / 可借鉴点

最值得吸收的一句话方法论：**用一只小模型当大模型的"上下文管家"**——不训练、不重写，只分段加打分，靠注意力这种免费信号做在线记忆管理，并以"近端高频复查、远端低频大整理"对抗意图漂移。这比"静态删历史"和"每步调大模型重写"都便宜，工程上几乎白拿。

**对 GUI Agent 的可借鉴点：**

GUI 长轨迹与 SE 同构甚至更重：每步操作常带回整屏截图/DOM/a11y 树（对应文中占 62.6% 的 Observation），视觉 token 更贵，百步级网页/桌面任务极易超窗。其方案可整套平移，只需替换"信号源"：

- **三层滚动压缩直接落地**：最近 2~3 步截图/操作原样保留保 grounding；中段按"当前意图"重新评分裁剪；远段冻结吃 prefix caching；在用户改需求、中途换目标等长任务转折处做全局刷新——GUI 同样有"20 步前看过的页面又要回去改"的焦点漂移。
- **"语义块"换成视觉/结构边界**：以窗口/弹窗/表单区块/DOM 子树/a11y 树片段为块单位整块取舍（如无关横幅、已滚过的长列表），而非按截图网格或像素硬切，避免拦腰截断正在交互的控件。
- **代理注意力换轻量信号**：GUI 无文本 query token，可改用 4B 级 VLM 对"下一步操作区域"与历史元素/截图的注意力打分；本文证明 4B 代理与 30B+ 主模型判断相关性达 0.6+ 就足以驱动稳定表现，不必每步请昂贵大模型重写视觉历史。
- **成本账本可作标杆**：单实例压缩开销仅约 66s、总 token 省 21%~42%、总成本降 33.6%——证实"原始保近端 + 结构化裁剪压远端"的分级记忆是 GUI 长截图任务控制 token 的现实路线。

## 7. 延伸阅读

- **AgentDiet**（"轨迹缩减"SOTA 基线）— arXiv:2509.23586
- **Trae Agent**（本文宿主智能体）— arXiv:2507.23370
- **ObsMask**（观察掩蔽启发式）— NeurIPS 2025 DL4C Workshop
- **LLMLingua-2**（token 级压缩代表）— arXiv:2403.12968
- **LongCodeZip**（代码长上下文 PPL 分段，Phase 1 依据）— arXiv:2510.00446
- **AttnComp**（RAG 注意力压缩，Phase 2 思想来源）— EMNLP 2025 Findings
- **Multi-SWE-bench**（多语言 issue 基准）— arXiv:2504.02605

---
*解读生成时间：2026-09-09 ｜ 解读人：WorkBuddy（AI）*
