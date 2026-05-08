# Memory OS：Agent 的记忆系统怎么设计

> Sources: Letta 官方文档与 MemGPT 论文整理稿（飞书剪存 PDF, 2026-05-08）
> Raw: [Letta - memory os](../../raw/ai-agents/letta-memory-os.md)

## Overview

**Memory OS 不是一个新组件，而是一个类比**：把 LLM agent 的记忆当成操作系统来设计——context window 是 RAM（容量小、永远可见、CPU 直接读），外部存储是 Disk（容量大、要主动调进来）。Agent 自己当"应用程序"，通过一组 memory tools 在 RAM 和 Disk 之间搬运信息。MemGPT 论文提出这个抽象，Letta 把它做成了开源框架，区分**五种记忆类型 × 两类作用域**，并把"何时读、何时写、何时压缩"交给 LLM 自主决定。理解这个框架，关键不是记住五个名字，而是理解一个核心权衡：**什么信息值得占 context（贵但快），什么信息扔到外存（便宜但要检索）**。

## 一、为什么需要 Memory OS：context window 的真问题

LLM 的 context window 有三个问题，单纯加大窗口都解决不了：

1. **token 成本线性涨**：哪怕模型支持 1M tokens，每次对话都把全量历史灌进去，成本和延迟都顶不住。
2. **信息稀释**：context 越长，模型越容易"看不见"中间的关键信息（lost-in-the-middle）。
3. **信息错配**：对话历史里 99% 的内容是当前问题不需要的，但你又不知道哪 1% 重要。

Memory OS 的回应：**别把 context window 当存储用**。它是工作内存（working memory），只放当前回合 LLM 真正需要"看见"的东西；其他信息扔到外部存储，等 agent 觉得需要时再用工具捞回来。

## 二、五种记忆类型：按"作用域 × 信息形态"切分

Letta 把记忆按两个维度切开：**是否在 context 内** × **是结构化摘要还是原始流水**。

| 类型 | 在 context？ | 形态 | 典型用途 |
|---|---|---|---|
| **Message Buffer** | 是（自动） | 最近 N 条消息原文 | 当前对话的上下文 |
| **Core Memory** | 是（pinned） | 结构化 blocks（human/persona/summary） | 用户画像、Agent 人设、对话摘要——agent 始终需要看见的关键事实 |
| **Archival Memory** | 否 | Vector DB，语义搜索 | 长期事实、外部知识、可被语义召回的细节 |
| **Recall Memory** | 否 | 完整对话日志，文本/日期搜索 | "我们上周三聊过 X 吗？"这类按精确字面/时间检索的需求 |
| **Filesystem** | 否 | 文件 + folders，open/grep/语义搜索 | PDF、论文、报告——已有的外部文档集 |

注意 Archival 和 Recall 的区别——这是新手最容易混的：

- **Archival** 是 agent 主动"沉淀"出来的事实，写入时已经过提炼。"用户 2024 年 1 月提到对坚果过敏" 适合放这里。
- **Recall** 是对话历史的**完整原始日志**，自动持久化，不挑选。等于一个可搜索的 git log。

> 经验：**Filesystem 不是自动记忆**，它是"让 agent 访问已有文档"的接口，需要手动上传。如果你想要自动从对话里学到的信息，永远是 Archival。

## 三、核心设计原则：Core 存"执行摘要"，Archival 存"完整细节"

这是 Letta 文档里最值得抄走的一条经验：

```
Core Memory:    "用户对坚果过敏"           （始终可见，几十字）
Archival Memory: "2024-01-15 用户提到对花生、杏仁、
                  核桃都过敏，症状是皮疹"     （按需检索，详细）
```

为什么这样切？

- Core Memory **每次对话都在 context 里占 token**——所以必须是"executive summary"级别，每个字都得是 agent 下回合可能用上的。默认每个 block 限制 2000 字符（项目里 human/summary 设到了 5000），就是逼你压缩。
- Archival Memory **不占 context**——所以可以放冗余的、长的、带时间戳的原始事实。代价是 agent 需要决定 "这个问题要不要去搜一下"。

写错地方的代价：把详细事实塞进 Core，context window 很快被占满，剩下消息都被挤到 Recall 去了；把摘要扔进 Archival，agent 每次都得搜一遍，对简单问题反应迟钝。

## 四、Agent 自主管理：为什么这是 Letta 的灵魂

Letta 不是把记忆"框架式地"管理掉，而是把一组工具暴露给 LLM，让 LLM 自己决定调用：

| 工具类 | 工具 | 干什么 |
|---|---|---|
| Core 编辑 | `memory_replace` / `memory_insert` / `memory_rethink` | Agent 主动改写自己脑子里的 block |
| Archival 读写 | `archival_memory_search` / `archival_memory_insert` | Agent 决定何时沉淀知识、何时去捞知识 |
| Recall 检索 | `conversation_search` / `conversation_search_date` | Agent 决定要不要翻历史对话 |
| Filesystem | `open_file` / `grep_file` / `search_file` | Agent 在外部文档里找东西 |

一个典型回合长这样：

```
用户问："我之前跟你说过我喜欢什么运动来着？"
   ↓
Agent 看 Message Buffer 没找到 → 看 Core Memory 也没找到
   ↓
Agent 自主调用 conversation_search("运动") + archival_memory_search("运动偏好")
   ↓
搜到："用户说喜欢游泳和跑步"，注入 context
   ↓
Agent 回答 + 调用 memory_replace 把这条事实写进 human block
```

**这是和"系统自动注入相关记忆"路线最关键的分歧**。自动注入路线（典型如 RAG-style memory）：每个 query 来了，外部系统先做检索，把 top-K 相关记忆灌进 context，然后让 LLM 回答。Letta 路线：什么都不灌，让 LLM 自己判断要不要搜、搜什么关键词。

两条路线各有代价：

- **自动注入**：每次都搜，召回率高，但召回噪声也会污染 context；agent 失去了"我不记得"这种诚实信号。
- **Agent 自主**：agent 可能错过该搜没搜的情况；好处是 context 更干净，agent 行为更像人——会主动说"等我查一下"。

Letta 押注后者，因为前者本质上是把 agent 当成 RAG 系统的后端模板，浪费了 LLM 的 reasoning 能力。

## 五、Sleep-Time Compute：把记忆整理放到后台

这是 Letta 比 MemGPT 更进一步的地方。普通流程里，agent 既要回答用户、又要顺手更新 memory，两件事抢同一次 LLM 调用。结果就是要么响应慢，要么 memory 整理潦草。

Sleep-Time 的方案：起两个 agent 共享同一组 memory blocks。

```
Multi-Agent Group
├─ Primary Agent       ←→  共享 Memory Blocks  ←→  Sleep-Time Agent
│  - 处理对话                                       - 异步更新记忆
│  - 快速响应                                       - 重组/优化记忆
```

Primary Agent 负责对话，响应不阻塞；Sleep-Time Agent 在后台触发（默认每 5 步一次），用 `memory_rethink` 重组 Core Memory，用 `memory_finish_edits` 提交批量改动。

这个抽象很 OS——主用户进程跑得快，后台守护进程做 GC、做整理。代价是多一个 agent 的 LLM 调用成本，所以默认不开启。

## 六、Context Engineering：上下文窗口的组装顺序

Agent 调 LLM 时，context 不是随便堆的，有固定顺序：

```
Context Window
1. System Prompt           ← Agent 基础指令（不变）
2. Core Memory Blocks      ← human / persona / summary（始终可见，缓存友好）
3. Tool Definitions        ← 可用工具的 JSON Schema（不变）
4. Message Buffer          ← 最近对话消息
5. [Tool Results]          ← 当前回合工具调用的结果（动态）
```

这个顺序的意义在于 **prompt cache 的边界**：1-3 是稳定的，4-5 是变化的。把不变的放前面，KV cache 命中率最大化。如果你把 Core Memory 放在 Message Buffer 后面，每次 Core 一变，前面缓存就废了。

当 Message Buffer 顶到 `max_message_buffer_length` 时，Letta 自动压缩：旧消息总结成 recursive summary，原文挪到 Recall Memory，最近 `min_message_buffer_length` 条消息保留以维持对话连续性。

## 七、设计你自己的 Agent Memory 系统：决策清单

不是每个 agent 都要照搬五种记忆。问自己几个问题：

1. **会有跨会话状态吗？** 没有 → 只要 Message Buffer，别折腾。
2. **有"用户画像"这种始终该可见的事实吗？** 有 → 加 Core Memory（human + 可能的 persona）。
3. **会积累需要语义召回的长期知识吗？** 会 → 加 Archival Memory（vector DB）。
4. **会被问"我们之前聊过 X 吗"这类历史问题？** 会 → 加 Recall Memory（持久化对话日志 + 文本搜索）。
5. **要让 agent 读已有文档（PDF/论文/报告）吗？** 要 → 加 Filesystem。
6. **响应延迟敏感 + 记忆需要复杂整理？** 是 → 考虑 Sleep-Time agent。

通用反模式：

- ❌ 把所有事实塞 Core Memory：context 爆掉、prompt cache 失效。
- ❌ 把全量对话历史灌 context：成本和 lost-in-the-middle 双重崩溃。
- ❌ 不区分 Archival 和 Recall：要么搜不到精确字面，要么语义搜索失效。
- ❌ 只让系统自动注入，不给 agent 工具：失去"我不知道"这种重要信号。

## 八、对项目当前实现的观察

PDF 里标注的项目状态显示：

- 已用：Core / Archival / Recall + 三件 memory 编辑工具（replace / insert / rethink）
- 未用：Filesystem、Sleep-Time、`conversation_search_date`、`memory_finish_edits`

合理。Filesystem 需要业务侧设计文档上传流程；Sleep-Time 增加成本，应该在记忆质量真的不够时再开；按日期搜索是低频需求。当前配置（human/persona/summary 三个默认 block，limit 分别 5000/2000/5000）是把"对用户的认识"和"对话总摘要"调宽了，把"agent 自己人设"压窄了——这个权衡符合"agent 模板化、用户个性化"的产品定位。

## See Also

- [Reactive vs. Proactive AI Agents：本质区别与选型](reactive-vs-proactive-ai-agents.md) —— Proactive agent 需要的是"内部状态"这个抽象，Memory OS 是一种具体的工程实现方案。注意这不是双向必要关系：Reactive agent 也可能挂外部记忆（RAG），Proactive agent 的"目标 + 世界模型"也可以用别的方式实现。
