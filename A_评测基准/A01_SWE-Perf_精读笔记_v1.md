# A01｜SWE-Perf: Can Language Models Optimize Code Performance on Real-World Repositories?

> 状态：**L2 初步精读（§3、§4 已重点阅读，§5 初读；未本地复现）**  
> 日期：2026-11-03  
> 发表：ICML 2026，PMLR 306:41651–41666  
> 官方发表：https://proceedings.mlr.press/v306/he26y.html  
> arXiv HTML：https://arxiv.org/html/2507.12415  
> 官方代码：https://github.com/SWE-Perf/SWE-Perf  
> DOI/arXiv 标识：arXiv:2507.12415

## 0. 一句话/百字摘要
SWE-Perf 提出用于**真实软件仓库代码性能优化**的 Benchmark：从真实优化 PR 筛出 140 个任务（最终覆盖 9 个仓库），保留原始仓库、专家 Patch、性能相关测试及执行环境，并以可应用性、正确性和统计意义上的性能收益三个指标评估 LLM。它是**评测基准**，不是一个新的自动优化 Agent。论文显示所评估的 LLM/Agent 与专家改动仍有差距。

## 1. 研究背景与定位（§1–§2）
- 常规代码基准偏“修复正确性”；局部效率基准多是单个函数。真实仓库的性能改进可能跨文件、模块和抽象层，且需要可运行环境与充分正确性验证。
- Benchmark 的必要性：没有真实专家改动及对应可测工作负载，就很难衡量优化效果究竟有多少提升空间。
- 容易混淆：SWE-Perf ≠ SWE-bench，前者评**性能优化**，后者主要评软件问题修复。
- 原文依据：§1、§2（https://arxiv.org/html/2507.12415）。

## 2. 任务输入输出（§3.1）
输入：指定版本的代码仓库 + 目标函数集（Oracle 和 Realistic 的目标信息不同）；输出：应用在原始仓库上的优化 Patch/代码。评价依赖性能相关测试、实际工作环境和专家 Patch。

### Oracle vs Realistic（§3.2 Phase 5）
- **Oracle（文件级）**：给出由专家 Patch 识别的直接修改目标函数及其上下文，降低定位难度。
- **Realistic（仓库级）**：给出与性能测试相关的目标函数（不一定就是最终要改的函数），Agent 需自主导航代码仓库、定位优化机会，可以跨文件修改。
- **关键限制**：论文的 Realistic 也并非完全无提示从零寻找所有热点，仍有与测试关联的目标函数信息；不要把它表述成任意未知项目的完全自主诊断。

## 3. 数据集构建的五阶段流程（§3.2）
1. 收集 GitHub PR：从 SWE-bench 采用的 12 个候选仓库开始，筛选可能相关的 PR；最终数据集落在 9 个仓库。
2. 测量 PR 前后代码：为对应版本构建 Docker 环境，用 pytest 运行单测并记录每条测试耗时。初筛阶段每个测试测量三次；论文注明阶段性资源限制，初筛 1 CPU / 16 GB，后期稳定性验证与评价使用不同 CPU 配置。
3. 筛出有价值的 PR：相关测试在前后版本均通过，三次测量的性能改善超过初筛阈值（论文为 0.3）；还要求测试确实执行修改代码，避免靠修改测试本身获得提升。
4. 验证提升稳定：预热、每项测试 20 次、IQR 过滤离群值，再使用 Mann–Whitney U 检验和逐步调整的下界估计，最终留下满足稳定最小改善门槛的实例。注意论文使用显著性水平 0.1 与最小增益 0.05，不能把这一门槛普遍化为所有生产系统的推荐设置。
5. 提取目标函数：AST / diff 用于 Oracle，动态运行调用信息（yappi）辅助 Realistic 目标构建。

**关键科学问题**：只有“测试覆盖被修改代码”仍不足以证明没有行为变化；后续可考虑隐藏测试、属性测试、差分测试等更强验证。

## 4. 评价框架（§4）
分层门禁：

1. **Apply**：生成 Patch 是否成功应用到原始代码（整体比例）。
2. **Correctness**：在该基准选择的性能相关测试上能否全部通过（整体比例）。这不是数学证明，也不等于通过整个仓库所有测试。
3. **Performance**：执行原始与修改后代码，预热和多次测量；计算每个性能相关测试的保守最小增益，再跨测试和实例求平均。

论文定义的单测试耗时改善率：

`Ratio = (R_original - R_modified) / R_original`

若运行时间从 1 秒变成 0.8 秒，改善率为 20%，加速比为 1.25x；二者不是同一数值。

论文用于报告的 Performance 是保守最小增益的平均，**不是优化成功比例，不是平均加速比，也不能直接当作每个案例的改善率**。

论文明确在评价阶段重新测量原始版本，而不是直接拿数据采集阶段的旧耗时比较，以减小不同运行环境导致的偏差。

### 可直接借鉴的实验原则
- Patch 必须可应用且构建成功；
- 正确性是硬门槛；
- 性能比较要同步测量基线；
- 需要预热、多次运行和保留原始测量数据；
- 用相同输入/环境/资源预算比较不同优化策略。

## 5. 原文部分实验结果（§5 Table 2）
论文表 2 在 Realistic 设置下：Claude-3.7-sonnet + OpenHands 的 Apply 为 87.86%，Correctness 为 77.86%，Performance 为 **2.26%**；专家 Performance 为 **10.85%**。这些是**按其定义的评测指标**，绝不能说是“仅有 2.26% 的任务优化成功”。

解释：目前通用 Agent 即使能提交且通过相关测试的 Patch，也可能仍未发现高价值优化。由于模型与数据的时代背景、评测设置均固定，不能把这一结果当成所有 2026 年模型的最新水平。

## 6. 局限与批判
- 以 Python 生态及其测试工具链为主，与 C++ / 多节点程序不同。
- 基准中目标函数有不同程度提示，不能代表任意程序零提示分析。
- 测试集及工作负载覆盖有限，存在仅对指定测试优化而不保持完整行为的风险。
- 测试任务非常短的情形易受噪声/测试框架开销影响；论文采用稳健测量程序，但对我们的工程还应做噪声敏感性实验。
- 以上最后一项属于研究者提出的审查要点，不是论文已证明的缺陷。

## 7. 对我们的价值
### 可复用思想
- Benchmark task schema（代码版本、Patch、工作负载、测试、运行环境、基线、结果）；
- Patch 可应用性 → Correctness → Performance 的三层验证；
- 在模型搜索工作流程之外独立维护可信的评价器。

### 不该直接照搬
- 仅 Python/pytest 的任务定义；后续要适配构建型项目、C++ 和跨节点任务。
- 精确的重复次数和统计阈值需依据实验开销、噪声和显著性需求确定。
- 某论文某模型的分数不能直接转用作我们系统的验收目标。

### 候选研究假设（尚未验证）
不同形式的 Profiler 证据与 Patch 搜索策略能否在相同测试预算下提升 Performance 与正确率，并减少假阳性加速？

## 8. 给 Codex 的候选工程需求（必须先对照现有源码）
- 定义通用 `OptimizationTask`：`repository`, `base_commit`, `workload_command`, `test_command`, `constraints`, `runtime_env`, `allowed_paths`；
- 定义 `PatchEvaluation`：`apply_status`, `build_status`, `correctness_status`, `runtime_samples_before`, `runtime_samples_after`, `performance_gain`, `artifact_paths`, `diagnostics`；
- 三层门禁做成与 LLM 解耦的确定性评测，保存所有原始样本和版本。
- 支持 dry-run 和 Mock；禁止未授权启动昂贵多节点实验。
- 增加测试：Patch 冲突、测试失败、结果退化、噪声过大、输入不一致、超时与评价器被修改。
- **不要立即改写已有架构；先由 Codex 检查哪些接口与已有 FinePerf / Benchmark 模块重复。**

## 9. 可用于开题的相关工作表达（初稿）
He 等人在 ICML 2026 提出 SWE-Perf，从真实开源仓库中筛选性能优化任务，并同时评价代码补丁可应用性、功能正确性和统计支持下的性能收益，为仓库级代码性能优化建立了相对统一的实验基础。该基准揭示了通用语言模型与 Coding Agent 在实际性能优化任务上仍存在显著差距，说明运行反馈、定位和验证等能力值得进一步研究。该工作侧重**评测体系**，并非直接提出通用自主优化方法。

## 10. 需要继续思考的精读问题
1. 为什么不是所有“变快的 PR”都能成为可靠 Benchmark 实例？
2. Oracle 与 Realistic 的输入信息差别，对比较 Agent 公平性有什么影响？
3. Apply / Correctness / Performance 三个指标为何必须分开？
4. 为什么专家最优不等于理论最优？
5. 如果要加入 C++ / GPU / RDMA 程序，哪些评测前提首先失效？
6. 论文的 Performance 与 PerfAgent 的 Opt@1 根本不同，如何避免错误比较？

## 11. 原文核查与代码阅读入口
- 论文 §3.1 任务定义、§3.2 数据筛选、算法 1 稳定性、§4 指标、§5 Table 2：https://arxiv.org/html/2507.12415
- 发表与作者：https://proceedings.mlr.press/v306/he26y.html
- 仓库 README：https://github.com/SWE-Perf/SWE-Perf
- 仓库 `data_collection/`：https://github.com/SWE-Perf/SWE-Perf/tree/main/data_collection
- 仓库 `generation/`：https://github.com/SWE-Perf/SWE-Perf/tree/main/generation
- 仓库 `evaluation/`：https://github.com/SWE-Perf/SWE-Perf/tree/main/evaluation

**审计说明**：本轮核对了仓库根目录和 README 对上述目录的说明；并未本地运行评价器或逐个核对目录下的具体函数。链接中的目录能否在当前网络访问，需要 Codex 下一步检查。
