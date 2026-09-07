# TIGPO: Temporal Instance-Graph Policy Optimization for Long-Horizon LLM Agents（时序实例图策略优化：让图式信用分配跨越策略更新）

> TL;DR：本文提出 TIGPO，通过为每个任务维护跨策略更新"持久化转移图"、并以"探索–重访"调度主动让当前策略回到旧任务，再以"跨时间对比"放大 advantage 参考集合，使历史经验只影响信用分配、绝不进入策略 loss，从而在不增加每轮 rollout 预算的前提下显著提升长程 LLM agent 在 ALFWorld 与 WebShop 上的表现。

---

## 论文档案（Metadata）

| 字段 | 内容 |
| --- | --- |
| 论文标题 | TIGPO: Temporal Instance-Graph Policy Optimization for Long-Horizon LLM Agents |
| arXiv ID / DOI | arXiv:2609.03383 |
| arXiv 链接 | https://arxiv.org/abs/2609.03383 |
| 发表出处（Venue） | 预印本（投稿评审中，首页标注 Under review as a conference paper at ICLR 2027） |
| 发布时间 | 2026-09-03 |
| 作者 | Jinwei Gan（单作者） |
| 所属机构（含国家） | 南京大学 计算机科学与技术系（Department of Computer Science, Nanjing University，中国） |
| 开源情况 / 代码 | ❌ 未在文中声明开源 |
| 类型标签 | RL、Online、Web、General |
| 训练方法标签 | Online RL（图式 / 组式 credit assignment：基于 relative advantage + clipped policy objective，非标准 PPO/GRPO 的自定义联合目标） |
| 关键词 | 持久化转移图（persistent transition graph）、跨更新信用分配、图式策略优化、Exploration–Revisit 采样、长程 LLM Agent、信用分配（credit assignment） |
| 来源渠道 | arxiv-api |
| PDF 存档 | 2026-09-03_TIGPO_Temporal_Instance-Graph_Policy_Optimization_for_Long-Horizon_LLM_Agents.pdf |

---

## 1. 研究背景与要解决的问题

长程 LLM agent 需要在多步交互中做一连串相互依赖的决策，奖励通常稀疏且延迟——往往整条轨迹结束后才知道成败。因此"把功劳分给正确动作"（credit assignment）是 LLM agent 强化学习的核心难题。PPO 靠学 critic 估计优势；GRPO 去 critic、在组内做奖励归一化得到相对优势；GiGPO 引入 episode 级与 anchor-state 级分组；GraphGPO 把同批 rollout 聚合为状态转移图，按中间状态到成功目标的最短路径距离给出 step 级信用。

这些方法的共同缺陷是 **batch-local（批次局部性）**：每次策略更新只用本批轨迹建图，用后即弃。早期策略发现的优质转移片段与后期策略发现的成功收尾即使属于同一任务，只要不在同一更新批次就无法在图里连成完整通路，尤其在 rollout 组很小时，图覆盖稀疏、相对优势对单条成败轨迹极度敏感。一个直观解法是直接回放历史轨迹，但旧策略产生的动作与 log-prob 早已过时，直接进 loss 会带来 off-policy 分布失配。由此引出本文核心问题：**图式信用分配能否跨策略更新累积经验，同时保持只优化当前策略的轨迹？**

## 2. 核心方法 / 思路

TIGPO 围绕三个互补机制回答上述问题：

**(a) 跨更新持久化实例图（Temporal Instance Graph）。** 每个任务实例 x 通过稳定身份 κ(x)（任务目标 + 初始环境配置）登记一张持久化转移图 H_x。更新时把当前 rollout 构成的图 G_x 与历史图取并集，得到时序图 Ḡ_x = H_x ∪ G_x；随后在 Ḡ_x 上重算到成功节点的最短路径距离，进而给每个"当前"转移计算时序图信用 q = C·γ^{d(s', g)}，再按同源状态归一化得到 step 级时序图优势。新发现的有效转移事后并入 H_x。关键约束：历史边只作为**结构参考**改变当前转移的信用，历史 token/动作/log-prob 一律不进策略 loss。

**(b) 探索–重访调度（Exploration–Revisit Sampling）。** 光存图不保证策略会再碰到同一任务。TIGPO 把每轮固定 B 个任务组分为探索组（照常采样新任务）与重访组（在当前策略下延迟 Δ 次更新后重试之前的任务），B_E + B_R = B，**总 rollout 预算 B·K 不变**；若合格可重访任务不足，余量自动转回探索。

**(c) 跨时间对比优化（Cross-Temporal Policy Optimization）。** 稀疏奖励下仅凭小组内归一化不稳定（一条成功轨迹可能主导参考）。对每个重访组，TIGPO 把它当轮的 episode 得分与"对应的早期探索组"的得分拼接（[Z_E; Z_R]），以早期分数作为**冻结参考**做归一化，得到跨时间 episode 优势，相当于与同一任务的早期策略直接比较。最终优势为 Â_TIGPO = λ_step·Â_TG + λ_ep·Â_CT（重访组）或 Â_ep（探索组），套用 clip 目标；expectation 只遍历当轮产生的轨迹。历史对 loss 的影响仅经由图连通性与参考分布两条"旁路"。

## 3. 关键实验结果

实验以 Qwen2.5-1.5B-Instruct 为基座，训练 250 个策略更新，每轮 64 条轨迹，评估取 3 个种子各 512 个 episode 的均值与方差；基线与 GraphGPO 在同等预算下复现。主要结果（Table 1）：

- **ALFWorld**：TIGPO 综合成功率 **91.28%**，超 GiGPO（91.02%）0.26 个百分点、超 GraphGPO（89.32%）1.96 个百分点；在六个子任务中拿最优三项（Pick 98.01 / Look 89.63 / Pick2 83.43），其中 Pick2 比 GraphGPO（74.13）高出 9.30 个百分点。相较而言，提示型方法（ReAct 12.8%、Reflexion 21.8%）与小型 RL 基线（PPO 54.4%、RLOO 69.7%、GRPO 77.9%）被大幅甩开。
- **WebShop**：TIGPO 平均任务得分 **88.65**、成功率 **77.54%**，成功率较 GiGPO（73.83）提升 3.71、较 GraphGPO（76.37）提升 1.17 个百分点。
- **消融（ALFWorld 综合成功率）**：当前更新图基线 89.32% → +持久化图 90.69% → +探索–重访调度 89.58%（单独加反而回退，因预算被摊薄、组内优势变弱）→ 完整 TIGPO（+跨时间对比）**91.28%**。跨时间对比一项带来 +1.70 个百分点，证明它是让重访生效的关键“补丁”。
- **诊断与开销**：TIGPO 与持久化图变体积累的任务图数量相当，但 TIGPO 的 task-memory / exact-state 命中率显著更高，说明调度确实让当前轨迹"撞上"有用历史。训练单步耗时 247.08s vs GraphGPO 258.32s（-4.35%），显存 84.31 vs 84.36 GB——作者谨慎指出 IQR 区间重叠，判定为**几乎零额外开销**。

## 4. 亮点与贡献（Why it matters）

- **首次把图式信用分配做成“跨更新持久化”**。它让不同策略版本发现的前缀与后缀能拼成完整成功路径，直接回应当前 batch-local 瓶颈。
- **干净的 on-policy 设计立场**：历史经验只以"图连通结构 + 冻结统计量"两种形式存在，从不作为优化样本进入 loss，绕开了经验回放的 off-policy 失配问题，设计上有很强的原则性。
- **调度器把“被动记忆”变成“主动存取”**：Exploration–Revisit 在**不增加预算**的前提下保证当前策略真的回到旧任务，诊断实验（命中率）也给出机制层面的实证。
- **低成本高回报**：不新增价值网络、不改生成流程，只在 advantage 计算侧加结构与统计信息，训练时间与显存几乎无变化——这对环境交互昂贵的长程 agent 训练尤为实际。

## 5. 局限与可改进点（个人点评，原创判断）

- **验证规模偏窄**：单作者、1.5B 基座、仅 ALFWorld（文本具身）与 WebShop（文本 Web）两个环境；主流 web/GUI agent 基准（如 WebArena、OSWorld、Mind2Web）完全缺席，泛化性存疑。
- **状态等价假设是软肋**：图法依赖"相同状态可合并、单一成功节点、最短路距离有意义"。真实 GUI 状态是半马尔可夫的 DOM/像素流，没有天然的状态指纹，若不解决状态抽象，该方法会水土不服。文中对 κ(x) 仅一句"环境配置相同"，未谈复杂状态空间如何判同。
- **超参敏感且较"手工"**：探索/重访比例固定为 8:8、重访延迟固定 10 步、λ_step=λ_ep=1、折扣因子两个环境各异（0.10 vs 0.20）。消融里“只加调度不加跨时对比”就掉点（89.58 vs 89.32），说明组件间依赖与超参调法较脆。
- **并非全面碾压**：ALFWorld 的 Heat 类 TIGPO（90.50）明显低于 GraphGPO（100.0）与 GiGPO（98.41）；"综合最优"掩盖了单类回退，值得追问是否与图合并的奖励/成功定义有关。
- **长程扩展隐忧**：每任务持久化图随训练单调增长（图 2 显示存储实例持续攀升），训练越长内存检索越重；自适应图压缩、去重与检索仍是开放问题。

## 6. 对 GUI Agent 的可借鉴点

尽管本文标为 gui 域，本质是通用 agent RL 方法论，其设计对 Web/GUI agent 的 RL 训练有直接迁移价值：

- **"历史拼接通路"缓解长程稀疏奖励**：GUI 任务（填表、多页下单、跨 App 操作）同样面临"早期策略找到有用中间步骤、后期才找到收尾"的问题。为每个任务/会话维护持久化页面流转图，可让旧版本的中间页与新版成功的后续点击跨更新连通，为当前动作提供密度更高的 step 级信用。
- **落地 GUI 的关键适配是状态指纹**：需定义可判等的状态表示（DOM 归一化、accessibility tree 结构化哈希、截图 embedding 相似度），才能 merge 节点、判定"回到同一页面"，这是整套图机制能否移植的前提。
- **Exploration–Revisit 思想适合昂贵 GUI 采样**：真实 GUI 环境采样成本高、在线任务分布杂，与其被动等相同任务再现，不如在固定预算内预留若干重访位，对新策略延迟重试历史失败/未完成的任务流；对线上用户请求亦可做成"困难任务重访缓存"。
- **"只借信用、不借梯度"可规避 GUI 奖励噪声**：GUI reward 噪声大、每组轨迹少，用历史分数做跨时间对比比回放更稳，也不破坏 on-policy 约束，几乎零额外开销，可叠加到现有 GRPO 系 Web agent 流水线。
- **省成本是硬卖点**：GUI agent RL 的最大瓶颈之一是环境交互昂贵；TIGPO 证明"吃历史红利"可以不增采样预算，这一取向对 GUI 域价值突出。

## 7. 延伸阅读

- **GraphGPO**（Cheng et al., 2026）：本文的直接前作与基线，提出 batch 内状态转移图 step 级信用分配；arXiv:2605.26684。
- **GiGPO**（Feng et al., 2025）：episode 级 + anchor-state 级组内相对优势；arXiv:2505.10978。
- **GRPO**（Shao et al., 2024）：无 critic 的组内相对优势，本文的组式基础；arXiv:2402.03300。
- **ReAct**（Yao et al., 2023）：思考-行动交替的 agent 范式；arXiv:2210.03629。
- **ALFWorld**（Shridhar et al., 2021）与 **WebShop**（Yao et al., 2022）：本文两个评测环境；arXiv:2010.03768 / arXiv:2207.01206。
- **RLOO**（Ahmadian et al., 2024）：leave-one-out 基线的 REINFORCE 式方法，作为 RL 基线；arXiv:2402.14740。

---

*解读生成时间：2026-09-07 ｜ 解读人：WorkBuddy（AI）*
