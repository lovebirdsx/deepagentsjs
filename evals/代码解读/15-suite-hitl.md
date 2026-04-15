# hitl：人工审批

**目录**：`evals/hitl/`

3 个测试用例验证 Human-in-the-Loop（HITL）中断机制——Agent 在执行敏感操作前暂停等待人工审批，审批后继续执行。

---

## 背景

DeepAgents 的 `interruptOn` 参数允许为特定工具配置中断规则。当 Agent 调用这些工具时，执行流程暂停，等待人工输入审批或拒绝。

---

## 测试用的特殊设置

HITL 测试**不使用 EvalRunner**（因为 Runner 不支持中断-恢复流程），而是直接创建 Agent：

```typescript
import { createDeepAgent } from "@deepagents/deepagents";
import { MemorySaver } from "@langchain/langgraph";

// 使用 MemorySaver checkpointer 支持状态持久化
const checkpointer = new MemorySaver();
const threadId = crypto.randomUUID();
const config = { configurable: { thread_id: threadId } };
```

---

## 工具与中断配置

```typescript
// 工具 1：每次调用都中断
const sampleTool = tool(async ({ data }) => `Processed: ${data}`, {
  name: "sample_tool",
  description: "Process some data",
  schema: z.object({ data: z.string() }),
});

// 工具 2：不中断
const getWeather = tool(async ({ city }) => `Weather in ${city}: Sunny`, {
  name: "get_weather",
  description: "Get weather",
  schema: z.object({ city: z.string() }),
});

// 工具 3：自定义中断条件
const getSoccerScores = tool(async () => "Score: 2-1", {
  name: "get_soccer_scores",
  description: "Get soccer scores",
  schema: z.object({}),
});

const agent = createDeepAgent({
  tools: [sampleTool, getWeather, getSoccerScores],
  checkpointer,
  interruptOn: {
    "sample_tool": true,                           // 总是中断
    "get_weather": false,                          // 从不中断
    "get_soccer_scores": {                         // 自定义
      allowedDecisions: ["approve", "reject"],     // 允许的审批选项
      message: "Allow soccer score lookup?",       // 提示消息
    },
  },
});
```

---

## 测试用例

### 1. test_hitl_agent（基础中断流程）

```typescript
// 第一轮：触发中断
const firstResult = await agent.invoke(
  { messages: [new HumanMessage("Use sample_tool with data='hello'")] },
  config
);

// 验证：执行被中断（返回 __interrupt__ 标记）
expect(firstResult.__interrupt__).toBeDefined();
expect(firstResult.__interrupt__[0].value).toMatchObject({
  tool_name: "sample_tool",
  // 工具调用详情
});

// 第二轮：人工审批，继续执行
const resumeResult = await agent.invoke(
  new Command({ resume: "approve" }),
  config
);

// 验证：任务完成
expect(resumeResult.messages.at(-1).content).toContain("Processed: hello");
```

**关键 API**：
- `agent.invoke()` 返回带 `__interrupt__` 字段的状态（中断时）
- `agent.invoke(new Command({ resume: "decision" }), config)` 继续执行

---

### 2. test_subagent_with_hitl（子代理 + 中断）

```typescript
// 主代理通过 task 工具委派给子代理
// 子代理执行时触发 HITL 中断

const firstResult = await agent.invoke(
  { messages: [new HumanMessage("Use task tool to process data with sample_tool")] },
  config
);

// 验证：task 工具调用发出，等待子代理中断
expect(firstResult.__interrupt__).toBeDefined();

// 审批
const finalResult = await agent.invoke(
  new Command({ resume: "approve" }),
  config
);

expect(finalResult.messages.at(-1).content).toContain("Processed");
```

---

### 3. test_subagent_with_custom_interrupt_on（子代理自定义中断配置）

```typescript
// 主代理有中断配置，但子代理可以有不同的中断配置
const agentWithCustomSubagent = createDeepAgent({
  tools: [sampleTool, getWeather],
  checkpointer,
  interruptOn: { "sample_tool": true },
  subagents: [{
    name: "task_handler",
    description: "Handles tasks without interruption",
    tools: [sampleTool, getWeather],
    // 子代理禁用了父代理的中断
    interruptOn: { "sample_tool": false },
  }],
});

// 通过子代理执行时不应触发中断
const result = await agentWithCustomSubagent.invoke(
  { messages: [new HumanMessage("Have task_handler use sample_tool")] },
  config
);

// 不应有中断
expect(result.__interrupt__).toBeUndefined();
```

---

## 中断配置选项

```typescript
type InterruptOnConfig = 
  | boolean                    // true: 总是中断; false: 从不中断
  | {
      allowedDecisions?: string[];  // 可选的审批选项列表
      message?: string;              // 显示给审批人的消息
    };

// interruptOn 参数类型
type InterruptOnParam = Record<string, InterruptOnConfig>;
```

---

## 中断-恢复流程图

```
agent.invoke(query)
    │
    ├─ LLM 决定调用 sample_tool
    │
    ├─ HITL 检查: interruptOn["sample_tool"] === true
    │
    ├─ 暂停执行，返回 { __interrupt__: [{value: {tool_name, args}}] }
    │          ← 此时 Agent 状态已持久化到 checkpointer
    │
[人工审批界面显示工具调用详情]
    │
    ├─ 人工选择: "approve" / "reject"
    │
agent.invoke(new Command({ resume: "approve" }), config)
    │         ← 使用相同 thread_id 恢复
    │
    ├─ 继续工具执行
    └─ 返回最终结果
```

---

## 参考链接

- [subagents：子代理委派 →](./13-suite-subagents.md)
- [架构总览 →](./01-架构总览.md)
