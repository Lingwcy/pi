---
title: Agent Loop 函数
tags:
  - component
  - implementation
---

# ⚙️ Agent Loop 函数 (深度实现剖析)

如果说 `AgentHarness` 是处理复杂现实业务的封装层，那么位于 `packages/agent/src/agent-loop.ts` 的 `agentLoop` 和 `runLoop` 就是纯粹的算法核心。它们剥离了持久化、剥离了队列概念，专门负责：**消息格式转换、流式网络解析、以及工具的执行调度**。

## 📍 源码位置与架构边界
`packages/agent/src/agent-loop.ts`

> [!warning] 隔离性宣言
> 此文件中的代码**绝对禁止**导入任何与 Node.js 绑定的模块（如 `fs`, `child_process`），也**禁止**持有对 `AgentHarness` 或 `Session` 的引用。所有的副作用都必须通过 `AgentLoopConfig` 参数的闭包注入进来。这保证了 `AgentLoop` 甚至可以在 Service Worker 中运行。

## 🔄 `runLoop`：不屈不挠的双层自动机

`runLoop` 是一个经典的 `while (true)` 结构，它支撑起了 Agent “思考-行动-再思考”的自动机行为。

### 外层循环：生命周期的呼吸
外层循环代表了 Agent 的一次“长工作周期”。
通常情况下，当大模型没有返回新的 `ToolCall` 时，它就应该结束了。但在复杂的场景中，用户可能通过 Harness 发送了“后续（Follow-up）”消息（比如：“你刚才写完的函数，麻烦再给我生成一个 UML 图”）。
外层循环在 Agent 即将退出时，会检查这个 `getFollowUpMessages` 回调。如果有消息，外层循环会像起搏器一样，再次唤醒内层循环，强迫 Agent 继续工作。

### 内层循环：工具使用的自主流转
这是 Agent 产生智能错觉的核心。
```typescript
while (hasMoreToolCalls || pendingMessages.length > 0) {
    // 1. 调用大模型，这可能要花十几秒
    const message = await streamAssistantResponse(...);
    
    // 2. 扫描返回的消息里有没有工具调用
    const toolCalls = message.content.filter(c => c.type === "toolCall");
    
    if (toolCalls.length > 0) {
        // 3. 执行这些工具（读文件、跑脚本等）
        const executedToolBatch = await executeToolCalls(...);
        
        // 4. 重置开关，让循环继续！这就是为什么 Agent 会连着调用七八次工具的原因
        hasMoreToolCalls = !executedToolBatch.terminate; 
    } else {
        // 5. 大模型说“我干完了”，开关关闭，跳出内层循环
        hasMoreToolCalls = false;
    }
}
```

## 🌊 流式响应解析器 (`streamAssistantResponse`)

这个函数是系统中最琐碎、但也最关键的协议转换器。
大模型厂商（如 Anthropic 或 OpenAI）并不会一次性返回一个完美的 JSON 对象。它们是通过 Server-Sent Events (SSE) 一块一块吐出碎片的。

`streamAssistantResponse` 作为一个状态机，监听底层的 `streamSimple` 返回的事件流：

1. **协议拦截**: 调用 `config.convertToLlm`，将系统内部的各种奇奇怪怪的消息（比如 UI 的提示框消息），安全地剥离或翻译成纯净的、大模型认识的标准格式。
2. **骨架生成 (`start`)**: 收到连接成功的事件时，在内存里创建一个空的 `AssistantMessage` 骨架，并向外层发出 `message_start` 事件。
3. **血肉填充 (`text_delta` 等)**: 当收到字符串碎片时，直接修改骨架对象的内存。
    ```typescript
    case "text_delta":
        // 骨架里的内容在实时增长
        partialMessage = event.partial; 
        context.messages[lastIndex] = partialMessage;
        // 通知 UI 层：“字符串长长了，更新打字机渲染吧”
        await emit({ type: "message_update", message: { ...partialMessage } });
        break;
    ```
4. **收尾 (`done`)**: 流结束，发出 `message_end` 事件。

## 🛠️ 工具并发引擎：`executeToolCalls` 的艺术

当大模型一次性返回了 3 个工具调用（例如：同时 `read_file("a.txt")`, `read_file("b.txt")`, `read_file("c.txt")`），如果串行执行，等待时间将是 3 倍。`executeToolCallsParallel` 函数展示了极高的并发设计技巧。

它将工具执行严格拆分为两个截然不同的阶段：

### 阶段一：串行鉴权准备 (Sequential Preparation)
即使开启了并行，工具的**准备和权限校验依然是串行的**。
*   **原因**：安全插件（配置在 `config.beforeToolCall` 中）可能会根据前一个工具的调用状态，来决定是否允许下一个工具的执行。如果并发鉴权，很容易产生安全漏洞。
*   **执行**：逐个遍历 Tool Call，通过 JSON Schema 验证参数，然后送入 Hook 系统拦截。如果有 Hook 拒绝，这个工具的状态直接被标记为 `Immediate Error`。

### 阶段二：并发爆发与有序回收 (Concurrent Burst & Ordered Finalization)
```typescript
// 1. 并发爆发：将所有合法的工具，打包成并行运行的 Promise 任务群
finalizedCalls.push(async () => {
    // 真实地去执行底层的代码（如访问文件系统）
    const executed = await executePreparedToolCall(preparation);
    // 允许后置插件（afterToolCall）去修饰或脱敏返回的字符串结果
    const finalized = await finalizeExecutedToolCall(..., executed, ...);
    return finalized;
});

// 2. 等待群完成，但保持索引对齐
// Promise.all 会等待所有人跑完。但返回数组的顺序，绝对等同于工具最初被放入 finalizedCalls 的顺序！
const orderedFinalizedCalls = await Promise.all(finalizedCalls.map(fn => fn()));

// 3. 有序回收：按照大模型吐出工具调用的原始顺序，依序发出事件和入库。
for (const finalized of orderedFinalizedCalls) {
    const toolResultMessage = createToolResultMessage(finalized);
    await emitToolResultMessage(toolResultMessage, emit);
}
```

> [!important] 为什么“有序回收”如此重要？
> 假设并发执行时，文件 C 极小，先读完；文件 A 极大，最后读完。如果不进行重排序，发送给 LLM 的上下文顺序将是 `[Result C, Result B, Result A]`。这种非确定性的乱序会导致 LLM 的自回归推理产生灾难性的“蝴蝶效应”，使得两次相同提示词的运行轨迹完全不同，根本无法复现和调试 Bug。因此，并发执行提升了时间效率，而“有序回收”保证了物理空间上的决定论。
