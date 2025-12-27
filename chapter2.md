# Chapter 2｜FRP 基础：用事件流描述 Agent

## 2.1 开篇：从“请求-响应”到“持续流”

在传统的 Web 开发或简单的 Chatbot Demo 中，我们习惯于**“回合制”**逻辑：用户发话 -> 系统等待 -> LLM 思考 -> 返回结果。这种线性的 `await request()` 模型在构建复杂的**实时（Real-time）Agent** 时会迅速失效。

**为什么？请看真实的 Agent 场景：**
> 用户正在说话（音频流 continuous inflow），LLM 正在输出上一句话的后半段（Token 流 continuous outflow）。此时，VAD（语音活动检测）突然判定用户语气激动（打断事件 interrupt event），系统需要立即停止 LLM 生成，清空播放缓冲区，同时记录日志，并开始处理新的语音输入。

这里没有单一的入口和出口，只有错综复杂的、并发的**事件流**。

FRP（Functional Reactive Programming）提供了一种数学上严谨的方式来驯服这种复杂性。它的核心心法是：**把时间作为第一公民，把一切变化抽象为流。** Agent 不再是一个按顺序执行指令的“过程”，而是一个**数据管道网络**。

**本章学习目标**：
1.  **思维重塑**：从“控制流（Control Flow）”转向“数据流（Data Flow）”。
2.  **核心词汇**：掌握 Stream、Signal、Effect、Backpressure 等 FRP 术语在 Agent 中的映射。
3.  **算子武器库**：熟练运用 `switchLatest`（打断）、`scan`（记忆）、`withLatestFrom`（上下文注入）等关键算子。
4.  **架构雏形**：学会如何构建“纯逻辑”与“副作用”分离的管道。

---

## 2.2 核心概念论述

### 2.2.1 Stream vs Signal：世界的两种形态

在构建 Agent 时，区分“间发生的”和“持续存在的”至关重要。

#### 1. Stream (事件流 / Observable Sequence)
*   **定义**：一系列离散的事件，**只在特定的时间点发生**。流是可以结束（Completed）或出错（Error）的。
*   **Agent 映射**：
    *   用户的键盘敲击 (`KeyPressStream`)
    *   LLM 吐出的一个 Token (`TokenStream`)
    *   工具返回的一个报错 (`ErrorStream`)
    *   VAD 检测到的“说话开始”信号 (`SpeechStartStream`)
*   **直观理解**：像雨点落下，或者传送带上的包裹。你不能问“现在的雨点是什么”，你只能接住掉下来的那一个。

#### 2. Signal (信号 / Behavior / Property)
*   **定义**：一个**随时间连续变化的值**。它在任何时刻都有且仅有一个当前值（Current Value）。
*   **Agent 映射**：
    *   当前的对话历史 (`ContextSignal`)
    *   当前的 Token 预算余额 (`BudgetSignal`)
    *   用户的在线状态 (`ConnectivitySignal`)
    *   Prompt 的前模板 (`PromptTemplateSignal`)
*   **直观理解**：像水位线、温度计、或者银行账户余额。你可以随时问“现在是多少”。

**ASCII 图解：**

```text
Stream (用户点击发送):
-------o-----------o-------o----> 时间
       ^           ^       ^
     Send        Send    Send

Signal (当前输入框内的文本):
       "H"         "Hel"   "Hello"
-------+-----------+-------+----> 时间
Value: "Hello" ------------------ (任意时刻去取，都能取到当前值)
```

> **Rule of Thumb**:
> *   如果你关心“**发生**的一瞬间”（触发动作），用 **Stream**。
> *   如果你关心“**现在**的状态是什么”（作为数据源或条件），用 **Signal**。
> *   Signal 通常由 Stream 通过 `scan` 算子生成；Stream 可以由 Signal 的变化采样（sample）而来。

### 2.2.2 扩展概念：Hot vs Cold Observables

在 LLM Agent 中，理解 Hot/Cold 极其重要，否则会造成**重复计费**或**状态不同步**。

*   **Cold Observable (冷流)**：只有被订阅（Subscribe）时才开始执行逻辑。**每个订阅者拥有独立的执行流**。
    *   *例子*：一个封装了 LLM API 请求的流。如果你订阅它两次，它会**发送两次 HTTP 请求**，消耗双倍 Token。
*   **Hot Observable (热流)**：无论有无订阅者，事件都在发生。所有订阅者**共享**同一个数据源。
    *   *例子*：用户的麦克风输入流。无论你是否在处理，用户的话都已经说出去了。
    *   *转换*：使用 `share()` 或 `multicast()` 算子可以将 Cold 转为 Hot。

### 2.2.3 时间语义：逻辑、物理与虚拟时间

FRP 赋予了我们操纵时间的能力，这对 Agent 的测试和稳定性至关重要。

1.  **物理时间 (Wall-clock Time)**：真实世界的秒数。
    *   *用途*：设置 API 超时（Timeout）、防抖（Debounce）、API 限流（Rate Limit）。
2.  **逻辑时间 (Ordering)**：关注事件的因果序（Happens-Before），不关注具体过几毫秒。
    *   *用途*：确保“用户先问，Agent 后答”，确保“Prompt 必须在 Tool 结果返回后更新”。
3.  **虚拟时间 (Virtual Time)**：**测试神器**。
    *   *概念*：Agent 内部的时间由一个 `Scheduler` 控制。在单元测试中，我们可以用 `TestScheduler`。
    *   *场景*：测试“如果用户 30 秒不说话，Agent 主动打招呼”。
    *   *威力*：在虚拟时间下，这 30 秒可以在 1 毫秒内模拟完成。测试既快又确定（Deterministic）。

---

## 2.3 常用算子映射 (The Agent's Toolkit)

Agent 的智能不全在 LLM，很大一部分在**算子组合**。以下是构建 Agent 的“原子动作”。

### 2.3.1 变换与过滤 (Transforming & Filtering)

*   **`map`**: $A \to B$
    *   *用例*：把 `AudioChunk` 转为 `Spectrogram`；把 `LLMResponse` 提取为 `ContentString`。
*   **`filter`**: $A \to A | \emptyset$
    *   *用例*：过滤掉置信度 < 0.8 的语音识别结果；过滤掉 `<|padding|>` Token。
*   **`scan` (累积)**: $(State, Event) \to State$
    *   *地位*：**状态机的灵魂**。
    *   *用例*：将离散的 Token 流累积成完整的句子；将多轮对话累积成 `MessageHistory`。
    ```text
    Events:  "Hi"      "Bot"       "Reply"
               |         |           |
    [scan] -> (s+"Hi") -> (s+"Bot") -> (s+"Reply")
               |         |           |
    State:   ["Hi"]    ["Hi","Bot"] ["Hi","Bot","Reply"]
    ```

### 2.3.2 组合与上下文 (Combining & Context)

*   **`merge`**: $StreamA, StreamB \to Stream(A|B)$
    *   *用例*：**多模态输入**。用户既可以打字，也可以说话。`merge(textInput, speechToText)` 让下游逻辑统一处理。
*   **`withLatestFrom`**: $StreamA, SignalB \to Stream(A, B_{current})$
    *   *地位*：**RAG 与 Context 注入的关键**。
    *   *语义*：当事件 A 发生时，顺便带上 B 当前的值。
    *   *用例*：用户发出提问 (Stream) 时，抓取当前的文档数据库索引 (Signal) 或当前的系统时间 (Signal)，组合成一个 `(Query, Context)` 对，送给 LLM。
*   **`combineLatest`**: $SignalA, SignalB \to Signal(A, B)$
    *   *用例*：UI 状态同步。当 `IsLoading` 信号或 `HasError` 信号任意一个变化时，更新 UI 的状态。

### 2.3.3 时序控制 (Timing & Flow Control)

*   **`debounce` (防抖)**:
    *   *用例*：用户打字搜索 RAG。不希望每敲一个键就 Embed 一次。等用户停顿 500ms 后再触发。
*   **`throttle` (节流)**:
    *   *用例*：UI 刷新率控制。LLM 生成 Token 速度可能极快（如 100 token/s），但 UI 没必要每毫秒重绘。`throttle(30ms)` 限制刷新频率。
*   **`window` / `buffer` (分窗/缓冲)**:
    *   *用例*：**动态批处理 (Dynamic Batching)**。不要来一个 Embedding 请求发一个。收集 100ms 内的所有请求，打包成一个 Batch 发送给 GPU。

### 2.3.4 高阶算子 (Higher-Order) —— Agent 的核心逻辑

这是最难理解但也最强大的部分**流的流 (Stream of Streams)**。

*   **`switchLatest` (切换最新)**:
    *   *语义*：当新的**任务流**到来时，**取消**旧的任务流。
    *   *场景*：**打断与改口**。
        1.  用户问 "A" -> 触发 LLM 流 A (正在生成...)
        2.  用户突然改口问 "B" -> 触发 LLM 流 B
        3.  `switchLatest` 自动 unsubscribe 流 A (导致停止生成、关闭连接)，只保留流 B 的输出。
        4.  结果：UI 永远不会出现“前一个问题的幽灵回答”。

*   **`exhaustMap` (耗尽/忽略)**:
    *   *语义*：当前任务未完成前，**忽略**新的任务。
    *   *场景*：**提交按钮防误触**。在“提交订单”工具执行完成前，忽略用户的重复点击。

*   **`concatMap` (排队)**:
    *   *语义*：当前任务完成后，再执行下一个。
    *   *场景*：**TTS 播放队列**。LLM 生成了三句话。第一句没播完，第二句必须排队，不能混音播放。

---

## 2.4 Effect 模型：副用隔离

FRP 提倡“纯逻辑”，但 Agent 必须“脏手”去干活（调 API、写 DB）。我们通过 **Managed Effects** 模式来解决。

**架构模式：Sandwich（三明治）**

1.  **Input Streams**: 用户的麦克风、键盘、系统时间。
2.  **Pure Logic (FRP Graph)**: 这一层**不执行**任何 API 调用。
    *   它接收输入，经过 `map/filter/scan` 等变换。
    *   它的输出不是“结果”，而是**意图 (Intent)** 或 **动作描述 (Action Description)**。
    *   例如：输出一个对象 `{ type: 'CALL_LLM', prompt: '...', temperautre: 0.7 }`。
3.  **Effect Runners (Impure)**: 这一层监听上述意图流。
    *   它拿到 `CALL_LLM` 对象，发起真正的 HTTP 请求。
    *   请求成功或失败后，它产生一个新的 **Result Event**。
4.  **Feedback Loop**: Result Event 也就是一个新的 Stream，流回 **Pure Logic** 层，触发下一步（如更新 UI 或进行下一轮推理）。

**循环图解**：

```text
       (Input Stream)
            |
            v
    +----------------+
    | Pure Agent Core| <---- 这是一个纯函数式的管道网络
    | (State Machine)|
    +----------------+
            | 输出 (Intent Stream: "请帮我查天气")
            v
    +----------------+
    | Effect Runner  | <---- 这里执行 HTTP / DB / Python
    | (Side Effects) |
    +----------------+
            | 反馈 (Result Stream: "天气是晴天")
            v
     (Loop back to Core)
```

---

## 2.5 错误通道与背压 (Error & Backpressure)

### 2.5.1 错误通道 (Error Channel)
在 FRP 中，标准的 Stream 一旦发生 Error 就会**终止 (Terminate)**。这对于 7x24 小时运行的 Agent 是不可接受的。

*   **策略 1：`catchError` & `retry`**
    *   在 Effect Runner 层面捕获网络错误，进行指数退避重试。
*   **策略 2：错误物化 (Materialize)**
    *   不要让流发出 Error 事件，而是让流发出一个“包装了错误的正常事件”。
    *   `Stream<T>` 变成 `Stream<Result<T>>`。
    *   `Result` 可能是 `{ status: 'success', data: ... }` 或 `{ status: 'error', error: ... }`。
    *   这样，核心逻辑流永远不会断开，可以根据 status 分支处理。

### 2.5.2 背压 (Backpressure)
**问题**：LLM 生成 token 的速度（比如 Groq 达到 500 token/s）远快于 TTS 朗读的速度（3 token/s）。如果不处理，内存会爆，或者 TTS 播放延迟巨大。

*   **策略 1：缓冲 (Buffer)**
    *   TTS 播放器维护一个队列。
*   **策略 2：丢弃 (Drop/Throttle)**
    *   对于 UI 渲染（如字幕滚动），如果来不及画，直接丢弃中间帧，只画最新的。
*   **策略 3：控制源头 (Pausing - 较难实现)**
    *   大部分 LLM API 不支持 TCP 级别的背压暂停。通常我们在客户端使用 Buffer 策略。

---

## 2.6 最小可运行示例 (概念管道)

我们来搭建一个最简单的 **语音对话 Agent** 的管道结构。

```text
Input: MicStream (AudioFrame)

1. VAD (Voice Activity Detection):
   SpeakingStream = MicStream -> map(detectVoice) -> distinctUntilChanged()
   // 输出: True, True, False, False...

2. UserIntent (Silence Detection):
   // 用户停止说话 1.5s 后视为一句话结束
   CommitStream = SpeakingStream 
     -> filter(isFalse) 
     -> debounce(1.5s)

3. STT (Speech to Text):
   // 只处理最后确认的那段音频
   TextStream = CommitStream 
     -> withLatestFrom(AudioBuffer) 
     -> map(callSTT_API) 
     -> switchLatest() // 如果识别还没回来用户又说话了，取消前一次识别

4. Prompt Assembly:
   // 将当前文本与历史记录组合
   ContextStream = TextStream 
     -> withLatestFrom(ConversationHistorySignal) 
     -> map(makePrompt)

5. LLM Response:
   TokenStream = ContextStream 
     -> map(callLLM_Stream_API) 
     -> switchLatest() // 用户打断时取消生成

6. TTS & UI:
   AudioOutput = TokenStream -> buffer(sentence) -> concatMap(callTTS)
   DisplayOutput = TokenStream -> scan(accumulateText)
```

---

## 2.7 本章小结

1.  **一切皆流**：输入是流，输出是流，连“状态”也是由流累积而成的信号。
2.  **算子即逻辑**：业务逻辑（如打断、防抖、批处理）不应该写在回调函数里，而应该表现为算子的组合。
3.  **副作用外置**：保持核心逻辑纯净，通过“意图对象”驱动副作用执行器。
4.  **时间可控**：利用虚拟时间进行单元测试，是保证复杂 Agent 逻辑健壮性的唯一途径。

---

## 2.8 练习题

### 基础题 (50%)

**Q1. 数据清洗 (Map/Filter)**
LLM 的流式输出不仅包含文本，还偶尔夹杂思考过程标签（如 `<think>...</think>`）。我们需要实时显示文本给用户，但必须隐藏思考过程。
输入流：`"Hel"`, `"lo"`, `"<"`, `"thi"`, `"nk>"`, `"hmm"`, `"</"`, `"think>"`, `" world"`
**要求**：设计一个算子组合，去除标签内的内容。
*(提示：这需要一个简单的状态机)*

<details>
<summary>点击查看参考答案</summary>
这单用 `filter` 很难，通常结合 `scan` 维护一个 `inTag` 状态。
State: `{ buffer: "", inTag: boolean, outputToUser: "" }`
Input: Token string.
Logic inside scan:
- 如果遇到 `<`，设置 `inTag=true`。
- 如果遇到 `>`，设置 `inTag=false`。
- 如果 `inTag` 为 true，将 token 存入 buffer（以防不是标签需要回吐，或者直接丢弃）。
- 如果 `inTag` 为 false，将 token 输出。
最终由 `scan` 输出的状态中提取 `outputToUser` 字段。
</details>

**Q2. 状态累积 (Scan)**
设计一个 **Token 计费器**。输入是 LLM 的 Response 事件流，每个事件包含 `{ usage: { prompt: 10, completion: 5 } }`。输出是一个 Signal，显示当前 Session 累计消耗。

<details>
<summary>点击查看参考答案</summary>
算子：`scan`
初始值：`0`
Accumulator 函数：`(acc, event) => acc + event.usage.prompt + event.usage.completion`
使用 `startWith(0)` 确保一开始就有值。
</details>

**Q3. 避免抖动 (Debounce)**
用户正在拖动 Slider 调整 LLM 的 `Temperature` 参数（0.0 - 1.0）。Slider 会产生密集的高频事件。我们希望在用户停止拖动 300ms 后，才向后台发送配置更新请求。

<details>
<summary>点击查看参考答案</summary>
`inputStream.debounceTime(300ms)`
这会丢弃中间所有的快速变化，只发射停止变化后的最后一个值。
</details>

**Q4. 多源合并 (Merge)**
UI 上有一个“重新生成”按钮 (`BtnStream`) 和一个快捷键 Ctrl+R (`KeyStream`)。
如何得到一个统一的 `RegenerateIntent` 流？

<details>
<summary>点击查看参考答案</summary>
`merge(BtnStream, KeyStream)`
得到一个新的流，下游不区分来源，只知道要执行重新生成。
</details>

---

### 挑战题 (50%)

**Q5. 竞态处理与打断 (SwitchLatest) - 核心考点**
场景：
1. 用户说 "查询天气" (T=0s)。
2. Agent 开始调用天气 Tool (耗时 3s)。
3. 在 T=1s 时，用户觉得慢，说 "算了，讲个笑话"。
4. Agent 必须**立即**取消天气查（如果可能），或者至少**丢弃**天气查询的结果，转而开始调用笑话 Tool。
请用伪代码描述这个 Stream 结构。

<details>
<summary>点击查看参考答案</summary>
```typescript
UserInputStream
  .map(input => {
     // 这一步把输入映射为一个 Observable (Tool Execution)
     // 注意：这里返回的是流，而不是结果
     return performToolExecution(input); 
  })
  .switchLatest() // 魔法在这里
  .subscribe(result => showResult(result));
```
当 T=1s 新的 Input 到来，`map` 产生了一个新的 Tool Execution Observable。`switchLatest` 看到新的 Observable 来了，会自动 unsubscribe 旧的（天气）Observable。这通常会触发 API Client 的 AbortController，真正取消网络请求。
</details>

**Q6. 上下文注入 (WithLatestFrom)**
RAG 场景。用户触发 "Ask" 事件 (Stream)。此时我们需要去取 UI 上选中的 "Knowledge Base ID" (Signal)，以及当前的 "User Subscription Level" (Signal) 来决定是否有限。
如何把这三者组合成一个 Request 对象？

<details>
<summary>点击查看参考答案</summary>
```typescript
AskStream.pipe(
  withLatestFrom(KbIdSignal, SubLevelSignal),
  map(([question, kbId, level]) => {
     return { query: question, kb: kbId, userLevel: level };
  })
)
```
注意：这里用 `withLatestFrom` 而不是 `combineLatest`。因为我们只想在用户“提问”的那一瞬间去读状态。如果用户没提问，仅仅切换了 KB ID，不应该触发查询。
</details>

**Q7. 智能重试 (RetryWhen + Zip)**
LLM API 经常报 503 Overloaded。请设计一个流逻辑：
遇到错误时，重试 3 次。
第 1 次等待 1s，第 2 次等待 2s，第 3 次等待 4s (指数退避)。
如果 3 次后还挂，抛出错误。

<details>
<summary>点击查看参考答案</summary>
```typescript
ApiStream.pipe(
  retryWhen(errors => 
    errors.pipe(
      // 产生索引 0, 1, 2
      zip(range(1, 3), (err, i) => i), 
      // 计算延迟
      flatMap(i => timer(Math.pow(2, i) * 1000)) 
    )
  )
)
```
注意：如果 retryWhen 返回的流完成了，原始流也会完成；如果返回的流报错了，原始流才会抛出错误。
</details>

**Q8. 动态批处理 (Buffer/Window)**
Embedding 接口很贵，而且支持 Batch 输入。
系统中有多个组件在并发请求 Embedding。
要求：
1. 如果积攒了 10 个请求，立即发车。
2. 如果不足 10 个，但距离第一个请求已经过了 50ms，也必须发车（低延迟要求）。

<details>
<summary>点击查看参考答案</summary>
这是 `buffer` 算子的经典场景，通常需要自定义 buffer 逻辑或使用特定库的扩展算子（如 RxJS 的 `bufferTime` 或 `bufferCount` 的变体）。
一种简单实现：
`requestStream.bufferTime(50, null, 10)`
(意思是：每 50ms 切分一次，或者虽然没到 50ms 但数量达到 10 个了也切分)。
拿到的是一个 `Array<Request>`，然后传给 Batch API。
</details>

---

## 2.9 常见陷阱与错误 (Gotchas)

1.  **Subscription 泄漏 (Memory Leak)**
    *   *现象*：Agent 运行一小时后越来越慢，或者同一句话被重复回复多次。
    *   *原因*：在组件销毁或用户登出时，没有 `unsubscribe` 那些长连接的流。
    *   *解法*：使用 `takeUntil(destroySignal)` 模式，或使用框架提供的自动管理（如 Angular 的 AsyncPipe，React 的 useEffect cleanup）。

2.  **Promise 与 Observable 混用的地狱**
    *   *现象*：流逻辑中突然出现 `async/await`，导致流变成了 `Observable<Promise<T>>`，下游无法处理。
    *   *原因*：在 `map` 里直接调用了异步函数。
    *   *解法*：凡是涉及异步操作，必须用 `flatMap` (mergeMap), `switchMap`, 或 `concatMap`，决不能用 `map`。

3.  **Hot vs Cold 的误解**
    *   *现象*：HTTP 请求被发送了多次。
    *   *原因*：定义了一个 Cold Observable（如 API 请求），然后在代码里 subscribe 了它两次（一次用于打印日志，一次用于显示 UI）。
    *   *解法*：对于副作用操作，使用 `share()` 或 `shareReplay(1)` 将其转为 Hot 流，确保副作用只执行一次。

4.  **Signal 初始值缺失**
    *   *现象*：UI 启动时一片空白，直到用户第一次交互才显示。
    *   *原因*：`combineLatest` 等待所有源流至少发射一次数据才会执行。如果某个 Signal 没有初始值（Initial Value），整个逻辑就会卡住（Hang）。
    *   *解法*：对所有 Signal 类型的流使用 `startWith(defaultValue)`。
