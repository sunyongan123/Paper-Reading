# Headroom-Drift Replay: A Primitive for Principled Replay Control in GRPO

> TL;DR：把旧轨迹复用拆成 Headroom（学习价值排序）+ Drift（策略漂移门控）两步的 GRPO 组级回放原语，不新增任何生成即提质降本。

---
## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | Headroom-Drift Replay: A Primitive for Principled Replay Control in GRPO |
| **arXiv ID / DOI** | arXiv:2609.03941v1（cs.LG） |
| **arXiv 链接** | https://arxiv.org/abs/2609.03941（点击直达） |
| **发表出处（Venue）** | COLM 2026（Conference on Language Modeling；页眉标注 "Published as a conference paper at COLM 2026"，同时以 arXiv 预印本发布） |
| **发布时间** | 2026-09-03 |
| **作者** | Hyun Bin Park、Du-Seong Chang（第一作者：Hyun Bin Park） |
| **所属机构** | 韩国西江大学（Sogang University）人工智能系，首尔（按 PDF 首页署名；致谢另列 NIPA/MSIT、IITP 等韩国国家级资助项目） |
| **开源情况 / 代码** | 未在文中声明开源（正文与附录均未提供代码仓库或模型链接） |
| **类型标签（论文类别）** | `RL` `Online` |
| **训练方法标签** | `RL (GRPO)`（组级回放增广的在线 RL 后训练；GRPO 主更新式不变） |
| **关键词** | Replay control; GRPO; Off-policy reuse; Sample efficiency; Policy drift; Agentic RL |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-03_Headroom-Drift_Replay_A_Primitive_for_Principled_Replay_Control_in_GRPO.pdf |

---
## 1. 研究背景与要解决的问题

基于 RL 的推理模型后训练越来越受"反复生成全新 rollout"的成本所困，在 agentic 场景尤甚——一次 rollout 要等模型与环境（检索、工具）多轮真实交互，墙钟时间是主导开销。回放（replay）通过复用旧轨迹能省下这笔钱，但朴素重放往往学不稳。作者指出，一条存起来的轨迹组可能因两种截然不同的原因而失效：要么仍有学习信号、却已落后于当前策略（stale）；要么贴近当前策略、但已没什么可学的了。因此可靠的回放需要两个独立判断：**还值不值得学**，与**现在还合不合身**。

已有工作（RePO、EFRame、ExGRPO、BAPO 等）把回放嵌在探索、过滤、混合策略优化等更大流水线里，回放自身的贡献难以被隔离研究。本文于是问一个聚焦问题：单靠"有原则的回放选择"本身，能走多远？为此作者把回放抽象成 GRPO 上一个可独立开关的最小干预，隔离研究回放侧控制对训练动力学的影响。

## 2. 核心方法 / 思路

总体设计一句话：**新策略的 on-policy 流一行不改，唯一改动是把按规则筛出的旧组并入混合 batch，让一切训练差异都追溯到回放侧控制。** 复用最小单位是"整组"而非单条回答，从而保留 GRPO 最核心的组内相对优势比较结构。

方法把回放拆成两个正交控制轴：

- **Headroom（学习价值排序）。** 以生成时策略为参照，看一条存储组"还剩多少方向性修正空间"：对优势 A_i>0 的回答里的 token，记 1−π(当前概率)；对 A_i<0 的记 π——即正向 token 还能往上抬多少、负向 token 还能往下压多少，再对组内全部 token 取平均。Headroom 越高，说明这组轨迹越"有得学"，排序靠前。
- **Policy Drift（当前策略兼容门控）。** 计算当前策略与生成该组的参考策略 π_gen(g) 在存储动作上的逐 token log 概率平方偏差并组内平均，超过阈值 τ 的组不放行。用 L2 平方而非带符号平均，避免正负偏移相互抵消；且门控还带来形式化红利：通过门控的组，其当前 Headroom 相对存储排序最多偏差 √τ（Proposition 2）。

单步流程（Algorithm 1）五阶段：A. 照常生成新鲜组并计算优势，把"组内结果有混合（非全对非全错）"的组标记为回放入口候选——全同奖励组的组内优势为零，无学习信号；B. 将缓冲区内旧组按 Headroom 降序排列；C. 依序对候选做 teacher-forced 前向重估（固定序列，无需自回归生成与真实环境交互），漂移不超 τ 则接纳，直到回放预算 K_rep 填满；D. 接纳组与新鲜组合成混合 batch 跑标准 GRPO 更新；E. 更新后才把本步入口候选追加进 FIFO 缓冲，杜绝同一步自回放。同一前向 pass 算出的概率既用于门控、又顺手刷新 Headroom 缓存。

## 3. 关键实验结果

三个奖励可验证、固定奖励的 GRPO 域。任务：数学用 AIME24/AMC23/MATH500/Minerva/OlympiadBench；Agentic Search（Search-R1 风格检索环境）用 NQ/TriviaQA/PopQA/HotpotQA/2WikiMultiHopQA/Musique/Bamboogle；多模态用 Geometry3K/MathVista/MathVision。模型：数学与多模态为 Qwen2.5-Math-1.5B，agentic 为 Qwen2.5-3B/7B-Instruct（8×H100）。头号指标 Mean@32（每输入 32 个样本取均值），并配每步墙钟成本（Table 1）。基线按"角色"分组：on-policy matched/larger、GRPO+replay matched/larger（纯近因回放）、DAPO、ExGRPO、BAPO。关键数字均核对正文与附录：

- **数学（基线最全）**：Headroom-Drift 的 Avg Mean@32 = 0.3533，超过 BAPO（0.3469）、DAPO（0.3409）、ExGRPO（0.3139）与两个 on-policy 基线（0.3356/0.3344），是唯一全面压线所有基线的域；且新鲜样本数 1,843,200 与 matched 持平、明显少于 on-policy larger 的 2,764,800，排除"纯靠更多新数据"的解释。
- **Agentic Search（Table 1 + 主文）**：Avg Mean@32 = 0.3577，仅比 naive replay matched（0.3548）高 0.0029（在噪声范围内），但 Best@32 = 0.4879 vs 0.4623 差距明显；相对 on-policy larger 则是严格帕累托改进——0.3577 @ 166.3s/步，胜过其 0.3212 @ 197.2s/步（后者多花 1.5 倍新鲜预算）。每步耗时排序：matched 146.4s < replay matched 154.4s < Headroom-Drift 166.3s < replay larger 187.3s < on-policy larger 197.2s < DAPO 232.5s。7B 规模复核：Avg Mean@4 由 0.3737 升至 0.3955、Weighted Mean@4 由 0.4158 升至 0.4298，7 项基准赢 6 项。
- **多模态（简化 3 基准）**：Avg Mean@32 = 0.4137，高于 matched 0.3986、larger 0.4005、DAPO 0.4056。
- **机制与消融**：终身复用以存储 Headroom=0.8 为分水岭（低于 0.8 的组均回放 <0.5 次、概率 <24%；高于的约 2.0 次、>56%）；来源年龄上 naive 100% 来自上一 步，Headroom-Drift 摊到 1–7 步且 44% 来自三步以前——门控下的"老高价值组"构成时间上多样化的纠偏信号。MATH-500 消融（Mean@32）：完整版 0.7215 > 仅 Headroom 0.7140 > 仅 Drift 0.6794。τ 只需对数间隔扫描（接受率 6.4%/40.2%/94.4% 三态即判可行域），单轮任务取 1e-3、Search-R1 取 1e-2 且 3B→7B 直接复用。按训练分数对齐后，回放组比 on-policy larger 更晚进入低熵区（237 vs 207）且停留更久（261 vs 180）。

## 4. 亮点与贡献（Why it matters）

1. **把回放重构成可独立评估的最小原语**：两条控制轴（剩余学习价值、当前策略兼容性）被显式分离并各自可消融，回放第一次作为"单独的控制问题"被研究——作者称可与 EFRame/ExGRPO/BAPO 等回放中心方法组合成可插拔控制层。
2. **零额外生成、零新增训练机制**：回放选择只靠 teacher-forced 前向计算，新鲜 on-policy 流与 GRPO 更新式完全不动，归因干净。
3. **Agentic 场景给出墙钟意义上的帕累托改进**：用"重估旧轨迹"置换"昂贵的新环境交互"，质量不降反升——这正是回放价值最稀缺的用例。
4. **回放与熵坍缩的关系提供了新观察**：按分数对齐后回放延迟熵坍缩，暗示有原则的复用本身是一种稳定化机制。
5. **工程上可落地**：τ 是粗校准而非细调参数、跨规模复用；对多奖励（Geo3K 中格式分污染正确性）的入口坍缩分析也具普适警示意义。

## 5. 局限与可改进点（个人点评）

- **最强卖点发生在最弱的统计对比上**：agentic 主设定里相对 naive replay 的 Avg Mean@32 只差 0.0029，作者自己也承认在噪声范围内；并且 naive replay matched 每步耗时 154.4s 其实比 166.3s 更便宜——"省时"只对 on-policy scaling 成立，对朴素回放并不成立。
- **规模与广度有限**：主实验 1.5B（数学）与 3B/7B（agentic），在 30B+ 级推理模型上，回放代价画像（生成贵 vs 环境贵）会变，结论未必平移；7B 复核还明确是"每方法单次运行"，无方差报告，0.0029 这类差值需更多重复。
- **入口规则偏窄**：只收"组内结果混合"的组，对奖励稀疏的长程 agent 任务（全组失败常见）缓冲会很快饿死；论文的多奖励分析也只在"如何定义成功"上打补丁，未解决稀疏奖励下无混合组可入的问题。
- **门控只测策略漂移、不测环境漂移**：对于环境本身在变的任务，旧轨迹的失效原因与策略无关，L2 门控对此无感（第 6 节展开）。
- **τ 仍手工按任务族设定**，作者把自适应门控（放宽/收紧）列为 future work；L1 vs L2 的取舍只有附录定性说明，缺系统的敏感性分析。

## 6. 对 GUI Agent 的可借鉴点（跨域参考专节）

GUI Agent 的 RL 训练与本文痛点高度同构：每回合要真实驱动浏览器/桌面、截图+辅助树解析，环境交互慢、并行受限，反复采新 rollout 极贵，而"旧轨迹能不能复用"没人系统回答。本文方案可逐条迁移：

- **"组"的切法**：GUI 训练里可用"同一任务/同一起始状态采样出的多条动作轨迹"作为组，组内相对优势（哪条轨迹更接近成功）天然成立；Headroom 的 token 级定义可直接套到动作 token 上——成功轨迹里概率仍低的动作值得抬、失败轨迹里高概率动作值得压，语义不变。
- **Drift 门控必须扩展为"双漂移"**：GUI 策略更新快、且界面本身在改版。建议在策略 log-prob 漂移外，叠加"屏幕/无障碍树相似度"作环境侧门控，把过期截图对应的整批轨迹拒之门外，否则 L2 门控会放过"策略没变但 UI 已变"的陈旧数据。
- **入口规则要按 GUI 稀疏奖励改造**：多步任务大多整条失败，"混合结果组"极少。可参照 Geo3K 教训先拆奖励源（是否点击到有效元素、是否达成子目标、是否最终成功），把"部分成功"定义成可用的混合信号，否则回放缓冲会在训练早期就干涸。
- **省时逻辑直接成立**：回放组只需在存储序列上做 teacher-forced 前向，GPU 并行即可，不占浏览器并发——本文"重估换交互"的帕累托叙事正是 GUI 训练最需要的收益形态。
- **警惕风险**：GUI 环境非平稳性强，门控阈值的 6.4%/40%/94% 三态诊断法（回放 KL 失衡即出局）值得移植，作为上线前的快速体检工具。

## 7. 延伸阅读

- **GRPO / DeepSeekMath（Shao et al., 2024, arXiv:2402.03300）与 DeepSeek-R1（Guo et al., 2025, Nature）**：本文所增广的 RL 后训练基础。
- **RePO（Li et al., 2025, arXiv:2506.09340）**：把回放直接织入 GRPO 循环的直系前作。
- **EFRame（Wang et al., 2025, arXiv:2506.22200）、ExGRPO（Zhan et al., ICLR 2026, arXiv:2510.02245）、BAPO（Xi et al., ICLR 2026）**：回放+探索/过滤/缓冲中心优化的对照系，本文称其原语可嵌入其中。
- **Prioritized Experience Replay（Schaul et al., 2016, arXiv:1511.05952）与 RETRACE/IMPALA 线（Munos et al., 2016; Espeholt et al., 2018）**：Headroom 与 Drift 两个控制轴的经典 RL 出处。
- **DAPO（Yu et al., 2025）**：本文最强的非回放基线；**熵动力学相关（Wang et al., 2026b, arXiv:2602.03392）**：熵坍缩分析的参照。

---
*解读生成时间：2026-09-06 09:30 ｜ 解读人：WorkBuddy（AI）*
