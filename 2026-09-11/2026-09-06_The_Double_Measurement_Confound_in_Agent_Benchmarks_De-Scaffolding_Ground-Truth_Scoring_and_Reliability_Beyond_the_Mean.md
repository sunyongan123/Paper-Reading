# The Double Measurement Confound in Agent Benchmarks: De-Scaffolding, Ground-Truth Scoring, and Reliability Beyond the Mean

> 一句话 TL;DR：agent benchmark 的分数被"脚手架替模型做决策"和"评分器只评形状"两个缺陷同时污染，二者互相掩盖；只有联合修复（去脚手架 + 真值打分）才能让分数重新识别模型能力。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | The Double Measurement Confound in Agent Benchmarks: De-Scaffolding, Ground-Truth Scoring, and Reliability Beyond the Mean |
| **作者 / 机构** | Yonghong Zhang¹\*、Shadi Motaali¹、Vu Phong Dinh²、Avin Piroutiniya³、Jorge E. López de Vergara¹、Luis de Pedro¹、Ricardo Correia¹、Isabel M. Parra¹、Yong Xie⁴\*；¹马德里自治大学（西班牙）、²IMDEA Nanociencia（西班牙）、³马德里康普顿斯大学（西班牙）、⁴西班牙国家研究委员会 CSIC（西班牙） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-06；arXiv preprint（cs.SE） |
| **arXiv 链接** | https://arxiv.org/abs/2609.09218 |
| **代码仓库** | ✅ GitHub：https://github.com/yonghongzhang-io/comtrade-openenv （ComtradeBench；含 seeded generator、八模型可靠性谱、2×2 干预与 τ-bench / BFCL 审计脚本） |
| **数据集地址** | 同上。以 **generator card** 形式发布（种子化程序生成器而非静态语料），`(task_id, seed)` 可完全再生 ground truth |
| **类型标签（论文类别）** | `Benchmark` `General` |
| **训练方法标签** | —（评测方法学，不训练任何模型） |
| **关键词** | agent benchmark；measurement validity；scaffolding；ground-truth scoring；worst-case reliability |
| **来源渠道** | arxiv-api |
| **PDF 存档** | 2026-09-06_The_Double_Measurement_Confound_in_Agent_Benchmarks_De-Scaffolding_Ground-Truth_Scoring_and_Reliability_Beyond_the_Mean.pdf |

---

## 2. 论文要解决的核心问题

- **问题**：benchmark 分数是"模型能力"的证据，还是"评测管线属性"的证据？Kimi 与 Claude 均分**完全相同**（97.5），**不含任何 LLM 的规则脚本**拿 96.8；Qwen2.5-7B 与 Llama-3.3-70B 拿到**逐种子字节相同**的分数向量。
- **为什么重要**：分数在指导部署决策；若不识别模型，"选哪个上线"就建立在不测量任何模型属性的数字上。
- **现有不足**：harness 敏感性研究证明换 harness 会改分数但不分解机制；自动评测器研究不问"谁执行了被评分的任务"；可靠性研究（pass^k）只看方差，不含种子化最坏情况。
- **Gap**：缺口是**联合识别问题**——只修评分器，它忠实报告脚手架的近乎完美执行；只去脚手架，形状评分器把真实差异压成噪声。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把分数写成 `S(m, c, j)`（模型 × 脚手架 × 评分器），证明两缺陷并存时分数不可识别，再用"去脚手架 + 真值 F1 + 尾部可靠性"的联合干预修回来，封装为 BenchAudit 协议与 validity card。

### 3.2 方法总览（Pipeline）

- **输入 / 输出**：已有 benchmark 的 harness、scorer 与种子化任务 → validity card（pass / fail / not probed）+ 修复后的可靠性谱。
- **模块**：(1) **测量模型** `S(m,c,j)`；(2) **脚手架谱**三档受控变量——**L0 全自动**（harness 做全部关键步骤）、**L1 半脚手架**（模型只拥有"故障时 retry 还是 skip"）、**L2 全裸**（模型拥有一切）；(3) **真值指标** ground-truth F1；(4) **可靠性指标** worst-case / CVaR@0.2 / Reliability@τ；(5) **BenchAudit**（所有权图 + 伪造提交探针 + 2×2 干预 + validity card）。

### 3.3 真正的创新点（去伪存真）

- **真·创新**：把 scaffold ownership 与 scorer validity 形式化为**联合识别问题**并给出判据 `D_claimed ⊆ D_model`；把"脚手架层级"从混淆项变成**受控自变量**；交付 validity card 并用自己提出的审计推翻**预注册**结论；提出**非退化条件**（通道须 `Var(A) > 0`），实测 L1 仪器失败（SKIP 120 次中被选 0 次）——是"不可识别"而非"低功效"。
- **工程组合**：CVaR、worst-group accuracy、bootstrap CI / Cliff's δ、Reliability@τ 都是既有工具，贡献在移植与协议化。
- **最关键设计**：**2×2 联合干预**。L0 下提交 payload 在全部 21 个模型对上 **SHA-256 完全相同**；只去脚手架时形状评分器把 **1.000 的真值差距压到 0.425**；只有联合格恢复 0.000→1.000。
- **证据不足**：完整 ownership 干预**只在作者自己的 ComtradeBench** 实现；**B1 实跑用了偏离的仪器**；**suite 小且饱和**（5 任务 × 10 种子）；**B2 零 slack 的翻转被归因为"资源算术"**；**τ-bench 的 η² 只能当排序读**。

---

## 4. 具体技术细节

### 4.1 模型结构

- **不训练、不微调任何模型**。被评测的是 8 个**纯文本 LLM**（非 MLLM，无视觉编码器）：Claude-Fable-5、Claude-Sonnet-4.6、GPT-5、Claude-Haiku-4.5、GPT-4o、GPT-4o-mini、Llama-3.3-70B、Qwen2.5-7B。
- **没有 GUI 动作空间**：交互只通过三个固定 MCP 工具 `get_task_info()`、`fetch_page(page, page_size)`、`submit_results(...)`，模型经 pinned API **冻结黑盒**调用。结论是"可靠性既不跟随规模也不跟随均分"——**Haiku-4.5 最坏情况从未低于 0.787**，而 GPT-4o 掉到 0.00。

### 4.2 训练流程

> 本文**不训练**。实际做的是"审计-修复协议 + 可靠性评测"。

| 阶段 | 目标 | 产出 | 判据 / 关键数字 |
|---|---|---|---|
| Stage 1 所有权映射 | harness 是否替模型做了执行关键决策 | 所有权图 | `D_claimed ⊆ D_model`？ |
| Stage 2 评分器探针 | scorer 是否准则有效 | 五类探针提交的评分分解 | 零模型调用，canned submission 灌进 `judge.py`；空提交 **0.648**，伪造与正确记录集同为 **0.987**，正确数据 + 矛盾自报掉 **0.398** |
| Stage 3 2×2 联合干预 | 两缺陷必须联合移除 | 2×2 格均值 + 离散度 | T3_duplicates，n = 5 seeds/cell，7 模型；L0 提交跨 21 个模型对 SHA-256 全同 |
| Stage 4 可靠性统计 | 报均值之外的可靠性 | 8 模型可靠性谱 + 尾部指标 | E1：8 模型在 L2 上跑 5 个执行任务，10 seeds/(model, task) |
| Stage 5 预注册压力测试 | 升级式压力是否比匹配均值的恒定压力更伤 agent | B1 null、B2/B2-slack exploratory | T9_clean_ab；L1 + 真值 coverage，n = 20 seeds/arm，按种子配对 |
| Stage 6 外部审计 | 两缺陷在别的 benchmark 上是否普遍 | τ-bench 与 BFCL 结论 + validity card | τ-bench 官方 scorer 零模型调用重放轨迹；BFCL AST scorer（400 × 5 submissions），400/400 通过、0/2000 偏离 |

### 4.3 推理流程

- **agent 侧是多步 loop**：`get_task_info()` → 反复 `fetch_page` 抓全分页 → 去重、丢 totals 行、重试瞬时故障 → `submit_results(...)`。对抗性**在环境里**（故障、重复、干扰行由 mock 注入，prompt 不描述）。
- 差异在于**谁执行这些步骤**：L0 由 harness 自动完成，L2 由模型完成全部。每个 (level, seed) 单次运行，靠 10 个种子估计可靠性。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | 场景 | 任务类型 | 数据规模 | 输入 / 输出 | 评测指标 |
|---|---|---|---|---|---|
| **ComtradeBench**（本文主角） | General（对抗性分页式数据抽取 ETL） | 抓全分页 → 去重 → 丢 totals 行 → 重试故障 → 提交清洗后记录集 | 10 个任务隔离故障族（含 T9_clean_ab）；E1 用 5 个执行任务；T7 被排除 | 文本任务 + 3 个 MCP 工具 / 记录集 + metadata | **ground-truth F1**（主）；shipped judge 六维 100 分制（对照）；worst-case / CVaR@0.2 / Reliability@τ / bootstrap CI / Cliff's δ |
| **τ-bench**（外部审计） | General（tool-agent-user 交互） | 工具使用 + 用户模拟 | 4 scaffold × 3 模型 × 15 任务/cell | 文本 / 轨迹 + 最终 DB 状态 | 官方 outcome-based reward（哈希 DB 状态） |
| **BFCL**（外部审计） | General（函数调用） | 函数调用规格合规 | 400 entries × 5 submissions | 文本 / 函数调用 | AST 匹配（对照规格，非执行成功） |

### 5.2 实验结果分析

**生产配置（L0）的失效证据**：Kimi 与 Claude 均分**都是 97.5**，Qwen2.5-7B 与 Llama-3.3-70B 的**逐种子 reward 向量字节相同**。评分器探针：空提交得 **0.648**，伪造与正确记录集**同为 0.987**——"评分器评的是报告而非数据"。

**去脚手架 + 真值评分后的可靠性谱（E1，L2，每模型 50 runs）**：

| 模型 | 均值 | 最坏情况 |
|---|---|---|
| Claude-Fable-5 | **0.974** | **0.824** |
| Claude-Sonnet-4.6 | 0.943 | — |
| GPT-5 | 0.930 | 0.00 |
| Claude-Haiku-4.5 | 0.924 | **0.787** |
| GPT-4o | 0.805 | **0.00** |
| GPT-4o-mini | 0.610 | 0.00 |
| Llama-3.3-70B / Qwen2.5-7B | 0.000 | 0.000 |

**均值之外的排序**：GPT-5 与 Haiku 均值只差 **0.006**，但 CVaR@0.2 差 **0.079**；GPT-4o 均值第五却最坏触底，且是**真实双峰**——T3 上其逐种子 F1 为 `[0,0,1,0,0,0,0,1,1,1]`，n = 30 复现定位到**是否曾变化过一次 off-by-one 的 `fetch_page(page=0)` 调用（14/14 vs 0/16）**。

**2×2 联合干预与消融（T3，7 模型）**：L0 两格中七个模型**全部并列**（judge 0.987、gtF1 1.000，离散度 0.000）——**只换评分器没用**。L2 × judge 离散度 0.948 但**排序是错的**：形状评分器把 1.000 的真值差距压到 0.425，还给全错提交打 0.523。只有 L2 × gtF1 恢复完整谱，离散度 **1.000**——"两缺陷互相掩盖"是**被测量的，不是被论证的**。

**脚手架层级的定性影响（L1）**：L1 只把"故障时 retry 还是 skip"交给模型，该通道**在本环境里退化**——120 个 episode 中 SKIP 被选 **0 次**，coverage 稳定约 0.83。

**预注册压力测试（B1/B2）**：B1 预注册臂三个模型 Δ = +0.000（p = 1.00）；B2 零 slack 下 Llama Δ = +0.087；**无 LLM 的 always-retry 规则对照拿到 +0.096**，比任何模型惩罚都大。**B1 是本文最精彩的自我推翻**：预注册指定的仪器跑出 **GPT-5 Δ = +0.011，p = .034**——按规则这是**确认**；但**同一批 episode** 上 GPT-5 的真值 F1 与 coverage 的 Δ 约为 0——这个"确认"完全活在真值不测量的 judged 维度里；论文由此给出漂亮的观察：**skip 在充裕时是成本、在稀缺时是保险**。

**外部审计**：τ-bench 官方 scorer 是 **outcome-based**（哈希 DB 状态），verbatim 通过得 1.0、损坏得 0.0——**形状评分器缺陷是 benchmark 特有的**；但**脚手架半边是不受控轴**：只改 scaffold flag，同一模型 reward 移动最多 **0.267** 并重排模型。BFCL 的 AST scorer 正确调用 400/400 通过、其余全失败（0/2000）——它是**规格合规**评分器而非执行成功评分器。该 slice 把 η² = **0.0293** 归给 scaffold、**0.0080** 归给模型：**测量方法解释的方差比被测量的属性更多**。

---

## 6. 亮点与贡献（Why it matters）

1. **"7 个模型提交字节相同的 payload"是近年最有冲击力的 benchmark 失效证据**——字节级同一性比 p 值更有说服力，应立刻检查自己 benchmark 的 scaffold 替模型做了多少事。
2. **"两缺陷互相掩盖、必须联合移除"是可迁移的思维模型**，解释了"修了评分器分数没变"与"去掉脚手架反而更糊"两种困惑。
3. **把最坏情况变成可估计量**：seeded adversary 让 worst-case / CVaR 可复现，n = 30 取证复现把 GPT-4o 双峰定位到一次 off-by-one 的 `fetch_page(page=0)`。
4. **"所有权是必要而非充分条件"**：把决策还给模型并不自动等于测到了模型；L1 的 SKIP 通道 120 次中 0 次被触发即证据。
5. **自我指涉的诚实性**：论文把审计用在自己身上推翻预注册结论，让一篇讲评测可信度的论文自己也可被评测。

## 7. 局限与可改进点（个人点评）

- **单域**：完整 ownership 干预只在作者自己的 ComtradeBench 上实现，最强主张的外部证据只有 τ-bench 上一次 n = 15/cell 的 slice，**外部效度尚未建立**。
- **B1 仪器偏离**：预注册的"确认"只活在评分器维度、且需另一仪器复现，它**既没有确认也没有证伪**；作为"评分器混淆"案例极好，作为"非平稳性"证据近乎为零。
- **L2 机械地板未解**：L2 把"对抗下决策错误"与"机械重提交失败"折进同一分数，触底层的 0.000 不完全是决策质量；本该诊断它的 L1 通道恰在本环境退化。

## 8. 对我们的启示 / 可借鉴点

> 本篇是 **Agent 方法论域**（benchmark 评测方法学，非 GUI 落地），以下先给通用启示，再单列对 GUI Agent 的可迁移点。

**通用启示**：报分数必须同时声明脚手架层级、评分判据（是否对准真值）与可靠性画像（worst-case / CVaR / Reliability@τ）；均值最后才看。并区分"不可识别"与"低功效"：通道 120 次都没被激活时，加样本量无用。

**对 GUI Agent 的可借鉴点**：

1. **给 GUI benchmark 画"决策所有权图"**：GUI scaffold 常替模型做坐标归一化/缩放、动作校验与重试、滚动等待与截图标注——恰是最易失败处。只要 `D_claimed ⊄ D_model`，报的就不是模型能力而是 system performance。
2. **引入"最坏情况"评测 + 取证式复现**：GUI 成功率天然高方差，只报均值会掩盖双峰。固定一组种子化扰动，同模型跑 10+ 次报 worst-case 与 CVaR@0.2；对触底案例做 n ≥ 30 复现，定位"哪个动作变体决定成败"。
3. **警惕"动作格式合规"冒充"任务成功"**：本文的形状评分器给伪造与正确记录集打同样的 0.987。**GUI 的 ground truth 应锚在环境状态（DOM/文件系统/数据库）而非动作形状上**；坐标预测类子任务用邻近度判分时须声明它测的是 grounding 精度。
4. **GUI 的 scaffold 敏感性可能比 τ-bench 更严重**：只换 scaffold flag 就让同一模型 reward 移动最多 0.267 并重排模型；GUI 差异更大（a11y tree、SoM 标注、坐标 vs 元素 ID）。**报 GUI 分数必须声明 harness 配置，不同 harness 不可直接对比**。
5. **"谁拥有 retry/skip 决策"是能力与安全双重关键**：L1 表明该通道在本环境退化，且"skip 在充裕时是成本、在稀缺时是保险"。GUI 的对应决策是页面未加载时重试还是跳过、操作失败时换策略还是回退——极易被 harness 用"自动重试 3 次"代劳，若不留在模型手里，RL 的 credit assignment 会错位。

## 9. 延伸阅读

- **AI Agents That Matter**（TMLR 2025）、**Holistic Agent Leaderboard**（ICLR 2026）——cost-aware 与 holistic agent 评测的奠基性批评。
- **Harness-Bench**（arXiv:2605.27922）——6×8 harness–model factorial，本文说它"覆盖脚手架轴但未分解"。
- **τ-bench**（ICLR 2025）、**BFCL**（ICML 2025）——本文的两个外部审计目标，分别代表 outcome-based 与 spec-based scorer。
- **WebArena / OSWorld / GAIA**（ICLR 2024）——本文点名的 GUI/通用 benchmark，脚手架受控性值得用 validity card 逐个过一遍。
- **ComtradeBench 仓库**：https://github.com/yonghongzhang-io/comtrade-openenv

---
*解读生成时间：2026-09-11 09:00 ｜ 解读人：WorkBuddy（AI）*
