# Prompt Caching：原理、用法和坑

> Sources: apidog blog, Unknown（飞书剪存 2025-09-10）
> Raw: [What is Prompt Caching? Best Practices Explained](../../raw/ai-agents/prompt-caching-best-practices.md)

## Overview

Prompt caching 解决一个浪费：**同一段 prompt 前缀被反复处理**。一个 agent 每次调用都要带上几千 token 的 system prompt + 工具定义 + few-shot 例子，模型每次都从头算一遍，浪费时间也浪费钱。Caching 让 LLM 把这段静态前缀的中间状态存下来，下次见到一模一样的前缀就直接"快进"，只算后面变化的那部分。

代价是这缓存非常死板：**前缀里多一个空格就失效**。所以收益取决于你能不能把 prompt 切成"前面绝对稳定、后面随用户变"两段。

## 它在做什么——一个比方

把 LLM 想成一个口译员。每次你跟他说话，他都要先把"工作守则"（你是谁、你能用哪些工具、有哪些例子）听一遍，然后才能听你今天的问题。Prompt caching 就是把"工作守则听完后的脑内状态"录下来——下次同一个口译员同一份守则，他直接从脑内状态恢复，跳过"再听一遍守则"。

哪段算"工作守则"？由你在 API 里画一刀决定（Anthropic 是 `cache_control: {type: "ephemeral"}` 标记）。从 prompt 开头到这一刀之间的所有内容，构成 cache key——模型对这段内容做哈希，下次完全匹配才命中。

## Anthropic 的 cache_control：怎么用

prompt 在 Anthropic API 里按固定顺序排：`tools` → `system` → `messages`。你可以在 system block、某条 message、甚至某个 tool 定义上加 `cache_control` 标记。**单次请求最多放 4 个 breakpoint**，模型会取最长能命中的那段当 cache。

最常见的两种用法：

**1. 缓存 system prompt（最简单的开始）**

```python
client.messages.create(
    model="claude-3-5-sonnet-20240620",
    system=[{
        "type": "text",
        "text": "（很长的 system prompt）",
        "cache_control": {"type": "ephemeral"}
    }],
    messages=[{"role": "user", "content": "今天的问题"}]
)
```

第一次调用：`cache_creation_input_tokens=N`（写入），`cache_read_input_tokens=0`。
5 分钟内再调一次同样的 system prompt：`cache_creation_input_tokens=0`，`cache_read_input_tokens=N`（命中）。

**2. 增量缓存对话历史**

把 `cache_control` 放在"上一轮 assistant 回复"上，下一轮就能命中前面整段历史：

```python
# Turn 2
messages=[
    {"role": "user", "content": "Hello"},                    # 上一轮，已被缓存
    {"role": "assistant", "content": "Hi there",
     "cache_control": {"type": "ephemeral"}},                # 缓存到这里
    {"role": "user", "content": "新问题"},                    # 后续动态部分
]
```

每多一轮把标记往后移一格，对话越长越省。

## 算账：什么时候真的值

Anthropic 的价格结构（写入贵 25%，读取便宜 90%）：

| 类型 | 相对基础输入价 |
|------|---------------|
| 普通 input token | 1× |
| Cache write | 1.25× |
| Cache read | 0.1× |

简单算个账：一段 10000 token 的 system prompt，base 价 $3/MTok：

- 不用 cache，调 10 次：`10 × 10000 × $3/M = $0.30`
- 用 cache，1 次写 + 9 次读：`(10000 × 1.25 + 9 × 10000 × 0.1) × $3/M = (12500 + 9000) × $3/M ≈ $0.064`
- **省 ~78%**。复用次数越多，越接近"省 90%"的理论值。

反过来，如果只调 2 次：`(12500 + 1 × 10000 × 0.1) × $3/M = $0.041`，对比 `2 × 10000 × $3/M = $0.06`，只省 30%。**调用次数越少，cache write 的 25% 溢价越扎眼**。所以缓存"调一次就丢"的前缀基本是亏的。

## 那些非常容易踩的坑

**1. 多一个空格就失效。** Cache key 是哈希，不是模糊匹配。`"You are an assistant."` 和 `"You are an assistant. "`（结尾多一个空格）是两份缓存。所以 prompt 一定要由稳定的代码生成，不要手拼字符串、不要带时间戳、不要把"今天日期"放进 system prompt。

**2. 最小长度门槛。** 短 prompt 不让缓存：

- Sonnet / Opus（3.5 / 3.7）：≥ 1024 token
- Haiku：≥ 2048 token

不到这个长度，标了 `cache_control` 也不缓存——你以为在省钱，其实没启用。

**3. TTL 只有 5 分钟（默认 ephemeral）。** 命中一次刷新一次，但**完全不动**就过期。低频调用根本攒不出复用，反而每次都付 25% 写入溢价。判断标准：你的 prefix 平均多久会被命中一次？>5 分钟基本无效。

**4. 改工具定义就废。** 不光改文本，改 `tool_choice`、加减一个 tool、甚至工具顺序变了——都会让 message 段的缓存失效，因为前缀里的 `tools` 段变了。Agent 框架动态拼工具列表的时候特别容易踩。

**5. 并发首次调用集体 miss。** 你同时发 100 个相同前缀的请求，第一个还没写完，剩下 99 个全是 cache miss、全部走 cache write。结果一次性付 100 份 25% 溢价。**冷启动场景下要么先打一次预热，要么等第一次返回再并发**。

**6. extended thinking 等参数会顺手把 message cache 干掉。** Anthropic 文档里写明，开关 thinking 设置可能让 message 缓存作废。Agent 跑着跑着切换模式，缓存就没了——你看 `cache_read_input_tokens` 一直是 0 还以为标记错了，其实是另一个参数把它废了。

## 监控：看两个数就够

每次调用 response 里有四个数：

- `input_tokens`：动态后缀（这次新算的）
- `cache_creation_input_tokens`：这次写入了多少
- `cache_read_input_tokens`：这次命中了多少
- `output_tokens`：生成的

健康指标是 **read / (read + creation) 的长期比值**——稳定运行的应用应该 >80%。如果一直在写、很少读，要么前缀有变量在跳、要么 TTL 内调用密度不够、要么没过最小长度门槛。

## 用得最值的几类场景

按"前缀稳定性 × 复用次数"两维看，下面这些最值：

- **Document Q&A / RAG**：长文档塞前缀，问题塞后缀。文档不变、问题变。
- **Few-shot prompt**：几十个示例放前面，新输入放后面。
- **Agent 系统**：庞大的 system prompt + 工具定义全是静态的，每个 step 调一次，复用极高。
- **多轮对话**：system 永远不变，对话历史可以增量缓存。

不值得的：用户每次问的问题各不相同、但又没有公共上下文（基本是普通单轮问答）；超短 prompt（连最小长度都不到）。

## 一句话总结

Prompt caching 不是黑魔法，就是把"反复处理同一段静态文本"这个浪费砍掉。能不能省钱看两件事：**前缀够不够稳**（一个空格之差都不行）和**复用够不够频**（5 分钟内能不能再用一次）。Agent 类系统天然占满这两条，所以受益最大。

## See Also

- [Claude Code Harness 架构拆解：四层模型 + 生产级 agent 工程经验](claude-code-harness-architecture.md) —— 真实生产 agent 怎么把"为缓存设计 prompt"落地：`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 标记、~80% 全局可缓存段、"决定 $0.02/session 还是 $0.20/session"
