# 📖 当日论文速览 · 2026-09-06

> 检索窗口：arXiv 近 3 日新增 ＋ GitHub Awesome 列表增量补充 ｜ 本日入选 **3 篇**并完成精读解读（均为 Agent 方法论域·跨域参考）
> 主题一句话：**"RL 信号去噪"专题** —— 三篇全部指向让 LLM/Agent 的强化学习训练更干净、更省、更准：拆穿 GRPO 的"伪优势"、为 GRPO 旧轨迹复用立规矩（Headroom-Drift Replay）、用"必要工具证据路径"奖励根治 Agentic VLM 的乱调用工具（NTEP）
> 标签图例：`类型标签`（论文类别）｜ `训练方法标签`（方法级）｜ 域标注：GUI / Agent 方法论·跨域参考

---

## 📄 今日论文

### 1. Spurious Advantage Hidden in GRPO
- **域**：Agent 方法论·跨域参考 ｜ **标签**：`RL` `General` ｜ 训练方法：`RL (GRPO)`（替换 GRPO 的 advantage 估计器做 RLVR，不新增训练阶段）
- **一句话**：点破 GRPO 优势估计中"蒙对答案也拿高梯度权重"的**伪优势**（有界答案/隐藏有界子集/search agent 三类场景），提出无参数估计器 SIGNBALANCE（保留 verifier 符号 + 全局尺度 + stop-gradient 类别力平衡）；0.5B 数学 Avg-8 **36.61 vs GRPO 34.24**、SAT-Math +6.26、3B 规模扩展 Avg-8 43.78 领跑，7B 搜索智能体 Avg-6 **37.80 vs Search-R1 36.00**，多跳最重的 2WikiMultiHopQA +7.62。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.04063) ｜ 代码：摘要声明将开源，文中未给仓库链接
- 📝 **[阅读完整解读](./2026-09-03_Spurious_Advantage_Hidden_in_GRPO.md)**

### 2. Headroom-Drift Replay: A Primitive for Principled Replay Control in GRPO
- **域**：Agent 方法论·跨域参考 ｜ **标签**：`RL` `Online` ｜ 训练方法：`RL (GRPO)`（组级回放增广的在线 RL 后训练，GRPO 主更新式不变）
- **一句话**：把 GRPO 的旧轨迹复用拆成两个独立决策——**Headroom 按剩余学习价值排序 + Drift 按与当前策略兼容性门控**，不新增任何生成即在数学/多模态/Agentic Search 三域击败朴素回放（Agentic Search Mean@32 **0.3577 vs 0.3548**、Best@32 0.4879 vs 0.4623），并对 on-policy scaling 构成墙钟帕累托改进；针对"agentic 环境 rollout 成本极高"这一痛点给出可即插即用的回放原语。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.03941) ｜ 代码：未在文中声明开源
- 📝 **[阅读完整解读](./2026-09-03_Headroom-Drift_Replay_A_Primitive_for_Principled_Replay_Control_in_GRPO.md)**

### 3. Making Every Tool Call Count: Necessary Tool-Evidence Path Rewards for Agentic Vision-Language Models
- **域**：Agent 方法论·跨域参考 ｜ **标签**：`RL` `Planning` `General` ｜ 训练方法：`RL (GRPO + NTEP 过程奖励)`（token 级 group-relative advantage、无 critic，含两阶段教师蒸馏式 NTEP 标注）
- **一句话**：提出 **NTEP（必要工具-证据路径）** 标注方案 + 路径级过程奖励（意图对齐 α、信息摄取 β、去重复目标正则），把 agentic VLM 的工具监督从"只看最终答案"升级为"每次调用是否推进必要证据"；NTEP-8B 在 7 个图像基准平均 **70.34（超最强同框架基线 +2.03）**、冗余调用 13.5%→1.0%、ATC 2.54→1.89（搜索类），并能零训练迁移到未见工具。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.03493) ｜ 代码：未在文中声明开源
- 📝 **[阅读完整解读](./2026-09-03_Making_Every_Tool_Call_Count_Necessary_Tool-Evidence_Path_Rewards_for_Agentic_Vision-Language_Models.md)**

---

## 📊 本日趋势归纳（按标签与域）

- **域结构**：Agent 方法论 3 篇（GUI 0 篇）。今日 arXiv 无新公告组（最新仍为 Fri, 4 Sep 2026，周末停更），本日入选的 3 篇是**用完整 16 组双域关键词对 09-03 公告组做摘要级全量甄别后捞回的昨日遗漏**——昨日 listing 标题级过滤+agent 补跑（7 篇）未覆盖到这三篇标题不含 GUI/agent 词的 RL 方法论论文。
- **类型分布**：`RL`（3）+ `General`（2）+ `Planning`（1）+ `Online`（1）——连续第二天 RL 域 0 GUI 新论文，但 Agent 方法论域把 RL 训练技术补给补上了。
- **主题信号：RL 训练信号的"精度治理"成为热点**。三篇同属"给 RLVR 信号去噪"族但角度各异：① advantage 估计器层面的**伪优势**（本文 1：猜对的 rollout 不该拿高权重）；② 训练数据复用层面的**回放控制**（本文 2：旧轨迹要不要用、用多少由价值与漂移双闸门决定）；③ 奖励信号层面的**过程级路径奖励**（本文 3：中间工具调用是否在获取必要证据，而非只看结局）。对 GUI Agent 的 RL 训练（屏幕轨迹长、动作空间有限、环境交互贵）三者分别对应"防猜对点击被放大""省 rollout 成本""让 grounding/工具调用有过程监督"。
- **方法学信号：全部是"手术刀式"改动而非新训练范式**。SIGNBALANCE 是 GRPO 内一行替换级估计器、HDR 是即插即用回放原语、NTEP 是奖励构造+标注协议——都强调不引入额外模型/生成、可与现有管线正交叠加，延续 Efficient GUI Agents 综述与近日解读指出的"成本可控的轻干预"取向。
- **GUI 关联提示**：三篇解读均单列"对 GUI Agent 的可借鉴点"。最直接的迁移是：GUI 每步动作本质是"有限候选集上的选择"（与多选题同构），GRPO 伪优势几乎必然存在——SIGNBALANCE 是低成本优先尝试项；NTEP 的"工具调用-证据路径"奖励则与 GUI agent 的截图裁剪/AXTree 读取/元素定位这些"取证据动作"一一对应。
- **GitHub 两个精选列表近 7 天无新增**（维护滞后，符合预期）。

---

## 🔧 检索通道说明（今日特殊）

- 08:30 自动任务执行：arXiv export API **部分可用**（8 组 GUI_KW 全部成功 + 4 组 AGENT_KW 超时），命中 37/相关 31；随后用网页回退通道（search_arxiv_web.py）补齐超时的 4 组，窗口内命中 0；**分类 listing 交叉验证确认最新公告组仍为 Fri, 4 Sep 2026（与 09-05 相同，arXiv 周末不发布新公告）**，即今日无 09-04 之后提交的新论文。
- **候选来源**：因无新公告组，对 09-03 公告组（昨日已处理组）用 API 全量摘要做二次甄别，从 31 篇相关中剔除：昨日已解读 3 篇（2609.03438/03834/03727）+ 主题偏离（UI 生成/设计）+ 具体应用噪声（VLN 导航、生物医药、OS 内核验证、安全运营、扫描探针、应急疏散等），**捞回昨日遗漏的 3 篇高价值 RL 方法论论文**（2609.04063/03941/03493）。PDF 3/3 下载成功。完整过程见 `logs/2026-09-06_检索日志.md`。

---

*生成时间：2026-09-06 ｜ 由 WorkBuddy「GUI 论文每日检索与解读」工作流自动生成*
