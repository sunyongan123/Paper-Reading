# When Synthetic Data Hurts: On Catastrophic Forgetting in Skill Retrieval for LLM Agents

> 一句话 TL;DR：在 34,396 个技能的 Agent 技能路由上证明——合成数据微调会提升分布内检索却引发灾难性遗忘，加持续学习式正则可在保住 OOD 的同时把分布内再提升 13.98%。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | When Synthetic Data Hurts: On Catastrophic Forgetting in Skill Retrieval for LLM Agents |
| **作者 / 机构** | Syed Shariyar Murtaza、Yifan Nie、Utkarsh Soni、Eugene Wen、Arvid Frydenlund；Manulife（加拿大，Toronto, 200 Bloor St E） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-09；EMNLP 2026 Industry Track（Oct 26–29, 2026）；arXiv:2609.10750v1 [cs.IR] |
| **arXiv 链接** | https://arxiv.org/abs/2609.10750（点击直达） |
| **代码仓库** | ✅ GitHub：https://github.com/manulife-ai/emnlp2026/ |
| **数据集地址** | 技能库来自公开站点 skillhub.club 与 skills.sh（Liu et al. 2026 收集并发布检索服务）；真实任务来自 SKILLSBENCH（benchflow-ai 公开 GitHub 仓库）与 TERMINAL-BENCH 2.0；合成数据与 273 对监督集未公开 |
| **类型标签（论文类别）** | `Distillation` `General` |
| **训练方法标签** | `SFT (LoRA)`：Qwen3-Embedding-0.6B 用 InfoNCE 微调 + Qwen3-Reranker-0.6B 用 listwise loss 微调；叠加持续学习式防遗忘正则 `Embedding Anchor` / `LwF` / `EWC` / `L2-init` |
| **关键词** | Skill Retrieval；Synthetic Data；Catastrophic Forgetting；LoRA；Continual Learning |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-09_When_Synthetic_Data_Hurts_On_Catastrophic_Forgetting_in_Skill_Retrieval_for_LLM_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **问题**：Agent 运行时从 34,396 条技能库检索"技能"（agent-readable 过程说明）注入上下文，技能选择成瓶颈。论文问：**用合成数据微调检索器/重排器，是否会破坏真实与 OOD 检索能力？**
- **重要性**：技能检索的真实监督本质是**多正样本**（一个任务可能同时依赖模糊匹配、数据库检索、PDF 读取、Excel 分析）；若不还原多正结构，微调会把排序拧向合成分布——最激进配置下 OOD recall 从 0.850 掉到 0.650。
- **现有不足**：Zheng et al. 2026 报告合成数据有增益，但**有条件的**：对轻量组件有益，对 encoder/reranker 微调可能有害；灾难性遗忘解法（LwF、EWC、L2-init）**从未在"合成数据驱动的 Agent 技能检索"这一部署规模下被验证**。
- **Research Gap**：在真实规模（34,396 技能、109 个 verifier 打分任务、约 1K trials）把该遗忘确立为**部署级失效模式**，并给出 benchmark 与止损配方。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

真实执行 harvest + 单正/多正两条合成轨道，LoRA 微调 0.6B 检索器与重排器，并比较四种防遗忘正则，以同时保住 OOD 与提升分布内。

### 3.2 方法总览（Pipeline）

- **输入**：34,396 条技能的 SKILL.md 目录与元数据；真实任务 + dockerized 环境 + verifier。**输出**：fine-tuned Qwen3-Embedding-0.6B + Qwen3-Reranker-0.6B + 分数融合栈。
- **模块**：①检索服务：metadata/full content 两粒度 + BM25/4B/RRF hybrid。②真实 harvest：Harbor 跑 SKILLSBENCH（84）与 TERMINAL-BENCH 2.0（89）→ 1,423 trials / 173 任务、1,116 次产出分数（307 失败）、861 次含注入技能；按 (task, skills) 聚合 `SR = n_passed/n_total`，均值 > 0.5 记正样本 → **109 个有奖励信号任务、75 个含正技能任务、273 对正样本（3.64 技能/任务）**。③Track A（单正）：1,772 个 anchor skills，GPT-5 3-shot 生成 200–600 词任务（Jaccard ≥ 0.70 拒泄漏）→ 1,669 任务；每任务 9 技能候选（1 gold + 4 near rank 2–5 + 4 distractor rank 11–50），soft_reward 用 bucket 条件 Beta（gold `Beta(0.74,0.23)`、near `Beta(4,3)`、distractor `Beta(1,9)`），> 0.5 判正 → **5,463 正对（1,331 gold / 4,122 near / 10 distractor）、覆盖 4,486 个唯一技能**。④Track B（多正）：真实正样本分 Tier-A（SR ≥ 0.9 且 ≥2 trials，权重 1.0）与 Tier-B（SR > 0.5，权重 0.7）；合成正样本走改写（约 20 种措辞）与 skill-first（1–4 个 co-used 技能）；每正技能配 7 负样本（3×Qwen3-4B top-50、2×语义相似未使用 cos≥0.40、1×BM25 top-50、1×随机），经 LLM judge 过滤 → **13,271 训练行（157 真实 / 4,256 改写 / 8,858 skill-first）+ 556 验证（20 真实）+ 78 真实测试 + 2,414 合成测试查询**。⑤训练与融合：检索器 LoRA + InfoNCE、重排器 LoRA + listwise loss + 四种正则；`s = α·s_retr + (1−α)·s_rerank`，可选叠 RRF。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：无新算法，新的是**问题定位与实证设计**：把"合成数据引发的灾难性遗忘"确立为 Agent 技能检索的部署失效模式，并以单正 Track A / 多正 Track B 两条轨道使"监督结构"成为可控自变量。
- **工程组合**：InfoNCE、LoRA、listwise loss、RRF、BM25、embedding anchor、L2-init、EWC、LwF 全是既有技术，价值在于放进**同一保守 LoRA 配方**横向比较。
- **对性能最关键的设计**：**LoRA 的 rank 与投影范围，而非正则项权重**——`r` 是"性能–保持"权衡的首要因素，`λ` 次要；低秩保守更新（r=8, α=16, attn-only, lr=5e-6）种子敏感性最小。四种正则差异几乎可忽略（Config A 下 Ring 2：0.6083 / 0.6077 / 0.6067 / 0.6076）。
- **证据不足 / 仅声称有效**：①Track B 三处质量门失效：**soft_reward 分布差异显著、正技能覆盖稀疏、两个 LLM judge 一致性仅中等**。②四种正则"差异可忽略"在 Ring 1（n=21）/Ring 3（n=10）上几乎无统计功效。③OOD 的"保持"可能只是**"学得少所以没忘"**——保守更新也放弃部分合成增益（Config B/C 的 Ring 2：0.6083→0.5685/0.5703）。④漂移机制诊断无表示层证据。

---

## 4. 具体技术细节

### 4.1 模型结构

- **检索器**：`Qwen3-Embedding-0.6B`（bi-encoder，纯文本，无视觉编码器），LoRA 微调；`Qwen3-Embedding-4B` 作 frozen 基线与负样本来源。**重排器**：`Qwen3-Reranker-0.6B`（cross-encoder），LoRA 微调。
- 全部用 HuggingFace PEFT LoRA + AdamW 在**单张 H100** 上训练，checkpoint 按验证集选择；另建 FAISS Flat-IP 索引覆盖全部 34,396 技能。Track A 用 Azure GPT-5；Track B 经 NVIDIA Data Designer（generator Claude Opus 4.7、judge GPT-5.5、audit Claude Opus 4.8）。

### 4.2 训练流程（若需要训练）

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| Stage 1：Track A 检索器 LoRA | 任务对齐到单个 gold 技能 | 任务→技能语义检索（单正） | GPT-5 生成的 1,669 个任务 + hybrid 构造的 9 技能候选 | 14,687 对 (task, skill)，5,463 正；候选 = 1 gold + 4 near(rank 2–5) + 4 distractor(rank 11–50) | InfoNCE；LoRA r=16, α=32 |
| Stage 2：Track B 检索器 LoRA | 还原真实部署的多正监督结构 | 多正检索 + 技能共现建模 | 13,271 训练行（157 真实 / 4,256 改写 / 8,858 skill-first） | (task, {skill1..skilln}) 行；每正技能配 7 负样本（3×Qwen3-4B top-50、2×语义相似未使用 cos≥0.40、1×BM25 top-50、1×随机）；正样本带权重 w（Tier-A 1.0 / Tier-B 0.7）缩放 loss 贡献 | 加权 InfoNCE。激进：all-projection、lr=2e-5、r=16/32；保守：attn-only、r=8, α=16、lr=5e-6 |
| Stage 3：防遗忘正则（叠加于 Stage 2） | 获得合成增益同时不破坏 frozen 模型的 OOD 行为 | 稳定性与可塑性的平衡 | 批内正样本 / anchor pool（EWC 用对角 Fisher 在 anchor pool 估计） | — | 四选一：**Embedding Anchor** `L_anchor = (1/\|B+\|)Σ_{s∈B+}‖f_θ(s) − f_θ0(s)‖²`，总 loss `L_InfoNCE + λL_anchor`（λ=0.1）；**L2-init** `(1/P)Σ(θi−θ*i)²`；**EWC** `(1/P)ΣFi(θi−θ*i)²`；**LwF** `T²·KL(p_t‖p_s)`，T=2.0 |
| Stage 4：Cross-encoder 重排器 LoRA | 检验合成监督能否从 bi-encoder 迁移到 point-wise 打分 | 候选列表细粒度相关性排序 | 在 Stage 3（conservative anchor λ=0.1）检索器上取 top-30 候选池 | 每条实例 = 20 项候选列表（保留全部正样本，其余均匀采样负样本补足） | listwise softmax cross-entropy；可选叠 **listwise LwF**：`KL(σ(s_frozen/T)‖σ(s_tuned/T))`，T=2，λ=0.1 |

> 量级：Track A ≈ 1.5 万对（14,687，正 5,463）；Track B ≈ 1.4 万行（13,271）。负采样两轨不同——Track A 用 rank 分桶 + Beta 软奖励，Track B 用"四路混合"7 负样本。

### 4.3 推理流程（若不训练或重点在推理）

两阶段检索：①fine-tuned anchor-regularized `Qwen3-Embedding-0.6B` 在 34,396 技能上语义检索（FAISS Flat-IP）；②fine-tuned `Qwen3-Reranker-0.6B` 对 top-30 候选 listwise 重排；③按 `s = α·s_retr + (1−α)·s_rerank` 融合，最优 α 在 0.45–0.55；④可选叠 BM25 的 RRF——**但论文报告 RRF 未提升重排器天花板**，故最终栈不含 BM25。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| Track A 评测池 | General（Agent 技能路由） | 技能检索（单正为主） | 75 任务 / 273 正样本对；5-fold task-disjoint CV（约 15/折） | 任务描述文本 → 技能库 | 技能排序列表 | HIT@10、RECALL@10 |
| Track B 三环（Ring 1/2/3） | General | 技能检索（多正） | Ring 1 = SKILLSBENCH 真实 n=21；Ring 2 = 合成留出技能 n=2,414；Ring 3 = TERMINALBENCH2 真实 n=10 | 同上 | 同上 | RECALL@10（含 10,000 次 paired bootstrap 95% CI） |
| 重排器评测 | General | 列表重排 | 真实 75 任务（OOD）+ 合成 1,056 任务（in-distribution） | 任务 + top-30 候选 | 重排后技能列表 | h@1、h@5、MRR |
| BEIR（外部泛化） | General | 通用检索（OOD 补充） | 6 个 BEIR 数据集 | query-document 文本 | 排序列表 | Recall@10、MRR@10 |

> Ring 1/Ring 3 真实数据视为 **OOD**：或排除在微调外，或在以合成为主的训练数据中仅占极小比例（13,271 行里仅 157 行真实）。

### 5.2 实验结果分析

- **Track A（表 1）**：frozen 0.6B 与 frozen 4B+BM25+RRF 打平（HIT@10 = 0.907、RECALL@10 = 0.604）。Synth-only 退化（HIT@10 −0.040、RECALL@10 −0.026）；Real-only 持平或略优（RECALL@10 +0.003）；Real+synth 的 HIT@10 持平但 RECALL@10 比 Real-only 低 0.021。HIT@10 不变而 RECALL@10 下降，即**部分 rank 重排**。
- **Track B（表 2）**：激进配方（r=16, α=32, Config A 全混合）合成环上过拟合（Ring 2 0.577 vs frozen 0.534），真实数据上崩塌（Ring 1 0.507 vs 0.549、Ring 3 0.650 vs 0.850）；r=32/α=64 同样（0.498 / 0.566 / 0.700）。BM25 最差（0.398 / 0.160 / 0.350）。**OOD recall 0.850 → 0.650 即此处**。
- **防遗忘正则（表 3）**：Config A 下 Cons. anchor / LwF / EWC / L2-init 的 Ring 1 全为 0.540（≈ frozen 0.5486）、Ring 3 全为 0.8500（= frozen），Ring 2 分别 0.6083 / 0.6077 / 0.6067 / 0.6076。相对 frozen 0.6B（Ring 2 = 0.5337），**提升 (0.6083 − 0.5337)/0.5337 = 13.98%**。
- **效应量与显著性（Appx. D）**：Ring 1（N=21）Δ = −0.009；Ring 3（N=10）Δ = 0.0，TOST 确认等效。激进 full-projection 负向位移明显（Δ = −0.051 至 −0.250），r=16/α=32 在 Ring 3 上显著退化（0/4/6，Δ = −0.400，p = 0.0312）。
- **重排器（表 4）**：真实 75 任务上检索器单独 h@5 = 0.840 / MRR = 0.735；与 frozen reranker 融合（α=0.55）→ 0.880 / 0.773；LoRA reranker + 融合 → **0.893 / 0.781**；LoRA reranker + listwise LwF（λ=0.1）合成分布内最佳 **h@5 = 0.861 / MRR = 0.744**。λ=0.3 时真实 h@5 升到 0.867 / MRR 0.765，合成降到 0.831/0.723。
- **Ablation 汇总**：①四种正则差异可忽略，**"保守"本身比"用哪种保守"更重要**；②rank 与投影范围第一，λ 次要；③低秩（r=8）种子敏感性最小；④BEIR 六个数据集上防遗忘方法保持或略升，激进微调下降；⑤BM25 的 RRF 无增益。

---

## 6. 亮点与贡献（Why it matters）

1. **把"合成数据的代价"变成可测**：给出反面证据与量化边界（0.850→0.650）；合成监督的有效性取决于是否还原真实评测的多正结构。
2. **给出可部署止损配方**：低 rank、attn-only、小学习率 + anchor 正则，不需 Fisher 矩阵或 teacher 前向。
3. **效应量与反直觉结论**：小样本用 10,000 次 paired bootstrap 与 TOST；0.6B 检索器能与 4B + BM25 + RRF 打平，说明**监督质量比规模更重要**。

## 7. 局限与可改进点（个人点评）

1. **"等效"结论建立在统计功效不足的土壤上**：Ring 1 仅 21 任务、Ring 3 仅 10 任务，单任务波动即可让 RECALL@10 变动约 5%，更准确说是"未观测到差异"。
2. **最大未解问题被绕过**：保守更新在 Config B/C 下把 Ring 2 从 0.6083 压到 0.5685/0.5703，稳健性有一部分是**用可塑性买来的**；且未测端到端成功率，13.98% 的排序提升是否转化为 SR 提升未回答。
3. **soft_reward 分布不匹配是根本性漏洞**：整条链路依赖 soft_reward 作监督，会把模型推向"合成分布最优"；更干净的方案是把它只当排序监督、真实二值标签作校准锚点。
4. **闭源依赖与场景局限**：generator/judge/audit 全为闭源模型，未放出合成数据与 273 对监督集；作者声明全英文、纯文本，结论外推需重验。

## 8. 对我们的启示 / 可借鉴点

- **合成数据必须做"遗忘压力测试"**：任何"造数据 → 微调检索/排序/路由模型"的流程都应固定加一组 OOD holdout，微调前后做 paired 逐任务对比 + 效应量检验；只看 in-distribution 指标会高估收益。
- **优先调"约束强度"，再调"正则种类"**：r、投影范围、lr 是第一杠杆，λ 第二；先把 rank 压到 8、只训 attention、lr 降到 5e-6。
- **警惕"监督结构不匹配"与指标自嗨**：下游评测是多正，训练数据就必须是多正；且应打通"检索质量 → 任务成功率"链路，BM25 的 RRF 也无增益。

### 对 GUI Agent 的可借鉴点

1. **"元素检索/控件路由"存在同构遗忘风险，且更严重**：GUI Agent 常从上千 DOM 节点或数百 API 中选目标，用合成"指令 → 控件"数据微调 grounding/检索模型同样会分布内提升、OOD 退化；Ring 结构可映射为 Ring 1 = 真实应用 holdout、Ring 2 = 合成留出界面、Ring 3 = 另一来源真实应用。
2. **GUI 的多正结构更极端**：一个任务常有多个可行路径与有效控件，合成数据极易退化为"一条指令一个唯一答案"；故合成数据必须显式构造多正。
3. **"低秩保守微调"是 GUI 适配的安全起点**：`r=8, α=16, attn-only, lr=5e-6` + embedding anchor。
4. **把"真实执行验证"当监督来源**：本文最有价值的数据是 861 次带注入技能的真实 trial（含 307 次失败），仅筛出 273 对可靠正样本；对 GUI 的等价物是"真实环境的成功轨迹"——**少而真 + 保守微调，可能优于大量合成 + 激进微调**。
5. **把"遗忘压力测试"写进上线**：每次更新 LoRA 或检索索引后自动跑固定 OOD 应用集合，检查是否出现"top-1 还在但 top-k 变差"——GUI 上即"以前能点的按钮点不到了"。

## 9. 延伸阅读

- **Liu et al. 2026** — 技能库与检索服务来源（skillhub.club / skills.sh）；**Zheng et al. 2026** — 报告合成数据对技能路由有增益，本文直接反例。
- **Kirkpatrick et al. 2017, EWC** / **Li and Hoiem 2018, LwF** / **Li et al. 2018, L2-init** — 三种防遗忘正则；**Hu et al. 2022, LoRA** — 微调载体；**Cao et al. 2007, Listwise LTR** — 重排 loss；**van den Oord et al. 2018, InfoNCE** — 对比损失。
- **Dai et al. 2022 (Promptagator) / Bonifacio et al. 2022 (InPars) / Gao et al. 2023 (HyDE)** — LLM 生成合成监督做低监督检索的标准范式，正是本文质疑的路线。
- **Thakur et al. 2021, BEIR** — 异构检索基准；**Lovon-Melgarejo et al. 2021** — 神经排序的灾难性遗忘；**Terminal-Bench 2.0 / SkillsBench / Harbor Framework 2026** — 执行验证环境。

---
*解读生成时间：2026-09-12 ｜ 解读人：WorkBuddy（AI）*
