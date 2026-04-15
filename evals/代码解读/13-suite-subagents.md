# subagents：子代理委派

**目录**：`evals/subagents/`

2 个测试用例验证子代理（Subagent）的路由机制——通过 `task` 工具将任务委派给命名子代理或通用子代理。

---

## 背景

DeepAgents 允许配置子代理，主代理可以通过 `task` 工具将复杂任务委派给子代理。子代理在隔离的上下文窗口中运行，有自己的工具集和系统提示。

---

## 测试用例

### 1. task calls weather subagent（委派给命名子代理）

```typescript
// 自定义一个 weather 工具
const getWeather = tool(
  async ({ city }) => `Weather in ${city}: Sunny, 25°C`,
  {
    name: "get_weather",
    description: "Get weather for a city",
    schema: z.object({ city: z.string() }),
  }
);

runner
  .extend({
    subagents: [{
      name: "weather_agent",
      description: "Specialist for weather queries. Use for any weather-related questions.",
      tools: [getWeather],
    }],
  })
  .run({ query: "What's the weather in Tokyo?" });

// 主代理应通过 task 工具委派给 weather_agent
expect(result).toHaveToolCallInStep(1, {
  name: "task",
  argsContains: { agent_name: "weather_agent" }
});

// 最终回答包含天气信息
expect(result).toHaveFinalTextContaining("Sunny");
```

**关键验证**：
- 第一步就通过 `task` 工具委派（不是自己处理）
- `agent_name` 参数正确指向 `weather_agent`
- 最终能从子代理获取结果并回答

---

### 2. task calls general-purpose subagent（委派给通用子代理）

```typescript
// 工具注册在主代理上（不配置命名子代理）
const listFiles = tool(...);  // 模拟文件列表工具

runner
  .extend({ tools: [listFiles] })
  .run({ query: "List all files and analyze their structure" });

// 不显式命名子代理时，应委派给通用子代理
expect(result).toHaveToolCallInStep(1, {
  name: "task",
  argsContains: { agent_name: "general-purpose" }
});
```

**关键验证**：当没有专用命名子代理时，DeepAgents 自动提供 `general-purpose` 子代理，可访问主代理的所有工具。

---

## task 工具参数结构

```typescript
// task 工具被调用时的参数
{
  agent_name: "weather_agent",    // 或 "general-purpose"
  prompt: "What's the weather in Tokyo?"  // 委派的任务描述
}
```

---

## 状态隔离说明

子代理执行时，以下状态被**隔离**（不继承自主代理）：
- `messages`（对话历史）
- `todos`（任务列表）
- `skillsMetadata`

以下状态被**共享**：
- `files`（虚拟文件系统）
- `memoryContents`

---

## 参考链接

- [hitl：含子代理的中断流程 →](./15-suite-hitl.md)
- [tool-selection：工具路由 →](./11-suite-tool-selection.md)
- [架构总览（子代理数据流） →](./01-架构总览.md)
