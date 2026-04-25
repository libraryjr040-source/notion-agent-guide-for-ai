# Ch3 数据操作

**覆盖域**：数据库 + 查询 + 搜索 + 分析
**组织原则**：场景驱动
**涉及 modules**：`notion/databases`、`search`、`notion/databases/query.ts`、`notion/databases/data-source-sqlite-tables.md`、`notion/analytics`

---

## 薄概念 DRY 层

### 核心实体关系

```
Database
 ├── dataSources（owned，定义 schema）
 │     └── schema: Record<key, PropertySchema>
 ├── views（table / board / calendar / list / gallery / timeline / chart / form / dashboard / map）
 │     └── dataSourceUrl → 指向 owned 或 external DataSource
 └── layouts（控制页面内打开数据库条目时的布局）
```

- **Database** 是容器，拥有零或多个 DataSource 和一或多个 View。
- **DataSource** 是一张「表」，`schema` 定义列（property）。一个 DataSource 在 SQLite 查询中对应一张同名表。
- **View** 是 DataSource 的展示方式。一个 View 通过 `dataSourceUrl` 指向一个 DataSource——可以是本 Database 拥有的，也可以是外部 Database 的（linked database 场景）。
- **Property** 是 DataSource 的列定义。每个 property 有 type（决定值格式）和 name（显示名）。
- **CREATE-\* 占位符**：在同一个 `createDatabase` 调用中，新建的 DataSource、Property、View 还没有真实 URL，用 `CREATE-*` 作临时 key 互相引用。系统在创建完成后替换为 compressed URL。
- **compressed URL**：所有已存在实体的唯一标识，格式如 `"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`（page）、`"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`（data source）、`"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`（view）。前缀语义匹配实体类型（`page-`、`data-source-`、`database-`、`view-`、`property-`、`user-`、`agent-`、`teamspace-`、`integration-`、`file-`）。API 调用中引用已有实体必须用 compressed URL，不可用 name。

### 本章能力边界速查

| 能做 | 不能做 |
|------|--------|
| 创建/更新/删除 Database | 管理 Database Integrations / Automations |
| 设计 schema（所有 property 类型） | 创建 typed tasks database |
| 创建所有 view 类型（含 chart、form、dashboard、map） | 修改 form 的 conditional logic |
| SQL 查询（SQLite 方言） | 写入 read-only property（created_time 等） |
| 跨数据源 JOIN | 跨 workspace 查询 |
| 创建 linked database | — |
| 创建双向 relation | — |
| 全源搜索（Notion + connectors + web） | — |
| Workspace / Page 级分析 | — |

### Module files 导航

| 文件 | 职责 |
|------|------|
| `modules/notion/databases/index.ts` | 函数签名 + JSONDatabaseConfiguration 结构 |
| `modules/notion/databases/dataSourceTypes.ts` | Property 类型定义 + DataSource 结构 |
| `modules/notion/databases/viewTypes.ts` | 所有 View 类型 + Filter / GroupBy / Sort / Chart 类型 |
| `modules/notion/databases/layoutTypes.ts` | 页面布局 module 定义 |
| `modules/notion/databases/data-source-sqlite-tables.md` | SQL 查询的列名映射、值编码、示例 |
| `modules/notion/databases/formula-spec.md` | Formula 语言完整规范 |
| `modules/notion/databases/query.ts` | querySql / queryView / queryMeetings 类型 |
| `modules/search/index.ts` | search 函数签名 |
| `modules/search/AGENTS.md` | search query 构造指南 |
| `modules/notion/analytics/index.ts` | analytics 系列函数签名 |

---

## S3.1 创建数据库

**场景入口**：从零创建一个带 schema 和 view 的数据库。

### 适用条件

- 需要一个全新的结构化数据容器
- 已知目标 parent（page / user / teamspace）

### 操作步骤

1. **确定 parent**：`{ type: "page", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" }` / `{ type: "user", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" }` / `{ type: "teamspace", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" }`
2. **设计 DataSource**：为每个 DataSource 分配一个 `CREATE-*` key（如 `"CREATE-main"`），`url` 字段必须与 key 一致
3. **设计 schema**：每个 property 也用 `CREATE-*` key（如 `"CREATE-title"`），display name 放在 `name` 字段
4. **设计 View**：用 `CREATE-*` key（如 `"CREATE-table"`），通过 `dataSourceUrl` 引用第 2 步的 DataSource key
5. **调用 `connections.notion.createDatabase`**

### 关键参数

- `inline: true` → 数据库直接渲染在 parent page 内容中（追加到底部）
- `inline: false`（默认）→ 数据库作为独立全页
- `replacesBlankParentPage: true` → 将一个空白 page 转化为数据库页面

### 实战案例

```js
await connections.notion.createDatabase({
  name: "Project Tracker",
  parent: { type: "page", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" },
  dataSources: {
    "CREATE-main": {
      url: "CREATE-main",
      name: "Projects",
      schema: {
        "CREATE-title": { type: "title", name: "Project Name" },
        "CREATE-status": {
          type: "status", name: "Status",
          groups: {
            to_do: [{ name: "Not started" }],
            in_progress: [{ name: "In progress" }, { name: "Blocked" }],
            complete: [{ name: "Done" }]
          }
        },
        "CREATE-owner": { type: "person", name: "Owner", limit: 1 },
        "CREATE-due": { type: "date", name: "Due Date" },
        "CREATE-priority": {
          type: "select", name: "Priority",
          options: [
            { name: "P0", color: "red" },
            { name: "P1", color: "orange" },
            { name: "P2", color: "blue" }
          ]
        }
      }
    }
  },
  views: {
    "CREATE-table": {
      type: "table", name: "All Projects",
      dataSourceUrl: "CREATE-main"
    },
    "CREATE-board": {
      type: "board", name: "By Status",
      dataSourceUrl: "CREATE-main",
      groupBy: { property: "Status", propertyType: "status", groupBy: "option",
                 sort: { type: "manual" } }
    }
  }
})
```

### 踩坑清单

- ⚠️ **key 不是 name**：`schema` 的 key 必须是 `CREATE-*` 或 compressed URL，不能用 property 的 display name。写 `"Title": { type: "title", name: "Title" }` 会失败，必须写 `"CREATE-title": { type: "title", name: "Title" }`。
- ⚠️ **DataSource url 必须与 key 一致**：`dataSources` 的 key 是 `"CREATE-main"`，则该 DataSource 的 `url` 字段也必须是 `"CREATE-main"`。
- ⚠️ **View 的 dataSourceUrl 必须引用存在的 key**：可以是同一调用中的 `CREATE-*` key，也可以是已有的 compressed URL 如 `"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`。
- ⚠️ **parent 为 page 时**：数据库会追加到该 page 内容的底部。如需调整位置，后续用 page 编辑工具重排。

### 深钻指针

- 函数签名 → `modules/notion/databases/index.ts` 中 `CreateDatabase` 类型
- Property 类型完整列表 → `modules/notion/databases/dataSourceTypes.ts`
- View 类型完整列表 → `modules/notion/databases/viewTypes.ts`

---

## S3.2 Schema 设计

**场景入口**：为数据库选择合适的 property 类型，设计完整的 schema。

### 适用条件

- 新建数据库时设计初始 schema
- 更新已有数据库时添加/修改 property

### Property 类型选择决策树

| 需求 | 推荐类型 | 备注 |
|------|----------|------|
| 页面标题（必须有且仅一个） | `title` | 每个 DataSource 必须有一个 title property |
| 自由文本 | `text` | 支持 rich text |
| 单选标签 | `select` | 需预定义 options，完整结构 → `dataSourceTypes.ts` |
| 多选标签 | `multi_select` | 需预定义 options |
| 工作流状态 | `status` | 分 to_do / in_progress / complete 三组，完整结构 → `dataSourceTypes.ts` |
| 数值 | `number` | 可配 number_format 和 show_as |
| 日期/时间 | `date` | 可配 date_format、time_format |
| 人员 | `person` | `limit: 1` 限制单人 |
| 关联其他数据源 | `relation` | 单向用 relation，双向用 `createTwoWayRelation`（见 S3.8） |
| 勾选框 | `checkbox` | — |
| 文件附件 | `file` | — |
| URL / Email / Phone | `url` / `email` / `phone_number` | — |
| 公式计算 | `formula` | 需写 formula 表达式，详见 `formula-spec.md` |
| 汇总关联数据 | `rollup` | 需指定 relationPropertyUrl + targetPropertyUrl + aggregation，完整配置 → `dataSourceTypes.ts` |
| 自增 ID | `auto_increment_id` | 只读 |
| 创建/编辑时间 | `created_time` / `last_edited_time` | 只读 |
| 创建/编辑人 | `created_by` / `last_edited_by` | 只读 |
| 地理位置 | `place` | schema type 始终为 `place`（UI 可能显示为 "Location"）；详见下方 Place Property 小节 |
| 按钮 | `button` | — |
| 验证状态 | `verification` | — |

### Relation Property（单向）

```js
"CREATE-project": {
  type: "relation",
  name: "Project",
  dataSourceUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  limit: 1
}
```

单向 relation 只在本数据源上创建 property。如需双向，见 S3.8。

### Formula Property

formula 表达式写在 `formula` 字段中，`resultType` 标注计算结果类型（`"text"` / `"number"` / `"checkbox"` / `"date"` / `"person"` / `"select"`）。

```js
"CREATE-days-left": {
  type: "formula",
  name: "Days Left",
  formula: "dateBetween(prop(\"Due Date\"), now(), \"days\")",
  resultType: "number"
}
```

⚠️ formula 语言是 Notion 专有的，不是 JavaScript。完整规范见深钻指针。

### Rollup Property

```js
"CREATE-task-count": {
  type: "rollup",
  name: "Task Count",
  relationPropertyUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  targetPropertyUrl: "dataSourceUrl",
  aggregation: "count"
}
```

完整 aggregation 选项与配置 → `dataSourceTypes.ts`。

### Place Property（地理位置）

`place` 是 Notion 数据库中的地理位置属性类型。Notion UI 上可能将其显示为 "Location"，但 **schema 中的 type 始终为 `place`，不是 `location`**。

与其他属性不同，place 属性在读写时 **展开为 5 个独立的子列**，不能直接通过属性名设值：

| 展开列名 | SQL 类型 | 说明 |
|----------|----------|------|
| `place:<Name>:address` | `TEXT` | 地址字符串（最简写法只需此列） |
| `place:<Name>:name` | `TEXT` | 地点名称（可选） |
| `place:<Name>:latitude` | `FLOAT` | 纬度（可选，geocoding 可自动填充） |
| `place:<Name>:longitude` | `FLOAT` | 经度（可选，geocoding 可自动填充） |
| `place:<Name>:google_place_id` | `TEXT` | Google Place ID（可选） |

其中 `<Name>` 是 schema 中定义的 property name（如 `"Office Location"`）。

**创建 place property**：

```js
"CREATE-location": {
  type: "place",
  name: "Office Location"
}
```

**设置 place 属性值**（通过 `createPage` 或 `updatePage`）：

```js
// ✅ 正确：使用展开的 place: 子列
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  propertyUpdates: {
    "place:Office Location:address": "1600 Pennsylvania Ave NW, Washington, DC 20500",
    "place:Office Location:name": "White House"
  }
})

// ❌ 错误：直接用属性名设值
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  propertyUpdates: {
    "Office Location": "1600 Pennsylvania Ave NW"  // 会失败！
  }
})
```

**最简写法**——只设 `address`，系统自动 geocoding 填充经纬度：

```js
{
  "place:Office Location:address": "350 5th Ave, New York, NY 10118"
}
```

**查询 place 属性**（通过 `querySql`）：

```js
const result = await connections.notion.querySql({
  dataSourceUrls: ["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"],
  query: `
    SELECT url, "place:Office Location:address", "place:Office Location:name",
           "place:Office Location:latitude", "place:Office Location:longitude"
    FROM "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"
    WHERE "place:Office Location:address" IS NOT NULL
  `
})
```

⚠️ map view 的 `mapBy` 必须引用 place 类型 property 的 name（见 S3.3）。

### 踩坑清单

- ⚠️ **title 必须有且只有一个**：每个 DataSource 的 schema 中必须恰好有一个 `type: "title"` 的 property。
- ⚠️ **property key 是 CREATE-\* 或 compressed URL**：不要用 display name 作 key。
- ⚠️ **formula 中的引号需要转义**：`prop("Name")` 在 JSON 字符串中要写成 `prop(\"Name\")`。
- ⚠️ **rollup 的 relationPropertyUrl 必须是本数据源上已有的 relation property 的 compressed URL**，不是 CREATE-\* key（除非在同一个 createDatabase 调用中刚创建）。
- ⚠️ **number_format、show_as、date_format 等可选项**：除非用户明确要求，否则不要设置，使用默认值。
- ⚠️ **place property 的值设置**：不能直接设 `"Location": "..."`，必须使用展开的 `place:<Name>:address` 等子列。最简只需设 `address`，geocoding 自动填充经纬度。
- ⚠️ **place vs location**：schema type 始终写 `place`，不是 `location`。UI 上可能显示 "Location" 但 API 层面只认 `place`。

### 深钻指针

- 所有 property 类型完整定义 → `modules/notion/databases/dataSourceTypes.ts`
- Formula 语言完整规范 → `modules/notion/databases/formula-spec.md`
- Property 值在 SQL 中的表现 → `modules/notion/databases/data-source-sqlite-tables.md`

---

## S3.3 View 管理

**场景入口**：为数据库创建或配置不同的展示视图。

### 适用条件

- 新建数据库时一并创建 views
- 为已有数据库添加新 view（通过 `updateDatabase`）

### View 类型速查

| 类型 | `type` 值 | 核心配置 | 典型场景 |
|------|-----------|----------|----------|
| 表格 | `"table"` | displayProperties, sorts, groupBy | 通用数据浏览 |
| 看板 | `"board"` | groupBy（必要）, cardSize, cover | 工作流管理 |
| 日历 | `"calendar"` | calendarBy（date property 名） | 日期驱动的排期 |
| 列表 | `"list"` | displayProperties | 简洁列表 |
| 画廊 | `"gallery"` | cover, cardSize | 视觉内容展示 |
| 时间线 | `"timeline"` | timelineBy, timelineByEnd | 甘特图式排期 |
| 图表 | `"chart"` | chartConfig（见下） | 数据可视化 |
| 表单 | `"form_editor"` | questions | 数据收集 |
| 仪表板 | `"dashboard"` | rows → widgets → 嵌套 views | 多图表组合 |
| 地图 | `"map"` | mapBy（place property 名） | 地理位置展示 |

### 通用 View 配置

所有 view 类型（chart 和 dashboard 除外）都支持 `displayProperties`、`simpleFilters`、`advancedFilter`、`sorts`、`groupBy`。不同 property 类型的 groupBy 需要不同配置结构——完整对照表 → `viewTypes.ts`。

### Chart View 配置

Chart view 的核心是 `chartConfig`，有四种图表类型：

**柱状图 / 条形图 / 折线图**（type: `"column"` / `"bar"` / `"line"`）：

```js
"CREATE-chart": {
  type: "chart", name: "Tasks by Status",
  dataSourceUrl: "CREATE-main",
  chartConfig: {
    type: "column",
    dataConfig: {
      type: "groups_reducer",
      groupBy: { property: "Status", propertyType: "status",
                 groupBy: "option", sort: { type: "manual" } },
      aggregationConfig: {
        seriesFormat: { displayType: "column" },
        aggregation: { aggregator: "count" }
      }
    }
  }
}
```

**饼图**（type: `"donut"`，用户说 "pie chart" 时用 donut）：

```js
chartConfig: {
  type: "donut",
  dataConfig: {
    type: "groups_reducer",
    groupBy: { property: "Priority", propertyType: "select",
               sort: { type: "manual" } },
    aggregationConfig: {
      aggregation: { aggregator: "count" }
    }
  }
}
```

**数字图**（type: `"number"`）：

```js
chartConfig: {
  type: "number",
  dataConfig: {
    type: "number_reducer",
    aggregationConfig: {
      aggregation: { aggregator: "sum", property: "Points" }
    }
  }
}
```

### Form View（表单）

Form view 的 type 是 `"form_editor"`（不是 `"form"`）：

```js
"CREATE-form": {
  type: "form_editor", name: "Submit Bug",
  dataSourceUrl: "CREATE-main",
  title: "Bug Report Form",
  description: "Please fill out this form to report a bug.",
  questions: [
    { property: "Project Name", required: true },
    { property: "Priority", description: "How urgent is this?" },
    { property: "Description" }
  ]
}
```

⚠️ `questions` 中的 `property` 必须引用 DataSource schema 中已有 property 的 **name**。status 类型的 property 不能用在 form 中。

### Dashboard View（仪表板）

Dashboard 是 widget 容器，每行最多 4 个 widget，每个 widget 包含一个内联 view（不能是 dashboard 或 form_editor）：

```js
"CREATE-dashboard": {
  type: "dashboard", name: "Overview",
  rows: [
    {
      id: "row-1",
      widgets: [
        {
          id: "widget-1",
          view: {
            type: "chart", name: "By Status",
            dataSourceUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
            chartConfig: { /* ... */ }
          }
        },
        {
          id: "widget-2",
          view: {
            type: "chart", name: "By Priority",
            dataSourceUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
            chartConfig: { /* ... */ }
          }
        }
      ],
      height: 400
    }
  ]
}
```

### Filter 配置

advancedFilter 支持嵌套的 and/or 组合：

```js
advancedFilter: {
  type: "group", operator: "and",
  filters: [
    { type: "property", property: "Status", propertyType: "status",
      operator: "status_is_not",
      value: { type: "is_option", value: "Done" } },
    { type: "property", property: "Owner", propertyType: "person",
      operator: "person_contains",
      value: { type: "relative", value: "me" } }
  ]
}
```

⚠️ Filter 中 property 用 **name**（display name），不是 CREATE-\* key 或 compressed URL。

### 踩坑清单

- ⚠️ **用户说 "pie chart" → 用 `type: "donut"`**，Notion 没有 pie chart 类型。
- ⚠️ **form_editor 不支持 status property**：form 的 questions 中不能引用 status 类型的 property。
- ⚠️ **dashboard widget 内不能嵌套 dashboard 或 form_editor**。
- ⚠️ **formula property 在 chart aggregation 或 groupBy 中**：必须用 formula 的 `resultType` 作为 `propertyType`，而非 `"formula"`。
- ⚠️ **calendar view 的 calendarBy**：必须引用一个 date 类型 property 的 name。
- ⚠️ **timeline view 需要 timelineBy**：引用 date property name，`timelineByEnd` 可选（用另一个 date property 标注结束日期）。
- ⚠️ **map view 需要 mapBy**：必须引用一个 `place` 类型 property 的 name（不是 `location`）。DataSource 中必须有 place property。

### 深钻指针

- 所有 View 类型定义 + Filter / GroupBy / Sort / Chart 完整配置 → `modules/notion/databases/viewTypes.ts`
- Layout（页面内打开条目时的布局） → `modules/notion/databases/layoutTypes.ts`

---

## S3.4 SQL 查询

**场景入口**：使用 SQL 从数据源中查询结构化数据。

### 适用条件

- 需要自定义过滤、排序、聚合
- 需要跨数据源 JOIN
- 需要 `json_each` 解析 multi-select / relation / person 等 JSON 数组列

### 操作步骤

1. **确定目标数据源**：获取 DataSource 的 compressed URL（通过 `loadDatabase` 返回的 configuration）
2. **构造 SQL**：使用 SQLite 方言，表名 = DataSource compressed URL（双引号包裹），列名 = property display name（有特殊字符时双引号包裹）
3. **调用 `connections.notion.querySql`**

完整的列名映射规则（date / place 展开列、checkbox 特殊值、JSON 数组类型、系统列冲突前缀等） → `data-source-sqlite-tables.md`。

### 实战案例

**基础查询**：

```js
const result = await connections.notion.querySql({
  dataSourceUrls: ["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"],
  query: `
    SELECT url, "Project Name", "Status", "Owner"
    FROM "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"
    WHERE "Status" = ?
    ORDER BY "date:Due Date:start" ASC
    LIMIT 20
  `,
  params: ["In progress"]
})
```

**multi-select 过滤（json_each）**：

```js
const result = await connections.notion.querySql({
  dataSourceUrls: ["dataSourceUrl"],
  query: `
    SELECT url, "Task", "Tags"
    FROM "dataSourceUrl" t
    WHERE EXISTS (
      SELECT 1 FROM json_each(t."Tags") WHERE value = ?
    )
  `,
  params: ["Urgent"]
})
```

**跨数据源 JOIN（通过 relation 列）**：

```js
const result = await connections.notion.querySql({
  dataSourceUrls: ["okrs", "teams"],
  query: `
    SELECT t.url, t."Task Name", p."Project Name"
    FROM "okrs" t
    JOIN "teams" p
      ON p.url IN (SELECT value FROM json_each(t."Project"))
    WHERE t."Status" != 'Done'
  `
})
```

**checkbox 过滤**：

```js
const result = await connections.notion.querySql({
  dataSourceUrls: ["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"],
  query: `
    SELECT url, "Task"
    FROM "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"
    WHERE "Is Blocked" = '__YES__'
  `
})
```

**聚合查询**：

```js
const result = await connections.notion.querySql({
  dataSourceUrls: ["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"],
  query: `
    SELECT "Status", COUNT(*) as cnt
    FROM "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"
    GROUP BY "Status"
  `
})
```

**place 属性查询**：

place 属性展开为 5 列，查询时必须使用展开列名（`place:<Name>:address` 等），不能直接用属性名：

```js
const result = await connections.notion.querySql({
  dataSourceUrls: ["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"],
  query: `
    SELECT url, "place:Office:address", "place:Office:latitude", "place:Office:longitude"
    FROM "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"
    WHERE "place:Office:address" LIKE '%New York%'
  `
})
```

### 踩坑清单

- ⚠️ **表名必须双引号包裹**：`FROM "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"` ✅，`FROM agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40` ❌。
- ⚠️ **date 展开列名**：date 属性展开为 3 列——`"date:<Name>:start"`、`"date:<Name>:end"`、`"date:<Name>:is_datetime"`。不能直接用属性名查询。
- ⚠️ **place 展开列名**：place 属性展开为 5 列——`"place:<Name>:address"`、`"place:<Name>:name"`、`"place:<Name>:latitude"`、`"place:<Name>:longitude"`、`"place:<Name>:google_place_id"`。不能直接用属性名查询或设值。
- ⚠️ **checkbox 特殊值**：checkbox 用 `"__YES__"` / `"__NO__"` 而非 `true` / `false`。
- ⚠️ **multi-select / relation / person 是 JSON 字符串**：不能直接 `= 'value'`，要用 `json_each` 解包。
- ⚠️ **与系统列名冲突的 property**：用 `"userDefined:"` 前缀区分，详见 `data-source-sqlite-tables.md`。
- ⚠️ **返回结果**：`querySql` 返回 `{ rows, hasMore }`。如果 `hasMore` 为 true，需加 `LIMIT` + `OFFSET` 分页。

### 深钻指针

- SQL 列名映射完整规范（所有 property 类型 → SQL 类型 + 值编码） → `modules/notion/databases/data-source-sqlite-tables.md`
- querySql 函数签名 → `modules/notion/databases/query.ts`

---

## S3.5 View 查询

**场景入口**：获取某个 view 当前展示的数据（含该 view 的 filter / sort / groupBy）。

### 适用条件

- 需要「view 里看到什么就查什么」，不需要自定义 SQL
- 快速获取已有 view 的数据快照

### 操作步骤

```js
const result = await connections.notion.queryView({
  viewUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"
})
// result.rows — 按 view 的 filter/sort 返回的行
// result.hasMore — 是否还有更多数据
```

### queryView vs querySql 选择策略

| 场景 | 选择 |
|------|------|
| 「这个 view 里有什么数据？」 | queryView |
| 需要自定义 WHERE / JOIN / GROUP BY | querySql |
| 需要跨数据源关联 | querySql |
| 需要聚合统计 | querySql |
| 只需要当前视图的数据快照 | queryView |
| 需要精确控制返回列 | querySql |

### 踩坑清单

- ⚠️ **queryView 返回的是 view 的当前状态**：包含该 view 的 filter、sort、groupBy 效果。如果 view 有 filter 隐藏了某些行，queryView 也不会返回那些行。
- ⚠️ **返回格式与 querySql 相同**：`{ rows, hasMore }`，rows 是 `Array<Record<string, value>>`。

### 深钻指针

- queryView 函数签名 → `modules/notion/databases/query.ts`

---

## S3.6 数据库更新

**场景入口**：修改已有数据库的 schema、views、name 等配置。

### 适用条件

- 需要给已有数据库添加/删除/修改 property
- 需要添加/修改 view
- 需要修改数据库名称或其他元数据

### 操作步骤

1. **加载数据库**：`connections.notion.loadDatabase({ url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" })` → 获取当前 configuration
2. **构造 edits 数组**：每个 edit 是一个 `EditJSONToolEdit` 对象
3. **调用 `connections.notion.updateDatabase`**

### Edit 命令类型

**`set`** — 设置路径上的值（新增或覆盖）：

```js
{ command: "set", path: ["dataSources", "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "schema", "CREATE-tags"],
  value: { type: "multi_select", name: "Tags",
           options: [{ name: "Frontend" }, { name: "Backend" }] } }
```

**`delete`** — 删除路径上的值：

```js
{ command: "delete", path: ["views", "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"] }
```

**`replaceString`** — 在路径上的字符串值中做子串替换：

```js
{ command: "replaceString", path: ["name"],
  replaceStr: "Old Name", newStr: "New Name", numOccurrences: 1 }
```

### 实战案例

**添加新 property + 新 view**：

```js
await connections.notion.updateDatabase({
  databaseUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  edits: [
    // 添加 property
    { command: "set",
      path: ["dataSources", "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "schema", "CREATE-points"],
      value: { type: "number", name: "Points", number_format: "number" } },
    // 添加 view
    { command: "set",
      path: ["views", "CREATE-timeline"],
      value: { type: "timeline", name: "Timeline",
               dataSourceUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
               timelineBy: "Start Date", timelineByEnd: "End Date" } }
  ]
})
```

**修改已有 property 的 options**：

```js
await connections.notion.updateDatabase({
  databaseUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  edits: [
    { command: "set",
      path: ["dataSources", "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "schema", "okrs", "options"],
      value: [
        { name: "P0", color: "red" },
        { name: "P1", color: "orange" },
        { name: "P2", color: "blue" },
        { name: "P3", color: "gray" }
      ] }
  ]
})
```

**删除 view**：

```js
await connections.notion.updateDatabase({
  databaseUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  edits: [
    { command: "delete", path: ["views", "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"] }
  ]
})
```

### 踩坑清单

- ⚠️ **path 中引用已有实体用 compressed URL**：如 `["dataSources", "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "schema", "okrs"]`，不要用 display name。
- ⚠️ **path 中新建实体用 CREATE-\***：如 `["dataSources", "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "schema", "CREATE-new-prop"]`。
- ⚠️ **edits 操作的是 JSONDatabaseConfiguration 结构**：path 从 configuration 的根开始，如 `["name"]`、`["dataSources", "...", "schema", "..."]`、`["views", "..."]`。
- ⚠️ **先 loadDatabase 再 updateDatabase**：需要知道已有实体的 compressed URL 才能正确引用。

### 深钻指针

- EditJSONToolEdit 类型定义 → `modules/notion/databases/index.ts`
- JSONDatabaseConfiguration 结构 → `modules/notion/databases/index.ts`

---

## S3.7 链接数据库

**场景入口**：创建一个引用其他数据库数据源的 linked database（linked view）。

### 适用条件

- 需要在另一个 page 上展示某个已有数据源的数据
- 需要为同一数据创建不同筛选/分组的入口

### 操作步骤

1. **获取源数据源 URL**：通过 `loadDatabase` 获取源数据库的 DataSource compressed URL
2. **创建 linked database**：`dataSources: {}` + views 引用外部 `dataSourceUrl`

### 实战案例

```js
// 假设已通过 loadDatabase 获得源数据源 URL: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"
await connections.notion.createDatabase({
  name: "My Tasks",
  parent: { type: "page", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" },
  inline: true,
  dataSources: {},  // 空！linked database 不拥有数据源
  views: {
    "CREATE-filtered-view": {
      type: "table", name: "My Open Tasks",
      dataSourceUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",  // 引用外部数据源
      advancedFilter: {
        type: "group", operator: "and",
        filters: [
          { type: "property", property: "Assignee", propertyType: "person",
            operator: "person_contains",
            value: { type: "relative", value: "me" } },
          { type: "property", property: "Status", propertyType: "status",
            operator: "status_is_not",
            value: { type: "is_option", value: "Done" } }
        ]
      }
    }
  }
})
```

### 踩坑清单

- ⚠️ **dataSources 必须为空对象 `{}`**：linked database 不拥有自己的数据源。
- ⚠️ **dataSourceUrl 必须是外部数据源的 compressed URL**：不是 database URL，是 data-source URL。通过 `loadDatabase` 的返回值在 `configuration.dataSources` 中找到。
- ⚠️ **`loadDatabase` 只返回 owned data sources**：外部数据源 URL 只出现在 view 的 `dataSourceUrl` 字段中，不出现在 `dataSources` 中。

### 深钻指针

- Linked database 说明 → `modules/notion/databases/AGENTS.md` 的 "Linked databases" 章节

---

## S3.8 双向关联

**场景入口**：在两个数据源之间创建双向 relation（两边都能看到对方）。

### 适用条件

- 需要 A 数据源和 B 数据源互相关联
- 单向 relation 不够——需要从两边都能导航

### 操作步骤

```js
const result = await connections.notion.createTwoWayRelation({
  sourceDataSourceUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  targetDataSourceUrl: "dataSourceUrl",
  sourcePropertyName: "Related Projects",
  targetPropertyName: "Related Tasks",
  sourcePropertyDescription: "Projects linked to this task",
  targetPropertyDescription: "Tasks linked to this project",
  limit: 1  // 可选，限制为单关联
})
// result.sourcePropertyUrl — 源数据源上新建的 relation property URL
// result.targetPropertyUrl — 目标数据源上新建的 relation property URL
```

### 踩坑清单

- ⚠️ **每次调用都会创建新的 relation property 对**：即使两个数据源之间已有 relation，再调一次会创建另一对。不会复用已有的。
- ⚠️ **不要用 schema 中的 relation type 手动创建双向 relation**：手动在 schema 中写 `type: "relation"` 只能创建单向 relation。双向必须用 `createTwoWayRelation`。
- ⚠️ **参数是 DataSource URL，不是 Database URL**：传 `"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`，不是 `"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`。

### 深钻指针

- createTwoWayRelation 函数签名 → `modules/notion/databases/index.ts`
- 双向 relation 说明 → `modules/notion/databases/AGENTS.md` 的 "Quick reference" 章节

---

## S3.9 搜索

**场景入口**：跨 Notion workspace、connected sources（Slack / Google Drive / GitHub / Jira 等）和 web 进行语义搜索。

### 适用条件

- 需要查找 workspace 中的页面、数据库、内容
- 需要搜索连接器来源（Slack 消息、Google Drive 文件等）
- 需要补充 web 搜索结果

### 操作步骤

```js
const result = await connections.search.search({
  queries: [
    {
      question: "Q3 roadmap changes",
      keywords: "Q3 roadmap",
      lookback: "30d"
    }
  ],
  includeWebResults: false  // 仅搜内部源
})
```

### Query 构造要点

| 参数 | 说明 | 注意事项 |
|------|------|----------|
| `question` | 一个聚焦的自然语言问题 | 保持接近用户原话，不要填充冗余修饰 |
| `keywords` | 2-4 个最具区分度的关键词 | 提取核心实体、缩写、专有名词 |
| `lookback` | 时间窗口 | `"default"` / `"all_time"` / `"7d"` / `"2w"` / `"3m"` / `"1y"` / ISO 日期如 `"2024-04-01"` |
| `includeNotionHelpdocs` | 是否包含 Notion 帮助文档 | 仅当用户问 Notion 产品用法时设为 true |

- **`includeWebResults`**（顶层参数）：默认 true。设为 false 则不搜 web。
- **多查询**：`queries` 数组可以包含多个不同的 question，适合复杂请求拆成多个子问题。
- **lookback 不要用自然语言**：`"last month"` ❌ → `"30d"` ✅。

### 搜索结果与 Citations

搜索结果中每个 result 包含 `id`、`type`、`title`、`snippet`、`url` 等字段。在 chat 回复中使用搜索结果的信息时，必须用 inline citation 引用对应结果的 URL：

```
根据内部文档，Q3 目标已更新。[^dataSourceUrl]
```

citation 只能引用结果中的 `url` 字段（compressed URL 或 web URL），不能引用搜索工具调用本身的 URL（如 `tool-xxx`）。

### 踩坑清单

- ⚠️ **lookback 必须用规范格式**：`"7d"` / `"2w"` / `"3m"` / `"1y"` / `"all_time"` / ISO 日期。不接受 `"last week"` 等自然语言。
- ⚠️ **keywords 要精炼**：2-4 个关键词，不要把整个 question 复制进来。
- ⚠️ **"search Notion" = 搜 workspace 内容**：不是搜 Notion 帮助文档。搜帮助文档要设 `includeNotionHelpdocs: true`。
- ⚠️ **citation 引用 result URL，不引用工具调用 URL**。
- ⚠️ **对于模糊的时间表述**："recent" / "latest" → 先用 `"1w"`，无结果再扩到 `"1m"` → `"all_time"`。

### 深钻指针

- search 函数签名 → `modules/search/index.ts`
- query 构造完整指南 → `modules/search/AGENTS.md`

---

## S3.10 Autofill Agents

**场景入口**：使用附着在数据库上的 autofill agent 批量填充行/属性。

### 适用条件

- 数据库上已配置 autofill custom agent
- 需要 AI 自动填充页面属性

### 关键约束

- **每次最多处理 20 行**：autofill agent 一次触发最多处理 20 行数据。
- **Autofill agent 通过 UI 创建**：不是通过 API 创建的。`loadDatabase` 返回的 `autofillCustomAgents` 字段列出已有的 autofill agent。
- **fillablePropertyNames**：autofill trigger 变量中包含的可填充属性名列表，定义了该 agent 可以操作哪些列。

### 判断何时用 autofill agent

| 场景 | 选择 |
|------|------|
| 需要 AI 根据页面内容自动填充标签/摘要 | Autofill agent |
| 需要批量更新已有数据（固定逻辑） | querySql + updatePage 循环 |
| 需要复杂的跨数据库逻辑 | Custom agent + 手动编排 |

### 踩坑清单

- ⚠️ **20 行限制是硬性的**：超过 20 行需要分批触发。
- ⚠️ **Autofill agent 不能通过 API 创建**：只能通过 Notion UI 在数据库设置中创建。API 只能查看已有的 autofill agent。
- ⚠️ **`autofillCustomAgents` 可能为 undefined**：如果数据库没有配置 autofill agent，该字段不存在。

### 深钻指针

- autofillCustomAgents 字段 → `modules/notion/databases/index.ts` 中 `DatabaseResult` 类型
- Autofill agent 创建流程 → Notion UI（非 API 操作），或参见 `modules/notion/agents/AGENTS.md`

---

## S3.11 Wiki 数据库

**场景入口**：在 wiki 数据库中创建页面。

### 适用条件

- `loadDatabase` 返回 `isWiki: true`
- 需要在该 wiki 中新增页面

### 操作步骤

1. **识别 wiki**：`loadDatabase` 返回中检查 `isWiki` 和 `wikiPageUrl`
2. **创建页面时使用 page parent**：

```js
// ✅ 正确：用 wikiPageUrl 作为 page parent
await connections.notion.createPage({
  parent: { type: "page", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" },  // wikiPageUrl 的值
  properties: { title: "New Wiki Article" },
  content: "Article content here."
})

// ❌ 错误：不要用 dataSource parent
await connections.notion.createPage({
  parent: { type: "dataSource", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" },  // 这会失败！
  properties: { title: "New Wiki Article" }
})
```

### 踩坑清单

- ⚠️ **wiki 页面的 parent 必须是 `{ type: "page", url: wikiPageUrl }`**：不能用 `{ type: "dataSource" }`。这是 wiki 数据库与普通数据库的核心区别。
- ⚠️ **`wikiPageUrl` 是 loadDatabase 返回的特殊字段**：它是 collection view page 的 URL，不是 DataSource URL。
- ⚠️ **识别 wiki 的唯一方式**：检查 `loadDatabase` 返回中的 `isWiki: true`。不能从数据库名称或其他特征判断。

### 深钻指针

- Wiki database 说明 → `modules/notion/databases/AGENTS.md` 的 "Wiki databases" 章节
- 创建页面 → `modules/notion/pages/AGENTS.md`

---

## S3.12 分析与统计

**场景入口**：获取 workspace 或 page 级别的使用分析数据。

### 适用条件

- 需要了解 workspace 的活跃度、内容使用情况
- 需要特定页面的访问统计

### 函数一览

| 函数 | 用途 | 关键参数 |
|------|------|----------|
| `getUserEngagementAnalytics` | workspace 用户活跃度概览 | `daysFilter` |
| `getContentEngagementAnalytics` | workspace 内容使用概览 | `daysFilter` |
| `getDailyUsersAnalytics` | 每日活跃成员趋势图 | `activeWindow`, `days?` |
| `listUsersAnalytics` | workspace 成员使用详情列表 | `timeRange`, `sort?` |
| `listContentAnalytics` | workspace 内容使用详情列表 | `timeRange`, `sort`, `filters?` |
| `getPageAnalyticsTimeSeries` | 单页面访问趋势 | `pageUrl`, `days?` |
| `getPageVisitors` | 单页面访问者列表 | `pageUrl`, `limit`, `sinceTimestampMs?` |

### daysFilter 可选值

`"last_7_days"` / `"last_28_days"` / `"last_90_days"` / `"last_365_days"` / `"all_time"`

### 实战案例

**workspace 用户活跃概览**：

```js
const engagement = await connections.notion.getUserEngagementAnalytics({
  daysFilter: "last_28_days"
})
// engagement.activeMembersCount — 活跃成员数
// engagement.topViewers — 阅读量 top 用户
// engagement.topEditors — 编辑量 top 用户
// engagement.topTeamspaces — 活跃 teamspace
```

**单页面访问趋势**：

```js
const timeSeries = await connections.notion.getPageAnalyticsTimeSeries({
  pageUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  days: 30
})
// timeSeries.points — Array<{ ds, totalViews, uniqueViews }>
```

**筛选特定 teamspace 的内容分析**：

```js
const content = await connections.notion.listContentAnalytics({
  timeRange: "last_28_days",
  sort: { field: "pageViews", direction: "desc" },
  filters: {
    inTeamspaces: ["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"]
  }
})
```

**查看页面访问者**：

```js
const visitors = await connections.notion.getPageVisitors({
  pageUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  limit: 50,
  includeTotalCount: true
})
// visitors.visitors — Array<{ userUrl, visitedAtMs, onTrustedDomain? }>
// visitors.totalCount — 总访客数
```

### 踩坑清单

- ⚠️ **daysFilter 和 timeRange 的值是固定枚举**：只能用 `"last_7_days"` 等，不能用 `"7d"` 或自定义天数（这是 search 的 lookback 格式，不是 analytics 的）。
- ⚠️ **getDailyUsersAnalytics 的 activeWindow**：只有三个选项 `"last_7_days"` / `"last_28_days"` / `"last_90_days"`，没有 `"all_time"`。
- ⚠️ **listContentAnalytics 的 sort 是必填的**：不像其他函数可省略 sort。
- ⚠️ **返回的 URL 是 compressed URL**：`userUrl`、`pageUrl`、`teamspaceUrl` 都是 compressed URL 格式，可直接用于后续 API 调用。

### 深钻指针

- 所有 analytics 函数签名 + 类型定义 → `modules/notion/analytics/index.ts`

---

## 本章 Module Files 索引

| 文件路径 | 职责 | 本章涉及场景 |
|----------|------|-------------|
| `modules/notion/databases/index.ts` | 数据库函数签名、JSONDatabaseConfiguration、EditJSONToolEdit | S3.1, S3.6, S3.7, S3.8, S3.10 |
| `modules/notion/databases/dataSourceTypes.ts` | Property 类型定义、DataSource 结构 | S3.1, S3.2 |
| `modules/notion/databases/viewTypes.ts` | View 类型、Filter、GroupBy、Sort、Chart 配置 | S3.3 |
| `modules/notion/databases/layoutTypes.ts` | 页面布局 module 定义 | S3.3 |
| `modules/notion/databases/data-source-sqlite-tables.md` | SQL 列名映射、值编码、查询示例 | S3.4, S3.5 |
| `modules/notion/databases/formula-spec.md` | Notion Formula 语言完整规范 | S3.2 |
| `modules/notion/databases/query.ts` | querySql / queryView / queryMeetings 类型 | S3.4, S3.5 |
| `modules/search/index.ts` | search 函数签名 | S3.9 |
| `modules/search/AGENTS.md` | search query 构造指南 + citation 规范 | S3.9 |
| `modules/notion/analytics/index.ts` | analytics 系列函数签名 + 类型 | S3.12 |
| `modules/notion/databases/AGENTS.md` | Database 模块总览、linked database、wiki | S3.7, S3.8, S3.11 |
| `modules/notion/pages/AGENTS.md` | 页面操作指南（wiki 创建页面） | S3.11 |
