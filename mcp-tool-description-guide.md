# MCP Tool Schema 描述规范

> 面向 aisecurity MCP Server 开发团队。
> 当前 8 个工具的 schema 由 Java 注解自动生成。本文说明问题所在、修复规则和具体示例。

---

## 核心原则：把 Agent 想象成一个什么都不了解我们系统的聪明人

**Agent 不是机器，它像一个刚入职、非常聪明、但对我们业务一无所知的新同事。**

它不知道：
- `busId` 和 `busCode` 和 `numberPlate` 有什么区别
- `isAir` 传 `"1"` 还是 `true` 还是 `"是"`
- `planPurchaseDate` 是什么格式
- 编辑车辆前要先查出 `busId`

**你写的描述，就是它唯一的说明书。** 写给这个聪明的新同事看，而不是写给已经知道所有细节的自己看。

---

## 整个系统里 Agent 能读到什么

在解释为什么 tool description 很重要之前，先说清楚 Agent 在我们系统里能读到哪些信息，以及你们的工具描述在哪个环节发挥作用。

### Agent 的信息来源（按读取顺序）

```
┌─────────────────────────────────────────────────────────┐
│ 1. SKILL.md（系统提示）                                   │
│    每轮对话开始时自动加载。包含角色定义、业务规则、          │
│    实体关系说明、操作流程。由我们团队维护。                  │
├─────────────────────────────────────────────────────────┤
│ 2. 对话历史                                               │
│    本轮会话的所有上下文。                                   │
├─────────────────────────────────────────────────────────┤
│ 3. tools/list 返回值  ← 你们负责的部分                    │
│    Agent 决定"要不要调用工具、调用哪个、传什么参数"时，      │
│    唯一依据就是这里的 name + description + inputSchema。   │
├─────────────────────────────────────────────────────────┤
│ 4. tools/call 返回值                                      │
│    工具执行结果。Agent 读完后决定下一步。                    │
├─────────────────────────────────────────────────────────┤
│ 5. RAG（知识库检索）                                       │
│    Agent 主动调用 query_kb 工具触发，返回相关文档片段。      │
│    适合大量业务文档的模糊查询，不适合字段格式这类精确信息。   │
└─────────────────────────────────────────────────────────┘
```

### 一个真实场景：用户问"帮我查一下粤B88888这辆车的电池容量"

**如果 tool description 写得差（现状）：**

Agent 看到：
```
tool: get_base_odsJituanBsBus_queryById
description: "集团车辆-通过id查询"
param id: "车辆ID"
```

Agent 的困境：用户给的是车牌号 `粤B88888`，工具要的是 `id`。
`id` 是车牌号吗？是 `busId`？是 `busCode`？
它猜不到，可能直接把 `粤B88888` 传进去，接口报错，任务失败。

---

**如果 tool description 写得好：**

Agent 看到：
```
tool: get_base_odsJituanBsBus_queryById
description: "按内部ID查询单辆车完整信息。
             id 是系统内部 busId，不是车牌号。
             如果只有车牌号，先用 get_list 过滤 numberPlate 字段拿到 busId。"
param id: "车辆内部ID，从列表接口的 busId 字段获取"
```

Agent 的行为：先调 `get_list` 用 `numberPlate="粤B88888"` 过滤，拿到 `busId`，再调 `queryById`。两步走，结果正确。

---

**RAG 能解决这个问题吗？**

RAG 适合"查业务规程、事故案例、操作手册"这类语义模糊的大型文档检索。
对于"这个参数传什么格式"这类**精确的技术约定**，RAG 不是正确的解法——
即使知识库里存了字段说明，Agent 也不一定会想到去查，查了也可能返回不相关的片段。

**字段格式、枚举值、必填声明，必须写在 tool description 里。这是唯一可靠的位置。**

---

## 问题背景

MCP 协议规定，Agent 通过 `tools/list` 获取工具列表，每个工具包含：

```json
{
  "name": "工具唯一标识",
  "description": "工具的自然语言说明",
  "inputSchema": {
    "type": "object",
    "properties": {
      "参数名": {
        "type": "string | integer | boolean | array | object",
        "description": "参数说明",
        "enum": ["可选值1", "可选值2"]
      }
    },
    "required": ["必填参数名"]
  }
}
```

Agent 完全依赖这个结构决定：是否调用、调用哪个、传什么参数。

当前描述存在三个系统性问题：

| 问题 | 示例 | 影响 |
|------|------|------|
| 描述重复工具名 | `"集团车辆-编辑 - 集团车辆-编辑"` | Agent 不知道何时用、有什么限制 |
| 参数描述只翻译字段名 | `"是否空调"` | Agent 不知道传 `"1"` 还是 `true` 还是 `"是"` |
| 写操作没有 `required` | `put_edit` 无必填声明 | Agent 不知道 `busId` 必填，可能漏传导致报错 |

---

## 修复规则

### 规则一：工具描述写 2 句话

**第一句**：这个工具做什么。
**第二句**：什么时候不用它 / 应该用哪个替代。

坏的写法：

```json
{
  "name": "put_base_odsJituanBsBus_edit",
  "description": "集团车辆-编辑 - 集团车辆-编辑"
}
```

好的写法：

```json
{
  "name": "put_base_odsJituanBsBus_edit",
  "description": "编辑集团车辆信息。busId 必填，其余字段按需传入，仅传需要修改的字段。查询车辆信息请用 get_base_odsJituanBsBus_queryById。"
}
```

---

### 规则二：有固定取值的参数必须加 `enum`，并在描述中说明含义

坏的写法：

```json
"isAir": {
  "type": "string",
  "description": "是否空调"
}
```

好的写法：

```json
"isAir": {
  "type": "string",
  "enum": ["0", "1"],
  "description": "是否配备空调：\"1\" 表示有空调，\"0\" 表示无空调"
}
```

---

### 规则三：格式敏感的参数必须写明格式和示例

坏的写法：

```json
"planPurchaseDate": {
  "type": "string",
  "description": "计划购买日期"
}
```

好的写法：

```json
"planPurchaseDate": {
  "type": "string",
  "description": "计划购买日期，格式 YYYY-MM-DD，例如 \"2024-03-15\""
}
```

---

### 规则四：写操作必须在 `inputSchema.required` 中声明必填字段

坏的写法（`required` 缺失，Agent 不知道 `busId` 是必填）：

```json
{
  "name": "put_base_odsJituanBsBus_edit",
  "inputSchema": {
    "type": "object",
    "properties": {
      "busId":   { "type": "string", "description": "车辆ID" },
      "busCode": { "type": "string", "description": "车辆编码" }
    }
  }
}
```

好的写法：

```json
{
  "name": "put_base_odsJituanBsBus_edit",
  "inputSchema": {
    "type": "object",
    "properties": {
      "busId":   { "type": "string", "description": "车辆内部ID，必填，从列表或 queryById 接口的 busId 字段获取" },
      "busCode": { "type": "string", "description": "车辆编码，仅在需要修改时传入" }
    },
    "required": ["busId"]
  }
}
```

---

### 规则五：列表接口的过滤参数要说明"不填时的行为"

坏的写法：

```json
{
  "pageNo":   { "type": "integer", "description": "页码" },
  "pageSize": { "type": "integer", "description": "每页数量" },
  "busType":  { "type": "string",  "description": "车辆类型" }
}
```

好的写法：

```json
{
  "pageNo":   { "type": "integer", "description": "页码，从 1 开始，默认 1" },
  "pageSize": { "type": "integer", "description": "每页条数，默认 10，建议不超过 100" },
  "busType":  { "type": "string",  "description": "按车辆型号过滤，例如 \"宇通ZK6125\"，不填则返回所有型号" }
}
```

## 各工具修复清单

| 工具名 | 需修复项 |
|--------|----------|
| `get_base_odsJituanBsBus_queryById` | 工具描述 |
| `get_base_odsJituanBsBus_list` | 工具描述、pageNo/pageSize 描述、过滤参数描述 |
| `get_base_odsJituanBsBus_busTypelist` | 同上 |
| `get_base_odsJituanBsBus_vehicleTypelist` | 同上 |
| `get_base_odsJituanBsBus_exportXls` | 工具描述（说明这是导出不是查询） |
| `post_base_odsJituanBsBus_add` | 工具描述、建议必填字段声明 |
| `put_base_odsJituanBsBus_edit` | 工具描述、`required: ["busId"]`、参数描述 |
| `post_base_odsJituanBsBus_importExcel` | 工具描述（说明是文件上传，不适合单条创建） |

---

> 以上修复请在服务端完成。如有问题请联系我们团队确认。
