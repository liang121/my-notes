# OpenClaw Plugin API: Extension Points and Deliberate Omissions

> Sources: openclaw codebase scan (src/plugins/types.ts), 2026-04-24
> Raw: [OpenClaw Plugin API 能力清单](../../raw/openclaw/openclaw-plugin-api.md)

## Overview

OpenClaw gateway 的定位是 **"协议路由 + 插件容器"**，不是 workflow 引擎。`OpenClawPluginApi` 接口给插件开发者四种切入方式：**注册扩展点**、**挂 hook**、**消费 runtime 能力**、**订阅事件**。配合一份明确的"不做什么"清单（cron / 持久化 / DB）——这些不是遗漏，是设计判断。clawos extension 是一个递归示例：它本身是一个用 plugin API 实现的插件，并在其上再开了一套面向 Next.js 应用的二级 API。

## 四种切入方式

### 1. `register*`：声明新能力

按"能力轴"分组：

- **AI / 模型**（11 个 register 方法）：text/reasoning provider、TTS、多模态理解、图像生成、Web 搜索、CLI backend，以及一批为 memory plugin 服务的"独占槽位" register（context engine、memory runtime、prompt section、flush plan、embedding provider）。
- **消息通道**：`registerChannel`（IM 通道，Slack/Discord/Telegram 等）、`registerInteractiveHandler`（approval 等交互回复）。
- **Agent 工具与命令**：`registerTool`（agent 调用）、`registerCommand`（**不走 LLM** 的斜杠命令，优先级高于 agent）。
- **HTTP / RPC**：`registerHttpRoute`（挂到 18789 gateway server）、`registerGatewayMethod`（WebSocket RPC，支持 OperatorScope）。clawos extension 主要走这条。
- **CLI / 服务**：`registerCli`（`openclaw xxx` 子命令，懒加载）、`registerService`（常驻后台任务）。

### 2. Hook：拦截生命周期

28 个钩子，覆盖：启停 / Session / Agent 运行 / LLM 输入输出 / 工具调用 / 消息流（in / out / dispatch / inbound claim）/ Subagent / 压缩。

特殊类：`before_prompt_build` 和 `before_agent_start` 是 **prompt injection 钩子**（在 `PROMPT_INJECTION_HOOK_NAMES` 里），允许在 prompt 构造完成前注入内容。

Hook 是做 observability / 审计 / 安全护栏 的主要手段——**非侵入**集成路径。

### 3. Runtime 能力（被动消费）

`api.runtime`（spawn subagent、读运行时状态）、`api.logger`、`api.config`、`api.resolvePath`、`api.pluginConfig`——加上一组只读 metadata（id / name / version / source / rootDir / registrationMode）。

### 4. 事件订阅

目前只有 `onConversationBindingResolved`——会话绑定解析完成时回调。

完整 API 表参见 [raw 源文件](../../raw/openclaw/openclaw-plugin-api.md)。

## 关键设计判断：刻意不做的能力

API surface 的边界本身就是设计决策。三件 openclaw plugin API **不提供**：

| 缺失能力 | 替代方案 | 理由 |
|---|---|---|
| Cron / 定时调度 | 自己 `setTimeout + cron-parser`（clawos SDK 的 `jobs.ts` 即如此） | gateway 是协议路由不是 workflow engine；cron 加进来要处理持久化、crash 补跑、共享调度器等复杂面 |
| 任务持久化 / job queue | 插件自管 | 同上 |
| 数据库抽象 | 插件自管存储 | 同上 |

**已知架构约束**：extension 若声明 cron 又被 gateway 的 idle-stop 关掉，到点不会触发。这是当前架构在"轻量 gateway + 插件自治"取舍下的代价。

设计精神：**给一个最小内核 + 完整钩子面**，复杂能力让上层（如 clawos extension）按需要自己叠。

## 元模式：插件之上的插件 API

clawos extension 本身是 openclaw 插件 (`openclaw-plugin-extensions/index.ts`)，对内只用三件事：

1. `registerHttpRoute` — host-bridge、SSE、export 下载
2. `registerGatewayMethod` — `extension.*` 系列 RPC（install/start/stop/list）
3. `on('gateway_stop', …)` — 清理子进程

然后**对外开了一套二级插件 API**——通过 host-bridge + spawn Next.js 进程，extension 开发者用 Next.js 写插件，不直接碰庞大的 `OpenClawPluginApi`。

这是一个值得记下的模式：**底层 API 提供完整性 + 通用性，上层 wrapper 提供易用性 + 受限性**。openclaw 没有把"易用"塞进 plugin API 自己，而是允许上层针对具体场景做受限糖。

## 一句话总结

> 在 gateway 进程里注册任何扩展点（模型 / 通道 / 工具 / 命令 / HTTP / RPC / CLI / 服务），或者以 hook 方式介入 agent 生命周期的任意阶段。Cron、持久化、DB 由插件自己负责。
