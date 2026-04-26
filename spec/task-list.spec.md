# task-list.spec.md — 任务列表页规格

> 业务进入系统后的主页 / 中心枢纽。展示当前用户能看到的全部生成任务,支持筛选、搜索、分页,以及向新建/详情页跳转。

**前置依赖:** [common.spec.md](common.spec.md)
**对应原型:** [task-list.html](../task-list.html)
**路由:** `/task-list.html`
**鉴权:** 需要(token 失效跳 login)

---

## 1. 页面定位

- 业务登录后的**默认落地页**
- 系统的**中心枢纽**:从这里跳到新建任务、任务详情、退出登录
- 显示**当前账号能看到的全部任务**(一期不区分权限,所有 `operator` 看到全部任务;二期可按 owner / 部门过滤)
- 支持业务高频操作:筛选(快速 chip / 多维度过滤 / 时间范围)、搜索、分页

---

## 2. 数据模型

### 2.1 Task(生成任务)

```typescript
interface Task {
  id: string;                       // 主键,'T20260425_000007',按 common.spec.md 第 2 节生成
  shop: string;                     // 店铺,自由文本,如 'Amazon-Z01097-US'
  asin: string;                     // ASIN,自由文本,如 'B08DNRKJX7'
  duration: number;                 // 视频总时长,秒,5 / 10 / 15 / ... / 60(5 的整数倍)
  segCount: number;                 // 段数 = duration / 5
  scriptModel: string;              // 脚本生成模型,'GPT-5.4' / 'Seed 2.0'
  styleTemplate: string;            // 风格模板,'aggressive' / 'balanced' / 'conservative'
  videoModel: string;               // 视频生成模型,'Seedance 2.0' / 'HappyHorse 1.0'
  createdBy: string;                // 创建者 username
  createdAt: string;                // ISO 8601
  updatedAt: string;                // ISO 8601
}
```

### 2.2 TaskListItem(列表项 — 响应给前端的视图模型)

聚合了 Task 本身 + 所有 SubTask 的状态摘要,用于列表渲染:

```typescript
interface TaskListItem {
  // ============ 来自 Task 表 ============
  id: string;
  shop: string;
  asin: string;
  duration: number;
  segCount: number;
  scriptModel: string;
  styleTemplateLabel: string;       // '激进版' / '平衡版' / '保守版'(已转中文)
  videoModel: string;
  createdAt: string;
  updatedAt: string;

  // ============ 创建人(权限相关) ============
  createdBy: string;                 // 创建者 username,如 'wang.yunying'
  createdByDisplayName: string;      // 创建者中文名,如 '王运营',前端展示用
  isMine: boolean;                   // 当前登录用户是否为创建者(后端按 token 判断后返回,免得前端再算)

  // ============ 来自 SubTask 聚合 ============
  segments: TaskListSegStatus[];    // 长度 = segCount,按段顺序

  // ============ 衍生字段(后端聚合计算) ============
  todoCount: number;                // 待办数量(待审核 + 失败)— 用于"需要处理" chip 计数
  pendingReviewCount: number;       // 待审核段数 — 用于"需要审核" chip 计数
  generatingCount: number;          // 生成中段数 — 用于"生成中" chip 计数
  hasFailed: boolean;               // 是否存在失败段
  hasRegenerating: boolean;         // 是否存在重做中段
}

interface TaskListSegStatus {
  seg: number;                      // 段号,1-indexed
  segRange: string;                 // '0-5s' / '5-10s' ...
  // 当前主版本(最新版本)的状态
  currentStatus: SegStatus;
  currentVer: number;               // 当前主版本号 v
  // 用于 tooltip 展示历史版本
  versionHistory: { v: number; status: SegStatus; }[];
}

type SegStatus =
  | 'generating'      // 生成中(首次或重做后,未审核)
  | 'pending'         // 待审核(已生成,未审核)
  | 'satisfied'       // 满意(已审核)
  | 'usable'          // 可用(已审核)
  | 'unusable'        // 不可用(已审核)
  | 'regenerating'    // 重做中(已审核段又触发了重做)
  | 'failed';         // 失败(生成调用失败)
```

**关键说明:**

- 一个段对应多个 SubTask(每个 SubTask 是一个版本)。`currentStatus` 取**最新一个 SubTask** 的状态
- `versionHistory` 用于 tooltip 显示完整历史
- 任务级**不**维护单一总状态,前端通过 `segments` 数组聚合判断显示

### 2.3 段状态颜色对应

| 状态 | 颜色 | 视觉(原型 `.seg-dot.xxx` 类) |
|---|---|---|
| `pending` | 灰色空心 | `.seg-dot.pending` |
| `generating` | 紫色转圈 | `.seg-dot.generating` |
| `regenerating` | 紫色转圈 | `.seg-dot.regenerating` |
| `satisfied` | 绿色实心 | `.seg-dot.satisfied` |
| `usable` | 蓝色实心 | `.seg-dot.usable` |
| `unusable` | 红色实心 | `.seg-dot.unusable` |
| `failed` | 红色边框带感叹号 | `.seg-dot.failed` |

---

## 3. UI 结构

### 3.1 顶部栏(共用 shell,见各页面共有)

- Logo / 标题
- **每日额度 chip:** `今日额度 120 / 600s` `Seedance 2.0` + 刷新按钮
- 用户菜单(点击展开退出登录)

额度 chip 展示当前用户**当日已用 / 总限额 + 模型名**。点击刷新按钮 → 重新调 `GET /api/quota`。

### 3.2 主区域(从上到下)

#### 3.2.1 标题行

```
任务管理                                        [+ 新建生成任务]
```

新建按钮主色,点击跳 `/task-create.html`(同 tab 跳转,**不**新窗口)。

#### 3.2.2 快速筛选 chip 行(可多选叠加,点击立即生效)

3 个 chip,从左到右:

| chip | 颜色 | 数字含义 | tooltip |
|---|---|---|---|
| `生成中` | 紫色 | 该用户名下存在"生成中"段的任务数 | 多行说明 |
| `需要审核` | 绿色 | 该用户名下存在"待审核"段的任务数 | 多行说明 |
| `需要处理` | 绿色 | 待审核 + 失败段的任务数(交集) | 多行说明 |

**交互:**

- 点击 chip → 切换 active 态,**立即触发列表刷新**(无需"查询"按钮)
- 多个 chip 同时选中时,效果是"交集"(满足所有勾选条件的任务)
- chip 文案 / 数字由后端 `GET /api/tasks/quick-stats` 返回,前端不本地计算

#### 3.2.3 多维度筛选行

- **创建人** select:`全部 / 我创建的 / [其他人姓名列表]`,默认"全部"
  - "我创建的"对应当前登录用户
  - 其他选项动态从后端获取(全公司有任务的所有创建人列表)
  - 一期所有用户互相可见,选项里展示真名(取自 User.displayName)
- **店铺** input:自由文本,精确匹配 `shop` 字段(一期为简单 LIKE 搜索)
- **TaskID 或 ASIN 搜索** input:支持 `id` 或 `asin` 模糊匹配
- **视频时长** select:全部 / 5s / 10s / ... / 60s
- **视频生成模型** select:全部 / Seedance 2.0 / HappyHorse 1.0(二期模型在 select 里 disabled)
- **时间筛选器**(时间维度 + 快捷选项 + 自定义范围,详见 3.2.4)
- **[查询]** 主色按钮:点击触发查询
- **[重置]** 中性按钮:清空所有筛选条件回到默认
- 右侧:`共 X 个任务(近 30 天),按更新时间倒序`(由后端返回总数 + 当前排序规则)

#### 3.2.4 时间筛选器(基于 `<details>` 折叠组件)

**Summary(收起态):**

```
📅 时间 [创建]  [最近 30 天]   ▼
```

- "创建/更新":点击切换时间维度 Tab(创建时间 / 更新时间)
- "最近 30 天":显示当前选择的时间范围
- 默认时间范围:**最近 30 天**(基于"创建时间")

**展开 Panel:**

3 个 section,从上到下:

1. **时间维度** Tab 二选一:`创建时间`(默认) / `更新时间`
2. **快捷选项**(单选互斥):`最近 7 天` / `最近 30 天`(默认) / `本周` / `本月`
3. **自定义范围**:开始日期 + 结束日期,两个 `<input type="date">`。选择后自动取消快捷选项的 active 态

**关闭逻辑:** 点击外部任意位置关闭 panel(基于 `<details>` + 全局 click 监听)。

### 3.3 任务列表(行卡片)

每行一个任务,从左到右 4 部分:

```
┌─ 任务卡 ────────────────────────────────────────────────────┐
│ TaskID  [👤 创建人姓名 (· 我)]                                  │
│ 店铺 · ASIN · 时长·段数 · 脚本模型·风格 · 视频模型              │
│ [段状态点 1][段状态点 2]...[段状态点 N]                         │
│                                                                  │
│                              更新 2026-04-25 15:08:42  [审核 / 查看] │
│                              创建 2026-04-25 15:06:18              │
└────────────────────────────────────────────────────────────────┘
```

**字段顺序(左侧 task-info):**

1. TaskID(font-mono,稍大)+ **创建人徽章**(右侧紧贴):
   - 自己创建:绿色徽章 `👤 王运营 · 我`(`.task-creator.me` 类,brand-bg 底色 + brand 字色)
   - 同事创建:灰色徽章 `👤 张燕霞`(`.task-creator` 默认类,bg-active 底色 + text-3 字色)
2. meta 行:`店铺 · ASIN · 时长 · 段数 · 脚本模型·风格 · 视频生成模型`(中点分隔)
3. 段状态点行:每段一个圆点,hover 显示 tooltip 说明 `第 N 段 · 时间段 · vN 状态(当前)·历史:vN 状态, vN 状态`

**字段顺序(右侧 task-actions):**

1. 时间块(更新在上,创建在下,精确到秒):
   ```
   更新 2026-04-25 15:08:42
   创建 2026-04-25 15:06:18
   ```
2. 操作按钮(根据 `isMine` 和 `pendingReviewCount` 三选一):
   - 自己创建 + `pendingReviewCount > 0` → `[审核]` 主色按钮
   - 自己创建 + `pendingReviewCount === 0` → `[查看]` 中性按钮
   - **同事创建(无论状态) → `[查看]` 中性按钮**(写操作权限只属于创建者)
3. 按钮 onclick 跳 `/task-detail.html?taskId={id}`(新 tab)

**视觉细节(已在原型 CSS 中定义):**

- 待办任务(`todoCount > 0`)左侧加 4px 绿色边条(`.task-row.has-todo`)
  - 注意:同事的任务即使有 `todoCount > 0`,也加边条(代表"该任务有待处理段"是客观状态),但**当前用户无法操作**
- 鼠标悬停整行有 hover 高亮

### 3.4 分页器

```
共 23 条 · 每页 [20] [<] [1] 2 [>] · 跳至 [__] 页
```

**字段:**

- 总数:`共 X 条`
- 每页大小:select,选项 `10 / 20 / 50`,默认 `20`
- 翻页按钮:`<` `>` 上一页/下一页,以及 `1 2 3 ...` 页码按钮
- 跳页:input + Enter 跳转

**API 行为:**

- 切页大小 → 重置到第 1 页 + 重新查询
- 翻页 → 当前筛选条件不变,只换 `page` 参数

---

## 4. 交互逻辑

### 4.1 筛选触发查询的时机

| 操作 | 立即触发查询 | 需点"查询" |
|---|---|---|
| 快速 chip 点击 | ✓ |  |
| 时间筛选器选择 |  | ✓ |
| 输入框输入 |  | ✓ |
| 下拉选择 |  | ✓ |
| 翻页 / 切页大小 | ✓ |  |
| 重置按钮 | ✓(清空 + 查询) |  |

理由:chip 是高频快速过滤,要立即响应;其他多维度筛选用户可能调整多个条件,集中点查询更高效。

### 4.2 重置按钮

清空所有筛选:
- 快速 chip 全部取消 active
- 输入框清空
- select 回到"全部"
- 时间筛选器回到"创建时间 + 最近 30 天"
- 然后立即触发查询

### 4.3 行操作按钮

- `[审核]` / `[查看]` 跳 `/task-detail.html?taskId={id}`,**新 tab 打开**(原型 `target="_blank"`)
- 业务可以同时打开多个详情页,在 tab 间切换审核多个任务

### 4.4 用户菜单

详见 common.spec.md 第 4 节(全局共用 shell)。

---

## 5. 状态机

无独立状态机。任务级状态由各 `SubTask` 状态聚合决定,详见 `task-detail.spec.md` 第 5 节。

---

## 6. API 契约

### 6.1 任务列表查询

**Endpoint:** `GET /api/tasks`

**鉴权:** 需要

**Query Params:**

```typescript
interface TaskListQuery {
  // 分页
  page?: number;                    // 默认 1
  pageSize?: number;                // 默认 20,可选 10 / 20 / 50

  // 排序
  sortBy?: 'createdAt' | 'updatedAt';   // 默认 'updatedAt'
  sortOrder?: 'asc' | 'desc';            // 默认 'desc'

  // 快速 chip(可多选,逻辑为交集)
  quickFilters?: ('generating' | 'pendingReview' | 'todo')[];

  // 多维度过滤
  shop?: string;                    // 店铺 LIKE 搜索
  createdBy?: string | 'me';        // 创建人筛选,传 'me' 表示当前用户,传 username 表示指定人
  keyword?: string;                 // TaskID 或 ASIN 模糊匹配
  duration?: number;                // 视频时长(秒),不传或 null 表示全部
  videoModel?: string;              // 视频生成模型,不传表示全部

  // 时间过滤
  timeField?: 'createdAt' | 'updatedAt';   // 默认 'createdAt'
  timeFrom?: string;                // ISO 8601,默认 30 天前
  timeTo?: string;                  // ISO 8601,默认现在
}
```

**Response:**

```typescript
{
  code: 0,
  data: {
    total: number;
    page: number;
    pageSize: number;
    list: TaskListItem[];          // 见第 2.2 节
  }
}
```

### 6.2 快速筛选统计

**Endpoint:** `GET /api/tasks/quick-stats`

**鉴权:** 需要

**用途:** 顶部 3 个 chip 的数字徽章。**与列表查询独立**(因为筛选条件改变时,chip 数字应该展示当前用户的全局统计,而不是筛选后)。

**Query Params:** 无(始终基于当前用户全部任务)

**Response:**

```typescript
{
  code: 0,
  data: {
    generating: number;             // 存在生成中段的任务数
    pendingReview: number;          // 存在待审核段的任务数
    todo: number;                   // 存在待审核或失败段的任务数(交集)
  }
}
```

### 6.3 当日额度

**Endpoint:** `GET /api/quota`

**鉴权:** 需要

**Response:**

```typescript
{
  code: 0,
  data: {
    items: [
      {
        model: string;              // 'Seedance 2.0'
        unit: 'seconds';
        dailyLimit: number;         // 600
        used: number;               // 120
        preempted: number;          // 25(已预占未结算)
      },
      // 一期视频生成模型只有一个,后续可扩展 LLM 模型额度
    ]
  }
}
```

前端取 `items[0]` 展示在顶部 chip。

### 6.4 创建人列表(用于筛选下拉)

**Endpoint:** `GET /api/tasks/creators`

**鉴权:** 需要

**用途:** 任务列表页"创建人"筛选下拉里的"其他人"选项,展示**有任务的所有用户**(去重)。

**Response:**

```typescript
{
  code: 0,
  data: {
    list: [
      { username: 'wang.yunying', displayName: '王运营', taskCount: 12 },
      { username: 'zhang.yanxia', displayName: '张燕霞', taskCount: 8 },
      // ...
    ]
  }
}
```

**前端处理:**

- 下拉里把 "我创建的" 永远放第一,然后显示其他人(按 displayName 字母序或 taskCount 倒序)
- 当前用户如果出现在 list 里,前端去重(因为已经有"我创建的"选项了)

---

## 7. 边界 & 错误处理

### 7.1 没有任何任务

后端返回 `{ total: 0, list: [] }`,前端显示空状态:

```
[图标] 还没有生成任务
       [+ 创建第一个任务]
```

按钮跳 `/task-create.html`。

### 7.2 筛选后无结果

后端返回 `{ total: 0, list: [] }`,前端显示:

```
没有匹配的任务,试试调整筛选条件 [重置]
```

### 7.3 网络异常

整个列表区域显示重试占位:

```
[图标] 加载失败,请检查网络 [重试]
```

### 7.4 长时间生成中的任务

用户看到 chip 显示"生成中:1",但实际是某个任务卡了 1 小时还没出结果。这种情况:

- 一期不主动报警,业务点击进入任务详情可看到具体哪段,自行联系研发处理
- 二期:后端定时扫描"超过 30 分钟还在 generating"的 SubTask,标记为 failed,触发返还额度

### 7.5 段数量过多导致段状态点过宽

最大 12 段(60s 任务),原型 CSS 已处理换行。研发实现时无需特殊处理。

---

## 8. 非功能性要求

- **响应时间:** 列表查询 P99 ≤ 800ms(20 条数据 + 段状态聚合)
- **缓存策略:** 不缓存列表(状态实时性要求高)。chip 统计可短缓存 30 秒
- **轮询:** 一期不轮询(用户主动刷新即可)。二期可考虑列表页 30 秒轮询一次

---

## 9. 开发者注意事项(原型 vs 真实)

### 9.1 原型任务数据写死

原型有 8 行硬编码任务,真实开发应:

```javascript
// 原型(写死 HTML)
<div class="task-row has-todo">...</div>
<div class="task-row">...</div>

// 真实开发
async function loadTasks() {
  const res = await api.get('/api/tasks', { params: query });
  setTasks(res.data.list);
  setTotal(res.data.total);
}
```

### 9.2 chip 数字必须独立 API

不要在前端把 `tasks` 数组本地 reduce 出 chip 数字 — 因为列表是分页的,本地数据不全。必须从 `/api/tasks/quick-stats` 拿。

### 9.3 段状态点的 tooltip 内容

原型在 HTML 里写死 `data-tip="第 1 段 · 0–5s · v1 待审核(当前)"`。真实开发应在 React/Vue 渲染时根据 `segments[i].versionHistory` 动态拼接。

### 9.4 时间筛选器的"本周/本月"边界

- "本周":周一 00:00:00 到 现在(用户所在时区)
- "本月":本月 1 日 00:00:00 到 现在
- 后端实现时注意时区转换

### 9.5 排序默认 updatedAt desc

业务最常关注"最近有动作"的任务(包括刚审核完的、刚重做完的),所以默认排 `updatedAt`。SubTask 状态变更要触发 Task.updatedAt 更新。

---

## 10. 涉及的数据库表

| 表名 | 用途 |
|---|---|
| `tasks` | 任务主表 |
| `subtasks` | 段级版本(详见 task-detail.spec.md) |
| `users` | 用户表(common.spec.md) |
| `quotas` | 额度表(common.spec.md) |

**索引建议:**

- `tasks (created_by, updated_at DESC)` — 列表查询主索引
- `tasks (asin)` — 关键词搜索
- `subtasks (task_id, seg, v)` — 状态聚合查询
- `subtasks (task_id, status)` — 任务级状态判断

---

## 11. 待确认

- [x] **多人查看权限:决议方案 B** — 所有用户能看到全部任务,但写操作只属于创建者:
  - 列表 / 详情都展示创建人姓名(自己绿色,同事灰色)
  - 同事的任务统一显示 `[查看]`(无 `[审核]` 按钮)
  - 详情页同事任务的所有写操作按钮(审核 / 重做 / 重试 / 合成)disabled
  - 二期"长视频合成完整模块"做"同商品跨任务合成"时,可基于此基础扩展
- [ ] chip 数字是实时统计还是 30 秒缓存?要看后端聚合查询性能
- [ ] 业务有删除任务的需求吗?一期 spec 中**没有删除接口**,如有需要再补
- [ ] 业务有批量操作需求吗(批量审核 / 批量重新生成)?一期**不做**
- [ ] 列表页是否需要展示某个段的视频缩略图?原型只用了状态点,**没有缩略图**。如果业务想"列表里看到视频效果",可考虑二期加
