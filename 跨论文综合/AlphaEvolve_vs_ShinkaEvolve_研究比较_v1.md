# AlphaEvolve 与 ShinkaEvolve：研究方法比较 v1

> 整理日期：2026-11-09。仅代表本阶段论文与官方公开资料的研究认识，不是本项目实验结果。

| 对比方面 | AlphaEvolve | ShinkaEvolve | 对我们的研究启发 |
|---|---|---|---|
| 任务重点 | 自动可评价的算法与程序搜索 | 开放式、多样性与样本高效的程序进化 | 先定义可信任务和评价器 |
| 候选生成 | LLM 基于父代和历史上下文生成 diff / 可编辑区域重写 | diff、full rewrite、crossover 等操作 | 将变异接口与具体模型解耦 |
| 候选历史 | 进化数据库，MAP-Elites / Island-inspired | 程序 DB/Archive、Island、父代采样 | 历史记录和当前活跃候选分开 |
| 选择机制 | 论文主要公开数据库架构思想 | 论文详细给出性能+子代数加权父代公式 | 具体公式需结合任务做公平对照 |
| 搜索冗余 | 通过多样性维护与 Prompt 历史 | 论文突出 Embedding+LLM Novelty Rejection | 去重须考虑误杀微小但有效的修改 |
| LLM | Gemini 2.0 Flash + Pro，兼顾吞吐与质量 | 集成 LLM，Bandit 动态优先级 | 先有单模型基线，再比较多模型成本 |
| 评价 | 用户定义 h、多维分数、级联和并行 | 自定义 Evaluator、重复运行与评分 | 强制独立正确性门禁，测试分层 |
| 工程扩展 | 异步分布式流水线，优化候选吞吐 | 具备多类运行后端和候选数据库实现 | 先完成最小闭环，再扩展吞吐 |
| 官方代码 | Google DeepMind 已公开 src 最小本地实现；非全部内部系统 | Sakana AI 官方框架 | 可以比较模块界面，不宜直接大规模复刻 |

## 共同的研究假设

相同总预算下，历史程序与验证反馈能否帮助 Agent 比独立 One-shot 生成更多**正确且真正高性能**的代码？其中哪些增益来自算法搜索，哪些来自评价器、模型能力和额外测试资源？

## 我们未来的对照要求

除了评测候选数量，还必须报告模型 API 费用、Token、总墙钟时间、构建/测试 CPU 或 GPU 资源与停止原因。比较性能时使用相同的工作负载与独立验证数据，不能把数学分数和代码加速百分比直接混合。

## 来源

- https://arxiv.org/html/2506.13131
- https://arxiv.org/html/2509.19349
- https://github.com/google-deepmind/alphaevolve
- https://github.com/SakanaAI/ShinkaEvolve
