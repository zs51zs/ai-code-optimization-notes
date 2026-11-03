# C02｜ShinkaEvolve：Towards Open-Ended and Sample-Efficient Program Evolution

> 状态：L1+｜2026-11-03｜未本地复现  
> 论文：https://arxiv.org/abs/2509.19349  
> ICLR 2026：https://proceedings.iclr.cc/paper_files/paper/2026/hash/7886b9bafe76c52fd568db10ff9772df-Abstract-Conference.html  
> 代码：https://github.com/SakanaAI/ShinkaEvolve

## 研究目标
LLM 进化式程序搜索往往耗费很多候选样本，ShinkaEvolve 研究在开放式发现任务上提高样本效率。

## 三个技术机制
1. Parent sampling：在利用高分候选与探索不同候选间平衡。
2. Novelty rejection sampling：避免反复评价重复/相似的代码候选。
3. Bandit-based LLM ensemble：在不同 LLM 之间做预算和选择策略。

## 官方代码可研究的入口
- 仓库 README 的 Quick Start / Python API。
- 入口概念 `ShinkaEvolveRunner`、`EvolutionConfig`、`DatabaseConfig`；
- `LocalJobConfig(eval_program_path=...)` 将评价器和进化循环隔离。
- README 展示 Slurm 适配例子；不代表自动支持任意分布式程序的性能分析。

## 与 PerfAgent 的关键区别
PerfAgent 的核心证据是热点分析和多轮反馈，ShinkaEvolve 的重点是候选种群管理和样本效率。我们可以研究二者组合，但目前只算研究假设，不是已证明的创新。

## 下次精读问题
- 父代选择的概率/得分公式及默认值？
- novelty filter 的特征定义与误杀风险？
- 评价器是否支持 correctness hard gate 与性能多目标？
- 与简单 random search / best-of-N 比较是否严格公平？

参考：https://github.com/SakanaAI/ShinkaEvolve
