# Ch1 概念索引

**覆盖域**：全局概念关系图 + 能力边界 + Module files 导航  
**组织原则**：概念关系  
**目标**：加载本章后，能快速定位任何概念的含义、归属 module、与其他概念的关系，并知道去哪里深钻。

---

## 薄概念 DRY 层

Notion Agent 的运行时由三条主线交织：

1. **结构线**：Workspace → Teamspace → Page / Database → DataSource → View / Property  
2. **行动线**：Agent → Instructions → Integration → Permission → Trigger  
3. **通信线**：Thread → Sub-thread → createAndRunThread → response  

所有实体通过 **compressed URL** 标识——形如 `"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`、`"dataSourceUrl"`、`"okrs"`。这是 API 调用中引用实体的唯一标准方式；系统在处理时自动解压为真实 URL。永远不要用裸字符串（如 `"abc123"`）替代 compressed URL。

**CREATE-\* 占位符** 仅在同一次创建调用中使用，用于在新实体互相引用时充当临时 key（如新 View 引用尚未创建的 DataSource）。创建完成后系统替换为真实 compressed URL。

### 核心概念速览

| 概念 | 归属 | 一句话 | 深钻 |
|------|------|--------|------|
| Workspace | 顶层 | 协作空间，容纳 Pages、Databases、Agents、Users | `modules/notion/AGENTS.md` |
| Teamspace | Workspace 下 | 团队级分组，可作为 Page/Database 的 parent | `modules/notion/teamspaces/AGENTS.md` |
| Page | 结构线 | 内容载体，有 parent、properties、content | `modules/notion/pages/AGENTS.md` |
| Database | 结构线 | DataSource + View 的容器 | `modules/notion/databases/AGENTS.md` |
| DataSource | Database 下 | 拥有 schema 的数据表，映射为 SQLite table | `modules/notion/databases/dataSourceTypes.ts` |
| View | Database 下 | DataSource 的可视化呈现（table / board / chart…） | `modules/notion/databases/viewTypes.ts` |
| Property | DataSource 下 | schema 中的一列，定义数据类型和约束 | `modules/notion/databases/dataSourceTypes.ts` |
| Agent | 行动线 | AI 行动者，拥有 instructions + integrations + triggers | `modules/notion/agents/index.ts` |
| Integration | Agent 下 | Agent 与某 module 的连接，控制可用工具和数据 | `modules/notion/integration.ts` |
| Permission | Integration 下 | 访问控制项，定义对页面/数据库的读写权限 | `modules/notion/integration.ts` |
| Trigger | Agent 下 | 自动触发条件（定时/事件/按钮/@提及） | `modules/notion/triggers.ts` |
| Thread | 通信线 | Agent 对话会话 | `modules/notion/threads/index.ts` |
| Sub-thread | Thread 下 | 通过 createAndRunThread 开的子线程 | `modules/notion/threads/index.ts` |
| compressed URL | 标识 | 系统分配的实体标识符，格式 `"prefix-id"` | `modules/notion/AGENTS.md` |
| CREATE-\* 占位符 | 标识 | 同一创建调用中的临时 key | `modules/notion/databases/AGENTS.md` |

---

## S1.1 全局概念地图

**适用条件**：需要理解 Workspace 到 Property 的层级关系时加载本节。

### 核心层级

```
Workspace
├── Teamspace（团队分组）
├── Page（内容页面）
│   ├── Sub-page（子页面）
│   └── Database（数据库，inline 或 full-page）
│       ├── DataSource（数据源，拥有 schema）
│       │   ├── Property（属性列）
│       │   ├── Page（数据行，即数据库中的页面）
│       │   └── Template（模板页面）
│       └── View（视图）
│           ├── table / board / calendar / list / gallery
│           ├── timeline / chart / map / dashboard
│           └── form_editor（表单）
├── Agent（AI 行动者）
└── User（用户）
```

### Page 的 parent 类型

Page 是最基础的内容单元。它的 parent 决定了它在结构中的位置：

| parent.type | 含义 | 典型场景 |
|-------------|------|----------|
| `"user"` | 用户私有顶层页面 | `{ type: "user", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" }` |
| `"page"` | 嵌套在另一个页面下 | `{ type: "page", url: "dataSourceUrl" }` |
| `"dataSource"` | 数据库中的一行 | `{ type: "dataSource", url: "okrs" }` |
| `"teamspace"` | 团队空间顶层 | `{ type: "teamspace", url: "teams" }` |
| `"agent"` | Agent 的附属页面（如 instructions page） | 只读出现，不可用于创建/移动 |

### Database 与 DataSource 的关系

- 一个 Database 可以拥有 **零或多个 DataSource**。
- 拥有 DataSource 的叫 **owned database**；DataSource 的 schema 定义了属性列。
- 零个 owned DataSource 的叫 **linked database**：它的 View 通过 `dataSourceUrl` 引用其他 Database 的 DataSource。
- `loadDatabase` 只返回 owned DataSource；外部 DataSource 的 URL 出现在 View 的 `dataSourceUrl` 字段中。

### Wiki Database 特例

当 `loadDatabase` 返回 `isWiki: true` 时，创建页面必须用 `{ type: "page", url: wikiPageUrl }` 作 parent，**不能**用 `{ type: "dataSource" }`。`wikiPageUrl` 是 database 返回结果中的专属字段。

### View 的 10 种类型

`table` · `board` · `calendar` · `list` · `gallery` · `timeline` · `chart` · `dashboard` · `form_editor` · `map`

每种 View 都通过 `dataSourceUrl` 绑定到一个 DataSource（可以是 owned 的也可以是外部的）。Dashboard 特殊——它不直接绑定 DataSource，而是包含一组 widget，每个 widget 内联一个 View。

### 关键注意点

- ⚠️ Page parent 为 `"agent"` 时只在 `loadPage` 结果中出现，**不能**用于 `createPage` 或 `updatePage` 的 parent 参数。
- ⚠️ Database 名称不是标识符。所有引用必须用 compressed URL（如 `"URL"`），不是显示名称。
- ⚠️ DataSource schema 的 key 是 `CREATE-*` 或 compressed URL，不是 property name。Property name 放在 `{ name: "Title" }` 中。

### 深钻指针

| 主题 | 文件 |
|------|------|
| Page 完整类型与函数 | `modules/notion/pages/index.ts` |
| Database 配置与函数 | `modules/notion/databases/index.ts` |
| DataSource schema 设计 | `modules/notion/databases/dataSourceTypes.ts` |
| View 类型定义 | `modules/notion/databases/viewTypes.ts` |
| Teamspace 操作 | `modules/notion/teamspaces/AGENTS.md` |
| Workspace 级概念 | `modules/notion/AGENTS.md` |

---

## S1.2 Agent 概念族

**适用条件**：需要理解 Agent 的构成、权限模型、触发机制时加载本节。

### Agent 的实体结构

```
Agent
├── agentUrl              ← compressed URL，唯一标识
├── instructionsPageUrl    ← 一个 Notion Page，定义行为和技能
└── configuration
    ├── name / description / icon
    ├── integrations      ← Record<integrationUrl, Integration>
    │   └── Integration
    │       ├── type       ← module 类型（"notion" / "slack" / "mcpServer"…）
    │       ├── name
    │       ├── permissions ← 访问控制列表
    │       └── state      ← module 特定状态
    └── triggers          ← Record<triggerUrl, Trigger>
        └── Trigger
            ├── enabled
            ├── state      ← trigger 配置（类型 + 参数）
            └── integrationUrl ← 关联的 integration（外部触发器需要）
```

### Integration → Permission 链路

每个 Integration 的 `permissions` 数组控制 agent 对资源的访问。以 Notion integration 为例：

| identifier 形式 | 含义 |
|-----------------|------|
| `"agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"` 或 `"dataSourceUrl"` | 对特定页面/数据库的权限 |
| `["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "okrs"]` | 对多个资源的权限 |
| `"*"` | workspace 级全局权限 |
| `{ type: "agent", url: "teams" }` | 与另一个 agent 的 interact 权限 |
| `"webSearch"` | 网络搜索权限 |
| `"helpdocsSearch"` | Notion 帮助文档搜索权限 |

actions 取值：
- 内容权限：`"edit"` / `"comment"` / `"view"`
- Agent 间通信：`"interact"` / `"disallow"`
- 搜索权限：`"allow"` / `"disallow"`

### Trigger 类型清单

| trigger state.type | 触发条件 |
|--------------------|----------|
| `"recurrence"` | 定时触发（cron 式） |
| `"notion.page.created"` | 数据库中新增页面 |
| `"notion.page.updated"` | 数据库中页面更新 |
| `"notion.page.deleted"` | 数据库中页面删除 |
| `"notion.page.discussion.comment.added"` | 页面新增评论 |
| `"notion.button.pressed"` | 按钮被按下 |
| `"notion.agent.mentioned"` | Agent 被 @提及 |
| `"notion.database.agent.updated"` | 数据库 agent 配置更新 |

### Instructions Page

- 本质是一个普通 Notion Page，通过 `instructionsPageUrl` 关联。
- 修改 instructions 用 `updatePage`（操作 instructions page），**不是** `updateAgent`。
- `updateAgent` 的 edits 操作的是 `AgentConfiguration`，不包含 instructions 内容。

### 关键注意点

- ⚠️ interact 权限是双向独立的：A 能调 B 不代表 B 能调 A。每个方向需要在各自的 Notion integration permissions 中声明。
- ⚠️ `updateAgent` 的 edit path 相对于 `AgentConfiguration`，不要在 path 中写 `"configuration"`、`"agentUrl"` 等外层字段。
- ⚠️ MCP integration 的权限独立于 Notion integration。给 agent 加 MCP 工具需要单独添加 `type: "mcpServer"` 的 integration。

### 深钻指针

| 主题 | 文件 |
|------|------|
| Agent 类型与 CRUD | `modules/notion/agents/index.ts` |
| Notion 权限类型 | `modules/notion/integration.ts` |
| Trigger 配置与变量 | `modules/notion/triggers.ts` |
| MCP Server 工具 | `modules/mcpServer/index.ts` |

---

## S1.3 Thread 概念族

**适用条件**：需要理解 Agent 间通信、子线程生命周期时加载本节。

### 生命周期

```
Parent Thread（用户发起或 trigger 触发）
│
├── createAndRunThread({ agentUrl, instructions })
│   → 同步等待 → { threadUrl, response }
│   └── Sub-thread（子线程）
│       ├── 子 agent 正常运行，可使用自己的全部工具
│       ├── 可创建/修改资源（页面、数据库等）
│       └── 返回的 response 只是最终文本回复
│
├── createAndRunThread({ threadUrl, instructions })
│   → 续写已有子线程（传入 threadUrl）
│
└── 最多 50 次 createAndRunThread 调用
    → 超出报错："max sub agents exceeded"
```

### 三个核心函数

| 函数 | 用途 |
|------|------|
| `createAndRunThread` | 开子线程或续写，同步返回 response |
| `queryThreads` | 列出当前 agent 的近期 threads |
| `investigateThread` | 对已有 thread 执行分析指令 |

### createAndRunThread 参数详解

| 参数 | 必填 | 说明 |
|------|------|------|
| `agentUrl` | 否 | 目标 agent 的 compressed URL。省略 = 调自己。`"personal-agent"` = 个人 agent |
| `threadUrl` | 否 | 已有线程 URL，用于续写 |
| `instructions` | 是 | 发给子 agent 的消息/指令 |

### 通信权限模型

- **Personal Agent** → Custom Agent：需要目标 agent 的 URL，无需额外权限。
- **Custom Agent** → Custom Agent：发起方的 Notion integration 中必须声明 `{ identifier: { type: "agent", url: "agent-target" }, actions: ["interact"] }`。
- **Custom Agent** → 自己：始终允许（省略 `agentUrl` 即可）。
- 未授权的调用会被直接拒绝。

### response 的本质

`response` 只包含子 agent 的最终文本回复。子 agent 在运行过程中执行的所有副作用（创建页面、修改数据库、commit 到 GitHub 等）**不会**反映在 response 中。如果需要子 agent 的操作结果，应让子 agent 在 response 中显式返回关键信息（如 commit hash、页面 URL）。

### 关键注意点

- ⚠️ 50 次上限是 **parent thread 级别** 的硬性限制，包括所有子线程调用的总和。
- ⚠️ response 只有文本——子 agent 创建的资源、修改的页面不会出现在 response 里。
- ⚠️ interact 权限不是自动双向的——A→B 和 B→A 各自独立配置。

### 深钻指针

| 主题 | 文件 |
|------|------|
| Thread 函数完整签名 | `modules/notion/threads/index.ts` |
| interact 权限配置 | `modules/notion/integration.ts` |
| Agent 间协作实战 | Ch4 S4.5, Ch5 S5.1 |

---

## S1.4 内容概念族

**适用条件**：需要理解页面内容的格式、Block 类型、Rich text 语法时加载本节。

### 层级关系

```
Page
└── content（Notion-flavored Markdown 字符串）
    └── Block（块）
        ├── 基础块：text / heading / list / to-do / divider / quote / code / equation / table
        ├── 高级块：toggle / callout / columns / synced_block / meeting-notes
        ├── 嵌入块：image / video / audio / file / pdf
        ├── 引用块：page（子页面）/ database（数据库块）
        └── Rich text（行内格式）
            ├── 格式：bold / italic / strikethrough / underline / inline code / link
            ├── 颜色：text colors + background colors
            ├── 特殊：inline math / citation / mention
            └── Mention 类型：user / page / database / data-source / agent / date
```

### Notion-flavored Markdown vs 标准 Markdown

本系统使用的是 **Notion-flavored Markdown**，与标准 Markdown 的核心差异：

| 特性 | Notion-flavored | 标准 Markdown |
|------|-----------------|---------------|
| Block 颜色 | `{color="red"}` 属性列表 | 不支持 |
| 下划线 | `<span underline="true">` | 不支持 |
| Toggle | `<details><summary>` | 不标准 |
| Callout | `<callout icon="emoji">` | 不支持 |
| Columns | `<columns><column>` | 不支持 |
| Mention | `<mention-user url="...">` | 不支持 |
| 多行引用 | `> Line1<br>Line2`（单行 `>`） | 多行 `>` |
| 空行 | `<empty-block/>` | 空行 |
| Table | HTML `<table>` 语法 | pipe 语法 |
| 子页面 | `<page url="...">Title</page>` | 不支持 |
| 数据库块 | `<database url="...">` | 不支持 |

### 内容编辑模式

`updatePage` 的 `contentUpdates` 支持两种模式：

1. **局部替换**（推荐）：提供 `oldStr` + `newStr`，精确替换匹配的内容段。`oldStr` 必须在页面中唯一匹配，除非设置 `replaceAllMatches: true`。
2. **全量替换**（最后手段）：只提供 `newStr`，覆盖整个页面内容。

### Database Block 铁律

页面内容中的 `<database>` 块代表嵌入的数据库。编辑页面时：
- **必须原样保留** `<database>` 块的属性和内容，否则更新被拒绝。
- 不能通过页面函数创建新的 `<database>` 块——用 `createDatabase` 创建后再用页面函数定位。
- 可以通过页面函数 **移动位置** 或 **删除** 已有的 `<database>` 块。

### 关键注意点

- ⚠️ 多行 quote 必须用 `<br>` 换行，不能用普通换行符——普通换行会变成多个独立 quote 块。
- ⚠️ `<page url="...">` 是子页面引用，移除该标签会 **删除** 子页面。如果只想引用不想移动，用 `<mention-page>`。
- ⚠️ Code block 内部不要转义特殊字符——内容是 literal 的。转义规则只在 code block 外生效。
- ⚠️ Heading 5 和 6 不支持，会被转为 heading 4。

### 深钻指针

| 主题 | 文件 |
|------|------|
| 完整 Markdown 语法规范 | `modules/notion/notion-markdown.md` |
| 页面内容写作规范 | `modules/notion/pages/page-content-spec.md` |
| 页面函数签名 | `modules/notion/pages/index.ts` |
| Notion Icons 列表 | `modules/notion/pages/notion-icons.md` |

---

## S1.5 数据概念族

**适用条件**：需要理解 DataSource schema 到 SQLite table 的映射、查询语法时加载本节。

### 映射关系总览

```
DataSource（Notion 概念）
│
├── schema: Record<propertyKey, PropertySchema>
│   key = CREATE-* 或 compressed URL
│   value = { type, name, ... }
│
└── 映射为 SQLite table
    ├── 表名 = DataSource 的 compressed URL（如 "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"）
    ├── 系统列：url (TEXT), createdTime (TEXT)
    └── 属性列：根据 property type 映射
```

### Property Type → SQLite Column 映射

| Property Type | SQL Type | 值格式 | 注意事项 |
|---------------|----------|--------|----------|
| title / text / url / email / phone_number | TEXT | 纯字符串 | — |
| number | FLOAT | 数值 | — |
| checkbox | TEXT | `"__YES__"` / `"__NO__"` | 不是 boolean |
| select / status | TEXT | option name 字符串 | — |
| multi_select | TEXT | JSON 数组字符串 | 用 `json_each` 过滤 |
| person | TEXT | JSON 数组（user URL） | limit=1 时可能是单值 |
| relation | TEXT | JSON 数组（page URL） | limit=1 时可能是单值 |
| date | 展开 3 列 | `"date:<Name>:start"` / `":end"` / `":is_datetime"` | start 为 ISO-8601 |
| place | 展开 5 列 | `"place:<Name>:address"` / `":name"` / `":latitude"` / `":longitude"` / `":google_place_id"` | 不能直接设置基础列 |
| created_time / last_edited_time | TEXT | ISO-8601（NOT NULL） | 只读 |
| created_by / last_edited_by | TEXT | user URL | 只读 |
| auto_increment_id | INTEGER | 数值 | 只读 |

### 列名冲突规则

如果 property name 与系统列名（`id`、`url`、`createdTime`）冲突，SQL 列名加 `userDefined:` 前缀。例如：property name 为 `"url"` → SQL 列名为 `"userDefined:url"`。

### 两种查询方式

| 方式 | 函数 | 适用场景 |
|------|------|----------|
| SQL 查询 | `querySql({ dataSourceUrls, query, params })` | 自定义过滤、跨表 JOIN、聚合 |
| View 查询 | `queryView({ viewUrl })` | 需要 view 已配置的过滤/排序 |

### SQL 查询铁律

- 表名必须双引号括起：`SELECT * FROM "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`
- 列名有空格/特殊字符时双引号括起：`"Due Date"`
- 多值字段（multi_select / person / relation）用 `json_each` 展开过滤
- 参数化查询用 `?` 占位，值通过 `params` 数组传入

### Property Schema 的 25 种类型

`title` · `text` · `url` · `email` · `phone_number` · `file` · `number` · `date` · `select` · `multi_select` · `status` · `person` · `relation` · `checkbox` · `created_time` · `last_edited_time` · `place` · `auto_increment_id` · `formula` · `location` · `rollup` · `created_by` · `last_edited_by` · `last_visited_time` · `button` · `verification`

### 关键注意点

- ⚠️ checkbox 在 SQL 中是 `"__YES__"` / `"__NO__"` 字符串，不是 boolean。但在 `updatePage` 的 `propertyUpdates` 中使用 `true` / `false`。
- ⚠️ date 和 place 属性在 SQL 和 propertyUpdates 中都使用展开列名，不能直接设置基础 property name。
- ⚠️ schema key 不是 property name。`{ "CREATE-title": { type: "title", name: "Name" } }` 中 key 是 `"CREATE-title"`，name 是 `"Name"`。用 `"Name"` 做 key 会失败。
- ⚠️ Property name 是 **大小写敏感** 的——`"Title"` 和 `"title"` 是不同的 key。

### 深钻指针

| 主题 | 文件 |
|------|------|
| SQLite 表结构完整规范 | `modules/notion/databases/data-source-sqlite-tables.md` |
| Property 类型定义 | `modules/notion/databases/dataSourceTypes.ts` |
| 查询类型签名 | `modules/notion/databases/query.ts` |
| Formula 语法 | `modules/notion/databases/formula-spec.md` |
| Page 属性值格式 | `modules/notion/pages/AGENTS.md`（Properties 节） |

---

## S1.6 搜索概念族

**适用条件**：需要从 workspace、连接器或 web 搜索信息时加载本节。

### 搜索链路

```
search({ queries, includeWebResults? })
│
├── queries: Array<SearchQueryInput>
│   ├── question    ← 单一聚焦问题
│   ├── keywords    ← 2-4 个核心实体词
│   ├── lookback?   ← 时间窗口
│   └── includeNotionHelpdocs? ← 是否包含 Notion 帮助文档
│
├── includeWebResults? ← 默认 true；设 false 仅搜内部
│
└── 返回 SearchEffectResult
    └── results: Array<SearchResult>
        ├── id / type / title / snippet / url
        ├── path / lastEdited / badges
        └── messages?（Slack/Teams 结果特有）
```

### lookback 取值规则

| 值 | 含义 |
|----|------|
| `"default"` | 系统默认时间窗口 |
| `"all_time"` | 全部时间（仅用于稳定/常青内容如密码） |
| `"7d"` / `"2w"` / `"3m"` / `"1y"` | 相对时间段 |
| `"2024-04-01"` | ISO 日期，搜索该日期之后的内容 |

⚠️ 不要用自然语言如 `"last month"` —— 必须转为具体值（如 `"30d"`）。

### Citation 格式

搜索结果用于 chat 回复时需要 citation：

- 行内标记：`Some fact.[^search-result-url]`
- 多个来源：`Important fact.[^agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40][^dataSourceUrl]`
- 不要添加脚注定义行
- 只有搜索结果中的 URL 可以 cite，不能 cite 工具调用本身的 URL（如 `toolu_018fBJdd6QX46iMq4yzDTrKZ`）

### 搜索结果类型

`block` · `properties` · `page` · `helpdoc` · `helpdoc-section` · `webpage` · `slack` · `microsoft-teams` · `gmail` · `google-calendar` · `notion-calendar` · `notion-mail` · `outlook` · `google-drive` · `github` · `jira` · `sharepoint` · `salesforce` · `discord` · `asana` · `linear` · `box` · `custom`

### 关键注意点

- ⚠️ keywords 不是完整问题的复述——提取 2-4 个最有区分度的核心词即可。
- ⚠️ 对于「最近的」、「上次的」等模糊时间，从 `"1w"` 开始尝试，无结果再扩大到 `"1m"`、`"all_time"`。
- ⚠️ 用户说「搜索 Notion」指的是搜索 workspace 内容，设 `includeWebResults: false`，不是搜索 Notion 帮助文档。

### 深钻指针

| 主题 | 文件 |
|------|------|
| Search 函数签名与结果类型 | `modules/search/index.ts` |
| Search 使用指南与示例 | `modules/search/AGENTS.md` |
| Notion 帮助文档搜索 | `modules/helpdocs/AGENTS.md` |

---

## S1.7 能力边界速查

**适用条件**：需要判断某操作是否可行、是否需要特殊权限时加载本节。

### ✅ 可以做

| 能力 | 关键函数/工具 |
|------|---------------|
| 创建/编辑/删除/归档页面 | `createPage` / `updatePage` / `deletePages` / `archivePages` |
| 创建/更新/删除数据库 | `createDatabase` / `updateDatabase` / `deleteDatabases` |
| SQL 查询数据 | `querySql` / `queryView` |
| 跨源搜索（workspace + 连接器 + web） | `search` |
| 创建/更新 Agent | `createAgent` / `updateAgent` |
| Agent 间通信 | `createAndRunThread` |
| 管理页面/数据库权限 | `loadPermissions` / `updatePermission` |
| 发送通知 | `sendNotification` |
| 查看/添加评论 | `getPageDiscussions` / `addCommentToDiscussion` |
| 调用 MCP Server 工具 | `listTools` / `runTool` |
| 列出/浏览 Teamspace | `listTeamspaces` / `getTeamspaceTopLevelPagesAndDatabases` |
| 用户查找与活动记录 | `loadUser` / `searchUsers` / `getUserActivity` |
| 管理用户连接 | `listUserConnections` / `createUserConnection` |
| 查看文件 | `viewFileUrl` |
| 查询会议记录 | `queryMeetings` / `loadMeetingNoteTranscript` |
| Analytics | `getUserEngagementAnalytics` / `getContentEngagementAnalytics` 等 |

### ❌ 不能做

| 限制 | 说明 |
|------|------|
| Workspace 管理操作 | 设置、角色、计费、安全、域名 |
| 数据库 automations 管理 | 不能通过 API 管理数据库内置自动化 |
| 创建 typed tasks database | 不支持 |
| 联系人 | 不能发邮件、打电话、发消息给外部联系人 |
| 使用 Notion 外部工具 | 除搜索连接器和 MCP 外，不能直接操作外部系统 |
| 编辑 Agent 自身配置 | Agent 不能在运行时修改自己的 instructions 以外的配置（除非有 `updateAgent` 权限且目标是其他 agent） |
| 后台持续监控 | Agent 不是后台进程，不能「持续监听」或「等待将来的信息」 |

### 🔐 需要特殊权限

| 操作 | 需要的权限 |
|------|------------|
| 读写特定页面/数据库 | Notion integration 中对应资源的 `"view"` / `"edit"` 权限 |
| Agent 间通信 | 发起方 Notion integration 中对目标 agent 的 `"interact"` 权限 |
| 网络搜索 | `{ identifier: "webSearch", actions: ["allow"] }` |
| 帮助文档搜索 | `{ identifier: "helpdocsSearch", actions: ["allow"] }` |
| MCP 工具调用 | 需要对应的 mcpServer integration |
| 分享权限管理 | 需要对目标资源的 `"edit"` 权限 |

### 关键注意点

- ⚠️ `sharing.ts` 的权限操作 **不等于** agent 权限——给用户加 page 权限不会让 agent 也能访问。Agent 权限通过 `loadAgent` / `updateAgent` 管理。
- ⚠️ `sendNotification` 的 `bodyContent` 限制 100 字符以内。
- ⚠️ 当不确定能否做某事时，优先查阅对应的 module files，而非猜测或拒绝。

### 深钻指针

| 主题 | 文件 |
|------|------|
| 完整函数清单 | `modules/notion/index.ts` |
| 页面权限管理 | `modules/notion/sharing.ts` |
| Agent 权限配置 | `modules/notion/integration.ts` |
| MCP Server 工具 | `modules/mcpServer/index.ts` |
| 通知函数 | `modules/notion/notifications.ts` |

---

## S1.8 Module Files 导航图

**适用条件**：需要定位某个文件的职责、决定加载顺序时加载本节。

### 顶层目录结构

```
modules/
├── notion/          ← 核心模块：页面、数据库、Agent、搜索等
├── search/          ← 跨源搜索（workspace + 连接器 + web）
├── mcpServer/       ← MCP 外部工具协议
├── helpdocs/        ← Notion 帮助文档搜索
├── system/          ← 运行时函数（wait / updateTodos）
├── web/             ← 公共网页搜索与抓取
└── fs/              ← 虚拟文件系统（只读，用于读取 module files）
```

### notion/ 子目录详解

```
modules/notion/
├── AGENTS.md                ← 入口：核心概念 + 文件路由表
├── index.ts                 ← 完整函数清单与导出类型
├── integration.ts           ← Notion 权限类型定义
├── triggers.ts              ← Trigger 配置与运行时变量类型
├── sharing.ts               ← 页面/数据库分享权限 API
├── notifications.ts         ← 通知函数
├── discussions.ts           ← 评论/讨论函数
├── notion-markdown.md       ← Notion-flavored Markdown 完整语法
├── search.ts                ← Notion 内部搜索类型
│
├── pages/
│   ├── AGENTS.md            ← 页面操作指南（parent 类型、属性格式、模板、移动）
│   ├── index.ts             ← 页面 CRUD 函数签名
│   ├── page-content-spec.md ← 页面内容写作规范
│   └── notion-icons.md      ← Notion Icon 名称列表
│
├── databases/
│   ├── AGENTS.md            ← 数据库操作指南（CREATE-*、linked DB、wiki）
│   ├── index.ts             ← 数据库 CRUD 函数签名
│   ├── dataSourceTypes.ts   ← Property schema 全部 25 种类型定义
│   ├── viewTypes.ts         ← View 全部 10 种类型 + filter/sort/groupBy
│   ├── layoutTypes.ts       ← 页面布局类型
│   ├── query.ts             ← 查询函数类型（querySql / queryView）
│   ├── data-source-sqlite-tables.md ← SQLite 映射完整规范
│   └── formula-spec.md      ← Formula 语法规范
│
├── agents/
│   ├── AGENTS.md            ← Agent 操作指南
│   └── index.ts             ← Agent CRUD 函数签名 + 类型
│
├── threads/
│   ├── AGENTS.md            ← Thread 操作指南
│   └── index.ts             ← Thread 函数签名（createAndRunThread 等）
│
├── teamspaces/
│   ├── AGENTS.md            ← Teamspace 操作指南
│   └── index.ts             ← Teamspace 函数签名
│
├── users/
│   ├── AGENTS.md            ← 用户查找 + 连接管理指南
│   └── index.ts             ← 用户函数签名
│
├── meetingNotes/
│   ├── AGENTS.md            ← 会议记录查询指南
│   └── index.ts             ← 会议记录函数签名
│
└── analytics/
    └── index.ts             ← Analytics 函数签名
```

### 其他模块

```
modules/search/
├── AGENTS.md                ← 搜索最佳实践与示例
├── index.ts                 ← search() 函数签名与结果类型
└── integration.ts           ← 搜索权限

modules/mcpServer/
├── AGENTS.md                ← MCP 使用指南
├── index.ts                 ← listTools / runTool 签名
└── integration.ts           ← MCP 权限

modules/system/
├── AGENTS.md                ← 系统工具指南
└── index.ts                 ← wait() / updateTodos() 签名

modules/web/
├── AGENTS.md                ← Web 搜索/抓取指南
└── index.ts                 ← web.search() / web.loadPage() 签名

modules/helpdocs/
├── AGENTS.md                ← 帮助文档搜索指南
└── index.ts                 ← helpdocs.search() 签名

modules/fs/
├── AGENTS.md                ← 文件系统指南
└── index.ts                 ← readDir / readFiles 签名
```

### 加载顺序建议

**场景：首次理解整体架构**
1. `modules/notion/AGENTS.md` → 全局路由
2. 本章（Ch1 概念索引）→ 概念关系总览
3. 按需深钻具体 module 的 AGENTS.md → index.ts

**场景：执行页面操作**
1. `modules/notion/pages/AGENTS.md` → 操作指南
2. `modules/notion/pages/index.ts` → 函数签名
3. `modules/notion/pages/page-content-spec.md` → 内容格式（创建/编辑内容时 MANDATORY）
4. `modules/notion/notion-markdown.md` → Markdown 语法细节

**场景：执行数据库操作**
1. `modules/notion/databases/AGENTS.md` → 操作指南
2. `modules/notion/databases/index.ts` → 函数签名
3. `modules/notion/databases/dataSourceTypes.ts` → Property 类型（设计 schema 时）
4. `modules/notion/databases/viewTypes.ts` → View 类型（创建 view 时）
5. `modules/notion/databases/data-source-sqlite-tables.md` → SQL 映射（查询时）

**场景：执行 Agent 操作**
1. `modules/notion/agents/AGENTS.md` → 操作指南
2. `modules/notion/agents/index.ts` → 函数签名
3. `modules/notion/integration.ts` → 权限类型（配置权限时）
4. `modules/notion/triggers.ts` → Trigger 类型（配置触发器时）

**场景：执行搜索**
1. `modules/search/AGENTS.md` → 最佳实践
2. `modules/search/index.ts` → 函数签名

**场景：使用 MCP 工具**
1. `modules/mcpServer/AGENTS.md` → 使用指南
2. `modules/mcpServer/index.ts` → 函数签名

### 关键注意点

- ⚠️ module 文件路径按 **module type** 组织（如 `modules/mcpServer/`），不是按 connection name（如 `mcpServer_github`）。调用时用 connection name（`connections.mcpServer_github.runTool`），读文件时用 module type（`modules/mcpServer/index.ts`）。
- ⚠️ `connections.ts` 文件列出了当前可用的所有 connection key，是确认可用 module 的权威来源。
- ⚠️ 每个 AGENTS.md 都有 file routing 指示——按指示加载，不要跳过。

---

## 章末：Module Files 完整清单

| 文件路径 | 职责 |
|----------|------|
| `connections.ts` | 当前可用 connection key 清单 |
| `modules/notion/AGENTS.md` | Notion 模块入口与路由 |
| `modules/notion/index.ts` | Notion 函数完整导出 |
| `modules/notion/integration.ts` | Notion 权限类型 |
| `modules/notion/triggers.ts` | Trigger 配置与变量类型 |
| `modules/notion/sharing.ts` | 分享权限 API |
| `modules/notion/notifications.ts` | 通知函数 |
| `modules/notion/discussions.ts` | 评论/讨论函数 |
| `modules/notion/notion-markdown.md` | Markdown 语法规范 |
| `modules/notion/search.ts` | 内部搜索类型 |
| `modules/notion/pages/AGENTS.md` | 页面操作指南 |
| `modules/notion/pages/index.ts` | 页面函数签名 |
| `modules/notion/pages/page-content-spec.md` | 内容写作规范 |
| `modules/notion/pages/notion-icons.md` | Icon 名称列表 |
| `modules/notion/databases/AGENTS.md` | 数据库操作指南 |
| `modules/notion/databases/index.ts` | 数据库函数签名 |
| `modules/notion/databases/dataSourceTypes.ts` | Property 类型定义 |
| `modules/notion/databases/viewTypes.ts` | View 类型定义 |
| `modules/notion/databases/layoutTypes.ts` | 页面布局类型 |
| `modules/notion/databases/query.ts` | 查询函数类型 |
| `modules/notion/databases/data-source-sqlite-tables.md` | SQLite 映射规范 |
| `modules/notion/databases/formula-spec.md` | Formula 语法 |
| `modules/notion/agents/AGENTS.md` | Agent 操作指南 |
| `modules/notion/agents/index.ts` | Agent 函数签名 |
| `modules/notion/threads/AGENTS.md` | Thread 操作指南 |
| `modules/notion/threads/index.ts` | Thread 函数签名 |
| `modules/notion/teamspaces/AGENTS.md` | Teamspace 指南 |
| `modules/notion/teamspaces/index.ts` | Teamspace 函数签名 |
| `modules/notion/users/AGENTS.md` | 用户与连接管理指南 |
| `modules/notion/users/index.ts` | 用户函数签名 |
| `modules/notion/meetingNotes/AGENTS.md` | 会议记录指南 |
| `modules/notion/meetingNotes/index.ts` | 会议记录函数签名 |
| `modules/notion/analytics/index.ts` | Analytics 函数签名 |
| `modules/search/AGENTS.md` | 搜索最佳实践 |
| `modules/search/index.ts` | 搜索函数签名 |
| `modules/mcpServer/AGENTS.md` | MCP 使用指南 |
| `modules/mcpServer/index.ts` | MCP 函数签名 |
| `modules/system/AGENTS.md` | 系统工具指南 |
| `modules/system/index.ts` | wait / updateTodos 签名 |
| `modules/web/AGENTS.md` | Web 搜索指南 |
| `modules/web/index.ts` | Web 函数签名 |
| `modules/helpdocs/AGENTS.md` | 帮助文档指南 |
| `modules/helpdocs/index.ts` | 帮助文档函数签名 |
| `modules/fs/AGENTS.md` | 文件系统指南 |
| `modules/fs/index.ts` | readDir / readFiles 签名 |

---

## 本章引入的新术语

| 术语 | 定义 | 首次出现 |
|------|------|----------|
| Workspace | 顶层协作空间，容纳 Pages、Databases、Agents、Users | Ch1 S1.1 |
| Teamspace | Workspace 下的团队级分组，可作为 Page/Database 的 parent | Ch1 S1.1 |
| Page | 内容载体，有 parent、properties、content | Ch1 S1.1 |
| Database | DataSource + View 的容器 | Ch1 S1.1 |
| DataSource | Database 拥有的数据表，有 schema（属性列定义），映射为 SQLite table | Ch1 S1.1 |
| View | DataSource 的可视化呈现，10 种类型 | Ch1 S1.1 |
| Property | DataSource schema 中的一列，25 种类型 | Ch1 S1.1 |
| Linked Database | 不拥有 DataSource 的 Database，View 引用外部 DataSource | Ch1 S1.1 |
| Wiki Database | isWiki=true 的 Database，创建页面必须用 wikiPageUrl 作 parent | Ch1 S1.1 |
| Notion-flavored Markdown | Notion 的 Markdown 变体，支持颜色、callout、toggle、mention 等 | Ch1 S1.4 |
| Block | 页面内容的基本单元（text / heading / list / callout / database…） | Ch1 S1.4 |
| Rich text | Block 内的行内格式层（bold / italic / link / mention…） | Ch1 S1.4 |
| Mention | 行内实体引用（user / page / database / data-source / agent / date） | Ch1 S1.4 |
| SearchQueryInput | search 函数的单个查询对象，含 question / keywords / lookback | Ch1 S1.6 |
| lookback | 搜索时间窗口参数，取值 default / all_time / 相对时段 / ISO 日期 | Ch1 S1.6 |
| Citation | 搜索结果在 chat 回复中的引用标记，格式 `[^url]` | Ch1 S1.6 |
| Module files | `modules/` 目录下的类型定义和指南文件集合 | Ch1 S1.8 |
| connection key | `connections.ts` 中声明的可用模块连接标识符 | Ch1 S1.8 |
