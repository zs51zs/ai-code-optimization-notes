# C01｜AlphaEvolve：A coding agent for scientific and algorithmic discovery

> 状态：L1｜2026-11-03  
> 论文：https://arxiv.org/abs/2506.13131  
> 官方说明：https://deepmind.google/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/  
> 数学问题相关资料：https://github.com/google-deepmind/alphaevolve_repository_of_problems

## 100 字理解
AlphaEvolve 把 LLM 用作候选代码修改生成器，将代码与自动评价器组合成进化式循环。探索如何通过候选库、反馈和搜索，在算法发现及计算基础设施优化中找到优秀实现。其核心机制是**优化搜索与可执行评价结合**，不是简单的单轮让模型改代码。

## 开源边界（重要）
`alphaevolve_repository_of_problems` 明确说明**不包含运行 AlphaEvolve 的核心代码**。不得误称已能直接安装 AlphaEvolve 主框架。可阅读论文与问题的验证代码；若要运行搜索，可研究 ShinkaEvolve/OpenEvolve 等独立项目，但不能将其声称为 Google 官方实现。

## 我们要思考
- 评价器是怎样确保“快但错误”无法通过？
- 多候选程序历史与一次次覆盖修改有什么区别？
- 什么样的任务有可自动量化的奖励？
- 以相同模型与运行预算比较多个框架，怎样公平？

## 进一步阅读
方法图、程序数据库、候选生成/评价、代表性应用、资源消耗与可开放程度。尚未对原论文逐段精读。
