# D01｜OpenHands Software Agent SDK

> 状态：L1｜2026-11-03｜尚未本地运行示例  
> 论文：https://arxiv.org/abs/2511.03690  
> 代码：https://github.com/OpenHands/software-agent-sdk  
> 官方文档：https://docs.openhands.dev/sdk

## 它解决的是什么问题
OpenHands SDK 提供构建 Coding Agent 的基础设施：LLM、Agent、Tool、Conversation、Workspace、Events 与远程 Agent Server。它**不是性能优化算法本身**，能用来承载我们实现的 Profiler、构建、测试与 Patch 优化工具。

## 重点架构边界
- `Agent`：选择下一动作、调用工具；
- `Tool`：把文件编辑、命令执行或专用能力暴露给模型；
- `Conversation`：Agent 运行上下文与工具反馈；
- `Workspace`：代码所在位置与操作环境，可本地或隔离工作区；
- `Agent Server`：远程管理运行会话。

## 推荐查看
- https://github.com/OpenHands/software-agent-sdk/tree/main/examples/01_standalone_sdk
- https://github.com/OpenHands/software-agent-sdk/tree/main/openhands-sdk
- https://github.com/OpenHands/software-agent-sdk/tree/main/openhands-tools
- https://github.com/OpenHands/software-agent-sdk/tree/main/openhands-workspace

## 需要 Codex 验证
1. 当前 FinePerf/Benchmark 框架是否已经自带 Agent 调度机制？
2. 引入 SDK 比最小 ReAct + 沙箱多出哪些依赖和运行成本？
3. 自定义 `profile`, `benchmark`, `correctness_gate` 是否能作为强类型工具？
4. 执行命令的权限和回滚边界是什么？
5. 集群权限是否仍通过可信的独立执行器管理？

结论：作为工程底座候选，不预设一定采用。
