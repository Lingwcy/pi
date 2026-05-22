---
title: Agent Loop (执行循环)
tags:
  - core-concept
  - execution
---

# ⚙️ Agent Loop (执行循环)

如果 `AgentHarness` 是汽车的车身和控制面板，那么 `Agent Loop`（执行循环）就是这辆汽车内部那台轰鸣的 V8 发动机。它是系统中最底层的自主执行逻辑，完全隐藏在 Harness 的封装之下。它不关心用户的 UI 长什么样，不关心持久化数据库用的是什么，它的唯一职责是：**与大语言模型 (LLM) 进行直接交互、处理复杂的流式输出、并在虚拟环境中执行工具，直到任务完成。**

## 🔄 核心工作流：双层状态机

Agent Loop 的本质是一个精密的**双层状态机循环** (`runLoop` 函数)。它需要处理两类截然不同的“连续性”问题。

```mermaid
flowchart TD
    Start((启动 runLoop)) --> OuterLoop{外部循环: 任务调度}
    
    OuterLoop --> InnerLoop{内部循环: 自主思考与执行}
    
    InnerLoop --> InjectSteer[检查并注入 Steer 转向消息]
    InjectSteer --> LLMRequest[调用大模型厂商 API\n启动流式接收]
    LLMRequest --> ParseStream[状态机消费流:\n发出 message_update 等事件]
    
    ParseStream --> HasTools{大模型是否要求调用工具?}
    
    HasTools -->|Yes| ExecTools[工具并发引擎\n验证 -> 执行 -> 汇总]
    ExecTools --> TurnEnd[发出 turn_end 事件\n(触发 Harness 保存点)]
    TurnEnd --> InnerLoop
    
    HasTools -->|No| TurnEndNoTool[发出 turn_end 事件\n(触发 Harness 保存点)]
    TurnEndNoTool --> CheckStop{达到全局终止条件?}
    
    CheckStop -->|Yes| Exit((退出 agent_end))
    CheckStop -->|No| OuterLoopCheck{外部排队: 有 Follow-up 消息?}
    
    OuterLoopCheck -->|有消息| InjectFollow[转移消息到待处理区]
    InjectFollow --> InnerLoop
    OuterLoopCheck -->|无消息| Exit

    style InnerLoop fill:#f9f,stroke:#333,stroke-width:2px
    style OuterLoop fill:#ccf,stroke:#333,stroke-width:2px
```

### 1. 内部循环 (Inner Loop)：Agent 的自主性
内部循环解决的是 **“模型自身发起的延续”**。
当模型分析了一个问题后，它可能决定调用 `read_file` 工具。收到文件内容后，它可能觉得还需要调用 `grep_search` 工具，然后再得出结论。
只要模型返回的结果中包含 `ToolCall`，内部循环就会不断地运转下去：`请求 -> 执行工具 -> 将工具结果追加到历史 -> 再次请求`。这就是 Agent 所谓“自主性”的来源。

### 2. 外部循环 (Outer Loop)：环境发起的延续
外部循环解决的是 **“环境强加的延续”**。
当模型说：“我已经解答完你的问题了”，它没有调用任何工具，内部循环本该结束。此时，外部循环接管，它会去询问配置（也就是 Harness 里的队列）：“在模型刚才思考的这几十秒里，用户有没有提交新的 `Follow-up`（后续）任务？”如果有，外部循环会将新任务喂给内部循环，迫使 Agent 重新开始工作，而无需切断底层的流。

## 🌊 流式响应状态机 (`streamAssistantResponse`)

在 `AgentLoop` 中，最复杂的函数莫过于 `streamAssistantResponse`。它负责将我们内部的 `AgentMessage` 对象“翻译”为各大厂商（如 Anthropic, OpenAI）的格式，并处理极其琐碎的流数据 (Streaming Data)。

> [!info] 为什么流式处理这么复杂？
> 因为大模型并不是一次性吐出完整的 JSON 工具调用请求。它是一个字符一个字符吐出的（如 `{"na` ... `me": "re` ... `ad_file"}`）。底层库（`@earendil-works/pi-ai`）会帮我们拼接这些碎片，而 `AgentLoop` 的职责是监听这些碎片的事件。

它的状态机逻辑如下：
1. **启动 (`start`)**: 收到第一批字节。此时 `AgentLoop` 会在内存中创建一个空的 `AssistantMessage` 骨架，并向外发出 `message_start` 事件。前端 UI 收到此事件后，通常会渲染一个空白的对话气泡。
2. **文本增量 (`text_delta` / `thinking_delta`)**: 收到新的文本片段。`AgentLoop` 会将片段拼接到刚才的骨架中，并发出 `message_update` 事件。UI 收到此事件，实现“打字机”效果。
3. **工具增量 (`toolcall_delta`)**: 同上，但拼装的是工具的参数。UI 通常用它来显示进度条或实时的 JSON 树。
4. **完成 (`done` / `error`)**: 流结束。`AgentLoop` 固化最终的 `AssistantMessage`，发出 `message_end`。

## 🛠️ 工具并发引擎：性能与安全的平衡

模型有时非常聪明，它会一次性返回 5 个不同的工具调用请求（比如同时读取 5 个不同的文件）。`AgentLoop` 中的 `executeToolCalls` 函数就是专门处理这种情况的。系统必须在“执行速度（并发）”和“数据安全（串行）”之间取得平衡。

### 策略 A: 顺序执行 (Sequential) - 安全的退避策略
*   **触发条件**：如果全系统的默认配置为顺序执行，**或者**这 5 个工具中哪怕只有 1 个工具在定义时标记了 `executionMode: "sequential"`，整个批次就会降级为顺序执行。
*   **执行方式**：排成一队。工具 A 的 `参数验证 -> Hook拦截检查 -> 实际运行 -> 结果修补` 完整走完后，才会开始处理工具 B。
*   **适用场景**：操作共享资源的敏感工具。例如：需要独占操作终端的 Bash 命令，或者会修改同一个配置文件的并发工具。

### 策略 B: 并行执行 (Parallel) - 高性能的默认模式
为了极大地缩短 Agent 的响应时间，并行模式采用了一种非常巧妙的 **“串行鉴权，并发执行，有序回收”** 算法：

1. **串行准备 (Sequential Preparation)**: 虽然执行是并行的，但所有工具触发的 `beforeToolCall` 钩子必须串行跑完。
   *   *原因*：防范竞态条件。如果安全插件需要根据上下文判断是否允许执行，并发修改上下文会导致安全漏洞。如果某个 Hook 拒绝了某个工具（返回 `block: true`），该工具会立即被标记为产生了一个伪造的错误结果，不会真正执行。
2. **并发爆发 (Concurrent Burst)**: 准备和鉴权完毕后，所有允许执行的工具被包装进 Promise 中，通过 `Promise.all` 同时射出（执行底层的 `tool.execute()`）。
3. **有序回收 (Ordered Finalization)**: 这是最核心的设计。无论这 5 个并发工具哪个先跑完，最终发送出的 `message_start` (针对 ToolResult) 和最终追加到会话历史中的顺序，**绝对严格按照模型最初吐出这 5 个工具请求的顺序排列**。
   *   *原因*：保证日志的决定论 (Determinism)。如果不强制排序，每次并发执行写入数据库的顺序都不一样，会导致 LLM 下一轮基于历史上下文的推理产生不可预测的蝴蝶效应。

---

> [!warning] 隔离与依赖注入 (Dependency Injection)
> 请注意，`AgentLoop` 本身是一个极其纯粹的函数库。它完全不认识 `Session`，不知道什么是“历史压缩”，也无法直接读取本地文件系统。
> 所有的扩展能力，都是通过 `AgentLoopConfig` 这个参数“注入”进来的：
> - `config.convertToLlm` 决定了如何将内部消息翻译为特定厂商格式。
> - `config.beforeToolCall` 注入了安全和干预逻辑。
> - `config.prepareNextTurn` 才是真正触发 Harness 将数据持久化到硬盘的钩子。
>
> 这种高度的解耦保证了 `AgentLoop` 甚至可以在没有 Node.js 环境的纯浏览器前端中独立运行。
