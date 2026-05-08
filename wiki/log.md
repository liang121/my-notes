# Wiki Log

## [2026-05-08] ingest | AI Coding Philosophy: Solved or Sharpened?
- Sources: coding-is-solved-boris-cherny.md, software-fundamentals-matt-pocock.md, vibe-coding-in-prod-eric.md
- Updated: Working with AI as a PM
- Updated: Leaf Nodes and Deep Modules
- Note: Wiki initialized. Three sources from ai-coding-thoughts/ ingested into raw/ai-coding/. Three articles compiled covering the philosophical debate, context-engineering methodology, and AI-friendly architecture.

## [2026-05-08] ingest | OpenClaw Plugin API: Extension Points and Deliberate Omissions
- Sources: openclaw-plugin-api.md
- Note: New topic `openclaw` created. Single source from openclaw-thoughts/ ingested into raw/openclaw/. Original ai-coding-thoughts/ and openclaw-thoughts/ directories removed.

## [2026-05-08] ingest | Agent 如何避免无限递归
- Sources: 飞书 Wiki "Agent 如何避免无限递归（研究 ROMA 框架的运行机制）"
- Note: New topic `ai-agents` created. Source ingested as PDF-derived markdown. Wiki article rewritten in 通俗易懂 style with concrete code/scenario examples per user request.

## [2026-05-08] ingest | Reactive vs. Proactive AI Agents：本质区别与选型
- Sources: Tekrevol blog (飞书剪存 PDF, 2025-09-08)
- Updated: Agent 如何避免无限递归 (added See Also cross-reference)
- Note: 原文为营销博客，内容浅；wiki 文章在保留核心分类骨架（Reactive/Proactive、五分类谱系）基础上加入批判性观点：纠正"Proactive 一定更高级"的暗含前提，把 LLM chatbot 不归 pure Reflex 的边界讲清楚，并把 Proactive 特有的死循环风险链到 avoiding-infinite-recursion。

## [2026-05-08] ingest | Memory OS：Agent 的记忆系统怎么设计
- Sources: Letta 官方文档与 MemGPT 论文整理稿（飞书剪存 PDF）
- Updated: Reactive vs. Proactive AI Agents (added See Also → memory-os, 措辞修正：从"必要基础设施"降级为"一种工程实现方案")
- Note: PDF 偏 Letta 框架手册风格（API、配置参数、工具列表）。wiki 文章重组为"概念→五种类型→设计原则→Agent 自主管理→Sleep-Time→Context Engineering→自家系统设计清单"的递进结构，强调 Memory OS 是"类比 OS 的设计理念"而非具体框架，并加入对项目当前实现状态的分析。
- 复盘：原本还加了 avoiding-infinite-recursion 的双向 See Also，老板质疑后撤回——陈旧 memory 更多导致"答错"而非"死循环"，因果链太弱属硬凑；reactive-vs-proactive 的链接保留但措辞从"必要基础设施"改为"一种工程实现方案"，避免夸大。

## [2026-05-08] ingest | Prompt Caching：原理、用法和坑
- Sources: apidog blog "What is Prompt Caching? Best Practices Explained"（飞书剪存 PDF, 2025-09-10）
- Note: 原文为产品博客式综合介绍（机制、Anthropic API 用法、价格、限制、最佳实践）。wiki 文章在保留核心知识骨架基础上做了两处增强：(1) 用具体数字算了"调 10 次省 78%、调 2 次只省 30%"的账，把"何时值得"量化；(2) 把"踩坑"独立成节，重点突出"多一个空格失效"、"并发首次集体 miss"、"动态工具列表破缓存"等 agent 框架最容易撞的边界。归到 ai-agents 是因为 agentic workflow 是这项技术受益最大的场景。

## [2026-05-08] ingest | Claude Code Harness 架构拆解：四层模型 + 生产级 agent 工程经验
- Sources: Rohit on X, "How I built harness for my agent using Claude Code leaks"（飞书剪存 PDF, 2026-04-13）
- Updated: Prompt Caching：原理、用法和坑（补充 See Also → claude-code-harness-architecture，作为"为缓存设计 prompt"原则的真实落地案例）
- Note: 原文是 Rohit 拆 Claude Code 源码（55 目录/331 模块）后整理的架构经验长文。wiki 文章按"四层模型 → loop 设计 → 工具/上下文 → 权限/重试 → sub-agent/infra → 扩展机制 → 抄什么清单"的工程视角重组，每一节都做"教程做法 vs Claude Code 做法"对比，并把代码块/类型签名保留以便回查。核心观点：业界三层模型不够用，layer 4 infrastructure（多租户/RBAC/隔离/持久化/分布式协调）是产品死掉的地方。落档到 ai-agents（agent 架构主题最贴）。

## [2026-05-08] ingest | Generative Benchmarking：用自己的 data 评估 retrieval 系统
- Sources: Kelly @ Chroma Research, YouTube 视频转录 PDF（飞书剪存）
- Note: 新建 `benchmark` topic。原文是 Chroma 研究员讲 generative benchmarking 的 talk transcript，结构线性（介绍 → MTEB 局限 → 三步流程 → notebook 演示 → representativeness 实验 → future）。wiki 文章按"thesis-first"重组：先抛 MTEB 的三个雷，再压缩三步 pipeline 为一张 ASCII 数据流图，把 representativeness 这块（最有价值）独立成节并提炼出三个反直觉发现（朴素生成会复刻 ground truth；越真实 recall 越低；排名换数据就翻）。补了"工程落地清单"和"局限与延伸"两节做批判性增强：点出本研究只覆盖 retrieval 而非 end-to-end RAG、Recall@K 简化只对单 doc 配对成立、生成 query 仍带 LLM bias、aligned judge 需要人工标注 anchor 这些边界。未做 See Also——现有库（ai-agents/ai-coding/openclaw）跟 RAG eval 关联弱，按 feedback memory 不硬凑。

## [2026-05-08] ingest | Parlant Guideline 系统：怎么设计，怎么约束住 LLM
- Sources: 用户上传 PDF "Parlant 中 Guideline 系统"（含 Parlant 源码引用 relationships.py、guideline_actionable_batch.py、relational_guideline_resolver.py、message_generator.py）
- Note: 原文偏教程/文档风格（关系类型定义 + 电商退货 Agent 端到端示例 + 6 个 Q&A）。wiki 文章按老板提问"怎么约束住 guideline"重组——开头先讲"朴素方案的三种崩法"（和稀泥/漏看/condition 误判）建立 motivation，再展开三阶段架构，然后用一张"五种 LLM 失败模式 vs 五个解法"对照表把"约束力"显式化，最后批判性地点出方案代价（关系图维护人工活、scale 上限、condition 表述质量决定准确率）和适用边界（仅 SOP 清晰场景）。See Also 链到 avoiding-infinite-recursion（控制流约束的另一面）和 reactive-vs-proactive（Parlant 是典型 reactive 设计）。未做反向 See Also 回填——关联弱到中等，按之前对 memory-os 的反思留单向。
