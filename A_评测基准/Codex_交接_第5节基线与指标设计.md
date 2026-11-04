# Codex 交接：SWE-Perf §5 实验启示（仅研究与设计，不要求立即重构）

目标：核对我们现有自动代码优化系统的评测设计，判断能否对照 SWE-Perf 的 Oracle / Agentless / OpenHands 三范式做可比实验。

## 必须先审计而非假定存在
- 当前代码仓库是否已有候选 Patch 生成、Git 隔离工作区、编译与正确性验证、性能重复测量、运行记录？
- 当前 FinePerf 的任何字段是否把 `Performance`、正确率、平均加速比、保守最小收益混用？
- 是否存在对照评测中向模型泄露 Expert Patch 的风险？

## 建议补充的最小评测输出
- 每个 task / model / method 的 apply_status、correctness_status、correctness_tests、raw_timings_original、raw_timings_candidate、gain_simple、gain_statistical、llm_calls、benchmark_calls、wall_clock_budget。
- Overall 全任务的性能分数，与只在正确任务上的分数，均需记录对应样本数量；若无正确样本，条件得分是 NA。
- Expert 参考收益务必用**该方法正确任务的同一集合**重新平均，避免 Figure 5 的可比性陷阱。

## 未来需要验证的最小对照
1. Single-pass 仅一次 LLM Patch；
2. Fixed pipeline（定位→生成→验证）；
3. Feedback Agent（定位→运行 profiler→生成→验证→迭代）。

三个系统要使用明确记录的任务、硬件、模型与调用/测试预算。不能从不同工具、不同任务集合的结果直接归因某种算法优越。

## 仅交付
`docs/evaluation/sweperf_section5_gap_analysis.md`：现状、可复用模块、指标/预算不一致点、最小改动路线与需要人工批准的高开销测试。**没有用户授权不提交集群实验，不修改敏感并发逻辑。**

参考：https://arxiv.org/html/2507.12415 和 https://github.com/SWE-Perf/SWE-Perf
