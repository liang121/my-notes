# AI Coding Philosophy: Solved or Sharpened?

> Sources: Boris Cherny (Anthropic), 2026-05-06; Matt Pocock (aihero.dev), 2026-05-06; Eric (Anthropic), 2026-04-23
> Raw: [Coding Is Solved — Boris Cherny](../../raw/ai-coding/coding-is-solved-boris-cherny.md); [Software Fundamentals — Matt Pocock](../../raw/ai-coding/software-fundamentals-matt-pocock.md); [Vibe Coding in Prod — Eric](../../raw/ai-coding/vibe-coding-in-prod-eric.md)

## Overview

到 2026 年，业内对"AI 时代工程师该怎么做"出现了三种立场：Boris Cherny 宣称 **coding is solved**，每天数十个 PR 由模型独立产出；Matt Pocock 反命题——**code is not cheap**，bad code 反而是史上最贵的；Eric 居中，提出 **vibe coding in prod** 这种"局部放手、关键把关"的方法论。三家观点表面冲突，本质上在争论同一件事：**模型变强后，工程师价值的剩余形态是什么**。

## 三家立场对照

| 维度 | Boris Cherny | Eric | Matt Pocock |
|---|---|---|---|
| 核心命题 | Coding is solved | Vibe coding in prod 是必修课 | Software fundamentals matter more |
| 对代码的态度 | "Model writes 100% of my code" | "Forget that the code exists, not the product" | "Code is not cheap. Bad code is the most expensive it's ever been" |
| Harness/工具层 | 重要性下降，prompt-injection/permission 等防护是临时脚手架 | 仍重要：plan、compact、leaf nodes 等模式 | 仍重要：grill-me、ubiquitous language、deep modules |
| 工程师剩余价值 | 跨学科 generalist；产品判断 | 做 Claude 的 PM；定义可验证抽象 | Strategic 层；20 年起步的设计基本功 |
| 标志性引用 | "Loops are the future." | "Ask not what Claude can do for you, but what you can do for Claude." | "AI is a sergeant on the ground. You are the strategic layer." |

## 核心张力：Code is cheap，还是 code is not cheap？

这是三家的真正分歧点。

- **Boris**: 隐含 code is cheap。Anthropic 内部"no manually written code anywhere"，重点是产品迭代速度而不是代码质量本身。
- **Matt**: 明确反对。**specs-to-code 反复跑编译器只会得到 garbage**——他亲身试过——是"vibe coding by another name"。他认为 AI 在好代码库里战力极强，所以 fundamentals 反而被放大了价值。
- **Eric**: 实操中调和——**叶子节点可以 vibe**，**主干必须 review**。tech debt 是"目前唯一在不读代码情况下无法衡量的指标"。

**注意冲突**：Matt 把 specs-to-code 整个否定了，但 Eric 在 22,000 行 PR 案例里实际上做的就是 spec-driven (虽然他不叫这个名)——区别在于 Eric 的 spec 限定在叶子节点 + 强可验证性，而 Matt 批判的是 spec-only / no-code-review 的极端版本。两人在"放手的范围"和"验证机制"上其实是同盟。

## 共识层（被分歧掩盖了）

剥开口号，三家都同意以下几点：

1. **上下文工程是关键**。Boris："give it a target and it will hill-climb until done"；Eric："be Claude's PM, give it everything"；Matt：grill-me + ubiquitous language。三家都把"前期投入对齐"当成核心动作，参见 [Working with AI as a PM](working-with-ai-as-pm.md)。
2. **架构边界比单文件质量更重要**。Eric 的叶子节点 = Matt 的 deep modules——见 [Leaf Nodes and Deep Modules](leaf-nodes-and-deep-modules.md)。
3. **指数曲线还在涨**。Boris 等"下一代模型"解决遗留代码库问题；Eric 强调"任务长度每 7 个月翻倍"；Matt 不否认这点，但坚持人类提供的"strategic 层"不会被这条曲线吃掉。

## 谁对？取决于在哪一层

- **小型项目 / 玩具应用 / 短生命周期工具**：Boris 立场最实用，code is cheap 的世界已经到了。
- **生产代码 / 长生命周期系统 / 团队协作**：Eric 的折中最稳——划清楚叶子节点和主干，分别对待。
- **基础设施 / 大型遗留代码库 / 性能关键系统**：Matt 立场最稳，省掉 fundamentals 注定踩坑。

老板自己的项目类型决定该听谁的。Boris 的"100% 写代码"是在 Anthropic 这个 AI 原生、流程从零搭起的特殊环境里跑出来的——拿到普通公司不一定可复制（Boris 自己也承认 Anthropic 的领先在"organizational structure and process"，不在技术）。

## 历史类比

Boris 用 **15 世纪的 printing press** 类比：印刷术之前识字率 10%、scribe 替不识字的国王写字；印刷术后 50 年内出版量超过此前 1000 年总和。他认为软件的民主化会比这快得多，"the best person to write accounting software is not an engineer, it's a really good accountant"。

Matt 不直接反驳这个类比，但他的回应可以这样理解：**印刷术让阅读民主化，但优秀的写作者仍然存在且更被需要**。AI 让写代码民主化，但优秀的系统设计者反而被放大了价值。

## See Also

- [Working with AI as a PM](working-with-ai-as-pm.md)
- [Leaf Nodes and Deep Modules](leaf-nodes-and-deep-modules.md)
