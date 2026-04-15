# memory-agent-bench：长上下文记忆

**目录**：`evals/memory-agent-bench/`

受 MemoryAgentBench 学术基准启发的评估套件，测试 Agent 在长上下文中的记忆化与检索能力。

---

## 测试用例

### 1. long context memorization（长上下文记忆化）

```typescript
// 查询中嵌入需要记忆的事实（混在大量无关文本中）
const query = `
  Read the following text and answer the question.
  
  [大量无关叙事文本...]
  Project Nightfall is assigned codename ECHO-7.
  [更多无关文本...]
  The project operates in the NORTHERN region.
  [更多无关文本...]
  
  What is the codename and region for Project Nightfall?
`;

runner.run({ query });

expect(result).toHaveFinalTextContaining("ECHO-7");
expect(result).toHaveFinalTextContaining("NORTHERN");
```

**考察**：从长文本中提取特定信息（信息密度低，需要注意力集中）。

---

### 2. conflict resolution latest fact wins（冲突消解：最新事实优先）

```typescript
const query = `
  Initial information: The server is running version 1.2.
  [其他内容...]
  Update: The server has been upgraded to version 2.0.
  
  What version is the server running?
`;

runner.run({ query });

// 应该回答 2.0，而非 1.2（更新的信息覆盖旧信息）
expect(result).toHaveFinalTextContaining("2.0");
```

**考察**：当同一实体有多条记录时，Agent 能识别"最新覆盖最旧"的语义。

---

### 3. file seeded retrieval mode（文件型检索模式）

```typescript
// 将信息拆分放入多个文件，模拟知识库场景
runner.run({
  query: "What is the escalation alias for tier-3 support?",
  initialFiles: {
    "/data/chunk1.txt": "Tier-1 support alias: support-basic@example.com",
    "/data/chunk2.txt": "Tier-2 support alias: support-plus@example.com",
    "/data/chunk3.txt": "Tier-3 support alias: support-enterprise@example.com",
  },
  // 系统提示指示使用文件
});

expect(result).toHaveFinalTextContaining("support-enterprise@example.com");
```

**考察**：Agent 能通过文件工具（glob/grep/read）在文件系统中检索答案。

---

## 与 memory 套件的区别

| 维度 | memory 套件 | memory-agent-bench 套件 |
|------|------------|------------------------|
| 记忆来源 | AGENTS.md 注入 | 查询文本或文件 |
| 测试重点 | 配置记忆的读取与影响 | 长上下文的理解与提取 |
| 对标基准 | 无特定基准 | MemoryAgentBench 学术基准 |

---

## 参考链接

- [memory：AGENTS.md 记忆 →](./07-suite-memory.md)
- [memory-multiturn：多轮记忆 →](./09-suite-memory-multiturn.md)
