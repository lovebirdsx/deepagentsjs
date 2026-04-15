# memory-multiturn：多轮记忆

**目录**：`evals/memory-multiturn/`

测试 Agent 在多轮对话中的记忆持久化能力——偏好是否被正确学习、不相关信息是否被正确过滤。

---

## 背景

多轮记忆测试需要模拟**多条消息的对话历史**，而非单次查询。Runner 需要支持预置对话历史。

---

## 测试用例

### 1. implicit preference remembered（隐式偏好被记住）

**场景**：用户在对话中隐式表达语言偏好，Agent 应学习并持久化。

```typescript
// 模拟多轮对话：第一轮用户纠正了语言选择
const conversationHistory = [
  { role: "user", content: "Write a hello world program" },
  { role: "assistant", content: "Here's a Python version: print('Hello')" },
  { role: "user", content: "I prefer C++, please use that instead" },
  { role: "assistant", content: "Here's C++: cout << 'Hello' << endl;" },
];

// 后续查询：Agent 应记住偏好
runner.run({
  query: "What programming language should you use for me?",
  // 携带对话历史
});

expect(result).toHaveFinalTextContaining("C++");
// 同时验证偏好已写入记忆
const agentsMd = result.files["/AGENTS.md"] ?? "";
expect(agentsMd).toContain("C++");
```

**考察**：Agent 能从对话中提取隐式偏好（不是用户明确说"记住这个"，而是通过行为表现出的偏好）。

---

### 2. explicit preference remembered（显式偏好被记住）

**场景**：用户明确给出指令，Agent 应严格遵守并持久化。

```typescript
const conversationHistory = [
  { role: "user", content: "Never use emojis in your responses." },
  { role: "assistant", content: "Understood, I won't use emojis." },
];

runner.run({ query: "Write a celebratory message for our team" });

// 不应包含 emoji
const finalText = getFinalText(result);
expect(finalText).not.toMatch(/[\u{1F600}-\u{1F64F}]/u);

// 偏好应写入记忆
const agentsMd = result.files["/AGENTS.md"] ?? "";
expect(agentsMd.toLowerCase()).toContain("emoji");
```

---

### 3. transient info not persisted（暂态信息不持久化）

**场景**：用户提到了当天的状态信息，这不应该被写入长期记忆。

```typescript
runner.run({
  query: "I'm exhausted today, I stayed up late. Help me write a status update."
});

// "疲惫"是今日状态，不应进入长期记忆
const agentsMd = result.files["/AGENTS.md"] ?? "";
expect(agentsMd.toLowerCase()).not.toContain("exhaust");
expect(agentsMd.toLowerCase()).not.toContain("tired");
expect(agentsMd.toLowerCase()).not.toContain("stayed up");
```

**考察**：Agent 能区分"值得长期保存的偏好/规则"和"当时语境下的暂态信息"。

---

## 核心挑战

这套测试考察的是 Agent 的**元认知能力**：

1. **信号识别**：什么信息值得被记住？
2. **持久化决策**：什么时候更新 AGENTS.md？
3. **暂态过滤**：如何区分长期规则和当下上下文？

这是难度较高的评估，对模型的推理能力要求较高。

---

## 参考链接

- [memory：AGENTS.md 记忆 →](./07-suite-memory.md)
- [memory-agent-bench：长上下文记忆 →](./08-suite-memory-agent-bench.md)
