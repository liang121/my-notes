# Claude Code Harness 架构拆解：四层模型 + 生产级 agent 工程经验

> Sources: Rohit on X, "How I built harness for my agent using Claude Code leaks", 2026-04-13
> Raw: [Rohit on X: How I built harness for my agent using Claude Code leaks](../../raw/ai-agents/2026-05-08-claude-code-harness-architecture.md)

## Overview

Rohit 把 Claude Code 源码（55 目录 / 331 模块的 TypeScript）拆开来读了一遍，得出一个核心观点：**业界讲的 "model weights / context / harness" 三层模型不够用，真实生产级 agent 还有第四层 infrastructure（多租户、RBAC、资源隔离、状态持久化、分布式协调）**。三层模型把 harness 当天花板，但真正决定产品能不能跑起来的是第四层；大多数团队卡在 demo 不能上生产，就是因为没把 layer 4 当工程问题处理。这篇文章把每一层的关键决策逐条拆开，是一份"自己造 harness 时该抄什么"的蓝图。

下文按照"四层模型 → agent loop 设计 → 工具与上下文 → 权限与重试 → sub-agent 与基础设施 → 扩展机制 → 总结清单"的顺序展开。每一节都尽量给出**和教程做法的对比**，因为作者反复强调：教程教的 while-loop + try-catch 在生产里会从五个方向同时塌。

## 一、三层 vs 四层框架

业界通行的三层：

1. **Model Weights**：冻结的智能，API 后面那坨权重
2. **Context**：prompt + 对话历史 + 检索文档
3. **Harness**：模型周围的脚手架——工具、循环、错误处理

三层模型本身没错。Princeton NLP 的 SWE-agent 论文证明过：同一个 GPT-4，光改 interface 设计就能在 SWE-bench 上提升 64%——性能增益主要来自第 2、3 层而不是第 1 层。

但拆开 Claude Code 源码会发现 Anthropic 不只是在为模型造 harness，更是在为整个系统造基础设施：

- 四级 CLAUDE.md 层级让企业管理员通过 MDM 强制策略，项目维护者定约定，开发者本地覆盖
- 文件锁加持的磁盘 task list 让并行 sub-agent 不互相污染状态
- Git worktree 隔离让 5 个 agent 在同一 repo 上各拿一个分支，零冲突
- 权限 pipeline 把 deny 规则从企业级一路级联到 session 级

这些都不是 harness、不是 context、不是 weights。**这是 infrastructure：多租户、RBAC、资源隔离、状态持久化、分布式协调。**

所以真正的框架是四层：

```
Layer 1: Model Weights      ——  冻结的智能
Layer 2: Context            ——  运行时输入
Layer 3: Harness            ——  agent 的设计环境
Layer 4: Infrastructure     ——  多租户/RBAC/隔离/持久化/协调
```

> "Most teams talk about the first three because they are interesting to think about. The fourth is where products die."
>
> —— 大多数团队止步于 layer 3，造的是 demo。Claude Code 是作者见过第一个把四层都当回事的 agent 系统。

行业把多数算力砸在 layer 1（更大的模型、更高的 benchmark）；真正赢的团队在投资 layer 3 和 4——更好的环境、更好的错误恢复、更好的权限、更好的上下文管理、更好的协调。

## 二、Agent Loop：用 async generator 而不是 while 循环

Claude Code 的核心在 `query.ts`（1729 行 TypeScript）。最重要的决策在函数签名里：

```typescript
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | ToolUseSummaryMessage,
  Terminal
>
```

`function*` 这一颗星比看上去重得多。Async generator **天然提供 streaming、cancellation、composability、backpressure 四样东西**，而不需要外面再贴。

### 教程做法（while loop）会塌的五个原因

```typescript
async function runAgent(input: string) {
  let messages = [{ role: "user", content: input }];
  while (true) {
    const response = await callModel(messages);
    messages.push(response);
    if (response.stop_reason === "end_turn") break;
    for (const toolCall of response.tool_calls) {
      const result = await executeTool(toolCall);
      messages.push(result);
    }
  }
  return messages;
}
```

教程能跑通，生产从五个方向塌：

| 缺什么 | 后果 | Generator 怎么解决 |
|---|---|---|
| **No streaming** | 用户盯着空屏幕等 10-30 秒 | 边出 token 边 yield StreamEvent；用户能看到 → 信任 → 给更多自主权 |
| **No cancellation** | Ctrl+C 要外接 abort 机制 | 调用方停止 `.next()`，finally 自动清理；AbortSignal 沿每层透传 |
| **No composability** | REPL/sub-agent/test 各写一份循环 | 一个 `query()` 函数，三种调用方，零重复 |
| **No backpressure** | 模型出得快、终端渲得慢 → 内存吃光 | 消费者不拉就暂停生产，长 session 内存有界 |
| **No error recovery in loop** | 错误处理在最外层 try-catch | 错误恢复是循环的 first-class state（见下） |

### 五阶段循环 + 错误恢复在循环内

每次迭代跑五个 phase，错误恢复**不在外层 try-catch，而是循环状态机的 first-class state**：

1. **Setup**：调模型前先跑 tool result budget、必要时压缩、校验 token
2. **Model Invocation**：通过依赖注入接口调 `queryModelWithStreaming()`，外面套处理 10 类错误的 retry。**关键**：streaming tool executor 在这个 phase 就开始执行工具——一个 Grep 调用在它的 input JSON 刚流完时就开始跑，比下一个工具调用到达还早几秒
3. **Error Recovery & Compaction**：模型回完后检查可恢复错误。`prompt-too-long` → 压缩重试；`max_output_tokens` 撞上 → 从 32K 升到 64K 重试；context overflow → 对媒体重的消息做反应式压缩
4. **Tool Execution**：streaming executor 没跑完的工具在这里跑完。结果到 UI。Haiku 异步生成 tool use summary，主模型不烧 token 在记账上
5. **Continuation Decision**：看 `stop_reason`、`maxTurns`、hooks 是否要求停、abort signal 是否触发。继续 → state++ → 回到 Phase 1

> "Each phase knows what can go wrong and has a specific recovery path. That is the difference between an agent that crashes on a rate limit and one that backs off, retries, falls back to a different model, and keeps working."

### 依赖注入让 loop 可测

```typescript
type QueryDeps = {
  callModel: (params) => AsyncGenerator<StreamEvent>
  microcompact: (messages) => Message[]
  autocompact: (messages, summary) => Message[]
  uuid: () => string
}
```

注入一个 mock `callModel` 就能验证 context overflow 处理、工具失败、cancellation——不用碰真 API。**多数 agent harness 不可测，因为 API 调用硬编码在循环里。Claude Code 的 loop 是带注入副作用的纯状态机。**

## 三、工具系统：并发分类 + Streaming 执行 + Result Budget

### 工具按并发行为分类

每个工具自描述并发性：

```typescript
type Tool<Input, Output> = {
  name: string
  call: (input, context) => Promise<Output>
  isConcurrencySafe: () => boolean    // 能并行吗？
  isReadOnly: () => boolean           // 纯读吗？
  isDestructive: () => boolean        // 会丢数据吗？
  maxResultSizeChars?: number          // 上下文 budget
}
```

`toolOrchestration.ts` 把工具调用分批：

- **只读工具**（Glob、Grep、Read、WebFetch）→ 并发跑，最多 10 个并行
- **写工具**（Bash mutation、Edit、Write）→ 串行跑

效果：搜 5 个文件并行、改 1 个文件串行——并行的速度 + 串行的安全，2-5x 加速、零 race。

### Streaming Tool Executor：流中执行工具

多数 harness 等模型生成完才开始执行工具。Claude Code **在流的中间就开始执行**——三次工具调用的 turn 能藏 2-5 秒延迟。

硬情况都处理了：

- 并行批次里某工具失败 → per-tool `siblingAbortController` 杀掉兄弟工具，**parent query controller 不死**，对话继续，模型收到 error 自我恢复
- Stream 失败 fallback 到非 streaming → 抛弃排队工具，给在飞的工具生成合成 error 结果
- 哪怕工具 2 比工具 1 先完成，**结果按原始顺序 yield**，叙事保持连贯

### Tool Result Budget：防止 cat 一个 1MB 文件淹掉上下文

```
1. 每个工具声明 maxResultSizeChars
2. 超限的结果落盘
3. 模型收到的是文件路径引用 + 前 N 字符预览
4. applyToolResultBudget() 在每次 API 调用前约束总 tool result token
```

> "Users will run cat on enormous files. They will pipe commands that produce megabytes of output. Without budgeting, the context fills with noise and the agent loses coherence."

这条不会出现在架构图里，但决定 agent 能不能撑过真实使用。

## 四、System Prompt + CLAUDE.md：为缓存设计

### Prompt 是结构化数组而非字符串

```
[GLOBALLY CACHEABLE — 所有用户/会话共享]
  1. Introduction (identity, capabilities)
  2. System (platform, tools, environment)
  3. Doing Tasks (coding guidelines, security)
  4. Actions (reversibility, blast radius)
  5. Using Your Tools
  6. Tone & Style
  7. Output Efficiency

=== SYSTEM_PROMPT_DYNAMIC_BOUNDARY ===

[PER-SESSION — 本地缓存，/clear 或 /compact 重算]
  8. Session Guidance
  9. Memory
  10. Environment Info (model/platform/git status)
  11. Language
  12. MCP Instructions (UNCACHED — 每轮重算)
```

`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 这条线**把约 80% 的 prompt 划进全局缓存**——所有用户每次 API 调用都不再重新 tokenize 那 577+ 行。线下面分两种：每 session 算一次的 memoized 段，和每轮重算的 volatile 段（最小化，因为 volatile 段一变会让它后面所有缓存全失效）。

> "No agent tutorial, framework doc, or conference talk I have encountered discusses designing the prompt for cache efficiency. It is one of the highest-leverage decisions in the codebase. At scale, this determines whether your agent costs $0.02 per session or $0.20."

### CLAUDE.md 四级层级 = 可组合记忆 = RBAC for agent behavior

```
优先级（低 → 高）：
1. Enterprise:  /etc/claude-code/CLAUDE.md          ——  企业级，可对接 MDM
2. User:        ~/.claude/CLAUDE.md                  ——  个人偏好
3. Project:     CLAUDE.md / .claude/CLAUDE.md        ——  项目约定
4. Local:       CLAUDE.local.md (private, gitignore) ——  开发者私货
```

高层覆盖低层。`@./docs/coding-standards.md` / `@~/shared-rules.md` 这种 `@` 指令实现组合。这是给 agent 行为做的 RBAC，冲突按优先级确定性解决。

### 用户上下文为什么不放 system prompt

git status / CLAUDE.md 内容 / 当前日期都注入成**第一条 user message**，包在 `<system-reminder>` 标签里：

```
<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
[contents of CLAUDE.md files]
# currentDate
Today's date is 2026-04-07.
</system-reminder>
```

原因：上下文每轮都变。放 system prompt 会让变化点之后的缓存全部失效。挪到 user message 里 system prompt 缓存稳定。**小细节，重大成本影响。**

## 五、四种压缩策略（按成本从低到高）

多数 harness 直接 truncate 旧消息或在撞上 context 上限时 crash。Claude Code 用四级策略，便宜的先跑：

| # | 策略 | 触发 | 成本 | 做什么 |
|---|---|---|---|---|
| 1 | **Tool Result Reuse Cache** | 每轮 API 调用前 | 趋零 | 工具结果未变就用缓存引用替换全文（同文件 Read 的场景能省几千 token/session） |
| 2 | **Snip** | 接近 token 上限、还没到 summarization 之前 | 无模型调用 | 从对话开头删消息，保留 "protected tail" |
| 3 | **Summarization** | token 跨阈值且 snip 不够 | 一次模型调用 | 单独模型调用总结旧消息，**系统跟踪 compaction state 防"总结的总结的总结"循环** |
| 4 | **Context Collapse** | 跑了好几小时的长会话；feature flag 开启 | 最贵 | 多阶段分层压缩：先 tool result → 再 thinking block → 再整段 |

### Hierarchy 为什么重要

- 多数实现压缩的 harness **直接跳到 summarization**——而 summarization 既要付压缩调用本身的 token，又要付 summary 的 token。Microcompact 和 snip 能零模型调用处理大量场景
- **Protected tail**：压缩跑的时候最近 N 条消息永远不会被总结掉。模型保持当前计划的全保真，不会忘记自己刚做了什么

## 六、权限系统：七阶段管线 + 五种模式 + Hooks 逃生舱

### 七阶段管线

多数 agent 是二元 toggle："允许/拒绝"。Claude Code 是 7 阶段：

```
Tool Call Requested
  → Stage 1: INPUT VALIDATION (Zod schema)
  → Stage 2: DENY RULES (enterprise → project → user → session)
  → Stage 3: ALLOW RULES (pattern matching)
  → Stage 4: TOOL-SPECIFIC CHECK
  → Stage 5: HOOKS (external scripts)
  → Stage 6: CLASSIFIER (ML model, optional)
  → Stage 7: USER PROMPT (interactive dialog)
  → ALLOW / DENY
```

规则用 glob：

```
Bash(git *)         → 允许所有 git 命令
Bash(npm test)      → 仅允许 npm test
Write(src/**/*.ts)  → 允许写 src/ 下的 TS
Read(**)            → 允许读任何文件
```

> "'Allow all bash' is too coarse. Nobody wants their agent running rm -rf /. 'Deny all bash' makes the agent useless. 'Allow git commands and npm test, prompt for everything else' is the sweet spot."

### 五种 Permission Mode：渐进信任而非二元开关

```typescript
type PermissionMode =
  | 'default'           // 每次工具使用都问
  | 'plan'              // 执行前展示计划
  | 'acceptEdits'       // 自动接受文件编辑
  | 'bypassPermissions' // 跳过所有检查（power user）
  | 'auto'              // ML 分类器决定
```

新用户从 default 起步，建立信任后挪到 acceptEdits 或 bypassPermissions。**不是安全和速度的二选一，是一个谱系。**

### Hooks 是逃生舱

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": { "tool": "bash", "action": ".*rm.*" },
      "commands": [{ "cmd": "/path/to/safety-check.sh" }]
    }]
  }
}
```

脚本收到 tool call 细节，返回 `{"decision": "approve"}` 或 `{"decision": "block"}`。组织可以构建自定义护栏：阻断破坏性操作、完成时 post Slack、每次文件写后跑 linter——**不改源码**。

## 七、重试是状态机：每类错误独立恢复路径

`services/api/withRetry.ts` 有 823 行，每行的存在都对应一次生产事故。

| HTTP 错误 | 处理策略 |
|---|---|
| **429 (Rate Limited)** | 看 Retry-After header：<20s 直接重试保 fast mode；>20s 进 30 分钟冷却；`overage-disabled` header 出现就永久禁 fast mode 并解释原因 |
| **529 (Server Overloaded)** | 跟踪连续 529 计数：3 次连续且有 fallback model → 切模型；后台任务 → bail 防级联；前台 → 退避重试 |
| **400 (Context Overflow)** | 解析错误抽出 actual 和 limit token；重算 `available = limit - input - 1000 (safety buffer)`；最低 3000 output token 的下限；带新 budget 重试 |
| **401/403 (Auth)** | 清 API key 缓存；强制刷 OAuth token；用新凭证重试 |
| **Network (ECONNRESET / EPIPE / timeout)** | 关 keep-alive socket pool；用新连接重试 |

退避公式：`delay = min(500ms × 2^attempt, 32s) + random(0, 0.25 × baseDelay)`

### 持久重试模式（CI/CD、后台 agent）

- 429/529 无限重试
- 最大 5 分钟退避
- 6 小时 reset cap
- 30 秒心跳防 idle kill

### Streaming 层独立可靠性

- 90 秒 idle timeout watchdog 中断 stream，45 秒先 warning
- chunk 之间 30+ 秒 gap 记录 stall（time-to-first-byte 之后）
- streaming 整体失败 → fallback 到非 streaming，**保留连续 529 计数**让 fallback 逻辑不会重复计数

> "A three-retry wrapper around fetch is not production reliability. A state machine that understands the semantics of every error class and has a specific recovery path for each one is."

## 八、Sub-Agent 系统：context 隔离 + worktree 隔离 + 文件锁协调

Claude Code 派生独立的 agent loop 实例，每个有自己的 context、tools、working directory：

```typescript
const input = z.object({
  description: z.string(),       // 3-5 词描述
  prompt: z.string(),
  subagent_type: z.string(),     // 'Explore' / 'Plan' / 'general-purpose'
  model: z.enum(['sonnet', 'opus', 'haiku']),
  run_in_background: z.boolean(),
  isolation: z.enum(['worktree', 'remote']),
})
```

### Context 隔离

```typescript
function createSubagentContext(parentContext, overrides) {
  return {
    abortController: new AbortController(),    // parent 的 child
    appState: noopSetAppState(),                // 不能 mutate parent
    readFileState: cloneFileStateCache(),       // clone LRU cache
    toolDecisions: clonedDecisions(),           // 权限一致
    agentId: uniqueId(),
    queryTracking: { chainId, depth: depth + 1 },
  }
}
```

- abort parent → 级联到所有 child
- child 不能 mutate parent state（appState 是 noop setter）
- file state cache clone 防止一个 agent 的读污染另一个的缓存

### Worktree 隔离（修改代码的 sub-agent）

```
getOrCreateWorktree(repoRoot, slug)
  → 验证 slug（max 64 字符、禁路径穿越）
  → 检查 worktree 是否已存在（fast resume）
  → git fetch（带 no-prompt 环境变量）
  → git worktree add 新分支 worktree-<slug>
  → symlink 大目录（node_modules、.cache）
  → 拷贝 CLAUDE.md、project settings、.env
  → return { path, branch, headCommit }
```

一 agent 一 worktree。并行 agent 共享 workspace 必出冲突。`node_modules` 用 symlink 防磁盘膨胀——5 个并行 agent 不需要 5 份依赖。

### 三种执行后端

| 后端 | 隔离级别 | 速度 |
|---|---|---|
| **In-process** | Node.js 直接跑，共享内存 | 最快 |
| **Tmux pane** | 终端 multiplexer 隔离，每个 agent 一个 tab | 中 |
| **Remote (CCR)** | 全机隔离 | 最慢 |

### Task Coordination

磁盘 task list + 文件锁，路径 `~/.claude/tasks/<taskListId>/<taskId>.json`。锁竞争用指数退避（30 次重试，5-100ms）。high water mark 防止 task ID 在 reset 后被重用。

## 九、Layer 4 Infrastructure：跨 session/agent/team 的工程问题

总结 layer 4 的工作面（这些是大多数 demo 没做的）：

### 状态持久化的三个层级

- **Session 内**：compaction（auto-compact 出的 summary 成为下一轮 loop 的起始 context）
- **跨 session**：CLAUDE.md 文件保存项目级记忆；Hooks 持久化任意状态到磁盘
- **跨 agent**：task coordination system 在多 agent 进程间维持状态

### 隔离与协调

| 工程问题 | Claude Code 的方案 |
|---|---|
| 文件系统隔离 | git worktree per sub-agent |
| 工具失败不级联 | siblingAbortController |
| 资源访问控制 | enterprise-level deny rules |
| 分布式协调 | task list 文件锁 |
| 分布式资源优化 | parent/child agent 共享 prompt cache |
| Keepalive | 持久重试模式的 30 秒心跳 |
| 并发 repo 访问 | worktree 管理 |

> "These are infrastructure problems with infrastructure solutions: locking, coordination, isolation, state management, access control, resource sharing."

### 为什么三层模型骗不了你

> "The three-layer model does not prepare you. It frames the harness as the ceiling. But the harness describes how one model instance interacts with one set of tools in one session. The moment you need multiple users, multiple sessions, multiple agents, or deployment to environments you do not control, you are in layer 4."

**分布式系统工程正在成为 agent builder 的核心能力。** 生产级 agent 系统跑在 CI 服务器、派生子进程、跨 session 共享状态、服务不同权限的用户。理解这一点的团队造能跑的 agent，止步于 harness 的团队造 demo。

## 十、四种扩展机制：composition over modification（不改源码）

| 机制 | 形式 | 关键特性 |
|---|---|---|
| **Skills** | 带 YAML frontmatter 的 markdown 文件（name/description/allowed-tools/model/context/user-invocable） | 五种来源：bundled / project / user / plugin / MCP；**path-based discovery**——skill 指定 `paths: ["*.tsx"]` 只在 agent 接触匹配文件时激活，agent 看到的是相关 skill 不是全部 skill |
| **Hooks** | 6 类：shell command / LLM evaluation / agentic verification / HTTP endpoint / TS callback / in-memory function。触发：PreToolUse / PostToolUse / SessionStart / FileChanged / Stop | 把 harness 接到现有基础设施：post Slack、跑 security scanner、触发 CI——不改 agent loop |
| **MCP** | 5 种传输：stdio / SSE / HTTP streaming / WebSocket / in-process。三级配置：enterprise managed / project / user | 通过标准协议给 agent 接外部系统（DB / API / 内部工具） |
| **Plugin Directories** | 含 skills / agents / hooks / config 的目录 | 顶层组合机制：加能力不动现有文件 |

四种机制共同原则：**composition over modification**——加而不改。核心更新不破坏扩展，扩展之间互不干扰。

> "Extension points that require no code changes. Markdown files, shell scripts, protocol-based tools cover 95% of extension needs. **If users fork your code to customize behavior, your architecture has a gap.**"

## 十一、UI 是力量倍增器

终端 UI 跑在 Ink (React for terminals) 的自定义 fork 上：251KB 渲染引擎，对一个 CLI 来说听起来重——但不重。

- 实时流式文字逐字渲染
- 动画 spinner 根据 stall 时长从 normal 渐变到 error-red
- diff 渲染：语法高亮 + 三行上下文 + 词级变化标记
- 多 agent 状态树显示活跃 agent 层级
- 6 种主题，含色盲友好选项
- 状态行：模型名 / USD 成本 / context window 使用百分比 / rate limit 利用率

> "Users who can see what an agent is doing... give that agent more autonomy. More autonomy means more useful work done. The UI is a force multiplier."

context window 使用条让用户不需要懂 tokenization 也能直觉感知剩余容量——快满了就知道该收尾或让 agent 压缩。

## 十二、抄什么：工程决策清单

不需要重造 Claude Code，需要的是它背后的工程决策：

1. **Async generator 写 agent loop**：streaming / cancellation / composability / backpressure 都内建。while loop 把这四样全留在桌上。
2. **工具按并发分类**：只读并行、改写串行；定义时打标，编排层处理 batching；2-5x 加速 + 零 race。
3. **Streaming 期间执行工具**：增量解析 tool call，input JSON 一完整立刻执行；多工具 turn 免费省延迟。
4. **System prompt 为缓存边界设计**：静态在前，动态在后，边界显式标记。生产级最高杠杆的成本优化。
5. **压缩做成层级而非单一策略**：cheap first（microcompact / snip）、expensive last（summarization / collapse）。便宜的能搞定就不烧贵的。
6. **错误恢复是循环里的 first-class state**：rate limit / context overflow / auth failure / network error 各有自己的状态机内恢复路径。**不是外层 try-catch。**
7. **从第一天就有 layer 4**：状态跨 session 怎么存？权限怎么 scale 到团队？并行后协调怎么搞？**事后改基础设施比一开始就为它设计难一个数量级。**
8. **扩展点不要求改源码**：markdown、shell script、protocol-based tool 覆盖 95% 扩展需求。**用户为定制行为 fork 你代码 = 架构有洞。**

四层。行业大多在卷 layer 1（更大模型、更高 benchmark）。**赢的团队投资 layer 3 和 4——更好的环境、更好的错误恢复、更好的权限、更好的上下文管理、更好的协调。**

源码就在那。模式有文档。决策可读。

> **Build the harness. Then build the infrastructure around it.**

## See Also

- [Agent 如何避免无限递归](avoiding-infinite-recursion.md) —— 错误恢复 + 上限保险丝是无限递归的解药；本文的"5 阶段循环 + 错误恢复 first-class state"是 production 级版本
- [Reactive vs. Proactive AI Agents：本质区别与选型](reactive-vs-proactive-ai-agents.md) —— harness 设计回答的是"agent 该怎么跑"，本文给的是 proactive agent 上生产时漏掉的四层
- [OpenClaw Plugin API: Extension Points and Deliberate Omissions](../openclaw/plugin-api.md) —— "composition over modification"在另一个生态的体现：扩展点设计哲学相通
