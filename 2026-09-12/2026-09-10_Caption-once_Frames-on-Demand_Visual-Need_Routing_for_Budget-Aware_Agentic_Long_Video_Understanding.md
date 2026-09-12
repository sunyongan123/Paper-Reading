# Caption-once, Frames-on-Demand: Visual-Need Routing for Budget-Aware Agentic Long Video Understanding

> 一句话 TL;DR：离线一次字幕建成"事件骨架 + clip 微日志"双轨索引并缓存，在线由 Visual-Need Router 按问题类型决定是否回取像素，把每查询视觉成本从交互深度的副产品变成可调超参。

---

## 1. 论文基本信息

| 字段 | 内容 |
|---|---|
| **论文标题** | Caption-once, Frames-on-Demand: Visual-Need Routing for Budget-Aware Agentic Long Video Understanding |
| **作者 / 机构** | Weitong Cai（QMUL）、Hang Zhang\*（Independent Researcher）、Yukai Huang（Durham University）、Yiqiao Xie（Imperial College London）、Shan Gao（Huawei）、Jiankang Deng（Imperial College London）、Songcen Xu（Huawei）、Jifei Song（Huawei）、Zhensong Zhang\*（Huawei）；\*通讯作者。机构：Queen Mary University of London、Durham University、Imperial College London（英国）、Huawei（中国） |
| **发表时间 / 会议 / 期刊 / arXiv** | 2026-09-10；EMNLP 2026 Main（原文 txt 未打印 venue 行，按任务元数据） |
| **arXiv 链接** | https://arxiv.org/abs/2609.11899 |
| **代码仓库** | ❌ 未开源（全文未给出任何项目仓库或代码链接） |
| **数据集地址** | 未公开新数据集；实验使用公开 benchmark：Video-MME、InfiniBench、LVBench（附录 D） |
| **类型标签（论文类别）** | `Online` `Planning` `General` |
| **训练方法标签** | `—（工程/推理框架，全零样本，不训练）`（论文明确说明 captioner 与 Router 均在"generic zero-shot setting"下运行，微调被列为 future work） |
| **关键词** | long video understanding；edge-cloud agentic framework；visual budget；on-demand frame retrieval；dual-track memory；MLLM |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-10_Caption-once_Frames-on-Demand_Visual-Need_Routing_for_Budget-Aware_Agentic_Long_Video_Understanding.pdf |

---

## 2. 论文要解决的核心问题

- **问题**：边缘设备上的长视频问答（数小时）要在极紧算力/带宽下工作，须全程保留 query-relevant 记忆，但漏掉一个很小的关键事件或属性就答错。
- **重要性**：它是交互式视频助手、端侧记忆、可穿戴辅具的底座；视觉成本若随视频长度或交互轮数线性增长，工程上跑不起来。
- **现有方法不足**：视觉压缩（少采帧/压 token）省钱但牺牲覆盖度与保真度；文本翻译（字幕、转录、多轮重访更新文本记忆）可扩展但丢失细粒度视觉证据，且查询期成本随轮数增长（VideoAgent、VideoTree、DrVideo、VideoLucy 即此路线）。
- **Research Gap**：预实验建立**视觉-文本对偶性**——CaptionQA 图像 92.32% vs 纯字幕 77.17%；InfiniBench 上 60s 字幕的 Chronological 56.46 超原视频 48.44，但 Global Appearance 53.49 vs 70.54、Character Actions 53.07 vs 67.04。即语言记忆在长程时序结构上可强于密集帧，但像素对属性级感知仍具决定性。缺口是**把"是否需要看像素"变成 query-conditioned 的一等成本项**——作者称这是首个按问题类型决定视觉访问的 agentic 视频 QA 设计。

---

## 3. 方法核心思想（三层拆解）

### 3.1 一句话概括方法

把视频时序结构一次性索引进语言（故事骨架 + 片段微日志，缓存在边缘、跨查询复用），查询时云侧 MLLM 走"故事优先"迭代回退循环，仅在属性级感知问题上由 Router 触发有界关键帧检索。

### 3.2 方法总览（Pipeline）

- **三层记忆**：**事件记忆**（TransNetV2 切事件，轻量 MLLM 生成场景概览/实体/事件流，**全量**注入作全局时间轴）→ **片段记忆**（30s 不重叠微日志，**不整体注入**，仅在定位到事件 e\* 时把重叠字幕**累积**嵌套注入）→ **视觉工作记忆**（FIFO，N_wm=16）。
- **四个 prompt 角色**：Captioning（仅离线）、Answering、Localization、Visual-Need Router；相对前作**删掉 Instruction Agent**，用片段记忆 + 纯文本组合替代查询期重字幕。
- **故事优先循环**（T≤5）：事件记忆试答 → 时序定位 → 片段富化重答 → Router 判定 → 按需抽帧（0.1 FPS、≤8 帧/事件）→ 多模态重答；耗尽则 must-answer。

### 3.3 真正的创新点（去伪存真）

- **真·方法创新**：Visual-Need Router 按问题类型门控视觉访问，把帧检索从流水线副产品变成显式、query-conditioned 的成本项（视觉成本是显式超参而非隐式后果）。
- **工程组合**：caption-once 非首提（自引 Zhang 2024a、Kahatapitiya 2025）；双轨记忆、均匀切片、FIFO 工作记忆、story-first 回退均为组件重组。
- **最关键设计**：只有事件记忆太稀疏（overall 57.3）；只有片段记忆到 65.5 但 Global Appearance 仅 38.00（**纯文本不能替代视觉**）；无 Router 的按需帧把外观类拉高却把 Chronological 打到 51.70（**无差别注入帧会稀释时序叙事**）；加 Router 压帧后回 55.10——**提精度靠看得更少**。
- **证据不足/仅声称**：Router 对 46.2% 问题取帧而 oracle 真受益仅 4.9%（9.4 倍过度触发），论文却称其为"feature 而非 bug"；零样本、无 routing 标注，oracle 自我参照；字幕质量无漏检率量化。

---

## 4. 具体技术细节

### 4.1 模型结构

边缘 Captioning Agent 为 `Qwen3-VL-8B-Instruct`，云侧推理 MLLM 为 `Qwen3-VL-32B-Instruct`，均**冻结**；后者实例化 Answering、Localization、Router 三角色（**vLLM** 服务）。超参：事件切分阈值 0.5、最短 60s、字幕 1 FPS、片段 30s 不重叠；在线 T=5、0.1 FPS、每事件 8 帧、FIFO 16。

### 4.2 训练流程（若需要训练）

**不训练任何模型权重，也不训练 Router。** 下表说明"数据/模型如何构造或调用"。

| 阶段 | 目标 | 构造/调用 | 数据来源 | 数据形态 | 判定方式 |
|---|---|---|---|---|---|
| ① 离线索引 | 建可复用的双轨文本记忆 | 冻结 Qwen3-VL-8B | 原始视频流（question-agnostic） | 事件级叙事档案 + 30s 片段微日志 | 无 loss；按配置指纹缓存（作者承认是瓶颈） |
| ② 在线推理 | 预算内作答 | 冻结 Qwen3-VL-32B（三角色） | ①缓存 + 查询 + 按需帧 | 多模态 prompt；Router 输出 need_visual 布尔 | 无 loss；终止于 conf=true 或 T 次后 must-answer |
| ③ 路由诊断 | 评估 Router 触发是否合理 | 配对重跑（no-frame vs always-frame） | Video-MME 全 2,700 题 | 每题两策略结果对比 | **outcome oracle**：仅当 always-frame 修正 no-frame 才算"帧有益" |

> Limitations 承认二者均在 **generic zero-shot** 下运行，"很可能受益于领域特定的监督或轻量微调"，建议从 oracle 轨迹蒸馏得到**连续校准分数**——**训练 Router 是明确留出的未来工作。**

### 4.3 推理流程（若不训练或重点在推理）

多步循环（T≤5）为"故事优先 → 定位 → 文本富化 → 路由门控 → 抽帧 → 多模态重答"的规划-执行-回退混合体，Router 是唯条件分支点。视觉预算三处硬封顶（每事件 8 帧、FIFO 16、Router 门控），故每查询视觉成本与视频总长无关。附录另测 InternVL3.5-8B+38B（64.3）与 Qwen3.5-9B+27B（69.4）。

---

## 5. Benchmark 与实验设置

### 5.1 Benchmark 一览表

| Benchmark | GUI 场景 | 任务类型 | 数据规模 | 输入格式 | 输出格式 | 评测指标 |
|---|---|---|---|---|---|---|
| Video-MME | General | 多选 QA（6 主域/30 子类） | 900 视频 / 2,700 题；短<2min、中 4–15min、长 30–60min | 视频（无字幕）+问题+选项 | 选项字母 | 分段与总体准确率 |
| InfiniBench | General（长片） | 4 个 grounding skill 的 MCQ | 均 53 分钟；全库 87.7K QA，仅取 4 个 skill | 同上 | 选项字母 | 分类准确率 |
| LVBench（附录 D） | General（小时级） | 6 维度 | 103 视频、117 小时、均 4,101s；1,549 题 | 同上 | 选项字母 | 6 维度与 overall |


### 5.2 实验结果分析

**Video-MME**：CFD **67.5 overall**（short 72.2 / medium 66.6 / long 63.6），均 **5.8 帧/问题**；Qwen3-VL-32B 的 75.9 需 **768 帧**，少两个数量级——论文明确**不追平 dense 上界**，只求有界预算下的精度-效率前沿。对比 agent-based（VideoLucy 64.7、MemVid 64.0、VideoTree 60.6、VideoAgent 46.4、DrVideo 51.7）全部匹配或超过。字幕平均覆盖 **233.1 秒**，且 **Re-caption? = ✗**。

**InfiniBench**：**14.8 帧/问题**；Chronological **55.10**（超 768 帧 dense 的 48.44）、Global Appearance 58.90、Scene Transitions 52.40、Character Actions 56.20；后两项领先 agent-based 但低于 dense。

**LVBench**：**52.9 overall**，超过全部开源 agent-based（MemVid 44.4、VCA 41.3、VideoAgent 29.3、VideoTree 28.8），与 AdaReTaKe-72B 的 53.3 持平（VideoLucy 58.8 用了 DeepSeek-R1 作 base）。最强维度 Key Information Retrieval **63.9**。**三 benchmark 超参完全相同。**

**帧预算与路由（核心数字）**：即使关掉 Router、每个定位事件都取帧，该流水线也把在线帧用量从 **768 降到 6.63 帧/查询**（**99.1% 更少、约 116× 缩减**）。Router 进一步降到 **5.76**（再降 **13.1%**），精度从 67.2 提到 **67.5**（+0.3）。预算敏感性：2/4 → 8/16 → 32/64 时帧用量 1.83 → 5.76 → 8.93，overall 只变 **0.4%**（67.2/67.5/67.6）。策略对比（精度/帧数）：No frames 65.3/0.00、**Router 67.5/5.76**、Always frames 67.2/6.63。Router 对 **46.2%** 的问题取帧，捕获 127/133（**95.5%**）的真受益案例，oracle 为 69.5% @ 0.64 帧/查询；片段字幕激活从 34.49 降到 **11.62** 条（**-66.3%**）。

**其他消融**：离线 captioner 2B→8B 大幅提升（62.6→67.5），到 32B 对长视频反退化（61.0 vs 63.6，归因"冗长累积"）；云侧推理模型 2B 49.7 → 8B 58.9 → 32B 67.5 单调递增；短视频 T=2 即饱和（71.9），长视频从 T=1 的 60.4 升到 T=20 的 64.0；serving time 为 no-frame 59.6s < always-frame 63.3s < Router **68.0s**。

---

## 6. 亮点与贡献（Why it matters）

1. **把"何时看像素"变成一等公民**：以往系统要么总用帧要么从不用帧；CFD 让"是否取帧"成为显式、可调优的变量。
2. **"预算敏感性极低"**：8× 预算只带来 0.4% 精度变化——长上下文 agent 里，钱花在**选择机制**上比花在**容量**上更值。
3. **反直觉但可信的消融**：Router 提精度靠"在时序问题上不看帧"（Chronological 51.70→55.10）。
4. **caption-once 摊销结构工程友好**：离线 229.3 s/video、在线 68.2 s/query 分离，加跨查询复用缓存，"一次索引、多轮问答"边际成本低。

---

## 7. 局限与可改进点（个人点评）

1. **Router 触发 46.2% vs 真受益 4.9%，差近一个数量级，论文却定性为"feature 而非 bug"**。二元门控的判别能力未被评估（无 precision/recall、无随机对照）；相对 always-frame 只省 13.1% 帧、只多 0.3 分。应按作者建议做连续校准分数并报告路由 ROC。
2. **oracle 自我参照**："always-frame 修正 no-frame 错误"把"帧有益"限定在本系统两策略差异上，漏掉"两策略都错但帧能救"的案例，也混淆了"帧有用"与"多一次 LLM 调用的随机性"，故两个比例都应打折扣。
3. **字幕质量是单点瓶颈却无量化**：论文承认关键视觉细节若未被 question-agnostic 记忆捕获就无法恢复，却没给漏检率，也没做"用 ground-truth 描述替换 captioner"的上界实验。
4. **"edge-cloud" 卖点基本未验证**：作者自承是模拟分离，通信开销与延迟变异性未刻画；captioner 是 8B，目标是单 GPU 入门服务器。
5. **省帧不等于省延迟**：Router 多一次 LLM 调用，端到端 serving time 反而最慢（68.0s vs always-frame 63.3s）；若瓶颈是延迟而非带宽，Router 就是负优化。它在属性重的 skill 上还**欠分配帧**，说明时序/感知二分在混合型问题上会系统性失手。
6. **评测全为多选、单一语言、无开放式与多轮交互**：CFD 最大优势"缓存索引跨查询复用"恰在多轮下才最能体现，论文却没测；也未与 Video-XL、LongVU 等压缩路线同协议 head-to-head。

---

## 8. 对我们的启示 / 可借鉴点

- **离线摊销 + 在线精简是端侧 agent 通用范式**：任何"同一份输入被反复查询"的场景（个人记忆、知识库、GUI 状态）都值得把 question-agnostic 索引一次做好、缓存复用。
- **用纯文本组合已缓存内容替代重新调用模型**：片段富化零视觉成本，多一层记忆不必多一次模型调用。
### 对 GUI Agent 的可借鉴点

**核心启发：把"要不要重新截图 / 看像素"变成显式预算，而不是每一步都截图。** 何时才需要看像素（把 CFD 的 perceptual vs temporal 二分改造成 GUI 的 perceptual vs structural 二分）：

1. **必须截图**：子目标涉及**属性判定**——颜色、图标、屏上文字、字体/尺寸、布局、遮挡，以及"动作后元素样式是否改变"（按钮 disabled→enabled、模态框出现）；这些在 AXTree/DOM 里常缺失。
2. **不必截图**：任务是**结构/流程性**的——上一步做了什么、还剩几步、下一元素引用已知、标题与 URL 已确认；用文本日志、AXTree 摘要与动作历史即可推理。
3. **用引用代替重看**：元素获得稳定引用（DOM selector、AX node id、缓存坐标）后直接复用，只在引用失效（重渲染、导航、元素消失）时重新观察；不可逆动作（提交、保存、确认订单）前才全量观察一次。
4. **预算耗尽时强制作答**：对应 must-answer fallback，治"截图—点击—失败"循环。

**可迁移设计**：① FIFO 视觉工作记忆：最近 N 张截图入定容队列、更早的只留文本摘要；② 双轨记忆："任务级 SOP/事件骨架 + 步骤级微日志"，前者全量注入、后者按阶段注入；③ 视觉需求门控先用规则 + 一次 LLM 判定，再按 future work 训练连续校准分数；④ **oracle-style 配对诊断标定阈值**——同批任务跑"每步都截图"与"只按 Router 截图"，看截图到底修正了哪些错误。

---

## 9. 延伸阅读

- **直接前作与对比对象**：VideoLucy（arXiv:2510.12422）；VideoTree；DrVideo（CVPR 2025）；MemVid（arXiv:2505.15872）；VideoAgent（ECCV 2024）。
- **caption-once 范式源头**：Zhang et al. 2024a；Kahatapitiya et al. — *Language Repository for Long Video Understanding*（Findings of ACL 2025）。
- **视觉-文本对偶性实证**：CaptionQA（Yang et al. 2025a）；InfiniBench（EMNLP 2025）。
- **视觉压缩路线（未 head-to-head）**：Video-XL；LongVU（NeurIPS 2024）；VideoChat-Flash（ICLR 2026）。
- **边缘/可穿戴**：EgoTrigger（IEEE TVCG 2025）；Neurosurgeon（2017）。
- **基准与基础模型**：Video-MME（CVPR 2025）；InfiniBench；LVBench；Qwen3-VL Technical Report（arXiv:2511.21631）；vLLM（Kwon 2025）。

---
*解读生成时间：2026-09-12 ｜ 解读人：WorkBuddy（AI）*
