# memory：AGENTS.md 记忆

**目录**：`evals/memory/`

10 个测试用例全面验证 AGENTS.md 记忆注入功能——记忆是否正确加载、是否影响行为、多源合并、缺失文件处理等。

---

## 背景

DeepAgents 通过 `memory` 参数指定 AGENTS.md 文件路径，这些文件的内容会被注入到系统提示中，成为 Agent 的"长期记忆"。

```typescript
runner.extend({
  memory: ["/AGENTS.md"],           // 单源记忆
  // 或：
  memory: ["/base/AGENTS.md", "/project/AGENTS.md"],  // 多源记忆
})
```

---

## 测试用例

### 1. memory basic recall（基础记忆回溯）

```typescript
runner
  .extend({ memory: ["/AGENTS.md"] })
  .run({
    query: "What is my favorite color?",
    initialFiles: {
      "/AGENTS.md": "The user's favorite color is blue."
    }
  });

expect(result).toHaveFinalTextContaining("blue");
```

**验证**：Agent 能从 AGENTS.md 中提取具体事实并在回答中使用。

---

### 2. memory guided behavior naming convention（记忆影响命名行为）

```typescript
runner
  .extend({ memory: ["/AGENTS.md"] })
  .run({
    query: "Create a new Python file for user authentication",
    initialFiles: {
      "/AGENTS.md": "Always use snake_case for Python file names."
    }
  });

// 验证文件名符合记忆中的规范
const fileNames = Object.keys(result.files);
expect(fileNames.some(name => name.match(/^\/[a-z_]+\.py$/))).toBe(true);
```

**验证**：记忆中的编码规范能影响 Agent 的实际行为（不仅仅是回答内容）。

---

### 3. memory influences file content（记忆影响文件内容）

```typescript
runner
  .extend({ memory: ["/AGENTS.md"] })
  .run({
    query: "Write a hello world script",
    initialFiles: {
      "/AGENTS.md": "Always use f-strings for string formatting in Python."
    }
  });

// 验证生成的代码使用了 f-strings
const fileContent = Object.values(result.files)[0];
expect(fileContent).toContain('f"');
```

---

### 4. memory multiple sources combined（多源记忆合并）

```typescript
runner
  .extend({ memory: ["/base/AGENTS.md", "/project/AGENTS.md"] })
  .run({
    query: "What are my preferences?",
    initialFiles: {
      "/base/AGENTS.md": "Global rule: Use TypeScript.",
      "/project/AGENTS.md": "Project rule: Use React for UI."
    }
  });

// 两个记忆文件的内容都应该被整合
expect(result).toHaveFinalTextContaining("TypeScript");
expect(result).toHaveFinalTextContaining("React");
```

**关键点**：多个 AGENTS.md 文件的内容是**拼接**（concatenation），而非覆盖。

---

### 5. memory with missing file graceful（缺失文件优雅处理）

```typescript
runner
  .extend({ memory: ["/nonexistent/AGENTS.md"] })
  .run({ query: "What do you know?" });

// 不应该崩溃，也不应该声称有记忆内容
expect(result).not.toHaveFinalTextContaining("error");
```

---

### 6. memory prevents unnecessary file reads（记忆避免重复读取）

```typescript
runner
  .extend({ memory: ["/AGENTS.md"] })
  .run({
    query: "What is in AGENTS.md?",
    initialFiles: { "/AGENTS.md": "Some config content" }
  });

// Agent 不应该调用 read_file 来读取 AGENTS.md
// 因为其内容已经通过记忆注入到系统提示中
expect(result).toHaveToolCallRequests(0);
expect(result).toHaveFinalTextContaining("Some config content");
```

**验证**：避免低效的重复加载——记忆已注入，不应再通过工具读取。

---

### 7. memory does not persist transient info（暂态信息不入记忆）

```typescript
runner
  .extend({ memory: ["/AGENTS.md"] })
  .run({
    query: "I'm at a coffee shop today. Help me write a function.",
    initialFiles: { "/AGENTS.md": "" }
  });

// "coffee shop" 这类暂态信息不应被持久化到 AGENTS.md
// （注意：这里测试的是 Agent 的行为决策，不是真实的记忆写入机制）
expect(result.files["/AGENTS.md"] ?? "").not.toContain("coffee shop");
```

---

### 8. memory updates user formatting preference（更新用户格式偏好）

```typescript
runner
  .extend({ memory: ["/AGENTS.md"] })
  .run({
    query: "I prefer responses in bullet points. Please remember this.",
    initialFiles: { "/AGENTS.md": "" }
  });

// Agent 应将用户偏好写入 AGENTS.md
const agentsMd = result.files["/AGENTS.md"] ?? "";
expect(agentsMd).toContain("bullet");
```

---

### 9. memory missing file graceful without claiming context

验证：当记忆文件不存在时，Agent 不应捏造记忆内容，也不应在回答中假装知道记忆中有什么。

---

### 10. memory middleware composite backend routing

验证：记忆中间件在使用 CompositeBackend 时的基础集成，确保路由正确。

---

## 关键设计点

### 记忆注入时机

记忆文件在 `beforeAgent` 钩子阶段加载，内容追加到系统提示末尾：

```
系统提示 =
  BASE_AGENT_PROMPT +
  用户自定义提示 +
  "# Memory\n" + AGENTS.md内容（多个文件拼接）
```

### 用例的层次结构

```
用例 1-3：单文件，验证"是否读取"和"是否使用"
用例 4：多文件，验证"是否合并"
用例 5, 9：缺失文件，验证"优雅降级"
用例 6：已注入，验证"不重复读取"
用例 7-8：动态内容，验证"写入判断"
```

---

## 参考链接

- [memory-agent-bench：长上下文记忆 →](./08-suite-memory-agent-bench.md)
- [memory-multiturn：多轮记忆 →](./09-suite-memory-multiturn.md)
- [skills：技能加载（类似机制） →](./10-suite-skills.md)
