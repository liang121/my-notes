# Anthropic's Boris Cherny: Why Coding Is Solved, and What Comes Next

- **来源**：Sequoia Capital / YouTube
- **URL**：https://www.youtube.com/watch?v=SlGRN8jh2RI
- **时长**：24:36
- **嘉宾**：Boris Cherny（Claude Code 创建者，Anthropic）
- **主持**：Lauren Reader（Sequoia）

---

## Part 1 — Thesis（主访谈：Boris 的核心观点）

### 1. Claude Code 的起源:押注 "product overhang"

Boris 于 2024 年底加入 **Anthropic Labs**(内部孵化器)，团队规模很小，一起做出了 **Claude Code、MCP、Desktop App** 三个产品后解散，现在由 Mike Krieger(前 Instagram 联合创始人，现 CPO)重组 "round two"。

立项动机是一个判断 —— **product overhang**：

> "The model can do all the stuff that no product has yet captured."

2024 年底 coding 的 SOTA 还是 IDE 里的 **type-ahead**(Sonnet 3.5 加持的 tab 补全)。Boris 赌下一代模型可以让 agent 把整段代码全写了。

冷启动很难：
- 头 6 个月几乎不可用，他自己也只用它写 ~10% 的代码。
- 首次发布后用户不少，但**没有指数增长**。
- 拐点是 **Opus 4(2025 年 5 月)**，之后每发一个模型再 inflect 一次(4.5 → 4.6 → 4.7)。

> "We were trying to build this thing that was pre-PMF, and we knew that it wouldn't have PMF for 6 months because we were building for the next model."

### 2. "Coding is solved" 是什么意思

- Boris 个人自 2025 年 10/11 月起 **the model writes 100% of my code**。
- 日常每天 **a few dozen PRs**，最高记录 **~150 PRs in a day**。
- Claude Code 自身代码库就是 **TypeScript + React** —— 选这套是因为它对模型来说 **on distribution**。
- 但不是处处都 solved：大型遗留代码库、冷门语言依然吃力。**"Usually the answer is just wait for the next model."**

### 3. 他自己的 setup(基本在手机上工作)

- 主战场是 **Claude iOS app** 的 code tab。
- 任意时刻挂着 **5–10 sessions**、**几百个 agents**；夜间跑 **几千个**。
- 杀手级原语是 **`/loop`** —— 让 Claude 用 cron 起一个周期性 job(每分钟 / 每 5 分钟 / 每天)：
  - 一个 loop 在 **babysit PRs**(修 CI、auto-rebase)。
  - 一个 keeps **CI healthy**(修 flaky test)。
  - 一个每 30 分钟从 Twitter 抓反馈做聚类。
- 新发布的 **routines** = 服务端版的 loop，关掉笔记本也跑。

> "Loops are the future."

### 4. 未来的团队形态：cross-disciplinary generalists

今天说 generalist 还是指 full-stack 工程师(iOS + web + server)。Boris 认为下一波是**跨学科的**通才 —— 工程师同时是优秀的 designer、PM、data scientist。

Claude Code 团队已经如此：EM、PM、designer、data scientist、finance、user researcher，**every single person on our team writes code**。

### 5. 是不是 SaaS apocalypse？两个预测

借 Hamilton Helmer 的 **7 Powers** 框架：

**会被 AI 削弱的 powers：**
- **Switching costs** —— 模型可以轻松把应用从一个平台搬到另一个。
- **Process power** —— Claude(尤其 4.7)非常会摸清楚 workflow，**"give it a target and it will hill-climb until done"**。

**仍然成立的 powers：**
- Network effects、scale economies、cornered resources。

**第二个预测**：未来 10 年颠覆型 startup 的数量会 **~10×**。原因是大公司要演化业务流程、重训员工、面对内部阻力；新公司可以 **build with AI natively from the ground up**。

> "It's the best time to build."

---

## Part 2 — Q&A(观众提问环节)

### Q1(Dan)：成功是模型的功劳还是产品决策的功劳？

Boris：一年前看是 **50/50**，现在模型变强、harness 的重要性在下降。

为什么产品仍然重要？因为他做过 YC，YC 反复灌输的是 **"build something people love"** —— 不管模型多强，你最终交付给用户的是体验。

未来 harness 的演化方向：把 **loops 做成 first-class**、更易做大规模 multi-agent；**安全机制(prompt injection、static command verification、permission mode、human-in-the-loop)**会随着模型对齐变好而逐渐淡化。

### Q2：编程会变成一项类似办公软件的全民技能吗？

Boris：**比那更彻底**，编程会像"我会发短信"一样自然。

他给出最直观的历史类比 —— **15 世纪欧洲的 printing press**：
- 印刷术之前，欧洲识字率约 10%，scribe 受雇于不识字的国王和贵族。
- 印刷术出现后 50 年内，欧洲出版的文献量超过此前 1000 年的总和；书的成本下降 ~100×。
- 之后几百年里，全球识字率攀升到 ~70%。

软件的民主化将**比这快得多**。推论：

> "The best person to write accounting software is not an engineer, it's a really good accountant — because they know the domain really well and coding is the easy part."

### Q3：Anthropic 内部和外部的差距，是 1 个月、3 个月还是 6 个月？

- **模型层面**：基本没有差距 —— Anthropic 内部用的也是公开模型(少量 metheus 试用，主要是 **Opus 4.7** 自己 dogfood)。
- **产品 / 流程层面**：差距更大。Anthropic 内部已经 **"no manually written code anywhere at the company"**，所有 SQL 都是模型写的，Claudes 之间在 **Slack 上互相通信**协调任务。

> "The place that we're ahead is not the technology... but the organizational structure and organizational process."

这正是 startup 的优势 —— 没有历史包袱。

### Q4(Jared)：multi-agent / 并行化怎么演化？

- 产品侧：靠 **prompt 调优**，让模型更倾向并行。
- 模型侧：4.7 已经会**自发起 loop** —— 例如 Boris 让它跑一个 data query，它发现数据在变，就主动说"我起一个 loop，每 30 分钟通过 Slack MCP 给你发一份 report"。
- 长期：用户不应该需要自己判断"什么时候该并行"，那是产品设计问题，更应该由模型自己处理。

### Q5：未来是云端 agent 还是本地 agent？

> "It doesn't matter."

Boris 认为再过几年，**起 agent、搭环境**这些事都由模型自己决定 —— 用本地模型还是云端模型不再是工程师要关心的问题。

### Q6(Jamie)：Co-work 怎么解决知识工作工具大多在云上的问题？

- 大部分知识工作工具(Salesforce、Docs、Calendar)已经原生在云上 —— 解法就是 **MCP**。
- 同一套 MCP connector 在 Claude AI、Co-work、Claude CLI、Claude Code 全部复用。
- 没有 MCP 的系统，靠 **computer use** 兜底(Anthropic 在这块走得最前，4.7 上速度慢但质量很好)。

> "It could be MCPs, APIs, just some sort of programmatic access — to the model, it's just tokens."

### Q7(Ryan)：现在该 build 什么产品，能在 6–12 个月后随模型变强而变得更有价值？

Boris 给的方向：
- **Claude Design** —— 现在已经不错，会变得很好。
- **Claude Code** 还有几个 feature 即将上线。
- **massively parallelizing agents**(loop、batch 这类原语)。
- **Computer use**。

---

## 延伸思考(老板可能关心的点)

- **`/loop` + routines** 是 Boris 反复 highlight 的工作方式 —— 老板自己已经在用 superpowers / loop skill，可对照他的用法（babysit PRs、CI 自愈、反馈聚类）扩展常驻 loop 列表。
- **harness 重要性递减**论值得警惕：Boris 暗示当前所有 permission mode、prompt-injection 防护都是**临时脚手架**，随模型对齐成熟会被裁掉；做 Claude Code 周边工具时不要过度投资在这一层。
- **"on distribution" 选型直觉**：自家代码库选 TypeScript + React 不是审美而是为了让模型更好用 —— 反推老板自己的项目选型时，可考虑"模型分布"是一个选型维度。
- **printing press 类比**最值得再细嚼：**领域专家 + AI > 工程师 + AI**(会计师写会计软件比工程师更合适)。这对老板做工具型产品的目标用户定义有直接启示。
