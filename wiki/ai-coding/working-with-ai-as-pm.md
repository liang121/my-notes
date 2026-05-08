# Working with AI as a PM

> Sources: Eric (Anthropic), 2026-04-23; Matt Pocock (aihero.dev), 2026-05-06; Boris Cherny (Anthropic), 2026-05-06
> Raw: [Vibe Coding in Prod — Eric](../../raw/ai-coding/vibe-coding-in-prod-eric.md); [Software Fundamentals — Matt Pocock](../../raw/ai-coding/software-fundamentals-matt-pocock.md); [Coding Is Solved — Boris Cherny](../../raw/ai-coding/coding-is-solved-boris-cherny.md)

## Overview

三家观点大部分时候在吵架，但在"如何把任务交给模型"这件事上罕见一致：**前期投入和模型对齐，远比 prompt 措辞或工具选择重要**。Eric 把这种姿态叫"做 Claude 的 PM"，Matt 用 grill-me + ubiquitous language 来落地，Boris 则把它简化成"give it a target and it will hill-climb until done"。三种说法指向同一个工程实践：**用前期 context engineering 换后期 vibe 自由度**。

## "Be Claude's PM"——Eric 的核心比喻

Eric 把 Claude 比作入职第一天的工程师：聪明、能干，但完全不了解你的代码库、客户、约束。直接丢一句 "implement this feature" 等于把新人扔进无人区。

Eric 的实操（22,000 行生产 PR 就这样做出来的）：

1. **开一个独立 session 做 context-gathering**。让 Claude 用 15–20 分钟探索 codebase、找相关文件、问澄清问题。
2. **共同写 plan**。把这次探索沉淀成 markdown 文件——不是 prompt，是 artifact。
3. **新 session + plan 文件 + "go"**。原来的探索 context 不需要了，模型只需要 plan。
4. **必要时 compact**：当 Claude 做到一个"人类也会在这里吃个午饭"的自然停顿点。把 10 万 token 探索压缩成几千 token plan 继续干活。

> "Ask not what Claude can do for you, but what you can do for Claude."

## "Grill Me"——Matt 的对话工具

Matt 从 Frederick Brooks 的 *The Design of Design* 借来了 **design concept** 的概念：两个人共同设计时，"那个东西的样子"是漂浮在两人之间的 ephemeral idea，不是 markdown 文件能装下的。AI 协作的失败模式之一就是"设计概念没对齐"。

Matt 的 grill-me skill 几行字：

> Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one by one.

效果：AI 会问 40–100 个问题。这看起来繁琐，但 Matt 认为**比 Claude Code 默认 plan mode 更优**——plan mode 急于产出 asset，grill-me 慢一拍但保证设计概念一致。

**对照 Eric**：两人方向一致——前期投入。差别在于 Eric 的"context-gathering session"侧重让 AI 主动探索 codebase（artifact-driven），Matt 的 grill-me 侧重让 AI 把人类思路榨干（dialogue-driven）。**老板可以两者结合用**：grill-me 对齐意图 → 让 Claude 探索 codebase → 写 plan → 新 session 执行。

## Ubiquitous Language——共享词汇表

Matt 借自 Domain-Driven Design 的概念，用 skill 自动扫 codebase 抽出一份术语 markdown 表。把它扔给 Claude 当固定 context。效果有两个：
1. **planning 更准**——AI 用对的词。
2. **思考链 (thinking trace) 变 less verbose**——Matt 自己读 trace 时观察到。

这条对应 Eric 强调的"don't over-constrain"似乎冲突，其实不矛盾：ubiquitous language 是**词汇约束**，不是**实现约束**。Matt 说要建立词汇，Eric 说不要在实现上设死规矩，两层完全可以并存。

## Boris 的极简版："给目标"

Boris 的工作方式比 Eric 和 Matt 都更极端——他 **以手机为主战场**，挂着 5–10 个 session、几百个 agent，夜间几千个。他没空做长 plan，靠的是：

1. **Hill-climbing**：模型 4.7 后特别会"摸清 workflow"——给一个目标，让它自己 hill-climb 直到完成。
2. **Loops**：把重复任务（修 CI、auto-rebase、抓 Twitter 反馈）做成 cron-like 周期 job。
3. **Routines**：服务端版的 loop，关掉笔记本也跑。

> "Loops are the future."

Boris 的方式可以理解为**前期 context 摊销在 loop 配置里**——一次性把 babysit-PR 这种重复活的"PM 心态"写进 loop spec，之后每次跑就直接干活，不再 PM。

## 综合方法论（融合三家）

老板自己的实操可以这样组织：

1. **判断任务类型**
   - 一次性、需要思考的功能 → Eric 的 "be PM"（context session + plan + execute）
   - 需求模糊、设计概念不清 → 加上 Matt 的 grill-me 前置
   - 重复性、有自然终止条件的任务 → Boris 的 loop / routine
2. **建项目级 ubiquitous language 文件**——一次性投入，长期收益（Matt 路线）。
3. **plan artifact 永久保存** —— 不只是一次性使用，下次类似任务直接复用（Eric 路线）。
4. **不要在实现层过度约束** —— Eric 反复强调"don't over-constrain，把它当 junior engineer"。约束放在词汇和接口层（Matt 的 deep modules + ubiquitous language），实现层放手。

## TDD as Context Tool

Eric 在 Q&A 里提到 TDD 在 vibe coding 里"非常有用"：
- 让 Claude 写 **3 个 end-to-end 测试**：happy path + 两个 error case。
- 通用、避免实现细节。
- 模型写完测试后 Eric 经常**只读 tests**——同意测试 + 测试通过 = 可信。

这本质上是**用测试做 PM 角色的延伸**——测试是 spec 的可执行形式，比 markdown plan 更精确。Matt 也支持 TDD，理由不同（驱动小步走、强迫慢下来）；两人理由互补，一个偏 spec，一个偏 pacing。

## 何时应该 *不* 这样做

Eric 自己警告：完全不懂代码的人**不应该**做生产级 vibe coding，因为"问不出对的问题"。PM 心态需要一定的领域 context 才能成立。这条对应 Boris 的 printing press 类比：**领域专家 + AI > 普通工程师 + AI**——但**白纸 + AI ≠ 专家**。

## See Also

- [AI Coding Philosophy: Solved or Sharpened?](ai-coding-philosophy.md)
- [Leaf Nodes and Deep Modules](leaf-nodes-and-deep-modules.md)
