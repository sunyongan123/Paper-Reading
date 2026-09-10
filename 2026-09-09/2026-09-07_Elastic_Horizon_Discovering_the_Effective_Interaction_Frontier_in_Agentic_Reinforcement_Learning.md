# Elastic Horizon: Discovering the Effective Interaction Frontier in Agentic Reinforcement Learning（弹性视界：在 Agentic RL 中发现有效交互前沿）

> 一句话 TL;DR：用「成功轨迹长度的 P90 + 余量 + EMA」把每集交互预算做成闭环控制器，自动追踪有效交互前沿 H*，7B/14B 双模型全最优且最多省 25% 每步 token。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Elastic Horizon: Discovering the Effective Interaction Frontier in Agentic Reinforcement Learning |
| **作者 / 机构** | Gangyi Zhang*、Junjie Meng*（*共同一作）、Letian Zhang、Wei Wu、Yang Zheng、Dong Wang†、Yang Liu、Guanjun Jiang、Chongming Gao†（9 人，†通讯）｜阿里云 Qwen 业务部（中国）、中国科学技术大学 USTC（中国）、独立研究者 |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-07；arXiv preprint（cs.AI） |
| **arXiv 链接** | https://arxiv.org/abs/2609.07247 |
| **代码仓库** | ✅ GitHub：https://github.com/junjie-meng/ElasticHorizon（含训练配置） |
| **数据集地址** | AppWorld（Trivedi et al., 2024）、BFCL v3（Patil et al., 2025），均为公开 benchmark/环境，未打包为单一下载地址 |
| **类型标签（论文类别）** | `RL` `Online` `General` |
| **训练方法标签** | `RL (GRPO)`（KL=0，基于 AgentEvolver 框架；创新点是训练期超参「每集交互预算」的闭环调度） |
| **关键词** | 有效交互前沿（effective interaction frontier）、horizon scheduling、agentic RL、闭环控制、成功轨迹长度、样本效率、GRPO |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-07_Elastic_Horizon_Discovering_the_Effective_Interaction_Frontier_in_Agentic_Reinforcement_Learning.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：agentic RL 里「每集最多与环境交互多少步」（interaction horizon / budget）这个超参该怎么定。场景是稀疏二元奖励的多轮工具调用/API 编排环境（AppWorld、BFCL）。
- **为什么重要**：horizon 定小了截断能力、定大了白烧算力且可能训练不稳（GRPO-K50 在 BFCL 上直接标注 ∗ 不稳定）。它卡住的是长程 agent 训练的样本效率与稳定性。
- **现有方法有什么不足**：主流是**开环 curriculum**——ScalingInter-RL 线性拉长（K_t = Kmin + δ·t）、TTI 乘法分段拉长（15→20→30→50），都单调涨到人为上限 K_max，隐含「预算越多一定越强」的假设。论文用固定预算扫描证伪：成功率随 K 先涨后平，越过任务相关阈值 H* 后进入平台带，再多预算无系统提升而算力随步数线性增长。
- **Research Gap**：作者主张「从『怎么把 horizon 拉长』转向『何时该停』」——需要一个能**自动探测并双向追踪 H*** 的闭环机制，替代靠训练步数驱动的单调 schedule。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法
不看训练步数，只看 agent「实际能解多长的题」，用成功轨迹长度的 P90 估计能力边界，反馈调节下一轮 rollout 的交互预算。

### 3.2 方法总览（Pipeline）
- **输入**：当前策略 π_θ 与任务分布；**输出**：下一轮 rollout 的预算 K_{t+1}。
- 四步闭环：①在预算 K_t 下采一批轨迹；②做一次 GRPO 更新；③把**成功轨迹的长度**存入 FIFO 缓冲 B（容量 N=100）；④控制器输出 K_{t+1} 反馈回 rollout。
- 控制器三阶段（Figure 2）：能力估计器 B̂_t = P90(B_t)；加余量 K^target_t = B̂_t + Δ（Δ=10）；EMA 平滑 K_{t+1} = (1−α)K_t + α·clip(K^target_t, K_min, K_max)（α=0.1）。

### 3.3 真正的创新点（去伪存真）
- **真·方法创新**：把「每集交互预算」从开环 schedule 变成由**成功轨迹长度分布**驱动的闭环控制器，且支持**双向收敛**（欠配 K₀=10 与过配 K₀=50 都能落进平台带）——这是只能单调递增的开环方法做不到的。
- **工程组合**：P90、EMA、FIFO、clip 都是现成组件；GRPO 更新路径完全未改（KL=0），控制器只是 rollout 循环外包一层。本身无新结构，但组合有效。
- **对性能提升最关键的设计**：**P90 分位数选型**（对比 P50 24.12 / P95 38.38 / Max 36.0，P90 39.47 最优）与**余量 Δ=10**（Δ=0 时预算塌缩到 6.8、SR 掉到 15.35）。二者共同决定收敛预算是否落在平台带内。
- **证据不足 / 仅声称有效**：论文明言 P90 无第一性推导，是「empirically motivated」；EMA 稳定性证明只对平稳目标成立，而训练中 H* 随策略漂移，作者自认「不是有收敛保证的算法」。定位应为「调得好的启发式」。

---

## 4. 具体技术细节

### 4.1 模型结构
- **base model**：Qwen2.5-7B 与 Qwen2.5-14B，纯 **LLM（无视觉编码器）**，输入是文本指令 + API 反馈，输出工具调用。
- 全参数微调（GRPO），无冻结/LoRA 说明。优化器 AdamW，lr=1e-6，batch size 16（micro-batch 8，grad accum 2），γ=1.0、GAE λ=1.0、clip 0.2，训练 200 步。

### 4.2 训练流程（elastic horizon 是重点）
本作是**单阶段 GRPO 训练**，核心是外加的 horizon 控制模块（非多阶段 SFT+RL）。GRPO 更新式沿用 Shao et al. (2024) 且 **KL 系数置 0**，与标准 GRPO 唯一区别是每步的采样预算由控制器给出。

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| 单阶段 RL（200 步） | 端到端多轮工具调用/API 编排 | 长程决策、跨 App 协调、函数调用 | AppWorld（9 App、457 API）与 BFCL v3（8 域、1000 多轮样本） | 轨迹 τ=(u,a₁,o₁,…,a_L,o_L)，稀疏二元奖励 r∈{0,1} | GRPO（KL=0），奖励为任务成功 1/失败 0 |

控制器超参：K₀=15、K_min=5、K_max=50、α=0.1、Δ=10、N=100、N_min=20（冷启动：|B|<20 时预算保持不动）。开销 O(N)/步，论文称 <0.1%。

### 4.3 推理流程
- 多步循环（ReAct 式 tool calling）：策略基于 (指令, 交互历史) 输出动作，接收环境观测与稀疏奖励；每集轨迹达当前预算 K 仍未成功则强制终止、记 0 分。
- 终止条件：任务成功（r=1）或触及 horizon 上限 K。**无独立的推理期机制**——本作是训练效率方法，与推理期预算分配（BATS）正交、可叠加。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| AppWorld | General（API 工具调用，非视觉） | 端到端跨 App 编排 | 9 个日常 App、457 个 API，标准 test split | 文本指令 + API 反馈 | API 调用序列 | 任务目标完成率（success rate） |
| BFCL v3 | General（函数调用） | 多轮有状态函数调用 | 8 个 API 域、multi-turn 1000 例 | 指令 + 函数文档 + 状态 | 函数调用 | 总体成功率 |

两 benchmark 均为稀疏二元奖励、轨迹上限 50 步。

### 5.2 实验结果分析

**主结果（mean@8 / best@8，%）**：

| 方法 | AppWorld 7B | BFCL 7B | AppWorld 14B | BFCL 14B |
|---|---|---|---|---|
| GRPO (K=15) | 21.49 / 42.40 | 66.87 / 71.62 | 56.57 / 71.16 | 68.80 / 74.13 |
| GRPO (K=50) | 31.14 / 49.20 | 60.37* / 68.32* | 65.78 / 80.63 | 64.12 / 73.61 |
| TTI | 37.28 / 46.14 | 55.75 / 71.22 | 60.08 / 77.73 | 70.54 / 76.35 |
| ScalingInter | 36.18 / 55.10 | 69.12 / 73.93 | 67.54 / 82.54 | 70.17 / 78.85 |
| **Elastic Horizon** | **39.47 / 56.87** | **71.62 / 80.45** | **77.85 / 90.42** | **73.62 / 80.49** |

- 饱和实证（RQ1）：AppWorld 7B 成功率从 K=10 的 ~17% 爬到 K=30 的 ~60%，K∈[30,60] 收敛同一平台带；BFCL 平台带更早（K∈[20,50]）。H* 任务相关：跨 App 编排的 AppWorld 高于函数调用的 BFCL。
- 边界自动发现（RQ2）：K₀=10 与 K₀=50 分别扩张/收缩，都稳定进平台带（AppWorld 收敛预算约 28），均不触 K_max。
- 训练后的 7B 模型在 AppWorld 上超过零样本 Qwen2.5-32B（39.06%）与 Qwen3-235B-A22B（33.85%）。
- **样本效率**：AppWorld 14B 中段（step ~140）每步 token 比 GRPO 省 25%，收敛后省 11%；7B 打到 SR=50% 累计 token 36.28M vs GRPO-K50 的 50.21M，省 28%。

**Ablation 说明**：①统计量——P90 全面最优（P50 24.12、P95 38.38、Max 36.0）；②余量——Δ=0 使预算塌到 6.8（SR 15.35），Δ=10 最优，但 SR 对 Δ 非单调、邻值差与种子方差相当；③EMA——α=0.1 稳（39.47），α=0.2 微振荡（30.92），α=1.0 直接跟原始 P90 不稳（24.78）。

---

## 6. 亮点与贡献（Why it matters）

1. **实证打掉「越长越好」的直觉**：两套 benchmark 的固定预算扫描首次系统刻画了饱和平台与 H* 的任务相关性。
2. **范式转换**：从「设计拉长 schedule」转向「测量该停在哪」，把 K_max、增长率两个需先验承诺的超参换成一次测量。
3. **信号便宜又通用**：纯观测（成功长度 P90）即可追踪能力，无需逐步奖励/价值函数/提示，任何稀疏二元奖励的 agentic RL 都能零成本套用。
4. **双向自适应 + 实打实省钱**：能从过配收缩是核心差异；最高省 25% 每步 token、累计省 28% 且性能不降反升，Green AI 叙事扎实。
5. **工程侵入极小**：只包一层 rollout 控制器，RL 更新路径不动，代码与配置已开源，易复现易迁移。

---

## 7. 局限与可改进点（个人点评）

- **本质仍是启发式**：P90/Δ/α 无理论保证，收敛预算的精确位置受 α、Δ 影响；EMA 稳定性分析假设目标平稳，而 H* 随策略漂移——这是「调得好的控制器」，不是收敛算法。
- **冷启动是真短板**：全靠成功轨迹估能力，AppWorld 7B 零样本仅 2.08%，早期缓冲填不满就长期保守持稳，等于把难任务的「黑暗期」成本转嫁给固定预算；缓解只有「保持预算不动」一种。
- **单一全局预算**：所有任务共享一个 K，难度方差大时简单任务白给长预算、难任务仍被截断，per-task/按难度分桶是明显下一步。
- **评测宽度有限**：只测 AppWorld/BFCL 两个 API/函数调用型环境、7B/14B、200 步；**没测视觉 GUI/网页环境**（截图开销大、正是最该省钱处）；token 只是算力代理，没算环境延迟与并行采样效率；14B 因算力只报单 seed，方差存疑。
- **与 GRPO 的交互未分析**：改预算会改变组内轨迹长度分布，进而影响组相对 advantage，论文未深究，只把 KL 置零当作干净 GRPO。

---

## 8. 对我们的启示 / 可借鉴点

最直接的价值判断：**「每集交互预算」不该是拍脑袋的常数或单调曲线，而是一个可以闭环测量、自动跟随的变量**。实现代价几乎为零——在 rollout 循环外加「成功长度 FIFO + P90 + EMA」小模块即可，且与 GRPO/DAPO、经验回放、奖励改造正交。其次是修正「先小后大 curriculum」：不是所有任务都值得把预算涨满，贴住平台带既保质量又省钱；若训练出现「长预算反而不稳」（本文 GRPO-K50 在 BFCL 标 ∗ 不稳定），第一反应应是怀疑越过了 H*，而非加熵正则硬扛。

**对 GUI Agent 的可借鉴点**：GUI 是这套思路最该落地的场景之一——每次交互都贵（截图渲染、VLM 前向、页面加载），且任务复杂度方差极大（改用户名 3~5 步 vs 跨 App 下单 20+ 步），固定步数上限必然一端浪费一端截断。落地：在 WebArena/OSWorld/AndroidWorld 的 RL 训练里维护成功轨迹长度 FIFO，用 P90(+Δ) 自动定下一轮每集最大步数，随能力双向收敛；它只依赖「成功/失败 + 步数」，与 GUI 稀疏二元奖励完美兼容。两个 GUI 侧要提防的坑：**冷启动**——GUI RL 早期成功轨迹极少（与 AppWorld 7B 零样本 2% 同构），建议用 SFT/示范 warm-up 或按论文保守持稳撑过攒样本期；**别绑单一全局预算**——GUI 任务池难度方差大，按难度分桶各配 K（对应论文 per-task horizon 展望），或把预算与「最大可接受失败步数」挂钩，防止 agent 在卡死任务上空转烧钱。

---

## 9. 延伸阅读

- **ScalingInter-RL / AgentGym-RL**（Xi et al., 2025）：线性拉长 horizon 的开环 curriculum，本文直接对照对象。
- **TTI（Thinking vs Doing）**（Shen et al., 2025, NeurIPS）：乘法 schedule 拉长 test-time interaction，代表「越给越多」假设。
- **BATS**（Liu et al., 2025, COLM 2026）：推理期注入剩余预算、按预算调规划的测试时方法，与本文训练期互补可叠加。
- **ArCHer**（Zhou et al., 2024）与 **RAGEN**（Wang et al., 2025）：多轮 agentic RL 的稳定性与 credit assignment 分析，可解释「长预算为何不稳」。
- **Self-paced / Curriculum Learning**（Bengio et al., 2009; Kumar et al., 2010）：以学习者状态定难度的思想源头。
- **WebRL / WebGym**（Qi et al., 2025）：GUI/网页 agent 的 RL 环境与在线 curriculum，是 EH 迁移到 GUI 的直接落点。

---
*解读生成时间：2026-09-10 ｜ 解读人：WorkBuddy（AI）*
