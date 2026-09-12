# Agentic Web Accessibility Auditing: Authoring and Evaluating Per-Criterion Worker Agents for WCAG

> 一句话 TL;DR：给 40 条 WCAG 准则各配一个"带浏览器工具的 VLM worker"，在 250 条 page–criterion 记录上把违规召回从 axe-core 的 0.36 提到 0.86，代价是精确率降到 0.56。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Agentic Web Accessibility Auditing: A Criterion-Specific Framework for Translating WCAG Requirements into Assessments |
| **作者 / 机构** | Arjun Mishra（UBC，加拿大）、Pranav Karthik（UBC，加拿大）、Byungjun Bae（ETRI 韩国电子通信研究院，韩国）、Dongwook Yoon（UBC，加拿大，通讯作者 yoon@cs.ubc.ca） |
| **发表时间 / 会议 / 期刊 / arXiv** | v1 2026-09-08，v2 2026-09-10；arXiv preprint（arXiv:2609.09379v2，cs.HC） |
| **arXiv 链接** | https://arxiv.org/abs/2609.09379（点击直达） |
| **代码仓库** | ❌ 未开源。论文称随预印本提供补充材料（`tool_library/` 含 MCP tool server 与浏览器工具源码、`worker_skills/` 含 40 份准则指令、`comparison_prompts/` 含批式与 coding-agent 提示词），但**未给出任何公开仓库 URL** |
| **数据集地址** | 未公开。250 条 page–criterion 记录由 Library Accessibility Alliance（LAA）公开审计报告构建，来源报告页 https://www.libraryaccessibility.org/testing ；数据集本身（含 SingleFile 快照与标签）未见下载链接，仅作补充材料附带 |
| **类型标签（论文类别）** | `Web` `Benchmark` `Planning` |
| **训练方法标签** | —（评测/工程）。所有模型均为冻结调用，无微调；框架侧"训练"实为人工撰写 40 份准则技能文档 |
| **关键词** | web accessibility、WCAG、LLM agents、automated auditing、inspectable evidence、benchmark |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_Agentic_Web_Accessibility_Auditing_Authoring_and_Evaluating_Per-Criterion_Worker_Agents_for_WCAG.pdf |

---

## 2. 论文要解决的核心问题

- **要解决什么问题**：Web 场景的 agentic 界面审计。无障碍评估要回答"不同需求的人能否感知、理解、操作界面"，WCAG 把要求组织成成功准则；论文要解决的是如何让自动化系统**按具名准则**组织取证与判定。
- **为什么重要**：无障碍缺陷直接决定残障用户能否使用学术平台等公共服务，漏检意味着审计结论不可信、修复资源错配。Power 等证明盲人用户遇到的障碍超出准则合规覆盖，Vigo 等证明单一依赖自动化工具会系统性漏检。
- **现有方法有什么不足**：① 规则引擎（axe-core）只能做有确定性判定过程的检查（本例 recall 0.36），需语义或行为证据的检查（如替代文本是否传达图片用途、控件能否被键盘进入并离开）天然覆盖不到；② 批式 VLM（含 GenA11y 式抽取）把整页压成一次判断，缺证据获取能力，uncued 召回 0.67 但 No Keyboard Trap 0/2、Keyboard 0/8；③ TaskAudit 最接近，但以"生成的任务"为单位、面向移动端，不按命名准则组织；④ AccessGuru / WebAccessVL 面向检测+修复，与"独立判断是否违规"是两种能力。
- **Research Gap**：把"具名准则 → 需要什么证据 → 用哪个可执行操作获取 → 观察到什么才可判定"显式化，并为每次判定保留可事后检视的执行记录。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把每条 WCAG 成功准则实例化为一个 worker：共享浏览器工具库 + 准则专属 Markdown 技能指令 + 首操作指引；worker 走 ReAct 循环反复取证，输出 true / false / error 的结构化判定与引用观察。

### 3.2 方法总览（Pipeline）

- **输入**：Chromium 中打开的保存网页快照（SingleFile 自包含 HTML，1280×720 视口）+ 一条目标 WCAG 准则 + 该准则配置（共享响应契约、技能文档、首工具指引、声明工具列表）。**输出**：判定（true 检出 / false 按本流程未检出 / error 执行未完成）+ impact 标签（**不是校准概率**）+ 执行日志。
- **模块**：① 共享执行基类（协调 headless Chromium，上限 30 次迭代）；② 共享工具库 31 个具名操作，覆盖 DOM 与无障碍树抽取、元素查询、截图与 computed style、对比度测量、键盘/指针命令、focus 序列记录、widget 探针（脚本化激活 tab/radio/dropdown 并记录键盘焦点序列）、axe 检查；③ MCP tool server 以具名调用暴露这些操作；④ 准则技能文档（Markdown，说明检视什么、例外、如何解释结果）。
- **连接**：工具结果作为 observation 进入下一步推理；声明列表是引导而非强制边界，越界调用会被日志标为 `non-included`。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：① "一条准则 = 一个 worker"的配置结构，把"规范性要求 → 证据需求 → 可执行操作 → 判定条件"四段显式绑定，使失败可在"哪条要求、缺哪种证据"的粒度上被检视；② 技能文档与执行机制分层维护（浏览器操作修复对所有 worker 生效，准则澄清写在各自文件）。
- **工程组合**：ReAct 循环、headless Chromium、axe-core、MCP tool server、截图+无障碍树输入都能在 BAGEL / Groundhog / AXNav / TaskAudit 找到先例；真正属于本文的工程量是把 40 条准则逐条翻译成指令文档。
- **对性能最关键的设计**：① **交互工具（widget 探针 + 键盘操作）**——Keyboard 7/8、No Keyboard Trap 2/2，而无交互批式 VLM 是 0/8 与 0/2；这是 worker 唯一大幅领先的准则族，也基本就是它整体召回优势的来源；② 准则级指令 + 声明工具集（相对 uncued 的 0.67 → 0.86，+19pp；仅加静态提示的 cued 只到 0.74）。
- **证据不足 / 仅声称有效**：① 实现广度 ≠ 检出能力，40 个 worker 里只有 15 条准则有正例，其余 25 条无法估计 recall；② **没有交互工具的消融**，比较条件同时改变工具、指令与模型，交互的净贡献从未被隔离；③ 负标签是"报告未列出"推断的，而报告是时间受限的部分审计，会直接影响 precision 的解释；④ 27 个候选新发现无独立裁决。

---

## 4. 具体技术细节

### 4.1 模型结构

base model 是 **MLLM，含视觉编码器**。输入不是纯像素：worker 同时收到**截图 + 无障碍树**，论文特意强调因此不能称为"纯图像评估"。参考 worker 记录为 **GPT-5.5**；另有 Gemini 3.1 Pro、Claude Sonnet 5 等 instrumented run，批式 VLM 另有 7 个配置，coding-agent 为 Claude Code 与 Codex。**全部冻结、无微调**；除 Gemma 4 31B 与 Qwen3.5 397B 外未给参数量。

### 4.2 训练流程

**不训练。** 全部为冻结模型调用，没有梯度更新、LoRA 或蒸馏。所谓"框架侧训练"实为作者手工撰写 40 份准则技能文档、配置工具映射、回看执行样例后修订指令；论文明确写"未比较撰写方法、未系统测量撰写时间"。因此知识注入完全依赖 prompt 工程与工具配置，文档是否覆盖该准则的全部失败模式也未系统评估。

### 4.3 推理流程

**多步推理，ReAct 式循环。** 模型解释 observation → 请求一个操作 → 接收结果 → 再决定是否继续；**终止条件是给出 true/false/error 判定，或触及 30 次迭代上限**。对比条件有两类更浅形态：批式 VLM 每页 3 次调用（按原则分组，共享同一份截图与无障碍树）；per-criterion VLM 每对一次调用，并显式提供 undetermined 弃权选项。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| LAA 250-row benchmark（主评测） | Web 学术平台静态快照 | 准则级违规判定 | 250 条 page–criterion（24 页 / 11 平台；78 正 / 172 推断负；覆盖 40 条准则，仅 15 条有正例） | SingleFile HTML；worker 侧为截图 + 无障碍树 + 工具输出 | true / false / error + impact + 引用观察 | micro-P / R / F1（弃权与错误不折算为负） |
| 灵敏度子集 | 同上 | 同上 | 排除开发暴露页 55 条；排除 axe 来源行 239 条；排除动态依赖行 240 条 | 同上 | 同上 | 同上（描述性） |

> 正例来自 LAA 报告已记录的 finding，负例是"该页该准则未被列出"的**推断**；每准则采 5 个负例（seed 42）。这种构造抬高正例占比，因此 precision 只能解释为"与这份标注样本的一致度"。

### 5.2 实验结果分析

**主结果（250 行 / 78 正例）**：

| 条件 | TP | FP | 弃权 | P | R | F1 |
|---|---|---|---|---|---|---|
| axe-core | 28 | 3 | 0 | 0.90 | 0.36 | 0.51 |
| Batch VLM, uncued | 52 | 33 | 0 | 0.61 | 0.67 | 0.64 |
| Batch VLM, cued | 58 | 33 | 0 | 0.64 | 0.74 | 0.69 |
| Per-criterion VLM | 46 | 28 | **92** | 0.62 | 0.59 | 0.61 |
| **Workers（参考运行）** | **67** | **53** | 0 | **0.56** | **0.86** | **0.68** |
| Claude Code, naive / parity | 47 / 60 | 26 / 49 | 0 | 0.64 / 0.55 | 0.60 / 0.77 | 0.62 / 0.64 |
| Codex, naive / parity | 25 / 42 | 10 / 24 | 0 | 0.71 / 0.64 | 0.32 / 0.54 | 0.44 / 0.58 |

**准则级差异**：1.3.1 与 4.1.2 各 17 个正例（合计 34/78），几个准则只有 1 个，所以聚合召回掩盖了极大方差。Keyboard 2.1.1 worker 7/8 vs uncued 0/8；No Keyboard Trap 2.1.2 worker 2/2 vs uncued 0/2（7 个批式配置全部漏掉这两个正例）；Focus Order 2.4.3 worker 6/7 vs uncued 4/7；Non-text Content 1.1.1 worker 6/8 但 per-criterion 7/8；Contrast 1.4.3 worker 3/5 vs uncued 4/5——"加交互工具"并非普遍加分，而是准则特异的。

**成本与弃权**：250 行 USD 估计——batch uncued $6.38、cued $7.29、per-criterion $14.98、Codex $32.76/$38.63、Claude Code $58.60/$74.52、GPT-5.5 $64.19、repeat $64.90；参考 worker 运行**无匹配用量记录，因此未分配成本**。两次 GPT-5.5 pass 在 241/250 条判定上一致（96.4%）。per-criterion VLM 在 92/250（36.8%）行弃权，含 27 个正例：全量召回 0.59，但在"已承诺"的 158 行（51 正）上条件召回 46/51 = 0.90；**同一子集上 cued 命中 45、uncued 与 worker 各 42**——0.90 完全是分母选择效应。

**Ablation 说明了什么**：① **提示线索有效但有限**：uncued → cued 召回 0.67→0.74；② **粒度更细不等于更好**：per-criterion 成本翻倍、弃权 36.8%、全量召回反降到 0.59；③ **审计指令很关键**：coding-agent 从 naive 到 criterion-enumerated，Claude Code 召回 0.60→0.77、Codex 0.32→0.54，但 FP 同时从 26→49 与 10→24，precision 双双下滑；④ **工具/交互的净贡献没有被消融**，这是最该补的实验；⑤ 灵敏度：排除开发暴露页后（55 条 / 15 正）workers 0.58/0.93、uncued 0.62/0.67、axe 1.00/0.40。此外 uncued VLM 与 worker 的**并集**召回 74/78（P 0.54 / R 0.95），**交集** 45 TP / 23 FP（P 0.66 / R 0.58）。

---

## 6. 亮点与贡献（Why it matters）

1. **"一条准则一个 worker"是真正可用的工程抽象**：把"审计结论到底测了什么"变成可回答的问题，失败可定位到具体准则与缺失的证据类型。
2. **交互证据的价值被量化得很具体**：Keyboard 7/8、No Keyboard Trap 2/2，而 7 个不同的批式 VLM 配置在这两条准则上全部漏检。对"GUI agent 能否替代静态检查"这个争论，这条证据有直接价值。
3. **诚实的"高条件召回"拆解**：主动指出 per-criterion VLM 的 0.90 是分母效应，并在同一子集上重算所有条件——这种自我拆台式的严谨度在 agent 论文里少见。

## 7. 局限与可改进点（个人点评）

1. **最关键的消融缺席**。全文核心叙事是"交互取证能救回静态方法漏掉的障碍"，却从没有"同一批 worker、撤掉键盘/widget 工具"的对照。比较条件同时改变了工具、指令与模型，因此那个 +19pp 有多少来自交互、多少来自指令，读者无从判断。
2. **40 个 worker 里 25 个无法评估 recall**。"实现了 40 条准则"极易被误读：有正例的只有 15 条，且两条贡献了 34/78 的正例。本文实际验证的是少数结构性准则加两条键盘准则。
3. **负标签的构造方式决定了 precision 不可信**。正例来自报告、负例来自"报告没写"，而报告是时间受限的部分审计，还可能对应与快照不同的页面版本；27 个候选发现既无独立裁决，所以 precision 落在 0.56–0.78 之间哪个位置其实不知道。
4. **快照环境的限制被低估**。所有条件跑在 SingleFile 静态文档上；但无障碍障碍恰恰高度依赖状态（弹窗、动态菜单、表单校验），因此"能操作保存的页面"与实际审计能力的差距可能比论文表述更大。

## 8. 对我们的启示 / 可借鉴点

1. **"能力 → 证据需求 → 可执行操作"的映射可直接迁移**。GUI agent 的技能设计同样应显式回答：这个技能需要什么观察（DOM？无障碍树？截图？焦点序列？），用什么操作获取，观察到什么才允许下结论。
2. **交互操作是差异化能力，但要按能力分类评估**。行为型检查（键盘/焦点）上能操作界面的 agent 优势压倒性（7/8 vs 0/8），但语义/视觉判断上单次 VLM 反而更好（Non-text Content 7/8 vs 6/8、Contrast 4/5 vs 3/5）。**不要用一个统一流程覆盖所有检查项**，而应按"这个检查需要什么证据"路由到最便宜且足够的方法。
3. **多轮取证 ≠ 更好，成本与弃权要计入指标**。per-criterion 条件成本翻倍、弃权 36.8%、全量召回反而下降。评测 GUI agent 时若把弃权行排除在分母外，就会得到漂亮但无意义的条件召回（0.59 → 0.90）；规范里应强制弃权与错误单列。
4. **不同方法的预测重叠度低，并集才有价值**（74/78），说明两类方法发现的是不同问题。落地建议是做"多方法 + 升级路由"，并把未决案例保留在产出里交人工。

## 9. 延伸阅读

- **准则相关元素抽取与模型判定**：GenA11y（FSE 2025），本文最直接的前作。
- **检测+修复路线**：AccessGuru（arXiv:2507.19549）、WebAccessVL（arXiv:2602.03850v3）、LLM Based Web Accessibility Repair（arXiv:2605.27716）。
- **交互式无障碍测试（web）**：BAGEL（CHI 2023）、Detecting and Localizing Keyboard Accessibility Failures（FSE 2021）、Detecting Dialog-Related Keyboard Navigation Failures（ICSE 2023）。
- **移动端 agentic 审计**：TaskAudit（CHI 2026）、ScreenAudit（CHI 2025）、Groundhog（ASE 2022）、AXNav（CHI 2024）。
- **评测方法论与人工监督**：Towards Scalable Web Accessibility Audit with MLLMs as Copilots（AAAI 2026）；Ma11y（ISSTA 2024）；Vigo et al. 2013；Power et al. 2012；ReAct（ICLR 2023）；To Trust or to Think（CSCW 2021）。

---
*解读生成时间：2026-09-11 09:00 ｜ 解读人：WorkBuddy（AI）*
