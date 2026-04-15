# 研究型 Agent

**文件**：`research/research-agent.ts`

这是 examples 中最完整的 Agent 示例，展示了一个**深度研究报告生成系统**：主 Agent 调度两个专用子代理（研究者 + 评审者），协作完成从搜索→写作→评审→修改的完整流程。

---

## 架构图

```
主 Agent（研究协调员）
    ├── 工具: internet_search（Tavily）
    ├── 子代理: research-agent（研究者）
    │   └── 工具: internet_search
    └── 子代理: critique-agent（评审者）
        └── 工具: (无，只读文件)
```

执行流程：
```
用户问题
    → 主 Agent 保存到 question.txt
    → 委派 research-agent 调研各子话题（并行）
    → 主 Agent 综合写 final_report.md
    → 委派 critique-agent 评审报告
    → 主 Agent 根据反馈修改（可多轮）
    → 输出最终报告
```

---

## 核心代码解读

### 1. 自定义搜索工具

```typescript
const internetSearch = tool(
  async ({ query, maxResults = 5, topic = "general", includeRawContent = false }) => {
    const tavilySearch = new TavilySearch({
      maxResults,
      tavilyApiKey: process.env.TAVILY_API_KEY,
      includeRawContent,
      topic,            // "general" | "news" | "finance"
    });
    return await tavilySearch._call({ query });
  },
  {
    name: "internet_search",
    schema: z.object({
      query: z.string(),
      maxResults: z.number().optional().default(5),
      topic: z.enum(["general", "news", "finance"]).optional().default("general"),
      includeRawContent: z.boolean().optional().default(false),
    }),
  },
);
```

**关键设计**：工具用 `zod` 定义 schema，LLM 可以自行决定搜索的 topic 类型和结果数量。

### 2. 研究者子代理

```typescript
const researchSubAgent: SubAgent = {
  name: "research-agent",
  description:
    "Used to research more in depth questions. Only give this researcher one topic at a time. " +
    "Do not pass multiple sub questions to this researcher. Instead, break down a large topic " +
    "into components, and call multiple research agents in parallel, one for each sub question.",
  systemPrompt: `You are a dedicated researcher. Conduct thorough research and then reply 
    with a detailed answer. Only your FINAL answer will be passed on to the user.`,
  tools: [internetSearch],   // 研究者有搜索工具
};
```

**重要**：description 明确告诉主 Agent 要**一次只给一个话题**，鼓励并行调用。这是利用 LLM 做编排决策的核心技巧。

### 3. 评审者子代理

```typescript
const critiqueSubAgent: SubAgent = {
  name: "critique-agent",
  description: "Used to critique the final report.",
  systemPrompt: `You are a dedicated editor. Check:
    - Section naming appropriateness
    - Text-heavy (not just bullet points)
    - Comprehensiveness and completeness
    - Deep analysis of causes, impacts, trends
    - Closely follows research topic
    - Clear structure and fluent language`,
  // 无 tools：评审者通过文件系统工具读取 final_report.md
};
```

**设计亮点**：评审者没有搜索工具，但有 deepagents 内置的文件系统工具（`read_file`），所以它能直接读 `final_report.md` 进行评审，不需要主 Agent 传入报告内容。

### 4. 主 Agent 系统提示关键片段

```typescript
const researchInstructions = `
The first thing you should do is write the original user question to question.txt.

Use the research-agent to conduct deep research. 
When you have enough information, write it to final_report.md.
You can call the critique-agent to get a critique.
After that (if needed) do more research and edit final_report.md.
You can do this however many times until you are satisfied.

Only edit the file once at a time (if you call this tool in parallel, there may be conflicts).

CRITICAL: Make sure the answer is written in the SAME LANGUAGE as the human messages!
`;
```

**注意**：
- 明确要求先保存问题（便于子代理读取）
- 警告不要并行写同一个文件（避免竞态）
- 报告语言跟随用户语言（国际化）

### 5. Agent 创建

```typescript
export const agent = createDeepAgent({
  model: new ChatAnthropic({
    model: "claude-sonnet-4-20250514",
    temperature: 0,        // 研究任务用 temperature=0 保证确定性
  }),
  tools: [internetSearch],                          // 主 Agent 也有搜索工具
  systemPrompt: researchInstructions,
  subagents: [critiqueSubAgent, researchSubAgent],  // 两个子代理
});
```

---

## 报告格式规范（系统提示中内嵌）

主 Agent 的提示中包含了一套完整的报告写作规范：

| 规范项 | 要求 |
|--------|------|
| 标题格式 | `#` 为标题，`##` 为节，`###` 为小节 |
| 引用格式 | `[Title](URL)` 行内引用 |
| 末尾来源 | `### Sources` 列表，顺序编号 |
| 语言 | 与用户消息语言一致 |
| 内容深度 | 段落而非项目列表，足够详细 |

---

## 关键设计模式总结

### 文件系统作为通信总线

子代理之间通过文件传递信息，而非返回值：
- 研究者写 → `research_notes.md`
- 主 Agent 写 → `final_report.md`
- 评审者读 → `final_report.md`，再写 → `critique_notes.md`

这样可以：
1. 避免大量文本在 LLM 上下文中堆积
2. 子代理可以随时读取对方的中间结果
3. 最终文件持久化，用户可以直接获取

### 多轮迭代（ReAct 循环内的编辑循环）

主 Agent 可以多次执行「写报告 → 评审 → 修改」，没有固定次数，完全由 LLM 决定何时满意停止。这利用了 LangGraph 的 ReAct 执行模型。

### 并行子代理调用

主 Agent 被提示"一次给研究者一个话题，然后并行调用多次"，这样 3 个方向的研究可以同时进行，显著提升速度。

---

## 运行方式

```bash
# 设置环境变量
export ANTHROPIC_API_KEY=...
export TAVILY_API_KEY=...   # 可选，不设则无网络搜索

# 运行（通过 LangGraph CLI 部署，参考 langgraph.json）
npx @langchain/langgraph-cli dev -c research/langgraph.json
```

---

*[← 项目结构总览](./01-项目结构总览.md) | [记忆与上下文 Agent →](./04-记忆与上下文Agent.md)*
