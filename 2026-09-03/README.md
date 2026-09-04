# 📖 当日论文速览 · 2026-09-03

> 检索窗口：arXiv 近 3 日新增 + GitHub Awesome 列表增量补充 ｜ 本日入选 **5 篇**并完成精读解读
> 主题一句话：**效率（系统综述）+ 专业流程技能（SOP）+ 蒸馏落地（CA-OPD）+ 人因评测（盲人可访问性）+ 测试时自适应（CoAdapt-GUI）**
> 标签图例：`类型标签`（论文类别）｜ `训练方法标签`（方法级）

---

## 📄 今日论文

### 1. Efficient GUI Agents: A Systems Survey of Observation, Memory, Action, and Runtime Optimization
- **标签**：`General` `Web` `Mobile` `Desktop` ｜ 训练方法：—（系统综述）
- **一句话**：从"观察/记忆/动作/系统"四层系统性梳理 GUI 智能体如何做快做省；指出反思/验证占任务延迟 76%~96%，"省下的成本"常转移到 parser/verifier/编排。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.02309) ｜ 代码：未开源
- 📝 **[阅读完整解读](./2026-09-03_Efficient_GUI_Agents_A_Systems_Survey_of_Observation_Memory_Action_and_Runtime_Optimization.md)**

### 2. OmegaUse-SOP: SOP Engineering for Professional Computer Use from Human Demonstrations
- **标签**：`Desktop` `GUI Grounding` `Planning` `General` ｜ 训练方法：—（Human-in-the-loop 技能工程）
- **一句话**：百度把专家演示变成可复用 SOP 技能（Observe→Reason→Configure→Execute），光伏仿真 PVsyst 专业任务让 Qwen3-VL/GPT-5.5/Opus-4.7 从 1~3/5 全部升到 **5/5**；去掉语义化模块掉回 2/5。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.02149) ｜ 💻 [Code: omegause-sop](https://github.com/baidu-frontier-research/omegause-sop) ｜ 🤝 [Code: HITL 实现](https://github.com/ethanyxx/co-work)
- 📝 **[阅读完整解读](./2026-09-03_OmegaUse-SOP_SOP_Engineering_for_Professional_Computer_Use_from_Human_Demonstrations.md)**

### 3. CA-OPD: Confidence-Aware On-Policy Distillation for Structured Visual Prediction
- **标签**：`Distillation` `GUI Grounding` `Online` ｜ 训练方法：`Distillation (On-policy)`、`SFT（学生初始化）`
- **一句话**：用教师置信度（而非排名）决定"纠错 vs 放行"，干预点补 CE 监督、放行点补分布蒸馏；Qwen3.5-0.8B 学生六项基准全面上涨（ScreenSpot-Pro 相对裸模型 **+9.50**）。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.02401) ｜ 代码：未开源
- 📝 **[阅读完整解读](./2026-09-03_CA-OPD_Confidence-Aware_On-Policy_Distillation_for_Structured_Visual_Prediction.md)**

### 4. Are We There Yet? Assessing Computer-Use Agents for Blind Users' Accessible Interaction with Desktop Applications
- **标签**：`Benchmark` `Desktop` `General` ｜ 训练方法：—（用户研究 + 多模型评测）
- **一句话**：8 名盲人 3 周日记研究 + 5 模型重放：最好模型 GPT-5 成功率仅 **52.5%**；失败集中在 grounding/隐藏路径/约束跟踪/终止识别，盲人更想要"可解释、可叫停"的半自动协作。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2609.00524) ｜ 💻 [Code: OLLA 原型](https://github.com/Satwikram/OLLA)
- 📝 **[阅读完整解读](./2026-09-03_Are_We_There_Yet_Assessing_Computer-Use_Agents_for_Blind_Users_Accessible_Interaction_with_Desktop_Applications.md)**

### 5. CoAdapt-GUI: Joint Workflow Context and Policy Adaptation for Unseen GUI Applications
- **标签**：`RL` `Online` `Mobile` `Reflection` ｜ 训练方法：`RL (GRPO 式 group-relative)`、`Online RL`、`LoRA（测试时自适应）`
- **一句话**：测试时同时演化"可迁移工作流上下文"+"LoRA 策略"，让移动 GUI agent 只靠自身试错上手陌生 App（成功率 45.0%/52.9%），无目标演示、无源界面状态。
- **链接**：📄 [arXiv](https://arxiv.org/abs/2608.11588) ｜ 代码：未开源
- 📝 **[阅读完整解读](./2026-09-03_CoAdapt-GUI_Joint_Workflow_Context_and_Policy_Adaptation_for_Unseen_GUI_Applications.md)**

---

## 📊 本日趋势归纳（按标签）

- **训练方法**：`Distillation (On-policy)`（CA-OPD 小模型落地新范式）＋ `RL/Online`（CoAdapt-GUI 测试时在线自适应）
- **应用形态**：`Desktop` 唱主角（综述/盲人评测/SOP 均为桌面域），`Mobile` 次之
- **系统与效率**：行业正从"纯刷成功率"转向"效率可部署性"，反思/验证成本成为新瓶颈
- **人因与评测**：盲人用户研究提醒 CUA 的下一个增量市场在可访问性与人机协作

---
*生成时间：2026-09-03 ｜ 由 WorkBuddy「GUI 论文每日检索与解读」工作流自动生成*
