# Chapter 17｜参考实现：一个端到端的 FRP Agent

## 1. 开篇与目标

设计一个实时 Agent 系统最难的不是调用 `openai.chat.completions.create`，而是处理“混乱”：用户在模型输出一半时打断、网络波动导致工具超时、并发请求引发的竞态条件、以及 Token 预算的动态管理。

本章将通过一套完整的**参考架构（Reference Architecture）**来解决这些混乱。我们使用类 Python 的伪代码（结合 Rx/ReactiveX 语义）来构建系统。

**本章学习目标**：
1.  **架构落地**：掌握“六边形架构”在 FRP Agent 中的具体目录结构与职责划分。
2.  **核心循环**：实现从 `Input Event` 到 `State Update` 再到 `Side Effect` 的闭环。
3.  **高级流控**：用代码实现 Speculative Decoding（投机解码）与 Dynamic Batching（动态批处理）。
4.  **全链路集成**：将 RAG、工具调用、预算控制和 UI 渲染整合到一个统一的流中。

---

## 2. 系统全景与目录结构

我们的架构遵循**纯逻辑内核（Functional Core）**与**命令式外壳（Imperative Shell）**分离的原则。

### 2.1 目录结构 (Project Layout)

```text
/src
  /agent_core               <-- [纯逻辑层] 无副作用，纯函数，可完全回放测试
    ├── events.py           # 定义所有事件 (UserAction, SystemSignal, ToolReceipt)
    ├── state.py            # 定义不可变状态树 (AgentState, ConversationHistory)
    ├── reducers.py         # 状态转移函数: (State, Event) -> State
    ├── selectors.py        # 派生状态逻辑: State -> Prompt, State -> UI_ViewModel
    └── triggers.py         # 复杂的触发器逻辑 (VAD, Interrupt logic)

  /runtime                  <-- [运行时层] 负责流的组装与调度
    ├── pipeline.py         # 主管道 wiring (merge, scan, map)
    ├── scheduler.py        # 批处理与速率限制策略
    └── loop.py             # 主循环入口

  /effects                  <-- [副作用层] 只有这里可以发起网络请求/IO
    ├── llm_gateway.py      # LLM API 包装 (支持 Stream/Batch/Speculative)
    ├── tool_runner.py      # 工具执行器 (Sandbox, Retry, CircuitBreaker)
    ├── rag_service.py      # 向量数据库检索与重排
    └── memory_store.py     # 持久化存储 (Postgres/Redis)

  /interfaces               <-- [适配器层] 输入输出转换
    ── websocket_server.py # WebSocket 协议适配
    ├── audio_processor.py  # 音频流分帧与 VAD 预处理
    └── debug_console.py    # 开发者后台数据流

  /tests                    <-- [测试层]
    ├── test_reducers.py    # 单元测试
    └── sim_scenarios.py    # 虚拟时间回放测试 (Time-Travel Testing)
```

### 2.2 核心数据流架构 (The Architecture Diagram)

这个 ASCII 图展示了数据如何在不同层级间流动。

```text
       [  Real World  ]
            | (Inputs: Audio/Text/Click)
            v
    +-------+-------+
    |   Interfaces  |  -> Normalize to Events
    +-------+-------+
            |  Stream<Event> (UserEvent, TimerEvent)
            v
    +-------+-------+      +----------------------+
    |  Runtime Hub  |<-----|  Effect Feedback Loop|
    +-------+-------+      +----------+-----------+
            |                         ^
            | (Merge All Streams)     | Stream<Event> (ToolResult, Error)
            v                         |
    +-------+-------+                 |
    |   Agent Core  |                 |
    | (Scan/Reduce) |                 |
    +-------+-------+                 |
            | Stream<State>           |
            v                         |
    +-------+-------+      +----------+-----------+
    | State Selector|----->|    Effect Runners    |
    +-------+-------+      | (LLM, Tools, RAG)    |
            |              +----------------------+
            | Derived View (UI Frame, Audio Buffer)
            v
    +-------+-------+
    |   Interfaces  |  -> Render to User
    +-------+-------+
```

---

## 3. 核心运行时：Event Bus 与 Stream Loop

这是系统的脊梁。我们不使用显式的回调函数，而是定义“流的拓扑结构”。

### 3.1 定义基本类型

```python
from typing import TypeVar, Generic, Union, Dict, Any
from dataclasses import dataclass
from datetime import datetime

# 基础事件定义
@dataclass(frozen=True)
class Event:
    id: str
    timestamp: float
    type: str
    payload: Dict[str, Any]

# 不可变状态定义
@dataclass(frozen=True)
class AgentState:
    history: list               # 对话历史
    variables: Dict[str, Any]   # 提取的槽位 (Slot Filling)
    pending_actions: list       # 待执行的副作用意图
    status: str                 # 'IDLE', 'THINKING', 'ACTING', 'SPEAKING'
    last_active: float          # 最后活跃时间
    token_usage: dict           # 预算消耗

    def copy(self, **changes):
        return replace(self, **changes)
```

### 3.2 主管道组装 (The Pipeline Wiring)

这段代码是 Agent 的“启动脚本”，它不执行逻辑，只连接电缆。

```python
def build_agent_pipeline(inputs: InputStreams, services: ServiceContainer):
    """
    组装 FRP 管道。
    inputs: 包含 user_input$, interrupt$, system_tick$ 等源流
    services: 包含 llm, tools, db 等副作用执行器
    """

    # 1. 反馈回路 (Feedback Loop) 的占位符
    # 这是一个 Subject，允许我们在管道下游把结果塞回上游
    action_result_stream = Subject()

    # 2. 合并所有事件源
    # - 用户输入
    # - 定时器心跳 (用于超时检查、环境播报)
    # - 工具/LLM 的执行结果
    all_events$ = merge(
        inputs.user_input$.map(lambda x: UserEvent(x)),
        inputs.interrupt$.map(lambda _: InterruptEvent()),
        inputs.system_tick$.map(lambda t: TickEvent(t)),
        action_result_stream  # 回环入口
    )

    # 3. 核心状态机 (The Brain)
    # 所有的业务逻辑都在 reducer 中
    state$ = all_events$.scan(
        reducer=core_reducer,
        seed=AgentState.initial()
    ).share()  # 广播给多个消费者

    # 4. 副作用意图识别 (Intent Detection)
    # 检查状态是否暗示需要执行外部操作 (调用 LLM, 搜索 DB)
    effect_intent$ = state$.map(lambda s: s.pending_actions) \
                           .distinct_until_changed() \
                           .filter(lambda actions: len(actions) > 0)

    # 5. 作用分发与执行 (Side Effect Dispatcher)
    # 将 Intent 路由给不同的执行器，并将结果发回 action_result_stream
    effect_intent$.subscribe(
        lambda actions: dispatch_effects(actions, services, action_result_stream)
    )

    # 6. UI 渲染流
    # 使用 throttle 避免 UI 刷新过快（如 60fps）
    ui_view$ = state$.throttle_time(0.016) \
                     .map(lambda s: render_ui_view_model(s))

    return ui_view$
```

---

## 4. PromptSignal：从状态到动态 Prompt

在 FRP 架构中，Prompt 不是硬编码的字符串，而是**状态的函数**。

### 4.1 动态 Prompt 构造器

```python
def derive_prompt(state: AgentState, config: Config) -> list[dict]:
    """
    Selector 函数：纯函数，(State) -> Messages
    """
    
    # 1. 动态系统指令 (System Prompt)
    # 包含时间、位置、当前模式
    system_content = config.base_prompt_template.format(
        time=datetime.fromtimestamp(state.clock_now).isoformat(),
        location=state.user_context.get('location', 'Unknown'),
        tool_status=format_tool_status(state.tool_registry)
    )

    # 2. 注入短期记忆与 RAG 上下文
    # 注意：RAG 的结果是在之前的 reducer 步骤中存入 state 的
    context_msgs = []
    if state.rag_evidence:
        context_msgs.append({
            "role": "system",
            "content": f"Relevant Context:\n{format_evidence(state.rag_evidence)}"
        })

    # 3. 历史记录压缩 (Context Budgeting)
    # 只有这里做截断，保持 State 里的 history 是完整的
    visible_history = smart_trim(state.history, max_tokens=config.context_window)

    return [{"role": "system", "content": system_content}] + \
           context_msgs + \
           visible_history
```

---

## 5. Tool Effect 执行器与回执

工具调用是异步的、易错的。我们需要优雅地处理它们。

### 5.1 Reducer 中的意图生成

```python
def core_reducer(state: AgentState, event: Event) -> AgentState:
    # ... 省略其他事件处理 ...
    
    if event.type == "LLM_RESPONSE_CHUNK":
        # 如果 LLM 输出表明要调用工具
        if is_tool_call_token(event.payload):
            # 将“待执行工具”加入 pending_actions
            # 注意：这里不执行工具，只描述“我想执行工具”
            new_action = ToolAction(
                id=uuid.uuid4(),
                tool_name=event.payload['name'],
                params=event.payload['args']
            )
            return state.copy(
                pending_actions=state.pending_actions + [new_action],
                status='ACTING'
            )
            
    return state
```

### 5.2 执行器 (The Runner)

```python
def dispatch_effects(actions: list[Action], services, result_sink: Subject):
    for action in actions:
        if isinstance(action, ToolAction):
            # 启动异步任务
            asyncio.create_task(
                run_tool_safely(action, services.tools, result_sink)
            )

async def run_tool_safely(action, tool_registry, result_sink):
    try:
        # Chapter 6: Circuit Breaker & Retry 逻辑在这里封装
        tool_func = tool_registry.get(action.tool_name)
        result_data = await tool_func(**action.params)
        
        event = Event(
            type="TOOL_RESULT",
            payload={
                "action_id": action.id,  # 关联 ID
                "status": "SUCCESS",
                "data": result_data,
                "receipt": generate_receipt(action, result_data) # 生成回执
            }
        )
    except Exception as e:
        event = Event(
            type="TOOL_ERROR",
            payload={"action_id": action.id, "error": str(e)}
        )
    
    # 将结果推回主管道
    result_sink.on_next(event)
```

---

## 6. Speculative Decoding 与 Batching

为了极致的性能，我们在 `Effect` 层引入高级流控。

### 6.1 Speculative Execution (投机执行) 的 FRP 实现

假设我们要在用户说话时预加载可能的工具（例如，用户提到“天气”，我们不仅预加载天气工具，甚至可以投机性地发起查询）。

```python
# 在 pipeline 组装层
def attach_speculative_layer(user_input_stream$, services):
    
    # 1. 快速意图识别流 (使用小模型)
    intent_stream$ = user_input_stream$ \
        .buffer_time(0.5) \  # 每 0.5 秒采样一次
        .filter(lambda buf: len(buf) > 0) \
        .switch_map(lambda text: services.small_model.predict_intent(text))
    
    # 2. 投机性预热 (Prefetch)
    intent_stream$.subscribe(lambda intent: 
        if intent == 'search':
            services.search_engine.warmup()
        elif intent == 'db_query':
            services.db_connection.ensure_active()
    )
```

### 6.2 Dynamic Batching (RAG 向量化)

```python
class EmbeddingService:
    def __init__(self):
        self._request_subject = Subject()
        
        # 内部批处理逻辑
        self._response_stream = self._request_subject \
            .buffer_with_time_or_count(time_span=0.05, count=64) \
            .filter(lambda batch: len(batch) > 0) \
            .flat_map(self._execute_batch_request) \
            .share()

    def get_embedding(self, text):
        # 返回一个 Future/Promise，等待流的响应
        req_id = uuid.uuid4()
        self._request_subject.on_next({"id": req_id, "text": text})
        
        return self._response_stream \
            .filter(lambda res: res['id'] == req_id) \
            .first()
            
    async def _execute_batch_request(self, batch):
        texts = [item['text'] for item in batch]
        # 真正发起一次 API 调用
        vectors = await self.api_client.embed(texts)
        # 重新拆分结果
        return [{"id": item['id'], "vector": vec} 
                for item, vec in zip(batch, vectors)]
```

---

## 7. RAG 管道与流式证据

实现“边生成，边展示引用”的效果。

```python
def rag_effect_runner(query, rag_service, result_sink):
    # RAG 不仅仅返回文本，还返回 Stream<Evidence>
    evidence_stream = rag_service.search_stream(query)
    
    evidence_stream.subscribe(
        on_next=lambda doc: result_sink.on_next(
            Event("RAG_DOC_FOUND", payload=doc)
        ),
        on_completed=lambda: result_sink.on_next(
            Event("RAG_SEARCH_FINISHED", payload={})
        )
    )

# 在 Reducer 中
def reducer(state, event):
    if event.type == "RAG_DOC_FOUND":
        # 增量更新证据列表，UI 可以立即渲染这个新文档
        return state.copy(
            evidence_list=state.evidence_list + [event.payload]
        )
    # ...
```

---

## 8. Trigger DSL (类似 VAD 的触发器)

```python
from operators import debounce, throttle

def create_advanced_trigger(audio_energy$, text_stream$):
    """
    定义什么时候 Agent 应该打断用户或开始说话。
    """
    
    # 1. 物理层 VAD：声音能量 > 阈值，持续 200ms
    is_speaking$ = audio_energy$ \
        .map(lambda e: e > 0.5) \
        .distinct_until_changed() \
        .debounce(0.2) 
        
    # 2. 语义层 VAD：检测到完整句子结束符 (Semantic Endpointing)
    is_sentence_end$ = text_stream$ \
        .filter(lambda t: t.endswith(('?', '.', '!')))
        
    # 3. 组合逻辑
    # 如果检测到“说话结束” 且 “句子完整”，触发 TURN_END
    turn_end$ = is_speaking$ \
        .filter(lambda speaking: speaking is False) \
        .with_latest_from(is_sentence_end$) \
        .map(lambda _: Event("USER_TURN_END"))
        
    return turn_end$
```

---

## 9. 本章小结

本章通过伪代码构建了一个完整的 FRP Agent 骨架：

1.  **Event Bus** 是血管，连接所有组件。
2.  **Reducer** 是心脏，纯净且负责状态维护。
3.  **Selector** 是大脑皮层，将状态转化为 Prompt 和 UI。
4.  **Effect Runners** 是四肢，负责处理肮脏的现实世界（IO/API）。

这个架构的威力在于：
*   **可测试性**：你可以编写一个 JSON 文件包含一系列 Event，喂给 Reducer，验证最终状态是否正确完全不需要 Mock 网络请求。
*   **可扩展性**：想加一个“情绪检测”模块？只需订阅 `audio_stream$`，产生 `EMOTION_DETECTED` 事件，并在 Reducer 中处理即可，无需修改现有的 LLM 逻辑。
*   **鲁棒性**：通过 `timeout`, `retry`, `circuit_breaker` 等算子，系统天生具备容错能力。

---

## 10. 练习题

### 基础题

1.  **Schema 设计**：
    设计 `ToolReceipt` 的数据结构。要求包含：调用工具的原始参数、工具返回的结果、以及一个 summary 字段（用于给 LLM 看的简化版结果，节省 Token）。
    <details>
    <summary>参考答案</summary>

    ```python
    @dataclass
    class ToolReceipt:
        tool_name: str
        call_id: str
        timestamp: float
        
        # 原始输入输出（用于日志和 Debug）
        raw_input: Dict[str, Any]
        raw_output: Any
        
        # 压缩视图（用于 Prompt）
        summary_for_llm: str 
        
        # 状态元数据
        is_success: bool
        execution_time_ms: int
    ```
    </details>

2.  **心跳实现**：
    使用你熟悉的语言（或伪代码），写一个 `tick_stream`，每秒发出一个事件，但如果 `state.status == 'SLEEPING'`，则停止发射事件以节省资源。
    <details>
    <summary>参考答案</summary>

    ```python
    # 需要 switch_map 配合状态流
    tick_stream$ = state_stream$ \
        .map(lambda s: s.status) \
        .distinct_until_changed() \
        .switch_map(lambda status: 
            interval(1.0) if status != 'SLEEPING' else empty()
        )
    ```
    </details>

3.  **日志脱敏**：
    实现一个简单的 Operator `mask_pii`，它接收 Event 流，将 payload 中的 `phone_number` 和 `email` 字段替换为 `***`。
    <details>
    <summary>提示</summary>
    这是一个 `map` 操作。尽量做深拷贝（deep copy）以避免修改原始事件影响其他消费者。可以使用正则表达式匹配常见的 PII 模式。
    </details>

### 挑战题

4.  **实现“乐观 UI 更新 (Optimistic UI Update)”**：
    当用户发送消息时，UI 立即显示“发送成功”（灰色状态），而不需要等待服务器确认。请描述在 FRP Reducer 中如何处理 `USER_SEND`（乐观）和 `SERVER_ACK`（确认）这两个事件，以及如果收到 `SERVER_ERROR` 如何回滚。
    <details>
    <summary>提示</summary>
    1. `USER_SEND`: append message to history with `status='pending'`.
    2. `SERVER_ACK`: find message by ID, update `status='sent'`.
    3. `SERVER_ERROR`: find message by ID, update `status='failed'` or remove it from history (rollback).
    关键在于每条消息需要一个临时的客户端生成的 UUID。
    </details>

5.  **设计 Token 预算熔断器**：
    在 `dispatch_effects` 之前拦截 LLM 请求。如果 `current_session_tokens > MAX_BUDGET`，则拦截请求，并自动派发一个 `Event("SYSTEM_MSG", "Budget Exceeded")` 给 Reducer，而不是调用 LLM API。
    <details>
    <summary>提示</summary>
    使用 `compose` 或 `pipe` 在流中插入逻辑。
    ```python
    llm_intent$
        .with_latest_from(budget_status$)
        .map(lambda (intent, budget): 
             if budget.remaining < 0: return SystemEvent("BUDGET_EXCEEDED")
             else: return intent
        )
    ```
    </details>

---

## 11. 常见陷阱与错误 (Gotchas)

### 11.1 无限递归 (Infinite Loops)
**场景**：Reducer 更新状态 -> 状态触发 Effect -> Effect 产生 Event -> Event 触发 Reducer。
**陷阱**：如果不小心，这会变成永动机。例如：Effect 失败发出 ERROR 事件，Reducer 收到 ERROR 决定重试，再次触发 Effect。
**对策**：
1.  **重试计数器**：在 Event payload 或 State 中必须包含 `retry_count`。
2.  **Base Condition**：Reducer 必须有逻辑 `if retry_count > 3: return state (give up)`。

### 11.2 幽灵订阅 (Ghost Subscriptions)
**场景**：在 Web 环境中，组件卸载了（如用户关闭了标签页或切换了由），但后台的 Stream 订阅还在运行，继续消耗 CPU 处理 VAD 或网络请求。
**对策**：
1.  确保所有的 `subscribe()` 返回的 `Subscription` 对象被收集。
2.  在销毁生命周期（如 `dispose()` 或 `__exit__`）中调用 `subscription.unsubscribe()`。
3.  使用 `take_until(destroy_signal$)` 模式自动管理生命周期。

### 11.3 状态的可变性污染
**场景**：在 Reducer 中直接修改了 `state.history.append(...)` 而不是返回一个新的 State 对象。
**后果**：由于 FRP 经常使用 `distinct_until_changed` 来优化性能，如果对象引用没变（只是内部内容变了），下游可能不会检测到变化，导致 UI 不刷新。
**对策**：始终使用 `dataclasses.replace` 或类似库（如 immer.js / pyrsistent）来保证不可变性。
