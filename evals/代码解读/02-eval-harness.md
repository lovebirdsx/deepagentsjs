# eval-harness 框架

**位置**：`internal/eval-harness/src/`

eval-harness 是所有评估套件共享的基础设施包，发布为 `@deepagents/evals`。它提供四个核心模块：Runner 注册表、DeepAgent 集成、自定义断言器和模型注册。

---

## 1. runners.ts — 核心类型与注册表

### 关键类型

#### EvalRunner（接口）

所有 Runner 的抽象接口：

```typescript
interface EvalRunner {
  name: string;

  // 执行 Agent，返回完整行为轨迹
  run(params: RunAgentParams): Promise<AgentTrajectory>;

  // 派生出一个携带配置覆盖的 Runner（不修改原 Runner）
  extend(overrides: Record<string, unknown>): EvalRunner;
}
```

#### RunAgentParams（运行参数）

```typescript
interface RunAgentParams {
  query: string;                           // 用户消息
  initialFiles?: Record<string, string>;   // 初始虚拟文件系统（路径 → 内容）
}
```

#### AgentTrajectory（行为轨迹）

```typescript
interface AgentTrajectory {
  steps: AgentStep[];              // 按顺序排列的 Agent 推理步骤
  files: Record<string, string>;   // 最终文件系统状态（路径 → 内容）
}

interface AgentStep {
  index: number;                // 步骤序号，从 1 开始
  action: AIMessage;            // LLM 响应（文本 + 工具调用列表）
  observations: ToolMessage[];  // 工具执行结果
}
```

### 注册表 API

```typescript
// 注册一个 Runner（通常在 setup.ts 中调用）
function registerRunner(runner: EvalRunner): void

// 按名称获取 Runner
function getRunner(name: string): EvalRunner

// 从 EVAL_RUNNER 环境变量获取默认 Runner
function getDefaultRunner(): EvalRunner
```

### 工具函数

```typescript
// 将 LangGraph 原始输出解析为 AgentTrajectory
function parseTrajectory(
  messages: BaseMessage[],
  files: Record<string, FileData | string>
): AgentTrajectory

// 提取最后一步的文本内容
function getFinalText(trajectory: AgentTrajectory): string
```

**parseTrajectory 实现逻辑：**
1. 遍历消息列表，将 `AIMessage` 识别为新步骤的开始
2. 收集紧随其后的所有 `ToolMessage` 作为该步骤的 observations
3. 步骤序号从 1 开始递增

---

## 2. deepagent.ts — DeepAgent 集成

### DeepAgentEvalRunner 类

`EvalRunner` 接口的主要实现，将 `createDeepAgent` 包装为可在测试中使用的 Runner。

```typescript
class DeepAgentEvalRunner implements EvalRunner {
  constructor(
    name: string,
    factory: (config?: Partial<CreateDeepAgentParams>) => DeepAgent,
    overrides?: Record<string, unknown>
  )
}
```

**关键设计决策：**
- 默认 Agent（无覆盖）在构造时**一次性创建**，重复 `run()` 调用复用同一个 Agent 实例
- 带覆盖的 Agent（通过 `extend()` 创建）在**每次 `run()` 时重新创建**，保证配置隔离
- 每次 `run()` 调用都生成唯一的 `thread_id`（UUID），实现状态隔离

#### run() 方法执行流程

```
run({ query, initialFiles })
    │
    ├─ 将 initialFiles 字符串转为 LangGraph FileData 格式
    │  { content: [文本], metadata: { createdAt: ISO时间戳 } }
    │
    ├─ 生成 UUID 作为 thread_id
    │
    ├─ 调用 agent.invoke({
    │    messages: [HumanMessage(query)],
    │    files: 转换后的 FileData
    │  }, { configurable: { thread_id } })
    │
    ├─ ls.logOutputs() 上报轨迹到 LangSmith
    │
    └─ parseTrajectory(result.messages, result.files)
       → 返回 AgentTrajectory
```

#### extend() 方法

```typescript
extend(overrides: Record<string, unknown>): EvalRunner {
  return new DeepAgentEvalRunner(
    this.name,
    this.factory,
    { ...this.overrides, ...overrides }  // 浅合并覆盖
  );
}
```

### 注册函数

```typescript
// 便捷函数：注册一个 DeepAgent 驱动的 Runner
function registerDeepAgentRunner(
  name: string,
  factory: (config?: Partial<CreateDeepAgentParams>) => DeepAgent
): void
```

---

## 3. matchers.ts — 自定义 Vitest 断言器

扩展 Vitest 的 `expect` 对象，增加 4 个专为 Agent 行为设计的断言器。每个断言器在执行时自动向 LangSmith 上报结构化 feedback。

### toHaveAgentSteps(n)

断言 Agent 轨迹恰好有 `n` 步。

```typescript
expect(result).toHaveAgentSteps(3);

// 自动上报到 LangSmith：
// { agent_steps: 3, expected_num_agent_steps: 3, match_num_agent_steps: true/false }
```

**失败时的错误信息**包含 `prettyTrajectory(trajectory)` 格式化的完整轨迹。

### toHaveToolCallRequests(n)

断言整个轨迹中总工具调用次数为 `n`。

```typescript
expect(result).toHaveToolCallRequests(5);

// 自动上报：
// { tool_call_requests: 5, expected_num_tool_call_requests: 5, match_num_tool_call_requests: true/false }
```

**工具调用计数**：通过 `getToolCallCount()` 累加每个步骤中 `action.tool_calls.length` 实现。

### toHaveToolCallInStep(step, match)

断言第 `step` 步中存在满足条件的工具调用。

```typescript
expect(result).toHaveToolCallInStep(1, {
  name: "read_file",              // 工具名称（精确匹配）
  argsContains: { path: "/data" }, // 参数子集匹配（部分键匹配）
  // 或：
  argsEquals: { path: "/data.txt" } // 参数完全匹配
});
```

- 步骤序号从 **1** 开始
- `argsContains` 和 `argsEquals` 互斥，只需提供一个
- `argsContains` 使用 `expect.objectContaining()` 语义

### toHaveFinalTextContaining(text, caseInsensitive?)

断言最终响应文本包含指定字符串。

```typescript
expect(result).toHaveFinalTextContaining("任务完成");
expect(result).toHaveFinalTextContaining("done", true);  // 不区分大小写
```

### 辅助函数

```typescript
// 格式化轨迹用于错误输出
function prettyTrajectory(trajectory: AgentTrajectory): string

// 统计总工具调用次数
function getToolCallCount(trajectory: AgentTrajectory): number

// 将 LangGraph files 格式规范化为 Record<string, string>
// 处理 plain string 和 { content: string[] } 两种格式
function coerceFiles(raw: Record<string, FileData | string>): Record<string, string>
```

---

## 4. setup.ts — 模型 Runner 注册

在 Vitest 的 `setupFiles` 阶段执行，注册 7 个具名模型 Runner：

| Runner 名称 | 模型 | 特殊配置 |
|------------|------|---------|
| `sonnet-4-5` | claude-sonnet-4-5-20250929 | 无 |
| `sonnet-4-5-thinking` | claude-sonnet-4-5-20250929 | 扩展思考，budget=5000 |
| `opus-4-6` | claude-opus-4-6 | 无 |
| `sonnet-4-6` | claude-sonnet-4-6 | 无 |
| `gpt-4.1` | gpt-4.1 | 无 |
| `gpt-4.1-mini` | gpt-4.1-mini | 无 |
| `o3-mini` | o3-mini | 无 |

每个 Runner 通过工厂函数创建：

```typescript
registerDeepAgentRunner("sonnet-4-6", (config) =>
  createDeepAgent({
    model: "anthropic:claude-sonnet-4-6",
    ...config,
  })
);
```

`...config` 允许测试套件通过 `runner.extend(overrides)` 注入自定义工具、系统提示等。

---

## 5. index.ts — 公共 API

```typescript
// 从各模块聚合导出
export * from "./runners.js";
export * from "./deepagent.js";
export * from "./matchers.js";   // 同时注册断言器（副作用）
// setup.ts 由 vitest.config.ts 的 setupFiles 独立引入
```

---

## 参考链接

- [架构总览 →](./01-架构总览.md)
- [运行与配置 →](./03-运行与配置.md)
- [设计模式与扩展 →](./21-设计模式与扩展.md)
