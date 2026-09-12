# Do Agents Know When They Succeed? Calibrating Agent Confidence from Internal Representations

> 一句话 TL;DR：从 LLM 残差流里读出多步 Agent 的成败置信度，用 LTD（轨迹动力学）与 ARP（动作表征探针）在三个交互式编程环境上零额外开销地压过表面 logprob 与 HTC。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Do Agents Know When They Succeed? Calibrating Agent Confidence from Internal Representations |
| **作者 / 机构** | Priyanka M. Mammen（UMass Amherst，美国）、Emil Joswin、Srujananjali Medicherla（Independent Research）；通讯 pmammen@umass.edu |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-08；NeurIPS 2026（页脚标注 40th Conference on Neural Information Processing Systems）；arXiv:2609.09448v1 [cs.AI] |
| **arXiv 链接** | https://arxiv.org/abs/2609.09448 |
| **代码仓库** | ❌ 未开源（正文与附录均未给出仓库链接） |
| **数据集地址** | 未新增公开数据；使用公开 benchmark InterCode（Bash/SQL/Python），官方实现 https://github.com/princeton-nlp/intercode |
| **类型标签（论文类别）** | `General` `Reflection` |
| **训练方法标签** | —（不训练 LLM；仅在冻结模型残差流上训练 L2 逻辑回归探针 + 单调 Platt 校准） |
| **关键词** | 置信度校准、机制可解释性、残差流探针、多轮 Agent、可靠性监控 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_Do_Agents_Know_When_They_Succeed_Calibrating_Agent_Confidence_from_Internal_Representations.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：在无标签的实时运行中判断多步 LLM Agent（Bash/SQL/Python 交互式编码）当前轨迹最终会不会成功，以决定是否触发接管或 fallback。
- **为什么重要**：Agent 正进入安全攸关场景（软件工程、金融、临床、机器人）。部署期没有标签，唯一抓手就是"评估步骤"给出的置信度；它不可靠，Agent 就会在错误轨迹上一路走到黑。
- **现有方法有什么不足**：① 单轮校准技术（temperature scaling、语义熵、verbalized confidence）假设模型孤立工作，而 Agent 在 harness 中多轮规划、调工具、接环境反馈，静态技术不成立；② 轨迹级外部信号（UProp、HTC）只吃输出层信号，抓不到失败前的内部状态；③ 已有机制可解释性工作大多停留在单轮（如工具调用决策）。核心论据取自 Azaria & Mitchell (2023)：内部状态比输出更能指示"是否在说谎"。
- **Research Gap**：补上"机制可解释性"与"长时程多轮 Agent"之间的缺口——首次把残差流级置信信号用于轨迹级成败预测。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把 Agent 生成时本就存在、却被推理框架丢弃的残差流隐状态取出来，用 LTD（轨迹几何漂移）与 ARP（动作表征探针）两种互补方式拟合轻量逻辑回归，输出轨迹成功概率。

### 3.2 方法总览（Pipeline）

- **输入 / 输出**：输入轨迹 τᵢ={(gᵢ,ₜ,oᵢ,ₜ)}（每步生成 + 环境观测）及每 token 的 log-prob 与残差状态 h^(ℓ)；输出轨迹级置信度 ĉᵢ≈P(yᵢ=1|τᵢ)。
- **LTD**：在"观测→推理→动作→反馈"端点取残差状态，算相邻状态的余弦位移与相对位移，对四类转移做 mean/final/variability/trend 汇总，加路径效率与层间漂移比，共 28 维。
- **ARP**：只取动作 span 末尾 token 的末层残差状态，剔除 submit 后 mean-pool 得 episode 表征，标准化 + PCA 到 64 维。
- **预测层**：两者共用同一个 L2 逻辑回归（L-BFGS；C 在 9 点指数网格上按内层 4 折 CV 的负 Brier 选取），与 HTC 属同类预测器，避免混淆表征优势与模型容量优势。
- **校准层**：ARP 接单调 Platt 映射 g(p̃)=σ(a·logit(p̃)+c)，约束 a≥0 以不破坏 AUROC。
- **连接方式**：特征→探针→概率→校准全部离线拟合；标准化、PCA、探针、校准器只用训练折统计量。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：把机制可解释性探针从单轮搬到**多轮 Agent 轨迹**，并给出两个新信号定义——LTD 的轨迹几何量化（层间漂移比、路径效率）与 ARP 配方（动作末状态 + 剔 submit + mean-pool + PCA64 + 单调 Platt）。5 折嵌套 CV、冻结折边界、折内拟合全部预处理，在这类论文中属少见克制。
- **工程组合**：逻辑回归探针、PCA、Platt scaling、teacher-forcing 取激活全是现成技术，LTD 的均值/末值/方差/趋势也是标准特征工程；价值在组合位置选得对。- **对性能最关键的设计**：**ARP 明显优于 LTD**——9 个设置里 ARP 拿下 8 个最佳 AUROC 与 8 个最佳 Brier，说明成败判据集中在"决定要做什么那一刻"的表征，而非过程曲线形状。次要关键是剔除 submit，防探针在终止语法上 shortcut。
- **证据不足 / 仅声称有效**：① **无组件消融**——28 维 LTD 特征、k=64、只用末层、mean-pool 均无敏感性分析；② "内部方法在每个设置都更好"靠**取 LTD/ARP 中较优者**成立（DeepSeek-Bash 上 ARP 的 0.650/0.167 均不如 HTC 的 0.698/0.165）；③ "零开销"只在附录 I 做理论论证，实验全程离线重放。

---

## 4. 具体技术细节

### 4.1 模型结构

base model 是**纯 LLM，不含视觉编码器**：Qwen2.5-Coder-7B-Instruct、Qwen2.5-Coder-14B-Instruct-AWQ、DeepSeek-Coder-6.7B-Instruct，本地 vLLM 服务（暴露 per-token logprob），单张 A100，greedy 解码。**LLM 全部冻结**，只做一次确定性 teacher-forced 前向记录残差流；被训练的只有挂在隐状态上的 L2 逻辑回归探针与 Platt 校准器。探针取末层残差（d=5,120），PCA 降到 k=64。

### 4.2 训练流程（若需要训练）

**不训练 LLM**，只训练读隐状态的探针。按阶段列出探针拟合流程：

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| Stage 0 轨迹采集 | 生成待评估轨迹 | — | InterCode 三环境 vLLM rollout，T=10 turns | 轨迹文本 + 每 token logprob | — |
| Stage 1 激活提取 | 拿内部表征 | — | 对同一轨迹做一次确定性 teacher-forced 前向 | h^(ℓ)ᵢ,ₜ∈R^d，按观测/推理/动作/反馈端点切分 | — |
| Stage 2 特征构造 | 得固定维度输入 | — | 同上 | LTD 28 维统计特征；ARP 动作末端状态 mean-pool→标准化→PCA64 | — |
| Stage 3 探针拟合 | 预测轨迹成败 | 内部状态是否线性编码成败 | 5 折分层外层 CV，折内再 4 折 | (uᵢ,yᵢ) 或 (x^LTDᵢ,yᵢ) | L2 逻辑回归（L-BFGS ≤2000 iter），C∈{1e-4…1e4}，内层按负 Brier 选 C |
| Stage 4 概率校准 | 输出可信概率 | — | 折内 out-of-fold 验证预测 | 标量概率 | 1D 逻辑映射，MLE 拟合，约束 a≥0 |

> 数据规模与成功率：Bash 200 题（33.5% / 30.3% / 22.5%）、SQL 1,014 题（71.3% / 67.4% / 50.0%）、Python 971 题（51.0% / 47.3% / 35.6%），依次对应 Qwen-14B / Qwen-7B / DeepSeek-6.7B。Bash 仅 200 题且成功率最低，可用正例仅约 45~67 条，是稳健性隐患。

### 4.3 推理流程（若不训练或重点在推理）

- **模型与输入输出**：同一冻结 LLM，输入"任务描述 + 历史 THOUGHT/ACTION/观测"，输出下一轮 THOUGHT 与单条 ACTION。
- **单步还是多步**：Agent 本体是**多步循环**（THOUGHT→ACTION→观测），终止于 `ACTION: submit` 或 T=10 上限；而**置信度计算本身是单步的**——动作末 token 隐状态已在 GPU 上、尚未过 lm_head，直接 hook 出来跑标准化→PCA→逻辑回归，无需二次前向或多次采样。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| InterCode-Bash | General（Linux shell，非 GUI） | 多轮命令执行、文件操作、进程编排 | 200 | 任务描述 + 纯文本历史观测（无截图） | `THOUGHT / ACTION: <命令 或 submit>` | 任务成功率；置信度 AUROC↑ / Brier↓ / ECE↓ |
| InterCode-SQL | General（MySQL） | 多轮 text-to-SQL（Spider schema） | 1,014 | 问题 + schema + 历史查询结果/报错 | `THOUGHT / ACTION: <SQL 或 submit>` | 同上 |
| InterCode-Python | General（持久 REPL） | MBPP 函数合成 + 单测 + traceback 调试 | 971 | 题目 + 执行反馈 | `THOUGHT / ACTION: python <code> 或 submit <fn>` | 同上 |

三环境 max horizon 均为 T=10；3 模型 × 3 环境共 9 个设置，所有方法共享同一批轨迹与冻结折边界。

### 5.2 实验结果分析

**主结果**（5 折嵌套 CV 的 out-of-fold AUROC / Brier / ECE）：

| 模型-环境（成功率） | Logprob (Cal.) | HTC | LTD | ARP |
|---|---|---|---|---|
| Qwen-14B · Bash (33.5%) | 0.627 / 0.218 / 0.112 | 0.743 / 0.196 / 0.095 | 0.761 / 0.183 / 0.069 | **0.814 / 0.162 / 0.070** |
| Qwen-14B · SQL (71.3%) | 0.624 / 0.197 / 0.018 | 0.743 / 0.176 / 0.052 | 0.771 / 0.169 / 0.044 | **0.842 / 0.144 / 0.055** |
| Qwen-14B · Python (51.0%) | 0.555 / 0.247 / 0.021 | 0.655 / 0.231 / 0.044 | 0.646 / 0.234 / 0.025 | **0.711 / 0.216 / 0.028** |
| Qwen-7B · Bash (30.3%) | 0.510 / 0.216 / 0.052 | 0.575 / 0.221 / 0.107 | 0.629 / 0.212 / 0.109 | **0.704 / 0.190 / 0.048** |
| Qwen-7B · SQL (67.4%) | 0.702 / 0.197 / 0.028 | 0.800 / 0.164 / 0.023 | 0.786 / 0.171 / 0.057 | **0.837 / 0.151 / 0.038** |
| Qwen-7B · Python (47.3%) | 0.509 / 0.249 / 0.047 | 0.684 / 0.222 / 0.020 | 0.659 / 0.229 / 0.028 | **0.718 / 0.214 / 0.024** |
| DeepSeek-6.7B · Bash (22.5%) | 0.482 / 0.175 / 0.006 | 0.698 / 0.165 / 0.075 | **0.727 / 0.162** / 0.093 | 0.650 / 0.167 / 0.042 |
| DeepSeek-6.7B · SQL (50.0%) | 0.454 / 0.251 / 0.042 | 0.726 / 0.211 / 0.041 | 0.624 / 0.239 / 0.039 | **0.788 / 0.187 / 0.029** |
| DeepSeek-6.7B · Python (35.6%) | 0.520 / 0.230 / 0.002 | 0.645 / 0.218 / 0.038 | 0.670 / 0.211 / 0.037 | **0.784 / 0.179 / 0.040** |

- **AUROC**：ARP 相对 Cal. Logprob 提升 0.14~0.29，相对 HTC 提升 0.02~0.10（DeepSeek-Bash 例外，降 0.048）。提升最猛的是成功率最低、最需监控的 Bash（Qwen-14B 0.627→0.814），对"安全监控"叙事有说服力。
- **但 ECE 反而变差**：ECE 最优几乎都是 Logprob（9 个设置里占 6 个），LTD 在 Qwen-7B-Bash（0.109）与 DeepSeek-Bash（0.093）为全场最差。即内部表征改善的是**排序（AUROC）**，不是**概率数值校准（ECE）**；正文只强调 best AUROC/Brier，回避了这点。
- **Ablation 说明了什么**：**本文没有组件消融**，只有三点准消融信息：① 方法递进（Logprob→HTC→LTD/ARP）证明"输出概率 < 轨迹级动态 < 内部表征"的单调收益；② ARP 与 LTD 的对照证明动作级表征优于轨迹几何汇总（8:1）；③ 附录 E.2 声明剔除 submit 是为防 shortcut，但未给出"不剔除时掉多少"的数字。28 维特征中哪一维起作用、k=64 是否最优、是否必须用末层，全部缺失。

---

## 6. 亮点与贡献（Why it matters）

1. **把可靠性监控成本压到近零**：不改 prompt、不做多次采样、不做二次前向，只在动作生成那一刻挂 hook（附录 I 给出 <0.05 ms 论证）。
2. **证明"内部状态 > 表面概率"在多轮 Agent 上成立**：3 模型 × 3 环境一致提升，且在最易失败的环境提升最大。
3. **ARP 配方可复用**：动作末端隐状态 + 剔终止动作 + mean-pool + 折内 PCA + 单调 Platt，换任务只需重定义动作边界。
4. **协议干净**：统一折边界、折内拟合全部预处理、只报 out-of-fold 指标。
5. **给出有价值观察**：成败信号集中在"决定要做什么"的隐状态，而非过程几何，对 step-level 监控有启发。

## 7. 局限与可改进点（个人点评）

- **ECE 退化被回避**：内部方法的 ECE 在多数设置不如最朴素的 logprob，甚至差一个数量级（DeepSeek-Bash：0.006 vs 0.093）。若用置信度做阈值化干预，AUROC 高但 ECE 差意味着阈值难定。
- **"每个设置都更好"靠择优**：DeepSeek-Bash 上单看 ARP 输给 HTC，靠 LTD 才保住"internal 至少一项最好"。
- **任务身份泄漏未被排除**：探针在同一批固定任务上训练与评估，任务难度本身有差异，探针可能学到"这是哪道题"而非"这条轨迹对不对"；缺 leave-task-out 划分折的对照，这是最该补的消融。
- **样本量小且不平衡**：Bash 仅 200 题、DeepSeek 成功率 22.5%（约 45 条正例），Brier/ECE 极不稳定，全文无方差或置信区间。
- **闭源模型完全用不了**：方法依赖残差流，API 模型拿不到隐状态。
- **"零开销"是理论论证而非实测**：实验全用离线重放，无在线端到端延迟与吞吐测量。
- **任务域过窄且无迁移实验**：Bash/SQL/Python 都是文本观测 + 确定性验证的干净场景，无截图、无视觉噪声、无不可逆动作；探针按模型按环境单独训，未验证跨环境迁移，也未测冷启动所需标注量。

## 8. 对我们的启示 / 可借鉴点

**通用方法论**：① 想做失败预警，先问"这个信号在推理框架里是不是本来就有、只是被丢了"——残差流、KV、动作末 token 状态都是免费信号；② 必须区分 ranking（AUROC）与 calibration（ECE），做干预决策应直接优化决策损失。

### 对 GUI Agent 的可借鉴点

1. **动作落地前插置信闸门**。GUI 失败常不可逆（删数据、下单、发消息），ARP 在动作末 token 取隐状态只需 <0.05 ms，适合做成"执行前闸门"：低于阈值就暂停或请求确认，比事后回滚便宜一个量级。
2. **把 LTD 改造成卡死探测器**。GUI 长时程典型失败是重复点同一元素、页面无进展；把 LTD 的路径效率与余弦位移中的隐状态换成"截图 patch 嵌入或 DOM 表征"，即可量化"有没有在前进"。
3. **重定义动作 span 边界**。GUI 动作是多字段输出（动作类型 + 元素/坐标 + 文本），取点落在哪个字段会显著影响探针质量；建议在"元素定位 token 末"与"参数生成完"两处分别探针做对照。
4. **必须做 leave-app / leave-task-out 划分**。GUI 数据集里 app、页面模板、任务族差异极大，探针极易学成"这是哪类 App 的题"；按 app 或 task 划折是判断信号真伪的最低门槛。
5. **冷启动靠环境自带的程序化判定**。WebArena/OSWorld/Android 断言能给出确定性成败标签，可批量标注探针训练轨迹；对拿不到隐状态的闭源 GUI 模型，可退化为"截图嵌入 + 动作文本嵌入"的可见表征探针，但须重验 AUROC。

## 9. 延伸阅读

- Azaria & Mitchell, *The Internal State of an LLM Knows When It's Lying*（EMNLP 2023 Findings）——本文核心动机。
- Zhang et al., *Agentic Confidence Calibration*（arXiv:2601.15778）——HTC 基线出处，最强的外部信号对手。
- Duan et al., *UProp*（arXiv:2506.17419）与 Zhao et al., *Uncertainty Propagation on LLM Agent*（ACL 2025）——轨迹级不确定性传播路线。
- Subramani et al., *MICE for CATs*（NAACL 2025）——同为内部置信，但面向工具调用单轮场景，可对照。
- 本地相关：`2026-09-02_Monitoring_Web_Agents_Without_Internal_Signals`、`2026-09-03_Do_GUI_Agents_Know_When_Not_to_Act`。

---
*解读生成时间：2026-09-11 09:00 ｜ 解读人：WorkBuddy（AI）*
