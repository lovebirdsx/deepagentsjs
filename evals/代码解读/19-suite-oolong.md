# oolong：长上下文推理

**目录**：`evals/oolong/`

对接 Oolong 学术基准的评估套件，测试 Agent 在超长上下文（数千 token）中的信息聚合和推理能力。这是结构最复杂的套件，包含数据加载、评分系统和多数据集管理。

---

## 背景

**Oolong Benchmark**（"Evaluating Long Context Reasoning and Aggregation Capabilities"，2025）：
- 测试模型在长上下文中进行**聚合推理**的能力（不仅是检索）
- 数据来源：HuggingFace `oolongbench/oolong-synth` 验证集
- 10 个源数据集，各类文本分类和推理任务

---

## 目录结构

```
evals/oolong/
├── load-oolong.ts     ← 数据加载与缓存
├── scoring.ts         ← 官方评分逻辑
├── make-tests.ts      ← 测试生成（主入口）
├── eval.test.ts       ← 测试注册
└── datasets/          ← 各数据集的答案类型定义
    ├── agnews.ts
    ├── spam.ts
    ├── imdb.ts
    └── ...（10 个）
```

---

## 核心类型

### OolongTask

```typescript
interface OolongTask {
  id: string;
  dataset: string;               // 来源数据集（如 "spam"、"ag_news"）
  contextLen: number;            // 上下文长度（token 数）
  contextWindowText: string;     // 实际的长上下文文本
  question: string;              // 问题
  taskGroup: string;             // 任务组（如 "classification"）
  task: string;                  // 具体任务（如 "count"、"majority"）
  answer: string;                // 标准答案
  answerType: "exact" | "numeric" | "comparison" | "date";
  inputSubset: string;           // 输入子集标识
  numLabels: number;             // 类别数量
  contextWindowId: string;       // 上下文窗口 ID
}
```

---

## load-oolong.ts — 数据加载

```typescript
// 从 HuggingFace 下载数据集（带本地缓存）
async function loadOolongTasks(options?: {
  maxPerDataset?: number;   // 默认 10（由 OOLONG_MAX_PER_DATASET 控制）
  contextLen?: number;      // 过滤特定长度（由 OOLONG_CONTEXT_LEN 控制）
}): Promise<OolongTask[]>
```

**缓存机制**：
1. 首次下载后保存到 `.oolong-cache/` 目录
2. 后续运行直接从缓存读取
3. 避免重复下载（数据集较大）

**环境变量控制**：
```bash
OOLONG_MAX_PER_DATASET=5    # 每个数据集最多 5 条（加速测试）
OOLONG_CONTEXT_LEN=2000     # 只测试 ~2000 token 的上下文
```

---

## scoring.ts — 官方评分逻辑

Oolong 基准使用官方评分函数，而非简单的字符串包含：

```typescript
function scoreOolongAnswer(
  prediction: string,
  gold: string,
  answerType: OolongTask["answerType"]
): number  // 返回 0~1 的分数
```

### 各 answerType 评分方式

```typescript
switch (answerType) {
  case "exact":
    // 解析答案（取 ":" 之后的部分，去除 Markdown 格式）
    // 精确匹配
    return normalize(prediction) === normalize(gold) ? 1 : 0;

  case "comparison":
    // 解析比较类答案
    // "more than", "less than", "same" 的子字符串匹配
    return prediction.toLowerCase().includes(gold.toLowerCase()) ? 1 : 0;

  case "numeric":
    // 数值类答案使用软评分：0.75 ^ |gold - pred|
    // 差 0 → 分数 1.0；差 1 → 0.75；差 2 → 0.5625；...
    const diff = Math.abs(parseFloat(gold) - parseFloat(prediction));
    return Math.pow(0.75, diff);

  case "date":
    // 灵活的日期解析和比较
    return parsedDateMatch(prediction, gold) ? 1 : 0;
}
```

**答案解析**（所有类型共用）：
```typescript
function parseAnswer(raw: string): string {
  // 提取 "Answer: X" 或 "答案：X" 格式
  const afterColon = raw.split(":").at(-1) ?? raw;
  // 去除 Markdown 格式（**bold**, `code`, etc.）
  return afterColon.replace(/[*`_]/g, "").trim();
}
```

---

## make-tests.ts — 测试生成

```typescript
export function oolongSuite(runner: EvalRunner): void {
  const tasks = await loadOolongTasks({
    maxPerDataset: parseInt(process.env.OOLONG_MAX_PER_DATASET ?? "10"),
    contextLen: process.env.OOLONG_CONTEXT_LEN
      ? parseInt(process.env.OOLONG_CONTEXT_LEN)
      : undefined,
  });

  ls.describe("oolong", () => {
    for (const task of tasks) {
      ls.test(`${task.dataset}/${task.id}`, {
        inputs: { query: task.contextWindowText + "\n\n" + task.question },
        referenceOutputs: { answer: task.answer },
      }, async ({ inputs }) => {
        const result = await runner.run({ query: inputs.query });
        const finalText = getFinalText(result);
        
        const score = scoreOolongAnswer(finalText, task.answer, task.answerType);
        
        // 使用 Oolong 官方评分（部分正确也有分）
        ls.logFeedback({ key: "oolong_score", score });
        ls.logFeedback({ key: "dataset", value: task.dataset });
        ls.logFeedback({ key: "context_len", score: task.contextLen });
        ls.logFeedback({ key: "task_type", value: task.task });
        
        // 通过阈值（≥0.75 视为正确）
        expect(score).toBeGreaterThanOrEqual(0.75);
      });
    }
  });
}
```

---

## 10 个源数据集

| 数据集 | 任务类型 | 典型问题 |
|--------|---------|---------|
| spam | 分类聚合 | "多少封是垃圾邮件？" |
| ag_news | 新闻分类 | "哪个类别最多？" |
| imdb | 情感分析 | "正面评价占多数还是负面？" |
| trec_coarse | 问题分类 | "最常见的问题类型是？" |
| negation | 逻辑推理 | "有多少包含否定？" |
| yahoo | 话题分类 | "体育类问题有几个？" |
| formality | 风格分析 | "正式文本的比例是？" |
| multinli | 自然语言推理 | "蕴含关系最多？" |
| metaphors | 修辞分析 | "使用了多少个比喻？" |
| app_reviews | 评分聚合 | "平均分是多少？" |

---

## 与其他套件的区别

| 维度 | oolong | memory-agent-bench |
|------|--------|-------------------|
| 上下文长度 | 数千 token | 数百 token |
| 任务类型 | 聚合推理 | 事实检索 |
| 评分方式 | 软评分（partial credit） | 精确匹配 |
| 数据来源 | 学术基准（HF） | 手工设计 |
| 成本 | 高（大量 token） | 低 |

---

## 注意事项

- oolong 套件因 token 消耗大，在 `all/eval.test.ts` 中被**注释掉**
- 建议单独运行：`cd evals/oolong && EVAL_RUNNER=sonnet-4-6 pnpm test:eval`
- `OOLONG_MAX_PER_DATASET=3` 可快速验证流程

---

## 参考链接

- [external-benchmarks →](./17-suite-external-benchmarks.md)
- [summarization：长上下文处理 →](./14-suite-summarization.md)
