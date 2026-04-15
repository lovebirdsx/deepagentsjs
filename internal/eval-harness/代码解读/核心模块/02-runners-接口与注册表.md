# runners — 接口定义、注册表与轨迹解析

**对应源文件：** `src/runners.ts`（51 行注册代码）和 `src/matchers.ts`（38~111 行，接口与注册表逻辑）

> **注意**：本框架的文件命名有些反直觉——`runners.ts` 实际上只做"注册"工作（setup 脚本），而接口定义、注册函数、轨迹解析逻辑都在 `matchers.ts` 的前半段（第 1~225 行）。

---

## 接口定义

### EvalRunner 接口

```typescript
// src/matchers.ts:49
interface EvalRunner {
  name: string;
  run(params: RunAgentParams): Promise<AgentTrajectory>;
  extend(overrides: Record<string, unknown>): EvalRunner;
}
```

**设计意图：**
- `name`：全局唯一标识，用于注册表的键值查找
- `run()`：核心执行方法，调用底层代理并返回结构化轨迹
- `extend()`：**不可变配置覆盖**——返回一个新实例，原实例不受影响

`extend()` 是一个非常关键的设计决策。它避免了"测试间状态污染"：每个测试可以用 `runner.extend({...})` 获得定制化版本，而不需要重新注册或修改全局状态。

### RunAgentParams 接口

```typescript
// src/matchers.ts:35
interface RunAgentParams {
  query: string;
  initialFiles?: Record<string, string>;
}
```

`initialFiles` 让测试可以预置文件系统状态，例如：

```typescript
runner.run({
  query: "修复 main.ts 中的 bug",
  initialFiles: {
    "main.ts": "function add(a, b) { return a - b; }  // bug: 应该是 +"
  }
})
```

---

## 注册表机制

```typescript
// src/matchers.ts:64
const runners: Record<string, EvalRunner> = {};

export function registerRunner(runner: EvalRunner): void {
  runners[runner.name] = runner;
}

export function getRunner(name: string): EvalRunner {
  const runner = runners[name];
  if (!runner) {
    const available = Object.keys(runners).join(", ");
    throw new Error(`Unknown eval runner "${name}". Available: ${available}`);
  }
  return runner;
}
```

注册表是一个模块级的简单对象字典。错误信息中会列出所有可用的运行器名称，方便调试。

### getDefaultRunner — 从环境变量解析运行器

```typescript
// src/matchers.ts:99
let _resolved: EvalRunner | null = null;

export function getDefaultRunner(): EvalRunner {
  const name = process.env.EVAL_RUNNER;
  if (!name) throw new Error(`No eval runner specified...`);
  if (_resolved == null) {
    _resolved = getRunner(name);
  }
  return _resolved;
}
```

**两个设计点：**
1. **环境变量驱动**：通过 `EVAL_RUNNER=sonnet-4-5 vitest` 在不修改代码的情况下切换模型
2. **单例缓存**：`_resolved` 变量确保同一个测试进程中只解析一次，避免重复查找

---

## 运行器注册（src/runners.ts）

`src/runners.ts` 是初始化脚本，在 vitest 的 `setupFile` 阶段执行：

```typescript
// src/runners.ts（完整内容）
import { ChatAnthropic } from "@langchain/anthropic";
import { ChatOpenAI } from "@langchain/openai";
import { createDeepAgent } from "deepagents";
import { registerDeepAgentRunner } from "./deepagent.js";

registerDeepAgentRunner("sonnet-4-5", (config) =>
  createDeepAgent({
    ...config,
    model: new ChatAnthropic({ model: "claude-sonnet-4-5-20250929" }),
  }),
);

// ... 另外 6 个模型的注册
```

每次调用 `registerDeepAgentRunner(name, factory)` 会：
1. 创建 `new DeepAgentEvalRunner(name, factory)`
2. 调用 `registerRunner(runner)` 写入全局字典

**已注册的 7 个运行器：**

| 名称                  | 模型                       | 特殊配置                                             |
| --------------------- | -------------------------- | ---------------------------------------------------- |
| `sonnet-4-5`          | claude-sonnet-4-5-20250929 | —                                                    |
| `sonnet-4-5-thinking` | claude-sonnet-4-5-20250929 | `thinking: { type: "enabled", budget_tokens: 5000 }` |
| `opus-4-6`            | claude-opus-4-6            | —                                                    |
| `sonnet-4-6`          | claude-sonnet-4-6          | —                                                    |
| `gpt-4.1`             | gpt-4.1                    | —                                                    |
| `gpt-4.1-mini`        | gpt-4.1-mini               | —                                                    |
| `o3-mini`             | o3-mini                    | —                                                    |

---

## 轨迹解析：parseTrajectory()

```typescript
// src/matchers.ts:122
export function parseTrajectory(
  messages: unknown[],
  files?: Record<string, unknown>,
): AgentTrajectory {
  const steps: AgentStep[] = [];
  let currentStep: AgentStep | null = null;

  for (const msg of messages.slice(1)) {  // 跳过第 0 项（HumanMessage）
    if (AIMessage.isInstance(msg)) {
      if (currentStep != null) steps.push(currentStep);
      currentStep = {
        index: steps.length + 1,
        action: msg,
        observations: [],
      };
    } else if (ToolMessage.isInstance(msg)) {
      if (currentStep != null) {
        currentStep.observations.push(msg);
      }
    }
  }

  if (currentStep != null) steps.push(currentStep);

  return { steps, files: coerceFiles(files) };
}
```

**算法说明：**

这是一个标准的"流扫描分组"算法：

```
状态机：
  IDLE → 遇到 AIMessage → NEW_STEP（创建新 step）
  NEW_STEP → 遇到 AIMessage → 先 push 旧 step，NEW_STEP（创建新 step）
  NEW_STEP → 遇到 ToolMessage → 追加到当前 step.observations
  
扫描结束后 → push 最后一个 step
```

**为什么跳过 index 0？**  
LangGraph 总是把原始用户输入（`HumanMessage`）放在 `messages[0]`，这不属于代理的"行为"，所以用 `.slice(1)` 跳过。

---

## 文件规范化：coerceFiles()

```typescript
// src/matchers.ts:160
function coerceFiles(raw: unknown): Record<string, string> {
  if (raw == null) return {};
  
  const files: Record<string, string> = {};
  for (const [path, value] of Object.entries(raw as Record<string, unknown>)) {
    if (typeof value === "string") {
      files[path] = value;  // 已经是字符串，直接用
    } else if (typeof value === "object" && "content" in value) {
      // FileData 格式：{ content: string[] } → 拼接为单字符串
      const content = (value as { content: string[] }).content;
      files[path] = Array.isArray(content) ? content.join("\n") : String(content);
    } else {
      throw new TypeError(`Unexpected file representation for ${path}: ${typeof value}`);
    }
  }
  return files;
}
```

该函数处理两种输入格式，支持 LangGraph 不同版本的输出。

---

## 辅助函数

### getFinalText() — 提取最后步骤的文本

```typescript
// src/matchers.ts:212
export function getFinalText(trajectory: AgentTrajectory): string {
  if (trajectory.steps.length === 0) return "";
  const last = trajectory.steps[trajectory.steps.length - 1];
  return typeof last.action.content === "string" ? last.action.content : "";
}
```

用于 `toHaveFinalTextContaining()` 断言。注意：`AIMessage.content` 可能是字符串或对象数组（多模态内容），这里只处理字符串情况。

### getToolCallCount() — 统计工具调用总数

```typescript
// src/matchers.ts:219
function getToolCallCount(trajectory: AgentTrajectory): number {
  return trajectory.steps.reduce(
    (sum, step) => sum + (step.action.tool_calls?.length ?? 0),
    0,
  );
}
```

### prettyTrajectory() — 调试输出格式化

```typescript
// src/matchers.ts:187
function prettyTrajectory(trajectory: AgentTrajectory): string {
  // 生成格式：
  // step 1:
  //   - write {"path": "main.ts", ...}
  // step 2:
  //   text: 任务已完成...
}
```

仅在断言失败时用于生成错误消息，帮助开发者快速定位问题。
