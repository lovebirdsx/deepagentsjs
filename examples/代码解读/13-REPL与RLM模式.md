# REPL 与 RLM 模式

**文件**：
- `repl/data-analysis-agent.ts` — QuickJS REPL 数据分析
- `repl/rlm-agent.ts` — 递归语言模型（RLM）并行子代理编排

这两个示例都使用 `@langchain/quickjs` 包提供的 QuickJS 沙箱 REPL，让 Agent 可以在安全环境中执行 JavaScript 代码。

---

## QuickJS 沙箱特性

- **完全隔离**：无网络访问、无 Node.js API（`fs`、`http` 等不可用）
- **轻量**：QuickJS 是纯 JS 实现的 JS 引擎，启动开销极低
- **完整 ES2020+**：支持 async/await、Promise、解构等现代语法
- **用途**：数学计算、数据变换、JSON 处理、逻辑运算——任何不需要外部资源的任务

---

## data-analysis-agent.ts — 数据分析 Agent

### 代码解读

```typescript
import { createQuickJSMiddleware } from "@langchain/quickjs";

const agent = createDeepAgent({
  model: new ChatAnthropic({ model: "claude-sonnet-4-5", temperature: 0 }),
  systemPrompt: dedent`
    You are a data analyst. Use the js_eval REPL to perform calculations
    and data transformations. Always show your work in code — never guess
    at arithmetic or statistics.
  `,
  middleware: [createQuickJSMiddleware()],  // 添加 QuickJS 中间件
});

await agent.invoke({
  messages: [new HumanMessage(dedent`
    I have sales data in /data/sales.json. Parse it, calculate the total
    revenue per region, find the top-performing region, and write a
    summary report to /reports/sales-summary.md.
  `)],
});
```

### Agent 的工作流程

```
1. read_file("/data/sales.json")          → 获取原始 JSON
2. js_eval(`                               → 在 QuickJS 中计算
    const data = JSON.parse(salesJson);
    const byRegion = {};
    data.forEach(row => {
      byRegion[row.region] = (byRegion[row.region] || 0) + row.revenue;
    });
    const topRegion = Object.entries(byRegion)
      .sort(([,a],[,b]) => b - a)[0];
    return { byRegion, topRegion };
   `)
3. write_file("/reports/sales-summary.md") → 写入结果
```

**设计价值**：LLM 在数学计算上容易出错（幻觉），通过 REPL 执行真实代码，保证计算准确性。

---

## rlm-agent.ts — 递归语言模型（RLM）

这是 `repl/` 中最有创新性的示例，实现了 **RLM（Recursive Language Model）** 模式。

### RLM 核心思想

传统方式：LLM 逐个调用子代理（串行）
```
LLM → task(researcher, "topic A")
LLM → task(researcher, "topic B")  // 等 A 完成后才执行
LLM → task(researcher, "topic C")  // 等 B 完成后才执行
```

RLM 方式：LLM 写代码，代码并行调用子代理
```
LLM → js_eval(`
  const results = await Promise.all([
    tools.task({ description: "topic A" }),
    tools.task({ description: "topic B" }),
    tools.task({ description: "topic C" }),
  ]);
  // 在代码中处理结果
`)
```

### 代码解读

```typescript
const agent = createDeepAgent({
  model: new ChatOpenAI("gpt-5.2"),
  systemPrompt: dedent`
    You are a research analyst that uses code to orchestrate sub-agents.

    **CRITICAL: Always use Promise.all to spawn sub-agents in parallel.**
    Never call tools.task() sequentially — always use Promise.all.

    When given a complex research task:
    1. Break it into independent sub-tasks
    2. Write a single js_eval call that spawns ALL sub-agents in parallel
    3. Aggregate results programmatically in the same code block
    4. Write your final synthesis to a file

    Example:
    \`\`\`
    const topics = ["topic A", "topic B", "topic C"];
    const results = await Promise.all(
      topics.map(topic =>
        tools.task({
          description: \`Research \${topic} in depth.\`,
          subagentType: "general-purpose",
        })
      )
    );
    const report = topics.map((t, i) => \`## \${t}\\n\${results[i]}\`).join("\\n\\n");
    await writeFile("/research.md", report);
    \`\`\`
  `,
  subagents: [generalPurpose],
  middleware: [
    createQuickJSMiddleware({
      ptc: ["task"],   // PTC：Programmatic Tool Calling，允许从 REPL 调用 task 工具
    }),
  ],
});
```

### PTC（Programmatic Tool Calling）

`ptc: ["task"]` 是 QuickJS 中间件的关键配置，它将指定的 deepagents 工具暴露给 REPL 沙箱：

```javascript
// 在 QuickJS 内部，tools 对象包含 ptc 配置的工具
const result = await tools.task({
  description: "Research renewable energy in Germany",
  subagentType: "general-purpose",
});
// result 是子代理的最终回复文字
```

`await writeFile(path, content)` 也在 REPL 中可用，用于写文件。

### 执行流程

```
Agent 收到"比较德国、中国、美国可再生能源政策"
    ↓
LLM 生成一次 js_eval 调用
    ↓
QuickJS 执行：
  Promise.all([
    tools.task("研究德国政策"),   ─┐
    tools.task("研究中国政策"),    ├─ 并行执行（3个子代理同时运行）
    tools.task("研究美国政策"),   ─┘
  ])
    ↓（等待全部完成）
代码聚合结果 → writeFile("/analysis.md", report)
    ↓
Agent 完成
```

与传统方式相比，3 个并行研究任务的总时间约等于 1 个任务的时间，速度提升 3 倍。

---

## 两个示例的对比

| 特性 | data-analysis-agent | rlm-agent |
|------|--------------------|-----------| 
| REPL 用途 | 数值计算、数据变换 | 并行子代理编排 |
| ptc 配置 | 无（REPL 只做计算） | `["task"]`（REPL 可调用子代理）|
| 核心价值 | 避免 LLM 数学幻觉 | 极致并行化 |
| 模型 | Claude（temperature=0） | GPT-5.2 |

---

## 何时使用 QuickJS REPL

**适合**：
- 精确数值计算（统计、聚合、排序）
- JSON 数据变换（提取、过滤、重组）
- 需要并行调用多个子代理的编排逻辑
- 复杂的多步逻辑（中间有分支判断）

**不适合**：
- 需要网络请求（用子代理替代）
- 需要文件系统操作（用 deepagents 内置工具）
- 复杂的 Node.js 生态（用沙箱后端替代）

---

*[← 流式输出完全指南](./12-流式输出完全指南.md) | [技能文件解读 →](./14-技能文件解读.md)*
