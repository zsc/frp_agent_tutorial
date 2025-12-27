# Chapter 4｜Prompt Management：内嵌环境状态播报 + 时间变化

## 1. 开篇段落

在传统的脚本式 LLM 开发中，Prompt 往往是一个静态的字符串模版（String Template），开发者通过简单的变量替换（如 f-string）来生成输入。这种方式在一次性脚本中尚可，但在**长生命周期的实时（Real-time）Agent** 系统中，它面临着巨大的挑战：
1.  **时空感知**：世界在流逝，用户的位置在移动，Agent 需要“知道”当下是几点、在哪里，而不需要用户每次都提醒。
2.  **上下文爆炸**：随着对话进行，History 无限增长，静态模版无法处理动态压缩和遗忘。
3.  **竞态与一致性**：后台的总结任务、工具执行结果、新的用户输入同时到来，手动维护 Prompt 的状态机极其痛苦。
4.  **性能与成本**：不加区分地重组 Prompt 会导致 KV Cache 失效，增加延迟和 Token 消耗。

在 FRP 范式下，**Prompt 不是一个字符串，而是一个随时间变化的信号（Signal）**。它是当前“世界状态（Context）”、“对话历史（History）”与“系统指令（System）”的**纯函数映射**。本章将指导你构建一个**反应式 Prompt 管道（Reactive Prompt Pipeline）**，实现环境感知的自动播报、长上下文的流式压缩以及基于 Token 预算的动态裁剪。

---

## 2. 文字论述

### 2.1 范式转换：从 String 到 `Signal<Prompt>`

在 FRP 中，Prompt 是系统的**派生状态（Derived State）**。我们不再编写“构建 Prompt”的命令式代码，而是定义依赖关系。

$$ Prompt_t = \text{Compose}(\text{System}, \text{Env}_t, \text{Mem}_t, \text{Input}_t) $$

当依赖的任何上游信号发生变化时，Prompt Signal 会自动重新计算。

```ascii
Stream: Time Tick (1min) ----+
                             |
Stream: GPS Location --------+--> [ Environment Signal ]
                             |      { time: "10:00", loc: "Home" }
Stream: Battery Level -------+               |
                                             v
Stream: User Inputs  -----> [ History Accumulator ]
                                             |
                                             v
                         +---------------------------------------+
                         |      combineLatest Operator           |
                         |  (System + Env + History + Input)     |
                         +-------------------+-------------------+
                                             |
                                             v
                                  [ Prompt Signal ]
                         (Ready for LLM / Token Counter)
```

### 2.2 “环境状态播报”策略：让 Agent 拥有感官

为了 Agent 显得智能，它需要像生物一样感知环境。这种感知必须是**非侵入式**的。

#### 2.2.1 信号源与语义映射
原始传感器数据往往过于嘈杂，不适合直接喂给 LLM。我们需要在 FRP 管道中插入**语义转换层**。

*   **时间**：`Timestamp` -> `Formatting` ("2023年10月27日 星期五 下午")
*   **位置**：`Lat/Long` -> `Reverse Geocoding` ("北京市海淀区") -> `Semantic Label` ("公司" / "家里" / "移动中")
*   **设备**：`Battery` -> `Status` ("电量充足" / "低电量模式")

#### 2.2.2 变更检测与节流
如果 GPS 坐标每秒微变，Prompt 信号也会每秒更新，这可能引发不必要的计算。
我们需要使用 `distinctUntilChanged` 算子，并提供自定义的**比较函数（Comparator）**。

> **Rule of Thumb (语义边界原则)**：
> 只有当环境变化跨越了“语义边界”时，才更新 Prompt 信号。
> *   **时间**：从“上午”变“下午”，或每隔 15 分钟。
> *   **位置**：从“家”变“地铁站”（而非坐标变动 10 米）。
> *   **网络**：从 WiFi 变 4G。

### 2.3 时间变化与 Prompt Patch

在 FRP 中，时间是一个特殊的输入源（Tick Stream）。但这里有一个棘手的问题：**时间流逝是否应该触发 LLM 说话？**

通常我们采用 `withLatestFrom` 模式，而不是 `combineLatest` 模式来处理时间：

1.  **被动感知（Passive Awareness）**：
    *   主触发器：用户输入（User Input Stream）。
    *   操作：当用户说话时，`withLatestFrom(TimeSignal)` 抓取当前时间。
    *   效果：Agent 只有在回答时才知道时间，平时不消耗推理资源。

2.  **主动触发（Active Triggering）**：
    *   主触发器：Time Signal。
    *   逻辑：`TimeSignal.filter(t => is_alarm_time(t))`。
    *   效果：只有到了特定时刻（如闹钟、日程提醒），时间流逝才会导致 Agent 主动发言（详见 Chapter 10）。

### 2.4 Prompt 模板分层：KV Cache 化的关键

LLM 的推理是基于 Transformer 的，使用了 KV Cache（键值缓存）。如果 Prompt 的前缀（Prefix）保持不变，计算量可以大幅减少。因此，Prompt 的结构设计必须符合**变化频率从低到高**的物理规律。

```ascii
[ Layer 1: Static System ]  <-- 变化率: 0 (Hit Rate: 100%)
"你是一个且有爱心的助手..."
"安全准则..."

[ Layer 2: Slow Context ]   <-- 变化率: 低 (Hit Rate: ~80%)
"用户画像: 程序员, 喜欢科幻..."
"长期记忆摘要..."

[ Layer 3: Dynamic Env ]    <-- 变化率: 中 (Hit Rate: ~40%)
"当前时间: 14:30"
"当前位置: 办公室"
"后台任务状态: 搜索中..."

[ Layer 4: Conversation ]   <-- 变化率: 高 (Hit Rate: ~10% via Rolling)
User: Hi
AI: Hello
User: ...

[ Layer 5: Ephemeral ]      <-- 变化率: 极高 (Hit Rate: 0%)
(思维链草稿, 错误回退提示)
```

**实现技巧**：在 FRP 代码中，我们将 Prompt 定义为一个 `List<Block>`。每次更新时，只重新渲染变化的 Block，并在最终合并成 String 之前，保留 Block 的结构以便于 Token 估算。

### 2.5 长对话压缩：双流模型（Dual-Stream）

为了解决 Context Window 限制，我们需要一个“后台清洁工”。在 FRP 中，我们可以并行处理对话流。

1.  **Main Stream**：处理即时对话，包含最近 N 条消息（Sliding Window）。
2.  **Summary Stream**：监听 History 的变化。
    *   当 `History.length > Threshold` 时，触发**副作用（Effect）**。
    *   调用 LLM 将最旧的 M 条消息压缩为一段 Summary。
    *   **关键点**：这个过程是异步的，不阻塞用户当前的对话。
    *   完成后，发射 `UpdateHistoryEvent`，将旧消息替换为 Summary Block。

### 2.6 注入攻击防护与结构化隔离

Prompt Signal 是由可信源（System）和不可信源（User/Web）混合而成的。在拼接前，必须经过 `Sanitize Operator`。

*   **隔离策略**：永远不要直接拼接。使用 XML 标签或特殊 Token 形成隔断。
*   **转义**：如果用户输入中包含 `</user_input>` 等试图闭合标签的字符，必须转义。

```ascii
User Input Stream ---> [ Sanitize Operator ] ---> [ Wrapper Operator ]
"Ignore instructions"      (No-op)                <user_msg>
                                                    Ignore instructions
                                                  </user_msg>
```

### 2.7 Token 预算与动态裁剪

在移动端或高并发场景下，Token 预算是硬约束。FRP 允许我们定义动态的**裁剪策略（Trimming Strategy）**。

我们可以定义一个 `BudgetSignal`（例如：剩余可用 Token 数）。
Prompt Signal 在生成最终文本前，会经过一个 `FitToBudget` 算子：
1.  计算当前总 Token。
2.  如果超标，根据**优先级（Priority）**丢弃 Block。
    *   Priority 0 (Low): 工具调用的冗长 JSON 返回值（替换为结果摘要）。
    *   Priority 1 (Mid): 较早的 RAG 检索片段。
    *   Priority 2 (High): 用户的上一句话。
    *   Priority 3 (Critical): System Prompt。

---

## 3. 本章小结

*   **Prompt 即 Signal**：放弃字符串拼接，使用 `combineLatest` 组合多源信息（时间、位置、记忆、输入）。
*   **分层与缓存**：遵循“静态在前，动态在后”的物理布局，最大化利用 LLM 的 Prefix Caching，降低首字延迟。
*   **环境去噪**：使用 `distinctUntilChanged` 和语义映射，防止物理世界的微小波动造成 Prompt 的剧烈抖动。
*   **异步压缩**：利用 FRP 的并发能力，在后台流式地压缩历史记录，实现“无限对话”的幻觉。
*   **安全与预算**：将安全清洗和预算裁剪作为 Prompt 管道中的标准算子（Operators），确保每次发往 LLM 的请求都是合规且不超支的。

---

## 4. 练习题

### 基础题 (50%)

**Q1. 信号组合的基础**
假设你有两个信号流：`UserInputStream`（用户发送消息时触发）和 `TimeStream`（每分钟触发一次）。
你的目标是构建一个 `PromptStream`。
如果你直接使用 `combineLatest([UserInputStream, TimeStream])`，会发生什么问题？当用户不说话，只有时间流逝时，Prompt 会怎样？这是否是我们想要的？请提出改进的算子组合。

<details>
<summary><strong>参考答案</strong></summary>

*   **问题**：`combineLatest` 会在*任一*输入流更新时发射新值。如果 `TimeStream` 每分钟更新，`PromptStream` 也会每分钟发射一次新的 Prompt。这可能导致 Agent 在用户没说话时，突然因为 Prompt 变了而重新触发推理（如果下游没有防抖逻辑），或者产生大量的无用日志。
*   **期望行为**：通常我们只希望在**用户说话时**，获取**最新的时间**注入 Prompt。
*   **FRP 解法**：使用 `UserInputStream.withLatestFrom(TimeStream)`。
    *   主驱动是用户输入。
    *   时间只是作为附加数据被“抓取”过来。
    *   这样只有用户说话时，Prompt 才会更新。

</details>

**Q2. 简单的 Token 估算器**
Prompt 分为三部分：System (固定 100 tokens), History (动态), User (动态)。
设计一个 `map` 函数逻辑，确保总 Prompt 不超过 1000 tokens。如果超了，优先裁剪 History 的头部。

<details>
<summary><strong>参考答案</strong></summary>

*   **逻辑伪代码**：
    ```python
    def fit_budget(system, history_list, user_input):
        budget = 1000
        current_tokens = len(system) + len(user_input)
        available_for_history = budget - current_tokens

        if available_for_history < 0:
            return Error("User input too long")

        trimmed_history = []
        for msg in reversed(history_list): # 从最新的开始保留
            if len(msg) <= available_for_history:
                trimmed_history.insert(0, msg)
                available_for_history -= len(msg)
            else:
                break # 预算耗尽

        return construct_prompt(system, trimmed_history, user_input)
    ```
*   **FRP 算子**：这个逻辑会被封装在一个 `map` 或 `transform` 算子中，串联在 Prompt Signal 的末端。

</details>

**Q3. 环境变化的去噪**
你的 Agent 有一个 `BatteryLevelStream`，每秒通过 API 返回电量百分比（如 99.1%, 99.0%, 98.9%...）。
你希望只有在电量发生“显著变化”（每下降 20%）或者“状态改变”（充电中/未充电）时才更新 Prompt。请描述过滤器逻辑。

<details>
<summary><strong>参考答案</strong></summary>

*   **逻辑**：需要维护一个“上一次广播的状态”作为闭包或累加器。
*   **算子**：`scan` (用于状态维持) + `distinctUntilChanged`。
*   **实现**：
    1.  将原始流映射为语义对象：`{ level_tier: floor(level / 20), is_charging: status }`。例如 99% -> tier 4, 81% -> tier 4, 79% -> tier 3。
    2.  应用 `distinctUntilChanged`。
    3.  只有当 tier 改变（如从 4 变 3）或充电状态改变时，信号才会向下游传播。

</details>

### 挑战题 (50%)

**Q4. 上下文快照与“幽灵读取” (Snapshot Latching)**
场景：用户问“现在几点了？”（T1时刻）。
FRP 管道构建 Prompt 需要 50ms。LLM 推理需要 2秒。
在 T1+100ms 时，`TimeStream` 更新了（跳了一分钟）。
在 T1+1s 时，用户位置更新了（从家里变到了路上）。
如果你的 Agent 正在流式输出回答，中间 Prompt Signal 变了，会发生什么？如何在 FRP 中确保 LLM 处理的是 T1 时刻的“快照”？

<details>
<summary><strong>参考答案</strong></summary>

*   **风险**：如果在生成过程中读取了新的 Signal，可能会导致生成的回答前后不一致（前半句基于旧时间，后半句基于新时间，如果代码逻辑是动态拉取的话），或者触发不必要的重新推理。
*   **解法**：**采样与锁定（Sample & Hold / Latch）**。
    *   当推理动作（Effect）被触发时，应该**捕获（Take）** 当前所有依赖信号的瞬时值，构建一个不可变的 `GenerationContext` 对象。
    *   这个对象传给 LLM 函数。
    *   在 LLM 完成之前，即使上游 Signal 变了，正在运行的 Effect 也不受影响。
    *   UI 层可以选择是否显示“环境已更新”的提示，但不能打断当前的推理流（除非有显式的 Cancellation 逻辑）。

</details>

**Q5. 异步摘要的一致性难题 (Advanced)**
你实现了一个后台摘要流。
状态：History = [A, B, C, D, E]。
触发摘要：对 [A, B, C] 进行摘要 -> Summary(ABC)。
**问题**：摘要是个耗时操作。在摘要生成期间，用户发了消息 F。
当前 History 变成了 [A, B, C, D, E, F]。
摘要完成，得到了 `Sum_ABC`。
如果在摘要回调中简单地执行 `History = [Sum_ABC] + [D, E]`，而忽略了 F，就会丢失消息 F。
请设计一个基于 Version Vector 或 Operational Transformation (OT) 思想的 FRP 状态更新机制。

<details>
<summary><strong>参考答案</strong></summary>

*   **核心思想**：状态更新必须是原性的，且基于“变更意图”而非“覆盖”。
*   **FRP 架构**：
    1.  **State**：`{ messages: [...], base_version: 5 }` (假设 E 是第 5 条)。
    2.  **User Action**：`AddMessage(F)` -> reducer -> `messages: [..., F], base_version: 6`。
    3.  **Summary Action**：
        *   Start: 记录 `target_range: 0..2` (A,B,C), `snapshot_version: 5`.
        *   Complete: 发射 `ApplySummaryEvent(summary, target_range, snapshot_version)`.
    4.  **Reducer 处理 ApplySummaryEvent**：
        *   检查 `target_range` 的消息是否还在（或者是否已经被别的摘要合并了）。
        *   将 `messages` 中对应 range 的部分替换为 summary。
        *   **关键**：由于 F 是追加在尾部的，它不在 `target_range` 内，因此替换操作是安全的：`[Sum_ABC, D, E, F]`。
        *   如果发生了冲突（比如另一条流删除了 B），则放弃本次摘要或重新计算。

</details>

**Q6. Prompt A/B 测试的动态路由**
你需要同时测两个不同版本的 System Prompt（版本 A 严肃，版本 B 幽默）。
要求：
1. 同一个会话（Session）必须保持版本一致。
2. 可以在运行时动态调整 A/B 的流量比例（例如从 50/50 调到 80/20）。
3. 这是一个 Configuration Signal。
请在 FRP 管道中设计这个路由逻辑。

<details>
<summary><strong>参考答案</strong></summary>

*   **设计**：
    1.  **Config Signal**：`{ version_a_ratio: 0.8 }`。
    2.  **Session Stream**：新会话开始时触发。
    3.  **Assignment Logic**：
        *   当 `SessionStart` 事件发生时，`withLatestFrom(ConfigSignal)`。
        *   根据随机数和 ratio 决定该 Session 的 `prompt_variant`。
        *   **Sticky Session**：将这个 decision 存入 `SessionState`。
    4.  **Prompt Construction**：
        *   `combineLatest(SessionState, ...)`
        *   `map` 函数中：`if state.variant == 'A' then use TemplateA else use TemplateB`.
    *   **关键点**：配置的变更不会影响*正在进行*的会话（保持一致性），只会影响*新*会话（因为路由决策是在 SessionStart 时做的 snapshot）。

</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 The "Ticking Bomb" (秒针炸弹)
*   **现象**：开发者为了精确，将时间格式化为 `"14:30:01"` 放入 Prompt。
*   **后果**：
    *   每一秒钟，Time Signal 都会变化。
    *   每一秒钟，Prompt Signal 都会更新。
    *   如果下游 UI 绑定了 Prompt Signal，UI 会疯狂闪烁。
    *   如果使用了 Debug 日志，磁盘会被写满。
    *   **最严重**：如果 Agent 的触发逻辑写成了 `combineLatest` 且没有防抖，Agent 可能会每秒钟都试图对这句话进行重新推理（虽然大部分 LLM API 还没返回上一句）。
*   **修正**：只精确到分钟，或者仅在 System Prompt 中更新时间，而不触发推理流。

### 5.2 缓存抖动 (Cache Thrashing)
*   **现象**：为了方便，每次构建 Prompt 时，随机打乱了例（Few-shot examples）的顺序，或者把 System Prompt 放在了 User Input 之后。
*   **后果**：LLM 无法利用 Prefix Caching。每次请求都需要重新处理所有 Token，导致首字延迟（TTFT）增加 2-5 倍，成本飙升。
*   **修正**：保持 Prompt 前缀的**字节级**一致性。静态内容永远放在最前。

### 5.3 幽灵上下文 (The "Ghost" Context)
*   **现象**：用户问“今天天气如何？” Agent 回答了。然后用户带着设备移动到了另一个城市。位置信号更新了。用户问“附近的餐馆？”
*   **Bug**：Agent 仍然推荐上一个城市的餐馆。
*   **原因**：Prompt Signal 虽然更新了，但 Agent 的 `SessionState` 或 `Memory` 中可能缓存了上一轮的思维链（Chain of Thought），其中包含了旧地点的“短期记忆”。
*   **修正**：当检测到关键环境变化（如城市变更）时，除了更新 Prompt，还应该发射一个 `ClearShortTermMemory` 事件，清除思维链中的陈旧信息。

### 5.4 所有的 Prompt 都是 f-string
*   **现象**：代码中充斥着 `prompt = f"..."`。
*   **后果**：难以维护，难以测试，无法统一处理安全性（Sanitization）和预算（Budgeting）。
*   **修正**：建立 `PromptBuilder` 对象或函数管道，将模版、数据、配置分离。

### 5.5 忽略了 Tokenizer 的差异
*   **现象**：使用 `string.length` 或简单的 `split(' ')` 来估算 Token 预算。
*   **后果**：对于中文或代码内容，估算误差极大。导致发送给 API 时超过上下文窗口限制（Context Window Exceeded Error）。
*   **修正**：在 FRP 管道中集成真实的 Tokenizer（如 `tiktoken` 或 `transformers`）。为了性能，可以建立一个 `LengthSignal`，只有当文本变化时才重新计算 Token 数。
