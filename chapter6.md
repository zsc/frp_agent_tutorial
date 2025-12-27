# Chapter 6｜工具 / API 调用：失效处理、回退、幂等与隔离

## 1. 开篇：从“调用函数”到“管理副作用”

在简单的脚本中，调用一个 API 只是写一行 `response = await client.get(...)`。但在一个长时运行、高并发、且由不确定的 LLM 驱动的 Agent 系统中，工具调用（Tool Calling）是系统的**风险中心**。

LLM Agent 的工具调用面临双重混沌：
1.  **内部混沌（模型侧）**：LLM 可能幻觉出不存在的函数、传错参数类型、或者陷入“死循环调用”。
2.  **外部混沌（环境侧）**：API 会超时、数据库会锁死、网络会抖动、第三方服务会限流。

在 FRP（Functional Reactive Programming）架构下，我们**不直接调用工具**。我们将工具调用建模为**副作用（Side Effect）的管理流程**：
*   LLM 发出的不是指令，而是**意图事件（Intent Event）**。
*   系统对意图进行**校验、去重、调度**。
*   执行器（Effect Runner）在隔离环境中运行工具，产生**结果流（Result Stream）**。
*   结果流被**回纳（Reduced）**进系统状态，再次驱动 LLM。

本章的目标是构建一个**反脆弱**的工具执行层：它不仅能执行任务，还能在失败中恢复、在拥堵中降级，并始终保持对 LLM 和用户的诚实。

---

## 2. 文字论述

### 2.1 Tool Intent vs. Effect Execution：纯逻辑与副作用的隔离
在 FRP 中，我们追求核心逻辑的“纯函数”化。这意味着 Agent 的决策过程（Input -> Planning）不应包含任何网络 I/O。

我们将过程拆分为两个独立的流：
1.  **意图流 (Intent Stream)**：这是“大脑”的输出它包含 `tool_name`、`arguments`、`reasoning`。它是纯数据，生成它的过程极快且无副作用。
2.  **执行流 (Execution Stream)**：这是“手脚”的动作。它监听意图流，执行 HTTP 请求、DB 查询或 Python 代码。它包含延迟、失败和重试。

**架构图：意图-执行-回执循环**

```ascii
[ LLM Decision Core ]
       | (emits)
       v
( ToolIntent Event )  <-- 纯数据： "我想查天气，地点北京"
       |
       +---> [ 1. 拦截器/中间件 ] (鉴权、预算检查、熔断检查)
       |
       v
[ Effect Scheduler ]  <-- 调度器 (控制并发、批处理)
       |
       +--- (async execution) ---> [ 外部 API / Python Sandbox ]
                                          |
                                    (耗时: 200ms - 30s)
                                          |
                                          v
( ToolReceipt Event ) <-- 结构化回执： "状态200，内容：25度，耗时300ms"
       |
       v
[ State Reducer ]     <-- 更新记忆，触发下一次 LLM 推理
```

这种分离的好处在于：**可测试性**。你可以在不联网的情况下，通过向意图流发送 Mock 事件来测试 Agent 的规划能力；也可以通过回放历史的意图事件来压力测试执行器。

### 2.2 失败分类学：构建精细的错误处理策略
将所有异常都 `try-catch` 并重试是初级做法。在流式系统中，我们需要根据错误的**语义**将它们分流到不同的处理管道：

| 错误类型 | 典型场景 | FRP 处理策略 | 目标接收方 |
| :--- | :--- | :--- | :--- |
| **瞬态错误 (Transient)** | 网络超时 (504)、连接重置、拥塞 | **Retry with Backoff** (重试流) | 执行器内部消化 |
| **语义错误 (Semantic)** | 参数缺失、类型错误、400 Bad Request | **Pass Through** (透传) | **LLM** (让它自己改) |
| **鉴权/配额 (Policy)** | Key失效 (401)、余额不足、限流 (429) | **Circuit Break** (熔断) | 运维/人类用户 |
| **致命错 (Fatal)** | 工具下线 (404)、沙箱崩溃 | **Fallback** (降级) | 规划器 (Planner) |

*   **Rule of Thumb**：如果是因为 LLM "太蠢"导致的错误（如传了非法参数），千万不要重试，直接把错误文本作为 Prompt 喂回给它，这叫 "Error Correction Prompting"。

### 2.3 Circuit Breaker (熔断器) 的响应式实现
当某个工具（如 Bing Search）的失败率飙升时，继续让 LLM 调用它只会浪费 Token 并增加延迟。我们需要一个响应式的熔断器。

在 FRP 中，熔断器是一个维护状态的 **Scan** 算子：
1.  **输入**：工具执行结果流。
2.  **逻辑**：在滑动时间窗口（Window）内，计算失败比率。如果 > 阈值，状态转为 `OPEN`。
3.  **输出**：**ToolAvailability Signal（工具可用性信号）**。

**关键点：熔断状态如何通知 LLM？**
这与传统微服务熔断不同。在 Agent 中，熔断器不仅要在代码层面拦截请求，还要**实时修改 System Prompt**。
*   **Prompt Patching**：当 `Availability Signal` 变为 `OPEN` 时，Prompt Manager 收到信号，在 Prompt 中注入一段话：*“注意：搜索工具当前处于维护模式，不可用。请利用你的内部知识回答。”*
*   这样，LLM 就会自然地停止生成调用该工具的 Token，而不是生成了调用后再被代码拦截，造成困惑。

### 2.4 Retry, Backoff 与 Jitter：流的时间控制
重试不仅是循环，更是对“时间流”的操作。
*   **Backoff（退避）**：第一次失败等 1s，第二次等 2s，第三次等 4s。
*   **Jitter（抖动）**：必须引入随机性。`delay = base_delay * 2^retries + random(0, 100ms)`。
*   **可视化时间线**：

```ascii
Attempt 1 (Fail)
|
+--- 1s (Backoff) ---+
                     |
                     Attempt 2 (Fail)
                     |
                     +------- 2s + Jitter -------+
                                                 |
                                                 Attempt 3 (Success)
                                                 |
                                                 v
                                          (Result Event Emitted)
```
在 FRP 中，这通常通过 `retryWhen` 算子配合 `timer` 和 `zip` 来优雅实现，而不是写复杂的 `while` 循环。

### 2.5 幂等键 (Idempotency Key) 与去重
Agent 系统中常发生“重复提交”：
1.  用户狂点“重试”按钮。
2.  LLM 在流式输出中断后，重新生成时重复了之前的工具调用文本。

**解决方案**：
*   **生成 Key**：Key = `Hash(ConversationID + TurnID + ToolName + SortedArguments)`。
*   **去重算子**：使用 `distinctKey` 或带有 TTL（生存时间）的缓存。
*   **行为**：如果 Key 命中缓存，直接返回缓存中的 `Result Event`，不再发起物理网络请求。对于写操作（如“发送邮件”），幂等性至关重要。

### 2.6 回退策略 (Fallback Strategies)
当工具彻底失效（熔断器打开或重试耗尽）时，系统不能 Crash，必须降级。FRP 的 `catchError` 或 `switchIfEmpty` 算子允许我们平滑地切换流的来源：

1.  **同类切换 (Switch to Alternative)**：Google Search 挂了 -> 切换到 Bing Search 流。
2.  **缓存回退 (Stale-while-error)**：无法获取实时股价 -> 返回 10 分钟前的缓存数据，并在 `Result Event` 中标记 `quality: degraded`。
3.  **空回退 (Graceful Empty)**：返回一个特殊的“空结果”对象，Prompt 模块将其渲染为：“系统尝试了搜索，但服务暂时不可用”。

### 2.7 结构化工具回执 (ToolReceipt)
工具返回给系统的不应只是一个 `String`。它必须是一个结构化对象，满足不同消费者的需求：

*   **消费者 A：LLM** -> 需要简洁的文本摘要（Token 越少越好）。
*   **消费者 B：UI** -> 需要富文本、图片 URL、JSON 数据（用于渲染卡片）。
*   **消费者 C：Metrics** -> 需要耗时、Token 消耗、原始状态码。

**Schema 设计建议**：
```json
{
  "id": "call_123",
  "status": "success",
  "payload": { ... },       // 原始数据
  "summary": "北京今日晴...", // 喂给 LLM 的摘要
  "display": {              // 喂给 UI 的组件数据
    "type": "weather_card",
    "icon": "sunny",
    "temp": 25
  },
  "meta": {                 // 喂给监控
    "latency_ms": 240,
    "provider": "weather_api_v3",
    "cached": false
  }
}
```

### 2.8 安全沙箱与隔离 (Sandboxing)
对于 Python Code Interpreter 或 SQL Tool，执行本身就是一个高风险操作。
*   **隔离作为副作用**：将沙箱执行视为一个**远程异步调用**。不要在 Agent 的主进程中 `exec()` 代码。
*   **资源限制**：在执行流中加入 `timeout` 算子。如果 Python 代码 5 秒未返回，主流直接切断，并发送 `SIGKILL` 给沙箱容器。
*   **只读模式**：对于数据库工具，FRP 可以在意图流阶段进行正则检查，如果检测到 `DROP` / `DELETE` 且没有高权限标记，直接在流中拦截并替换为“权限拒绝”事件。

---

## 3. 本章小结

*   **意图与执行分离**：让 LLM 负责“想”，让 Runtime 负责“做”。
*   **错误分层处理**：网络错误重试，语义错误反馈，权限错误熔断。
*   **Prompt 联动**：熔断器状态必须实时反映在 Prompt 中，防止 LLM 做无用功。
*   **幂等性保护**：防止 LLM 的不确定性导致现实世界的重复副作用。
*   **回执结构化**：一个结果，多种视图（LLM 视图、UI 视图、监控视图）。

---

## 4. 练习题

### 基础题

**Q1. 错误分类实践**
对于以下错误，请分别归类为（瞬态、语义、权限、致命）并给出处理动作：
1. HTTP 502 Bad Gateway
2. JSONDecodeError: Expecting value
3. HTTP 403 Forbidden (API Key expired)
4. HTTP 400 Bad Request (Invalid date format)

<details>
<summary>参考答案</summary>

1.  **瞬态** -> 重试 (Retry with backoff)。
2.  **致命/语义** -> 视情况而定。如果是工具返回的 JSON 坏了是工具故障（致命）；如果是 LLM 生成的 JSON 坏了，是语义错误 -> 反馈给 LLM。
3.  **权限** -> 熔断/停止，报警。
4.  **语义** -> 反馈给 LLM："Date format is invalid, please use YYYY-MM-DD"。
</details>

**Q2. 幂等键设计**
如果 Agent 有一个工具 `send_email(to, subject, body)`。
用户说：“帮我给老板发邮件请假”。Agent 调用了一次。
用户觉得 Agent 没反应，又说了一遍：“发了吗？没发快发”。
如何设计幂等键防止发两封邮件？

<details>
<summary>参考答案</summary>

*   **Key 组成**：`Hash(User_ID, Date, Function_Name="send_email", Argument_Hash)`。
*   **策略**：
    1.  如果仅仅是对参数哈希，可能用户真的想发两封一样的。
    2.  更好的做法是引入 `TimeWindow`。在 5 分钟内，相同的（收件人+主题+正文）哈希会被视为重复。
    3.  或者，Agent 在内部状态中维护 `PendingActions` 列表，第二次请求时检查是否已有相的 Action 在 `Pending` 或 `RecentlyCompleted` 状态。
</details>

**Q3. 超时算子**
在 FRP 中，如果一个工具调用流 `stream.timeout(3000)` 触发了超时，通常会发生什么？原来的 HTTP 请求会自动断开吗？

<details>
<summary>参考答案</summary>

*   **流的行为**：流会抛出 `TimeoutError` 并终止订阅。
*   **副作用的行为**：**不一定**。这取决于你的 Effect Runner 实现。
*   **正确做法**：必须利用 FRP 的 `unsubscribe` 回调或 `AbortController` 信号。当流超时取消订阅时，触发回调去调用 `xhr.abort()` 或 `process.kill()`。否则会产生“僵尸请求”在后台空耗资源。
</details>

### 挑战题

**Q4. 熔断器与 Prompt 的动态一致性**
你设计了一个机制，当工具挂了，Prompt 会更新。
场景：用户问“今天天气如何？” -> Agent 生成调用意图 -> 工具熔断器状态为 OPEN -> 请求被拦截。
此时 Agent 处于“等待结果”状态。你该如何把“工具不可用”这个信息传回给 Agent，让它优雅地回复用户？

<details>
<summary>参考答案</summary>

*   **步骤**：
    1.  拦截器拦截意图后，**人工合成**一个 `ToolReceipt` 事件。
    2.  事件内容：`status: error`, `error_type: circuit_breaker_open`, `content: "Tool [Weather] is currently unavailable due to high failure rate."`
    3.  这个合成事件像普通结果一样流回 LLM。
    4.  LLM 看到这个“结果”，会根据上下文生成：“抱歉，天气服务暂时不可用，我无法查询。”
    5.  同时，下一轮对话的 System Prompt 才会更新包含“不要调用 Weather”的指令。
</details>

**Q5. 幻觉参数纠正流**
LLM 调用 `lookup_user(name="John Doe")`，但 API 只接受 `user_id`。API 返回 400。
设计一个自动化的 FRP 流程，在不打扰用户的情况下尝试修正这个问题。假设你有一个辅助工具 `search_user_id(name)`。

<details>
<summary>参考答案</summary>

这是一个 **"Chain of Repair"** 模式：
1.  `ToolExecutionStream` 收到 400 错误，错误信息 "Invalid param: name. Expected: id"。
2.  进入 `catchError` 分支，不仅不报错，反而通过 `flatMap` 触发一个新的子意图：`Call(search_user_id, name="John Doe")`。
3.  子工具返回 ID `12345`。
4.  逻辑层利用这个结果，重新构造原始请求：`Call(lookup_user, id="12345")`。
5.  执行成功。
6.  **注意**：这需要 Agent 具备 "Self-Correction" 的策略配置，或者在 Effect Runner 层硬编码特定的修复逻辑（不太推荐硬编码，最好由 LLM 自己根据错误提示进行多轮思考）。
</details>

**Q6. 大数据量回执处理**
如果工具是 `read_file`，返回了 1MB 的文本，直接喂给 LLM 会撑爆 Context Window。请描述一个基于流的处理方案。

<details>
<summary>参考答案</summary>

1.  **分流处理**：ToolReceipt 分为 `metadata` (大小, 类型) 和 `blob` (实际内容)。
2.  **Blob 存储**：将 1MB 内容存入临的 Vector Store 或 Blob Storage，生成一个引用 ID `ref::file_chunk_01`。
3.  **Prompt 策略**：
    *   **方案 A (摘要)**：触发一个由小模型（如 GPT-3.5-Turbo）运行的“摘要流”，将 1MB 压缩为 500 tokens 摘要，放入 Receipt。
    *   **方案 B (RAG)**：仅在 Receipt 中返回引用 ID："File loaded. ID: ref::... Use 'search_file' to read content."
4.  LLM 看到的是：“文件已读取（1MB），内容摘要如下...”。
</details>

**Q7. 幽灵写入 (Ghost Writes)**
用户让 Agent "向数据库写入一条记录"。Agent 调用了工具，但在收到回执前，用户的网络断了（前端 WebSocket 断开）。Agent 的后台流应该继续吗？还是回滚？

<details>
<summary>参考答案</summary>

*   **原则**：如果是**读**操作，可以取消以节省资源；如果是**写**操作，通常应该**继续完成**，保证数据一致性。
*   **FRP 实现**：
    *   用户断开触发 `ConnectionLost` 事件。
    *   对于标为 `Safe` (Read-only) 的流，执行 `takeUntil(ConnectionLost)`。
    *   对于标记为 `Critical` (Write) 的流，使用 `detach` 或忽略取消信号，直到 Effect 完成。
    *   **持久化**：将结果写入 `UnreadNotifications` 队列，等用户重连后推送：“你刚才要求的写入已完成”。
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

1.  **“僵尸”进程 (Zombie Processes)**
    *   *现象*：用户在前端疯狂点击“停止”，UI 停了，但服务器 CPU 占用率 100%，因为后台的 Python 解释器或复杂的 SQL 查询根本没收到取消信号。
    *   *调试*：检查 Effect Runner 是否正确绑定了 FRP 的 `dispose/unsubscribe` 到底层驱动的 `abort()` 方法。

2.  **上下文爆炸 (Context Explosion)**
    *   *现象*：爬虫工具抓取了一个巨大的 HTML 页面直接返回。LLM 的 Context 瞬间被填满，导致后续无法生成或产生巨额 Token 账单。
    *   *对策*：在 Tool Runner 层设置**硬截断**（如最大 2000 字符），并在末尾添加 `...[truncated]` 标记。

3.  **工具参数的 JSON 注入**
    *   *现象*：LLM 生成的 JSON 参数格式稍微有点错（如多了一个逗号），导致 `JSON.parse` 失败。
    *   *对策*：不要使用标准的严通过 `json.loads`。使用 `json5` 或专门的 "Fuzzy JSON Parser"（很多 LLM 库自带），它们能容忍常见的格式错误。

4.  **死循环的“我错了”**
    *   *现象*：Agent 调用工具 -> 报错 -> Agent 道歉并重试（参数没变） -> 报错 -> Agent 道歉并重试...
    *   *对策*：必须检测**连续重复的错误**。如果同一个工具连续报错 3 次且参数相似，强制触发致命错误，终止对话流。

5.  **隐私数据泄露到 Prompt**
    *   *现象*：工具返回了包含用户手机号的原始日志，被直接塞进了 Prompt 历史。虽然没有输出给用户，但这些数据留在了 LLM 提供商的日志中。
    *   *对策*：定义 PII（个人敏感信息）过滤器算子，在 `ToolReceipt` 进入 State 之前进行正则清洗。
