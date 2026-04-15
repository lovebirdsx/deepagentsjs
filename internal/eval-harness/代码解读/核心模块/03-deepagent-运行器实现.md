# deepagent — 运行器实现

**对应源文件：** `src/deepagent.ts`（94 行）

这个文件包含两件事：
1. `DeepAgentEvalRunner` 类——`EvalRunner` 接口的具体实现
2. `registerDeepAgentRunner()` 便利函数——简化注册流程

---

## DeepAgentEvalRunner 类

```typescript
// src/deepagent.ts:19
export class DeepAgentEvalRunner implements EvalRunner {
  name: string;
  private agent: DeepAgent;

  constructor(
    name: string,
    private factory: (config?: Partial<CreateDeepAgentParams>) => DeepAgent,
    overrides?: Record<string, unknown>,
  ) {
    this.name = name;
    this.agent = factory(overrides);  // 构造时立即创建默认代理
  }
  // ...
}
```

### 构造函数设计

构造函数接受三个参数：
- `name`：运行器标识（如 `"sonnet-4-5"`）
- `factory`：代理工厂函数，接受可选的覆盖配置并返回 `DeepAgent` 实例
- `overrides`（可选）：本次实例的配置覆盖

**关键设计：** 构造时立即调用 `factory(overrides)` 创建代理，并缓存为 `this.agent`。这意味着无覆盖的"基础运行器"只创建一次代理，后续 `run()` 调用复用同一实例。

---

## extend() — 配置覆盖机制

```typescript
// src/deepagent.ts:32
extend(overrides: Record<string, unknown>): EvalRunner {
  return new DeepAgentEvalRunner(this.name, this.factory, overrides);
}
```

**这是框架最重要的设计模式之一。**

`extend()` 创建一个新的 `DeepAgentEvalRunner` 实例，但传入了 `overrides`。新实例在构造时会调用 `factory(overrides)`，产生一个配置不同的代理。

使用场景：

```typescript
const baseRunner = getRunner("sonnet-4-5");

// 测试 1：默认配置
const traj1 = await baseRunner.run({ query: "..." });

// 测试 2：自定义系统提示
const customRunner = baseRunner.extend({
  systemPrompt: "你只能用 Python 回答问题"
});
const traj2 = await customRunner.run({ query: "..." });

// 两个 runner 互不干扰，共享同一个 factory 函数
```

---

## run() — 核心执行方法

```typescript
// src/deepagent.ts:36
async run(params: RunAgentParams): Promise<AgentTrajectory> {
  // 1. 构建 LangGraph 输入
  const inputs: Record<string, unknown> = {
    messages: [{ role: "user", content: params.query }],
  };

  // 2. 处理初始文件
  if (params.initialFiles != null) {
    const files: Record<string, unknown> = {};
    for (const [filePath, content] of Object.entries(params.initialFiles)) {
      const now = new Date().toISOString();
      files[filePath] = {
        content: content.split("\n"),  // 按行分割
        created_at: now,
        modified_at: now,
      };
    }
    inputs.files = files;
  }

  // 3. 生成唯一线程 ID
  const threadId = uuidv4();
  const config = { configurable: { thread_id: threadId } };

  // 4. 调用 LangGraph
  const result = await this.agent.invoke(inputs, config);

  // 5. 解析并返回轨迹
  if (typeof result !== "object" || result == null) {
    throw new TypeError(`Expected invoke result to be object, got ${typeof result}`);
  }
  const r = result as Record<string, unknown>;
  return parseTrajectory(
    r.messages as unknown[],
    r.files as Record<string, unknown>,
  );
}
```

### 步骤详解

**步骤 2：初始文件格式转换**

测试代码传入简洁的字符串格式，但 LangGraph 需要 `FileData` 格式：

```typescript
// 测试代码传入（简洁）
initialFiles: {
  "src/main.ts": "line1\nline2\nline3"
}

// 转换后传给 LangGraph（FileData 格式）
files: {
  "src/main.ts": {
    content: ["line1", "line2", "line3"],  // split("\n")
    created_at: "2024-01-01T00:00:00.000Z",
    modified_at: "2024-01-01T00:00:00.000Z"
  }
}
```

**步骤 3：UUID 线程 ID**

```typescript
const threadId = uuidv4();  // 如 "550e8400-e29b-41d4-a716-446655440000"
```

每次 `run()` 调用生成一个新的 UUID。这是 LangGraph 的对话隔离机制——相同的 `thread_id` 会共享对话历史，不同的 `thread_id` 是独立的会话。评估测试中每次调用都需要全新的独立会话，所以用 UUID 保证唯一性。

**步骤 4：调用 LangGraph**

```typescript
const result = await this.agent.invoke(inputs, config);
```

`this.agent` 是一个 LangGraph 编译后的图（CompiledGraph），`invoke()` 是同步执行整个图直到终止状态。`result` 包含：
- `result.messages`：完整消息列表（包括中间步骤）
- `result.files`：执行结束时的文件系统状态

**步骤 5：错误处理**

```typescript
if (typeof result !== "object" || result == null) {
  throw new TypeError(`Expected invoke result to be object, got ${typeof result}`);
}
```

简单的类型守卫，防止 LangGraph 返回意外类型（理论上不会发生，但这是防御性编程）。

---

## registerDeepAgentRunner() — 便利注册函数

```typescript
// src/deepagent.ts:88
export function registerDeepAgentRunner(
  name: string,
  factory: (config?: Record<string, unknown>) => DeepAgent,
): void {
  registerRunner(new DeepAgentEvalRunner(name, factory));
}
```

这是一个薄薄的包装函数，把"创建 + 注册"两步合并为一步。`src/runners.ts` 中的 7 次调用都通过它完成。

**如果不用这个函数，等价写法：**

```typescript
// 等价于
registerRunner(
  new DeepAgentEvalRunner("sonnet-4-5", (config) =>
    createDeepAgent({
      ...config,
      model: new ChatAnthropic({ model: "claude-sonnet-4-5-20250929" }),
    })
  )
);
```

---

## 类型依赖关系

```
deepagent.ts 依赖：
  ├── "uuid"                    → uuidv4()
  ├── "./index.js"              → registerRunner, parseTrajectory, EvalRunner, ...
  └── "deepagents"              → CreateDeepAgentParams, DeepAgent
      └── (deepagents 包提供 createDeepAgent 工厂和 DeepAgent 类型)
```

注意 `deepagent.ts` 从 `"./index.js"` 导入——这是循环引用的临界点，但因为 `index.ts` 只做重导出（`export * from "./matchers.js"`），所以运行时不会有问题。
