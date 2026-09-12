# T1: Terminal Agent Reinforcement Learning for Long-Horizon Tasks

> 一句话 TL;DR：用 PPO 在真实云沙箱里训练 122B MoE 终端智能体，靠 TITO + 路由重放（R3）解决训练/推理不一致，把 Terminal-Bench 2.1 从 43.8% 推到 64.0%。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | T1: Terminal Agent Reinforcement Learning for Long-Horizon Tasks |
| **作者 / 机构** | Junyao Yang、Yucheng Shi（\*同等贡献）、Leowei Liang（†通讯）；腾讯混元基础模型前沿团队、新加坡国立大学、佐治亚大学、印第安纳大学、马里兰大学学院公园分校 |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-10；arXiv preprint（cs.LG），37 页技术报告 |
| **arXiv 链接** | https://arxiv.org/abs/2609.11042（点击直达） |
| **代码仓库** | ⚠️ PDF 首页标注 Project Page 与 HuggingFace 入口，但文本层未保留具体 URL，无法确认真实链接；论文未给出公开 GitHub 仓库。记作：未公开 |
| **数据集地址** | 未公开。TMax-15k（14,601）、RST-38k（37,484）、T1-15k（15,000）三个训练池均无下载地址 |
| **类型标签（论文类别）** | `RL` `Online` `General` |
| **训练方法标签** | `RL (PPO)`、`Online RL`（一步异步）、`MoE Routing Replay (R3)`、`TITO`；另做过 `RL (GRPO)` 对照但失败 |
| **关键词** | 终端智能体、长程强化学习、MoE 训练稳定、密集过程奖励、训练-推理一致性、可执行验证器 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-10_T1_Terminal_Agent_Reinforcement_Learning_for_Long-Horizon_Tasks.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：在真实 Linux 终端里用**执行结果奖励**（而非偏好模型）对前沿规模稀疏模型做 RL 后训练，完成需数百次工具调用的"定位缺陷—打补丁—重编译—自证修复"任务。
- **为什么重要**：作者视终端为智能体操作生产基础设施的必经一步；不能稳定优化执行反馈，智能体就只停留在"描述怎么做"。
- **现有方法的不足**：合成流水线多用 SFT 验证，RL 证据局限于较小稠密模型 + 二值奖励，增益常仅几个点；二值奖励对部分完成零可见性——作者首次二值奖励训练**从未超过 SFT 基线**；MoE 专家权重占 121.4B 中的 116.0B，两栈 kernel 数值差会翻转 TopK 专家选择，harness 又在每个 turn 边界重渲染历史，使梯度落到并非生成该行为的子网络；GRPO 实测两步平盘（51.7%）。
- **Research Gap**：缺一套"前沿规模稀疏模型上、用执行反馈、对长程轨迹稳定 RL 的完整配方"。论文主张补上三件：TITO+R3+critic 调度的稳定化栈、可执行验证器的密集奖励、与评测完全 OOD 的训练语料，以证明增益是能力迁移而非过拟合。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把每条终端任务当作自包含可验证 episode，用一步异步 PPO 配密集过程奖励训练 122B MoE，并以 token 保真（TITO）与专家路由保真（R3）把重要性比率钉死在真实采样策略上。

### 3.2 方法总览（Pipeline）

- **输入**：`task_<id>/`（`task.toml` 资源上限、`instruction.md` 长程指令、`environment/` Docker 规格、`tests/` 逐断言 held-out 验证器、`solution/` 不可见）；**输出**：CTRF 奖励 → 标量密集奖励 → PPO 权重。
- **模块**：①Tasks（RST 递归合成出 T1-15k）；②Training Framework（slime 异步 PPO，SGLang 副本 oversampling，Megatron 训练，critic 先更、actor 后更，每步发布一次权重）；③Sandbox（Daytona 云沙箱，上限 500 turn）。Trajectory Assembler 把多轮日志规范化为 TITO 拼接样本，携带 token id、loss mask、log-prob、router experts 四路流。
- **不变量**：四路流每阶段施加相同偏移；观测与"胶水"token 以 m=0 进上下文、不产生梯度。

### 3.3 真正的创新点（去伪存真）

**真·方法创新**
- **TITO 边界修复层级**：把"turn i 输出必须是 turn i+1 提示的 bit-exact 前缀"落成 strict → normalized（97×17 网格有界修复）→ retokenized（回退文本等价，追加采样到的 a_i 而非重编码的 ã_i）→ split 四档规则。
- **R3 路由重放**：只回放选择 mask、softmax 仍作用于当前 logits，既消除 TopK 不连续性又保留 router 梯度。
- **密集奖励的绝对计数**（r=P/S，S=20 全局固定）：论证"绝对通过数优于比例"（10/20 与 2/4 比例相同但验证进度差一个量级）；S 取断言数分布的约 90 分位并跨步固定，以保 critic 目标一致。

**工程组合**：RST 合成、oversampling 截尾、8 维 LLM 审计、critic warm-up 与 slime/Megatron/SGLang/Daytona 集成；上下文并行的 recurrent scan 切分属系统工程。

**对性能提升最关键的设计**：论文证据指向**密集奖励 + critic 校准**——二值 run 只到 47.2%（低于 SFT 49.4%），密集 run 到 59.9%/64.0%，且两次跳升（61.8% @ step 70、64.0% @ step 110）恰落在 explained variance 最高的区间。TITO+R3 把 |Δlogp| 从 0.021 压到 0.013，但它不直接涨点，也没有"关掉 R3 看掉多少分"的消融。

**证据不足 / 仅声称有效**：无单轴奖励消融（作者自认对比是 campaign 级，四变量同时变）；retokenized 是条件化近似，影响 2.6% 的 token；验证器无运行时防篡改；领域强项仅基于 5 条与 9 条任务。

---

## 4. 具体技术细节

### 4.1 模型结构

Qwen3.5-122B-A10B：总参 121.4B（专家 116.0B、稠密 5.86B），48 层、每层 256 选 8，激活约 10B；纯文本 LLM，无视觉编码器。全参数 RL（非 LoRA）；critic 是同架构第二份 122B 副本（LM 头换标量 value head），与 actor 强制 offload、时分复用同组设备，各自需单独塞进 95GiB。

### 4.2 训练流程

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| Stage 0：Critic Warm-Up | 策略更新前先校准 value | 预测轨迹最终密集分数（回归） | TMax-15k，1 epoch | 多轮终端轨迹 + 标量 | 价值回归 L_V（Eq.17，clipped value loss，ε_v=0.2）；**只保存 critic 权重**，不跨 run 传优化器动量 |
| Stage 1：SFT 初始化（外部产出） | 提供 RL 起点 | 模仿参考解 | RST-38k | 轨迹文本 | 论文标注"outside our scope"，未给细节 |
| Stage 2：PPO 主训练 | 用执行奖励提升长程终端能力 | 环境理解、任务分解、部分失败恢复、停止决策 | T1-15k（15,000 条，逐断言验证器） | TITO 拼接 token 流 T=(T1..TN)，含 mask m、log-prob q、路由记录 R∈[E]^(ρ×L×k)（48 层×8 专家×4B≈1.5KiB/位置，33k 轨迹约 48MiB） | PPO clip loss（Eq.18），ε=0.2 对称裁剪，**两个 KL 项全关**；GAE γ=λ=1，优势由 critic 前值 V_old 锚定；Adam β=(0.9,0.98)、weight decay 0.1、常数学习率（actor 1.0e-6 / critic 1.5e-5）；batch 512（提交 560 取先完成 512）、84k 上下文、MoE 负载均衡系数设为 **0** |
| 奖励计算 | 逐断言结果 → 标量 | — | 验证器 CTRF 报告 | 每 trial 一个标量，广播到该 trial 所有 chunk，落在最后一个 response token | r = P/S，S=20 固定；报告缺失/不可解析时回退二值 b（"已解出任务永不为 0"）；不做归一化（每任务每步仅一条轨迹，无组统计量） |

### 4.3 推理流程

- **推理模型**：T1（RL 后 122B MoE）；输入指令 + 逐 turn 终端返回，输出 shell 命令。模型与 harness 交换 **token id 而非文本**，并回传 log-prob 与每层专家路由。
- **多步循环**：Terminus-2（Harbor 托管）——assistant turn 发命令、tool turn 返回输出；最多 60 turn，agent wall 3600s、verifier wall 900s，每 trial 新建 10GiB 沙箱并销毁；启用主动上下文压缩；解码 96k 上下文、temperature 0.1、top-p 0.95、top-k 20，每任务 3 次尝试。终止条件为完成、turn 上限或超时。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| Terminal-Bench 2.1 | Desktop（Linux 终端/云沙箱） | 端到端长程终端任务 | 89 条 held-out | 长程指令 + 终端观测（纯文本） | shell 命令序列 | resolved 率（执行判分） |
| Long-Horizon Terminal Bench (LHTB) | Desktop（终端） | 超长程、抗记忆/抗捷径 | 46 条，9 类 | 同上 | 同上 | 平均奖励（连续部分分） |
| Terminal-Bench Hard (TBH) | Desktop（终端） | 更难的独立任务分布 | 100 条 | 同上 | 同上 | resolved 率 |
| 训练池 TMax-15k / RST-38k / T1-15k | — | 训练用可验证任务 | 14,601 / 37,484 / 15,000 | task.toml + instruction + tests | CTRF 逐断言 | 二值（TMax）/ 逐断言（T1-15k，抽样 93% 含逐断言记录） |

三个评测集均不向训练池贡献任务。论文提示 harness 会显著改变分数：Opus 4.6 在 Claude Code 下 70.1、Terminus-2 下 63.8。

### 5.2 实验结果分析

- **主结果（同 harness）**：T1 在 Terminal-Bench 2.1 达 **64.0%**，超 GPT-5.4（54.8）、DeepSeek-V4-Flash（56.9）、Claude Opus 4.6（63.8），接近 Claude Opus 4.7（66.1）。自家轨迹：base 43.8% → RST-SFT 49.4%（+5.6pp）→ T1 64.0%（RL 再 +14.6pp），总增益 20.2pp 中 **RL 占 72.3%**。
- **长程迁移**：LHTB 上 18.9 → 23.6 → **27.9**（RL 相对 SFT +18.2%），与 Gemini-3.1-Pro 持平，超 GPT-5.4（27.2）与 GLM-5.1（26.7），落后 GLM-5.2（31.6）。注意 Claude Sonnet 4.6 在 TB 2.1 低于 T1 却在 LHTB 拿到 37.3——长程能力不能由 TB 2.1 排序外推。
- **更难分布与对照 campaign**：TBH 上 20.0 → 28.3 → **38.0%**，超 DeepSeek-V4-Pro（36.0）；二值奖励 + TMax-15k（从 base 起）只到 47.2%，未追上 SFT 的 49.4%；密集奖励 + RST-38k 到 59.9%；GRPO 对照 51.7% 平盘。
- **Ablation 结论**：①**奖励密度最关键**（campaign 级证据，缺单轴消融）；②**critic 校准是长程提升前提**：cold-start critic 初始 EV=−33.6、58 步中 30 步为负，warm-up 后稳定在 0.71–0.86；27B 实验把 critic/actor 学习率比从 10× 提到 20× 可把 EV 从 −39 拉到 +0.11，但**奖励始终持平**；③**稳定化机制**：|Δlogp| 0.021→0.013、loss 区 token drift 0.0000%；④**固定尺度有效**：训练奖励从 0.250 升到约 0.345 后稳定在 0.34–0.36（约每轨迹 7 个通过断言），未被 1.0 封顶；⑤**turn 增长有界**：10.4 → 20.7 次工具调用，60 步后走平；⑥**难度分层**：Easy 三档均 100%，Medium 78/58/56%，Hard 仅 33/30/20%，RL 增益主要在 Medium。

---

## 6. 亮点与贡献（Why it matters）

1. **把"训练-推理不一致"变成可测量指标**：给出 |Δlogp| 定义、loss 区的 mask 加权归一化（按 micro-batch 数归一会把统计量放大三个数量级）与前后对照。
2. **密集验证奖励的论证最有迁移价值**：绝对通过数 + 全局固定 S=20 + 不做优势归一化 + critic 成唯一基线，与"每任务单样本、长程、混合难度"自洽。
3. **诚实记录失败**：GRPO 为何失效、cold-start critic 的 EV=−33.6、上下文压缩 98.3% 失败导致 turn 从 22 涨到 30。

## 7. 局限与可改进点（个人点评）

- **因果归因偏弱**："RL 占 72.3% 增益"是同一模型三段 checkpoint 的纵向对比；"密集奖励优于二值"只有 campaign 级证据。作者自己开出正确药方（同池同初始化同 R3 的二值 run）却没做，故目前是**强相关而非因果**。
- **评测预算不对等**：与 GPT-5.6 Sol 对比时 T1 最大输入 56k vs 120k、输出 8192 vs 32768，故 debugging 100.0% / sysadmin 88.9%（n=5、n=9）几乎无法作为能力结论。
- **效率代价被低估**：评测平均 94.4 turn、失败案例 164–473 turn 才超时，论文却没给出**成本归一化后的性能**，而 3 倍 turn 即约 3 倍成本。
- **可复现性与数据偏斜**：训练池与首页链接均无可用 URL；T1-15k 严重偏斜（前 5 类占 67.9%，debugging 1.26%、文档 0.56%），而作者又把失败归因于"ML/数据科学覆盖不足"，二者因果未被分离。
- **口径矛盾**：critic 学习率 30× vs 15× 对不上；优化器放置、权重同步时序等仍标注"未完成"——这是一份 bring-up 报告，而非可精确复现的配方。

## 8. 对我们的启示 / 可借鉴点

- **训练侧**：多轮 agent RL 中"重要性比率的分子分母必须来自同一策略"应是硬约束；建议落地 loss 区内 token drift 率、专家选择重放一致率、mask 加权 |Δlogp| 三个度量（**按 Σm_j 归一，不能按 micro-batch 数**）。
- **奖励侧**：环境若能给逐断言结果，优先用"绝对通过数 × 固定分母"而非比例或组内归一化；分母固定在整个 run 的尺度上，否则 critic 回归目标会随批次漂移。

### 对 GUI Agent 的可借鉴点

1. **把训练/推理不一致审计搬到 GUI**：GUI 也有截图预处理、坐标归一化、动作重渲染造成的编码-解码往返。可类比 TITO：**记录模型实际采样出的动作 token id（含坐标数字的分词切分），训练时复用同一 id 序列**，而不是把动作序列化成文本再重新分词——坐标类 token（如 "0.83"）重分词时极易改变切分，正对应本文的 retokenized 情形。
2. **多步 GUI 轨迹照搬边界修复层级**：strict → 有界修复 → 文本等价回退 → 开新 chunk，适配"截图 + 动作"交替的拼接；关键是**回退时追加采样到的动作而非重编码的动作**。
3. **把密集奖励下沉到子目标断言**：GUI 任务可拆成"打开应用 / 到达目标页 / 输入正确内容 / 提交成功"等断言，用绝对计数 + 固定分母替代 0/1 信号，缓解长程 GUI"全败批次无梯度"。
4. **混合难度批次下不做优势归一化**：GUI benchmark 也是 Easy 全满、Hard 长期低分，组内归一化会抹掉跨任务难度差；建议让 critic 当唯一边线，并监控其 EV。

## 9. 延伸阅读

- RST: Recursive Synthesis for Long-Horizon Terminal Tasks, arXiv:2608.05466 —— T1-15k/RST-38k 来源。
- Long-Horizon Terminal Bench, arXiv:2607.08964；Terminal-Bench, arXiv:2601.11868 —— 主评测集。
- Stabilizing MoE RL by Aligning Training and Inference Routers, arXiv:2510.11370 —— R3 原始提出。
- KPop / IcePop / SAT / GSPO / DPPO —— 从目标函数侧处理同一问题的平行路线。
- slime（github.com/THUDM/slime）—— 本文训练框架基座。

---
*解读生成时间：2026-09-12 ｜ 解读人：WorkBuddy（AI）*
