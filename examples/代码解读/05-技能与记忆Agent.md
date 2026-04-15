# 技能与记忆 Agent

**文件**：`skills-memory/skills-memory-agent.ts`

综合演示 **Skills（技能）** 和 **AgentMemory（代理记忆）** 两个机制的协同使用，同时展示了如何在代码中动态创建 SKILL.md 文件。

---

## 核心架构

```
createDeepAgent({
    model,
    backend: FilesystemBackend({ rootDir: tempDir, virtualMode: true }),
    skills: ["/skills/"],           // 从后端加载技能目录
    middleware: [memoryMiddleware],  // 添加 AgentMemory 中间件
})
```

---

## 代码解读

### 1. 环境检测（createSettings）

```typescript
const settings = createSettings();

console.log(settings.userDeepagentsDir);         // ~/.deepagents
console.log(settings.projectRoot);               // /path/to/project（检测 .git）
console.log(settings.getUserSkillsDir("my-agent")); // ~/.deepagents/my-agent/skills
console.log(settings.getProjectSkillsDir());     // /project/.deepagents/skills
console.log(settings.getUserAgentMdPath("my-agent")); // ~/.deepagents/my-agent/AGENTS.md
```

`createSettings()` 通过向上查找 `.git` 自动检测项目根目录，返回所有标准路径。

### 2. 动态创建临时技能

```typescript
function createSampleSkills(skillsDir: string): void {
  // web-research 技能
  const webResearchDir = path.join(skillsDir, "web-research");
  fs.mkdirSync(webResearchDir, { recursive: true });
  fs.writeFileSync(path.join(webResearchDir, "SKILL.md"), `---
name: web-research
description: Structured approach to conducting thorough web research
---

# Web Research Skill

## When to Use
- User asks you to research a topic
...
`);

  // code-review 技能
  const codeReviewDir = path.join(skillsDir, "code-review");
  fs.mkdirSync(codeReviewDir, { recursive: true });
  fs.writeFileSync(path.join(codeReviewDir, "SKILL.md"), `---
name: code-review
description: Systematic code review process with best practices
---
...
`);
}
```

**目录结构**：
```
tempDir/
└── skills/
    ├── web-research/
    │   └── SKILL.md
    └── code-review/
        └── SKILL.md
```

### 3. AgentMemory 中间件

```typescript
const memoryMiddleware = createAgentMemoryMiddleware({
  settings,
  assistantId: AGENT_NAME,   // "my-agent"
});
```

`createAgentMemoryMiddleware` 是与 `memory` 参数不同的记忆机制：

- **`memory` 参数**：只读，加载外部 AGENTS.md 文件
- **`createAgentMemoryMiddleware`**：读写，Agent 可以在对话中**主动写入**记忆，持久化到 `~/.deepagents/my-agent/AGENTS.md`

### 4. Agent 创建

```typescript
const agent = await createDeepAgent({
  model,
  backend: new FilesystemBackend({
    rootDir: tempDir,
    virtualMode: true,   // 虚拟模式：路径相对于 rootDir 计算
  }),
  skills: ["/skills/"],   // 路径相对于 backend rootDir
  middleware: [memoryMiddleware],
});
```

**virtualMode 说明**：
- 开启时：所有文件路径视为相对于 `rootDir` 的虚拟路径
- 关闭时：路径映射到实际文件系统绝对路径

---

## SKILL.md 格式详解

每个技能必须包含 YAML frontmatter：

```markdown
---
name: web-research                 # 必须：技能唯一标识符（kebab-case）
description: Structured approach   # 必须：描述（最大 1024 字符）
---

# 技能标题

## When to Use
- 触发条件描述

## Workflow / How to Use
- 具体操作步骤

## Best Practices
- 注意事项
```

**关键约束**：
- `name`：kebab-case，最大 64 字符
- `description`：最大 1024 字符，LLM 用它决定是否使用该技能
- 文件大小：最大 10MB

---

## Skills 渐进式加载机制

技能**不会**一次性全部加载到上下文，而是：

1. Agent 启动时只加载所有技能的 `name + description`（元数据）
2. LLM 看到技能列表，自行决定需要哪个
3. 当 LLM 决定使用某技能时，完整的 SKILL.md 内容才被加载

这样大量技能不会撑爆上下文窗口。

---

## 生产环境推荐目录结构

```
~/.deepagents/
└── my-agent/
    ├── AGENTS.md          # 用户级记忆（AgentMemory 写入此处）
    └── skills/
        ├── web-research/
        │   └── SKILL.md
        └── code-review/
            └── SKILL.md

/project/
└── .deepagents/
    ├── AGENTS.md          # 项目级记忆
    └── skills/
        └── project-specific-skill/
            └── SKILL.md
```

```typescript
// 生产用法
const agent = createDeepAgent({
  backend: new FilesystemBackend({ rootDir: os.homedir() }),
  skills: [
    settings.getUserSkillsDir("my-agent"),   // 用户技能
    settings.getProjectSkillsDir(),           // 项目技能（后者覆盖同名技能）
  ],
  middleware: [memoryMiddleware],
});
```

---

*[← 记忆与上下文 Agent](./04-记忆与上下文Agent.md) | [层级嵌套 Agent →](./06-层级嵌套Agent.md)*
