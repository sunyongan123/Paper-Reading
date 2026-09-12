# Grounding Agent Memory: Environment-Probing Curation for Enterprise Agents

> 一句话 TL;DR：给任务后 curator agent 开放只读环境工具，让记忆在写入前被"探测验证"，无需重训模型即显著提升长时程 Agent 的正确率并降本。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Grounding Agent Memory: Environment-Probing Curation for Enterprise Agents |
| **作者 / 机构** | Susheel Suresh\*（通讯，sussuresh@microsoft.com）、Hazel Mak、Sahil Bhatnagar、Chhaya Methani、Alejandro Gutierrez Munoz；Microsoft Corporation（美国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-10；arXiv preprint，arXiv:2609.11060v1 [cs.AI] |
| **arXiv 链接** | https://arxiv.org/abs/2609.11060（点击直达） |
| **代码仓库** | ❌ 未开源（harness 基于公开的 GitHub Copilot SDK：https://github.com/github/copilot-sdk 搭建，论文未提供自身代码） |
| **数据集地址** | CLBench（Asawa et al. 2026, arXiv:2606.05661）；Adapted APEX（Vidgen et al. 2026, arXiv:2601.14242），原榜 https://www.mercor.com/apex/apex-agents-leaderboard/ |
| **类型标签（论文类别）** | `Reflection` `General` |
| **训练方法标签** | 不训练（无权重更新；外挂 distiller + curator agent，仅给 curator 加只读世界工具） |
| **关键词** | Agent Memory；Post-task Curation；Environment Probing；Continual Learning；Cost-aware Evaluation |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-10_Grounding_Agent_Memory_Environment-Probing_Curation_for_Enterprise_Agents.pdf |

---

## 2. 论文要解决的核心问题

- **问题**：跨会话企业 Agent 任务流中环境会静默漂移（`E_{i+1} ~ Δ(E_i)`），每任务开新 session，只有异步 curator agent 能写持久记忆，其证据限于"已完成轨迹 + 终局评分"。
- **重要性**：轨迹级偏差有五类——记实例答案而非过程、继承低效路径、断言不可验证的适用范围、未访问区域留盲点、环境变化后过期。更关键是成本：验证推到任务期，task agent 就得用稀缺 tool call 反复复核不确定记忆；这正是 CLBench 上 query 高达 8.8 的原因。
- **现有不足**：Mem0、Dreams、ACE、ReasoningBank、ReMe 等 post-task curation 都只作用于"既有记录 / 已完成轨迹 / feedback / usage signal"。结构性批评：**reflection 再强，也无法恢复任务策略从未观测到的状态，也无法判断轨迹推导的规则在漂移后是否仍成立**。
- **Research Gap**：补"写时证据质量"这一正交维度——不改 task agent、retriever、schema、写权限，也不重训模型，只给异步 curator 一个最小权限只读世界工具子集，让它在提交前独立核验、限定范围、刷新过期条目；可归因性来自 task-time 接口与 CRUD 生命周期固定，增益只能来自写入质量。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

任务关闭后，让异步 curator agent 通过 `propose–probe–commit` 流程，用只读环境工具验证候选记忆后再写入持久索引。

### 3.2 方法总览（Pipeline）

- **输入/输出**：任务流 `S=(S1,…,SN)`；task agent `Aθ` 接收 `Si` 与环境工具 `TEi`，仅经 `memory_read` 只读访问 `M_{i-1}`；输出新索引 `Mi` 供 `S_{i+1}` 使用。
- **四模块**：①**轨迹产出** `τi = Aθ(Si; TEi, R(·,M_{i-1}))`，含 request、memory reads、action–observation 对、提交答案；终局反馈 `gi` 任务关闭后才到。②**Distiller `Dψ`** 把轨迹压成证据包 `di = Dψ(τi)`，保留任务目标、检索到的记忆及其使用或矛盾、关键 action–observation、成败过程、工具约定、未解决假设；**看不到终局反馈，无 memory/env 工具，不能写记忆**。③**Curator agent `Cφ`** 独立 session，唯一有 CRUD 权限，输入 `di`、`gi` 与既有相关记录，可回查 staged 轨迹；记录含 category、confidence、applies_to、lemma、provenance。④**Environment-Probing 扩展** 只额外给 curator 只读环境工具子集（CLBench 为数据库查询；APEX 为只读 MCP）与两条探测指令，其余 prompt、模型、CRUD 策略、schema 全不变。
- **六类探测**：区分实例答案与可复用关系、比对更短路径、换 slice 验证关系、检查前置条件、检查轨迹遗漏状态、怀疑漂移时重查环境。
- **边界**：只有 curator 能写；探测在关键路径与任务预算之外，不入任务轨迹、不暴露未来任务；无安全只读面时退回 trajectory-only；继承平台鉴权与审计。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：把"写时验证"从记忆系统中隔离为可独立测量的问题——curator 侧只读世界工具 + propose–probe–commit 循环与六类探测原语；更值得记的是实验设计：冻结 task-time 接口、retriever、schema、CRUD 生命周期与 task agent，把增益严格归因于写入质量。
- **工程组合**：GHCP SDK harness、Mem0 式 CRUD、distiller 预处理、`e5-base-v2` 检索、`r_i = p_i(1 − q_i/B)` 的 pass-discounted reward、平台 connectors/MCP 作只读面——均为既有组件。
- **对性能最关键的设计**：①**探测**：同为索引式记忆，加 probing 后 drift CLBench reward 20.00→22.60、queries 5.6→4.7、cost $1.99→$1.68；no-drift 下 Sonnet 0.673→0.748、Opus 0.696→0.721。②**索引式紧凑记忆 vs Full ICL**：input token 5.42M→2.13M→1.69M，可复用经验不必让上下文随部署年龄线性增长。
- **证据不足 / 仅声称有效**：①"探测后的记录更 actionable"只有 Figure 3 定性对照（representative records 而非一对一改写），**无记录质量量化指标**。②probing 收益归因被作者降级为"机制解释"（CI 重叠）。③并非普适：`941eba66` 上为 −0.04。④成本口径不对称：distiller 与 curator 的 token/$ 被排除在 task-agent cost 外，"reward per dollar"只覆盖一半成本。⑤APEX 只取 480 任务中的 90 个。

---

## 4. 具体技术细节

### 4.1 模型结构

纯 LLM、无视觉编码器。task agent、distiller、curator 共享基座：主实验 `gpt-5.4`（xhigh reasoning effort）；no-drift 跨模型用 Sonnet 4.6（high）与 Opus 4.7（xhigh）。参数 `θ` 全程固定，**不训练、不微调**。检索用 `intfloat/e5-base-v2`。运行时为 Python GitHub Copilot SDK + 容器内 headless Copilot CLI server mode（JSON-RPC）。

### 4.2 训练流程（若需要训练）

| 阶段 | 目标 | 训练什么能力 | 数据来源 | 数据形态 | 训练目标 / Loss |
|---|---|---|---|---|---|
| 不训练 | —（完全 prompt-based + 外挂 agent，无梯度更新） | 不涉及；能力来自基座 LLM 的 in-context 指令遵循与工具调用 | 无训练数据；仅用 CLBench / APEX 的评测任务流与轨迹 | — | 无 loss；无负采样；无 LoRA/全参微调 |

> 论文刻意不训练：可归因性论证正建立在"权重固定、task-time 接口固定"之上；若引入微调，reward 增益就无法再解释为 write-time evidence quality。

### 4.3 推理流程（若不训练或重点在推理）

- **多步、双循环**。任务期是标准 tool-calling agent loop：fresh session → 环境工具 + `memory_read` → 交替 tool call/observation → 提交答案；无显式 ReAct 或规划-执行分解，终止即提交 answer，随后 `gi` 到达。
- **curation 循环（异步、离关键路径）**：distiller 一轮（输出 `<overview>/<history>/<work_done>/<technical_details>/<important_files>/<next_steps>/<checkpoint_title>`）→ curator 多轮 propose–check–reconcile–commit，终止于"不再有正当理由的 CRUD 操作"；必须早于 `S_{i+1}` 暴露，以防未来任务泄漏与部分写入。
- **评测协议**：CLBench 每配置 5 次洗牌 paired run；APEX stateful 5 次、stateless 3 次；报告 95% Student-t CI；no-drift 固定 30 题顺序、5 组 paired run。折扣预算 `B=15`（CLBench SQL queries）与 `B=100`（APEX Archipelago calls）；多评分标准任务用 strict pass。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| CLBench（drift，主实验） | General（企业数据分析 / 隐藏 SQLite 探索） | schema 发现 + join/编码发现 + SQL QA；第 20 题后未公告迁移（改名、拆列、软删除） | 40 questions；5 paired seeded runs | 自然语言问题 + DB 查询工具（schema 隐藏；含美元/美分、epoch-ms/ISO 时间戳陷阱） | SQL 与最终答案 | strict pass rate、Eq.3 reward（B=15）、queries/question、tokens、task-agent USD |
| CLBench（no-drift，跨模型） | General | 同上，schema 固定，隔离"稳定 join/编码复用"与"迁移恢复" | 30 questions 固定顺序；Sonnet 4.6 与 Opus 4.7 各 5 paired runs | 同上 | 同上 | mean reward 与相对 paired no-memory 的 lift |
| Adapted APEX | General（管理咨询，多文档办公） | PDF/XLSX/DOCX/PPTX（含嵌入图）发现 + 代码执行定量分析 + MCP 风格 artifact 产出 | 原 480 任务取 management-consulting，按 (domain, world_id) 分 6 world / 90 题（每 world 11–18 题） | 咨询任务指令 + 只读 MCP/Archipelago 工具 | artifact / 答案 | strict pass reward（B=100）、tool calls、tokens、USD、reward gain per dollar |

### 5.2 实验结果分析

- **主结果（CLBench drift，gpt-5.4）**：No Memory 39%±4、reward 8.60±0.83、8.8±0.3 queries、3.14M input tokens、$3.38±0.70 → Full ICL 61%±11、21.39±3.83、3.0 queries、5.42M、$2.01 → GHCP+Mem 70%±16、20.00±6.52、5.6±1.4、2.13M、$1.99 → **GHCP+Mem (w/ Env Probing) 73%±5、22.60±2.07、4.7±0.3、1.69M、$1.68±0.14**。学习曲线：probe 0.565 vs trajectory-only 0.500 vs no memory 0.215；迁移边界处 0.541 vs 0.486。
- **跨模型（no-drift 30 题）**：Opus 4.7 上 Mem lift +0.252（→0.696）、probing +0.263（→0.721）；Sonnet 4.6 上 Mem +0.351（→0.673）、probing +0.421（→0.748）。probing 在两模型上均取得最高 mean reward。
- **APEX（90 题 / 6 world）**：**全部 18 项 memory-vs-baseline reward 比较均为正**；tool calls 从基线均值 30.0–71.6 降至 13.9–28.4；最贵的 `941eba66` 从 71.6 calls 降至 17.7–19.3，input 从 53.92M tokens 降至 6.56–7.67M，成本从 $54.30 降至 $7–$9；probing 在 6 个 world 中的 5 个取得最佳 reward gain per dollar。
- **Ablation 结论**：①memory vs no-memory：正确率与 reward 双升、query 与成本双降，收益不是靠暴力多查。②Full ICL vs 索引式记忆：先验轨迹含可复用信息（query 最少）但上下文随流增长（5.42M tokens）；索引记录能在不携带完整交互史的前提下保留 schema/关系/文件地图/过程。③probing vs trajectory-only：drift 下 reward +2.60、queries −0.9、cost −$0.31；no-drift 下 Sonnet +0.075、Opus +0.025；APEX 上 6 个 world 中 5 个提升，最大增量 +1.77（`2a87e5cb`）与 +1.09（`2f84c98b`），`941eba66` 为 −0.04。

---

## 6. 亮点与贡献（Why it matters）

1. **把 memory 问题从"存/取"重构成"写时证据质量"**：以往工作改记忆内容、表示、检索或重写，本文指出缺口在写入前的证据边界。
2. **零模型改动、零生产写权限**：curator 只拿已有 connectors/MCP 只读子集，探测在关键路径外，无安全只读面时自动降级。
3. **成本是一等公民**：CLBench 上 cost $3.38→$1.68 与 reward 8.60→22.60 同时发生，打破"记忆=堆上下文"的直觉；跨 GPT-5.4 / Sonnet 4.6 / Opus 4.7 一致。
4. **中间产物可审计**：记录带 category、confidence、applies_to、provenance，Figure 3 展示从"答案锚定式警告"到"可执行过程"的转变。

## 7. 局限与可改进点（个人点评）

1. **成本口径不完整、记录质量无量化**：probing 增加 curator 的 tool call 与上下文却被排除在 task-agent cost 外；记录质量只有 Figure 3 定性对照，缺复用率、贡献度、错误率。
2. **子群效应被自己否决**：`941eba66` 为负、Sonnet 增益大于 Opus，作者诚实标注 CI 重叠——但这意味着**仍不知道何时该开 probing**，缺"轨迹是否留下未解决 join/位置/过程"的判别器。
3. **评测域偏窄且为文本化**：CLBench 是 SQLite、APEX 是文档/表格流，无 GUI 截图、无多模态、无真实生产的权限与延迟约束。
4. **只读探测有"探测不到"的边界**：设了 fallback 但未量化回退比例，也未讨论探测偏差（curator 只验证易验证的假设，反而强化"看起来被验证过"的错误记录）；未报告索引随任务流的变化与检索精度衰减。

## 8. 对我们的启示 / 可借鉴点

- **把"验证"放在写入侧而非读取侧**：读取侧验证花任务预算，写入侧异步、可审计、不占关键路径；凡构建长期记忆的 Agent 系统都应把 validation 从 task-time 挪到 curation-time。
- **用"冻结其他变量"隔离机制**：固定 task agent/retriever/schema/CRUD，只动一个写时变量，归因无歧义；做 RL/记忆/工具类改进时也应构造正交对照，而非同时改三处再宣称整体有效。
- **成本与正确率必须同图呈现**：`r = p(1 − q/B)` 把"少用工具"写进奖励，避免"用 3 倍 query 换 2% 准确率"被误读为进步；只读最小权限则是安全与工程双赢。

### 对 GUI Agent 的可借鉴点

1. **GUI 版探测清单可直接照搬**：CLBench 的 probe 是查表/join key/编码/迁移后字段，APEX 是查文件位置/文档相关性/工作簿内容/工具约定；映射到 GUI 即用只读 accessibility tree / DOM 快照 / 控件坐标验证"该按钮在当前版本是否仍叫这个名字""该菜单路径是否仍存在"。
2. **用只读探测消除重复探索**：GUI Agent 的 tool call 极贵（每步截图 + 多模态推理）；"写时探测 → 读取侧少查"可把 query 从 8.8 降到 4.7。可设计 GUI 版 propose–probe–commit：候选经验（如"该设置项在 Advanced → Network 下"）先由只读 DOM 查询验证层级路径，再入库。
3. **"成功"不等于"过程正确"**：GUI 任务常因"多点了一次恰好也过了"留下错误操作序列；建议在 GUI 经验库写入前加独立 verifier agent，区分"实例答案"与"可复用操作过程"。
4. **异步 curation 适合 GUI 等待窗口 + schema 可借用**：GUI Agent 在模型推理与页面加载时同样有空闲窗口，可把经验压缩/验证放在这些窗口（与 AgentZip 的"LLM 等待期做重活"同源）；`category` + `confidence` + `applies_to` 可适配"应用版本 / 页面路径 / 控件类型"作用域，`trap` 类（"不要点 X 否则弹确认框"）价值很高。

## 9. 延伸阅读

- **Asawa et al. 2026, CLBench** — arXiv:2606.05661，主 benchmark 与"记忆会编码伪泛化/过期信念"论断来源；**Vidgen et al. 2026, APEX-Agents** — arXiv:2601.14242。
- **Xiong et al. 2026, How Memory Management Impacts LLM Agents** — ACL 2026，识别错误传播与错配的经验回放；**Chhikara et al. 2025, Mem0** — ECAI 2025，显式 CRUD 的生产级记忆；**Cao et al. 2026, ReMe** — ACL 2026 Findings，加验证/去重/效用剪枝/任务条件重写，与本文最接近的 curation 侧增强。
- **Ouyang et al. 2025, ReasoningBank**（arXiv:2509.25140）／**Zhang et al. 2025, ACE**（arXiv:2510.04618）／**Shinn et al. 2023, Reflexion**（NeurIPS 2023）——procedural memory 与自演化 playbook。
- **Packer et al. 2023, MemGPT**（arXiv:2310.08560）／**Park et al. 2023, Generative Agents**（UIST 2023）／**Xu et al. 2025, A-MEM**（arXiv:2502.12110）／**Rasmussen et al. 2025, Zep**（arXiv:2501.13956）——分层、生成式与图谱化记忆路线。

---
*解读生成时间：2026-09-12 ｜ 解读人：WorkBuddy（AI）*
