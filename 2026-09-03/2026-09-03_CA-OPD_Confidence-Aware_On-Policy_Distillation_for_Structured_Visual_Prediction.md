# CA-OPD: Confidence-Aware On-Policy Distillation for Structured Visual Prediction

> 一句话 TL;DR：用"教师置信度"把关学生自回归 rollout——低支持 token 被教师改写并给 CE 监督、高支持 token 保留并给分布蒸馏，阈值随训练从严到宽，让 0.8B 小模型在 GUI grounding 与 OCR 上显著逼近 9B 教师。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | CA-OPD: Confidence-Aware On-Policy Distillation for Structured Visual Prediction |
| **作者 / 机构** | Menghao Li\*、Linjie Mu\*、Yin Wang\*、Haotian Hu\*§（§=项目负责人）、Yannian Gu、Lujiayi Xue、Fanyi Wang†（*=共同一作，†=通讯作者）；天津大学、上海交通大学、StepX、中国科学技术大学 |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-02；arXiv preprint（cs.CV，未标注会议/期刊） |
| **arXiv 链接** | https://arxiv.org/abs/2609.02401 |
| **代码仓库** | ❌ 未开源（全文无代码/数据链接） |
| **数据集地址** | 未公开（仅使用公开 benchmark：ScreenSpot-v2/Pro、OCRBench-v2、CC-OCR、OmniDocBench、RefCOCO、MMBench） |
| **类型标签（论文类别）** | `Distillation` `GUI Grounding` `Online` |
| **训练方法标签** | `Distillation (On-policy)` `SFT` |
| **关键词** | 知识蒸馏；On-Policy；置信度门控；自回归错误累积；GUI Grounding；OCR |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-02_CA-OPD_Confidence-Aware_On-Policy_Distillation_for_Structured_Visual_Prediction.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：在"把结构化视觉输出（GUI 坐标、OCR 文本）序列化为 token 的自回归 VLM"这一统一范式下，如何把大模型能力蒸馏进小模型，同时不被自回归的**错误累积**拖垮。

- **为什么重要**：自回归结构里每一个已生成的 token 都成为下一个 token 的输入，局部小错会污染整条轨迹。GUI grounding 里一个坐标数字打错，后续所有数字 token 都条件在错误前缀上——这正是端侧小模型做 GUI Agent 时精度掉链子的根因之一。

- **现有方法有什么不足**：作者点名三类范式并给出结构性批评——
  1. **Offline KD / sequence-level KD**（Kim & Rush 2016）：学生照着教师生成的"干净轨迹"学，但推理时走的是自己生成的轨迹，存在 train-inference mismatch。
  2. **标准 OPD（GKD 式，Agarwal et al. 2024）**：让学生自产 rollout 解决了分布偏移，但"学生不可靠时（尤其训练早期）一个错误 token 一旦进前缀，后续所有教师监督状态都被带偏"。
  3. **交错蒸馏 SKD（Xu et al. 2025）**：教师按"排名"把关学生提议、必要时替换。作者指出两个盲区：其一，**排名 ≠ 绝对置信度**——排第三可能意味着 90 分也可能只是刚及格，同排名 token 的概率质量可差一个量级；其二，**教师出手纠正这件事本身是高质量教学信号**，现有方法只把它当轨迹修复工具，浪费了"这位置你错了"的明确反馈。

- **Research Gap**：作者主张补上的是——**"rollout 构建"与"token 级监督"应被同一把尺子联合优化**。前人把"要不要改写轨迹"和"改写后怎么监督"割裂成两件事；CA-OPD 用一个置信度门控开关同时决定二者。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

用一个"教师是否认可该 token"的置信度判断，同时决定**轨迹里放谁的 token**（学生保留 vs 教师改写）与**这个位置用什么 loss 教**（分布蒸馏 vs 交叉熵），并把门控阈值按训练进度从严到宽退火。

### 3.2 方法总览（Pipeline）

- **输入**：图像 I + 文本指令 q（x=(I,q)）；**输出**：token 序列 y（坐标或文字）。
- **模块与数据流**（自回归逐 token 循环，见正文 Algorithm 1）：
  1. **学生提议**：在状态 s_t=(x, y_<t) 上，学生采样候选 token ŷ_t。
  2. **教师打分**：计算教师对该候选的负对数似然 NLL = −log π_T(ŷ_t | s_t)，NLL 越小=教师越支持。给定阈值 τ，NLL 超阈值即判"不可靠"（r_t=1）。
  3. **按判决分流建轨迹**：r_t=1 时用教师 argmax token 改写并写回前缀；r_t=0 时保留学生 token。后续生成都条件在这条"学生/教师混合前缀"上（离散 rollout 操作，不回传梯度）。
  4. **监督与判决对齐**：被改写位置给 CE（L^R_t = −log π_S）；被保留位置给**教师 top-64 的截断 forward KL**（省算力，且被 clamp 到 ≥0）。
  5. **从严格到宽松的退火**：阈值按余弦从 τ_start=0.693 松到 τ_end=1.386（等价于教师概率下限 α 从 0.50 降到 0.25，前 10% 步保持最严）。
- **多教师路由**：grounding 与 OCR 各配一个域微调的 Qwen3.5-9B 教师，样本按域动态路由到对应教师。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：① 把把关标准从"排名"换成"绝对置信度（NLL/概率质量）"，并实证排名会漏掉"排名高但绝对支持不足"的错误（Figure 2b：被门控触发的提议中位排名第 3、100% grounding/88% OCR 落在 top-25）；② **干预决策与 token 级监督的耦合**——同一个 r_t 既决定轨迹又决定 loss（CE vs KL），这是全文最有解释力的一个设计。
- **工程组合**：top-k 截断 forward KL、argmax 写回、分块 rollout（每次 chunk 至多 32 token 再查询教师）、多教师按域路由、余弦退火——单独看都是已有组件，但组合成一个自洽的"从严到宽放权"训练协议是新的。
- **对性能提升最关键的设计**：ablation 显示**"在正确位置（低支持处）真写回"**是收益的充分条件——随机位置干预（同 3.87% 改写率）只到 40.20 与 OPD 持平；有门控但不写回也只有 40.92。其次是**退火方向**（反转调度掉到 44.36）。
- **证据不足 / 仅声称有效**：教师"置信度校准良好"是核心假设但无自校准方案；`λ_CE / λ_KL` 加权版本（Eq.13）只在公式里给出、无实验支撑；对"教师是否可靠"这一前提没有敏感性分析。

---

## 4. 具体技术细节

### 4.1 模型结构

- 全部是 **MLLM**（视觉编码器 + 自回归解码结构，非纯 LLM）：学生为 SFT-initialized **Qwen3.5-0.8B**；教师为两个分别做 grounding / OCR 域微调的 **Qwen3.5-9B**（冻结，仅推理打分与改写）。
- 学生可训练、教师冻结；训练精度 bfloat16 + fp32 master weights，FSDP 并行（学生 4 GPU + 教师 4 副本，共 8×A800）。学生与教师共享 tokenizer（limitations 明确排除了 tokenizer 不匹配的场景）。

### 4.2 训练流程（分阶段）

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| Stage 1：SFT 初始化 | 让学生先具备基础格式与任务能力 | 基础 grounding/OCR 序列生成 | 40K OCR + 6K RefCOCO 样本 | 图像 + 指令 + 标注前缀 | masked CE：`L_SFT = −Σ m_t log π_S(y*_t | x, y*_<t)` |
| Stage 2：CA-OPD 蒸馏 | 用教师置信度把关、蒸馏结构化预测能力 | 提升 grounding/OCR 精度并防止轨迹偏置 | 15,100 条（11,184 grounding + 3,916 OCR） | 学生自产 rollout + 教师门控改写 | `L = r_t·CE(教师 argmax) + (1−r_t)·top-64 forward KL`（被改写处 CE、保留处分布蒸馏） |

> 优化细节：batch 32、471 步、LR 1e-6 常数无 warmup、AdamW(β=0.9,0.999)、weight decay 0.01、梯度裁剪 1.0；rollout 温度 1.0；教师 argmax 解码；门控打分用教师 top-32、保留位蒸馏用 top-64；max prompt 8192 / response 1536 token。所有对比方法（Offline KD / OPD / SKD）共享同一初始化、数据与 471 步预算，各跑 3 次取均值。

### 4.3 推理流程

训练产物是标准自回归 VLM，**单步解码**（无 ReAct/多步规划）。评测时：GUI grounding 用官方 Qwen3.5 tool-call prompt（assistant turn 预填到坐标字段），每例采样 8 次（temp 0.7 / top-p 0.8 / top-k 20 / presence penalty 1.5），命中率=预测点落入目标框的 trial 占比；OCR 类统一 greedy 解码、关 thinking。推理阶段**不再有教师参与**，门控只存在于训练。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| ScreenSpot-v2 | Mobile/Desktop/Web 混合真实截图 | GUI grounding 点选 | 论文未明示 | 截图 + 指令 | 坐标点 | 8 次采样命中率（落入目标框占比） |
| ScreenSpot-Pro | Desktop 专业高分辨率 | GUI grounding 点选 | ~1581（由 Table 11 分桶 822+312+447 推断） | 高分辨率截图 + 指令 | 坐标点 | 8 次采样命中率 |
| OCRBench-v2 | 文档/自然场景 OCR | 识别/定位/抽取/解析/推理等 8 类 | 论文未明示 | 图像 + 文本 | 文字/坐标 | 官方 scorer（EN/ZH macro） |
| CC-OCR | 综合 OCR | 4 track 综合 | 论文未明示 | 图像 | 文字 | macro average（4 tracks） |
| OmniDocBench v1.5 | PDF 文档解析 | 全文解析 | 论文未明示 | 文档页 | 结构化文本 | full-page 1−NED |
| RefCOCO/+/g（retention） | General 指代 | referring grounding | 8 个官方 split | 图像 + 指代 | bbox | ACC@0.5 |
| MMBench（retention） | General 多模态 | 多模态 QA | EN-dev-v1.1 | 图像 + 问题 | 答案 | accuracy |

### 5.2 实验结果分析

**主结果（Table 1，3 次运行均值）**：CA-OPD（annealed α:0.50→0.25）在全部 6 个目标任务上胜出——

| 指标 | SFT-init | OPD | SKD | Offline KD | CA-OPD | 相对 0.8B 裸模型 | 相对 OPD |
|---|---|---|---|---|---|---|---|
| ScreenSpot-v2 | 80.46 | 87.34 | 87.63 | 87.96 | **88.41** | +9.08（79.33→） | +1.07 |
| ScreenSpot-Pro | 38.19 | 40.82 | 41.20 | 44.23 | **45.92** | +9.50（36.42→） | +5.10 |
| OCRBench-v2 EN | 50.98 | 50.79 | 51.30 | 51.56 | **52.47** | +6.72（45.75→） | +1.68 |
| OCRBench-v2 ZH | 51.31 | 52.45 | 52.43 | 52.91 | **53.22** | +9.77（43.45→） | +0.77 |
| CC-OCR | 66.40 | 63.93 | 63.83 | 66.99 | **67.32** | +5.13 | +3.39 |
| OmniDocBench | 70.49 | 68.57 | 68.36 | 70.40 | **70.43** | +8.15 | +1.86 |

关键判断：① 相对 OPD 的增益（尤其 SS-Pro +5.10、CC-OCR +3.39）说明收益来自**置信度干预机制本身**而非"做了蒸馏"；② 教师 9B 上界 SS-Pro 62.92 / OCRv2-EN 66.03，0.8B 仍有明显距离，但相对裸模型已大幅收敛；③ **保留能力**不退化：RefCOCO/MMBench 上 CA-OPD（81.90/80.16）与 SFT-init 基本持平。

**Ablation 说明（Table 2 + 附录）**：
- **位置比频次重要**：随机位置干预（同 ~3.87% 改写率）只到 SS-Pro 40.20，与 OPD 相当；把 SKD 改写率从 0.16% 拉到 3.41% 也只到 44.00，仍差 CA-OPD 1.92 分。
- **必须真写回前缀**：只门控不写回仅 40.92——只诊断不修复轨迹无效。
- **调度方向关键**：fixed α=0.25 为 44.58、反转余弦 44.36，均低于退火 45.92；sampled 写回 43.61、替换处用 KL 45.26 也略低。
- **分桶收益均匀**（Table 3）：按输出长度/初始水平/教师拒绝密度切分，CA-OPD 对 OPD 全部为正；"全对"组从 96.03 掉到 93.83（OPD 掉到 90.18），缓解但未消除能力侵蚀。
- **稳定性**：无门控 OPD 在 SS-Pro 上 run 间标准差 1.46，退火门控降到 0.72。

代价：训练每步 ~59.3s，是 OPD（39.6s）的 1.5 倍，主因是交错式教师查询，门控本身几乎不花钱。

---

## 6. 亮点与贡献（Why it matters）

1. **把关标准从"排名"升级为"绝对置信度"**：用 Figure 2b 的 counterfactual 证据（门控触发提议中位排名第 3、88–100% 落在 top-25）证明排名门控会系统性漏掉低绝对支持的错误，判断依据更贴近教师真实态度。
2. **"干预决策 = 监督信号"的优雅耦合**：一个 r_t 同时决定"放谁进轨迹"和"用什么 loss 教"，把 state visitation 与 knowledge transfer 统一进同一个自回归过程，理论清晰、可解释性强。
3. **从严格到宽松的进度式放权**：比固定阈值更符合教学直觉，并用消融证明"退火方向正确才有效"，为 online 训练里的"探索/控制预算"提供了一个可复用的调度范式。
4. **实验纪律好**：多教师路由、统一预算、3 次独立运行、消融把"位置/写回/调度/损失形式"拆得干净，结论可信度高。
5. **对 GUI 落地的直接价值**：0.8B 小模型在 ScreenSpot 系逼近 9B 教师，为端侧低成本 GUI grounding 模型提供了省算力的蒸馏配方。

## 7. 局限与可改进点（个人点评）

- **对教师校准质量高度敏感**：整个门控建立在"NLL 能反映真实可靠性"上，作者在 Limitations 自认域偏移需重调下限，但没给任何自适应校准方案。若教师与学生"同病相怜"（系统性偏差），这个过滤器会把错误当成真理灌输，全文无兜底讨论。
- **验证面偏窄**：学生仅 0.8B、只训 471 步 / 15.1K 条数据；GUI 侧只做了 grounding 点选（无端到端多步操作成功率），很难外推到完整 GUI Agent 的增益。single-teacher 实验（Table 14）是单跑，policy-gradient 基线（G.2）也是单跑未重复，都只是 sanity check。
- **能力侵蚀只是缓解而非消除**：全对样本 96.03→93.83，作者诚实报告了这一点，"保留能力不退化"的表述需要打折扣。
- **成本与复现门槛**：1.5× 训练开销来自分块逐位置查教师；top-k 截断不归一化 + clamp、chunk 重启生成等实现细节多，未开源下复现成本不低。

## 8. 对我们的启示 / 可借鉴点

- **"干预决策即监督信号"是可复用的设计模式**：在 online RL / 微调里，"这条轨迹要不要信、要不要回滚"与"从这条轨迹学什么"通常拆成两套逻辑；CA-OPD 用一个开关耦合二者，减少超参面、增强一致性。做在线策略迭代时值得照搬。
- **筛选用绝对置信度而非排名**：凡是用教师/奖励模型给候选打分再决定是否采纳的环节（rollout 过滤、数据筛选），都应检查是否只用了相对序而丢了绝对强度信息——后者在低质量长尾上更灵敏。
- **进度式放权 = 自动化的探索预算控制**：从严格到宽松的余弦阈值，可比拟到持续学习/在线适应场景："早期多信旧策略、后期多信新数据"。
- **多教师按域路由 + 防遗忘同表报告**：GUI grounding 与 OCR 各配专家教师、样本按域路由，避免单一教师"全能但都不精"；把 RefCOCO/MMBench 这类 OOD 保留指标与目标指标同表，能及时发现"蒸馏猛如虎、回头忘光光"，建议固化为评测闸口。

## 9. 延伸阅读

- **GKD**（Agarwal et al., ICLR 2024）：On-policy 蒸馏奠基作，本工作的 OPD 基线。
- **SKD**（Xu et al., ICLR 2025）：交错采样蒸馏，本文"排名门控"的直接对比对象。
- **MiniLLM**（Gu et al., ICLR 2024）：reverse-KL 生成式蒸馏，属"改损失面"的另一路线。
- **DAgger / Scheduled Sampling**：序列预测"轨迹暴露"问题的经典解法，与本文阈值退火精神相通。
- **ScreenSpot-v2 / ScreenSpot-Pro、OS-Atlas**：GUI grounding 必知基准与基座。
- **StepX-Edge**（Wang et al., 2026，参考文献）：同团队端侧 UI VLM，可连读理解其部署动机。

---
*解读生成时间：2026-09-10 ｜ 解读人：WorkBuddy（AI）*
