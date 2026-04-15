# tool-usage-relational：关系型数据链

**目录**：`evals/tool-usage-relational/`

16 个测试用例，从单步查询到 5 步并行链，全面验证 Agent 在关系型数据模型上的多步工具使用能力。

---

## 数据模型

### 实体定义

**用户（6 条）**：
```typescript
interface User {
  id: string;
  name: string;
  email: string;
  location: string;      // 关联 Location.id
  favorite_color: string;
  favorite_foods: string[];  // 关联 Food.id[]
}
```

**地点（5 条）**：
```typescript
interface Location {
  id: string;
  city: string;
  current_time: string;
  current_weather: string;
}
```

**食物（7 条）**：
```typescript
interface Food {
  id: string;
  name: string;
  calories: number;
  allergic_ingredients: string[];
}
```

---

## 工具集（16 个）

```typescript
// 用户类
get_current_user_id()                   → string
list_user_ids()                         → string[]
find_users_by_name(name)                → User[]
get_user_name(user_id)                  → string
get_user_email(user_id)                 → string
get_user_location(user_id)              → string  // location_id
get_user_favorite_color(user_id)        → string
get_user_favorite_foods(user_id)        → string[]  // food_ids

// 地点类
find_locations_by_name(name)            → Location[]
get_city_for_location(location_id)      → string
get_current_time_for_location(location_id) → string
get_weather_at_location(location_id)    → string

// 食物类
find_foods_by_name(name)                → Food[]
get_food_name(food_id)                  → string
get_food_calories(food_id)              → number
get_food_allergic_ingredients(food_id)  → string[]
```

所有工具均通过 `vi.fn` 包装为 spy，返回预定义数据集中的内容。

---

## 测试用例（按复杂度分级）

### 单步（1工具）

```typescript
// single tool: list user IDs
runner.run({ query: "List all user IDs" });
expect(tools.listUserIds.spy).toHaveBeenCalled();
expect(result).toHaveFinalTextContaining("user-1");  // 预期 ID 之一
```

### 两步链

```typescript
// two tools: user name from current ID
// 步骤 1: get_current_user_id → "user-3"
// 步骤 2: get_user_name("user-3") → "Carol"
runner.run({ query: "What is the current user's name?" });
expect(result).toHaveFinalTextContaining("Carol");
```

### 三步链

```typescript
// three tools: current user location city
// get_current_user_id → get_user_location → get_city_for_location
runner.run({ query: "What city is the current user in?" });
```

### 四步链（含并行）

```typescript
// four tools: current user food names
// get_current_user_id → get_user_favorite_foods → [get_food_name] (并行)
runner.run({ query: "What are the current user's favorite foods?" });

// 期望：第 3 步并行调用多个 get_food_name
expect(result).toHaveToolCallInStep(3, { name: "get_food_name" });
// 并且第 3 步有多个工具调用（并行）
```

### 五步链

```typescript
// five tools: find user city weather time and food details
// list_user_ids → find_users_by_name → get_user_location + get_user_favorite_foods
//              → get_city_for_location + get_weather_at_location + [get_food_calories]
runner.run({
  query: "For user named Bob: what's the weather in his city and total calories of his favorite foods?"
});
```

---

## 测试设计模式

### 数据关系图

```
User ──location──→ Location
  └──favorite_foods──→ Food[]
```

测试用例利用这些外键关系，要求 Agent 理解"先查 ID，再用 ID 查详情"的关系型查询模式。

### 并行机会识别

当查询需要同一类型的多个数据项时（如多个食物名称），Agent 应识别为可并行操作：

```
× 串行（慢）:
  get_food_name("food-1") → 等结果 → get_food_name("food-2") → ...

✓ 并行（快）:
  同步发出: [get_food_name("food-1"), get_food_name("food-2"), ...]
```

### 效率断言

```typescript
// 验证在特定步骤确实发出了并行调用
const step3 = result.steps[2];  // 0-indexed
expect(step3.action.tool_calls.length).toBeGreaterThan(1);
```

---

## 测试用例完整列表

| 编号 | 名称 | 工具步数 | 并行 |
|------|------|---------|------|
| 1 | single tool: list user IDs | 1 | 否 |
| 2 | two tools: user name from current ID | 2 | 否 |
| 3 | two tools: user email from name | 2 | 否 |
| 4 | three tools: current user location city | 3 | 否 |
| 5 | three tools: user weather from name | 3 | 否 |
| 6 | three tools: user favorite color from name | 3 | 否 |
| 7 | four tools: current user food names | 4 | 是 |
| 8 | four tools: user food calorie total | 4 | 是 |
| 9 | four tools: find user city weather time | 4 | 是 |
| 10 | four steps: find user city weather time and food details | 4-5 | 是 |
| 11 | five tools: allergic ingredients for user | 5 | 是 |
| 12-16 | 更复杂的交叉关系查询 | 4-5 | 是 |

---

## 参考链接

- [tool-selection：工具路由 →](./11-suite-tool-selection.md)
- [subagents：子代理委派 →](./13-suite-subagents.md)
