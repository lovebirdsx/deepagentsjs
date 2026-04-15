# all：聚合套件

**目录**：`evals/all/`

批量执行所有评估套件的聚合入口。使用同一个 Runner 在单次 Vitest 运行中跑完所有套件，适合模型全量基线对比。

---

## 文件结构

```
evals/all/
├── package.json
├── vitest.config.ts
└── eval.test.ts       ← 唯一的测试文件
```

没有 `index.ts`，不导出套件函数。

---

## eval.test.ts 结构

```typescript
import { getDefaultRunner } from "@deepagents/evals";

// 导入所有套件函数
import { basicSuite } from "@deepagents/evals-basic";
import { filesSuite } from "@deepagents/evals-files";
import { todosSuite } from "@deepagents/evals-todos";
import { memorySuite } from "@deepagents/evals-memory";
import { skillsSuite } from "@deepagents/evals-skills";
import { subagentsSuite } from "@deepagents/evals-subagents";
import { summarizationSuite } from "@deepagents/evals-summarization";
import { hitlSuite } from "@deepagents/evals-hitl";
import { toolSelectionSuite } from "@deepagents/evals-tool-selection";
import { toolUsageRelationalSuite } from "@deepagents/evals-tool-usage-relational";
import { followupQualitySuite } from "@deepagents/evals-followup-quality";
import { externalBenchmarksSuite } from "@deepagents/evals-external-benchmarks";
import { tau2AirlineSuite } from "@deepagents/evals-tau2-airline";
import { memoryAgentBenchSuite } from "@deepagents/evals-memory-agent-bench";
import { memoryMultiturnSuite } from "@deepagents/evals-memory-multiturn";

// oolong 因 token 消耗大，默认注释
// import { oolongSuite } from "@deepagents/evals-oolong";

const runner = getDefaultRunner();

// 依次注册所有套件
basicSuite(runner);
filesSuite(runner);
todosSuite(runner);
memorySuite(runner);
skillsSuite(runner);
subagentsSuite(runner);
summarizationSuite(runner);
hitlSuite(runner);
toolSelectionSuite(runner);
toolUsageRelationalSuite(runner);
followupQualitySuite(runner);
externalBenchmarksSuite(runner);
tau2AirlineSuite(runner);
memoryAgentBenchSuite(runner);
memoryMultiturnSuite(runner);
// oolongSuite(runner);
```

---

## 设计意图

### 为什么要有 all 套件？

1. **统一对比**：所有套件在同一次运行中使用同一个模型，结果在 LangSmith 中归属同一个实验
2. **回归检测**：每次模型版本更新时，运行 all 套件可以快速发现能力退化
3. **基线建立**：为新模型建立完整的能力基线

### 为什么 oolong 被注释？

oolong 测试用例数量多（默认每数据集 10 条 × 10 数据集 = 100 条），每条包含数千 token，运行成本极高。通常只在专项评估时运行。

---

## 运行命令

```bash
cd evals/all
EVAL_RUNNER=sonnet-4-6 \
LANGSMITH_API_KEY=... \
ANTHROPIC_API_KEY=... \
pnpm test:eval
```

---

## 参考链接

- [架构总览 →](./01-架构总览.md)
- [运行与配置 →](./03-运行与配置.md)
