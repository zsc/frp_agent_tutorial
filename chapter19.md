# Appendix B｜数据结构与协议：Event/State/Receipt/Trace Schema

## 1. 开篇段落：协议即系统

在传统的面向对象编程（OOP）中，系统的骨架是类（Classes）和接口（Interfaces）；而在函数式响应式编程（FRP）中，**系统的骨架是数据流（Data Streams）中的 Schema**。

由于 FRP 将“行为”抽象为算子（Operators），系统中流动的数据结构（Payloads）就成为了定义系统能力的唯一约束。一个设计良好的 Schema 体系能够带来以下巨大的工程红利：
1.  **Time Travel Debugging**：只要数据结构是可序列化的（Serializable），我们就能录制并回放任何 Bug。
2.  **前后端解耦**：Schema 定义了 UI 渲染层与 Agent 逻辑层的契约。
3.  **异构系统集成**：标准的信封格式使得 Agent 可以轻松挂载到 WebSocket、gRPC 或 HTTP Server 上。

本章不讨论具体的数据库选型（SQL vs NoSQL），而是专注于定义**内存中流动的数据协议**。我们将详细拆解四个核心维度：`Event`（输入事实）、`State`（聚合状态）、`Receipt`（副作用结果）和 `Trace`（观测元数据）。

---

## 2. 文字论述与详细 Schema 定义

### B.1 Event Schema：不可变的原子事实

Event 是系统中发生的“事情”。在 FRP 语义下，Event 必须是 **Immutable（不可变）** 的。一旦产生，它就是历史的一部分，不可修改。

#### 2.1.1 通用信封 (The Universal Envelope)
为了让 Runtime 能够统一处理日志、路由、重试和序列化，所有具体的业务事件都必须包裹在一个标准信封中。

```typescript
/**
 * 标准事件信封 (The Envelope)
 * T: 具体的 Payload 类型
 */
interface AgentEvent<T = any> {
  // --- 标识区 ---
  
  // 唯一 ID。荐使用 UUID v7，因其包含时间戳信息，在数据库索引和日志排序中性能更好
  id: string; 

  // 事件类型。采用点分命名法，e.g., "user.voice.chunk", "sys.lifecycle.stop"
  type: string; 

  // 发生时间。必须是 ISO 8601 UTC 字符串，e.g., "2023-10-27T10:00:00.123Z"
  // 切勿使用本地时间或单纯的 Unix Timestamp (调试时难以阅读)
  timestamp: string;

  // --- 逻辑控制区 ---
  
  // 来源标识。用于区分多路输入，e.g., "ws_connection_1", "timer_system"
  source: string;
  
  // 逻辑时钟/序列号 (可选)。用于在分布式或乱序网络中重建因果关系
  sequenceId?: number;

  // --- 元数据区 (Metadata) ---
  // 存放不影响业务逻辑，但对基础设施至关重要的信息
  meta: {
    correlationId: string; // 链路追踪 ID
    userId?: string;       // 归属用户
    clientVersion?: string; // 客户端版本 (用于处理兼容性)
    retryCount?: number;   // 重试次数
    priority?: 'high' | 'normal' | 'low'; // 调度优先级
  };

  // --- 数据区 ---
  // 实际的业务负载，必须是 Plain JSON Object
  payload: T;
}
```

#### 2.1.2 关键 Payload 定义详解

**A. 用户多模态输入 (User Input)**

实时 Agent 的输入远不止文本。我们需要处理流式的音频、打断信号以及非语言的 UI 交互。

```typescript
// 文本输入
interface TextPayload {
  content: string;
  isFinal: boolean; // 用于流式 ASR 的中间结果 (partial results)
}

// 音频片段 (用于 VAD 和 ASR)
interface AudioChunkPayload {
  format: 'pcm' | 'opus';
  sampleRate: number;
  data: string; // Base64 编码的音频帧，或者指向共享内存/S3 的引用
  sequence: number; // 帧序号，防止乱序
}

// 意图/控制信号 (Control Signals)
interface ControlPayload {
  action: 'interrupt' | 'undo' | 'regenerate';
  targetEventId?: string; // 针对哪个之前的事件进行操作 (e.g., 打断了哪句话)
}
```

**B. 模型生成流 (LLM Generation Stream)**

LLM 输出在 FRP 中通常被拆解为细粒度的流事件，以便 UI 实现打字机效果并尽早触发工具。

```typescript
interface ModelDeltaPayload {
  // 增量内容
  delta: string; 
  
  // 累积内容 (可选，用于 UI 容错，防止丢包导致乱码)
  accumulated?: string;
  
  // 思考过程 (针对 CoT 模型，如 o1/r1)
  reasoningDelta?: string;
  
  // 当前生成状态
  status: 'generating' | 'function_calling' | 'completed';
  
  // 如果触发了工具调用
  toolCalls?: {
    id: string;
    name: string;
    argumentsDelta: string; // JSON 片段
  }[];
}
```

---

### B.2 State Schema：确定性的系统快照

State 是 Event 流在时间轴上的投影。公式：`State(t) = Reducer(InitialState, Events[0...t])`。
设计 State 的核心原则是 **Single Source of Truth（单一事实来源）**。

#### 2.2.1 状态树结构 (The State Tree)

```typescript
interface AgentState {
  // --- 1. 配置与静态信息 (Config) ---
  config: {
    agentId: string;
    mode: 'chat' | 'copilot' | 'background';
    modelParams: { temp: number; model: string };
  };

  // --- 2. 记忆与上下文 (Memory) ---
  // 这部分通常需要持久化到 DB
  context: {
    turnCount: number;
    messages: OpenAICompatibleMessage[]; // 完整的对话历史
    variables: Record<string, any>;      // 提取出的实体记忆 (e.g., userName="Alice")
    summary?: string;                    // 长对话的摘要
  };

  // --- 3. 运行时瞬态 (Runtime Volatile) ---
  // 这部分丢失后不影响数据一致性，但影响当前体验
  runtime: {
    // 状态机核心状态
    lifecycle: 'idle' | 'listening' | 'thinking' | 'acting' | 'speaking';
    
    // 当前正在积累的输入 (比如用户正在说话，VAD 尚未截止)
    inputBuffer: {
      transcript: string;
      audioEnergy: number; // 用于 UI 绘制波形
    };

    // 当前 LLM 正在生成的草稿
    outputBuffer: {
      content: string;
      currentToolCall?: { name: string; args: string };
    };
    
    // 待处理的副作用 (Pending Effects)
    // 记录发出了哪些请求但还没收到回执，用于处理超时和取消
    pendingEffects: Record<string, {
      type: 'tool' | 'retrieval';
      startedAt: string;
      timeoutAt: string;
    }>;
  };

  // --- 4. 诊断与预算 (Diagnostics) ---
  diagnostics: {
    lastError?: { code: string; msg: string; timestamp: string };
    tokenUsage: {
      sessionTotal: number;
      lastTurn: number;
      budgetRemaining: number;
    };
    latency: {
      lastP99: number;
    };
  };
}
```

#### ASCII 图解：状态的层级

```text
Root State
├── Config (Read-onlyish)
│   └── "Mode: Strict"
├── Context (Persistent)
│   ├── Msg[0]: User "Hi"
│   ├── Msg[1]: Bot "Hello"
│   └── Vars: { "city": "Beijing" }
└── Runtime (Transient)
    ├── Lifecycle: "Thinking"
    ├── InputBuffer: (empty)
    └── PendingEffects
        └── Tool: "WeatherAPI" (id: req_123, status: sent)
```

---

### B.3 ToolReceipt Schema：双重视图协议

这是构建 **Agentic UI** 的关键。Tool/API 的返回值不能只是给 LLM 看的字符串，必须包含给 UI 渲染用的结构化数据。

#### 2.3.1 Receipt 结构设计

```typescript
interface ToolReceipt {
  // --- 关联信息 ---
  callId: string;        // 对应 LLM tool_call 的 ID
  toolName: string;      // 工具名称
  timestamp: string;     // 执行完成时间

  // --- 执行结果状态 ---
  status: 'success' | 'error' | 'timeout' | 'skipped' | 'cancelled';
  
  // --- 视图 1: 给 LLM 看 (Compact View) ---
  // 目标：最小化 Token 消耗，传递核心逻辑事实
  // 示例: "Stock AAPL is 150.00 (+1.2%)"
  systemContent: string; 

  // --- 视图 2: 给用户看 (Rich View) ---
  // 目标：提供最佳交互体验，由 UI 组件库解析
  // 示例: { type: "stock_card", data: { symbol: "AAPL", chart: [...] } }
  uiContent?: {
    viewType: string;    // 前端根据此字段选择组件
    title?: string;
    payload: any;        // 具体的 JSON 数据
    interactions?: {     // 允许用户直接在结果上操作
      label: string;
      actionId: string;  // 点击后触发新的 Event
    }[];
  };

  // --- 视图 3: 给开发者看 (Debug View) ---
  debugInfo?: {
    rawRequest: any;     // 原始 HTTP 请求
    rawResponse: any;    // 原始 HTTP 响应
    latencyMs: number;
    costCurrency: number;
  };

  // --- 错误详情 (当 status != success) ---
  error?: {
    code: string;        // e.g., "rate_limit_exceeded"
    message: string;     // 人类可读
    isRetryable: boolean; // 是否建议 Agent 重试
    suggestion?: string;  // 给 LLM 的修复建议
  };
}
```

---

### B.4 Trace Schema：异步因果链

在流式系统中，`Function A` 调用 `Function B` 的传统堆栈并不存在。存在的是 `Event A` 触发了 `Effect B`，`Effect B` 产生了 `Event C`。我们需要记录这种**因果链（Causality）**。

#### 2.4.1 Span 与 Link

基于 OpenTelemetry 语义的简化版：

```typescript
interface TraceSpan {
  traceId: string;       // 全局会话 ID
  spanId: string;        // 当前操作 ID
  parentSpanId?: string; // 父操作 ID

  name: string;          // e.g., "executor.run_tool"
  kind: 'PRODUCER' | 'CONSUMER' | 'INTERNAL';

  startTime: string;
  endTime?: string;

  // 关键：将 Trace 与 Event 绑定
  links: {
    triggerEventId: string; // 哪个 Event 开启了这个 Span
    resultEventIds: string[]; // 这个 Span 产出了哪些 Event
  }[];

  // 属性与指标
  attributes: {
    "llm.model"?: string;
    "llm.input_tokens"?: number;
    "tool.name"?: string;
    "error.type"?: string;
  };
}
```

---

## 3. 本章小结

*   **协议先行**：在写一行代码之前，先确定 Event 和 State 的 JSON Schema。
*   **信封模式**：使用统一的 Event Envelope，包含 `id`, `type`, `timestamp`, `meta`，将业务数据隔离在 `payload` 中。
*   **状态分离**：区分持久化的 `Context`（存 DB）和瞬态 `Runtime`（存内存/Redis），避免不必要的 I/O 开销。
*   **回执分层**：ToolReceipt 必须明确区分 `systemContent`（给 AI 省 Token）和 `uiContent`（给人类看效果）。
*   **不可变性**：Event 是历史事实，不可变更；State 是衍生结果，可随时重算。

---

## 4. 练习题

### 基础题
1.  **JSON 设计**：请设计一个 `TimerEvent` 的 Payload，用于触发 Agent 的定时任务（比如每 5 分钟提醒一次）。需要包含哪些字段来防止任务堆积？
2.  **State 拆解**：如果 Agent 支持“用户正在输入时展示波形动画”，这个波形的振幅数据（`amplitude`）应该放在 `AgentState` 的哪个部分？为什么不应该放在 `Context` 里？
3.  **Receipt 转换**：一个查询数据库的工具返回了 100 条 SQL 记录。请写出它的 `ToolReceipt`，要求 `systemContent` 只包含前 3 条摘要，而 `uiContent` 包含分页浏览所需的数据。

### 挑战题
4.  **Schema 演进与向后兼容**：
    *   场景：系统上线半年后，你需要将 `ToolReceipt` 中的 `uiContent` 字段从简单的 JSON 对象改为支持多 Tab 的数组结构 `uiTabs: []`。
    *   问题：如何修改 Schema 定义，使得旧版本的日志（包含 `uiContent`）在回放（Replay）时仍然能被新版代码正确处理？请给出“Upcasting（向上转型）”的伪代码逻辑。
5.  **隐私与审计**：
    *   场景：你的 Agent 被部署在医疗场景，所有 `Event` 都会被持久化。
    *   问题：如何在 Schema 层面设计一种机制，使得我们既能保留 Debug 所需的链路信息（如发生错误的工具参数），又能确保不存储用户的 PII（敏感个人信息，如病历号、姓名）？提示：考虑 Crypto-shredding 或影子字段。

---

### 参考答案

<details>
<summary><strong>点击展开参考答案</strong></summary>

#### 基础题
1.  **TimerEvent 设计**：
    ```typescript
    interface TimerPayload {
      taskId: string;        // 任务定义的唯一标识
      scheduledTime: string; // 计划执行时间
      actualTime: string;    // 实际触发时间
      missedTicks: number;   // 如果系统卡顿，错过了多少次 tick (用于背压处理)
      isRecurring: boolean;
    }
    ```
    *   *Hint*: `missedTicks` 很重要，如果积压太多，策略层可以选择丢弃或合并执行。

2.  **波形数据位置**：
    *   **位置**：`state.runtime.inputBuffer` 或单独的 `state.ui.ephemeral`。
    *   **原因**：这是高频、易变、无业务含义的数据。放入 `Context` 会导致数据库写入爆炸，且该数据在会话结束后无价值。甚至可以不进主 State，直接由前端处理，除非后端需要基于音量做 VAD 逻辑。

3.  **DB 查询 Receipt**：
    ```json
    {
      "toolName": "query_users",
      "systemContent": "Found 100 users. Top 3: Alice, Bob, Charlie. (Data truncated, ask specifically to filter)",
      "uiContent": {
        "viewType": "data_table",
        "payload": {
          "total": 100,
          "columns": ["id", "name", "role"],
          "rows": [ ...100 records... ], // 前端负责分页
          "pageSize": 10
        }
      }
    }
    ```

#### 挑战题
4.  **Schema 演进 (Upcasting)**：
    *   **思路**：在数据进入 Reducer 之前，通过一个 `Upcaster` 层将旧数据标准化为新数据。
    *   **伪代码**：
      ```typescript
      function upcastEvent(rawEvent: any): AgentEvent {
        // 假设旧版 schemaVersion 为 1 或 undefined
        if (!rawEvent.v || rawEvent.v < 2) {
          if (rawEvent.type === 'tool.result' && rawEvent.payload.uiContent) {
            // 迁移逻辑：把旧的单一对象包进新的 Tabs 数组
            rawEvent.payload.uiTabs = [{
              title: "Result",
              content: rawEvent.payload.uiContent
            }];
            delete rawEvent.payload.uiContent;
          }
          rawEvent.v = 2; // 标记升级完成
        }
        return rawEvent;
      }
      ```

5.  **隐私保护 Schema**：
    *   **方案**：在 Schema 中分离 `Data` 和 `References`。敏感数据加密存储在专用 Vault，日志中只存指针。
    *   **Schema 示例**：
      ```typescript
      interface SensitivePayload {
        // 原始字段（在落盘日志前被擦除/哈希化）
        userInput?: string; 
        
        // 脱敏后的用于分析的特征
        msgLength: number;
        intentCategory: string;
        
        // 指向加密保险箱的引用 (仅供最高权限 Debug 使用)
        secureRef: "vault://users/123/logs/abc-xyz";
      }
      ```
    *   *机制*：Logger 中间件在序列化前，检测 PII 字段，将其替换为 `***` 或生成 `secureRef`，确保普通日志文件中不含明文。

</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 "Fat Event" 综合症
*   **错误**：在 `Event` 中携带巨大的二进制数据（如 10MB 的 PDF 文件内容）。
*   **后果**：序列化/反序列化阻塞主线程，网络带宽耗尽，日志存储成本激增。
*   **Rule of Thumb**：**Payload 应该只包含元数据和引用**。文件内容应上传到 S3/Blob Storage，Event 中只传 `fileUrl` 或 `blobId`。

### 5.2 混淆 UI State 与 Business State
*   **错误**：在 `AgentState` 中存储 `isMenuOpen: boolean` 或 `hoverIndex: number`。
*   **后果**：后端逻辑被前端交互细节污染。当从 Web 迁移到 语音助手（无屏幕）时，这些状态变得毫无意义且难以维护。
*   **修正**：`AgentState` 只存储**业务意图**（如 `recommendedActions` 列表）。菜单是否打开是前端 View Model 的事，不要传回给 Agent。

### 5.3 不可序列化的对象
*   **错误**：`state.dbConnection = new MongoClient(...)` 或 `event.callback = () => { ... }`。
*   **后果**：无法保存快照，无法通过网络传输 State，Time Travel Debugging 失效。
*   **修正**：State 中只存 `connectionStatus: 'connected'`。具体的连接实例由 Runtime 的 `EffectManager` 在闭包中维护，不暴露给纯逻辑层。

### 5.4 缺乏版本号 (Versioning)
*   **错误**：直接修改 Event 结构而不加 `version` 字段。
*   **后果**：上线新版后，旧的客户端发来的事件导致后端 Crash；或者回放旧日志时报错。
*   **Rule of Thumb**：所有持久化的数据结构（State/Event）都应包含 `schemaVersion` 字段。
