# summarization：历史压缩

**目录**：`evals/summarization/`

3 个测试用例验证对话历史压缩中间件的行为——大文件分块处理、历史卸载到文件系统、压缩后的可追问性。

---

## 背景

当对话历史接近模型 token 限制时，DeepAgents 的 `summarizationMiddleware` 会自动压缩历史，并将摘要写入文件系统（`/conversation_history/` 目录）。

---

## 测试用例

### 1. summarize continues task（压缩后任务继续）

**场景**：读取一个超大文件（1600+ 个函数），逐块找到一个标记值。

```typescript
// 生成一个包含 1600+ 个函数定义的大 Python 文件
const bigPythonFile = generateFunctions(1600) +
  "\nMAGIC_END_MARKER = 'FOUND_IT_42'\n" +
  generateFunctions(200);  // 标记在文件中间某处

runner
  .extend({
    // 触发较低的 token 阈值，确保压缩被激活
    middleware: [createSummarizationMiddleware({ maxTokens: 5000 })],
  })
  .run({
    query: "Read large_file.py in chunks of ≤100 lines. Find MAGIC_END_MARKER value.",
    initialFiles: { "/large_file.py": bigPythonFile }
  });

// 即使触发了历史压缩，任务仍应完成
expect(result).toHaveFinalTextContaining("FOUND_IT_42");
```

**考察**：历史压缩不应导致任务中断或信息丢失（标记值仍能被找到）。

---

### 2. summarization offloads to filesystem（历史卸载到文件系统）

```typescript
runner.run({
  query: "Read large_file.py in 50-line chunks and summarize each chunk",
  initialFiles: { "/large_file.py": largePythonFile }
});

// 验证历史被写入文件
const historyFiles = Object.keys(result.files).filter(
  path => path.startsWith("/conversation_history/")
);
expect(historyFiles.length).toBeGreaterThan(0);

// 验证文件包含摘要标记
const historyContent = result.files[historyFiles[0]];
expect(historyContent).toContain("## Summarized at");
```

**考察**：触发历史压缩时，被压缩的历史以 Markdown 格式保存到 `/conversation_history/` 目录，包含时间戳标记。

---

### 3. summarization preserves followup answerability（压缩后仍可追问）

**这是最严格的测试**：历史压缩后，Agent 仍能回答关于早期消息内容的追问。

```typescript
// 第一轮：读取大文件
runner.run({
  query: "Read large_file.py in chunks. After reading, tell me the first standard library import after __future__",
  initialFiles: { "/large_file.py": largePythonFile }
});

// 期望：即使中间触发了历史压缩，仍能回答关于文件内容的具体问题
expect(result).toHaveFinalTextContaining("logging");
// （large_file.py 中 __future__ 之后第一个标准库是 logging）
```

**考察**：摘要质量——被压缩的历史必须保留足够的信息，使后续追问可以被回答。

---

## 压缩机制说明

### 触发条件

```
当前消息 token 数 > maxTokens 阈值
→ 触发历史压缩
```

### 压缩流程

```
1. 选取早期的若干条消息进行压缩
2. 调用 LLM 生成摘要（"以下是对话历史摘要：..."）
3. 将原始消息替换为摘要消息
4. 将原始消息内容写入 /conversation_history/{timestamp}.md
5. 继续对话
```

### 文件格式

```markdown
## Summarized at 2026-04-15T10:30:00Z

### Key Information Retained
- 已读取 large_file.py 的前 200 行
- 发现 import logging 在 from __future__ 之后

### Original Messages
[被压缩的原始消息摘录...]
```

---

## 参考链接

- [files：文件操作（分页读取） →](./05-suite-files.md)
- [oolong：长上下文推理 →](./19-suite-oolong.md)
