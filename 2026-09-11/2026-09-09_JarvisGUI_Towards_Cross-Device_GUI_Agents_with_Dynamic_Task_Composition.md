# JARVISGUI: Towards Cross-Device GUI Agents with Dynamic Task Composition

> 一句话 TL;DR：用 slot 类型系统 + 采样式任务图组合，把 Android/Windows/Ubuntu 三端串成可自动生成、可程序化判分的跨设备 GUI benchmark，实测 SOTA 开源 Agent 在跨设备依赖任务上几乎全军覆没。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | JARVISGUI: Towards Cross-Device GUI Agents with Dynamic Task Composition |
| **作者 / 机构** | Zixiang Chen\*、Yuheng Lu\*（同等贡献），Zeming Liu†（通讯），Zihao Cheng、Jizeng Bai、Ziye Huang、Zhiyin Lin、Zihan Li、Yuhang Guo、Yunhong Wang、Haifeng Wang；北京航空航天大学（中国）、北京理工大学（中国）、百度公司（中国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-09；arXiv preprint（arXiv:2609.10451v1，cs.AI），未见 Venue |
| **arXiv 链接** | https://arxiv.org/abs/2609.10451 |
| **代码仓库** | ❌ 未开源（脚注仅写 "will be publicly available at JarvisGUI"，无任何 URL，属占位声明） |
| **数据集地址** | ❌ 未公开（同上，无链接） |
| **类型标签（论文类别）** | `Benchmark` `Mobile` `Desktop` `General` |
| **训练方法标签** | —（综述/评测/工程） |
| **关键词** | Cross-Device GUI Agent；Benchmark；Type System；Task Composition；Long-Horizon Planning |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-09_JarvisGUI_Towards_Cross-Device_GUI_Agents_with_Dynamic_Task_Composition.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：真实工作流常横跨手机、PC、服务器（拍照 → 传云 → Ubuntu 上转黑白 PNG → 传回手机 `/sdcard/Pictures`）。本文要问：现有 GUI Agent 能否完成这种跨异构设备的端到端任务？答案是基本不能。
- **为什么重要**：跨设备任务天然长程、有依赖、有中间态（文件位置、剪贴板、登录会话）。它卡住的是 Agent 从 demo 走向生产力这一步；不解决，所有单设备指标都会系统性高估 readiness。
- **现有方法有什么不足**：核心是**单设备假设**——ScreenSpot/AndroidControl/AITW/OSWorld/OmniBench 等全部把任务限制在单一 OS 内。唯一承认跨设备重要性的 CRAB 被点名：**仅 18 条跨设备任务**，且**不建模真实文件/数据传输**。此外主流 Agent（含 UI-TARS-2、AutoGLM）的官方推理实现只接收**单个活跃平台**观测。
- **Research Gap**：作者自认的缺口是首次系统性研究跨设备 GUI 任务，并用**可自动组合、可验证、可扩展**的建模框架把跨设备 benchmark 规模化，暴露被现有 benchmark 遮蔽的能力缺口。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把每个原子 GUI 任务形式化为「带类型的输入槽 → 输出槽」转换，用类型兼容性约束做采样式任务图拼接以自动合成跨设备长程任务，并在多 OS 虚拟环境中按最终环境状态程序化判分。

### 3.2 方法总览（Pipeline）

- **输入**：自动改写生成的自然语言指令 + 三平台实时截图 + 交互历史。**输出**：逐步抽象动作（平台 + 工具动作 + 元素描述），Grounder 补坐标，环境按 evaluator 判分。
- **五类模块**：①**类型系统**——`T` 带偏序 `⪯`，合法性由 `τ_out ⪯ τ_in` 保证（如 `xlsx_file ⪯ ordinary_file`），是自动组合的「编译期检查」。②**任务图** `G=(V,E)`，节点为任务实例 `T_v=(I_v,O_v,D_v,Φ_v,P_v)`，边表示上游输出槽喂给下游输入槽。③**数据采集**——人工标注**模板任务**（描述与 evaluator 均用 `{{slot}}` 占位符实现 input-agnostic）→ 采样式自动组合（含 65 条文件传输辅助任务）→ 值传播 `(O,D,Φ)←M(v_I)` → Qwen3-32B 重写指令。④**质量过滤**——Qwen3-30B-A3B-Thinking-2507-FP8 按三维 5 分制打分，三项全 5 分才保留。⑤**四层环境**——Docker+QEMU/KVM → 统一观测/动作接口 → Planner-Grounder → 并行调度 + 终态判分。
- **全局成功判据**：`Success(G) ⟺ ∀v∈V, ∀φ∈Φ_v, φ(s_v^final)=1`——任一子任务任一 evaluator 不过即整链失败。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：①**slot-typing 原子任务建模**——用带偏序的类型系统形式化「中间产物可传递性」，使组合合法性可**自动校验**，让跨设备任务从「手写 18 条」变成「按类型采样组合」。②**参数化 evaluator 动态实例化**——evaluator 以文本占位符指定判据、运行时按上游产物格式化，解决「任务动态生成但判分须确定」的矛盾。③首次把「跨设备状态传递」作为显式评测维度。
- **工程组合**：Docker+QEMU/KVM 虚拟化、统一观测/动作空间、Planner-Grounder 两阶段架构（附录自承是 "representative instantiation"）、Qwen3-32B 指令重写、LLM-as-Judge——均为成熟组件拼装。
- **对性能提升最关键的设计**：本文**无组件消融**，无法 ablation 归因。从「暴露缺口」目标看最关键的是 **MD 类别**——它把整体 TSR 从 atomic 42.4% 打到 8.0%；以及**纳入 Windows**（最高仅 18.4%，远低于 Ubuntu 60.7%）。
- **证据不足 / 仅声称有效**：①**"first to systematically study cross-device tasks" 过度声称**，CRAB 已开先河。②**「Windows 差因预训练数据偏置」是纯事后假说**，无语料统计、无对照。③**Llama 4 Maverick 近零分被当作能力证据**，但作者自述是「无法稳定产出规定 action 格式」——属 scaffold 格式鲁棒性问题。④**无 scaffold 消融**，benchmark 难度与 scaffold 质量无法解耦。⑤抽检 50 条组合任务有 4 条描述不精确（约 8%）未剔除。---

## 4. 具体技术细节

### 4.1 模型结构

- **base model 全部是 MLLM（含视觉编码器）**：6 个被评测 Agent 为 Qwen3-VL-30B-A3B-Instruct、HOLO2-30B-A3B、UI-TARS-1.5-7B、MAI-UI-8B、UI-Venus-Ground-7B、GUI-Owl-32B，参数量 7B/8B/30B(A3B)/32B。
- **Planner** 用通用 VL 模型 **Qwen3-VL-Plus** 做跨平台高层推理；**Grounder** 是各 Agent 自带 grounding 模块，只在指定平台预测坐标。选通用模型而非专用 GUI 模型，因其在单设备数据上 post-train，泛化受限。
- **是否冻结/微调：不训练、不微调任何模型**，全部现成权重或 API 零样本推理。

### 4.2 训练流程（若需要训练）

**不训练。** 本文是纯 benchmark + 评测工作，不含 SFT / RL / 蒸馏。其阶段化流程只存在于 benchmark 构建侧：

| 阶段 | 目标 | 产出能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| S1 模板标注 | 得 input-agnostic 原子任务 | 可参数化实例化 | 人工标注者 | 描述 + `{{slot}}` + 参数化 Python evaluator | 无 loss；须人类 20 步内可完成，evaluator 执行前报 incomplete、执行后报 complete |
| S2 自动组合 | 规模化生成跨设备 workflow | 类型合法任务图 | S1 原子任务 + 65 条文件传输辅助任务 | 有向图（节点=任务实例，边=类型兼容槽连接） | 无 loss；由 `τ_out ⪯ τ_in` 保证 well-typed |
| S3 指令重写 | 合并子任务描述成连贯指令 | 最终自然语言描述 | 组合任务的全部子任务描述 | 纯文本 | 无 loss；Qwen3-32B 须严格保留全部程序性细节 |
| S4 LLM 过滤 | 剔除不真实/不连贯/不可评估任务 | 150 条高质量组合任务 | 采样的 2000 条候选 | 文本任务 + 三维评分 JSON | 无 loss；三维均须 5 分；与人工判断 Cohen's Kappa ≈ 0.92 |
### 4.3 推理流程（若不训练或重点在推理）

- **推理模型**：Planner = Qwen3-VL-Plus；Grounder = 各被评测 Agent 的 grounding 模块。
- **输入/输出**：Planner 每步接收**三平台截图** + 指令 + 历史（含前序 `step_plan` 与框架执行错误），输出**恰好一个** JSON 步骤 `{current_status, platform, step_plan, element_description, tool action}`，并被硬约束**禁止输出坐标**；Grounder 再在选定平台截图上预测坐标。
- **单步还是多步：多步。** 采用显式 **Observation–Planning–Action** 协议循环，机制是「规划-执行」两级，终止条件为任务结束或达步数上限，随后评测层拉取终态执行全部 evaluator。框架把非法动作错误注入历史使 Planner 感知无效输出，但**无反思/重规划模块**，中途步骤失败（如云传输登录超时）后后续规划即被证明徒劳。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| Atomic Tasks | Mobile（Android 24）/ Desktop（Ubuntu 56、Windows 38） | 单设备变参执行鲁棒性 | 118 条，31 App | 三平台截图 + 指令 | 抽象动作 + 坐标 | TSR |
| Comp. SW | 单平台带依赖 | 单设备多步依赖 | 50 条，12 App | 同上 | 同上 | TSR / SSR |
| Comp. MI | 跨平台无依赖 | 多设备独立子任务协调 | 50 条，26 App | 同上 | 同上 | TSR / SSR |
| Comp. MD | 跨平台带依赖 | 跨设备状态传递 + 长程关键路径 | 50 条，18 App | 同上 | 同上 | TSR / SSR |
| 合计 | Android+Windows+Ubuntu | 混合 | 150 组合任务 / 442 子任务（Ubuntu 187、Windows 138、Android 52）+ 65 文件传输辅助 | — | — | — |

`TSR = (1/|T|)·Σ1_success(t)`；`SSR = (1/|T_comp|)·Σ|s_passed|/|s_total|`。JarvisGUI 是 Table 1 中唯一同时满足 CD/MS/DE/DT 四项的 benchmark，CRAB 仅部分满足 CD。

### 5.2 实验结果分析

**主结果（Table 3，Planner 统一为 Qwen3-VL-Plus）**

| 模型 | 规模 | Android | Windows | Ubuntu | Atomic 总 TSR | SW TSR | MI TSR | MD TSR | Multi 总 TSR | Multi 总 SSR |
|---|---|---|---|---|---|---|---|---|---|---|
| UI-Venus_Ground | 7B | 33.3 | 18.4 | 51.8 | 37.3 | 4.0 | 0.0 | 0.0 | 1.3 | 16.7 |
| UI-TARS-1.5 | 7B | **50.0** | 18.4 | 44.6 | 37.3 | 14.0 | 4.0 | 0.0 | 6.0 | 18.8 |
| MAI-UI | 8B | 37.5 | 7.9 | 39.3 | 28.8 | 8.0 | 4.0 | 0.0 | 4.0 | 16.1 |
| GUI-Owl | 32B | 41.7 | 18.4 | 51.8 | 39.0 | 12.0 | 8.0 | 2.0 | 7.3 | 18.1 |
| Qwen3-VL-Instruct | 30B(A3B) | 37.5 | 13.2 | 50.0 | 35.6 | 10.0 | 2.0 | 0.0 | 4.0 | 18.1 |
| HOLO2 | 30B(A3B) | 41.7 | 15.8 | **60.7** | **42.4** | **16.0** | 6.0 | 2.0 | **8.0** | **19.0** |

- **关键数字**：最强 Agent（HOLO2）atomic 仅 42.4%，组合任务 TSR 仅 **8.0%（12/150）**、子任务 SSR 19.0%（84/442）；**MD 类最好 2.0%**（GUI-Owl、HOLO2），UI-Venus/UI-TARS/MAI-UI/Qwen3-VL 全部 **0.0%**。
- **95% bootstrap CI（Table 9）**：HOLO2 atomic 42.4% [33.9, 51.7]（50/118）、组合 8.0% [4.0, 12.7]（12/150）、子任务 19.0% [14.8, 23.6]（84/442）。
- **平台偏置与规模**：Windows 最高仅 18.4%，Ubuntu 60.7%，Android 50.0%；GUI-Owl-32B / HOLO2-30B 未显著优于 7B 的 UI-TARS-1.5（39.0/42.4 vs 37.3），瓶颈不在参数量。换 planner 换成 Kimi K2.6 后 Atomic 42.4→48.3、Multi 8.0→**11.3**，跨设备退化依旧（Table 8）；Llama 4 Maverick 因格式失败全任务近零分。

**Ablation 说明了什么**

- **本文没有组件消融**（benchmark 论文无模型组件可消融），可替代的定量证据只有两类：①**任务粒度对照**（Atomic→SW→MI→MD：42.4%→16.0%→6.0%→2.0%）单调坍塌，强有力地定位「跨设备依赖」是主因，而非「多步」或「多设备观测」本身；②**Planner 替换对照**（Table 8）说明瓶颈不特定于某 planner，但 Llama 4 的近零分是格式失败，属混淆证据。
- **缺失的关键消融**：三屏同时输入 vs 按需单屏输入、有无 Grounder 解耦、有无执行错误反馈——这三项直接决定「跨设备认知/视觉过载」的解释是否成立，均未做。误差分析（Figure 4）另显示子任务数 ≥4 时所有模型成功率**趋近于零**，失败模式是「未执行的依赖步骤被当成已完成」的级联错误。

---

## 6. 亮点与贡献（Why it matters）

1. **把「中间产物可传递」变成可编译约束**：类型系统让跨设备任务图合法性可自动校验，是把 GUI benchmark 从「人工写任务」推进到「按语义组合任务」的关键一步，这套 slot/evaluator 参数化思路可直接借鉴。
2. **给出足够刺眼的负结果**：组合任务 8.0%、MD 任务 2.0%、子任务 ≥4 即崩，是对「GUI Agent 已接近可用」叙事的有力反证。
3. **明确三类被遮蔽的能力缺口**：state-transfer awareness、cross-platform contextual reasoning（「C 盘」→ Windows）、long-horizon dependency management（剪贴板/临时路径中间态）。
4. **提供跨设备工作流真实性的 HCI 证据**：67.5% 用户拥有 ≥2 台设备、37% 的多设备案例在单任务内顺序使用多设备。

## 7. 局限与可改进点（个人点评）

- **最关键问题是没有 scaffold 消融**。结论全架在「Qwen3-VL-Plus planner + 专用 Grounder」这一强组合上；Windows 只有 18.4%、MD 只有 2.0% 时，无法判断这是 benchmark 太难还是 scaffold 太弱。
- **「跨设备认知过载」缺直接证据**。归因于「同时处理三屏降低视觉 grounding 精度」，却无「按需单屏输入」的对照；若换单屏后性能不变，结论就应是「状态推断」而非「视觉过载」。Windows 偏置归因于「预训练语料分布」也属空口假说。
- **规模偏小且可复现性有硬门槛**。150 条组合任务 / 442 子任务，TSR 的 CI 宽达 ±4~5 个点；环境依赖 Docker + KVM，受限/云环境可能无法运行。
- **场景覆盖与标注噪声**：缺无障碍场景；50 条抽检中 4 条描述不精确却未清理——MD 类只有 50 条，2~4 条歧义就足以让 2% 的 TSR 不可解释。

## 8. 对我们的启示 / 可借鉴点

**对 GUI Agent 的落地启示（本篇为 GUI 域 benchmark）：**

1. **评测维度应显式包含「跨设备状态传递」**。只按平台分层会漏掉 MI/MD 这类最有信息量的类别；建议引入 SW/MI/MD 三分法，并把「上游产物是否被下游正确消费」作为独立指标。
2. **参数化 evaluator 可直接复用**：同一套判分逻辑可在任意参数实例上复用，无需为每条任务重写判分代码。
3. **类型系统是低成本的数据合成质量门**：一条 `τ_out ⪯ τ_in` 约束即可在组合阶段拦掉多数语义不连贯的任务图，比事后 LLM 过滤更便宜、更确定。
4. **长程失败是级联而非线性**：≥4 子任务即近零成功率，说明必须给 Agent 加**中间态显式记账**（「文件已上传云」须被验证而非假设成功）。别用单一强 scaffold 的出分当能力上界。

## 9. 延伸阅读

- **CRAB: Cross-environment Agent Benchmark for Multimodal Language Model Agents**（Xu et al., ACL Findings 2025）——跨设备评测先行者（仅 18 条任务）。
- **OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks in Real Computer Environments**（Xie et al., NeurIPS 2024）——动态桌面环境评测标杆，本文模板标注与配置设计的参考。
- **UI-TARS-2 Technical Report: Advancing GUI Agent with Multi-turn Reinforcement Learning**（Wang et al., 2025）——端到端 Agent 官方 formulation 只在单活跃 GUI 环境内定义循环，是「必须用两阶段 Planner-Grounder」的关键论据。

---
*解读生成时间：2026-09-11 09:00 ｜ 解读人：WorkBuddy（AI）*
