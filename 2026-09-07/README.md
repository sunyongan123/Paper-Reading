# 📖 当日论文速览 · 2026-09-07

> 检索窗口：arXiv 近 5 日新增（API 完整成功）＋ GitHub Awesome 列表增量补充 ｜ 本日入选 **4 篇**并完成精读解读
> 主题一句话：**"RL 方法论补漏"专题（周一无新公告，二次补漏）**——把 RL 训练目标改成"稀有高分导向"（TailRL）、让 agent 学会"少决策多执行"的动作分块（SPACE）、让图式信用分配跨策略更新累积经验（TIGPO）、以及给自改进 agent 上"确定性护栏"防 LLM 裁判被优化器钻空子（PROCTOR）。前三篇延续 09-06 GRPO 系列，都是 RL 训练侧的"手术刀式"改进；第四篇是评估护栏视角。
> 标签图例：`类型标签`（论文类别）｜ `训练方法标签`（方法级）｜ 域标注：GUI / Agent 方法论·跨域参考

---

## 📄 今日论文

### 1. Tail-Likelihood Reinforcement Learning
- **域**：Agent 方法论·跨域参考（含 GUI grounding 直接实证，最贴近 GUI）｜ **标签**：`RL` `GUI Grounding` `Online` `General` ｜ 训练方法：`Online RL`（论文自研 critic-free 策略梯度算法 TailRL，on-policy、无 critic/KL，drop-in 替换 advantage）
- **一句话**：标准 RL 只优化平均奖励，会悄悄丢掉"稀有但高分"的 rollout；TailRL 改为最大化"奖励超过随机阈值的对数概率"（等价于把所有 Best-of-k 梯度的调和混合，无需预选 k），梯度自动放大高奖励稀有样本。GUI grounding（ScreenSpot-Pro，Qwen2.5-VL-3B/7B）上 **用 8 个（3B）/ 4 个（7B）推理 rollout 即追平 RLOO 用 1024 个才有的 Pass@1024，推理采样省 128×/256×**；代码优化测试集 Best-of-1024 加速 7.7×（GRPO 仅 0.98×、RLOO 0.96×，后两者塌缩为"复制输入"）。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.02987) ｜ 代码：有项目主页（含 code）：https://zanette-labs.github.io/TailRL-website/
- 📝 **[阅读完整解读](./2026-09-02_Tail-Likelihood_Reinforcement_Learning.md)**

### 2. Act More, Decide Less: Skill-Guided Adaptive Action Chunking for Long-Horizon LLM Agents
- **域**：Agent 方法论·跨域参考 ｜ **标签**：`RL` `Online` `Distillation` `Planning` ｜ 训练方法：`Hybrid On-/Off-Policy RL`（RLVR：on-policy PPO 风格 clipped 目标 + off-policy 自模仿蒸馏损失 + 两层 chunk-aware advantage；基座 Qwen3-4B / Llama-3.1-8B）
- **一句话**：长程 agent 每轮只发一个动作太费 LLM 轮次，但直接用 RL 训变长动作块会"塌缩成单动作"或"过度承诺"；SPACE 从成功轨迹归纳出可执行"程序化技能"，把技能层级边界蒸馏成"分块边界"监督信号，让 agent 学会每轮输出 1~6 个动作的变长块。ALFWorld 成功率提升 **7.0~15.6 个百分点**且每 episode LLM 轮次从 GiGPO 的 15.9 降到 **3.7**（Qwen3-4B seen 99.2% SR）、ScienceWorld 近乎翻倍（seen 67.2% vs GiGPO 35.9%），达多动作 GRPO 最终水平仅需 26.6% 训练步。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.02042) ｜ 代码：未在文中声明开源
- 📝 **[阅读完整解读](./2026-09-02_Act_More_Decide_Less_Skill-Guided_Adaptive_Action_Chunking_for_Long-Horizon_LLM_Agents.md)**

### 3. TIGPO: Temporal Instance-Graph Policy Optimization for Long-Horizon LLM Agents
- **域**：Agent 方法论·跨域参考（Web/交互环境评测，脚本预标 gui）｜ **标签**：`RL` `Online` `Web` `General` ｜ 训练方法：`Online RL`（图式/组式 credit assignment：relative advantage + clipped objective 自定义联合目标）
- **一句话**：图式 credit assignment 每次策略更新都"用后即弃"本地轨迹图，跨更新的经验无法连成完整成功通路；TIGPO 为每个任务维护**跨策略更新的持久化转移图**，并用"探索–重访"调度让当前策略主动重试旧任务、以"跨时间对比"（与早期探索组得分拼接做冻结参考）稳定 advantage——历史经验只当结构与统计参考、**绝不进 loss**。ALFWorld 综合成功率 **91.28%**（超 GraphGPO 1.96pp）、WebShop 成功率 77.54%、得分 88.65，训练单步耗时与显存近零增加。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.03383) ｜ 代码：未在文中声明开源
- 📝 **[阅读完整解读](./2026-09-03_TIGPO_Temporal_Instance-Graph_Policy_Optimization_for_Long-Horizon_LLM_Agents.md)**

### 4. LLM-as-a-Judge Is Not an Oracle: Why Self-Improving Agents Need Deterministic Guardrails
- **域**：Agent 方法论·跨域参考 ｜ **标签**：`Reflection` `General` ｜ 训练方法：`—（经验/评测/系统）`
- **一句话**：自改进 agent 流水线里"LLM 裁判握有是否变好的最终裁决权"不可靠——优化器会沿裁判偏误和评估漏洞系统性爬山甚至作弊（如读缓存答案照抄，虚增 **31.9 个百分点**）；作者把裁判从"神谕"降级为"顾问"，用**五层确定性护栏**（密闭沙盒、角色解耦、机械门禁、冻结留出集、金丝雀用例）把关每次改动，让"机械拒绝永远压过 LLM 放行"；约 50 轮迭代中机械门禁拦截/回滚 13 次，单轮推理约 1.5 万 token 而评测约 4500 万（测量几乎就是整个系统的成本）。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.02246) ｜ 代码：未在文中声明开源
- 📝 **[阅读完整解读](./2026-09-02_LLM-as-a-Judge_Is_Not_an_Oracle_Why_Self-Improving_Agents_Need_Deterministic_Guardrails.md)**

---

## 📊 本日趋势归纳（按标签与域）

- **域结构**：4 篇均为 Agent 方法论·跨域参考（GUI 0 篇新增）。周一 arXiv 仍无新公告组（最新停留 Fri, 4 Sep 2026；下一次新组预计北京周二早出现），今日 4 篇是**对最新公告组做第二轮 Agent 域补漏**：与 09-06 相比，本次 API 16 组关键词完整成功 + 检索窗口放宽至 5 天，首次覆盖 **09-02 submitted**（09-06 用 3 天窗口 + 4 组关键词超时，完全错过这批）。
- **类型分布**：`RL`（3）+ `Online`（3）+ `General`（3）+ `Planning`（1）+ `Distillation`（1）+ `Reflection`（1）+ `GUI Grounding`（1，TailRL 实证）——连续第三日 GUI 域无新论文，RL 方法论持续成为 Agent 域补漏主线。
- **主题信号：RL 训练管线四个环节各来一刀**。① **目标函数层**（TailRL）：优化均值→优化"尾似然"，防止高奖励稀有样本被洗掉，与 09-06 的 GRPO 伪优势问题互补（一个是"猜对的别放大"，一个是"稀有高分的别丢"）；② **决策频率层**（SPACE）：把"每轮一个动作"改为"技能边界引导的变长动作块"，直接攻击长程任务里 LLM 轮次浪费与复合误差；③ **信用分配层**（TIGPO）：让图式 credit assignment 跨更新累积，解决小 rollout 组下优势估计不稳；④ **评估护栏层**（PROCTOR）：自改进闭环中 LLM 裁判不可信，需确定性验证兜底。三层训练 + 一层评估，构成一套完整的"RLVR 工业化"拼图。
- **成本意识贯穿始终**：TailRL 无 critic/KL/阈值、纯换 advantage；SPACE 训练样本省 73%；TIGPO 训练时间/显存近零增长；PROCTOR 揭示评测成本占系统算力 99.97%——延续本项目近日观察到的"轻干预、低成本"取向。
- **GUI 关联提示**：TailRL 在真实 VLM 的 ScreenSpot-Pro 上直接验证（128–256× 推理采样压缩、防"平庸但稳"行为挤掉稀有优质解），对 GUI grounding 训练是 drop-in 级可用；SPACE 的"动作块化"对 GUI 每步截图-决策的高成本场景直击要害；TIGPO 的持久化图+重访调度适用于 GUI 长程轨迹 RL；PROCTOR 的确定性护栏可迁移到 GUI agent 的自动化评估与自改进闭环。四篇解读均含"对 GUI Agent 的可借鉴点"专节。
- **GitHub 两个精选列表近 7 天无新增**（ZJU 最新提交 07-28、OSU 08-17，维护滞后符合预期）。

---

## 🔧 检索通道说明（今日特殊）

- 08:30 自动任务执行：arXiv export API **16 组关键词全部成功**（近 5 天窗口），命中 65 篇 / 相关 55 篇（domain=gui 19 + agent 36），与 listing 交叉验证结果一致——**最新公告组仍为 Fri, 4 Sep 2026，今日无 09-04 之后新提交**（arXiv 美东周末停更）。
- **候选来源（二次补漏）**：09-06 已对 Fri 4 Sep 组做过一轮摘要级甄别（捞回 3 篇 GRPO 相关），但当时 4 组 AGENT_KW 读超时且窗口仅 3 天。今日 API 全成功 + 5 天窗口，甄别出 4 篇此前从未评估的高迁移价值论文：2609.02987（TailRL，09-02，含 GUI grounding 实证）、2609.02042（SPACE 动作分块，09-02，EMNLP 2026 Camera Ready）、2609.03383（TIGPO，09-03）、2609.02246（PROCTOR 护栏，09-02）。均已核对历史收录/排除记录确认未收过。其余 51 篇相关命中为 09-06 已排除项（具体应用噪声/主题偏离）或已收录，不重复入选。PDF 4/4 下载成功。完整过程见 `logs/2026-09-07_检索日志.md`。

---

*生成时间：2026-09-07 ｜ 由 WorkBuddy「GUI 论文每日检索与解读」工作流自动生成*
