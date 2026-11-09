# PerfAgent × ShinkaEvolve：两种自动代码优化路线的交叉对照

> 研究积累文件。事实以两篇论文的实际实验设定为界，研究设想标注为“假设”，尚未完成工程验证。

| 问题 | PerfAgent | ShinkaEvolve | 我们可以研究什么 |
|---|---|---|---|
| 下一次改什么 | 依据最新 Profiler 热点 | 依据评价记录、父代及启发样本 | 运行证据能否改善 LLM 变异方向 |
| 从哪里修改 | 当前候选及已验证最佳候选 | 存档的多个程序父代 | 多分支是否减少陷入局部最优 |
| 如何分配模型调用 | 有限迭代、工具与费用预算 | Bandit 动态选择不同 LLM | 在公平成本下实现可验证的搜索效率提升 |
| 如何拒绝无效修改 | 构建/测试/性能反馈 | Patch 合法性与新颖性过滤、任务评价器 | 将正确性门禁和重复过滤置于昂贵性能复测之前 |
| 如何记录经验 | 每轮 Profiling 和结果反馈 | Archive、island、meta-scratchpad | 历史经验是否比当前热点独立增加收益 |
| 如何降低风险 | 控制 Reward Hacking、验证正确性 | 限定可改区、独立验证、避免公开测试过拟合 | 独立隐藏工作负载，禁止修改评价器 |

## 初步的四组实验设想（尚未实施）

在相同任务集、候选评估次数或等量总成本、模型和硬件条件下比较：

- A：One-shot LLM 优化（无 Profiler、无种群）。
- B：Profiler 反馈迭代（PerfAgent 风格）。
- C：多候选进化搜索（ShinkaEvolve 风格）。
- D：Profiler + 多候选进化搜索（候选假设方案）。

首先验证 D 是否优于 B/C，然后对父代选择、新颖性过滤、Profiler、历史摘要做消融，记录 Apply、Correctness、可靠性能收益、调用费用、实际耗时、每题成功率和搜索过程。

**避免的错误**：不能把 PerfAgent 的 Opt@1 直接与 ShinkaEvolve 的圆排布适应度或 ALE-Bench 的分数比较。它们不是同一种指标，也不使用相同的任务集。

## 待解答的问题

1. 反馈数据是否真的帮助选择更有潜力的父代，还是只增加 token 消耗？
2. 性能提升很小却结构创新的候选是否应该保留？如何同时定义性能与多样性？
3. 如何在有限预算下选择更强模型、重测候选或尝试新父代？
4. 如何防止候选针对固定 benchmark 做缓存或硬编码？

来源：https://arxiv.org/html/2607.19653 ； https://arxiv.org/html/2509.19349


## Figure 5 / Figure 9 新增研究认识（2026-11-09 整理）

- **搜索策略并不等价：** Independent Best-of-N 始终从原始代码生成候选；Hill Climbing 集中改进当前最好候选；加权父代采样在多个历史候选之间分配搜索机会。
- **Figure 9 的证据范围：** 在论文展示的 Circle Packing 实验和相同正式候选评测次数下，加权父代采样、模型集成及新颖性过滤表现更好；并未证明在真实仓库代码优化、相同 API 费用和总墙钟时间下依旧如此。
- **潜在研究问题：** Profiler 热点能否辅助选择父代和生成变异？新颖性过滤能否降低昂贵的正确性测试/性能评测成本？多分支搜索是否确实优于反复优化单一最佳 Patch？
- **实验设计原则：** 既要在相同正式评测次数下比较，也要在相同端到端预算下比较，并独立报告正确性、性能改善、费用、耗时、重复率及多随机种子波动。

## 待交接 Codex 的问题——先记录，**暂不实施**

以下条目仍属于论文理解与实现待核查项。应先阅读 ShinkaEvolve 官方仓库，确认实现与论文版本间的关系，再整理成正式开发任务：

1. **Archive：** 程序代码、父代 ID、评分、评测日志、代码哈希如何持久化？是否可回滚及重建实验轨迹？
2. **Parent Sampler：** 论文加权采样公式在当前源码中如何实现？按哪些条件筛选候选？岛屿子种群如何管理？
3. **Novelty Filter：** Embedding、相似度阈值与 LLM 新颖性判断在哪里实现？重复代码如何处理？其拒绝成本如何记录？
4. **Evaluator：** 候选执行和评分如何隔离？哪些正确性门禁与私有测试需要由我们独立补充？
5. **预算：** 如何区分 LLM 生成费用、过滤费用、编译与性能复测费用？
6. **兼容性：** 哪些模块可作为设计参考，不依赖 ShinkaEvolve 代码版本细节？是否能接入现有 Agent 的评测器？

**目前 Codex 的正式优先任务仍是 SWE-Perf + PerfAgent 的现有架构审查。** 在这份审查完成、且 ShinkaEvolve 源码核对后，再决定是否新增进化搜索实现任务。不要将上述问题误解为已获得开发授权。


## 第三轮源码核查补充（2026-11-09）

- 最新公开版本已统一 `ShinkaEvolveRunner`，不能把论文或旧版 API 当作最新实现。
- `programs` 与 `archive` 两表区分持久化历史和当前候选池；只允许正确候选进入当前 archive，`fitness` / `crowding` 负责替换。
- 当前 `WeightedSamplingStrategy` 相比论文简化式额外采用 MAD 对得分差做尺度归一化。
- 当前新颖性阈值默认 `0.99`，论文/旧笔记中的 `0.95` 不是当前源码默认值；高相似候选有无 LLM 裁判时行为不同。
- `run_shinka_eval()` 的正确性 `validate_fn` 可选，因此我们自己的可信正确性、隐藏工作负载和性能复测必须由独立评测器负责。
- 设计建议：先集成“候选库 + 可信评价”，再设计 Best-of-N、Hill Climbing、Archive Sampling 对照，最后评估 Profiler 证据与种群搜索协同；不要提前开发多岛迁移和模型 Bandit。

源码入口：
https://github.com/SakanaAI/ShinkaEvolve/blob/main/shinka/database/dbase.py
https://github.com/SakanaAI/ShinkaEvolve/blob/main/shinka/database/parents.py
https://github.com/SakanaAI/ShinkaEvolve/blob/main/shinka/core/novelty_judge.py
https://github.com/SakanaAI/ShinkaEvolve/blob/main/shinka/core/wrap_eval.py
