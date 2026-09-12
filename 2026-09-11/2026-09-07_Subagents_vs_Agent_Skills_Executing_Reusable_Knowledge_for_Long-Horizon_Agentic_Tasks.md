# Subagents vs Agent Skills: Executing Reusable Knowledge for Long-Horizon Agentic Tasks

> 一句话 TL;DR：在 SkillsBench 上证明"可复用技能包怎么执行"比它写了什么更重要——把技能做成带显式输入/输出契约的过程性知识并以子代理（独立上下文）执行，显著优于把 SKILL.md 直接灌进主上下文。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Subagents vs Agent Skills: Executing Reusable Knowledge for Long-Horizon Agentic Tasks |
| **作者 / 机构** | Wasu Top Piriyakulkij（第一作者，Cornell University；工作完成于 Microsoft Research Cambridge 实习期间）；Rachel Lawrence、Alicia Curth、Sushrut Karmalkar、Niranjani Prasad（Microsoft Research Cambridge）。机构：Cornell University（美国）、Microsoft Research Cambridge（英国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-07；arXiv preprint（cs.AI），标注 Preprint |
| **arXiv 链接** | https://arxiv.org/abs/2609.09233 |
| **代码仓库** | ❌ 未开源（全文未提及任何代码仓库或项目主页） |
| **数据集地址** | 使用公开基准 **SkillsBench**（原文参考文献 [15]，arXiv:2602.12670，含 87 个长时程任务）；本次 txt 未给出可下载链接，故不臆造 URL。作者**自建**的 procedural + I/O contract 技能包集（覆盖 87 个任务中的 64 个）未公开，需按 A.4 流程自行合成 |
| **类型标签（论文类别）** | `General` `Planning` `Benchmark` |
| **训练方法标签** | —（无训练；评测/工程：执行模式对照实验，全程无权重更新） |
| **关键词** | Agent Skills；Subagent；长时程任务；上下文管理；输入输出契约；SkillsBench |
| **来源渠道** | arxiv-api |
| **PDF 存档** | 2026-09-07_Subagents_vs_Agent_Skills_Executing_Reusable_Knowledge_for_Long-Horizon_Agentic_Tasks.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：通用 LLM Agent 如何用"可复用知识库"完成长时程任务。场景是 tool-calling Agent + 技能包（描述 + SKILL.md + 脚本）。非 GUI。
- **为什么重要**：长时程任务要在越来越长的轨迹上推理，context 塞满 verbose 工具输入输出与中间推理；技能包本意是注入领域知识，主流用法反而加剧上下文过载。
- **现有方法有什么不足**：论文点名 Agent Skills 的执行范式（把 SKILL.md 加载进主上下文、靠 Agent 自己照做）。它指出 agent skill **不像 RL 里的 skill**（后者是被高层控制器调用的 temporally extended policy），只是把"配方"读进 context，执行仍由主策略在同一 context 完成——于是"技能里有关键知识"与"调用技能有用"是两件事。理论依据是 context 长度导致性能下降（Lost-in-the-middle、bandwidth-limited attention）；Claude Code / Codex 的 subagent 则主要用于并行降时延。
- **Research Gap**：把 subagent 重新定位为"可复用知识的执行载体"，并回答"什么样的技能包适合被 subagent 执行"。答案：过程性知识 + 显式输入输出契约（类比 RL options）。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

不发明新模型，而提出**表示 + 执行**的配对主张：技能包写成 $s=((q_{in},h,q_{out}),\,m,\,R)$，执行时不在主上下文加载指令，而是为每个子任务 spawn 一个以该指令为初始条件的新上下文，只把最终输出交回主 Agent。

### 3.2 方法总览（Pipeline）

- **输入/输出**：SkillsBench 任务 + 技能库 → 任务答案或产物文件路径。
- **两套执行机制**：**Agent Skill** 构造 $k=\mathrm{AgentSkill}(s)$，调用无参数、直接返回指令文本 $E(k,\varnothing)=m_k$，过程由 $\pi_{LLM}$ 在主上下文逐步执行；**Subagent** 构造 $k=\mathrm{Subagent}(s)$，调用时**不**暴露指令，执行器实例化独立策略 $\pi^{(k)}(\cdot)\triangleq\pi_{LLM}(m_k\oplus\cdot)$，新上下文 $c^{(k)}_0=x_t$，跑完后**只返回最后一轮响应**。
- **契约设计**：option $\omega=(I_\omega,\pi_\omega,\beta_\omega)$ 对应技能包——$q_{in}\leftrightarrow$ initiation set、$m\leftrightarrow$ option policy、$q_{out}\leftrightarrow$ termination condition。
- **信息流**：主 Agent 只经 $(x_t,r^{(k)}_T)$ 与子代理通信，中间轨迹不可见；收益是 **peak context 下降**，代价是**总 token 上升**。

### 3.3 真正的创新点（去伪存真）

**真·方法创新**
1. **把"技能包接口"提升为可执行性的一阶条件**。此前 skill-package 学习工作只关心"怎么造技能包"并默认 agent-skill 执行；本文首次指出**执行方式与技能包结构必须匹配**，判据是有无 $q_{in}/q_{out}$。
2. **把 subagent 重新定义为"信息封装机制"**：起作用的不是"多开窗口"，而是"任何单个窗口的峰值负担下降"（peak context 而非 total token）。
3. **一套可复用的技能包合成流程**（详见 4.2），把"好技能包"从人工手艺变成可复现 pipeline。

**工程组合**：subagent 机制 harness 已支持，options 是 1999 年的 RL 概念，技能包格式沿用 Anthropic 规范；组件层面无新算法。

**对性能最关键的设计**：**显式输入输出契约**。用原有人工包（无 I/O 契约）时 agent skill 在所有模型上**追平或超过** subagent；换成 procedural + I/O contract 包后趋势**反转**，subagent 胜出且小模型增益最大。同一机制、同一 harness、同一 benchmark，唯一自变量是契约有无。第二个关键设计：**hybrid 执行**（路由节点 inline、叶子 subagent）在四种库结构下一致最优。

**证据不足 / 仅声称有效**
- **正文无任何准确率数值**，只有方向性描述，无法定量核验或估计效应量。
- **合成包 vs 人工包不是受控比较**（作者自认内容不同），无法排除"只是合成技能更好"；缺"内容相同、只切换契约有无"的消融。
- **peak context 证据被作者自己削弱**：弱模型上下降比例很小甚至反转（Ministral-8B 28.1%、Mistral-Large-3.1 32.8%），作者称只有成功率相近的模式间才可比——等于承认该机制在弱模型（恰是增益最大那档）上无法验证。

---

## 4. 具体技术细节

### 4.1 模型结构

- 横向评测 **7 个 executor**：Ministral-8B、Gemma-4-12B-it、Qwen3.5-9B、Mistral-Large-3.1、gpt-5.4-mini、gpt-5.3-codex、Kimi-K2.6。**全是纯文本 LLM，无视觉编码器，不含 MLLM**。
- **全部冻结、零微调**，模型只作为 $\pi_{LLM}$ 被调用；自变量是上下文组织方式与执行机制。开源模型 8B/9B/12B，闭源参数量未公开。
- 承载框架是 **OpenHands**，作者对其做了实质修改（延长 MCP timeout、移除原生 skills、换成极简 system prompt），所有 LLM 调用经自控 proxy 记录。

### 4.2 训练流程（若需要训练）

**不训练。** 无权重更新、无 RL、无 SFT、无模型蒸馏：

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| — | 不训练（training-free 评测研究） | 无 | 无 | 无 | 无 |

论文里的"蒸馏"**不是模型蒸馏**，而是**技能包合成**：先用 OpenHands + GPT-5.3 Codex 在全部 SkillsBench 任务上跑 **3 次独立 run**，收集**成功轨迹**；再用 **Copilot CLI（Claude Opus 5 驱动）**以最小人工介入合成技能包（subagent 环节由 GPT-5.5 驱动）。作者先在任务 edit-pdf 上产出 3–5 个样例并人工校验"描述里确有 expected input/output"，再以此为范例批量生成。最终为 **87 个任务中的 64 个**成功合成，全文结果都在该子集上。

### 4.3 推理流程

- **输入**：任务描述 + 可用技能/工具清单（名字 + 描述）。**输出**：任务答案，或产物文件的**精确路径**（子代理被明确要求：产出文件则给出确切路径，否则把结果文本放进 final message，不得空手结束）。
- **多步推理，且是两层循环**。主 Agent 是标准 tool-calling 循环 $c_{t+1}=c_t\oplus(r_t,k_t,x_t,o_t)$ 直到产出最终答案；每次 subagent 调用会**嵌套**一个完整的多步循环。终止条件是子代理返回受 $q_{out}$ 约束的终值 + 主 Agent 判定任务完成。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| SkillsBench | 非 GUI（通用 Agent / 长时程工具调用） | 长时程 agentic 任务 | 87 个任务；合成包覆盖 **64 个**，结果基于该子集 | 任务描述 + 技能/工具清单 | 答案或产物路径 | Avg task accuracy；另测 peak context、总 token、技能调用数 |
| SkillsBench + distracting tools（自建扰动） | 非 GUI | 同上 + 无关工具描述 | 干扰技能数 **0 → 263** | 同上 + 无关工具描述 | 同上 | Avg task accuracy vs 干扰工具数 |
| 技能库组织（附录 A.1，自建） | 非 GUI 
### 5.2 实验结果分析

**主结果（方向性，正文未给数值）**：Figure 2 覆盖 7 个 executor。用 **SkillsBench 原有人工技能包（无 I/O 契约）**时 agent skill **在所有模型上追平或超过** subagent；改用 **procedural + I/O contract 合成包**后趋势反转，**subagent 全面胜出且小模型增益最大**。

**peak context 与 token 成本（Figure 4）**：subagent 在 peak context 上占优的任务比例随模型能力递增——Ministral-8B 28.1%、Mistral-Large-3.1 32.8%、Gemma-4-12B 45.2%、Qwen3.5-9B 59.4%、gpt-5.4-mini 70.3%、Kimi-K2.6 79.7%、**gpt-5.3-codex 95.3%**；同时总 token 消耗**显著更高**；随干扰技能数从 0 增至 263，subagent 的准确率衰减也明显更平缓（Figure 3/6）。

**技能库组织（附录 A.1，Figure 5，Qwen3.5-9B）**：四种库结构（flat / tree / graph / two-level task tree）× 三种执行模式（fully agent skill / fully subagent / **hybrid**）。结论：层次化组织在已有好契约技能包之上**还有额外增益**，**graph 优于 tree**，**hybrid 在四种库结构下一致最优**。

**Ablation 说明了什么**：本文用"技能包 × 执行模式"的二维对照替代模块消融。核心结论：**契约的有无是决定 subagent 是否有效的开关**（无契约 → agent skill 更好；有契约 → subagent 更好），**hybrid** 是第二个独立增益来源。

---

## 6. 亮点与贡献（Why it matters）

1. **给出可执行判据**：技能包值不值得做成 subagent，看它有没有 $q_{in}/q_{out}$、是不是过程性的。可直接写成技能库的 lint 规则。
2. **把上下文问题从"多长"重述为"峰值多高"**：total token 增加但 peak context 下降仍能提升效果——应降低任一时刻单窗口的负担，而非压缩总长度。
3. **hybrid 是立刻可用的工程结论**：路由/检索节点 inline、叶子能力 subagent，在四种库结构下一致最优。

## 7. 局限与可改进点（个人点评）

- **最大问题是全文没有数字**。所有核心 claim 都只有柱状图与趋势描述；不给 mean±CI、显著性检验与各模型绝对准确率，读者无法判断是 2 个百分点还是 20 个百分点，也无法评估方差。
- **缺少最关键的受控消融**：固定技能**内容**、只切换契约的有无。当前"人工包 vs 合成包"混淆了内容质量与接口设计；补一个"把合成包的 q_in/q_out 删掉、其余不动"的 arm 才能把自变量钉死。
- **"小模型增益最大"与"peak context 机制"的张力未解决**：增益最大的模型恰恰是 peak context 下降最少的。作者的解释（弱模型过早终止）自洽但无独立测量，机制解释目前是叙事。
- **只在单一 harness（OpenHands）上验证**，且作者对其做了实质改动（延长 timeout、移除原生 skills、换 system prompt），这些改动本身可能影响结论；"subagent 更优"是否在 Claude Code / Codex 上成立未知——而这两个恰是 subagent 的真实主场。
- **可复现性受限**：无代码仓库、无合成技能包发布；自建核心资产未公开，第三方无法复现主结果。

## 8. 对我们的启示 / 可借鉴点

- **上下文预算是峰值而非总量**：做记忆压缩 / 检索 / 轨迹摘要时优先削减单窗口峰值负担，而非缩短总长度。
- **知识资产的接口要显式化**：技能、工具、子模块都应写清"何时可调用、需要什么输入、返回什么"，这既是可用性约束也是可检查的契约。
- **"路由 inline + 叶子隔离" + 强约束返回值**：只做分派的节点留在主上下文更省 token，有 I/O 契约的叶子隔离到子上下文；子代理必须"不得空手结束：文件给确切路径，否则给结果文本"。

### 对 GUI Agent 的可借鉴点

1. **把"页面操作原语"做成带 I/O 契约的叶子技能并隔离执行**。如"在列表中定位目标项并点击"、"填写表单某字段并验证"、"滚动直到元素可见"——都是过程性、输入输出明确的能力，天然适合 subagent 化；"当前该走哪个流程"这类路由则留在主上下文 inline。
2. **GUI 长时程任务的上下文膨胀比文本任务更严重**（每步都带截图描述 / accessibility tree / 动作历史）。peak context 视角直接适用：让"某一子目标"的完整探索发生在独立窗口，只回传状态摘要。
3. **子代理的输入必须自带可观测性信息**。GUI 子代理看不到主 Agent 的截图历史，主 Agent 必须显式传给它当前截图（或结构化描述）、目标元素线索与期望输出形式——这就是 $q_{in}$ 在 GUI 的形态。
4. **$q_{out}$ 应包含"状态变更确认"**。GUI 动作成功与否常需回读界面确认，建议把输出契约定为"动作序列 + 执行后状态验证结论 + 新截图引用"，而非单纯"我点了"。
## 9. 延伸阅读

- Anthropic, *Agent Skills*（2025 文档）—— SKILL.md 格式规范来源。
- Li et al., *SkillsBench*, arXiv:2602.12670 —— 全部实验的基准。
- 技能包学习线（本文认为全用 agent-skill 执行）：*EvoSkill*、*CoevoSkills*、*Trace2Skill*、*SkillX*。

---
*解读生成时间：2026-09-11 09:00 ｜ 解读人：WorkBuddy（AI）*
