# Generative Benchmarking：用自己的 data 评估 retrieval 系统

> Sources: Kelly @ Chroma Research, 2026-05-08
> Raw: [Chroma Research - Generative Benchmarking](../../raw/benchmark/2026-05-08-chroma-generative-benchmarking.md)

## Overview

AI 系统的 eval 不能像 unit test 那样写 boolean 断言——"relevance"是主观、依赖 context 的。所以业界的做法是把 data 当成 test suite：准备一组 `(query, relevant doc)` 对，跑 retrieval 看 model 能不能命中。问题在于：通用 benchmark（如 MTEB）跟你的真实业务数据差异大，且 embedding model 多半在训练里见过这些 public dataset。Chroma 提出的 **generative benchmarking** 把整个 pipeline 翻转过来——用你自己的 corpus + LLM 生成 query，构造完全 domain-specific 的 eval set。

这份研究最值得记住的不是流程本身（流程很朴素：filter → generate → eval），而是它沿途揭示的几个反直觉发现：朴素地"让 LLM 看着 doc 出 query"会复刻 ground truth；越像真实用户的 query，模型 recall 反而越低；MTEB 排名换到你的数据上经常会翻。

## 为什么不能直接信 MTEB

MTEB 是 embedding model 的事实标准 leaderboard，但拿它当选型依据有三个雷：

1. **Domain mismatch**：MTEB 的 corpus 偏 Wikipedia、QA 学术集，跟 internal knowledge bot、客服 agent、产品文档机器人这些真实场景差距很大。
2. **Query 风格 mismatch**：MTEB 的 `(query, doc)` 是"为 QA 任务优化过的"，query 干净、答案明确。生产里用户问的话往往含糊、partial match，public dataset 体现不出这种 ambiguity。
3. **训练集污染**：embedding model 在 train 阶段大概率见过 MTEB 这些公开 dataset。leaderboard 上的高分有可能是 memorization 而不是 retrieval 能力。

所以 MTEB 排名第一的 model 在你的库上未必第一——后面 W&B 的实验会给出反例。

## 三步 pipeline

把 Chroma 的方法剥到最小骨架就是三步，每一步都依赖 LLM：

```
your docs ──[LLM judge]──▶ filtered docs ──[LLM gen]──▶ (query, doc) pairs ──▶ 跑 embedding ──▶ Recall@K
                ▲                              ▲
                │                              │
        user-provided context          context + example queries
```

**Step 1：filter docs**
不是所有 doc 都适合生 query——内容空、纯导航页、过期的要剔掉。把每份 doc 连同 user-provided context 喂给一个 "aligned LLM judge"，按 **relevance**（这 doc 跟用户场景相关吗）和 **completeness**（信息是否完整）两个维度打分。

> "Aligned" 听上去玄，本质是用一小批人工标注的 ground truth 反复调 judge prompt，让它的判断分布贴近人类。这是 LLM-as-a-judge 路线的常规做法，不是新东西。

**Step 2：generate queries**
对通过的每份 doc，把 doc + context + example queries 一起塞给 LLM 生 query。example queries 是关键——它锚定了 query 的风格（用户实际怎么问），不然 LLM 默认会生成"教科书式 QA query"，跟生产数据脱节。

**Step 3：跑 retrieval 算 recall**
有了 `(query, doc)` 对，用候选 embedding model 把 query embed，去 collection 里 retrieve top-K。本场景里每个 query 只对应 1 份 relevant doc，所以 **Recall@K** 退化为"target doc 落在 top K 的比例"，简单清楚。

## Representativeness：这套方法真的代表生产吗

整个 research 的核心张力其实在这里——synthetic data 早就有人做，但**生成的 query 到底有多像真实用户**？这块结论比方法本身更值钱。

### 发现 1：朴素生成会泄漏 ground truth

第一版 naive baseline：只把 doc 喂给 LLM，让它出一个 relevant query。结果在 MTEB 9 个 dataset 上，**LLM 生成的 query 跟 dataset 里的 ground truth query 完全相同或几乎相同**。

这意味着：
- LLM 训练时见过这些 public dataset，看到 doc 能直接复读出原 query。
- 用这种 query 去测 embedding model，等于让 model"做自己见过的题"——recall 会被严重高估。

修复办法很巧：把 doc **和** ground truth query 一起给 LLM，明确要求它生成一个**与之 distinct 的 query**。这样既能验证"新 query"的 representativeness，又避免了泄漏。

### 发现 2：越真实，recall 越低

在 W&B 的真实生产数据（documentation bot 的 docs + 真实用户提问）上：
- 排名一致性 ✅：naive 生成的 query 和 with-context-and-examples 生成的 query，给出的 embedding model 排名跟用 ground truth queries 评出来的排名一致。
- recall 数值 ❌：**ground truth queries 和"context + examples"生成的 queries 的 recall scores 反而比 naive 生成的更低**。

意思是：query 越贴近真实用户问法，embedding model 越打脸。naive 生成出来的 query 太像"标准答题卡"，让模型显得很强；一旦加入真实场景的含糊、口语、关键词错位，recall 就掉。

> 实践含义：如果你看自家系统的 retrieval recall 不错就以为没问题，要警惕你的 eval set 是不是"太干净了"。该比的不是绝对分数，而是用 representative query 的相对排名。

### 发现 3：MTEB 排名换数据就翻

具体反例：**Gina embeddings v3 在所有 English MTEB task 上都比 text-embedding-3-large 强**，但在 W&B 的数据上 **text-embedding-3-large 反而更好**。

这不是说 3-large 全方位更强，而是说**排名是 use-case 相关的**。任何 leaderboard，包括 MTEB，告诉你的最多是"在它那批数据上谁强"。落到你的 corpus 上，必须重新排。

## 工程落地清单

如果要在自家系统上跑一遍这套方法，最小集合是：

- [ ] 准备一段说明 bot 用途、用户画像的 **context** 文本（不是 system prompt，是给生成 LLM 看的"业务背景"）
- [ ] 收集 **3-10 条 example queries**（贴近真实风格，越口语越好）；如果没有真实日志，先模拟，但要让产品/运营审过
- [ ] 给 LLM judge 写一版 filter prompt（relevance + completeness），用 20-50 份 doc 人工标注校准
- [ ] 跑 query generation；每份 doc 至少出 1 个，可以出多个保证 diversity
- [ ] 用同一个 embedding model 同时 embed query 和 doc（**一定要同 model**，否则不可比）
- [ ] 跑 Recall@K（K=1, 5, 10），把不同候选 model 的曲线画出来
- [ ] **永远只看相对排名 + 相对差距**，不要把绝对 recall 当成 SLA

如果手上已经有真实 user query 日志，更应该把日志当锚——既能校准 generation prompt，也能直接 mine 出 corpus 里的 knowledge gap（某些 query 永远找不到 relevant doc → 缺内容）。

## 局限与延伸

这套方法不是银弹，几个边界要清楚：

- **只覆盖 retrieval，没回答 end-to-end RAG**。最终 answer 质量还跟 reranker、prompt、generation model 有关，retrieval recall 高不等于回答好。
- **每 query 对应 1 份 relevant doc 的简化**让 Recall@K 公式很漂亮，但真实场景一个 query 可能命中多份 doc，或者一份都没有。这套 metrics 就不够用了，需要 NDCG、MRR 之类。
- **生成 query 仍然带 LLM 风格 bias**。哪怕给了 examples，LLM 也很难复现真实用户的拼写错误、复合 intent、上下文省略。research 自己也承认 "production traffic 才是终极 ground truth"。
- **filter 阶段的 LLM judge alignment 是隐藏成本**。要做出靠谱的 judge，得有人工标注作为 anchor，初期投入不小。

研究自己列的未来方向其实就是这些缺口：(1) 用生产 traffic 反向迭代 generated query；(2) 把 corpus quality 作为独立研究对象；(3) 把 generative benchmarking 模板搬到 retrieval 之外的 AI 任务上。
