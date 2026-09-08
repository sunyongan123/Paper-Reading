# Persistent Teacher Anchoring for Tool-Using Agents

> 学生逐块提议、老师逐块校验，只有老师放行的整轮才能执行工具，防止轨迹越走越偏。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | Persistent Teacher Anchoring for Tool-Using Agents |
| **arXiv ID / DOI** | 2609.04773v1 [cs.LG] |
| **arXiv 链接** | https://arxiv.org/abs/2609.04773（点击直达） |
| **发表出处（Venue）** | arXiv preprint（预印本） |
| **发布时间** | 2026-09-04 |
| **作者** | Hyun Bin Park、Kyungho Song、Sangmin Lee、Du-Seong Chang（未标注通讯作者） |
| **所属机构** | 西江大学 Sogang University（韩国）、密歇根大学安娜堡分校 University of Michigan, Ann Arbor（美国） |
| **开源情况 / 代码** | 未在文中声明开源（正文注明基于 verl + sglang 复现，无代码链接） |
| **类型标签（论文类别）** | `Distillation` `RL` `General` |
| **训练方法标签** | `Distillation`（on-policy forward-KL，teacher top-256 稀疏 logit，K=3 验证）+ 下游 `RL`（PPO 框架 / GRPO 式优势估计） |
| **关键词** | On-policy 知识蒸馏；proposer-verifier 生成；工具调用智能体；pre-RL 对齐；持久化 lookahead 调度 |
| **来源渠道** | arxiv-api + 每日 listing |
| **PDF 存档** | 2026-09-04_Persistent_Teacher_Anchoring_for_Tool-Using_Agents.pdf |

---

## 1. 研究背景与要解决的问题

蒸馏是大模型后训练的常客，**on-policy knowledge distillation（OPKD）** 已成为主流：进入下游 RL 前，先让"学生"在**自己生成的轨迹**上匹配老师给的逐 token 分布，以塑造初始策略。问题在于学生 rollout 越长，越可能闯进老师不会访问的状态，teacher–student 分布差距沿轨迹**累积**；在工具使用场景代价被放大——学生自己写的工具调用会在老师信号到达前**真的执行出去**，返回的观测既非老师也非学生生成，却拼进前缀、左右后续所有生成。

文本侧的 **proposer-verifier 生成**家族（弱模型逐片段提议、强模型决定保留哪些文本）旨在对抗这种漂移：起于解码加速（speculative decoding、BiLD、RSD），后入训练阶段（SWITCH、SKD、SCoRe）。作者指出其家族性盲区：决策边界都在**文本流**内部，**未定义"一段文本何时可触发工具执行"**，直接套到工具运行时产生四个缺口：(i) 调用可能在老师验证完整个回合前就被执行；(ii) 调用一旦执行，事后修改会与已返回观测不一致；(iii) 环境回写的观测既不属于"保留提议"也不属于"老师替换"两个分支；(iv) 蒸馏目标没说观测 token 算不算模仿目标——一视同仁等于教学生复述环境。本文的 Persistent Teacher Anchoring（PTA）即补上"执行事件"这一维。

## 2. 核心方法 / 思路

PTA 是一种"**学生提议、由老师批准才落地**"（student-induced, teacher-committed）的 rollout 构造，名字三个词各有实指：

**块级验证（沿用 SKD）。** 学生每次提议一个 chunk；老师基于已验证前缀逐 token 判定——落在老师 top-K 支撑内就保留；遇到首个被拒 token 则用老师 top-1 替换，并**丢弃该 chunk 其后由已失效前缀生成的后缀**，学生再从修正前缀重新提议。验证过的 chunk 成为追加到"待定助手轮"的最小原子单位。

**整轮承诺（新增）。** 当待定轮到达 EOS 或工具调用结束标签，PTA 把**整轮** finalize 为"已承诺助手轮"并入轨迹，**只有此时**工具运行时才解析并执行调用——未经验证文本绝不作用到环境。环境观测作为独立 span 追加进前缀、供后续生成作上下文，但与 SKD 直套工具运行时"先执行后验证"形成对比。

**源感知目标。** 蒸馏 loss（逐位置 forward KL）**只施加在已承诺助手位置**；prompt 与观测 token 留作前缀、绝不是模仿目标。形式上只是改损失作用域，机制上防止学生"背环境"。

**持久锚定与 lookahead（调度创新）。** 学生参数每步更新而**老师验证器全程固定**。同步训练中慢样本拖住整批、先完成者释放的槽位闲置；因为已验证状态只暴露在"chunk 边界或完整环境转移之后"且验证器不变，跨梯度步携带状态是安全的：lookahead 用空闲槽提前推进未来样本——当前批前完成的晋升进本步更新，cutoff 时未完成的作为 **carryover** 暂停在原子边界，学生更新后仍由同一验证器把关新 chunk，PTA 承诺语义不被破坏。附录给出紧凑刻画：committed token 服从混合分布 π=(1−r)·p̄_θ + r·δ_{x⋆}，与纯学生策略的总变差距离恰等于被拒概率 r；K=3 时近 on-policy 端。所谓 persistent，指"老师决定保留哪些文本、哪些调用能执行"这一锚定作用跨越学生参数更新的时间边界持续生效。

## 3. 关键实验结果

两个工具环境：**检索**（Search-R1 风格：NQ/PopQA/HotpotQA/Musique；老师 Qwen3-32B、学生 Qwen3-1.7B）与**感知**（DeepEyes 风格 bbox 缩放工具：VStar/HRBench4K/HRBench8K；Qwen3-VL-32B→Qwen3-VL-2B）。四设定 Base / Direct RL / OPKD+RL / PTA+RL（EM、mean@4/best@4），**下游 RL 预算一致**。

- **检索**：PTA+RL 在 weighted best@4、macro mean/best@4 领先，macro best@4 **34.59** vs OPKD+RL 32.07（**+2.52**）；增益集中在多跳 Musique——mean@4 8.20→10.93、best@4 13.94→18.78，符合"早期未验证交互会改写后续检索路径"的假设。
- **感知**：PTA+RL 四项聚合全领跑，macro mean@4 **+5.32**（63.59 vs 58.27）、macro best@4 **+2.80**（70.00 vs 67.20）；最高分辨率 HRBench8K 增益最大（cross mean@4 43.75→51.50）。
- **pre-RL 检查点**：PTA 六个感知指标全超 OPKD，最大差距 HRBench8K best@4 58.00→65.00——优势在交给奖励优化的策略里已存在。
- **决定性消融**：只禁掉"解析到工具调用内部的重写"（其余机制全保留），性能跌破 PTA **与** OPKD 所有指标（H8 mean@4 57.88→48.13）；而工具调用内替换仅占全部 committed 助手 token 的 **0.05%（检索）/0.82%（感知）**，学生自提议保留率高达 90.5%/88.4%——主体仍是学生文本，是极少数落在"会作用到环境"内容上的修正决定了全部优势。
- **Top-N 稳定性**：PTA 在 N=256 就维持高 teacher retained mass，OPKD 小 N 快速塌陷、N=1024 仍有低覆盖位置；学生熵在 PTA 下更低更稳——稀疏 logit 预算下，老师已承诺前缀让老师分布更易保真。
- **Lookahead**：吞吐 0.519→0.644 samples/s（**+24%**），平均每步晋升 20.2 个样本（B=32）。
- **显著性**：PopQA、Musique、HRBench8K 的差异 p<0.01，HRBench4K p<0.1；每设定仅训练一次。

## 4. 亮点与贡献（Why it matters）

- 精确诊断 proposer-verifier 家族"决策只在文本流内、缺执行事件"的结构缺陷，补上 turn-level 承诺门，相当于增加"何时允许副作用落地"这第二决策维度。
- 观测的**源感知**处理（进上下文、不进 loss）是干净普适的规则，对任何带工具观测的蒸馏/SFT 管线都直接可用。
- 抓住"训练中老师固定、学生更新"的事实，把已验证 rollout 状态做成跨更新持久对象，白捡 24% 吞吐并给出承诺语义不破的证明。
- 评测协议值得学：蒸馏好坏不看 loss，而看"**固定下游 RL 预算下最终策略跑多高**"，并与 pre-RL checkpoint 交叉验证，因果链完整。
- 0.05%/0.82% 的极小修正量决定胜负，定量揭示"安全关键位置"的杠杆远大于泛泛的 token 覆盖。

## 5. 局限与可改进点（个人点评）

- **覆盖有限且作者自认。** 只含检索与感知两类工具（观测为文本/裁图），代码执行、数据库、多智能体未测；GUI 场景尤其不同——观测是持续可交互的界面状态，动作副作用不可回滚，结论需重新验证。
- **验证器成本始终在。** 每 chunk 一次老师前向（训练前期每样本近 109 个 chunk）；lookahead 只填了闲置时间没消成本，换模型规模、工具延迟与任务长度后 trade-off 可能反转。GUI 若每步都用大模型验证截图，成本结构更不利。
- **参数面窄。** 固定阈值、chunk=128、确定性 top-1 替换、K=3、验遍所有助手 token；从老师分布采样替换、只验工具调用等替代实例未对照。
- **统计与复现。** 每设定一次训练，显著性只刻画评测采样而非训练 seed 波动；VStar 仅 191 题；未开源。
- **过度保护的反作用未评。** pre-RL 阶段学生几乎没经历过被纠掉的坏调用及其后果，作者让下游 RL 再去暴露，但恢复/纠错能力是否被削弱、坏错误态是否拖慢 RL 未被直接评测——对强副作用环境是要害。
- **persistent 依赖老师完全固定**；中途升级验证器即破 carryover 语义，多阶段训练换老师的场景未讨论。

## 6. 对我们的启示：对 GUI Agent 的可借鉴点

- **把"动作提议"与"动作提交"分层。** GUI Agent 现状常是拿老师轨迹 hard target 直接 SFT（off-policy 冷启动）或跳过对齐直接环境 RL；PTA 给出第三条路——执行前带老师把关的 on-policy 对齐。对应 GUI：小/快模型提议点击、输入、滚动，大模型或带 grounding 的验证器放行；GUI 动作副作用不可逆，"验证完整回合再提交"天然契合。
- **观测不做 loss 是 GUI 训练的地雷。** GUI 里截图/DOM 前后状态是"上一步动作的结果"，当模仿目标会教学生背界面而非学决策。应给"助手动作 token"与"环境状态 token"打源标签、loss 只落动作——PTA 的源感知机制可直接移植进 GUI 的 SFT/蒸馏管线。
- **高危动作审查门。** GUI 中下单、删除、发送、授权弹窗等不可逆操作，恰似文中"占 0.05% 却决定胜负"的工具调用内修正：与其提高全局教师修正率，不如做**关键动作级审查门**（高价值/不可逆动作必须过老师或确定性规则），成本低、收益集中，与本库冲突感知/终止类工作互补。
- **persistent lookahead 在 GUI 更值钱。** GUI rollout 长（几十步 + 大截图），同步训练闲置比文本场景更严重；若验证器（如 grounding 模型）训练中固定，可跨梯度步 carryover 已验证的界面前缀，大幅省 rollout 开销。
- **评测协议直接借用。** "固定下游 RL 预算、比最终策略质量而非对齐 loss"——GUI Agent 的蒸馏/预训练论文照此设计对照更有说服力。

## 7. 延伸阅读

- **Search-R1**（Jin et al., COLM 2025）、**DeepEyes**（Zheng et al., ICLR 2026）：两个下游 RL 环境来源。
- **SKD**（Xu et al., ICLR 2025）、**SWITCH**（Koo et al., NAACL Findings 2025）、**SCoRe**（Lyu et al., ICML 2026）、**RSD**（Liao et al., ICML 2025）、**Speculative Decoding**（Leviathan et al., ICML 2023）：proposer-verifier 训练/解码家族。
- **ToolACE**（Liu et al., ICLR 2025）、**Magnet**（Yin et al., ACL 2025）：构造并验证工具轨迹做离线蒸馏的对照路线。
- **FIRST**（Shum et al., EMNLP 2024）：稀疏 logit / top-N 蒸馏。
- **Qwen3 / Qwen3-VL**（Qwen Team 2025；Bai et al. 2025）：teacher–student 对所用模型。

---
*解读生成时间：2026-09-08 19:20 ｜ 解读人：WorkBuddy（AI）*
