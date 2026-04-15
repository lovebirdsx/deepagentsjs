# tool-selection：工具路由

**目录**：`evals/tool-selection/`

8 个测试用例验证 Agent 的工具路由能力——直接请求工具、间接推断工具、以及多工具链式调用。

---

## 工具集定义

使用 `vi.fn` spy 包装 8 个模拟工具，便于断言调用情况：

```typescript
function spiedTool<T>(fn: T, config: ToolConfig): { tool: StructuredTool; spy: vi.Mock } {
  const spy = vi.fn(fn);
  return { tool: tool(spy, config), spy };
}

const tools = {
  slackDm: spiedTool(async ({ recipient, message }) => `DM sent to ${recipient}`, {
    name: "slack_send_dm",
    description: "Send a direct message on Slack",
    schema: z.object({ recipient: z.string(), message: z.string() }),
  }),
  slackChannel: spiedTool(...),   // slack_post_channel
  githubIssue: spiedTool(...),    // github_create_issue
  githubPr: spiedTool(...),       // github_create_pr
  linearIssue: spiedTool(...),    // linear_create_issue
  gmailEmail: spiedTool(...),     // gmail_send_email
  webSearch: spiedTool(...),      // web_search
  calendarEvent: spiedTool(...),  // calendar_create_event
};
```

---

## 测试用例

### 直接请求（3个）

用户明确说出要用的工具/渠道。

#### 1. direct request slack dm

```typescript
runner.run({ query: "Send John a Slack DM saying the deploy is done" });

expect(tools.slackDm.spy).toHaveBeenCalledWith(
  expect.objectContaining({ recipient: expect.stringContaining("John") }),
  expect.anything()
);
```

#### 2. direct request github pr

```typescript
runner.run({
  query: "Create a GitHub PR titled 'Fix bug #123' for the main branch"
});

expect(tools.githubPr.spy).toHaveBeenCalledWith(
  expect.objectContaining({ title: expect.stringContaining("Fix bug #123") }),
  expect.anything()
);
```

#### 3. direct request multiple tools

```typescript
runner.run({
  query: "Create a GitHub issue AND a Linear issue for the same bug"
});

expect(tools.githubIssue.spy).toHaveBeenCalled();
expect(tools.linearIssue.spy).toHaveBeenCalled();
```

---

### 间接请求（3个）

用户描述意图，Agent 需要推断使用哪个工具。

#### 4. indirect schedule meeting

```typescript
runner.run({ query: "Schedule a meeting with the team for tomorrow at 2pm" });

// Agent 推断：日程安排 → calendar_create_event
expect(tools.calendarEvent.spy).toHaveBeenCalled();
```

#### 5. indirect notify team

```typescript
runner.run({ query: "Let the team know we're going live this afternoon" });

// Agent 推断：通知团队 → slack_post_channel（不是 DM）
expect(tools.slackChannel.spy).toHaveBeenCalled();
```

#### 6. indirect email report

```typescript
runner.run({ query: "Email the weekly report to stakeholders" });

// Agent 推断：发送邮件 → gmail_send_email
expect(tools.gmailEmail.spy).toHaveBeenCalled();
```

---

### 链式调用（2个）

一个任务需要多个工具按顺序调用。

#### 7. chain search then email

```typescript
runner.run({
  query: "Find the current price of AAPL stock and email it to the finance team"
});

// 步骤 1：搜索
expect(result).toHaveToolCallInStep(1, { name: "web_search" });
// 步骤 2：发邮件（使用搜索结果）
expect(result).toHaveToolCallInStep(2, { name: "gmail_send_email" });
```

#### 8. chain create issue then notify

```typescript
runner.run({
  query: "Create a GitHub issue for the login bug, then notify the team on Slack"
});

expect(result).toHaveToolCallInStep(1, { name: "github_create_issue" });
expect(result).toHaveToolCallInStep(2, { name: "slack_post_channel" });
```

---

## 断言模式总结

```typescript
// 模式 1：Spy 被调用
expect(tools.slackDm.spy).toHaveBeenCalled();

// 模式 2：Spy 以特定参数被调用
expect(tools.githubPr.spy).toHaveBeenCalledWith(
  expect.objectContaining({ title: "..." }),
  expect.anything()  // 第二参数是 LangChain config，用 anything() 忽略
);

// 模式 3：特定步骤有特定工具调用（链式顺序验证）
expect(result).toHaveToolCallInStep(1, { name: "web_search" });
expect(result).toHaveToolCallInStep(2, { name: "gmail_send_email" });

// 模式 4：多个 Spy 都被调用（并行任务）
expect(tools.githubIssue.spy).toHaveBeenCalled();
expect(tools.linearIssue.spy).toHaveBeenCalled();
```

---

## 参考链接

- [tool-usage-relational：关系型数据工具链 →](./12-suite-tool-usage-relational.md)
- [subagents：子代理工具路由 →](./13-suite-subagents.md)
