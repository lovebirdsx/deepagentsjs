# cli.ts — 命令行入口

**文件路径：** `src/cli.ts`  
**代码行数：** ~332 行  
**入口命令：** `npx deepagents-acp`（package.json `bin` 字段指向编译后的 `dist/cli.js`）

---

## 功能概述

CLI 是 `DeepAgentsServer` 的命令行包装器，支持通过命令行参数或环境变量配置 Agent，然后启动服务器。

---

## 参数格式支持

`normalizeArgs()` 函数统一处理三种参数格式：

```bash
# 标准格式（推荐）
--name my-agent

# 等号分隔格式
--name=my-agent

# 空格分隔（Zed 等 IDE 通过 args 数组传入时可能带空格）
"--name my-agent"  # 作为单个字符串
```

---

## 支持的参数

| 参数 | 短参数 | 环境变量 | 默认值 |
|---|---|---|---|
| `--name` | `-n` | - | `"deepagents"` |
| `--description` | `-d` | - | `"AI coding assistant powered by DeepAgents"` |
| `--model` | `-m` | - | `"claude-sonnet-4-5-20250929"` |
| `--workspace` | `-w` | `WORKSPACE_ROOT` | `process.cwd()` |
| `--skills` | `-s` | - | `.deepagents/skills`, `./skills` |
| `--memory` | - | - | `.deepagents/AGENTS.md`, `./AGENTS.md` |
| `--debug` | - | `DEBUG=true` | `false` |
| `--log-file` | `-l` | `DEEPAGENTS_LOG_FILE` | `null` |
| `--help` | `-h` | - | - |
| `--version` | `-v` | - | - |

---

## 默认路径自动发现

当 `--skills` / `--memory` 未指定时，自动在 workspaceRoot 下查找约定路径：

```typescript
const defaultSkillPaths = [
  path.join(workspaceRoot, ".deepagents", "skills"),
  path.join(workspaceRoot, "skills"),
]

const defaultMemoryPaths = [
  path.join(workspaceRoot, ".deepagents", "AGENTS.md"),
  path.join(workspaceRoot, "AGENTS.md"),
]
```

---

## `main()` 函数

```typescript
async function main() {
  const options = parseArgs(process.argv.slice(2))

  if (options.help)    { showHelp();    process.exit(0) }
  if (options.version) { showVersion(); process.exit(0) }

  // 解析 workspace（参数 > 环境变量 > cwd）
  const workspaceRoot = options.workspace || process.env.WORKSPACE_ROOT || process.cwd()

  // 组装技能/记忆路径
  const skills = options.skills.length > 0
    ? options.skills.map(p => path.resolve(workspaceRoot, p))
    : defaultSkillPaths

  // 启动 stderr 日志（stdout 保留给 ACP）
  if (options.debug || options.logFile) {
    console.error("[deepagents-acp] Starting...", { agent, model, workspace, skills })
  }

  // 创建并启动服务器
  const server = new DeepAgentsServer({
    agents: { name, description, model, backend, skills, memory },
    serverName: "deepagents-acp",
    workspaceRoot,
    debug: options.debug,
    logFile: options.logFile,
  })

  await server.start()  // 阻塞直到 stdin 关闭
}
```

**注意**：CLI 中硬编码了 `backend: new FilesystemBackend({ rootDir: workspaceRoot })`，如果 IDE 客户端支持 FS 能力，`DeepAgentsServer.createBackend()` 内部会在 `handleInitialize` 后升级为 `ACPFilesystemBackend`。

---

## 版本显示

```typescript
function showVersion() {
  const packageJson = JSON.parse(fs.readFileSync("../package.json"))
  console.log(`deepagents-acp v${packageJson.version}`)
}
```

从 `package.json` 动态读取版本号，避免硬编码。

---

## Zed 集成示例（来自 `--help` 输出）

```json
{
  "agent": {
    "profiles": {
      "deepagents": {
        "name": "DeepAgents",
        "command": "npx",
        "args": ["deepagents-acp", "--log-file", "/tmp/deepagents.log"],
        "env": {}
      }
    }
  }
}
```
