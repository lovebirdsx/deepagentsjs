# matchers — vitest 断言扩展与 LangSmith 集成

**对应源文件：** `src/matchers.ts`（226~388 行为 matcher 部分）

这部分实现了四个自定义 vitest 断言，让测试代码可以对 `AgentTrajectory` 进行声明式断言，并自动将评估指标上报到 LangSmith。

---

## TypeScript 类型扩展

vitest 使用模块增强（module augmentation）来扩展 `expect()` 的类型：

```typescript
// src/matchers.ts:240
interface CustomMatchers {
  toHaveAgentSteps(expected: number): void;
  toHaveToolCallRequests(expected: number): void;
  toHaveToolCallInStep(
    step: number,
    match: {
      name: string;
      argsContains?: Record<string, unknown>;
      argsEquals?: Record<string, unknown>;
    },
  ): void;
  toHaveFinalTextContaining(text: string, caseInsensitive?: boolean): void;
}

declare module "vitest" {
  interface Assertion extends CustomMatchers {}
  interface AsymmetricMatchersContaining extends CustomMatchers {}
}
```

这段 `declare module "vitest"` 是 TypeScript 的声明合并语法，它让 `expect(traj).toHaveAgentSteps(2)` 能通过类型检查，并提供代码补全。

`expect.extend({...})` 才是运行时注册，与类型声明是分离的。

---

## LangSmith 集成模式

所有 matcher 都遵循相同的 LangSmith 上报模式：

```typescript
import * as ls from "langsmith/vitest";

// 上报三个指标：实际值、期望值、是否匹配
ls.logFeedback({ key: "agent_steps", score: actual });
ls.logFeedback({ key: "expected_num_agent_steps", score: expected });
ls.logFeedback({ key: "match_num_agent_steps", score: actual === expected ? 1 : 0 });
```

**上报的所有指标：**

| 指标 key                          | 类型 | 说明               |
| --------------------------------- | ---- | ------------------ |
| `agent_steps`                     | 数值 | 实际步骤数         |
| `expected_num_agent_steps`        | 数值 | 期望步骤数         |
| `match_num_agent_steps`           | 0/1  | 步骤数是否匹配     |
| `tool_call_requests`              | 数值 | 实际工具调用总数   |
| `expected_num_tool_call_requests` | 数值 | 期望工具调用总数   |
| `match_num_tool_call_requests`    | 0/1  | 工具调用数是否匹配 |

**这个设计的价值：** 即使测试通过了，LangSmith 也记录了量化指标。你可以在 LangSmith 的实验仪表板中对比多个模型运行的平均步骤数、工具调用数，而不仅仅是知道"测试通过/失败"。

---

## toHaveAgentSteps

```typescript
expect.extend({
  toHaveAgentSteps(received: AgentTrajectory, expected: number) {
    const actual = received.steps.length;

    ls.logFeedback({ key: "agent_steps", score: actual });
    ls.logFeedback({ key: "expected_num_agent_steps", score: expected });
    ls.logFeedback({ key: "match_num_agent_steps", score: actual === expected ? 1 : 0 });

    return {
      pass: actual === expected,
      message: () =>
        `expected ${expected} agent steps but got ${actual}\n\ntrajectory:\n${prettyTrajectory(received)}`,
      actual,
      expected,
    };
  },
})
```

**使用示例：**
```typescript
expect(trajectory).toHaveAgentSteps(2);
// 断言代理恰好执行了 2 个步骤
// 失败时打印完整的 trajectory 内容
```

matcher 返回对象中的 `actual` 和 `expected` 字段会在 vitest 的差异输出中显示，方便对比。

---

## toHaveToolCallRequests

```typescript
toHaveToolCallRequests(received: AgentTrajectory, expected: number) {
  const actual = getToolCallCount(received);  // 累计所有 step 的工具调用数
  // ...上报指标...
  return {
    pass: actual === expected,
    message: () => `expected ${expected} tool call requests but got ${actual}\n\ntrajectory:\n${prettyTrajectory(received)}`,
    actual,
    expected,
  };
}
```

`getToolCallCount` 计算所有 step 中 `action.tool_calls` 数组长度之和。注意这是"请求"次数，不是"成功"次数——即使工具执行失败，计数也会增加。

---

## toHaveToolCallInStep — 最复杂的 matcher

```typescript
toHaveToolCallInStep(
  received: AgentTrajectory,
  stepNum: number,          // 1-indexed
  match: {
    name: string;
    argsContains?: Record<string, unknown>;  // 参数子集匹配
    argsEquals?: Record<string, unknown>;    // 参数精确匹配
  },
)
```

### 匹配逻辑（层层过滤）

```typescript
// 1. 验证步骤号
if (stepNum <= 0) {
  return { pass: false, message: () => "step must be positive (1-indexed)" };
}
if (stepNum > received.steps.length) {
  return { pass: false, message: () => `expected at least ${stepNum} steps...` };
}

// 2. 获取该步骤的所有工具调用
const step = received.steps[stepNum - 1];
const toolCalls = step.action.tool_calls ?? [];

// 3. 按名称过滤
let matches = toolCalls.filter((tc) => tc.name === match.name);

// 4. argsContains：子集匹配（每个指定的 k-v 都必须存在）
if (match.argsContains != null) {
  matches = matches.filter(
    (tc) =>
      Object.entries(match.argsContains!).every(
        ([k, v]) => (tc.args as Record<string, unknown>)[k] === v,
      ),
  );
}

// 5. argsEquals：精确匹配（JSON 序列化比较）
if (match.argsEquals != null) {
  matches = matches.filter(
    (tc) => JSON.stringify(tc.args) === JSON.stringify(match.argsEquals),
  );
}

return { pass: matches.length > 0, ... };
```

**使用示例：**

```typescript
// 只验证工具名称
expect(traj).toHaveToolCallInStep(1, { name: "write_file" });

// 验证工具名称 + 部分参数
expect(traj).toHaveToolCallInStep(1, {
  name: "write_file",
  argsContains: { path: "src/main.ts" },  // 只要 path 字段匹配就行
});

// 验证工具名称 + 完整参数
expect(traj).toHaveToolCallInStep(2, {
  name: "bash",
  argsEquals: { command: "npm test" },  // 所有参数必须完全一致
});
```

**注意：** `argsEquals` 使用 `JSON.stringify` 比较，所以参数的键顺序影响结果。`argsContains` 更灵活，推荐优先使用。

---

## toHaveFinalTextContaining

```typescript
toHaveFinalTextContaining(
  received: AgentTrajectory,
  text: string,
  caseInsensitive = false,  // 默认大小写敏感
) {
  const finalText = getFinalText(received);

  if (received.steps.length === 0) {
    return { pass: false, message: () => "...trajectory has no steps" };
  }

  const haystack = caseInsensitive ? finalText.toLowerCase() : finalText;
  const needle = caseInsensitive ? text.toLowerCase() : text;

  return {
    pass: haystack.includes(needle),
    message: () =>
      `expected final text to contain ${JSON.stringify(text)} (caseInsensitive=${caseInsensitive})\n\nactual final text: ${JSON.stringify(finalText)}`,
    actual: finalText,
    expected: text,
  };
}
```

**使用示例：**

```typescript
// 大小写敏感（默认）
expect(traj).toHaveFinalTextContaining("任务完成");

// 大小写不敏感
expect(traj).toHaveFinalTextContaining("task complete", true);
```

**注意：** `getFinalText()` 只提取最后一个 step 的 `action.content`（当它是字符串时）。如果最后一步是工具调用（`content` 为空或 `tool_calls` 数组），这个 matcher 会返回空字符串。

---

## 错误消息设计

所有 matcher 失败时都会打印 `prettyTrajectory(received)`，输出格式如下：

```
expected 2 agent steps but got 3

trajectory:
step 1:
  - read {"path": "src/main.ts"}
step 2:
  - write {"path": "src/main.ts", "content": "..."}
step 3:
  text: 我已经完成了文件修改。以下是主要改动...
```

这让开发者在不查看日志的情况下，直接从测试报告中理解代理实际做了什么。
