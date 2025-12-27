# 《用 FRP（Functional Reactive Programming）搭建 LLM 实时 Agent：从抽象到落地》

> 目标：用 **FRP 的“事件流 + 时间语义 + 可组合算子”**，把实时 LLM Agent 需要处理的一切——prompt management、异步用户动作、工具/API 失效、speculative、dynamic batching、RAG/DB/Python toolcall、VAD 式触发器、race condition、token budget、日志持久化、直观 UI（含音效）、后台 debug 可视化——抽象成可测试、可观测、可扩展的系统。

---

## 阅读路线建议

- **想先把系统跑起来**：Chapter 1 → 2 → 3 → 4 → 6 → 7 → 10
- **重点解决实时与并发**：Chapter 2 → 5 → 8 → 9 → 11
- **重点解决质量与可观测性**：Chapter 12 → 13 → 14
- **偏工程化/上线**：Chapter 14 + Appendix

---

## 章节目录（index）

### [Chapter 1｜问题空间与总体架构：为什么用 FRP](chapter1.md)
1.1 实时 LLM Agent 的“痛点清单”与设计目标  
1.2 FRP 适配点：时间事件、组合性、可回放  
1.3 典型系统分层：UI/Ingress → FRP Runtime → Planner/Policy → Tools → Memory/Store  
1.4 关键抽象一览：Event、Stream、Signal、Reducer、Effect、Clock、Scheduler  
1.5 “一次对话”到“持续运行系统”的范式迁移  
1.6 本书约定：术语、示例域、目录结构、代码风格（如适用）

---

### [Chapter 2｜FRP 基础：用事件流描述 Agent](chapter2.md)
2.1 Stream vs Signal：离散事件与连续状态  
2.2 时间语义：逻辑时间、物理时间、虚拟时间（可回放测试）  
2.3 常用算子映射：map/merge/switchLatest/debounce/throttle/window/scan  
2.4 “状态机”如何用 scan/reducer 表达  
2.5 Effect 模型：把副作用（LLM/工具/IO）从纯逻辑剥离  
2.6 错误通道：error stream、retry/backoff、circuit breaker 的 FRP 表达  
2.7 背压与流控：buffer、drop、latest、priority queue  
2.8 最小可运行示例：从用户输入流到 LLM 输出流

---

### [Chapter 3｜核心运行时：Agent 作为“可组合的反应系统”](chapter3.md)
3.1 Runtime 的职责边界：调度、隔离、观测、资源预算  
3.2 “一切皆事件”：用户、工具、定时器、模型 token、UI、网络、系统状态  
3.3 AgentState 设计：可序列化、可回放、可部分更新  
3.4 Reducer 规范：决定性（deterministic）与副作用外置  
3.5 Effect 执行器：并发策略、隔离级别、超时、取消（cancellation）  
3.6 Checkpoint 与 Snapshot：恢复、回滚、跨版本迁移  
3.7 多 Agent 与层级：parent/child stream、supervisor tree  
3.8 插件化：Tool、Memory、Policy、UI 的可插拔协议

---

### [Chapter 4｜Prompt Management：内嵌环境状态播报 + 时间变化](chapter4.md)
4.1 Prompt 不是字符串：把 prompt 视为“可变信号（Signal）”  
4.2 “环境状态播报”策略：系统时间/时区、位置、设备、网络、任务上下文  
4.3 时间变化如何进 prompt：tick stream → state signal → prompt patch  
4.4 Prompt 模板分层：系统/开发者/会话/任务/工具回执/安全护栏  
4.5 Prompt Patch：增量更新 vs 全量重建（cache 与一致性）  
4.6 长对话压缩：摘要流、记忆流、引用片段（grounding）  
4.7 注入攻击防护：边界标记、工具输出隔离、引用策略  
4.8 Token 预算与 prompt 的动态裁剪（与 Chapter 9 联动）

---

### [Chapter 5｜用户异步动作：打断、撤回、改口、多模态输入](chapter5.md)
5.1 用户输入流建模：text/voice/gesture/button、编辑事件、撤回事件  
5.2 “打断”语义：interrupt stream + cancellation token  
5.3 版本化输入：InputRevision、CRDT/OT 思路与简化实现  
5.4 多模态融合：时间窗对齐（windowing）与迟到事件（late events）  
5.5 语音场景：partial transcript、最终稿、端点检测  
5.6 UI 意图事件：hover、focus、copy、“继续说”、反馈按钮  
5.7 反应式 UX：输入变化如何驱动 speculative 与工具预热

---

### [Chapter 6｜工具 / API 调用：失效处理、回退、幂等与隔离](chapter6.md)
6.1 Tool 作为 Effect：请求/响应/超时/取消/重试 都是事件  
6.2 失败分类：网络、鉴权、配额、语义错误、不可恢复错误  
6.3 Circuit Breaker 与 Bulkhead：FRP 的实现方式  
6.4 Retry/Backoff/Jitter：可观测 + 可配置 + 可测试  
6.5 幂等键与去重：重复触发、重放、断线重连  
6.6 回退策略：替代工具、降级结果、只读模式、提示用户  
6.7 结构化工具回执：ToolReceipt schema（供 prompt/日志/UI）  
6.8 安全沙箱：Python/DB/文件系统隔离（与 Chapter 15/Appendix 联动）

---

### [Chapter 7｜RAG / DB / Python toolcall：把知识管道变成事件流](chapter7.md)
7.1 RAG 管道拆解：query → retrieve → rerank → synthesize  
7.2 检索触发器：基于意图、基于不确定性、基于时间/位置  
7.3 增量检索：streaming retrieval + streaming synthesis  
7.4 DB 查询：schema-aware、参数校验、最小权限  
7.5 Python 工具：执行环境、资源限制、结果序列化  
7.6 缓存与一致性：embedding cache、result cache、stale-while-revalidate  
7.7 证据引用与可追溯：引用片段的 ID 与 UI 展示  
7.8 多来源融合：合并冲突、置信度、去重、时间优先级

---

### [Chapter 8｜Speculative Exec & Speculative Decoding：让系统“更快也更稳”](chapter8.md)
8.1 两类 speculative：执行（tools）vs 解码（tokens）  
8.2 触发条件：高置信路径、低风险操作、用户可见延迟预算  
8.3 Speculative Exec：预取、预热、并行候选计划  
8.4 Speculative Decoding：草稿模型/多分支、接受-拒绝（accept/reject）  
8.5 与 FRP 的契合：merge/switchLatest + cancel  
8.6 一致性与撤销：如何处理“先展示后回滚”  
8.7 成本控制：投机比率、阈值、自适应策略  
8.8 指标闭环：命中率、浪费率、延迟收益

---

### [Chapter 9｜Dynamic Batching & 并发：吞吐、迟与公平性](chapter9.md)
9.1 batching 的对象：模型请求、embedding、检索、DB、工具调用  
9.2 批处理窗口：time-based vs size-based vs adaptive  
9.3 优先级与公平：交互优先、后台任务、长尾保护  
9.4 背压与队列：drop/compact/latest、优雅降级  
9.5 资源预算：CPU/GPU/IO/并发连接数  
9.6 多租户：隔离、配额、噪声邻居  
9.7 取消传播：从 UI 到模型到工具的端到端取消  
9.8 实战模板：一个可配置的 BatchScheduler

---

### [Chapter 10｜事件触发器：做一个“类似 VAD”的 Agent Trigger System](chapter10.md)
10.1 触发器类型：时间触发、阈值触发、模式触发、外部事件  
10.2 VAD 类比：从“声学端点”到“意图端点”  
10.3 触发器 DSL：when/and/or/until、cooldown、hysteresis  
10.4 低抖动设计：debounce/throttle + 状态锁存  
10.5 联动 prompt：触发器驱动“环境播报”与“注意力切换”  
10.6 触发器可观测：命中原因、上下快照、误触分析  
10.7 安全触发器：高风险动作二次确认、人类 in-the-loop  
10.8 触发器测试：虚拟时间回放与边界用例

---

### [Chapter 11｜Race Condition 与一致性：并发世界的“真相维护”](chapter11.md)
11.1 竞态来自哪里：用户打断、工具回包乱序、多分支推理  
11.2 FRP 的典型解法：switchLatest、takeUntil、sequenceId、monotonic clock  
11.3 事件排序：逻辑时钟、Lamport/Vector clock（简化版实践）  
11.4 事务语义：原子更新、乐观并发控制（OCC）  
11.5 冲突解决：last-write-wins vs domain-specific merge  
11.6 UI 一致性：流式输出与最终状态对齐  
11.7 可重放调试：记录事件序列 → 复现竞态  
11.8 “正确性预算”：哪些一致性必须强，哪些可以弱

---

### [Chapter 12｜Token Budget 控制：预算就是产品体验](chapter12.md)
12.1 预算分解：prompt tokens / completion tokens / tool tokens / RAG tokens  
12.2 在线估算：tokenizer、上界估计、流式扣费  
12.3 策略：截断、摘要、引用、延迟检索、分段回答  
12.4 多轮规划：先问澄清 vs 直接回答 vs 先检索  
12.5 风险动作预算：高成本工具与高风险操作的 gating  
12.6 在 FRP 中实现：BudgetSignal + Guard operator  
12.7 面向不同模型：上下文窗口差异与动态路由  
12.8 指标：每任务 token、每用户 token、无效 token（浪费率）

---

### [Chapter 13｜日志持久化与可观测性：Tracing/Replay/Metric 一体化](chapter13.md)
13.1 日志的对象：Event、State、Effect、Span、UI Frame  
13.2 结构化日志：schema、版本、向后兼容  
13.3 分布式追踪：traceId/spanId、跨工具链路  
13.4 Replay：用事件日志重建状态（debug 的“时光机”）  
13.5 指标体系：效果/token/latency/失败率/命中率（RAG、speculative、batching）  
13.6 采样策略：全量 vs 采样、按风险/异常提升采样  
13.7 隐私与脱敏：PII、密钥、用户内容隔离（ Chapter 15 联动）  
13.8 面向产品：把可观测性做成“可解释性 UI”

---

### [Chapter 14｜人类直观 UI（含音效）与后台 Debug 可视化](chapter14.md)
14.1 UI 也是反应式：UI = f(StateSignal, EventStream)  
14.2 流式输出呈现：token stream、段落稳定器、排版抖动控制  
14.3 音效与触觉：事件映射（成功/失败/提醒/打断/回滚）  
14.4 “发生了什么”面板：工具调用时间线、引用证据、预算条  
14.5 Debug 模式：效果对比、token/latency 分解、瀑布图  
14.6 在线剖析：热路径、队列深度、批处理窗口、取消率  
14.7 可视化事件图：stream DAG、状态机图、触发器命中树  
14.8 面向运营：回放链接、问题标注、用户反馈闭环

---

### [Chapter 15｜安全、隐私与治理：让 Agent 可上线、可合规](chapter15.md)
15.1 Threat Model：prompt injection、数据外泄、工具滥用、越权  
15.2 权限与最小化：tool allowlist、参数校验、数据隔离  
15.3 内容安全：策略层（policy）、模型层（moderation）、UI 层提示  
15.4 机密管理：key vault、短期 token、审计  
15.5 人类在环：高风险操作审批、可撤销、双确认  
15.6 数据保留：日志保留策略、删除/导出、合规要求  
15.7 红队与评测：攻击样本库、回归测试  
15.8 上线前清单：安全闸门与演练

---

### [Chapter 16｜评测与持续迭代：从“能用”到“好用”](chapter16.md)
16.1 评测维度：正确性、时延、稳定性、可解释、成本  
16.2 任务基准：脚本化对话、随机化事件、噪声注入  
16.3 A/B 与灰度：策略变更、模型路由、触发器阈值  
16.4 质量闭环：用户反馈 → 样本归因 → 策略更新  
16.5 失败分析：工具失败、检索失败、竞态失败、预算失败  
16.6 模型与提示版本化：可回滚、可追踪  
16.7 线上异常自动归类与告警  
16.8 成本优化：批处理、缓存、投机、路由综合调参

---

### [Chapter 17｜参考实现：一个端到端的 FRP Agent（伪代码 + 架构图）](chapter17.md)
17.1 项目结构与模块拆分  
17.2 Event Bus / Stream Runtime 的最小实现  
17.3 PromptSignal 与 BudgetSignal 的实现  
17.4 Tool Effect 执行器与回执  
17.5 RAG 管道与证据引用  
17.6 Speculative 与 batching 的组合示例  
17.7 Trigger DSL 示例（VAD 类）  
17.8 UI/Debug 面板的事件对接

---

## 附录（可选）

### [Appendix A｜术语表 & FRP 算子速查](chapter18.md)
A.1 术语对照（FRP/并发/LLM）  
A.2 算子速查表（常见问题 → 对应算子组合）  
A.3 常见坑：switchLatest 滥用、背压缺失、取消不传递

### [Appendix B｜数据结构与协议：Event/State/Receipt/Trace Schema](chapter19.md)
B.1 Event schema（含序号/时间戳/来源/关联）  
B.2 State schema（可序列化与迁移）  
B.3 ToolReceipt schema（UI 与 prompt 复用）  
B.4 Trace/Span schema（跨服务一致）

### [Appendix C｜测试方法：虚时间、回放、混沌工程](chapter20.md)
C.1 虚拟时间驱动的单测  
C.2 事件回放测试（确定性回归）  
C.3 混沌注入：延迟、乱序、丢包、超时、限流  
C.4 性能基准：吞吐/时延/成本曲线

### [Appendix D｜部署与运维：配置、灰度、告警](chapter21.md)
D.1 配置系统：阈值、预算、路由策略  
D.2 灰度与回滚：版本矩阵  
D.3 告警与 SLO：延迟、错误率、成本  
D.4 容灾：断网/降级/只读模式

---

## 文件组织（预期）
- `index.md`（本文件）
- `chapter1.md` … `chapter17.md`
- `appendix-a.md` … `appendix-d.md`
