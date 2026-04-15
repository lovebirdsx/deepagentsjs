# adapter.ts — 消息格式适配器

**文件路径：** `src/adapter.ts`  
**代码行数：** ~257 行  
**职责：** ACP 消息格式 ↔ LangChain 消息格式的双向转换，以及工具调用的元信息生成

---

## 概述

LangChain/LangGraph 使用自己的消息类型（`HumanMessage`、`AIMessage`、`ToolMessage`），而 ACP 使用 `ContentBlock[]`（包含 text、image、resource 等块）。adapter 层完全隔离了这两种格式，让 server.ts 不需要直接处理格式细节。

---

## ACP → LangChain 方向

### `acpContentToLangChain(content: ContentBlock[])`

把 ACP 的 `ContentBlock[]` 转为 LangChain 的 `content` 格式：

```typescript
// 优化：单个纯文本块直接返回 string（避免不必要的数组）
if (content.length === 1 && content[0].type === "text") {
  return content[0].text
}

// 多块或非文本：返回对象数组
content.map(block => {
  switch (block.type) {
    case "text":  → { type: "text", text }
    case "image": → { type: "image_url", image_url: "data:image/png;base64,..." }
                     或 { type: "image_url", image_url: url }
    case "resource": → { type: "text", text: "[Resource: uri]\ntext内容" }
    default:      → { type: "text", text: String(block) }
  }
})
```

**Resource 块的处理方式**：resource 没有 LangChain 的直接对应类型，所以转成带前缀的文本块（`[Resource: uri]\n内容`），这是一种务实的降级处理。

### `acpPromptToHumanMessage(content: ContentBlock[])`

最终用户消息的入口，直接封装：

```typescript
return new HumanMessage({ content: acpContentToLangChain(content) })
```

---

## LangChain → ACP 方向

### `langChainContentToACP(content)`

```typescript
// string → 单个 text 块
if (typeof content === "string") {
  return [{ type: "text", text: content }]
}

// 数组 → 每个 text 块取 text 字段，其他类型 JSON 序列化
content.map(block => {
  if (block.type === "text") return { type: "text", text: block.text }
  return { type: "text", text: JSON.stringify(block) }  // 降级
})
```

### `langChainMessageToACP(message: BaseMessage)`

包装 `langChainContentToACP`，从 `BaseMessage.content` 提取并转换。

---

## 工具调用相关

### `extractToolCalls(message: AIMessage): ToolCallInfo[]`

从 LangChain `AIMessage` 提取工具调用列表：

```typescript
return message.tool_calls.map(tc => ({
  id: tc.id ?? crypto.randomUUID(),
  name: tc.name,
  args: tc.args,
  status: "pending",
}))
```

### `todosToPlanEntries(todos)`

把 LangGraph 中 `write_todos` 工具产生的 todo 列表转为 ACP `PlanEntry[]`：

```typescript
return todos.map(todo => ({
  content: todo.content,
  priority: todo.priority ?? "medium",
  status: todo.status === "cancelled" ? "skipped" : todo.status,
  // "cancelled" → "skipped"（ACP 不支持 cancelled 状态）
}))
```

---

## 工具元信息生成

### `getToolCallKind(toolName)` — 工具分类

用于 IDE 显示不同的图标：

```typescript
const readTools    = ["read_file", "ls"]
const searchTools  = ["grep", "glob"]
const editTools    = ["write_file", "edit_file"]
const executeTools = ["execute", "shell", "terminal"]
const thinkTools   = ["write_todos"]

// 返回: "read" | "search" | "edit" | "execute" | "think" | "other"
```

### `formatToolCallTitle(toolName, args)` — 用户友好标题

```typescript
switch (toolName) {
  case "read_file":  → `Reading ${args.path}`
  case "write_file": → `Writing ${args.path}`
  case "edit_file":  → `Editing ${args.path}`
  case "ls":         → `Listing ${args.path}`
  case "grep":       → `Searching for "${args.pattern}"`
  case "glob":       → `Finding files matching ${args.pattern}`
  case "task":       → `Delegating: ${args.description}`
  case "write_todos":→ `Planning tasks`
  default:           → `Executing ${toolName}`
}
```

### `extractToolCallLocations(toolName, args, workspaceRoot)` — 文件定位

为支持文件操作的工具提取位置信息，让 IDE 可以实时跟随（follow-along）：

```typescript
// 只有这些工具有路径信息
const toolsWithPaths = ["read_file", "write_file", "edit_file", "ls", "grep", "glob"]

// 组合绝对路径
const absPath = filePath.startsWith("/") ? filePath : `${workspaceRoot}/${filePath}`
const line = args.line ?? args.startLine

return [{ path: absPath, ...(line != null ? { line } : {}) }]
```

---

## ID 生成

```typescript
// 会话 ID：sess_ + 16位 hex
generateSessionId() → `sess_${uuid.replace(/-/g, "").slice(0, 16)}`

// 工具调用 ID：call_ + 12位 hex
generateToolCallId() → `call_${uuid.replace(/-/g, "").slice(0, 12)}`
```

---

## 文件 URI 工具

```typescript
fileUriToPath(uri) → 去掉 "file://" 前缀
pathToFileUri(path) → 加上 "file://" 前缀
```

---

## 设计亮点

1. **单层职责**：adapter 只做格式转换，不包含任何业务逻辑
2. **优雅降级**：不认识的内容类型一律转为文本，而不是报错
3. **性能优化**：单纯文本消息直接返回 string，避免数组包装
4. **状态映射**：`cancelled → skipped` 处理了 LangGraph 与 ACP 状态枚举的差异
