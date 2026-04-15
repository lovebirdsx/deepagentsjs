# todos：任务列表

**目录**：`evals/todos/`

验证 Agent 对 `write_todos` 工具的正确使用——每完成一项任务都应更新 TODO 列表，且必须按顺序逐步更新，最终返回完成确认。

---

## 测试用例

### 1. write todos sequential updates returns text

**场景**：5 个串行任务，每完成一个更新 TODO 列表。

```typescript
runner.run({
  query: `
    Complete these 5 tasks in order, updating todos after each:
    1. Say "Starting task 1"
    2. Say "Starting task 2"
    ...
    5. Say "Starting task 5"
    When done, respond with exactly: DONE
  `
});

// 至少调用 6 次 write_todos（初始化 + 5 次状态更新）
const writeTodosCalls = countWriteTodosCalls(result);
expect(writeTodosCalls).toBeGreaterThanOrEqual(6);

// 最终必须回复 "DONE"
expect(result).toHaveFinalTextContaining("DONE");
```

### 2. write todos three steps returns text

**场景**：3 个任务，2 次状态更新的简化版本。

```typescript
// 至少调用 4 次 write_todos（初始化 + 3 次更新）
expect(writeTodosCalls).toBeGreaterThanOrEqual(4);
expect(result).toHaveFinalTextContaining("DONE");
```

---

## 辅助函数

```typescript
// 统计轨迹中 write_todos 的调用次数
function countWriteTodosCalls(trajectory: AgentTrajectory): number {
  return trajectory.steps.reduce((count, step) => {
    const toolCalls = step.action.tool_calls ?? [];
    return count + toolCalls.filter(tc => tc.name === "write_todos").length;
  }, 0);
}
```

---

## 验证的行为模式

1. **初始化**：开始任务前创建 TODO 列表
2. **逐步更新**：每完成一项，立即标记为完成
3. **任务序列化**：按顺序执行，不跳过更新
4. **完成确认**：所有任务完成后输出明确的完成信号

---

## 参考链接

- [basic：基础行为 →](./04-suite-basic.md)
- [eval-harness 框架 →](./02-eval-harness.md)
