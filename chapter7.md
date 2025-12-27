# Chapter 7｜RAG / DB / Python toolcall：把知识管道变成事件流

## 1. 开篇段落
在构建复杂的 Agent 时，最昂贵的操作往往不是 LLM 推理本身，而是获取外部信息的 IO 操作：检索向量库（RAG）、查询关系型数据库（DB）或执行代码（Python Sandbox）。在传统的请求-响应（Request-Response）模型中，这些操作是“阻塞”的——Agent 必须暂停思考，直到数据返回。

**FRP（函数式响应编程）改变了这一范式**。我们将外部知识视为一条条**“流动的管道（Streaming Pipelines）”**。
*   **知识即事件**：数据不是静态存在那里的，而是随时间推移“到达”的。
*   **异步非阻塞**：Agent 发出查询意图，立即准备处理其他逻辑（如 UI 交互），数据到达时会自动触发“状态更新（State Update）”。
*   **副作用管理**：将“只读查询”与“写操作”严格区分，通过 Effect 系统隔离风险。

本章将深入探讨如何用 FRP 原语构建一个高并发、低延迟、可观测的知识获取系统。

## 2. 文字论述

### 7.1 RAG 管道拆解：从 RPC 到 Observable DAG
不要将 RAG 简单视为 `f(query) -> documents`。在生产环境中，它是一个有向无环图（DAG），每个节点都是一个可被观测的流。

#### 传统视角 vs. FRP 视角
*   **传统**：串行执行。Rewrite -> Embed -> Vector Search -> Keyword Search -> Merge -> Rerank -> Return。任何一步卡住，全链路等待。
*   **FRP**：数据流网络。各个环节解耦，下游算子订阅上游算子的输出。

#### 架构图示
```text
UserQueryEvent
      │
      ├──> [Query Rewriter] ──> RewrittenQueryStream
      │                                 │
      │                                 ├──(map)──> [Embedding Service] ──> VectorStream
      │                                 │
      │                                 └──(map)──> [Keyword Search] ─────> KeywordStream
      │                                                                         │
      ▼                                                                         │
[Intent Classifier]                                                             │
      │                                                                         ▼
      └──> (filter: is_navigational?) ──> [Knowledge Graph] ──────────> GraphStream
                                                                               │
                                                                               ▼
                                                   [Fusion Operator (combineLatest + buffer)]
                                                                               │
                                                                               ▼
                                                                     [Reranker & Filter]
                                                                               │
                                                                               ▼
                                                                     [ContextPatch Signal]
```

**关键设计模式**：
1.  **并行扇出 (Fan-out)**：使用 `share()` 算子将同一个查询信号分发给多个检索源（Vector, Keyword, KG）。
2.  **弹性聚合 (Fan-in)**：使用 `zip`（等待所有返回）或 `combineLatest`（有更新就触发）。
3.  **部分可用性**：如果在规定时间窗口内 VectorStream 返回了但 KeywordStream 超时，`timeoutWith` 算子可以决定是继续等待还是直接使用部分结果向下游流转。

### 7.2 检索触发器：不仅仅是用户提问
在实 Agent 中，检索不应只由用户的显式提问触发。我们需要定义多种“触发源（Trigger Source）”。

1.  **显式意图流 (Explicit Intent Stream)**
    *   通过意图分类模型，当检测到 `UserAsk(Query)` 事件时触发。这是最基础的。
2.  **推测性触发流 (Speculative Trigger Stream)**
    *   **基于不确定性**：监控 LLM 生成流的 Token Logprobs。如果某个实体名词（Entity）的置信度低于阈值（如 0.4），或者 LLM 输出了特殊的占位符 `<LOOKUP>`，立即发射检索事件。
    *   **预取 (Prefetching)**：用户还在打字（`TypingStream`），基于未完成的输入片段进行模糊检索，提前预热缓存。
3.  **环境上下文流 (Contextual Trigger Stream)**
    *   **时间/位置变化**：当 `SystemClock` 跨越了关键时间点（如早上 8:00），或 `LocationSignal` 发生位移，自动触发一次“上下文刷新检索”，更新 Prompt 中的“周边信息”。

### 7.3 增量检索与流式合成 (Streaming Synthesis)
为了追求极致的**首字延迟（TTFT）**，我们不能等到所有文档都 Rerank 完毕才开始生成。

#### 策略 A：Race (竞速模式)
适用于简单问答。
*   **算子**：`race(SourceA, SourceB, Cache)`。
*   **逻辑**：Cache 往往最快。如果 Cache 命中，直接使用，并取消 SourceA 和 SourceB 的网络请求（利用 FRP 的 `unsubscribe` 机制节省 Token 和带宽）。

#### 策略 B：Progressive Refinement (渐进式优化)
适用于复杂研究任务。
*   **算子**：`scan` (累加器) + `switchMap`。
*   **流程**：
    1.  **T+100ms**: 关键词搜索返回 3 个结果。Prompt 更新，LLM 开始生成“基于目前信息，大概是...”。
    2.  **T+500ms**: 向量搜索返回 10 个更精准结果。Prompt 再次更新。
    3.  **LLM 行为**：此时通过 `switchLatest` 机制，如果前一个回答尚未发给用户，则丢弃；如果已发，则在 UI 上发送一个 `CorrectionEvent`（修正补丁）或者生成一个新的段落补充细节。

### 7.4 DB 查询：Schema 感知与副作用隔离
数据库操作必须严格区分**Query（查）**和**Command（改）**。

#### Schema as Signal (模式即信号)
Prompt 需要知道数据库长什么样。但 Schema 可能会变。
*   建立一个 `DatabaseSchemaSignal`。
*   当 DB 管理员修改表结构时，推送新 Schema。
*   Agent 的 System Prompt 订阅此 Signal。
*   **效果**：Schema 变更后，Agent 的下一次 SQL 生成会自动适配新字段，无需重启服务。

#### 只有 Effect 才能改变世界
*   **SQL 生成**：这是纯函数逻辑（String -> String）。
*   **SQL 执行 (SELECT)**：这是只读副作用，产生 `DBResultStream`。
*   **SQL 执行 (INSERT/UPDATE)**：这是危险副作用。
    *   必须经过 `ApprovalGuard`（审批守卫算子）。
    *   必须实现 `Transactional`（事务性）。如果流在中途被取消（用户点了停止），事务必须回滚。FRP 的 `finalize` 回非常适合处理这种 cleanup 逻辑。

### 7.5 Python 工具：沙箱与流式输出
Python Code Interpreter 是 Agent 最强大的“大脑扩展”，也是最大的风险点。FRP 将 Python 执行视为一个长生命周期的**Observable Session**。

#### 执行生命周期流
```text
[ExecuteRequest] 
   │
   ├──> (Start Sandbox) ──> [ContainerReady]
                                   │
                                   ├──> (stdin input)
                                   │
   ┌───────────────────────────────┼──────────────────────────────┐
   │                               │                              │
   ▼                               ▼                              ▼
(Stdout Stream)             (Stderr Stream)                (Files Stream)
   │                               │                              │
   │ map(RenderLogUI)              │ map(AutoDebug)               │ map(Upload)
   ▼                               ▼                              ▼
[User UI: Console]          [ErrorCorrectionAgent]         [AttachmentUI]
```

#### 关键处理 Rule-of-Thumb
1.  **不要缓冲 (No Buffering)**：Python 的 `stdout` 必须逐行（甚至逐字符）流向 UI。用户看到 print 就像看到 Agent 在思考，这能极大缓解等待焦虑。
2.  **超时即销毁 (Timeout = Kill)**：使用 `takeUntil(StopSignal)` 或 `timeout(30s)`。一旦触发，FRP 框架的 teardown 逻辑必须强制 `kill -9` 容器，防止僵尸进程。
3.  **错误作为值 (Error as Value)**：Python 报错（Exception）不应中断 Agent 的主线程，而应作为 `ComputationFailed` 事件进入流，触发 LLM 的自我修正（Self-Correction）分支。

### 7.6 缓存与一致性：Stale-while-revalidate
外部调用既慢又贵。FRP 天然支持高级缓存模式。

*   **MemoryStream**：所有的 `Effect` 输出都根据 Request Hash 存入内存或 Redis。
*   **Stale-while-revalidate 算子组合**：
    ```text
    inputStream.pipe(
        switchMap(req => 
            merge(
                getFromCache(req),           // 立即发出（如果是旧数据）
                fetchFromNetwork(req)        // 稍后发出（新数据）
            )
        )
    )
    ```
*   **UI 处理**：UI 先收到旧数据展示（带“更新中...”角标），几秒后收到新数据自动刷新。这比单纯的 Loading 转圈体验好得多。

### 7.7 证据引用与可追溯性
为了解决幻觉，知识必须“带票据上车”。

#### 数据结构：Rich Chunk
进入 Prompt 的不仅仅是文本，而是结构化对象：
```json
{
  "chunk_id": "vec_8821_chunk_5",
  "text": "2023年营收增长15%...",
  "meta": { "source": "Q3_Report.pdf", "page": 12 },
  "score": 0.89
}
```

#### 引用流 (Citation Stream)
1.  **Prompt 注入**：将 Chunk 格式化为 `[Ref: vec_8821_chunk_5] 2023年营收...`。
2.  **生成监控**：当 LLM 输出 `[Ref: vec_8821_chunk_5]` 时，前端解析器拦截该 Token。
3.  **反向查找**：前端利用 ID 在本地缓存的 `RetrievalResults` 中找到对应的 Meta，渲染高亮链接。
4.  **一致性检查**：如果 LLM 捏造了一个不存在的 ID，`VerificationStream` 会报警并标记该回答为“不可信”。

### 7.8 多来源融合 (Fusion)
当 SQL 查询结果（精准）与 RAG 结果（模糊）同时到达时，如何合并？

*   **Conflict Resolution Strategy (冲突解决策略)**：
    *   **Tiered Merge (分层合并)**：定义优先级 `DB > API > VectorStore > KeywordSearch`。
    *   **Timestamp Weighing (时间加权)**：如果 RAG 检索出的文档是 3 年前的，而 DB 数据是昨天的，FRP 的 `reducer` 函数应根据 timestamp 剔除旧信息。
*   **Operator**: `zip(dbStream, ragStream).pipe(map(mergeLogic))`。
    *   注意：这里通常不用 `race`，因为我们希望综合两者信息。

---

## 3. 本章小结
*   **管道化思维**：RAG 不再是一个函数调用，而是一个可拆解、可观测、可并行的事件处理 DAG。
*   **时间是第一公民**：利用 FRP 处理异步竞态、超时取消和增量更新，让 Agent 的反应速度接近人类直觉。
*   **Schema 与 State 同步**：数据库结构的变化本身就是一种环境信号，必须实时推送到 Agent 的上下文中。
*   **流式反馈**：无论是 Python 的 print 还是文档检索的中间状态，都应作为事件流实时反馈给用户，提升透明度。

---

## 4. 练习题

### 基础题

**Q1: 基础管道搭建**
请用文字描述（或伪代码）搭建一个包含“缓存层”的检索管道。要求：
1. 收到查询 `Q`。
2. 同时查询内存缓存 `Cache` 和 远程向量库 `Remote`。
3. 如果 `Cache` 命中，立即返回并终止 `Remote` 请求。
4. 如果 `Cache` 未命中，等待 `Remote` 返回并更新 `Cache`。

<details>
<summary><strong>参考答案</strong></summary>

*   **思路**：这是典型的 `race` 变种，或者使用 `concat` 配合条件判断。
*   **FRP 逻辑**：
    1.  定义 `cacheStream = lookupCache(Q)`。
    2.  定义 `remoteStream = fetchRemote(Q).pipe(tap(updateCache))`。
    3.  组合：`concat(cacheStream, remoteStream).pipe(first(result => result != null))`。
    4.  **解释**：`concat` 会先订阅 cacheStream。如果 cache 返回非空结果，`first` 算子满足条件，整个流结束（remoteStream 甚至不会被订阅，节省资源）。如果 cache 为空，`concat` 继续订阅 remoteStream。
</details>

**Q2: 失败重试 (Retry with Backoff)**
RAG 服务经常网络波动。请设计一个流，当检索失败时：
1. 立即重试一次。
2. 如果还失败，等待 500ms 重试。
3. 再失败，等待 1s 重试。
4. 超过 3 次失败，返回一个默认的“搜索服务不可用”事件，而不是抛出异常崩溃。

<details>
<summary><strong>参考答案</strong></summary>

*   **思路**：使用 `retryWhen` 或 `retry` 配合 `timer` 和 `zip`。
*   **FRP 逻辑**：
    ```text
    searchStream.pipe(
        retryWhen(errors => 
            errors.pipe(
                zip(range(1, 3), (err, i) => i), // 限制重试次数
                mergeMap(i => timer(i * 500))    // 指数退避或线性退避
            )
        ),
        catchError(err => of(new ServiceUnavailableEvent())) // 兜底降级
    )
    ```
</details>

**Q3: 节流防抖**
用户在搜索框快速输入 "Python", "Python t", "Python tut", "Python tutorial"。如果每个字符都触发 RAG，系统会崩溃。如何优化？

<details>
<summary><strong>参考答案</strong></summary>

*   **思路**：`debounceTime` 是标准解法。
*   **FRP 逻辑**：
    1.  `TypingStream` 产生按键事件。
    2.  应用 `debounceTime(300)`：只有当用户停止输入超过 300ms 后，才通过最新的值。
    3.  应用 `distinctUntilChanged()`：防止用户输入 "abc" -> 删除 "c" -> 再输入 "c" 导致重复触发（如果此时值变了 "abc" 且没变）。
    4.  应用 `switchMap(query => search(query))`：如果在搜索 "tut" 的过程中用户又输入了 "orial"，取消前一次搜索，只搜最新的。
</details>

### 挑战题

**Q4: 增量合成的“热替换”难题**
场景：用户问“昨天发生了什么”。
1.  **T+0.1s**: 关键词检索返回“无大事发生”。LLM 开始输出：“昨天似乎很平静...”
2.  **T+1.5s**: 深度向量检索返回“发生里氏 7.0 级地震”。
3.  此时 LLM 已经输出了 10 个 Token。
请设计一个机制，既能利用新信息，又不让用户觉得 UI 出了 Bug（文字突然消失或鬼畜）。

<details>
<summary><strong>参考答案</strong></summary>

*   **提示**：这是一个 UX + 技术综合题。FRP 负责提供“打断”能力，UI 负责“平滑”。
*   **思路**：
    1.  **Versioned Stream**: 每个输出片段都带有 `version_id`。
    2.  **Trigger**: 当高置信度的新证据（地震）到达时，触发 `ContextUpdateEvent`。
    3.  **FRP Action**:
        *   立即发送 `InterruptSignal` 停止当前的 LLM 生成流。
        *   发送一个 UI 事件 `StatusEvent("发现新情报，正在修正...")`。
        *   创建一个新的 Prompt（包含地震信息），带上指令“用户之前看到了一半‘昨天很平静’，请自然地转折或纠正”。
    4.  **UI 渲染**:
        *   前端收到打断信号，保留已输出的“昨天似乎很平静...”，将光标变为加载态。
        *   新流到达，LLM 生成可能是：“...哎呀，刚收到最新消息，实际上发生了大地震。”
        *   或者（更激进地）：UI 撤回上一句话（带删除线动画），重新打印新内容。FRP 能够轻松实现撤回，因为所有 Token 都是存储在 `Scan` 状态中的列表。
</details>

**Q5: 跨工具的事务回滚**
Agent 任务：“在数据库创建一个用户，并给他发一封欢迎邮件”。
流程：DB Insert -> Email Send。
如果在发邮件失败了，如何利用 FRP 机制回滚 DB 的插入？

<details>
<summary><strong>参考答案</strong></summary>

*   **提示**：这涉及到 SAGA 模式或补偿事务（Compensating Transaction）在流中的实现。
*   **思路**：
    1.  我们将操作建模为 observables 序列：`createUser$` 和 `sendEmail$`。
    2.  `createUser$` 成功后，产生一个 `UserCreatedEvent`（包含 ID 和一个**补偿闭包** `deleteUser$`）。
    3.  使用 `concatMap` 连接 `sendEmail$`。
    4.  如果 `sendEmail$` 抛出错误 (`catchError`)：
        *   捕获错误。
        *   执行上一步保留的补偿闭包 `deleteUser$`。
        *   等待回滚完成后，向用户抛出最终失败事件。
    5.  **FRP 特性**：可以利用 `resource` 或 `using` 算子，当流意外终止（uncaught error）时自动触发清理逻辑。
</details>

**Q6: Python 沙箱的资源泄露**
如果用户让 Python 脚本打开了一个 10GB 的文件进行处理，然后立刻发出了 `Cancel` 信号。普通的线程取消可能无法立即释放文件句柄或内存。FRP 如何确保资源被清理？

<details>
<summary><strong>参考答案</strong></summary>

*   **提示**：关注 FRP 的 `TeardownLogic` 或 `Unsubscribe` 回调。
*   **思路**：
    1.  **封装**：将 Python 执行环境封装在一个自定义的 Observable 中。
    2.  **Teardown 注册**：在 Observable 创建时，返回一个清理函数（Teardown function）。这个函数包含 `docker kill` 或 `os.kill(pid)` 的逻辑。
    3.  **传播**：当用户 UI 点击“停止”，或者上游 `takeUntil` 触发，FRP 框架会自动调用整个链路下游的清理函数。
    4.  **强制性**：清理函数应设计为幂等的，并且采用“发射并遗忘（Fire and Forget）”策略调用底层 OS 命令，确保即使主线程卡顿也能杀掉子进程。
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

1.  **Backpressure 缺失导致的内存爆炸**
    *   **场景**：RAG 检索返回了 10,000 个 chunks，Python 脚本 print 了 100MB 的日志。
    *   **问题**：如果下游（UI 或 LLM）处理速度慢于上游生产速度，内存会积压。
    *   **调试技巧**：监控 Node/Process 的内存使用曲线。
    *   **解决**：在流上显式加 `take(N)`（截断），`sampleTime`（采样），或实现真正的 Pull-based Backpressure（虽然复杂，但对于大文件处理是必须的）。

2.  **Stale Closure (过闭包) 问题**
    *   **场景**：在 `mergeMap` 或回调中使用了外部变量（如 `currentPrompt`）。当流延时执行时，外部变量已经变了，但闭包里还是旧值。
    *   **调试技巧**：日志打印闭包内变量的值 vs 全局最新值。
    *   **解决**：永远不要依赖闭包外的可变状态。将所有需要的状态作为参数（Event Payload）在流中传递，或者使用 `withLatestFrom(stateSignal)` 算子在执行时刻获取最新状态。

3.  **Error Swallowing (错误吞噬)**
    *   **景**：管道中间某一步报错，Observable 终止（Complete），整个 Agent 突然“沉默”了，没反应。
    *   **原理**：根据 ReactiveX 协议，Error 事件是终止事件。
    *   **解决**：
        *   在关键算子（如 API 请求）内部使用 `catchError` 并返回一个代表错误的 Value（如 `Result.Failure`），而不是让 Error 冒泡炸断主管道。
        *   主管道应永远保持 Alive (`retry` or `catchError(return empty)` with logging)。

4.  **Python 输出乱序**
    *   **场景**：Python 的 `stdout` 和 `stderr` 是两个流，但在 UI 上需要合并展示。简单合并可能导致报错信息出现在日志之前，误导 Debug。
    *   **解决**：尽量保持时序，或者在 UI 层根据 Timestamp 重新排序。更好的是在 Python 侧包装一个结构化 Logger，将 stdout/stderr 统一封装为带序号的 `LogEvent`。

5.  **RAG 结果相关性低时的“强行回答”**
    *   **场景**：RAG 没搜到东西，返空列表。LLM 接收到空 Context，开始产生幻觉。
    *   **解决**：在 Pipeline 中加入 `filter` 或 `gate`。如果 `RetrievalResult.isEmpty`，则切换到 `NoDataPrompt`，或者直接阻断 RAG 分支，让 LLM 依靠自身训练数据（并提示用户“未找到外部资料，基于通用知识回答”）。
