# followup-quality：追问质量

**目录**：`evals/followup-quality/`

6 个参数化测试用例，评估 Agent 在面对模糊请求时提出的澄清问题的质量——问题是否相关、是否精准、是否避免了无效信息。

---

## 核心考察点

当用户提出模糊请求时，Agent 应该：
1. **提问**（而非猜测执行）
2. **提相关问题**（关注核心缺失信息）
3. **避免提不相关问题**（不问用户已隐含提供的信息）

---

## 测试用例结构

每个测试用例包含：

```typescript
interface FollowupCase {
  name: string;
  query: string;
  mustContainOneOf: string[];  // 最终回复必须包含至少一项
  mustNotContain?: string[];   // 最终回复不得包含任何一项
}
```

---

## 6 个测试用例

### 1. vague data analysis（模糊的数据分析请求）

```typescript
{
  query: "Analyze my data",
  mustContainOneOf: [
    "data source", "format", "file", "dataset", "what data", "which data"
  ],
  // Agent 必须询问数据在哪里/是什么格式
}
```

### 2. vague send report（模糊的定期报告请求）

```typescript
{
  query: "Send report weekly",
  mustContainOneOf: [
    "content", "format", "what to include", "report include", "about"
  ],
  mustNotContain: [
    "time", "when", "day", "monday", "friday", "morning", "afternoon"
  ],
  // 用户已经说了"每周"，不应该再问时间
  // 应该问报告内容是什么
}
```

### 3. vague monitor system（模糊的系统监控请求）

```typescript
{
  query: "Monitor production",
  mustContainOneOf: [
    "metric", "threshold", "alert", "what to monitor", "which system"
  ],
}
```

### 4. vague summarize emails（模糊的邮件摘要请求）

```typescript
{
  query: "Summarize emails daily",
  mustContainOneOf: [
    "format", "deliver", "send", "output", "where"
  ],
  // 用户已说"每天"，不问时间；问摘要如何交付
}
```

### 5. vague customer support（模糊的客服请求）

```typescript
{
  query: "Help respond faster",
  mustContainOneOf: [
    "channel", "platform", "context", "ticket", "current process"
  ],
  // 应问通过什么渠道、什么样的上下文
}
```

### 6. detailed calendar brief（详细但仍需追问）

```typescript
{
  query: "Set up a daily briefing at 9am for my calendar events",
  mustContainOneOf: [
    "deliver", "send", "email", "notification", "where", "how"
  ],
  mustNotContain: [
    "time", "when", "9", "schedule", "frequency"
  ],
  // 时间已经说了，不应再问；应问如何接收摘要
}
```

---

## 断言逻辑

```typescript
ls.test(testCase.name, { inputs: { query: testCase.query } }, async ({ inputs }) => {
  const result = await runner.run({ query: inputs.query });
  const finalText = getFinalText(result).toLowerCase();

  // 断言 1：必须提问（包含问号）
  expect(finalText).toContain("?");

  // 断言 2：必须包含至少一个相关词
  const hasRelevantSignal = testCase.mustContainOneOf.some(
    keyword => finalText.includes(keyword.toLowerCase())
  );
  expect(hasRelevantSignal).toBe(true);

  // 断言 3：不得包含不相关词
  if (testCase.mustNotContain) {
    for (const forbidden of testCase.mustNotContain) {
      expect(finalText).not.toContain(forbidden.toLowerCase());
    }
  }

  // 上报到 LangSmith
  ls.logFeedback({
    key: "followup_has_question_mark",
    score: finalText.includes("?") ? 1 : 0,
  });
  ls.logFeedback({
    key: "followup_relevant_signal",
    score: hasRelevantSignal ? 1 : 0,
  });
});
```

---

## 测试设计亮点

### 参数化测试

所有用例通过数组 + for 循环注册，避免代码重复：

```typescript
const CASES: FollowupCase[] = [...];

for (const testCase of CASES) {
  ls.test(testCase.name, { inputs: { query: testCase.query } }, async ({ inputs }) => {
    // 统一逻辑
  });
}
```

### mustNotContain 的价值

`mustNotContain` 字段验证 Agent 能识别"已知信息"，不会提冗余问题。这是比 `mustContainOneOf` 更严格的测试维度——不仅要问对，还要避免问错。

---

## 参考链接

- [basic：基础行为 →](./04-suite-basic.md)
- [设计模式与扩展 →](./21-设计模式与扩展.md)
