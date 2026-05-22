---
title: 持久化 Harness 设计 (详细参考)
tags:
  - reference
  - durability
---

# 💾 持久化 Harness 设计 (详细参考)

在理想的计算机科学理论中，如果一个程序崩溃了，重新启动时它应该能从断点精准恢复，就像什么都没发生过一样。但在 AI Agent 的现实世界里，要实现 `AgentHarness` 的完全持久化与精准恢复，几乎是不可能的。本参考文档探讨了这背后的妥协与设计。

## 🧱 半持久化目标 (Semi-durable Reality)

为什么不能做到 100% 的精准恢复？因为 Agent 的状态不仅仅是一堆可以序列化为 JSON 的字符串。
**Agent 的生命力依赖于外部宿主程序在运行时 (Runtime) 动态注入的 JS 闭包和对象。**

这些**不可序列化**的依赖包括：
- 工具的实现逻辑 (`tool.execute` 函数里面的代码你怎么序列化到数据库里？)
- 大模型提供商的 API 客户端和 Auth 凭证。
- 各种扩展程序和 Hook 处理器。
- 资源加载器。

因此，系统的设计目标妥协为**半持久化 (Semi-durable)**：
1. **核心原则**: `Session` 的追加日志 (Append-only Log) 是**世界上唯一且全部的持久化真相来源**。Harness 绝不会私自去写什么 `harness_state.json` 边车文件。
2. **职责划分**: Harness 负责把自身的状态变化（如队列操作、配置变更）转化为 `Session` 日志项。宿主应用程序负责在重启时，将包含正确代码逻辑的工具、模型和 Hook 重新注入到系统中。
3. **恢复边界**: 恢复操作不是从一次网络请求的中途继续（中断的 TCP 流无法接续），而是从上一个安全的**持久化边界（Durable Boundary）**重新开始。

## 📜 关键的持久化条目

为了让系统能够从灰烬中重生，Harness 需要在 Session 中记录以下关键动作（不仅仅是对话消息）：
- `queue_enqueued` / `queue_consumed`: 转向队列或后续队列的消息进入和被消耗的时刻。
- `pending_write_enqueued` / `applied`: 那些被延迟处理的配置修改指令的生命周期。
- `operation_started` / `finished` / `interrupted`: 宏观操作（如一次分支摘要生成）的状态。
- `turn_started` / `turn_finished`: Agent 一次回合的边界。
- `tool_call_started` / `finished`: 工具调用的边界。

每一次调用这些公共 API，只有当上述这些日志成功落盘后，Promise 才会被 Resolve。

## 🔄 重启与恢复模型 (The Recovery Flow)

当系统崩溃后重新启动，恢复流程如下：

1. **宿主装配**: App 像平时一样启动，注册好它所知道的所有工具、模型和 Hook。
2. **打开黑匣子**: Harness 连接到 `Session` 数据库。
3. **归约演算是宇宙的尽头 (Reduce the Universe)**: 
   - Harness 开始从头到尾重放整个 Session 日志。
   - 这就像 Redux 的 Reducer 一样，通过重放历史，Harness 在内存中重新构建出了崩溃前一秒的配置状态、消息树、以及哪些队列里还有没处理完的任务。
4. **对齐验证**: Harness 检查从历史中算出来的依赖，当前的 App 是否都提供了？（比如，历史记录说需要用 `WeatherTool`，但 App 重启后没有注册这个工具，此时必须抛出警告）。
5. **处理烂摊子 (Reconciliation)**: 处理那些有 `started` 记录，但没有 `finished` 记录的未竟事业。

## 🛡️ 恢复策略：保守即安全

面对崩溃时正在执行的操作，系统的默认策略是**极度保守的**：

- **未完成的 Agent 回合 (Turn)**: 
  - *策略*: 标记为 `interrupted`。退回到闲置 (idle) 状态。**不自动重试**大模型的生成。因为大模型生成是非确定性的，且耗费极高。
- **未完成的大模型网络请求**: 
  - *策略*: 直接作废，标记中断。
- **未完成的工具调用 (Tool Call)**: 
  - *策略*: 追加一条明确的错误日志 `Error: System crashed during execution` 作为该工具的结果。
  - *原因*: 这是最危险的区域！假设这是一个 `delete_database` 的工具，它可能已经执行成功了，只是结果还没来得及写回系统就崩溃了。如果系统自动重试这个工具，后果不堪设想。除非该工具在定义时显式声明了自身是**幂等的 (idempotent / retry-safe)**，否则绝对禁止自动重试。
- **未完成的历史压缩 (Compaction)**: 
  - *策略*: 如果找不到最终的 `compaction_summary` 节点，说明压缩没成功。由于这只是内部数据的整合，不涉及外部破坏，系统可以在空闲时**静默重新运行**压缩算法。

> [!tip] 开发者启示
> 如果你在为 Agent 编写一个涉及外部支付或者资源分配的 Tool，请务必将其设计为“支持幂等重试”的逻辑（例如带有防重 Token），并在工具的文档中明确标出。这是应对混沌物理世界的唯一法则。
