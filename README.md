# TodoList v2 系统设计方案

> 本文档是 TodoList v2 的设计与复现规范，脱离具体代码实现，只谈设计决策、架构分层、数据契约、业务规则与运维约定。凭此方案应能完整复现整套系统。

---

## 0. 文档定位

- 目标读者：接手或重建本系统的工程师 / AI 代理。
- 覆盖范围：技术栈、分层架构、目录约定、数据模型、业务规则、API 契约、服务编排、前端架构、主题、工程配置、运维操作、部署、测试、数据迁移、错误协议。
- 一句话定位：**单用户、带周期任务与执行记录的待办系统，前后端同构于 Next.js，以 REST API 为权威操作入口，网页 UI 为辅助**。

---

## 1. 项目定位与目标

### 1.1 业务定位
- **单用户**系统（无多租户、无鉴权层）。
- 区别于普通 TodoList 的核心能力：
  - **周期任务**：工作日 / 每周（指定星期）/ 每月（指定日期），任务永不因完成而消失。
  - **执行记录而非完成态**：用历史记录表达完成，支持补录与软撤销。
  - **逾期优先级自动升级**：一次性任务逾期时优先级升一档。
  - **日历视图**：按日聚合完成记录与待办项。

### 1.2 设计目标
1. 业务规则可独立测试：纯逻辑层无 IO 依赖。
2. 前端不耦合 ORM：服务层统一产出 DTO。
3. 契约优先：OpenAPI 作为接口唯一真源。
4. 云原生可运维：双健康探针、统一错误协议、幂等部署。

### 1.3 非目标
- 不做多用户 / 鉴权 / 权限。
- 不做 SEO（首页是登录后场景，全客户端渲染可接受）。
- 不追求大数据量（单用户，内存分页是合理简化）。

---

## 2. 技术栈

| 层面 | 选型 | 版本约束 | 设计意图 |
|---|---|---|---|
| 框架 | Next.js（App Router） | 16.x | 同构、路由即文件、API route 内建 |
| 语言 | TypeScript | 5.x，`strict` | 全量类型安全 |
| UI 库 | React | 19.x | 客户端组件为主 |
| 样式 | Tailwind CSS | v4 | CSS 变量驱动主题，无 tailwind.config.js |
| ORM | Prisma | 7.x | 类型安全 + 生成产物 |
| DB 驱动 | `@prisma/adapter-mariadb` | 匹配 Prisma 版本 | driver-adapter 模式接 MariaDB |
| 数据库 | MySQL / MariaDB | 兼容 | 关系型，软删除 + 索引 |
| 测试 | Vitest | 3.x | 纯逻辑单测，node 环境 |
| 包管理 | pnpm | 10.x | 锁定单一包管理器 |
| 构建/校验 | `next build` + `eslint` | 随框架 | flat config |

> 关键约束：Next.js 16 与历史训练数据可能存在 API/约定差异，写码前须查阅 `node_modules/next/dist/docs/` 对应指南，遵守 deprecation 通告。

---

## 3. 整体架构

经典五层单向依赖，纯逻辑层是防腐核心：

```
┌─────────────────────────────────────────────────────────────┐
│  前端层  Client Components (useState/useEffect)              │
│   主容器组件 + 子组件 + fetch 封装                             │
│        ↓  HTTP fetch                                         │
├─────────────────────────────────────────────────────────────┤
│  API 层  App Router Route Handlers (RESTful)                │
│   参数解析/白名单校验 → 转 service → 统一错误处理               │
├─────────────────────────────────────────────────────────────┤
│  服务层  services (Prisma 编排 + Model→DTO 转换)             │
│   只接已校验的强类型 Query/Input                              │
├─────────────────────────────────────────────────────────────┤
│  纯逻辑层  task-logic + cycle-engine (无 IO 纯函数)          │
│   逾期/排序/筛选/校验/周期触发 —— 单测目标                      │
├─────────────────────────────────────────────────────────────┤
│  数据层  PrismaClient (driver-adapter) + schema             │
└─────────────────────────────────────────────────────────────┘
```

### 3.1 依赖纪律
- 纯逻辑层 **不得** import Prisma / `next/`；输入用最小接口形状（如 `OverdueTaskShape`），便于脱离 DB 单测。
- 服务层向下依赖纯逻辑层 + Prisma，向上暴露 DTO。
- API 层只做参数校验 + 调 service + 错误兜底，不写业务规则。
- 前端只认 DTO 类型，不认 Prisma Model 类型。

---

## 4. 目录结构

```
todolistV2/
├── prisma/
│   └── schema.prisma              # 数据模型，生成产物 output 到 src/generated/prisma
├── public/
│   └── openapi.yaml              # API 契约（权威真源，亦可通过 GET /openapi.yaml 获取）
├── scripts/
│   ├── migrate.ts                # 旧库→新库数据迁移（--dry-run 预演，幂等）
│   ├── start-prod.ps1            # 生产幂等启动（拉起 nginx + next start）
│   └── start-todolist.bat        # 开机自启（复制到 shell:startup）
├── src/
│   ├── app/
│   │   ├── layout.tsx            # 根布局 + ThemeProvider 挂载
│   │   ├── page.tsx              # 首页 → 渲染主容器组件
│   │   └── api/                  # RESTful 路由（见 §7）
│   │       ├── tasks/route.ts
│   │       ├── tasks/[id]/route.ts
│   │       ├── tasks/[id]/executions/route.ts
│   │       ├── tasks/[id]/executions/[execId]/route.ts
│   │       ├── calendar/route.ts
│   │       ├── tags/route.ts
│   │       ├── health/route.ts
│   │       └── health/ready/route.ts
│   ├── components/               # 全部 'use client'
│   │   ├── 主容器.tsx             # 状态中枢 + 三组状态
│   │   ├── 任务条目.tsx
│   │   ├── 任务表单对话框.tsx      # 新建/编辑
│   │   ├── 完成对话框.tsx          # 完成/补录（区分补充 vs 新建）
│   │   ├── 执行记录对话框.tsx      # 历史 + 撤销
│   │   ├── 日历视图.tsx            # 月/周切换
│   │   ├── 通用 UI.tsx            # Modal 等基础组件
│   │   ├── 主题 Provider.tsx      # ThemeProvider/useTheme
│   │   └── 主题定制对话框.tsx
│   ├── lib/
│   │   ├── prisma.ts             # PrismaClient 单例 + driver-adapter
│   │   ├── services.ts           # 业务编排 + DTO 转换
│   │   ├── task-logic.ts         # 逾期/排序/筛选/校验纯函数
│   │   ├── cycle-engine.ts       # 周期触发引擎
│   │   ├── dates.ts              # UTC 存储/本地展示日期工具
│   │   ├── errors.ts             # AppError + RFC 9457
│   │   └── client-api.ts         # 前端 fetch 封装
│   ├── types/                    # TS 类型/接口（DTO/Input/Query）
│   └── generated/prisma/         # Prisma 生成产物（不入库）
├── next.config.*                 # 框架配置
├── tsconfig.json                 # strict + @/* → ./src/*
├── vitest.config.ts              # node 环境，src/**/*.test.ts，复用 @ alias
├── postcss.config.mjs            # @tailwindcss/postcss（v4 接入）
├── eslint.config.mjs             # Next.js flat config
├── package.json
└── AGENTS.md                     # 代理协作约定（Next.js 16 特殊提示）
```

---

## 5. 数据模型

四张表，统一软删除 + 驼峰字段映射蛇形列名。

### 5.1 tasks（任务）
| 字段 | 类型 | 语义 |
|---|---|---|
| id | UUID(36) | 主键，迁移数据沿用旧 ID |
| title | varchar(200) | 必填，≤200 字符 |
| priority | enum high/medium/low/none | 默认 none |
| dueDate | date? | 截止日期；周期任务为模板级提示 |
| description | text? | 任务描述 |
| repeatType | enum workday/weekly/monthly? | 周期类型，空=一次性 |
| repeatDays | JSON? | weekly 取 1-7（1=周一 ISO 约定）；monthly 取 1-31 |
| sortOrder | BigInt | 自定义排序权重，默认 0 |
| createdAt / updatedAt | datetime | 时间戳，updatedAt 自动 |
| deletedAt | datetime? | 软删除标记，默认查询过滤 |

**索引**：`dueDate`、`(priority, dueDate)`、`createdAt`。

### 5.2 executions（执行记录）
| 字段 | 类型 | 语义 |
|---|---|---|
| id | UUID | 主键 |
| taskId | varchar(36) | FK→tasks.id，级联删除 |
| completedDate | date | 执行对应日期（可补录过去） |
| completedAt | datetime | 实际完成时间 |
| executedAt | datetime? | 首次执行时间，可空 |
| completionNote | text? | 完成备注 |
| revoked | boolean | 软撤销标记，默认 false |
| revokedAt | datetime? | 撤销时间 |
| createdAt | datetime | 创建时间 |

**索引**：`taskId`、`completedDate`。

### 5.3 tags / task_tags（标签）
- `tags`：`id` + `name`(唯一，varchar 100) + `createdAt`。
- `task_tags`：`(taskId, tagId)` 联合主键，归一化多对多，两侧级联删除。

### 5.4 设计要点
1. **完成态不入库**：`completed` / `completedToday` 由是否存在有效（未撤销）executions 派生计算。
2. **软删除两级**：任务软删除（deletedAt）+ 执行软撤销（revoked），历史可追溯。
3. **BigInt 排序键**：避免 JS number 精度问题，可用 `BigInt(Date.now())` 填充。
4. **JSON 存周期配置**：repeatDays 灵活承载周/月取值数组。

---

## 6. 核心业务规则（纯逻辑层）

本层是系统智识核心，全部为无 IO 纯函数，单测必覆盖。

### 6.1 周期触发引擎（cycle-engine）
- `isWorkday(d)`：周一至周五。
- `isDueOnDate(repeatType, repeatDays, d)`：
  - workday → 工作日
  - weekly → `repeatDays` 含该日 ISO 周序（1=周一…7=周日）
  - monthly → `repeatDays` 含该日日期
- `listOccurrences(repeatType, repeatDays, from, to)`：逐日扫描 [from,to]，返回所有到期日。扫描时把时刻设为 12:00 避免夏令时/时区跳变。

### 6.2 逾期计算（computeOverdue）
输入用最小接口 `OverdueTaskShape`（repeatType / repeatDays / dueDate / createdAt）+ `OverdueExecShape[]`（completedDate），**刻意脱离 ORM 类型以利单测**。

分治：
- **周期任务**：以上次执行日 +1 为基线，列出 [基线, today-1] 区间内应到期但未执行的发生日 → 任一未执行则 `recurringOverdue=true`，`overdue` 同之。
- **一次性任务**：`dueDate < today && 未完成` → `overdue=true`，`recurringOverdue=false`。

### 6.3 优先级升级（ESCALATE）
```
none → low → medium → high → high（封顶）
```
- 仅一次性任务逾期时触发：`effectivePriority = 升级映射[原 priority]`。
- 周期任务不升级（用 `recurringOverdue` 单独标记）。
- 排序与展示用 `effectivePriority` 而非原始 priority。

### 6.4 多级排序（sortTasks）
优先级排序时分三组，组间不可越级：
```
未完成(0) → 周期逾期(1) → 已完成(2)
```
组内多级：`effectivePriority` 降序 → `dueDate` 升序（空视为 9999-12-31）→ `sortOrder` 升序。

dueDate 排序：直接按截止日期，平局回退 effectivePriority + sortOrder。createdAt 排序：按创建时间字符串比较。

### 6.5 筛选（filterTasks）
支持维度：status(active/completed/recurring)、priority、repeatType、overdue(bool)、dueFrom/dueTo（仅匹配有 dueDate 的任务）。
- `completed` 不含周期任务。
- `overdue=true` 命中 `overdue || recurringOverdue`。

### 6.6 输入校验（validateTaskInput）
- title 必填 ≤200 字符。
- priority/repeatType 枚举白名单。
- dueDate 必须 YYYY-MM-DD。
- tags ≤20 个。
- weekly repeatDays ∈ [1,7] 非空；monthly repeatDays ∈ [1,31] 非空。

---

## 7. API 契约

RESTful + Problem Details (RFC 9457)。权威真源为 `public/openapi.yaml`（亦可通过 `GET /openapi.yaml` 获取）。

### 7.1 Base URL
- 生产：`http://localhost:5030`（nginx 反代到内部 5031）
- 开发：`http://localhost:3000`

### 7.2 健康检查
| 接口 | 说明 |
|---|---|
| GET /api/health | 存活探针，不依赖 DB，200 `{status:"ok",service,version,time}` |
| GET /api/health/ready | 就绪探针，探测 DB；正常 200，不可用 503 `{checks:{database:"up"\|"down"}}` |

调用顺序：先 health 确认进程，再 ready 确认 DB。

### 7.3 任务列表 GET /api/tasks
参数（全可选）：

| 参数 | 取值 | 说明 |
|---|---|---|
| status | all/active/completed/recurring | completed 不含周期任务 |
| tag | 标签名 | 精确匹配 |
| search | 任意文本 | 匹配标题/描述/标签 |
| priority | high/medium/low/none | 按录入优先级过滤 |
| repeatType | workday/weekly/monthly | 周期类型 |
| overdue | true/false | 含周期逾期 |
| dueFrom/dueTo | YYYY-MM-DD | 截止区间，只匹配有截止日期的任务 |
| sortBy | priority/dueDate/createdAt | 默认 priority |
| sortDir | asc/desc | 默认 asc |
| page | 整数 ≥1 | 默认 1 |
| pageSize | 整数 1..500 | 默认 100 |

响应（分页包装）：
```json
{ "items": [TaskDTO...], "total": 52, "page": 1, "pageSize": 100, "hasMore": false }
```

### 7.4 任务详情/增删改
| 接口 | 说明 |
|---|---|
| GET /api/tasks/{id} | 详情 |
| POST /api/tasks | 新建，201 返回完整任务 |
| PUT /api/tasks/{id} | 全量更新，缺省字段按 undefined 处理 |
| DELETE /api/tasks/{id} | 软删除，返回 `{ok:true}` |

TaskInput：
```json
{ "title": "写周报", "priority": "medium", "dueDate": "2026-08-08",
  "description": "...", "tags": ["工作"], "repeatType": "weekly", "repeatDays": [5] }
```

### 7.5 执行记录
| 接口 | 说明 |
|---|---|
| GET /api/tasks/{id}/executions | 完成历史（含已撤销） |
| POST /api/tasks/{id}/executions | 完成/补录/新建，body `{completedDate(默认今天), completionNote, executedAt}` |
| DELETE /api/tasks/{id}/executions/{execId} | 软撤销，返回更新后记录 |

状态语义：
- 一次性任务只能有一条有效执行记录，重复完成 → 409 `TASK_ALREADY_COMPLETED`。
- 周期任务可多次补录（含同日多条），不因完成而隐藏。

### 7.6 标签与日历
- GET /api/tags → `[{id, name, count}]`，count 为该标签下任务数，按 count 降序 + 中文 locale 排序。
- GET /api/calendar?date=YYYY-MM-DD&range=week|month → 按日聚合的 `completion`（完成）与 `pending`（待办）项。

### 7.7 TaskDTO 关键派生字段
`completed` / `completedToday` / `overdue` / `recurringOverdue` / `effectivePriority` / `completionCount` / `latestCompletionNote` / `tags`。全部由服务层从 Model 派生，前端只读。

---

## 8. 服务层编排规则

### 8.1 职责
Prisma 查询编排 + Model→DTO 转换 + 防腐。绝不把 Prisma Model 直接吐给前端。

### 8.2 listTasks 编排
1. 软删除过滤 `deletedAt: null` + 搜索/标签 where。
2. 一次性 `findMany` 带上 executions + taskTags（按时间升序）。
3. 每个 task：过滤出有效 executions → 计算 overdue/effectivePriority → 转 DTO。
4. 内存 `filterTasks` → `sortTasks`。
5. 内存分页 `slice`（page/pageSize，pageSize 上限 500）。

> 取舍：排序/筛选在内存做，因为 effectivePriority/overdue 是 DTO 层派生，DB 无法直接 ORDER BY。单用户规模可接受；若规模增长，需在 DB 层物化这些派生字段。

### 8.3 createTask / updateTask
- 校验 → 写 task → 同步标签（`syncTags` 全删重建 task_tags，tags 用 upsert 归一）→ 回读 getTask 返回完整 DTO。
- `sortOrder` 新建时用 `BigInt(Date.now())`。

### 8.4 createExecution
- 校验完成日期格式。
- 一次性任务：若已存在有效 execution → 409。
- 周期任务：无重复限制，允许同日多条。
- 写入 completedDate/completedAt/executedAt/completionNote。

### 8.5 getCalendar 聚合
1. 算 week/month 区间 [from,to]（周以周一为起）。
2. 查区间内有效 executions → 按完成日分桶为 `completion` 项。
3. 查全部未删除任务：
   - 周期任务 → `listOccurrences` 补未执行的 `pending` 格子（标 `recurring:true`）。
   - 一次性任务 → dueDate 落在区间且未完成 → `pending` 项。
4. 返回 `{date, range, days:[{date, items}]}`。

### 8.6 日期处理约定（dates 模块）
- 存储 UTC，展示本地日期。
- `dateUTC(date)` / `todayLocal()` / `toDateOnlyISO(yyyy-mm-dd)` / `parseDateOnly` / `localDateTime` / `addDays` / `isValidDateOnly`。
- 对外契约：日期一律 `YYYY-MM-DD` 本地日期。

---

## 9. 前端架构

### 9.1 渲染模型
全客户端渲染，无 RSC 业务页。`app/page.tsx` 直接渲染主容器组件；`layout.tsx` 挂 ThemeProvider。

### 9.2 状态管理
**不引入任何状态管理库**，主容器组件用 `useState` + `useCallback` + `useEffect` 统管。状态分三组：
1. **列表态**：tasks / tags / stats / calData。
2. **过滤器态**：status / tagFilter / search(含 300ms debounce) / sortBy / sortDir。
3. **对话框态**：formTask / completeTarget / executionsTask / deleteTarget / dayDetail / customizerOpen。

数据流：
```
用户操作 → apiXxx() → 成功后 refresh() → 并发重拉 tasks/tags/stats/calendar
```

### 9.3 全分页拉取
因排序/筛选依赖后端 DTO 派生字段，DB 分页无意义，前端 `fetchAllTasks` 循环拉满所有页（每页 100）后在客户端持有全量。

### 9.4 fetch 封装
薄封装 `api<T>`：统一 JSON 头、解析错误体 `message`、抛 `ApiError(status, message)`。导出 `apiGet/apiPost/apiPut/apiDelete`。

> 注意：前端 `ApiError` 与后端 `AppError` 是两套独立类型，靠 HTTP 体的 `message` 字段对齐语义。

### 9.5 组件清单
| 组件 | 职责 |
|---|---|
| 主容器 | 状态中枢 + 布局（左侧栏统计/标签/排序/外观 + 主栏搜索/工具栏/列表或日历） |
| 任务条目 | 单项展示 + 完成切换/编辑/删除/执行记录入口 |
| 任务表单对话框 | 新建/编辑，含标题/优先级/截止/描述/标签/周期规则 |
| 完成对话框 | 完成/补录，区分「补充」（补录指定日期）与「新建」（再记一条） |
| 执行记录对话框 | 历史列表 + 撤销 + 追加记录 |
| 日历视图 | 月/周切换、日期计算、格子点击进详情 |
| 通用 UI | Modal 遮罩/ESC/宽度、按钮/输入样式常量 |
| 主题 Provider | 上下文 + 默认主题加载 + 选择持久化 |
| 主题定制对话框 | 可视化编辑 CSS 变量 |

### 9.6 键盘快捷键
- `N` → 新建任务。
- `/` → 聚焦搜索框。
（输入框/文本域内不触发。）

### 9.7 响应式
- 桌面：左侧 220px 固定栏 + 主栏 780px。
- 移动：隐藏侧栏，标签横滚 chip + 顶栏主题选择。

---

## 10. 主题系统

- CSS 变量驱动（`--background`/`--foreground`/`--card`/`--primary`/`--ring`/`--success`/`--danger` 等）。
- 预设：亮色 / 暗色。
- 自定义：用户可命名保存自定义主题，选择项编码为 `custom:{id}`。
- 持久化：选择与自定义主题存本地，ThemeProvider 启动时加载并应用。
- 主题切换不触达业务态，纯外观层。

---

## 11. 工程配置

| 文件 | 关键约定 |
|---|---|
| tsconfig.json | `strict` 开启；`jsx: react-jsx`；`@/*` → `./src/*`；target ES2017 |
| vitest.config.ts | node 环境；`include: src/**/*.test.ts`；复用 `@` alias |
| postcss.config.mjs | `@tailwindcss/postcss` 插件（Tailwind v4 无主 config） |
| eslint.config.mjs | Next.js flat config（core-web-vitals + typescript） |
| package.json scripts | `dev`/`build`/`start`/`lint`/`test` |

npm 脚本：
- `npm run dev` → `next dev`（端口 3000）
- `npm run build` → `next build`
- `npm test` → `vitest run`
- `npm run lint` → `eslint`

---

## 12. 运维操作设计（todolist-ops）

本系统对外提供一个全局操作指南 skill `todolist-ops`，作为 AI 代理以 API 而非页面操作本系统的权威入口。其设计约束如下。

### 12.1 skill 元信息
- 名称：`todolist-ops`
- 适用项目：本仓库
- 触发场景：任何涉及本项目的「操作、查询、维护」类请求（查询/筛选/统计/分析任务，新建/更新/删除，完成/补录/撤销，健康检查，读 OpenAPI 契约，跑测试/构建/迁移）。
- 入口原则：**AI 以 REST API 为操作入口，页面仅作参考**。

### 12.2 能力矩阵
| 能力 | 入口 |
|---|---|
| 服务存活/就绪验证 | GET /api/health、/api/health/ready |
| 查询任务（筛选/搜索/排序/分页） | GET /api/tasks |
| 新建/更新/删除任务 | POST /api/tasks、PUT/DELETE /api/tasks/{id} |
| 完成/补录/新建执行记录 | POST /api/tasks/{id}/executions |
| 撤销执行记录 | DELETE /api/tasks/{id}/executions/{execId} |
| 标签聚合 | GET /api/tags |
| 日历数据 | GET /api/calendar |
| 运行单元测试 | npm test |
| 旧库→新库数据迁移 | npx tsx scripts/migrate.ts（--dry-run 预演） |

### 12.3 操作纪律（AI 应该怎么做）
1. **以契约为准**：调用前先读 `public/openapi.yaml` 或 `references/api.md`，不凭记忆构造参数。
2. **先查后改**：写操作前先 GET 确认任务存在与当前状态；查询用分页，不无界拉全量。
3. **按错误码处理**：错误遵循 RFC 9457；400 修正参数、404 停止、409 按语义处理，不盲目重试。
4. **日期一律 YYYY-MM-DD**（本地日期）；dueFrom/dueTo 只匹配有截止日期的任务。
5. **状态语义**：
   - 一次性任务完成 = 创建一条 executions；已完成后重复完成 → 409 `TASK_ALREADY_COMPLETED`。
   - 周期任务永远存在；对已完成周期任务勾选时区分「补充」（补录指定日期）与「新建」（再记一条），不直接撤销。
   - 撤销 = 软撤销（revoked=true），历史保留；重复撤销 → 409 `EXECUTION_ALREADY_REVOKED`。
   - 删除 = 软删除（deletedAt），默认查询不可见。
   - 逾期：一次性任务逾期 effectivePriority 自动升一级；周期逾期以 recurringOverdue 单独标记。
6. **改代码纪律**：先读仓库根 AGENTS.md（Next.js 16 与训练数据不同，写码前查 `node_modules/next/dist/docs/` 对应指南）；改完必须跑 `npm test` 与 `npm run build` 验证。

### 12.4 skill 资产结构
```
todolist-ops/
├── SKILL.md              # 入口指南
└── references/
    ├── api.md            # 全部接口参数/响应/示例
    └── schema.md         # 数据模型与业务规则
```
契约以仓库 `public/openapi.yaml` 为权威，references 为辅助速查。

---

## 13. 部署架构（本机生产）

### 13.1 拓扑
```
外部请求 → nginx :5030（对外入口）
              ↓ 反代
        Next.js 生产服务 :5031（next start，运行 .next 构建产物）
              ↓
        MariaDB/MySQL（本机单独管理）
```

- 生产跑的是 `.next` 构建产物，**不是源码**。
- MySQL 由本机单独管理，不在 Next.js 进程内。

### 13.2 启动与自启
- `scripts/start-prod.ps1`：幂等启动。5030 未开则拉起 nginx，5031 未开则拉起 Next.js。
- `scripts/start-todolist.bat`：登录自启，复制到 `shell:startup`。

### 13.3 日志
- `D:\tmp\todolist-v2\next.out.log` / `next.err.log`。

### 13.4 环境变量
- `DATABASE_URL`：MySQL 连接串（PrismaMariaDb adapter 消费）。
- 开发态 PrismaClient 挂 `globalThis` 防热重载重复实例化。

---

## 14. 错误处理协议（RFC 9457）

### 14.1 错误类
- `AppError(status, code, message)`：业务错误基类。
- `handleError(err)`：统一兜底，输出 `application/problem+json`。

### 14.2 错误体格式
```json
{
  "type": "about:blank",
  "title": "Not Found",
  "status": 404,
  "code": "TASK_NOT_FOUND",
  "message": "任务不存在"
}
```
`Content-Type: application/problem+json`。

### 14.3 错误码表
| code | 状态 | 含义 |
|---|---|---|
| VALIDATION_ERROR | 400 | 参数/输入不合法 |
| TASK_NOT_FOUND | 404 | 任务不存在或已删除 |
| EXECUTION_NOT_FOUND | 404 | 执行记录不存在 |
| TASK_ALREADY_COMPLETED | 409 | 一次性任务重复完成 |
| EXECUTION_ALREADY_REVOKED | 409 | 执行记录已撤销 |
| INTERNAL_ERROR | 500 | 服务器内部错误 |

未知非 AppError 异常 → 500 INTERNAL_ERROR，并 `console.error` 记录。

---

## 15. 测试策略

### 15.1 范围
`npm test` = `vitest run`，覆盖纯逻辑层 5 个模块 54 例：
- task-logic（逾期/排序/筛选/校验）
- cycle-engine（周期触发）
- dates（UTC/本地日期工具）
- theme-engine（主题引擎）
- errors（错误处理）

### 15.2 纪律
- 修改 `src/lib` 下纯逻辑（逾期、周期、日期、筛选、排序、主题）后**必须**跑 `npm test`。
- 改任何代码后跑 `npm run build` 验证构建。
- 纯逻辑层输入用最小接口形状，确保测试无需 mock DB。

### 15.3 设计要求
- 核心业务规则（逾期边界、周期触发、优先级升级、排序分组）必须有对应测试用例。
- 边缘场景：空 executions、同日多条补录、跨月/跨年周期、撤销后重算完成态、软删除不可见。

---

## 16. 数据迁移设计（旧库 → 新库）

### 16.1 场景
从旧库 `todolist` 迁移到新库 `todolist_v2`。

### 16.2 命令
```
npx tsx scripts/migrate.ts            # 执行迁移
npx tsx scripts/migrate.ts --dry-run  # 预演，只统计不写入
```

### 16.3 规则
- 幂等：可重复运行。
- 旧库**只读**，迁移过程不写旧库。
- 周期任务折叠：旧库周期任务不保留周期属性，降级为普通任务 + 历史明细。
- ID 沿用：迁移数据沿用旧 ID。
- 期望规模（以旧库实时为准）：约 52 任务 / 45 执行记录 / 11 标签。

---

## 17. 复现清单

凭本方案复现项目的最小步骤序列：

1. **脚手架**：`pnpm create next-app`（TS + App Router + Tailwind v4），锁定 Next 16.x / React 19.x。
2. **依赖**：加 `prisma @prisma/client @prisma/adapter-mariadb mariadb mysql2`；dev 加 `vitest tsx`。
3. **schema**：按 §5 写 `prisma/schema.prisma`，generator output 到 `src/generated/prisma`，`prisma generate`。
4. **数据层**：写 `src/lib/prisma.ts`（driver-adapter + 全局单例）。
5. **纯逻辑层**：按 §6 实现 `cycle-engine.ts` / `task-logic.ts` / `dates.ts`，输入用最小接口。
6. **服务层**：按 §8 实现 `services.ts`，Model→DTO 转换 + 内存分页 + 日历聚合。
7. **错误层**：按 §14 实现 `errors.ts`。
8. **API 路由**：按 §7 实现 8 个 route handler，参数白名单校验 + handleError。
9. **前端**：按 §9 实现主容器 + 子组件 + fetch 封装，全 client。
10. **主题**：按 §10 实现 ThemeProvider + CSS 变量。
11. **契约**：写 `public/openapi.yaml` 并暴露 `GET /openapi.yaml`。
12. **配置**：tsconfig(strict + @/*)、vitest(node + src/**/*.test.ts)、postcss(tailwind v4)、eslint(flat)。
13. **测试**：按 §15 补 5 模块 54 例单测。
14. **迁移**：按 §16 写 `scripts/migrate.ts`。
15. **部署**：按 §13 写 `scripts/start-prod.ps1` + `start-todolist.bat`，配 nginx :5030 → :5031。
16. **运维 skill**：按 §12 写 `todolist-ops/SKILL.md` + `references/api.md` + `references/schema.md`。
17. **验证**：`npm test` + `npm run build` 全绿；`/api/health` 与 `/api/health/ready` 双探针 200。

---

## 18. 设计取舍备忘

| 取舍 | 理由 | 风险/演进点 |
|---|---|---|
| 内存分页 + 前端全量拉取 | effectivePriority/overdue 是 DTO 派生，DB 无法 ORDER BY | 规模增长需物化派生字段到 DB |
| 全 client 渲染，不用 RSC | 首页是登录后场景，无需 SEO | 若需 SEO/流式需重构 |
| `syncTags` 全删重建 | 简单正确，单用户无写放大风险 | 量大可改 diff 同步 |
| Prisma driver-adapter | 类型安全 + 接 MariaDb | 锁定 Prisma 7.x 语义 |
| 执行记录表达完成态 | 天然支持撤销/补录/重复完成 | 查询需 join，已加索引 |
| 软删除两级 | 历史可追溯 | 默认查询须带 deletedAt:null |
| 无状态管理库 | 单容器 useState 足够 | 组件膨胀后可抽 Context/Zustand |
| BigInt 排序键 | 避 JS number 精度 | 序列化须转 Number |
