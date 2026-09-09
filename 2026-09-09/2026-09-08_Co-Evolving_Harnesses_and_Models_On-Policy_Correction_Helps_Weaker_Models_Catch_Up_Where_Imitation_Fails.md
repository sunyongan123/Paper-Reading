# Co-Evolving Harnesses and Models: On-Policy Correction Helps Weaker Models Catch Up Where Imitation Fails

> 在已进化的 harness 下，整轨模仿专家会让弱模型全面倒退；改用 on-policy 专家定点纠错，才能保住「模型—harness 契合」并继续提升。

---

## 论文档案（Metadata）

| 字段 | 内容 |
|---|---|
| **论文标题** | Co-Evolving Harnesses and Models: On-Policy Correction Helps Weaker Models Catch Up Where Imitation Fails |
| **arXiv ID / DOI** | 2609.09134v1 [cs.AI] |
| **arXiv 链接** | https://arxiv.org/abs/2609.09134（点击直达） |
| **发表出处（Venue）** | arXiv preprint |
| **发布时间** | 2026-09-08 |
| **作者** | Zhou Yu（通讯作者，zhou.yu@salesforce.com）、Bin Bi、Shiva Kumar Pentyala、Shubham Mehrotra、Sougata Chaudhuri、Shilpa Bhagavath、Zeyuan Chen、Ran Xu、Phil Mui、James Zhu、Sitaram Asur（共 11 人） |
| **所属机构** | Salesforce AI |
| **开源情况 / 代码** | ❌ 未在文中声明开源（正文无 GitHub 链接或代码可用性声明） |
| **类型标签（论文类别）** | `Online` `SFT` `Reflection` |
| **训练方法标签** | `LoRA-SFT`（两类数据：专家轨迹模仿、on-policy 定点纠错轨迹） |
| **关键词** | Agent Harness；Harness Evolution；Co-Evolution；On-Policy Correction；LoRA；Enterprise Agent |
| **来源渠道** | arxiv-api |
| **PDF 存档** | papers/2026-09-08_Co-Evolving_Harnesses_and_Models_On-Policy_Correction_Helps_Weaker_Models_Catch_Up_Where_Imitation_Fails.pdf |

---

## 1. 研究背景与要解决的问题

一个 LLM Agent 的表现由两部分共同决定：模型权重，以及包在模型外的「harness」——系统提示词、工具集、执行钩子（hooks）、上下文管理脚手架。近年研究证明，把 harness 当作可搜索的程序做自动演化，能把便宜的小模型抬到接近前沿模型的水准，这对成本敏感的企业级工具调用任务价值很大。

于是出现两根可调杠杆：**改 harness** 与**调模型权重**。既有工作大多只用一根。本文要回答：低成本约束下这两根杠杆应如何组合、能否「协同演化」？作者沿最直觉的路径尝试——先演化 harness，再让更强的专家示范、由小模型模仿——结果出乎意料：模仿在演化后的 harness 下**全面失效**。论文的核心贡献是解释为何失败，并给出一个能同时用好两根杠杆的训练配方。

## 2. 核心方法 / 思路

打个比方：harness 演化＝给「新员工」反复修订岗位说明书和工具台。每轮由反思式元智能体（gemini-3.1-pro-preview）针对失败提出修改，只保留验证集上确有提升的版本。员工（弱模型）为 Qwen3-Coder-30B-A3B 与 Gemma4-26B-A4B，专家为 Gemini。实验基于 Yang et al. 的 7 个企业 agent 任务（工资审计、库存告警、浏览器自动化等）。研究逻辑三步推进：

1. **观测**：为弱模型演化的 harness，专家拿来直接用得更好（7 任务均值 93.6% vs 78.0%），说明存在可榨取的能力缺口，应由专家来教弱模型。
2. **失败**：把专家在演化 harness 下的成功轨迹转格式做 LoRA-SFT，7 个任务全部倒退（平均 −14.9）。同样的配方在**未演化** harness 下却有效（+6.3）——问题出在「模仿 × 演化 harness」的交互而非模仿本身。归因：知识类失败在减少（模型确实学会了领域配方，使用率 30.8%→76.1%），但**规划类失败**从 1.1% 窜升到 14.6%；模型学走了专家的规划风格却没有执行它的能力，与那套围绕弱模型原生规划风格打磨出的 harness 不再匹配。
3. **解决（on-policy 定点纠错）**：不从专家轨迹出发，而从小模型**自己**在演化 harness 下的 rollout 出发：元级 MLE agent 自动定位每个失败 rollout 出错的那一回合，只让专家**就地改写该回合**（一句纠错策略＋修正的工具调用），其余步骤原样保留；再以约 500 行数据（400 纠错＋50 条自身成功样本，专家改写做 N=3 择优）做 LoRA-SFT。既注入知识，又不扰动小模型的原生规划节奏。

## 3. 关键实验结果

- **Harness 演化**：Qwen 平均成功率由基础 harness 的 29.2% 升至演化 harness 的 78.0%（+48.8）。
- **向上迁移**：专家在演化 harness 上达 93.6%（自带 harness 84.4%，+9.2），且在 93.6%–100% 的 rollout 中真实触发各演化组件（领域计算配方 94.2%，弱模型仅 30.8%）。
- **全轨模仿倒退**：均值 78.0%→63.1%（−14.9），7 任务全跌（−4.2 ~ −29.9；最大跌幅在工资审计 −29.9、网站管理 −20.3、浏览器自动化 −16.0）；同配方在基础 harness 下为 +6.3（29.2%→35.5%）。换 Gemma 在 Webarena 复现：harness 演化 46.7%→55.6%，模仿后反而跌到 41.1%（比演化基线低 14.5）。
- **失败归因**：模仿使规划类失败占比 1.1%→14.6%，知识类 46.2%→44.5%——知识在涨、规划在崩。个案中模型 78 步里重复查同一视图 29 次、始终不输出完成，答案在手仍得 0 分。
- **on-policy 定点纠错**：均值 78.0%→79.7%（+1.7），5/7 任务上涨（网站管理 +5.6、库存告警 +2.2、代码重构 +2.2 等），另 2 个饱和任务基本持平（预算审批 −0.6、工资审计 −0.4）；规划类失败仅 1.1%→1.8%，知识类降至 43.2%。成本：LoRA rank 16/64、2 epochs，双 H200 单次训练不足 1 小时。

结论：定点纠错拿到了模仿本应有的知识增益，却避免了模型—harness 契合被破坏，使两根杠杆真正可叠加，并能直接套进迭代协同演化循环。

## 4. 亮点与贡献（Why it matters）

1. 实证一个反直觉现象：在已贴合小模型的演化 harness 下，用更强专家的轨迹做标准 SFT 会系统性倒退，且跨两个模型家族复现——给「先蒸馏后微调」类流水线敲了警钟。
2. 把原因归因清楚：是规划风格漂移破坏契合而非没学到知识；「知识类 vs 规划类失败占比」是可迁移的诊断指标。
3. 给出低门槛配方：失败回合定位＋专家单步改写＋LoRA，约 500 行数据、训练 <1 小时。
4. 提出「harness 与权重须协同演化」原则：harness 一旦贴合某模型，后续权重更新应保持契合而非无意破坏。

## 5. 局限与可改进点（个人点评）

- 评测面窄：仅 7 个企业任务（浏览器类仅 1 个），结论外推需谨慎。
- 强弱配对单一：弱模型限 Qwen/Gemma，教师固定 Gemini，未验证开源强模型教师是否改变结论。
- 上限仍在：纠错后 79.7% 距专家 93.6% 仍有差距，作者也承认轻量 LoRA 的容量限制，且未见与更大规模微调或 RL 的对比。
- 流程串行：harness 演化与微调分步而非联合优化；失败定位/择优依赖 LLM-as-judge，噪声未量化。
- 可复现性：未开源，只报 3 次运行均值±SEM、无显著性检验，且正文称环境改动使绝对数字与 Yang et al. 不完全可比。

## 6. 对我们的启示 / 可借鉴点

最有价值的不是具体数字而是一条纪律：**教师信号应落在学生自己访问过的状态上**，并持续监控「规划 vs 知识」两类失败的此消彼长。对我们的 RL/Post-training 流程，「全轨模仿」对应把奖励/轨迹全部取自离线专家、忽视学生自身分布的做法；on-policy 定点纠错则与「带局部专家修正的在线训练」同源，且数据效率极高，适合作为正式 RL 前的预热。「失败回合定位＋单步改写」这套由元 agent 自动化的数据合成管线，也比整轨清洗便宜得多，还能保留学生自己的风格。

**对 GUI Agent 的可借鉴点：** GUI Agent 的 harness 通常由系统提示、屏幕/无障碍树读取工具、点击前校验 hook、上下文压缩策略组成。若你的方案是用小模型＋为它定制的 GUI harness，再用 GPT-4o/Gemini 的完整操作轨迹做 LoRA 微调，本文是明确警告：大模型轨迹里藏着的「操作节奏」（何时截图确认、滚动粒度、是否校验元素可见性、何时收尾提交）未必是小模型＋该 harness 承受得起的——知识学来了，行为却会水土不服，典型症状是「知道答案却永远不点提交」或跳过多步校验。更稳的做法是把 GUI 场景也跑成协同演化闭环：让 agent 在目标站点自我 rollout，用失败反思演化 harness（补工具、加「点击前校验可点击」hook、外化站点约定）；只在它自身轨迹出错的那个回合（表单校验失败、元素未找到后死循环重试、漏关弹窗）让强模型就地改写，再做 LoRA；多轮交替。本文的 planning/knowledge 失败归因亦可作为 GUI 微调后掉点的诊断手段。

## 7. 延伸阅读

- Yang et al., 2026. *Better Harnesses, Smaller Models*（arXiv:2607.08938）——任务套件与 harness 演化框架来源。
- Agrawal et al., 2025. *GEPA: Reflective Prompt Evolution*（arXiv:2507.19457）——反思式 harness 演化算法。
- Lauffer et al., 2025（arXiv:2512.14895）、Ye et al., 2026（arXiv:2607.04574）——on-policy 教师纠错。
- Xu et al., 2026. *Adapting the Interface, Not the Model*（arXiv:2605.22166）——harness 向上迁移。
- Co-Harness（2607.22688）、SIA（2605.27276）、HarnessForge（2606.01779）——协同演化同期工作。

---
*解读生成时间：2026-09-09 ｜ 解读人：WorkBuddy（AI）*
