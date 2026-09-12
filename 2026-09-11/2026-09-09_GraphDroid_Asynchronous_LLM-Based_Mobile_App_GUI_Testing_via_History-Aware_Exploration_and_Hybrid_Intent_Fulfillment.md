# GraphDroid: Asynchronous LLM-Based Mobile App GUI Testing via History-Aware Exploration and Hybrid Intent Fulfillment

> 一句话 TL;DR：把 LLM 的 intent 生成与探索解耦——用 UTG 聚类记忆做异步 intent 合成、简单 intent 交给 EVS 引导的启发式遍历、只有复杂 intent 才调 GUI Agent，41 个 Android App 上代码覆盖率 20.83%、成本不到最强纯 LLM 基线的 1/8。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | GraphDroid: Asynchronous LLM-Based Mobile App GUI Testing via History-Aware Exploration and Hybrid Intent Fulfillment |
| **作者 / 机构** | Xiaolei Li（香港科技大学 / 南方科技大学 / 广州市香港科大霍英东研究院）、Jialun Cao（香港科技大学）、Zhijian Hou（香港城市大学）、Yuzhi Zhao（香港城市大学）、Yepang Liu†（南方科技大学，通讯）、Shing-Chi Cheung\*（香港科技大学，通讯） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-09；arXiv preprint（arXiv:2609.10031v1，cs.SE）；作者自述为 author-created preprint，正式版见 ACM DOI 10.1145/3832783.3837449 |
| **arXiv 链接** | https://arxiv.org/abs/2609.10031 |
| **代码仓库** | ✅ 开源：https://doi.org/10.5281/zenodo.21512256（源码 + benchmark 一体） |
| **数据集地址** | ✅ 同上 Zenodo DOI（含 41 个 App 的评测 benchmark）；App 来源为 LLMDroid / LLMExplorer 公开 benchmark 与 Google Play |
| **类型标签（论文类别）** | `Mobile` `Planning` `Benchmark` |
| **训练方法标签** | —（综述/评测/工程；无训练，直接调用 gpt-4.1 与 UI-TARS-1.5） |
| **关键词** | GUI Testing；LLM Agent；UI Transition Graph；Cluster-based Memory；Asynchronous Intent Generation；Hybrid Intent Fulfillment |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-09_GraphDroid_Asynchronous_LLM-Based_Mobile_App_GUI_Testing_via_History-Aware_Exploration_and_Hybrid_Intent_Fulfillment.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：移动 App GUI 测试中，**需多步动作序列才能触发的复杂功能**难以覆盖，任一步漏掉该功能与下游页面就永远测不到；同时需压低 LLM 推理成本。
- **为什么重要**：复杂功能既是**测试目标本身**，又是**通往下游功能的网关**；测不到「提问」，其提交/编辑/删除页全不可达。
- **现有方法不足**：LLM-augmented（LLMDroid、LLMExplorer）依赖传统算法，无法系统性触发复杂多步功能；Pure LLM-based（GPTDroid、VisionDroid、DroidAgent）有三个结构性缺陷——**历史上下文丢失**（更早历史压成摘要，识别不出其暴露的未覆盖功能）、**同步 intent 生成阻塞探索**、**每步都调 LLM 造成无谓开销**。
- **Research Gap**：补上「保留完整历史 + 不阻塞探索 + 按 intent 复杂度分流」的组合缺口。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把探索历史组织成 UTG 并做**语义/导航双重聚类**，让 LLM 在「功能模块」粒度异步生成 intent；再按预测步数分流——单步交给 EVS 引导的 DFS，多步才给 GUI Agent。

### 3.2 方法总览（Pipeline）

- **输入**待测 App 与预算 `T_max`；**输出**扩展后的 UTG 与 logcat 崩溃列表。
- **Phase 1 Cold Start**：DFS 建初始 UTG，深度约束 `d=3`（三次点击规则），超出即回退；不用 LLM。
- **Phase 2 异步 Intent 生成**：`C ← SpectralClustering(G)`，对每簇**并行**发起 intent 合成查询，后台 DFS 继续探索（从不阻塞）并统一去重。
- **Phase 3 混合履行**：按 `predicted_steps=1` 分简单集（EVS 引导 DFS）与复杂集（逐个交给 GUI Agent）；新状态扩展 UTG，下轮更新聚类记忆。**Phase 2/3 交替至预算耗尽**。
- **关键模块**：①**状态抽象** `Sim = max(J(W_i,W_j), cos(v_i,v_j))`，融合 DOM 的 Jaccard 与视觉余弦，> `τ=0.95` 并入（纯 DOM 会因「视觉同而 DOM 异」失败）。②**谱聚类** `W_ij = Edge + Sim`，兼顾相似与导航连接，簇大小限 `[2,20]`。③**Intent 合成**：prompt 为簇内截图 + 转移边，**已访问 widget 用红框标注**；三步 CoT 输出 JSON。④**去重**：Sentence-BERT 过滤高相似，再由 LLM 删已覆盖/合并同指。⑤**EVS** `= (N_intent − visits) / (N_widget · N_intent)`；>0 点未访问 widget，=0 按 `∝ EVS(s)/dist_G` 导航。⑥**复杂履行**：`θ`-greedy 选 intent，导航后调 **UI-TARS-1.5**。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：①**Cluster-based memory 作为「上下文作用域」**：按功能模块切簇，在模块级中间粒度推理——整图塞一次压垮推理（Variant 2），逐状态 query 又缺跨状态上下文（Variant 3 使预测步数 1.73→1.40）。②**异步 intent 生成**：记忆从历史构建 → 生成不依赖当前状态 → 可与 DFS 并行，是范式解锁而非工程 trick。③**EVS 把 Go-Explore 的 archive 思想适配到 intent 覆盖**，编码「剩余目标 × 已尝试比例」作导航权重。
- **工程组合**：谱聚类、Sentence-BERT、DFS/DroidBot、`θ`-greedy、现成 gpt-4.1 与 UI-TARS-1.5。
- **对性能提升最关键的设计**：①**聚类分区**最关键——Variant 2 全面最差（26.09/15.27/43.1），intent 238→104。②**完整聚类记忆**次之（31.51/16.94/52.1），RQ3 硬支撑：**82.3% 的 intent 来自最近 5 步之外**。③**混合履行两向都不可缺**——Variant 5 涨到 $1.91/6.97s 且三项全降；Variant 6 虽便宜快但 activity/states 掉到 32.15%/63.6。
- **证据不足 / 仅声称有效**：①**履行率只在单 App（Renpho Health）测**（82.4%/72.7%）却论证整个分流策略；失败的 17.6% 简单 intent **静默丢弃不回退**，41 App 累计影响未量化。②**「成本 <1/8」是选择比较对象的结果**：$1.41 vs DroidAgent $11.97 成立，但为 LLMDroid（$0.75）的 **1.9 倍**、LLMExplorer 的 **2.7 倍**。③**去重不彻底**：906 条移除 208 后**仍残留 235 条**（约 34%）。④**12/41 无法插桩**，代码覆盖率只在 29 个上报。

---

## 4. 具体技术细节

### 4.1 模型结构

- **base model 混合使用，均非本文训练**：**生成侧**默认 **gpt-4.1**（统一用于所有基线；同时接收截图与文本，按多模态使用，但未说明视觉编码器细节），RQ5 替换为 **Qwen3-VL-Plus**（MLLM）。**履行侧**为 **UI-TARS-1.5**（GUI 原生 MLLM，含视觉编码器，在原图上 grounding 输出动作）。**辅助侧**为视觉嵌入模型与 **Sentence-BERT**。
- **参数量未报告**：三个 LLM 的规模与上下文长度均缺，嵌入模型型号亦缺。
- **全部冻结、零训练**。RQ5 显示换生成模型差异很小（36.73/20.79/74.8 vs 36.94/20.83/73.5），换履行 Agent 才明显下降（34.54/19.95/68.5）——有效性来自框架而非特定 LLM。

### 4.2 训练流程（若需要训练）

**不训练**：无 SFT/RL/蒸馏/LoRA，也不微调 UI-TARS-1.5 或 gpt-4.1。

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| — | 不训练 | — | — | — | — |

多阶段运行流程属推理控制流而非训练：

| 阶段 | 目标 | 产出的能力 | 数据来源 | 数据形态 | 判据 |
|---|---|---|---|---|---|
| Phase 1 Cold Start | 建初始 UTG | 可达性探索 | App 运行时 | UI 状态 + 边 | 无 loss；DFS 深度 3 |
| Phase 2 异步 Intent 生成 | 识别未覆盖功能 | 功能级语义理解 | 聚类记忆 | 每簇截图 + 转移三元组 | 无 loss；三步 CoT |
| Phase 2 去重 | 消除跨簇冗余 | — | 所有簇 intent | 文本 + SBERT | 无 loss；阈值 + LLM 判断 |
| Phase 3 简单履行 | 覆盖单步 intent | 纯启发式 | UTG + EVS | 状态图 | 无 loss；EVS 采样 |
| Phase 3 复杂履行 | 用 GUI Agent 覆盖多步 intent | 多步执行 | 起始截图 | 截图 + intent | 无 loss；`θ`-greedy，预算 3×步数 |

### 4.3 推理流程（若不训练或重点在推理）

- **推理模型**：gpt-4.1（生成 + 去重）+ UI-TARS-1.5（履行）+ 视觉嵌入模型与 Sentence-BERT。
- **输入 / 输出**：生成侧 = 簇内截图（含红框）+ 转移边 → JSON intent 列表；简单履行 = UTG + EVS → DFS 动作；复杂履行 = 起始截图 + intent 文本 → GUI 动作序列。
- **多步，嵌套两层循环**：**外层** Phase 2 ↔ Phase 3 交替至预算耗尽，每轮 Phase 3 扩展 UTG 后 Phase 2 重建聚类记忆；**内层**简单履行是 EVS 引导 DFS 循环（终止于全 EVS<0.1），复杂履行是逐个 intent 的 `θ`-greedy 循环 + Agent 多步执行（终止于完成或超 3×步数）。Phase 2 按簇并行、与后台 DFS 并发；**无显式反思/重规划**。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| GraphDroid 自建 App Benchmark | Mobile（Android） | 端到端 GUI 探索测试 | **41 个真实 Android App**，覆盖 21 个类别（12 取自 LLMDroid、19 取自 LLMExplorer、10 来自 Google Play） | App 安装包（部分插桩）+ UI 状态 | 探索轨迹（UTG）+ logcat | Activity Coverage / Code Coverage（方法级） / # Discovered States / Cost per App / Time per Action |
| Themis bug benchmark | Mobile（Android） | 崩溃复现 | **52 个可复现崩溃** | App 安装包 | 崩溃列表 | # Crashes Detected |

**评测设置**：每个工具在每个 App 上跑 **2 小时**、重复 **3 次**报平均（RQ2 报去重后 bugs）。29/41 个 App 可插桩，报告 activity + code coverage；其余 12 个只报 activity，故额外引入 discovered states（用 `τ=0.95` 统一去重）。

### 5.2 实验结果分析

**主结果（Table 1，41 个 App，2 小时预算）**

| 方法 | 类型 | Activity Coverage | Code Coverage | # States |
|---|---|---|---|---|
| DQT | 传统（无 LLM） | 27.11% | 16.32% | 42.9 |
| GPTDroid | Pure LLM | 17.15% | 13.10% | 28.5 |
| VisionDroid | Pure LLM | 20.35% | 14.09% | 34.1 |
| DroidAgent | Pure LLM | 24.68% | 15.27% | 39.2 |
| LLMExplorer | LLM-augmented | 26.41% | 16.34% | 44.8 |
| **LLMDroid**（最强基线） | LLM-augmented | **28.36%** | **17.19%** | **48.7** |
| **GraphDroid** | 本文 | **36.94%** | **20.83%** | **73.5** |

- **关键数字**：对最强基线 LLMDroid，activity **+30.3%**、code **+21.2%**、states **+50.9%**；对最强 pure LLM 基线 DroidAgent，分别为 **+49.7% / +36.4% / +87.5%**。
- **成本（Table 4）**：GraphDroid **$1.41/App、5.33s/action**；DroidAgent $11.97（不到其 1/8）、VisionDroid $3.55、GPTDroid $1.29、LLMDroid **$0.75**、LLMExplorer **$0.53**。
- **分来源（Table 2）**：31 个 benchmark App activity 35.35%→45.63%、states 40.6→61.0；10 个 Google Play App activity **6.68%→10.01%**（+49.9%）、states 73.7→112.3（+52.4%）。
- **阈值鲁棒性（Table 3）**：阈值 0.75→0.95 扫描（22.3/32.0/51.6/58.4/73.5），每档均高于所有基线。
- **Bug 检测（RQ2）**：自建 benchmark 共 21 个 distinct bug，GraphDroid 暴露 **19 个**（13 崩溃 + 6 非崩溃），基线 LLMDroid 11、DQT 9、LLMExplorer 7、DroidAgent 6、VisionDroid 3、GPTDroid 1；**7 个此前未知、4 个已修复**。Themis 上检测 **13/52**（DroidAgent 9、LLMDroid 8）；10 小时重跑中两者只能发现其 13 个崩溃中的 5 与 9 个。
- **Intent 分析（RQ3）**：平均 **381 条 intent/App**，过滤 37.9% 后剩 **237 条**；**82.3% 来自最近 5 步之外**；59.3% 单步 / 31.9% 两步 / 8.8% ≥3 步；400 条人工校验**100% 一致**；**49.2% 的状态与 30.2% 的 activity 在履行阶段首次到达**，代码覆盖率中单步 39.4% + 多步 23.6% = **63.0%**。

**Ablation 说明了什么（Table 7）**

| 变体 | Activity Cov. | Code Cov. | # States | Cost($) | Time(s) |
|---|---|---|---|---|---|
| 1 w/o Cluster-based Memory | 31.51% | 16.94% | 52.1 | 1.52 | 5.98 |
| 2 w/o Cluster Partition | **26.09%** | **15.27%** | **43.1** | 1.02 | 5.14 |
| 3 w/o Cross-state Context | 34.61% | 19.38% | 67.5 | **3.55** | 5.42 |
| 4 w/o Async Generation | 29.52% | 16.27% | 46.8 | 1.25 | **8.21** |
| 5 w/o Simple Fulfillment | 27.83% | 15.90% | 46.7 | 1.91 | 6.97 |
| 6 w/o Complex Fulfillment | 32.15% | 18.23% | 63.6 | 1.05 | 4.69 |
| **GraphDroid** | **36.94%** | **20.83%** | **73.5** | 1.41 | 5.33 |

- **六个变体全部劣化**。Variant 2 最惨（−10.85 / −5.56 / −30.4，intent 238→104），证明聚类分区是必需设计；Variant 3 覆盖率仅小降但**成本翻倍到 $3.55**、预测步数 1.73→1.40，说明缺跨状态上下文会让模型退化成低层操作；Variant 4 量化同步阻塞代价（动作时间 5.33s→**8.21s，+54.0%**）；Variant 5/6 双向劣化说明分流是覆盖手段而非省钱；Variant 1 与 RQ3 的 82.3% 互证。

---

## 6. 亮点与贡献（Why it matters）

1. **「上下文作用域」的中间粒度发现最有价值**：整图塞入压垮推理、逐状态查询退化成低层操作且成本翻倍、按功能模块聚类刚好；该结论可迁移到任何用 LLM 处理图结构历史的工作。
2. **异步 intent 生成是记忆设计的范式收益**：记忆从历史构建 → 生成不依赖当前状态 → 可并行；可直接复用「把依赖从 now 挪到 history」来解锁并行。
3. **复杂度分流是双向收益**：两个方向都掉分，说明「简单功能用便宜算法、复杂功能用贵模型」既是省钱手段也是覆盖手段。
4. **EVS 把「还剩多少目标」与「已尝试多少空间」压成标量作导航权重**；实测规模也硬（41 App × 21 类别），**7 个新 bug、4 个已修复**是实际价值的直接证明。

## 7. 局限与可改进点（个人点评）

- **Fulfillment rate 的单 App 验证是最大软肋**：82.4%/72.7% 全来自 Renpho Health 一个 App，却用来论证整个分流策略；失败的 17.6% 简单 intent 按设计静默丢弃、不回退，累计损失未量化。
- **成本叙事的选择性基准**：应表述为「以约 2 倍于轻量方案的成本换取 50% 状态覆盖增量」，而非只挑最贵的 DroidAgent。
- **绝对覆盖率与去重都偏弱**：Google Play 上 activity 仅 10.01%（九成页面未触及）；906 条 intent 移除 208 条后仍残留 235 条重复（约 34%），「237 条/App」被高估约三分之一，冗余 intent 直接消耗预算。
- **12/41 无法插桩是个不小的洞**：缺失的恰是功能最复杂、最有测试价值的商用 App，评测只依赖敏感度更低的 activity 与 states。
- **其它**：Bug 的 10 小时重跑有选择偏倚（只在 GraphDroid 至少发现一个崩溃的 App 上重跑基线）；RQ5 数值与主表不可比（47.3% vs 36.94%）；嵌入模型无型号、LLM 参数量与版本未报告；无反思/重试机制。

## 8. 对我们的启示 / 可借鉴点

**对 GUI Agent 的落地启示（本篇为 GUI 域移动端测试论文，Mobile 场景）：**

1. **给 Agent 找一个「不大不小」的上下文作用域，比堆长上下文更有效**：Variant 2/3 展示了「压垮」与「退化 + 成本翻倍」两种失败；做长程 GUI Agent 记忆时，按功能模块聚类是被实测验证的中间粒度，可迁移到 Mobile 与 Desktop。
2. **把「依赖当前状态」改成「依赖历史」可解锁并行**：online Agent 中规划/检索/总结若依赖当前观测就会阻塞主循环；改成基于历史快照的异步任务，可用极小改动换来显著延迟收益（5.33s vs 8.21s）。
3. **按难度分流必须双向验证**：判据（LLM 预测步数）质量直接决定收益，400 条校验 100% 一致说明它可靠。
4. **EVS 式「剩余目标 × 已探索空间」标量适合做导航优先级**：GUI Agent 长程导航可直接借鉴 `EVS(s)/dist_G`。
5. **覆盖率评测必须报绝对数字**：10.01% vs 6.68% 只写「+49.9%」会严重误导。
6. **GUI Agent 能力上限被外部暴露**：复杂 intent 失败主因是「GUI Agent 动作错误」；OceanEx 那个需按顺序填三个字段再点 Options 才能触发的崩溃，所有基线都因无法在一次访问内完成整段序列而错过。
7. **「探索历史 + 视觉状态抽象」对 Desktop/Web 同样适用**：`max(DOM Jaccard, 视觉余弦)` 解决「视觉同而 DOM 异」。

## 9. 延伸阅读

- **GPTDroid**（Liu et al., ICSE 2024）——pure LLM-based 范式代表，本文三大局限的直接批评对象。
- **VisionDroid**（Liu et al., 2024）——视觉驱动的 pure LLM 基线。
- **DroidAgent**（Li et al., 2024）——最强 pure LLM 基线，「成本低于其 1/8」的参照物。
- **LLMDroid**（2024/2025）——LLM-augmented 最强基线与主要性能对手。
- **LLMExplorer**（2025）——缓解状态空间爆炸的 LLM-augmented 工具。
- **DroidBot**（Li et al., ICSE-C 2017）——本文 Cold Start 与简单履行所用 DFS 基础。
- **Go-Explore**（Ecoffet et al., Nature 2021）——EVS 的思想来源。
- **Themis**（2023）——本文泛化验证所用基准。
- **Beyond Static GUI Agent**（Chen et al., ASE 2025）——同期动态记忆路线。

---

*解读生成时间：2026-09-11 09:00 ｜ 解读人：WorkBuddy（AI）*
