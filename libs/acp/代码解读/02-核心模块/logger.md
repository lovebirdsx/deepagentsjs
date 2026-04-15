# logger.ts — 日志系统

**文件路径：** `src/logger.ts`  
**代码行数：** ~301 行

---

## 设计背景

由于 `stdout` 专属于 ACP JSON-RPC 协议通信，日志**不能写 stdout**，否则会污染协议流。因此系统提供了两个输出通道：

- **stderr**（调试模式）：`--debug` 开启，实时输出，方便开发调试
- **文件**（生产调试）：`--log-file` 指定路径，以追加模式写入，带时间戳

---

## 核心接口

```typescript
export type LogLevel = "debug" | "info" | "warn" | "error"

export interface LoggerOptions {
  debug?: boolean       // 是否输出到 stderr
  logFile?: string      // 日志文件路径
  minLevel?: LogLevel   // 最小输出级别
  prefix?: string       // 日志前缀（默认 "[deepagents-acp]"）
  timestamps?: boolean  // stderr 是否带时间戳（默认 false）
}
```

---

## 日志级别数值

```typescript
const LOG_LEVELS = { debug: 0, info: 1, warn: 2, error: 3 }
```

`warn` 和 `error` 级别无论是否开启 debug 模式都会输出到 stderr：

```typescript
if (this.debug || levelNum >= LOG_LEVELS.warn) {
  console.error(stderrMessage)
}
```

---

## 文件流初始化

```typescript
private initFileStream(logFilePath: string): void {
  // 1. path.resolve() 确保绝对路径
  // 2. fs.mkdirSync(dir, { recursive: true }) 自动创建目录
  // 3. fs.createWriteStream(path, { flags: "a" }) 追加模式
  // 4. 写入启动分隔符（===...=== Started at ISO时间戳）
  // 5. 监听 error 事件 → 出错时置 null（静默降级）
}
```

每次服务器启动都会写入分隔符，便于在同一文件中区分多次运行的日志。

---

## 消息格式

```typescript
// 文件日志（带时间戳）
[2025-04-15T10:00:00.000Z] [deepagents-acp] [DEBUG] message content

// stderr 日志（不带时间戳，默认）
[deepagents-acp] [DEBUG] message content
```

---

## 公共方法

```typescript
log(...args)          // debug 级别（主要用途：server.ts 内部跟踪）
debug_log(...args)    // debug 级别（别名）
info(...args)         // info 级别
warn(...args)         // warn 级别（始终输出到 stderr）
error(...args)        // error 级别（始终输出到 stderr）
logLevel(level, ...args) // 动态指定级别

// 流管理
close(): Promise<void>  // 写入关闭分隔符，关闭文件流
flush(): Promise<void>  // 触发 drain（确保写入完成）
```

---

## `nullLogger` — 空日志器

```typescript
export const nullLogger: Logger = {
  log: () => {},
  debug_log: () => {},
  info: () => {},
  warn: () => {},
  error: () => {},
  // ...
} as unknown as Logger
```

当完全不需要日志时可使用，避免 null 检查。

---

## 关闭时序

```typescript
// server.stop() 调用
await this.logger.close()

// close() 实现
fileStream.write(shutdownMessage, () => {
  fileStream.end(() => {
    this.fileStream = null
    resolve()
  })
})
```

确保关闭前所有待写内容都刷盘完毕，避免日志截断。

---

## 使用模式

```typescript
// server.ts 内部统一使用 this.log()
private log(...args: unknown[]): void {
  this.logger.log(...args)
}

// 调用示例
this.log("Prompt received:", { sessionId, preview: "..." })
this.log("Tool completed:", { toolId, status: "completed" })
```

所有 server.ts 内的日志均通过 `this.log()` 包装，便于统一控制。
