# Parlant Guideline 系统：怎么设计，怎么约束住 LLM

> Sources: 用户上传 PDF "Parlant 中 Guideline 系统"（含 Parlant 源码引用 relationships.py、guideline_actionable_batch.py、relational_guideline_resolver.py、message_generator.py）, 2026-05-08
> Raw: [Parlant 中 Guideline 系统](../../raw/ai-agents/2026-05-08-parlant-guideline-system.md)

## 一句话总结

Parlant 的 guideline 系统不是"把规则塞进 prompt"，而是**把规则做成一张关系图**，运行时分三阶段：LLM 筛"哪些规则相关"→ 图算法处理冲突/连锁/依赖（**纯算法，不是 LLM**）→ LLM 用极短 prompt 生成 + 自检。

**约束力是多层叠加的，不是单点**：LLM 只在它擅长的环节（condition 判断、自然语言生成、自检）出场，关键决策点（哪些规则被压制 / 必须连带 / 前置依赖）全交给确定性算法兜底。这不是"prompt 写得严"，也不只是"减少 LLM 看到的规则数"——是把"靠 LLM 自觉跟随"的部分尽可能拆出来变成可测试、可审计的工程逻辑。

## 为什么不能直接把 guideline 塞 prompt

最朴素的做法：把全部 50 条 guideline 一股脑写进 system prompt，让 LLM 自己判断该听哪条、忽略哪条。

这种做法在 5 条以内还能用，到 20 条以上就开始崩，典型崩法有几种：

**崩法 1：和稀泥**
客户愤怒投诉质量问题，prompt 里同时有"提供折扣挽留""帮助退货""无条件退货""转接人工"四条。LLM 给出的回复是：

> "我们可以为您提供 10% 折扣，同时也可以帮您退货，**如果您需要**还可以转人工。请问您想怎么处理？"

把决策又踢回给客户。本来该果断"无条件退货 + 立即转人工"，结果四条全提一遍，自相矛盾。

**崩法 2：漏看**
prompt 里有 50 条规则，LLM 只跟随了其中 30 条，剩下 20 条像没看到一样。这不是模型不努力，是注意力机制在长 prompt 里就这表现。

**崩法 3：condition 误判**
guideline 长这样：`"When 客户第三次询问同一问题且之前未得到满意答复, Then 转接人工"`。这条要求 LLM **数次数 + 判断同义 + 评估满意度** 三件事一起做。50 条规则一起判，错一两个很正常。

Parlant 的设计就是冲着这三种崩法去的，思路是：**别让 LLM 同时承担"匹配规则"和"生成回复"两件事，也别让它判断"哪条规则该被压制"——这是图算法的活。**

## 三阶段架构

```
全部 Guidelines (50 条)
    ↓
阶段 1：Matching（LLM，分批并发）
    ↓
匹配的 (4 条)
    ↓
阶段 1.5：关系图过滤（纯算法，无 LLM）
    ↓
过滤后的 (2 条)
    ↓
阶段 2：Generation（LLM，单次）
    ↓
最终回复
```

每个阶段单独说。

### 阶段 1：Matching——只问"哪些规则相关"

任务被简化到极致：每条 guideline 拆成 `condition`（何时触发）+ `action`（触发后做什么），这一阶段只判断 condition，**不需要执行 action**。

LLM 看到的 prompt 大致这样：

```
对话历史：用户说："这个产品质量太差了！我要退货！"

请判断以下 5 条 guideline 的 condition 是否满足：
1) Condition: 客户想退货
2) Condition: 客户提到产品质量问题
3) Condition: 客户愤怒
4) Condition: 在退货流程中
5) Condition: 查看产品时

返回 JSON：每条给出 applies (true/false) 和 rationale。
```

**关键设计 1：分批并发**。50 条不一次问完，按 batch_size=5 分成 10 批，并发发起 10 个 LLM 请求。延迟 ≈ 单批延迟（~1.5s），不是 10×单批。

**关键设计 2：batch_size 越小越准**。文档给的实测准确率：

| batch_size | 准确率 |
|---|---|
| 1 | ~95% |
| 5 | ~90% |
| 10 | ~85% |

为什么？因为 LLM 在一次调用里要做的判断越多，越容易"差不多就行"。所以重要 guideline 可以单独用 batch_size=1 走精确通道。

**关键设计 3：rationale 必填**。LLM 不光给 true/false，还要解释"为什么 applies=true"——这一是给人审计用，二是约束 LLM 不能随便瞎给（要写理由就得真想想）。

### 阶段 1.5：关系图过滤——把"互斥"从 LLM 身上拿走

这是整个系统的"约束力"核心。

LLM 在阶段 1 给出"匹配列表" `[G2_折扣, G3_退货, G8_质量退货, G9_转人工]`。如果直接把这 4 条全送进生成阶段，就回到"和稀泥"崩法。

Parlant 的做法是：**用一张预先定义的关系图，确定性地剪掉冲突**。这一步**完全不用 LLM**，纯 NetworkX 图算法。

四种关系类型：

| 关系 | 语义 | 谁该用 |
|---|---|---|
| **PRIORITY** | A 和 B 同时匹配时，只激活 A | 解互斥（"愤怒 > 挽留"） |
| **ENTAILMENT** | A 激活时，B 也必须激活 | 流程链条（"退货 → 必问订单号"） |
| **DEPENDENCY** | A 仅在 B 也激活时才生效 | 限定子步骤范围（"问订单号"只在退货流程里有意义） |
| **DISAMBIGUATION** | A 激活且多个 B 同时匹配时，反问客户 | 处理歧义（"查询限额？ATM 还是信用卡？"） |

### 图算法具体怎么跑

四种关系不共用一张大图——Parlant 按关系类型**维护 4 张独立的 NetworkX 有向图**（`relationships.py`）：

```python
self._graphs: dict[RelationshipKind, networkx.DiGraph] = {
    RelationshipKind.PRIORITY: ...,
    RelationshipKind.ENTAILMENT: ...,
    RelationshipKind.DEPENDENCY: ...,
    RelationshipKind.DISAMBIGUATION: ...,
}
```

每张图都是有向图（`source → target`）。系统启动时从数据库批量加载关系记录，遍历建边：

```python
graph = networkx.DiGraph()
for rel in db.find(kind="priority"):
    graph.add_edge(rel.source.id, rel.target.id, id=rel.id)
```

为什么分图？因为四种关系的过滤算法走向不一样——有的要查"祖先"，有的要查"后代"，有的要查"邻居"。各自独立，不会互相干扰。

#### PRIORITY：反向 BFS 找所有祖先

**问题**：对当前 guideline `g`，谁优先于它？包括间接的（A 优先于 B，B 优先于 g → A 也间接优先于 g）。

**做法**：把图反向，从 `g` 出发跑 BFS，得到所有"上游"节点：

```python
graph_reversed = priority_graph.reverse()
ancestors = networkx.bfs_edges(graph_reversed, g.id)
# ancestors 就是所有直接 + 间接优先于 g 的节点
```

**过滤逻辑**（`relational_guideline_resolver.py:91-216`）：

```python
matched_ids = {g.id for g in matched_guidelines}
result = []
for g in matched_guidelines:
    higher = bfs_predecessors(priority_graph, g.id)
    # 只有当"压它的人"自己也在匹配列表里，压制才生效
    if any(h in matched_ids for h in higher):
        log_skip(g, by=h)
        continue
    result.append(g)
```

**关键的一笔**：判断条件是"压它的人**也在匹配列表**"，不是"图里存在压制关系"。如果 G9 没被匹配（客户不愤怒），那 G9 → G2 这条边今天就不生效，G2 该上还是上。**这是 contextual prioritization，不是全局静态优先级**。

跑刚才的例子：

```
G2 → 祖先有 {G8, G9}，且 G8、G9 都在匹配列表 → 删
G3 → 祖先有 {G9}，G9 在匹配列表 → 删
G8 → 祖先空 → 保留
G9 → 祖先空 → 保留
```

#### ENTAILMENT：正向 BFS 把后代加进结果

PRIORITY 是"减"，ENTAILMENT 是"加"。

**问题**：当前 `g` 在结果里，按业务规则应该被 `g` 强制带出来的下游 guideline 还没在结果里——补上。

**做法**：从 `g` 正向走 BFS，把所有后代加进结果集：

```python
for g in result:
    descendants = networkx.bfs_edges(entailment_graph, g.id)
    result.extend(descendants)
```

举个例子：如果 `G3_退货` 进了结果集，按 `G3 --ENTAIL--> G4_问订单号 --ENTAIL--> G6_验证退货期`，G4 和 G6 都被自动加进来——哪怕阶段 1 的 LLM 漏了它们。这就堵上了"LLM 漏看流程子步骤"的口。

#### DEPENDENCY：检查"我依赖的是不是激活了"

**问题**：`A --DEPEND_ON--> B` 表示"A 仅在 B 也激活时才有意义"。如果 B 不在匹配列表，A 就该被过滤——哪怕 A 自己的 condition 看起来满足了。

**做法**：对每个 guideline，查它的 dependency 出边，看目标是否激活：

```python
for g in matched_guidelines:
    deps = list(dependency_graph.successors(g.id))
    if deps and not any(d in matched_ids for d in deps):
        skip(g)  # 依赖的前置没激活，过滤掉
```

举例：`G4_问订单号 --DEPEND--> G3_退货`。如果用户随口问了句"我的订单号是多少"，G4 的 condition（"在退货流程中"）可能被 LLM 误判为 true，但 G3（"客户坚持退货"）没匹配——DEPENDENCY 这一步会把 G4 砍掉，避免在错误上下文里执行子步骤。

**这是给阶段 1 的 LLM 兜底**——LLM 误判 condition 时，DEPENDENCY 限定了子步骤的"激活范围"。

#### DISAMBIGUATION：检测多 target 同时命中，触发反问

**问题**：客户说"我想查询限额"，同时匹配了 `G_ATM限额` 和 `G_信用卡限额`两条 guideline，LLM 在生成阶段会瞎猜其中一个或都讲一遍。

**做法**：DISAMBIGUATION 关系结构是 `source --DISAMBIGUATE--> [target_1, target_2, ...]`。算法检测：

```python
for source in matched_guidelines:
    targets = list(disambiguation_graph.successors(source.id))
    matched_targets = [t for t in targets if t in matched_ids]
    if len(matched_targets) >= 2:
        # 触发反问，不进入正常生成路径
        emit_disambiguation_question(source, matched_targets)
```

注意这条**不是过滤，是路径切换**——直接把流程从"生成回复"改成"反问客户"，让用户自己选。把"乱猜"风险显式上交给用户，比 LLM 蒙错强。

#### 算法跑完输出什么

四步过完，原本 `[G2, G3, G8, G9]` 变成 `[G8, G9]`。日志能清清楚楚追溯：

```
[INFO] Skipped: G2_折扣 deactivated due to priority by G8_质量退货
[INFO] Skipped: G2_折扣 deactivated due to priority by G9_转人工
[INFO] Skipped: G3_退货 deactivated due to priority by G9_转人工
```

**为什么这一步要做成纯算法**？因为"压制关系"是业务 SOP，不是模型推理。让 LLM 推断"愤怒 > 挽留"等于把审计责任推给一个不可解释的黑盒。写成图 + 算法执行，这事就变成可测试、可改、可查日志的工程问题。出 bug 时不用调 prompt，改图就行。

### 阶段 2：Generation——只把过滤后的几条送进 prompt

prompt 里只有 2 条 guideline 和对话历史，LLM 没机会"漏看"或"和稀泥"：

```
相关 Guidelines:
1. When 客户提到产品质量问题, Then 无条件接受退货并致歉
2. When 客户非常愤怒, Then 立即转接人工客服

请生成对客户的回复。
```

输出果断、无矛盾：

> "非常抱歉给您带来糟糕的体验！对于产品质量问题，我们会无条件为您办理退货。同时，我将立即为您转接客服经理。"

加上 **Revision 自检**——生成完让 LLM 再看一遍自己写的，对照每条 guideline 打 `instructions_followed` / `instructions_broken` 标签，如果 `further_revisions_required=true` 就重写。这是兜底机制，防止 LLM 在生成阶段还是漏了某条。

## 怎么"约束住" guideline：五种失败模式 vs 五个解法

老板问的核心是"怎么约束住"。我把 LLM 不听话的失败模式列一遍，对照 Parlant 给出的解法：

| 失败模式 | 朴素方案的问题 | Parlant 的解法 |
|---|---|---|
| **LLM 漏看某条规则** | 50 条塞 prompt，注意力稀释 | 阶段 1 + 1.5 把进生成 prompt 的 guideline 减到 2-3 条 |
| **LLM 同时执行互斥规则**（折扣+退货+转人工同时给） | 让 LLM 自己判断哪条优先，结果"和稀泥" | 关系图 PRIORITY，纯算法剪枝，LLM 看不到被压制的规则 |
| **LLM 错判 condition**（"客户提到质量问题"漏掉） | 50 条一起问，准确率掉到 85% | 分批并发（batch_size 越小越准）+ retry + temperature 调整 + few-shot |
| **LLM 自己脑补不存在的连锁规则** | 让 LLM 推断 guideline 之间的关系 | 关系全人工定义，存数据库，可审计 |
| **LLM 在生成阶段又漏了** | 没有兜底 | Revision 自检循环，对照 guideline 打标签 |

注意这五个解法**不是任意一个就够**——它们是组合起来才有效的链条。比如只做"减少 prompt 中数量"但不做关系图过滤，就还是会出现"客户既愤怒又投诉质量"时同时给折扣的情况，因为 LLM 会觉得"折扣"和"退货"都相关。

## 几个看似奇怪但其实对的设计选择

**不用向量库做匹配**。向量检索只能做"语义相似度"，但 condition 经常需要推理（"第三次询问同一问题且未得到满意答复"——这要数次数 + 判同义 + 评满意度，向量做不了）。所以匹配阶段必须用 LLM。

**关系手动定义，不让 LLM 自己学**。这看起来"不够 AI"，但**关系图就是业务 SOP 的可视化**——业务方可以审，合规可以查，出问题能改。让 LLM 自动推断关系等于把 SOP 交给一个不可解释的黑盒，企业级场景下没人敢用。

**condition / action 分离**。这是阶段 1 能高效跑的前提。匹配只看 condition（true/false 二分类，简单），生成只用 action（自然语言输出，灵活）。如果两个揉在一起，LLM 既要判断该不该执行又要决定怎么执行，准确率会更糟。

**DISAMBIGUATION 不是给 LLM 偷懒，是给它兜底**。当系统检测到客户输入有歧义（"查询限额"匹配到多条 guideline），不让 LLM 猜，强制反问客户。这是把"乱猜"风险显式上交给用户，比 LLM 蒙错强。

## 这套方案的代价

不是没成本：

**1. 关系图维护是人工活，不 scale 到无穷大**。文档给的例子是 50 条规则，500 条还能撑，5000 条就开始痛。Parlant 给的优化（自适应 batch_size、缓存、分组）都是缓解，不是根治。

**2. 适用场景有限**。这套方案吃业务 SOP 清晰、动作可枚举的场景——客服、订票、表单流程、合规审核。开放型场景（创作、咨询、闲聊）写不出有意义的 condition/action，关系图也没法画。

**3. 两次 LLM 调用 = 两份成本**。文档说总延迟 < 3 秒（matching 并发 ~1.5s + generation ~1s），token 消耗反而比"全塞 prompt"低 25%（因为 generation 阶段 prompt 很短）。但调用次数翻倍，每次都要排队、都要算速率限制。

**4. condition 表述质量直接决定准确率**。`"客户愤怒"` 和 `"客户使用感叹号 ≥3 次或包含'垃圾/差/退钱'等词"` 两种写法，LLM 匹配准确率差很多。这等于把"prompt engineering"压力转移到了"guideline writing"上。

## 怎么往自家系统上套

如果手里也有一个需要约束 LLM 行为的 agent，可以照这套思路改：

1. **把规则拆成 condition + action**。不要写"如果客户愤怒就转人工"，写两半："Condition: 客户愤怒"；"Action: 转人工"。这一步把"判断"和"执行"分开，单独可优化。

2. **画一张冲突表**。手动列出"这两条规则同时匹配时该听谁"，这就是 PRIORITY 关系。先列出 10-20 对最关键的，剩下的让默认行为兜底。

3. **匹配阶段单独跑一次 LLM 调用**。不要直接把规则塞主 prompt。哪怕只是 prompt 上加一段 "请先判断这些规则哪些相关，再生成回复"——这种半截方案也比全塞强，但不如真正分两次调用。

4. **生成阶段加 self-review**。让 LLM 生成完后再调一次，问"你刚才有没有违反这些规则"。简单粗暴但有效。

5. **写日志**。每一步过滤决策都记下来（哪条被谁压制、为什么），出问题能查。这是企业级跑得通的前提。

## See Also

- [Agent 如何避免无限递归](avoiding-infinite-recursion.md) —— 另一种"约束 agent"的角度：从控制流上设上限和终止条件，与本文从"内容选择"上做约束互补
- [Reactive vs. Proactive AI Agents：本质区别与选型](reactive-vs-proactive-ai-agents.md) —— Parlant 的 guideline 系统本质是 reactive 路径（condition 触发 action），适合 SOP 清晰的客服场景
