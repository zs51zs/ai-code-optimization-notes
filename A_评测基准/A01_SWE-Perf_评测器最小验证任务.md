# A01 SWE-Perf 精读 → Codex 工程研究任务（候选，不要求立即执行）

## 来源
- 论文：https://arxiv.org/html/2507.12415 （§3、§4）
- 评测工具：https://github.com/SWE-Perf/SWE-Perf/blob/main/evaluation/README.md
- 对应精读：`../A_评测基准/A01_SWE-Perf_第3-4节精读_v2.md`

## 目标
审计当前 Performance Evaluation/FinePerf 模块是否已经实现“Patch 应用 → 正确性 → 性能测量”的独立、确定性门禁。优先复用现有模块；不得仅因论文存在就重构主 Agent。

## 先检查
1. 现有 `BenchmarkRunner` 或同等接口及日志数据模型。
2. 是否保存基线和候选版本的 commit、命令、输入、CPU/内存配置、每次采样与失败原因。
3. 是否允许任意 Patch 修改评测脚本、工作负载或判定函数。
4. 是否存在 Apply/Build/Correctness/Performance 各自独立的输出状态。
5. 是否有现成的 MockRunner 与可重复的本地单机测试。

## 最小测试矩阵
- Patch 格式冲突：Apply 失败，不进入计时。
- 可应用但构建失败：Build 失败。
- 可构建但功能错误：Correctness 失败，不宣称性能成功。
- 正确但无性能收益：Performance 不通过。
- 正确且显著变快：记录候选及完整测量证据。
- 某轮结果波动过大：不判定“已证实优化”。

## 指标说明
- 明确区分 speedup、runtime reduction 和 SWE-Perf performance metric。
- 不可将 SWE-Perf 的 2.26% 报成“优化成功率”。
- Phase 3 初筛的 30% 与 Phase 4 保守 5% 不要作为我们系统的固定参数；设计为可配置策略。
- 统计检验时保留所有原始测量值，以及清洗后的样本和规则。

## 执行限制
- 优先使用本地 Mock 以及小型项目；不默认提交 Slurm 或长时间多节点实验。
- 在采用论文算法 1 前先查源码实现，验证正确性与失败/超时处理，明确多候选反复检验偏差。
- 交付给研究者：代码位置对应表、测试命令、输出样例、尚存限制和需要人工审批的资源事项。
