# A01 SWE-Perf 精读（三）：§5 Experiments 与 Figure 5（V3）

- 阅读日期：2026-11-03
- 主题：AI Agent 自动代码性能优化评测
- 阅读深度：L2（论文方法与实验）；尚未复现官方评测脚本
- 论文：https://arxiv.org/html/2507.12415
- PDF：https://arxiv.org/pdf/2507.12415v2
- 正式发表：https://proceedings.mlr.press/v306/he26y.html
- 源码：https://github.com/SWE-Perf/SWE-Perf
- 资料状态：来自论文 v2 第 5 节及附录 C；GitHub 仅做了仓库结构核对，未运行模型和性能测试

## 0. 一句话结论
SWE-Perf 在 140 个历史性能优化任务上比较直接 LLM、固定工作流 Agentless 与交互式 OpenHands；模型的测试正确率已经较高，但总体性能收益仍远低于人类参考 Patch。论文提示性能优化 Agent 需要独立处理正确性、实际性能反馈及跨函数/跨模块分析。

## 1. Figure 5：为什么不能当成模型直接排行榜？

- 图 5 的每个 Model 柱仅计算对应模型**正确完成**的任务集合 C_m 上的 Performance。
- 同组 Expert 也在同一个 C_m 上重新计算人类历史补丁的 Performance，不是全部 140 道题上的固定人类均值。
- 因为 C_m 随模型变化，各模型的 Expert 柱不同，因此 Model 柱之间的比较同时混入了任务组成差异。
- 图 5 是 Oracle/File-Level 设置下的模型对照，不是 OpenHands 的系统表现；Figure 6 用相同的筛选思想比较 Oracle/Agentless/OpenHands。

符号：

```
C_m = { i | 模型 m 对任务 i 生成了通过相关测试的 Patch }
Model_m  = sum_{i in C_m} P_i^(m) / |C_m|
Expert_m = sum_{i in C_m} P_i^(expert) / |C_m|
```

### Figure 5 选取的真实读数（百分比）

| 模型 | Model | 对应 Expert | 相差（百分点） |
|---|---:|---:|---:|
| DeepSeek-V3 | 1.3 | 6.5 | 5.2 |
| DeepSeek-R1 | 1.8 | 8.9 | 7.1 |
| Claude-4-opus | 1.6 | 10.0 | 8.4 |
| Claude-4-sonnet | 2.5 | 8.5 | 6.0 |
| Gemini-2.5-pro | 1.8 | 8.9 | 7.1 |
| OpenAI-o3 | 1.8 | 12.2 | 10.4 |

备注：Figure 5 中有一列标记为 `Claude-3.7-opus`，而 Table 2 使用的是 `Claude-3.7-sonnet`。这属于图表/命名一致性疑点，不能未经进一步核查擅自当作两个不同实验或补写模型数据。

### Figure 6（正确任务子集）

| 方法 | Model | Expert |
|---|---:|---:|
| Oracle（Claude-3.7-sonnet） | 2.0% | 8.3% |
| Agentless（Claude-3.7-sonnet） | 0.6% | 8.8% |
| OpenHands（Claude-3.7-sonnet） | 2.9% | 11.4% |

Table 2 中 OpenHands 全部任务总体 Performance = 2.26%，Correctness = 77.86%，因此 2.26 / 0.7786 ≈ 2.90%，与 Figure 6 相符。这个换算使用论文总体得分对无正确 Patch 任务赋予零贡献的评分逻辑；最终以官方评价实现为准。

## 2. §5.1 Baselines：三类系统是什么

1. Oracle / Direct model：已提供从人类补丁反推出的修改目标函数与相关文件，模型采用一次性生成 Patch。**Oracle 不是向模型展示人类优化补丁，也不是全自动发现热点。**
2. Agentless：固定多阶段工作流，局部定位→修改→候选验证/选择，原本用于缺陷修复，不是专门设计的性能优化 Agent。
3. OpenHands：交互式 Agent，可以读取/探索仓库、执行工具、修改文件并迭代。Agentless 与 OpenHands 使用相同 Claude-3.7-sonnet 后端；但输入、交互次数、工程开销及环境设定并不完全一致，不能直接做严格的“Agent 方法因果提升”结论。

## 3. §5.2 Main Results：Table 2（总体基准分数）

| Setting | Method | Apply | Correctness | Performance |
|---|---|---:|---:|---:|
| Expert | Human Patch | 100.00% | 100.00% | 10.85% |
| Oracle | Claude-3.7-sonnet | 66.43% | 61.43% | 1.24% |
| Oracle | Claude-4-sonnet | 73.57% | 70.00% | 1.76% |
| Oracle | Claude-4-opus | 85.71% | 78.57% | 1.28% |
| Oracle | Gemini-2.5-Pro | 95.00% | 83.57% | 1.48% |
| Oracle | OpenAI-o3 | 78.57% | 76.43% | 1.37% |
| Realistic | Agentless + Claude-3.7-sonnet | 88.57% | 70.71% | 0.41% |
| Realistic | OpenHands + Claude-3.7-sonnet | 87.86% | 77.86% | 2.26% |

备注：这里摘录关键行，完整表还包括 GPT-4o、o1、DeepSeek-V3/R1、Qwen3 等。两个不同设置不能只按 Performance 数值简单排名。论文的 `Performance` 是保守最小改善率的任务平均，不等于 Patch 通过比例/普通 speedup。

- 在该测试设置下，OpenHands 达 2.26%，人类参考 10.85%，相差 8.59 **个百分点**。
- 在 Oracle 中，Claude-4-sonnet 的总体 Performance 1.76%（所比较模型中较高），Gemini-2.5-Pro 的 Apply / Correctness 分别为 95% / 83.57%，三个指标排名不一致。
- Figure 4 按仓库分组：在 `sklearn` 组 OpenHands 3.6% > Expert 3.2%，只说明分组均值，不意味着总体反超专家；其他组有大幅差距，例如 `sympy` 中 Expert 31.8%。

## 4. §5.3 能力分析：四个方面

### 4.1 Correctness-conditioned（Fig 5/6）
去掉错误 Patch 对总体平均分的贡献后，模型仍大幅落后相应任务上的人类参考；说明问题不只是编译/测试通过率，也包括真正的优化策略与对程序行为的分析。

### 4.2 Original runtime（Fig 9）
Figure 9 横轴是**原始程序/相关测试本身的运行时间**，不是 Agent 思考时长、搜索预算或整个优化过程的耗时。论文观察到人类参考 Patch 在较长运行时间样本上存在更高的性能收益，而模型改善较平坦。但图像来自分组统计，不能推导“程序运行更久导致人类更会优化”。

### 4.3 Target functions（Fig 7/8）
目标函数增多时，优化整体更难，尤其 Realistic 设置中 OpenHands 与专家差距扩大。应注意函数数量是复杂性的代理变量，和代码库大小/依赖结构等相关，不是已证实的唯一因果原因。

### 4.4 Keyword / Word clouds（Fig 10/11）
作者使用补丁新增文本的词云，推测模型更偏低层数据结构和基础功能，人类更偏领域语义和数据完整性。词云仅为描述性证据，受到项目、文本过滤、环境配置路径、异常 tokens 等影响，不能单凭词云证明模型“不会领域优化”。

## 5. 论文与实验设计的局限

- 数据集仅来自 9 个 Python 开源仓库，不覆盖通用 C++、GPU、RDMA、分布式程序的广泛情况。
- Expert Patch 是人类参考而非理论最优解。
- Oracle 直接给出人类改过的位置，和 Realistic 的信息条件并不一样；比较三者时需注意预算/工具/上下文的差异。
- 正确性指**相关测试通过**，不是所有可能输入上的语义等价形式证明。
- Figure 5/6 按每个方法的正确样本过滤，因此不能不控制任务子集就做跨方法的纯模型能力排名。
- Figure 4/7/8/9 的图形趋势反映相关性与分组均值，不能自动解释成因果规律。

## 6. 对我们 AI Code Optimization Agent 的可执行启示

1. 同时报告四类指标：Apply、Correctness、全任务 Performance、条件于正确样本的 Performance（并报告正确任务数）。
2. 保留“原始版本、模型 Patch、性能数据、正确性报告、对应 Expert Patch”（Expert 仅用于离线参照、不得提前泄露给候选生成流程）。
3. 对比三种基线：单次 LLM 修改、固定多阶段流水线、带性能分析和反馈的交互式 Agent；严格记录每个候选的模型调用和工具预算。
4. 按同一任务 ID 设计配对实验；对不同模型成功任务集合不同的问题，增加共同正确子集评估。
5. 使用相同硬件环境重测 baseline 和 candidate；单次/一次最优计时不得用于科研结论。
6. 专门评价多函数/跨文件代码优化，观察定位错误、修改幅度、功能回归与性能退化。
7. 对后续 ParaLedger 做领域扩展时，补充 TPS、尾延迟、正确性/一致性、分布式同步/通信指标，而不是将当前 Python Benchmark 分数直接外推。

## 7. 下一轮可做的研究问题

- RQ-A：Profiler 证据能否帮助 Agent 在 Realistic 的较多候选函数中更准地定位瓶颈？
- RQ-B：相同调用/Benchmark 预算下，交互式 Agent 与多候选演化搜索哪个更优？
- RQ-C：当任务以函数数量、基线运行时间划分子组时，加入 profiler/调用图反馈的收益是否一致？
- RQ-D：加入独立正确性门禁与复测后，Agent 的真实可信收益相较“最佳单次测量”有何变化？

## 8. Codex 接口/实验要点

数据结构建议包含：`task_id`, `repository_commit`, `patch_id`, `setting`, `method`, `model`, `correctness_pass`, `apply_pass`, `runtime_samples_pre`, `runtime_samples_post`, `gain_estimate`, `gain_conservative`, `budget_llm_calls`, `budget_test_runs`, `profiler_artifact`, `run_config`。

避免混淆 `perf_all`, `perf_on_correct`, `expert_on_same_correct_subset`；对正确子集为空的情况应返回 NA 而不是 0。论文的重复测量阈值和统计测试可作为基线实现，不能直接认为适用于一切性能环境。

## 9. 下次精读待核查

- 官方代码中的 `Performance` 与未通过正确性任务的具体处理流程。
- `OpenHands` 的最大迭代限制及真实工具使用成本（论文附录 C 声称最大迭代 2000），与其他基线预算是否可比。
- Figure 5 的 `Claude-3.7-opus` 标注与 Table 2 `Claude-3.7-sonnet` 的差异。
- 选择一个真实任务在仓库中核对 `instance_id`、expert patch、model patch、correctness/performance log。
