# SiLR: Structure-Preserving Admission and Process Reward for LLM Tool Agents

> 一句话 TL;DR：把门禁重构为 ReAct 提案流上的搜索算子，用保留违规几何结构（support + severity）的 product order 同时做运行时准入与 GRPO 过程奖励，证明标量投影的失败是表示层面的不可能性。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | SiLR: Structure-Preserving Admission and Process Reward for LLM Tool Agents |
| **作者 / 机构** | Chenyu Zhou\*、Qiliang Jiang\*（并列一作）；Shuning Wu；Xu Zhou（通讯，zhouxu_nus@u.nus.edu）。机构：东京科学大学 Institute of Science Tokyo（日本）、浙江大学（中国）、新加坡国立大学 NUS（新加坡） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-04；arXiv preprint，分类 [cs.AI] |
| **arXiv 链接** | https://arxiv.org/abs/2609.04629（点击直达） |
| **代码仓库** | ❌ 未开源（文中未声明开源，机制细节依赖补充材料） |
| **数据集地址** | 未公开独立数据集；评测环境为开源 Gym-ANM（arXiv:2103.07932）与 CityLearn v1.0（开源 Gym 环境） |
| **类型标签（论文类别）** | `RL` `Online` `General` |
| **训练方法标签** | `RL (GRPO) + LoRA`（策略侧）；运行期门禁为确定性 verifier + shadow 仿真（非学习模块） |
| **关键词** | 运行时安全、准入语义、标量投影陷阱、过程奖励、shadow 执行、ReAct |
| **来源渠道** | arxiv-api + listing |
| **PDF 存档** | 2026-09-04_SiLR_Structure-Preserving_Admission_and_Process_Reward_for_LLM_Tool_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：电网调度、数据中心等关键基础设施的 LLM 工具 Agent，在**起步即在违规中**（post-violation）时，门禁该如何判定"仍不安全的动作是否算可接受的恢复性进展"并放行，从而让 Agent 通过连续多步回到安全区。
- **为什么重要**：自治控制下系统常在违规态启动，需要多个动作才能恢复。拦截不等于恢复（enforcement is not recovery），若门禁放行准则不对，会把多步恢复整个卡死，或把轨迹钉死在"残差平台"上无法脱困。
- **现有方法有什么不足**：SARC Pre-Action Gate、AgentSpec、GuardAgent、ShieldAgent、VeriGuard 共享一个隐含前提——系统从安全态出发，门禁只需拦截越界，因此都不处理 post-violation recovery admission。terminal gate（仅放行完全恢复终态）安全却死锁：每个中间提案都 S≠∅，多动作场景 0/21。自然的放宽是标量门（聚合违规惩罚不上升即放行），但它有"标量投影陷阱"：把多维违规几何压成总分后，会接受"总分下降却把某支路推向更糟"的局部改进提案（例：(10,10)→(19,0.1)，总分 20→19.1 反降，单支路却被推到 19）。
- **Research Gap**：作者 claim 补上了"准入语义"这一缺口——门禁采用哪种形式准则（而非严格程度）对多步恢复有一阶影响，且标量替代对该偏序不 sound 是**表示不可能性**（命题 1），不是阈值能调好的。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

用 product order（过载支路集合不扩张 + 逐支路严重度分量不增）作为唯一机制，在**两个设计点**复用：运行时准入 gate + GRPO 过程奖励。

### 3.2 方法总览（Pipeline）

- **输入**：ReAct 循环中每步 LLM 的工具调用提案 a_t + 当前状态 s_t；**输出**：PASS / SAFE_PROGRESS / FAIL 分级裁决（FAIL 则同一步内重新生成）。
- **模块**：① shadow verifier 在模拟器深拷贝上执行 `ŝ_{t+1} = solve(apply(copy(s_t), a_t))`；② 三个谓词 ψ1（工具/参数校验）∧ ψ2（support 包含，S(ŝ)⊆S(s)）∧ ψ3（逐支路严重度上限，σi(ŝ) ≤ max(ασi(s), σi(s)+ε)，α=1.05、ε=10⁻³）；③ product order 门禁据此裁决。
- **连接**：LLM 在信任边界外，verifier+模拟器在信任边界内；准入权威完全在 verifier，LLM 的推理不参与任何裁决。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：把门禁从"过滤器"重定义为 ReAct 提案流上的**搜索算子**，并给出标量投影陷阱的形式化证明（命题 1 用反链构造证伪所有标量替代）；用同一 product order 统一解释准入死锁与奖励被 hack 两类现象。
- **工程组合**：shadow 执行、ReAct、GRPO、LoRA 均为已有组件，本身不新；α/ε 的 ε-支配松弛借鉴多目标优化（Laumanns 2002）。
- **对性能提升最关键的设计**：ψ2（support 包含）贡献了 liveness（20/21 已接近全恢复），ψ3（逐支路 severity 守卫）贡献了安全性/遏制（magnitude redistribution 攻击只有 ψ3 能挡住）；奖励侧 drift 项是几何奖励的核心（补上 drift 才能救回 coupling regime）。
- **证据不足 / 仅声称有效**：shadow fidelity 假设（live 复现 shadow post-state）仅在确定性稳态求解器域成立，GUI 等域无对应；"一个机制两设计点"的统一性更多是概念主张，两处实验各自独立验证，缺少联合消融。

---

## 4. 具体技术细节

### 4.1 模型结构

纯文本 LLM（**非 MLLM，无视觉编码器**），工具调用式 Agent。主评测用 Qwen3-14B（vLLM 服务，temperature 0）；策略训练用 Qwen3-8B + LoRA 微调（greedy decoding，无 CoT）。门禁/verifier 为非学习的确定性模块（Newton–Raphson 潮流求解器 + 谓词）。

### 4.2 训练流程

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| Stage 1（唯一） | 让策略"内化"恢复几何，去掉门后仍能自恢复 | 按 product order 下降的多步恢复策略 | Gym-ANM 挖矿出的 9 个最难多动作场景（ungated 基线无法可靠恢复） | ReAct 轨迹（C_step=8，每步 shadow 裁决） | GRPO；过程奖励 ρgeom = W2·Σ_f σ(E∩f)/σ(f) + W3·Σ_f Σ_{k∈U∩f}(σk−ŝk)+/σ(f) − Wd·min(1, max_{k∈U}(ŝk−σk)+/σk)，权重 (W2,W3,Wd)=(0.6,0.3,0.3) |

**过程奖励设计（重点）**：分级裁决 + 保留的支路状态 Φ 构造成 GRPO 信号，无需人工逐步标注。三个关键选择：① support 消除（E=S\Ŝ）权重 0.6 高于存活支路降严重度（U=S∩Ŝ）权重 0.3，呼应准入序；② drift 项（Wd=0.3）惩罚任何被推得更糟的存活支路；③ 按约束族归一化并等权平均，防止高严重度族劫持信号。对照：count 投影 ρcount=(|S|−|Ŝ|)/|S| 至多 B+1 个取值、98% 动作区间平坦（命题 3），verdict-only 则把 ρ 压成常数。

### 4.3 推理流程

多步 ReAct 循环：每控制步观察 → 提案 → shadow 验证 → 准入/FAIL 再生（每步预算 C_prop=3）→ 应用；终止于 S=∅（完全恢复）或步数耗尽 C_step=8。verifier 单次调用 17.5–30ms，相对 LLM 单步可忽略。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| Gym-ANM ANM6-Easy（电网） | General（物理仿真） | 多步恢复/安全控制 | 24 个 MPC 可恢复多动作场景（自 600 场景池挖矿分类：153 单动作/24 多动作/252 MPC-residual/171 trivial） | 文本状态 + 工具调用 | setpoint 动作 | recovery rate / 最终残差惩罚 / 不安全动作放行数 |
| CityLearn v1.0（楼宇储能） | General（物理仿真） | 多步恢复/能量调度 | 3 个 district 场景 CL-1~CL-3 + 4 楼宇扩展套件 | 文本状态 | 充放电动作 | recovery rate |
| 攻击/枚举池 | General | 安全遏制 | ANM 558、CityLearn 11311 个物理不安全动作；双约束族 42410 个 | — | — | 不安全动作放行数 |

### 5.2 实验结果分析

**主结果（RQ1）**：matched N=7（21 episodes/policy）——terminal 0/21、ungated 1/21、最优标量 9/21、support-only（Grid-Agent 式）20/21、Structured 21/21（残差 0.000）。24 场景全量 N=5：terminal 0/120、最优标量 96/120、support-only 113/120、Structured 120/120（scenario-clustered Wilcoxon p=0.004）。

**RQ2 标量松弛**：η∈{0,0.05,0.10,0.20} 恢复 5/21、9/21、7/21、7/21——非单调本身就是陷阱信号（更宽 slack 反而更差）。RQ3：MPC 可行性 oracle 从标量平台一步归零，证明平台由门造成而非策略无能。

**RQ4 安全**：注入/观测投毒/stall 攻击 60 条 0 成功、良性 0/15；magnitude redistribution 攻击标量 43/120、support-only 11/120、Structured 0/120。全池枚举：ANM 558 个不安全动作 sup-norm 放行 46、lex 88、aggregate 108、support-only 258、product order 0；CityLearn 11311 个中 sup-norm 6761、product order 0；双约束族 42410 个中 support-only 放行最多 63.2%、各标量 40.6–53.6%、product order 0。

**RQ5 跨模型/跨域**：ANM 四模型（Qwen3-8B/14B、Gemma-3-12B、Llama-3.1-8B）terminal 全 0/15、Structured 全 15/15；CityLearn 中 terminal 5/15、Structured 10–15/15（Gemma 在 CL-2 停滞是能力问题，非门语义问题）。

**Ablation（训练侧）**：几何奖励 reward-level confusion 0.000 vs count 0.25 vs binary 1.0（sign p=1.2×10⁻⁷，Spearman 0.986）；去掉门后 ungated recovery——几何 0.844 > 基线 0.778 > verdict-only 0.733 > count 0.667；15 留出场景几何 14/15 vs 基线 9/15（p=0.04）。压力 regime（16 seeds）：σ-het 几何 1.00/severity-scalar 0.00/count 0.80；coupling 几何 1.00/severity-scalar 0.04/count 0.12/count+drift 1.00——只有完整几何奖励同时覆盖两种 regime。

---

## 6. 亮点与贡献（Why it matters）

- **概念重构**：门禁是搜索算子而非过滤器，用反链构造（命题 1/3）把"标量聚合准入失败"钉死为表示不可能性，而非阈值问题。
- **一个机制、两个设计点**：同一 product order 既做准入准则又做过程奖励，统一解释准入死锁与奖励 hack 两类现象。
- **工程可行性高**：单步 shadow 即可，常数每步代价（~20ms），无需备份策略或轨迹级 lookahead，对黑盒 LLM 直接可用；门语义跨 4 模型族、2 个物理域原样迁移。
- **安全架构**：准入权威从 LLM 移入确定性 verifier，对 compromised-prompt/observation 具备架构级（非提示加固式）遏制。
- **顺带贡献确定性过程监督**：仿真器给出无需人工逐步标注的密集过程奖励，reward 与 oracle 一阶值 Spearman 相关 0.986。

---

## 7. 局限与可改进点（个人点评）

- **域假设偏窄**：依赖"工具 API 存在确定性 shadow"与 shadow fidelity。电网/楼宇有稳态求解器支撑是理想域；GUI、Web、多数真实工具无精确可克隆的确定性仿真，ψ2/ψ3 成立条件难以复制。
- **规模偏小**：ReAct 仅 8 步、每步 3 提案；24 个多动作场景是"挖矿+分类"的合成场景，episode 数 N=5~7，统计功效有限；α/ε 与奖励权重 (0.6,0.3,0.3) 手工设定。
- **训练侧验证不足**：仅 Qwen3-8B+LoRA、9 个场景做过程奖励微调，模型族间差异（Gemma 在 CityLearn CL-2 停滞）提示"策略内化恢复"尚未规模化验证。
- **威胁模型不完整**：不含对 verifier/模拟器本身的攻击；FAIL 后同一步再生受预算约束，若可接受提案不存在可能饿死（liveness 仅在 oracle audit 中被部分回答）。
- **未开源**：机制细节依赖 paper 描述，第三方复现成本高。

---

## 8. 对我们的启示 / 可借鉴点

本篇属【Agent 方法论域】（通用工具 Agent，非 GUI 落地），故单列：

**对 GUI Agent 的可借鉴点**：

- **概念迁移直接**：把"工具调用"换成"GUI 操作"，把"支路违规状态"换成"界面状态向量"。GUI 任务大量"起步即在违规"——权限弹窗、残留 overlay、错误登录态先于主任务存在，需先恢复再执行；把"被遮挡/缺失的关键元素、被破坏的页面状态"记为 support 与严重度，SiLR 就给"部分修复是否可放行"提供了可判定标准（先 shadow 验证不破坏已恢复组件）。
- **警惕标量投影陷阱**：很多 GUI Agent 用"子目标完成数/总分"做门控或奖励，易接受"总分上涨却破坏某必要组件"的提案（如为省步数提前关掉后面还要用的标签页/对话框）。把每窗口/每子目标做成 product order（都不允许倒退 + drift 惩罚），能在运行时打回这类提案重试。
- **过程奖励设计**：不要用 binary 成功或"完成几步"的 count，而是按目标轴分解出几何奖励并加 drift 项——对应 GUI 训练里"画面状态不得被训练动作推向不可逆"的正则化。
- **shadow-execute 近似可用**：Web 用 headless 快照/会话克隆低成本试执行，模拟器型（Mobile/Desktop 虚拟机）可状态快照回滚；"准入权威归 verifier、LLM 在信任界外"的思路也适合 GUI 高危操作（删除、转账、外发）用确定性规则而非模型自检裁决。

---

## 9. 延伸阅读

- 同域运行时治理：SARC、AgentSpec、GuardAgent、ShieldAgent、VeriGuard、Grid-Agent（rollback 式对照）。
- 安全 RL：Recovery RL、Model-Predictive Shielding、Shielding（Alshiekh et al. 2018）。
- 过程/结果奖励与 GRPO：Lightman et al.（Let's Verify Step by Step）、Uesato et al.、DeepSeekMath（GRPO）。
- 奖励 hack：Skalse et al.（Reward Hacking）、Krakovna et al.（Specification Gaming）。
- 本库相关解读：2026-09-02《LLM-as-a-Judge Is Not an Oracle》（确定性护栏）、2026-09-03《Making Every Tool Call Count》（工具路径奖励）。

---
*解读生成时间：2026-09-08 ｜ 解读人：WorkBuddy（AI）*
