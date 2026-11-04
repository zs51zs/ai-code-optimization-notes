# B01 · PerfAgent 论文精读（第一次）

- 题名：PerfAgent: Profiler-Guided Iterative Refinement for Repository-Level Code Optimization
- 作者：Ryan Deng 等
- 版本：arXiv:2607.19653v1（2026-07-22）；目前按公开预印本记录，不宣称已正式录用
- 论文 HTML：https://arxiv.org/html/2607.19653
- 论文 PDF：https://arxiv.org/pdf/2607.19653
- 关联实验材料（需自行核对内容完整性）：https://zenodo.org/records/22921624
- 本次阅读范围：§I–III 全文；§IV 重点表 II–IV、VI–VII、案例；§VI 局限
- 阅读等级：L2 深入方法和论文数据；尚未在本地复现/逐行检查作者源代码
- 研究主线：通用 AI Agent 自动代码性能优化；ParaLedger 只是未来应用验证场景

## 1. 一句话结论

PerfAgent 并不是新的 LLM，而是一套叠加在通用 Coding Agent 外面的“性能采样摘要 + 目标驱动迭代控制器 + 选择性测试”工作流。它利用运行证据决定优化方向，每轮验证代码正确性和效果，并保存历次正确候选中最快的补丁。

依据：论文 §I、§III，Fig. 1。

## 2. 研究问题与三类失败

| 失败模式 | 后果 | 提议模块 |
|---|---|---|
| Missing real bottleneck | 仅看 Python 顶层代码，忽略 C/C++、Cython/Rust 等本地扩展热点 | Curated profiler usage |
| Premature termination | 第一次得到有收益的 Patch 就停止 | Objective-driven loop controller |
| Insufficient testing | 优化了目标脚本却破坏其他功能路径 | Selective validation |

## 3. 系统结构

- 内层 Agent：Mini-SWE-Agent 读取仓库、编辑文件、提交候选 Patch。
- 外层控制器：拦截 Agent 结束信号，构建和测试候选，重新测量速度，重新采样，并决定继续。
- 初次采用原代码的 profiler 摘要；之后采用上次修改版的 profiler 摘要。
- 每个任务最多 θ=5 次候选提交；按照测试工作负载测得的速度，从正确候选中保存最快者，而不是最后一个候选。
- 关键实现是外部结构化反馈，不是只用提示词“请继续优化”。

来源：§III-A/C、Fig.1。

## 4. Profiler 证据管道——§III-B、Fig.2

1. 用 py-spy 对目标工作负载采样；论文设置约 100 Hz。
2. 将原 workload 调整为约 10 秒的重复执行窗口以收集足够样本。
3. 过滤数据加载、初始化等 setup 相关样本。
4. 解析调用栈并汇总热点：文件、函数、行号、完整调用上下文、样本占比、是否属于外部库。
5. 同时提供 self-time 与 total-time：前者描述函数自身耗时，后者描述含子调用的累计耗时。二者帮助判断应重写函数本身，还是减少它的调用次数。
6. 用另一次 LLM 调用把结构化结果压缩成可操作的自然语言摘要，供 Coding Agent 使用。

解释：Profiler 采样比例只是“采样时的时间分布近似”，并非对原因的严格证明。开销、采样偏差、跨语言符号解析都要验证。

示意反馈（虚构）：

```yaml
workload: replace_large_strings
hotspots:
  - symbol: PyArray_Scalar
    language: C
    self_share: 0.38
    inclusive_share: 0.42
    call_path: [numpy.char.replace, _vec_string, PyArray_Scalar]
    action_hint: 减少逐元素标量封装开销
```

**提醒：** 此 YAML 只是我方设计建议，不是作者公开接口规范。

## 5. 迭代控制与正确性：§III-C/D

- 接收到 Agent 的 STOP 后截取 Patch，不立即结束。
- 重新构建，利用 pytest-testmon 的代码覆盖数据选择受影响的测试。失败则把错误反馈给 Agent，不能当作合格最优版本。
- 正确性通过后重新测量、重新 Profiler，提供新的热点与实际速度反馈。
- 每轮保留目前已通过测试且测得最快的版本；继续直到资源预算/轮次结束。
- 作者报告选择性测试相对全套测试在 GSO 减少 66%–99% 的测试数，在 SWE-fficiency-Lite 减少 47%–98%。这些是特定仓库的结果，不可直接推广。
- **关键局限：pytest-testmon 主要依据 Python 覆盖信息，可能漏掉 native-extension 内部行为的变化**，不能替代最后的完整或领域专用正确性门禁。

来源：§III-C/D、§VI。

## 6. 指标：不要和 SWE-Perf 混淆

两个基准：GSO（102 题，10 个仓库，包含 native 语言修改）；SWE-fficiency-Lite（100 题，9 仓库）。

设原始程序耗时 T0，Agent 版本 TA，专家版本 TH，则：

- Speedup_Agent = T0 / TA
- Speedup_Human = T0 / TH
- SR = Speedup_Agent / Speedup_Human
- Correctness：通过基准规定的正确性测试的任务比例
- Sp@1：正确且至少实现 1.2x 加速的任务比例（按论文定义）
- Opt@1：正确且 SR >= 0.95 的任务比例（按 GSO 的定义，论文统一采用）
- Hack-Adj.：在 Opt@1 基础上扣除被识别为投机/作弊的优化

重要：Opt@1 与 SWE-Perf 的 Performance 不是相同指标，也不是平均性能提升率。

## 7. 主实验表（GPT-5.1，论文 Table II）

| 方法 | GSO Correctness | GSO Hack-Adj | SWE-Lite Correctness | SWE-Lite Hack-Adj |
|---|---:|---:|---:|---:|
| OpenHands | 88.2% | 19.6% | 82% | 26% |
| Codex | 89.2% | 17.7% | 80% | 39% |
| Loop only | 91.2% | 29.4% | 80% | 46% |
| Loop + Tests | 96.1% | 20.6% | 93% | 46% |
| Loop + Profiler | 95.1% | 34.4% | 83% | 57% |
| 完整 PerfAgent | 96.1% | **39.2%** | 90% | **74%** |

注意：原表同时给出 Sp@1 和未调整的 Opt@1；这里仅摘录核心项，避免信息过载。

### 消融结论
- 仅迭代 + 时间反馈已经能提升表现，但可能带来正确性或 reward hacking 风险。
- 加测试反馈主要提高正确率，并不一定单独提高优化成功率：GSO 中 Loop+Tests 的 Hack-Adj 反而低于 Loop only。
- 加 Profiler 提高性能优化能力，增加测试提高可靠性；**组合后**通常最好，但不能据此声称收益严格相加。
- “让 Agent 自己学着用 Profiler”基线 A_P 在 GSO Hack-Adj 16.7%，明显低于 PerfAgent 39.2%；SWE-Lite 是 44% 对 74%（Table VII）。

## 8. 案例：numpy.char.replace（§IV-D2、Fig.3）

目标任务：numpy__numpy-1b861a2。

| 轮次 | 修改方向 | 报告加速比 | 结果 |
|---|---|---:|---|
| 1 | C 级标量参数 fast path | 1.40x | 继续 |
| 2 | Python 级别避免数组转换 | 2.36x | 继续 |
| 3 | 创建专用 C 内核 chararray_replace | 5.56x | 继续 |
| 4 | 消除每元素重复的字符串长度扫描 | 5.87x | 选为最优 |
| 5 | 尝试进一步修改 | 5.85x | 退化，拒绝 |

最重要的发现：新一轮 Profiler 暴露了之前 Python 层修改无法消除的底层 C 开销，引导 Agent 穿越抽象边界。这样“修改哪里”也会随迭代重新判断，而不是固定一个初始热点。

## 9. 预算、公平比较与投机行为

- 每任务 θ=5，$5 预算，200 步上限，作者使用特定 AWS 实例；不是部署到任意环境即可获得同样结果。
- GSO 上，完整 PerfAgent 的 Hack-Adj 为 39.2%，OpenHands oracle best@5 为 26.5%；平均成本分别 $2.88 和 $11.01（Table VI）。
- SWE-fficiency-Lite 上对应 74% vs 68%，平均成本 $4.25 vs $9.91。
- “oracle best@5”依赖正式评测时的隐藏结果挑选最佳，并非可部署的普通优化器。
- 作者发现一些 Agent 通过缓存 benchmark 循环中的结果使计时看起来变快，所以提供 hack-adjusted 评价，强调 workload 隔离和反投机验证。
- $10 预算消融使 SWE-Lite 的 Opt@1 从 75% 上升到 79%，说明预算会影响结论。

## 10. 研究限制与尚未验证的部分

- 研究是在 Mini-SWE-Agent 上实现的；“适用于其他 agent”是设计上的可移植性，不等于已经实证证明在 Codex、OpenHands 等底座上同样有效。
- 两类基准以 Python 生态为主，对 C++ 单体应用和多节点程序的外推需验证。
- GSO 有隐藏性能测试，SWE-fficiency-Lite 性能评价主要是提供的脚本，更容易受过拟合/缓存投机影响。
- 实验主要是单次 agent run；小幅差异需谨慎解释。
- 论文目前未在本文已核查资料中确认一个完整且可直接复现的作者官方源码仓库。Zenodo 有结果、patch、轨迹和 prompts 的工件描述，但不能仅据此断言它包含 PerfAgent 的全部运行代码。

## 11. 与我们系统的连接（工程建议，不是论文结果）

将主要能力设计为可替换模块，而非绑定单个 benchmark：

1. `ProfilerAdapter`：Python py-spy / 未来 C++ perf / 分布式多节点采样。
2. `ProfileSummary`：结构化热点与调用链，保存原始证据，LLM 只负责摘要解释。
3. `OptimizationController`：控制迭代预算、是否继续、失败回滚及 best-patch。
4. `CorrectnessGate`：受影响测试 + 最终完整测试 + 领域专项测试。
5. `BenchmarkEvaluator`：与编码 agent 分离，独立报告真实耗时、统计不确定性。
6. `CandidateStore`：存储 patch hash、构建状态、测试日志、性能结果。

### 建议科研对照

在固定相同模型、同样任务、相近预算的条件下比较：

- Base Agent：无性能证据、单轮修改；
- + Runtime Loop：多轮 + 运行时间反馈；
- + Curated Profiler：多轮 + 热点结构化反馈；
- + Tests：多轮 + 选择性测试；
- Full：全部组合。

优先在受控 Python/C++ 实验上完成，后续使用 ParaLedger 作为具有特殊通信/并行语义的应用场景。

## 12. 给开题报告的谨慎表述

“现有研究显示，为通用编码智能体提供运行时性能采样、结构化热点反馈与正确性门禁，能够显著提高部分真实仓库优化任务的成功率。但目前相关实验对特定语言生态、工作负载与 agent 底座的依赖较强，跨语言、多轮候选评价效率以及复杂分布式程序中的通用性能证据表示仍值得进一步研究。”

以上为研究动机，不应称之为我们已经证实的创新点。

## 13. 下一轮精读任务

- 论文 Fig. 2：采样与整理详细步骤；结合一个小程序运行 py-spy 示范。
- Table II/III：完整数值、配对胜负及统计证据。
- Appendix B：pytest-testmon 受影响测试筛选方法和开销。
- 核查 Zenodo 工件结构、作者是否正式释放完整运行代码；不要混用另一个名称相似的 eBPF perf-agent 仓库。
- 精读 GSO / SWE-fficiency-Lite 的测评脚本，明确怎样判定缓存投机行为。

