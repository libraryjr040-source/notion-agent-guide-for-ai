# 《Notion Agent 使用指南》全书大纲

> 组织原则：多原则并存，不追求一元纯粹性。

## 全书设计共识

1. **定位**：AI-first 操作性知识库，module files 作为深钻入口被引用而非复述
2. **全书组织**：多原则并存，不追求一元纯粹性
3. **主体章节**：按操作域合并（跨 module 而非单 module），4-5 章
4. **概念索引**：独立成章，薄而精，作查询入口
5. **踩坑专区**：独立成章，按因果机制组织横切失败模式
6. **章内结构**：薄概念 DRY 层 + 场景条目（DRY 层体量待首章写完后校准）
7. **章节独立性**：各章独立可加载，不假设线性阅读
8. **成长性**：新模式追加到对应操作域章节，新失败模式追加到踩坑专区

---

## Ch1 概念索引

**覆盖域**：全局概念关系图 + 能力边界
**组织原则**：概念关系
**目标**：读者（AI）加载本章后，能快速定位任何概念的含义、归属 module、与其他概念的关系，并知道去哪里深钻。

### 场景清单

- S1.1 全局概念地图：Workspace → Page → Database → DataSource → View → Property 的层级关系
- S1.2 Agent 概念族：Agent → Instructions → Triggers → Connections → Integrations → Permissions 的关系
- S1.3 Thread 概念族：Thread → Sub-thread → createAndRunThread → response 的生命周期
- S1.4 内容概念族：Page content → Notion-flavored Markdown → Block types → Rich text 的层级
- S1.5 数据概念族：DataSource → Schema → Property → SQLite table → Column 的映射关系
- S1.6 搜索概念族：Search → queries → keywords → lookback → results → citations 的链路
- S1.7 能力边界速查：什么能做 / 什么不能做 / 什么需要特殊权限
- S1.8 Module files 导航图：module 目录结构 → 每个文件的职责 → 加载顺序建议

---

## Ch2 内容操作

**覆盖域**：页面 + Markdown + 模板 + 分享
**组织原则**：场景驱动
**涉及 modules**：`notion/pages`, `notion/sharing`, `notion/notion-markdown.md`, `notion/pages/page-content-spec.md`
**目标**：读者（AI）加载本章后，能完成任何页面级别的创建、编辑、移动、分享操作。

### 场景清单

- S2.1 创建页面：从零创建 + 从模板创建 + wiki 页面创建（特殊 parent 逻辑）
- S2.2 编辑页面内容：contentUpdates 的 oldStr/newStr 模式 + 全量替换 + 多处替换
- S2.3 页面属性操作：读属性 → 改属性 → 特殊属性类型（date 展开、place 展开、relation）
- S2.4 移动页面：updatePage 改 parent + 不同 parent 类型的约束
- S2.5 Notion-flavored Markdown 实战：高级 block 类型组合（toggle + callout + columns + table）
- S2.6 模板管理：创建模板 + 设置默认模板 + 删除模板
- S2.7 页面分享与权限：loadPermissions → updatePermission → 不同 access level
- S2.8 页面验证：verification 属性的设置与清除
- S2.9 批量页面操作：archivePages / unarchivePages / deletePages
- S2.10 文件与媒体：viewFileUrl + 图片/视频/PDF 嵌入
- S2.11 讨论与评论：getPageDiscussions → addCommentToDiscussion
- S2.12 Presentation Mode：用 divider 做幻灯片 + 注意事项

---

## Ch3 数据操作

**覆盖域**：数据库 + 查询 + 搜索 + 连接器
**组织原则**：场景驱动
**涉及 modules**：`notion/databases`, `search`, `notion/databases/query.ts`, `notion/databases/data-source-sqlite-tables.md`
**目标**：读者（AI）加载本章后，能完成数据库创建、schema 设计、SQL 查询、跨源搜索等数据操作。

### 场景清单

- S3.1 创建数据库：CREATE-* 占位符系统 + dataSources + views + schema 设计
- S3.2 Schema 设计：property 类型选择 + status groups + select options + relation + formula
- S3.3 View 管理：table / board / calendar / list / gallery / timeline / chart / form 各类型
- S3.4 SQL 查询：querySql 的 SQLite 方言 + 表名/列名引用 + json_each 技巧 + 跨表 JOIN
- S3.5 View 查询：queryView 的使用场景 + 与 querySql 的选择策略
- S3.6 数据库更新：updateDatabase 的 edits 模式 + set / delete / replaceString
- S3.7 链接数据库：创建 linked database + 外部 dataSourceUrl 引用
- S3.8 双向关联：createTwoWayRelation 的使用 + 源/目标数据源
- S3.9 搜索：search 模块的 query 构造 + keywords + lookback + includeWebResults + citations
- S3.10 Autofill agents：20 行限制 + 何时用 autofill agent + 创建流程指针
- S3.11 Wiki 数据库：isWiki 识别 + wikiPageUrl parent 约束
- S3.12 分析与统计：analytics 系列函数的使用场景

---

## Ch4 Agent 工程 ⭐ Pilot

**覆盖域**：Custom Agent + 自动化 + MCP + Skill
**组织原则**：场景驱动
**涉及 modules**：`notion/agents`, `mcpServer`, `notion/threads`, `notion/triggers`, `notion/agents/agent-setup-guide.md`
**目标**：读者（AI）加载本章后，能完成 agent 创建、权限配置、MCP 集成、trigger 设置、skill 编写等全部 agent 工程操作。

### 场景清单

- S4.1 创建 Custom Agent：createAgent + icon/name/description + integrations 配置
- S4.2 权限配置：Notion 权限（workspace/page/database/agent interact）+ webSearch + helpdocsSearch
- S4.3 MCP 集成：mcpServer 连接 + listTools + runTool + 权限独立性
- S4.4 Trigger 设置：recurrence / page_created / page_updated / page_deleted / button_pressed / agent_mentioned
- S4.5 Agent 间通信：createAndRunThread 跨 agent 调用 + interact 权限 + 自调自
- S4.6 Sub-thread 模式：开子 thread 验证/委派 + 50 次调用上限 + response 只返回最终文本
- S4.7 Skill 编写：Instructions 页面的写作范式 + 上下文加载策略
- S4.8 Agent 更新：updateAgent 的 edits 模式 + 添加/删除 integrations + 修改 triggers
- S4.9 Autofill Agent：createAutofillAgent + 与普通 agent 的区别
- S4.10 多 Agent 协作流程设计：总编/著作郎/校勘官模式 + 执行手册模式 + 状态传递
- S4.11 Agent 调试：queryThreads + investigateThread + 排查通信失败
- S4.12 Notification 发送：sendNotification 给用户的使用场景

---

## Ch5 多 Agent 架构

**覆盖域**：线程 + 通信 + 工作流 + 预算
**组织原则**：场景驱动
**涉及 modules**：`notion/threads`, `notion/agents`, `system`
**目标**：读者（AI）加载本章后，能设计和实现多 agent 协作的完整工作流。

### 场景清单

- S5.1 线程生命周期：thread 创建 → 运行 → 返回 → 续写
- S5.2 跨 session 状态管理：执行手册模式 + Notion 页面作状态源 + loadPage 恢复进度
- S5.3 Context 桥梁设计：BOOK_SUMMARY 模式 + 摘要传递 vs 全文传递 + token 预算管理
- S5.4 并行写作模式：分身（自调自）并行 + 汇总合并
- S5.5 写→审→修循环：派写 → 派审 → 分支（通过/不通过）→ 修订上限
- S5.6 异常处理：MCP 断连 / agent 超时 / 严重质量问题 → 记录 + 决策
- S5.7 全流程编排：Phase 0 初始化 → Phase 1-N 逐章 → Phase N+1 跨章审查 → Phase N+2 收尾
- S5.8 通信协议设计：指令自包含原则 + 输入/输出约定 + 链路设计
- S5.9 GitHub MCP 在多 agent 中的使用：repo 操作 + commit + push + 文件读写
- S5.10 Notification 在工作流中的角色：完成通知 + 异常告警

---

## Ch6 踩坑与优化

**覆盖域**：横切失败模式 + 性能 + Prompt
**组织原则**：因果机制
**涉及 modules**：横切所有 module
**目标**：读者（AI）加载本章后，遇到问题能快速定位原因并找到解法。

### 场景清单

- S6.1 身份与权限类：agent 无权访问页面 / interact 权限缺失 / MCP 未授权
- S6.2 数据格式类：date 展开列名错误 / place 展开方式错误 / checkbox 值格式 / JSON 数组 vs 单值
- S6.3 引用与标识类：用 name 而非 URL 引用 / CREATE-* 与 compressed URL 混淆 / property key 大小写
- S6.4 内容编辑类：oldStr 匹配失败 / 多处匹配未设 replaceAllMatches / database block 被修改导致拒绝
- S6.5 Wiki 与特殊数据库类：wiki parent 错误 / linked database 的 dataSource 来源
- S6.6 Search 与查询类：lookback 用自然语言 / keywords 太泛 / web results 未关闭
- S6.7 Agent 通信类：createAndRunThread 被拒（无 interact 权限）/ response 只有文本 / 50 次上限
- S6.8 MCP 类：工具不存在 / 参数格式错误 / 连接状态异常
- S6.9 Context 管理类：token 窗口溢出 / 加载过多无关 context / BOOK_SUMMARY 过旧
- S6.10 性能优化：批量操作 vs 逐条操作 / 并行调用策略 / 避免重复 loadPage

---

## Pilot 计划

**首发章节**：Ch4 Agent 工程
**选择理由**：跨 module 密度最高，最能检验「不复述 module files」的铁律是否成立
**验收标准**：校勘官五把尺全部 ✅ → 确认写作流程可复制 → 推进其余章节
