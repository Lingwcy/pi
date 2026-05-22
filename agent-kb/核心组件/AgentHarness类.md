---
title: AgentHarness 类
tags:
  - component
  - implementation
---

# 🏛️ AgentHarness 类

如果说 `AgentHarness.md` (在核心概念中) 描述了系统的设计哲学，那么本文将带你直接深入代码的战壕。`AgentHarness` 是位于 `packages/agent/src/harness/agent-harness.ts` 的一个巨大的门面类 (Facade Class)。它是外部世界与 Agent 核心互动的唯一安全入口。

## 📍 源码位置与概览
`packages/agent/src/harness/agent-harness.ts`

这个类长达近千行代码，但其结构非常清晰。它主要由三部分组成：
1. **状态容器**: 持有所有的活配置 (Live Config) 和排队队列。
2. **公开 API**: 供前端 UI 调用的业务方法（如 `prompt`, `compact`, `setModel`）。
3. **私有编排逻辑**: 连接高级 API 与底层 `AgentLoop` 纯函数的胶水代码。

## 🧠 核心内部状态 (Private State)

为了实现其承诺的并发控制与多层隔离，Harness 的实例维护了极其复杂的内部状态字典：

```typescript
class AgentHarness {
    // ---------------- 基础设施 ----------------
    private env: ExecutionEnv;             // 跨平台的环境接口 (文件系统、Shell执行器)
    private session: Session;              // 对话树的持久化层引用
    private phase: AgentHarnessPhase;      // 状态机指针 (idle, turn, compaction...)
    private handlers: Map<string, Set<Handler>>; // Hook 系统的监听器注册表

    // ---------------- 活配置 (Live Config) ----------------
    private model: Model<any>;             // 当前大模型实例
    private thinkingLevel: ThinkingLevel;  // 思考等级 (off, low, high)
    private streamOptions: StreamOptions;  // 底层网络流配置 (超时, 重试次数)
    private resources: Resources;          // 当前挂载的知识库、技能
    private tools: Map<string, AgentTool>; // 注册工具池
    private activeToolNames: string[];     // 当前被勾选激活的工具名称

    // ---------------- 并发缓冲队列 ----------------
    private pendingSessionWrites: PendingSessionWrite[]; // 忙碌时的挂起修改
    private steerQueue: UserMessage[];     // 转向队列
    private followUpQueue: UserMessage[];  // 后续队列
    private nextTurnQueue: AgentMessage[]; // 强制预备队列

    // ---------------- 运行态控制 ----------------
    private runAbortController?: AbortController; // 供 abort() 随时切断底层网络的开关
    private runPromise?: Promise<void>;    // 用于 waitForIdle() 的锁，确保外部可以等待 Agent 彻底停机
}
```

## 🔍 核心工作流：一场史诗级的 `executeTurn` 源码级拆解

当 UI 调用 `harness.prompt("帮我重构这段代码")` 时，系统到底发生了什么？真正的魔法发生在私有的 `executeTurn` 函数中。

### 阶段 1：组装与拦截
首先，Harness 需要将用户的输入转化为底层的结构，并允许插件插手。
1. **清空预备队列**: 如果 `nextTurnQueue` 里有消息（例如：系统强制插入了一段先决条件），它们会被取出来，并**放置在用户的 prompt 之前**。这保证了即使用户发起了请求，系统级的隐式指令也总是拥有更高的优先级。
2. **触发 `before_agent_start` 钩子**: 
   ```typescript
   // 插件可以在这里最后一次修改 systemPrompt，或者在 messages 前后塞入隐藏的上下文。
   const beforeResult = await this.emitHook({
       type: "before_agent_start",
       prompt: text,
       systemPrompt: turnState.systemPrompt,
       ...
   });
   ```

### 阶段 2：准备隔离沙箱
为了隔离外部环境，Harness 必须为底层的 `AgentLoop` 准备一个绝对安全的执行沙箱。
1. **接管中断权**: 初始化一个全新的 `AbortController` 并赋值给 `this.runAbortController`。从这一刻起，用户点击 UI 的停止按钮，就能精准切断即将发生的网络请求。
2. **构建闭包锁**: 创建 `getTurnState` 闭包函数。无论底层的 `AgentLoop` 循环跑了多久，它获取配置的唯一途径就是调用这个函数，而这个函数只会返回一开始被“快照”的 `TurnSnapshot`。这就彻底免疫了用户中途切换大模型的干扰。

### 阶段 3：映射回调与点火
由于 `AgentLoop` 只是一个纯函数库，它不懂什么是 Hook。Harness 使用了高超的设计模式——**适配器模式 (Adapter)**，将自身的 Hook 系统伪装成 `AgentLoopConfig` 需要的回调函数。

```typescript
// 节选自 createLoopConfig()
const config: AgentLoopConfig = {
    model: snapshot.model, // 使用隔离快照中的模型
    
    // 将底层过滤历史的需求，映射给上层的 "context" 钩子
    transformContext: async (messages) => {
        const result = await this.emitHook({ type: "context", messages });
        return result?.messages ?? messages;
    },

    // 将底层拦截工具的需求，映射给上层的 "tool_call" 钩子
    beforeToolCall: async ({ toolCall, args }) => {
        const result = await this.emitHook({ type: "tool_call", ... });
        return result ? { block: result.block, reason: result.reason } : undefined;
    },
    
    // ... 将 afterToolCall 映射给 tool_result 钩子
};
```

装填完毕后，Harness 调用 `runAgentLoop(messages, config, ...)`。此时，程序的控制流被移交到了黑盒引擎之中。

### 阶段 4：失败料理与善后 (`finally` 块)
无论是模型成功给出了完美的答案，还是因为网络原因引发了超时崩溃，或者用户主动点击了停止，代码最终都会落入外层的 `finally` 块。
**这是整个系统的定海神针。**

```typescript
} finally {
    try {
        // 1. 将大模型思考这几十秒期间，用户在 UI 上点击切换模型的动作（在 Pending 队列里），一次性安全地写入数据库。
        await this.flushPendingSessionWrites(); 
    } finally {
        // 2. 销毁打断控制器，防止内存泄漏
        this.runAbortController = undefined;
        // 3. 将系统的外层并发锁解除，允许下一次对话或历史压缩操作
        this.phase = "idle";
    }
}
```

## 🛠️ 其他关键能力简述

### 并发写入缓冲 (`appendMessage` 与 `setModel`)
如果你试图调用 `harness.setModel()`：
- 检查 `this.phase === "idle"`。
- **Yes (闲置状态)**: 爽快地直接调用 `this.session.appendModelChange()`，立刻落盘。
- **No (忙碌状态)**: 绝对不能写库！把指令包装成对象 `{ type: "model_change", provider, modelId }`，默默塞进 `this.pendingSessionWrites` 数组里，留给上面提到的 `finally` 块去处理。

### 会话操作代理
诸如 `harness.compact()`（历史压缩）或 `harness.navigateTree()`（树状分支穿梭）等宏大的历史操作。
它们在被调用时，第一件事就是将 `phase` 改为 `compaction` 或 `branch_summary`。这同样是一把互斥锁。在这个状态下，任何试图发送新对话的操作都会被瞬间弹回，保证了在对历史记录动大手术时，没有人能在这个脆弱的树上长出新叶子。

---

> [!tip] 阅读源码的最佳切入点
> 不要试图从头到尾读这 1000 行代码。
> 1. 先看 `constructor`，了解它有哪些工具和依赖。
> 2. 跳到 `createLoopConfig`，看它是怎么把自己装扮成底层的配置对象的。
> 3. 精读 `executeTurn` 的 `try...finally` 结构，体会其坚如磐石的防御性编程风格。
