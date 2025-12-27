# Chapter 13｜日志持久化与可观测性：Tracing/Replay/Metric 一体化

> **开篇**
>
> 在构建实时 LLM Agent 时，我们面临着传统后端从未有过的调试噩梦：
> *   **非确定性**：同样的 Prompt 输入，LLM 可能产生不同的输出。
> *   **异步并发**：用户在说话，RAG 在检索，工具在执行，UI 在渲染，这四件事可能同时发生。
> *   **状态复杂性**：Bug 往往不是“代码崩溃”，而是“Agent 在第 5 轮对话时忘记了第 1 轮设定的约束”。
>
> 传统的 "Printf Debugging" 或简单的错误日志在这里毫无意义。你需要的是一个**“时空胶囊”**。
>
> 本章将利用 FRP 的核心特性——**流的不可变性（Immutability）**与**副作隔离（Effect Isolation）**，构建一套集 **结构化日志（Logging）**、**分布式链路追踪（Tracing）** 和 **确定性回放（Deterministic Replay）** 于一体的可观测性系统。
>
> 我们的终极目标是：**当线上出现 Bug，开发者可以下载日志文件，在本地 IDE 中按下一键，Agent 就像录像带倒带一样，在本地精确复现当时的每一帧状态。**

---

## 13.1 日志的对象模型：解构 Agent 的生命周期

在 FRP 架构中，我们需要记录的不再是散乱的文本，而是**五类核心对象**。这五类对象构成了 Agent 运行的完整全息图。

### 1. Event (不可变的事实)
外部世界输入系统的所有信息。它们是系统的“驱动源”。
*   **User Events**: 文本输入、语音片段、按钮点击、中断信号。
*   **System Events**: 时钟 Tick、网络状态变更、配置热更新。
*   **Callback Events**: 第三方 API 的异步回调（Webhook）。

### 2. State Snapshot (系统的记忆)
`State = scan(reducer, initial, EventStream)`。
每当 Event 被处理，State 就会更新。我们需要记录关键节点的**状态快照（Snapshot）**或**增量差异（Diff）**。
*   包含：对话历史（ChatHistory）、Prompt 缓冲区、当前激活的工具、Token 消耗计数器。

### 3. Effect (系统的意图)
这是 FRP 纯逻辑层计算出的“副作用请求”。**注意：记录 Effect 意图，不等于记录 Effect 结果。**
*   例子：`{ type: "LLM_REQ", prompt: "...", temp: 0.7 }` 或 `{ type: "TOOL_CALL", name: "search", query: "..." }`。

### 4. Effect Result (外界的反馈)
这是副作用执行后返回的数据。在我们的模型中，Result 本质上也是一种 **Event**，通常被回注到流中。
*   例子：`{ type: "LLM_RESP", text: "你好", tokens: 15 }` 或 `{ type: "TOOL_ERR", code: 500 }`。

### 5. Span (时间跨度)
用于性能分析的操作区间。
*   包含：`trace_id`, `span_id`, `start_time`, `end_time`, `status`, `attributes`。

---

## 13.2 结构化日志 Schema 设计

为了实现自动化分析，日志必须严格结构化。以下是生产级的 Schema 建议（基于 JSON Lines 或 Protobuf）。

### 通用信封 (The Envelope)
所有日志记录共享的外层结构。

```json
{
  "ver": "1.0",                  // Schema 版本，用于向后兼容
  "ts_phy": 1715000000123,       // 物理时间戳 (Wall Time)
  "ts_log": 4501,                // 逻辑时钟/序列号 (Logical Clock)，绝对递增
  "trace_id": "req-xyz-999",     // 全链路 ID (Session ID or Request ID)
  "span_id": "span-abc-111",     // 当前操作 ID
  "parent_id": "span-abc-000",   // 父操作 ID
  "kind": "EVENT | STATE | EFFECT | SPAN_END", 
  "payload": { ... }             // 具体业务数据
}
```

### 关键 Payload 定义

#### (1) Event Payload (输入)
```json
{
  "type": "USER_AUDIO_CHUNK",
  "data_ref": "s3://logs/audio/chunk_4501.pcm", // 大数据存引用
  "duration_ms": 200,
  "vad_score": 0.95
}
```

#### (2) Effect Payload (意图)
```json
{
  "target": "LLM_Service",
  "action": "completion",
  "params_hash": "a1b2c3d4", // 用于快速比对
  "body_snapshot": {         // 完整记录 Prompt，这是 Debug 的关键！
    "messages": [...],
    "temperature": 0.5
  }
}
```

#### (3) Tool Receipt (回执)
```json
{
  "tool_name": "weather_api",
  "input_args": {"city": "Beijing"},
  "output_raw": "{\"temp\": 22}", // 原始回包
  "latency_ms": 450,
  "is_cache_hit": false
}
```

> **Rule of Thumb**: **不要只记结果，要记“导致结果的原因”。**
> 对于 LLM 调用，必须记录完整的 Prompt（包含 System Prompt 和 History）。很多时候 Bug 是因为 Context 截断算法把关键信息截丢了，只看 Output 是看不出来的。

---

## 13.3 分布式追踪：跨越 FRP 的异步流

在微服务中，Trace Context 通过 HTTP Header 传递。在 FRP 内存流中，Context 需要在 Operator 之间传递。

### 难点：流的合并与竞态
当 `Stream A`（用户输入）和 `Stream B`（定时器）合并时，下游的 Trace Context 应该算谁的？

### 解决方案：Context Propagation Rules

1.  **Event 携带 Context**：每个 Event 对象内部必须携带 `trace_id` 和 `span_context`。
2.  **Map/Filter 透传**：`stream.map(e => process(e))` 输出的新事件自动继承输入事件的 Context。
3.  **Merge 竞争**：`merge(streamA, streamB)`。谁触发了这次更新，下游就继承谁的 Context。
4.  **CombineLatest 混合**：当 A 和 B 结合产生 C 时，C 的 Context 通常继承自 **"主驱动方" (Primary Driver)**，或者创建一个新的 Span `link` 到 A 和 B。
5.  **SwitchLatest 截断**：
    *   这是 FRP 中最需要关注的。
    *   当新流切断旧流时，必须显式记录旧流的 **`Span Cancelled`** 事件。
    *   统计指标时，要计算 **"Wasted Time"**（被 Cancel 的 Span 耗时）。

**ASCII 视图：流式链路追踪**

```text
[User Input] --(Trace: T1)--> [ Debounce ] --(T1)--> [ LLM Request Start ]
                                   |
(User types again before 500ms)    X (Cancel T1) --> [ Log: Span T1 Dropped ]
                                   |
[User Input] --(Trace: T2)--> [ Debounce ] --(T2)--> [ LLM Request Start ] ...
```

---

## 13.4 Replay：确定性回放（The Time Machine）

这是本章的核心。FRP 架构如果设计得当，"Debug" = "Replay"。

### 原理：Dependency Injection + Event Sourcing
Agent 的核心逻辑是纯函数：`NextState = f(CurrentState, Event)`。
只要我们能重现 `Event` 序列，就能重现 `State` 变迁。

### 实施步骤

#### 1. 隔离副作用 (Isolation)
代码中严禁出现以下直接调用：
*   `new Date()` / `Date.now()`
*   `Math.random()`
*   `fetch()` / `db.query()`

**必须** 抽象为接口，通过 `Environment` 注入：
*   `env.clock.now()`
*   `env.random.next()`
*   `env.tools.call()`

#### 2. Mock 执行器 (The Replay Driver)
在回放模式下，我们将真实的 `Environment` 替换为 `ReplayEnvironment`。

*   **Time**: 读日志中的 `ts_log`，快进式地推进虚拟时间。
*   **API Calls**: 当 Agent 发起 Effect（如查询天气）时，ReplayEnv **不发网络请求**，而是去日志里查找：*“在逻辑时间 4501，是否有 tool_name='weather' 的 Receipt？”*
    *   如果有，直接返回日志里记录的 Result。
    *   如果没有（可能是代码逻辑改了，导致发起了新的请求），抛出 `ReplayDivergenceError`（回放偏离错误）。

#### 3. 状态重建验证
回放结束后，对比回放生成的最终 State Hash 与日志中记录的 State Hash。如果一致，说明逻辑复现成功。

> **Rule of Thumb**: **随机性必须被记录。**
> 如果使用了 `temperature > 0`，LLM 的输出是随机的。回放时，我们**不应该**重新调用 LLM（即使是 Mock），而是直接使用日志中记录的 `Completion Text` 作为 Mock 的返回值。这样才能保证我们在调试“当时那个幻觉”，而不是生成一个新的幻觉。

---

## 13.5 指体系：Agent 特有的健康度

除了 CPU/RAM，我们需要关注以下三类指标。

### 1. 质量与体验 (Quality & UX)
*   **Interruption Rate**: 用户打断 Agent 发言的频率。高频率通常意味着 Agent 啰嗦或反应慢。
*   **Sentiment Drift**: 用户情绪在对话前后的变化（需轻量级 NLP 模型打分）。
*   **Correction Ratio**: 用户紧接着发出 "不对"、"错了" 或撤回消息的比例。

### 2. 性能与流式特性 (Performance)
*   **TTFT (Time To First Token)**: 首字延迟。
*   **Token Generation Rate**: 生成速度 (tokens/sec)。
*   **E2E Latency**: 意图完成总耗时。
*   **Frozen Time**: UI 没有任何更新的最长间隔（检测卡顿）。

### 3. 成本与效率 (Cost & Efficiency)
*   **Token Utilization**: `Useful Output Tokens` / `Total Input Tokens`。
*   **Speculative Wasted Rate**: 投机执行（Speculative Execution）产生的 Token 被丢弃的比例。如果该比例 > 50%，说明投机策略过于激进。
*   **Cache Hit Rate**: Embedding 缓存或 RAG 结果缓存的命中率。

---

## 13.6 日志持久化策略与隐私

### 采样与分级 (Tiered Storage)

| 数据类型 | 采样率 | 存储位置 | 保留期 | 用途 |
| :--- | :--- | :--- | :--- | :--- |
| **Metrics** (Counter/Gauge) | 100% | Prometheus/Datadog | 1年+ | 趋势分析、告警 |
| **Meta Logs** (TraceID, Status) | 100% | ES/ClickHouse | 3个月 | 检索、统计成功率 |
| **Full Payload** (Prompt, Response) | 1-5% (Head) <br> 100% (Error) | S3/OSS (Cold) | 2周 | 深度调试、回放 |
| **Binary** (Audio/Image) | 0.1% | S3 Deep Archive | 3天 | 极少数Case分析 |

### 隐私脱敏 (PII Scrubbing)
LLM 的 Prompt 经常包含 PII。在写入磁盘前，必须经过 **Scrubber Pipeline**。

*   **Regex Replacement**: 替换手机号、邮箱、身份证。
*   **Named Entity Redaction**: 使用小模型（如 BERT-NER）识别并替换人名/地名。
*   **Token Isolation**: 将敏感数据（Sensitive Data）单独存储在加密库，日志中只留 `ref_id`。只有持有特定私钥的审计员才能解开 `ref_id`。

---

## 13.7 可视化：从后台到前台

可观测性不仅给开发者看，也可以部分暴露给用户（Explainability）。

### 开发者面板
*   **Gantt Chart**: 瀑布图展示 RAG、Tool、LLM 的并行与串行关系。
*   **Token Replay**: 在时间轴上逐个 Token 重新打字，配合当时的 State 变化。

### 用户可见的 "Thinking Process"
*   将后端的 `Effect Start/End` 事件映射为 UI 上的状态指示器。
*   *Example*:
    *   `[Effect: Retrieve]` -> UI 显示 "正在查阅文档库..."
    *   `[Effect: Python]` -> UI 显示 "正在计算数据..."
    *   `[Effect: Error]` -> UI 显示 "搜索超时，正在重试..."

---

## 本章小结

1.  **Event Stream 是单一事实来源**：State 只是 Event 的投影，Log Event 等于 Log Everything。
2.  **Context 必须并在流中传递**：解决异步流 Trace 断裂的问题。
3.  **Mock 也就是 Replay**：通过截 Effect 执行器，配合日志中的历史回执，实现无需真实后端的确定性回放。
4.  **指标要关注“浪费”**：在实时 Agent 中，被 Cancel 的计算、无效的 Retrieve 是优化重点。

---

## 练习题

### 基础题

**1. 日志字段设计**
设计一个 `ToolCallEvent` 的 JSON 结构。要求：如果该工具调用失败了，我能通过这个日志知道它是第几次重试（Retry Count），以及它属于哪一轮对话（Turn ID）。

<details>
<summary>参考答案</summary>

```json
{
  "type": "TOOL_CALL",
  "trace_id": "sess-123-turn-5", // 包含 Session 和 Turn 信息
  "span_id": "call-retry-2",
  "ts": 1620000000,
  "payload": {
    "tool": "stock_api",
    "params": {"symbol": "AAPL"},
    "meta": {
      "turn_id": 5,
      "retry_count": 2, // 第2次重试
      "max_retries": 3,
      "reason_for_retry": "Previous timeout"
    }
  }
}
```
</details>

**2. 耗时计算**
在 FRP 中，用户输入 "Hello" 触发了 LLM 请求。
Event A: `UserInput` (ts: 100)
Event B: `LLMStart` (ts: 150)
Event C: `LLMFirstToken` (ts: 300)
Event D: `LLMEnd` (ts: 600)
请计算：(1) 系统处理延迟 (Overhead) (2) 首字延迟 (TTFT) (3) 生成速度 (假设生成了 30 tokens)。

<details>
<summary>参考答案</summary>

(1) **System Overhead**: Event B - Event A = 150 - 100 = 50ms (包含 Debounce、Prompt 组装耗时)
(2) **TTFT**: Event C - Event A = 300 - 100 = 200ms (用户视角的首字延迟)
(3) **Speed**: 30 tokens / (600 - 300)ms = 30 / 0.3s = 100 tokens/sec
</details>

### 挑战题

**3. 处理回放偏离 (Divergence)**
你在 Debug 一个线上的 Bug，下载了日志进行回放。但是你本地的代码比线上版本新，修改了 Prompt 模板（加了一句 System Instruction）。
当你运行回放时，Agent 生成的 Prompt Hash 与日志里的不一致。
请问：
1.  回放系统应该报错停止，还是继续？
2.  如果继续，后续的 Mock Tool Call 能否匹配上？
3.  如何设计一种“模糊匹配”机制应对轻微的代码变更？

<details>
<summary>参考提示</summary>

1.  **默认策略**：应该报警 (Warn) 但允许尝试继续。因为 Debug 的目的往往就是验证“新代码能否修复旧 Bug”。
2.  **匹配问题**：如果 Prompt 变了，LLM 的输出（在 Mock 模式下）可能就对不上了。比如日志里 LLM 说“调用工具 A”，但新 Prompt 导致 LLM 说“调用工具 B”。此时必须停止回放，因为路径分叉了。
3.  **模糊匹配**：
    *   对于 Tool Call Mock，不要仅匹配 Prompt Hash。应该匹配 **"Tool Name + Arguments"**。
    *   即使 Prompt 变了，只要 LLM 依然决定调用同一个工具且参数相同，回放就可以继续。这增强了回放的鲁棒性。
</details>

**4. 隐私与长上下文的冲突**
你需要记录 Agent 的完整上下文以便 Debug "遗忘" 问题，但 Context 中包含了用户上传的机密 PDF 内容（10MB 文本）。
如果全量记日志，存储成本爆炸且有合规风险。如果不记，Debug 时不知道 PDF 内容是什么。
请设计一个方案解决这个问题。

<details>
<summary>参考提示</summary>

**方案：Content-Addressable Storage (CAS) + Zero-Knowledge Logs**
1.  **PDF 解析阶段**：将 PDF 文本提取后，计算 Hash (SHA-256)。
2.  **存储**：将 PDF 文本存入一个安全的、有 TTL（自动过期）的 Blob 存储，Key 就是 Hash。
3.  **日志记录**：在 Event Log 中，只记录 `content_hash: "sha256:..."` 和 `content_length: 50000`。
4.  **回放**：
    *   如果 Blob 还在（有效期内），回放器自动下载内容填充。
    *   如果 Blob 已过期/被删，回放器使用 "Placeholder Text"（如 `[CENSORED CONTENT - 5000 tokens]`）填充。虽然无法 Debug 具体语义，但可以 Debug Token 预算计算、上下文裁剪逻辑等与具体内容无关的 Bug。
</details>

---

## 常见陷阱与错误 (Gotchas)

### 1. `JSON.stringify` 的性能陷阱
**错误**：在主线程的流处理管道中直接同调用 `JSON.stringify(huge_state_object)`。
**后果**：当 State 包含长对话历史时，序列化可能耗时 10-50ms，这会阻塞事件循环，导致音频处理丢帧或 UI 掉帧。
**修正**：
*   使用 `worker_thread` 进行日志序列化。
*   或者使用 `Flatted` 等库的异步版本。
*   仅记录 Diff 而非全量 State。

### 2. 时间戳混乱
**错误**：在日志分析时，混用了 `Event_Created_Time` (事件产生的时刻) 和 `Log_Written_Time` (日志落盘的时刻)。
**后果**：在高并发或网络拥塞下，落盘时间可能滞后几秒，导致分析出的 Latency 指标完全错误。
**修正**：永远信任 Event 内部携带的 `timestamp` 字段。

### 3. "Heisenbug" (海森堡 Bug)
**错误**：开启详细日志（Debug Mode）时 Bug 消失，关闭时 Bug 复现。
**原因**：通常是因为日志操作本身引入了 I/O 延迟，改变了 `race condition` 的获胜者。
**修正**：
*   确保日志操作是完全异步的（Fire-and-forget）。
*   在 FRP 中使用逻辑时钟（Sequence ID）而非物理时间来决定竞态结果（见 Chapter 11）。
