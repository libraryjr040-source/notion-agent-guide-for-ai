# Ch4 Agent 工程

**覆盖域**：Custom Agent 创建 · 权限 · MCP 集成 · Trigger · 通信 · Sub-thread · Skill · 更新 · Autofill · 多 Agent 协作 · 调试 · 通知
**涉及 modules**：`notion/agents`、`mcpServer`、`notion/threads`、`notion/triggers.ts`、`notion/integration.ts`、`notion/notifications.ts`、`notion/users`、`system`

---

## 薄概念 DRY 层

本节只建关系图谱，不讲解——讲解是 module files 的事。

### 核心实体

| 概念 | 一句话 | module files 指针 |
|------|--------|-------------------|
| Agent | workspace 内的 AI 行动者，拥有 name / description / icon / instructions / integrations / triggers | `modules/notion/agents/index.ts` |
| Instructions Page | Agent 的指令页面，本质是一个普通 Notion Page，通过 `instructionsPageUrl` 关联 | `modules/notion/agents/index.ts` → `Agent.instructionsPageUrl` |
| Integration | Agent 与一个 module 的连接（Notion、Slack、MCP Server 等），通过 `integrations` 记录管理 | `modules/notion/agents/index.ts` → `AgentIntegration` |
| Permission | 挂在 Notion integration 下，控制 Agent 对 workspace 内容的访问粒度 | `modules/notion/integration.ts` |
| Trigger | Agent 的自动触发条件，挂在 `triggers` 记录下 | `modules/notion/triggers.ts` |
| Thread | Agent 与用户（或其他 Agent）的对话线程 | `modules/notion/threads/index.ts` |
| Sub-thread | 通过 `createAndRunThread` 开的子线程，同步运行并返回最终文本 | `modules/notion/threads/index.ts` → `CreateAndRunThread` |
| MCP Server | 外部工具服务，Agent 通过 integration 连接后可调用其 tools | `modules/mcpServer/index.ts` |

### 关键关系

- 一个 Agent 有且仅有一个 Instructions Page
- 一个 Agent 有零到多个 Integrations，每个 Integration 对应一个 module type
- Notion Integration 下挂零到多个 Permissions，控制内容访问和搜索能力
- 一个 Agent 有零到多个 Triggers，每个 Trigger 可关联一个 Integration
- Agent 间通信需要双向 interact 权限：调用方的 Notion integration 需要对目标 Agent 设置 `{ identifier: { type: "agent", url: "dataSourceUrl" }, actions: ["interact"] }`
- Sub-thread 的 response 只包含最终文本，不包含中间工具调用的结果

### 标识符系统

- 已有实体使用 compressed URL（如 `"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`、`"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`、`"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`）
- 新建实体使用 `CREATE-*` 占位符（如 `CREATE-notion-1`、`CREATE-trigger-1`）
- 同一次 create/update 调用中，`CREATE-*` 可互相引用（如 trigger 的 `integrationUrl` 引用同批新建的 integration）
- 调用返回后，系统分配真实 URL，后续操作必须使用真实 URL

---

## S4.1 创建 Custom Agent

### 场景入口

从零创建一个 Custom Agent，包含基本信息和初始 integrations / triggers 配置。

### 适用条件

- 需要一个新的 AI 行动者来执行特定任务
- 已知 Agent 的职责、需要的 module 连接和触发方式

### 操作步骤

1. 确定 Agent 的 name、description、icon
2. 规划 integrations：至少需要一个 Notion integration 来访问 workspace 内容
3. 规划 triggers：确定 Agent 的激活方式
4. 调用 `connections.notion.createAgent`：

```typescript
const agent = await connections.notion.createAgent({
  name: "审阅助手",
  description: "自动审阅数据库中新增的文档",
  icon: { type: "agent_icon", shape: "check", color: "green" },
  integrations: {
    "CREATE-notion-1": {
      type: "notion",
      name: "Notion",
      permissions: [
        { identifier: "*", actions: ["edit"] },
        { identifier: "webSearch", actions: ["allow"] },
        { identifier: "helpdocsSearch", actions: ["allow"] },
      ],
    },
  },
  triggers: {
    "CREATE-trigger-1": {
      enabled: true,
      state: { type: "notion.agent.mentioned" },
      integrationUrl: "CREATE-notion-1",
    },
  },
})
// agent.agentUrl → 后续操作使用此 URL
// agent.instructionsPageUrl → 用 updatePage 写入指令内容
```

5. 创建完成后，通过 `connections.notion.updatePage` 向 `agent.instructionsPageUrl` 写入指令内容

### 关键参数

**Icon 类型**（三选一）：

| 类型 | 格式 | 示例 |
|------|------|------|
| Agent Icon | `{ type: "agent_icon", shape, color }` | `{ type: "agent_icon", shape: "book", color: "blue" }` |
| Emoji | `{ type: "emoji", emoji }` | `{ type: "emoji", emoji: "🤖" }` |
| URL | `{ type: "url", url }` | `{ type: "url", url: "https://..." }` |

**Agent Icon Shapes / Colors**：完整枚举见 `modules/notion/agents/index.ts` → `agentIconShapes`、`agentIconColors`。

### 常见陷阱

- ⚠️ **创建后 instructions 页面为空**：`createAgent` 不接受 instructions 内容参数。必须在创建后通过 `updatePage({ url: agent.instructionsPageUrl, contentUpdates: [...] })` 单独写入。
- ⚠️ **Integration key 与 trigger 的 integrationUrl 必须对应**：trigger 需要关联来自外部 app 的 integration 时，`integrationUrl` 必须指向同批创建的 integration key 或已有 integration URL。对于 Notion 原生 trigger 类型（如 `notion.agent.mentioned`、`notion.button.pressed`），`integrationUrl` 应指向 Notion integration。
- ⚠️ **CREATE-\* 只在同一次调用内有效**：跨调用引用必须使用返回的真实 compressed URL。

### 深钻指针

- Agent 类型定义与 createAgent 签名：`modules/notion/agents/index.ts`
- 页面内容编辑：`modules/notion/pages/AGENTS.md`

---

## S4.2 权限配置

### 场景入口

配置 Agent 的 Notion integration 权限——控制它能访问什么内容、能否搜索 web / helpdocs、能否与其他 Agent 通信。

### 适用条件

- 创建 Agent 时设定初始权限
- 更新已有 Agent 的权限范围
- 需要授权 Agent 间通信

### 推荐路径

权限挂在 Notion integration 的 `permissions` 数组下。每个 permission 对象的 `identifier` 决定权限类型，`actions` 决定访问级别。

#### 内容访问权限

| identifier | actions | 含义 |
|-----------|---------|------|
| `"*"` | `["edit"]` | workspace 全局编辑 |
| `"*"` | `["view"]` | workspace 全局只读 |
| `"*"` | `["comment"]` | workspace 全局评论 |
| `"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"` | `["edit"]` | 特定页面编辑 |
| `"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"` | `["view"]` | 特定数据库只读 |
| `["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "dataSourceUrl"]` | `["edit"]` | 多个页面编辑 |

注意：`"*"` 必须独立使用，不能与其他 identifier 组合在同一个 permission 对象中。

#### 搜索权限

| identifier | actions | 含义 |
|-----------|---------|------|
| `"webSearch"` | `["allow"]` | 允许 web 搜索 |
| `"webSearch"` | `["disallow"]` | 禁止 web 搜索 |
| `"helpdocsSearch"` | `["allow"]` | 允许搜索 Notion 帮助文档 |

webSearch 还支持可选的 `allowedDomains` 字段，限制搜索结果来源：

```typescript
{ identifier: "webSearch", actions: ["allow"], allowedDomains: ["docs.example.com"] }
```

#### Agent 间通信权限

```typescript
{ identifier: { type: "agent", url: "dataSourceUrl" }, actions: ["interact"] }
```

要禁止已有的通信权限：

```typescript
{ identifier: { type: "agent", url: "dataSourceUrl" }, actions: ["disallow"] }
```

### 实战案例

创建一个只读 Agent，仅能访问一个数据库，可搜索 web：

```typescript
const agent = await connections.notion.createAgent({
  name: "查询助手",
  icon: { type: "agent_icon", shape: "globe", color: "blue" },
  integrations: {
    "CREATE-notion": {
      type: "notion",
      name: "Notion",
      permissions: [
        { identifier: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", actions: ["view"] },
        { identifier: "webSearch", actions: ["allow"] },
      ],
    },
  },
})
```

### 常见陷阱

- ⚠️ **数据库级权限会继承到内部页面**：给 Agent 某个数据库的 edit 权限，数据库内的页面会继承该权限，无需对每个页面单独配权。
- ⚠️ **interact 权限是单向的**：Agent A 要调用 Agent B 的 `createAndRunThread`，需要 Agent A 的 Notion integration 中配置 `{ identifier: { type: "agent", url: "dataSourceUrl" }, actions: ["interact"] }`。如果 B 也需要回调 A，B 的配置中也需要对 A 的 interact 权限。
- ⚠️ **`sharing.ts` 的权限与 Agent 权限独立**：`loadPermissions` / `updatePermission` 管理的是页面/数据库的分享权限（用户级别），不影响 Custom Agent 的访问。Agent 权限通过 `loadAgent` / `updateAgent` 管理。
- ⚠️ **actions 是数组**：即使只有一个 action 也必须用数组形式，如 `["edit"]`，不是 `"edit"`。

### 深钻指针

- 权限类型定义：`modules/notion/integration.ts` → `NotionModulePermissionAiConfigurable`
- Agent 权限管理说明：`modules/notion/AGENTS.md` → sharing.ts 段落

---

## S4.3 MCP 集成

### 场景入口

为 Agent 添加 MCP Server 连接，使其能调用外部工具（如 GitHub、Figma 等）。

### 适用条件

- Agent 需要访问外部服务的 API
- 外部服务已提供 MCP Server

### 操作步骤

1. **发现可用 MCP Server**：通过 `connections.notion.listUserConnections()` 查看 `available` 中 `moduleType: "mcpServer"` 的条目，获取 name 和 url
2. **在 Agent 上添加 MCP integration**：

```typescript
// 创建时直接添加
const agent = await connections.notion.createAgent({
  name: "开发助手",
  integrations: {
    "CREATE-notion": {
      type: "notion",
      name: "Notion",
      permissions: [{ identifier: "*", actions: ["edit"] }],
    },
    "CREATE-github": {
      type: "mcpServer",
      name: "GitHub",  // 使用 listUserConnections 返回的 name
    },
  },
})
```

3. **调用 MCP 工具**：

```typescript
// 列出可用工具
const { tools } = await connections.mcpServer_github.listTools()

// 调用特定工具
const result = await connections.mcpServer_github.runTool({
  toolName: "get_file_contents",
  toolArguments: { owner: "org", repo: "repo", path: "README.md" },
})
// result.content → 工具返回内容数组
```

### 关键参数

- `listTools` 无参数，返回 `{ tools: McpServerListTool[] }`，每个 tool 含 `name`、`description`、`inputSchema`
- `runTool` 参数：`{ toolName: string, toolArguments: Record<string, unknown> }`
- 返回 `McpServerToolCallResult`：`{ content: McpToolResultContentItem[], name, statusCode? }`
- content 项的类型定义见 `modules/mcpServer/index.ts` → `McpToolResultContentItem`

### 常见陷阱

- ⚠️ **MCP 没有 AI 可配置权限**：与 Notion integration 不同，MCP Server integration 不接受 `permissions` 或 `state`。认证完全由用户在 UI 中完成。`modules/mcpServer/integration.ts` 明确定义 `McpServerModulePermissionAiConfigurable = never`。
- ⚠️ **连接名不要猜测**：MCP Server 的 `name` 必须来自 `listUserConnections()` 返回的 `available` 或 `connected` 列表，不可自行编造。
- ⚠️ **调用路径 vs 文件路径**：module files 路径统一为 `modules/mcpServer/`，但运行时调用使用 connection key（如 `connections.mcpServer_github`），不是文件路径。
- ⚠️ **MCP integration 每个独立**：一个 Agent 可以有多个 MCP integration（如 GitHub + Figma），每个都是独立的 connection key。

### 深钻指针

- MCP 工具类型与调用签名：`modules/mcpServer/index.ts`
- MCP integration 类型：`modules/mcpServer/integration.ts`
- 用户连接管理：`modules/notion/users/AGENTS.md`

---

## S4.4 Trigger 设置

### 场景入口

配置 Agent 的自动触发条件——定时、页面事件、按钮点击或被 @mention。

### 适用条件

- Agent 需要在特定事件发生时自动执行
- 需要定时轮询或监控数据库变更

### Trigger 类型速查

| type | 用途 | 需要 url/配置 |
|------|------|---------------|
| `recurrence` | 定时触发 | frequency + interval + timezone |
| `notion.page.created` | 数据库新建页面 | 数据库 URL |
| `notion.page.updated` | 数据库页面更新 | 数据库 URL + 可选 properties 过滤 |
| `notion.page.deleted` | 数据库页面删除 | 数据库 URL |
| `notion.page.discussion.comment.added` | 页面新增评论 | 数据库 URL |
| `notion.database.agent.updated` | 数据库 Agent 配置更新 | 可选 collectionId |
| `notion.button.pressed` | Notion 按钮点击 | 无额外配置 |
| `notion.agent.mentioned` | 被 @mention | 无额外配置 |

### 操作步骤

#### 定时触发（Recurrence）

```typescript
// 每天上午 9 点触发
{
  "CREATE-trigger-daily": {
    enabled: true,
    state: {
      type: "recurrence",
      frequency: "day",
      interval: 1,
      timezone: "America/Chicago",
      hour: 9,
      minute: 0,
    },
    integrationUrl: "CREATE-notion-1",  // 关联 Notion integration
  },
}
```

```typescript
// 每周一三五触发
{
  "CREATE-trigger-weekday": {
    enabled: true,
    state: {
      type: "recurrence",
      frequency: "week",
      interval: 1,
      timezone: "America/Chicago",
      hour: 8,
      minute: 30,
      weekdays: ["MO", "WE", "FR"],
    },
    integrationUrl: "CREATE-notion-1",
  },
}
```

```typescript
// 每月最后一个周五
{
  "CREATE-trigger-monthly": {
    enabled: true,
    state: {
      type: "recurrence",
      frequency: "month",
      interval: 1,
      timezone: "America/Chicago",
      hour: 17,
      minute: 0,
      monthly_restriction: {
        type: "weekdays_in_month",
        weekdays: ["FR"],
        week_numbers: [-1],  // -1 表示当月最后一周
      },
    },
    integrationUrl: "CREATE-notion-1",
  },
}
```

#### 数据库页面事件

```typescript
// 监听数据库新页面
{
  "CREATE-trigger-page-created": {
    enabled: true,
    state: {
      type: "notion.page.created",
      url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",  // 目标数据库 URL
    },
    integrationUrl: "CREATE-notion-1",
  },
}

// 监听特定属性的变更
{
  "CREATE-trigger-page-updated": {
    enabled: true,
    state: {
      type: "notion.page.updated",
      url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
      properties: ["Status", "Priority"],  // 只在这些属性变更时触发
    },
    integrationUrl: "CREATE-notion-1",
  },
}
```

#### 数据库 Agent 更新

```typescript
// 监听数据库上 Agent 配置的变更（用于 autofill agent 重跑等场景）
{
  "CREATE-trigger-agent-updated": {
    enabled: true,
    state: {
      type: "notion.database.agent.updated",
    },
    integrationUrl: "CREATE-notion-1",
  },
}
```

#### 被 @mention 触发

```typescript
{
  "CREATE-trigger-mention": {
    enabled: true,
    state: { type: "notion.agent.mentioned" },
    integrationUrl: "CREATE-notion-1",
  },
}
```

#### 按钮点击触发

```typescript
{
  "CREATE-trigger-button": {
    enabled: true,
    state: { type: "notion.button.pressed" },
    integrationUrl: "CREATE-notion-1",
  },
}
```

### 关键参数

**Recurrence 频率选项**：
- `hour`：每 N 小时
- `day`：每 N 天（可指定 hour/minute）
- `week`：每 N 周（可指定 weekdays）
- `month`：每 N 月（可指定 monthly_restriction）
- `year`：每 N 年

**跨 trigger 类型的共享参数**：

| 参数 | 适用 trigger 类型 | 说明 |
|------|-------------------|------|
| `debounceTimeoutSeconds` | page.created, page.updated | 防抖秒数，合并短时间内的多次事件 |
| `propertyFilters` | page.created, page.updated, page.deleted | 更细粒度的属性值过滤（类型为 `AutomationEventConfiguration["pagePropertiesEdited"]`） |
| `viewId` | page.created, page.deleted | 可选，限定为某个 view 范围内的页面事件 |

**page.updated 特有参数**：
- `properties`：属性名或属性 URL 数组，仅监听这些属性的变更
- `shouldIgnorePageContentUpdates`：设为 true 则忽略内容变更，只监听属性变更

### 常见陷阱

- ⚠️ **timezone 必须提供**：recurrence trigger 必须指定 `timezone`（IANA 格式如 `"America/New_York"`），默认使用用户当前时区。
- ⚠️ **database URL vs page URL**：数据库事件 trigger 的 `url` 必须是数据库 URL，不是数据库内页面的 URL。
- ⚠️ **integrationUrl 必须关联**：trigger 需要通过 `integrationUrl` 关联到对应的 integration。对于 Notion 原生事件（如 page.created），关联 Notion integration；对于外部 app 事件，关联对应的 app integration。
- ⚠️ **week_numbers 中 -1 表示最后一周**：不是倒数第一个 weekday，而是当月最后一周中符合条件的 weekday。

### 深钻指针

- Trigger 配置类型定义：`modules/notion/triggers.ts` → `RecurrenceTriggerConfig`、`DatabasePageCreatedTriggerConfig` 等
- Trigger 运行时变量：`modules/notion/triggers.ts` → `*TriggerVariables` 类型
- Trigger 在 Agent 上的结构：`modules/notion/agents/index.ts` → `AgentTrigger`

---

## S4.5 Agent 间通信

### 场景入口

让一个 Custom Agent 调用另一个 Custom Agent 执行任务并获取结果。

### 适用条件

- 任务需要分工：不同 Agent 负责不同职责（如「总编」调度「著作郎」写作）
- 需要利用另一个 Agent 的特殊 integrations 或 instructions

### 操作步骤

1. **确保双方权限配置正确**：
   - 调用方 Agent A 的 Notion integration 中需要：
     ```typescript
     { identifier: { type: "agent", url: "dataSourceUrl" }, actions: ["interact"] }
     ```
   - 如果 B 需要回调 A，B 也需要对 A 的 interact 权限

2. **发起调用**：

```typescript
const { threadUrl, response } = await connections.notion.createAndRunThread({
  agentUrl: "dataSourceUrl",
  instructions: "请执行以下任务：...",
})
// response → B 的最终文本回复
// threadUrl → 可用于后续 continue 对话
```

3. **继续已有对话**（可选）：

```typescript
const { response: followUp } = await connections.notion.createAndRunThread({
  agentUrl: "dataSourceUrl",
  threadUrl: threadUrl,  // 继续之前的对话
  instructions: "请修订第三段",
})
```

### 关键参数

- `agentUrl`：目标 Agent 的 compressed URL。省略则调用自身。`"personal-agent"` 是特殊常量，仅 Personal Agent 调用自身时使用。Custom Agent 不可用此值。
- `threadUrl`：可选，传入已有 thread URL 以续写对话
- `instructions`：发送给目标 Agent 的消息/指令
- 返回值：`{ threadUrl: string, response: string }`

### 常见陷阱

- ⚠️ **response 只是最终文本**：Sub-agent 运行期间可能创建页面、写入数据库、调用 MCP 工具，但这些**副作用不会出现在 response 中**。response 仅包含 Agent 最终回复的文本。如果需要知道 sub-agent 做了什么，让它在回复中报告（如 commit hash、页面 URL 等）。
- ⚠️ **interact 权限被拒会直接报错**：没有 interact 权限时，`createAndRunThread` 会抛出 "request denied" 错误，不会静默失败。
- ⚠️ **Custom Agent 只能调用自身或已授权的其他 Agent**：不能调用任意 Agent。Personal Agent 可以调用 Custom Agent（通过 URL），但 Custom Agent 不能调用 Personal Agent。

### 深钻指针

- createAndRunThread 签名：`modules/notion/threads/index.ts` → `CreateAndRunThread`
- interact 权限类型：`modules/notion/integration.ts` → `NotionModulePermissionAiConfigurable`

---

## S4.6 Sub-thread 模式

### 场景入口

开子线程执行验证、委派、并行写作等任务。

### 适用条件

- 需要隔离一段工作（如技术验证）不污染主对话
- 需要并行分配多个子任务
- 需要调用自身的另一个实例处理部分工作

### 推荐路径

#### 自调自模式

省略 `agentUrl` 或传入自身 URL，会启动自身的新实例：

```typescript
// 开子 thread 验证一个技术问题
const { response } = await connections.notion.createAndRunThread({
  instructions: "验证 querySql 是否支持 LIKE 操作符，写一个测试查询并报告结果。",
})
// response 包含子线程的验证结果
```

#### 并行分发模式

```typescript
// 并行让多个分身处理不同场景
const scene1 = connections.notion.createAndRunThread({
  instructions: "写 S4.1 创建 Custom Agent 的章节内容",
})
const scene2 = connections.notion.createAndRunThread({
  instructions: "写 S4.2 权限配置的章节内容",
})

const [result1, result2] = await Promise.all([scene1, scene2])
// 汇总两个分身的输出
```

#### 委派给其他 Agent

```typescript
const { response } = await connections.notion.createAndRunThread({
  agentUrl: "dataSourceUrl",
  instructions: "审阅以下内容并返回修改意见：...",
})
```

### 关键参数

- **50 次调用上限**：一个 parent thread 最多调用 `createAndRunThread` 50 次。超出后调用会失败并返回 `max sub agents exceeded`。
- **同步执行**：`createAndRunThread` 是同步的（await 会阻塞直到 sub-agent 运行完毕并返回）。
- **子线程独立**：子线程有自己的工具调用和 context，不共享父线程的对话历史。
- **续写支持**：传入 `threadUrl` 可续写已有子线程，子线程会保留之前的对话上下文。

### 常见陷阱

- ⚠️ **50 次上限是硬限制**：无法绕过。在需要大量子任务的场景下，需要合理规划调用预算，将多个小任务合并为少量大任务。
- ⚠️ **子线程的副作用是真实的**：子线程中的 createPage、updatePage、MCP 调用等操作会实际执行。子线程不是沙箱——它是一个完整运行的 Agent 实例。
- ⚠️ **response 可能很长也可能很短**：response 取决于 sub-agent 的最终文本回复长度。如果需要结构化数据，在 instructions 中明确要求返回格式。
- ⚠️ **并行调用会同时消耗子线程配额**：5 个并行的 createAndRunThread 会消耗 5 次配额。

### 深钻指针

- createAndRunThread 完整类型：`modules/notion/threads/index.ts` → `CreateAndRunThread`
- system.wait 用于长时间任务间歇：`modules/system/index.ts` → `Wait`
- system.updateTodos 用于跟踪进度：`modules/system/index.ts` → `UpdateTodos`

---

## S4.7 Skill 编写

### 场景入口

编写 Agent 的 Instructions 页面——定义 Agent 的行为模式、工作流程和约束规则。

### 适用条件

- 新建的 Agent 需要写入指令
- 需要调整已有 Agent 的行为

### 推荐路径

1. **获取 instructions 页面 URL**：
   ```typescript
   const agent = await connections.notion.loadAgent({ url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" })
   const instructionsUrl = agent.instructionsPageUrl
   ```

2. **写入指令内容**：
   ```typescript
   await connections.notion.updatePage({
     url: instructionsUrl,
     contentUpdates: [{
       newStr: "## 你的职责\n\n你是一个代码审阅助手...\n\n## 工作流程\n\n1. 收到审阅请求后...\n\n## 铁律\n\n- 不修改代码，只提意见",
     }],
   })
   ```

### Instructions 页面写作范式

一个好的 Instructions 页面通常包含：

1. **角色定义**：一两句话说清 Agent 是谁、做什么
2. **输入协议**：Agent 期望收到什么格式的输入
3. **工作流程**：分步骤的操作流程（用编号列表）
4. **输出协议**：Agent 应该返回什么、格式是什么
5. **铁律/约束**：不可违反的规则
6. **上下文加载策略**（如果 Agent 需要读取外部资源）：
   - 哪些文件必须加载
   - 加载顺序
   - 不加载什么（context 窗口管理）

### 上下文加载策略设计

当 Agent 需要处理跨 session 的长期任务时，Instructions 中应规划 context 加载策略：

- **必加载清单**：每次 session 开始时必须加载的文件/页面（如 OUTLINE、SUMMARY）
- **按需加载清单**：根据当次任务决定是否加载（如某个章节的 refs）
- **禁止加载清单**：明确不要加载的内容（如其他章节全文，以节省 context 窗口）
- **桥梁文件**：用摘要文件（如 BOOK_SUMMARY）替代全文传递，在 context 窗口有限的情况下维持跨章感知

### 常见陷阱

- ⚠️ **Instructions 页面就是普通 Notion 页面**：使用 `updatePage` 的 `contentUpdates` 编辑，遵循 Notion-flavored Markdown 语法。不要用 `updateAgent` 编辑 instructions 内容——`updateAgent` 只能编辑 Agent 的 configuration（name、icon、integrations、triggers）。
- ⚠️ **Instructions 不能通过 createAgent 设置**：创建时 instructions 页面为空，必须后续通过 updatePage 写入。
- ⚠️ **过长的 Instructions 会挤压工作 context**：Instructions 内容会被 Agent 每次 session 加载。保持精炼，将详细参考资料放在外部文件中按需加载。

### 深钻指针

- Agent instructions 页面关联：`modules/notion/agents/index.ts` → `Agent.instructionsPageUrl`
- 页面内容编辑：`modules/notion/pages/AGENTS.md`、`modules/notion/pages/page-content-spec.md`
- Notion-flavored Markdown 语法：`modules/notion/notion-markdown.md`

---

## S4.8 Agent 更新

### 场景入口

修改已有 Agent 的 configuration——name、description、icon、integrations、triggers。

### 推荐路径

`updateAgent` 使用 edits 数组（与 `updateDatabase` 模式相同），每个 edit 操作在 `AgentConfiguration` 上执行。三种 edit 命令（`set` / `delete` / `replaceString`）的基本用法、path 规则和单字段修改示例见 `modules/notion/agents/index.ts` → `UpdateAgent`。

本节只覆盖 module files 未说明的增量场景。

### 增量场景 1：同时创建 Integration 和关联 Trigger

同一次 `updateAgent` 调用中可以用 `CREATE-*` 互相引用，实现跨 integration 类型的组合操作：

```typescript
await connections.notion.updateAgent({
  agentUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  edits: [
    {
      command: "set",
      path: ["integrations", "CREATE-slack-1"],
      value: { type: "slack", name: "Slack" },
    },
    {
      command: "set",
      path: ["triggers", "CREATE-trigger-1"],
      value: {
        enabled: true,
        integrationUrl: "CREATE-slack-1",  // 引用同批创建的 integration
        state: { type: "slack.app.mention" },
      },
    },
  ],
})
```

### 增量场景 2：修改 Integration 权限（数组覆盖行为）

`set` 某个 integration 的 `permissions` 会**整体覆盖**现有数组，没有 append 操作。安全做法：先 `loadAgent` 获取当前 permissions，合并后 set 完整新数组。

```typescript
// 先读取当前权限
const agent = await connections.notion.loadAgent({ url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" })
const currentPerms = agent.configuration.integrations["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"].permissions

// 合并后整体覆盖
await connections.notion.updateAgent({
  agentUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  edits: [
    {
      command: "set",
      path: ["integrations", "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "permissions"],
      value: [
        ...currentPerms,
        { identifier: { type: "agent", url: "okrs" }, actions: ["interact"] },
      ],
    },
  ],
})
```

### 常见陷阱

- ⚠️ **set permissions 是全量覆盖**：只想加一条权限时，必须先读后写。直接 set 新数组会丢失旧权限。
- ⚠️ **CREATE-\* 在 updateAgent 中同样有效**：可以在 edits 中同时创建新 integration 和新 trigger，并用 CREATE-* 互相引用。

### 深钻指针

- updateAgent 完整类型、基础 edit 命令（set / delete / replaceString）与单字段示例：`modules/notion/agents/index.ts` → `UpdateAgent`
- edit 命令类型复用自数据库的 EditJSONToolEdit 模式

---

## S4.9 Autofill Agent

### 场景入口

创建专门用于自动填充数据库行/属性的 Agent。

### 适用条件

- 数据库中有需要 AI 自动填充的属性（如摘要、分类、标签）
- 需要在页面创建或更新时自动补全信息

### 概念说明

Autofill Agent 是一种特殊用途的 Custom Agent，专门附着在数据库上，负责批量填充行/属性。在 `loadDatabase` 的返回结果中，`autofillCustomAgents` 字段列出了附着在该数据库上的 autofill agents：

```typescript
const db = await connections.notion.loadDatabase({ url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" })
// db.autofillCustomAgents → [{ url: "teams", name: "Auto Tagger" }]
```

### 与普通 Custom Agent 的区别

| 维度 | 普通 Custom Agent | Autofill Agent |
|------|------------------|----------------|
| 触发方式 | 多种 trigger / @mention / 手动调用 | 绑定到数据库的页面事件 |
| 操作范围 | workspace 任意内容 | 聚焦于目标数据库的属性填充 |
| 附着位置 | 独立存在 | 附着在特定数据库上 |
| 运行时变量 | 取决于 trigger 类型 | 包含 `fillablePropertyNames`（Agent 可填充的属性名列表） |

### 关键参数

当 Autofill Agent 被数据库事件触发时，trigger 变量中包含 `fillablePropertyNames`——这是用户指定的、允许 Agent 填充的属性名列表。Agent 应只操作这些属性。

### 常见陷阱

- ⚠️ **Autofill Agent 的创建目前通过 Notion UI 完成**：module files 中暴露的 API 不包含专门的 `createAutofillAgent` 函数。`DatabaseResult.autofillCustomAgents` 是只读字段，用于查询已附着的 autofill agents。
- ⚠️ **fillablePropertyNames 是用户配置的约束**：Agent 应尊重此列表，不要填充列表之外的属性。

### 深钻指针

- DatabaseResult.autofillCustomAgents 字段：`modules/notion/databases/index.ts` → `DatabaseResult`
- 页面创建/更新 trigger 变量中的 fillablePropertyNames：`modules/notion/triggers.ts` → `DatabasePageCreatedTriggerVariables`、`DatabasePageUpdatedTriggerVariables`

---

## S4.10 多 Agent 协作流程设计

### 场景入口

设计多个 Agent 协作完成复杂任务的工作流。

### 适用条件

- 任务涉及多个专业领域（如写作 + 审阅 + 排版）
- 需要职责分离和质量闭环
- 单个 Agent 的 context 窗口不足以承载全部工作

### 推荐路径

#### 模式一：总编-著作郎-校勘官

**核心思想**：职责分离 + 链式传递 + 质量闭环

| 角色 | 职责 | 关键能力 |
|------|------|----------|
| 总编 | 全流程调度、context 管理、摘要维护 | createAndRunThread 调用其他 Agent |
| 著作郎 | 接收指令、加载 context、写作、交稿 | 加载外部文件 + 写入内容 + commit |
| 校勘官 | 审阅 draft、返回逐条意见 | 读取已有内容 + 对照标准检查 |

**通信链路**：总编 → 著作郎（派写）→ 总编 → 校勘官（派审）→ 总编

**关键设计**：
1. 指令自包含：每次 createAndRunThread 的 instructions 包含全部必要信息，不依赖子 Agent「记住」之前的对话
2. 返回值约定：子 Agent 不返回全文，返回摘要（如 commit hash + 字数 + 问题清单）
3. 桥梁文件：用 BOOK_SUMMARY 这类摘要文件传递跨章 context，避免加载全文

**权限配置**：
```typescript
// 总编需要 interact 著作郎和校勘官
permissions: [
  { identifier: "*", actions: ["edit"] },
  { identifier: { type: "agent", url: "dataSourceUrl" }, actions: ["interact"] },
  { identifier: { type: "agent", url: "okrs" }, actions: ["interact"] },
]
```

#### 模式二：执行手册模式

**核心思想**：用 Notion 页面作为持久化状态源，跨 session 恢复进度

1. 创建一个「执行手册」页面，包含：
   - 任务清单及每项的状态（待处理 / 进行中 / 已完成）
   - 当前阶段标记
   - 累积的中间结果（如已完成章节的 commit hash）
2. Agent 每次启动时：
   - `loadPage` 读取执行手册
   - 根据状态决定下一步
   - 执行完毕后 `updatePage` 更新状态
3. 下次触发时从中断处继续

#### 模式三：分身并行

**核心思想**：一个 Agent 开多个子 thread 调用自身的不同实例，并行处理后汇总

```typescript
// 总编分配 3 个章节给著作郎的 3 个分身
const ch1 = connections.notion.createAndRunThread({
  agentUrl: "dataSourceUrl",
  instructions: "写 Ch1...",
})
const ch2 = connections.notion.createAndRunThread({
  agentUrl: "dataSourceUrl",
  instructions: "写 Ch2...",
})
const ch3 = connections.notion.createAndRunThread({
  agentUrl: "dataSourceUrl",
  instructions: "写 Ch3...",
})
const results = await Promise.all([ch1, ch2, ch3])
// 汇总 3 个分身的返回摘要
```

### 状态传递策略

| 方式 | 适用场景 | 示例 |
|------|----------|------|
| instructions 内嵌 | 一次性任务，信息量小 | createAndRunThread 的 instructions 中包含全部指令 |
| Notion 页面 | 跨 session 持久化状态 | 执行手册页面记录任务进度 |
| 外部存储（GitHub） | 大型产物（如章节全文） | commit 到 repo，传递 commit hash |
| 桥梁文件 | 跨章/跨任务的 context 压缩 | BOOK_SUMMARY 记录每章要点 |

### 常见陷阱

- ⚠️ **50 次子线程上限是全局的**：一个 parent thread 中所有 createAndRunThread 调用（无论目标 Agent 是谁）共享 50 次配额。在多 Agent 协作中，总编需要仔细分配调用预算。
- ⚠️ **子 Agent 的副作用可能冲突**：并行的子 Agent 如果操作同一个页面或文件，可能产生竞争条件。设计时确保每个子 Agent 操作独立的资源。
- ⚠️ **指令自包含原则**：不要假设子 Agent 记得之前的对话。每次 createAndRunThread 的 instructions 应包含执行所需的全部信息（repo 地址、文件路径、context 文件清单、写作风格要求等）。
- ⚠️ **不要在 response 中传大文本**：子 Agent 应将大型产物写入 Notion 页面或外部存储，只在 response 中返回指针（URL、hash）和摘要。

### 深钻指针

- createAndRunThread 完整类型：`modules/notion/threads/index.ts`
- system.updateTodos 进度跟踪：`modules/system/index.ts`
- system.wait 任务暂停与恢复：`modules/system/index.ts`

---

## S4.11 Agent 调试

### 场景入口

排查 Agent 行为异常——查看历史线程、分析对话记录、诊断通信失败。

### 适用条件

- Agent 行为不符合预期
- createAndRunThread 调用失败
- 需要回溯某次运行的详细过程

### 操作步骤

#### 1. 查询历史线程

```typescript
const { threads } = await connections.notion.queryThreads({
  agentUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",  // 可选，默认当前 Agent
  limit: 10,
  timeStart: "2025-01-01T00:00:00Z",  // 可选
  timeEnd: "2025-01-31T23:59:59Z",    // 可选
})
// threads → [{ url, createdTime, title? }, ...]
```

#### 2. 分析特定线程

```typescript
const { analysis } = await connections.notion.investigateThread({
  threadUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  instructions: "这次运行中，Agent 是否成功读取了 OUTLINE.md？如果失败，失败原因是什么？",
})
// analysis → 基于 thread transcript 的分析报告
```

### 常见调试场景

| 症状 | 可能原因 | 排查方式 |
|------|----------|----------|
| createAndRunThread 报 "request denied" | 缺少 interact 权限 | loadAgent 检查调用方的 permissions |
| Sub-agent response 为空或无关 | instructions 不清晰 | investigateThread 分析子线程 |
| Agent 没有执行预期操作 | 权限不足 / instructions 不明确 | loadAgent 检查权限 + investigateThread 分析行为 |
| 报 "max sub agents exceeded" | 子线程调用超过 50 次 | 优化调用策略，合并小任务 |
| MCP 工具调用失败 | 工具名错误 / 参数格式错误 / 未授权 | listTools 检查可用工具 + 确认认证状态 |

### 关键参数

- `queryThreads` 的 `agentUrl` 可选——省略时查询当前 Agent 自身的线程
- `investigateThread` 的 `instructions` 应包含具体的调查问题，而非泛泛的「分析这次运行」
- `timeStart` / `timeEnd` 使用 ISO-8601 格式

### 常见陷阱

- ⚠️ **investigateThread 是分析工具不是重放工具**：它基于 thread transcript 做分析，不会重新执行线程中的操作。
- ⚠️ **queryThreads 只返回元数据**：返回的是 `{ url, createdTime, title? }`，不包含对话内容。需要内容分析时用 investigateThread。
- ⚠️ **权限问题往往是最常见的 root cause**：Agent 间通信失败、页面操作被拒，首先检查 Agent 的 Notion integration permissions。

### 深钻指针

- queryThreads / investigateThread 签名：`modules/notion/threads/index.ts`
- Agent 权限检查：`modules/notion/agents/index.ts` → `loadAgent` 返回的 configuration.integrations

---

## S4.12 Notification 发送

### 场景入口

向用户发送 Notion 通知——任务完成通知、异常告警、进度报告。

### 适用条件

- Agent 完成了长时间任务，需要通知用户
- 发生了需要用户介入的异常
- 定时报告类场景

### 操作步骤

```typescript
await connections.notion.sendNotification({
  headerContent: "Ch4 写作完成",
  bodyContent: "draft v1 已 commit，共 8500 字",  // 不超过 100 字符
  sendToWorkflowOwner: true,
})
```

发送给特定用户：

```typescript
await connections.notion.sendNotification({
  headerContent: "审阅请求",
  bodyContent: "Ch4 draft 待审阅，请查看",
  userUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  sendToWorkflowOwner: false,
})
```

### 关键参数

函数签名与参数定义见 `modules/notion/notifications.ts` → `SendNotification`。核心使用要点：

- `bodyContent` 不超过 100 字符，超出可能被截断
- `sendToWorkflowOwner: true` 且省略 `userUrl` → 发给 workflow owner
- 指定 `userUrl` 且 `sendToWorkflowOwner: false` → 发给特定用户

### 常见陷阱

- ⚠️ **bodyContent 100 字符限制**：超过 100 字符可能被截断。将详细信息放在 Notion 页面中，通知只包含摘要和指针。
- ⚠️ **不要滥用通知**：Notification 是提醒工具不是通信工具。Agent 间通信用 createAndRunThread，不用 sendNotification。
- ⚠️ **sendToWorkflowOwner 和 userUrl 的关系**：如果只想发给 workflow owner，设 `sendToWorkflowOwner: true` 并省略 `userUrl`。如果想发给特定用户，设 `userUrl` 并设 `sendToWorkflowOwner: false`。

### 深钻指针

- sendNotification 签名：`modules/notion/notifications.ts` → `SendNotification`

---

## 本章涉及的 Module Files 完整清单

| 文件路径 | 内容 |
|----------|------|
| `modules/notion/agents/AGENTS.md` | Agent 模块概述与文件路由 |
| `modules/notion/agents/index.ts` | Agent 类型定义、createAgent、updateAgent、loadAgent |
| `modules/notion/integration.ts` | Notion integration 权限类型（content / search / agent interact） |
| `modules/notion/triggers.ts` | 全部 trigger 配置类型与运行时变量类型 |
| `modules/notion/threads/index.ts` | queryThreads、investigateThread、createAndRunThread |
| `modules/notion/notifications.ts` | sendNotification |
| `modules/notion/pages/AGENTS.md` | 页面操作指南（含 instructions 页面编辑） |
| `modules/notion/pages/index.ts` | createPage、updatePage、loadPage 等页面函数 |
| `modules/notion/pages/page-content-spec.md` | 页面内容格式规范 |
| `modules/notion/notion-markdown.md` | Notion-flavored Markdown 语法参考 |
| `modules/notion/databases/index.ts` | DatabaseResult.autofillCustomAgents 字段 |
| `modules/notion/users/AGENTS.md` | 用户连接管理（含 MCP Server 发现） |
| `modules/notion/users/index.ts` | listUserConnections、createUserConnection |
| `modules/mcpServer/index.ts` | MCP 工具类型、listTools、runTool |
| `modules/mcpServer/integration.ts` | MCP integration 类型（无 AI 可配权限） |
| `modules/system/index.ts` | wait、updateTodos（进度跟踪与任务暂停） |
| `modules/notion/AGENTS.md` | Notion 模块总览与文件路由 |

---

## 本章引入的新术语

| 术语 | 定义 | 首次出现 |
|------|------|----------|
| Custom Agent | workspace 内用户创建的 AI 行动者，拥有独立的 instructions、integrations、triggers | S4.1 |
| Instructions Page | Agent 的指令页面，本质是 Notion Page，通过 instructionsPageUrl 关联 | S4.1 |
| Integration | Agent 与一个 module 的连接记录，含 type、name、permissions | S4.1 |
| Permission（Agent 层） | 挂在 Notion integration 下的访问控制项，含 identifier + actions | S4.2 |
| interact 权限 | Agent 间通信的授权，identifier 为 `{ type: "agent", url: "dataSourceUrl" }`，actions 为 `["interact"]` | S4.2 |
| MCP Server | 外部工具服务协议，Agent 通过 mcpServer integration 连接后调用其 tools | S4.3 |
| Trigger | Agent 的自动触发条件，含 enabled、state（类型配置）、integrationUrl | S4.4 |
| Recurrence Trigger | 定时触发器，支持 hour/day/week/month/year 频率 | S4.4 |
| Sub-thread | 通过 createAndRunThread 开的子线程，同步运行并返回最终文本 | S4.6 |
| 50 次上限 | 单个 parent thread 调用 createAndRunThread 的硬性次数限制 | S4.6 |
| Skill | Agent 的 Instructions Page 中编写的行为定义，包含角色、流程、约束 | S4.7 |
| Autofill Agent | 专门附着在数据库上、负责批量填充属性的特殊 Custom Agent | S4.9 |
| fillablePropertyNames | Autofill Agent trigger 变量中的属性名列表，限定可填充范围 | S4.9 |
| 执行手册模式 | 用 Notion 页面作持久化状态源，实现跨 session 恢复进度的多 Agent 协作模式 | S4.10 |
| 桥梁文件 | 用摘要文件（如 BOOK_SUMMARY）替代全文传递的 context 管理策略 | S4.10 |
| CREATE-\* 占位符 | 同一次 create/update 调用中用于标识新建实体的临时 key | 薄概念层 |
| compressed URL | 系统分配的实体标识符，格式为 `"prefix-id"` | 薄概念层 |
