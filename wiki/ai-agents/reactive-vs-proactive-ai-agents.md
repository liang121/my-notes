# Reactive vs. Proactive AI Agents：本质区别与选型

> Sources: Tekrevol blog, 2025-09-08
> Raw: [Reactive vs. Proactive AI Agents](../../raw/ai-agents/reactive-vs-proactive-ai-agents.md)

## Overview

Reactive 和 Proactive 是 AI agent 的两种决策模式。一句话区分：**Reactive 是"看到什么做什么"，Proactive 是"为了到达什么而做什么"**。区别不在"AI 强不强"，而在内部有没有**世界模型 + 目标 + 记忆**这三件套。下文整理它们的本质差异、Russell-Norvig 经典五分类怎么对应到这个谱系，以及实际选型时该看什么——并指出原文里几个值得打折扣的说法。

## 本质区别：差的不是模型大小，是"内部状态"

| 维度 | Reactive | Proactive |
|------|---------|-----------|
| 决策依据 | 当前输入 | 当前输入 + 历史 + 对未来的预测 |
| 内部状态 | 无（或极简） | 世界模型 + 目标 + 学到的策略 |
| 触发器 | 只有外部刺激 | 外部刺激 **+ 内部目标驱动** |
| 失败模式 | 反应错（规则不全） | 规划错（目标偏移、循环） |

关键不是"哪个更智能"，是 **agent 有没有'内部驱动力'**：Reactive 没收到输入就不做事，Proactive 哪怕没人戳它，也会因为"目标还没达成"自己往前推一步。

## 经典五分类怎么落到这个谱系

源自 Russell-Norvig 《AIMA》经典分类，原文复述了一遍。把它放回 Reactive↔Proactive 的轴上看更清楚：

```
更 Reactive ←─────────────────────────────────────→ 更 Proactive
Reflex   →   Model-based reflex   →   Goal-based   →   Utility-based   →   Learning
（纯刺激响应）  （加内部世界模型）    （加目标+规划）   （加效用函数）     （加自我改进）
```

- **Reflex agent** —— 纯 if-then。门口的红外感应灯。
- **Model-based reflex** —— 有"环境长什么样"的内部模型，所以即使刺激相同，状态不同时反应也不同。比如知道"现在是夜里"的温控器。
- **Goal-based** —— 第一个真 Proactive：会问"做这一步，离目标更近吗"。规划、搜索算法在这里登场。
- **Utility-based** —— 多目标时按效用函数权衡。比如"快"和"省油"的取舍。
- **Learning agent** —— 从反馈里改自己的策略/模型/效用函数。强化学习、在线学习都属此类。

> 注意一个常见误读：**LLM-based chatbot 不是单纯的 Reflex agent**。它有上下文窗口（轻量记忆）、有 system prompt（轻量目标）、可以调工具（影响环境）。把它直接钉在 Reactive 一端会低估它能做的事。

## 选型：看任务，不看"先进程度"

原文给了一张"业务成熟度 → agent 类型"的表，但那张表暗含一个错误前提：**Proactive 一定比 Reactive 更高级、更值得追求**。实际选型应该反过来——**先看任务本身需要什么**：

**选 Reactive 的场景**
- 输入到输出的映射是确定的（垃圾邮件过滤、关键词路由）
- 延迟敏感（每多一层规划就多几十毫秒）
- 训练数据少、上下文窗口贵
- 出错代价低、可由用户立即纠正

**选 Proactive 的场景**
- 单步决策不够，需要多步规划（订机票、修代码、做研究）
- 环境会变化，需要适应（库存预测、自动驾驶）
- 用户期望"被服务"而不是"被回应"（私人助理类）

**注意一个常见的误用：把 Proactive 套在不该套的地方。** 用 Goal-based agent 写一个本来 5 行 if-else 就能搞定的客服分流，就是过度工程——更慢、更贵、更难调。

## Proactive Agent 特有的失败模式

原文没强调，但这是实践中最痛的：**Proactive 越主动，越容易陷入死循环**。因为它有"目标还没达成"这个内部驱动力，会自己反复尝试。常见反模式：

- planner → executor → critic → "不满意，再来一次" → 无限循环
- "重试到成功为止"——但"成功"的判定不严密
- 子 agent 互相调，谁也不知道终止条件

防这类问题的具体策略，看 [Agent 如何避免无限递归](avoiding-infinite-recursion.md)。简单说就是：**给目标量化的成功条件 + 硬性次数/时长上限 + 状态机写死 + 异常分支预案 + 监控日志**。

## 一句话总结

Reactive vs Proactive 的本质是 **"有没有内部目标驱动"**。选型时不要追求"看起来更智能"，要问：**这个任务，agent 不被外部戳一下，需不需要自己往前走一步？** 答"不需要"就 Reactive，答"需要"才 Proactive，并且在 Proactive 那条路上把死循环防线先做好。

## See Also

- [Agent 如何避免无限递归](avoiding-infinite-recursion.md) —— Proactive agent 因为"自己往前推"特别容易陷的坑
- [Memory OS：Agent 的记忆系统怎么设计](memory-os-agent-memory-design.md) —— Proactive agent 需要"内部状态"这个抽象，Memory OS 是一种工程实现方案；不是必须的（Reactive agent 也可能用外部记忆/RAG）
