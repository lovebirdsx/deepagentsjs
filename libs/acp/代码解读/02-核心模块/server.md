# server.ts — 核心服务器

**文件路径：** `src/server.ts`  
**代码行数：** ~1419 行  
**核心类：** `DeepAgentsServer`

---

## 概述

`DeepAgentsServer` 是整个包的核心，负责：
1. 实现 ACP 协议的所有服务端方法
2. 管理多个 Agent 实例和会话状态
3. 驱动 LangGraph Agent 流式运行
4. 把 Agent 输出实时推送到 IDE

---

## 类结构

```typescript
export class DeepAgentsServer {
  // 连接与状态
  private connection: AgentSideConnection | null        // ACP 连接对象
  private agents: Map<string, DeepAgent>                // 已创建的 Agent 实例
  private agentConfigs: Map<string, DeepAgentConfig>    // Agent 配置
  private sessions: Map<string, SessionState>           // 活跃会话
  private checkpointer: MemorySaver                     // LangGraph 检查点（持久化）
  private clientCapabilities: ACPCapabilities           // IDE 客户端能力标记
  private currentPromptAbortController: AbortController // 当前请求的取消控制器
  private acpBackends: Map<string, ACPFilesystemBackend>// ACP 文件后端实例

  // 配置
  private readonly serverName: string
  private readonly serverVersion: string
  private readonly debug: boolean
  private readonly workspaceRoot: string
  private readonly authMethods: ACPAuthMethod[]
  private readonly logger: Logger
}
```

---

## 构造函数

```typescript
constructor(options: DeepAgentsServerOptions) {
  // 1. 初始化基本配置（serverName、workspaceRoot、authMethods 等）
  // 2. 创建 Logger 实例
  // 3. 创建共享 MemorySaver（用于 LangGraph 会话持久化）
  // 4. 将 agents 配置数组注册到 agentConfigs Map
}
```

**关键设计**：`checkpointer` 是共享的，所有 Agent 实例和会话共用同一个 `MemorySaver`，通过 `thread_id` 区分不同会话的状态。

---

## 启动流程：`start()`

```typescript
async start(): Promise<void> {
  // 1. 注册信号处理器（SIGINT/SIGTERM → 优雅关闭）
  // 2. 创建 ReadableStream（包装 process.stdin）
  // 3. 创建 WritableStream（包装 process.stdout）
  // 4. ndJsonStream(output, input) → 创建 NDJSON 传输层
  // 5. new AgentSideConnection(agentHandler, stream) → 建立 ACP 连接
  // 6. await connection.closed → 阻塞直到连接断开
}
```

**注意**：`ndJsonStream` 的参数顺序是 `(output, input)`，即先写出、再读入。

---

## ACP 方法处理器

通过 `createAgentHandler(conn)` 返回一个 `Agent` 对象，包含 7 个方法：

```typescript
{
  initialize,      // 握手，协商能力
  authenticate,    // 认证（当前为 no-op）
  newSession,      // 创建会话
  loadSession,     // 加载/恢复会话
  prompt,          // 处理用户输入（主入口）
  cancel,          // 取消当前操作
  setSessionMode,  // 切换模式（agent/plan/ask）
}
```

### `handleInitialize(params)`

接收客户端的能力声明，存储到 `this.clientCapabilities`：

```typescript
// 解析 clientCapabilities
const fs = clientCaps.fs  // { readTextFile, writeTextFile }
this.clientCapabilities = {
  fsReadTextFile: fs?.readTextFile ?? false,
  fsWriteTextFile: fs?.writeTextFile ?? false,
  terminal: clientCaps.terminal !== undefined,
}

// 返回服务器能力
return {
  protocolVersion,
  agentInfo: { name, version },
  agentCapabilities: {
    loadSession: true,
    promptCapabilities: { image: true, embeddedContext: true },
    sessionCapabilities: { modes: true, commands: true },
  },
  authMethods,
}
```

### `handleNewSession(params, conn)`

```typescript
// 1. 生成 sessionId（generateSessionId()）和 threadId（crypto.randomUUID()）
// 2. 从 configOptions.agent 选择 Agent 名称（默认取第一个）
// 3. 创建 SessionState 存入 sessions Map
// 4. 懒加载 createAgent()（第一次用到时才创建）
// 5. 异步发送 available_commands_update 通知（内置命令 + 自定义命令）
// 6. 返回 { sessionId, modes }
```

### `handlePrompt(params, conn)` — 主入口

```typescript
async handlePrompt(params, conn) {
  // 1. 验证 sessionId 和 agentName 存在
  // 2. 创建 AbortController（用于取消）
  // 3. 提取 prompt: ContentBlock[]
  // 4. 检查 slash command → 匹配则直接返回
  // 5. acpPromptToHumanMessage(prompt) → 转为 LangChain HumanMessage
  // 6. streamAgentResponse(session, agent, humanMessage, conn)
  // 7. return { stopReason }
}
```

---

## 核心流式循环：`streamAgentResponse()`

这是最复杂的方法（约 130 行），驱动 LangGraph Agent 运行并把结果流式推送：

```typescript
async streamAgentResponse(session, agent, humanMessage, conn) {
  const config = {
    configurable: { thread_id: session.threadId },
    signal: abortController.signal,  // 取消信号
  }

  const stream = await agent.stream({ messages: [humanMessage] }, config)

  for await (const event of stream) {
    // 取消检查
    if (abortController.signal.aborted) {
      // 标记所有活跃 toolCall 为 cancelled
      return "cancelled"
    }

    // 从事件中提取 messages（LangGraph 事件结构多样）
    let messages = []
    if (event.messages)                    messages = event.messages
    else if (event.model_request?.messages) messages = event.model_request.messages
    else if (event.tools?.messages)         messages = event.tools.messages

    for (const message of messages) {
      if (AIMessage.isInstance(message))   await handleAIMessage(...)
      if (ToolMessage.isInstance(message)) await handleToolMessage(...)
    }

    // 处理 todos（计划列表）
    if (event.todos) {
      await sendPlanUpdate(session.id, conn, todosToPlanEntries(event.todos))
    }
  }
  return "end_turn"
}
```

**LangGraph 事件结构**：不同 Node 输出的事件 key 不同（如 `model_request`、`tools`），需要多路径提取 messages。

---

## AI 消息处理：`handleAIMessage()`

```typescript
async handleAIMessage(session, message: AIMessage, activeToolCalls, conn) {
  // 处理文本内容
  if (typeof message.content === "string") {
    await sendMessageChunk(session.id, conn, "agent", contentBlocks)
  }

  // 处理思考内容（extended thinking 模型）
  if (block.type === "thinking") {
    await sendMessageChunk(session.id, conn, "thought", [...])
  }

  // 处理工具调用
  const toolCalls = extractToolCalls(message)
  for (const toolCall of toolCalls) {
    await sendToolCall(session.id, conn, toolCall)  // 通知 tool_call
    toolCall.status = "in_progress"
    await sendToolCallUpdate(session.id, conn, toolCall)  // 更新状态
    activeToolCalls.set(toolCall.id, toolCall)
  }
}
```

---

## 工具结果处理：`handleToolMessage()`

```typescript
async handleToolMessage(session, message: ToolMessage, activeToolCalls, conn) {
  const toolCall = activeToolCalls.get(message.tool_call_id)
  
  // 判断是否为错误
  const isError = message.status === "error" || content.includes("error")
  
  toolCall.status = isError ? "error" : "completed"
  toolCall.result = resultContent
  
  await sendToolCallUpdate(session.id, conn, toolCall)
  activeToolCalls.delete(toolCallId)
}
```

---

## Slash Command 处理

在发给 Agent 之前拦截，匹配则直接返回：

| 命令 | 效果 |
|---|---|
| `/plan` `/agent` `/ask` | 设置 `session.mode`，发送确认消息 |
| `/clear` | 清空 `session.messages`，生成新 `threadId`（等效新会话） |
| `/status` | 返回 agent 名称、模式、模型、skill/memory 数量等 |

---

## 权限请求：`requestToolPermission()`

当 Agent 配置了 `interruptOn` 时，对应工具调用会触发权限弹窗：

```typescript
async requestToolPermission(session, conn, toolCall) {
  // 1. 检查会话缓存的永久决策（allow_always / reject_always）
  const cached = session.permissionDecisions.get(toolCall.name)
  if (cached === "allow_always") return "allow"
  if (cached === "reject_always") return "reject"

  // 2. 向 IDE 发送权限请求（显示对话框）
  const result = await conn.requestPermission({
    sessionId, toolCall,
    options: [allow-once, allow-always, reject-once, reject-always]
  })

  // 3. 缓存 always 类决策
  if (optionId === "allow-always") {
    session.permissionDecisions.set(toolCall.name, "allow_always")
  }
}
```

---

## 会话历史回放：`replaySessionHistory()`

加载会话时（`session/load`），将历史对话推送给 IDE：

```typescript
async replaySessionHistory(session, conn) {
  // 1. 从 LangGraph checkpointer 取历史 messages
  const checkpoint = await checkpointer.getTuple({ thread_id })
  const messages = checkpoint.checkpoint.channel_values.messages

  // 2. 按消息类型重播
  for (const message of messages) {
    if (HumanMessage)  → sendMessageChunk(..., "user", ...)
    if (AIMessage)     → sendMessageChunk(..., "agent", ...) + sendToolCall(...)
    // ToolMessage 不单独重播（已由 tool_call_update 覆盖）
  }
}
```

---

## Agent 实例化：`createAgent()`

```typescript
private createAgent(agentName: string) {
  const config = agentConfigs.get(agentName)

  // 选择 Backend
  const backend = this.createBackend(config)
  // 优先级：config.backend > ACPFilesystemBackend > FilesystemBackend

  const agent = createDeepAgent({
    model, tools, systemPrompt, middleware,
    subagents, responseFormat, contextSchema,
    interruptOn, store,
    backend,
    skills, memory,
    checkpointer,    // 共享检查点
    name,
  })

  this.agents.set(agentName, agent)
}
```

**Backend 选择逻辑（`createBackend()`）**：

```
config.backend 存在？
  → 使用自定义 backend
客户端支持 fsReadTextFile 且 fsWriteTextFile？
  → ACPFilesystemBackend（读写走 IDE 代理）
否则
  → FilesystemBackend（直接读写本地文件系统）
```

---

## ACP 通知发送方法

| 方法 | ACP 事件类型 | 用途 |
|---|---|---|
| `sendMessageChunk()` | `agent_message_chunk` / `user_message_chunk` / `agent_thought_chunk` | 流式文本 |
| `sendToolCall()` | `tool_call` | 工具调用开始 |
| `sendToolCallUpdate()` | `tool_call_update` | 工具状态/结果更新 |
| `sendPlanUpdate()` | `plan` | 计划列表更新 |

---

## 常量定义

```typescript
// 三种模式
const AVAILABLE_MODES = [
  { id: "agent", name: "Agent Mode", description: "Full autonomous agent" },
  { id: "plan",  name: "Plan Mode",  description: "Planning and discussion" },
  { id: "ask",   name: "Ask Mode",   description: "Q&A without file changes" },
]

// 内置 slash 命令
const DEFAULT_COMMANDS = [
  { name: "plan",   description: "Switch to plan mode" },
  { name: "agent",  description: "Switch to agent mode" },
  { name: "ask",    description: "Switch to ask mode" },
  { name: "clear",  description: "Clear conversation context" },
  { name: "status", description: "Show current session status" },
]

// 默认认证方式
const DEFAULT_AUTH_METHODS = [
  { id: "anthropic", type: "env_var", vars: [{ name: "ANTHROPIC_API_KEY" }] },
  { id: "openai",    type: "env_var", vars: [{ name: "OPENAI_API_KEY" }] },
  { id: "deepagents-setup", type: "agent" },
]
```
