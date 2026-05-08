# OpenClaw Plugin API 能力清单

> Source: openclaw/src/plugins/types.ts (OpenClawPluginApi 类型，约 1680 行起)
> Collected: 2026-04-24
> Published: Unknown
> Note: 本地代码扫描结果，非外部发布材料。用途：给插件开发者的能力面参考，clawos extension 是在此基础上再包一层"用 Next.js 写插件"的糖。

---

## 一、`register*` 类：注册插件功能（主菜）

每调一次 `registerXxx` 就相当于往 gateway 里装一个新能力。

### 1. AI / 模型能力

| API | 用途 |
|---|---|
| `registerProvider` | 文本/推理模型 provider（OpenAI、Anthropic、Ollama、DeepSeek 等 40+ 内建都走这个） |
| `registerSpeechProvider` | 语音合成 TTS |
| `registerMediaUnderstandingProvider` | 多模态理解（图片 / 视频 → 文字） |
| `registerImageGenerationProvider` | 图像生成 |
| `registerWebSearchProvider` | Web 搜索 |
| `registerCliBackend` | 文本 CLI backend（本地 CLI 跑 agent 时选的引擎） |
| `registerContextEngine` | context engine（管上下文窗口 / 压缩策略，独占槽位） |
| `registerMemoryRuntime` | 记忆插件的活动 runtime 适配器（独占槽位） |
| `registerMemoryPromptSection` | 记忆插件的 system prompt 片段 builder（独占槽位） |
| `registerMemoryFlushPlan` | 记忆插件的 pre-compaction flush plan resolver（独占槽位） |
| `registerMemoryEmbeddingProvider` | 记忆的 embedding provider 适配器（多个可共存） |

### 2. 消息通道

| API | 用途 |
|---|---|
| `registerChannel` | IM 通道（Slack / Discord / Matrix / WhatsApp / Telegram / Signal / 飞书 / Line…） |
| `registerInteractiveHandler` | 交互式回复处理器（approvals 等） |

### 3. Agent 工具 & 命令

| API | 用途 |
|---|---|
| `registerTool` | 给 agent 挂一个新工具，agent 在对话里能调用 |
| `registerCommand` | 自定义斜杠命令，**不走 LLM**，纯脚本式，优先级高于内置命令和 agent |

### 4. HTTP / RPC 扩展（clawos extension 主要走这条）

| API | 用途 |
|---|---|
| `registerHttpRoute` | 把 HTTP 路由挂到 gateway 18789 端口的 server 上。clawos extension 的 host-bridge、SSE hub、export endpoint 都是用这个 |
| `registerGatewayMethod` | 给 gateway WebSocket RPC 加一个新方法（`client.call('xxx.yyy')` 那种）；支持 `scope: OperatorScope` |

### 5. CLI / 服务

| API | 用途 |
|---|---|
| `registerCli` | 注册 `openclaw xxx` 子命令，支持 `commands` / `descriptors` 做懒加载 |
| `registerService` | 注册后台服务（常驻任务） |

---

## 二、Hook：生命周期钩子（`api.on(hookName, handler)` / `registerHook`）

非侵入式集成的标准手段。共 **28 个钩子**，能"拦截并改写" LLM 输入输出、工具调用、消息派发，是做 observability / 审计 / 加护栏 的主要途径。

| 分类 | 钩子名 |
|---|---|
| **启停** | `gateway_start` / `gateway_stop` / `before_install` |
| **Session** | `session_start` / `session_end` / `before_reset` |
| **Agent 运行** | `before_agent_start` / `agent_end` / `before_model_resolve` / `before_prompt_build` |
| **LLM 输入输出** | `llm_input` / `llm_output` |
| **工具调用** | `before_tool_call` / `after_tool_call` / `tool_result_persist` |
| **消息流** | `message_received` / `message_sending` / `message_sent` / `inbound_claim` / `before_dispatch` / `before_message_write` |
| **Subagent** | `subagent_spawning` / `subagent_delivery_target` / `subagent_spawned` / `subagent_ended` |
| **压缩** | `before_compaction` / `after_compaction` |

其中 `before_prompt_build` 与 `before_agent_start` 属于 prompt injection 钩子（见 `PROMPT_INJECTION_HOOK_NAMES`）。

---

## 三、运行时能力（被动使用）

```ts
api.runtime      // PluginRuntime：spawn subagent、读运行时状态等
api.logger       // 写 gateway 标准日志
api.config       // OpenClawConfig 全局配置
api.resolvePath  // 把 ~/xxx、相对路径展开成绝对路径
api.pluginConfig // 插件自身配置
```

---

## 四、事件订阅

| API | 用途 |
|---|---|
| `onConversationBindingResolved(handler)` | 会话绑定解析完成时回调 |

---

## 五、插件元信息（自述）

```ts
api.id
api.name
api.version
api.description
api.source
api.rootDir
api.registrationMode
```

---

## 六、**没有** 的能力（常见误区）

以下能力 openclaw plugin API **不提供**，要自己实现：

- ❌ **Cron / 定时调度**：没有 `registerCronJob` / `registerSchedule`，整个 types.ts 里只有一处 docstring 提到 `cron` 作为 agent run origin 的枚举值。需要定时就自己 `setTimeout + cron-parser`（clawos extension SDK 的 `jobs.ts` 就是这么干的）
- ❌ **任务持久化**：没有内建 job queue / durable task
- ❌ **数据库抽象**：插件要自己管存储

### 为什么不提供 cron（设计判断）

- openclaw gateway 定位是"协议路由 + 插件容器"，不是 workflow 引擎
- cron 是纯时序问题，几十行代码可解，加到 gateway 要处理持久化、crash 补跑、多插件共享调度器等复杂问题，性价比低
- 目前 extension 若声明 cron 又被 idle-stop，到点不会触发——这是**已知的架构约束**

---

## 七、clawos extension 的定位（对照参考）

clawos extension 本身就是一个 openclaw 插件（`openclaw-plugin-extensions/index.ts`），主要用到：

1. `registerHttpRoute`：挂 `/api/extensions/host-bridge`、SSE、export 下载
2. `registerGatewayMethod`：挂 `extension.*` 系列 RPC（install / start / stop / list 等）
3. `on('gateway_stop', …)`：gateway 退出时清理子进程

然后它**在这之上再对外开一套"二级插件 API"**（通过 `host-bridge` + spawn Next.js 进程），让 extension 开发者用 Next.js 的姿势写插件，不用直接碰庞大的 `OpenClawPluginApi`。

---

## 一句话总结

openclaw 给插件开发者的能力可以概括为：
**"在 gateway 进程里注册任何扩展点（模型 / 通道 / 工具 / 命令 / HTTP / RPC / CLI / 服务），或者以 hook 方式介入 agent 生命周期的任意阶段"**。

Cron、持久化任务、数据库这些由插件自己负责。
