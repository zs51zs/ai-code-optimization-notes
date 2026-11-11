# Codex 交接：研究目标重新对齐、V0/V1 最小闭环与后续版本路线（v1）

> **归档位置**：`Codex交接/综合阶段_Codex_V0V1工程审查与阶段实施计划_v1.md`  
> **整理日期**：2026-11-10  
> **用途**：可以直接提供给 Codex 的项目交接。**本文件不等于授权执行所有开发阶段。**

## 一、首先明确研究目标

项目名称暂称 **AI Agent 自主代码性能优化研究**。

研究目标：构建一个面向真实代码仓库、能够读取任务、定位性能问题、生成修改、验证正确性、测量可靠性能收益、利用反馈继续探索并报告完整证据的可扩展 Agent 实验平台。

- 研究主体是 **Agent 自动代码性能优化方法**，不是普通 Bug 修复任务。
- ParaLedger 是后续重要验证场景之一，但不是研究范围的唯一对象。
- 当前没有确定最终创新点，不能把“集成四篇论文”直接包装成创新。
- Codex 当前是**项目的工程实施助手**；最终研究系统也可以调用其他 Coding Agent/模型，它们不是同一概念。

## 二、与已有研究文档的关系

请在可访问时阅读这些资料（按实际文件名定位）：

1. `A_评测基准/` 下 SWE-Perf 的 §3–§5 精读与评测算法分析。
2. `B_反馈优化Agent/B01_PerfAgent_精读笔记_v1.md`。
3. `C_进化式搜索/C02_ShinkaEvolve_精读笔记_v1.md`。
4. `C_进化式搜索/C01_AlphaEvolve_精读笔记_v1.md`。
5. `跨论文综合/四论文到Agent_完整工作流与迭代开发路线_v1.md`。
6. `跨论文综合/性能反馈与多候选搜索融合_统一架构与研究假设_v1.md`。
7. 现有 `AGENTS.md`、README、架构和测试配置，以及以前生成的工程审查报告（如有）。

若文件不可读，**应说明缺失路径，不得宣称已阅读**。若现有工程已完成部分任务，优先复用和核实，不要重复创建另一套框架。

## 三、职责与边界

### Codex 负责

- 读取真实工程，识别已有模块、实际调用路径、构建与测试方法。
- 提出模块复用方案、记录技术债和缺口；给出具体文件/函数作为依据。
- 在得到授权后，实现限定范围内的最小功能并执行相关测试。
- 输出实验环境、日志、Patch、测试结果及失败情况。

### 研究负责人和协作分析负责

- 选择研究问题和评价任务、判断创新性。
- 审核技术方案与实验公平性。
- 决定何时从 V0/V1 进入高级研究模块。
- 对论文结论、研究假设、工程结果做明确区分。

### 本阶段禁止未经授权的行为

不得擅自大规模重构、批量调用高费用模型、进行长时间集群作业、改动 ParaLedger 等重要项目、修改隐藏测试/正式评价器换取更高分，也不得把未运行的测试写成已通过。

## 四、当前立即执行：Phase A 工程审查（默认仅审查）

请先检查以下项目。每行输出 `已实现并验证 / 已实现但未验证 / 部分实现 / 未实现 / 目前不需要`，并附具体证据：

| 领域 | 检查问题 | 优先级 |
|---|---|---|
| Git 状态 | 能否恢复精确 base Commit；候选补丁如何关联任务？ | P0 |
| Workspace | 能否隔离候选文件修改，防止串扰；失败如何回滚？ | P0 |
| Build & Correctness | 如何运行测试；检测超时与构建失败；是否允许 Agent 改测试？ | P0 |
| Benchmark | 原始版与候选版是否同环境测量；是否保存全部样本？ | P0 |
| 数据持久化 | 是否记录 Patch、候选 ID、任务 ID、环境、日志和成本？ | P0 |
| Agent Adapter | 是否能调用现有 Coding Agent 返回机器可应用的 Patch？ | P1 |
| Feedback Controller | 有无验证结果反馈、最佳正确候选、停止原因？ | P1 |
| Profiler | 是否有语言/工具适配器、可追溯热点输出？ | P1 |
| Candidate Pool | 是否能保存多父代版本和切换分支？ | P2 |
| Novelty / Cascade | 去重、级联评价与防投机是否已有基础？ | P2 |

**P0/P1/P2 表示审查与后续实现优先级，非本次全部必须开发。**

若已有以前的《项目目标与工程审查》报告，请对照本任务更新缺口，不要无条件重新开展全部调查。

## 五、建议的 V0 评价底座（下一阶段需明确批准）

### V0 最小输入

```yaml
task_id: example-001
repository: <local path or approved URL>
base_commit: <immutable commit hash>
workspace: <isolated directory>
allowed_edit_paths: [src/]
build_command: <explicit command>
correctness_command: <explicit command>
benchmark_command: <explicit command>
workload_id: <versioned id>
limits:
  timeout_s: <value>
  max_memory_mb: <value>
```

### V0 必须具备

1. 根据原始 Commit 创建干净且隔离的执行工作区。
2. 给定一个人工准备的 Patch，尝试应用；准确报告失败原因。
3. 运行构建、正确性与测试超时控制，禁止候选修改正式评测器。
4. 对 Original 与 Candidate 执行相同配置的重复性能测量，原始样本可追溯。
5. 独立计算和保存性能指标；若结论不稳健，应标注不确定，而非直接宣告成功。
6. 保存代码版本、日志、硬件/依赖环境、测试命令、工作负载、失败原因。
7. 最终能够恢复原始状态；不可把工作区的不可控副作用带入下次任务。

### V0 最小验收用例

| 用例 | 期待结果 |
|---|---|
| 正确且明显更快的人工 Patch | 正确性通过，记录可复测收益 |
| 行为错误但运行很快 | 正确性失败，不可被选为 Best Valid |
| 构建失败的 Patch | 清晰分类失败，不能进入性能比较 |
| 退化但正确的 Patch | 正确性通过，性能指标识别退化 |
| 重复运行同一候选 | 可关联同一版本和工作负载，保存全部样本 |

首次演示优先使用小型可控代码任务；这只是**Smoke Test**，不能代替论文正式实验。

## 六、V1 One-shot Agent 最小闭环（V0 合格后申请授权）

```text
TaskSpec → CodingAgent 一次输出 Patch → V0 Evaluator
         → 正确性/性能报告 → 持久化候选 → 结束
```

最低验收：

- 不依赖人工在中间修改源码，即可完成一次端到端任务；
- 生成的 Patch 能被唯一标识、查看、回放；
- 模型失败、超时、格式错误有明确状态；
- 最终报告与原始构建/测试/计时记录对应；
- 成本、模型版本和允许编辑的范围得到记录。

**不要求 V1 就有 Profiler、候选进化、自动 Prompt 搜索、分布式调度。**

## 七、后续版本规划（本轮不授权实施）

| 版本 | 主要新增能力 | 对照实验 | 进入条件 |
|---|---|---|---|
| V2 | 多轮反馈、回滚、最佳正确候选保存 | One-shot、同预算 Best-of-N | V1 可稳定重放 |
| V3 | Profiler 接口与结构化热点摘要 | 无/有 Profiler；不同摘要形式 | V2 与评价器可靠 |
| V4 | 多候选 Archive 与不同父代策略 | Best-of-N、Hill Climbing、Population | V3 与候选历史完善 |
| V5 | 证据感知分支切换、按需 Profiling、预算调度 | 2×2 因子与策略消融 | 单独机制效果已测清楚 |

并行记录两个成本口径：**正式候选评测次数**与**模型/Profiler/构建/测试端到端成本**。统一任务集、模型、评价器、硬件及正确性标准，结果尽量做多随机种子复测。

## 八、建议的数据与接口边界（先设计，不急着重构）

```text
TaskSpec
  ├── RepoRevision
  ├── AllowedEditScope
  ├── CorrectnessSuite
  ├── BenchmarkWorkload
  └── BudgetSpec

Candidate
  ├── candidate_id / parent_id
  ├── patch / code_revision
  ├── apply_build_correctness_status
  ├── timing_samples / profiler_artifact
  ├── estimated_gain / validation_status
  └── resource_usage

Services
  ├── CodingAgentAdapter
  ├── WorkspaceManager
  ├── VerifierAndBenchmark
  ├── CandidateStore
  └── SearchController (V2+)
```

接口是讨论草案，优先利用已有成熟实现。不必为了满足名称而重构现有代码。

## 九、Codex 需要输出什么

建议在工程文档目录生成：

`AI_Agent_现有工程审查_V0V1最小闭环方案.md`

必须包含：

1. 项目目标、非目标与现有代码结构。
2. P0/P1/P2 功能状态表，每项对应代码位置或运行证据。
3. 现有 AGENTS.md、测试/CI、环境及安全边界。
4. V0 的最小实施步骤、测试矩阵与验收标准。
5. V1 的 Agent 输入输出与实验记录方案。
6. 可以直接复用的组件、必须新增的组件、潜在风险。
7. 需要用户决策的事项（技术选型、初始任务、预算）。
8. 尚未确定或存在冲突的信息，**不得推测为已完成**。

完成后汇报，并等待授权。**默认到此为止，不自行进入大规模编码。**

## 十、可以直接发送给 Codex 的短消息

> 我们已完成 SWE-Perf、PerfAgent、ShinkaEvolve、AlphaEvolve 四篇核心论文的第一轮研究整合。本阶段请以 `Codex交接/综合阶段_Codex_V0V1工程审查与阶段实施计划_v1.md` 为任务依据，并参考跨论文综合的系统路线与反馈/多候选融合研究文档。先确认当前工程已有能力，完成 Phase A 审查与 V0/V1 最小闭环设计。请标明具体源码证据，不要凭文档宣称已经实现，不要启动复杂进化开发或未经授权的正式实验。完成后输出报告，等待进一步批准。

## 论文入口

- SWE-Perf: https://arxiv.org/abs/2507.12415
- PerfAgent: https://arxiv.org/abs/2607.19653
- ShinkaEvolve: https://arxiv.org/abs/2509.19349
- AlphaEvolve: https://arxiv.org/abs/2506.13131
