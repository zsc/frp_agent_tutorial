# Chapter 3｜核心运行时：Agent 作为“可组合的反应系统”

## 3.1 开篇：Runtime 的职责边界——从“脚本”到“引擎”

在简单的 Demo 中，Agent 可能只是一个 `while(true)` 循环里的一堆 Python 函数调用。但在生产级的实时系统中，这种写法会迅速崩溃：它无法优雅地处理用户打断、网络超时、并发工具调用或 Token 预算超标。

我们需要一个 **FRP Runtime（运行时）**。

如果把 Agent 的 Prompt 和策略逻辑比作**游戏脚本**，那么 Runtime 就是**游戏引擎**。
游戏引擎负责物理碰撞、渲染循环、输入映射；同理，Agent Runtime 负责将混沌的现实世界抽象为有序的流。

**Runtime 的核职责矩阵**：

| 领域 | 职责描述 | 典型问题示例 |
| :--- | :--- | :--- |
| **I/O 归一化** | 将 HTTP/WebSocket/Timer 统一为 Event | "用户说话"和"定时器到期"应该是一回事吗？ |
| **时空管理** | 维护逻辑时钟与物理时钟的映射 | "现在"是几点？重放昨天的数据时，"现在"是昨天。 |
| **状态护栏** | 保证 State 的原子性与一致性 | 两个工具同时返回结果，谁先谁后？谁覆盖谁？ |
| **副作用托管** | 隔离并调度所有外部调用 (Effect) | 用户这句不说了，后台正在跑的 SQL 查询要杀掉吗？ |
| **可观测性** | 提供“上帝视角”的 Trace 上下文 | 这一步推理为什么这么慢？是模型慢还是网络卡？ |

本章将详细解构这个引擎的内部构造。

---

## 3.2 “一切皆事件”：归一化输入与事件总线

在 FRP 架构中，**没有“调用”，只有“事件”**。
传统的 Request/Response 模型是同步阻塞的，而 Event Stream 模型异步非阻塞的。

### 3.2.1 通用事件信封 (The Universal Envelope)

为了让 Runtime 能处理任何类型的输入，我们需要一个标准化的“信封”。所有进入系统的数据包，必须先被封装。

```text
Structure: StandardEvent <T>
-----------------------------------------------------------------------
| Field          | Description                                        |
-----------------------------------------------------------------------
| id             | UUID v4 (唯一标识)                                 |
| sequence_id    | 逻辑时钟序号 (单调递增，用于排序与去重)               |
| type           | 事件类型枚举 (e.g. USER_INPUT, SYSTEM_TICK, TOOL_OK) |
| timestamp      | 物理生成时间 (ISO8601)                               |
| payload        | 泛型数据 T (实际内容)                                |
| trace_context  | 分布式追踪 ID (TraceID, SpanID, ParentID)            |
| source         | 事件来源 (UI, Timer, Worker-A)                       |
-----------------------------------------------------------------------
```

### 3.2.2 事件分类学

Runtime 内部处理的事件远不止“用户输入”和“模型输出”。我们需要处理四大类事件流：

1.  **交互流 (Interaction Stream)**:
    *   `UserAudioChunkEvent`: 原始音频帧（高频）。
    *   `UserTextEvent`: ASR 转录后的文本或打字内容。
    *   `UserIntentEvent`: UI 交互，如点击“暂停”、“重新生成”。

2.  **系统流 (System Stream)**:
    *   `TickEvent`: 心跳，携带 `delta_time`，驱动倒计时和超时逻辑。
    *   `LifeCycleEvent`: `START`, `SHUTDOWN`, `SLEEP`。
    *   `ConnectivityEvent`: 网络状态变化，WebSocket 断开/重连。

3.  **模型与工具流 (Computation Stream)**:
    *   `ModelTokenEvent`: LLM 吐出的流式 Token。
    *   `ToolReceiptEvent`: 工具执行完成的回执（含结果或错误栈）。
    *   `BudgetAlarmEvent`: Token 预算耗尽告警。

4.  **合成流(Synthetic Stream)**:
    *   `VADTriggerEvent`: 语音活动检测触发（用户开始说话/停止说话）。
    *   `SilenceTimeoutEvent`: 用户沉默超时（Bot 主动搭话）。

**架构图：事件漏斗**

```text
[WebSocket] --\
[Timer API] ----\
[HTTP Hook] ------+--> [Ingress Adapters] --> [Event Normalizer]
[Microphone] ---/            (Raw Data)       (Add Timestamp/ID)
                                                      |
                                                      v
                                               [Main Event Bus]
                                              (The Central Stream)
```

---

## 3.3 AgentState 设计：状态树与不可变性

状态（State）是 Agent 的“记忆”。在 FRP 中，State 是 Stream 经过 `scan` (accumulate) 算子产生的 **Signal**。

### 3.3.1 状态树结构

不要把所有东西塞进一个扁平的字典。推荐使用**组合式状态树**：

```text
AgentState (Root)
├── Meta
│   ├── SessionID
│   ├── StartTime
│   └── LastActiveTime
├── Environment (环境感知)
│   ├── SystemTime
│   ├── UserLocation
│   └── DeviceStatus
├── Conversation (核心会话)
│   ├── History (消息列表)
│   ├── CurrentTurn (当前轮次草稿)
│   └── ContextSummary (长记忆摘要)
├── RuntimeStatus (控制面)
│   ├── Stage (Idle / Listening / Thinking / Speaking)
│   ├── PendingEffects (正在执行的副作用列表)
│   └── TokenUsage (累计消耗)
└── Buffer (缓冲区)
    └── PartialInput (尚未提交的用户输入片段)
```

### 3.3.2 为什么必须是不可变 (Immutable)？

在 JavaScript/TypeScript 中可能使用 Immer.js，在 Python 中使用 Pydantic 的 `frozen=True` 或 `dataclasses(frozen=True)`。

1.  **比较（Diffing）效率**：React 式的 UI 更新依赖于引用比较 (`oldState !== newState`)。如果是不可变对象，判断是否需要重绘 UI 极其快
2.  **并发安全**：由于旧状态不会被修改，正在读取旧状态用于渲染或日志的线程不会发生数据竞争。
3.  **历史回溯**：保存 `State` 对象的引用列表，即自动获得了“撤销/重做”功能。

---

## 3.4 Reducer 规范：纯函数核心

Reducer 是 Runtime 的心脏。它定义了：**当发生 X 时，如果当前处于 Y 状态，那么变成 Z 状态，并计划执行副作用 E。**

### 3.4.1 函数签名

$$ (CurrentState, Event) \Rightarrow (NewState, EffectDescriptor[]) $$

注意返回值包含两部分：
1.  **NewState**: 立即生效的内存变更。
2.  **EffectDescriptor[]**: **描述**需要做什么，而不是直接做。

### 3.4.2 案例：状态机逻辑

让我们看一个具体的 Reducer 逻辑片段（伪代码逻辑）：

```text
Function MainReducer(state, event):
    
    // 场景：用户正在说话，打断了机器人的思考
    if state.Stage == "Thinking" AND event.Type == "VAD_USER_START_TALKING":
        return (
            state.copy(
                Stage = "Listening",
                Buffer.append(event.payload)
            ),
            [ 
              Cmd.CancelCurrentLLM(),  // 这是一个描述对象，不是函数调用
              Cmd.UI.ShowStatus("Listening...") 
            ]
        )

    // 场景：工具调用成功返回
    if state.Stage == "ToolUse" AND event.Type == "TOOL_SUCCESS":
        return (
            state.copy(
                Stage = "Synthesizing",
                Conversation.History.add(ToolMessage(event.data))
            ),
            [ Cmd.RunLLM(prompt=build_prompt(state)) ]
        )

    return (state, []) // 默认无变化
```

**关键原则**：Reducer 内部**零副作用**。禁止 `print`，禁止 `requests.get`，禁止 `time.sleep`。它必须是数学意义上的纯函数，这保证了系统的**确定性（Determinism）**。

---

## 3.5 Effect 执行器：副作用的“隔离病房”

Reducer 吐出的 `EffectDescriptor` 只是数据的“处方”。Runtime 中的 **Effect Orchestrator** 负责按处方抓药。

### 3.5.1 Effect 描述符结构

```text
EffectDescriptor
----------------
| id        | 关联的 Trigger Event ID (用于溯源)
| type      | HTTP / DB / LLM / TIMER / UI
| payload   | 请求参数 (URL, SQL, Prompt)
| policy    | 执行策略 (RetryCount, Timeout, Priority)
| key       | 幂等键 (用于去重)
```

### 3.5.2 执行与生命周期管理

Effect Orchestrator 监听 Reducer 的输出流。当收到 Effect 列表时：

1.  **Diff & Schedule**: 比较当前正在运行的 Effect 和新收到的 Effect。
    *   *New*: 启动新的 Executor。
    *   *Obsolete*: 如果新列表中没有旧的 Effect ID，说明该任务被取消了 -> 发送 Abort 信号。
    *   *Same*: 保持运行，不做操作。

2.  **Execution Isolation**: 每个 Effect 在独立的异步任务（Promise/Future/Task）中运行。

3.  **Feedback Loop**:
    *   任务成功 -> 发出 `EffectResolvedEvent`。
    *   任务失败 -> 发出 `EffectFailedEvent`。
    *   这些 Event 会再次进入 Event Bus，触发下一次 Reducer 循环。

### 3.5.3 复杂的并发控制：Switch vs Merge

Runtime 需要根据 Effect 类型选择不同的 FRP 策略：

*   **SwitchStrategy (最新优先)**: 适用于 LLM 推理。如果用户改口了，旧的 LLM 请求应该立即作废。
    *   *FRP 算子*: `switchLatest`
*   **MergeStrategy (并发执行)**: 适用于并行搜索。同时查 Google 和 Wikipedia，谁先回来无所谓，都要。
    *   *FRP 算子*: `merge`
*   **QueueStrategy (串行执行)**: 适用于写数据库日志。必须按顺序写，不能并发。
    *   *FRP 算子*: `concat`

---

## 3.6 Checkpoint 与 Snapshot：从崩溃中重生

在实时 Agent 中，Session 可能会持续很久（数小时），Event Log 会变得巨大。我们不能每次恢复都从第 1 个事件开始重放。

### 3.6.1 混合持久化策略

Runtime 需要一个后台进程，执行以下逻辑：

1.  **WAL (Write-Ahead Log)**: 每进入 Bus 的 Event 立即追加写入磁盘/DB。这是“真理来源”。
2.  **Periodic Snapshot**:
    *   规则：每处理 100 个事件 或 每 5 分钟。
    *   动作：将当前 `AgentState` 序列化为 JSON/Binary，存入 Object Store，标记 Version N。
    *   同时：截断（Rotate）旧的 Log，只保留 Version N 之后的 Log。

### 3.6.2 恢复流程 (Hydration)

当服务重启或发生 Pod 漂移时：
1.  从 DB 读取最新的 Snapshot (Version N)。
2.  反序列化为 `AgentState` 对象。
3.  读取 Version N 之后的所有 WAL Events。
4.  运行 `Reducer(State, Event)` 循环（**注意：此时禁用 Effect 执行器**），快速“快进”到崩溃前的状态。
5.  恢复正常模式，启用 Effect 执行器。

---

## 3.7 多 Agent 与层级 (Supervisor Tree)

复杂的 Agent 不是单体，而是**分形（Fractal）**的。

*   **主控 (Main Agent)**: 负责路由和全局上下文。
*   **专家 (Sub-Agent)**: 负责特定领域（如写代码、机票）。

### 3.7.1 Stream of Streams

在 Runtime 中，Sub-Agent 本身可以被视为一个 **Long-running Effect**。

1.  Main Reducer 发出 `Effect: SpawnSubAgent("Coder")`。
2.  Runtime 启动一个新的 Stream 实例（子运行时）。
3.  **管道连接 (Piping)**:
    *   Main Agent 将部分 Event (如用户的特定指令) 转发给 Sub-Agent。
    *   Sub-Agent 的输出 Event (如代码结果) 被包装成 Main Agent 的输入 Event。

这种结构类似于 Erlang 的 Supervisor Tree：如果 Sub-Agent 崩溃，Main Agent 可以收到 `ChildCrashEvent`，并决定是重启它还是向用户道歉。

---

## 3.8 插件化架构：Runtime 扩展点

为了保持 Runtime 核心代码的通用性，所有具体业务能力都应通过插件注入。

**插件接口定义 (Protocol)**:

1.  **ToolPlugin**:
    *   `manifest`: JSON Schema，用于注入 Prompt。
    *   `handler`: `(payload) => Promise<Result>`，实际执行逻辑。
    *   `cleanup`: 资源回收钩子。

2.  **MemoryPlugin**:
    *   `retrieve(state, query)`: 如何根据当前 State 查找向量库。
    *   `observe(event)`: 哪些 Event 值得被存入长期记忆。

3.  **PolicyPlugin** (高级):
    *   允许替换默认的 Reducer 逻辑。例如，在此插件中实现 ReAct 循环或 COT（思维链）逻辑，而不修改核心 Runtime。

---

## 3.9 本章小结

1.  **Runtime 是引擎**：它负责时间、并发、IO 和状态管理，让业务逻辑保持纯粹。
2.  **Reducer 模式**：$(State, Event) \rightarrow (NewState, Effect[])$ 是核心循环。
3.  **Effect 是描述符**：把“想做某事”和“真正做某事”分开，实现了可测试和可回放。
4.  **状态树**：结构化的、不可变的状态树是系统的单一事实来源（Single Source of Truth）。
5.  **韧性**：通过 WAL + Snapshot 实现崩溃恢复；通过 Supervisor Tree 实现错误隔离。

---

## 3.10 练习题

### 基础题

**练习 1：Effect vs Event 辨析**
在 Runtime 日志中看到了下条目，请标记它是 Event (E), State Change (S), 还是 Effect Descriptor (D):
1.  `Payload: { "text": "Hello world" }, Source: User`
2.  `Action: HTTP_POST, URL: api.weather.com`
3.  `CurrentStage changed from "Idle" to "Fetching"`
4.  `Payload: { "temp": 25 }, Source: WeatherAPI`

<details>
<summary>参考答案</summary>

1.  **Event (E)**: 外部输入的事实。
2.  **Effect Descriptor (D)**: Reducer 发出的意图，尚未执行。
3.  **State Change (S)**: Reducer 计算后的结果。
4.  **Event (E)**: 外部 API 返回的结果被封装成了事件。
</details>

**练习 2：纯函数检查**
以下 Reducer 代码有什么问题？请指出至少两处违反 Runtime 规范的地方。
```python
def bad_reducer(state, event):
    if event.type == "ASK_TIME":
        current_time = datetime.now() # 1
        print(f"User asked for time: {current_time}") # 2
        db.log_access(event.user_id) # 3
        return state.update(response=str(current_time)), []
```

<details>
<summary>参考答案</summary>

1.  **隐式时间依赖**: `datetime.now()` 使函数变得不确定。在重放时，时间会变成重放时的时间，而不是事件发生时的时间。应使用 `event.timestamp` 或从 `state.env.system_time` 获取。
2.  **IO 副作用 (Print)**: 不应在 reducer 中打印，应通过日志中间件观察流。
3.  **IO 副作用 (DB)**: `db.log_access` 是写数据库操作，必须封装为 `Effect` 返回，不能直接调用。
</details>

### 挑战题

**练习 3：设计“优雅打断”机制**
场景：LLM 正在生成长文本（Stream Token Event 持续涌入），此时用户说了一句“算了，换个话题”。
请描述 Runtime 内部各组件如何协同工作，实现立即停止生成并切换状态。

<details>
<summary>参考答案 (提示)</summary>

1.  **Input**: Ingress 收到音频，VAD 触发 `UserInterruptionEvent`。
2.  **Reducer**:
    *   收到 Interruption Event。
    *   计算 NewState: 将 `Stage` 从 `Speaking` 设为 `Listening`；空 `PendingOutputBuffer`。
    *   生成 Effects: `[ Cmd.CancelTask(target="LLM_Generation"), Cmd.AudioPlayer.Stop() ]`。
3.  **Effect Orchestrator**:
    *   收到 `CancelTask` 指令。
    *   查找正在运行的 LLM 调用任务，调用其 `AbortController.abort()`。
    *   LLM 连接断开，停止产生新的 Token Event。
4.  **State Consistency**: 即使还有几个在网络上“飞”的 Token Event 随后到达，由于 State 已经变更为 `Listening`，Reducer 会根据当前状态丢弃这些迟到的 Token。
</details>

**练习 4：实现“乐观更新 (Optimistic UI)”**
用户发送消息后，为了体验流畅，我们希望 UI 立刻显示消息已上屏，而不是等服务器确认。但在 FRP Runtime 中，State 必须由 Event 驱动。如何设计 Event 序列来支持这种体验，同时又能处理发送失败的情况？

<details>
<summary>参考答案 (提示)</summary>

设计两个 Event：
1.  **`UserSubmitEvent` (Intent)**: 用户按下回车。
    *   Reducer 处理：将消息加入 `state.conversation`，但标记状态为 `pending`（灰色）。
    *   Effect：发起网络请求发送消息。
2.  **`MessageAckEvent` (Fact)**: 服务器确认收到。
    *   Reducer 处理：找到那条 `pending` 消息，标记为 `confirmed`（实色），并更新 `message_id`。
3.  **`MessageFailEvent` (Fact)**: 发送失败。
    *   Reducer 处理：将消息标记为 `error`，或从 State 中移除并触发 Toast 提示。
</details>

---

## 3.11 常见陷阱与错误 (Gotchas)

1.  **Trap: Effect 的结果直接修改 State**
    *   *错误写法*: Effect 的回调函数里写 `globalState.prop = value`。
    *   *后果*: 绕过了 Event Bus 和 Reducer，导致状态变更不可追踪，时间旅行失效，并发竞态。
    *   *修正*: Effect 的回调只能 `emit(NewEvent)`。

2.  **Trap: 忽略了“去抖动 (Debounce)”在 Runtime 层的必要性**
    *   *现象*: 用户的麦克风有点接触不良，100ms 内触发了 5 次 `DeviceConnect/Disconnect` 事件。
    *   *后果*: Reducer 疯狂状态切换，触发了 5 次重连 Effect，导致服务器被 DDoS。
    *   *修正*: 在 Ingress 层或 Reducer 前置管道使用 `debounce` 或 `throttle` 算子，过滤高频噪声。

3.  **Trap: 混淆逻辑时间与物理时间**
    *   *场景*: 你在做 Timeout 检查，使用了 `Date.now()`。
    *   *后果*: 当你以 10 倍速回放测试时，Timeout 逻辑会全部错误，因为物理时间的流逝速度和回放的逻辑时间流逝速度不匹配。
    *   *修正*: Runtime 内部维护一个 `VirtualClock`，由 `TickEvent` 驱动。所有计时逻辑都基于这个虚拟时钟。

4.  **Trap: 死循环事件**
    *   *场景*: Reducer 收到 Event A -> 发出 Effect B -> Effect B 成功后发出 Event A。
    *   *后果*: 无限递归，栈溢出或资源耗尽。
    *   *修正*:
        *   在 Event Envelope 中加入 `causality_chain`，检测到环路则熔断。
        *   Reducer 逻辑必须收敛：保状态变更后，同样的 Event 不会再次触发同样的 Effect。
