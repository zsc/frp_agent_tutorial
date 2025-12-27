# Chapter 1｜问题空间与总体架构：为什么用 FRP

## 1.1 开篇段落：从“下棋”到“即时战略”

构建一个传统的 Chatbot 就像在写一个回合制游戏（如国际象棋）：用户走一步（Input），系统思考（Processing），系统走一步（Output）。在这个模型中，时间是静止的，当系统在“思考”时，世界被冻结。

然而，构建一个**生产级、实时的 LLM Agent** 更像是开发一个即时战略游戏（RTS，如星际争霸）。
在 Agent 正在生成一段长回复的同时，后台的数据库查询可能刚刚超时，环境噪音检测器触发了 VAD（语音活动检测）信号，系统时钟跨过了整点需要触发定时任务，而用户此时失去耐心点击了“取消”按钮。

在这个混乱的并发界里，传统的 `request-response` 编程模式（无论是一个巨大的 `while` 循环，还是嵌套的 `async/await`）会迅速坍塌成无法维护的“面条代码”。我们需要一种数学上严谨的抽象来驯服**时间**和**变化**。

本章将介绍 **Functional Reactive Programming (FRP)** 如何作为构建实时 Agent 的“操作系统内核”，将异步事件流、状态管理和副作用编排统一在一个声明式的框架下。

## 1.2 文字论述

### 1.2.1 实时 Agent 的“七大工程地狱”

在深入解决方案之前，我们必须痛苦地直面问题。为什么高阶 Agent 如此难写？因为如果不加抽象，以下七个问题会通过“笛卡尔积”的方式组合爆炸：

1.  **竞态条件 (Race Conditions) 的多维爆发**
    *   *经典场景*：用户说“打开灯”，Agent 调用 Tool A。100ms 后用户说“不，等一下”，Agent 试图取消。但 Tool A 的网络请求已经发出。Tool A 返回时，Agent 该如何处理这个“迟到的真相”？是丢弃、报错还是回滚？
    *   *FRP 视角*：这是事件序列的**顺序**与**时效性**问题。

2.  **状态的时间相关性 (Temporal State)**
    *   Prompt 不是一个静态字符串。它是一个**活的有机体**。它包含“当前时间”（随秒变化）、“最近工具的输出”（随事件变化）、“Token 预算剩余”（随生成变化）。
    *   手动拼接字符串不仅低效，而且容易导致“状态撕裂”（State Tearing，即 Prompt 的一部分认为现在是早上，另一部分却引用了晚上的数据）。

3.  **资源预算与流控 (Budgeting & Backpressure)**
    *   LLM 的 Token 是昂贵的，推理是慢的。如果外部事件（如传感器数据或日志流）以 100Hz 的频率涌入，直接喂给 LLM 会瞬间耗尽预算并阻塞系统。
    *   系统必须具备**背压（Backpressure）**机制：丢弃旧数据、采样、或合并数据。

4.  **异步副作用地狱 (Async Side-effect Hell)**
    *   Agent 需要执行 Python 代码、查询向量库、调用 API。这些操作不仅耗时，而且会失败。
    *   当一个“推测性执行（Speculative Execution）”正在后台跑，而前台逻辑决定不再需要它时，如何**干净地**清理副作用（Kill process, Close socket, Rollback transaction）？

5.  **不可预测性与非确定性**
    *   LLM 的输出每次可能不同，外部 API 的延迟不可控。
    *   这就要求系统必须具备极强的**容错性**和**回退（Fallback）**机制，而这些机制不能侵入主业务逻辑。

6.  **可观测性黑洞**
    *   当 Agent 发疯（死循环或幻觉）时，看静态日志是没有用的。你需要知道：“在 T=10.5s 时，系统的**完整状态快照**是什么？正在进行的流有哪些？”
    *   传统日志记录的是“发生了什么”，而我们需要记录的是“因果关系链”。

7.  **复杂的取消语义 (Cancellation Propagation)**
    *   用户的一个闭嘴”指令，必须像闪电一样穿透整个系统：停止 TTS 播放、停止 LLM 生成、向 Python 解释器发送 SIGINT、取消正在排队的数据库查询。

### 1.2.2 为什么 FRP 是“银弹”？

FRP（Functional Reactive Programming）基于两个核心数学概念：**随时间变化的值（Signals）** 和 **离散的事件序列（Streams）**。

*   **声明式 vs 命令式**：
    *   *命令式*：`monitor_input()` -> `if input changed` -> `update_state()` -> `check_condition()` -> `call_api()`。
    *   *FRP*：`API_Call_Stream = Input_Stream.filter(condition).map(to_request)`。
    *   我们描述**数据之间的关系**，而不是**控制流的跳转**。

*   **时间的可控性**：
    *   在 FRP 中，时间可以是物理时间，也可以是**虚拟时间**。
    *   这意味着我们可以编写单元测试，在 1 毫秒内模拟 Agent 运行 24 小时的行为，或者“回放”昨晚发生的一个 Bug，且保证 100% 的确定性。

*   **组合性 (Composability)**：
    *   处理“重试”逻辑不需要改写业务代码，只需要在流上挂一个 `.retry(3)` 算子。
    *   处理“防抖”逻辑只需要挂一个 `.debounce(500ms)`。
    *   这种像乐高积木一样的组合能力，是应对复杂度的唯一解法。

### 1.2.3 总体系统分层架构

我们将一个完备的 FRP Agent 系统划分为四个象限。请记住这张图，后续章节将反复填充其中的细节。

```ascii
      [ 现实世界 / Reality ]                   [ 记忆与存储 / Persistence ]
             |      ^                                     ^      |
   (Sensors) |      | (Actuators)              (Log/DB)   |      | (Retrieval)
             v      |                                     |      v
+-----------------------------------------------------------------------+
|  Layer 1: I/O 适配层 (Ingress / Egress)                               |
|-----------------------------------------------------------------------|
|  * UserMsgStream (Text/Audio)   * TimeTickStream (Clock)              |
|  * SystemEventStream (Interrupt) * ToolResultStream (Async returns)   |
|  --> 归一化为标准 Event 对象 -->                                        |
+-----------------------------------------------------------------------+
                |
                v  (Event Stream)
+-----------------------------------------------------------------------+
|  Layer 2: FRP 核心运行时 (The Brain)                                   |
|-----------------------------------------------------------------------|
|  [ 状态机 / Reducers ] <---- (Previous State)                         |
|     (State, Event) => NewState                                        |
|                                                                       |
|  [ 信号衍生 / Derived Signals ]                                       |
|     State -> PromptContextSignal (实时 Prompt)                        |
|     State -> BudgetSignal (实时预算)                                  |
|                                                                       |
|  [ 决策策略 / Policy (LLM) ]                                          |
|     PromptSignal + EventStream -> DecisionStream                      |
|                                                                       |
|  [ 调度器 / Scheduler ]                                               |
|     处理流的合并 (Merge)、竞态 (Switch)、防抖 (Debounce)                |
+-----------------------------------------------------------------------+
                |
                v  (Effect Description Stream)
+-----------------------------------------------------------------------+
|  Layer 3: 副作用执行层 (Effect Runners)                                |
|-----------------------------------------------------------------------|
|  * 此层不含业务逻辑，只负责执行指令并返回结果流 *                          |
|  [LLM Runner]  [Python Runner]  [RAG Fetcher]  [TTS Player]           |
|  (支持取消、重试、超时控制)                                             |
+-----------------------------------------------------------------------+
```

### 1.2.4 关键抽象概念字典 (The Vocabulary)

在本书中，我们将严格遵守以下术语定义：

1.  **Event (事件)**:
    *   系统中发生的最小原子单位。必须包含 `id`, `timestamp`, `type`, `payload`。
    *   *例子*：`{ type: "USER_VOICE_CHUNK", payload: <bytes>, time: 1001 }`。

2.  **Stream (流)**:
    *   随时间发生的一系列 **Event**。它是离散的。
    *   *隐喻*：传送带上的包裹。
    *   *典型用途*：用户输入、工具返回结果、错误通知。

3.  **Signal (信号/行为)**:
    *   随时间连续变化的 **值**。在任何时刻进行采样（Sample），都有一个定义良好的值。
    *   *隐喻*：温度计的读数、水箱的水位。
    *   *典型用途*：当前 Token 消耗量、连接状态、构造好的 Prompt 字符串。
    *   *关键区别*：Stream 处理的是“变化”，Signal 处理的是“状态”。

4.  **Reducer (归约器)**:
    *   一个纯函数 `(CurrentState, Event) -> NextState`。
    *   它是系统状态演变的唯一途径。它不执行副作用，只计算新状态。

5.  **Effect (副作用描述)**:
    *   一个**数据对象**（Data Object），描述了“想要做什么”，而不是“正在做什么”。
    *   *例子*：`UseSearchTool(query="weather")` 是一个 Effect 对象。真正的 HTTP 请求由 Layer 3 执行。
    *   这样做的好处是：我们可以拦截、修改、记录、甚至丢弃 Effect，而无需 mock 复杂的外部环境。

6.  **Subscription (订阅)**:
    *   连接 Stream 和 Observer 的桥梁。
    *   **Disposable (可丢弃资源)**：每个订阅都必须返回一个“句柄”，调用该句柄可以取消订阅（切断水管）。这是处理取消逻辑的关键。

### 1.2.5 范式迁移：从“对话”到“持续运行的回路”

新手最容易犯的错是将 Agent 视为“请求-处理-响应”的 Web Server。
在 FRP 架构下，Agent 是一个**永远运行的回路 (Infinite Loop)**：

> `Inputs` 驱动 `State` 更新，`State` 驱动 `Prompt` 变化，`Prompt` 驱动 `LLM` 产生 `Effect`，`Effect` 的执行结果变回 `Inputs`。

这个回路在一秒钟内可能运转数十次（处理音频帧），也可能在等待用户时休眠。但**架构本身是不阻塞的**。

---

## 1.3 本章小结

*   **痛点**：实时 Agent 面临竞态、时变状态、资源限制和异步副作用的复杂交织。
*   **解法**：FRP 提供了一套基于时间流的数学抽象，取代了脆弱的回调和锁。
*   **架构**：输入归一化 (Ingress) -> 纯逻辑变换 (Runtime) -> 副作用执行 (Runners)。
*   **核心对象**：Stream（离散事件）、Signal（连续状态）、Reducer（状态更新逻辑）、Effect（副作用意图）。
*   **心智模型**：不要思考“第一步做什么，第二步做什么”，要思考当数据流过管道时，如何变换它”。

---

## 1.4 练习题

### 基础题 (熟悉材料)

**Q1. Stream vs Signal 辨析**
请判断以下场景应该建模为 Stream 还是 Signal，并简述理由。
1.  **Token 计数器**：记录当前会话总共消耗了多少 Token。
2.  **特定的按键按下**：用户按下了 "Esc" 键。
3.  **网络连接质量**：当前是 "4G", "WiFi", 还是 "Offline"。
4.  **LLM 吐出的每一个字**：生成过程中的 chunks。

<details>
<summary>点击查看参考答案</summary>

1.  **Signal**。这是一个随时间累积的状态值，任何时刻问“现在用了多少？”都有答案。
2.  **Stream**。这是离散的、瞬时的事件。如果在这一秒没按，就没有值。
3.  **Signal**。这是一个持续存在的环境状态。
4.  **Stream**。这是一系列离散的数据包序列，生成结束后流就完成了（Complete）。
</details>

**Q2. 纯函数 Reducer**
假设当前状态是 `Context { conversation_history: [], is_speaking: false }`。
发生了一个事件 `Event { type: "USER_START_TALKING" }`。
请写出一个符合 FRP 规范的伪代码 Reducer。如果在 Reducer 里写 `player.stop()` 会有什么问题？

<details>
<summary>点击查看参考答案</summary>

**伪代码 Reducer:**
```text
function reducer(state, event) {
  if (event.type == "USER_START_TALKING") {
    // 返回一个新的状态副本，不要修改原对象
    return {
      ...state,
      is_speaking: true, // 标记状态，但不执行动作
      interrupt_trigger: true // 可以设置一个标记位
    };
  }
  return state;
}
```
**问题**：如果在 Reducer 里直接调用 `player.stop()`，这就不再是纯函数了。
1.  **不可测试**：必须 Mock 播放器才能测试状态逻辑。
2.  **不可回放**：重放日志时，播放器会真的停止，这可能是副作用。
3.  **竞态**：Reducer 运行极其频繁，副作用执行应该由下游的 Effect Runner 专门处理（比如通过监听 `is_speaking` 号的变化来触发停止）。
</details>

**Q3. 架构定位**
你在 Layer 3 (Effect Runners) 发现了一个 Bug：Python 代码执行超时没有抛出异常，而是卡死了。你应该在 Layer 2 (Runtime) 修复它，还是 Layer 3？为什么？

<details>
<summary>点击查看参考答案</summary>

应该在 **Layer 3** 修复。
Layer 3 负责封装副作用的“物理执行细节”，包括超时机制、进程隔离、异常捕获。
Layer 2 只负责发出“请执行代码”的指令，并期望收到“成功”或“失败”的事件流。Layer 2 不应该知道 Python 解释器是如何被管理的。
</details>

### 挑战题 (开放性思考)

**Q4. "时光机" 调试设计**
为了实现类似 Redux DevTools 的“时间旅行”调试（拖动进度条查看 Agent 在过去某一刻的状态），根据本章的架构，你需要持久化保存哪些数据？只需要保存 State 吗？

<details>
<summary>点击查看提示与答案</summary>

**提示**：State 是计算出来的结果，Input 是源头。

**答案**：
只需要保存 **初始状态 (Initial State)** 和 **Ingress 层的所有输入 Event Stream (带时间戳)**。
只要有这两样东西，配合确定性的 Reducer 和逻辑，我们就可以从头重新计算出任何时刻的 State。
*优化*：为了性能，可以每隔 N 个事件保存一个 State 快照 (Snapshot)，回放时从最近的快照开始计算。
单纯保存历史 State 列表也是一种方法，但它无法重现“为什么变为这个状态”的逻辑路径，且体积可能很大。
</details>

**Q5. 竞态条件：SwitchLatest 算子**
场景：
1. 用户输入 "查询 A" -> 触发 Search(A)。
2. 用户输入 "查询 B" -> 触发 Search(B)。
3. Search(B) 先返回结果。
4. Search(A) 后返回结果。

如果不做处理，界面可能会先显示 B 的结果，然后突然闪烁成 A 的结果（错误的旧数据）。
在 FRP 中，`switchLatest` (或 `switchMap`) 算子如何解决这个问题？请描述其内部机。

<details>
<summary>点击查看参考答案</summary>

`switchLatest` 的机制是：**只保留最新的“流的流”**。
当“触发 Search(B)”这个事件发生并生成了一个新的 Promise/Stream 时，`switchLatest` 会自动**取消订阅 (Unsubscribe)** 之前的 Search(A) 流。
这意味着：
1. 如果 Search(A) 还在请求中，它会被取消（abort request）。
2. 如果 Search(A) 已经返回但还在管道中排队，它的结果会被丢弃，因为下游已经切换到了 B 的频道。
这保证了输出永远对应最后一次输入。
</details>

**Q6. Prompt 作为 Signal**
如果我们将 Prompt 视为一个随时间变化的 Signal。
假设 Prompt 模板包含这句话：`"现在时间是 {CurrentTime}。"`
如果 `CurrentTime` 每秒更新一次，Prompt Signal 也会每秒更新一次。
这是否意味着我们每秒都要把巨大的 Prompt 发送给 LLM？如何优化？

<details>
<summary>点击查看参考答案</summary>

**绝对不是**。这正是 FRP  **Derived Signal (衍生信号)** 和 **Sampling (采样)** 的概念。
1.  虽然 Prompt Signal 每秒都在变，但 LLM 调用是一个**离散的 Event**。
2.  只有当需要调用 LLM 时（例如收到用户消息，或触发了任务），我们才对 Prompt Signal 进行**采样 (Sample)**，获取它在那一瞬间的值。
3.  此外，还可以使用 `distinctUntilChanged` 算子，如果时间的精度只需要到“分钟”，那么在同一分钟内的秒数变化不会触发 Prompt Signal 的下游更新。
</details>

---

## 1.5 常见陷阱与错误 (Gotchas)

1.  **命令式思维残留**
    *   *错误*：在 Stream 的 `.map()` 函数里直接写 `DB.save(data)`。
    *   *后果*：这是在“纯变换”过程中夹带私货（副作用）。这会导致测试困难，且如果流被多次订阅，数据库会被写入多次。
    *   *修正*：`.map()` 应该返回一个 `SaveEffect` 对象，由后续的 Effect Runner 统一处理。

2.  **忽视“热”流 (Hot Observable) 的资源泄漏**
    *   *错误*：订阅了一个无限的事件流（如时钟滴答或麦克风），但在组件销毁或用户断开时忘记 `unsubscribe`。
    *   *后果*：内存泄漏，后台跑着无数个僵尸任务，CPU 飙升。
    *   *Rule-of-Thumb*：在 FRP 中，**创建订阅是责任，销毁订阅是义务**。尽量使用框架提供的 `takeUntil(destroyStream)` 模式自动管理生命周期。

3.  **过度设计的 Reducer**
    *   *错误*：试图把所有东西都塞进一个巨大的全局 State Reducer。
    *   *后果*：性能下降，任何微小的 UI 变动都触发整个状态树的重绘。
    *   *修正*：状态应该分层。UI 的临时状态（如输入框的高亮）不需要进入 Agent 的核心记忆（Memory）。

4.  **混淆 Event 和 Effect**
    *   *错误*：把 `LogEvent` (发生了日志) 和 `WriteLogEffect` (去写日志) 混为一谈。
    *   *修正*：Event 是“过去时”（Fact），Effect 是“将来时”（Intent）。Event 驱动 State，State 驱动 Effect。
