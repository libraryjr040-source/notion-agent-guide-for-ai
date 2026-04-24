# Ch6 踩坑与优化

> **定位**：横切失败模式速查 + 性能优化策略。读者（AI Agent）遇到问题时加载本章，按症状定位根因，找到修复方案。
>
> **组织原则**：按因果机制分类，而非按 module 分类。同一个 module 的不同失败模式可能分散在不同 section。
>
> **独立性**：本章独立可加载，不假设已读过其他章节。每个场景条目自包含。

---

## 薄概念 DRY 层

本章涉及的核心概念关系：

- **compressed URL**：系统分配的实体标识符（如 `"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`、`"dataSourceUrl"`），是所有 API 调用中引用实体的唯一合法方式。永远不要自行构造——只使用 `loadPage`、`loadDatabase`、`loadAgent` 等函数返回的值。
- **CREATE-\* 占位符**：仅在同一次 `createDatabase` / `createAgent` 调用内部使用的临时 key，系统会在创建完成后替换为真实 compressed URL。跨调用使用 CREATE-\* 必定失败。
- **property key**：数据源 schema 中属性的列名，大小写敏感。`"Status"` ≠ `"status"`。
- **展开列名**：date 和 place 类型的属性在 SQL 查询和 propertyUpdates 中使用展开格式（如 `"date:Due Date:start"`、`"place:Location:address"`），而非直接使用属性名。
- **interact 权限**：Custom Agent 间通信的授权机制，必须在调用方的 Notion integration permissions 中显式配置。

**Module files 导航**：

| 概念域 | 关键文件 |
|--------|----------|
| 页面操作 | `modules/notion/pages/index.ts`, `modules/notion/pages/AGENTS.md` |
| 页面内容格式 | `modules/notion/pages/page-content-spec.md`, `modules/notion/notion-markdown.md` |
| 数据库与 schema | `modules/notion/databases/index.ts`, `modules/notion/databases/dataSourceTypes.ts` |
| SQL 查询与列映射 | `modules/notion/databases/data-source-sqlite-tables.md` |
| View 与 form | `modules/notion/databases/viewTypes.ts` |
| Agent 配置 | `modules/notion/agents/index.ts`, `modules/notion/integration.ts` |
| Agent 通信 | `modules/notion/threads/index.ts` |
| 搜索 | `modules/search/index.ts`, `modules/search/AGENTS.md` |
| MCP | `modules/mcpServer/index.ts`, `modules/mcpServer/AGENTS.md` |
| 分享权限 | `modules/notion/sharing.ts` |

---

## S6.1 身份与权限类

**失败模式类别**：Agent 的权限配置不足或不正确，导致操作被拒绝。

### 6.1.1 Agent 无权访问页面 / 数据库

**典型症状**：调用 `loadPage`、`updatePage`、`loadDatabase` 等函数时返回权限错误，提示无法访问目标资源。

**根因分析**：Custom Agent 的 Notion integration 中 `permissions` 数组未包含目标页面/数据库的 compressed URL，或者 actions 级别不足（例如只有 `"view"` 但需要 `"edit"`）。

**修复方案**：

1. 通过 `loadAgent` 检查当前 agent 的 `configuration.integrations` 下 Notion integration 的 `permissions` 数组
2. 确认目标资源的 compressed URL 是否在 permissions 中，且 actions 包含所需级别
3. 通过 `updateAgent` 添加或升级权限：

```jsonc
// 添加对特定页面的编辑权限
{
  "command": "set",
  "path": ["integrations", "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "permissions"],
  "value": [
    { "identifier": "URL", "actions": ["edit"] },
    { "identifier": "*", "actions": ["view"] }
    // ... 保留原有权限项
  ]
}
```

**预防策略**：
- 使用 `"*"` 通配符授予 workspace 级别访问（适合需要广泛读写的 agent）
- 注意 `"*"` 必须作为唯一 identifier，不能与其他标识符组合在同一权限项中
- 页面/数据库权限通过 `sharing.ts` 的 `updatePermission` 管理的是用户级分享，不等同于 agent 权限——agent 权限必须通过 `loadAgent` / `updateAgent` 单独配置

**深钻指针**：`modules/notion/integration.ts`（权限类型定义）、`modules/notion/agents/index.ts`（updateAgent edits 格式）

### 6.1.2 interact 权限缺失

**典型症状**：调用 `createAndRunThread` 向另一个 Custom Agent 发消息时被拒绝，错误提示无权通信。

**根因分析**：调用方 agent 的 Notion integration permissions 中缺少对目标 agent 的 interact 权限声明。interact 权限是单向的：A 要调 B，A 的权限中必须有 B 的 interact 声明。

**修复方案**：

在调用方 agent 的 Notion integration permissions 中添加：

```jsonc
{ "identifier": { "type": "agent", "url": "okrs" }, "actions": ["interact"] }
```

**预防策略**：
- 在设计多 agent 架构时，提前绘制通信拓扑图，确认每条通信链路的 interact 权限
- interact 权限不包含在 `"*"` 通配符中——`"*"` 只覆盖页面/数据库访问，agent 间通信必须逐个声明
- 自调自（agent 调自己的 `createAndRunThread`）不需要 interact 权限，只需省略 `agentUrl` 参数

**深钻指针**：`modules/notion/integration.ts`（interact 权限格式）、`modules/notion/threads/index.ts`（createAndRunThread 签名与约束）

### 6.1.3 MCP 未授权

**典型症状**：调用 `connections.mcpServer_xxx.runTool` 时报错，提示连接不可用或工具不存在。

**根因分析**：Agent 的 `configuration.integrations` 中没有对应的 mcpServer 类型 integration，或者 MCP Server 连接在 workspace 层面未建立。

**修复方案**：

1. 检查 `loadAgent` 返回的 `configuration.integrations`，确认是否有 `type: "mcpServer"` 的集成
2. 如果没有，通过 `updateAgent` 添加 MCP integration：

```jsonc
{
  "command": "set",
  "path": ["integrations", "CREATE-mcp-github"],
  "value": {
    "type": "mcpServer",
    "name": "GitHub"
  }
}
```

3. 注意：MCP 连接需要在 Notion workspace 的 Settings 中预先建立，API 只能在 agent 层面引用已有的连接

**预防策略**：
- MCP integration 的权限独立于 Notion integration——添加 MCP 不会自动获得 Notion 页面访问权限，反之亦然
- 每种 MCP 连接在 `connections.ts` 中有独立的 key（如 `connections.mcpServer_github`），确认 key 名称与实际配置一致

**深钻指针**：`modules/mcpServer/index.ts`（listTools / runTool 接口）、`modules/mcpServer/AGENTS.md`

---

## S6.2 数据格式类

**失败模式类别**：属性值的格式与系统期望不匹配，导致写入失败或数据异常。

### 6.2.1 date 展开列名错误

**典型症状**：通过 `updatePage` 设置日期属性时报错，或日期值未正确写入。直接使用属性名（如 `"Due Date": "2025-01-15"`）作为 key 会失败。

**根因分析**：date 类型属性在 propertyUpdates 和 SQL 查询中必须使用展开列名格式，而非直接属性名。系统将一个 date 属性展开为三列。

**修复方案**：

使用展开格式的三个 key：

```jsonc
{
  "url": "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  "propertyUpdates": {
    "date:Due Date:start": "2025-01-15",
    "date:Due Date:end": null,
    "date:Due Date:is_datetime": 0
  }
}
```

- `start`：ISO-8601 日期或日期时间字符串（必填）
- `end`：ISO-8601 字符串（日期范围时设置，单日期设为 `null`）
- `is_datetime`：`1` 表示日期时间，`0` 表示纯日期，`null` 默认为 `0`

**预防策略**：
- 看到 schema 中 `type: "date"` 的属性，立即切换到 `"date:<属性名>:start/end/is_datetime"` 模式
- SQL 查询中同理：`SELECT "date:Due Date:start" FROM "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`
- 属性名中的空格、特殊字符保持原样，不要转义

**深钻指针**：`modules/notion/databases/data-source-sqlite-tables.md`（Date 列展开规则）、`modules/notion/pages/AGENTS.md`（Property value formats → Date）

### 6.2.2 place 展开方式错误

**典型症状**：通过 `updatePage` 设置地点属性时报错，或数据写入位置不正确。

**根因分析**：与 date 类似，place 类型属性也使用展开列名格式，且展开为五列。直接使用属性名设置值会失败。

**修复方案**：

```jsonc
{
  "url": "dataSourceUrl",
  "propertyUpdates": {
    "place:Location:address": "1600 Pennsylvania Ave NW, Washington, DC 20500"
  }
}
```

完整展开列：
- `"place:<属性名>:address"`：地址字符串
- `"place:<属性名>:name"`：可选地点名称
- `"place:<属性名>:latitude"`：纬度（FLOAT）
- `"place:<属性名>:longitude"`：经度（FLOAT）
- `"place:<属性名>:google_place_id"`：可选 Google Place ID

**预防策略**：
- 最简有效格式是只设 `address`，系统可通过 geocoding 填充经纬度
- 永远不要尝试 `"Location": "some address"` 这种直接赋值

**深钻指针**：`modules/notion/databases/data-source-sqlite-tables.md`（Place 列展开规则）、`modules/notion/pages/AGENTS.md`（Property value formats → Place）

### 6.2.3 checkbox 值格式

**典型症状**：在 SQL 查询中用 `true` / `false` 过滤 checkbox 属性无结果。

**根因分析**：checkbox 属性在 SQLite 中存储为 TEXT 类型，值是 `"__YES__"` 和 `"__NO__"`，不是布尔值。但在 `propertyUpdates` 中，使用原生布尔值 `true` / `false`。两个上下文的格式不同。

**修复方案**：

- **SQL 查询**中过滤 checkbox：

```sql
SELECT url FROM "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" WHERE "Is Active" = '__YES__'
```

- **propertyUpdates** 中设置 checkbox：

```jsonc
{ "url": "okrs", "propertyUpdates": { "Is Active": true } }
```

**预防策略**：
- 记住规则：SQL 查 `"__YES__"` / `"__NO__"`，写用 `true` / `false`
- `NULL` 在 SQL 中默认视为 `false`

**深钻指针**：`modules/notion/databases/data-source-sqlite-tables.md`（Checkbox 列说明）

### 6.2.4 JSON 数组 vs 单值

**典型症状**：relation / person / multi-select 属性在查询和写入时行为不一致，或 `json_each` 解析失败。

**根因分析**：
- **SQL 查询返回值**：multi-select、person（无 limit）、relation（无 limit）返回 JSON 数组字符串（如 `'["A","B"]'`）；person（limit=1）、relation（limit=1）可能返回单个 JSON 字符串
- **propertyUpdates 写入值**：multi-select 使用数组 `["A", "B"]`；person 和 relation 根据 limit 使用单值或数组

**修复方案**：

- SQL 中解析数组值用 `json_each`：

```sql
SELECT url FROM "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" t
WHERE EXISTS (
  SELECT 1 FROM json_each(t."Tags") WHERE value = 'Important'
)
```

- SQL 中处理 limit=1 的 relation：

```sql
SELECT url, json_extract("Project", '$') AS project_url
FROM "dataSourceUrl"
```

- propertyUpdates 写入：

```jsonc
// multi-select: 始终用数组
{ "Tags": ["Important", "Urgent"] }

// relation (limit=1): 单值字符串
{ "Project": "OPTIONAL_NOTICE" }

// relation (无 limit): 数组
{ "Related Tasks": ["connector-*-1", "https://www.notion.so/7fd3f523da6944cd8119bcfeb2d8df72"] }

// person (limit=1): 单值字符串
{ "Owner": "okrs" }

// person (无 limit): 数组
{ "Reviewers": ["okrs", "teams"] }
```

**预防策略**：
- 先通过 `loadDatabase` 或 `loadDataSource` 查看 schema，确认属性是否有 `limit: 1`
- SQL 查询中，对可能是数组的列，默认用 `json_each` 最安全

**深钻指针**：`modules/notion/databases/data-source-sqlite-tables.md`（Relation / Person / Multi-select 列说明）、`modules/notion/pages/AGENTS.md`（Property value formats）

---

## S6.3 引用与标识类

**失败模式类别**：使用了错误的标识符格式引用实体，导致系统无法解析。

### 6.3.1 用 name 而非 URL 引用实体

**典型症状**：在 `createDatabase` 的 schema 中用属性名作为 key（如 `"Title": { type: "title", ... }`），或在 view 定义中用视图名作为 key，系统报错或行为异常。

**根因分析**：Notion 的所有实体（data source、property、view）以 compressed URL 作为唯一标识。名称是显示用的 `name` 字段，不是 key。

**修复方案**：

```jsonc
// ❌ 错误：用属性名作 key
"schema": {
  "Title": { "type": "title", "name": "Title" }
}

// ✅ 正确：新建用 CREATE-* 占位符
"schema": {
  "CREATE-title": { "type": "title", "name": "Title" }
}

// ✅ 正确：已有属性用 compressed URL
"schema": {
  "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40": { "type": "title", "name": "Title" }
}
```

**预防策略**：
- 所有 record key 只用两种格式：`CREATE-*`（新建）或 `xxx`（已有）
- display name 只出现在对象内部的 `name` 字段中

**深钻指针**：`modules/notion/databases/AGENTS.md`（Identifiers: CREATE-\* vs compressed URLs）、`modules/notion/databases/index.ts`（CreateDatabase 注释）

### 6.3.2 CREATE-\* 与 compressed URL 混淆

**典型症状**：在 `updateDatabase` 的 edits 中对已有实体使用 `CREATE-*` key，导致系统创建了重复实体而非更新现有实体；或反过来，在 `createDatabase` 中对新实体使用了不存在的 compressed URL。

**根因分析**：
- `CREATE-*` 是临时占位符，仅在同一次 create 调用中有效。创建完成后，系统返回真实 compressed URL，后续必须使用真实 URL
- compressed URL 是系统分配的，永远不要自行构造（如手写 `"https://www.notion.so/34cc9fa44e9780738c20f6409bc60091"`）

**修复方案**：

1. `createDatabase` / `createAgent`：全部用 `CREATE-*`
2. `updateDatabase` / `updateAgent`：已有实体用 compressed URL（从 create 或 load 的返回值中获取），新增实体用 `CREATE-*`
3. 一次 create 调用中，`CREATE-*` 可以互相引用（如 view 的 `dataSourceUrl` 引用 `"CREATE-main"`），这是它存在的核心价值

**预防策略**：
- 每次 create / load 后，立即记录返回的 compressed URL，后续操作全部使用这些 URL
- 永远不要跨调用复用 `CREATE-*` 标识符

**深钻指针**：`modules/notion/databases/AGENTS.md`（Identifiers 章节）

### 6.3.3 property key 大小写错误

**典型症状**：`updatePage` 的 `propertyUpdates` 或 `querySql` 中引用了属性名但无效果，或报属性不存在。

**根因分析**：property key 是大小写敏感的。schema 中定义为 `"Status"` 的属性，用 `"status"` 引用会失败。

**修复方案**：

1. 通过 `loadDatabase` 或 `loadDataSource` 获取 schema，记录每个属性的精确 `name`
2. 在 `propertyUpdates` 和 SQL 查询中使用完全匹配的名称

```jsonc
// ❌ 大小写不匹配
{ "propertyUpdates": { "status": "Done" } }

// ✅ 精确匹配 schema 中的 name
{ "propertyUpdates": { "Status": "Done" } }
```

**预防策略**：
- 养成习惯：操作数据库页面前，先 `loadDatabase` 查看 schema
- 如果属性名与系统列名冲突（`id`、`url`、`createdTime`），需使用 `userDefined:` 前缀：如 `"userDefined:url"`

**深钻指针**：`modules/notion/pages/AGENTS.md`（Property naming）、`modules/notion/databases/data-source-sqlite-tables.md`（Column naming）

---

## S6.4 内容编辑类

**失败模式类别**：`updatePage` 的 `contentUpdates` 操作因匹配或格式问题被拒绝。

### 6.4.1 oldStr 匹配失败

**典型症状**：`updatePage` 报错，提示 `oldStr` 在页面内容中找不到匹配。

**根因分析**：
- `oldStr` 必须精确匹配页面当前内容中的一段文本（包括空格、换行、缩进）
- 页面内容可能在上次 `loadPage` 后被其他操作修改过
- Notion-flavored Markdown 的特殊语法（如 `{color="red"}`、`<callout>` 标签）可能被遗漏

**修复方案**：

1. 确保使用最新的页面内容生成 `oldStr`——如果不确定内容是否最新，先 `loadPage` 刷新
2. `oldStr` 需要包含足够的上下文使其在页面中唯一
3. 不要在 `oldStr` 或 `newStr` 中包含 `<content>` / `</content>` 包装标签

```jsonc
// ❌ oldStr 太短，可能在页面中多处出现
{ "oldStr": "Done", "newStr": "Completed" }

// ✅ 包含足够上下文确保唯一性
{ "oldStr": "- Status: Done\n- Priority: High", "newStr": "- Status: Completed\n- Priority: High" }
```

**预防策略**：
- `oldStr` 尽量大到唯一，但不要不必要地大
- 避免在同一个 `updatePage` 调用中对同一页面发多次 contentUpdates——合并为一次调用
- 永远不要对同一个页面 URL 并行调用 `updatePage`

**深钻指针**：`modules/notion/pages/index.ts`（UpdatePage 注释和 contentUpdates 规范）

### 6.4.2 多处匹配未设 replaceAllMatches

**典型症状**：`updatePage` 报错，提示 `oldStr` 匹配到多处，但未指定处理方式。

**根因分析**：默认情况下，`oldStr` 必须在页面中唯一匹配。如果存在多处匹配且未设置 `replaceAllMatches: true`，系统会拒绝操作以防止意外修改。

**修复方案**：

根据意图选择策略：

```jsonc
// 场景 A：全局查找替换（如统一术语）
{
  "oldStr": "旧术语",
  "newStr": "新术语",
  "replaceAllMatches": true
}

// 场景 B：只改特定位置——扩大 oldStr 上下文使其唯一
{
  "oldStr": "## 第二节\n旧术语出现在这里",
  "newStr": "## 第二节\n新术语出现在这里"
}
```

**预防策略**：
- 全局替换场景（重命名术语、统一格式）：设 `replaceAllMatches: true`
- 定点编辑场景（插入内容、修改特定段落）：不设 `replaceAllMatches`，通过扩大 `oldStr` 确保唯一

**深钻指针**：`modules/notion/pages/index.ts`（contentUpdates 中 `replaceAllMatches` 说明）

### 6.4.3 database block 被修改导致拒绝

**典型症状**：`updatePage` 的 contentUpdates 被整体拒绝，即使你修改的不是 database block 部分。

**根因分析**：页面内容中的 `<database url="...">...</database>` 块代表嵌入的数据库。如果 contentUpdates 的 `oldStr` 或 `newStr` 中包含 database block 且其属性或内容被修改（哪怕只是空格变化），整个更新会被拒绝。

**修复方案**：

1. 编辑页面内容时，如果 `oldStr` 范围覆盖了 database block，必须原封不动地保留 `<database>` 标签及其属性和内部文本
2. 不要通过页面内容编辑来创建新的 database block——使用 `createDatabase` 并设 parent 为目标页面
3. 可以通过 `updatePage` 重新定位已有 database block（移动位置）或移除 database block

```jsonc
// ❌ 修改了 database block 的文本
{
  "oldStr": "<database url=\"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40\" inline=\"false\">Tasks</database>",
  "newStr": "<database url=\"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40\" inline=\"false\">My Tasks</database>"
}

// ✅ 保持 database block 原样，只修改周围内容
{
  "oldStr": "## 任务\n<database url=\"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40\" inline=\"false\">Tasks</database>",
  "newStr": "## 任务列表\n<database url=\"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40\" inline=\"false\">Tasks</database>"
}
```

**预防策略**：
- 把 database block 当作不可变的嵌入物——位置可移、内容不可改
- 需要修改数据库本身（名称、schema、view），使用 `updateDatabase`

**深钻指针**：`modules/notion/pages/page-content-spec.md`（Database blocks 章节）

---

## S6.5 Wiki 与特殊数据库类

**失败模式类别**：Wiki 数据库和 linked database 的特殊行为导致操作偏差。

### 6.5.1 Wiki parent 错误

**典型症状**：在 wiki 数据库中创建页面时，使用 `{ type: "dataSource", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" }` 作为 parent，页面创建失败或位置不正确。

**根因分析**：Wiki 数据库（`isWiki: true`）的页面创建必须使用 `{ type: "page", url: wikiPageUrl }` 作为 parent，而不是 dataSource parent。`wikiPageUrl` 是 wiki 的 collection view page，从 `loadDatabase` 的返回值中获取。

**修复方案**：

```jsonc
// 1. 先 loadDatabase 获取 wiki 信息
const db = await loadDatabase({ url: "URL" })
// db.configuration.isWiki === true
// db.configuration.wikiPageUrl === "integration://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40/notion/86b06b1b-9c54-4a7f-ac10-fefcf3d177a9"

// 2. 用 wikiPageUrl 作为 parent
await createPage({
  parent: { type: "page", url: "integration://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40/notion/86b06b1b-9c54-4a7f-ac10-fefcf3d177a9" },  // wikiPageUrl
  properties: { title: "新 Wiki 页面" }
})
```

**预防策略**：
- 操作数据库页面前，先 `loadDatabase` 检查 `isWiki` 字段
- 如果 `isWiki === true`，强制切换到 `{ type: "page", url: wikiPageUrl }` 模式

**深钻指针**：`modules/notion/databases/AGENTS.md`（Wiki databases 章节）、`modules/notion/pages/AGENTS.md`（Wiki databases 说明）

### 6.5.2 Linked database 的 dataSource 来源

**典型症状**：调用 `loadDatabase` 后发现 `dataSources` 为 `{}`，以为数据库是空的。

**根因分析**：linked database 不拥有自己的 data source——它的 view 引用其他数据库的外部 data source。`loadDatabase` 只返回 owned data sources，因此 linked database 的 `dataSources` 始终为 `{}`。

**修复方案**：

1. 检查 views 中的 `dataSourceUrl` 字段——这是 linked database 引用的外部 data source
2. 对外部 data source 执行 `loadDataSource` 获取 schema
3. 创建 linked database 时，`dataSources` 设为 `{}`，views 中用外部 `dataSourceUrl`

```jsonc
// 创建 linked database
await createDatabase({
  name: "Tasks View",
  parent: { type: "page", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/00429878-83b7-4209-8fc4-4f4757072e75" },
  dataSources: {},  // linked database 不拥有 data source
  views: {
    "CREATE-view": {
      type: "table",
      name: "All Tasks",
      dataSourceUrl: "okrs"  // 引用外部 data source
    }
  }
})
```

**预防策略**：
- `dataSources: {}` 不代表数据库为空，可能是 linked database
- 关键判断指标：view 的 `dataSourceUrl` 是否指向本数据库之外的 data source

**深钻指针**：`modules/notion/databases/AGENTS.md`（Linked databases 章节）

---

## S6.6 Search 与查询类

**失败模式类别**：搜索参数格式错误或策略不当，导致结果质量差或返回空结果。

### 6.6.1 lookback 用自然语言

**典型症状**：传入 `lookback: "last month"` 或 `lookback: "yesterday"`，搜索行为异常或参数被忽略。

**根因分析**：`lookback` 只接受以下格式：
- `"default"`
- `"all_time"`
- `<number><d|w|m|y>`（如 `"7d"`、`"2w"`、`"3m"`、`"1y"`）
- ISO 日期字符串（如 `"2024-04-01"`）

自然语言（`"last month"`、`"yesterday"`、`"this week"`）不被接受。

**修复方案**：

将自然语言转换为合法格式：

| 用户说的 | 应转换为 |
|----------|----------|
| "昨天" | `"1d"` |
| "上周" | `"1w"` |
| "上个月" | `"30d"` 或 `"1m"` |
| "最近的" / "最新的" | `"1w"`（先试），无结果再扩展到 `"1m"` |
| 永久性内容（密码、政策） | `"all_time"` |

**预防策略**：
- 默认用 `"default"`，除非用户明确指定时间范围
- 渐进扩大策略：`"1w"` → `"1m"` → `"all_time"`

**深钻指针**：`modules/search/AGENTS.md`（lookback 格式说明和示例）、`modules/search/index.ts`（SearchQueryInput 类型）

### 6.6.2 keywords 太泛

**典型症状**：搜索返回大量不相关结果，或关键内容被淹没。

**根因分析**：`keywords` 字段应提取 2-4 个最具区分度的词——实体名、缩写、ID、专有名词。把整个问题复述一遍到 keywords 中会稀释搜索精度。

**修复方案**：

```jsonc
// ❌ keywords 太泛
{
  "question": "What is the Q3 roadmap update?",
  "keywords": "what is the Q3 roadmap update for our team"
}

// ✅ 精准关键词
{
  "question": "What is the Q3 roadmap update?",
  "keywords": "Q3 roadmap update"
}
```

**预防策略**：
- keywords 不是问题的复述，是搜索引擎的锚点
- 优先提取：专有名词 > 缩写/ID > 动作动词 > 通用词
- 不要加 "in our workspace"、"find docs about" 这类填充词

**深钻指针**：`modules/search/AGENTS.md`（Writing search queries 章节）

### 6.6.3 Web results 未关闭

**典型症状**：用户要求搜索 workspace 内容，但结果被公网搜索结果稀释，干扰判断。

**根因分析**：`includeWebResults` 默认为 `true`。当用户明确要求搜索内部内容时，需手动设为 `false`。

**修复方案**：

```jsonc
await search({
  queries: [{
    question: "Our onboarding checklist",
    keywords: "onboarding checklist",
    lookback: "all_time"
  }],
  includeWebResults: false  // 只搜索内部源
})
```

**预防策略**：
- 用户说 "search Notion" / "search our workspace" → 设 `includeWebResults: false`
- 用户问题涉及公司内部信息（政策、密码、文档）→ 通常也应关闭 web results
- 想搜索 Notion 产品帮助文档时，设 `includeNotionHelpdocs: true`，不需要在 keywords 中加 "helpdocs"

**深钻指针**：`modules/search/AGENTS.md`（includeWebResults 和 includeNotionHelpdocs 说明）

---

## S6.7 Agent 通信类

**失败模式类别**：`createAndRunThread` 的使用约束导致通信失败或数据丢失。

### 6.7.1 createAndRunThread 被拒（无 interact 权限）

**典型症状**：调用 `createAndRunThread({ agentUrl: "URL" })` 时被拒绝。

**根因分析**：Custom Agent 向另一个 Custom Agent 发消息需要 interact 权限。该权限在调用方的 Notion integration 中声明，而非目标方。

**修复方案**：见 S6.1.2（interact 权限缺失）。

**预防策略**：
- 自调自不需要 interact 权限——省略 `agentUrl` 参数即可
- Personal Agent 调 Custom Agent 不需要 interact 权限
- 只有 Custom Agent → Custom Agent（非自己）需要 interact

**深钻指针**：`modules/notion/threads/index.ts`（createAndRunThread 注释中的权限规则）

### 6.7.2 Response 只有文本

**典型症状**：期望从 `createAndRunThread` 的返回中获取结构化数据（如 JSON 对象），但只收到纯文本字符串。

**根因分析**：`createAndRunThread` 返回的 `response` 字段只包含子 agent 的最终文本回复。子 agent 在运行过程中可能执行了页面创建、数据库更新等副作用操作，但这些操作的结果不会通过 response 返回——response 只是文本。

**修复方案**：

1. 如果需要子 agent 传回结构化信息，在 instructions 中要求子 agent 以特定格式（如 JSON）输出到 response 文本中，然后在父 agent 端解析
2. 如果需要子 agent 的操作产物（如创建的页面 URL），让子 agent 在 response 中显式报告
3. 对于复杂数据传递，让子 agent 将结果写入约定的 Notion 页面，父 agent 通过 `loadPage` 读取

**预防策略**：
- 设计子 agent 的 instructions 时，明确规定返回格式
- 不要假设 response 包含 tool call 结果或中间状态——它只是最终文本

**深钻指针**：`modules/notion/threads/index.ts`（createAndRunThread 返回类型说明）

### 6.7.3 50 次上限

**典型症状**：在一个 parent thread 中多次调用 `createAndRunThread` 后，后续调用失败，提示 `max sub agents exceeded`。

**根因分析**：单个 parent thread 最多调用 `createAndRunThread` 50 次（无论是调同一个 agent 还是不同 agent）。这是硬性限制。

**修复方案**：

1. 合并调用：将多个小任务合并到一次 `createAndRunThread` 的 instructions 中
2. 复用 thread：通过 `threadUrl` 参数续写已有子 thread，而非每次创建新 thread
3. 分层架构：如果 50 次不够，设计多层 agent 架构——父 agent 调 coordinator，coordinator 再各调 50 次

**预防策略**：
- 在多 agent 工作流设计阶段就预估调用次数
- 并行写作模式中，谨慎控制分身数量
- 每次调用都有价值——不要用 `createAndRunThread` 做可以通过其他函数完成的简单操作

**深钻指针**：`modules/notion/threads/index.ts`（50 次上限说明）

---

## S6.8 MCP 类

**失败模式类别**：MCP Server 工具调用因参数、命名或状态问题失败。

### 6.8.1 工具不存在

**典型症状**：调用 `runTool({ toolName: "xxx" })` 时报工具不存在。

**根因分析**：
- 工具名称拼写错误
- MCP Server 的工具列表已更新，但 agent 使用了旧名称
- 混淆了不同 MCP 连接的工具（如用 GitHub MCP 的 key 调 Slack MCP 的工具）

**修复方案**：

1. 先调 `listTools()` 获取当前可用工具列表及其精确名称
2. 确认使用正确的 connection key（如 `connections.mcpServer_github.runTool`，而非 `connections.mcpServer.runTool`）

**预防策略**：
- 每次操作 MCP 前，先 `listTools()` 确认工具可用性
- 连接 key 在 `connections.ts` 中定义，对照确认

**深钻指针**：`modules/mcpServer/index.ts`（RunToolInput 类型）、`modules/mcpServer/AGENTS.md`（连接 key 与路径的区别）

### 6.8.2 参数格式错误

**典型症状**：`runTool` 调用成功但返回错误信息，提示参数缺失或类型不匹配。

**根因分析**：每个 MCP 工具有自己的 `inputSchema`，参数通过 `toolArguments` 传入。常见错误：
- 必填参数遗漏
- 参数类型错误（如 string 传了 number）
- 嵌套对象结构不正确

**修复方案**：

1. 从 `listTools()` 返回中读取目标工具的 `inputSchema`
2. 按 schema 构造 `toolArguments` 对象
3. 注意 `required` 字段列出的必填参数

```jsonc
// listTools 返回中查看 inputSchema
{
  "name": "create_or_update_file",
  "inputSchema": {
    "required": ["owner", "repo", "path", "content", "message", "branch"],
    "properties": {
      "owner": { "type": "string" },
      "sha": { "type": "string", "description": "Required if file exists" }
      // ...
    }
  }
}

// 按 schema 构造调用
await runTool({
  toolName: "create_or_update_file",
  toolArguments: {
    owner: "my-org",
    repo: "my-repo",
    path: "docs/guide.md",
    content: "# Guide",
    message: "add guide",
    branch: "main",
    sha: "abc123"  // 更新已有文件时必须提供
  }
})
```

**预防策略**：
- 更新已有文件必须提供 `sha`（从 `get_file_contents` 返回中获取）
- 不要猜测参数格式——始终先读 `inputSchema`

**深钻指针**：`modules/mcpServer/index.ts`（McpServerListTool 和 RunToolInput 类型）

### 6.8.3 连接状态异常

**典型症状**：MCP 工具调用超时或返回连接错误。

**根因分析**：MCP Server 是外部服务，可能因网络、认证过期、服务端故障等原因不可用。这不是 agent 配置问题，而是运行时环境问题。

**修复方案**：

1. 重试一次——可能是临时网络抖动
2. 如果持续失败，记录错误并通知用户（通过 `sendNotification`）
3. 在多 agent 工作流中，将 MCP 失败作为异常分支处理，不要让整个流程崩溃

**预防策略**：
- 设计工作流时，将 MCP 调用视为可能失败的外部依赖
- 关键操作的结果应及时持久化到 Notion 页面，避免 MCP 断连导致进度丢失

**深钻指针**：`modules/mcpServer/AGENTS.md`

---

## S6.9 Context 管理类

**失败模式类别**：context 窗口被低价值内容占满，导致关键信息被截断或遗忘。

### 6.9.1 Token 窗口溢出

**典型症状**：Agent 在长对话或多步操作后，开始「遗忘」早期指令或上下文，输出质量明显下降。

**根因分析**：AI Agent 的 context 窗口有 token 上限。当加载的内容（页面、搜索结果、MCP 返回）累计超过窗口容量时，早期内容会被截断。

**修复方案**：

1. 加载内容前评估必要性——不是所有相关内容都需要加载
2. 使用摘要替代全文（桥梁文件模式）
3. 将长流程拆分为多个 sub-thread，每个 thread 有独立的 context 窗口

**预防策略**：
- 铁律：不加载其他章节全文，只读 BOOK_SUMMARY 中的摘要
- 大页面只加载需要的部分——如果只需要属性信息，`loadPage` 后只关注 properties
- 搜索结果的 snippet 通常已足够，不需要对每个结果都 `loadPage`

### 6.9.2 加载过多无关 context

**典型症状**：Agent 执行速度变慢，或在回复中混入了与当前任务无关的信息。

**根因分析**：习惯性地 "加载一切" 会快速耗尽 context 窗口。常见过度加载：
- 操作单个页面前加载了整个数据库的所有页面
- 搜索后对每个结果都执行 `loadPage`
- 读取了与当前场景无关的 module files

**修复方案**：

1. 按需加载：先明确任务需要什么信息，再决定加载什么
2. 搜索结果的 snippet 和 properties 通常已经足够回答问题
3. SQL 查询（`querySql`）可以精确获取需要的列和行，避免全量加载

**预防策略**：
- 操作数据库页面？先 `querySql` 定位目标行，再对特定页面 `loadPage`
- 需要跨页面信息？设计桥梁文件（如 BOOK_SUMMARY）集中存储摘要
- module files 按需读取——`AGENTS.md` 中有 file routing 指引，遵循指引加载

### 6.9.3 BOOK_SUMMARY 过旧

**典型症状**：跨章引用的信息与实际章节内容不一致，术语或结论过时。

**根因分析**：BOOK_SUMMARY 是人工（总编）维护的摘要文件。如果总编未在章节验收后及时更新，后续章节的著作郎会基于过时摘要写作。

**修复方案**：

1. 写作前检查 BOOK_SUMMARY 中目标章节的 `Status` 字段——`⚪ Not started` 表示该章还没有可用摘要
2. 如果摘要信息与已知事实矛盾，在交稿摘要中向总编报告
3. 不要自行修改 BOOK_SUMMARY——这是总编的职责

**预防策略**：
- 总编协议：每章验收通过后，立即更新 BOOK_SUMMARY
- 著作郎协议：交稿时返回新增术语清单，方便总编更新 glossary 和 BOOK_SUMMARY

---

## S6.10 性能优化

**失败模式类别**：非最优的调用模式导致操作缓慢或触发限制。

### 6.10.1 批量操作 vs 逐条操作

**场景**：需要删除/归档多个页面，或查询多个数据源。

**低效模式**：
```jsonc
// ❌ 逐条删除
await deletePages({ pageUrls: ["user://332d872b-594c-816c-a9a9-000276e2496b"] })
await deletePages({ pageUrls: ["user://34cc9fa4-4e97-8194-b96b-0027cef5b078"] })
await deletePages({ pageUrls: ["https://www.notion.so/337c9fa44e978052a7fae5508bd47d1c"] })
```

**优化方案**：
```jsonc
// ✅ 批量删除
await deletePages({ pageUrls: ["user://332d872b-594c-816c-a9a9-000276e2496b", "user://34cc9fa4-4e97-8194-b96b-0027cef5b078", "https://www.notion.so/337c9fa44e978052a7fae5508bd47d1c"] })
```

**同理适用于**：
- `archivePages` / `unarchivePages`：支持批量 URL 数组
- `querySql`：`dataSourceUrls` 支持多个数据源，一次查询中 JOIN 多表
- `deleteDatabases`：支持批量 URL 数组

**规则**：凡是接受 URL 数组的函数，尽量一次调用传入所有目标。

### 6.10.2 并行调用策略

**场景**：需要加载多个独立资源或执行多个无依赖的操作。

**低效模式**：
```jsonc
// ❌ 串行加载无依赖的资源
const page1 = await loadPage({ url: "https://www.notion.so/88a395759257472b89f9d38ae3a1df2f" })
const page2 = await loadPage({ url: "https://www.notion.so/16367caadd7e43a9ac3ae6592a8c6370" })
const db = await loadDatabase({ url: "integration://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40/notion/86b06b1b-9c54-4a7f-ac10-fefcf3d177a9" })
```

**优化方案**：
```jsonc
// ✅ 并行加载——在同一个 tool call block 中发起所有无依赖调用
// （通过 callFunction 工具的并行调用能力）
```

**但永远不要并行**：
- 对同一个页面 URL 的多次 `updatePage`——必须串行，合并为一次调用
- 有依赖关系的操作（如先 createDatabase 再用返回的 URL createPage）

**规则**：
- 无依赖 → 并行
- 有依赖 → 串行
- 同目标写操作 → 合并为一次调用

### 6.10.3 避免重复 loadPage

**场景**：多次更新同一个页面。

**低效模式**：
```jsonc
// ❌ 每次更新前都重新加载
await loadPage({ url: "https://www.notion.so/edc97e89538a42358144f698122e1174" })
await updatePage({ url: "https://www.notion.so/edc97e89538a42358144f698122e1174", propertyUpdates: { Status: "In progress" } })
await loadPage({ url: "https://www.notion.so/edc97e89538a42358144f698122e1174" })  // 不必要的重复加载
await updatePage({ url: "https://www.notion.so/edc97e89538a42358144f698122e1174", contentUpdates: [{ oldStr: "...", newStr: "..." }] })
```

**优化方案**：
```jsonc
// ✅ 加载一次，合并多个更新到一次调用
await loadPage({ url: "https://www.notion.so/edc97e89538a42358144f698122e1174" })
await updatePage({
  url: "https://www.notion.so/edc97e89538a42358144f698122e1174",
  propertyUpdates: { Status: "In progress" },
  contentUpdates: [{ oldStr: "...", newStr: "..." }]
})
```

**规则**：
- 首次操作前 `loadPage` 一次
- 后续对同一页面的多次修改，合并到一次 `updatePage`
- 除非收到页面已过期的提示，否则不重复加载
- `updatePage` 的返回值包含更新后的页面内容，可作为后续操作的基础

---

## 章末：Module files 完整清单

本章涉及的全部 module files（按目录组织）：

| 目录 | 文件 | 涉及场景 |
|------|------|----------|
| `modules/notion/pages/` | `index.ts` | S6.4（contentUpdates）、S6.10（updatePage 合并） |
| `modules/notion/pages/` | `AGENTS.md` | S6.2（property formats）、S6.3（property naming）、S6.5（wiki） |
| `modules/notion/pages/` | `page-content-spec.md` | S6.4（database blocks） |
| `modules/notion/` | `notion-markdown.md` | S6.4（Markdown 语法） |
| `modules/notion/databases/` | `index.ts` | S6.3（CREATE-\* 系统）、S6.5（linked database） |
| `modules/notion/databases/` | `AGENTS.md` | S6.3（identifier 规则）、S6.5（wiki / linked db） |
| `modules/notion/databases/` | `data-source-sqlite-tables.md` | S6.2（列展开、checkbox、JSON 值） |
| `modules/notion/databases/` | `dataSourceTypes.ts` | S6.2（property 类型定义） |
| `modules/notion/agents/` | `index.ts` | S6.1（updateAgent 权限配置） |
| `modules/notion/` | `integration.ts` | S6.1（权限类型、interact 格式） |
| `modules/notion/threads/` | `index.ts` | S6.7（createAndRunThread 约束） |
| `modules/notion/` | `sharing.ts` | S6.1（sharing vs agent 权限区别） |
| `modules/search/` | `AGENTS.md` | S6.6（查询构造策略） |
| `modules/search/` | `index.ts` | S6.6（SearchQueryInput 类型） |
| `modules/mcpServer/` | `index.ts` | S6.8（MCP 工具调用接口） |
| `modules/mcpServer/` | `AGENTS.md` | S6.8（连接 key 与路径区别） |
| `modules/notion/` | `notifications.ts` | S6.8（异常通知） |
| `modules/system/` | `index.ts` | S6.9（wait / updateTodos） |
