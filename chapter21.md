# Appendix D｜部署与运维：配置、灰度、告警与容灾

## 1. 开篇段落

欢迎来到本书的“深水区”。在前文中，我们讨论了如何编写代码；而在本章，我们将讨论如何**运行**代码。

对于传统的无状态 RESTful 服务，运维通常意味着“CI/CD 管道”、“Kubernetes HPA”和“ELK 日志”。但对于一个**基于长连接（WebSocket/WebRTC）、状态敏感、极其依赖不稳定外部 API（LLM）、且每一毫秒都在烧钱**的 FRP Agent 系统，运维面临着全新的挑战：

1.  **配置即信号（Configuration as Signal）**：你不能每次调整 Prompt 或 Temperature 都重启服务，这会切用户的语音通话。配置必须是实时流动的。
2.  **版本矩阵爆炸**：Agent 的行为由“代码版本 + 模型版本 + Prompt 版本 + RAG 知识库版本”共同决定。灰度发布不仅仅是切流量，更是切“上下文”。
3.  **可观测性盲区**：CPU 和内存正常，并不代表 Agent 正常。它可能正在疯狂输出乱码（幻觉）或者陷入“你好-你好-你好”的死循环（Looping），这需要基于流的语义监控。
4.  **成本熔断**：如果代码写了个 Bug 导致死循环，传统的服务只是卡死，LLM Agent 则会让你在一夜之间破产。

本章将提供一套完整的“Day 2 Operations”指南，确保你的 Agent 不仅能跑通，还能在生产环境中稳健生存。

---

## 2. 核心论述

### 2.1 配置即信号 (Configuration as Signal)

在 FRP 架构中，**静态配置文件是反模式**。所有的配置（Prompt 模板、LLM 参数、工具开关、RAG 阈值）都应该被视为随时间变化的**Signal信号）**。

#### 2.1.1 配置的层级与合并
一个生产级 Agent 的配置通常是多层合并（Merge）的结果：
1.  **Global Signal**：全系统通用的，如 API Key 轮换、全局熔断开关。
2.  **Tenant/Group Signal**：租户级别的，如“VIP 用户使用 GPT-4，普通用户使用 GPT-3.5”。
3.  **Session Signal**：会话级别的，如用户刚才手动开启了“详细模式”。

```mermaid-ascii
[Etcd/Consul] --> [Config Source Stream]
                        |
                        v
                 +---------------+
                 | Global Config | (Base)
                 +---------------+
                        |
[User Profile DB] -> [Tenant Config] (Override)
                        |
[UI Toggle Event] -> [Session Config] (Override)
                        |
                        v
              [Merged Config Signal]  <-- 最终业务逻辑订阅此信号
```

#### 2.1.2 热更新与“配置屏障”
当配置在一次对话中间发生变化时，如何处理？
*   **立即生效**：如下一个 Token 的采样温度。
*   **下一轮生效**：如 System Prompt 的修改。如果在生成过程中修改 System Prompt，可能导致 LLM 精神分裂。

**FRP 模式：`sample` 与 `withLatestFrom`**
我们通常不在流的中间处理配置变化，而是在**事件触发的瞬间**对配置进行采样。

```text
// ❌ 错误：在闭包中读取外部变量，可能导致同一句话前半段和后半段逻辑不一致
process(input) {
   let cfg = globalConfig; // 不要这样做
   ...
}

// ✅ 正确：将配置视为输入流的“伴随数据”
contextStream = userInputStream.withLatestFrom(configSignal)
    .map(([input, config]) -> {
        return createExecutionContext(input, config); // 配置被快照（Snapshot）
    });
```

### 2.2 复杂的灰度发布 (Canary & Shadowing)

LLM Agent 的灰度不仅是流量的灰度，更是**状态的灰度**。

#### 2.2.1 连接耗尽 (Connection Draining)
你不能直接杀掉旧版本的 Pod。
1.  **标记下线**：将 v1 Pod 标记为 `Draining`。负载均衡器停止分发新连接。
2.  **软性超时**：对于超过 1 小时的长连接，发送“系统维护，请刷新”的 UI 事件，诱导用户重连。
3.  **硬性强制**：设置 4-12 小时的宽限期，之后强制断开。

#### 2.2.2 影子流量 (Shadow Traffic / Dark Launching)
如何测试一个新的 Prompt 是否会让 RAG 检索效果变差？直接上线风险太大。FRP 的**多播（Multicast）**特性是实现影子流量的神器。

```mermaid-ascii
User Input Stream
      |
      +-------+ (share/multicast)
      |       |
      v       v
 [Prod Flow] [Shadow Flow (v2 Prompt)]
      |       |
      |       +--> [LLM (Dry Run)] --> [Evaluator]
      |                                   |
      v                                   v
 [User Output]                     [Log Difference]
                               (Latency, Token Count, Embedding Distance)
```
*注意：影子流量会消耗双倍的 Token 成本，通常只对 1% - 5% 的流量开启。*

#### 2.2.3 版本矩阵 (The Version Matrix)
由于 Agent 依赖多个组件，发布时必须记录完整的版本指纹：
`BuildID = { Code: v1.2, Prompt: v3.5, Model: gpt-4-0613, KB_Index: 2024-01-05 }`
在排查“为什么昨天好用的 Agent 今天变傻了”时，这个指纹至关重要。

### 2.3 告警与 SLO：关注“流的健康”

监控传统的 QPS 和 Latency 不足以描述 Agent 的健康状况。我们需要监控**流的物理特性**和**语义特性**。

#### 2.3.1 物理层监控（基于时间与背压）
*   **TTFT (Time To First Token)**：首字延迟。这是用户感知的最重要指标。
    *   *SLO*: 95% 请求 TTFT < 1.5s。
*   **Stream Stalls (流停顿)**：在生成过程中，两个 Token 之间的间隔超过阈值（如 5s）。这通常意味着 LLM 推理卡顿或网络抖动。
*   **Backpressure Drop Rate**：由于系统处理不过来（如入库、UI 渲染），而在 buffer 中被丢弃的事件量。

#### 2.3.2 语义层监控（基于内容）
*   **Loop Detection (死循环检测)**：检测输出流中 n-gram 的重复率。如果 Agent 开始复读机，必须触发告警并熔断。
*   **Tool Error Burst**：如果工具调用连续失败率超过 20%，说明外部依赖挂了。
*   **Safety Filter Rate**：内容安全拦截率突增，可能意味着正在遭受注入攻击。

#### 2.3.3 成本层监控（Wallet Protection）
*   **Token Burn Velocity**：每分钟消耗的 Token 总量。
*   *告警*：如果 `CurrentVelocity > 3 * MovingAverage(1h)`，立即触发 **Cost Circuit Breaker**，暂停非 VIP 服务或降级到廉价模型。

### 2.4 容灾与降级策略 (Disaster Recovery)

当 OpenAI 挂了，或者私有模型显存爆了，Agent 必须“优雅地变笨”，而不是“直接死亡”。

#### 2.4.1 多级回退 (Multi-Level Fallback)
利用 FRP 的 `catchError` 和 `switch` 实现无缝降级。

```mermaid-ascii
[Plan A: GPT-4] --(Error/Timeout)--> [Plan B: GPT-3.5] --(Error)--> [Plan C: Rule-Based]
      |                                   |                             |
  (Best Quality)                    (Lower Quality)              ("Sorry I can't help")
```

#### 2.4.2 熔断器 (Circuit Breaker)
FRP 中的熔断器不仅仅是一个开关，它是一个**状态机转换器**。
*   **Closed**: 正常流转。
*   **Open**: 遇到错误阈值，流被切断，直接返回 `ErrorEvent`。
*   **Half-Open**: 定时器触发（如 60s 后），允许 1 个测试事件通过。如果成功，转为 Closed；失败则回到 Open。

#### 2.4.3 状态恢复 (State Hydration)
如果运行 Agent 的容器崩溃（OOM），K8s 会重启它。新的 Pod 必须能接管旧的 WebSocket 连接（如果是 Sticky Session）或让用户重连后无感。
*   **Redis as Hot Store**: 每处理完一个 Event，State 的增量变化（Delta）必须异步写入 Redis。
*   **Replay**: 新 Pod 启动时，从 Redis 加载最近的 Checkpoint，并重放此后的 Event Log将内存中的 FRP 状态机恢复到崩溃前的那一刻。

---

## 3. 本章小结

*   **配置是流**：利用 FRP 的组合算子（`combineLatest`, `withLatestFrom`）将配置注入到业务流中，杜绝全局变量，实现线程安全的热更新。
*   **部署需耐心**：长连接应用必须处理 Connection Draining，利用影子流量（Shadowing）进行无损的 Prompt 测试。
*   **监控需深入**：除了 CPU/Latency，更要关注 TTFT、流停顿、死循环率和 Token 燃烧速度。
*   **活着最重要**：设计多级降级策略（Fallback Stream），在依赖崩溃时保持最低限度的服务能力。

---

## 4. 练习题

### 基础题

**1. 配置热更新的算子选择**
你需要实现这样一个逻辑：每当用户说话时，使用当前时刻配置中心指定的 `SystemPrompt`。如果配置中心在 LLM 生成过程中更新了 Prompt，**不影响**当前正在生成的这句话，只影响下一句话。请选择合适的 FRP 模式并解释。

<details>
<summary>点击查看提示与答案</summary>

*   **提示**：比较 `combineLatest` 和 `withLatestFrom` 的区别。思考谁是“主驱动力”。
*   **参考答案**：
    应使用 `userInputStream.withLatestFrom(configStream)`。
    *   **原因**：`combineLatest` 会在 `configStream` 更新时**立即**触发下游（导致如果配置变了，Agent 可能会突然重新说话，这是不符合预期的）。而 `withLatestFrom` 只有在 `userInputStream` 发出事件（用户说话）时，才去“拉取”最新的配置。这完美符合“只影响下一句话”的需求。
</details>

**2. 简易死循环检测器**
LLM 有时会陷入“I am, I am, I am...”的循环。设计一个简单的 FRP 流逻辑，实时检测输出流，如果最近 5 个 Token 完全相同，则抛出 `LoopDetectedError`。

<details>
<summary>点击查看提示与答案</summary>

*   **提示**：使用 `buffer` 或 `scan` 保存最近的状态。
*   **参考答案**：
    ```text
    tokenStream
      .scan((history, newToken) -> {
          // 只保留最近 5 个
          let newHistory = [...history, newToken].slice(-5);
          return newHistory;
      }, [])
      .map(history -> {
          if (history.length < 5) return "ok";
          if (history.every(t -> t === history[0])) throw new LoopDetectedError();
          return "ok";
      })
    ```
</details>

**3. TTFT 监控埋点**
在一个从 `UserEvent` 到 `TokenStream` 的处理链中，应该在哪里记录时间戳来计算 Time To First Token？

<details>
<summary>点击查看提示与答案</summary>

*   **提示**：你需要两个时间点：用户输入结束时，和 LLM 输出第一个 token 时。
*   **参考答案**：
    1.  **T0**: 在 `UserInputStream` 收到完整请求时，通过 `map` 操作符给事件打上 `startTime = Date.now()`。
    2.  **T1**: 在 LLM Response 流的**第一个**事件到达时（可以使用 `take(1)` 或者在 scan 中检测 index=0）。
    3.  **计算**: `TTFT = T1 - T0`将此指标推送到监控流 `MetricStream`。
</details>

### 挑战题

**4. 成本熔断器 (Cost Circuit Breaker) 实现**
设计一个全系统级别的保护机制：限制每分钟全系统最多消耗 1,000,000 Tokens。如果超过，**所有**新的非 VIP 用户请求直接被拒绝（返回“系统繁忙”），直到下一分钟窗口开始。要求考虑分布式环境（多个 Pod）。

<details>
<summary>点击查看提示与答案</summary>

*   **提示**：单机内存无法统计全局量。需要借助 Redis + Lua 脚本或滑动窗口算法。在 FRP 中，这应该是一个异步的 `Gate`。
*   **参考答案**：
    1.  **全局计数器**: 使用 Redis 的 `INCR` 和 `EXPIRE` 实现分钟级计数器。
    2.  **Token 预估流**: 在调用 LLM 前，先根据 Prompt 长度估算 Token 消耗（Input Token），或者在流结束后上报实际消耗。
    3.  **准入控制 (Admission Control)**:
        构建一个 `GlobalBudgetSignal`，它每秒从 Redis 拉取（订阅）当前消耗量。
        ```text
        requestStream
          .withLatestFrom(globalBudgetSignal)
          .filter(([req, usage]) -> {
              if (req.user.isVip) return true;
              return usage < 1000000;
          })
          .else(emit("System Busy"))
        ```
    *   *进阶*：为了减少 Redis 压力，可以在 Pod 本地做预聚合（Batch Update），每 5 秒同步一次，但这会牺牲一定的精确度。
</details>

**5. 智能回退策略 (Smart Fallback)**
设计一个流，首先尝试调用 `Tool A` (高精度但慢)，如果 `Tool A` 在 2 秒内没反应，**不取消** `Tool A`（因为它可能马上就好），而是并发启动 `Tool B` (低精度但快)。最终取最先返回的那个有效结果。

<details>
<summary>点击查看提示与答案</summary>

*   **提示**：这是 `amb` (race) 算子和 `delay` 的组合应用。
*   **参考答案**：
    这是“Hedging Request”（对冲请求）模式。
    ```text
    streamA = callToolA();
    streamB = callToolB().delay(2000); // 延迟 2 秒启动 B

    // amb (ambiguous) 算子会取两个流中最早发出数据的那个
    resultStream = amb(streamA, streamB).take(1);
    ```
    *解释*：
    1. 0s: A 开始。
    2. 0s-2s: 如果 A 返回，`amb` 选中 A，B 还没启动就被取消了（因为 B 有 delay）。
    3. 2s: 如果 A 还没回，B 启动。
    4. 2s+: 现在 A 和 B 都在跑。谁先回，`amb` 就选谁，并自动取消另一个（unsubscribe）。
</details>

**6. 混沌工程：模拟网络抖动**
在测试环境中，如何利用 FRP 的特性，不修改业务代码，强行让所有 LLM 的回复都增加 500ms - 3000ms 的随机延迟，并有 10% 的概率直接抛出“网络错误”，以验证前端 UI 的健壮性？

<details>
<summary>点击查看提示与答案</summary>

*   **提示**：在 Effect 执行层注入一个“中间件”算子。
*   **参考答案**：
    在开发/测试配置中，通过依赖注入替换 `LLMService` 的实现，或者在流的管道中插入一个 `ChaosOperator`：
    ```text
    llmStream
      .flatMap(event -> {
         // 1. 随机错误注入
         if (Math.random() < 0.1) return throwError(new NetworkError());
         
         // 2. 随机延迟注入
         let delayTime = 500 + Math.random() * 2500;
         return of(event).delay(delayTime);
      })
    ```
    这个 Operator 可以由一个全局的 `ChaosConfigSignal` 控制，随时开关。
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

1.  **配置更新导致的状态不一致**
    *   *现象*：在一次长对话中，中途更新了 `MaxToken` 限制，导致 `Buffer` 算子的窗口逻辑错乱。
    *   *调试*：永远不要直接修改正在运行的 Operator 的参数。Operator 初始化时的参数通常是闭包固定的。如果需要动态调整 Operator 的行为（如动态窗口大小），需要使用支持 Signal 参数的高级 Operator，或者使用 `switchMap` 重建流。

2.  **"僵尸"流与存泄漏**
    *   *现象*：运维监控显示 Pod 内存随时间线性增长，即便用户量稳定。
    *   *原因*：在 WebSocket 断开连接时，没有正确 dispose 所有的 Subscription。特别是在处理 `retry` 或 `timeout` 时产生的临时流。
    *   *Rule of Thumb*：每一个 `subscribe()` 调用，都必须有一个对应的 `takeUntil(destroySignal)` 或显式的 `unsubscribe()`。

3.  **过度降级 (Degradation Loop)**
    *   *现象*：主模型响应稍微慢了一点（比如 1.1s），立刻触发了降级到备用模型。备用模型瞬间被打爆，导致全站瘫痪。
    *   *对策*：Fallback 策略应该有“抖动（Jitter）”和“冷却（Cooldown）”。不要因为一次超时就判定服务不可用。应结合滑动窗口错误率来决策。

4.  **日志记录了敏感信息 (PII Leak)**
    *   *现象*：在 Debug 模式下，为了方便，把整个 Event 对象（包含用户 Prompt 和 PII）打印到了 ELK/Datadog，违反 GDPR。
    *   *对策*：实现一个 `LogSanitizer` 操作符。在流的末端记录日志前，自动正则替换掉 Email、Phone、API Key 等敏感字段。

5.  **忽视了取消信号的传递**
    *   *现象*：用户在前端点了“停止生成”，UI 停了，但后端的 Agent 还在傻傻地跑 Python 代码、查数据库、调 LLM，浪费大量资源。
    *   *对策*：FRP 的核心优势就是 Cancellation Propagation。确保你的 HTTP/RPC 客户端支持 `AbortController` 或 `CancelToken`，并在流被 unsubscribe 时触发它。
