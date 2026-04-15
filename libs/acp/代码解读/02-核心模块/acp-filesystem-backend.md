# acp-filesystem-backend.ts — ACP 文件系统后端

**文件路径：** `src/acp-filesystem-backend.ts`  
**代码行数：** ~95 行  
**继承：** `FilesystemBackend`（来自 `deepagents` 包）

---

## 解决的问题

直接读写磁盘文件有一个局限：**无法读取编辑器中未保存的缓冲区内容**。

当用户在 IDE 中编辑了一个文件但还没有保存，AI Agent 如果直接读磁盘，会得到旧内容。`ACPFilesystemBackend` 通过 ACP 协议把读写操作代理给 IDE 客户端，从而可以：

- 读取编辑器当前缓冲区（含未保存修改）
- 写操作由 IDE 接管（支持撤销、变更追踪、差异高亮）

---

## 类结构

```typescript
export class ACPFilesystemBackend extends FilesystemBackend {
  private conn: AgentSideConnection   // ACP 连接对象
  private currentSessionId: string | null  // 当前会话 ID

  constructor(options: { conn: AgentSideConnection; rootDir: string })
  setSessionId(sessionId: string): void    // 在 session 建立后设置
  read(filePath, offset?, limit?): Promise<ReadResult>
  write(filePath, content): Promise<WriteResult>
}
```

---

## 读操作：`read()`

```typescript
async read(filePath, offset?, limit?) {
  // 没有 sessionId → 直接走本地文件系统
  if (!this.currentSessionId) return super.read(filePath, offset, limit)

  const absPath = this.resolveAbsPath(filePath)  // 相对路径 → 绝对路径
  try {
    const result = await this.conn.readTextFile({
      sessionId: this.currentSessionId,
      path: absPath,
    })

    let text = result.content ?? result.text ?? ""

    // 支持分页（offset/limit）
    if (offset != null || limit != null) {
      const lines = text.split("\n")
      const start = offset ?? 0
      const end = limit != null ? start + limit : lines.length
      text = lines.slice(start, end).join("\n")
    }

    return { content: text }
  } catch {
    // 失败时回退到本地文件系统
    return super.read(filePath, offset, limit)
  }
}
```

**回退策略**：ACP 读取失败（如客户端不支持、文件未打开）时，静默回退到 `super.read()`，透明降级。

---

## 写操作：`write()`

```typescript
async write(filePath, content) {
  if (!this.currentSessionId) return super.write(filePath, content)

  const absPath = this.resolveAbsPath(filePath)
  try {
    await this.conn.writeTextFile({
      sessionId: this.currentSessionId,
      path: absPath,
      content,
    })
    return { path: absPath, filesUpdate: null }
  } catch {
    return super.write(filePath, content)  // 失败回退
  }
}
```

---

## 不覆盖的操作

`ls`、`glob`、`grep` 等操作没有对应的 ACP 协议方法，直接继承 `FilesystemBackend` 的实现（操作本地文件系统）。

这是一个务实的设计：只代理 ACP 支持的操作，其余保持本地行为。

---

## 生命周期

```
DeepAgentsServer.handleInitialize()
  → 检测到客户端支持 fsReadTextFile + fsWriteTextFile
  → createBackend() 创建 ACPFilesystemBackend 实例
  → 存入 acpBackends Map

DeepAgentsServer.handleNewSession()
  → acpBackend.setSessionId(sessionId)  // 关联会话

DeepAgentsServer.handleLoadSession()
  → acpBackend.setSessionId(sessionId)  // 恢复时重新关联
```

`setSessionId` 必须在每次 session 建立/加载后调用，因为 ACP 的文件操作请求需要携带 `sessionId`。

---

## 路径解析

```typescript
private resolveAbsPath(filePath: string): string {
  if (path.isAbsolute(filePath)) return filePath
  return path.resolve(this.cwd, filePath)  // cwd 来自父类 FilesystemBackend
}
```

Agent 内部使用相对路径时，这里自动补全为绝对路径再发给 IDE。

---

## ACP 响应格式兼容

注意代码中的多字段尝试：
```typescript
let text = result.content ?? result.text ?? ""
```

这是因为 ACP SDK 不同版本的 `readTextFile` 返回字段可能是 `content` 或 `text`，做了向前兼容处理。
