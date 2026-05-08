# Knowledge Base Index

## ai-coding

如何在 AI/LLM 时代有效编程：方法论、工作流、架构选择，以及关于"代码还重不重要"的论战。

| Article | Summary | Updated |
|---------|---------|---------|
| [AI Coding Philosophy: Solved or Sharpened?](ai-coding/ai-coding-philosophy.md) | "coding is solved" 与"fundamentals matter more"的张力，三家观点对照 | 2026-05-08 |
| [Working with AI as a PM](ai-coding/working-with-ai-as-pm.md) | 上下文工程：grill-me、计划-压缩-执行、ubiquitous language | 2026-05-08 |
| [Leaf Nodes and Deep Modules](ai-coding/leaf-nodes-and-deep-modules.md) | AI 友好的架构：在边界清晰的地方放手，在主干处人类把关 | 2026-05-08 |

## ai-agents

Agent 编排框架的设计原理、运行机制与稳定性问题（ROMA、LangGraph、AutoGen 等）。

| Article | Summary | Updated |
|---------|---------|---------|
| [Agent 如何避免无限递归](ai-agents/avoiding-infinite-recursion.md) | 四类根因 + 17 条预防策略，归到目标/上限/状态/外部/监控五个面 | 2026-05-08 |
| [Reactive vs. Proactive AI Agents：本质区别与选型](ai-agents/reactive-vs-proactive-ai-agents.md) | 区别在"有没有内部目标驱动"；五分类谱系；选型不追求"先进"看任务 | 2026-05-08 |
| [Memory OS：Agent 的记忆系统怎么设计](ai-agents/memory-os-agent-memory-design.md) | Letta/MemGPT 抽象：context window=RAM、外存=Disk；五种记忆类型 + Agent 自主管理 + Sleep-Time 后台整理 | 2026-05-08 |
| [Prompt Caching：原理、用法和坑](ai-agents/prompt-caching.md) | LLM 前缀缓存机制 + Anthropic cache_control 用法 + 价格账 + 6 类常见踩坑 | 2026-05-08 |
| [Claude Code Harness 架构拆解：四层模型 + 生产级 agent 工程经验](ai-agents/claude-code-harness-architecture.md) | 三层 → 四层框架（infra 是产品死掉的地方）；async generator loop、工具并发分类、压缩层级、7 阶段权限、重试状态机、sub-agent worktree 隔离、composition over modification | 2026-05-08 |
| [Parlant Guideline 系统：怎么设计，怎么约束住 LLM](ai-agents/parlant-guideline-system.md) | 规则做成关系图 + 三阶段（LLM 匹配 → 图算法过滤 → 短 prompt 生成 + 自检）+ 五种 LLM 失败模式 vs 五个解法 | 2026-05-08 |

## openclaw

OpenClaw gateway 与插件生态的设计、能力面和扩展模式。

| Article | Summary | Updated |
|---------|---------|---------|
| [OpenClaw Plugin API: Extension Points and Deliberate Omissions](openclaw/plugin-api.md) | 四种切入方式（register / hook / runtime / event）+ 刻意不做的能力（cron / 持久化 / DB）+ clawos extension 元模式 | 2026-05-08 |

## benchmark

AI 系统的评估方法论：benchmark 设计、retrieval/embedding eval、leaderboard 与生产数据的差距。

| Article | Summary | Updated |
|---------|---------|---------|
| [Generative Benchmarking：用自己的 data 评估 retrieval 系统](benchmark/generative-benchmarking.md) | MTEB 三个雷 + Chroma 三步生成 pipeline + 三个反直觉发现（朴素生成泄漏 ground truth / 越真实 recall 越低 / 排名换数据就翻） | 2026-05-08 |
