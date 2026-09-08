# SiLR: Structure-Preserving Admission and Process Reward for LLM Tool Agents

> 用"保留违规几何结构"的 product order 同时做运行时准入与 GRPO 过程奖励，破解标量门禁的"投影陷阱"。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | SiLR: Structure-Preserving Admission and Process Reward for LLM Tool Agents |
| **arXiv ID / DOI** | 2609.04629v1 [cs.AI] |
| **arXiv 链接** | https://arxiv.org/abs/2609.04629（点击直达） |
| **发表出处（Venue）** | arXiv preprint |
| **发布时间** | 2026-09-04 |
| **作者** | Chenyu Zhou、Qiliang Jiang（并列一作）；Shuning Wu；Xu Zhou（通讯，zhou_x_nus@u.nus.edu） |
| **所属机构** | 东京科学大学 Institute of Science Tokyo（日本）；浙江大学（中国）；新加坡国立大学 NUS（新加坡） |
| **开源情况 / 代码** | ❌ 未在文中声明开源 |
| **类型标签（论文类别）** | `RL` `Online` `General` |
| **训练方法标签** | `RL (GRPO) + LoRA`（策略侧）；运行期门禁为确定性 verifier + shadow 模拟（非学习模块） |
| **关键词** | 运行时安全、准入语义、标量投影陷阱、过程奖励、shadow 执行、ReAct |
| **来源渠道** | arxiv-api + listing |
| **PDF 存档** | 2026-09-04_SiLR_Structure-Preserving_Admission_and_Process_Reward_for_LLM_Tool_Agents.pdf |

---

## 1. 研究背景与要解决的问题

电网调度、数据中心、金融结算等关键基础设施正越来越多地由 LLM 工具 Agent 接管控制动作，运行时安全成为硬需求。已有治理系统（SARC 的 Pre-Action Gate、AgentSpec、GuardAgent、ShieldAgent、VeriGuard 等）共享一个隐含前提：系统从安全状态出发，门禁只需要阻止越界。但基础设施在自治控制下常常"起步即在违规中"——例如电网已有多条支路同时过载，需要连续多个动作才能回到安全区。此时问题反转过来了：某个动作即使不把系统带出违规，它是否属于"可接受的恢复性进展"？作者称之为 post-violation recovery admission，它和单纯拦截不安全动作在结构上不同：拦截不等于恢复（enforcement is not recovery）。论文核心论断是：门禁采用哪种"准入语义"（放行/拒绝的形式准则），而非门禁的严格程度，对能否完成多步恢复有一阶影响。只放行完全恢复终态的 terminal gate 安全，却会把多步恢复全部卡死；放宽为"聚合违规惩罚不上升（允许小松弛）"的标量门又在表示能力上不足——这正是论文命名的 scalar projection trap（标量投影陷阱）。

## 2. 核心方法 / 思路

作者先把门禁重新定位：在 ReAct 循环里，被拒的提案之后会在同一状态上再生成一个提案，因此门禁不是过滤器，而是作用于提案流的**搜索算子**，其准入准则直接决定哪些轨迹可达。标量投影陷阱即由此而生：把多维违规几何压成一个聚合分数后，门会接受"第一个让总分下降"的局部改进提案、提前结束本步搜索，把轨迹钉在残差平台上。例：(10,10)→(19,0.1)，过载集合未变、总分 20→19.1 反降，单支路却被推向 19。

SiLR（Structure-preserving In-the-Loop Recovery）的解法是保留几何结构。定义支路级违规状态 Φ(s)=(S(s),σ(s))：S 为过载支路集合（support），σ 为逐支路严重度向量，并在其上定义 product order：S 不扩张 且 逐支路 σ 都不增，才判为可接受进展。门禁在模拟器深拷贝上对每个提案做 shadow-execute，通过 ψ1（工具/参数校验）、ψ2（support 包含）、ψ3（逐支路严重度上限，α=1.05、ε=10⁻³）即准入，并输出 PASS / SAFE_PROGRESS / FAIL 分级裁决，FAIL 则在同一步内重新生成（每步预算 C_prop=3）。命题 1 证明：任何标量替代（加权和、max 范数、字典序）对该偏序都不 sound，缺口是表示层面的不可能性，不是阈值能调好的。信任边界上 LLM 完全不可信、verifier+模拟器可信；确定性仿真带来准入不变量（命题 2）：无论 LLM 行为如何，过载支路集合永不扩张，严重度有松弛包络。verifier 单次调用 17.5–30ms，相对 LLM 一步可忽略。

同一 product order 被复用到第二个设计点——过程奖励。把分级裁决与 Φ 构造成 GRPO 奖励（LoRA 微调 Qwen3-8B，权重 (0.6,0.3,0.3)）：按"消除支路 > 存活支路降严重度"排序，加 drift 项惩罚任何被推得更糟的存活支路，并按约束族归一化，防止高严重度族劫持信号。其 count 投影与 severity 标量作为对照（命题 3：count 奖励至多 B+1 个取值，98% 动作区间平坦）。

## 3. 关键实验结果

评测主战场是 Gym-ANM 电网（ANM6-Easy、Qwen3-14B、C_step=8），挖掘出 24 个 MPC 可恢复的多动作场景，并扩展到 CityLearn 楼宇能量调度与 Qwen3-8B/14B、Gemma-3-12B、Llama-3.1-8B 四族模型。匹配 N=7（每策略 21 条）下：terminal 门 0/21、无门 1/21、最优标量 9/21、Grid-Agent 式 support-only 20/21、SiLR 21/21。24 场景全量（N=5，各 120 条）：terminal 0/120、最优标量 96/120、support-only 113/120、SiLR 120/120（场景簇 Wilcoxon p=0.004）。标量松弛 η∈{0,0.05,0.10,0.20} 恢复数为 5/9/7/7——非单调本身就是陷阱信号；标量门失败的轨迹停在非零残差平台，而 MPC 可行oracle 从该平台一步归零，证明平台由门造成而非策略无能。安全性：注入/观测投毒/stall 攻击 60 条 0 成功、良性对照 0/15；幅值再分配攻击（支路集合与总分都不变、单支路逼近失效）标量放行 43/120、support-only 11/120、SiLR 0/120。全池枚举（ANM 558 / CityLearn 11311 个物理不安全动作）各标量均有 46–7650 例漏放，SiLR 两域均为 0；双约束族（电压+支路）42410 个不安全动作中标量放行 40.6–53.6%、support-only 高达 63.2%、product order 0。训练侧：几何奖励的 reward-level confusion 0.000 vs count 0.25（sign p=1.2×10⁻⁷）；去掉门后，几何奖励训练的策略是唯一高于未训练基线的（ungated 0.844 vs 0.778），verdict-only 0.733、count 0.667；15 个留出场景 ungated 恢复 14/15 vs 基线 9/15（p=0.04）。σ 异质/耦合两个压力 regime（16 seeds）下只有几何奖励全恢复：severity-scalar 0.00/0.04、count 0.80/0.12、几何 1.00/1.00。

## 4. 亮点与贡献（Why it matters）

- 概念重构：门禁是 ReAct 提案流上的搜索算子，用反链构造证明（命题 1/3）把"标量聚合准入失败"钉死为表示不可能性，而非阈值问题。
- 一个机制、两个设计点：同一个 product order 同时充当运行时准入准则与 GRPO 过程奖励，统一解释了准入死锁与奖励被 hack 两类现象。
- 工程可行性高：单步 shadow 即可，常数每步代价（~20ms），无需备份策略或轨迹级 lookahead，对黑盒 LLM 直接可用；门语义跨 4 模型族、2 个物理域原样迁移。
- 安全架构：准入权威从 LLM 移入确定性 verifier，对 compromised-prompt/observation 具备架构级（非提示加固式）遏制。
- 顺带贡献确定性过程监督：仿真器给出无需人工逐步标注的密集过程奖励，reward 与 oracle 一阶值 Spearman 相关 0.986。

## 5. 局限与可改进点（个人点评）

- 域假设偏窄：依赖"工具 API 存在确定性 shadow"与 shadow fidelity。电网/楼宇是稳态求解器支撑的理想域；GUI、Web、多数真实工具没有这种精确可克隆的确定性仿真，ψ2/ψ3 的成立条件难以复制。
- 规模偏小：ReAct 仅 8 步、每步 3 提案，多动作场景是"挖矿+分类"出来的 24 个合成场景，episode 数 N=5~7，统计功效有限；α/ε 与奖励权重 (0.6,0.3,0.3) 属手工设定。
- 训练侧只用 Qwen3-8B+LoRA、9 个场景做过程奖励微调，模型族间差异（Gemma 在 CityLearn CL-2 停滞）提示"策略内化恢复"尚未充分验证规模化。
- 威胁模型不含对 verifier/模拟器的攻击；FAIL 后在同一步再生受预算约束，若可接受提案不存在可能饿死（liveness 问题只在 oracle audit 里被部分回答）。
- 未声明开源，机制细节（补充材料）依赖 paper 描述，第三方复现成本高。

## 6. 对 GUI Agent 的可借鉴点（跨域参考）

方法迁移到 GUI 场景的对应关系很直接：把"工具调用"换成"GUI 操作"，把电网支路违规状态换成界面状态向量。GUI 任务大量是"起步即在违规"——权限弹窗、残留 overlay、错误登录态先于主任务存在，需要先恢复再执行；若把"被遮挡/缺失的关键元素、被破坏的页面状态"记为 support 与严重度，SiLR 就给"部分修复是否可放行"提供了可判定标准（先 shadow 验证不破坏已恢复组件）。

标量投影陷阱在 GUI 上尤其值得警惕：很多 GUI Agent 用"子目标完成数/总分"这类聚合进度做门控或奖励，很容易接受"让总分上涨却破坏某个必要组件"的提案（比如为了省步数提前关掉后面还要用的标签页/对话框）而陷入死胡同。把每窗口/每子目标做成 product order（都不允许倒退 + drift 惩罚），能在运行时把这类提案打回重试。过程奖励设计的启示：不要用 binary 成功或"完成了几步"的 count，而是按目标轴分解出几何奖励并加 drift 项——这在 GUI 训练里对应"画面状态不得被训练中产生的动作推向不可逆"的正则化。

GUI 环境的 shadow-execute 没有电网那么精确，但近似可用：Web 可用 headless 快照/会话克隆低成本试执行；模拟器型（Mobile/Desktop 虚拟机）可状态快照回滚。另外"准入权威归 verifier、LLM 在信任界外"的思路也适合 GUI 安全：对高危操作（删除、转账、外发）用确定性规则而非模型自检做放行裁决。

## 7. 延伸阅读

- 同域运行时治理：SARC、AgentSpec、GuardAgent、ShieldAgent、VeriGuard、Grid-Agent（rollback 式对照）。
- 安全 RL：Recovery RL、Model-Predictive Shielding、Shielding（Alshiekh et al. 2018）。
- 过程/结果奖励与 GRPO：Lightman et al.（Let's Verify Step by Step）、Uesato et al.、DeepSeekMath（GRPO）。
- 奖励 hack：Skalse et al.（Reward Hacking）、Krakovna et al.（Specification Gaming）。
- 本库相关解读：2026-09-02《LLM-as-a-Judge Is Not an Oracle》（确定性护栏）、2026-09-03《Making Every Tool Call Count》（工具路径奖励）。

---
*解读生成时间：2026-09-08 ｜ 解读人：WorkBuddy（AI）*
