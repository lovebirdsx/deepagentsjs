# files：文件操作

**目录**：`evals/files/`

最大的核心评估套件之一，18 个测试用例系统覆盖 Agent 的文件操作能力，包括正确性验证、效率验证和边界情况处理。

---

## 测试用例概览

| 编号 | 名称 | 核心验证点 |
|------|------|-----------|
| 1 | read file | 基础文件读取 |
| 2 | write file | 基础文件写入 |
| 3 | write files in parallel | 并行写入多个文件 |
| 4 | ls directory | 目录列出（肯定） |
| 5 | ls directory missing file | 目录列出（否定） |
| 6 | edit file | 就地编辑 |
| 7 | read then write derived | 读取后派生写入 |
| 8 | read files in parallel | 并行读取多个文件 |
| 9 | grep finds matching paths | grep 路径搜索 |
| 10 | glob lists markdown files | glob 模式匹配 |
| 11 | write files with verification | 写入后验证 |
| 12 | write files ambiguous confirmation | 模糊确认处理 |
| 13 | find magic phrase deep nesting | 深层嵌套中高效搜索 |
| 14 | identify quote author parallel reads | 并行读取找作者 |
| 15 | identify quote author unprompted efficiency | 无提示的效率优化 |
| 16 | read file truncation recovery | 大文件分页读取 |
| 17 | read file empty file | 空文件处理 |

---

## 关键测试详解

### 基础文件读写（用例 1-2）

```typescript
// 读取：通过 initialFiles 预置文件内容
runner.run({
  query: "What does /data.txt say?",
  initialFiles: { "/data.txt": "Hello, World!" }
});
expect(result).toHaveFinalTextContaining("Hello, World!");

// 写入：断言 files 状态中存在文件
runner.run({ query: "Write 'test content' to /output.txt" });
expect(result.files["/output.txt"]).toContain("test content");
```

### 并行操作（用例 3, 8）

```typescript
// 要求 Agent 在单步中发出多个工具调用
runner.run({
  query: "Write three separate files: /a.txt, /b.txt, /c.txt"
});

// 验证效率：断言第 1 步就完成所有写入（并行工具调用）
expect(result).toHaveToolCallInStep(1, { name: "write_file" });
// 且总步数较少（并行 vs 串行）
```

### 深层嵌套高效搜索（用例 13）

这是一个**效率测试**：在深层嵌套目录结构中找到一个"魔法短语"。

```typescript
// 预置深层目录结构
const initialFiles = {
  "/level1/level2/level3/level4/data.txt": "The magic phrase is: ABCD1234",
  "/level1/other.txt": "no magic here",
  // ... 更多无关文件
};

runner.run({
  query: "Find the magic phrase in the files",
  initialFiles
});

// Agent 应该使用 grep 而非逐个 read_file
expect(result).toHaveFinalTextContaining("ABCD1234");
// 预期步数较少（用 grep 效率高）
```

**考察点**：Agent 是否知道对深层搜索使用 `grep` 而非低效的逐个 `read_file`。

### 大文件分页读取（用例 16）

```typescript
// 预置超过单次读取限制的大文件（>100 行）
const bigContent = Array.from({ length: 500 }, (_, i) => `Line ${i + 1}`).join("\n");

runner.run({
  query: "What is on line 450?",
  initialFiles: { "/large.txt": bigContent }
});

// Agent 应使用 offset/limit 参数分页读取
expect(result).toHaveFinalTextContaining("Line 450");
// 验证 Agent 能处理文件被截断并用分页参数恢复的情况
```

### 并行读取找作者（用例 14-15）

```typescript
const initialFiles = {
  "/quotes/q1.txt": '"To be or not to be" - Shakespeare',
  "/quotes/q2.txt": '"I think therefore I am" - Descartes',
  "/quotes/q3.txt": '"Give me liberty" - Henry',
};

// 用例 14：明确提示可以并行
// 用例 15：不提示，测试 Agent 是否主动并行

runner.run({
  query: "Who said 'I think therefore I am'?",
  initialFiles
});

expect(result).toHaveFinalTextContaining("Descartes");
// 效率断言：不应逐个串行读取
```

---

## 覆盖的工具

| 工具 | 测试用例 |
|------|---------|
| `read_file` | 1, 7, 8, 14, 15, 16, 17 |
| `write_file` | 2, 3, 7, 11, 12 |
| `edit_file` | 6 |
| `ls` | 4, 5 |
| `grep` | 9, 13 |
| `glob` | 10 |

---

## 断言模式

```typescript
// 文件内容断言
expect(result.files["/output.txt"]).toContain("期望内容");

// 最终文本断言
expect(result).toHaveFinalTextContaining("期望文本");

// 效率断言：特定步骤有特定工具调用
expect(result).toHaveToolCallInStep(1, {
  name: "write_file",
  argsContains: { path: "/output.txt" }
});

// 步数断言（效率验证）
expect(result).toHaveAgentSteps(2);
```

---

## 参考链接

- [summarization：大文件进阶处理 →](./14-suite-summarization.md)
- [tool-usage-relational：工具链式调用 →](./12-suite-tool-usage-relational.md)
