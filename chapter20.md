# Appendix C｜测试方法：虚时间、回放与混沌工程

## 1. 开篇段落

在构建实时 LLM Agent 时，开发者通常面临三大测试噩梦：**非确定性**（LLM 的输出每次可能不同）、**时间依赖性**（超时、去抖动、竞态条件难以复现）以及**外部副作用**（API 调用既慢又费钱）。传统的单元测试（Mock/Stub）往往难以覆盖复杂的异步流交互。

FRP 架构的最大优势之一，就是能将“时间”抽象为可操控的变量。本章将介绍如何利用 FRP 的特性建立一套坚固的测试体系：利用**虚拟时间（Virtual Time）**瞬间完成长时序逻辑的测试，利用**事件回放（Event Replay）**实现 100% 的确定性回归，以及通过**流式混沌工（Stream-based Chaos）**主动注入故障以验证系统的鲁棒性。

**学习目标**：
- 理解并掌握虚拟时间调度器（Virtual Scheduler）的原理与应用。
- 学会设计基于“事件日志（Event Log）”的确定性回放系统。
- 掌握如何在事件流中注入延迟、错误和乱序，进行混沌测试。
- 建立 Agent 性能基准测试的指标体系。

---

## 2. 文字论述

### C.1 虚拟时间驱动的测试 (Virtual Time Driven Testing)

在真实世界中，测试一个“用户停止输入 5 秒后触发自动保存”的逻辑，你需要真的等 5 秒。这使得测试套件运行缓慢且脆弱。

在 FRP 中，所有的算子（debounce, throttle, delay, interval）都依赖于一个`Scheduler`（调度器）。通过替换底层的调度器，我们可以接管时间。

#### 概念模型：上帝视角的时间轴

虚拟时间调度器就像电影剪辑软件的时间轴游标。你可以随意将游标拖动到 `t=5000ms` 的位置，而不需要实地流逝 5 秒。

```text
[真实时间调度]
Wall Clock: 0s .....(waiting)..... 5s -> 触发 Action
Test Time:  5s+ (测试非常慢)

[虚拟时间调度]
Logical T:  0s -> (jump) -> 5000ms -> 触发 Action
Test Time:  <1ms (瞬间完成)
```

#### Rule of Thumb: 依赖注入时间
> **法则**：永远不要在 Agent 的核心逻辑中直接调用 `System.currentTimeMillis()` 或 `setTimeout`。应该将 `Clock` 和 `Scheduler` 作为依赖项注入（Dependency Injection）。在生产环境注入 `RealScheduler`，在测试环境注入 `TestScheduler`。

#### 应用场景
1.  **超时测试**：模拟工具调用 30秒无响应，触发 fallback 逻辑。在虚拟时间中，这只是推进一步时钟。
2.  **Debounce/Throttle 验证**：精确验证用户在第 10ms、200ms、550ms 输入字符时，debounce(500ms) 到底会在哪个时刻触发。
3.  **长周期任务**：测试“每天早上 8 点发送问候”，不需要修改系统时间，只需将虚拟时钟拨动 24 小时。

### C.2 事件回放测试 (Event Replay Testing)

Agent 的状态是由事件流驱动的。根据 FRP 的公式 `State = reduce(Events, InitialState)`，只要**初始状态一致**且**事件序列一致**，**最终状态必然一致**。这是构建“时光机”调试器的基础。

#### 挑战：LLM 的非确定性
虽然代码逻辑是确定性的，但 LLM 的输出（Effect）不是。如果直接重放用户输入，LLM 可能会生成不同的回复，导致后续逻辑分叉。

#### 解决方案：记录“Effect 结果”而非“重新执行 Effect”
在录制模式下，我们需要记录两类事件：
1.  **Inbound Events**：用户的输入、系统信号。
2.  **Effect Completion Events**：LLM 的回复、API 的返回结果、DB 的查询结果。

在回放模式下，我们**禁掉**真实的副作用执行器（Effect Runner），改为从日志中读取“Effect Completion Events”并注入到流中。

```text
[录制阶段 (Recording)]
User Input --> [Logic] --> Call LLM (Effect) --> (Real LLM API)
                                                      |
                                                      v
Log File: [ {t:10, type:User, data:"Hi"}, {t:200, type:LLM_Res, data:"Hello"} ]

[回放阶段 (Replaying)]
Log File --> [Replay Engine] --(inject t:10)--> [Logic]
                                                  |
             (Skip Real API) <--- [Logic] <-------+
                                                  |
Log File --(inject t:200)-------------------------+-> [Logic] --> Verify State
```

#### Rule of Thumb: 边界隔离
> **法则**：回放测试的边界应划定在“纯逻辑”与“副作用”之间。凡是涉及 IO（网络、磁盘、随机数）的操作，在回放时都应视为“外部输入事件”进行 Mock 或重放。

### C.3 混沌工程与流式故障注入 (Stream-based Chaos)

分布式系统中最难调试的是**部分失败**和**时序异常**。FRP 的流式特性使得我们可以像搭积木一样，在数据流管道中插入捣乱”的算子。

#### 常见的混沌算子 (Chaos Operators)

1.  **延迟注入 (Latency Injection)**：
    在工具调用的响应流上加一个 `delay(random(100ms, 5000ms))`。
    *目标*：验证 UI 是否展示 Loading 状态，竞态处理（switchLatest）是否正确丢弃了过期的包。

2.  **异常注入 (Error Injection)**：
    随机将流中的正常数据映射为 `Error Event`。
    *目标*：验证 Retry 策略（指数退避）、Circuit Breaker（熔断）以及错误提示 UI。

3.  **乱序注入 (Reordering/Shuffling)**：
    利用 buffer 收集一段时间的事件，打乱顺序后再发出。
    *目标*：验证系统是否依赖消息到达的物理顺序，是否正确使用 `sequenceId` 或逻辑时钟进行排序。

4.  **背压风暴 (Backpressure Storm)**：
    以极高频率发出事件（模拟热点点击或死循环）。
    *目标*：验证 `throttle`、`debounce` 或 Token Bucket 限流器是否生效，确保内存不泄漏。

### C.4 能基准 (Performance Benchmarking)

针对实时 Agent，性能不仅仅是 RPS（Requests Per Second），更多关乎用户体验指标。

| 指标 | 说明 | FRP 关注点 |
| :--- | :--- | :--- |
| **TTFT (Time To First Token)** | 从用户动作结束到首个字符上屏的时间 | 测量 Pipeline 中各 Stage 的耗时分布 |
| **End-to-End Latency** | 完成整个任务（含工具调用）的总耗时 | 观察并发算子（merge/zip）是否有效并行化 |
| **Token Throughput** | 生成速度 (tokens/sec) | 监控流式处理是否存在阻塞主线程的情况 |
| **Jitter (抖动)** | 响应时间的方差 | 确保 GC 或长任务不会造成 UI 卡顿（丢帧） |

---

## 3. 本章小结

1.  **虚拟时间**是测试异步逻辑的“银弹”，它将物理时间解耦，使测试瞬间完成且结果确定。
2.  **确定性回放**要求我们将“副作用的结果”视为输入流的一部分进行录制，从而剥离 LLM 的随机性。
3.  **混沌测试**在 FRP 中非常易实现，只需在流上组合故障算子（Delay, Error, Shuffle）。
4.  测试不仅是为了找 Bug，更是为了确立**性能基线**和**一致性保证**，这对于复杂的实时 Agent 至关重要。

---

## 4. 练习题

### 基础题

**Q1. 虚拟时间的基本原理**
为什么在单元测试中使用 `VirtualScheduler` 比使用 `sleep()` 或 `setTimeout` 更好？请列举至少三个理由。

**Q2. 回放与副作用**
在进行回放测试时，Agent 触发了一个“查询天气”的工具调用（Tool Call）。在回放模式下，是否应该真实发送 HTTP 请求去查询天气 API？为什么？

**Q3. 混沌算子**
如果你想测试 Agent 在网络极不稳定的情况下（丢包率 20%）的表现，你应该在 FRP 的哪个环节插入什么逻辑？

**Q4. 指标定义**
对于一个流式输出的 Agent，为什么 TTFT (Time To First Token) 比总生成时间更影响用户的主观等待体验？

### 挑战题

**Q5. 解决“当前时间”的悖论 (思题)**
假设 Agent 的 Prompt 中包含：“现在是 {Current_Time}，请回答...”。
如果在 2024年1月1日 录制了一段对话，而在 2025年1月1日 进行回放测试。
如果不做处理，Prompt 中的时间变了，LLM 的回答可能完全不同（例如询问“今天是星期几”）。
请利用“依赖注入”和“Prompt Patching”的概念，设计一个方案，确保回放时 LLM 看到的“当前时间”依然是 2024年1月1日。

**Q6. 竞态条件重现**
设计一个测试用例，利用虚拟时间重现以下 Bug：
用户快速发送 "A" (t=0) 和 "B" (t=100)。
LLM 针对 "A" 的处理很慢，在 t=500 返回结果。
LLM 针对 "B" 的处理很快，在 t=300 返回结果。
如果系统没有正确使用 `switchLatest`，UI 可能会先显示 B 的结果，然后被 A 的结果覆盖（错误）。
请用伪代码描述如何在测试中编排这个时序。

---

### 练习题参考答案

<details>
<summary><strong>点击展开答案</strong></summary>

**A1. 虚拟时间的优势**
1.  **速度**：无需真实等待，测试运行速度提升数千倍。
2.  **确定性**：消除了系统负载、OS 调度导致的微小时间误差，不仅解决 flaky tests，还能精确断言毫秒级逻辑。
3.  **可控性**：可以模拟极其罕见的时序边界情况（如两个事件在同一毫秒发生）。

**A2. 回放中的副作用**
不应该。
1.  **确定性破坏**：真实 API 返回的天气可能变了，导致 Agent 后续行为与录制时不一致，回放失败。
2.  **成本与速度**：真实调用慢且可能产生费用。
3.  **副作用风险**：如果工具是“删除文件”或“发送邮件”，回放不应再次执行破坏性操作。
应该是从日志中读取上次录制的“API 返回结果”并注入系统。

**A3. 丢包模拟**
在 Agent 接收外部信号的 Input Stream 或工具返回的 Response Stream 上，插入一个 `filter` 算子。
逻辑：`stream.filter(() => Math.random() > 0.2)`。
这会随机丢弃 20% 的事件，从而测试系统的超时重试或容错机制。

**A4. TTFT 的重要性**
因为 LLM 生成通常较慢。TTFT 决定了用户“感觉到系统开始响应”的时间。一旦首字出现，用户就会开始阅读，后续生成的延迟会被阅读时间掩盖。如果 TTFT 长，用户会以为系统死机或网络断开。

**A5. 冻结时间的方案**
1.  **抽象 Clock**：定义一个 `IClock` 接口，包含 `now()` 方法。
2.  **Prompt 构造**：Prompt 中的时间不直接取 `Date.now()`，而是取 `IClock.now()`。
3.  **录制时**：记录下 Session 开始时的绝对时间戳 $T_0$。
4.  **回放时**：注入一个 `FixedClock` 或 `OffsetClock`，使其返回 $T_0$ (或者 $T_0 + \text{elapsed\_virtual\_time}$)。
这样，Prompt 里的字符串构建在回放时与录制时完全二进制一致。

**A6. 竞态重现伪代码**
```text
// 伪代码
setup:
  scheduler = new VirtualScheduler()
  results = []
  agent = new Agent(scheduler)
  agent.outputStream.subscribe(r => results.push(r))

test:
  // 模拟输入流
  scheduler.schedule(t=0,   () => agent.input("A"))
  scheduler.schedule(t=100, () => agent.input("B"))

  // 模拟 LLM 响应流 (Mock)
  // 这里的关键是：针对 A 的响应比 B 晚
  mockLLM.onRequest("A").delay(500).respond("Result A")
  mockLLM.onRequest("B").delay(200).respond("Result B") // B 会在 t=100+200=300 返回

  // 执行虚拟时间到 t=600
  scheduler.advanceTo(600)

  // 断言
  // 如果正确使用了 switchLatest，"Result A" 应该被丢弃或忽略
  // 最终状态应该是 "Result B"
  // 如果 results 包含 ["Result B", "Result A"]，则测试失败（发生了回退）
  assert(results.last() == "Result B")
  assert(!results.includes("Result A")) // 视具体实现，A 可能根本不应该推送到流中
```

</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 1. 混淆 Microtasks 和 Macrotasks
在 JavaScript/Node.js 环境中，Promise (Microtask) 和 `setTimeout` (Macrotask) 的优先级不同。
**陷阱**：在使用虚拟时间调度器时，如果你的代码混合使用了 FRP 的 `delay`（通常由调度器控制）和原生的 `Promise.resolve`，可能会导致执行顺序与预期不符。
**对策**：尽量将所有异步操作通过 FRP 的 `fromPromise` 或 `observeOn(scheduler)` 纳管，确保统一由调度器排序。

### 2. 测试中的“副作用泄漏”
**陷阱**：在编写测试时，忘记 Mock 某个日志上报接口或分析埋点。虽然测试通过了，但每次跑单元测试都会向你的分析服务器发送垃圾数据。
**对策**：在测试环境的 `teardown` 阶段检查是否有未被 Mock 的网络请求发出（许多测试框架提供此功能）。

### 3. 随机数种子未固定
**陷阱**：Agent 内部逻辑可能使用了 `Math.random()`（例如用于负载均衡或随机问候语）。这会导致即使是回放测试也偶尔失败。
**对策**：在测试环境中，Mock 全局的随机数生成器，或者使用带有固定 Seed 的随机数生成器。

### 4. 忽略了取消（Cancellation）的验证
**陷阱**：测试往往关注“成功流程”。很容易忘记测试“用户在 LLM 输出一半时点击取消”的场景。如果不测试，可能会导致后台 goroutine/Promise 泄漏，或者 Token 预算被默默耗尽。
**对策**：在混沌测试中专门加入“随机取消”的机器猴子。

### 5. Log 文件过大
**陷阱**：全量录制所有事件（包含每一帧的 Token 变化、每一帧的 UI 渲染事件）会导致回放日志体积爆炸。
**对策**：仅录制“关键帧”事件（输入、最终输出、工具回执）。中间状态可以通过重放逻辑计算得出。
