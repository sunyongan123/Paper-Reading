# 📖 当日论文速览 · 2026-09-05

> 检索窗口：arXiv 近 3 日新增（API 仍不可达，走网页搜索 + 分类 listing 交叉验证回退）＋ GitHub Awesome 列表增量补充 ｜ 本日入选 **1 篇**并完成精读解读
> 主题一句话：**GUI Agent 的"停手能力"** —— 当指令本身矛盾或与界面不符时，主流 agent 会过度顺从地盲执行（可行任务成功率 >70%，冲突成功率 <10%）；CONFLICTGUARD 用"可行性验证 + 条件激活引导"在推理期把冲突成功率拉到近六成且不伤可行任务。
> 标签图例：`类型标签`（论文类别）｜ `训练方法标签`（方法级）

---

## 📄 今日论文

### 1. Do GUI Agents Know When Not to Act? Enabling Conflict-Aware Termination for Multimodal GUI Agents
- **标签**：`GUI Grounding` `Benchmark` `Mobile` ｜ 训练方法：—（方法/评测：推理期可行性验证 + 条件激活引导，不训练模型参数）
- **一句话**：发布 CONFLICTGUI 冲突基准（2,364 可行 + 1,122 指令内冲突 + 1,174 指令-界面冲突，由 AMEX/AndroidControl/AITZ 加 VLM 注入冲突构建）并实证主流 GUI agent 的"执行偏向型过度顺从"；提出的 CONFLICTGUARD 把 5 个开源模型平均冲突成功率从 **6.91% → 58.63%**、假执行率 73.37%→32.76%，可行成功率仅 −2.6 点（Qwen3-VL-32B 冲突 SR 达 77.79%），推理零延迟开销，并可在 VenusBench-GD 上迁移出 +53.95~+73.42 点增益。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.03438) ｜ 💻 [Code](https://github.com/serein356/ConflictGuard)（✅ 已开源，代码+数据）
- 📝 **[阅读完整解读](./2026-09-03_Do_GUI_Agents_Know_When_Not_to_Act_Enabling_Conflict-Aware_Termination_for_Multimodal_GUI_Agents.md)**

---

## 📊 本日趋势归纳（按标签）

- **类型分布**：`Benchmark`（1）+ `GUI Grounding`（1）+ `Mobile`（1）——今日无 RL/Online/Planning/Web/Desktop 新论文入选。
- **主题信号：可靠性从"做得到"延伸到"该不该做"**。前两日（09-03/09-04）我们关注的是"执行阶段"的进步——Grounding 蒸馏、SOP 工程化、世界模型规划、运行期失败监控；今日这篇把问题移到更前端：**在执行任何动作之前，先判断指令是否可执行**。它与 09-04 的《Monitoring Web Agents Without Internal Signals》形成一条互补链：**监控侧可以纯黑盒判断"是否已走偏"（只需可观测信号），而干预侧目前仍需白盒隐状态做激活引导（本文明确承认对闭源 API 不适用）**——两条合起来提示：GUI Agent 的"何时停"正在成为继 Grounding/Planning 之后的独立研究维度。
- **方法学上的"轻干预"趋势在延续**：09-03 的 OmegaUse-SOP 用提示级 SOP 工程、09-04 两篇都不重训主干，今日这篇用"提示 + 表征工程"的组合在推理期改行为——连续三天入选论文均倾向**不训练/少训练的增量干预**，这与 Efficient GUI Agents 综述指出的"运行时成本控制"判断互相印证。
- **与前日对照的可借鉴点**：可行性验证提示 ≈ 零成本（部分模型 +18~+46 点冲突 SR）；条件门防"普遍拒答"的设计对做 RL/SFT 数据配比也有启发（不要因噎废食地惩罚可行执行）。
- **GitHub 两个精选列表近 7 天无新增**（ZJU-REAL 最新合入 2026-07-28 的 ClawBench；OSU 最新条目为 2026-08-12 CoAdapt-GUI，已解读）——维护滞后，符合预期。

---

## 🔧 检索通道说明（今日特殊）

arXiv export API 端点今日仍整体不可达（HTTP 429 / 读超时，8 组关键词全部失败），沿用 09-04 建立的**回退双通道**：① `tools/search_arxiv_web.py` 网页搜索命中窗口内 3 篇相关论文（其中 OmegaUse-SOP 已解读、PalmClaw 属 7 月论文 v2 修订记入 backlog）；② 对 cs.AI/cs.LG/cs.HC/cs.CL/cs.SE 的 **9 月 4 日（Fri）公告组**做全量 listing 交叉验证（含 ?skip= 翻页，共 361 篇唯一论文），标题级过滤命中 GUI Agent 论文 1 篇（2609.03438），与网页搜索互为印证。另复核 cs.HC/cs.SE 全量标题，其余 UI 相关条目（Declarative UI Generation / Editable Visual Design / GUI 设计心理学）均偏离"GUI Agent 操作"主题，未入选（详见 `logs/2026-09-05_检索日志.md`）。

---

*生成时间：2026-09-05 ｜ 由 WorkBuddy「GUI 论文每日检索与解读」工作流自动生成*
