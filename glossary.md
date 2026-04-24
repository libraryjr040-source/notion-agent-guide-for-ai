# Glossary — 术语表

> 每章验收通过后，总编将本章新术语追加至此。著作郎写作时加载本文件确保术语一致。

| 术语 | 定义 | 首次出现 |
|------|------|----------|
| Custom Agent | workspace 内用户创建的 AI 行动者，拥有独立的 instructions、integrations、triggers | Ch4 S4.1 |
| Instructions Page | Agent 的指令页面，本质是 Notion Page，定义 agent 的行为和技能 | Ch4 S4.1 |
| Integration | Agent 与一个 module 的连接记录，控制 agent 可访问的工具和数据 | Ch4 S4.1 |
| Permission（Agent层） | 挂在 Notion integration 下的访问控制项，定义 agent 对页面/数据库的读写权限 | Ch4 S4.2 |
| interact 权限 | Agent 间通信的授权，允许一个 agent 向另一个 agent 发起 createAndRunThread | Ch4 S4.2 |
| MCP Server | 外部工具服务协议（Model Context Protocol），通过 listTools/runTool 调用 | Ch4 S4.3 |
| Trigger | Agent 的自动触发条件，包括定时、页面事件、按钮、@提及等类型 | Ch4 S4.4 |
| Recurrence Trigger | 定时触发器，按 cron 表达式周期性触发 agent | Ch4 S4.4 |
| Sub-thread | 通过 createAndRunThread 开的子线程，用于 agent 间通信或自调自 | Ch4 S4.6 |
| 50 次上限 | 单个 parent thread 的子线程调用硬性限制 | Ch4 S4.6 |
| Skill | Instructions Page 中编写的行为定义，描述 agent 在特定场景下的操作模式 | Ch4 S4.7 |
| Autofill Agent | 专门附着在数据库上的属性填充 Agent，通过 UI 创建（非 API） | Ch4 S4.9 |
| fillablePropertyNames | Autofill trigger 变量中的可填充属性名列表 | Ch4 S4.9 |
| 执行手册模式 | 用 Notion 页面作持久化状态源的跨 session 协作模式 | Ch4 S4.10 |
| 桥梁文件 | 用摘要文件替代全文传递的 context 管理策略（如 BOOK_SUMMARY.md） | Ch4 S4.10 |
| CREATE-\* 占位符 | 同一次 API 调用中标识新建实体的临时 key，系统替换为真实 URL | Ch4 薄概念层 |
| compressed URL | 系统分配的实体标识符，格式为 `prefix-id`，是所有 API 调用中引用实体的标准方式 | Ch4 薄概念层 |
| Phase | 多 Agent 协作中的宏观执行阶段，在执行手册中追踪 | Ch5 S5.7 |
| 通信协议 | Agent 间通过 instructions 字符串传递结构化信息的约定格式 | Ch5 S5.8 |
| 指令自包含原则 | 派发给 sub-agent 的 instructions 必须包含完成任务所需的全部信息，不依赖外部上下文 | Ch5 S5.3 |
| SHA | Git commit 的唯一哈希标识，用于 GitHub MCP 的文件更新操作中防止冲突 | Ch5 S5.9 |
