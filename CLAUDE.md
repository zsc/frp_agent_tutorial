（交流可以用英文，所有文档中文）

## 项目目标
LLM 实时 agent 的抽象，如果以 Functional Reactive Programming 为基础来搭，怎么搞好？要处理 prompt management(内嵌环境状态播报，时间变化),用户异步动作，工具/API 失效处理, speculative exec, speculative decoding, dynamic batching, RAG / Python toolcall/DB, 类似 VAD 一样的事件触发器， race condition 处理, token budget 控制, 日志持久化，人类直观 UI（含音效）, 后台 debug (效果/token/latency)与可视化， 等等（再想一些） 编写一份基于以上主题的中文教程markdown。
要包含大量的习题和参考答案（答案默认折叠）。

文件组织是 index.md + chapter1.md + ...
不写代码。
提供 rule-of-thumb。

## 章节结构要求
每个章节应包含：
1. **开篇段落**：简要介绍本章内容和学习目标
2. **文字论述**：以文字论述为主，适当配上ASCII 图说明。
3. **本章小结**：总结关键概念和公式
4. **练习题**：
   - 每章包含6-8道练习题
   - 50%基础题（帮助熟悉材料）
   - 50%挑战题（包括开放性思考题）
   - 每题提供提示（Hint）
   - 答案默认折叠，不包含代码
5. **常见陷阱与错误** (Gotchas)：每章包含该主题的常见错误和调试技巧
