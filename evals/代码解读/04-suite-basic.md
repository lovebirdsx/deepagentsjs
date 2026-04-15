# basic：基础行为

**目录**：`evals/basic/`

测试 Agent 的两项最基础能力：系统提示遵从和工具使用决策。这是验证 Agent 基本健全性的"冒烟测试"。

---

## 测试用例

### 1. system prompt: custom system prompt

**验证目标**：Agent 能遵从自定义系统提示中的身份设定。

```typescript
runner
  .extend({ systemPrompt: "Your name is Alice. ..." })
  .run({ query: "What is your name?" });

expect(result).toHaveFinalTextContaining("Alice");
```

**关键点**：
- 使用 `runner.extend()` 注入自定义系统提示
- 自定义提示与内置 `BASE_AGENT_PROMPT` 合并
- 验证 LLM 能在推理时使用提示中的具体信息

---

### 2. avoid unnecessary tool calls

**验证目标**：对于无需工具的简单问题，Agent 不应该调用任何工具。

```typescript
runner.run({ query: "What is 2+2?" });

expect(result).toHaveToolCallRequests(0);
expect(result).toHaveFinalTextContaining("4");
```

**关键点**：
- 断言工具调用次数为 0
- 测试 Agent 的"工具使用克制性"
- 这是一个常见的 Agent 退化问题：过度依赖工具调用

---

## 设计模式

这套评估展示了最基本的两种断言模式：

```typescript
// 模式 1：文本包含断言
expect(result).toHaveFinalTextContaining("期望内容");

// 模式 2：工具调用计数断言
expect(result).toHaveToolCallRequests(0);

// 模式 3：Runner 配置覆盖
runner.extend({ systemPrompt: "自定义提示" }).run({ query: "..." });
```

---

## 参考链接

- [eval-harness 框架 →](./02-eval-harness.md)
- [files：文件操作 →](./05-suite-files.md)
