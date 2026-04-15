# index.ts — 入口与副作用导入

**对应源文件：** `src/index.ts`（6 行）

```typescript
// import to get proper namespace declarations
import "./matchers.js";

export * from "./deepagent.js";
export * from "./matchers.js";
export * from "./runners.js";
```

---

## 这 6 行做了什么

### 第 2 行：副作用导入

```typescript
import "./matchers.js";
```

这一行只导入模块的副作用（side effects），不导入任何具名导出。

`matchers.ts` 中有一个顶层调用：
```typescript
expect.extend({
  toHaveAgentSteps(...),
  toHaveToolCallRequests(...),
  toHaveToolCallInStep(...),
  toHaveFinalTextContaining(...),
});
```

这个 `expect.extend()` 调用在模块被导入时立即执行，将自定义 matcher 注册到 vitest 的全局 `expect` 对象中。

**为什么要 import 两次？**（第 2 行 `import "./matchers.js"` 和第 5 行 `export * from "./matchers.js"`）

- 第 5 行的 `export *` 会重导出 `matchers.ts` 的所有导出（类型定义等），**但不保证副作用**在某些构建工具中能被触发
- 第 2 行的裸 import 确保副作用（`expect.extend()`）一定被执行
- 注释说明了这个意图：`// import to get proper namespace declarations`

这是 TypeScript 项目中处理"带副作用模块"的惯用写法。

### 第 4~6 行：重导出

```typescript
export * from "./deepagent.js";  // DeepAgentEvalRunner, registerDeepAgentRunner
export * from "./matchers.js";   // AgentStep, AgentTrajectory, RunAgentParams, EvalRunner, 等
export * from "./runners.ts";    // registerRunner, getRunner, getDefaultRunner, parseTrajectory, getFinalText
```

将三个模块的所有公开导出聚合为单一入口，用户只需要：

```typescript
import { getDefaultRunner, AgentTrajectory } from "@deepagents/evals";
// 而不是
import { getDefaultRunner } from "@deepagents/evals/runners";
import { AgentTrajectory } from "@deepagents/evals/matchers";
```

---

## 导出清单

从 `@deepagents/evals` 可导入的所有名称：

**来自 deepagent.js：**
- `DeepAgentEvalRunner`（class）
- `registerDeepAgentRunner`（function）

**来自 matchers.js（类型）：**
- `AgentStep`（interface）
- `AgentTrajectory`（interface）
- `RunAgentParams`（interface）
- `EvalRunner`（interface）

**来自 runners.js（注意：这里是 matchers.ts 的前半段，命名混淆）：**
- `registerRunner`（function）
- `getRunner`（function）
- `getDefaultRunner`（function）
- `parseTrajectory`（function）
- `getFinalText`（function）

---

## 关于 `./setup` 导出路径

`package.json` 中有一个额外的导出路径：

```json
"./setup": {
  "import": {
    "types": "./dist/index.d.ts",
    "default": "./dist/index.js"
  }
}
```

这个路径指向同一个文件（`dist/index.js`），主要是为了支持 vitest 的 `setupFiles` 配置：

```typescript
// vitest.config.ts（使用此框架的项目中）
export default defineConfig({
  test: {
    setupFiles: ["@deepagents/evals/setup"],
  }
});
```

通过 `setupFiles` 引入时，`index.ts` 被执行，`import "./matchers.js"` 触发 `expect.extend()`，所有自定义 matcher 在测试开始前就已注册好。
