# Ch5 多 Agent 架构

**覆盖域**：线程生命周期 · 跨 session 状态 · Context 桥梁 · 并行模式 · 写审修循环 · 异常处理 · 全流程编排 · 通信协议 · GitHub MCP · Notification
**涉及 modules**：`notion/threads`、`notion/agents`、`system`、`mcpServer`、`notion/notifications.ts`、`notion/pages`

---

## 薄概念 DRY 层

本节建关系图谱，不讲解——讲解是 module files 的事。Ch4 已覆盖 Agent、Integration、Permission、Trigger、Sub-thread 等基础概念；本章聚焦这些原语在多 Agent 协作中的**组合模式**。

### 核心实体（本章新增 / 侧重）

| 概念 | 一句话 | module files 指针 |
|------|--------|-------------------|
| Thread 生命周期 | 一次 `createAndRunThread` 调用从创建到返回的完整过程，包括续写 | `modules/notion/threads/index.ts` → `CreateAndRunThread` |
| 50 次上限 | 单个 parent thread 的 `createAndRunThread` 调用硬上限（50 次），所有目标 Agent 共享 | `modules/notion/threads/index.ts` 注释 |
| 执行手册 | 用 Notion 页面作持久化状态源，跨 session 恢复进度的协作模式 | `modules/notion/pages/index.ts` → `LoadPage` / `UpdatePage` |
| 桥梁文件 | 用摘要替代全文传递的 context 管理策略（如 BOOK_SUMMARY.md） | 无专属 API——是设计模式 |
| 通信协议 | 多 Agent 间 instructions / response 的格式约定 | 无专属 API——是设计模式 |
| Phase | 全流程编排中的阶段划分单位（初始化 / 逐章 / 跨章审查 / 收尾） | 无专属 API——是设计模式 |
| wait | 暂停当前 run 并在指定秒数后恢复执行 | `modules/system/index.ts` → `Wait` |
| updateTodos | Agent 内部进度追踪，保持可见性 | `modules/system/index.ts` → `UpdateTodos` |

### 关键关系

- 一个 parent thread 持有 50 次上限，跨所有目标 Agent 共享
- `createAndRunThread` 的 `response` 只含最终文本——副作用（页面创建、MCP 调用、文件写入）不在 response 中
- 续写（传 `threadUrl`）保留子线程的对话上下文，但每次续写仍消耗 1 次配额
- 执行手册模式和桥梁文件模式是互补的：执行手册管**进度状态**，桥梁文件管**知识 context**
- `wait` 让 Agent 暂停后恢复，适用于轮询等待外部操作完成的场景
- `updateTodos` 仅追踪 Agent 内部进度，不影响 Notion 页面上的 to-do 内容

### 与 Ch4 的边界

| 关注点 | Ch4 覆盖 | Ch5 覆盖 |
|--------|----------|----------|
| createAndRunThread 的**调用方式** | ✅ 签名、参数、返回值 | — |
| createAndRunThread 的**架构模式** | — | ✅ 配额分配、并行策略、链式编排 |
| interact 权限**如何配置** | ✅ permission 格式 | — |
| interact 权限在**多角色流程中的拓扑** | — | ✅ 权限图设计 |
| 单个 Agent 的 Skill 编写 | ✅ Instructions 页面范式 | — |
| 多个 Agent 的**协议设计** | — | ✅ 指令自包含、输入/输出约定 |

---

## S5.1 线程生命周期

### 场景入口

理解一次 `createAndRunThread` 调用从发起到返回的完整过程，以及续写机制。

### 适用条件

- 需要向另一个 Agent（或自身的新实例）委派任务
- 需要对已完成的子线程追加指令
- 需要规划多轮交互的对话策略

### 核心模式

#### 阶段一：创建并运行

```typescript
const { threadUrl, response } = await connections.notion.createAndRunThread({
  agentUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",       // 目标 Agent（省略 = 自调自）
  instructions: "执行任务 X...",  // 完整指令
})
```

调用是**同步阻塞**的——`await` 会等到子 Agent 运行完毕并产生最终回复。期间子 Agent 拥有完整的工具访问权限，可以创建页面、查询数据库、调用 MCP 工具。

**返回值**：
- `threadUrl`：子线程标识，可用于后续续写
- `response`：子 Agent 的最终文本回复（仅文本，不含中间工具调用结果）

#### 阶段二：续写

传入已有 `threadUrl`，子线程保留之前的对话上下文：

```typescript
const { response: round2 } = await connections.notion.createAndRunThread({
  agentUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  threadUrl: threadUrl,           // 续写已有线程
  instructions: "基于上一轮结果，修改第三段",
})
```

#### 阶段三：调查（事后分析）

线程运行结束后，可通过 `investigateThread` 回溯分析：

```typescript
const { analysis } = await connections.notion.investigateThread({
  threadUrl: threadUrl,
  instructions: "子 Agent 是否成功读取了 OUTLINE.md？如果失败，原因是什么？",
})
```

### 生命周期状态图

```
创建 → 运行中（子 Agent 执行工具调用）→ 返回 response
                                           ↓
                              续写（传 threadUrl）→ 运行中 → 返回
                                           ↓
                              调查（investigateThread）→ 返回 analysis
```

### 常见陷阱

- ⚠️ **每次续写消耗 1 次配额**：续写不是"免费"的。一个 parent thread 的 50 次上限包括所有创建和续写调用。在设计多轮交互时，将配额纳入规划。
- ⚠️ **续写保留上下文，但不保留工具状态**：子 Agent 的对话历史被保留，但工具（如 MCP 连接）的状态可能因超时而需重新建立。
- ⚠️ **response 与副作用分离**：子 Agent 可能创建了 10 个页面、commit 了 3 个文件，但 response 只包含它最终回复的文本。需要知道副作用结果？在 instructions 中明确要求子 Agent 在回复中报告（如 commit hash、页面 URL）。
- ⚠️ **investigateThread 是只读分析**：它基于 transcript 做分析，不会重新执行线程中的操作，也不会触发子 Agent。

### 深钻指针

- `createAndRunThread` / `queryThreads` / `investigateThread` 完整签名：`modules/notion/threads/index.ts`
- Sub-thread 基础用法（自调自、委派、并行）：Ch4 S4.6

---

## S5.2 跨 session 状态管理

### 场景入口

让 Agent 在多次 session 之间保持进度——上次做到哪里、哪些已完成、哪些待处理。

### 适用条件

- 任务跨越多次触发（定时 trigger、手动 @mention）
- 单次 session 无法完成全部工作（50 次上限、context 窗口、时间约束）
- 需要在异常中断后从断点恢复

### 核心模式

#### 模式一：执行手册页面

用 Notion 页面作为持久化状态源。Agent 每次启动时 `loadPage` 读取状态，完成工作后 `updatePage` 写回。

**执行手册页面结构示例**：

```markdown
# 写书项目执行手册

## 当前阶段
Phase 2 — 逐章写作

## 章节状态
| 章 | 状态 | commit hash | 备注 |
|----|------|-------------|------|
| Ch1 | ⚪ 未开始 | — | — |
| Ch4 | ✅ 已通过 | abc1234 | 2轮修订 |
| Ch5 | 🔵 写作中 | — | 著作郎已派写 |

## 待处理
- [ ] Ch5 draft 交稿后派审
- [ ] Ch1 场景清单待确认
```

**读取 → 决策 → 更新 循环**：

```typescript
// 1. 读取执行手册
const handbook = await connections.notion.loadPage({ url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40" })
// 2. 解析当前状态（从 content 中提取）
// 3. 决定下一步行动
// 4. 执行行动
// 5. 更新执行手册
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  contentUpdates: [{
    oldStr: "| Ch5 | 🔵 写作中 | — | 著作郎已派写 |",
    newStr: "| Ch5 | 🟡 审阅中 | def5678 | draft v1 已提交 |",
  }],
})
```

#### 模式二：updateTodos 进度追踪

`updateTodos` 适用于**单次 session 内**的多步骤任务进度可视化，不跨 session 持久化：

```typescript
// 开始任务
const state = await connections.system.updateTodos({
  todos: [
    { text: "加载 context 文件", status: "done" },
    { text: "写 Ch5 draft", status: "in_progress" },
    { text: "commit 到 GitHub", status: "pending" },
  ],
})

// 完成后更新
await connections.system.updateTodos({
  todos: [
    { id: state.todos[1].id, text: "写 Ch5 draft", status: "done" },
    { id: state.todos[2].id, text: "commit 到 GitHub", status: "in_progress" },
  ],
})
```

#### 模式三：wait + 轮询

当 Agent 需要等待外部操作完成时：

```typescript
// 暂停 60 秒后继续
const result = connections.system.wait({
  seconds: 60,
  reason: "polling",
  message: "等待 CI 完成",
})
// result.resumeAtMs 包含恢复时间
```

### 选择策略

| 需求 | 推荐模式 | 原因 |
|------|----------|------|
| 跨多次 session 保持进度 | 执行手册页面 | Notion 页面持久化，任何 session 可读取 |
| 单次 session 内多步骤可视化 | updateTodos | 轻量，无需创建额外页面 |
| 等待外部操作完成 | wait | 暂停 run 并自动恢复 |
| 跨 Agent 传递状态 | 执行手册页面 + instructions 内嵌 | 页面可被多 Agent 共读，关键状态在指令中直接传递 |

### 常见陷阱

- ⚠️ **updateTodos 不跨 session 持久化**：它只在当前 run 中维持进度列表。下次 session 需要从 Notion 页面或其他持久源恢复状态。
- ⚠️ **执行手册页面的并发写入风险**：如果多个 Agent 同时 `updatePage` 同一个执行手册页面，`oldStr` 匹配可能因内容已变更而失败。设计时确保同一时刻只有一个 Agent 更新执行手册。
- ⚠️ **wait 的 seconds 是近似值**：系统在指定秒数后重新入队，但实际恢复时间取决于队列负载。不要依赖精确定时。
- ⚠️ **状态粒度要适中**：执行手册中记录的应该是「Ch5 已提交 draft」这种里程碑级状态，不是「已写完第 3 段」这种过细粒度。过细的状态会导致频繁更新和 context 膨胀。

### 深钻指针

- `loadPage` / `updatePage`：`modules/notion/pages/index.ts`
- `wait` / `updateTodos`：`modules/system/index.ts`
- 执行手册模式初介绍：Ch4 S4.10

---

## S5.3 Context 桥梁设计

### 场景入口

在多 Agent / 多 session 协作中，如何让下游 Agent 获得足够的 context 而不撑爆 token 窗口。

### 适用条件

- 多个 Agent 需要共享跨章节 / 跨任务的知识
- 单个任务的全部 context 超过一次 session 的 token 窗口
- 需要在 instructions 中传递前序工作的结果

### 核心模式

#### 模式一：桥梁文件（BOOK_SUMMARY 模式）

**原理**：维护一个摘要文件，总编在每个阶段完成后更新。下游 Agent 只加载摘要，不加载全文。

**结构**：

```markdown
# BOOK_SUMMARY — 跨章记忆桥梁

## Ch4 Agent 工程
- **Status**: ✅ 已通过
- **Word count**: ~15,000
- **Key points**: 12 个场景全覆盖；薄概念层建立实体关系图谱
- **New terms**: Custom Agent, Instructions Page, Integration, ...
- **Cross-references**: modules/notion/agents/, modules/mcpServer/
```

**使用规则**：
1. 每章 Agent 加载桥梁文件获取全局视野
2. 不加载其他章节全文——桥梁文件是替代品
3. 总编负责在验收通过后更新桥梁文件
4. 桥梁文件体量控制：每章摘要不超过 10 行

#### 模式二：instructions 内嵌 context

对于一次性传递、量小的 context，直接写在 `createAndRunThread` 的 instructions 中：

```typescript
await connections.notion.createAndRunThread({
  agentUrl: "dataSourceUrl",  // 校勘官
  instructions: `# 派审指令

## 审阅对象
- repo: libraryjr040-source/notion-agent-guide-for-ai
- 文件: src/chapter-05/README.md
- commit: abc1234

## 审阅标准
1. 场景覆盖度
2. 术语一致性（参照 glossary.md）
3. compressed URL 格式

## 输出格式
逐条列出问题，标注严重程度`,
})
```

#### 模式三：外部存储 + 指针传递

大型产物（章节全文、代码文件）写入外部存储（GitHub repo、Notion 页面），只在 instructions / response 中传递指针：

```typescript
// 著作郎写完后返回给总编的不是全文，而是：
// "commit hash: abc1234 | 字数: 12000 | 新术语: Phase"
// 总编拿到 commit hash，传给校勘官的 instructions 中包含 hash 而非全文
```

### Token 预算管理

| context 来源 | 估算策略 | 优化手段 |
|-------------|----------|----------|
| Instructions 页面 | 每次 session 自动加载，体量越大越挤压工作空间 | 保持精炼，详细参考放外部 |
| createAndRunThread 的 instructions | 完全由调用方控制 | 只传必需信息，大文本用指针替代 |
| loadPage 加载的页面 | 按需加载，每页占用 token | 只加载当前任务需要的页面 |
| GitHub 文件 | 通过 MCP 读取，全文进入 context | 只读需要的文件，不做「全 repo 扫描」 |

### 常见陷阱

- ⚠️ **桥梁文件过时比没有更危险**：如果总编忘记在验收后更新 BOOK_SUMMARY，下游 Agent 会基于过时信息做决策。桥梁文件的更新必须是流程中的强制步骤。
- ⚠️ **instructions 内嵌 context 的体量控制**：`createAndRunThread` 的 instructions 没有硬性字符限制，但过长的 instructions 会挤压子 Agent 的工作 context。经验法则：instructions 控制在 2000 字以内，超出部分用指针指向外部。
- ⚠️ **不要在 response 中传大文本**：response 是从子 Agent 返回到父线程的唯一文本通道。大文本应写入 Notion 页面或 GitHub，response 只传摘要和指针。
- ⚠️ **glossary 是术语桥梁**：多 Agent 协作中术语不一致是常见问题。所有 Agent 在写作前加载 glossary.md，新术语在 draft 末尾列出，由总编统一更新 glossary。

### 深钻指针

- `createAndRunThread` 的 instructions 参数：`modules/notion/threads/index.ts`
- `loadPage` 按需加载页面：`modules/notion/pages/index.ts`
- GitHub MCP 文件读取：`modules/mcpServer/index.ts`（通过 `runTool` 调用 `get_file_contents`）

---

## S5.4 并行写作模式

### 场景入口

让一个调度 Agent 同时派发多个子任务，并行执行后汇总结果。

### 适用条件

- 多个子任务之间无依赖关系（如不同章节的写作、不同数据源的查询）
- 需要缩短端到端时间
- 子任务操作的资源互不重叠

### 核心模式

#### 分身并行（自调自）

Agent 开多个子线程调用自身的不同实例：

```typescript
// 著作郎把 3 个独立场景分配给 3 个分身
const s1 = connections.notion.createAndRunThread({
  instructions: "写 S5.1 线程生命周期。输出：完整场景条目 Markdown。",
})
const s2 = connections.notion.createAndRunThread({
  instructions: "写 S5.2 跨 session 状态管理。输出：完整场景条目 Markdown。",
})
const s3 = connections.notion.createAndRunThread({
  instructions: "写 S5.3 Context 桥梁设计。输出：完整场景条目 Markdown。",
})

const [r1, r2, r3] = await Promise.all([s1, s2, s3])
// r1.response, r2.response, r3.response → 3 段独立内容
// 汇总、调整衔接、统一风格后 commit
```

#### 跨 Agent 并行

不同 Agent 并行处理各自擅长的任务：

```typescript
// 总编同时派写和派查
const writeTask = connections.notion.createAndRunThread({
  agentUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",  // 著作郎
  instructions: "写 Ch5 draft...",
})
const researchTask = connections.notion.createAndRunThread({
  agentUrl: "okrs",  // 调研员
  instructions: "调查 thread 续写的 context 保留机制...",
})

const [writeResult, researchResult] = await Promise.all([writeTask, researchTask])
```

### 汇总合并策略

并行分身返回后，调度 Agent 需要：

1. **收集所有 response**
2. **检查一致性**：术语是否统一、风格是否对齐、有无内容重叠
3. **调整衔接**：添加过渡段、统一编号、消除矛盾
4. **统一 commit**：将合并后的完整内容一次性写入目标位置

### 配额规划

| 并行度 | 消耗配额 | 剩余可用 | 适用场景 |
|--------|----------|----------|----------|
| 3 个分身 | 3 次 | 47 次 | 中等章节拆分 |
| 5 个分身 | 5 次 | 45 次 | 大型章节高并行 |
| 3 并行 + 每个 1 轮续写 | 6 次 | 44 次 | 分身需要修订 |
| 3 并行 + 汇总后 1 轮审阅 | 4 次 | 46 次 | 写+审流程 |

### 常见陷阱

- ⚠️ **并行子任务必须操作独立资源**：两个分身同时 `updatePage` 同一个页面，`oldStr` 匹配会因并发修改而失败。设计时确保每个分身写入不同的页面或文件。
- ⚠️ **并行调用同时消耗配额**：5 个并行的 `createAndRunThread` 消耗 5 次配额，与串行相同。并行节省的是时间，不是配额。
- ⚠️ **分身没有共享 context**：每个分身是独立的 Agent 实例，不共享父线程的对话历史。需要共享的 context 必须在每个分身的 instructions 中重复传递（或指向同一个桥梁文件）。
- ⚠️ **汇总合并是不可省略的步骤**：并行写作的产出需要统一编辑。如果直接将分身输出拼接，几乎必然出现术语不一致、风格差异、编号冲突。

### 深钻指针

- `createAndRunThread` 并行用法：`modules/notion/threads/index.ts`
- 子线程的 50 次上限说明：`modules/notion/threads/index.ts` 注释

---

## S5.5 写→审→修循环

### 场景入口

实现「派写 → 派审 → 根据审阅结果决定通过或修订」的质量闭环。

### 适用条件

- 产出物需要质量把关（如文档 draft、代码、翻译）
- 需要将「写作」和「审阅」职责分离到不同 Agent
- 需要控制修订轮次，防止无限循环

### 核心模式

```
总编 ──派写──→ 著作郎 ──返回摘要──→ 总编 ──派审──→ 校勘官 ──返回意见──→ 总编
                                                                            ↓
                                                                    意见分析：通过？
                                                                    ├── 是 → 验收
                                                                    └── 否 → 派修 → 著作郎 → ... (max N 轮)
```

#### 步骤 1：派写

```typescript
const { response: writeReport } = await connections.notion.createAndRunThread({
  agentUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",  // 著作郎
  instructions: `# 派写指令：Ch5

## GitHub Repo
- Owner: org-name
- Repo: book-repo
- 目标路径: src/chapter-05/README.md

## 场景清单
S5.1 ~ S5.10（完整清单）

## 写作要求
（风格、术语、格式等完整说明）

## 输出要求
commit + 返回：commit hash + 字数 + 新术语`,
})
// writeReport → "commit: abc1234 | 字数: 12000 | 新术语: Phase"
```

#### 步骤 2：派审

```typescript
const { response: reviewReport } = await connections.notion.createAndRunThread({
  agentUrl: "dataSourceUrl",  // 校勘官
  instructions: `# 派审指令：Ch5

## 审阅对象
- repo: org-name/book-repo
- 文件: src/chapter-05/README.md
- commit: abc1234

## 审阅标准
1. 场景覆盖度：S5.1-S5.10 是否全部覆盖
2. 独立可加载性：本章独立加载后能否直接使用
3. 不复述 module files：有无 API 签名/参数表的冗余复述
4. 术语一致性：与 glossary.md 是否一致
5. 技术准确性：代码示例中的 identifier 是否使用 compressed URL

## 输出格式
逐条列出问题 + 严重程度（critical / major / minor）`,
})
```

#### 步骤 3：决策分支

```typescript
// 解析审阅报告
const hasCritical = reviewReport.includes("critical")
const hasMajor = reviewReport.includes("major")

if (!hasCritical && !hasMajor) {
  // 通过——更新执行手册 + 更新 BOOK_SUMMARY
} else if (revisionCount < MAX_REVISIONS) {
  // 派修——将审阅意见转发给著作郎
  const { response: revisionReport } = await connections.notion.createAndRunThread({
    agentUrl: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",  // 著作郎
    instructions: `# 修订指令：Ch5

## 校勘官意见
${reviewReport}

## 修订要求
逐条修订，不重写全章。commit message: "ch5: revision round ${revisionCount + 1}"

## 输出
commit hash + 修订摘要`,
  })
  revisionCount++
  // 再次派审...
} else {
  // 达到修订上限——记录遗留问题，标记为「有条件通过」
}
```

### 修订上限设计

| 参数 | 推荐值 | 原因 |
|------|--------|------|
| 最大修订轮次 | 2-3 轮 | 超过 3 轮通常说明指令不清晰，应调整指令而非继续修订 |
| 每轮配额消耗 | 2 次（派修 + 派审） | 3 轮修订 = 6 次额外消耗 |
| 全流程配额预估 | 派写 1 + 派审 1 + 修订 2×2 = 6 次 | 单章完整闭环约消耗 6 次配额 |

### 常见陷阱

- ⚠️ **修订指令必须包含具体意见**：不要只说「请修订」，要将校勘官的逐条意见完整转发。著作郎没有校勘官的对话上下文（指令自包含原则）。
- ⚠️ **修订不等于重写**：在修订指令中明确要求「逐条修订，不重写全章」。否则著作郎可能借修订之名推翻原有结构。
- ⚠️ **修订上限是安全阀**：达到上限时应记录遗留问题而非继续循环。遗留问题可以在跨章审查阶段统一处理。
- ⚠️ **审阅标准必须可操作**：「写得好不好」不是可操作标准。「代码示例中的 identifier 是否使用 compressed URL 格式」才是。审阅标准越具体，修订越高效。

### 深钻指针

- `createAndRunThread` 用于链式调用：`modules/notion/threads/index.ts`
- Agent 间通信的 interact 权限配置：Ch4 S4.2、S4.5

---

## S5.6 异常处理

### 场景入口

多 Agent 工作流中出现 MCP 断连、Agent 超时、严重质量问题时的应对策略。

### 适用条件

- `createAndRunThread` 调用失败或返回异常 response
- MCP 工具调用（`runTool`）返回错误
- 审阅发现不可接受的质量问题
- 达到 50 次上限

### 异常分类与处置

#### 类型一：MCP 断连 / 工具调用失败

**表现**：`runTool` 抛出错误或返回 `statusCode` 非 200

**处置**：
1. 记录错误信息（写入执行手册或日志页面）
2. 判断是否可重试（网络波动 → 重试；认证过期 → 需人工介入）
3. 如不可重试，通知用户：

```typescript
await connections.notion.sendNotification({
  headerContent: "MCP 工具调用失败",
  bodyContent: "GitHub get_file_contents 失败，请检查认证状态",
  sendToWorkflowOwner: true,
})
```

#### 类型二：子 Agent 超时 / 无响应

**表现**：`createAndRunThread` 长时间未返回，或返回的 response 为空/无关内容

**处置**：
1. 检查子 Agent 的 instructions 是否清晰（用 `investigateThread` 分析）
2. 检查子 Agent 的权限配置（用 `loadAgent` 确认 permissions）
3. 如有需要，重新派发（使用新的 thread，不续写失败的 thread）

#### 类型三：严重质量问题

**表现**：校勘官报告 critical 级别问题（如整章结构错误、大量内容缺失）

**处置**：
1. 如果在修订上限内 → 派修，但在指令中强调 critical 问题优先
2. 如果已达修订上限 → 记录问题，通知用户决定是否人工介入
3. 不要尝试自动修复结构性问题——这通常说明原始指令存在歧义

#### 类型四：50 次上限耗尽

**表现**：`createAndRunThread` 返回 `max sub agents exceeded`

**处置**：
1. 这是不可恢复的——当前 parent thread 无法再开子线程
2. 将当前进度写入执行手册页面
3. 通知用户在新的 session 中继续

```typescript
// 将进度持久化
await connections.notion.updatePage({
  url: "agent://755c9fa4-4e97-8185-a342-00033edae600/698ee8ff-a788-49fe-b2f4-608ee81dfc40",
  contentUpdates: [{
    oldStr: "| Ch5 | 🔵 写作中",
    newStr: "| Ch5 | ⚠️ 配额耗尽，需在新 session 继续",
  }],
})
await connections.notion.sendNotification({
  headerContent: "50 次上限耗尽",
  bodyContent: "Ch5 写审流程中 50 次上限已满，请开新 session 继续",
  sendToWorkflowOwner: true,
})
```

### 异常记录模板

在执行手册中记录异常时，使用统一格式：

```markdown
### 异常记录
| 时间 | 类型 | 描述 | 处置 | 状态 |
|------|------|------|------|------|
| Phase 2 Ch5 | MCP 断连 | GitHub API 认证过期 | 通知用户 | 待人工处理 |
| Phase 2 Ch5 | 质量问题 | compressed URL 格式不统一 | 2轮修订未完全修复 | 留待终验 |
```

### 常见陷阱

- ⚠️ **不要在失败的子线程上续写**：如果子 Agent 运行异常，用新的 `createAndRunThread`（不传 `threadUrl`）重新开始，而非续写。续写会继承有问题的上下文。
- ⚠️ **重试有配额成本**：每次重试都消耗 1 次配额。在重试前先分析失败原因（用 `investigateThread`），避免盲目重试浪费配额。
- ⚠️ **异常通知要简洁**：`sendNotification` 的 `bodyContent` 限制 100 字符。详细错误信息写入执行手册，通知只传摘要。
- ⚠️ **配额耗尽时不要尝试任何子线程调用**：包括尝试「用子线程保存进度」——此时应直接用当前 Agent 自身的工具调用来持久化状态。

### 深钻指针

- `sendNotification`：`modules/notion/notifications.ts`
- `investigateThread` 诊断分析：`modules/notion/threads/index.ts`
- `loadAgent` 检查权限：`modules/notion/agents/index.ts`

---

## S5.7 全流程编排

### 场景入口

设计一个完整的多 Agent 协作流程——从项目初始化到所有章节完成并收尾。

### 适用条件

- 项目包含多个阶段、多个产出物
- 需要一个调度 Agent 统领全局
- 单次 session 无法完成全部工作

### 核心模式：Phase 编排

#### Phase 0：初始化

**目标**：搭建协作基础设施

**操作**：
1. 创建 GitHub repo（通过 MCP）或确认 repo 存在
2. 初始化项目文件：OUTLINE.md、BOOK_SUMMARY.md、glossary.md、各章占位文件
3. 创建执行手册页面
4. 确认所有参与 Agent 的权限配置（interact 权限、MCP 权限、内容权限）

**配额消耗**：0 次子线程（总编自己执行）

#### Phase 1-N：逐章写审

**目标**：每章经历「派写 → 派审 → 修订（可选）→ 验收」闭环

**单章流程**：

```
总编 loadPage(执行手册) → 确定当前章
  ↓
总编 createAndRunThread(著作郎, 派写指令) → 1 次
  ↓
总编 createAndRunThread(校勘官, 派审指令) → 1 次
  ↓
通过？── 是 → 更新执行手册 + 更新 BOOK_SUMMARY
      └── 否 → createAndRunThread(著作郎, 修订指令) → 1 次
                createAndRunThread(校勘官, 再审) → 1 次
                （最多重复 2 轮）
```

**单章配额**：2-6 次（写+审 = 2，每轮修订 +2，最多 3 轮 = 6）

**多章配额规划**：

| 章数 | 最小配额（全部一次通过） | 最大配额（每章 3 轮修订） |
|------|------------------------|--------------------------|
| 3 章 | 6 次 | 18 次 |
| 5 章 | 10 次 | 30 次 |
| 6 章 | 12 次 | 36 次 |

当章数较多时，单个 parent thread 的 50 次上限可能不够——需要**分 session 执行**。

#### Phase N+1：跨章审查

**目标**：检查跨章一致性（术语、交叉引用、风格）

**操作**：
1. 总编加载 BOOK_SUMMARY 获取全书概览
2. 派校勘官做跨章审查（每章 1 次调用）
3. 汇总跨章问题，统一派修

#### Phase N+2：收尾

**目标**：最终验收、更新所有元数据文件

**操作**：
1. 更新 BOOK_SUMMARY 所有章节为最终状态
2. 更新 glossary.md 为最终版本
3. 通知用户项目完成

### 跨 session 衔接

当单次 session 的 50 次上限不足以完成全部 phase 时：

1. 在当前 session 结束前，将进度写入执行手册
2. 通知用户进度状态
3. 下次 session 启动时：
   - `loadPage` 读取执行手册
   - 从中断处继续下一个 phase 或下一个章节

### 常见陷阱

- ⚠️ **不要试图在一个 session 中完成全部 phase**：6 章 × 6 次/章 = 36 次，加上初始化和收尾，很容易超过 50 次上限。合理规划每个 session 处理 2-3 章。
- ⚠️ **BOOK_SUMMARY 更新是 phase 之间的硬性步骤**：每章验收通过后，必须立即更新 BOOK_SUMMARY，再进入下一章。否则下一章的 Agent 会拿到过时的 context。
- ⚠️ **Phase 0 不要跳过**：确认权限配置、repo 结构、占位文件——这些准备工作如果缺失，后续 phase 会频繁出错。
- ⚠️ **跨章审查（Phase N+1）的配额预留**：在逐章阶段就要预留跨章审查的配额（约 1 次/章），不要把 50 次全部用在逐章写审上。

### 深钻指针

- `createAndRunThread` 链式调用：`modules/notion/threads/index.ts`
- 执行手册模式：本章 S5.2
- 写审修循环：本章 S5.5
- 桥梁文件更新：本章 S5.3

---

## S5.8 通信协议设计

### 场景入口

设计多 Agent 之间 instructions / response 的格式约定，确保指令不歧义、返回值可解析。

### 适用条件

- 多个 Agent 组成协作链路
- 需要在不同 Agent 之间传递结构化信息
- 需要减少因格式不一致导致的沟通失败

### 核心原则

#### 原则一：指令自包含

每次 `createAndRunThread` 的 instructions 必须包含子 Agent 执行所需的**全部信息**。不假设子 Agent「记住」任何之前的对话。

**自包含清单**：

| 信息类型 | 说明 | 示例 |
|----------|------|------|
| 任务描述 | 做什么 | "写 Ch5 的完整 draft" |
| 资源定位 | 在哪里做 | repo owner/name、branch、file path |
| Context 文件清单 | 需要先读什么 | OUTLINE.md、BOOK_SUMMARY.md、glossary.md |
| 约束条件 | 怎么做 | 写作风格、术语要求、格式标准 |
| 输出格式 | 返回什么 | "commit hash + 字数 + 新术语列表" |

#### 原则二：结构化 instructions

使用 Markdown 标题组织 instructions，方便子 Agent 解析：

```markdown
# 派写指令：Ch5

## 任务
写第五章完整 draft

## GitHub Repo
- Owner: `org-name`
- Repo: `book-repo`
- Branch: `main`
- 目标路径: `src/chapter-05/README.md`

## Context 文件
1. OUTLINE.md
2. BOOK_SUMMARY.md
3. glossary.md

## 写作要求
...

## 输出格式
commit hash + 字数 + 新术语
```

#### 原则三：结构化 response

约定 response 的返回格式，使调用方可以可靠地解析：

```
commit: abc1234
字数: 12000
新术语: Phase, 桥梁文件
验证发现: 无重大问题
```

对于审阅类 response，使用一致的问题格式：

```
## 审阅结果：未通过

### critical
1. S5.3 缺失，未覆盖 Context 桥梁设计

### major
1. 代码示例中 agent URL 使用裸字符串而非 compressed URL
2. glossary 中的「50 次上限」未在正文中使用

### minor
1. S5.1 最后一段缺少深钻指针
```

### 链路设计

多 Agent 协作的通信链路应该是**显式的、单向的、有终止条件的**：

```
总编 ──(派写指令)──→ 著作郎 ──(摘要 response)──→ 总编
总编 ──(派审指令)──→ 校勘官 ──(审阅 response)──→ 总编
总编 ──(修订指令)──→ 著作郎 ──(修订摘要)──→ 总编
```

**关键约束**：
- 每条链路是单向的：A → B → A，不存在 A → B → C → A 的三方链路
- 总编是唯一的路由节点，所有信息通过总编中转
- 著作郎和校勘官之间不直接通信

### 常见陷阱

- ⚠️ **不要假设子 Agent 会主动报告**：如果 instructions 中没有要求「返回 commit hash」，子 Agent 可能只回复「已完成」。输出格式必须在 instructions 中明确要求。
- ⚠️ **指令中的代码/路径必须精确**：repo name、文件路径、branch 名——任何一个错误都会导致子 Agent 操作失败。从执行手册或 context 文件中直接复制，不要凭记忆。
- ⚠️ **避免嵌套调度**：子 Agent 不应再开子线程调度其他 Agent（除非是预设的「自调自」分身模式）。多层嵌套会导致配额消耗不可控且调试困难。
- ⚠️ **response 不是可靠的结构化数据通道**：response 是自由文本，子 Agent 可能不严格遵循约定格式。调用方应做容错解析，不要假设完美格式。

### 深钻指针

- `createAndRunThread` 的 instructions / response：`modules/notion/threads/index.ts`
- 指令自包含原则在实践中的应用：本章 S5.5（派写/派审指令示例）

---

## S5.9 GitHub MCP 在多 Agent 中的使用

### 场景入口

在多 Agent 工作流中通过 GitHub MCP 进行 repo 操作——读文件、写文件、commit、push。

### 适用条件

- 工作流产出物存储在 GitHub repo 中（文档、代码、配置文件）
- 多个 Agent 需要读写同一个 repo 的不同文件
- 需要通过 commit 记录追踪版本和变更

### 核心操作

#### 读取文件

```typescript
const result = await connections.mcpServer_github.runTool({
  toolName: "get_file_contents",
  toolArguments: {
    owner: "org-name",
    repo: "book-repo",
    path: "OUTLINE.md",
  },
})
// result.content[0] → { type: "text", text: "successfully downloaded..." }
// result.content[1] → { type: "resource", resource: { text: "文件内容..." } }
```

#### 创建或更新单个文件

```typescript
await connections.mcpServer_github.runTool({
  toolName: "create_or_update_file",
  toolArguments: {
    owner: "org-name",
    repo: "book-repo",
    path: "src/chapter-05/README.md",
    content: chapterContent,      // 文件完整内容
    message: "ch5: draft v1",     // commit message
    branch: "main",
    sha: "原文件的 SHA",           // 更新已有文件时必须提供
  },
})
```

#### 批量推送多个文件

```typescript
await connections.mcpServer_github.runTool({
  toolName: "push_files",
  toolArguments: {
    owner: "org-name",
    repo: "book-repo",
    branch: "main",
    message: "ch5: draft v1 + glossary update",
    files: [
      { path: "src/chapter-05/README.md", content: chapterContent },
      { path: "glossary.md", content: updatedGlossary },
    ],
  },
})
```

#### 读取目录结构

```typescript
const result = await connections.mcpServer_github.runTool({
  toolName: "get_file_contents",
  toolArguments: {
    owner: "org-name",
    repo: "book-repo",
    path: "src",  // 目录路径
  },
})
// 返回目录下的文件/子目录列表
```

### 多 Agent 协作中的 GitHub 使用模式

| 角色 | GitHub 操作 | 说明 |
|------|-------------|------|
| 总编 | 读取 OUTLINE / BOOK_SUMMARY / glossary | 获取全局 context，更新桥梁文件 |
| 著作郎 | 读取 context 文件 → 写入章节 → commit | 交稿载体是 commit |
| 校勘官 | 读取章节文件 | 审阅对象来自 repo |

### SHA 管理

`create_or_update_file` 更新已有文件时**必须提供原文件的 SHA**。获取方式：

1. 先 `get_file_contents` 读取文件，返回中包含 SHA
2. 将 SHA 传入 `create_or_update_file` 的 `sha` 参数

**替代方案**：使用 `push_files` 可以一次性推送多个文件，不需要单独管理 SHA（它基于 branch HEAD 自动处理）。

### 常见陷阱

- ⚠️ **SHA 必须匹配**：如果文件在你读取 SHA 和写入之间被其他 Agent 修改了，commit 会失败（409 Conflict）。在多 Agent 并行写同一 repo 时，确保每个 Agent 操作不同的文件路径。
- ⚠️ **push_files 是原子操作**：多个文件要么全部成功，要么全部失败。适合需要同时更新多个文件的场景（如章节 + glossary）。
- ⚠️ **content 是完整文件内容**：`create_or_update_file` 和 `push_files` 的 content 字段是文件的**完整内容**，不是增量 diff。确保传入的是完整文本。
- ⚠️ **MCP 工具名和参数不要猜测**：先用 `listTools()` 确认可用工具和参数 schema，再调用。工具名错误或参数格式错误会直接报错。
- ⚠️ **每个 Agent 需要自己的 MCP integration**：如果著作郎需要操作 GitHub，著作郎的 Agent 配置中必须有 GitHub MCP integration。总编的 MCP integration 不能代替。

### 深钻指针

- MCP 工具类型与调用签名：`modules/mcpServer/index.ts`
- MCP integration 配置：Ch4 S4.3
- `push_files` 与 `create_or_update_file` 的区别：GitHub MCP 工具文档（通过 `listTools()` 获取）

---

## S5.10 Notification 在工作流中的角色

### 场景入口

在多 Agent 工作流的关键节点向用户发送通知——完成通知、异常告警、进度报告。

### 适用条件

- 长时间运行的工作流需要让用户知道进度
- 发生需要用户介入的异常
- 阶段性里程碑达成

### 核心模式

#### 完成通知

```typescript
await connections.notion.sendNotification({
  headerContent: "Ch5 写作完成",
  bodyContent: "draft v1 已 commit (abc1234)，共 12000 字",
  sendToWorkflowOwner: true,
})
```

#### 异常告警

```typescript
await connections.notion.sendNotification({
  headerContent: "⚠️ MCP 断连",
  bodyContent: "GitHub API 调用失败，进度已保存至执行手册",
  sendToWorkflowOwner: true,
})
```

#### 阶段报告

```typescript
await connections.notion.sendNotification({
  headerContent: "Phase 2 进度",
  bodyContent: "Ch4 ✅ Ch5 ✅ Ch6 🔵 进行中",
  sendToWorkflowOwner: true,
})
```

### 通知时机设计

| 时机 | 通知类型 | bodyContent 示例 |
|------|----------|------------------|
| 单章 draft 完成 | 完成通知 | "Ch5 draft 已 commit" |
| 单章审阅通过 | 完成通知 | "Ch5 验收通过" |
| 修订达上限仍未通过 | 异常告警 | "Ch5 2轮修订后仍有 critical 问题" |
| MCP / 权限错误 | 异常告警 | "GitHub API 失败，需检查认证" |
| 50 次上限耗尽 | 异常告警 | "配额耗尽，请开新 session 继续" |
| 全部章节完成 | 完成通知 | "全书 6 章全部通过验收" |

### Notification vs createAndRunThread

| 维度 | Notification | createAndRunThread |
|------|-------------|--------------------|
| 方向 | Agent → 用户 | Agent → Agent |
| 目的 | 提醒、告警 | 委派任务、获取结果 |
| 交互性 | 单向、无返回值 | 双向、有 response |
| 体量 | bodyContent ≤ 100 字符 | instructions 无硬限制 |

### 常见陷阱

- ⚠️ **bodyContent 100 字符限制**：超出可能被截断。将详情写入执行手册，通知只传摘要。
- ⚠️ **不要用 Notification 做 Agent 间通信**：Notification 是发给用户的提醒，Agent 间通信用 `createAndRunThread`。
- ⚠️ **通知不要过于频繁**：每次工具调用都发通知会造成信息轰炸。只在里程碑和异常时通知。
- ⚠️ **sendToWorkflowOwner 的语义**：当设为 `true` 且省略 `userUrl` 时，发给 workflow owner（通常是启动 Agent 的用户）。如果需要发给特定用户，设置 `userUrl` 并将 `sendToWorkflowOwner` 设为 `false`。

### 深钻指针

- `sendNotification` 签名与参数：`modules/notion/notifications.ts`
- Notification 基础用法：Ch4 S4.12

---

## 本章涉及的 Module Files 完整清单

| 文件路径 | 内容 |
|----------|------|
| `modules/notion/threads/index.ts` | createAndRunThread、queryThreads、investigateThread |
| `modules/notion/agents/index.ts` | loadAgent、createAgent、updateAgent |
| `modules/notion/notifications.ts` | sendNotification |
| `modules/notion/pages/index.ts` | loadPage、updatePage（执行手册读写） |
| `modules/system/index.ts` | wait（暂停恢复）、updateTodos（进度追踪） |
| `modules/mcpServer/index.ts` | listTools、runTool（GitHub MCP 调用） |
| `modules/notion/integration.ts` | interact 权限类型定义 |
| `modules/notion/triggers.ts` | trigger 配置类型（定时触发跨 session） |

---

## 本章引入的新术语

| 术语 | 定义 | 首次出现 |
|------|------|----------|
| Phase | 全流程编排中的阶段划分单位，如初始化、逐章写审、跨章审查、收尾 | S5.7 |
| 通信协议 | 多 Agent 间 instructions / response 的格式约定，包括自包含原则和结构化格式 | S5.8 |
| 指令自包含原则 | 每次 createAndRunThread 的 instructions 必须包含子 Agent 执行所需的全部信息 | S5.8 |
| SHA（GitHub） | 文件版本标识符，更新已有文件时必须提供以避免冲突 | S5.9 |
