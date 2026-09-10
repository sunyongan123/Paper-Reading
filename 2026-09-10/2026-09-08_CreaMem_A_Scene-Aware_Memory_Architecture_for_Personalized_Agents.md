# CreaMem: A Scene-Aware Memory Architecture for Personalized Agents

> 一句话 TL;DR：按用户"生活场景"分区记忆，并让每条经历同时以"时间线事件 + 场景特质"双编码落盘，检索时均衡采样 + RRF 融合，多跳/时序推理显著提升。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | CreaMem: A Scene-Aware Memory Architecture for Personalized Agents |
| **作者 / 机构** | Qixuan Sun、Yue Que、Bowei He、Jin Guo\*（通讯）、Dihang Yang、Wenchang Situ、Chen Ma\*（通讯）；香港城市大学（中国香港）、MBZUAI（阿联酋）、AgentWoods Inc.（美国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-08；arXiv preprint（cs.CL） |
| **arXiv 链接** | https://arxiv.org/abs/2609.08550（点击直达） |
| **代码仓库** | ⚠️ 正文声明"已发布到公开 GitHub 仓库"，但未给出具体 URL，待核实 |
| **数据集地址** | LoCoMo（arXiv:2402.17753）、LongMemEval-S（arXiv:2410.10813），均为公开 benchmark |
| **类型标签（论文类别）** | `General` `Reflection` `Benchmark` |
| **训练方法标签** | —（工程/系统架构，无模型训练） |
| **关键词** | 长期记忆、场景感知、个性化 Agent、双编码、混合检索、多跳推理 |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_CreaMem_A_Scene-Aware_Memory_Architecture_for_Personalized_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：面向通用（非 GUI）个性化 LLM Agent 的长期记忆组织与检索——如何让跨周/跨月的交互累积"记得住、也检得出"。
- **为什么重要**：作者的核心判断是"光 retention 不够，记忆的**组织结构**决定信息能否被检索到"，这是维持跨会话连贯性的中枢设计问题。组织差，长对话里的关键事实会被无关条目淹没。
- **现有方法不足**：作者把既有工作分三类并逐一批评——①扁平存储（MemGPT、MPNet、Contriever、MPC）用无差别池做 RAG，无关条目争抢检索位；②施加结构（RecurSum 摘要树、SeCom 话题分段、RAPTOR 多层摘要、HippoRAG 2 知识图谱）仍局限在**单一记忆池**，不同场景内容仍互相竞争；③记忆分区（MemoryOS 按时间尺度、MIRIX 按认知功能）只是换轴切，**未按"生活场景"组织**，同层内仍存在跨场景干扰。
- **Research Gap**：两点被系统性忽视——①**缺乏场景意识**（unrelated scenes 共享检索空间 → 撑大搜索空间 + 跨场景干扰）；②**单视角编码**（每条经历只从一个角度写，难以召回同一事件的互补侧面）。这两点恰好对应认知心理学的 Conway 自传体记忆（按人生情境组织）与 Tulving 双重记忆系统（情节/语义互补编码、回忆时协同），作者将其作为两条设计公理。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把记忆沿"Life / Work / Interest"三个生活场景分区，并对同一经历做"情节条目 + 场景特质条目"双编码，检索时按记忆均衡采样 + RRF 全局融合。

### 3.2 方法总览（Pipeline）

- **输入**：一段（批处理的）对话消息或一条用户问题；**输出**：注入 prompt 的检索上下文 + 最终回答。
- **模块**（写入侧）：`Meta Memory Manager` 路由 + `Episodic Memory` + 三个 `Life Scene Memories`（Life/Work/Interest）+ `Core Memory`（跨场景静态画像）。
- **模块**（检索侧）：`Planner LLM` 选记忆 + 出关键词 → 每记忆并行 BM25 与 embedding 余弦 → 合并去重 → RRF 全局重排取 top-N → 注入 prompt；Planner 判"上下文不足"再触发每记忆 3 条的补查。
- **连接（数据流）**：写入侧"宽进严筛"——分类不确定时多触发记忆组件，避免静默丢信息；检索侧"每记忆贡献等量候选"——防止单记忆垄断候选池。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：①**按生活场景分区**（Life/Work/Interest + 独立 Episodic）是此前工作未做过的组织轴；②**双编码**（同一经历既写时间线条目、又写场景内特质条目，且特质强制留在场景内）。
- **工程组合**：BM25+embedding 混合检索、RRF、SQLite 存储、LLM 抽取与合并，均为现成组件，本身不新，组合有效。
- **对性能提升最关键的设计**：ablation 证明增益主要来自**场景感知 + 双编码**（Full System 比 Episodic-only 高 9.2 点），而 RRF 的贡献几乎为零（附录：RRF vs BM25-only 仅 +0.06 / −0.91 点），均衡采样 vs 动态分配也只差 +0.26 点。**即"组织方式"是主因，"融合/分配技巧"是次要**。
- **证据不足 / 仅声称有效**：覆盖率（GPT-4o 71.00%）与路由准确率（exact 67.94%）其实不高，说明"三分类天然覆盖对话内容"这一前提只是**弱成立**，作者用"人工评分 80–90%"交叉背书才勉强撑住。

---

## 4. 具体技术细节

### 4.1 模型结构

- **base model**：GPT-4o-mini 统一作 backbone（Meta Memory Manager、各记忆 extractor、Planner LLM 全用它），temperature=0；**无微调、无训练**，纯 LLM-in-the-loop 系统。
- **embedding**：text-embedding-3-small；存储为本地 SQLite 表。
- **judge**：GPT-4o 作 4o-Judge；ablation 另用 Gemini-3 作 backbone+judge 复跑；附录还验证了 Qwen3.6-35B-A3B。
- **条目 schema**：Episodic 条目为七元组 `(eid, 时间戳, actor, 事件类型, 一句话摘要, 完整上下文, 摘要嵌入, 详情嵌入)`；场景特质条目为四元组 `(sid, 特质内容, 重要度 w∈[0,1], 内容嵌入)`。

### 4.2 训练流程

无训练。写入是纯 LLM 抽取流程：Meta Memory Manager 判定 R(m)⊆{Core,Episodic,Life,Work,Interest} → 各 extractor 出候选条目 → 与最相似既有条目比对后决定"合并 or 新插入"（SQLite）。Core Memory 是自由文本块，达 90% 字符上限（L=2000）时 LLM 自动压缩。批处理：累积 3 条用户消息触发一次抽取。

### 4.3 推理流程（重点）

多步、带条件补查的检索推理：

1. **Planner 选路**：`(K, R) = f_plan(q)`，产出 3–6 个关键词 + 待查记忆子集（解析失败则回退全量、以问题本身作关键词）。
2. **每记忆双路检索**：BM25 与 `cos(vq, v_i)` 各取 top-k；每记忆贡献**等量候选**，防垄断。LoCoMo 上 k=20、LongMemEval-S 上 k=40。
3. **全局融合**：合并去重后按 RRF（k0=60）重排取 top-N 注入 prompt。
4. **充分性补查**：Planner 判上下文不足时，再对所选记忆各补 3 条，多一轮检索。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| LoCoMo | General（非 GUI 对话） | 长期记忆 QA（Single-hop/Multi-hop/Open-domain/Temporal） | 10 段约 300 轮、每段约 9K token，1,540 题 | 纯文本对话 | 文本答案 | 4o-Judge 准确率、F1、BLEU-4、ROUGE-1/2/L、BertScore |
| LongMemEval-S | General（非 GUI 对话） | 多会话记忆保持 QA | 500 题 | 纯文本对话 | 文本答案 | 同上 |

### 5.2 实验结果分析

**主结果（GPT-4o-mini backbone，统一 MemGAS 评测协议）**：

- **LoCoMo**：CreaMem 4o-Judge **54.61**，显著高于最强基线 HippoRAG 2（45.62）、SeCom（44.21）、MemoryOS（43.96）、MemGAS（41.07）；F1 19.34、ROUGE-1 20.01、ROUGE-2 9.04、BertScore 85.30，**全指标最优**，且 token（2,805）与结构化基线同量级。
- **LongMemEval-S**：CreaMem **66.40**，高于 MemGAS（60.20）、HippoRAG 2（57.60）；仅用 **7,250 token**，对比 Full History 103,137 token 却只有 50.60——**约 1/14 上下文拿到最高准确率**。
- 分类别看（附录）：CreaMem 在 Overall/Multi-hop/Temporal 领先，Single-hop/Open-domain 分别被 SeCom、MemoryOS 反超（Multi-hop 领先 2.83、Temporal 领先 19.93）。

**Ablation（Gemini-3 复跑，固定每问 20 条、内容完全相同、只重组结构）**：

- **组件**：3-scene 比 1-scene 高 **2.3 点**（证明分区非冗余）；Full System **75.1%** 比 Episodic-only（65.9%）高 **9.2 点**，Temporal 上差距拉到 **11 点以上**；且 Episodic-only 反而**多耗 42% token**（详情字段保留完整事件叙述）。
- **分区数量**：1/2/3/4/5 场景中 **3-scene 最佳**——更粗则无关场景争抢检索位，更细则记忆过度碎片化。
- **超参**：准确率对 k **非单调**，k=30 达峰、k=40 下降，超额检索是噪声；作者刻意取 k=20 对齐基线 token 预算以隔离"组织"与"上下文量"两个变量。

---

## 6. 亮点与贡献（Why it matters）

1. **把认知科学公理落成可跑 schema**：Conway（按人生情境组织）+ Tulving（情节/语义互补）直接决定分区与双编码，不是贴标签式的"受启发"。
2. **消融干净**：用"同一份内容、同一检索预算、只换结构"把"组织方式"与"信息量"解耦，2.3/9.2 点增益因此可信，且反直觉地证明"更少 token + 更高分"。
3. **点破了一个被忽略的真痛点**：跨场景干扰——此前分区工作换轴不换池，本文是首个沿"生活场景"切分并验证其收益的。
4. **效率论证有力**：1/14 token 拿最高准确率，直接反驳"堆上下文"路径依赖。
5. **对分类法本身做了实证审计**（覆盖率 + 人工交叉），比只声称"分类合理"扎实；并诚实暴露了路由/抽取的失败边界。

## 7. 局限与可改进点（个人点评）

- **三分类是固定且偏粗的本体**，作者自己承认未必跨用户/文化/领域迁移；更关键的是**寒暄、闲谈这类"无场景"消息正是覆盖率缺口主因**，暴露了"万物皆可归入三场景"前提的脆弱。
- **系统开销偏大**：端到端 2.9 秒/查询（A-Mem 1.98 秒），充分性检查还可能触发第二轮检索；写时成本（批处理粒度、抽象层级）难以横向比较，论文**没给完整账本**，组织优势可能被更高抽取/合并开销部分抵消。
- **评测口径依赖 LLM 裁判**，且主表（GPT-4o-mini）与消融表（Gemini-3）**不在同一 backbone**，增益幅度不可直接跨表叠加。
- **失败模式是结构性的**：①关联条目检索不全（Nate 的"比赛"与"Street Fighter"分存两条、只召回其一）；②检索到了但不会合并（John 的 Under Armour 代言，两条证据都进了上下文却未连接）。论文只把这两类列为"动机"，**没给机制**，也没有 Recall@k/NDCG 这类检索侧诊断。

## 8. 对我们的启示 / 可借鉴点

本文是 Agent 方法论域（非 GUI 落地），**对 GUI Agent 的可借鉴点**如下：

- **"场景分区"→"App/任务域分区"**：GUI 世界天然分场景（办公套件/浏览器/通讯/设计软件）。把"用户习惯用快捷键""该 App 弹窗要先点确认"存为**场景特质**，把"某天在某个文件上做了什么"存为**情节条目**，跨 App 任务（邮件取数→表格汇总→回发）即可获得互补视角。
- **双编码 → 操作记忆 + 界面特质记忆**：一条 GUI 经历同样值得存两份——时间线（在哪页、点了什么、结果如何）+ 界面特质（布局规律、易错控件、验证码时机）。查"上次报表怎么导的"靠前者，查"这系统一般怎么处理权限弹窗"靠后者。
- **均衡采样 + RRF 是零成本防垄断方案**：GUI 上下文常需同时塞截图特征、DOM 摘要、历史操作、用户偏好，来源异构且数量不等，"每来源等量候选 + RRF"可直接复用；而"宽进严筛"契合 GUI 信息稀疏——一步截图往往同时暗示偏好、任务状态、界面约束，宁可多路由几个桶。
- **警惕地采纳"组织 > 上下文量"**：GUI Agent 常靠堆历史截图"记住"状态，代价是 token 与延迟双高。但注意本文消融显示 RRF 本身贡献近零——**真正的杠杆是"分区 + 双编码"，不是检索融合的花活**。
- **两类失败在 GUI 同样高发**：建议在记忆条目上显式存**指向原始轨迹的链接**，并预留"回退到原始片段"的兜底检索，应对"关联条目分存/检索到却合并失败"。

## 9. 延伸阅读

- LoCoMo、LongMemEval-S：本文两个长期记忆 benchmark
- MemGAS、HippoRAG 2、A-Mem、MemoryOS、MIRIX、SeCom、RAPTOR、RecurSum、MPC、MemGPT、MemoryBank：三类记忆范式代表
- Conway Self-Memory System（2000）、Tulving 多重记忆系统（1972）：认知科学依据
- Cormack et al.（2009）：RRF 融合算法
- Mem0 / MemAgent / PlugMem：正文引用但未展开的分区/agentic 记忆近期工作

---
*解读生成时间：2026-09-10 ｜ 解读人：WorkBuddy（AI）*
