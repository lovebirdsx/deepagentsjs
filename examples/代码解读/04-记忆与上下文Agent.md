# 记忆与上下文 Agent

**文件**：`memory/memory-agent.ts`，`memory/AGENTS.md`

演示如何通过 `memory` 参数加载 AGENTS.md 文件，在 Agent 启动时将项目上下文注入系统提示，让 Agent 无需用户解释就知道项目规范。

---

## AGENTS.md 文件

`memory/AGENTS.md` 是 [agents.md 规范](https://agents.md/) 的一个示例：

```markdown
# Project Context

## Build Commands
- **Install dependencies**: `npm install`
- **Build**: `npm run build`
- **Test**: `npm test`

## Code Style Guidelines
- Use TypeScript for all new code
- Follow ESLint rules in the project
- Prefer functional programming patterns
- Use descriptive variable names

## Architecture Notes
This project uses a middleware-based architecture where:
- Middleware can modify agent behavior at different stages
- State flows through the middleware chain
- Each middleware can contribute to the system prompt

## Important Reminders
- Always check for existing implementations before creating new ones
- Write tests for new functionality
- Document public APIs with JSDoc comments
```

**格式特点**：
- 纯 Markdown，无特殊格式要求
- 用 `##` 组织章节（构建命令、代码风格、架构说明、注意事项）
- 简洁、具体，避免冗长

---

## 核心代码解读

```typescript
const exampleDir = path.dirname(new URL(import.meta.url).pathname);

// 1. 创建 FilesystemBackend，让框架能读取 AGENTS.md
const backend = new FilesystemBackend({
  rootDir: exampleDir,   // 以 examples/memory/ 为根目录
});

// 2. 创建带记忆的 Agent
const agent = createDeepAgent({
  model: "claude-haiku-4-5-20251001",
  systemPrompt: `You are a helpful coding assistant.
    When asked about project context, code style, or build commands,
    refer to the memory that was loaded from AGENTS.md files.`,
  backend,
  // memory 参数：AGENTS.md 文件路径数组（支持多个，按顺序合并）
  memory: [
    path.join(exampleDir, "AGENTS.md"),
  ],
});
```

---

## 关键参数说明

### `backend`

必须配置，用于让框架读取文件系统上的 AGENTS.md：

```typescript
new FilesystemBackend({
  rootDir: exampleDir,   // 文件读写的根目录
})
```

如果不配置 backend，默认使用 `StateBackend`（内存状态），无法读取磁盘文件。

### `memory: string[]`

接受**绝对路径**数组，指向 AGENTS.md 文件：

```typescript
memory: [
  "/home/user/.deepagents/AGENTS.md",    // 用户级（全局）
  "/project/AGENTS.md",                  // 项目级（局部）
]
```

- 多个文件按**顺序合并**（后面的文件内容追加在前面之后）
- 在 Agent 启动时**一次性加载**，注入系统提示
- 与 Skills 的区别：Memory 是**永远可用**的，Skills 是**按需披露**的

---

## 加载机制（内部原理）

```
createDeepAgent({ memory: [paths] })
    ↓
createMemoryMiddleware({ backend, sources: paths })
    ↓ 在 beforeAgent 钩子触发时
backend.read(path) → 读取每个 AGENTS.md 文件内容
    ↓
将所有内容合并 → 追加到系统提示末尾
    ↓
LLM 在每次推理时都能看到这些上下文
```

---

## 多 AGENTS.md 文件的典型用法

```typescript
const settings = createSettings();   // 自动检测项目根目录

const agent = createDeepAgent({
  backend: new FilesystemBackend({ rootDir: workspaceDir }),
  memory: [
    // 用户级：全局规范，适用于所有项目
    settings.getUserAgentMdPath("my-agent"),      // ~/.deepagents/my-agent/AGENTS.md

    // 项目级：本项目特有的上下文
    settings.getProjectAgentMdPath(),              // /project/.deepagents/AGENTS.md
  ],
});
```

通过 `createSettings()` 可以自动获取标准路径，无需硬编码。

---

## 与 Skills 的区别

| 对比点 | Memory（AGENTS.md） | Skills（SKILL.md） |
|--------|--------------------|--------------------|
| 加载时机 | Agent 启动时全量加载 | 按需（LLM 决定用哪个） |
| 上下文占用 | 始终占用 | 只有被选中的技能占用 |
| 适合内容 | 项目上下文、规范、架构 | 具体操作流程、工具用法 |
| 修改生效 | 下次 Agent 启动 | 实时（每次推理前检查） |

---

*[← 研究型 Agent](./03-研究型Agent.md) | [技能与记忆 Agent →](./05-技能与记忆Agent.md)*
