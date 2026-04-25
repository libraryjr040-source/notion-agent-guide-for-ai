# Ch2 内容操作

> **覆盖域**：页面 + Markdown + 模板 + 分享  
> **涉及 modules**：`notion/pages`, `notion/sharing`, `notion/notion-markdown.md`, `notion/pages/page-content-spec.md`  
> **加载本章后你能做什么**：完成任何页面级别的创建、编辑、移动、分享、验证、归档操作。

---

## 薄概念 DRY 层

本章围绕 **Page** 展开。以下是核心实体关系速览——详细定义请深钻对应 module files，此处只建立心智模型。

### 核心实体关系

```
Page
 ├─ parent（决定页面归属）
 │   ├─ { type: "user" }      → 顶层私有页面
 │   ├─ { type: "page" }      → 子页面 / wiki 页面
 │   ├─ { type: "dataSource" } → 数据库内页面
 │   └─ { type: "teamspace" }  → teamspace 顶层页面
 ├─ properties（键值对，由 parent 决定 schema）
 ├─ content（Notion-flavored Markdown 字符串）
 ├─ icon（emoji / URL / Notion Icon / null）
 └─ verification（verified / none）
```

### 关键区分

| 概念 | 说明 | 深钻指针 |
|------|------|----------|
| Page parent | 决定页面属于谁，也决定属性 schema | `modules/notion/pages/AGENTS.md` |
| Page content | Notion-flavored Markdown 格式的字符串 | `modules/notion/notion-markdown.md` |
| Page properties | 非 dataSource 页面只有 `title`；dataSource 页面由 schema 定义 | `modules/notion/pages/AGENTS.md` → Properties 小节 |
| contentUpdates | 增量编辑（oldStr/newStr）或全量替换 | `modules/notion/pages/index.ts` → UpdatePage |
| Sharing / Permission | 页面级的分享控制：user / workspace / public | `modules/notion/sharing.ts` |
| Notion-flavored Markdown | Notion 特有的 Markdown 方言，支持 toggle、callout、columns 等高级 block | `modules/notion/notion-markdown.md` |
| Presentation Mode | 用 `---` 分隔幻灯片的演示模式 | `modules/notion/pages/page-content-spec.md` → Presentation Mode 小节 |

### Module files 加载顺序建议

1. `modules/notion/pages/AGENTS.md` — 页面操作的总指南
2. `modules/notion/pages/index.ts` — 函数签名和类型定义
3. `modules/notion/notion-markdown.md` — 内容格式规范（写内容前必读）
4. `modules/notion/pages/page-content-spec.md` — 页面内容撰写规范（创建/编辑页面前必读）
5. `modules/notion/sharing.ts` — 分享与权限函数
6. `modules/notion/discussions.ts` — 讨论与评论函数

---

## S2.1 创建页面

**你要干什么**：从零创建一个新页面，或基于模板创建，或在 wiki 数据库中创建页面。

### 推荐路径

#### 路径 A：从零创建普通页面

1. 确定 parent：
   - 顶层私有页面 → `{ type: "user", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" }`（传当前用户的 URL）
   - 子页面 → `{ type: "page", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" }`
   - 数据库内 → `{ type: "dataSource", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" }`
   - teamspace 顶层 → `{ type: "teamspace", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" }`
2. 调用 `connections.notion.createPage`，传入 parent、properties、content、icon
3. 设置 title 和 icon（创建时就设，不要事后补）

```ts
await connections.notion.createPage({
  parent: { type: "page", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" },
  icon: "📋",
  properties: { title: "项目周报 2025-W28" },
  content: "## 本周进展\n\n- 完成了 API 对接\n- 修复了 3 个 bug\n\n## 下周计划\n\n- 发布 v2.0"
})
```

#### 路径 B：从模板创建

1. 先通过 `loadDatabase` 获取数据源的模板信息（data source 上的 `default_page_template` 和 `page_templates`）
2. 调用 `createPage` 时传入 `pageTemplate` 参数
3. `pageTemplate` 和 `content` 不能同时使用

```ts
await connections.notion.createPage({
  parent: { type: "dataSource", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" },
  pageTemplate: "dataSourceUrl",  // 模板页面的 compressed URL
  properties: { Title: "从模板创建的任务" }
})
```

⚠️ 如果 data source 有 default template，除非用户明确要求不用模板或指定其他模板，否则应使用默认模板。

#### 路径 C：在 wiki 数据库中创建页面

1. 通过 `loadDatabase` 确认 `isWiki: true` 并获取 `wikiPageUrl`
2. parent 必须用 `{ type: "page", url: wikiPageUrl }`，**不能**用 `{ type: "dataSource" }`

```ts
// 先加载数据库确认 wiki 状态
const db = await connections.notion.loadDatabase({ url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" })
// db.configuration.isWiki === true
// db.configuration.wikiPageUrl === "okrs"

await connections.notion.createPage({
  parent: { type: "page", url: "okrs" },  // wikiPageUrl，不是 dataSource URL
  properties: { title: "Wiki 新条目" },
  content: "这是一个 wiki 页面。"
})
```

### 踩坑清单

- ⚠️ **wiki parent 错误**：wiki 数据库中创建页面用 `{ type: "dataSource" }` 会失败。必须用 `{ type: "page", url: wikiPageUrl }`。触发条件：对 `isWiki: true` 的数据库创建页面时用错 parent 类型。解法：先 `loadDatabase` 检查 `isWiki`，若为 true 则取 `wikiPageUrl` 作为 parent。
- ⚠️ **agent parent 不可用**：`loadPage` 返回结果中可能出现 `{ type: "agent" }` 的 parent，但 `createPage` 不支持 agent 作为 parent。触发条件：试图在 agent 下创建页面。解法：选择其他 parent 类型。
- ⚠️ **pageTemplate 与 content 互斥**：两者不能同时传入。触发条件：同时设置 `pageTemplate` 和 `content`。解法：用模板时不传 content，需要自定义内容时不传 pageTemplate。
- ⚠️ **标题重复**：不要在 content 开头用 H1 重复页面标题，标题已由 properties 渲染在内容上方。

### 深钻指针

- `modules/notion/pages/AGENTS.md` → Tips for new pages
- `modules/notion/pages/index.ts` → `CreatePage` 类型签名
- `modules/notion/pages/page-content-spec.md` → 页面内容撰写规范
- `modules/notion/databases/AGENTS.md` → Wiki databases 小节

---

## S2.2 编辑页面内容

**你要干什么**：修改已有页面的正文内容——局部替换、插入新内容、或全量重写。

### 推荐路径

#### 路径 A：局部替换（首选）

1. 先 `loadPage` 获取当前内容（如果本 session 还未加载过该页面）
2. 确定需要替换的 `oldStr`（必须在页面中唯一匹配）
3. 构造 `newStr` 替换内容
4. 调用 `updatePage` 的 `contentUpdates`

```ts
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  contentUpdates: [{
    oldStr: "## 本周进展\n\n- 完成了 API 对接",
    newStr: "## 本周进展\n\n- 完成了 API 对接\n- 新增了搜索功能"
  }]
})
```

#### 路径 B：多处替换（find-and-replace）

对页面中所有匹配位置执行相同替换，设置 `replaceAllMatches: true`。

```ts
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  contentUpdates: [{
    oldStr: "旧术语名",
    newStr: "新术语名",
    replaceAllMatches: true
  }]
})
```

#### 路径 C：全量替换（最后手段）

传一个不带 `oldStr` 的 contentUpdate，`newStr` 为完整新内容。

```ts
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  contentUpdates: [{
    newStr: "## 全新内容\n\n页面被完全重写。"
  }]
})
```

### 关键决策树

```
需要编辑页面内容？
├─ 修改局部内容 → 路径 A（oldStr/newStr）
├─ 全局查找替换 → 路径 B（replaceAllMatches: true）
├─ 页面几乎全部重写 → 路径 C（无 oldStr 的全量替换）
└─ 多处不同修改 → 一次 updatePage 调用中传多个 contentUpdate 对象
```

### 踩坑清单

- ⚠️ **oldStr 多处匹配**：如果 `oldStr` 在页面中匹配到多个位置且未设 `replaceAllMatches: true`，调用会报错。触发条件：用了过短的 oldStr（如单个词）。解法：扩大 oldStr 范围使其唯一，或确认是全局替换场景后设 `replaceAllMatches: true`。
- ⚠️ **oldStr 匹配不到**：oldStr 必须与页面内容精确匹配（包括空格、换行）。触发条件：页面已被修改但你用的是旧版 content 中的字符串。解法：如果不确定当前内容，先 `loadPage` 刷新。
- ⚠️ **database block 被修改**：页面中的 `<database>` block 必须原样保留（属性和内文都不能改）。触发条件：contentUpdates 的 oldStr/newStr 范围涵盖了 database block 且改动了它。解法：让 oldStr/newStr 避开 database block，或保持其完全不变。
- ⚠️ **不要包含 wrapper 标签**：contentUpdates 中的 oldStr 和 newStr 只能包含 `<content>` 和 `</content>` 之间的内容，不能包含这两个标签本身。
- ⚠️ **同一页面不要并行 updatePage**：对同一个页面的多处修改应合并到一次 `updatePage` 调用中（传多个 contentUpdate 对象），不要并行调用。
- ⚠️ **避免冗余 loadPage**：如果本 session 已经加载过该页面且没有收到「页面已过期」的提示，不要重复加载。依赖已有内容和 updatePage 返回结果。

### 深钻指针

- `modules/notion/pages/index.ts` → `UpdatePage` 类型签名（contentUpdates 参数）
- `modules/notion/pages/page-content-spec.md` → 修改页面时的格式规范
- `modules/notion/notion-markdown.md` → Notion-flavored Markdown 完整语法

---

## S2.3 页面属性操作

**你要干什么**：读取页面属性、修改属性值，特别是处理 date 展开、place 展开、relation 等特殊类型。

### 推荐路径

1. **读属性**：`loadPage` 返回的 `properties` 对象包含所有当前属性值
2. **改属性**：`updatePage` 的 `propertyUpdates` 参数（注意不是 `properties`——`properties` 只用于 `createPage`）
3. **确认 schema**：如果页面在 dataSource 中，先 `loadDatabase` 或 `loadDataSource` 确认属性名和类型

属性值的完整格式规范（各类型的值应该怎么传）请深钻 → `modules/notion/pages/AGENTS.md` → Property value formats 小节。以下只展示跨 module 组合流程和踩坑点。

### 常规属性更新

```ts
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  propertyUpdates: {
    Title: "新标题",
    Status: "In progress",
    Points: 8,
    "Is Urgent": true,
    Tags: ["Bug", "P0"],
    Assignee: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
    Reviewers: ["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "dataSourceUrl"],
    "Related Tasks": ["dataSourceUrl", "okrs"],
  }
})
```

### 特殊属性：Date 展开

Date 属性在 API 中展开为三个键——这是最容易出错的属性类型：

```ts
// 单日期
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  propertyUpdates: {
    "date:Due Date:start": "2025-07-15",
    "date:Due Date:end": null,
    "date:Due Date:is_datetime": 0
  }
})

// 日期范围
await connections.notion.updatePage({
  url: "dataSourceUrl",
  propertyUpdates: {
    "date:Sprint:start": "2025-07-14",
    "date:Sprint:end": "2025-07-25",
    "date:Sprint:is_datetime": 0
  }
})
```

### 特殊属性：Place / Location 展开

Place 属性展开为五个键，最简形式只需 address：

```ts
await connections.notion.updatePage({
  url: "okrs",
  propertyUpdates: {
    "place:Location:address": "1600 Pennsylvania Ave NW, Washington, DC 20500"
  }
})
```

⚠️ **不要直接设置基础属性名**：`"Location": "some address"` 是错误的，必须用 `"place:Location:address"` 展开键。

### 特殊属性：Relation

```ts
// limit 1 的 relation → 单个页面 compressed URL
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  propertyUpdates: {
    "Parent Task": "teams"  // limit 1
  }
})

// 无 limit 的 relation → 页面 URL 数组
await connections.notion.updatePage({
  url: "dataSourceUrl",
  propertyUpdates: {
    "Sub-tasks": ["URL", "example.com", "OPTIONAL_NOTICE"]  // 多选
  }
})
```

### 清除属性值

任何属性都可以通过设为 `null` 来清除：

```ts
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  propertyUpdates: {
    Status: null,
    "date:Due Date:start": null,
    "date:Due Date:end": null,
    "date:Due Date:is_datetime": null
  }
})
```

### 踩坑清单

- ⚠️ **属性名大小写敏感**：`Title` 和 `title` 是不同的键。必须使用 `loadDatabase` 或 `loadDataSource` 返回的 schema 中的精确键名。触发条件：凭记忆写属性名。解法：先加载 schema 确认。
- ⚠️ **date 展开键名格式**：必须是 `"date:<属性名>:start"`，冒号前后无空格。触发条件：写成 `"Due Date start"` 或 `"Due Date.start"`。解法：严格使用 `date:` 前缀格式。
- ⚠️ **place 展开键名格式**：必须是 `"place:<属性名>:address"`，不能直接用基础属性名。触发条件：写成 `"Location": "地址"`。解法：使用 `place:` 前缀展开键。
- ⚠️ **系统列名冲突**：如果属性名与系统列名（`id`、`url`、`createdTime`）冲突，API 中使用 `"userDefined:"` 前缀。触发条件：数据源中有名为 `url` 的自定义属性。解法：使用 `"userDefined:url"` 作为键。
- ⚠️ **createPage 用 `properties`，updatePage 用 `propertyUpdates`**：两个函数的参数名不同。触发条件：在 updatePage 中传 `properties`。解法：更新时用 `propertyUpdates`。

### 深钻指针

- `modules/notion/pages/AGENTS.md` → Properties 小节 + Property value formats（各属性类型的完整值格式）
- `modules/notion/pages/index.ts` → `CreatePage`、`UpdatePage` 类型签名
- `modules/notion/databases/data-source-sqlite-tables.md` → 属性类型与 SQL 列映射（对照参考）

---

## S2.4 移动页面

**你要干什么**：把一个页面移到新的 parent 下（另一个页面、数据源、或 teamspace）。

### 推荐路径

1. 调用 `updatePage`，传入 `parent` 参数指定新 parent
2. 页面会从旧 parent 下移除，出现在新 parent 下

```ts
// 移到另一个页面下
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  parent: { type: "page", url: "dataSourceUrl" }
})

// 移到数据源中
await connections.notion.updatePage({
  url: "okrs",
  parent: { type: "dataSource", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" }
})

// 移到 teamspace 顶层
await connections.notion.updatePage({
  url: "teams",
  parent: { type: "teamspace", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" }
})
```

### 踩坑清单

- ⚠️ **不能移到 agent 下**：虽然 `loadPage` 可能返回 `{ type: "agent" }` 的 parent，但 `updatePage` 不支持把页面移到 agent 下。触发条件：试图用 `{ type: "agent" }` 作为 parent。解法：选择 page / dataSource / teamspace 作为 parent。
- ⚠️ **「移动」不是「加别名」**：用户说"移动页面"时，应该改 parent，不要在目标位置添加子页面链接/别名。
- ⚠️ **移动可与其他更新合并**：`updatePage` 的 `parent`、`propertyUpdates`、`contentUpdates` 可以在同一次调用中完成。

### 深钻指针

- `modules/notion/pages/AGENTS.md` → Moving pages 小节
- `modules/notion/pages/index.ts` → `UpdatePage` 的 `parent` 参数

---

## S2.5 Notion-flavored Markdown 实战

**你要干什么**：在页面内容中使用高级 block 类型——toggle、callout、columns、table 等组合。

### 核心原则

- 页面内容是 **Notion-flavored Markdown** 格式的字符串
- 基础 Markdown（标题、列表、粗体、斜体、代码块等）与标准 Markdown 一致
- 高级 block 类型使用 XML 风格标签
- **缩进用 tab**，不用空格
- 特殊字符需转义：`\ * ~ \` $ [ ] < > { } | ^`（代码块内不转义）

各高级 block 类型（toggle、callout、columns、table、synced block、page 等）的完整语法和属性请深钻 → `modules/notion/notion-markdown.md` → Advanced Block types 小节。以下只展示 module files 中没有的——多种 block 组合的实战案例和常见踩坑。

### 实战案例：多种高级 block 组合

以下展示 callout + columns + toggle + table 嵌套组合在一个页面中的写法：

```markdown
<callout icon="🎯" color="blue_bg">
	**本周目标**：完成 v2.0 发布准备
</callout>

<columns>
	<column>
		## 已完成
		- [x] API 对接
		- [x] 单元测试
	</column>
	<column>
		## 进行中
		- [ ] 集成测试
		- [ ] 文档更新
	</column>
</columns>

<details>
<summary>详细进度</summary>
	<table header-row="true">
		<tr>
			<td>模块</td>
			<td>进度</td>
		</tr>
		<tr>
			<td>前端</td>
			<td>80%</td>
		</tr>
		<tr>
			<td>后端</td>
			<td>95%</td>
		</tr>
	</table>
</details>
```

### 多行引用的正确写法

```markdown
> 第一行<br>第二行<br>第三行
```

⚠️ **不要**在引用中用普通换行——每个 `>` 开头的行会变成独立的引用块：

```markdown
> 这是引用 1
> 这是引用 2（这是另一个独立引用，不是同一个引用的第二行！）
```

### 踩坑清单

- ⚠️ **toggle children 未缩进**：toggle 和 toggle heading 的内容必须 tab 缩进，否则不会包含在 toggle 内。触发条件：children 没有缩进。解法：所有 children 内容用 tab 缩进。
- ⚠️ **callout 内用 HTML 格式化**：callout 内应使用 Markdown 格式化（`**bold**`），不用 HTML（`<strong>`）。触发条件：习惯性用 HTML 标签。解法：统一使用 Markdown。
- ⚠️ **table cell 中放 block 类型**：table cell 只支持 rich text，放入标题、列表等 block 类型不会正确渲染。解法：只在 cell 中使用 inline 格式化。
- ⚠️ **`<page>` 标签误用**：用已有页面 URL 的 `<page>` 标签会把该页面移动为子页面；删除 `<page>` 标签会移除子页面。如果只是想引用页面，用 `<mention-page>`。
- ⚠️ **代码块内转义**：代码块（` ``` `）内的内容是 literal，不要转义特殊字符。只有代码块外才需要转义。
- ⚠️ **inline code 中不能换行**：inline code（`` ` ``）中不能用普通换行，用 `<br>` 代替。

### 深钻指针

- `modules/notion/notion-markdown.md` — 完整语法参考（全部 block 类型、rich text、颜色、mention、高级 block 等）
- `modules/notion/pages/page-content-spec.md` — 页面内容撰写规范（创建 vs 修改的风格指导）

---

## S2.6 模板管理

**你要干什么**：为数据库创建模板页面、设置默认模板、删除模板。

### 推荐路径

#### 创建模板

1. 确认目标数据源 URL（通过 `loadDatabase`）
2. 调用 `createPage`，parent 为数据源，设 `asTemplate: true`
3. 模板的属性必须使用 owning data source 的 schema 键名（大小写敏感）

```ts
await connections.notion.createPage({
  parent: { type: "dataSource", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" },
  asTemplate: true,
  properties: { Title: "Bug 报告模板" },
  content: "## Bug 描述\n\n## 复现步骤\n\n1. \n2. \n3. \n\n## 预期行为\n\n## 实际行为"
})
```

#### 设置默认模板

通过 `updateDatabase` 的 edits 修改 data source 的 `default_page_template` 字段。该字段的值是模板页面的 compressed URL，必须是已存在于该 data source `page_templates` 列表中的模板。

```ts
// 假设 loadDatabase 返回的 data source 中已有模板 dataSourceUrl
await connections.notion.updateDatabase({
  databaseUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  edits: [{
    command: "set",
    path: ["dataSources", "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "default_page_template"],
    value: "dataSourceUrl"
  }]
})
```

清除默认模板（不再自动使用模板创建新页面）：

```ts
await connections.notion.updateDatabase({
  databaseUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  edits: [{
    command: "set",
    path: ["dataSources", "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "default_page_template"],
    value: null
  }]
})
```

#### 删除模板

调用 `deletePages`，传入模板页面的 URL。系统会自动从数据源的模板列表中移除，若是默认模板也会自动清除。

```ts
await connections.notion.deletePages({
  pageUrls: ["dataSourceUrl"]
})
```

### 踩坑清单

- ⚠️ **模板属性键名必须精确匹配**：模板的 properties 必须使用 data source schema 中的精确键名。触发条件：键名大小写不对。解法：先 `loadDatabase` 确认 schema。
- ⚠️ **默认模板必须已存在**：`default_page_template` 的值必须是已通过 `createPage({ asTemplate: true })` 创建并存在于 `page_templates` 列表中的模板 URL。设置不存在的 URL 会失败。
- ⚠️ **默认模板的使用**：如果 data source 已有 default template，创建新页面时应默认使用它（除非用户明确要求）。data source 上的 `default_page_template` 字段记录了默认模板 URL。

### 深钻指针

- `modules/notion/pages/AGENTS.md` → Template pages 小节
- `modules/notion/databases/AGENTS.md` → Templates 说明
- `modules/notion/databases/dataSourceTypes.ts` → `DataSource` 类型（`default_page_template`、`page_templates` 字段定义）
- `modules/notion/databases/index.ts` → `UpdateDatabase` 的 edits 模式

---

## S2.7 页面分享与权限

**你要干什么**：查看页面的当前分享设置，给用户/workspace/公开添加或修改访问权限。

### 推荐路径

1. 调用 `loadPermissions` 查看当前权限列表
2. 调用 `updatePermission` 设置或移除一个权限项
3. 每次 `updatePermission` 只能操作一个权限项（一个 user / workspace / public）

Access level 的完整定义（`full_access` / `can_edit` / `can_edit_content` / `can_comment` / `can_view` / `no_access` 各自的含义）请深钻 → `modules/notion/sharing.ts` → `SharingAccessLevel` 类型。

#### 查看权限

```ts
const perms = await connections.notion.loadPermissions({ url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" })
// perms.items → PermissionItem[]
```

#### 给用户添加编辑权限

```ts
await connections.notion.updatePermission({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  item: {
    type: "user",
    userUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
    accessLevel: "can_edit"
  }
})
```

#### 设置 workspace 级权限

```ts
await connections.notion.updatePermission({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  item: {
    type: "workspace",
    accessLevel: "can_view"
  }
})
```

#### 公开分享（Share to web）

```ts
await connections.notion.updatePermission({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  item: {
    type: "public",
    accessLevel: "can_view",
    publicOptions: {
      allowDuplicate: true,
      allowSearchEngineIndexing: false
    }
  }
})
```

#### 移除权限

```ts
await connections.notion.updatePermission({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  item: {
    type: "user",
    userUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
    accessLevel: "no_access"
  }
})
```

### 踩坑清单

- ⚠️ **sharing.ts 的权限 ≠ agent 权限**：`updatePermission` 控制的是页面/数据库级别的人员分享权限，不是 agent 的访问权限。给 agent 配权限用 `loadAgent` / `updateAgent`。触发条件：想让 agent 访问某页面，用了 `updatePermission`。解法：通过 `updateAgent` 修改 agent 的 integrations permissions。
- ⚠️ **一次只能设一个权限项**：`updatePermission` 每次只接受一个 `item`。需要设置多个权限时逐个调用。
- ⚠️ **public 的 publicOptions**：公开分享时可以控制是否允许复制为模板 (`allowDuplicate`) 和是否允许搜索引擎索引 (`allowSearchEngineIndexing`)。还可以设过期时间 (`expirationTimestamp`，unix 毫秒)。

### 深钻指针

- `modules/notion/sharing.ts` — `LoadPermissions`、`UpdatePermission` 类型签名、`SharingAccessLevel` 完整定义、`PermissionItem` 联合类型

---

## S2.8 页面验证

**你要干什么**：标记页面为已验证（verified），或清除验证状态。

### 推荐路径

通过 `updatePage` 的 `verification` 参数操作：

#### 标记为已验证

```ts
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  verification: { status: "verified" }
})
```

#### 带过期天数的验证

```ts
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  verification: {
    status: "verified",
    expirationDays: 90
  }
})
```

#### 清除验证状态

```ts
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  verification: { status: "none" }
})
```

### 踩坑清单

- ⚠️ **verification 是页面级属性**：所有页面都可以有 verification，不限于 dataSource 内的页面。它是内置属性，不需要在 schema 中定义。
- ⚠️ **verification 可与其他更新合并**：可以在同一次 `updatePage` 调用中同时更新 verification、content、properties。

### 深钻指针

- `modules/notion/pages/index.ts` → `UpdatePage` 的 `verification` 参数

---

## S2.9 批量页面操作

**你要干什么**：批量归档、取消归档、或删除页面。

### 推荐路径

#### 归档页面

归档后页面仍在 workspace 中，但默认隐藏。

```ts
await connections.notion.archivePages({
  pageUrls: ["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "dataSourceUrl", "okrs"]
})
```

#### 取消归档

```ts
await connections.notion.unarchivePages({
  pageUrls: ["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "dataSourceUrl"]
})
```

#### 删除页面（移入回收站）

```ts
await connections.notion.deletePages({
  pageUrls: ["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "dataSourceUrl"]
})
```

### 关键区分

| 操作 | 效果 | 可恢复 |
|------|------|--------|
| `archivePages` | 页面隐藏，content 中显示 `archived` 属性 | 是，通过 `unarchivePages` |
| `deletePages` | 页面移入回收站 | 是，在回收站中恢复（需 UI 操作） |

### 踩坑清单

- ⚠️ **不要对 deleted 或 locked 页面调 updatePage**：如果页面的 `<page>` 标签含 `deleted` 或 `locked` 属性，不要尝试 `updatePage`。
- ⚠️ **deletePages 也用于删除模板**：在模板页面 URL 上调 `deletePages` 会同时从数据源的模板列表中移除。

### 深钻指针

- `modules/notion/pages/index.ts` → `ArchivePages`、`UnarchivePages`、`DeletePages` 类型签名

---

## S2.10 文件与媒体

**你要干什么**：在页面中嵌入图片、视频、PDF、音频、文件，或获取文件的可访问 URL。

### 推荐路径

#### 获取文件 URL

`viewFileUrl` 用于获取 Notion 内部文件的可访问 URL（例如上传到 Notion 的附件）。调用后文件会自动附加到当前对话的 transcript 中。

```ts
const result = await connections.notion.viewFileUrl({ url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" })
// result.url → 可访问的文件 URL
```

#### 在页面内容中嵌入媒体

在 Notion-flavored Markdown 中使用以下语法：

```markdown
![图片描述](agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40)

<video src="dataSourceUrl">视频标题</video>

<pdf src="okrs">PDF 文档</pdf>

<audio src="teams">音频文件</audio>

<file src="URL">附件</file>
```

图片也可以使用完整 URL：

```markdown
![示例图片](https://example.com/image.png)
```

### 踩坑清单

- ⚠️ **src 中的 URL 格式**：`src` 属性支持两种合法格式：compressed URL（如 `src="agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40"`）和裸完整 URL（如 `src="https://example.com/file.pdf"`）。但**不能**把完整 URL 塞进双花括号里——例如写成 `src="` + `` + `https://example.com` + `` + `"` 这种形式会导致解析失败，因为系统会尝试将花括号内的内容当作 compressed URL 解压，而完整 URL 不是合法的 compressed URL。简言之：要么用 compressed URL `agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40`，要么用裸 URL `https://example.com/file.pdf`，不要混搭。
- ⚠️ **图片用 Markdown 语法**：图片使用 `![描述](URL)` 格式，不用 `<img>` 标签。

### 深钻指针

- `modules/notion/index.ts` → `ViewFileUrl` 类型签名
- `modules/notion/notion-markdown.md` → Image / Video / Audio / PDF / File block 语法

---

## S2.11 讨论与评论

**你要干什么**：获取页面上的讨论列表，或在讨论/页面上添加评论。

### 推荐路径

#### 获取页面讨论

```ts
const discussions = await connections.notion.getPageDiscussions({
  pageUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  includeResolved: false
})
// discussions → NotionDiscussion[]
```

#### 在已有讨论中回复

```ts
await connections.notion.addCommentToDiscussion({
  discussionUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",  // 从 getPageDiscussions 返回的讨论 URL
  text: "已修复，请查看最新版本。"
})
```

#### 在页面上添加页面级评论

将 `discussionUrl` 传为页面 URL 即可添加页面级讨论评论：

```ts
await connections.notion.addCommentToDiscussion({
  discussionUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",  // 传页面 URL → 页面级评论
  text: "**注意**：这个页面需要更新数据。"
})
```

#### 附带文件的评论

`addCommentToDiscussion` 支持可选参数 `attachedFileIds`，用于在评论中附加文件。传入文件 ID 的数组：

```ts
await connections.notion.addCommentToDiscussion({
  discussionUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  text: "请查看附件中的截图。",
  attachedFileIds: ["agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40", "dataSourceUrl"]
})
```

### 评论格式

`text` 参数支持 **inline-only** Markdown：粗体、斜体、行内代码等。不支持 block 级元素（标题、列表、代码块等）。

### 踩坑清单

- ⚠️ **text 只支持 inline Markdown**：评论文本不能包含标题、列表、代码块等 block 类型。只支持 bold、italic、code、link 等 inline 元素。触发条件：在评论中使用 `##` 或 `- ` 列表。解法：只使用 inline 格式化。
- ⚠️ **discussionUrl 可以是页面 URL**：传页面 URL 时会创建页面级评论；传讨论 URL 时会在已有讨论中追加回复。
- ⚠️ **attachedFileIds 是文件 ID 而非页面 URL**：传入的是文件 ID 数组，不是页面的 compressed URL。确保使用正确的文件标识符。

### 深钻指针

- `modules/notion/discussions.ts` → `GetPageDiscussions`、`AddCommentToDiscussion` 类型签名（含 `attachedFileIds` 参数）

---

## S2.12 Presentation Mode

**你要干什么**：把一个 Notion 页面做成可以演示的幻灯片。

### 推荐路径

1. 设置清晰的页面标题（自动成为第一张幻灯片）
2. 用 `---`（divider）分隔每张幻灯片
3. 每张幻灯片保持简洁——一个标题加少量要点

```ts
await connections.notion.createPage({
  parent: { type: "page", url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" },
  icon: "🎤",
  properties: { title: "Q3 产品发布会" },
  content: `## 议程

- 产品回顾
- 新功能发布
- Q\&A

---

## 产品回顾

- 用户增长 35%
- 核心功能使用率翻倍
- NPS 从 42 提升到 67

---

## 新功能：智能搜索

- 语义搜索 + 关键词搜索
- 跨数据源查询
- 实时索引更新

---

## Q\&A

感谢参与！`
})
```

### 幻灯片规则

| 规则 | 说明 |
|------|------|
| 第一张幻灯片 | 自动为页面标题（从 title 属性渲染） |
| 幻灯片分隔 | 每个 `---` 开始一张新幻灯片 |
| 连续 divider | 连续的 `---` 或之间只有空 block 不会产生空幻灯片 |
| 演示方式 | 页面菜单中的「Present」或快捷键 Cmd+Option+P / Ctrl+Alt+P |

### 踩坑清单

- ⚠️ **Plus 计划要求**：Presentation Mode 需要 Plus 计划或以上。如果用户在 Free 计划上，需要告知此限制。
- ⚠️ **divider 的双重角色**：`---` 在普通上下文是分割线，在 Presentation Mode 中是幻灯片分隔符。已有页面中的 `---` 在开启演示模式后都会变成幻灯片边界。
- ⚠️ **每张幻灯片内容量**：保持简洁，一个标题加少量要点。过多内容会在演示时溢出。

### 深钻指针

- `modules/notion/pages/page-content-spec.md` → Presentation Mode (Slide Decks) 小节

---

## Module files 完整清单

本章涉及的所有 module files：

| 文件路径 | 职责 |
|---------|------|
| `modules/notion/pages/AGENTS.md` | 页面操作总指南：parent 类型、属性格式、移动、模板、icon |
| `modules/notion/pages/index.ts` | 页面函数签名：CreatePage、LoadPage、UpdatePage、DeletePages、ArchivePages、UnarchivePages |
| `modules/notion/pages/page-content-spec.md` | 页面内容撰写规范：创建/修改风格、database block 规则、Presentation Mode |
| `modules/notion/notion-markdown.md` | Notion-flavored Markdown 完整语法：所有 block 类型、rich text、颜色、mention、高级 block |
| `modules/notion/sharing.ts` | 分享函数签名：LoadPermissions、UpdatePermission、SharingAccessLevel、PermissionItem |
| `modules/notion/discussions.ts` | 讨论函数签名：GetPageDiscussions、AddCommentToDiscussion（含 attachedFileIds） |
| `modules/notion/index.ts` | Module 总入口：ViewFileUrl 等通用函数 |
| `modules/notion/pages/notion-icons.md` | Notion Icon 列表（使用 Notion Icon 时查阅） |
| `modules/notion/databases/AGENTS.md` | 数据库指南（Wiki databases、Templates 相关小节） |
| `modules/notion/databases/dataSourceTypes.ts` | DataSource 类型定义（default_page_template、page_templates 字段定义） |
| `modules/notion/databases/index.ts` | 数据库函数签名（UpdateDatabase 的 edits 模式） |
| `modules/notion/databases/data-source-sqlite-tables.md` | 属性类型与 SQL 列映射（属性格式对照参考） |

---

## 本章新术语

| 术语 | 定义 | 首次出现 |
|------|------|----------|
| contentUpdates | `updatePage` 中用于增量编辑页面正文的参数，支持 oldStr/newStr 局部替换和全量替换两种模式 | Ch2 S2.2 |
| propertyUpdates | `updatePage` 中用于修改页面属性的参数，与 `createPage` 的 `properties` 参数名不同 | Ch2 S2.3 |
| date 展开键 | Date 属性在 API 中拆分为三个键的命名格式：`date:<属性名>:start`、`date:<属性名>:end`、`date:<属性名>:is_datetime` | Ch2 S2.3 |
| place 展开键 | Place/Location 属性在 API 中拆分为五个键的命名格式：`place:<属性名>:address` 等 | Ch2 S2.3 |
| replaceAllMatches | contentUpdates 中的选项，设为 true 时对页面中所有匹配位置执行相同替换（find-and-replace 模式） | Ch2 S2.2 |
| Presentation Mode | Notion 页面的幻灯片演示模式，用 `---` divider 分隔幻灯片，需 Plus 计划 | Ch2 S2.12 |
| default_page_template | DataSource 上的字段，记录默认模板页面的 URL，新建页面时自动应用 | Ch2 S2.6 |
| verification | 页面级内置属性，可标记为 verified（可选过期天数）或 none | Ch2 S2.8 |
| attachedFileIds | `addCommentToDiscussion` 的可选参数，用于在评论中附加文件 | Ch2 S2.11 |
