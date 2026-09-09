# Elastic Horizon: Discovering the Effective Interaction Frontier in Agentic Reinforcement Learning（弹性视界：在 Agentic RL 中发现"有效交互边界"）

> **TL;DR**：交互步数上限并不总是越大越好——训练会饱和；Elastic Horizon 用"成功轨迹长度的 P90"做闭环控制，自动扩张/收缩每集交互预算到有效边界 H*，在 AppWorld/BFCL 上 7B/14B 双模型拿到最优成功率，同时每步 token 最多省 25%。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | Elastic Horizon: Discovering the Effective Interaction Frontier in Agentic Reinforcement Learning |
| **arXiv ID / DOI** | arXiv:2609.07247v1（cs.AI） |
| **arXiv 链接** | https://arxiv.org/abs/2609.07247（点击直达） |
| **发表出处（Venue）** | EMNLP 2026（accepted，17 页） |
| **发布时间** | 2026-09-07 |
| **作者** | Gangyi Zhang*、Junjie Meng*（*共同一作）、Letian Zhang、Wei Wu、Yang Zheng、Dong Wang†、Yang Liu、Guanjun Jiang、Chongming Gao†（共 9 人，†通讯） |
| **所属机构** | 阿里云 Qwen 业务部（Qwen Business Unit of Alibaba，中国）；中国科学技术大学（USTC，中国）；另有独立研究者 |
| **开源情况 / 代码** | ✅ 有代码：https://github.com/junjie-meng/ElasticHorizon |
| **类型标签（论文类别）** | `RL`、`Online`、`General` |
| **训练方法标签** | `RL (GRPO)`——GRPO（KL=0）在 Qwen2.5-7B/14B 上训练，基于 AgentEvolver 框架；创新点是训练超参"每集交互预算"的闭环调度 |
| **关键词** | 有效交互边界（effective interaction frontier）、horizon scheduling、agentic RL、闭环控制、成功轨迹长度、样本效率、GRPO |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-07_Elastic_Horizon_Discovering_the_Effective_Interaction_Frontier_in_Agentic_Reinforcement_Learning.pdf |

---

## 1. 研究背景与要解决的问题

训练长程 agent 时，"每集最多与环境交互多少步"（interaction horizon / budget）是必须定的超参。近两年主流做法是**开环 curriculum**：要么线性拉长（ScalingInter-RL，AgentGym-RL 的工作），要么分段成倍拉长（TTI，"Thinking vs Doing"），最终都单调涨到一个人为设定的上限 K_max。这些方法隐含一个假设——**交互预算越多，agent 一定越强**。

本文用实验质疑这个假设。在 AppWorld 和 BFCL 上做固定预算扫描，发现成功率对预算的响应是**先涨后平**：预算低于某个任务相关的阈值 H*（under-capacity）时涨得很快，一旦越过 H* 就进入平台期，再多给预算也不见系统性提升，而算力成本随步数线性增长——这就是"**有效交互边界**"（effective interaction frontier）。甚至在 BFCL 上固定长预算（K=50）比短预算（K=15）更差且训练不稳定。于是问题变成：**能不能自动探测到 H*，并在训练中一直贴着它走？**

## 2. 核心方法 / 思路

Elastic Horizon 是一个**闭环控制器**：不按训练步数开环推进，而是拿 agent "实际表现出来的能力"反馈调节下一轮 rollout 的交互预算。整体训练流程四步循环：①在预算 K_t 下 rollout 一批轨迹；②正常做一次 GRPO 更新；③把**成功轨迹的长度**存入 FIFO 缓冲 B（容量 N=100）；④由控制器算出下一轮预算 K_{t+1} 反馈回 rollout。

控制器核心是**能力边界估计**：

- **信号选型**：用成功轨迹长度的 **90 分位数 P90(B)** 作为能力边界估计。为什么不用均值？成功轨迹长度呈右偏（heavy-tailed），均值（AppWorld 训练 100 步时约 18）严重低估"接近最大能力的解题长度"；为什么不用 max？个别低效乱逛的成功轨迹会把预算撑爆（max=50）。P90 兼顾"够到近乎全部可达任务"与"滤掉顶部 10% 离群噪声"，且对重尾分布是稳健的高分位估计。
- **余量与平滑**：目标预算 K_tgt = clip(P90 + Δ, K_min, K_max)，Δ=10 留出"最近发展区"式的探索余量，防止过早收敛到次优边界；再对 K 做 EMA 平滑（α=0.1），压低采样方差带来的抖动。
- **双向收敛**：能力涨 → P90 上移 → 预算扩张；接近任务复杂度天花板 → P90 停滞 → 预算自动收缩。因此从欠配（K₀=10）和过配（K₀=50）两个极端初始化都能落进平台区——这是只能单调增加的 open-loop 调度做不到的。
- **冷启动**：缓冲里成功轨迹不足 N_min=20 时保持当前预算不动，避免早期成功样本太少时 P90 不可靠。

控制器的计算开销是 O(N)/步，论文称 <0.1%，几乎免费。它与推理期预算分配方法（如 BATS）正交：一个是训练效率，一个是部署性能。定位上也可看作"以轨迹长度为学习信号的 self-paced curriculum"。

## 3. 关键实验结果

- **饱和确实存在（RQ1）**：AppWorld 7B 固定预算扫描，成功率从 K=10 的约 17% 一路爬到 K=30 的约 60%，之后 K∈[30,60] 的 2 倍预算区间收敛到同一平台带；BFCL 平台带更早，K∈[20,50]。任务越复杂（AppWorld 跨 App 编排）H* 越高——说明 H* 是任务相关的、静态 K 天然两边都吃亏。
- **能自动发现边界（RQ2）**：从 K₀=10 与 K₀=50 出发，控制器分别扩张/收缩并都稳定在平台带内（AppWorld 收敛后的预算约 28 附近），两个初始化都不触到 K_max。
- **全面最优（RQ3，mean@8）**：7B 上 EH 在 AppWorld 39.47%（ScalingInter 36.18 / TTI 37.28 / GRPO-K50 31.14 / GRPO-K15 21.49），BFCL 71.62%；14B 上 AppWorld **77.85%（best@8 90.42%）**、BFCL 73.62%（best@8 80.49%），均压过所有基线（ScalingInter 67.54/70.17、TTI 60.08/70.54、GRPO-K50 65.78/64.12）。14B 训练的 7B 模型还能在 AppWorld 上超过零样本的 Qwen2.5-32B（39.06）与 Qwen3-235B-A22B（33.85）。
- **样本效率**：AppWorld 14B 上，EH 在训练中段（约 step 140）比 GRPO-K50 每步少花 **25%** token，收敛后仍少 11%；AppWorld 7B 在成功率打到 50% 时累计 token 36.28M vs GRPO-K50 的 50.21M，**省约 28%**。
- **消融支撑设计**：P90 优于 P50（24.12）/P95（38.38）/Max（36.0）等统计量；headroom Δ=0 会把预算塌缩到 6.8、模型停在只会解简单任务的次优解；EMA α=0.1 最稳，α=1.0 直接跟原始 P90 会震荡。

## 4. 亮点与贡献（Why it matters）

1. **实证挑战主流假设**：用两套 benchmark 的固定预算扫描，把"越长越好"的直觉打回原形，明确提出有效交互边界与饱和平台现象。
2. **第一次把 horizon 变成闭环控制的量**：从"设计一个拉长的 schedule"转向"测量该停在哪"，K_max、增长率这些人工超参被一个可测信号取代。
3. **信号既便宜又稳健**：P90 + EMA 用纯观测（成功轨迹长度）就能追踪能力，不需要逐步奖励/价值函数/提示，任何稀疏二元奖励的 agentic RL 都能直接套。
4. **双向自适应 + 显著省钱**：能从上往下收缩是核心差异；最高省 25% 每步 token、累计省约 28% 且性能不降反升，"Green AI"叙事扎实。
5. **工程侵入极小**：只在 rollout 循环外包一层控制器，RL 更新路径完全不动，代码与训练配置已开源，易于复现与移植。

## 5. 局限与可改进点（个人点评）

- **P90/Δ/α 仍是启发式**：论文坦承没有一阶原理推导，收敛预算会落在平台带内但精确位置受 α、Δ 影响；EMA 稳定性分析只对平稳目标成立，而训练中目标本身在漂移——这本质是个"调得好的启发式"，不是有收敛保证的算法。
- **冷启动是真短板**：全靠成功轨迹长度估计能力，若早期成功率极低（AppWorld 7B 零样本只有 2.08%），缓冲填不满就得一直保守持稳，等于把"难任务要先熬过黑暗期"的成本转嫁给了固定预算——论文只提"保持预算不动"一种保守缓解。
- **单一全局预算**：所有任务共享一个 K，对难度方差大的任务分布是次优的——简单任务白给长预算、难任务仍被截断；per-task / 按难度分桶的预算分配是明显的下一步，论文自己也列了。
- **评测宽度有限**：只做 AppWorld/BFCL 两个 API/函数调用型环境、7B/14B 两种规模、200 步训练；没测视觉 GUI/网页环境（截图开销大、每步成本高，正是它最该省钱的地方）；token 只是算力代理指标，没算环境延迟与并行采样效率。
- **与 GRPO 的关系存疑**：预算改变会改动组内轨迹长度分布，进而影响组相对 advantage——论文没分析这一点，只把 KL 置零当作干净的 GRPO。

## 6. 对我们的启示 / 可借鉴点

对我们做 agent RL 最直接的价值是：**"每集交互预算"不该是拍脑袋的常数或单调递增曲线，而是一个可以闭环测量、自动跟随的变量**。实现的工程代价几乎为零——在我们的 rollout 循环外加一个"成功长度 FIFO + P90 + EMA"的小模块即可，且与 GRPO/DAPO、经验回放、奖励改造等任何方法正交。其次是它对"先小后大 curriculum"的修正：不是所有任务都值得把预算涨到满，贴住平台带既能保质量又能省钱；若我们的训练出现"长预算反而训练不稳"的现象（论文里 GRPO-K50 在 BFCL 直接标注不稳定），第一时间就该怀疑越过了 H*，而不是去加熵正则硬扛。

**对 GUI Agent 的可借鉴点：** GUI 环境是最适合这套思路的场景之一，因为它的每次交互都贵（截图渲染、VLM 前向、页面加载），且任务复杂度方差极大——"改个用户名"约 3~5 步，"跨 App 下单支付"可能要 20+ 步，固定步数上限必然一端浪费、一端截断。落地方式：在 WebArena/OSWorld/AndroidWorld 这类环境的 RL 训练里维护"成功轨迹长度 FIFO"，用 P90(+余量 Δ) 自动设定下一轮的每集最大步数，让它随 agent 能力从低到高双向收敛；它只依赖"成功/失败 + 步数"，与 GUI 常见的稀疏二元奖励完美兼容，不需要逐步奖励或动作级 hint，比靠 planner 拆子目标的路子更省事。特别提醒两点 GUI 侧的坑：一是**冷启动**——GUI RL 早期成功轨迹往往极少（与 AppWorld 7B 零样本 2% 同构），可先用 SFT/示范 warm-up 或像本文一样保持预算不动撑过攒样本期；二是**不要把全部任务绑一个全局预算**——GUI 任务池难度方差大，更推荐按任务难度分桶各配一个 K（对应论文"per-task horizon"的展望），或把预算与"最大可接受失败步数"挂钩防止 agent 在卡死任务上无限空转。

## 7. 延伸阅读

- **ScalingInter-RL / AgentGym-RL**（Xi et al., 2025）：线性拉长 horizon 的开环 curriculum，本文的直接对照对象。
- **TTI（Thinking vs Doing）**（Shen et al., 2025, NeurIPS）：乘法 schedule 拉长 test-time interaction，是"越给越多"假设的代表。
- **BATS**（Liu et al., 2025）：推理期注入剩余预算、按预算调规划的测试时方法，与本文训练期互补，可叠加使用。
- **WebRL / WebGym / Endless Terminals**：GUI 与终端 agent 的 RL 环境与 curriculum 训练工作，是 Elastic Horizon 迁移到 GUI 的直接落点。
- **Self-paced / Curriculum Learning**（Bengio et al., 2009; Kumar et al., 2010）：以学习者状态决定难度的思想源头。
- **ArCHer**（Zhou et al., 2024）与 **RAGEN**（Wang et al., 2025）：多轮 agentic RL 的稳定性与 credit assignment 分析，可解释"长预算为何不稳"。

---
*解读生成时间：2026-09-09 ｜ 解读人：WorkBuddy（AI）*
