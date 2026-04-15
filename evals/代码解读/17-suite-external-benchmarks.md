# external-benchmarks：外部基准

**目录**：`evals/external-benchmarks/`

从 FRAMES、Nexus、BFCL v3 三个学术/工业基准中精选的测试用例，验证 Agent 在标准化任务上的表现。

---

## 数据来源

```
evals/external-benchmarks/data/
└── curated_cases.json    ← 精选的测试用例集合
```

### 用例结构

```typescript
interface BenchmarkCase {
  benchmark: "frames" | "nexus" | "bfcl_v3";
  id: string;                          // 在原始基准中的唯一 ID
  prompt: string;                      // 测试提问
  files?: Record<string, string>;      // 预置文件（如果需要）
  answer_snippets: string[];           // 期望答案中应包含的片段
  difficulty?: string;                 // 难度等级（如 "easy" | "hard"）
}
```

---

## 三个基准简介

### FRAMES
- **来源**：Google DeepMind
- **特点**：需要多步骤推理和事实检索的问答任务
- **典型任务**：复杂因果推理、多跳问答

### Nexus
- **来源**：工业界工具使用基准
- **特点**：测试工具调用的准确性和参数传递
- **典型任务**：API 调用序列、工具链推理

### BFCL v3（Berkeley Function-Calling Leaderboard v3）
- **来源**：UC Berkeley
- **特点**：函数调用准确性基准
- **典型任务**：复杂 schema 参数解析、嵌套函数调用

---

## 断言逻辑

```typescript
// 加载所有用例
const cases: BenchmarkCase[] = JSON.parse(
  fs.readFileSync("./data/curated_cases.json", "utf-8")
);

for (const testCase of cases) {
  ls.test(`${testCase.benchmark}/${testCase.id}`, {
    inputs: { query: testCase.prompt },
    referenceOutputs: { answer_snippets: testCase.answer_snippets },
  }, async ({ inputs }) => {
    const result = await runner.run({
      query: inputs.query,
      initialFiles: testCase.files,
    });

    const finalText = getFinalText(result).toLowerCase().replace(/\s+/g, "");

    if (testCase.answer_snippets.length > 0) {
      // 归一化比较：去空格、去引号、小写
      const normalize = (s: string) =>
        s.toLowerCase().replace(/\s+/g, "").replace(/['"]/g, "");

      for (const snippet of testCase.answer_snippets) {
        expect(finalText).toContain(normalize(snippet));
      }
    } else {
      // 无期望片段时，只验证非空回答
      expect(finalText.length).toBeGreaterThan(0);
    }

    ls.logFeedback({ key: "benchmark", value: testCase.benchmark });
    ls.logFeedback({ key: "difficulty", value: testCase.difficulty ?? "unknown" });
    ls.logFeedback({ key: "agent_steps", score: result.steps.length });
  });
}
```

---

## 归一化比较

答案比较使用归一化处理，提高鲁棒性：

```typescript
const normalize = (s: string) =>
  s.toLowerCase()           // 忽略大小写
   .replace(/\s+/g, "")    // 忽略所有空白
   .replace(/['"]/g, "");  // 忽略引号

// "The answer is 42" → "theansweris42"
// "the answer is '42'" → "theansweris42"  ✓ 匹配
```

---

## 数据驱动设计的价值

这套测试展示了**数据驱动**评估的最佳实践：

1. **与代码分离**：测试用例在 JSON 文件中，增减用例无需改代码
2. **可追溯**：每个用例有 benchmark 来源和 ID，可追溯到原始基准
3. **可扩展**：未来添加新基准只需在 JSON 中增加用例
4. **标准化比较**：归一化函数处理常见格式差异

---

## 参考链接

- [oolong：另一个外部基准 →](./19-suite-oolong.md)
- [tau2-airline：策略基准 →](./18-suite-tau2-airline.md)
