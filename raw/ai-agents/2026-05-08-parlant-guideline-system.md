# Parlant 中 Guideline 系统

> Source: 用户上传 PDF（attachment, 1778229468870-3ce7f2cd-d36d-4188-88ed-deb52942c33d.pdf）
> Collected: 2026-05-08
> Published: Unknown

本文档详细解释 Parlant 中 Guideline 系统的两个核心问题：

1. **Guidelines 的关系图是如何构建的**
2. **Guidelines 是如何被 Agent 实际使用的**

以一个**电商退货客服 Agent**作为完整示例贯穿全文。

---

## 第一部分：Guideline 关系图的构建

### 核心原理

**关键事实**：

- ❌ **不使用**向量数据库
- ❌ **不使用** LLM 自动推断关系
- ✅ **使用**手动定义的关系 + NetworkX 图算法

关系图是一个**确定性的、人工定义的规则系统**，通过图数据结构高效查询。

### 关系类型

Parlant 支持 4 种核心关系类型（定义在 `relationships.py:43-63`）：

#### 1. PRIORITY (优先级)

**语义**：当 SOURCE 和 TARGET 同时匹配时，只激活 SOURCE。

```python
await high_priority_guideline.prioritize_over(low_priority_guideline)
```

**用例**：
- "转接人工客服" 优先于 "推销产品"
- "处理客户投诉" 优先于 "营销推荐"

#### 2. ENTAILMENT (蕴含)

**语义**：当 SOURCE 激活时，TARGET 也必须自动激活。

```python
await source_guideline.entail(target_guideline)
```

**用例**：
- "When 客户下单, Then 确认订单详情" 蕴含 "When 确认订单详情, Then 告知送达时间"
- 形成完整的业务流程链

#### 3. DEPENDENCY (依赖)

**语义**：SOURCE 仅在 TARGET 也激活时才生效。

```python
await specific_guideline.depend_on(baseline_guideline)
```

**用例**：
- "询问订单号" 依赖于 "处理退货流程"（只有在退货流程中才询问订单号）
- 确保子步骤在正确的上下文中执行

#### 4. DISAMBIGUATION (消歧)

**语义**：当 SOURCE 激活且多个 TARGETS 同时激活时，询问客户选择。

```python
await ambiguous_observation.disambiguate([option_1, option_2, ...])
```

**用例**：
- 客户说"查询限额" → 询问是"ATM 限额"还是"信用卡限额"

---

### 存储与图结构

#### 数据库存储

关系存储在文档数据库中（如 MongoDB）：

```json
{
  "id": "rel_001",
  "source": "G9_handoff_angry",
  "source_type": "guideline",
  "target": "G2_offer_discount",
  "target_type": "guideline",
  "kind": "priority",
  "creation_utc": "2025-01-01T00:00:00Z"
}
```

#### 内存图结构

系统使用 **NetworkX** 构建内存中的有向图（`relationships.py:178, 278-308`）：

```python
import networkx

# 按关系类型维护独立的图
self._graphs: dict[RelationshipKind, networkx.DiGraph] = {}

# 启动时从数据库加载
graph = networkx.DiGraph()
for relationship in database.find(kind="priority"):
    graph.add_edge(
        relationship.source.id,
        relationship.target.id,
        id=relationship.id
    )
```

**查询优化**：使用 BFS（广度优先搜索）查找间接关系。

```python
# 查询 "谁优先于 guideline_X？"
graph_reversed = graph.reverse()  # 反转图
priority_edges = networkx.bfs_edges(graph_reversed, guideline_X)
# 返回所有直接和间接优先的 guidelines
```

---

### 电商示例：定义 Guidelines 和关系

#### 场景设定

电商退货客服 Agent，需要处理：
- 普通退货请求
- 产品质量问题
- 客户愤怒情绪
- 退货流程管理

#### 步骤 1：创建 Guidelines

```python
# ========== 基础服务 Guidelines ==========

G1_greet = await agent.create_guideline(
    condition="对话开始时",
    action="礼貌问候客户，询问如何帮助"
)

G2_offer_discount = await agent.create_guideline(
    condition="客户想退货",
    action="提供 10% 折扣券挽留客户，避免退货"
)

G3_process_return = await agent.create_guideline(
    condition="客户坚持要退货",
    action="帮助客户完成退货流程"
)

# ========== 退货流程步骤 ==========

G4_ask_order_number = await agent.create_guideline(
    condition="在退货流程中",
    action="询问客户的订单号"
)

G5_load_last_order = await agent.create_guideline(
    condition="客户无法提供订单号",
    action="加载客户最近的订单，询问是否是这个订单"
)

G6_verify_return_window = await agent.create_guideline(
    condition="获得订单信息后",
    action="检查是否在 30 天退货期内"
)

G7_reject_expired = await agent.create_guideline(
    condition="订单超过 30 天退货期",
    action="礼貌告知无法退货，并解释原因"
)

# ========== 特殊情况处理 ==========

G8_quality_issue = await agent.create_guideline(
    condition="客户提到产品质量问题",
    action="无条件接受退货，并致歉"
)

G9_handoff_angry = await agent.create_guideline(
    condition="客户非常愤怒或使用不当言辞",
    action="立即转接人工客服"
)

# ========== 营销相关 ==========

G10_upsell_accessories = await agent.create_guideline(
    condition="客户查看产品详情时",
    action="推荐相关配件"
)
```

#### 步骤 2：建立 PRIORITY 关系

```python
# 客户愤怒时，转接人工优先于所有其他操作
await G9_handoff_angry.prioritize_over(G2_offer_discount)
await G9_handoff_angry.prioritize_over(G3_process_return)
await G9_handoff_angry.prioritize_over(G10_upsell_accessories)

# 质量问题时，直接退货优先于挽留折扣
await G8_quality_issue.prioritize_over(G2_offer_discount)

# 处理退货优先于追加销售
await G3_process_return.prioritize_over(G10_upsell_accessories)
```

**优先级图可视化**：

```
              G9_handoff_angry  (最高优先级)
                    ↓ 优先于
        ┌───────────┼───────────┐
        ↓           ↓           ↓
   G2_offer    G3_process   G10_upsell
   _discount   _return      _accessories
        ↑           ↑
        |           |
   G8_quality      |
   _issue          |
        ↑          |
        └──────────┘
```

#### 步骤 3：建立 ENTAILMENT 关系

```python
# 处理退货时，必须询问订单号
await G3_process_return.entail(G4_ask_order_number)

# 获得订单后，必须验证退货期
await G4_ask_order_number.entail(G6_verify_return_window)
await G5_load_last_order.entail(G6_verify_return_window)
```

**蕴含链可视化**：

```
G3_process_return (客户坚持退货)
    ↓ ENTAIL
G4_ask_order_number (询问订单号)
    ↓ ENTAIL
G6_verify_return_window (验证退货期)
    ↓ 可能触发
G7_reject_expired (拒绝过期退货)

替代路径：
G5_load_last_order (加载最近订单)
    ↓ ENTAIL
G6_verify_return_window
```

#### 步骤 4：建立 DEPENDENCY 关系

```python
# 只有在"处理退货流程"激活时，这些子步骤才有意义
await G4_ask_order_number.depend_on(G3_process_return)
await G5_load_last_order.depend_on(G3_process_return)
await G6_verify_return_window.depend_on(G3_process_return)
await G7_reject_expired.depend_on(G3_process_return)
```

**依赖图可视化**：

```
        G3_process_return  (基准线 - Baseline)
              ↑ 被依赖
    ┌─────────┼──────────┬──────────┐
    |         |          |          |
G4_ask    G5_load    G6_verify  G7_reject
_order    _last      _window    _expired
_number   _order

如果 G3 未激活，则 G4, G5, G6, G7 全部被过滤掉
```

#### 步骤 5：完整关系图总览

```
              G9_handoff_angry
              (客户愤怒 → 转接人工)
                    ↓ PRIORITY
        ┌───────────┼───────────┐
        ↓           ↓           ↓
   G2_offer    G3_process   G10_upsell
   _discount   _return      _accessories
   (挽留折扣)  (处理退货)   (推荐配件)
        ↑           ↑           ↑
        |           |           |
   PRIORITY    PRIORITY    PRIORITY
        |           |           |
   G8_quality      |           |
   _issue          └───────────┘
   (质量问题)

退货流程依赖链 (DEPENDENCY + ENTAILMENT):

    G3_process_return (基准)
        ↓ ENTAIL
    G4_ask_order_number ←── DEPEND ──┐
        ↓ ENTAIL                      |
    G6_verify_return_window ←── DEPEND ─┤
        ↓ 条件触发                     |
    G7_reject_expired ←──────── DEPEND ┘

    G5_load_last_order ←─── DEPEND ───→ G3_process_return
        ↓ ENTAIL
    G6_verify_return_window
```

---

## 第二部分：Guideline 的运行时使用

### 两阶段架构

**核心问题**：如何在不把所有 Guidelines 都放入生成 Prompt 的情况下，判断哪些 Guidelines 相关？

**解决方案**：使用**两次独立的 LLM 调用**，每次有不同的 Prompt。

```
全部 Guidelines (50 条)
    ↓
┌────────────────────────────────┐
│ 阶段 1: Guideline Matching     │
│ (独立 Prompt，分批并发)         │
│ 任务：判断哪些 Guidelines 匹配   │
└────────────────────────────────┘
    ↓
匹配的 Guidelines (10 条)
    ↓
┌────────────────────────────────┐
│ 阶段 1.5: 关系图过滤            │
│ (纯图算法，无 LLM)              │
│ 任务：应用优先级、依赖关系       │
└────────────────────────────────┘
    ↓
过滤后的 Guidelines (3 条)
    ↓
┌────────────────────────────────┐
│ 阶段 2: Message Generation     │
│ (独立 Prompt，单次 LLM 调用)    │
│ 任务：生成最终回复               │
└────────────────────────────────┘
    ↓
最终消息
```

### 阶段 1: Guideline Matching

#### 目的

判断当前对话状态下，哪些 Guidelines 的 **condition** 被满足。

#### Matching Prompt 结构

位置：`guideline_actionable_batch.py:196-300`

```
┌────────────────────────────────────────┐
│  GUIDELINE MATCHING PROMPT             │
├────────────────────────────────────────┤
│                                        │
│  ## 任务说明                            │
│  你的任务是评估哪些 guideline 的 condition  │
│  适用于当前对话状态。                     │
│                                        │
│  每个 guideline 包含两部分：               │
│  - condition: 何时应该激活                │
│  - action: 激活后做什么（不需要现在执行）   │
│                                        │
│  ## Agent Identity                     │
│  你是一个名为"退货助手"的电商客服 AI         │
│                                        │
│  ## Context Variables                  │
│  - customer_tier: "VIP"                │
│  - previous_orders: 15                 │
│                                        │
│  ## Interaction History                │
│  [完整对话历史]                          │
│                                        │
│  ## Guidelines 列表 ← 关键！             │
│  1) Condition: 客户想退货                │
│     Action: 提供折扣挽留                 │
│                                        │
│  2) Condition: 客户提到产品质量问题       │
│     Action: 无条件退货                   │
│                                        │
│  3) Condition: 客户愤怒                  │
│     Action: 转接人工                     │
│                                        │
│  ... (本批次的所有 guidelines)            │
│                                        │
│  ## 输出格式                             │
│  返回 JSON:                              │
│  {                                     │
│    "checks": [                         │
│      {                                 │
│        "guideline_id": "1",            │
│        "condition": "客户想退货",        │
│        "rationale": "客户明确说了'我要退货'", │
│        "applies": true                 │
│      },                                │
│      ...                               │
│    ]                                   │
│  }                                     │
└────────────────────────────────────────┘
```

#### 分批策略

**问题**：如果有 100 条 Guidelines，一次性放入 Prompt 会导致 Token 超限。

**解决方案**：动态分批 + 并发处理（`guideline_actionable_batch.py:378-388`）

```python
def _get_optimal_batch_size(guidelines_count):
    if guidelines_count <= 10:
        return 1   # 每批 1 条 (精确判断)
    elif guidelines_count <= 20:
        return 2   # 每批 2 条
    elif guidelines_count <= 30:
        return 3   # 每批 3 条
    else:
        return 5   # 每批 5 条
```

**示例**：50 条 Guidelines → 10 批（每批 5 条）→ 并发发起 10 个 LLM 请求

```
Batch 1 (G1-G5)   ┐
Batch 2 (G6-G10)  |
Batch 3 (G11-G15) |
Batch 4 (G16-G20) ├── 并发调用 LLM
Batch 5 (G21-G25) |
...               |
Batch 10 (G46-G50)┘

汇总结果 → Matched Guidelines
```

#### 代码流程

位置：`guideline_actionable_batch.py:84-140`

**第 1 步：构建包含本批 guidelines 的 prompt**

```python
guidelines_text = "\n".join(
    f"{i}) Condition: {g.condition}. Action: {g.action}"
    for i, g in batch_guidelines.items()
)

prompt = PromptBuilder()
prompt.add_interaction_history(context.interaction_history)
prompt.add_section(
    name="guidelines",
    template="Guidelines List: ###\n{guidelines_text}\n###",
    props={"guidelines_text": guidelines_text}
)
```

**第 2 步：调用 LLM 生成匹配判断**

```python
inference = await schematic_generator.generate(
    prompt=prompt,
    hints={"temperature": 0.7}
)
```

**第 3 步：解析 LLM 返回的结果**

```python
matches = []
for check in inference.content.checks:
    if check.applies:  # LLM 判断此 guideline 匹配
        matches.append(GuidelineMatch(
            guideline=batch_guidelines[check.guideline_id],
            score=10,
            rationale=check.rationale
        ))

return matches
```

#### 实际示例

**用户消息**："这个产品质量太差了！我要退货！"

**Batch 1 Prompt**：

```
Guidelines:
1) Condition: 对话开始时. Action: 问候
2) Condition: 客户想退货. Action: 提供折扣挽留
3) Condition: 客户坚持退货. Action: 处理退货流程
4) Condition: 在退货流程中. Action: 询问订单号
5) Condition: 客户无法提供订单号. Action: 加载最近订单

Interaction History:
- User: "这个产品质量太差了！我要退货！"

请判断每条 guideline 的 condition 是否适用。
```

**LLM 返回**：

```json
{
  "checks": [
    { "guideline_id": "1", "applies": false, "rationale": "对话刚开始，但客户已明确需求" },
    { "guideline_id": "2", "applies": true, "rationale": "客户明确说'我要退货'" },
    { "guideline_id": "3", "applies": true, "rationale": "客户语气坚决" },
    { "guideline_id": "4", "applies": false, "rationale": "还未进入退货流程" },
    { "guideline_id": "5", "applies": false, "rationale": "未询问订单号" }
  ]
}
```

**Batch 2 Prompt**：

```
Guidelines:
6) Condition: 获得订单后. Action: 验证退货期
7) Condition: 超过退货期. Action: 拒绝退货
8) Condition: 客户提到产品质量问题. Action: 无条件退货
9) Condition: 客户愤怒. Action: 转接人工
10) Condition: 查看产品时. Action: 推荐配件

Interaction History:
- User: "这个产品质量太差了！我要退货！"
```

**LLM 返回**：

```json
{
  "checks": [
    { "guideline_id": "6", "applies": false },
    { "guideline_id": "7", "applies": false },
    { "guideline_id": "8", "applies": true, "rationale": "客户明确提到'质量太差'" },
    { "guideline_id": "9", "applies": true, "rationale": "使用了感叹号和强烈措辞" },
    { "guideline_id": "10", "applies": false }
  ]
}
```

**所有 Batch 汇总结果**：

```python
matched_guidelines = [
    G2_offer_discount,   # 客户想退货
    G3_process_return,   # 客户坚持退货
    G8_quality_issue,    # 客户提到质量问题
    G9_handoff_angry     # 客户愤怒
]
```

---

### 阶段 1.5: 关系图过滤

#### 目的

应用预先定义的关系图，过滤冲突的 Guidelines。

**关键**：这个阶段**不使用 LLM**，纯粹是图算法。

#### 过滤逻辑

位置：`relational_guideline_resolver.py:91-216`

```python
# 输入：匹配的 Guidelines
matched_guidelines = [G2, G3, G8, G9]
matched_ids = {g.id for g in matched_guidelines}

result = []

for guideline in matched_guidelines:
    # 查询所有优先于当前 guideline 的关系 (使用 BFS)
    priority_relationships = relationship_store.list_relationships(
        kind=RelationshipKind.PRIORITY,
        indirect=True,  # 包括间接关系
        target_id=guideline.id
    )

    # 检查是否有更高优先级的 guideline 也在匹配列表中
    deprioritized = False
    for rel in priority_relationships:
        if rel.source.id in matched_ids:
            # 找到了优先级更高的 guideline
            deprioritized = True
            logger.info(
                f"Skipped: {guideline.id} deactivated due to "
                f"priority by {rel.source.id}"
            )
            break

    if not deprioritized:
        result.append(guideline)

# 应用 ENTAILMENT 关系，添加隐含的 guidelines
for guideline in result:
    entailed = relationship_store.list_relationships(
        kind=RelationshipKind.ENTAILMENT,
        source_id=guideline.id
    )
    result.extend(entailed)

return result
```

#### 实际示例

**输入**：

```python
matched_guidelines = [G2, G3, G8, G9]
```

**步骤 1: 检查 G2 (提供折扣挽留)**

```
查询谁优先于 G2?
priority_graph.query(target=G2)

图遍历结果：
G8 --PRIORITY--> G2
G9 --PRIORITY--> G2

检查 G8 和 G9 是否在匹配列表中
G8 in matched_guidelines? ✓ Yes
G9 in matched_guidelines? ✓ Yes

结论：G2 被 G8 和 G9 压制，移除
```

**步骤 2: 检查 G3 (处理退货流程)**

```
查询谁优先于 G3?
G9 --PRIORITY--> G3

检查 G9 是否在匹配列表中
G9 in matched_guidelines? ✓ Yes

结论：G3 被 G9 压制，移除
```

**步骤 3: 检查 G8 (质量问题无条件退货)**

```
查询谁优先于 G8?
priority_graph.query(target=G8)

结果：无

结论：G8 保留
```

**步骤 4: 检查 G9 (愤怒客户转接人工)**

```
查询谁优先于 G9?
结果：无

结论：G9 保留
```

**过滤后结果**：

```python
filtered_guidelines = [G8, G9]
```

**日志输出**：

```
[INFO] Skipped: G2_offer_discount deactivated due to contextual prioritization by G8_quality_issue
[INFO] Skipped: G2_offer_discount deactivated due to contextual prioritization by G9_handoff_angry
[INFO] Skipped: G3_process_return deactivated due to contextual prioritization by G9_handoff_angry
```

---

### 阶段 2: Message Generation

#### 目的

使用过滤后的 Guidelines 生成最终回复。

#### Generation Prompt 结构

位置：`message_generator.py:300-500`

```
┌────────────────────────────────────────┐
│  MESSAGE GENERATION PROMPT             │
├────────────────────────────────────────┤
│                                        │
│  ## 任务说明                            │
│  生成对客户的回复消息                     │
│                                        │
│  ## Agent Identity                     │
│  你是一个名为"退货助手"的电商客服 AI         │
│                                        │
│  ## Interaction History                │
│  - User: "这个产品质量太差了！我要退货！"  │
│                                        │
│  ## 相关 Guidelines ← 只有过滤后的!       │
│                                        │
│  Guideline #1:                         │
│  - When: 客户提到产品质量问题             │
│  - Then: 无条件接受退货，并致歉            │
│                                        │
│  Guideline #2:                         │
│  - When: 客户非常愤怒                    │
│  - Then: 立即转接人工客服                 │
│                                        │
│  ## 重要原则                            │
│  - 遵循所有提供的 Guidelines              │
│  - 如果 Guidelines 冲突，优先客户安全      │
│  - 不要编造信息                          │
│                                        │
│  ## 输出格式                             │
│  生成自然的回复消息                       │
└────────────────────────────────────────┘
```

**关键区别**：这个 Prompt 只包含 2 条 Guidelines，而不是全部 50 条！

#### 实际示例

**输入**：

```python
filtered_guidelines = [G8_quality_issue, G9_handoff_angry]
interaction_history = [
    Event(source=CUSTOMER, message="这个产品质量太差了！我要退货！")
]
```

**Generation Prompt**：

```
你是一个电商客服 AI。

对话历史：
- User: "这个产品质量太差了！我要退货！"

相关 Guidelines:
1. When 客户提到产品质量问题, Then 无条件接受退货，并致歉
2. When 客户非常愤怒, Then 立即转接人工客服

请生成对客户的回复。
```

**LLM 生成**：

```
非常抱歉给您带来了糟糕的体验！对于产品质量问题，我们会无条件为您办理退货。
同时，我将立即为您转接我们的客服经理，确保您的问题得到妥善处理。
```

#### Revision 系统

LLM 生成后会进行自我评估（详见 `guideline-conflict-resolution.md`）：

```json
{
  "revision_number": 1,
  "content": "非常抱歉给您带来了糟糕的体验！...",
  "instructions_followed": [
    "Guideline #1: 无条件接受退货并致歉",
    "Guideline #2: 转接人工客服"
  ],
  "instructions_broken": [],
  "followed_all_instructions": true,
  "further_revisions_required": false
}
```

---

## 第三部分：完整端到端示例

### 场景：愤怒客户的质量投诉

#### 初始状态

- **Agent**：电商退货客服，拥有 50 条 Guidelines
- **用户消息**："这个产品质量太差了！我要退货！而且你们客服态度太差！"

### 执行流程

#### 阶段 1: Guideline Matching (分 10 批并发)

**Batch 1 (G1-G5)**：
- Matched: G2 (客户想退货), G3 (客户坚持退货)

**Batch 2 (G6-G10)**：
- Matched: G8 (客户提到质量问题), G9 (客户愤怒)

**其他 8 个 Batch**：
- 无匹配

**汇总结果**：

```python
matched_guidelines = [G2, G3, G8, G9]
```

#### 阶段 1.5: 关系图过滤

**查询关系图**：

```
检查 G2 (提供折扣):
  ├─ G8 优先于 G2? ✓ (质量问题优先于挽留)
  └─ G9 优先于 G2? ✓ (愤怒优先于挽留)
  → 移除 G2

检查 G3 (处理退货):
  └─ G9 优先于 G3? ✓ (愤怒优先于正常流程)
  → 移除 G3

检查 G8 (质量问题):
  └─ 无更高优先级
  → 保留 G8

检查 G9 (客户愤怒):
  └─ 无更高优先级
  → 保留 G9
```

**过滤结果**：

```python
filtered_guidelines = [G8, G9]
```

**日志**：

```
[INFO] GuidelineMatcher: Matched 4 guidelines
[INFO] RelationalGuidelineResolver: Applying priority relationships
[INFO] Skipped: G2_offer_discount deactivated due to priority by G8_quality_issue
[INFO] Skipped: G2_offer_discount deactivated due to priority by G9_handoff_angry
[INFO] Skipped: G3_process_return deactivated due to priority by G9_handoff_angry
[INFO] Final guideline count: 2
```

#### 阶段 2: Message Generation

**Generation Prompt**：

```
你是一个电商客服 AI。

对话历史：
- User: "这个产品质量太差了！我要退货！而且你们客服态度太差！"

相关 Guidelines:
1. When 客户提到产品质量问题, Then 无条件接受退货，并致歉
2. When 客户非常愤怒, Then 立即转接人工客服

生成对客户的回复。
```

**LLM 生成**：

```
非常抱歉给您带来了如此糟糕的体验！

对于产品质量问题，我们会无条件为您办理退货并全额退款。
同时，我已将您的情况标记为高优先级，现在立即为您转接我们的客服经理，
确保您的问题得到最妥善的处理。

请稍等片刻，经理马上就会接入。再次为我们的失误向您道歉。
```

**自我评估**：

```json
{
  "instructions_followed": [
    "Guideline #1: 无条件接受退货并致歉",
    "Guideline #2: 立即转接人工客服"
  ],
  "instructions_broken": [],
  "followed_all_instructions": true,
  "all_facts_and_services_sourced_from_prompt": true,
  "further_revisions_required": false
}
```

### 对比：如果没有关系图会怎样？

**假设没有优先级过滤**，生成 Prompt 包含所有 4 条 Guidelines：

```
Guidelines:
1. 提供 10% 折扣券挽留客户
2. 帮助客户完成退货流程
3. 无条件接受退货并致歉
4. 立即转接人工客服
```

**可能的问题生成**：

```
非常抱歉给您带来不好的体验！

我们可以为您提供 10% 的折扣券作为补偿，同时也会无条件为您办理退货。
如果您需要，我可以转接人工客服。请问您想怎么处理呢？
```

**问题分析**：

- ❌ 同时提供折扣和退货，自相矛盾
- ❌ "如果您需要"转接人工，不够主动
- ❌ 没有优先处理愤怒情绪
- ❌ 让客户选择，增加摩擦

**有关系图后**：

```
非常抱歉给您带来了如此糟糕的体验！
我们会无条件为您办理退货，现在立即为您转接客服经理。
```

- ✅ 清晰、果断、无矛盾
- ✅ 优先处理情绪问题
- ✅ 不提及不适用的选项

---

## 完整数据流图

```
用户: "这个产品质量太差了！我要退货！而且你们客服态度太差！"
    ↓
┌────────────────────────────────────────┐
│ 阶段 1: Guideline Matching             │
├────────────────────────────────────────┤
│                                        │
│ All Guidelines (50 条)                  │
│   ↓                                    │
│ 分批策略：10 批 × 5 条/批                │
│   ↓                                    │
│ ┌──────┬──────┬──────┬─────┐ (并发)    │
│ │Batch1│Batch2│Batch3│ ... │           │
│ │G1-G5 │G6-G10│G11-G15│G46+│           │
│ └──────┴──────┴──────┴─────┘           │
│   ↓     ↓      ↓      ↓                │
│  LLM#1 LLM#2 LLM#3 LLM#10              │
│   ↓     ↓      ↓      ↓                │
│ [G2,G3][G8,G9] []     []               │
│   └──────┴──────┴──────┘               │
│           ↓                            │
│   Matched: [G2, G3, G8, G9]            │
└────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────┐
│ 阶段 1.5: 关系图过滤 (纯算法，无 LLM)    │
├────────────────────────────────────────┤
│                                        │
│ Input: [G2, G3, G8, G9]                │
│   ↓                                    │
│ 查询 PRIORITY 关系图 (NetworkX BFS):    │
│   ↓                                    │
│ G8 --priority--> G2 ✓ (G8 在列表中)     │
│ G9 --priority--> G2 ✓ (G9 在列表中) → 移除 G2 │
│ G9 --priority--> G3 ✓ (G9 在列表中) → 移除 G3 │
│   ↓                                    │
│ 查询 ENTAILMENT 关系 (无)               │
│   ↓                                    │
│ 查询 DEPENDENCY 关系 (无需过滤)          │
│   ↓                                    │
│ Output: [G8, G9]                       │
└────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────┐
│ 阶段 2: Message Generation             │
├────────────────────────────────────────┤
│                                        │
│ Prompt 构建：                           │
│ - Interaction History                  │
│ - Filtered Guidelines: [G8, G9] ← 只有 2 条! │
│   ↓                                    │
│ 调用 LLM (单次):                        │
│   ↓                                    │
│ 生成："非常抱歉... 无条件退货... 转接经理..." │
│   ↓                                    │
│ Revision 自我评估:                       │
│ - instructions_followed: [G8, G9]       │
│ - instructions_broken: []               │
│ - further_revisions_required: false     │
│   ↓                                    │
│ 输出最终消息                             │
└────────────────────────────────────────┘
    ↓
最终回复发送给用户
```

---

## 附录：常见问题

### Q1: 为什么不用向量数据库做 Guideline 匹配?

**A**：向量检索只能做**语义相似度匹配**，但 Guideline 匹配需要**推理能力**。

**示例**：

```
Guideline: "When 客户第三次询问同一问题且之前未得到满意答复，Then 转接人工"

向量检索能做的：
    找到语义相似的对话（如包含"第三次"、"问题"等关键词）

LLM Matching 能做的：
    1. 计数客户询问次数
    2. 判断是否同一问题
    3. 评估之前的答复是否满意
    4. 综合判断 condition 是否满足
```

### Q2: 两次 LLM 调用会不会很慢?

**A**：实际上很快，因为：

1. **并发处理**：10 个 Matching Batch 同时发起
2. **简单任务**：Matching 只需判断 true/false (低延迟)
3. **总延迟**：≈ max(batch_time) + generation_time < 3 秒

**性能对比**：

```
传统方案 (所有 guidelines 放入一个 prompt):
- Prompt 长度：~50KB
- LLM 延迟：~5 秒
- Token 消耗：~20K tokens

Parlant 方案:
- Matching: 10 批并发，每批 ~5KB，延迟 ~1.5 秒
- Generation: 1 次，~10KB，延迟 ~1 秒
- 总延迟：~2.5 秒
- Token 消耗：~15K tokens (节省 25%)
```

### Q3: 如果 Guidelines 太多 (如 500 条)，怎么办?

**A**：系统有多层优化：

1. **预过滤**：只加载 Agent 关联的 Guidelines
2. **分批大小自适应**：Guidelines 越多，batch size 越大
3. **缓存策略**：常用 Guidelines 优先匹配
4. **分层设计**：可以设置 Guideline 的优先级或分组

**示例**：

```python
# 500 条 guidelines
batch_size = 10  # 动态调整为 10 条/批
batch_count = 50  # 50 个批次

# 并发处理 (假设 LLM 支持 50 并发)
total_time ≈ 单个 batch 时间 ≈ 2 秒
# 仍然可以在可接受的时间内完成
```

### Q4: 如果 LLM 在 Matching 阶段判断错误怎么办?

**A**：系统有多重保障：

1. **Retry 机制**：Matching 失败会重试 3 次（`guideline_matching/guideline_matcher.py:159-171`）
2. **Temperature 调整**：失败后降低 temperature 提高准确性
3. **Few-shot Examples**：Prompt 中包含示例指导 LLM
4. **Logging & Monitoring**：记录所有匹配决策，便于审计
5. **人工审查**：可以查看 rationale 理解 LLM 的判断逻辑

**示例**：

```json
{
  "guideline_id": "G8",
  "condition": "客户提到产品质量问题",
  "rationale": "客户说'质量太差'，明确提到质量",
  "applies": true
}
```

如果发现错误，可以：
- 调整 Guideline 的 condition 表述
- 添加 few-shot example 指导 LLM
- 使用更强的 LLM 模型

### Q5: 关系图是否支持动态修改?

**A**：完全支持！关系存储在数据库中，可以随时修改：

```python
# 创建新关系
await new_guideline.prioritize_over(old_guideline)

# 删除关系
await relationship_store.delete_relationship(relationship_id)

# 修改关系类型
await relationship_store.delete_relationship(old_rel_id)
await source.entail(target)  # 从 PRIORITY 改为 ENTAILMENT
```

**重新加载**：系统启动时自动加载，或者手动触发重新构建图

```python
graph = await relationship_store._get_relationships_graph(kind)
```

### Q6: Guideline Matching 的准确率如何?

**A**：取决于多个因素：

1. **Guideline 质量**：condition 表述是否清晰
2. **LLM 能力**：更强的模型准确率更高
3. **Few-shot Examples**：提供示例可显著提升准确率
4. **Batch Size**：批次越小，判断越精确

**实践经验**：

```
Batch Size = 1：准确率 ~95%
Batch Size = 5：准确率 ~90%
Batch Size = 10：准确率 ~85%
```

**优化建议**：
- 重要 Guidelines 使用 batch_size=1
- 使用 Claude 3.5 Sonnet 或 GPT-4 级别模型
- 定期审查匹配日志，调整 condition 表述

---

## 总结

Parlant 的 Guideline 系统是一个**精心设计的多层架构**：

### 构建阶段 (离线)
- ✅ 手动定义 Guidelines 和关系（确定性、可控）
- ✅ 存储在文档数据库 + NetworkX 图（高效查询）
- ✅ 支持 4 种关系类型（PRIORITY、ENTAILMENT、DEPENDENCY、DISAMBIGUATION）

### 运行阶段 (在线)
- ✅ **阶段 1**：LLM 批量匹配 Guidelines（并发处理）
- ✅ **阶段 1.5**：图算法过滤冲突（纯规则，无 LLM）
- ✅ **阶段 2**：LLM 生成最终回复（只用过滤后的 Guidelines）

### 核心优势
- 🎯 **准确性**：LLM 推理匹配，比向量检索更智能
- ⚡ **性能**：分批并发，总延迟 < 3 秒
- 🔧 **可维护性**：关系图可视化、可审计、可测试
- 🛡️ **合规性**：人工定义规则，符合企业要求

这是一个**由人类定义规则，由 AI 智能执行，由算法高效优化**的完美结合！
