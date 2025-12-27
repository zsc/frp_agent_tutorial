# Chapter 8｜Speculative Exec & Speculative Decoding：让系统“更快也更稳”

## 1. 开篇：用“废弃”换取“时间”

在实时 Agent 系统中，**延迟（Latency）** 是用户体验的头号杀手。传统的“用户输入 -> LLM 思考 -> 工具调用 -> LLM 生成”是一个串行过程，极其缓慢。

**投机（Speculation）** 的核心思想是：**不要等确定了再做，而是在“大概率”发生时就提前做。** 如果猜对了，我们赢得时间；如果猜错了，我们丢弃结果（付出计算成本）。

在 FRP（Functional Reactive Programming）架构中，投机机制有着天然的优势：**FRP 极其擅长“取消”和“切换”**。当更准确的意图（Signal）到来时，旧的投机流（Stream）可以被 `switchLatest` 等算子瞬间切断，确系统状态的一致性，而无需复杂的锁或回调清理逻辑。

**本章目标**：
1. 区分 **工具投机（Exec）** 与 **解码投机（Decoding）** 的应用场景。
2. 掌握利用 FRP 算子（`merge`, `race`, `switchLatest`）管理并发分支的方法。
3. 学习如何在 UI 上优雅地处理“先猜后改”的显示逻辑。
4. 建立投机机制的成本控制与监控体系。

---

## 2. 文字论述

### 8.1 两类投机：行动（Exec） vs 解码（Decoding）

虽然都叫“投机”，但在 Agent 架构中处于不同层级：

1.  **Speculative Decoding (Token Level)**:
    *   **对象**：LLM 的文本生成。
    *   **原理**：用一个小模型（Draft Model）快速生成后续 token 的“草稿”，大模型（Verify Model）并行验证。如果验证通过，直接接受；不通过则回滚。
    *   **FRP 视角**：这是一个高频的 `Event Stream` 处理。输入流是 Draft Token Stream，经过一个 `VerifyFilter` 算子，输出 Verified Token Stream。

2.  **Speculative Execution (Tool/Plan Level)**:
    *   **对象**：外部工具调用、数据库检索、API 请求。
    *   **原理**：在 LLM 还没完全输出 `{"tool": "weather", "city": "Beijing"}` 之前，根据用户输入的“天气”和上下文“北京”，提前发起查询。
    *   **FRP 视角**：这是一个并发的 `Effect` 管理。我们同时启动多个 `Async Task`，哪个先返回且符合最终意图，就用哪个。

### 8.2 触发条件：什么时候该“赌一把”？

不能盲目投机，否则资源浪费严重。需要定义触发器（Triggers）：

*   **高置信路径（High Confidence Path）**：
    *   用户输入特定的“热词”（如 "买"、"搜索"、"播放"）。
    *   历史习惯（用户每天早上 9 点都问天气）。
*   **低风险操作（Idempotent/Read-only）**：
    *   **可以投机**：搜索、检索 RAG、读数据库、生成图片预览。
    *   **严禁投机**：发送邮件、购买支付删除数据、修改配置。
*   **用户可见延迟预算（Latency Budget）**：
    *   当系统检测到当前网络延迟很高，或者 LLM 响应变慢时，主动提高投机积极性（Aggressiveness），试图弥补时间。

### 8.3 Speculative Exec：FRP 的并发与竞赛

在 FRP 中，实现投机执行通常涉及 `merge`（合并流）和 `race`（竞赛）模式。

**ASCII 示意图：预加载（Pre-fetching）模式**

```text
Time -------------------------------------------------------->

User Stream:   "查一下" ---> "北" ---> "京" ---> "天" ---> "气" (完成)
                    |          |
Trigger:            |          +-> [识别意图:可能查天气]
                    |                  |
Speculative Stream: |                  +-> [API: Search "Beijing"] (提前跑)
                    |                                  |
Actual LLM Plan:    |                                  +---------> [Tool Call: Weather(BJ)]
                    |                                                   |
Result Selection:   +---------------------------------------------------+-> (发现缓存中有结果) -> 立即返回
```

**关键算子逻辑**：
*   **Fork**：当输入流满足某种模式（Pattern）时，分叉出一个副流去执行副作用。
*   **Cache/Hold**：副流的结果并不直接推给 UI，而是存入一个临时的 `Observable<Result>` 或缓存池。
*   **Join**：当主流程（LLM 确实发出了工具调用）到达时，先去缓存池检查；如果有，直接通过（0ms 延迟）；如果不一样（猜错了），则丢弃缓存，发起真实请求。

### 8.4 Speculative Decoding：流式修正

在 Token 生成层面，FRP 视其为“快速流”和“慢速修正流”的结合。

```text
Draft Stream (Fast):  A -> B -> C -> D -> E ...
Verify Stream (Slow):      (Check A:OK) -> (Check B:OK) -> (Check C:Fail, real is X)
Final Stream (UI):    A -> B -> [Rollback C,D,E] -> X ...
```

这要求 UI 组件（Observer）必须支持**回退（Retract）**事件。如果你直接把 token 打印到屏幕上且不可修改，投机解码会导致用户看到乱码闪烁。

### 8.5 一致性与撤销：SwitchLatest 的魔法

**Race Condition** 是投机的大敌：如果投机的请求比真实的请求慢返回，且真实请求已经改变了上下文，怎么办？

FRP 的 `switchLatest` (或 `switchMap`) 是终极武器。

*   **定义**：将高阶流（Stream of Streams）压平，每当新流产生时，自动取消（Unsubscribe）旧流。
*   **场景**：
    1.  用户输入 "查一下 A 公司的股价"。
    2.  系统投机启动 `Search(A)`。
    3.  用户立刻改口（打字飞快）"...不对，是 B 公司"。
    4.  系统产生新意图 `Search(B)`。
    5.  `switchLatest` 收到 B 的信号，自动向 `Search(A)` 的 Promise/Task 发送 Cancel 信号。网络请求中断，资源释放。

### 8.6 成本控制：自适应投机

投机是用 **Compute/Token Cost** 换 **Latency**。这个汇率必须划算。

我们可以引入一个 **Speculation Budget Signal（投机预算信号）**：
*   `SuccessRate Signal`：过去 10 次投机，几次用上了？
*   `Cost Signal`：当前 Token 消耗速率。

**Rule of Thumb**：
*   如果 `SuccessRate` > 80%，加大投机力度（生成更长的草稿，预取更多页面）。
*   如果 `SuccessRate` < 20%，进入保守模式，关闭投机。

---

## 3. 本章小结

*   **核心权衡**：投机是利用闲置算力或额外成本来对抗延迟。
*   **安全性原则**：只对**无副作用（Idempotent）**或**只读（Read-only）**的操作进行投机执行（Speculative Exec）。
*   **FRP 优势**：利用 `switchLatest` 实现自动取消，利用 `race` 实现多路竞争，利用 `Stream` 抽象处理 Token 的回滚。
*   **UI 配合**：前端必须能处理“撤回”或“即时修正”的信号，以支持投机解码的视觉呈现（如 Ghost text）。
*   **反馈闭环**：必须监控投机的“命中率”和“浪费率”，动态调整策略。

---

## 4. 练习题

### 基础题

**Q1. 概念辨析**
简述 Speculative Decoding 和 Speculative Execution 的主要区别是什么？它们分别优化了 Agent 的哪部分性能？

<details>
<summary>点击查看提示与答案</summary>

*   **提示**：一个关注“怎么说得快”，一个关注“怎么做得快”。
*   **答案**：
    *   **Speculative Decoding**：关注 Token 生成阶段。利用小模型快速生成草稿，大模型验证。优化的是 **Token/sec（生成速度）**。
    *   **Speculative Execution**：关注工具使用阶段。在 LLM 决定调用工具前提前执行查询。优化的是 **TTFT (Time To First Token) 或 任务完成总耗时**。
</details>

**Q2. 安全边界**
以下哪些操作适合进行“投机执行”？为什么？
1. 查询天气 API
2. 这里的用户名为 "admin"，尝试删除用户
3. 在 RAG 向量库中检索文档
4. 发送一条 Slack 消息给老板
5. 预加载网页内容的 Summary

<details>
<summary>点击查看提示与答案</summary>

*   **提示**：考虑操作是否是“只读”的，是否有不可逆的副作用。
*   **答案**：
    *   **适合**：1, 3, 5。因为它们通常是只读的（Read-only）或幂等的（Idempotent），即使猜错了，除了浪费一点带宽和算力，不会破坏系统状态。
    *   **不适合**：2, 4。删除数据和发送消息具有明显的副作用（Side Effects）。一旦投机执行了但用户本意并非如此，后果无法撤回。
</details>

**Q3. FRP 算子选择**
你需要实现一个逻辑：同时向 Google Search, Bing Search, 和 DuckDuckGo 发起相同的查询，**只要有一个**返回了结果，就立即使用该结果，并**取消**另外两个请求。应该使用哪种 FRP 模式？

<details>
<summary>点击查看提示与答案</summary>

*   **提示**：这是一个赛跑游戏。
*   **答案**：使用 `race` (或 `amb`) 算子。
    *   `result$ = race(google$, bing$, ddg$)`
    *   当第一个流发射据时，`race` 会自动取消订阅其他未完成的流（前提是流的实现支持 cancellation，如 AbortController）。
</details>

### 挑战题

**Q4. UI 闪烁问题**
在使用 Speculative Decoding 时，小模型生成的草稿可能在 100ms 后被大模型拒绝并重写。如果直接将流接入 UI，用户会看到文字疯狂跳变。请设计一个基于 FRP 的 UI 渲染策略来减轻这种不适感。

<details>
<summary>点击查看提示与答案</summary>

*   **提示**：不要把所有数据都视为“已确认”。引入“状态”区分。
*   **答案**：
    1.  **数据结构分离**：Event 流中包含 `VerifiedText` 和 `SpeculativeText`。
    2.  **UI 样式区分**：`VerifiedText` 显示为黑色实体；`SpeculativeText` 显示为灰色（Ghost text）或带有光标闪烁效果。
    3.  **Buffer/Debounce**：在视觉上，可以引入极短的 `buffer`（如 50ms），只有当草稿长度超过 N 个字符或置信度极高时才上屏，减少微小的动。
    4.  **平滑过渡**：当投机失败回滚时，使用 CSS 动画渐变消失，而不是生硬的文本替换。
</details>

**Q5. 成本计算器**
假设大模型推理成本为 $C_{big}$/token，小模型为 $C_{small}$/token。投机解码的接受率（Acceptance Rate）为 $\alpha$ (0~1)。草稿长度为 $K$。请给出一个简化的公式，判断在什么情况下开启投机解码是**省钱**的？（注：通常投机解码是为了速度，不一定省钱，但这里我们只讨论成本）。

<details>
<summary>点击查看提示与答案</summary>

*   **提示**：比较（纯大模型）与（小模型 + 大模型验证）的期望成本。注意验证过程通常是一次性并行处理 $K$ 个 token。
*   **答案**：
    *   **纯大模型成本**：生成 1 个 token 期望成本 $\approx C_{big}$
    *   **投机模式成本**：生成 1 个有效 token，我们需要跑一次小模型生成 $K$ 个草稿（成本 $K \cdot C_{small}$），然后大模型并行验证这 $K$ 个（成本 $\approx C_{big}$，假设并行验证成本接近一次推理）。这 $K$ 个草稿中，期望有 $\alpha \cdot K$ 个被接受。
    *   **平均每个有效 Token 的成本**：$\frac{C_{big} + K \cdot C_{small}}{\alpha \cdot K}$
    *   **省钱条件**：$\frac{C_{big} + K \cdot C_{small}}{\alpha \cdot K} < C_{big}$
    *   化简得：$\alpha > \frac{1}{K} + \frac{C_{small}}{C_{big}}$
    *   **结论**：如果接受率 $\alpha$ 足够高，且小模型足够便宜，才可能在省钱的同时提速。否则，通常是**花钱买时间**。
</details>

**Q6. 动态投机策略**
设计一个 FRP 管道，根据当前的“系统负载”动态调整投机策略。如果是高峰期（Load High），关闭投机以节省算力；如果是低谷期，开启投机以优化延迟。

<details>
<summary>点击查看提示与答案</summary>

*   **提示**：使用 `combineLatest` 将配置流与业务流结合。
*   **答案**：
    ```text
    LoadSignal = Monitor.cpuLoad$() // 0.0 ~ 1.0
    ConfigSignal = LoadSignal.map(load => load > 0.8 ? "OFF" : "ON")

    UserRequest$
      .withLatestFrom(ConfigSignal)
      .flatMap(([req, mode]) => {
         if (mode === "ON") {
            return race(speculativePath(req), standardPath(req));
         } else {
            return standardPath(req);
         }
      })
    ```
    *   利用 `withLatestFrom` 读取当前系统状态，动态决定走哪条逻辑分支。
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 陷阱：对副作用（Side-effects）进行投机
**错误**：用户输入“帮我订一张机票...”，系统预测用户大概率会订，于是提前调用了 `BookingAPI.createOrder()`。
**后果**：用户还没确认，钱就扣了，或者订单生成了。
**修正**：**Read/Write 分离**。只能对 `searchFlights`（查航班）进行投机，绝不能对 `bookFlight`（订航班）进行投机。在 Tool 定义中增加 `isSafe` 或 `idempotent` 标记。

### 5.2 陷阱：缓存中毒 (Cache Poisoning)
**错误**：投机执行的结果被写入了全局缓存，但这个结果是基于错误的假设（例如假设用户在“北京”，实际上用户在“南京”）。虽然 UI 上修正了，但后续请求可能读到错误的缓存。
**修正**：投机产生的数据应标记为 `tainted` 或 `speculative`，存储在临时的、请求级别的 Scope 中，只有当被“采纳（Accepted）”后，才提升为全局缓存。

### 5.3 陷阱：取消失效 (Cancellation Leak)
**错误**：FRP 逻辑中发出了取消信号（Unsubscribe），但底层的 Promise 或 HTTP Client 不支持 AbortSignal。
**后果**：后台依然在跑繁重的任务，虽然前台不显示结果，但服务器资源被耗尽（Resource Exhaustion），导致“拒绝服务（DoS）”。
**修正**：确保所有 Async Effect 都绑定 `AbortController`。在 FRP 算子的 `dispose/teardown` 阶段必须触发 `abort()`。

### 5.4 陷阱：过度投机导致吞吐量下降
**错误**：为了致的单用户延迟，对每个 Token 都做 5 路并发投机。
**后果**：并发数爆炸，导致整个系统的吞吐量（Throughput）断崖式下跌，所有用户的请求都变慢了。
**调试技巧**：监控 **Speculation Efficiency（投机效率）** = (采纳的 Token 数 / 生成的总 Token 数)。如果效率低于 30%，说明在做无用功，应通过 Token Bucket 限制并发。
