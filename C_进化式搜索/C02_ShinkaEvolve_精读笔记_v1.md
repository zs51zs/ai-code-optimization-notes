# C02｜ShinkaEvolve 精读笔记 v1：多候选进化搜索与样本效率

> 状态：已完成前两轮精读（§3 Method、§4 结果导读、Figure 5、Figure 9 消融），尚未完成官方源码逐模块核对和实验复现；教学数值均标为示例。

## 一、论文信息

- 论文：ShinkaEvolve: Towards Open-Ended And Sample-Efficient Program Evolution
- 作者：Robert Tjarko Lange、Yuki Imajuku、Edoardo Cetin；Sakana AI。
- 发表于 ICLR 2026；初版 arXiv 2025-09-17。
- arXiv HTML：https://arxiv.org/html/2509.19349
- ICLR 正式版本：https://proceedings.iclr.cc/paper_files/paper/2026/hash/7886b9bafe76c52fd568db10ff9772df-Abstract-Conference.html
- 代码：https://github.com/SakanaAI/ShinkaEvolve
- 官方博客：https://sakana.ai/shinka-evolve/

## 二、研究问题与前两篇论文的关系

- SWE-Perf：如何构建可执行的真实优化任务，如何评价 Apply / Correctness / Performance。
- PerfAgent：使用 Profiler 反馈 + 外部优化控制器，沿多轮代码修改不断改善候选。
- ShinkaEvolve：维护多个已评价的程序候选，研究有限评测次数下如何选择父代、生成不同策略、抑制冗余并动态选择 LLM。

**核心区别**：PerfAgent 更强调“下一轮该修改什么”，ShinkaEvolve 更强调“下一轮从哪个候选出发，如何兼顾局部改进与探索”。两者有交集，不是互斥方法。

## 三、§3 三阶段方法

### 3.1 Parent and Inspiration Sampling（父代与启发样本）

ShinkaEvolve 建立保存程序、得分和元数据的 archive；采用多个 island 子种群并行探索，偶尔迁移程序以交换知识。给 LLM 的上下文包括一个主要 parent，以及从历史优秀程序和其他程序中取样的 inspiration。

**父代选择策略 1：按排名的幂律采样**

设候选按适应度排名，`r_i = 1` 是当前最佳：

`p_i = r_i^(-alpha) / sum_j r_j^(-alpha)`

- `alpha=0`：近似均匀采样。
- `alpha` 很大：偏向只选最佳程序，近似贪心。

**父代选择策略 2：兼顾得分与“已产生后代数量”的加权采样**

对候选 `P_i`：

`a0 = median(F(P_1), ..., F(P_n))`

`s_i = sigmoid(lambda * (F(P_i) - a0))`

`h_i = 1 / (1 + N(P_i))`

`w_i = s_i * h_i`

`p_i = w_i / sum_j(w_j)`

- `F(P_i)`：程序适应度，越高越好。
- `N(P_i)`：该程序已经产生的后代数；被开发过很多次的父代会受到惩罚。
- `lambda`：适应度影响强度。

**教学演算（非作者实验）**：候选 A/B/C 的得分分别为 90/80/70，子代数为 9/1/0，设 `lambda=0.1`、适应度中位数 80。对应权重约为 0.0731/0.2500/0.2689，归一化后的选择概率约为 12.35%/42.23%/45.43%。当前最优秀的 A 并不是最常被选中的候选，因为它已被大量开发。

重要区分：这里的 `h_i` 不是代码向量“语义新颖性”，而是一个防止反复开发同一父代的启发式因子。

### 3.2 Mutation and Novelty Assessment（生成与新颖性过滤）

ShinkaEvolve 采用三种变异操作：

1. `diff`：SEARCH/REPLACE 形式的局部修改。
2. `full`：改写允许改变的代码区域，同时保护不可修改内容。
3. `cross`：选取另一段归档程序作为参考，由 LLM 进行交叉重组。

代码通过 `EVOLVE-BLOCK-START` 和 `EVOLVE-BLOCK-END` 标记可变区域；非法 Patch 可得到解析反馈并重新生成。

**Novelty Rejection Sampling**：

1. 只提取可修改代码块，计算其 embedding。
2. 与子种群归档程序的 embedding 做余弦相似度比较。
3. 如果最大相似度超过阈值（论文举例 `eta=0.95`），再用 LLM 判断候选是否真的具有实质差异。
4. 明显重复的候选可以直接放弃或重采样，避免浪费昂贵的编译、测试和评价资源。

注意：embedding 近似不等于程序语义等价；新颖性过滤不能取代正确性验证。

### 3.3 Execution, Feedback, LLM Bandit（执行、反馈与动态模型选择）

程序评价器负责返回标量 fitness、公有指标和文字反馈；这些信息入库，供后续变异使用。评价器依赖具体任务的人为定义：例如圆形排布的目标是半径之和，算法竞赛的目标是得分。它并非自动知道所有程序“什么才是性能更好”。

ShinkaEvolve 根据 UCB1 多臂老虎机思想动态选择生成模型。模型的奖励基于相对改进，而不是子代绝对分数：

`r_i^u = exp(max(r_i - r_i^b, 0)) - 1`

`r_i^b` 是父代得分与初始程序得分的较大值。若没有提升，则奖励为零；大的正向突破得到更多奖励。论文还对奖励进行归一化以适应不同任务分值尺度。

**Meta-scratchpad**：周期性总结历史成功策略、失败教训、优化建议，加入后续 LLM 上下文。该模块是跨代经验总结，而不是直接参与测试判定。

## 四、§4 四类任务（第一轮导读）

| 任务 | 核心目标 | 论文报告的结果 | 注意事项 |
|---|---|---|---|
| 26 圆排布 | 单位正方形内 26 个不重叠圆的半径和最大化 | 约 150 次候选评价得到具有竞争力的解 | 数值可行性验证的容差影响得分，严格检验要单列 |
| AIME 数学推理 Agent | 自动搜索一个有调用预算的推理工作流 | 75 代；发现多专家→批判→汇总的 7 次调用方案 | 任务是优化 Agent scaffold，不是加快源代码运行 |
| ALE-Bench | 改善已有程序在启发式算法竞赛中的表现 | 对 10 个任务平均提升约 2.3% | 依靠公开测试优化，需要私有测试检查泛化 |
| MoE 负载均衡损失 | 搜索训练损失函数的设计 | 发现一种新的补充正则化策略 | 需要真实训练验证，不等价于代码性能优化 |

论文中圆排布约 150 个评测取得优秀效果的陈述是作者报告的特定任务结果，不能推导出 ShinkaEvolve 在任意代码仓库里 150 次就能找到最佳优化。

## 五、§5 消融结果（首次导读；详见下文第二轮精读）

- 加权父代选择的最终搜索效果优于总从原始程序开始的 Best-of-N 和只选当前最优候选的 Hill Climbing。
- 自适应 bandit LLM 集成优于固定均匀模型集成和论文测试的单模型方案。
- embedding 重复过滤改善了评测样本利用率；LLM 新颖性判断带来的边际价值要考虑额外调用成本。

这些结论主要源于论文给出的实验任务与预算。Figure 9 的逐图分析已在下文第二轮精读中补充；不应笼统宣称所有工程场景都适用。

## 六、与 PerfAgent 的重要差异

| 维度 | PerfAgent | ShinkaEvolve |
|---|---|---|
| 最突出机制 | Profiler 引导下一轮代码修改 | 程序种群与父代选择引导探索 |
| 候选结构 | 连续迭代、保留最佳正确候选 | 多条分支的 Archive + Islands |
| 反馈重点 | 热点、测试结果、性能计时 | 适应度、公有指标、文本评估、历史成功策略 |
| 搜索多样性 | 依赖 Agent 和控制器多轮策略 | 父代加权采样、交叉重组、新颖性过滤 |
| 资源约束 | 每任务费用、步数、迭代次数 | 候选数、模型成本、评价次数、并行任务 |
| 需要额外注意 | 重复计时被缓存投机、正确性盲区 | 适应度投机、重复候选、公开测试过拟合 |

这并不表示 PerfAgent 只有一条 Patch 分支，也不表示 ShinkaEvolve 缺乏反馈；上表比较的是论文各自强调的核心机制。

## 七、工程实现的正确边界

截至 2026-11-05，官方仓库已经持续升级，与 2025 初版论文中的具体 API 不完全相同。README 已列出：

- `pip install shinka-evolve`
- `shinka_launch variant=circle_packing_example`
- 现代接口 `ShinkaEvolveRunner`
- 任务基础文件 `initial.py` 和 `evaluate.py`
- 可在本地或 Slurm 环境执行评测任务，也有给 Codex/Claude 工作流使用的 Skills。

官方安装与评测 API 见： https://github.com/SakanaAI/ShinkaEvolve/blob/main/README.md

评测器最低应承担：

1. 限制可修改区域，防止修改测试基准。
2. 在独立工作区执行生成的候选程序。
3. 先执行正确性/可行性验证，不正确不授予正向性能分数。
4. 在受控环境下重复测量，并记录工作负载、环境和随机种子。
5. 隔离隐藏测试，防止过拟合及 Reward Hacking。
6. 保存候选代码的哈希、父代 ID、代码差异、测试记录、耗时与模型费用。

上面是基于论文经验提出的工程建议，不代表所有条目都已由 ShinkaEvolve 项目默认提供。

## 八、下一轮问题清单

1. 加权父代选择究竟改善了哪些搜索现象？为什么 Hill Climbing 会早早停滞？
2. `similarity > 0.95` 的过滤会不会误删本来很有潜力的微小优化？
3. Bandit 奖励只偏好正向提升，会不会过度鼓励一次性的高风险方案？如何与费用结合？
4. 选择一个开放测试上的最优程序后，如何用独立测试验证实际泛化？
5. 能否结合 PerfAgent 的 Profiler 证据与 ShinkaEvolve 的多候选搜索？如何设计消融证明两种机制的独立增益？

## 九、阅读来源

- 论文 §3 Method：https://arxiv.org/html/2509.19349#S3
- 论文 §4 Results：https://arxiv.org/html/2509.19349#S4
- 论文 §5 Ablations：https://arxiv.org/html/2509.19349#S5
- ICLR 2026：https://proceedings.iclr.cc/paper_files/paper/2026/hash/7886b9bafe76c52fd568db10ff9772df-Abstract-Conference.html
- 官方实现：https://github.com/SakanaAI/ShinkaEvolve
- Sakana AI 官方技术介绍：https://sakana.ai/shinka-evolve/

**阅读状态**：方法首轮精读与 Figure 5/9 第二轮精读已完成。未复现实验；尚未完成官方源码逐模块核对。

---

# 第二轮补充精读：Figure 5 进化树与 Figure 9 三项消融（2026-11-06）

> 证据：arXiv:2509.19349 论文 §4.1、§5，PDF 第 6、10 页。此处是论文解读，不是我们自己复现实验。
> 论文：https://arxiv.org/pdf/2509.19349

## 十、Figure 5：如何读进化轨迹与进化树

### 10.1 圆形排布任务与横纵轴

任务：在单位正方形放置 26 个互不重叠的圆，尽可能最大化所有圆的半径之和。图中的纵轴 Evolved Performance Score 表示优化任务的适应度，并非 SWE-Perf 的代码性能增益，也不是运行时间加速比。横轴 Number of Evaluated LLM Program Proposals 为被实际评价的 LLM 程序提案数量，不是 Agent 运行秒数。

左图四个图例：
- **Best Score**（红色阶梯线）：到第 k 次评估为止的历史最佳得分，因 max 运算而单调不降；
- **Individual Evals**（黑色散点）：每一个候选实际取得的得分，允许较历史最佳显著更低；
- **Path to Best Node**（蓝色虚线）：最终最佳程序的祖先程序形成的路径，不能等同于每次历史最好值；
- **Cumulative Cost**（橙色虚线、右侧坐标）：累计 API 费用，区别于横轴的“评估次数”。

作者将搜索观察为初期迅速改进、中期持续探索、后期重启和精细调整三段；这是对实验结果的总结，而不是框架强制划分的阶段。

### 10.2 右图的节点与多分支关系

节点是程序候选，边表示父代到子代的关系，节点形状区分 Diff / Full Rewrite / Crossover 等编辑，叉号表示无效程序，颜色表示适应度。黑色高亮路径表示通向最佳结果的祖先路线。**历史得分最高的程序不一定是未来改进最有潜力的父代**；一些分数暂时较低的中间方案（stepping stones）仍可能通过变异或重组贡献后续突破。

需要区分：图中全部候选的历史树不等于实际 archive 在每个时刻同时保留所有程序（作者使用有限大小的归档库）。

## 十一、Best-of-N、Hill Climbing、Weighted Sampling

| 策略 | 每轮父代如何确定 | 是否利用历史新候选作为父代 | 主要优势 | 主要风险 |
|---|---|---|---|---|
| Best-of-N | 始终从初始 P0 重新生成 | 否 | 独立、多样的初次尝试 | 难以累积多代局部改进 |
| Hill Climbing | 始终选历史表现最好的程序 | 是，只集中最优者 | 初期能迅速改进 | 容易在局部最优附近反复尝试 |
| Weighted Sampling | 按表现和探索程度的权重选择多个历史候选中的父代 | 是，包含暂时次优的候选 | 兼顾开发与探索 | 随机性和超参数带来额外复杂性 |

ShinkaEvolve 的 Best-of-N 是“不断以初始程序作为父代”的特定基线；不要与 PerfAgent 对比实验中运行多次 OpenHands、再用隐藏评测结果选最优的 best@5 混为一谈。两者都包含多个独立候选，但具体工作流和预算口径不同。

Weighted 机制示意：设 `F(P_i)` 为得分，`N(P_i)` 为已有子代数，`a0` 为得分中位数，则

`s_i = sigmoid(lambda * (F(P_i) - a0))`

`h_i = 1 / (1 + N(P_i))`

`p_i = (s_i * h_i) / sum_j(s_j * h_j)`。

这避免始终只选择第一名，但不意味着“低分候选必然好”，更不保证探索一定获得突破。

## 十二、Figure 9：三个组件分别做了什么

所有图的横轴均为 Number of Evaluated LLM Program Proposals，纵轴均为 Evolved Performance Score，实验任务均为 Circle Packing，观察范围约为 150 个被评价候选。

### 12.1 左图：父代选择消融

- **Best-of-N**（蓝）：始终从初始程序提议修改；
- **Hill Climbing**（橙）：始终从当前最高分程序提议修改；
- **Novelty Weighted**（绿）：同时考虑适应度和父代已有子代数。

观察：蓝线最终最低，橙线前期迅速提高但后期较早平台化，绿线持续改进且最终最高。说明在本 Circle Packing 设置下，保留并采样多条历史候选路线具有更高样本效率。不能仅据此推论在所有真实仓库中也成立。

### 12.2 中图：模型选择消融

- **Single LLM Model**（蓝）：使用单一模型，作者注明 GPT-5-nano；
- **LLM Model Ensemble**（橙）：从事先规定的多个模型中均匀采样；
- **+ Bandit Prioritization**（绿）：根据历史的相对改进奖励，自适应提高部分模型被选中的概率。

观察：固定 ensemble 相对单模型改善明显；bandit 比均匀 ensemble 略高。论文图注描述 bandit 相对固定 ensemble 的效果为“slightly improves”，因此不能把差距说成大幅提升，更不能因为正文措辞就推断已经做过统计显著性检验。

### 12.3 右图：新颖性过滤消融

- **No Rejection**（蓝）：不检查重复，所有候选直接进入评测；
- **Threshold Rejection**（橙）：使用代码 embedding 的高余弦相似度识别重复，文中例子阈值 0.95；
- **+ LLM-Novelty-Judge**（绿）：在 embedding 标记为高度相似时，增加 LLM 对是否具有实质差异的判断。

观察：有过滤明显优于完全无过滤；在 embedding 检查之外使用 LLM 新颖性判断只有边际提升。Embedding 近似不等于语义等价；高相似程序仍可能包含关键微小优化，故需关注误删率和额外费用。

### 12.4 图 9 的证据边界

- 只表明在展示的 Circle Packing 实验中哪种策略达到更高分值，不能直接跨域套用到 Python/C++ 性能优化；
- 横轴为“已经评价的程序数量”，**不是**相同 Token、API 美元、CPU/GPU 时间或端到端耗时；新颖性过滤还可能增加被拒绝的 LLM 生成调用和判别调用；
- 曲线是示例实验轨迹，不是带误差棒的多次独立运行置信区间；不应依据肉眼可见的微小差距声称统计显著；
- 论文未在 Figure 9 中独立消融所有机制，例如 meta-scratchpad、island 等，因此不能把最终增益全部归因于图中三项。

## 十三、对我们代码优化 Agent 的实验设计建议（研究假设，不是已实现结果）

**问题**：在统一的正确性、性能测量规则下，哪些搜索策略提高可信性能收益/预算？

建议同一个基础 Agent、相同任务集、相同环境、相同可用工具，分别实验：

1. Independent Best-of-N：从原始代码独立生成 N 个候选；
2. Greedy Hill Climbing：始终从当前最好且正确的候选继续修改；
3. Weighted Population Search：维护多个候选并加权采样父代；
4. Feedback + Population（后期研究）：加上 PerfAgent 风格的 Profiler 证据反馈。

每个策略记录：Apply、Correctness、真实性能收益、性能样本分布、候选数、无效/重复候选率、模型 token 和费用、构建与测试时间、达到给定收益阈值所需的预算，以及不同随机种子的方差。

控制至少两种预算：① 相同完整性能评测次数；② 相同端到端实际预算（例如 API 费 + 测试 CPU 时间）。两种预算可导致不同排序。

初步工程实施顺序建议：可靠评测器 → 独立多候选基线 → 贪心多轮基线 → 候选归档与父代加权选择 → 过滤冗余 → 动态 LLM / Profiler 混合策略。当前尚未授权 Codex 开发此阶段功能，先把它保留为后续研究线索。

## 十四、本轮需要记住的研究问题

- 为什么 Best-of-N 相比多代搜索更难积累复杂的程序改进？
- 次优候选什么时候成为有价值的 stepping stone？有什么可操作的预测指标？
- 过滤重复候选节省的是评价器预算，还是整个系统的总预算？
- 如何公平比较不同搜索策略并排除隐藏的预算差异？
- 如何证明性能反馈与多分支候选选择之间存在互补增益，而不是单纯因为增加了 API 调用？

**更新状态**：完成 Figure 5、Figure 9 的第二轮方法与消融精读；尚未自己复现作者实验。
