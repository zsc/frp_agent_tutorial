# Chapter 9｜Dynamic Batching & 并发：吞吐、延迟与公平性

## 1. 开篇段落

在构建生产级 LLM Agent 时，我们很快会撞上一堵墙：**资源与效率的矛盾**。
单用户的 Demo 往往运行流畅，但当系统扩展，或者单个 Agent 需要在一次交互中调用 5 个工具、检索 20 个文档片段、并同时生成回复时，串行处理会导致延迟（Latency）爆炸。反之，如果不加节制地并发，不仅会瞬间耗尽 API Rate Limit（速率限制），还会因为过多的并发连接导致客户端崩溃。

本章的核心任务是构建一个**智能流量调度层**。我们将利用 FRP 的流式特性，实现以下目标：
1.  **动态批处理（Dynamic Batching）**：像“拼车”一样，自动合并碎片化请求，利用 LLM/Embedding API 的批量理能力提升吞吐（Throughput）。
2.  **并发控制（Concurrency Control）**：限制同时飞行的请求数量，防止系统过载。
3.  **优先级与公平性（Fairness）**：确保后台的“长考”任务不会阻塞用户的“秒回”需求。
4.  **背压（Backpressure）**：当系统处理不过来时，优雅地丢弃或排队，而不是崩溃。

---

## 2. 核心论述

### 2.1 批处理的对象：不仅是 Token

在 Agent 系统中，凡是涉及 **I/O** 和 **昂贵计算** 的环节，都是批处理的潜在客户。我们需要识别出哪些流（Stream）是可以合并的。

| 场景 | 是否适合 Batch | 理由 | 典型策略 |
| :--- | :--- | :--- | :--- |
| **Embeddings** | ⭐⭐⭐⭐⭐ (极佳) | 向量数据库和 Embedding API 对批量极度友好。100 条文本发 1 次请求比发 100 次快得多且省配额。 | `bufferTime(50ms)` 或 `bufferCount(20)` |
| **Log/Trace** | ⭐⭐⭐⭐⭐ (极佳) | 写入 DB 或观测平台。实时性要求低吞吐要求高。 | `bufferTime(5s)` 或 `bufferCount(100)` |
| **Tool Calls** | ⭐⭐⭐ (中等) | 若多个工具之间无依赖（如同时查天气和汇率），应并发或合并请求（如果 API 支持）。 | `zip` 或 `forkJoin` (并发) |
| **Chat Compl** | ⭐ (较差) | 对话通常是线性的。Batch 主要用于 "Speculative Execution"（并发生成多个草稿）。 | 主要是并发而非合并 |
| **UI Updates** | ⭐⭐ (特殊) | 避免每生成一个 token 就重绘 DOM。 | `animationFrame` 节流 |

### 2.2 动态批处理策略：FRP 的时间魔法

FRP 的核心优势在于处理“时间窗口”。我们不需要写复杂的 `setTimeout` 和数组操作，而是使用算子组合来定义“发车机制”。

#### 策略 A：时间与数量的竞争 (Time-Size Race)
这是最通用的策略。
> **Rule of Thumb**: "凑齐 N 个就发车，或者等了 T 毫秒还没凑齐也发车。"

在 FRP 中，这通常表现为 `bufferTime` 和 `bufferCount` 的组合，或者更层的 `window` 操作。

```ascii
Stream (Inputs):  --a---b-------c--d--e--f--g--------->
                    |   |       |  |  |  |  |
Batcher Logic:    [ Wait max 50ms OR max 3 items ]
                    |           |        |
                    v           v        v
Output (Batches): --[a,b]-------[c,d,e]--[f,g]-------->
Reason:           Timeout     Full     Timeout
```

#### 策略 B：自适应窗口 (Adaptive Window)
静态窗口（如固定 50ms）在流量波动时不够从容。
*   **低负载时**：我们希望窗口为 0ms，即时响应。
*   **高负载时**：我们希望窗口延长（如 200ms），以提高“拼车率”，减少 API 调用次数。

在 FRP 中，这可以通过**背压反馈**实现：监测当前飞行中（In-flight）的请求数。如果飞行数很少，说明系统空闲，立刻发送；如果飞行数很多，说明系统拥堵，自动增大 buffer 时间。

### 2.3 优先级调度：多车道模型

并非所有事件都生而平等。
*   **P0 (User Interactive)**: 用户正在盯着屏幕，必须 <200ms 响应。
*   **P1 (Reasoning)**: 模型中间思考过程，稍慢可接受。
*   **P2 (Background)**: 异步的记忆整理、日志上传。

如果我们简单地把所有请求塞进同一个 Batch 队列，P2 的大量日志可能会阻塞 P0 的用户请求（Head-of-Line Blocking）。

**设计模式：多级优先级队列 + 抢占式填充**

```ascii
High Priority Stream ($H): --H1--H2----------------H3---->
Low Priority Stream ($L):  --L1--L2--L3--L4--L5--L6------->

[ Scheduler Logic ]
1. Check $H. If distinct items exist, flush immediately.
2. If Batch has space left, fill with items from $L.
3. If $H is empty but $L is full, flush $L.

Output Batch Stream:
Batch 1: [H1, H2, L1, L2] (H triggered flush, L piggybacked)
Batch 2: [L3, L4, L5, L6] (L triggered flush by size/time)
Batch 3: [H3]             (H triggered flush)
```

这种模式在 FRP 中通常通过合并多个流（`merge`），但给每个事件打上 `priority` 标签，并在 buffer 的 closing selector 中加入逻辑来实现。

### 2.4 背压与流控 (Backpressure)

当 LLM 的生成速度只有 50 tokens/s，而工具产生数据的速度是 500 items/s 时，系统必须进行流控，否则内存会爆（OOM）。

FRP 提供了三大流控原语，对应不同的业务场景：

1.  **Lossy (有损) - Throttle / Debounce / Audit**
    *   **场景**：用户快速拖动进度条、VAD 音量检测、UI 渲染。
    *   **逻辑**：只保留时间窗口内的第一个或最后一个，丢弃中间的。
    *   **代价**：丢失中间状态。

2.  **Lossless (无损) - Buffer / Queue**
    *   **场景**：日志、计费 Token 统计。
    *   **逻辑**：全部存入内存队列，慢慢消费。
    *   **风险**：如果生产持续 > 消费，队列无限增长导致 Crash。需要设置 **High Watermark（高水位）**，超过水位则强制丢弃或报错。

3.  **Switching (切换) - switchLatest**
    *   **场景**：RAG 检索、对话生成
    *   **逻辑**：**"喜新厌旧"**。当新的请求（Intent）到来时，如果旧的请求还没处理完，直接**取消**旧请求。
    *   **价值**：这是 Agent 响应速度感知的关键。用户改口时，Agent 应立即停止上一句的推理。

### 2.5 并发限制 (Concurrency Limiting / Bulkhead)

为了防止 Agent 将后端服务（如 Docker 容器、数据库连接池）打挂，我们需要限制并发度。
FRP 的 `mergeMap` (或 `flatMap`) 算子通常带有一个 `concurrency` 参数。

*   `mergeMap(handler, concurrency=5)`: 同时最多处理 5 个内部流。第 6 个事件到来时，会被暂时挂起（Pending），直到前 5 个中有 1 个完成。

**舱壁模式 (Bulkhead Pattern)**：
不要让一个耗时的工具（如图像生成）占满所有并发槽位，导致轻量级的工具（如计算器）无法执行。
*   应该为不同类型的任务建立独立的“泳道”：
    *   `ImageGenStream`: maxConcurrency = 1
    *   `WebSearchStream`: maxConcurrency = 5
    *   `CodeExecStream`: maxConcurrency = 2

---

## 3. 本章小结

1.  **Batching 是 IO 密集型任务的救星**：对 Embedding 和 Log 务必使用 Batch，对 LLM 推理慎用。
2.  **FRP 是实现 Batching 的最佳 DSL**：通过 `buffer`、`window`、`debounce` 等算子的声明式组合，替代了复杂的定时器状态机。
3.  **优先级队列**：利用“顺风车”机制，让低优先级任务利用高优先级任务的 Batch 剩余空间，既保证了延迟又提升了吞吐。
4.  **背压显性化**：不要假设系统能处理无限速率的输入。明确你的 Agent 在过载时是选择“丢弃”、“排队”还是“覆盖”。
5.  **隔离并发**：使用舱壁模式（Bulkhead）防止某个慢工具拖垮整个 Agent 的响应能力。

---

## 4. 练习题

### 基础题

**Q1. 算子行为预测**
给定输入流事件（时间点:值）：`0ms:A`, `10ms:B`, `40ms:C`, `60ms:D`, `90ms:E`。
请描述以下三种配置的输出 Batch：
1.  `bufferTime(50ms)`
2.  `bufferCount(2)`
3.  `buffer(debounceTime(30ms))` (提示：debounce 在静默 30ms 后触发)

<details>
<summary><strong>参考答案</strong></summary>

1.  **bufferTime(50ms)**:
    *   T=50ms: 输出 `[A, B, C]` (0, 10, 40 都在 0-50 区间)
    *   T=100ms: 输出 `[D, E]` (60, 90 都在 50-100 区间)
2.  **bufferCount(2)**:
    *   T=10ms (B到达): 输出 `[A, B]`
    *   T=60ms (D到达): 输出 `[C, D]`
    *   E 还在缓冲区等待下一个伙伴。
3.  **buffer(debounceTime(30ms))**:
    *   A(0) -> wait 30? No, B(10) came. Reset timer.
    *   B(10) -> wait 30? Yes, at 40ms C came. Reset timer? (注意：Debounce 行为取决于具体实现，通常 C 的到来会重置 B 的计时)。
    *   假设标准 debounce：
        *   A(0)..B(10)..C(40)..D(60)..E(90).. -> 120ms (90+30) 没有任何新事件。
        *   输出 `[A, B, C, D, E]` 在 T=120ms。这是一个陷阱题，debounce 适合做“输入停止检测”，如果不停止，它会一直攒着。

</details>

**Q2. 场景选择**
在以下场景中，你会选择 Drop (丢弃)、Queue (排队) 还是 Latest (最新) 策略？
1.  Agent 正在朗读长文本，用户不停点击“下一句”按钮。
2.  Agent 接收股票市场的实时价格流（每秒 100 次更新），每 5 秒做一次分析。
3.  Agent 将用户的操作审计日志发送到合规服务器，网络断开了 1 分钟。

<details>
<summary><strong>参考答案</strong></summary>

1.  **Latest**: 用户只关心跳转到最新的位置，中间的点击如果处理不过来应忽略，但最后一次点击必须生效。
2.  **Latest (Sample)**: 每 5 秒取样一次最新的价格即可，中间的价格波动对于 5 秒粒度的分析可能不重要（除非要做高频交易，那是另一回事）。
3.  **Queue**: 审计日志绝不能丢。必须在本地排队（持久化更好），等待网络恢复后重放。

</details>

**Q3. 简单的并发限制**
如果你的 Python Sandbox 只能同时运行 1 个代码块，但 Agent 逻辑中并发发出了 3 个代码执行请求。使用 `mergeMap(execute, concurrency=1)` 会发生什么？会报错吗？

<details>
<summary><strong>参考答案</strong></summary>

不会报错。
请求 1 会立即执行。
请求 2 和 3 会进入“挂起（Pending）”状态，也就是排队。
一旦请求 1 完成，请求 2 自动开始执行。
这正是 FRP 处理资源争用的优雅之处。

</details>

---

### 挑战题

**Q4. 设计：Embedding 优化器 (The Smart Batcher)**
设计一个 Batching 逻辑，满足以下苛刻条件：
1.  **成本控制**：API 调用次数越少越好（尽量凑满 20 条）。
2.  **延迟控制**：任何一条数据进入缓冲区后，等待时间不得超过 100ms。
3.  **容量限制**：整个 Batch 的 Token 总数不能超过 8192（即使条数没满 20 条，Token 满了也要发）。

*提示：你需要维护一个累加状态（scan）。*

<details>
<summary><strong>提示与参考思路</strong></summary>

**思路**：
这不简单的 `bufferTime` 或 `bufferCount` 能解决的。你需要自定义一个 **Accumulator (累加器)**。

1.  源数据流 `ItemStream`。
2.  使用 `scan` 算子维护当前 Batch 的状态：`{ count: 0, tokens: 0, items: [], firstItemTime: null }`。
3.  对于每个新 Item：
    *   检查 `tokens + newItem.tokens > 8192`? 如果是 -> 触发 Emit，重置状态。
    *   检查 `count + 1 >= 20`? 如果是 -> 触发 Emit，重置状态。
    *   如果是 Batch 的第一个元素，开启一个 100ms 的 Timer。
4.  Timer 到期时 -> 触发 Emit（如果 Batch 非空）。
5.  这个逻辑在 FRP 中通常可以用 `window` 配合自定义的 Closing Notifier 来实现，或者编写一个自定义的 `Operator`。

</details>

**Q5. 错误处理的“连坐”与隔离**
你做了一个 Batcher，把 10 个用户的 Prompt 合并发送给 LLM。结果其中 1 个用户的 Prompt 包含违禁词，导致 API 返回 400 Error，整个 Batch 失败。
请设计一个**自动降级与恢**流程，使得其他 9 个用户不受影响。

<details>
<summary><strong>参考答案</strong></summary>

**“分治重试”策略 (Divide and Conquer Retry)**：

1.  **捕获错误**：当 Batch 请求收到 400 错误时，不要透传给下游。
2.  **拆分 (Split)**：将失败的 Batch（例如 10 个）拆分为两个小 Batch（5 + 5）或者退化为 10 个单独的请求（Singular fallback）。
3.  **重试 (Retry)**：重新发送这些拆分后的请求。
4.  **递归**：如果 5 个的 Batch 还是失败，继续拆分，直到找到那个唯一的“坏苹果”。
5.  **结果缝合**：将成功的 9 个结果分别回传给对应的 Subject/Promise，将那个失败的 1 个结果标记为 Error。
*这在 FRP 中可以通过 `catchError` + `expand` (递归) 或 `switchMap` 里的重试逻辑来实现。*

</details>

**Q6. 竞态条件：Speculative Execution 的撤销**
Agent 预测用户可能需要“查询天气”，因此在用户还在打字时就提前发起了天气查询（Speculative）。
1.  Stream A: 用户打字流。
2.  Stream B: 投机触发的工具调用流。
3.  情况：用户打字结束，明确说“我不需要天气，我要听歌”。
4.  此时工具调用（HTTP请求）已经发出但未返回。
如何利用 FRP 的特性，确保天气查询的结果返回时，**不会** 消耗 Token 去进行下一步的 LLM 处理，也不会展示在 UI 上？

<details>
<summary><strong>参考答案</strong></summary>

这是 `switchLatest` (或 `switchMap`) 的经典用例。

*   **架构**：
    `UserIntentStream` -> `switchMap(intent => executeTool(intent))`
*   **流程**：
    1.  Intent A (可能查天气) 发出 -> `switchMap` 订阅工具调用 Observable。
    2.  Intent B (听歌) 发出 -> `switchMap` **立即取消** (Unsubscribe) Intent A 的内部 Observable。
*   **底层**：
    Observable 的 Teardown Logic（清理逻辑）会被触发。
    *   如果 HTTP 请求库支持 `AbortSignal`，则网络请求会被 Abort。
    *   使网络请求无法 Abort（服务器已处理），客户端的回调函数永远不会被调用，后续的流程（LLM 处理、UI 渲染）自然被切断。

</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 1. 僵尸请求 (Zombie Requests)
*   **现象**：UI 已经切换了页面，但后台还在疯狂请求 API，且回来后报错“Component destroyed”。
*   **原因**：FRP 的流必须显式管理生命周期。组件销毁时没有 `unsubscribe` 或发送 `complete` 信号。
*   **调试**：检查 Pending 请求数是否随着页面刷新单调增加。

### 2. 只有 Buffer 没有 Flush
*   **现象**：日志系统永远少最后几条日志。
*   **原因**：使用了 `bufferCount(10)`，结果最后只积攒了 7 条，程序就退出了。
*   **对策**：必须配合 `bufferTime` 使用（混合策略），或者在程序退出钩子（Shutdown Hook）中强制 `flush` 所有缓冲区。

### 3. "长尾"阻塞 (Head-of-Line Blocking)
*   **现象**：为了凑够 Batch，强行等待了 200ms，导致原本可以 50ms 完成的请求被迫延迟到 250ms+。
*   **对策**：区分链路。对于 Latency Sensitive 的链路（如首字生成），**禁用 Batch** 或设置极短的窗口（如 10ms）。Batch 主要用于 Throughput Sensitive 的链路（如 RAG 索引构建）。

### 4. 误用的 `merge` 导致乱序
*   **现象**：用户先发了 "Hi"，后发了 "Bye"。结果 Agent 先回了 "Bye" 的回复，后回了 "Hi" 的回复。
*   **原因**：使用了 `mergeMap` 处理 LLM 请求，且没做并发限制。"Hi" 的处理比 "Bye" 慢，导致结果乱序返回。
*   **对策**：对于有因果关系的对话流，必须使用 `concatMap`（严格串行）或者带 `orderId` 的重排序机制。
