# B01｜PerfAgent：Profiler-Guided Iterative Refinement for Repository-Level Code Optimization

> 状态：L1+（方法与主要消融已核对；**尚未完整精读论文/复现源码**）｜2026-11-03  
> 原文 HTML：https://arxiv.org/html/2607.19653  
> arXiv：https://arxiv.org/abs/2607.19653  
> 官方实现：本轮尚未核验存在，**不得把其他 repo 冒充官方代码**。

## 核心研究问题
通用 Coding Agent 容易错过真正热点、见到一点提升就停止、缺乏足够正确性测试。PerfAgent 通过 curated profiler summary、objective-driven controller 和 selected correctness tests 处理这三类问题。这里讲的是**性能反馈闭环方法**，不是一个通用 Benchmark。

## 工作流程（§III）
1. 在原始仓库的工作负载上进行采样 Profiling。
2. 用 py-spy 采样；过滤设置阶段堆栈，聚合 self-time/total-time、路径、调用上下文、是否库外代码等信息；再把摘要送入模型。
3. Coding Agent 生成 Patch；外层控制器应用、构建、执行选择性测试。
4. 正确则重新采样和计时，失败则返回构建/测试错误。
5. 即使 Agent 发出停止信号，也在预算内继续迭代，保留历轮中速度最高且通过验证的候选。

## §IV 代表性实验信息
- 在 GSO（102 任务）与 SWE-fficiency-Lite（100 任务）上评估；论文使用 GPT-5.1、Kimi-K2。
- 报告的 `Opt@1` 为达到人类专家加速水平一定比例（≥0.95）且正确的任务占比；`Hack-Adj` 还考虑检测到的奖励投机。
- 在 GPT-5.1 条件下，GSO 的 OpenHands Hack-Adj. 为 19.6%，PerfAgent 为 39.2%；SWE-fficiency-Lite 分别为 26% 和 74%。这是论文对应条件的结果，不能跨 Benchmark 与 SWE-Perf 的 Performance 混用。
- 消融提示：循环反馈、Profiler 信息、选择性测试的作用不同，组合方法最好；选择性验证可增加正确性，但本身不一定提升最优加速效果。
- **重要限制**：预算、隐藏测试、reward hacking 与工作负载泛化问题必须保留在复现设计中。

## 下一次 L2 精读任务
- §III-B：如何把采样原始数据归一成 Agent 能理解的证据？
- §III-C：STOP 拦截、迭代预算、最好候选选择；如果失败回滚到哪里？
- §III-D：pytest-testmon 如何选择测试；选择性测试无法形式化保证行为全等。
- §IV-C：逐项分析消融，而非只看最终分数。
- §IV-E：成本、reward hacking、开源模型与泛化限制。

## Codex 候选需求（待代码审查）
Profiler 适配接口；结构化性能摘要；每轮候选隔离；正确性门禁；可控迭代预算；保留最优正确版本。**先做可验证设计，不应照论文假定官方代码可直接安装**。

参考章节：https://arxiv.org/html/2607.19653
