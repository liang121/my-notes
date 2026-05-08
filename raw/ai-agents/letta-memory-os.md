# Letta - memory os

> Source: 飞书剪存 PDF（用户上传，clawos.chat 附件链接），整理自 Letta 官方文档与 MemGPT 论文
> Collected: 2026-05-08
> Published: Unknown

## Letta Memory 架构全解

Letta 基于 MemGPT 论文，把 LLM 的记忆分为 **in-context（上下文内）** 和 **out-of-context（上下文外）** 两大类：

```
Context Window (RAM)
  ├─ Core Memory (Memory Blocks)
  │    - human
  │    - persona
  │    - summary
  └─ Message Buffer (最近的对话消息)

         ↑↓ (通过 Tools 读写)

External Storage (Disk)
  ├─ Archival Memory (Vector DB)
  │    - 语义搜索
  │    - 知识库/事实
  ├─ Recall Memory (对话历史 DB)
  │    - 时间搜索
  │    - 文本搜索
  └─ Letta Filesystem (文件系统 - Folders & Files)
       - PDF/TXT/MD/JSON 等文档
       - 文件浏览/搜索
```

## 项目使用状态标记

| 功能 | 状态 | 说明 |
|---|---|---|
| Core Memory (human/persona/summary) | ✅ 使用中 | letta-memory.service.ts |
| Memory Tools (replace/insert/rethink) | ✅ 使用中 | Agent 自主调用 |
| Archival Memory | ✅ 使用中 | 通过 tools 访问 |
| Recall Memory | ✅ 使用中 | conversation_search |
| Letta Filesystem | ❌ 未使用 | 需手动上传文件 |
| Sleep-Time Compute | ❌ 未使用 | 后台记忆管理 |
| conversation_search_date | ❌ 未使用 | 按日期搜索 |
| memory_finish_edits | ❌ 未使用 | Sleep-time 专用 |

## 五种 Memory 类型详解

| Memory 类型 | 位置 | 特点 | 用途 | 项目状态 |
|---|---|---|---|---|
| Message Buffer | In-Context | 最近的对话消息，始终可见 | 当前对话上下文 | ✅ 自动 |
| Core Memory | In-Context | 结构化的 memory blocks，始终可见，可编辑 | 用户画像、Agent 人设、摘要 | ✅ 使用中 |
| Archival Memory | Out-of-Context | Vector DB 存储，语义搜索 | 长期知识、事实、外部数据 | ✅ 使用中 |
| Recall Memory | Out-of-Context | 完整对话历史，时间/文本搜索 | 历史对话检索 | ✅ 使用中 |
| Letta Filesystem | Out-of-Context | 文件/文件夹结构，支持文档 | 外部文档集（报告、论文） | ❌ 未使用 |

## 各 Memory 详细说明

### 1. Message Buffer（消息缓冲）
- **什么**：最近的对话消息
- **特点**：始终在 context window 中，Agent 直接可见
- **限制**：受 context window 大小限制
- **压缩**：当满了时，旧消息被总结后移到 Recall Memory

### 2. Core Memory（核心记忆） 使用中
- **什么**：结构化的 memory blocks（如 human、persona、summary）
- **特点**：
  - 始终 pinned 在 context window（始终可见）
  - Agent 可以主动编辑（通过 `memory_replace`、`memory_insert` 等工具）
  - 每个 block 可配置字符限制（默认 2000，项目中 human/summary 设为 5000）
- **用途**：存储"执行摘要"级别的关键信息
- **项目配置**：

```typescript
// letta-memory.service.ts
DEFAULT_MEMORY_BLOCKS = [
  { label: 'human',   limit: 5000 },  // 用户信息
  { label: 'persona', limit: 2000 },  // Agent 人设
  { label: 'summary', limit: 5000 },  // 对话摘要
]
```

#### 默认 Memory Blocks 详解

**Human Block（用户画像）**
- 定义：存储正在对话的**用户**的信息
- 官方描述："Stores key details about the person you are conversing with, allowing for more personalized and friend-like conversation."
- 典型内容：用户姓名、职业背景、沟通偏好、个人喜好、重要事实
- 更新频率：**频繁更新** - Agent 在对话中不断学习用户信息并更新
- 示例：`"用户叫张三，是产品经理，喜欢简洁的回答，对坚果过敏"`

**Persona Block（Agent 人设）**
- 定义：存储 **Agent 自己**的身份、性格和行为准则
- 官方描述："Stores details about your current persona, guiding how you behave and respond. This helps you to maintain consistency and personality in your interactions."
- 典型内容：Agent 的名字、角色定位、性格特点、行为偏好
- 更新频率：较少更新，但 Agent **可以**编辑自己的 persona 以保持一致性
- 示例：`"I am a friendly assistant. I prefer concise answers and enjoy helping with technical questions."`
- 注意：官方推荐用**第一人称**（"I am..."）而非第二人称（"You are..."）

**Human vs Persona 对比**

| | Human Block | Persona Block |
|---|---|---|
| 描述对象 | 用户（对话对象） | Agent（自己） |
| 内容 | 用户的信息、偏好、事实 | Agent 的身份、性格、行为准则 |
| 更新频率 | 频繁（随着了解用户） | 较少（保持一致性） |
| 目的 | 个性化对话 | 保持人格一致 |

### 3. Archival Memory（归档记忆） 使用中
- **什么**：Vector DB 中的长期存储
- **特点**：
  - Out-of-context（不占用 context window）
  - 语义搜索（通过 embedding 相似度）
  - 可存储大量信息（百万级条目）
- **用途**：详细事实、外部知识、长期记忆
- **工具**：`archival_memory_search`、`archival_memory_insert`

### 4. Recall Memory（回忆记忆） 使用中
- **什么**：完整的对话历史日志
- **特点**：
  - 自动持久化到数据库
  - 支持文本搜索和日期搜索
- **用途**：检索过去的对话
- **工具**：
  - `conversation_search` - 文本搜索 ✅ 使用中
  - `conversation_search_date` - 日期搜索 ❌ 未使用

### 5. Letta Filesystem（文件系统） 未使用
- **什么**：文件/文件夹管理系统
- **特点**：
  - 支持 .pdf, .txt, .md, .json 等格式
  - 文件上传后自动分块 + embedding
  - 保持文件结构，可浏览/搜索
- **用途**：让 Agent 访问外部文档集（报告、论文、记录）
- **工具**：
  - `open_file` - 打开/读取特定文件位置
  - `grep_file` - 正则表达式搜索
  - `search_file` - 语义搜索
- **注意**：需要**手动上传文件**，不是自动记忆

## Memory Tools 完整列表

### Core Memory 工具

| 工具 | 用途 | 项目状态 |
|---|---|---|
| `memory_replace` | 替换 block 中的内容 | ✅ 使用中 |
| `memory_insert` | 在 block 中插入新内容 | ✅ 使用中 |
| `memory_rethink` | 重新组织/精简整个 block | ✅ 使用中 |
| `memory_finish_edits` | 完成批量编辑（Sleep-time 专用） | ❌ 未使用 |

### Archival Memory 工具

| 工具 | 用途 | 项目状态 |
|---|---|---|
| `archival_memory_search` | 语义搜索归档记忆 | ✅ 使用中 |
| `archival_memory_insert` | 写入新内容到归档记忆 | ✅ 使用中 |

### Recall Memory 工具

| 工具 | 用途 | 项目状态 |
|---|---|---|
| `conversation_search` | 文本搜索对话历史 | ✅ 使用中 |
| `conversation_search_date` | 按日期搜索对话历史 | ❌ 未使用 |

### Filesystem 工具 未使用

| 工具 | 用途 |
|---|---|
| `open_file` | 打开/读取特定文件位置 |
| `grep_file` | 正则表达式搜索文件 |
| `search_file` | 语义搜索文件内容 |

### Legacy 工具（已废弃）

| 工具 | 替代方案 |
|---|---|
| `core_memory_append` | → `memory_insert` |
| `core_memory_replace` | → `memory_replace` |

## Query 进来时的工作流程

```
用户 Query: "我之前跟你说过我喜欢什么运动来着？"
    ▼
1. Agent 收到 Query
   - Message Buffer 中有最近对话
   - Core Memory 中有用户画像
    ▼
2. Agent 自主决定是否需要搜索
   LLM 分析：这是个历史问题，需要检索
    ▼
3. Agent 调用搜索工具
   - conversation_search("运动")          ← 搜 Recall Memory
   - archival_memory_search("运动偏好")    ← 搜 Archival Memory
    ▼
4. 搜索结果注入 Context
   找到："用户说喜欢游泳和跑步"
    ▼
5. Agent 生成回复
   "你之前说喜欢游泳和跑步！"
    ▼
6. Agent 可能更新 Core Memory
   memory_replace: 更新 human block
   "用户喜欢游泳和跑步"
```

## 关键点：Agent 自主管理

Letta 的核心理念是 **Agent 自己决定何时读写记忆**：

1. Agent 有一套 memory tools（见上表）
2. Agent 根据对话内容**自主决定**：
   - 是否需要搜索外部记忆
   - 是否需要更新 core memory
   - 是否需要把信息存入 archival memory

## Sleep-Time Compute（睡眠时计算） 未使用

### 什么是 Sleep-Time Agent

Sleep-Time 是 Letta 的**异步记忆管理机制**：

```
Multi-Agent Group
├─ Primary Agent (主 Agent)        ←→ 共享 Memory Blocks
│    - 处理对话                       ↑
│    - 快速响应                       ↓
└─ Sleep-Time Agent (后台运行)
     - 异步更新记忆
     - 重组/优化记忆
```

### 特点
- **不阻塞响应**：记忆更新在后台异步进行
- **更高质量**：有更多时间重组/优化记忆
- **主动学习**：在空闲时间处理和整合信息

### 配置

```python
client.agents.create(
    enable_sleeptime=True,        # 启用 sleep-time
    sleeptime_agent_frequency=5,  # 每 5 步触发一次（默认）
)
```

### Sleep-Time Agent 专用工具
- `memory_rethink` - 重新组织记忆
- `memory_finish_edits` - 完成批量编辑

### 用例
- 处理大量对话历史
- 分析上传的文档并更新记忆
- 在空闲时间优化记忆结构

## Context Engineering（上下文工程）

### Context Window 组装顺序

```
Context Window
1. System Prompt
   └─ Agent 的基础指令
2. Core Memory Blocks
   └─ human / persona / summary / 自定义 blocks
3. Tool Definitions
   └─ 可用工具的 JSON Schema
4. Message Buffer
   └─ 最近的对话消息
5. [Tool Results / Search Results]
   └─ 工具调用返回的结果（动态注入）
```

### 配置参数

| 参数 | 说明 | 默认值 |
|---|---|---|
| `include_base_tools` | 是否包含默认 memory tools | true |
| `context_window_limit` | context window token 限制 | 模型最大值 |

## 高级功能

### 1. 自定义 Memory Blocks

除了默认的 human/persona，可以添加自定义 blocks：

```python
memory_blocks=[
    {"label": "human",       "value": "...", "limit": 5000},
    {"label": "persona",     "value": "...", "limit": 2000},
    {"label": "goals",       "value": "用户当前目标...", "limit": 2000},  # 自定义
    {"label": "preferences", "value": "用户偏好...",   "limit": 2000},  # 自定义
]
```

### 2. API 直接操作 Memory

除了 Agent 自主管理，可以通过 API 直接修改：

```python
# 读取所有 blocks
for block in client.agents.blocks.list(agent_id):
    print(f"{block.label}: {block.value}")

# 更新特定 block
client.agents.blocks.update(agent_id, block_id, value="new content")
```

### 3. 多 Agent 共享 Memory

多个 Agent 可以共享同一个 Memory Block：

```
Agent A ──┐
          ├─→ 共享的 Memory Block
Agent B ──┘
```

用于多 Agent 协作场景。

### 4. 禁用默认工具

创建 Agent 时可以禁用默认 memory tools：

```python
client.agents.create(
    include_base_tools=False,  # 禁用默认工具
    tools=["custom_tool_1", "custom_tool_2"],  # 只用自定义工具
)
```

## 最佳实践

> **Core Memory 存"执行摘要"，Archival Memory 存"完整细节"**

- Core Memory: `"用户对坚果过敏"`（始终可见）
- Archival Memory: `"2024-01-15 用户提到对花生、杏仁、核桃都过敏，症状是皮疹"`（按需检索）

> **Filesystem 用于外部文档，不是自动记忆**

- 自动记忆对话信息 → 用 Archival Memory
- 让 Agent 访问现有文档 → 用 Filesystem（需手动上传）

## 配置参数

### `context_window_limit`
- **作用**：限制 Agent 的 context window 大小（token 数）
- **当设置后会发生什么**：
  - Agent 的上下文会被限制在这个 token 数内
  - 当消息累积接近上限时，Letta 会自动触发**压缩/摘要**：
    - 旧消息被总结成 recursive summary
    - 旧消息被移到 archival memory（可通过搜索工具检索）
    - 保留最近的消息保持连续性

### `message_buffer_autoclear`
- **作用**：每次对话后自动清空消息历史
- **设置 `true` 后**：
  - Agent **不记住**之前的对话消息
  - 但仍保留：
    - Core memory blocks（human、persona、summary）
    - Archival memory
    - Recall memory
- **官方建议**：除非有高级用例，否则不推荐开启

### `max_message_buffer_length` / `min_message_buffer_length`
- **作用**：控制保留多少条消息（而不是 token 数）

| 参数 | 说明 |
|---|---|
| `max_message_buffer_length` | 消息缓冲区最大消息数，超过后触发压缩 |
| `min_message_buffer_length` | 压缩后至少保留的消息数，确保对话连续性 |

例如：设置 max=50, min=10
- 当消息数达到 50 条时，触发压缩
- 压缩后保留最近 10 条消息 + 生成的摘要

## 参考资料

- Agent Memory | Letta
- Understanding memory management | letta-sdk
- Context engineering | Letta Docs
- Sleep-time agents | Letta Docs
- Letta Filesystem | Letta Docs
- Core memory | letta-sdk
- Archival memory | letta-sdk
- MemGPT | Letta Docs
- Sleep-time Compute | Letta Blog
- Introducing Letta Filesystem | Letta Blog
