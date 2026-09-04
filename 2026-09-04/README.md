# 📖 当日论文速览 · 2026-09-04

> 检索窗口：arXiv 近 3 日新增（API 不可用，走网页搜索 + 分类 listing 交叉验证通道）＋ GitHub Awesome 列表增量补充 ｜ 本日入选 **2 篇**并完成精读解读
> 主题一句话：**Web Agent 走出"下一步动作"范式** —— 一篇用判别式世界模型服务测试时规划（Planning/RL），一篇用纯可观测轨迹信号做在线失败风险监控（Online/监控）
> 标签图例：`类型标签`（论文类别）｜ `训练方法标签`（方法级）

---

## 📄 今日论文

### 1. Discriminative World Models for Web Agents
- **标签**：`Planning` `RL` `Web` `Benchmark` ｜ 训练方法：`RL (GRPO)`
- **一句话**：用"判别式状态匹配"训练 Web 世界模型——预测必须能区分真实后果与其它候选动作的后果（Qwen3-8B + GRPO，数据来自 Go-Browse 分支图）；留出集匹配 **80.80%** vs 同数据监督式 SFT 仅 47.77%，WebPRMBench 平均 BoN 55.80%→**72.70%**，把 WebArena-Lite 端到端成功率从 Best-of-5 的 21.82% 抬到 **28.48%**。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.02885) ｜ 💻 [项目主页](https://dhruvpendharkar.github.io/dwm/)（部分开源，暂无代码仓库链接）
- 📝 **[阅读完整解读](./2026-09-02_Discriminative_World_Models_for_Web_Agents.md)**

### 2. Monitoring Web Agents Without Internal Signals: Observable Trajectories and Key-Step Supervision
- **标签**：`Web` `Online` `Benchmark` ｜ 训练方法：—（监控/评测方法研究）
- **一句话**：不依赖 logits/隐状态等内部信号，仅凭可观测轨迹（Macro 行为统计 + Micro 重复采样一致性）+"关键步"时间对齐监督，即可对闭源 Web agent 做前缀级失败风险预测：20% 误切预算下检测率 **44.3%/44.5%**，超需内部信号的 HTC-Full（41.9%/40.4%）。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.02057) ｜ 代码：未开源
- 📝 **[阅读完整解读](./2026-09-02_Monitoring_Web_Agents_Without_Internal_Signals_Observable_Trajectories_and_Key-Step_Supervision.md)**

---

## 📊 本日趋势归纳（按标签）

- **训练方法**：`RL (GRPO)`（世界模型判别式目标）是本日唯一显式训练方法；第 2 篇属评测/监控方法研究（不训练策略）
- **领域重心回到 `Web`，且都绕开"直接预测下一动作"**：第 1 篇把世界模型训练目标从"复述 HTML/AXTree"改为"能否帮下游 ranker 区分候选动作"（训练-使用目标对齐）；第 2 篇则把注意力从"怎么让 agent 更聪明"转向"运行中如何低成本判断它是否已走偏"。两条线合起来是 Web agent 走向可靠部署的典型信号：**规划要先于动作（think before act），监控要跟得上执行（watch while act）**。
- **方法学呼应昨日**：与 CA-OPD 的"对齐下游目标"、Efficient GUI Agents 综述指出的"反思/验证成本"同一脉络——今日两篇分别从"训练目标对齐"与"运行时成本控制（提前止损）"两端呼应。
- **纯 GUI Grounding/Desktop/Mobile 今日无新增**；GitHub 两个精选列表近 7 天亦无新增（维护滞后）。

---

## 🔧 检索通道说明（今日特殊）

arXiv export API 端点今日整体不可达（HTTP 429 / SSL 重置 / 超时，WebFetch 亦超时），已启用**网页搜索回退**并新增工具 `tools/search_arxiv_web.py`；另对 cs.AI/cs.LG/cs.HC/cs.CL/cs.SE 的 9 月 3 日分类公告做了全量 listing 交叉验证，补回 2 篇 API 关键词口径易漏的 "Web Agents" 论文（详见当日检索日志 `logs/2026-09-04_检索日志.md`）。

---

*生成时间：2026-09-04 ｜ 由 WorkBuddy「GUI 论文每日检索与解读」工作流自动生成*
