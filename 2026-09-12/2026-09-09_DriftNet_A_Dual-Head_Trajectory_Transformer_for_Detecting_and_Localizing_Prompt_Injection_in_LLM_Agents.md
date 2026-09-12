# DriftNet: A Dual-Head Trajectory Transformer for Detecting and Localizing Prompt Injection in LLM Agents

> 一句话 TL;DR：把已记录的工具调用轨迹当序列读，用一个 <2M 参数的双头 Transformer 在一次前向中同时给出"轨迹是否被劫持"与"每步属于 benign/injection_point/hijacked/failed_injection 中的哪一类"。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | DriftNet: A Dual-Head Trajectory Transformer for Detecting and Localizing Prompt Injection in LLM Agents |
| **作者 / 机构** | Asif Pinjari、Mithun Paul Saint-Germain；School of Informatics, Computing, and Cyber Systems, Northern Arizona University（美国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-09；arXiv preprint（cs.CR） |
| **arXiv 链接** | https://arxiv.org/abs/2609.10892（点击直达） |
| **代码仓库** | ⚠️ 正文未给出 DriftNet 自身代码仓库；仅脚注 1 给出配套基准 AgentDrift 的仓库：https://github.com/Asif-0209/AgentDrift |
| **数据集地址** | AgentDrift 基准（CC BY 4.0 公开释放）：https://github.com/Asif-0209/AgentDrift ；数据集论文 arXiv:2609.06972 |
| **类型标签（论文类别）** | `Benchmark` `General` `Reflection` |
| **训练方法标签** | 监督训练（自建 <2M 参数 trajectory Transformer，冻结句编码器，非 LLM 微调） |
| **关键词** | 间接提示注入、轨迹级检测、步级序列标注、注入点定位、双头 Transformer、Agent 安全 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-09_DriftNet_A_Dual-Head_Trajectory_Transformer_for_Detecting_and_Localizing_Prompt_Injection_in_LLM_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **问题**：注入成功的破坏有固定"形状"——良性前缀、被污染的 observation、之后一串服务攻击者的动作；论文**事后取证式检测 + 定位**：攻击从哪步进入、哪些步被污染、哪些被抵抗。
- **重要性**：只给"轨迹有问题"无法行动；需**回滚到哪、审计哪些、不信任哪个源**，故必须区分"入口"与"被污染区间"。
- **现有方法不足**：(1) **运行时防御**（CaMeL、MELON、AttriGuard、DreamGuard）需**控制运行中的 Agent**，不适用事后取证；(2) **内部探针/注意力定位**（BASIS、AttnLocate）需**白盒权重/激活**，对闭源失效；(3) **轨迹级守卫**（R-Judge、ATBench、AgentDoG）只给整条判决、把被抵抗的注入**折叠进安全类**；有步级输出的（StepGuard、StepShield、TrajAD）**只给一个索引或 span**。
- **Research Gap**：补**输出形态**缺口——首个有监督地**联合输出轨迹判决与四类步级标签**、且**只看日志**。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

每步序列化后用**冻结句编码器**嵌入，拼上四个**身份无关的世界特征**，送入**≤3 层、<200 万参数 Transformer 编码器**，两头共享上下文联合训练。

### 3.2 方法总览（Pipeline）

- **输入**：轨迹 `x = (s_1…s_T)`，`3 ≤ T ≤ 11`，`s_t = (tool_t, thought_t, args_t, obs_t)`；外加**世界上下文 W**（用户、组织、联系人表）——同一动作是协作还是外泄取决于收件人是否在 W 中，**无法仅从步文本恢复**。
- **输出**：轨迹级概率 `ŷ`；每步 `ẑ_t ∈ {B, I, H, F}`：I=注入指令步、H=服务注入目标、F=携带注入但被抵抗、B=benign。
- **模块**：每步按 `TOOL/THOUGHT/ARGS/OBS` 拼接，用冻结的 `all-mpnet-base-v2` 编码为 `e_t ∈ R^768`（预计算缓存）；世界特征由 `args` 对 W 算四个布尔/比例量（读 Agent 做了什么），拼接得 `x_t ∈ R^772`；主干 = 线性投影 + 正弦位置编码 + L 层编码器（4 头、FFN 512）；Head 1 均值池化出轨迹概率，Head 2 做 4 类 softmax。

### 3.3 真正的创新点（去伪存真）

- **真·创新**：(1) **联合输出形态**——首个同时产出"轨迹判决 + 每步四类注入标签"的有监督检测器，`failed_injection` 单独成类，强制区分"注入存在"与"注入成功"；(2) **两个严格定位指标** `EM_I`（注入点精确匹配）、`IoU_H`（劫持区间 IoU）；(3) **可信任协议**：task-disjoint split + 20 组超参扫描 + 测试集**只评一次** + 基线同 split 重训。
- **最关键设计**：**双向序列建模**——surface baseline full hijack recall 0.902，partial 仅 0.111、delayed 仅 0.171，DriftNet 抬到 0.986/0.932；次级是**类别加权** `α = (0.08, 0.76, 0.34, 2.81)`，使占 2.1% 步的 `failed_injection` F1 = 1.000（218/218）。
- **证据不足**：(1) **四个世界特征无消融**（只论证、无数字）；(2) **冻结 vs 微调编码器无对比**；(3) **"小模型替代 LLM 守卫"是跨数据集类比**；(4) **校准只被警告、未解决**（train loss 0.007 vs val loss 0.1）。

---

## 4. 具体技术细节

### 4.1 模型结构

- **参数量与冻结策略**：句编码器 `all-mpnet-base-v2` **完全冻结**、embedding **预计算缓存**；可训练的是投影层 + L 层编码器 + 两个头。`L=2` **1,318,533** 参数，`L=3` **1,845,637**，均低于 200 万，比同类 4B 守卫**小三个数量级**。`d_model = 256`、4 头、FFN 512。语料仅 3–11 步（均值 5.67）。

### 4.2 训练流程

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| **单阶段联合训练**（无多阶段流程） | 一次前向同时产出轨迹级判决与步级四分类 | 从日志序列识别"良性前缀 → 污染读 → 越界写"漂移模式的序列建模能力；区分"注入存在"与"注入成功" | AgentDrift 的 **task-disjoint split**：训练 9,081 / 验证 1,733 / 测试 1,722 条轨迹。全语料 **12,536 条轨迹 / 71,024 条步级标签**，由 Llama-3.3-70B-Instruct 经 AI-VERDE 网关生成，经闭词表结构校验器 + LLM judge 复核 + 人工审计 1,200+ 条，估计标签正确率 99.6% | 每步序列化为 TOOL/THOUGHT/ARGS/OBS 四段拼接字符串 → 冻结编码器得 `e_t ∈ R^768`；对轨迹联系人世界算 4 维 `w_t ∈ R^4`；拼接得 `x_t ∈ R^772`；轨迹长 3–11 步。**label 定义**：轨迹级 `y ∈ {0,1}`（是否被成功注入劫持）；步级 `z_t ∈ {B,I,H,F}`，真值标签串服从基准固定正则文法——良性 `B+`、全劫持 `B+IH+`、部分劫持 `B+IH^{1..2}B+`、延迟执行 `B+IB+HB+`、失败攻击 `B+FB+` | 联合目标 `L = L_traj + λ L_step`，`λ = 1`。`L_traj` = 轨迹级**类别加权二元交叉熵**，正类权重 `w+ = 1.27`。`L_step` = 步级**类别加权交叉熵**，padding 用 ignore index 排除，逆频率权重 `α = (0.08, 0.76, 0.34, 2.81)`。验证 loss 用**完全相同**的加权公式，使训练/验证 loss 可直接比较 |

- **优化**：AdamW + 解耦权重衰减；三角学习率（前 10% warmup 后线性衰减到 0），`η_peak ∈ {1e-3, 3e-4, 1e-4}`；最多 40 轮，验证 F1 早停（patience 8）；batch 32、seed 42、梯度裁剪 1.0。

### 4.3 推理流程

- **输入输出**：冻结编码器 + <2M 主干；输入 = 工具调用日志 + 世界上下文，输出 = 轨迹判决 `ŷ` + 每步标签 `ẑ_{1:T}`。
- **单次前向**，无 ReAct 循环与迭代；阈值 0.5，测试集所有数字来自**唯一一次**运行。
- **部署形态（Fig. 2）**：Agent 执行 → 日志序列化 → 冻结通道嵌入 → 一次前向 → "分诊报告"（哪步进入 I、污染哪些 H、哪些被挡下 F）。它**不能**执行前阻止动作，与运行时防御互补。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| **AgentDrift**（全程用其 task-disjoint split） | General（email、banking、web、coding、medical 五域） | 注入检测（轨迹级）+ 注入点与劫持区间定位（步级） | 全语料 12,536 轨迹 / 71,024 步标签；split 训练 9,081 / 验证 1,733 / 测试 1,722 轨迹（测试含 9,796 步：7,048 benign、775 injection_point、1,755 hijacked、218 failed_injection） | 纯文本：轨迹逐步序列化（TOOL/THOUGHT/ARGS/OBS）+ 4 维世界特征；另需世界上下文（联系人表） | 轨迹级 `{0,1}` + 每步 `{B,I,H,F}` | 轨迹级 P/R/F1；每真实类别 flag rate（attacked 类即 recall，其余为误报率）；步级每类 P/R/F1；**注入点精确集合匹配 EM_I**；**劫持区间 IoU_H** |

语料构成：benign 4,000（31.9%）、attacked 5,536（44.2%）、failed_attack 1,500（12.0%）、hard_negative 1,500（12.0%）；步级 benign 51,639（72.7%）、injection_point 5,536（7.8%）、hijacked 12,349（17.4%）、failed_injection 1,500（2.1%）；攻击覆盖 6 个目标族与 3 种顺从形态。**基线**是刻意粗糙的 **surface baseline**——对 6 个轨迹级特征做 balanced class weight 的**逻辑回归**，无序列建模与语义嵌入；stratified split 公开值 recall 0.554 / F1 0.647，本 split 重训后 P 0.816 / R 0.579 / F1 0.678。

### 5.2 实验结果分析

**主结果（测试集，仅评一次）**：

| 指标 | Surface LR | DriftNet |
|---|---|---|
| 轨迹级 P / R / F1 | 0.816 / 0.579 / 0.678 | **0.982 / 0.985 / 0.983**（TP 763 / FP 14 / FN 12 / TN 933） |
| attacked flag rate（= recall） | 0.579 (449/775) | **0.985 (763/775)** |
| benign 误报率 | 0.090 (47/525) | **0.015 (8/525)** |
| failed_attack 误报率 | 0.170 (37/218) | **0.000 (0/218)** |
| hard_negative 误报率 | 0.083 (17/204) | **0.029 (6/204)** |

**按顺从形态的攻击 recall**：surface LR full hijack 0.902，partial 仅 **0.111**、delayed 仅 **0.171**；DriftNet **0.998 / 0.986 / 0.932**（recall 升、两类误报同时降）。数据集论文另称约 **45%** 攻击**不带**表面线索。

**步级与严格定位**：步级（9,796 步）benign F1 0.994、injection_point **0.981**、hijacked **0.983**、failed_injection **1.000**（218 步全对）。定位（775 条被攻击轨迹）：注入点**精确匹配 98.7%**、劫持区间**平均 IoU 0.979**（97.0% 完美恢复）；within-one-step = exact-match。按形态：full 99.8% / IoU 0.998；partial 98.6% / 0.964；delayed **94.9% / 0.936**。banking recall 最低（0.955，占 12 漏检中的 5 个）。

**Ablation**：**20 组随机超参扫描**（lr/dropout/wd/宽度 {128,256}/深度 {1,2,3}）**全部落在 0.9827–0.9934 验证 F1 内，跨度仅 0.011**，前五名只差 **0.0014**。**surface baseline 同 split 重训**——F1 从 0.678 到 0.983。

**错误分析**：26 个错误逐条读：12 个漏检（8 delayed、3 partial、1 full）**自信而非边缘**（11/12 <0.12），其中 11/12 被劫持动作**无世界特征签名**，**9/12 的 injection_point 标注无可读注入指令**——疑为**语料残留噪声**（约 0.4%）；14 个误报（8 benign + 6 hard negative）同样自信（12/14 >0.99）。

---

## 6. 亮点与贡献（Why it matters）

1. **把"检测"重定义为"分诊"**：显式分开入口点（I）、污染区间（H）、被抵抗注入（F），一个判决对应三个运维动作（回滚、审计、隔离源）。
2. **把"注入存在 ≠ 注入成功"变成可训练类别**：`failed_injection` 单独成类并给 2.81 逆频率权重，最终 218/218 全对、轨迹级 0 误报——稀有安全类须显式建模。
3. **用"只读日志"的弱假设换通用性**：不需权重、激活、重执行，对**闭源 Agent**、**事后取证**、**任何有工具调用日志的系统**均适用。

## 7. 局限与可改进点（个人点评）

- **轨迹级 0.983 的成色被论文自曝的语料伪影稀释，且未定量扣除**：Limitations 坦白——world identity 到多数类标签的**查表**在 stratified split 上达 86.1%，本 task-disjoint split 上 **86.9%**（多数类基线 55.0%）。四个世界特征**按构造免疫**，但**冻结 embedding 读的是含姓名、公司名、地址的原始步文本**。定位指标 `EM_I`/`IoU_H` 不受影响，但**轨迹级 F1 与 recall 未排除这条捷径**，只写进 Limitations、**未做测量界定**。
- **无组件消融**：四个世界特征、冻结 vs 微调编码器、单头 vs 双头、`λ` 与类权重 `α` 全无敏感性分析；20 组扫描只覆盖 lr/dropout/wd/宽度/深度，**恰好排除声称的创新点**。
- **12 个漏检里 9 个标注本身无注入指令，对 benchmark 可信度的质疑比论文承认的更大**：26/1722 ≈ **1.5%** 总错误率与 0.4% 估计标签错误率间有未解释的差；这 9 条标签若错，**其步级标签（I、H）也错**。

## 8. 对我们的启示 / 可借鉴点

- **"日志即接口"引入 Agent 安全/监控层**：有工具调用日志 + 世界上下文即可步级判断，作为**不侵入 Agent 实现**的旁路审计组件，适合第三方/闭源。
- **稀有安全类须显式建模 + 逆频率加权**："先加权、再验证测试集上是否真学会"可搬到任何含"攻击被挡下"类的数据集。
- **指标要能暴露"差一步"**：per-class F1 掩盖稳定偏移，`EM_I`/`IoU_H` 直接暴露；步级标注应默认加"精确匹配"指标。

### 对 GUI Agent 的可借鉴点

1. **轨迹序列化格式可直接迁移**：GUI 每步同为 `(action, thought, args, obs)`，可套用"冻结句编码器 + 轻量 Transformer + 双头读出"做**注入检测 / 劫持定位 / 风险步标注**。
2. **"世界特征"在 GUI 场景更丰富且近零成本**：URL 域名是否白名单、包名是否授权、剪贴板读写、路径是否越出工作目录、是否触发权限弹窗。
3. **把"Agent 自己挡住了攻击"当独立正例建模**：GUI 常遇恶意弹窗、诱导文案、伪造确认框，"看到但没点"与"没看到"语义完全不同，应单独成类加权。
4. **轻量可私有化部署对 GUI 尤其合适**：GUI 每步涉及截图/无障碍树，监控若调 LLM 成本与延迟不可接受；"冻结编码器 + 1.8M 主干 + 单次前向"说明**监控不必是 LLM judge**。
5. **"分诊报告"优于"安全分数"**：输出"回滚到第 k 步 / 审计第 m..n 个动作 / 不信任第 j 个来源"，而非 0.97 置信度。
6. **警惕 benchmark 的"世界身份"捷径**：86.9% 查表准确率警告——轨迹残留标识串会让模型学"这是哪个世界"而非"行为是否越界"；构建 GUI 轨迹数据集应**做世界/应用留出划分**、报告"查表基线"。

## 9. 延伸阅读

- **配套基准**：AgentDrift（arXiv:2609.06972，含"世界身份规律性 86.1%"伪影测量）。
- **步级守卫**：StepGuard（arXiv:2608.24777，4B、RL，本文最主要对照）；StepShield（arXiv:2601.22136）；TrajAD（arXiv:2602.06443）；TraceSafe（arXiv:2604.07223）。
- **轨迹级守卫**：ATBench（arXiv:2604.02022）；AgentDoG（arXiv:2601.18491）；BASIS（arXiv:2608.08027）；AttnLocate（arXiv:2608.24022）；When AUC 0.998 Is Not Enough（arXiv:2606.22864）。
- **运行时防御**：MELON（ICML 2025）；CaMeL（arXiv:2503.18813）；AttriGuard（arXiv:2603.10749）；InjecAgent（ACL 2024 Findings）；Agent Security Bench（ICLR 2025）；AgentDojo（NeurIPS 2024 D&B）；Kill-Chain Canaries（arXiv:2603.28013）。

---
*解读生成时间：2026-09-12 ｜ 解读人：WorkBuddy（AI）*
