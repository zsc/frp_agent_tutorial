# [Chapter 10｜事件触发器：做一个“类似 VAD”的 Agent Trigger System](chapter10.md)

## 1. 开篇

在传统的 Request-Response（请求-响应）模型中，LLM 是被动的：它静静地等待用户按下“发送”键。但在**实时（Real-time）Agent** 的世界里，Agent 必须是一个**主动的观察者**。它需要像生物一样，从连续不断的、充满噪声的环境信号中，敏锐地捕捉到“变化的瞬间”。

在语音交互中，**VAD (Voice Activity Detection)** 负责判断“什么时候有人在说话”。如果 VAD 做得不好，Agent 要么会像个聋子（漏听），要么会像个神经质（把风声当人声）。

我们将这一概念化为 **Semantic VAD（语义/意图活动检测）**。本章将教你如何使用 FRP 的“流”和“算子”，构建一个通用的触发器系统。这个系统不仅能检测声音，还能检测**沉默、打断、股价波动、数据库变更**，甚至是**用户的犹豫**。

**本章目标**：
1.  **泛化 VAD 哲学**：理解 Attack（起势）、Sustain（保持）、Release（释放）在非语音场景的应用。
2.  **信号处理工程**：掌握 Debounce（去抖）、Throttle（节流）、Hysteresis（迟滞/施密特触发器）等核心稳定性算法。
3.  **DSL 设计**：构建一套声明式的 `when(...).for(...).then(...)` 语法来描述触发逻辑。
4.  **联动与抑制**：解决触发器之间的冲突（Race），以及如何将触发事件转化为 Prompt 上下文。

---

## 2. 文字论述

### 2.1 触发器的分类学：从离散到连续

在 FRP 视角下，触发器是将**连续变化的信号（Signal）** 坍缩为 **离散的事件（Event）** 过程。我们可以将触发器分为四类：

1.  **阈值触发 (Threshold Trigger)**：最类似 VAD。
    *   *例*：麦克风音量 > -20dB；用户愤怒指数 > 0.8；股票跌幅 > 5%。
2.  **模式触发 (Pattern Trigger)**：基于时间序列的特征。
    *   *例*：连续 3 次工具调用失败；用户输入了 "Hello" 紧接着又撤回了。
3.  **状态触发 (State Trigger)**：基于“没有事情发生”或“特定状态持续”。
    *   *例*：**尴尬的沉默**（双方都不说话超过 5 秒）；任务执行卡死（Loading 状态超过 30 秒）。
4.  **外部事件 (External Trigger)**：系统信号。
    *   *例*：定时器 Tick；API Webhook 回调；用户点击 UI 按钮。

### 2.2 核心算法：让触发器“抗噪”

真实世界的信号是脏的。光线会闪烁，麦克风有底噪，情感模型的输出会跳变。直接 `if (value > threshold) trigger()` 是新手的典型错误，会导致 Agent 疯狂抽搐。

#### A. 迟滞比较器 (Hysteresis / Schmitt Trigger)
这是 VAD 稳定的基石。我们需要**两个阈值**，构建一个“死区（Dead Zone）”。
*   **High Threshold (开启阈值)**：必须超过这个值，状态才变为 ON。
*   **Low Threshold (关闭阈值)**：必须低于这个值，状态才变为 OFF。

**ASCII 图解：迟滞效应**

```text
Input Value (Signal)
   ^
   |        /^\       (Noise spike)
 T_High |--/---\-----/--\---------- (Trigger ON)
   |      /     \   /    \      /--\
 T_Low |--/------\-/------\----/----\---- (Trigger OFF)
   |     /        V        \  /      \
   |----/-------------------\/--------\--> Time

Output State (Clean Event Stream)
   ^
 ON|       _______           __________
   |______|       |_________|          |__
```
*注：如果没有 T_Low，中间那个小波谷（V 处）就会导致一次错误的“关闭-再开启”，造成 Agent 语无伦次。*

#### B. 时间滤波：Debounce vs. Throttle vs. Window

FRP 提供了时间维度的滤波器，这是防止“帕森式”触发的关键。

| 算子 | 行为描述 | 适用场景 |
| :--- | :--- | :--- |
| **Debounce** (防抖) | “等它停稳了再说”。只有当信号停止变化 T 秒后，才发射最新值。 | 用户停止输入检测；搜索框自动补全。 |
| **Throttle** (节流) | “每 T 秒最多处理一次”。无论输入多快，按固定节奏采样。 | 进度条更新；高频传感器数据的日志记录。 |
| **Audit/Sample** | “当另一个流触发时，采样当前流的值”。 | 当用户点击“发送”时，采样当前的地理位置。 |
| **Window/Buffer** | “收集过去 T 秒内的所有事件”。 | 检测“短时间内的高频错误”（如 10s 内 5 次 API 失败）。 |

### 2.3 泛化 VAD 模型：ASR 包络

音频处理中的 **ADSR (Attack, Decay, Sustain, Release)** 包络模型可以完美映射到 Agent 的触发逻辑中。

1.  **Attack (起势)**：信号超过阈值需要持续多久才算触发？
    *   *防误触*：门关的声音很大但只有 200ms，不应触发“用户说话”。
    *   *FRP 实现*：`signal.filter(> thresh).window(time).filter(all_true)`
2.  **Sustain (保持)**：触发后，即使信号短暂跌落，状态依然保持。
    *   *防断音*：用户说话中间的停顿（Hysteresis 解决一部分，Sustain 解决更长的停顿）。
3.  **Release (释放)**：信号消失多久后，才判定事件结束？
    *   *确定终点*：决定何时调用 LLM 进行回复。

### 2.4 Trigger DSL：声明式定义

为了管理复杂的触发逻辑，我们不应该写死代码，而应定义 DSL。

**Rule-of-Thumb**：DSL 应该读起来像英语句子。

```text
// 场景：检测“用户困惑”
Trigger("UserConfused")
  .fromSource(UserFaceExpressionStream)  // 数据源：面部表情
  .map(data => data.confusionScore)      // 提取特征
  .filter(score => score > 0.8)          // 阈值判定
  .sustain(3.seconds)                    // 必须持续困惑3秒（防止眨眼误判）
  .cooldown(30.seconds)                  // 触发一次后，30秒内不再触发（防止骚扰）
  .suppressIf(AgentIsSpeaking)           // 抑制条件：如果 Agent 正在解释，别打断
  .emitEvent({ type: "help_needed" })    // 产生事件
```

### 2.5 触发器与 Prompt 的联动

触发器不是终点，而是 Prompt 工程的起点。触发事件必须携带**上下文快照**。

当 `Trigger` 激活时，它应该生成一个结构化的 **Observation**，注入到 LLM 的上下文中：

*   **Bad**: `[System]: User is confused.` (LLM 不知道为什么)
*   **Good**: `[System Event]: Trigger 'UserConfused' fired. Reason: Facial confusion score 0.85 for 3s. Current Topic: 'Quantum Physics'. Suggestion: Offer simplified explanation.`

### 2.6 抑制（Suppression）与优先级

在多触发器系统中，**冲突**是必然的。
*   **场景**：VAD 检测到用户说话（想打断），但同时“低电量报警”触发了。
*   **优先级仲裁**：
    *   `Safety > UserInterrupt > SystemCorrection > IdleChat`
*   **FRP 实现**：
    *   使用 `combineLatest` 合并所有触发流。
    *   通过 `Gate` 或 `Filter` 算子，当高优先级流为 Active 时，阻断低优先级流。

---

## 3. 本章小结

*   **万物皆 VAD**：任何从连续数据中提取离散意图的过程，都应遵循 Attack-Sustain-Release 的包络模型。
*   **稳定性是生命线**：没有迟滞（Hysteresis）和去抖（Debounce）的触发器是不可用的。务必处理好信号边缘的抖动。
*   **上下文感知**：触发器不仅是开关，它是信息的载体。触发时必须捕获当时的“案发现场”（Snapshot）给 LLM。
*   **声明式优于命令式**：使用 DSL 组合算子，而不是编写嵌套的 `if-else` 和 `setTimeout`。

---

## 4. 练习题

### 基础题（熟悉概念与算子）

**Q1: 解释为什么在检测“用户停止输入”时，`Debounce` 是比 `Throttle` 更正确的选择？**
<details>
<summary>点击查看答案</summary>

*   **提示**想象用户在连续打字。
*   **答案**：
    *   **Throttle** 会按照固定的时间间隔（例如每 500ms）触发一次。如果用户连续打字 10 秒，Throttle 会触发 20 次中间状态，这会导致 Agent 在用户还没写完句子时就抢答。
    *   **Debounce** 的逻辑是“只有当信号流**静止**了 T 时间后，才发射最后一个值”。这完美符合“用户打完字并停顿思考”的语义，保证 Agent 拿到的是完整的句子。
</details>

**Q2: 假设你有一个 `MicVolume`（麦克风音量）流（0.0 ~ 1.0）。请用伪代码或文字描述，如何实现一个带有迟滞的 VAD 逻辑：开启阈值 0.6，关闭阈值 0.2，初始状态为 OFF。**
<details>
<summary>点击查看答案</summary>

*   **提示**：使用 `scan` 或状态机变量。
*   **答案**：
    *   **核心逻辑**：
        ```javascript
        Stream(MicVolume).scan((isTalking, volume) => {
            if (!isTalking && volume > 0.6) return true;  // 超过高阈值 -> 开启
            if (isTalking && volume < 0.2) return false; // 低于低阈值 -> 关闭
            return isTalking;                            // 否则保持死区内的状态
        }, false) // 初始值为 false
        .distinctUntilChanged(); // 只在状态真正改变时发射事件
        ```
</details>

**Q3: 什么是“幽灵触发（Ghost Trigger）”？在多模态场景下（语音+视觉），如何利用多模态融合来减少幽灵触发？**
<details>
<summary>点击查看答案</summary>

*   **提示**：VAD 听到电视声音；视觉看到照片。
*   **答案**：
    *   **幽灵触发**：指非目标事件（如噪音、背景人声、误触）导致触发器错误激活。
    *   **多模态融合解法**：使用 `AND` 逻辑（在 FRP 中通常是 `combineLatest` + `map`）。
    *   **逻辑**：`Trigger = VAD_Active AND Face_Detected`。
    *   只有当麦克风有声音，**且**摄像头里检测到有人脸（甚至口型在动）时，才触发 Agent。这能有效过滤掉电视背景声或只有画面没有声音的场景。
</details>

### 挑战题（深入思考与架构设计）

**Q4: “尴尬的沉默”触发器设计。**
**需求：当 LLM 回复结束后，如果 8 秒内既没有用户语音，也没有用户打字，也没有系统正在运行的工具任务，Agent 应主动发起话题。但在 8 秒倒计时期间，任何用户活动都应立即重置该计时器。**
<details>
<summary>点击查看答案</summary>

*   **提示**：`switchMap` 是处理“取消/重置”的神器。
*   **答案**：
    *   **流定义**：
        1.  `LLM_Done_Stream`: LLM 完成回复的瞬间发射。
        2.  `Activity_Stream`: `Merge(UserVoice, UserTyping, ToolRunning)`。任何活动都发射数据。
    *   **FRP 拓扑**：
        ```text
        SilenceTrigger = LLM_Done_Stream
            .switchMap(() => {
                // LLM 完成后，开启一个 8秒 的计时器
                return Timer(8000).takeUntil(Activity_Stream)
            })
            // 如果 Activity_Stream 在 8秒内发射了数据，takeUntil 会终止 Timer，
            // switchMap 会因新的 LLM_Done 而切换（如果有多轮）。
            // 只有当 Timer 完整走完 8秒 没被打断，这里才会发射值。
        ```
    *   **关键点**：`switchMap` 保证了新的回合开始会覆盖旧的计时；`takeUntil` 保证了用户打断会立即杀死计时器。
</details>

**Q5: 设计一个“智能打断（Smart Interrupt）”系统。**
**需求：用户在 Agent 说话时说话（Barge-in）。但是，如果用户只是说 "嗯"、"对"、"啊"（Backchannel/对答词），Agent 不应停止；只有当用户说出实质性内容或语速急促时才打断。**
<details>
<summary>点击查看答案</summary>

*   **提示**：这需要两级 VAD。一级检测声音，二级检测语义。Speculative Execution (投机执行) 也可以用在这里。
*   **答案**：
    *   **架构分层**：
        1.  **Level 1 (Energy VAD)**: 检测到声音，立刻将 Agent 音量 **Duck**（压低音量，而非完全停止），进入“监听态”。
        2.  **Level 2 (ASR Stream)**: 实时转录用户的开头几个字。
        3.  **Level 3 (Semantic Filter)**:
            *   定义“非打断词表”：["嗯", "是的", "继续", "OK"]。
            *   如果是词表内的词 -> 恢复 Agent 音量，忽略输入。
            *   如果是其他词 -> 发送 `Interrupt_Event`，完全停止 Agent，并处理用户新指令。
    *   **FRP 实现**：这是一个 `Conditional Stream`。ASR 结果流经 Filter，只有通过 Filter 的事件才会触发 Cancellation Token。
</details>

**Q6: 调试与回放。你构建的触发器系统在上线后，用户反馈“经常莫名其妙地自动说话”。你无法复现。请设计一个基于 Log 的“时间旅行调试器”所需的数据结构。**
<details>
<summary>点击查看答案</summary>

*   **提示**：仅仅记录“触发了”是不够的，需要知道“为什么触发”以及“当时所有传感器的状态”。
*   **答案**：
    *   **核心思想**：记录所有 **Input Streams** 的原始事件（带时间戳），而不是记录中间状态。
    *   **Schema 设计**：
        ```json
        {
          "session_id": "123",
          "timeline": [
            {"t": 1000, "stream": "MicVolume", "value": 0.1},
            {"t": 1050, "stream": "MicVolume", "value": 0.8}, // 疑似噪音
            {"t": 1100, "stream": "FaceDetect", "value": false},
            {"t": 1200, "stream": "TriggerFired", "id": "VAD", "decision_snapshot": {"vol": 0.8, "thresh": 0.6}}
          ]
        }
        ```
    *   **调试方法**：编写一个 Test Runner，加载这段 JSON，使用虚拟时钟（Virtual Clock）按时间戳将事件注入到你的 FRP 逻辑中。你就可以在本地 100% 复现当时的逻辑判断过程，观察是哪个阈值设置得不对。
</details>

**Q7: (新增) 资源预算与频率限制。如果你的 Trigger 是基于“股票价格变化”触发 RAG 搜索，当市场剧烈波动时，每秒触发 10 次会导致 Token 耗尽和 API 封禁。请设计一个流控策略。**
<details>
<summary>点击查看答案</summary>

*   **提示**：结合 `Throttle` 和 `Window`，以及“变化幅度”检查。
*   **答案**：
    *   **策略 1: 变化幅度门控 (Significant Change Gate)**
        *   `filter(current => abs(current - last_triggered_price) > 2%)`
        *   只有价格变化超过 2% 才允许通过。
    *   **策略 2: 自适应节流 (Adaptive Throttle)**
        *   使用 `throttle(time)`。如果 budget 充足，time = 1min；如果 budget 紧张，time = 10min。
    *   **策略 3: 批量聚合 (Batching)**
        *   使用 `bufferTime(1 minute)`。
        *   将 1 分钟内的所有变动打包成一条消息：“过去 1 分钟，AAPL 变动了 5 次，当前价格 X，最低 Y，最高 Z”。
        *   这既节省了 Token，又给了 LLM 更宏观的市场趋势息。
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 1. 自身回环 (Self-Excitation / Echo Loop)
*   **现象**：Agent 说话的声音被自己的麦克风收录，触发 VAD，Agent 以为用户打断，于是停下来听，结果听到的是自己的回声/尾音。Agent 再次试图解释，进入死循环。
*   **Gotcha**：不要指望软件 AEC (回声消除) 完美。
*   **Fix**：
    *   **逻辑互锁 (Logical Interlock)**：在 FRP 中，`AgentSpeaking` 信号必须作为 VAD 触发器的**硬抑制源**（Suppressor）。
    *   `VAD_Stream = Raw_Audio_Activity.filter(() => !Agent.isSpeaking)`

### 2. 状态不同步导致的竞态 (State Desync Race Condition)
*   **现象**：触发逻辑依赖 `AgentState` 变量。当事件 A 发生时，代码去读取 `AgentState`，但此时状态更新的事件还在队列里排队，导致读取了旧状态。
*   **Gotcha**：在反应式系统中混用“推（Push）”和“拉（Pull）”模式。
*   **Fix**：**永远不要读变量，只通过 `withLatestFrom` 算子组合流**。这能保证在该时刻，你拿到的一定是逻辑时钟对齐的最新状态。

### 3. 上下文窗口污染 (Context Pollution)
*   **现象**：设置了过于灵敏的触发器（如“每分钟报时”或“每一笔小额交易通知”）。几小时后，Prompt 上下文被数千条无用的 `[System Event]` 填满，挤掉了真正重要的对话记忆，且导致推理成本爆炸。
*   **Fix**：
    *   **短期记忆 vs 长期记忆**：高频触发器只能进入“短期流动槽（Scratchpad）”，只有关键事件才写入对话历史。
    *   **自动摘要**：FRP 管道中增加一个 `Summarize` 节点，每累积 10 条低级事件，合成一条高级摘要。

### 4. 惊群效应 (Thundering Herd)
*   **现象**：一个系统事件（如“网络恢复”）同时满足了 10 个 Agent/任务的触发条件。导致瞬间并发 10 个 LLM 请求，打爆限流。
*   **Fix**：在触发器的输出端增加 **Jitter (随机抖动)**。
*   `TriggerStream.delay(() => Math.random() * 2000)`。让大家错峰触发。
