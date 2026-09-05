# 📖 当日论文速览 · 2026-09-05

> 检索窗口：arXiv 近 3 日新增 ＋ GitHub Awesome 列表增量补充 ｜ 本日入选 **3 篇**并完成精读解读（1 篇 GUI 域 + 2 篇 Agent 方法论域·跨域参考）
> 主题一句话：**GUI Agent 的"停手能力" + Agent 方法论双视角** —— 一篇把可靠性从"做得到"延伸到"该不该做"（冲突识别停手）；另两篇来自通用 Agent 域（贝叶斯世界模型愿景、主动服务决策框架），为 GUI Agent 提供跨域思路参照
> 标签图例：`类型标签`（论文类别）｜ `训练方法标签`（方法级）｜ 域标注：GUI / Agent 方法论·跨域参考

---

## 📄 今日论文

### 1. Do GUI Agents Know When Not to Act? Enabling Conflict-Aware Termination for Multimodal GUI Agents
- **域**：GUI ｜ **标签**：`GUI Grounding` `Benchmark` `Mobile` ｜ 训练方法：—（方法/评测：推理期可行性验证 + 条件激活引导，不训练模型参数）
- **一句话**：发布 CONFLICTGUI 冲突基准（2,364 可行 + 1,122 指令内冲突 + 1,174 指令-界面冲突，由 AMEX/AndroidControl/AITZ 加 VLM 注入冲突构建）并实证主流 GUI agent 的"执行偏向型过度顺从"；提出的 CONFLICTGUARD 把 5 个开源模型平均冲突成功率从 **6.91% → 58.63%**、假执行率 73.37%→32.76%，可行成功率仅 −2.6 点（Qwen3-VL-32B 冲突 SR 达 77.79%），推理零延迟开销，并可在 VenusBench-GD 上迁移出 +53.95~+73.42 点增益。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.03438) ｜ 💻 [Code](https://github.com/serein356/ConflictGuard)（✅ 已开源，代码+数据）
- 📝 **[阅读完整解读](./2026-09-03_Do_GUI_Agents_Know_When_Not_to_Act_Enabling_Conflict-Aware_Termination_for_Multimodal_GUI_Agents.md)**

### 2. Semantic Bayesian World Models
- **域**：Agent 方法论·跨域参考 ｜ **标签**：`General` `Planning` ｜ 训练方法：—（position/vision 论文，无训练/实验）
- **一句话**：position 文主张用"本体即先验 + 贝叶斯更新 + 动作干预（do-演算）"构建语义贝叶斯世界模型，把知识图谱从布尔事实库升级为可交换的信念网；为 GUI Agent 的启示是**显式建模界面状态不确定性**——"观察层别丢概率"，将截图/AXTree 的歧义（如模糊控件）建模为带不确定性的信念而非硬断言。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.03834) ｜ 代码：未声明开源
- 📝 **[阅读完整解读](./2026-09-03_Semantic_Bayesian_World_Models.md)**

### 3. Proactive Service Agents: A Unified Decision Framework, Methods, and Evaluation
- **域**：Agent 方法论·跨域参考 ｜ **标签**：`General` `Planning` `Benchmark` ｜ 训练方法：—（综述，不训练/不评测自家模型）
- **一句话**：把"主动服务"统一为受授权与风险硬约束的"沉默/询问/协助/执行"四模式 POMDP 门控决策，指出离线分类分数≠部署收益并给出带三轴证据描述符的评测协议；对 GUI Agent 的启示是**何时该主动询问用户/何时该停等授权**这类"交互时机"问题应被显式决策化，而非靠提示词碰运气。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.03727) ｜ 代码：未声明开源
- 📝 **[阅读完整解读](./2026-09-03_Proactive_Service_Agents_A_Unified_Decision_Framework_Methods_and_Evaluation.md)**

---

## 📊 本日趋势归纳（按标签与域）

- **域结构（双域首日）**：GUI 1 篇 + Agent 方法论 2 篇。Agent 方法论论文虽非 GUI 落地，但解读中均单列"对 GUI Agent 的可借鉴点"——两条跨域线索：① 显式建模**不确定性/信念**（贝叶斯世界模型）；② 把**交互时机**（问/等/做）决策化（主动服务框架）。二者都可迁移到 GUI 的"何时停、何时问、多确信"问题。
- **类型分布**：`Benchmark`（2）+ `Planning`（2）+ `GUI Grounding`（1）+ `General`（2）+ `Mobile`（1）——今日无 RL/Online/Desktop/Web 新论文入选。
- **主题信号：可靠性从"做得到"延伸到"该不该做"**。前几日我们关注"执行阶段"的进步（Grounding 蒸馏、SOP 工程化、世界模型规划、运行期失败监控）；今日 GUI 域这篇把问题移到更前端：**执行任何动作前先判断指令是否可执行**。它与 09-04 的《Monitoring Web Agents Without Internal Signals》形成互补链：**监控侧可纯黑盒判断"是否已走偏"，干预侧目前仍需白盒隐状态做激活引导**（本文明确承认对闭源 API 不适用）——"何时停"正成为继 Grounding/Planning 之后的独立研究维度。
- **方法学上"轻干预"趋势延续**：连续四天入选论文多倾向不训练/少训练的增量干预（SOP 工程、提示+表征工程、推理期门控），与 Efficient GUI Agents 综述指出的"运行时成本控制"判断互相印证。
- **GitHub 两个精选列表近 7 天无新增**（维护滞后，符合预期）。

---

## 🔧 检索通道说明（今日特殊 + 双域升级）

- 08:30 自动任务执行时 arXiv export API 仍不可达（HTTP 429 / 读超时），走**回退双通道**：网页搜索 + cs.AI/LG/HC/CL/SE 的 9 月 4 日公告全量 listing 交叉验证（361 篇），命中 GUI 论文 1 篇（2609.03438）。
- **10:00 双域升级补录**：项目规则升级为"GUI 域 + Agent 方法论域"双域检索（关键词扩至 16 组，`search_arxiv.py`/`search_arxiv_web.py` 同步更新，结果带 `domain=gui|agent` 标记，每日上限 15 篇）。API 此时已恢复，用升级脚本补跑 Agent 域并核验原始提交日（排除 6 月旧论文 CoMAP 的 v2 修订等），补录 2 篇窗口内 Agent 方法论论文进入解读。完整过程见 `logs/2026-09-05_检索日志.md`。

---

*生成时间：2026-09-05 ｜ 由 WorkBuddy「GUI 论文每日检索与解读」工作流自动生成*
