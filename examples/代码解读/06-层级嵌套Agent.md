# 层级嵌套 Agent

**文件**：`hierarchical/hierarchical-agent.ts`

演示**三层嵌套代理架构**：主 Agent → 研究专家（自身也是一个 DeepAgent）→ 事实核查子代理。展示了 `CompiledSubAgent` 类型的用法。

---

## 架构图

```
主 Agent（协调员）
    ├── 工具: get_weather（直接处理天气查询）
    └── 子代理: research-specialist [CompiledSubAgent]
            ├── 工具: get_news
            ├── 工具: analyze_data
            └── 子代理: fact-checker [SubAgent]
                    └── 工具: verify_claim
```

**三种工具类型**（仅占位，实际返回 mock 数据）：
- `get_weather`：天气查询（主 Agent 直接使用）
- `get_news`/`analyze_data`：新闻和数据分析（研究专家使用）
- `verify_claim`：事实核查（最深层子代理使用）

---

## 代码解读

### 1. 最深层：事实核查子代理（SubAgent）

```typescript
const factCheckerSubAgent: SubAgent = {
  name: "fact-checker",
  description:
    "A fact-checking agent that can verify claims. " +
    "Use this when you need to validate specific facts or statements.",
  systemPrompt:
    "You are a fact-checking specialist. Use the verify_claim tool to check " +
    "the accuracy of any claims or statements you receive.",
  tools: [verifyClaim],
};
```

普通 `SubAgent` 类型，在运行时由 deepagents 编译。

### 2. 第二层：研究专家（自身是 DeepAgent）

```typescript
const researchDeepAgent = createDeepAgent({
  systemPrompt: "You are a research specialist...",
  tools: [getNews, analyzeData],
  subagents: [factCheckerSubAgent],   // 自己也有子代理
});
```

`researchDeepAgent` 是一个**已编译的 DeepAgent**（`CompiledGraph`），拥有完整的 deepagents 运行时：文件系统工具、TODO 管理、摘要压缩等。

### 3. 第一层：主 Agent（使用 CompiledSubAgent 包装）

```typescript
export const mainAgent = createDeepAgent({
  systemPrompt:
    "- For weather queries, use the get_weather tool directly\n" +
    "- For research/analysis/news queries, delegate to research-specialist",
  tools: [getWeather],
  subagents: [
    {
      name: "research-specialist",
      description:
        "A specialized research agent with its own tools and sub-agents. " +
        "It can search for news, analyze data, and verify facts. " +
        "Use this for any research, analysis, or investigation tasks.",
      runnable: researchDeepAgent,   // 传入已编译的 DeepAgent
    } satisfies CompiledSubAgent,
  ],
});
```

**关键**：`runnable: researchDeepAgent` 使用 `CompiledSubAgent` 类型，直接传入已构建好的 Agent，而非在运行时重新编译。

---

## 三种子代理类型对比

| 类型 | 声明方式 | 编译时机 | 适用场景 |
|------|---------|---------|---------|
| `SubAgent` | `{ name, description, tools, systemPrompt }` | Agent 启动时 | 轻量级专用子代理 |
| `CompiledSubAgent` | `{ name, description, runnable }` | 事先已编译 | 复杂子代理，需要完整 deepagents 能力 |
| `AsyncSubAgent` | `{ name, description, graphId, url }` | 远程部署 | 后台长时运行任务 |

---

## 执行示例

```typescript
// 查询 1：天气 → 主 Agent 直接处理
const result1 = await mainAgent.invoke({
  messages: [new HumanMessage("What's the weather in San Francisco?")],
});
// 主 Agent 调用 get_weather 工具，不委派子代理

// 查询 2：研究 → 委派给研究专家
const result2 = await mainAgent.invoke({
  messages: [
    new HumanMessage(
      "Research the latest developments in renewable energy and provide an analysis.",
    ),
  ],
});
// 主 Agent → research-specialist → get_news + analyze_data
// 如果需要核查事实 → fact-checker → verify_claim
```

---

## 设计要点

### LLM 驱动的路由决策

主 Agent 的系统提示明确说明何时使用工具、何时委派：

```
- For weather queries, use the get_weather tool directly
- For research, analysis, or news queries, delegate to research-specialist
```

这种**基于自然语言的路由规则**比代码硬编码路由更灵活，但也要求系统提示足够清晰。

### 状态隔离

子代理有**独立的状态**（messages、files、todos），不会泄露到父 Agent：
- 研究专家写的中间文件不会出现在主 Agent 的 files 中
- 研究专家的消息历史不会污染主 Agent 的上下文

### 能力继承

`CompiledSubAgent`（researchDeepAgent）内置了 deepagents 的完整中间件栈：
- 文件系统工具（ls, read_file, write_file...）
- TODO 列表管理
- 对话历史摘要
- Anthropic 提示缓存

这意味着研究专家可以在调研过程中读写文件、管理任务，像一个独立的工作代理。

---

*[← 技能与记忆 Agent](./05-技能与记忆Agent.md) | [异步子代理并行研究 →](./07-异步子代理并行研究.md)*
