# ACP 服务器集成

**文件**：`acp-server/server.ts`

演示如何用 `deepagents-acp` 包启动一个 **Agent Client Protocol (ACP)** 服务器，让 IDE（Zed、JetBrains 等）直接调用 DeepAgent 作为 AI 编程助手。

---

## 核心代码

```typescript
import { DeepAgentsServer } from "deepagents-acp";
import { FilesystemBackend } from "deepagents";

const workspaceRoot = process.env.WORKSPACE_ROOT ?? process.cwd();

const server = new DeepAgentsServer({
  agents: [
    {
      name: "coding-assistant",
      description: "AI coding assistant with full filesystem access...",

      model: "claude-sonnet-4-5-20250929",

      // 文件系统后端：读写工作区真实文件
      backend: new FilesystemBackend({ rootDir: workspaceRoot }),

      // 技能加载（从工作区查找）
      skills: [
        path.join(workspaceRoot, ".deepagents", "skills"),
        path.join(workspaceRoot, "skills"),
      ],

      // 记忆文件
      memory: [
        path.join(workspaceRoot, ".deepagents", "AGENTS.md"),
        path.join(workspaceRoot, "AGENTS.md"),
      ],

      systemPrompt: `You are an AI coding assistant integrated with an IDE through ACP.
        Workspace: ${workspaceRoot}
        1. First understand the codebase structure
        2. Make a plan before making changes
        3. Test your changes when possible`,
    },
  ],

  serverName: "deepagents-acp-server",
  serverVersion: "0.0.1",
  workspaceRoot,
  debug: process.env.DEBUG === "true",
});

server.start();
```

---

## 关键设计点

### 1. 文件系统后端 = 真实文件操作

```typescript
backend: new FilesystemBackend({ rootDir: workspaceRoot })
```

与 StateBackend（内存）不同，`FilesystemBackend` 让 Agent 的 `write_file`、`edit_file` 等工具直接操作磁盘上的真实代码文件。这对 IDE 集成至关重要。

### 2. 工作区感知的技能和记忆

```typescript
skills: [
  path.join(workspaceRoot, ".deepagents", "skills"),  // 项目私有技能
  path.join(workspaceRoot, "skills"),                  // 通用技能
],
memory: [
  path.join(workspaceRoot, ".deepagents", "AGENTS.md"), // 项目上下文
  path.join(workspaceRoot, "AGENTS.md"),               // 根目录上下文
],
```

Agent 自动加载工作区根目录下的 `AGENTS.md`，了解项目规范后开始工作。

### 3. 单行启动

`DeepAgentsServer` 封装了所有 ACP 协议实现，开发者只需：
1. 定义 Agent 配置
2. 调用 `server.start()`

---

## IDE 配置示例（Zed）

```json
{
  "agent": {
    "profiles": {
      "deepagents": {
        "name": "DeepAgents",
        "command": "npx",
        "args": ["tsx", "examples/acp-server/server.ts"],
        "cwd": "/path/to/deepagentsjs"
      }
    }
  }
}
```

IDE 通过标准 IO 与服务器通信（stdin/stdout），服务器将消息转换为 DeepAgent 调用。

---

## ACP 与其他接入方式对比

| 方式 | 通信协议 | 适用场景 |
|------|---------|---------|
| ACP Server | 标准 IO（IDE 集成） | Zed/JetBrains 插件 |
| Agent Protocol Server | HTTP REST | 远程异步子代理 |
| LangGraph Platform | LangGraph SDK | 多图部署 |
| 直接 `invoke()` | 函数调用 | Node.js 脚本、API 路由 |

---

*[← 自托管异步子代理服务器](./08-自托管异步子代理服务器.md) | [后端对比与选型 →](./10-后端对比与选型.md)*
