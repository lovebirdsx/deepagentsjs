# skills：技能加载

**目录**：`evals/skills/`

7 个测试用例验证 SKILL.md 技能文件的发现、读取、组合与编辑能力。

---

## 背景

DeepAgents 的技能系统（Anthropic Agent Skills 规范）允许通过 Markdown 文件动态扩展 Agent 能力。每个技能文件有 YAML 前置元数据：

```markdown
---
name: skill-name
description: When to use this skill and what it does
---

## How to use this skill
具体操作步骤...
```

技能通过 `runner.extend({ skills: ["/path/to/skills/dir"] })` 加载。

---

## 测试用例

### 1. read skill full content（读取完整技能内容）

```typescript
runner
  .extend({ skills: ["/skills/base/"] })
  .run({
    query: "How do I deploy the application?",
    initialFiles: {
      "/skills/base/deploy.md": `---
name: deploy
description: How to deploy the application to production
---

## Deploy Steps
1. Run npm build
2. Push to main branch
3. CI/CD will handle the rest
`
    }
  });

// Agent 应从 skill 文件中读取并回答
expect(result).toHaveFinalTextContaining("npm build");
```

---

### 2. read skill by name（按名称读取特定技能）

```typescript
runner
  .extend({ skills: ["/skills/base/"] })
  .run({
    query: "Use the 'testing' skill to help me",
    initialFiles: {
      "/skills/base/testing.md": `---
name: testing
description: Run tests for the project
---

Run: npm test
`,
      "/skills/base/deploy.md": "... deploy skill ...",
    }
  });

// 应读取 testing skill，不应读取 deploy skill
expect(result).toHaveFinalTextContaining("npm test");
// 效率断言：只读了一个技能文件
```

---

### 3. combine two skills（组合两个技能）

```typescript
runner
  .extend({ skills: ["/skills/base/"] })
  .run({
    query: "I need to test and then deploy",
    initialFiles: {
      "/skills/base/testing.md": "...",
      "/skills/base/deploy.md": "...",
    }
  });

// 两个技能的内容都应被使用
expect(result).toHaveFinalTextContaining("npm test");
expect(result).toHaveFinalTextContaining("npm build");
```

---

### 4. update skill typo fix no read（无需读取的技能编辑）

```typescript
// 用户明确指出了错误所在，Agent 无需先读取文件
runner
  .extend({ skills: ["/skills/base/"] })
  .run({
    query: "Fix the typo in the deploy skill: change 'buld' to 'build'",
    initialFiles: {
      "/skills/base/deploy.md": "Run npm buld",
    }
  });

// 直接编辑，不需要先 read_file
expect(result.files["/skills/base/deploy.md"]).toContain("npm build");
// 效率断言：不应先读取文件
expect(result).not.toHaveToolCallInStep(1, { name: "read_file" });
```

**考察**：当 Agent 拥有足够信息时，能直接使用 `edit_file` 而非先读后写。

---

### 5. update skill typo fix requires read（需要读取的技能编辑）

```typescript
runner
  .extend({ skills: ["/skills/base/"] })
  .run({
    query: "There's a typo in the deploy skill. Find and fix it.",
    initialFiles: {
      "/skills/base/deploy.md": "Run npm buld  # buld should be build",
    }
  });

// 必须先读取才能找到错误
expect(result).toHaveToolCallInStep(1, { name: "read_file" });
expect(result.files["/skills/base/deploy.md"]).toContain("npm build");
```

---

### 6. find skill in correct path（在正确路径找技能）

```typescript
// 存在多个技能目录时，在正确目录操作
runner
  .extend({ skills: ["/skills/base/", "/skills/project/"] })
  .run({
    query: "Update the deploy skill in the project directory to add a step",
    initialFiles: {
      "/skills/base/deploy.md": "Base deploy: npm build",
      "/skills/project/deploy.md": "Project deploy: npm run deploy",
    }
  });

// 应修改 /skills/project/deploy.md，而非 /skills/base/deploy.md
expect(result.files["/skills/project/deploy.md"]).toContain("new step");
expect(result.files["/skills/base/deploy.md"]).toBe("Base deploy: npm build");
```

---

## SKILL.md 规范

### 前置元数据字段

| 字段 | 必需 | 约束 |
|------|------|------|
| `name` | **必需** | 1-64 字符，小写字母和连字符 |
| `description` | **必需** | 1-1024 字符，描述何时使用 |
| `license` | 可选 | 开源协议 |
| `compatibility` | 可选 | 环境要求 |

### 技能发现机制

```
1. 从 skills 参数指定的目录路径扫描
2. 递归查找所有 *.md 文件
3. 解析 YAML 前置元数据
4. 将技能名称+描述注入系统提示
5. Agent 通过 read_file 工具按需读取完整内容
```

---

## 参考链接

- [memory：类似的配置注入机制 →](./07-suite-memory.md)
- [external-benchmarks →](./17-suite-external-benchmarks.md)
