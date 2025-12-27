# Appendix A｜术语表 & FRP 算子速查 (The Agent Developer's Rosetta Stone)

## 1. 开篇段落

在构建实时 LLM Agent 时，开发者往往面临“三种语言”的认知摩擦：
1.  **AI 语言**：关注 Prompt、Token、Context Window、RAG、Tool Call。
2.  **分布式系统语言**：关注 Latency、Throughput、Backpressure、Consistency、CAP。
3.  **FRP（函数响应式编程）语言**：关注 Stream、Observable、Subscription、Operator、Marble Diagram。

本附录是这三种语言的“罗塞塔石碑（Rosetta Stone）”。它不是一本通用的 RxJS/ReactiveX 手册，而是**专门为 Agent 开发者定制的战术手册**。

**本章目标**：
*   **消除歧义**：明确“流（Stream）”在 Token 生成和事件总线中的不同含义。
*   **模式匹配**：当你脑海中有一个 Agent 行为（如“说话被打断”、“工具超时重试”）时，能迅速找到对应的 FRP 拓扑结构。
*   **决策辅助**：提供一张“算子决策树”，帮助你在几十个算子中选出对的那一个。

---

## A.1 跨域术语对照表 (The Grand Map)

我们将术语分为三个层级：**实体（名词）**、**行为（动词）**、**时空属性（形容词/副词）**。

### A.1.1 实体映射：系统里有什么？

| FRP 术语 | 对应 Agent 概念 | 对应系统/后端概念 | 深度解析与直觉模型 |
| :--- | :--- | :--- | :--- |
| **Stream / Observable** | **事件流 / Token 流** | Channel / Topic / Pipe | **传送带**。无论传送带上是用户的语音帧，还是 LLM 吐出的一个个字，它代表了“未来可能到达的数据”。它是*懒惰*的，没人看它就不转。 |
| **Event / Item** | **Token / Action / Signal** | Message / Packet | **包裹**。传送带上的货物。可能是 `SystemPrompt`（大包裹），也可能是 `Token "the"`（小包裹）。 |
| **Signal / Behavior** | **Memory / Context** | State Variable / Cache | **仪表盘读数**。不同于流（发生过的历史），Signal 代表“此时此刻”的值。例如：当前的对话历史列表、当前的 API 余额。Signal = Stream + 初始值。 |
| **Subscription** | **Agent 启动 / 监听** | Consumer / Listener | **插头**。只有插上插头，传送带才通电运行。一个 Stream 可以有多个插头（Multicast）。 |
| **Subscriber / Observer** | **UI / Logger / Speaker** | Sink / Handler | **终端处理机**。消费数据的地方。如：TTS 模块消费文本流将其变为声音，DB 模块消费日志流将其落盘。 |
| **Teardown / Dispose** | **任务取消 / 停止生成** | Cleanup / Close FD | **拉闸**。非常重要。用户切走页面时，必须触发 Teardown，不仅停止 UI 渲染，还要沿着链路向上游发送“取消信号”，杀掉后端的 Python 进程。 |

### A.1.2 行为映射：系统在做什么？

| FRP 术语 | 对应 Agent 场景 | 核心逻辑 |
| :--- | :--- | :--- |
| **Map / Project** | **格式化 / 提取** | $f(x)$。一对一变换。如：把 `ToolResult` JSON 对象转成自然语言字符串。 |
| **FlatMap / MergeMap** | **并发执行工具** | 也是 $f(x)$，但 $x$ 变成了一个新流。如：收到用户指令（事件） -> 触发 Google 搜索（流）。**这是副作用的入口。** |
| **Switch (SwitchLatest)** | **打断 / 上下文切换** | **喜新厌旧**。一旦新任务来了，旧任务如果还没做完，直接杀掉。这是 Agent 响应性的核心。 |
| **Scan / Reduce** | **记忆构建** | **滚雪球**。把过去所有的对话（Events）累积成一个 List（State）。 |
| **Filter** | **安全护栏 / 路由** | 卫语句。如：只允许置信度 > 0.9 的 Tool Call 通过；拦截包含敏感词的 Prompt。 |
| **Combine / Zip** | **多模态融合** | **拼装**。等待“图片”和“文本”都到齐了，才打包发给 Vision Model。 |

### A.1.3 时空属性：关于时间与压力

| 术语 | 解释 | Agent 场景示例 |
| :--- | :--- | :--- |
| **Backpressure (背压)** | 下游处理慢，上游怎么办？ | LLM 生成速度（100 token/s）快于 TTS 朗读速度（50 token/s）。需要缓冲（Buffer）或丢弃（Drop）。 |
| **Debounce (防抖)** | 等待平静期。 | VAD（语音活动检测）。用户闭嘴 500ms 后，才认为这句话说完了，开始发送给模型。 |
| **Throttle (节流)** | 限制最大频率。 | UI 渲染 Token。不要每来一个字重绘一次，每 30ms 重绘一次，节省 CPU。 |
| **Race (竞态)** | 谁快用谁。 | Speculative Execution（投机执行）。同时跑“快速模型”和“精准模型”，如果快速模型置信度高且先返回，就用它。 |
| **Hot vs Cold** | 直播 vs 录播。 | 麦克风是 Hot（不听也在录/丢失），API 调用是 Cold（调用了才开始，每次调用都是新的）。 |

---

## A.2 算子决策树 (Operator Decision Tree)

当你面临一个逻辑问题时，请查阅此表来选择算子。

**Q1: 我要处理的是“数据变形”还是“时间控制”？**
*   **数据变形** -> Q2
*   **时间/流程控制** -> Q3

**Q2: 变形是一对一，还是一对多（或涉及异步任务）？**
*   **一对一** (e.g., String -> JSON) -> `map`
*   **涉及异步/副作用** (e.g., Query -> Search -> Result) -> **Q2.1**
    *   **Q2.1: 新任务来了，旧任务怎么办？**
        *   旧任务必须取消（打断）-> `switchMap` (最常用)
        *   旧任务必须做完（排队）-> `concatMap` (保证顺序)
        *   旧任务继续做，新任务也做（并发）-> `mergeMap` (高吞吐)
        *   旧任务正在做，新任务直接忽略（去重）-> `exhaustMap` (防手抖点击)

**Q3: 我有多个流，怎么合在一起？**
*   **逻辑“或” (OR)**：任何一个流来数据都行 -> `merge`
*   **逻辑“与” (AND)**：必须大家都有据 -> `combineLatest` (取最新) 或 `zip` (严格配对)
*   **逻辑“赛跑” (RACE)**：只取最快的那个 -> `amb` / `race`
*   **逻辑“主从” (Trigger)**：A 发生时，取 B 的当前值 -> `withLatestFrom`

**Q4: 数据来得太快/太乱怎么办？**
*   **攒一波一起发** -> `buffer` / `window` (Dynamic Batching)
*   **太吵了，等静下来再发** -> `debounce` (VAD)
*   **太快了，按固定节奏发** -> `throttle` (UI Rendering)
*   **只要第 N 个** -> `take`, `skip`, `sample`

---

## A.3 核心设计模式速查 (Pattern Cheat Sheet)

以下是 Agent 开发中最高频出现的 6 种 FRP 模式。

### Pattern 1: The "Interruptible Thought" (可打断的思考)
**场景**：用户在 Agent 还在生成长文本时插话，Agent 必须立即停止生成并处理新输入。
**关键算子**：`switchMap` (或 `switchLatest`)

```ascii
User Audio:    ---[Audio A]-----------[Audio B]---->
                  | (start gen A)      | (ABORT gen A, start gen B)
                  v                    v
LLM Process:      [Gen A_1, A_2...]    [Gen B_1, B_2...]
                  |                    |
Output Stream: ---[A_1, A_2]-----------[B_1, B_2...]-->
                  (A_3, A_4 never output)
```
> **Rule of Thumb**: 将整个“语音转文字 -> LLM 推理 -> TTS”的链条封装在一个 Observable 中，然后用 `switchMap` 挂载到 User Input 上。

### Pattern 2: Dynamic Batching (动态批处理)
**场景**：为了省钱或提高 GPU 利用率，将 50ms 内到达的 Embedding 请求合并发送。
**关键算子**：`bufferTime` 或 `buffer(debounce)`

```ascii
Requests:      --R1--R2----R3--R4------R5---->
Buffer(50ms):  ------[R1,R2]-----[R3,R4]-----> (R5 pending)
Process:       ------Batch(1,2)---Batch(3,4)->
```
> **Rule of Thumb**: 使用 `bufferTime(maxAge, maxCount)` 双限制，即“要么等 50ms，要么攒够 10 个”，避免流量低时无限等待。

### Pattern 3: Speculative Execution (投机执行/赛马)
**场景**：同时查询 Cache 和 LLM，或者同时查询 Google 和 Bing。谁先回来用谁。
**关键算子**：`amb` (Ambiguous) 或 `race`

```ascii
Stream A (Cache):  ----------(Empty/Slow)---|
Stream B (LLM):    ----[Result]-------------|
Result Stream:     ----[Result]-------------> (A ignored)
```
> **Gotcha**: 即使 A 输了，如果它是 Cold Observable，它的副作用可能仍在后台运行。需要确保输掉的流能响应 Unsubscribe 信号来取消网络请求。

### Pattern 4: The "Stabilizer" (VAD / 防抖)
**场景**：用户说话时有停顿。不能把每次停顿都当成结束。
**关键算子**：`debounceTime`

```ascii
Volume > Threshold: -Y-Y-N-N-Y-N-N-N-N-N-N--->
Raw Signal:         -1-1-0-0-1-0-0-0-0-0-0--->
Debounce(500ms):    --------------------[0]--> (Only fires here)
```
> **Rule of Thumb**: 真正的 VAD 通常比简单的 `debounce` 复杂，往往结合 `distinctUntilChanged`（只有状态从 Talking 变 Silent 才触发）。

### Pattern 5: Memory Accumulation (状态累积)
**场景**：维护多轮对话历史。
**关键算子**：`scan`

```ascii
New Msg:       ---[User:Hi]---[AI:Hola]------>
Scan (List):   ---[List_1]----[List_2]------->
                  [Hi]        [Hi, Hola]
```
> **Rule of Thumb**: `scan` 是流式系统的“内存条”。务必配合 `shareReplay(1)`，否则新加入的 UI 组件（如打开 Debug 面板）会从头开始重放整个历史，导致计算浪费。

### Pattern 6: Robust Retry (指数回退重试)
**场景**：Tool Call 失败（如 429 Too Many Requests），需要重试，但不能立即重试，要越等越久。
**关键算子**：`retryWhen` + `zip` + `timer`

```ascii
Attempt 1:     --X (Error)
Wait:          ----(1s)
Attempt 2:             --X (Error)
Wait:                  --------(2s)
Attempt 3:                     --[Success]-->
```
> **Rule of Thumb**: 不要简单的 `retry(3)`。在 LLM 场景下，拥塞控制至关重要。使用 `retryWhen` 实现指数回退（Exponential Backoff）。

---

## A.4 常见陷阱深度剖析 (Deep Dive into Gotchas)

### A.4.1 The "Lazy Stream" Trap (懒惰执行陷阱)
*   **现象**：代码写得很完美，Prompt 拼好了，Log 打印了 "Stream Created"，但是 LLM 就是不发请求，网络面板一片空白。
*   **原因**：**Observables 是懒惰的 (Lazy)**。如果没有人 `subscribe()`（或者没有连接到 UI/Audio Output 等最终消费者），中间的逻辑根本不会运行。这与 Promise（一创建就执行）截然不同。
*   **修复**：确保流的末端有一个 Subscriber，或者使用 `publish()` + `connect()` 强制启动（不推荐，容易失控），或者在调试时手动 `.subscribe()`。

### A.4.2 The "Infinite Wait" (死锁)
*   **现象**：使用了 `combineLatest` 或 `zip`，但 Agent 卡住了，不输出任何东西。
*   **原因**：
    1.  `combineLatest` 需要**所有**输入流都至少产生过**一个**值才会触发第一次输出。如果有一个流（如 `UserLocation`）迟迟没有初始值，整个流程就会卡死。
    2.  `zip` 需要所有流产生的数据**严格一对一配对**。如果流 A 产生了 3 个事件，流 B 只产生了 2 个，`zip` 会一直等 B 的第 3 个事件。
*   **修复**：为流添加 `startWith(defaultValue)` 以确保有初始值。

### A.4.3 The "Clock Skew" (时钟偏差)
*   **现象**：在回放测试（Replay）中，逻辑完全乱了，或者 `bufferTime` 切分的数据包大小不一致。
*   **原因**：FRP 算子默认使用 `Scheduler.now()`（物理系统时间）。在快速回放日志时，处理 1 小时的日志可能只需要 1 秒物理时间，导致基于时间的算子（debounce, throttle, bufferTime）全部失效。
*   **修复**：**必须引入 VirtualTimeScheduler**。所有的时序算子都应接受一个可选的 Scheduler 参数。在测试和回放模式下，传入虚拟时钟（详见 Chapter 2 & Chapter 20）。

### A.4.4 Cancellation Propagation Failure (取消失效)
*   **现象**：UI 上点击了停止，但后台日志显示 Python Tool 还在跑，甚至跑完后又触发了新的 LLM 生成。
*   **原因**：
    1.  使用了 Promise 混用。Promise 一旦创建无法取消（原生），如果 FRP 链条中间断开变成 Promise 再转回来，取消信号会丢失。
    2.  Tool 执行逻辑没有监听 AbortSignal。
*   **修复**：
    1.  尽量全程使用 Observable。如果必须用 Promise，确保包裹在 `from(promise)` 中并不是万能的，要看 Promise 内部支持不支持取消。
    2.  编写 Tool/Effect 时，务必接受一个 `AbortSignal` 或 `CancellationToken`，并在 Unsubscribe 逻辑中触发它。

---

## 本章小结

*   FRP 将 Agent 开发从“回调地狱”和“状态同步噩梦”中解救出来，将其转化为“管道连接”问题。
*   掌握 `switchMap` 也就掌握了 Agent 的“注意力管理”。
*   理解 `Hot` vs `Cold` 以及 `Lazy` 特性，是避免诡异 Bug 的基础。
*   本附录中的“算子决策树”和“设计模式”应作为开发时的案头参考。

---

## 练习题

### 基础题
1.  **翻译题**：将以下自然语言需求翻译成算子组合。
    *   “当用户停止输入超过 1 秒后，发送请求，但如果用户又输入了，就取消上一次请求。”
    *   “把 LLM 返回的 Markdown 文本流，每 50ms 渲染一次到屏幕上。”
    *   “同时监听键盘输入和麦克风输入，无论谁有动静，都作为‘用户活动’事件。”

2.  **纠错题**：以下代码有什么问题？
    ```javascript
    // 目标：每秒打印一次 "Heartbeat"
    const timer$ = interval(1000);
    // 没有 subscribe
    ```

### 挑战题
3.  **模式设计：RAG 的“混合去重”**
    你需要实现一个检索流：
    *   来源 A：本地向量数据库（快，50ms）
    *   来源 B：网络搜索（慢，2s）
    *   要求：
        1. 必须等待 A 返回。
        2. 如果 A 的置信度 > 0.9，直接使用 A，取消 B（节省成本）。
        3. 如果 A 的置信度 < 0.9，等待 B 回，然后合并 A 和 B 的结果。
        4. 如果 B 超时（3s），仅使用 A 的结果（如果 A 存在）。
    *   *提示：这需要组合 `filter`, `takeUntil` 或条件分支逻辑。试着画出流的拓扑图。*

4.  **架构思考：全局暂停**
    系统有一个全局的 `isPaused$` 信号（由 UI 暂停按钮控制）。当 `isPaused$` 为 `true` 时，所有的 Tool Call 和 LLM 生成都应该被挂起（不仅仅是丢弃结果，而是真正的暂停进度，如果可能的话），或者至少不再触发新的步骤。在 FRP 中如何优雅地将这个信号“广播”给系统中成百上千个细碎的流？
    *   *提示：考察 `windowToggle` 或自定义的 `pausable` 高阶算子。*

### 练习题参考答案

<details>
<summary>点击展开答案</summary>

**1. 翻译题**
*   `userInput$.pipe(debounceTime(1000), switchMap(req => apiCall(req)))`
*   `markdownStream$.pipe(throttleTime(50))` (注意 throttle 和 debounce 的区别，这里要持续渲染)
*   `merge(keyboard$, microphone$)`

**2. 纠错题**
*   Observables 是 Lazy 的。没有 `.subscribe()`，`interval` 永远不会启动，定时器不会运行，什么都不会发生。

**3. 模式设计：RAG 混合去重**
*   思路：这不是简单的 `race`，而是有条件依赖。
*   伪代码逻辑：
    ```javascript
    const sourceA$ = localDb.query().pipe(share()); // 必须 share，因为要被用多次
    const sourceB$ = webSearch.query();

    return sourceA$.pipe(
        mergeMap(resultA => {
            if (resultA.score > 0.9) {
                return of(resultA); // 满足条件，直接返回，忽略 B
            } else {
                // 不满足，启动 B，并与 A 合并
                return sourceB$.pipe(
                    timeout(3000), // B 超时控制
                    catchError(() => of(emptyResult)), // 超时后 B 视为空
                    map(resultB => mergeResults(resultA, resultB))
                );
            }
        })
    );
    ```
    *注：这里利用了 FRP 的懒惰性。只有进入 `else` 分支订阅 `sourceB$` 时，网络搜索才会真正发出。*

**4. 架构思考：全局暂停**
*   这在 FRP 中是一个高级话题。
*   **方案一（拦截器）**：在所有 Effect 的入口处 `pipe(skipUntil(isPaused$.pipe(filter(v => !v))))` —— 这很难维护。
*   **方案二（switchMap 切换）**：
    `isPaused$.pipe(switchMap(paused => paused ? NEVER : rootTaskStream))`。
    但这会完全重置流，而不是“挂起”。
*   **方案三（自定义 pausable 算子）**：
    需要实现一个带缓冲的算子。当 paused 时，将上游事件存入 buffer；当 unpaused 时，释放 buffer。
    对于“正在运行的副作用”（如 API Call），通常无法“暂停”，只能选择“取消”或“继续执行但暂时不处理回调”。在 Agent 语境下，“暂停”通常意味着“停止调度新的 Step”，这可以通过控制 Scheduler 或主 Loop 的 Trigger 来实现。

</details>
