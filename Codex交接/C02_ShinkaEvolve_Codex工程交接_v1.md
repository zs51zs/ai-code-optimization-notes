# C02｜ShinkaEvolve → Codex 工程交接 v1（源码核查后）

> 状态：**正式研究交接与工程审查任务，暂不授权开发或执行实验**。
> 适用时机：结合先前 SWE-Perf / PerfAgent 架构审查报告。若之前的工程审查尚未完成，先把本文登记为后续审查输入，不抢占原任务。
> 版本：2026-11-09。ShinkaEvolve 公开 `main` 可能继续变化，开始实际编码前需固定 commit SHA。

## 一、最重要的方向纠偏

我们研究的是面向真实代码仓库、在正确性与测量可信前提下进行自动性能优化的 AI Agent，不是简单复刻 ShinkaEvolve，也不是将项目限定为 ParaLedger。ParaLedger 是未来重要但非唯一的验证场景。当前创新点尚未确定，任何“融合 PerfAgent + ShinkaEvolve 必然更好”的断言均未经实验验证。

本次只允许**读源码、对照已有实现、形成设计与实验方案**。不要自动安装 ShinkaEvolve skills、不要启动付费进化、不要直接重构 Agent，不要修改用户的集群项目和生产代码。

## 二、本次优先阅读

本地知识库：
1. `A_评测基准/` 中的 SWE-Perf 正确性及性能评价笔记；
2. `B_反馈优化Agent/B01_PerfAgent_精读笔记_v1.md`；
3. `C_进化式搜索/C02_ShinkaEvolve_精读笔记_v1.md`（重点最后“第三轮源码静态精读”）；
4. `跨论文综合/PerfAgent_vs_ShinkaEvolve_方法对照_v1.md`；
5. 此前 Codex 已完成的架构盘点（若有）。

官方源码：
- `shinka/database/dbase.py`：SQLite `programs`、`archive`、`add`、`_update_archive`、`sample`。
- `shinka/database/parents.py`：`WeightedSamplingStrategy`、`PowerLawSamplingStrategy`、`BestOfNSamplingStrategy`。
- `shinka/database/inspirations.py`：参考候选选择。
- `shinka/core/novelty_judge.py` / `async_novelty_judge.py`：相似度、LLM 判断与拒绝。
- `shinka/core/wrap_eval.py`：`run_shinka_eval()` 及 JSON 输出。
- `shinka/core/async_runner.py`：`ShinkaEvolveRunner` 与提案、评测、存储的异步链路。
- `docs/configuration.md`：当前默认参数。

如果本地没有互联网访问权限，应报告无法核对，不得臆造代码行号或已完成的运行测试。

## 三、必须回答的六个问题

1. **候选库**：我们的工程是否保存原始 commit、父代 ID、Patch、版本、正确性、原始性能样本、预算、失败原因？有无不可覆盖的版本记录？
2. **评测独立性**：能否在相同环境下构建/测试 Original 与 Candidate？不能把可选 `validate_fn` 当成充分正确性保证。工作负载与评价器是否和 Agent 可修改工作区隔离？
3. **已有搜索策略**：是否已经实现 One-shot、Independent Best-of-N、从最好 Patch 出发的迭代？这些策略的预算与输入能否公平比较？
4. **父代抽样**：以后实现候选库搜索时，当前架构是否容许以不同策略选择父代，并实现相同 Seed、任务和预算的对照？暂不要求实现 MAD 加权采样。
5. **候选去重**：是否已支持 Patch/代码哈希去重、重复率与拒绝成本记录？对 Embedding 去重和 LLM 判断，先做可行性/开销分析，不要直接集成。
6. **运行系统**：如果将来并行评测，能否区分候选提案并发、评测并发、总费用和总墙钟时间？是否能在失败/中断后重建状态？

对每一项结论，标注“已在源代码及测试确认 / 只有设计文档 / 部分实现 / 尚未实现 / 尚未验证”，并列出实际代码证据与路径。

## 四、建议的最小候选数据契约（讨论稿）

```text
Candidate:
  candidate_id, task_id, parent_id, source_commit, patch_path,
  status, correct, evaluation_id, raw_runtime_samples,
  verified_gain, scalar_score, model_id, iteration,
  generation_cost, eval_cost, wall_time, log_paths
```

这不是要求马上增加所有数据库列；请结合现有工程确定最少字段，保证原始样本、评价结果和 Patch 之间可追溯。

## 五、建议后续（需用户批准）的实验计划

- Baseline 1：One-shot。
- Baseline 2：Independent Best-of-N。
- Baseline 3：Hill Climbing（始终从最佳正确候选出发）。
- Proposed 4：Archive Parent Sampling（允许从多个正确候选抽父代）。
- Research 5：Profiler + Archive Sampling（检验两类证据是否互补，仍是假设）。

使用两类预算口径：固定正式评测次数，以及固定端到端费用/墙钟时间。跨组保持任务集、模型版本、工作负载、正确性门禁和测试硬件尽量一致。报告 Apply、Correctness、保守性能收益、达到门槛比例、重复候选率、成本和波动。不要将 `combined_score` 与 SWE-Perf 的 Performance 或 PerfAgent 的 Opt@1 直接混用。

## 六、这次交付什么

请输出：`C02_ShinkaEvolve_源码机制与现有架构差距审查.md`。包含：
- 官方源码审查入口与具体函数；
- 现有系统候选数据流与真实代码位置；
- 六个问题逐项盘点表；
- 最小设计/可复用接口建议；
- 正确性、统计测量、Reward Hacking 和并发/预算风险；
- 后续实验基线和验收标准；
- 待用户批准的任务及优先级。

**不得将此文档作为直接开始编码、安装插件、启动 LLM 付费实验、创建 Slurm 作业或修改 ParaLedger 的授权。** 完成报告后停止并回报。

## 官方来源

https://github.com/SakanaAI/ShinkaEvolve
https://github.com/SakanaAI/ShinkaEvolve/blob/main/shinka/database/parents.py
https://github.com/SakanaAI/ShinkaEvolve/blob/main/shinka/database/dbase.py
https://github.com/SakanaAI/ShinkaEvolve/blob/main/shinka/core/novelty_judge.py
https://github.com/SakanaAI/ShinkaEvolve/blob/main/shinka/core/wrap_eval.py
https://github.com/SakanaAI/ShinkaEvolve/blob/main/docs/configuration.md
