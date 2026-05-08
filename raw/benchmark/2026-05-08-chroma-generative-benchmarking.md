# Chroma Research - Generative Benchmarking

> Source: https://www.youtube.com/watch?v=YXWeZK5gcVE
> Collected: 2026-05-08
> Published: Unknown

大家好，我是 Kelly，目前在 Chroma 从事研究工作。最近我完成了 generative benchmarking 相关的研究，今天就来和大家分享一下。

## 什么是 benchmarking

首先，我们先明确"benchmarking"是什么。我们可以把它类比为 software engineering 中的 code tests。举个简单的例子，unit test 会检查多个数值的总和是否大于某个给定值。这类测试依赖的是 **deterministic logic**——给定一个 input，我们就能预期到一个特定的 output。这使得测试过程可靠，并且在相同条件下完全可复现。

但到了 AI systems 这里，情况就变得模糊了。我们可以以"retrieval"为例来理解，retrieval 是许多使用 RAG 技术的 LLM（Large Language Model）应用中的第一步。RAG 指的是 retrieval augmented generation：用户提出 query 后，系统会先 retrieve 相关 documents，再将这些 documents 输入 LLM，从而得到更贴合事实的 response。

假设你正在开发一个 customer support app，用户提出一个简单问题，比如"what is the return policy？"，你的 model 检索到了几份 documents。你能看出前两份 documents 是 relevant 的，但该如何对此进行测试呢？实际场景中，可能存在多份 relevant documents，也可能一份都没有；而且我们无法提前预知用户会问什么问题。你没法像写代码一样"hardcode"定义"relevance"，AI 的大多数应用场景都存在这个问题。所有这些 evaluations 都带有主观性，且依赖具体 context，无法通过简单的 tests 来实现。

因此，我们转而使用 data 来评估 AI systems：创建一个"benchmark"。在我们的场景中，这个 benchmark 就是一组"example queries paired with relevant documents"。比如，刚才提到的"what is the return policy？"这个 query，会对应一份内容类似"we accept returns within 30 days of purchase"的 document。对于每一组"query-document pair"，我们会测试该 query 能否 retrieve 到对应的 relevant document。遍历所有"query-document pairs"后，我们就能得到 recall、precision 等 metrics。

所以在 AI systems 中，data 本身就成了"test suite"。我们无法将 tests 简化为"boolean assertions"，而是需要衡量 system 在大量 real-world examples 中的表现。总而言之，benchmark 为我们提供了一种 standardized way，用于 evaluate 和 compare 不同 AI systems 的 performance，让我们能获得可推广到更广泛 scenarios 的 unified metrics。

## MTEB 的三个局限

具体到"retrieval"任务，目前 industry 内评估 embedding models 的 standard 是 MTEB（Massive Text Embedding Benchmark）。只要有新的 embedding model 发布，MTEB score 往往是衡量其 performance 的首要 indicator——这让 model selection 变得非常简单：如果 model A 在 MTEB task 上的表现优于 model B，那你就会选择 model A，因为 scores 直接表明了它更出色。

但遗憾的是，实际情况并没有这么简单。一个 model 在 MTEB 上表现优异，并不意味着它在你的 retrieval system 中（使用你的 data）也能有同样好的表现。原因有三点：

1. **benchmark datasets 的通用性**：这些 datasets 是 generic 的，无法全面覆盖实际应用中可能出现的各类 domains。比如，internal knowledge assistants、customer support agents、documentation bots 等真实应用，它们处理的 data 与 Wikipedia dataset 这类 data 差异很大。
2. **与 real production environments 的差异**：这些 public datasets 通常使用为"question answering tasks"设计的、经过优化的"query-document pairs"；但在 real production environments 中，用户的 queries 往往 vague，能匹配到的 documents 也可能只是 partially matched。真实 production data 中存在大量 ambiguity，而 public datasets 未必能体现这一点。
3. **model training data 的重叠**：大多数 embedding models 在 training 过程中可能已经接触过这些 public datasets。因此，我们很难判断 model 是真的擅长 retrieval，还是只是 memorized the task 本身。

## Generative benchmarking 的方法

为解决这些 limitations，我们提出了"generative benchmarking"——一种直接基于你的 data 生成的 benchmark 方法。我们利用你的 corpus of documents，结合 user provided context，通过 LLM 生成 queries。这种方法的优势在于：

- benchmark 与你的 domain 高度相关（因为使用的是你的 data）；
- 能处理 ambiguous queries（生成的 queries 贴合真实用户的提问方式）；
- 使用你自己的 data 会产生 embedding model"未见过"的"query-document pairs"，从而确保我们评估的是 model 真实的 retrieval 能力。

接下来，我将结合 CHS documentation，实际演示这种方法的操作流程：

**第一步，filter documents**：先筛选出 contextually relevant 且 informative 的 documents。比如，有些 documents 本身没什么有用信息，就需要被排除。筛选时，我们会将每一份 document 和 user provided context 输入一个"aligned LLM judge"。这里的"alignment"指的是，我们利用 human labeled ground truth 对 judge 进行了 iterated optimization。这种方法借鉴了"Eval"框架——该框架能让 LLM judges 与 human preferences 保持一致。要知道，未经 alignment 的 LLM judges，其 judge 方式未必与人类一致，因此在使用前必须先与 human judgment 对齐。

举个例子：我们先准备一段介绍"Chroma 是什么、how the spot is used"的 context，再将其与待筛选的 document 一起输入 LLM judge，完成 document 筛选。

**第二步，query generation**：筛选出合格 documents 后，我们开始生成 queries。将每一份 document、user provided context 以及 example queries 输入 LLM——context 和 example queries 能引导 query generation，使其更贴近 real user queries。

**第三步，构建"query-document pairs"并执行 retrieval**：将生成的 queries 与它对应的源 document 配对，然后用 embedding model 执行 retrieval task。由于我们有明确的"query-document pairs"，就能通过判断 model 是否成功 retrieve 到 target documents，来计算 recall 等 metrics。

这里解释一下 recall 的计算方式：recall = 检索到的 relevant documents 数量 ÷ relevant documents 的总数量。在我们的场景中，每个 query 对应 1 份 relevant document，因此"Recall@K"衡量的就是"relevant document 出现在 top K 个 retrieved results 中的频率"。

## Notebook 演示步骤

接下来，我会用这个 notebook 演示具体操作。这个 notebook 是 open source 的，大家也可以用自己的 data 来运行。

操作步骤如下：

1. 首先 install 并 import 所需的 libraries，然后设置 variables。加载 Chroma docs，定义 context 和 example queries（context 用于说明 this bot 的功能和 users might use it 的方式，example queries 则是 users might ask our technical support bot 的问题示例），同时定义 collection name。如果大家要用自己的 data，只需 uncomment 相关 code lines 并填写信息即可。
2. 加载 API keys，设置 clients，然后创建 Chroma collection（如果已有 collection，可直接跳到 2.3 步骤）。先定义 ids 和 documents，再使用"text-embedding-3-large"model 对 documents 进行 embedding 处理（embed.py 文件中还定义了其他 embedding functions，大家也可以自定义）。embedding 完成后，创建 Chroma collection 并将 embedding 后的 documents 添加进去。如果已有 Chroma collection，直接按对应方式 load data 即可。
3. **document quality filtering**：基于"relevance"和"completeness"两个 criteria 筛选 documents——"relevance"检查 document 是否与之前提供的 context 相关，"completeness"检查 document 的 overall quality。定义筛选 criteria 后，运行 function 筛选 documents，filter out 掉 judge 判定为 irrelevant 的 documents，之后就能查看筛选 results（包括 passed 和 failed 的 document examples）。
4. **生成"golden data set"**：用筛选后的 documents 生成由"query-document pairs"组成的"golden data set"。将 document、user provided context 和 example queries 输入 LLM，运行对应 code cell 即可生成。
5. **执行 evaluation**：设置 queries 及其 ids，使用与 document embedding 相同的 model 对 queries 进行 embedding 处理；然后定义一个 data frame 来存储"query-document matches"关系，用于 retrieval evaluation；最后运行 benchmark，得到"text-embedding-3-large"model 在 Chroma docs dataset 上的 results。
6. **结果保存与 model comparison**：建议 save results，以便在多个 embedding models 间进行比较。我已经用另外三个 models（OpenAI Small、Gina、Voyage）运行过测试，大家可以在另一个 notebook 中查看 model performance comparison：先 install 并 import libraries，load 各 models 的 results，创建包含所有 metrics 的 combined data frame，然后运行 code 生成 visualization 图表。从图表中可以看到，Voyage 3 Large 的表现优于其他 models，因此它可能是更优选择。

## Representativeness 研究

需要说明的是，"synthetic data set generation"本身并不是一项 novel technique——它的应用非常广泛，相关 research 也很多。但目前，关于"synthetic data 在多大程度上能 represent the ground truth"（在我们的场景中，ground truth 指 production environments 中的 real user queries）的 research 还比较有限。

我们可以简单地把所有 documents 输入 LLM，让它 generate queries；也可以为每份 document 生成 multiple queries 以保证 diversity。生成 queries 的 methods 有很多，但核心问题始终是"这些 methods 生成的 queries 能否贴合你的 actual use case"。因此，我们 research 的核心重点是"representativeness"——我们使用 production use case 中的 real data（以 Weights & Biases 的 documentation bot 为例），并利用这些 ground truth user queries，来验证我们 generated 的 queries 是否具有 representativeness。

我们的目标是生成一组最贴近 real user queries 的"generated queries"——只有这样，你才能确信自己正在 measuring 的是真正需要的 metrics。一个 query generation method 可能非常 complex 和 thorough，但如果它生成的 queries 与 users 实际提问不相符，那么用这个 benchmark 衡量的 results 就没有 practical value。

接下来，我将用一些 results 来 demonstrate 这种"representativeness"：

我们的 research 首先从 MTEB 的 public datasets 入手，主要目的是 validate 我们能否 generate"与给定 ground truth（即 original dataset 中的 queries）具有 representativeness"的 queries。我们先尝试了一种 naive query generation method：只向 LLM 输入 document，让它 generate a relevant query。结果发现，在我们使用的 9 个 datasets 上，LLM generated 的 queries 与 ground truth 完全 identical 或 near identical——这揭示了一个 important finding：LLMs 在 training 过程中可能已经接触过大量 public datasets，因此它们有概率直接 generate the ground truth query。如果使用这种 method，就可能出现"将完全相同的 queries 进行对比"的情况。

为了 demonstrate"new unseen generated queries 也具有 representativeness"，我们需要更 tailored 的 approach：这次我们同时向 LLM 输入 document 和 ground truth query，并要求它 generate a query distinct from the one provided。然后，我们用"ground truth queries"和"generated queries"分别执行 retrieval task，对比两者的 metrics。

以 English Wikipedia dataset 为例，我们发现两种 query types 下，embedding models 的 performance ranking 是 consistent 的。比如，无论是使用 ground truth queries 还是 generated queries，"text-embedding-3-large"model 的"Recall@1"都高于"Mini LM"model。

此外，我们还利用 Weights & Biases 的 real production data 验证了"representativeness"：我们使用其 technical documentation bot 的 documents 和 user queries（users 会问"how to use Weights & Biases""how to fix certain errors"等问题），一方面采用之前提到的"输入 context + example queries"的 generation method，另一方面采用"只输入 document"的 naive generation strategy。结果显示，两种 methods 生成的 queries 都能保持 embedding models 的 performance ranking 一致；同时，"ground truth queries"和"generated queries with context and examples"的 recall scores，反而比"naively generated queries"更低——这一现象很有意思，它说明：queries 越贴近 realistic 场景，models 的表现反而越差。

另外，我们的 results 还与 MTEB scores 存在矛盾：Gina embeddings v3 在所有 English MTEB tasks 上的表现都优于 text-embedding-3-large，但在我们的测试中，text-embedding-3-large 反而表现更好。这进一步 reinforces 我们的 core claim：model 在 MTEB 等 public benchmarks 上的 performance，并不总能 generalize 到 real-world performance。这并不是说 text-embedding-3-large 一定比 Gina model 更 superior，而是说明"performance rankings 会随 specific use case 的变化而变化"。

## Future Directions

基于这项 research，未来还有一些值得探索的 directions：

1. **incorporating production traffic 以优化 benchmark dataset 和 corpus**：我们的 generative benchmarking method 为"没有 log user queries"的场景提供了良好起点；但如果有 real user queries，就可以基于这些 queries iterate on generated queries，使其更 alignment。同时，production traffic 还能帮助 identifying knowledge gaps——如果某些 queries 在 corpus 中始终找不到 relevant documents，就可以据此 update corpus。未来，围绕"automatically detecting this"的 research 将非常有价值。
2. **提升 corpus quality**：进一步 research 如何提高 corpus 的 quality，将有助于提升 benchmark 的 accuracy。
3. **拓展 generative benchmarking 的应用 domains**：目前的 research 主要针对 retrieval task，未来可将这种 method 拓展到其他 AI domains。

## 总结

总结一下：今天我们介绍了 benchmarking 的重要性，以及我们提出的 generative benchmarking method 为何能帮助你"以更 realistic 的方式 evaluate 你的 retrieval system"。目前，我们已将所有 code 开源，大家可以在自己的 data 上使用——code repo 已发布在 GitHub 上，同时我们还发布了 technical report，对 research 内容进行了更深入的阐述（比今天分享的内容更详尽）。最后，Chroma 的 cloud service 现已 available，大家可以 sign up 使用，一分钟内就能快速上手。谢谢大家。
