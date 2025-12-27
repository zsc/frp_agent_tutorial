# Chapter 16｜评测与持续迭代：从“能用”到“好用”

## 1. 开篇
在构建实时 Agent 的过程中，开发者往往容易陷入“Demo 陷阱”：在开发环境里，对着几个精心挑选的 Prompt 和测试用例，系统表现完美。然而上线后，面对真实用户的网络抖动、模糊指令、以及长尾的竞态条件，系统体验可能迅速崩塌。

传统的 LLM 评测（如 MMLU 分数）只关注“文本生成的质量”，而忽略了实时 Agent 的核心挑战**时序正确性、交互流畅度与系统稳定性**。

本章的目标是建立一套**全维度的质量保障体系**。我们将学习如何利用 FRP 的“时间旅行”能力（Replay）来构建确定性的测试集，如何量化“好用”（不仅仅是“正确”），以及如何通过闭环数据驱动系统的持续迭代。

---

## 2. 核心论述

### 2.1 评测的四个维度
对于一个实时 Agent，评测不能只看“回答对了没有”。我们需要关注以下四个维度：

1.  **正确性 (Correctness)**：
    *   *语义正确*：RAG 检索是否准确？幻觉程度如何？
    *   *工具使用正确*：参数是否符合 Schema？调用逻辑是否闭环？
    *   *状态一致性*：Prompt 中的状态（如时间、位置）是否与真实环境同步？

2.  **时延与流畅度 (Latency & Fluidity)**：
    *   *TTFT (Time To First Token)*：用户说完到第一个字蹦出来的耗时。
    *   *E2E Latency*：完成整个任务（如查询数据库并回答）的总耗时。
    *   *Interruption Handling*：打断后，旧的输出多久能停止？新的输出多久能跟上？
    *   *Jitter (抖动)*：Token 输出是否匀速？UI 是否频繁闪烁？

3.  **稳定性 (Stability)**：
    *   *Crash Rate*：系统崩溃或死锁的频率。
    *   *Fallback Rate*：工具失效导致降级服务的比例。
    *   *Race Condition Defects*：因并发导致的状态错乱（如回答了上一轮的问题）。

4.  **成本 (Cost Efficiency)**：
    *   *Token Usage*：完成任务消耗的 Token 总量。
    *   *Waste Rate*：被丢弃的计算（如 Speculative Execution 失败、打断后废弃的生成）占比。

### 2.2 利用 FRP 事件流进行“确定性回放评测”

在传统架构中，复现一个“偶发 Bug”极其困难（比如：用户在工具返回前 100ms 说话，导致 UI 卡死）。但在 FRP 架构中，一切皆是**事件流**。

如果我们记录了 Session 的所有输入事件（Input Event Stream）和时间戳，理论上我们可以将这些事件**注入**到一个纯净的 Runtime 实例中，从而**100% 复现**当时的状态演变。

```ascii
[真实用户会话]
User Input:  A---B-------C--> (A: "Hello", B: "Check weather", C: "Stop")
Network:     ----|delay|----> (模拟真实网络延迟)
                 V
[FRP Recorder] -> 保存为 .jsonl (Event Log)

---------------------------------------------------

[自动化评测管线 (CI/CD)]
1. 加载 Event Log
2. 启动 "Virtual Clock Scheduler" (虚拟时钟调度器)
3. 注入事件 A (t=0), B (t=2s), C (t=5s)
4. 捕获 Agent Output Stream
      |
      V
[Assertion / Comparison]
- 检查 t=5.1s 时是否立即触发了 Cancellation
- 检查最终输出是否匹配 "Golden Record"
```

**Rule of Thumb**：如果你的 Agent 逻辑是纯函数式的（依赖 State 和 Event），那么“录制 + 回放”就是最高效的回归测试手段。

### 2.3 任务基准与“混沌注入”

除了回放真实数据，我们还需要构造合成数据来测试边界情况，这被称为“混沌注入（Chaos Injection）”。

*   **延迟注入**：强制 RAG 检索耗时增加 5s，测试 Agent 是否会发出“正在查询中...”的安抚语（Filler words）。
*   **噪声注入**：在用户语音流中插入无意义的噪音事件，测试 VAD 和 Intent Filter 的鲁棒性。
*   **乱序注入**：强制让 Tool B 的结果比 Tool A 先返回，测试 `merge` 或 `switchLatest` 逻辑是否正确处理了顺序。

### 2.4 A/B 测试与灰度发布

实时 Agent 的参数调整（如 Speculative Decoding 的阈值、Batching 的窗口大小）往往是权衡（Trade-off）。

*   **配置即代码**：将所有阈值参数化（Config Object）。
*   **路由策略**：
    *   *User ID Hash*：50% 用户走 V1 策略，50% 走 V2。
    *   *Session Type*：复杂任务走高配模型，闲聊走低配模型。
*   **观察指标**：在 A/B 测试中，重点关注“负向指标”（如用户打断率、重试率）。如果 V2 版本虽然响应快，但用户频繁打断修正，说明质量下降了。

### 2.5 质量闭环：从反馈到迭代

建立一个自动化的飞轮：

1.  **收集 (Collect)**：用户在 UI 上的点赞/点踩，或者隐式反馈（如用户说“不对，我是说...”）。
2.  **归因 (Attribute)**：利用 Trace ID 定位到是哪个环节出了问题（是听错了？检索错了？还是模型推理错了？）。
3.  **修复 (Fix)**：
    *   如果是 Prompt 问题 -> 修改 Prompt 模板 -> 跑回归测试。
    *   如果是知识缺失 -> 更新 RAG 数据库。
    *   如果是逻辑 Bug -> 修改 FRP 算子组合。
4.  **验证 (Verify)**：在 Golden Dataset 上运行，确保没有 Regression。

---

## 3. 本章小结

*   **评测不只是打分**：实时 Agent 的评测必须包含时间维度（时延、抖动、竞态）。
*   **FRP 是测试神器**：利用事件流的不可变性和确定性，可以实现“像素级”的场景回放和调试。
*   **混沌工程**：主动注入网络延迟和错误，验证系统的弹性（Resilience）。
*   **成本意识**：高性能往往伴随着高 Token 消耗（如投机执行），需要在评测中监控 ROI（投入产出比）。
*   **数据闭环**：没有反馈机制的 Agent 是无法进化的，必须打通“用户行为 -> 调试日志 -> 代码/Prompt 修正”的链路。

---

## 4. 练习题

### 基础题

**1. 指标定义辨析**
在实时语音对话 Agent 中，区分以下指标，并说明哪个对用户体验影响最大：
A. LLM 生成完整回答的总耗时 (Total Generation Time)
B. 用户停止说话到听到第一个声音的耗时 (Voice-to-Audio Latency)
C. 语音转文字的耗时 (ASR Latency)

> **Hint**：考虑 conversational turn-taking（轮替）的自然度。

<details>
<summary>参考答案</summary>

**B. Voice-to-Audio Latency** 对用户体验影响最大。
*   **解析**：这是用户感知到的直接延迟。如果这个时间过长（如超过 500ms），用户会感到对话不自然，甚至以为设备没听见而开始重复说话，导致打断逻辑混乱。
*   A 影响的是长内容的等待时间，但在流式播放下，只要开头快，总时长用户容忍度较高。
*   C 只是 B 的一部分，是技术指标而非端到端体验指标。
</details>

**2. 废弃率计算**
系统开启了 Speculative Execution（投机执行），每当用户输入停顿超过 200ms 就触发一次轻量级意图识别。
假设：
*   用户平均每句话停顿 3 次。
*   最终只有最后一次输入是完整的意图。
*   每次投机消耗 50 tokens。
*   最终完整执行消耗 500 tokens。
求：该策略下的 Token 浪费率（Waste Rate = 浪费的 Token / 总消耗 Token）。

> **Hint**：计算浪费的次数和总消耗。

<details>
<summary>参考答案</summary>

*   用户每句话触发投机次数：3 次。
*   有效执行：1 次（最后一次）。
*   无效投机（浪费）：2 次。
*   浪费 Token：2 * 50 = 100 tokens。
*   有效 Token：1 * 50（最后一次投机） + 500（完整执行）= 550 tokens。（注：如果最后一次投机能复用则不算浪费，这里假设投机是为了预热或预取，若未被利用则视为浪费，题目简化处理）。
*   若假设前两次投机完全丢弃，最后一次投机是正确路径的一部分：
    *   总消耗 = 3 * 50 + 500 = 650 tokens。
    *   浪费 = 2 * 50 = 100 tokens。
    *   Waste Rate = 100 / 650 ≈ 15.4%。
</details>

**3. 回放测试的设计**
你记录了一段日志：`[T=0s, User: "Open file"], [T=1s, User: "Cancel"], [T=1.5s, Tool: "File Opened"]`。
在 FRP 回放测试中，你应该断言（Assert）系统在 `T=1.6s` 时的状态是什么？

> **Hint**：考虑 FRP 中的 `takeUntil` 或 `switchLatest` 逻辑以及副作用的处理。

<details>
<summary>参考答案</summary>

*   **断言**：Agent 应该处于“空闲”或“已取消”状态，且 UI **不应该**显示“File Opened”的结果（或者显示后立即撤回/灰）。
*   **原因**：用户在 T=1s 发出了 Cancel 指令。FRP 的流处理应该在 T=1s 时切断了 Tool 的下游监听（Subscriber）。虽然 T=1.5s 工具返回了结果（因为网络请求已发出），但在 Runtime 层面，该结果应被丢弃或忽略，不应触发后续的业务逻辑或 UI 更新。
</details>

---

### 挑战题

**4. 评测“打断”的平滑度**
设计一个自动化测试方案，用于量化评测 Agent 处理“打断（Interruption）”的性能。你需要测量哪个具体的时差？如何用脚本模拟？

> **Hint**：涉及三个时间点：打断发生、音频停止、新音频开始。

<details>
<summary>参考答案</summary>

**方案设计**：
1.  **模拟场景**：Agent 正在播放一段长语音（比如天气预报）。
2.  **触发动作**：在播放到第 2 秒时，脚本注入一个新的 User Audio Event（模拟用户插话）。
3.  **测量指标**：
    *   **T_stop**：从注入打断事件 到 Agent 输出 `AudioStop` 指令（停止播放）的时间差。越短越好。
    *   **T_response**：从注入打断事件 到 Agent 输出针对新话题的第一个 Token 的时间差。
    *   **Overlap Duration**：旧音频和用户新语音重叠的时长。
4.  **自动化实现**：使用虚拟时钟。Agent 认为自己在播放音频，收到打断信号后，检查 Event Log 中 `StopSignal` 的时间戳。如果 `T_stop > 200ms`，则测试不通过（感觉迟钝）。

</details>

**5. 幻觉与引用一致性检测**
Agent 使用了 RAG 技术。如何编写一个自动化的 "Model-based Evaluator" 来检测回答中的幻觉？

> **Hint**：输入是（Query, Retrieved Docs, Agent Answer）。利用另一个强大的 LLM（Judge Model）。

<details>
<summary>参考答案</summary>

**流程**：
1.  **准备数据**：从日志中提取三元组 `{Query, Retrieved_Chunks[], Agent_Response}`。
2.  **构建 Judge Prompt**：
    "你是一个公正的法官。请检查 Agent_Response 中的每一句话。
    如果这句话包含事实性陈述，请验证该陈述是否被 Retrieved_Chunks 中的内容所支持。
    输出格式 JSON：{ 'sentence': '...', 'supported': true/false, 'citation_id': '...' }"
3.  **计算指标**：
    *   **Faithfulness Score**：支持的句子数 / 总事实句数。
    *   **Citation Accuracy**：标注的引用 ID 是否真正包含了对应信息。
4.  **报警**：如果 Faithfulness Score < 95%，则标记该会话为“潜在幻觉”，人工复核。
</details>

**6. 动态 Batching 的 ROI 分析**
你为了提高吞吐量，实现了一个 `debounce(50ms)` 的动态 Batching 策略来合并 Embedding 请求。
这虽然节省了 API 调用次数，但也增加了 50ms 的延迟。
请设计一个公式来决定是否应该开启这个策略。

> **Hint**：不仅看钱，还要看延迟对留存率的影响（假设已知）。

<details>
<summary>参考答案</summary>

**决策公式**：
Score = (Cost_Savings * Weight_Cost) - (Latency_Increase * Weight_User_Churn)

其中：
*   **Cost_Savings**：由于 Batching 减少的请求次数带来的 API 费用节省（或 GPU 推理效率提升折算的金额）。
*   **Latency_Increase**：平均增加的延迟（这里是 0~50ms 的期望值，约 25ms）。
*   **Weight_User_Churn**：每增加 1ms 延迟导致的用户流失带来的预期损失（LTV 损失）。

**Rule of Thumb**：
*   如果是**面向用户的高频交互**（如实时语音），50ms 延迟感知很强，通常**不开启** Batching 或使用极小的窗口（5-10ms）。
*   如果是**后台异步任务**（如生成摘要、存入记忆），延迟不敏感，**开启** Batching 并设置更大的窗口（200ms+）以最大化吞吐。
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 1. "Heisenberg" 观测效应
*   **误区**：为了评测性能，在每个 FRP 算子（Operator）中加了详细的 `console.log` 或同步的磁盘写入。
*   **后果**：日志记录本身阻塞了事件流，导致测出来的延迟比实际高，甚至改变了竞争条件（Race Condition）的结果。
*   **对策**：使用异步的、采样率可控的 Trace 系统；或者在生产环境仅记录二进制的 Event Stream，离线再解析。

### 2. 过度拟合 "Golden Set"
*   **误区**：一旦测试通过，就认为万事大吉。但 RAG 的数据库每天都在更新，LLM 的版本也在微调。
*   **后果**：三个月前的“标准答案”现在可能是错的（例如：你可以问“现在的美国总统是谁”）。
*   **对策**：Golden Set 必须有**生命周期管理**。对于时效性问题，测试用例应当是动态的（Dynamic Assertions），或者定期废弃旧用例。

### 3. 忽略了 "长尾" 状态
*   **误区**：评测时总是从“全新会话”开始测试。
*   **后果**：忽略了长时间运行后 Context Window 爆满、内存泄漏、或者状态机处于怪异中间态（比如工具调用了一半挂起）的情况。
*   **对策**：进行 Soak Testing（浸泡测试），即在回放测试中，连续拼接 100 个会话的历史，测试 Agent 在第 101 个会话时的表现（是否还能记住？是否变慢？）。

### 4. 盲目追求低延迟
*   **误区**：为了极致的 TTFT，不仅使用了流式输出，还使用了非常激进的 Speculative Decoding。
*   **后果**：用户经常看到 Agent 输出了半句话，然后突然撤回重写（因为投机失败）。这种“闪烁”的体验比稍微慢一点但稳定的体验更糟糕。
*   **对策**：引入 "Stability" 指标。在 UI 层设置一个极短的 Buffer（如 20ms），或者惩罚撤回操作的评测分数。
