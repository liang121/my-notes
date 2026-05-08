# Leaf Nodes and Deep Modules

> Sources: Eric (Anthropic), 2026-04-23; Matt Pocock (aihero.dev), 2026-05-06
> Raw: [Vibe Coding in Prod — Eric](../../raw/ai-coding/vibe-coding-in-prod-eric.md); [Software Fundamentals — Matt Pocock](../../raw/ai-coding/software-fundamentals-matt-pocock.md)

## Overview

Eric 和 Matt 看似在争论 AI 时代代码还重不重要（Eric "forget the code"、Matt "code is not cheap"），但他们对**架构**的处方惊人一致：**给 AI 一块边界清晰、内部可塑、外部可验证的空间，不要让 AI 碰主干**。Eric 叫这种空间 **leaf nodes（叶子节点）**，Matt 叫 **deep modules**——同一个想法的两种表述，且都借自 John Ousterhout 的 *A Philosophy of Software Design*。

## 同一个想法，两种叫法

### Eric 的 Leaf Nodes

Eric 的论证从"tech debt 是唯一无法不读代码就衡量的指标"出发：

- **叶子节点** = 没有其他模块依赖它的末梢功能。
- 在叶子节点上即使产生 tech debt，也不会**向上污染**整个 codebase。
- 因此叶子节点是 **可以放心 vibe code** 的地方。
- **主干 / 核心架构**仍需要工程师亲自设计和 review，确保未来扩展性。

22,000 行 PR 的关键决策之一就是"把改动集中在叶子节点"。

### Matt 的 Deep Modules

Matt 从 Ousterhout 引入：

> Deep modules — lots of functionality hidden behind a simple interface.
> Shallow modules — not much functionality, complex interface.

shallow modules 充斥的 codebase 让 AI 容易迷失（"AI 走不到正确的 module、不理解依赖、不理解你的代码"）。AI 自己也很喜欢生产 shallow modules——这是个负反馈循环。

Matt 的方案：把相关代码**包到一个深模块里**。**接口由人类设计**（控制力强），**实现交给 AI**（gray box）。

> Design the interface, delegate the implementation.

## 为什么是同一回事

看它们的可观测属性：

| 属性 | Leaf Node (Eric) | Deep Module (Matt) |
|---|---|---|
| 边界形态 | 没有上游依赖 | 简单的对外接口 |
| 内部 tech debt | 容许（不向上扩散） | 容许（隐藏在接口后） |
| 人类把关位置 | 输入输出可验证 | 接口设计 |
| AI 工作位置 | 节点内部 | 模块内部 |
| 失败时的爆炸半径 | 仅自身 | 仅模块内部 |

两人都在做一件事：**给 AI 一块爆炸半径有限的工作空间**。Eric 从依赖图角度切（"它没有上游"），Matt 从抽象边界角度切（"它对外接口窄"）。一个项目里这两种属性常常同时出现——一个 deep module 通常也是一个 leaf node。

## 实操：一个 AI 友好的 codebase 长什么样

综合两家：

1. **少量、宽松、深的模块**，而不是大量小函数和散点 utility。
2. **每个模块对外一个稳定接口**——人类亲自设计、文档清晰、用 ubiquitous language（参见 [Working with AI as a PM](working-with-ai-as-pm.md)）。
3. **接口处可测试**——TDD 测的是边界行为，不是实现细节。Eric 的 "3 个 end-to-end 测试"模式特别适合 deep modules。
4. **模块内部允许 vibe**——AI 写、AI 测、AI 重构都可以；只要边界稳定就不影响别人。
5. **跨模块的"主干"由人类把关**——edits、refactors that cross module boundaries 必须有工程师介入。

Matt 的 grey box 比喻：**外面盒子是黑的（你只看接口），里面盒子是 grey 的（需要时可看，但默认不看）**。这同时减轻人脑负担和 AI 探索负担。

## 反模式：Shallow Modules + Specs-to-Code

Matt 直接点名：**AI 特别擅长制造 shallow modules**——大量小函数、复杂依赖、没有清晰边界。配上 specs-to-code 的工作流（不读代码 → 跑编译器 → 出 garbage → 再跑编译器 → 更多 garbage），就是 *vibe coding by another name* 的恶化版。

如果老板观察到自己的项目正在变成 shallow modules（AI 经常找不到正确的文件、上下文塞不下、改一个地方破坏另一个），这是 deep module 重构的明确信号。Matt 提到他有个 "improve codebase architecture" skill 专门做这种重构。

## 跟 tech debt 衡量难题的关系

Eric 强调的"tech debt 是 vibe coding 唯一例外"在 deep module 框架下变得更可操作：

- **跨模块 tech debt** = 主干/接口设计问题——人类必须读、必须审。
- **模块内 tech debt** = 局部问题——容许、爆炸半径有限、未来可整模块替换。

这等于把 tech debt 划成了"人类必读"和"可暂存"两类，给"什么时候必须读代码"提供了清晰的判断标准。

## 历史血缘

两家都引用同一本书——Ousterhout *A Philosophy of Software Design*。Matt 还引用 *The Pragmatic Programmer*（software entropy、outrunning your headlights）和 Frederick Brooks *The Design of Design*（design concept、design tree）。这些书都是 AI 时代之前的**人机协作**经典——只不过协作对象当年是同事，现在是 AI。**软件设计的根本约束不是计算机能力，是协作认知带宽**——这也是 Matt "fundamentals matter more" 论点的底层逻辑。

## See Also

- [AI Coding Philosophy: Solved or Sharpened?](ai-coding-philosophy.md)
- [Working with AI as a PM](working-with-ai-as-pm.md)
