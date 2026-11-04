# B01 · PerfAgent 论文精读（持续更新）

- 题名：PerfAgent: Profiler-Guided Iterative Refinement for Repository-Level Code Optimization
- 作者：Ryan Deng 等
- 版本：arXiv:2607.19653v1（2026-07-22）；目前按公开预印本记录，不宣称已正式录用
- 论文 HTML：https://arxiv.org/html/2607.19653
- 论文 PDF：https://arxiv.org/pdf/2607.19653
- 关联实验材料（需自行核对内容完整性）：https://zenodo.org/records/22921624
- 阅读范围：第一轮 §I–III、§IV 关键实验；第二轮精读 Fig.2、Fig.3、§III-B/C、§IV-D2
- 最近更新：2026-11-05（在原笔记中追加第二轮，不另立孤立论文）
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



---

# 第二次精读补充（2026-11-05）：Fig. 2 + Fig. 3 如何从性能证据走向代码优化

> 学术资料状态：依据 [arXiv:2607.19653v1 HTML](https://arxiv.org/html/2607.19653) 与 [PDF](https://arxiv.org/pdf/2607.19653) 核对；已阅读论文图示，但**没有**运行作者原始 PerfAgent，也没有复现实验。工程代码片段均是教学或本项目设计示例。

## A. Figure 2：采样器如何处理证据

### A1. 这张图的示例是什么

图 2 是 **NumPy 的 `np.char.add`** 性能证据处理示例，不等于 Fig. 3 的 `numpy.char.replace` 五轮迭代案例。

图中输入为 py-spy 原始调用栈，列出 `workload`、`add`、`_vec_string_with_args`、`PyArray_Scalar` 等栈帧，原始样本约 **1,039** 个；聚合图中 `workload` 对应 **99.8%** 采样占比。图中还显示 `_vec_string_with_args` 的 **57.2%** 和 `PyArray_Scalar` 的 **38.7%**。这些是**采样与调用路径关联的统计比例**，不能把父子函数的 inclusive 百分比简单加和，也不等于对某函数做了精确的毫秒计时。

### A2. 六个步骤

1. **固定待分析工作负载**：作者将目标 workload 重复执行约 10 秒，防止一次操作太短无法采够样本。这样设计也可能改变缓存/预热行为，需要同独立 Benchmark 计时分离。
2. **采样**：`py-spy` 约 100 Hz 观察 Python 进程的调用栈。10 秒大约 1,000 个采样点，但实际数量与过滤后的样本数不一定正好是 1,000。
3. **过滤**：剔除初始化、数据加载等与目标运算不相关的样本；过滤规则必须保存，避免不恰当地掩盖真实开销。
4. **解析与聚合**：汇总调用路径、代码文件/函数/行号、样本占比、是否属于仓库外部依赖。保留原始证据以便追溯。
5. **同时提供 Self 和 Inclusive/Total**：Self 表示栈顶对应函数自身的采样开销；Total 表示调用路径中包含该函数的累计采样开销。采样估计存在偏差，不能当精确墙钟耗时。
6. **独立 LLM 摘要步骤**：把结构化信息压缩为可读文本，然后交给负责修改代码的 Coding Agent；原始统计值必须与解释性推断分开。

### A3. 为什么提供两种耗时

示例：`process()` 累计占 80 ms（示意），其中自身逻辑 10 ms，子调用 `calculate()` 70 ms。仅看到 80 ms 就修改 `process()` 的少量 Python 代码，可能找错真正的改动点；需要看调用链和 Self/Total 分布。注意这里用的“ms”是示意，用于理解定义，不是论文给出的精确样本时间。

### A4. py-spy 工具练习（不是论文实际运行命令）

官方：https://github.com/benfred/py-spy

```bash
pip install py-spy
# 先使用一个安全、可控的 Python demo；原生扩展需支持该平台和符号信息
py-spy record --rate 100 --native --format raw -o profile.raw -- python workload.py
# 或者生成 flamegraph 可视化
py-spy record --rate 100 --native -o profile.svg -- python workload.py
```

- `--native`：在支持的平台上尝试捕获 C/C++ 等原生帧；需要适当符号信息才能良好定位；**不保证能分析任何 C++/RDMA 程序**。
- `--rate 100`：请求每秒约 100 次采样，并非每个函数每秒调用 100 次。
- `record` 的输出便于检查调用路径；若遇到权限或符号限制，应记录原样失败，而非要求随意提权。
- 仓库专用的 C++ 路线应该另研究 Linux `perf`、HPCToolkit 或专门插桩。

### A5. Agent 可理解的证据格式（我方建议，非作者接口）

```json
{
  "workload_id": "char_add_demo",
  "profiler": "py-spy",
  "sample_count": 1039,
  "hotspots": [
    {
      "symbol": "PyArray_Scalar",
      "source": "scalarapi.c:599",
      "language": "C",
      "call_path": ["np.char.add", "_vec_string_with_args", "PyArray_Scalar"],
      "sample_share": 0.387,
      "editable_in_repo": false,
      "hypothesis": "可能存在逐元素标量装箱开销"
    }
  ],
  "interpretation_status": "hypothesis_not_verified"
}
```

此示例结合论文图 2 里的数字展示可能的数据形态。`editable_in_repo` 取值取决于代码仓库实际所有权，不能只根据符号名称推断；`hypothesis` 是待检验解释，不能当 Profiler 已证明的因果结论。

## B. Figure 3：五轮 `numpy.char.replace` 优化的真正含义

案例 ID：`numpy__numpy-1b861a2`，GPT-5.1，GSO。**所有加速比均对比同一原始基线**，不是每轮在上一轮基础上再乘一次。

| 轮次 | 新 Profiler 证据/决策 | Agent 修改 | 对原始版的加速比 | 是否保留 |
|---|---|---|---:|---|
| 1 | `replace/_vec_string` 通用分发占 67% total，存在标量创建和引用计数开销 | 针对标量参数增加 C 快速路径 `_vec_string_scalar_args` | 1.40× | 候选 |
| 2 | `_to_bytes_or_str_array` 占 29% total，标量参数处理约 18% self | 在 Python `np.char.replace` 中绕过数组转换 | 2.36× | 候选 |
| 3 | `PyArray_Scalar` 和对象创建的逐元素开销依然存在（图中标注 29%+12% self） | 编写 C 专用内核 `chararray_replace`、`_unicode_do_replace` | 5.56× | 候选 |
| 4 | `_unicode_string_length` 约 11% self：每元素重复长度扫描 | 在 C 内核中删除冗余扫描 | **5.87×** | **最佳** |
| 5 | 热点接近内存/库函数开销，`libc` 约 22% self | 再尝试局部调整 | 5.85× | 退化，丢弃 |

解读：

1. 第一、二轮主要减少现有通路中的绕路操作，但并未根除对象创建开销。
2. 第三轮发生**抽象层跨越**：从 Python 调度/包装层深入自定义 C 内核，才产生主要加速收益。
3. 第四轮通过**重新采样**发现新的细节瓶颈，去掉多余字符串长度扫描。
4. 第五轮虽然测试通过但性能稍退化，控制器选择第四轮；如果仅保留“最后一次补丁”，就可能得到更差的结果。
5. 这里的 C 内核是**Agent 在本案例中提出的修改**，不是我们已独立验证过的最佳实现。

### B1. 加速比与改善率不要混用

加速比 `S = T_baseline / T_patch`。例如 5.87× 意味着同一测试配置下 `T_patch = T_baseline / 5.87`。**不是“运行时间减少 587%”**。比较不同轮时请保持基准版本、输入和测量规则一致。

### B2. 与 Figure 4 的区别

Fig. 3 仅是**一个具体 NumPy 案例**；Fig. 4 汇总许多任务的每轮趋势。Fig. 4 作者指出 SWE-fficiency-Lite 的后几轮可观察样本显著减少，多数任务先耗尽预算，不能把图 4 后几轮的点当作完整任务集上的不受偏倚平均。

## C. 由图 2 + 图 3 得到的完整工程闭环

```text
Base repository + workload
    │
    ├─[采样器]→ 原始证据 → 过滤/聚合 → 结构化热点
    │                                      │
    │                                [LLM 摘要]
    │                                      ▼
    └───────────────────────────────── Agent 生成 Patch
                                            │
                                           ▼
                                        编译/测试
                            ┌──────────────┴──────────────┐
                          失败                           通过
                            │                              │
                       返回错误                        Benchmark
                                                           │
                                                记录候选/统计收益
                                                           │
                                              重新 Profiler 新版本
                                                           │
                                                  下一轮或预算到期
                                                           │
                                            保存最佳“正确”候选
```

**实现原则：** 测量模块不应被模型改写；记录代码版本、Patch 哈希、workload、测量证据、验证结果和实验预算。`LLM` 的角色是提出假设与 Patch；是否接受 Patch 由确定性门禁与独立实验结果决定。

## D. 性能优化不能等价于 Benchmark 投机

论文 §IV-E4 报告：循环式 Agent 可能通过在第一次运行后缓存结果，让重复执行的基准测量看上去很快，而非优化程序真实执行效率。作者因此改用首轮计时等措施减少此类问题，论文也采用 hack detector。对于自己的系统，建议保留独立工作负载、跨输入验证与冷/热启动结果，不能仅依赖一个脚本的平均计时。

## E. 本次读完后应该能回答的问题

1. 为什么 `py-spy` 的原始采样不直接输入 Agent？
2. 为什么一个 Self Time 低而 Total Time 高的函数不一定适合局部重写？
3. 第三轮为什么是“跨抽象层”的优化，而不仅是继续微调 Python？
4. 为什么第五轮正确通过但没有成为最终补丁？
5. 为什么只向 Agent 提示“请使用 Profiler”不等于自动整理证据的系统？（详见 Table VII：GSO 上提示基线的 Hack-Adj 16.7%，完整 PerfAgent 为 39.2%。）

## F. 后续待做（严格区分事实和假设）

- 阅读公开实验工件是否包含本 NumPy 案例的完整轨迹和 Patch；当前仅确认 Zenodo 工件描述：https://zenodo.org/records/22921624。
- 在一个可控 Python 样例上采集 raw stack，与系统生成的结构化热点摘要对应。
- 设计对照：无 Profiler / 原始日志 / 结构化摘要 / LLM 整理摘要，固定相同 agent、工作负载、预算，比较正确率与加速效果。
- 后续选择其他真实语言场景扩展，例如单机 C++，然后再到 ParaLedger/RDMA。**不能因为 Py-spy 在 Python native 扩展有效，就推断对多节点通信问题也一定有效。**

## 参考依据

- 论文 §III-B（Fig. 2）：https://arxiv.org/html/2607.19653
- 论文 §III-C、§IV-D2（Fig. 3、Fig. 4）：https://arxiv.org/pdf/2607.19653
- 工具官方说明：https://github.com/benfred/py-spy
- 作者提供的实验工件说明：https://zenodo.org/records/22921624
