# tau2-airline：策略合规

**目录**：`evals/tau2-airline/`

受 Tau2 Airline 基准启发的 14 个测试用例，验证 Agent 在航空客服场景中对业务策略的遵从性——正确的工具调用顺序、身份验证前置、操作后发送确认。

---

## 场景设定

模拟航空公司客服 Agent，有 7 个操作工具和一套严格的业务策略。

---

## 工具集（7 个，全部 spy 包装）

```typescript
const tools = {
  lookup_reservation: spiedTool(
    async ({ confirmation_code }) => ({
      code: confirmation_code,
      passenger: "John Doe",
      flight: "AA123",
      date: "2026-05-01",
      seat: "14A",
      status: "confirmed"
    }),
    { name: "lookup_reservation", schema: z.object({ confirmation_code: z.string() }) }
  ),

  verify_identity: spiedTool(
    async ({ last_name, email }) => ({ verified: true }),
    { name: "verify_identity", schema: z.object({ last_name: z.string(), email: z.string() }) }
  ),

  rebook_flight: spiedTool(...),
  cancel_flight: spiedTool(...),
  request_refund: spiedTool(...),
  seat_upgrade: spiedTool(...),
  send_confirmation: spiedTool(...),
};
```

---

## 业务策略（注入系统提示）

```typescript
const POLICY_PROMPT = `
You are an airline customer service agent. Follow these policies:

1. IDENTITY VERIFICATION: Always verify customer identity before making 
   any changes to reservations (rebook, cancel, refund, upgrade).
   
2. RESERVATION LOOKUP: Always look up the reservation before taking any action.

3. CONFIRMATION: Always send a confirmation after successfully completing 
   any operation.

4. ACCURACY: Be concise and accurate. Don't make assumptions.
`;
```

---

## 14 个测试任务

```typescript
interface AirlineTask {
  taskId: string;
  query: string;
  requiredTool: string;    // 核心必须调用的工具
  expectedText: string;   // 期望最终回复包含的文本
}

const TASKS: AirlineTask[] = [
  {
    taskId: "T01",
    query: "I need to cancel my flight. Confirmation: ABC123. Name: Doe, email: doe@example.com",
    requiredTool: "cancel_flight",
    expectedText: "cancelled",
  },
  {
    taskId: "T02",
    query: "Rebook me to a later flight. Confirmation: XYZ456. Name: Smith, email: smith@example.com",
    requiredTool: "rebook_flight",
    expectedText: "rebooked",
  },
  // ... 共 14 个
];
```

---

## 断言逻辑（每个任务 4 项断言）

```typescript
for (const task of TASKS) {
  ls.test(`task ${task.taskId}`, { inputs: { query: task.query } }, async ({ inputs }) => {
    const result = await runner
      .extend({
        tools: Object.values(tools).map(t => t.tool),
        systemPrompt: POLICY_PROMPT,
      })
      .run({ query: inputs.query });

    // 策略断言 1：必须查询预订
    expect(tools.lookup_reservation.spy).toHaveBeenCalled();

    // 策略断言 2：敏感操作前必须验证身份
    expect(tools.verify_identity.spy).toHaveBeenCalled();

    // 策略断言 3：必须调用了核心操作工具
    expect(tools[task.requiredTool].spy).toHaveBeenCalled();

    // 策略断言 4：操作后必须发送确认
    expect(tools.send_confirmation.spy).toHaveBeenCalled();

    // 内容断言：最终回复包含预期文本
    expect(result).toHaveFinalTextContaining(task.expectedText);

    // 重置 spy（为下一个测试准备）
    Object.values(tools).forEach(t => t.spy.mockClear());
  });
}
```

---

## 策略遵从的顺序验证

除了"是否被调用"，还验证**调用顺序**符合策略：

```typescript
// 验证 lookup_reservation 在 verify_identity 之前被调用
const lookupCallOrder = tools.lookup_reservation.spy.mock.invocationCallOrder[0];
const verifyCallOrder = tools.verify_identity.spy.mock.invocationCallOrder[0];
const requiredCallOrder = tools[task.requiredTool].spy.mock.invocationCallOrder[0];
const confirmCallOrder = tools.send_confirmation.spy.mock.invocationCallOrder[0];

expect(lookupCallOrder).toBeLessThan(verifyCallOrder);
expect(verifyCallOrder).toBeLessThan(requiredCallOrder);
expect(requiredCallOrder).toBeLessThan(confirmCallOrder);
```

---

## 设计价值

这套测试展示了如何评估 **"策略合规性"**，这与一般的功能测试不同：

| 维度 | 普通功能测试 | 策略合规测试 |
|------|------------|------------|
| 关注点 | 操作是否完成 | 操作的执行方式是否合规 |
| 验证内容 | 工具是否被调用 | 工具调用的顺序和前置条件 |
| 业务价值 | 功能可用性 | 合规性与风险控制 |

---

## 参考链接

- [tool-selection：工具路由 →](./11-suite-tool-selection.md)
- [external-benchmarks：外部基准 →](./17-suite-external-benchmarks.md)
