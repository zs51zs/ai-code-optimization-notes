# B01 · PerfAgent 对 Codex 的工程交接（先审查，后实现）

## 研究目标

通用 AI Agent 自动代码性能优化是主线；ParaLedger/RDMA/Slurm 是以后重要但非唯一的扩展场景。参考 PerfAgent（https://arxiv.org/html/2607.19653）方法，检查现有项目是否具备：采样证据、可重复验证、外部循环控制、最佳候选保存。

## 请先输出审查报告

在不重构现有系统的前提下，定位真实代码：

1. 何处调度 Agent 的循环/任务结束？能否捕获 STOP/提交 Patch？
2. 是否存在 profiler 工具适配接口？支持何种语言/进程？
3. 热点信息是否结构化保存（函数、行号、call path、self/inclusive time、原始证据路径）？
4. 代码修改是否被隔离到独立 worktree/工作区？
5. 正确性测试与运行时间测量是否分离？失败后能否回滚？
6. 是否存储每轮补丁、构建/测试日志、测量次数、性能方差、Git commit？
7. 有没有把“单次计时改善”和“统计上可信的收益”混为一谈？
8. 有没有防止 benchmark 数据泄漏、硬编码结果、测试路径投机的措施？

## 首期最小实现（仅在当前代码合适时）

- 建立 `ProfilerAdapter`，支持 Mock 数据和一种实际可用的单机采样器；Python 路线可调查 py-spy，C++ 路线可调查 perf。
- 建立 `OptimizationController`，最多 3–5 轮，保存每轮 Patch 与候选指标；禁止因模型说结束而提前忽略既定评测。
- 建立 `CorrectnessGate`，在每轮检查候选，失败不得选为最优。
- 建立 `CandidateStore`：patch hash、父版本、测试与测量日志、是否合格、最优标记。
- 构建最小对照：Base / Loop-only / Loop+Profiler / Loop+Tests / Full；预算相近，任务、模型固定。
- 真实集群作业需人工明确授权；不自动改动 ParaLedger 重要并发逻辑。

## 验收

- 使用不依赖集群的小程序可重现完整循环，包含一次正确提速和一次回退的模拟情形。
- 最优候选不一定是最后一次提交，且所有已接受候选通过 correctness gate。
- 使用真实而不是虚构的时延数据进行最终性能声明；模拟数据需明确标注。
- 输出 architecture diff、测试命令、限制和与 PerfAgent 论文的差异说明。

## 注意

论文作者在 §VI 明确：pytest-testmon 的覆盖主要针对 Python，对 native C/C++ 修改可能出现测试盲区。因此不要直接把此工具当成跨语言正确性保证。论文在 Mini-SWE-Agent 上验证，不得称其已经证明对我们的 Codex 现有版本同样有效。


---

## 2026-11-05 新增：Figure 2 / Figure 3 的专项工程核对

### 仅审查与小型技术验证，不直接重构核心系统

1. **证据处理**：审查是否保存原始 profile，是否把 setup 采样排除规则、样本总数、Self/Inclusive、调用栈和仓库/外部依赖标记记录为结构化格式。
2. **py-spy 适配器**：如果本地有可用 Python workload，在可控环境中验证 `py-spy` 的 raw/speedscope 采样和 `--native` 能否采集原生扩展帧；无权限或不支持时记录失败，不要通过修改系统安全配置强行启用。
3. **统一证据模型**：提出 `ProfileEvidence`（原始可追溯）和 `BottleneckHypothesis`（模型推断）两种不同类型；禁止把“样本占比”描述成“因果瓶颈已证实”。
4. **重新采样**：确认每轮优化之后是否针对新版本真正 re-profile，而不是复制初始摘要；正确性失败不得被标为最优。
5. **最佳补丁**：用模拟候选 1.40×、2.36×、5.56×、5.87×、5.85× 验证只保留第 4 轮。实际性能数据必须来自运行，不得使用论文案例数字充当本项目实测。
6. **防投机评价**：隔离或冻结 Benchmark harness，检查输入变化、冷启动/热启动、缓存和预计算漏洞；最终性能比较需独立复测。
7. **语言适配边界**：py-spy 不应作为 C++ standalone 或 RDMA 的统一采样器；预留其他 Profiler adapter 接口即可。ParaLedger 不在首期开发范围内。

### 推荐输出

- `docs/perfagent_fig2_fig3_mapping.md`：现有架构到各步骤的对应关系、缺口、无需改动的能力。
- `docs/profiler_evidence_schema.md`：采样事实与模型推断分离的 schema、字段定义、示例。
- `tests/test_best_patch_selector.py`：仅测试确定性的正确性/速度排序逻辑。
- `docs/validation_protocol.md`：正确性、计时、Profile、最终确认的不同门禁。

首轮仅提出最小修改计划及可执行小型实验，不要以“复刻 PerfAgent”名义大范围重构。详见笔记 `B_反馈优化Agent/B01_PerfAgent_精读笔记_v1.md` 第二次精读补充。


---

## 2026-11-05 第三轮追加：成本、公平比较、Reward Hacking 审查要求

> 先完成论文知识库的 Git Commit + Push，再进行本轮交接。当前文件是候选工程审查范围，不代表已批准代码修改或大型实验。

### 必须审查的六组问题

1. **五轮上限与停止原因**：现有系统是否记录 `max_submissions`、`actual_submissions`、`stop_reason`（预算、步数、正确性失败、构建失败、完成等）？论文 θ=5 是实验超参数，不能硬编码成通用最优。
2. **真实全流程成本**：除 LLM token/API 费用，能否追踪代码阅读、工具调用、测试、构建、Profiler 与实际耗时？费用口径应记录，不能用论文 $2.88/$4.25 当成本预测。
3. **候选基线公平性**：能否设计 one-shot、independent best@K（**不能让生产方法使用隐藏评价结果**）、feedback@K 的公平对照？至少报告候选数匹配与成本匹配两种实验。
4. **Reward Hacking 防御**：Agent 是否能修改 benchmark、测试脚本、隐藏输入或成绩汇总逻辑？是否检测全局缓存、测试栈识别、硬编码输出、跳过计算、针对固定数据形态投机？合法缓存不得无证据地一律拒绝。
5. **独立复测与选优偏差**：是否把开发期评分与最终独立测试分离？记录冷启动/热启动、变化输入、不同规模、跨进程重启以及测量方差；是否能区分“最快的测得候选”和“经过复测的可信候选”？
6. **指标正确性**：SWE-Perf 的 Performance、GSO 的 Opt@1/Sp@1、一般 Speedup 与 PerfAgent Hack-Adjusted 是否分开实现，禁止误称为一个指标？

### 建议交付内容（审查阶段，不要求直接实现）

- `docs/research/perfagent_budget_and_search_baselines_review.md`：现有代码位置、证据、缺口、建议测试矩阵和资源估算。
- `docs/research/benchmark_integrity_review.md`：评分器边界、潜在投机入口、隐藏输入的保护方法、缓存的正当性判断。
- `docs/research/candidate_selection_protocol.md`：候选正确性、性能统计、独立复测、最佳候选记录与回滚规则。
- 对现有代码文件提供函数级证据，不要声称只因存在文件名就已有完整功能。

### 参考论文实验数据（不得作为本项目成果）

- Table VI：GSO PerfAgent Hack-Adj 39.2%，oracle OpenHands best@5 26.5%，费用 $2.88 vs $11.01；SWE-Lite 分别 74% vs 68%，$4.25 vs $9.91。
- Table IX：SWE-Lite $5 预算第五轮仅 6/100 提交，$10 预算仅 12/100；总 Opt@1 75%→79%、Hack-Adj 74%→77%。
- Reward Hacking：原测量方案中 loop-only A_L 62% Opt@1、44% Hack-Adj，18 个可疑 Patch（14 个持久缓存/状态）；修改首轮计时并重跑后检测数量 18→3。不得混用旧、新方案的指标。
- 方法适用性：PerfAgent 在 Mini-SWE-Agent 上实现，基线运行了 OpenHands/Codex，尚不能断言当前我们的 Codex 工程已验证 PerfAgent 的性能提升。

下一步：将审查报告交回研究讨论，和后续 ShinkaEvolve / AlphaEvolve 文献比较后再决定正式原型与实验，不允许擅自执行生产或集群环境改动。
