# 📖 当日论文速览 · 2026-09-11

> 处理日：2026-09-11（周五）｜ 目标公告组：**Thu, 10 Sep 2026**（6 分类 cs.HC/cs.AI/cs.LG/cs.CL/cs.SE/cs.CR listing 去重 382 条）。
> 双域筛选后相关候选 **58 篇（GUI 15 + Agent 方法论 43）**，与已有解读按标题/arXiv ID 去重后 **0 冲突**（本组为新公告组）；按「GUI 域优先 → 日期新 → 主题贴合」取 **15 篇**（GUI 8 + Agent 方法论 7），余 **45 篇**记入 `logs/2026-09-11_backlog.json`。
> 主题一句话：**GUI 域本日的主线是"跨设备与鲁棒性"**——JarvisGUI 把 Android/Windows/Ubuntu 串成可程序化判分的跨设备评测，TRACE 把视觉 token 剪枝重述为"写入时不可逆准入"以在缓存复用下保住 grounding 精度，GraphDroid 用"意图生成与探索解耦"把移动端 GUI 测试成本压到最强纯 LLM 基线的 1/8；同时安全侧一次出现两篇（AgentHijack 的视觉 patch 攻击、多模态注入的实验性评测），说明 **GUI/Computer-Use Agent 的攻击面正在被系统化测绘**。**Agent 方法论域则集中在"记忆与知识的存用分离"与"评测可信度"**——RD-Forget 拆开"存什么/用什么"、Subagents vs Skills 证明"技能怎么执行比写了什么更重要"、Double Measurement Confound 指出 benchmark 分数被"脚手架代劳 + 评分器只评形状"双重污染。
> 本日入选 **15 篇**并完成博士组会级精读解读；GitHub awesome 两列表近 7 天无新增（ZJU 停 07-28、OSU 停 08-12）。
> 标签图例：`类型标签`（论文类别）｜ `训练方法标签`（方法级）｜ 域标注：GUI / Agent 方法论·跨域参考

---

## GUI 域（本领域，8 篇）

### JarvisGUI：用 slot 类型系统把三端串成可自动生成的跨设备 GUI 评测

- 标签：`Benchmark` `Mobile` `Desktop` `General` ｜ 训练方法：`—（综述/评测/工程）`
- 域标注：**GUI**
- 一句话结论：用 slot 类型系统 + 采样式任务图组合，把 Android/Windows/Ubuntu 三端串成可自动生成、可程序化判分的跨设备 GUI benchmark，实测 SOTA 开源 Agent 在跨设备依赖任务上几乎全军覆没。
- [arXiv](https://arxiv.org/abs/2609.10451) ｜ 代码：❌ 未开源（脚注仅写 "will be publicly available at JarvisGUI"，属占位声明）
- **[阅读完整解读](./2026-09-09_JarvisGUI_Towards_Cross-Device_GUI_Agents_with_Dynamic_Task_Composition.md)**

### TRACE（GUI）：把缓存复用的视觉 token 剪枝重述为"写入时不可逆准入"

- 标签：`GUI Grounding` `Mobile` `Desktop` `Web` ｜ 训练方法：`—（综述/评测/工程；training-free，全程不训练）`
- 域标注：**GUI**
- 一句话结论：把 cache 复用下的视觉 token 剪枝重述为「写入时不可逆准入」，用布局先验 + 嵌套排序 + 原生 token 补覆盖 + 单调 KV 收缩，在 5% 预算下把 GUI grounding 精度保住 55.19%（对比基线 34.20%）。
- [arXiv](https://arxiv.org/abs/2609.10297) ｜ 代码：❌ 未开源（仅声明 "The source code will be released"，未提供 URL）
- **[阅读完整解读](./2026-09-09_TRACE_Trajectory-robust_Admission_with_Evidence_Ordering_for_Efficient_GUI_Agents.md)**

### GraphDroid：把"意图生成"与"探索"解耦，移动端 GUI 测试成本降到 1/8

- 标签：`Mobile` `Planning` `Benchmark` ｜ 训练方法：`—（综述/评测/工程；无训练，直接调用 gpt-4.1 与 UI-TARS-1.5）`
- 域标注：**GUI**
- 一句话结论：用 UTG 聚类记忆做异步 intent 合成、简单 intent 交给 EVS 引导的启发式遍历、只有复杂 intent 才调 GUI Agent，41 个 Android App 上代码覆盖率 20.83%、成本不到最强纯 LLM 基线的 1/8。
- [arXiv](https://arxiv.org/abs/2609.10031) ｜ 代码：✅ [Zenodo 开源（源码 + benchmark 一体）](https://doi.org/10.5281/zenodo.21512256)
- **[阅读完整解读](./2026-09-09_GraphDroid_Asynchronous_LLM-Based_Mobile_App_GUI_Testing_via_History-Aware_Exploration_and_Hybrid_Intent_Fulfillment.md)**

### Procedural Memory Under Change：程序性记忆"不匹配"是否必然致错？

- 标签：`Web` `Online` `Benchmark` `Reflection` ｜ 训练方法：`—（评测/工程；全程无梯度更新，本地冻结推理）`
- 域标注：**GUI**
- 一句话结论：用冻结 prompt 的受控实验检验"程序性记忆与当前任务不匹配时是否必然致错"，在 32 个一次性单元格中**未观察到任何预定义干扰签名**——是一个对"记忆有害"这一直觉的反证性结果。
- [arXiv](https://arxiv.org/abs/2609.09774) ｜ 代码：❌ 未开源（附录明确"No public repository upload or archival deposition is claimed by this draft"）
- **[阅读完整解读](./2026-09-09_Procedural_Memory_Under_Change_Reuse_and_Interference_in_Controlled_Web_Tasks.md)**

### Agentic Web Accessibility Auditing：给 40 条 WCAG 准则各配一个 VLM worker

- 标签：`Web` `Benchmark` `Planning` ｜ 训练方法：`—（评测/工程；模型全部冻结调用，"训练"实为人工撰写 40 份准则技能文档）`
- 域标注：**GUI**
- 一句话结论：给 40 条 WCAG 准则各配一个"带浏览器工具的 VLM worker"，在 250 条 page–criterion 记录上把违规召回从 axe-core 的 0.36 提到 0.86，代价是精确率降到 0.56。
- [arXiv](https://arxiv.org/abs/2609.09379) ｜ 代码：❌ 未开源（论文称随预印本提供补充材料，但未给出任何公开仓库 URL）
- **[阅读完整解读](./2026-09-08_Agentic_Web_Accessibility_Auditing_Authoring_and_Evaluating_Per-Criterion_Worker_Agents_for_WCAG.md)**

### The Menu Is an Execution Prior：把"工具菜单"当作在线 Agent 的执行先验

- 标签：`Online` `Planning` `Web` `General` ｜ 训练方法：监督训练（非 LLM 微调）——训练 retriever（0.416M）与 reranker（0.449M）两个小 Transformer，文本编码器 BGE-large-en-v1.5 冻结
- 域标注：**GUI**
- 一句话结论：把"执行前给 agent 看的短工具菜单"当作执行先验，用状态路径（entry→bridge→target→terminal）学出"选谁进菜单 + 谁排前面"，在不改 agent 的前提下把 ToolBench 在线成功率从 0.737 提到 0.898。
- [arXiv](https://arxiv.org/abs/2609.09395) ｜ 代码：❌ 未开源（论文未给出任何仓库、项目主页或代码可用性声明）
- **[阅读完整解读](./2026-09-08_The_Menu_Is_an_Execution_Prior_State-Path_Tool_Menus_for_Online_Agents.md)**

### AgentHijack：一个固定位置视觉 patch 就能让 Computer-Use Agent 执行恶意命令

- 标签：`Desktop` `Web` `Benchmark` ｜ 训练方法：`—（安全评测 / 对抗 patch 优化，不训练模型权重）`
- 域标注：**GUI**
- 一句话结论：训练一个 700×500 固定位置视觉 patch，驱动 Computer-Use Agent 真的执行恶意终端命令，并用 T-ASR/TAPR/E2E-ASR 三层指标量化视觉信号在 agent 流水线中的衰减。
- [arXiv](https://arxiv.org/abs/2609.09212) ｜ 代码：❌ 未开源（论文未给出仓库地址）
- **[阅读完整解读](./2026-09-06_AgentHijack_Visual_Patch_Attacks_on_Multimodal_Computer-Use_Agents.md)**

### 多模态 Prompt Injection 评测：尝试率是完成率的十倍，模型远比框架重要

- 标签：`Benchmark` `General` ｜ 训练方法：`—（评测 / 工程，不训练任何模型）`
- 域标注：**GUI**
- 一句话结论：用 MMPIBench 把同一批多模态注入攻击跑过 6 个 agent 框架 × 5 个前沿模型 × 6 种视觉载体，并逐阶段定位攻击死在哪里——结论是"尝试率是完成率的十倍"，且模型远比框架重要。
- [arXiv](https://arxiv.org/abs/2609.09404) ｜ 代码：❌ 未开源（论文称 "will be released on publication"，截至解读时未公开）
- **[阅读完整解读](./2026-09-08_An_Experimental_Evaluation_of_Multimodal_Prompt_Injection_Attacks_on_Agentic_AI_Frameworks.md)**

---

## Agent 方法论域（跨域参考，7 篇）

### Belief-State Engine：把贝叶斯滤波外挂给 LLM，补上"部分可观测"这块短板

- 标签：`Planning` `Online` `General` ｜ 训练方法：`—（无训练；外挂贝叶斯滤波 + 提示工程，LLM 权重冻结）`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：指出 LLM Agent 在部分可观测下失效的根因是"历史条件策略 ≠ 信念可测策略"，提出把贝叶斯滤波器外挂为 Belief-State Engine，只把后验信念喂给 LLM，并用四公理与六定理为这一分工给出健全性保证。
- [arXiv](https://arxiv.org/abs/2609.10036) ｜ 代码：✅ [bse-llm](https://github.com/debdipta-h/bse-llm)（论文脚注声明，未独立验证可用性）
- **[阅读完整解读](./2026-09-09_Belief-State_Engine_Augmenting_LLMs_for_Principled_Planning_Under_Partial_Observability.md)**

### Subagents vs Agent Skills：技能"怎么执行"比它"写了什么"更重要

- 标签：`General` `Planning` `Benchmark` ｜ 训练方法：`—（无训练；执行模式对照实验，全程无权重更新）`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：在 SkillsBench 上证明"可复用技能包怎么执行"比它写了什么更重要——把技能做成带显式输入/输出契约的过程性知识并以子代理（独立上下文）执行，显著优于把 SKILL.md 直接灌进主上下文。
- [arXiv](https://arxiv.org/abs/2609.09233) ｜ 代码：❌ 未开源（全文未提及任何代码仓库或项目主页）
- **[阅读完整解读](./2026-09-07_Subagents_vs_Agent_Skills_Executing_Reusable_Knowledge_for_Long-Horizon_Agentic_Tasks.md)**

### Building the Harness Automatically：把"如何花预算"蒸馏成 197 词文本

- 标签：`Distillation` `General` ｜ 训练方法：`—（无权重训练；代码级自博弈实践 + 一次性文本蒸馏、密封冻结后零样本部署）`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：让 Agent 在代码沙箱里反复写优化器并配对评测，把"如何花预算"的搜索纪律一次性蒸馏成 197 词的文本 harness，密封冻结后在未见模型与目标上把 Gemini Flash 的 regret 降 48%。
- [arXiv](https://arxiv.org/abs/2609.09468) ｜ 代码：⚠️ 匿名评审仓库 [llm-opt-sandbox](https://anonymous.4open.science/r/llm-opt-sandbox-E5B3/README.md)
- **[阅读完整解读](./2026-09-08_Building_the_Harness_Automatically_Self-Play_in_Code_Distills_a_Text_Harness_for_Black-Box_Optimization.md)**

### Do Agents Know When They Succeed：从残差流读出成败置信度

- 标签：`General` `Reflection` ｜ 训练方法：`—（不训练 LLM；仅在冻结模型残差流上训练 L2 逻辑回归探针 + 单调 Platt 校准）`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：从 LLM 残差流里读出多步 Agent 的成败置信度，用 LTD（轨迹动力学）与 ARP（动作表征探针）在三个交互式编程环境上零额外开销地压过表面 logprob 与 HTC。
- [arXiv](https://arxiv.org/abs/2609.09448) ｜ 代码：❌ 未开源（正文与附录均未给出仓库链接）
- **[阅读完整解读](./2026-09-08_Do_Agents_Know_When_They_Succeed_Calibrating_Agent_Confidence_from_Internal_Representations.md)**

### What Should an Agent Forget：把"存什么"和"用什么"拆开

- 标签：`General` ｜ 训练方法：`—（不训练；training-free 框架，curation/selection/answering 全靠冻结 LLM 提示工程）`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：RD-Forget 把 Agent 记忆的"存什么"和"用什么"拆开——源档案全量保留，用查询条件化的记忆视图 + 同槽替换抑制过期事实，training-free 地提升五类记忆任务。
- [arXiv](https://arxiv.org/abs/2609.10263) ｜ 代码：❌ 未开源（全文未提供代码或项目页）
- **[阅读完整解读](./2026-09-09_What_Should_an_Agent_Forget_Separating_What_Is_Stored_from_What_Is_Used.md)**

### TRACE（Agent）：用"验证不对称性"造 oracle 标签做 RL

- 标签：`RL` `SFT` `Benchmark` `General` ｜ 训练方法：`SFT`（对 Claude Opus 4.8 教师轨迹拒绝采样，1,200 条）+ `RL (GRPO)`（含 DAPO 类稳定化改进）
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：把"验证不对称性"工程化——先用可控模拟器注入隐藏干预、再生成观测，干预本身即 oracle 标签，从而在缺乏自然 verifier 的归因诊断任务上做 RL，让 35B 模型超过 Claude Opus 5。
- [arXiv](https://arxiv.org/abs/2609.10315) ｜ 代码：❌ 未开源（全文未提供代码、环境或项目页，模拟器与 oracle 均未发布）
- **[阅读完整解读](./2026-09-09_TRACE_Training_Reasoning_Agents_for_Causal_Exploration_with_Synthesized_Rewards.md)**

### Double Measurement Confound：Benchmark 分数被"脚手架代劳 + 评分器只评形状"双重污染

- 标签：`Benchmark` `General` ｜ 训练方法：`—（评测方法学，不训练任何模型）`
- 域标注：**Agent 方法论 · 跨域参考**
- 一句话结论：agent benchmark 的分数被"脚手架替模型做决策"和"评分器只评形状"两个缺陷同时污染，二者互相掩盖；只有联合修复（去脚手架 + 真值打分）才能让分数重新识别模型能力。
- [arXiv](https://arxiv.org/abs/2609.09218) ｜ 代码：✅ [comtrade-openenv](https://github.com/yonghongzhang-io/comtrade-openenv)（含 seeded generator、八模型可靠性谱、2×2 干预与 τ-bench / BFCL 审计脚本）
- **[阅读完整解读](./2026-09-06_The_Double_Measurement_Confound_in_Agent_Benchmarks_De-Scaffolding_Ground-Truth_Scoring_and_Reliability_Beyond_the_Mean.md)**

---

## 本日趋势归纳

**按标签统计**（每篇 1~4 标签）：`General`×12 ｜ `Benchmark`×9 ｜ `Planning`×5 ｜ `Web`×5 ｜ `Online`×3 ｜ `Reflection`×3 ｜ `Mobile`×2 ｜ `Desktop`×2 ｜ `GUI Grounding`×1 ｜ `RL`×1 ｜ `SFT`×1 ｜ `Distillation`×1
**按域分布**：GUI 8 篇（跨设备评测 1、效率/grounding 1、移动测试 1、程序性记忆 1、无障碍审计 1、工具菜单 1、安全攻击 2）｜ Agent 方法论 7 篇（记忆与知识资产化 3、规划与可观测性 1、评测可信度 2、RL 奖励构造 1）

**几个值得注意的方向信号**：
1. **跨设备（cross-device）成为 GUI 评测的新边界**：JarvisGUI 首次把 Android/Windows/Ubuntu 三端纳入统一任务图并做程序化判分，结果暴露"单设备高分 ≠ 真实可用"——这是继"混合 GUI+CLI"之后 GUI 域第二次扩边界，提示 GUI Agent 的能力评估正在向真实工作流靠拢。
2. **安全侧一天两篇，攻击面被系统化测绘**：AgentHijack 证明固定位置视觉 patch 即可劫持 Computer-Use Agent 执行恶意命令（并用三层 ASR 量化信号衰减），多模态注入评测则给出"尝试率是完成率十倍、模型比框架重要"的定量结论。二者合起来说明：**GUI Agent 的视觉输入通道是独立于文本的、可被单独攻击的信任边界**。
3. **效率路线的关注点从"剪多少"转向"何时不可逆"**：TRACE（GUI）把缓存复用下的 token 剪枝重述为写入时准入问题，指出一旦写入 KV cache 就无法撤回——这是工程上很关键但长期被忽视的约束。
4. **Agent 域"记忆存用分离"成为新共识**：RD-Forget 拆开存储与使用、Do Agents Know 从内部表征读置信度、Subagents vs Skills 强调技能的"执行契约"——三篇共同指向"不要把知识直接灌进上下文，而要设计检索/执行时的条件化视图"。
5. **评测可信度问题被正面提出**：Double Measurement Confound 指出"脚手架代劳 + 评分器只评形状"两个缺陷互相掩盖，只有联合修复才能让分数重新识别模型能力——对 GUI benchmark（尤其依赖脚本判分的）是直接警告。
6. **开源率仍偏低**：15 篇中仅 3 篇给出明确可访问的代码/仓库（GraphDroid、Belief-State Engine、Double Measurement Confound），1 篇为匿名仓库，11 篇未开源或仅占位声明。

---
*本速览为 2026-09-11 批次（补录：该批次精读已完成，速览与检索日志于 2026-09-12 补齐）*
