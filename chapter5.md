# Chapter 5｜用户异步动作：打断、撤回、改口、多模态输入

## 1. 开篇段落

在构建实时 LLM Agent 时，最大的挑战往往不是模型本身，而是**不可预测的人类行为**。
在传统的 Request-Response（回合制）系统中，用户提交输入后，界面通常会被锁定或进入加载状态，直到服务器响应。这是一种“伪实时”。

但在真正的实时系统（Real-time Agent）中，输入通道是**永远开放**的。用户是一个混乱的、高并发的事件源：
*   他们会在 Agent 说话说到一半时插嘴（**打断**）。
*   他们会盯着屏幕看，一边用鼠标选中文本，一边说“把这个翻译了”（**多模态并发**）。
*   他们会说“帮我定一张去北京……啊不对，去上海的票”（**流式修正**）。
*   他们甚至可能在 Agent 执行耗时任务时，点击“取消”按钮，然后立即发起新任务。

如果试图用传统的 `if-else` 或状态机回调来处理这些情况，代码很快就会变成无法维护的“意大利面条”。本章将展示如何利用 FRP 的**时间流（Time Stream）**语义，将这些混乱的用户行为抽象为有序的事件序列，实现优雅的并发控制、自动撤销和上下文对齐。

**学习目标**：
1.  **打断与取消**：掌握 `switchLatest` 范式，实现“后来者居上”的自动任务切换。
2.  **流式修正**：如何处理 ASR（语音识别）的中间态与最终态，以及用户的撤回/编辑操作。
3.  **多模态时序对齐**：解决视觉事件（手势/点击）与听觉事件（语音）在物理时间上的不同步问题。
4.  **反应式 UX**：利用输入流的元数据（Hover, Focus）驱动预测性执行（Speculative Execution）。

---

## 2. 文字论述

### 2.1 用户输入流建模：从 String 到 Event

首先，我们要摒弃“用户输入 = 字符串”的观念。在 FRP 架构中，用户输入是一条永不停歇的**事件流（Event Stream）**。我们需要构建一个高内聚的 `UserAction` 类型系统。

#### 输入事件分类
我们在系统中定义的输入事件不仅仅包含内容，还包含**意图状态**：

1.  **Ephemeral (瞬态事件)**：
    *   `Typing(char)`: 用户正在敲击键盘。
    *   `AudioFragment(blob)`: 麦克风采集到的音频切片。
    *   `Gaze/Hover(target)`: 用户的注意力焦点变化（未确认意图）。
    *   *处理原则*：这类事件通常用于更新 UI 状态（如波形图）、触发预加载或作为上下文缓存，**不直接触发 LLM 推理**。

2.  **Interim (中间态事件)**：
    *   `ASRPartial(text)`: 语音识别的实时转录（文字还在变动）。
    *   *处理原则*：用于 UI 展示（灰色文字）和推测性解码（Speculative Decoding），但需警惕“幻觉触发”。

3.  **Commit (提交态事件)**：
    *   `Message(text)`: 用户按下回车。
    *   `ASRFinal(text)`: VAD（语音活动检测）判定说话结束，且 ASR 锁定结果。
    *   `IntentAction(id)`: 点击了“生成日报”按钮。
    *   *处理原则*：**这是触发 Agent 思考与执行的唯一信号。**

4.  **Meta (元控制事件)**：
    *   `Cancel/Stop`: 用户点击停止生成。
    *   `Edit(originalId, newContent)`: 修改历史指令。
    *   `Undo`: 撤销上一步操作。

**Rule of Thumb**：**输入即信号（Signal），提交即采样（Sample）。** 系统的 Prompt 上下文应该由各种瞬态信号（位置、选区、时间）不断累积而成，而提交事件则是对这些信号的一次“快照”。

### 2.2 “打断”语义：FRP 的核心超能力

在传统开发中，“打断”意味着你必须手动检查每一个异步操作的句柄，调用 `.abort()`，清理回调重置 UI。在 FRP 中，这通过算子组合天然实现。

#### `switchLatest`：只关心当下
最核心的算子是 `switchLatest`（或 `switchMap`）。它的语义非常符合人类直觉：**“一旦我有新想法，旧想法就不重要了。”**

想象两个流：
*   **Outer Stream**: 用户意图流（User Intents）。
*   **Inner Stream**: Agent 执行流（Agent Execution，包含思考、工具调用、回复生成）。

当用户发出新意图 "B" 时，`switchLatest` 会自动：
1.  **退订（Unsubscribe）** 旧意图 "A" 对应的 Inner Stream。
2.  **触发清理逻辑**（FRP 库会自动调用流的 dispose/teardown 函数，例如 abort HTTP 请求）。
3.  **订阅（Subscribe）** 新意图 "B" 的 Inner Stream。

**ASCII 示意图：打断与资源释放**

```text
User Intent Stream:
-----(ID:1 "查天气")--------------------(ID:2 "不对，查股价")----->

Derived Execution Stream (switchMap):
     |                                   |
     +--> [ Network Request (Weather) ]  |
          [ Processing...             ]  |
          [ Output: "Beijing is..."   ]  | <--- 这里的后续被切断
               X (Cancelled/Aborted)     +--> [ Network Request (Stock) ]
                                              [ Processing...           ]
                                              [ Output: "AAPL is..."    ]
```

#### 传播取消（Cancellation Propagation）
打断不仅仅是停止 LLM 生成 token。FRP 的链式结构能确保取消信号一直传播到底层：
*   **UI 层**：停止打字机效果，移除“思考中”动画。
*   **网络层**：`AbortController` 中断 TCP 连接。
*   **工具层**：如果工具支持（如数据库查询），发送 `KILL query`；如果工具不支持（如发邮件），则需要在“副作用执行前”再次检查活跃状态（FRP 的 `takeUntil` 模式）。

### 2.3 语音流处理：Partial, Final 与 VAD

语音交互是实时 Agent 的高频场景。难点在于**不稳定性**。

#### ASR 的抖问题
流式 ASR 输出是修正性的。
`t=1`: "我想"
`t=2`: "我想去"
`t=3`: "我想去吃"
`t=4`: "我想去池塘" (ASR 错误猜测)
`t=5`: "我想去吃饭" (ASR 修正) -> **Final**

如果我们在 t=4 时就触发了 Agent（基于激进的推测），用户可能会看到 Agent 正在搜索“池塘”。几毫秒后 ASR 修正为“吃饭”，此时必须**撤回**刚才的搜索。

**设计模式：稳定性防抖（Stability Debounce）**
不建议直接使用简单的 `debounceTime`（会增加延迟）。建议结合**文本相似度**或**状态标记**：
1.  维护一个 `LastProcessedText`。
2.  当新 `Partial` 到来时，如果与 `LastProcessedText` 差异过大（语义剧变），且 VAD 显示用户有短暂亦或是，则触发一次轻量级的“意图预判”。
3.  只有收到 `Final` 事件，才触发完整的 Agent 流程。

#### 抢话（Barge-in）
当 Agent 正在 TTS 播报回复时，VAD 检测到用户说话（User Voice Start）。
此时必须立即触 `AudioInterrupt` 事件：
1.  **静音 TTS**：Agent 立即闭嘴。
2.  **清空音频缓冲**：防止播放残留内容。
3.  **监听新指令**：进入“倾听”模式。

**注意**：要过滤回声（Echo Cancellation）。如果你没有硬件回声消除，Agent 自己的声音会被麦克风录入，导致 Agent 自己打断自己。这通常需要在音频输入流上做一个 Filter：`inputStream.filter(() => !agentIsSpeaking)`。

### 2.4 多模态融合：时间窗对齐（Temporal Alignment）

这是最容易产生 Race Condition 的地方。
场景：用户指着屏幕上的图表（Visual Event），问道：“这个数据为什么异常？”（Audio Event）。

问题在于：
1.  **先指后说**：手指悬停在图表上，0.5秒后开始说话。
2.  **先说后指**：嘴里说着“看这里”，0.5秒后手指才移上去。
3.  **说完松手**：说完“这个”之后，手马上移开了。

如果简单地使用 `withLatestFrom`，当 Audio Event 触发时，Visual Event 可能已经变为 `null`（鼠标移开了），或者还没更新。

**解决方案：带记忆的上下文流**
我们需要构建一个**“短期记忆窗口”（Sliding Window Context）**。

```text
Visual Stream ($hover):
...[Chart A]...[Chart A]...[null]........[Chart B]......>

Voice Stream ($voice):
.............................[Start].."解释这个"..[End]..>

Aligned Context Strategy:
当 Voice Event 发生时，不要只看 Current Value。
要看 [Voice.Start - 1.0s, Voice.End] 这个时间段内：
1. 出现频率最高的元素？
2. 最后被激活的元素？
3. 或者是通过 LLM 模糊匹配（"这个"指代的是视觉历史中的哪一个）？
```

**FRP 实现思路**：
使用 `buffer` 或 `scan` 算子维护一个最近 2 秒的 `List<UIEvent>`。当语音意图触发时，将这段历史与语音一起打包发给 Agent。

### 2.5 撤回与修改：版本控制与因果一致性

用户说：“发邮件给张三。” -> Agent 正在写草稿。
用户紧接着：“哎不对，是李四。”

这不仅仅是“打断”，这是**修正（Correction）**。
我们需要引入类似数据库的事务 ID 或 Git 的版本概念：

*   **InputRevision**：每个输入不仅仅有 ID，还有 `ParentID`。
    *   Event 1: { id: 101, text: "发给张三" }
    *   Event 2: { id: 102, text: "是李四", ref_id: 101, type: "correction" }
*   **Agent 的处理**：
    *   Agent 收到 Event 2 时，发现它是对 101 的修正。
    *   系统查找 101 产生的副作用（ToolCall: DraftEmail）。
    *   如果副作用未提交，直接丢弃，用新参数重新生成。
    *   如果副作用已提交（邮件已发），Agent 需要自动生成一个**补偿动作（Compensating Action）**，例如“撤回邮件”或“发送更正邮件”。

---

## 3. 本章小结

*   **事件驱动**：用户输入不是字符串，是包含生命周期（Ephemeral -> Partial -> Final -> Correction）的复杂事件对象。
*   **打断即切换**：利用 FRP 的 `switchLatest` 算子，可以将复杂的打断逻辑简化为声明式的流切换，自动处理资源清理和取消传播。
*   **多模态时序**：物理时间的一致性是不可靠的。必须通过“时间窗口”和“缓冲历史”来对齐视觉（手势）与听觉（语音）的意图。
*   **防抖与纠错**：在实时流中，不仅要防抖（Debounce）以减少系统震荡，还要处理修正（Correction），维护操作的因果一致性。

---

## 4. 练习题

### 基础题

**Q1. 基础打断实现**
构造一个流处理管道：
1. 监听用户文本输入 `input$`。
2. 只有当用户停止输入超过 500ms 后才触发请求（防抖）。
3. 如果在请求处理期间用户又输入了新字符，立即取消旧请求并重新计时。
请用伪代码描述算子链。

<details>
<summary>点击展开参考答案</summary>

**提示**：结合 `debounceTime` 和 `switchMap`。

**答案**：
```typescript
input$
  .debounceTime(500) // 1. 等待静默
  .switchMap(text => { // 3. 自动切换与取消
      return requestLLM(text) // 2. 发起请求
          .takeUntil(cancelSignal$) // 可选：显式取消信号
  })
  .subscribe(result => updateUI(result));
```
`switchMap` (即 `map` + `switchLatest`) 是关键，它保证了任何时刻只有一个活跃的 Inner Observable (请求)。

</details>

**Q2. 语音状态机**
设计一个流，将原始的 VAD 信号（True/False）转化为三种事件：`SpeechStart`, `SpeechEnd`, `SpeechInterrupt`。其中 `SpeechInterrupt` 仅在 Agent 正在说话（`agentSpeaking$ == true`）且检测到 VAD 为 True 时触发。

<details>
<summary>点击展开参考答案</summary>

**提示**：需要组合 `vad$` 和 `agentSpeaking$` 两个流。

**答案**：
```typescript
vad$
  .distinctUntilChanged() // 只关注状态跳变
  .withLatestFrom(agentSpeaking$) // 获取当前 Agent 状态
  .flatMap(([isUserSpeaking, isAgentSpeaking]) => {
      if (isUserSpeaking) {
          if (isAgentSpeaking) return Stream.of("SpeechInterrupt", "SpeechStart");
          return Stream.of("SpeechStart");
      } else {
          return Stream.of("SpeechEnd");
      }
  })
```
注意：`SpeechInterrupt` 通常需要优先处理，直接路由到音频播放器的 `stop()` 方法。

</details>

**Q3. 简单的撤回逻辑**
用户发送消息后 5 秒内点击“撤回”。如何设计流来处理这个“撤回窗口”？

<details>
<summary>点击展开参考答案</summary>

**提示**：发送消息产生的不是由结果，而是一个带有“可撤销句柄”的对象。

**答案**：
消息发送流产生一个 `PendingAction` 对象，放入一个缓冲池。
UI 上的“撤回”按钮触发 `undo$` 流。
`undo$` 流携带 `messageId`，在缓冲池中查找对应的 Action。
如果找到且未超时（<5s）：
1. 触发 Action 的 `cancel()` 方法（如果是 API 请求）。
2. 发出 `RemoveMessage` 事件更新 UI。
FRP 思路：`send$.delay(5000)` 作为“不可撤销”的提交点，在 5秒内 `undo$` 可以阻断该流。

</details>

**Q4. 输入流合并**
你需要同时监听“语音识别流”和“键盘输入流”。如果用户正在打字，语音识别的结果（可能是背景人声）应该被忽略。如何实现？

<details>
<summary>点击展开参考答案</summary>

**提示**：Gate / Filter 模式，利用 `combineLatest` 或 `merge` 加状态位。

**答案**：
1. 创建 `isTyping$` 信号：`keyboard$.map(() => true).merge(keyboard$.debounce(2000).map(() => false))`。
2. 过滤语音流：
   ```typescript
   validVoice$ = rawVoice$.withLatestFrom(isTyping$)
     .filter(([text, typing]) => !typing) // 如果正在打字，丢弃语音
     .map(([text, _]) => text);
   ```
3. 最后合并：`finalInput$ = validVoice$.merge(keyboard$)`。

</details>

### 挑战题

**Q5. 多模态“时光倒流”**
用户先指了一个图表（Visual Event），然后手移开了，2秒后说“刚才那个图表...”。此时 `withLatestFrom` 取到的是空。
请设计一个 Operator 或逻辑，能够根据语音内容中的关键词（“刚才那个”），回溯过去 5 秒内的 Visual Stream 历史，找到最可能的对象。

<details>
<summary>点击展开参考答案</summary>

**提示**：`ReplaySubject` 结合 `Scan` 维护历史队列。

**答案**：
1. 维护一个 `visualHistory$`，使用 `scan` 保存过去 5 秒的所有 visual events（带时间戳）。
2. 当收到语音包含“刚才”、“上一个”等指代词时：
   取出 `visualHistory$` 的当前值（一个列表）。
   按时间倒序遍历，或者让 LLM 作为一个纯函数 `f(history_list, user_query)` 来选择最匹配的元素。
3. 这种“时序模糊匹配”是多模态 Agent 的核心逻辑之一。

</details>

**Q6. 并发编辑冲突解决 (OT 思想)**
两个用户（User A, User B）同时操作同一个 Agent 上下文。A 修改了 Prompt，B 正在根据旧 Prompt 等待回复。
如何利用 FRP 流检测这种冲突并向 B 发出警告？

<details>
<summary>点击展开参考答案</summary>

**提示**：利用 Sequence ID 或 State Hash 对比。

**答案**：
1. 全局状态流 `globalState$` 每次更新都有一个递增 `version`。
2. User B 的请求流发出时，捕获当前的 `requestVersion = 10`。
3. 当 Agent 准备返回结果给 B 时（可能是流式输出），对比 `currentGlobalVersion`。
4. 如果 `currentGlobalVersion > requestVersion`，说明在 B 等待期间状态变了。
5. 在输出流中注入一个 `StateMismatchWarning` 事件，UI 弹窗提示：“上下文已变更，结果可能过时，是否刷新？”

</details>

**Q7. 流式工具调用的“后悔药”**
Agent 决定调用一个耗时工具（如“扫描全盘文件”）。工具开始运行了。用户突然说“算了别找了”。
请设计一个端到端的 FRP 链路，确保用户的语音指令能穿透到 Python 解释器或 Shell 进程中杀死该任务。

<details>
<summary>点击展开参考答案</summary>

**提示**：Cancellation Token 的全链路传。

**答案**：
1. 用户语音 "算了" -> 意图识别 -> `CancelIntent`。
2. 主控制流 `agentLoop$` 收到 `CancelIntent`。
3. `agentLoop$` 是一个 `switchLatest` 结构，新意图会导致旧的 Observable 被 unsubscribe。
4. 工具执行的 Observable 在 `return/dispose` 回调中，必须包含清理逻辑：
   - 如果是本地进程：`subprocess.kill()`
   - 如果是远程 API：发送 HTTP DELETE 或 Cancel 信号。
5. **关键点**：工具封装层必须返回一个带有 dispose 逻辑的 Observable，不能是单纯的 Promise。

</details>

**Q8. 乐观 UI (Optimistic UI) 的回滚流**
为了极致响应，用户发出指令后，UI 立即假设成功并更新（例如把 todo 项划掉）。但 1 秒后 Agent 返回“操作失败”。
如何用 FRP 处理这种“先斩后奏”及其失败回滚？

<details>
<summary>点击展开参考答案</summary>

**提示**：`merge` 两个流：一个是立即的乐观值，一个是延迟的真实值（可能包含 error。

**答案**：
```typescript
action$.flatMap(action => {
    const optimistic = Stream.of(new SuccessEvent(action)); // 立即假装成功
    const realRequest = execute(action)
        .map(result => new ConfirmedEvent(result)) // 真实确认
        .catch(err => Stream.of(new RollbackEvent(action, err))); // 失败回滚

    return Stream.merge(optimistic, realRequest);
})
```
Reducer 接收到 `SuccessEvent` 更新 UI 样式（半透明）；接收到 `ConfirmedEvent` 变为实心；接收到 `RollbackEvent` 则恢复原样并报错。

</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 幽灵状态 (Zombie State)
*   **现象**：用户打断了 Agent，UI 看起来已经停止了，但后台日志显示 Agent 还在疯狂调用工具，甚至几分钟后突然弹出一个无关的报错。
*   **原因**：FRP流虽然被 unsubscribe 了，但流内部的**副作用（Side Effect）**没有实现正确的清理（Teardown）逻辑。例如，启动了一个 `setInterval` 或发出了一个没有 `AbortSignal` 的 `fetch` 请求。
*   **调试**：检查每一个自定义的 Observable 创建函数（`new Observable(...)`），确保其返回的 teardown function 能够真正停止正在进行的物理操作。

### 5.2 乱序到达 (Out-of-Order Arrival)
*   **现象**：用户先发了 Short Request (A)，紧接着发了 Long Request (B)。结果 A 先处理完显示了，B 正在处理，突然 A 的结果又覆盖了 B 的 Loading 状态，或者 B 先返回了，A 后返回覆盖了 B。
*   **原因**：没有使用 `switchLatest` 或 `concatMap`，而是使用了 `mergeMap`（并行处理）。导致结果回来的顺序只取决于网络延迟，而非用户意图顺序。
*   **Fix**：对于必须线性的对话流，严禁使用 `mergeMap`。

### 5.3 VAD 切割过碎
*   **现象**：用户说一句话稍微停顿了一下，Agent 就以为说完了开始回答，导致对话变成：“我想要...”“好的！”“...一个苹果。”“明白了！”。
*   **原因**：VAD 的 `silence_threshold` 时间太短，或者没有配置“语音拼接”策略。
*   **Fix**：
    1. 增大 VAD 阈值（如从 300ms 增至 700ms）。
    2. 引入“语义完整性检测”：在 Final 提交前，先用一个小模型（或正则/N-gram）判断句子是否完整。如果不完整，即使 VAD 触发也不提交，而是等待下一个片段（Wait-for-more）。

### 5.4 内存泄漏
*   **现象**：Agent 运行一小时后越来越卡。
*   **原因**：在高频事件流（如鼠标移动、音频波形）上使用了 `scan` 或 `buffer` 但没有限制大小，导致数组无限膨胀；或者订阅了 EventBus 但在组件销毁时忘记 unsubscribe。
*   **Fix**：
    1. 总是对高频流使用 `takeUntil(destroy$)` 模式。
    2. 给所有 buffering 算子加上最大长度限制（如 `buffer(count=100)`）。

### 5.5 "回声"触发死循环
*   **现象**：Agent 说的话被自己的麦克风听见，被识别为用户输入，Agent 再次回答，限循环。
*   **原因**：缺少 AEC（回声消除）或软件层的“Agent 说话时屏蔽麦克风输入”。
*   **Fix**：在 FRP 链路最前端加一个 Gate：`micStream.filter(() => !speaker.isPlaying)`。虽然这会导致 Agent 说话时无法被打断（半双工），但比死循环要好。完美的方案需要硬件 AEC 或基于原始音频数据的减法处理。
