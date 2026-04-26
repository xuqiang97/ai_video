# task-detail.spec.md — 任务详情 / 审核页规格

> 业务对单个任务的所有 5 秒视频片段进行逐段审核(满意 / 可用 / 不可用)、单段重做、失败重试,以及合成长视频的核心工作页。

**前置依赖:** [common.spec.md](common.spec.md), [task-list.spec.md](task-list.spec.md)
**对应原型:** [task-detail.html](../task-detail.html)
**路由:** `/task-detail.html?taskId={id}`
**鉴权:** 需要

---

## 1. 页面定位

业务每天 80% 时间在这里:

- **审核** — 逐段打"满意 / 可用 / 不可用"三档结论
- **重做** — 对已审核段触发新一次生成(产生新版本)
- **重试** — 对失败段重新调用视频生成
- **合成长视频** — 选段排序后异步合成下载

写操作权限:**仅创建者**(`task.createdBy === currentUser.username`)。同事打开同一页面只能查看,所有写操作按钮 disabled,详见第 9 节。

---

## 2. 数据模型

### 2.1 Task

详见 [task-list.spec.md](task-list.spec.md) 第 2.1 节。

### 2.2 SubTask(段级版本 — 核心实体)

```typescript
interface SubTask {
  id: string;                       // 'S20260425_000003',按 common.spec.md 第 2 节生成
  taskId: string;                   // 外键 → Task.id
  seg: number;                      // 段号,1-indexed,1..task.segCount
  v: number;                        // 版本号,从 1 开始,同 (taskId, seg) 下递增

  status: SegStatus;                // 见 2.3 节
  prompt: string;                   // 该版本最终发给视频生成模型的提示词

  // 审核结论(仅 status=satisfied/usable/unusable 时有值)
  conclusion: '满意' | '可用' | '不可用' | null;
  reviewNote: string;               // 审核备注,可为空
  reviewedBy: string | null;         // 审核人 username
  reviewedAt: string | null;         // ISO 8601

  // 视频文件信息(仅 status=satisfied/usable/unusable/regenerating(已有视频时显示老版本)时有值)
  videoUrl: string | null;
  videoKey: string | null;          // 对象存储 key
  videoFileSize: number | null;     // 字节数

  // 失败信息(仅 status=failed 时有值)
  errorMessage: string | null;      // 错误原因,展示给业务

  // 时间字段
  createdAt: string;                // ISO 8601
  startedAt: string | null;         // 视频生成开始时间(进入 generating/regenerating 时记录)
  completedAt: string | null;       // 生成完成时间(成功或失败)
  generatedAt: string | null;       // 别名 = completedAt(已生成成功的别名,前端展示用)

  // 衍生字段(后端计算返回,不存表)
  elapsedSec: number | null;        // 生成成功:completedAt - startedAt;生成中:Date.now() - startedAt
}
```

### 2.3 SegStatus 状态枚举

```typescript
type SegStatus =
  | 'generating'      // 首次生成中(任务刚提交,还没出第一个版本)
  | 'pending'         // 待审核(已生成,业务还没打结论)
  | 'satisfied'       // 满意(已审核)
  | 'usable'          // 可用(已审核)
  | 'unusable'        // 不可用(已审核)
  | 'regenerating'    // 重做中(已审核段触发了新一次生成)
  | 'failed';         // 生成失败(API 调用失败 / 模型返回异常 / 超时)
```

### 2.4 Composition(长视频合成版本)

```typescript
interface Composition {
  id: string;                       // 'C20260425_000001'
  taskId: string;                   // 外键 → Task.id
  name: string | null;              // 用户起的版本名,可为空

  // 选段记录(按合成顺序)
  picks: Array<{
    seg: number;                    // 段号
    ver: number;                    // 该段的版本号
    subTaskId: string;              // 对应 SubTask.id
  }>;

  durationSec: number;              // 总时长 = picks.length * 5

  status: 'composing' | 'success' | 'failed';

  // 文件信息(success 后填)
  videoUrl: string | null;
  videoKey: string | null;
  fileSize: string | null;          // 展示用,如 '2.3 MB'(后端格式化好返回)

  // 失败信息(failed 时填)
  errorMessage: string | null;

  // 时间字段
  createdBy: string;
  createdAt: string;
  startedAt: string | null;
  completedAt: string | null;
  composedSec: number | null;       // 合成耗时(秒,success 后填)

  // 衍生字段
  elapsedSec: number | null;        // 合成中:Date.now() - startedAt
}
```

### 2.5 段级状态机

```
               ┌─────────────┐
   首次生成 →  │ generating  │
               └──────┬──────┘
                      ↓ 模型调用成功
               ┌─────────────┐    ┌──────────┐
               │   pending   │ ←──│  failed  │ ←  模型调用失败
               └──────┬──────┘    └────┬─────┘
                      ↓                  │
            打结论 satisfied/usable/    │ 业务点重试
                  unusable               ↓
                      ↓               重新生成 → generating
               ┌──────────────────┐
               │ satisfied/usable │
               │     unusable     │
               └──────┬───────────┘
                      ↓ 业务点重做(产生新版本)
               ┌──────────────┐
               │ regenerating │
               └──────┬───────┘
                      ↓ 模型调用成功
                  pending(新版本独立计算状态机)
                  或 failed(若失败)
```

**关键:**

- **每次重做 / 重试都创建新 SubTask 记录**,旧版本保留(版本号递增)
- 段的"当前主状态"= **最新一个 SubTask 的 status**
- 任务级**不**维护单一总状态,前端展示时按 `task.subtasks` 聚合

### 2.6 任务级状态聚合规则(给前端用)

任务在列表 / 详情顶部展示的统计数据:

```typescript
function aggregateTaskState(subtasks: SubTask[]) {
  // 取每段最新版本
  const latestByseg = groupBy(subtasks, 'seg').mapValues(arr => maxBy(arr, 'v'));

  return {
    total: latestBySeg.length,
    satisfied: count latestByseg.values where status='satisfied',
    usable: count where status='usable',
    unusable: count where status='unusable',
    pending: count where status='pending',
    failed: count where status='failed',
    generating: count where status in ['generating', 'regenerating'],
  };
}
```

---

## 3. 页面布局总览

```
┌─ 顶部 shell(全局共用,用户菜单等) ─────────────────────┐
└──────────────────────────────────────────────────────────┘

┌─ 任务概要条 ───────────────────────────────────────────┐
│ T20260425_000007                  [🎬 合成长视频  ✓2 ⟳1] │
│ 店铺 · ASIN · 时长·段数 · 脚本模型 · 风格 · 视频模型     │
│ · [创建人] · 创建时间 · 更新时间                         │
└──────────────────────────────────────────────────────────┘

┌─ 三栏布局 ──────────────────────────────────────────────┐
│ ┌─ 左 ──────┬─ 中(主审核区) ────────┬─ 右 ─────────┐ │
│ │ 分镜片段  │ 视频播放器(16:9)      │ 商品图文资料 │ │
│ │ [段1 ▼]   │                          │ [商品图轮播] │ │
│ │ [段2 ▼]   │ 版本切换 [v1][v2][v3]   │ 标题/类目    │ │
│ │ [段3 ◉]   │                          │ 尺寸/卖点   │ │
│ │ [段4 ▼]   │ 当前版本结论 / 待审核   │              │ │
│ │ [段5 ▼]   │ / 失败 / 重做中         │              │ │
│ │ [段6 ▼]   │                          │              │ │
│ │           │ 本段最终提示词 [子任务ID]│              │ │
│ └───────────┴──────────────────────────┴──────────────┘ │
└────────────────────────────────────────────────────────┘
```

---

## 4. 顶部任务概要条

### 4.1 字段(从左到右)

```
T20260425_000007                                   [🎬 合成长视频  ✓2 ⟳1]
店铺:Amazon-Z01097-US · ASIN:B08DNRKJX7 · 时长:30s · 6 段
· 脚本模型:GPT-5.4 · 风格:激进版 · 视频模型:Seedance 2.0
· 创建人:王运营·我 · 创建:2026-04-25 14:35:21 · 更新:2026-04-25 14:55:30
```

| 字段 | 内容 | 显示规则 |
|---|---|---|
| TaskID | font-mono,稍大 | 总在最左 |
| 店铺 / ASIN / 时长 · 段数 / 脚本模型 / 风格 / 视频模型 | 来自 Task 字段 | 中点分隔 |
| **创建人** | 徽章(自己绿色 / 同事灰色) | 紧邻"创建"时间左侧 |
| 创建 / 更新 | font-mono,精确到秒 | |
| 合成长视频按钮 | 主色,带状态徽章 | 详见 7.1 节 |

### 4.2 操作按钮

- `[🎬 合成长视频]` 主色按钮 — 点击打开合成 Modal,详见第 7 节
- 按钮文字右侧带 3 类徽章(根据 compositions 状态分布):
  - `✓N`(white 22% 透明 + check 图标)= 已合成成功数
  - `⟳N`(white 半透明 + 旋转 spinner)= 合成中数
  - `⚠N`(red 45% 透明 + alert 图标)= 失败数
- 0 数量的徽章不显示

---

## 5. 中栏:审核主区(根据当前选中段的状态渲染不同 UI)

中栏内容**完全由当前段的"主版本状态"决定**。前端逻辑用 if/else 渲染不同视图。

### 5.1 满意 / 可用 / 不可用(已审核)

```
┌─ 视频播放器(16:9) ──────────────────────┐
│ [模拟视频画面]                              │
└─────────────────────────────────────────────┘

[v1][v2 当前]                                  ← 版本切换 Tab

┌─ 已审核结论卡 ──────────────────────────────┐
│ [✓ 满意 / ⓘ 可用 / ✕ 不可用] (大色卡)        │
│                                                │
│ 备注:画面整体很出彩,首段开场金光感强,...   │
│                                                │
│         [✏ 修改结论]  [↻ 重做本段]            │
└────────────────────────────────────────────┘

┌─ 本段最终提示词 [子任务ID]            v1 生成耗时 8 分 28 秒 · 生成时间 2026-04-25 14:42:18 ┐
│ 【亚马逊美国站点商品卖点介绍视频5s分镜脚本】:│
│ 暖金色光影中,金色陶瓷生肖马摆件...         │
│ 【要求】:虽然分镜脚本是中文,但视频画面中..│
└────────────────────────────────────────────┘
```

**字段说明:**

- 状态色卡:满意=绿,可用=蓝,不可用=红
- 备注:`SubTask.reviewNote`,可能为空(显示"无备注")
- `[✏ 修改结论]` 按钮:进入"修改结论"形态,详见 5.6 节
- `[↻ 重做本段]` 按钮:打开重做 Modal,详见 6.1 节
  - **disabled 条件:** 该段已存在 `generating` / `regenerating` 状态的版本(避免并发重做)

### 5.2 待审核

```
┌─ 视频播放器 ──────────────────────────────┐
│ [模拟视频画面]                              │
└─────────────────────────────────────────────┘

[v1 当前]

┌─ 三档审核 ─────────────────────────────────┐
│ 请打结论:                                   │
│ ┌──────────┬──────────┬──────────┐       │
│ │ ✓ 满意   │ ⓘ 可用   │ ✕ 不可用 │       │  ← 三个大按钮
│ └──────────┴──────────┴──────────┘       │
│                                              │
│ 备注(可选):[textarea]                     │
│                                              │
│                          [⬇ 提交审核]       │
└────────────────────────────────────────────┘

┌─ 本段最终提示词 [子任务ID] ──────────────┐
│ ...                                          │
└────────────────────────────────────────────┘
```

**交互:**

- 三档按钮单选,选中后高亮(对应状态色)
- 备注 textarea 可选,maxLength 500
- `[⬇ 提交审核]` 按钮 — 必须先选一档才启用
- 提交后:
  - 调用 `POST /api/subtasks/{id}/review`
  - 成功 → 5 秒撤回提示条出现(底部),5 秒内可点 [撤回] 还原成待审核状态
  - 段状态切换为 satisfied / usable / unusable

### 5.3 失败

```
┌─ 失败信息卡 ──────────────────────────────┐
│ ⚠ 第 6 段(25–30s)视频生成失败             │
│   v1 调用 Seedance 2.0 失败 · 2026-04-25 14:46:07 │
│                                              │
│ 错误信息:                                   │
│ 视频生成接口超时(>60s 无响应),请重试    │
│                                              │
└─────────────────────────────────────────────┘

┌─ 本段最终提示词(可在重试前编辑)[子任务ID]┐
│ 【...提示词模板...】                        │
│ {内容,只读 - 默认}                         │
└─────────────────────────────────────────────┘

┌─ 额度返还提示 ─────────────────────────────┐
│ ✓ 上次生成预占的 5s 额度已返还              │
└─────────────────────────────────────────────┘

[✏ 编辑提示词后重试]  [↻ 直接重试]
```

**交互:**

- `[直接重试]`:打开重试 Modal,提示词不修改,直接重新生成
- `[编辑提示词后重试]`:打开重试 Modal,提示词区域可编辑
- 重试 Modal 详见 6.2 节
- 重试不消耗新额度(预占已返还,新一次再预占,一致)

**额度文案约定:** 一律使用"返还"或"预占" — **禁止说"免费"**

### 5.4 重做中 / 生成中

```
┌─ 紫色等待区(居中) ────────────────────┐
│            ⟳(大 spinner)                  │
│                                              │
│        第 5 段 v2 生成中                    │
│                                              │
│        已耗时 07m 05s                       │
│                                              │
│        预计 5–15 分钟生成完毕                │
│                                              │
│ 已有 1 个历史版本(点击切到 v1 查看)        │
└────────────────────────────────────────────┘

┌─ 本次重做使用的提示词 [子任务ID] ───────┐
│ 【...提示词模板...】                        │
│ {内容,只读}                                 │
└─────────────────────────────────────────────┘
```

**前端轮询:**

- 每 5–10 秒调 `GET /api/subtasks/{subTaskId}` 拿最新状态
- 当状态从 `generating` / `regenerating` 变为 `pending` / `failed` 时,刷新整段渲染
- 已耗时由前端实时计算 `Date.now() - startedAt`,每秒滚动

### 5.5 段切换(版本 Tab)

每段下面横向 Tab,展示**所有版本**:

```
[v1 不可用] [v2 可用] [v3 满意 当前] [v4 重做中]
```

- 每个 Tab 颜色对应该版本状态(满意=绿、可用=蓝、不可用=红、重做中=紫)
- 点击 Tab 切换到对应版本(中栏整体重新渲染)
- "当前"标识 = 最新版本(`v` 最大那个)。但用户可点击切到老版本查看
- **禁止操作的版本**(generating / regenerating)切换后中栏显示等待区

### 5.6 修改结论形态

点击 `[✏ 修改结论]` 后,中栏切换到**编辑形态**(类似 5.2 节但回显之前结论 + 备注):

```
┌─ 修改结论 ──────────────────────────────┐
│ 当前结论:                                 │
│ [✓ 满意 ◉][ⓘ 可用 ○][✕ 不可用 ○]         │ ← 回显之前选择
│                                            │
│ 备注:[textarea,回显之前内容]            │
│                                            │
│           [取消]  [💾 保存修改]           │
└─────────────────────────────────────────┘
```

- "保存修改"按钮文案要明确(不是"提交审核",避免业务误以为新一次审核)
- 保存:调用 `POST /api/subtasks/{id}/review`(同一个接口,后端识别是修改),回到正常已审核态
- 取消:直接回到 5.1 形态,不发请求
- **修改结论永远启用**(独立于该段是否有 generating/regenerating 版本)— 业务可能想边等重做边改老版本结论

### 5.7 提示词包装组件

所有"本段最终提示词" / "本次重做使用的提示词"统一用一个 wrap 组件:

```html
<div class="prompt-template-wrap">
  <div class="prompt-template-fixed">【亚马逊美国站点商品卖点介绍视频5s分镜脚本】:</div>
  <div class="prompt-template-readonly">{prompt}</div>
  <div class="prompt-template-fixed">【要求】:虽然分镜脚本是中文,但...</div>
</div>
```

- 灰底卡片 + 绿色"分镜脚本"标识 + 中间内容块 + 灰色"要求"标识
- **只读场景**用 div(中栏所有场景)
- **可编辑场景**用 textarea(重做 / 重试 Modal)

---

## 6. 重做 / 重试 Modal(共用一个组件)

### 6.1 重做 Modal

**触发:** 点 `[↻ 重做本段]` 按钮

```
┌─ 重做第 N 段(将生成 vK) ─────────────[×]┐
│                                              │
│ ⓘ 重做将创建新版本 vK                       │
│   预占 5s 视频生成额度,生成失败会自动返还  │
│                                              │
│ 提示词(可编辑):                          │
│ ┌───────────────────────────────────────┐ │
│ │ 【...】                                  │ │
│ │ [textarea 当前提示词,可编辑]            │ │
│ │ 【要求】:...                            │ │
│ └───────────────────────────────────────┘ │
│                                              │
│                       [取消]  [⬇ 提交重做]  │
└────────────────────────────────────────────┘
```

### 6.2 重试 Modal(失败段重试)

**触发:** 点 `[↻ 直接重试]` 或 `[✏ 编辑提示词后重试]`

```
┌─ 编辑提示词后重试第 N 段 ─────────────[×]┐
│                                              │
│ ⓘ 上次失败 5s 已返还,本次重新预占          │
│   失败仍返还,成功扣除                      │
│                                              │
│ 提示词(可编辑):                          │
│ ┌───────────────────────────────────────┐ │
│ │ ...                                      │ │
│ └───────────────────────────────────────┘ │
│                                              │
│                       [取消]  [⬇ 提交重试]  │
└────────────────────────────────────────────┘
```

### 6.3 共用实现细节

- 同一个 `<div id="redo-modal">`,通过 `mode` 参数(`'redo'` / `'retry'`)切换文案
- 提示词初始值 = 当前版本的 `prompt`
- 提交按钮 disabled 条件:`prompt.trim().length === 0`
- 提交流程:
  - 调用 `POST /api/subtasks/{id}/redo` 或 `/api/subtasks/{id}/retry`
  - 成功 → 关闭 Modal,刷新该段为重做中 / 生成中状态
  - 失败 → toast 错误,留在 Modal

---

## 7. 长视频合成模块

### 7.1 入口按钮(顶部概要条右上角)

```
[🎬 合成长视频  ✓2 ⟳1 ⚠1]
```

- 主色按钮
- 右侧多状态徽章组(`compose-badges`),根据 compositions 数组聚合显示:
  - `✓N` 合成成功(白色 22% 透明,check 图标)
  - `⟳N` 合成中(白色半透明 + mini-spinner 旋转)
  - `⚠N` 合成失败(红色 45% 透明,alert 图标)
- 0 数量徽章不显示
- 按钮永远启用(即使 0 段已审核也能点开 Modal 看说明)

### 7.2 合成 Modal

**布局(680px 宽,90vh 高):**

```
┌─ 🎬 合成长视频 ─────────────────────[×]┐
│                                            │
│ ⓘ 一期简易版本                             │
│   目前只支持在当前任务里勾选已审核为「满意」│
│   或「可用」的版本合成。                   │
│   同段多个已审核版本可同时选(支持复用、扩展时长)│
│   顺序自由调。二期将支持同商品跨任务合成、 │
│   配字幕、配音频。                         │
│                                            │
│ 选择视频片段并调整顺序                     │
│ (同段多个已审核版本都可勾选,顺序可调)   │
│       [⚡ 快速勾选最新已审]  [🧹 清空]    │
│                                            │
│ ┌─ 段+版本树形选择列表 ─────────────────┐│
│ │ 第 1 段 0–5s                          ││  ← 段标题(灰底,粘性)
│ │   ☑ v1 [满意]                  (1)[↑↓]││  ← 版本行
│ │ 第 2 段 5–10s                          ││
│ │   ☑ v1 [可用]                  (2)[↑↓]││
│ │ 第 3 段 10–15s                         ││
│ │   ☐ v1 [不可用]                (灰显) ││
│ │   ☐ v2 [可用]                          ││
│ │   ☑ v3 [满意]                  (3)[↑↓]││
│ │   ☐ v4 [生成中]                (灰显) ││
│ │ 第 4 段 15–20s                         ││
│ │   ☐ v1 [待审核]                (灰显) ││
│ │ ...                                    ││
│ └────────────────────────────────────────┘│
│                                            │
│ ┌─ 已选统计 ─────────────────────────────┐│
│ │ 已选 3 段 · 总时长 15 秒  合成后视频时长 15 秒│
│ │ ─────────────────────────              ││
│ │ 拼接顺序: (1) 第1段 v1 → (2) 第2段 v1   ││
│ │           → (3) 第3段 v3                ││
│ └────────────────────────────────────────┘│
│                                            │
│ 版本名(可选):[__________________]      │
│                                            │
│ 已合成版本(2 个)                          │
│ ┌────────────────────────────────────────┐│
│ │ ✓  默认全段版 [合成成功]                ││
│ │    第1段 v1 + 第2段 v1 · 10s            ││
│ │    C20260425_000001 · 6 分 18 秒 · 2.3 MB ││
│ │                          [👁 预览][⬇下载]││
│ │                                          ││
│ │ ⟳  简版试合成 [合成中]                  ││
│ │    第1段 v1 · 5s                        ││
│ │    C20260425_000002 · 已耗时 00m 23s     ││  ← 实时滚动
│ └────────────────────────────────────────┘│
│                                            │
│              [关闭]  [⮕ 提交合成任务]     │
└──────────────────────────────────────────┘
```

### 7.3 段+版本选择列表

**渲染规则:**

- 6 段(假设 30s 任务)从上到下展示
- 每段一个**段标题行**(灰底,左侧不可勾选,显示段号 + 时间段),粘性吸顶
- 缩进 32px 展示该段所有版本作为可勾选行
- 版本行结构:`[checkbox] vN [状态标签] [顺序号(已勾时)] [上下箭头(已勾时)]`

**勾选规则:**

| 版本状态 | checkbox | 视觉 |
|---|---|---|
| satisfied / usable | 启用,可勾 | 默认底色;勾后绿底 |
| unusable / failed / pending / generating / regenerating | disabled | 灰显 0.5 opacity |

**已勾选时:**

- 右侧显示 **绿色顺序号** `(1) (2) (3)` 等(在白色背景 + 绿色边的圆形里)
- 显示**上下箭头**调整顺序
  - 第一个 ↑ disabled
  - 最后一个 ↓ disabled

### 7.4 排序算法

- 维护 `_pickedList: Array<{ seg, ver, subTaskId }>`
- 勾选 → push 到末尾;取消 → splice 移除
- 顺序号 = `index + 1`
- 上下箭头交换相邻元素

### 7.5 已选统计行

```
已选 3 段 · 总时长 15 秒                   合成后视频时长 15 秒
─────────────────────────────────────────────────
拼接顺序: (1) 第1段 v1 → (2) 第2段 v1 → (3) 第3段 v3
```

- 段数 = `_pickedList.length`
- 总时长 = `段数 × 5`
- 拼接顺序按 `_pickedList` 顺序展示
- 0 段时:显示提示"至少选择 1 段才能合成"

### 7.6 已合成版本列表渲染

每个 Composition 一行:

| 状态 | 图标 | 操作按钮 |
|---|---|---|
| success | ✓ 绿对号 | `[👁 预览][⬇ 下载]` |
| composing | ⟳ 紫色 spinner(旋转) | (无,等待中) |
| failed | ⚠ 红色叹号 | `[↻ 重试]` |

**字段展示:**

```
[图标] {版本名} [状态标签] {段构成} · {总时长}s
        {compositionId} · {耗时/已耗时} · {文件大小} · {创建时间}
```

### 7.7 提交合成

点击 `[⬕ 提交合成任务]`(选中 ≥1 段才启用):

```
1. 调用 POST /api/tasks/{taskId}/compositions(详见 8.5 节)
2. 成功:
   - 立即在"已合成版本"列表头部插入一条新记录(status=composing)
   - toast"已提交合成任务 C20260425_NNNNNN"
   - 清空当前选择列表 + 版本名输入框(让用户可继续提交其他版本)
   - **不**自动关闭 Modal(业务可能想连续提交多个版本)
   - 同时更新顶部按钮的 ⟳ 徽章数 +1
3. 失败:
   - toast 错误,留在 Modal
```

### 7.8 在线预览(视频 Lightbox)

点击 `[👁 预览]` 打开全屏预览:

```
┌─ 深色蒙层 ──────────────────────────────[×]┐
│                                                │
│  ┌─ 16:9 视频区 ─────────────────────────┐  │
│  │ {渐变占位}      [10s 时长徽章 右上]     │  │
│  │       ▶  (大播放图标)                    │  │
│  │ 00:00 / 00:10  [进度条]                 │  │
│  └────────────────────────────────────────┘  │
│                                                │
│  ┌─ 信息卡 ─────────────────────────────┐  │
│  │ 默认全段版                    [⬇ 下载]│  │
│  │ C20260425_000001 · 第1段 v1 + 第2段 v1│  │
│  │ · 10s · 2.3 MB · 2026-04-25 14:50:18  │  │
│  └────────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

**关闭方式:**

- 右上角 [×]
- 点击蒙层
- 按 `Esc`

**下载按钮:** 直接下载当前预览的版本(避免业务关闭后再回 Modal 找)。

### 7.9 下载

```typescript
async function downloadComposed(compositionId) {
  // 调用接口拿临时下载 URL
  const res = await api.get(`/api/compositions/${compositionId}/download`);
  if (res.code === 0) {
    // 触发浏览器下载
    const a = document.createElement('a');
    a.href = res.data.url;
    a.download = `ai-video-${compositionId}.mp4`;
    a.click();
    toast.info(`已开始下载 ${a.download}`);
  }
}
```

---

## 8. API 契约

### 8.1 任务详情

**Endpoint:** `GET /api/tasks/{taskId}`

**鉴权:** 需要

**Response:**

```typescript
{
  code: 0,
  data: {
    // ============ Task 字段 ============
    id: string;
    shop: string;
    asin: string;
    duration: number;
    segCount: number;
    scriptModel: string;
    styleTemplate: 'aggressive' | 'balanced' | 'conservative';
    styleTemplateLabel: string;
    videoModel: string;
    createdBy: string;
    createdByDisplayName: string;
    isMine: boolean;                // 当前用户是否创建者(决定写权限)
    createdAt: string;
    updatedAt: string;

    // ============ 商品图文资料(创建时录入,任务详情冗余存) ============
    productInfo: ProductInfo;       // 见 task-create.spec.md 第 3.1 节

    // ============ SubTask 列表(每段所有版本) ============
    subtasks: SubTask[];            // 见第 2.2 节,按 seg ASC, v ASC 排序
  }
}
```

### 8.2 单个子任务详情(轮询用)

**Endpoint:** `GET /api/subtasks/{subTaskId}`

**鉴权:** 需要

**用途:** 前端对 generating / regenerating / composing 状态的子任务每 5-10 秒轮询一次,拿最新状态。

**Response:**

```typescript
{
  code: 0,
  data: SubTask    // 见第 2.2 节
}
```

### 8.3 段审核

**Endpoint:** `POST /api/subtasks/{subTaskId}/review`

**鉴权:** 需要(且必须是 task.createdBy)

**Request:**

```typescript
{
  conclusion: '满意' | '可用' | '不可用';
  reviewNote: string;             // 可空字符串
}
```

**Response 成功:**

```typescript
{
  code: 0,
  data: SubTask    // 返回更新后的子任务,前端覆盖本地状态
}
```

**Response 错误:**

```typescript
// 该子任务状态不允许审核(不是 pending,也不是已审核)
{ code: 4002, message: "当前状态不允许审核" }

// 不是创建者
{ code: 4030, message: "只有任务创建者可以审核" }
```

**关键:** 既支持首次审核(status: pending → satisfied/usable/unusable),也支持修改结论(status: 已审核 → 新结论)。后端按当前状态自动判断。

### 8.4 重做 / 重试

**Endpoint:** `POST /api/subtasks/{subTaskId}/redo` / `POST /api/subtasks/{subTaskId}/retry`

**Request:**

```typescript
{
  prompt: string;                  // 用户编辑后的提示词(原值或修改值)
}
```

**Response 成功:**

```typescript
{
  code: 0,
  data: {
    newSubTask: SubTask;           // 新创建的子任务,status=generating/regenerating
    quotaPreempted: number;        // 预占的额度(秒)
    quotaAvailable: number;        // 操作后剩余额度
  }
}
```

**Response 失败:**

```typescript
// 重做:被重做的版本不是已审核状态
{ code: 4002, message: "只能对已审核段重做" }

// 重试:被重试的版本不是失败状态
{ code: 4002, message: "只能对失败段重试" }

// 同段已经在生成中
{ code: 4002, message: "该段当前已有生成中的版本,请等待完成" }

// 额度不足
{ code: 4002, message: "视频生成额度不足" }
```

**后端逻辑:**

1. 校验 SubTask 状态(redo: 已审核;retry: 失败)
2. 校验同 (taskId, seg) 没有 generating/regenerating 状态的版本
3. 校验额度足够(redo 消耗 5s,retry 是预占已返还后再预占 5s)
4. 创建新 SubTask 记录(v = 同段最大 v + 1, status = regenerating)
5. 预占额度
6. 派发视频生成任务到队列

### 8.5 提交长视频合成

**Endpoint:** `POST /api/tasks/{taskId}/compositions`

**Request:**

```typescript
{
  name: string | null;               // 版本名,可空
  picks: Array<{
    seg: number;
    ver: number;
    subTaskId: string;               // 冗余,后端校验确实存在
  }>;
}
```

**Response 成功:**

```typescript
{
  code: 0,
  data: Composition    // 新建合成记录,status=composing
}
```

**Response 失败:**

```typescript
// 选段中有未审核或不可用的
{ code: 4000, message: "存在不可合成的版本(选段必须为已审核满意/可用)" }

// 段总数为 0
{ code: 4000, message: "至少选择 1 段" }
```

**后端逻辑:**

1. 校验所有 picks 对应的 SubTask 都是 satisfied / usable
2. 创建 Composition 记录
3. 派发 FFmpeg 拼接任务到队列(status=composing)
4. 返回新记录

**注意:** 长视频合成**不消耗视频生成额度**(只是文件操作)。如果有"合成额度"概念,需另立。一期不做。

### 8.6 单个合成版本详情(轮询用)

**Endpoint:** `GET /api/compositions/{compositionId}`

**Response:**

```typescript
{
  code: 0,
  data: Composition
}
```

### 8.7 重试合成

**Endpoint:** `POST /api/compositions/{compositionId}/retry`

**用途:** 失败的合成版本重试

**Response:**

```typescript
{
  code: 0,
  data: Composition    // 同记录,status 变回 composing
}
```

### 8.8 下载长视频

**Endpoint:** `GET /api/compositions/{compositionId}/download`

**用途:** 拿临时签名下载 URL

**Response:**

```typescript
{
  code: 0,
  data: {
    url: string;                   // 临时签名 URL,例如 30 分钟有效
    expiresAt: string;             // 失效时间
    fileName: string;              // 'ai-video-C20260425_000001.mp4'
    fileSize: number;
  }
}
```

---

## 9. 权限规则(同事 vs 自己)

按方案 B(详见 task-list.spec.md 第 11 节决议):

### 9.1 同事任务(`isMine === false`)的写操作禁用

| UI 元素 | 自己任务 | 同事任务 |
|---|---|---|
| 三档审核按钮 | 启用 | disabled |
| 提交审核按钮 | 启用 | disabled |
| 修改结论按钮 | 启用 | disabled |
| 重做本段按钮 | 启用 | disabled |
| 重试 / 编辑提示词后重试 | 启用 | disabled |
| 合成长视频按钮 | 启用 | 启用(可看历史合成,但无法**提交新合成**) |
| 提交合成任务按钮 | 启用 | disabled |
| 下载已合成视频 | 允许 | **允许**(便利共享) |
| 预览已合成视频 | 允许 | 允许 |

### 9.2 同事任务的视觉提示

页面顶部加 banner:

```
ⓘ 该任务由 [👤 张燕霞] 创建,你只能查看和下载已合成视频
```

蓝色 info 卡,占满宽,放在概要条上方。

---

## 10. 边界 & 错误处理

### 10.1 任务不存在 / 不属于本系统

`GET /api/tasks/{taskId}` 返回 404 → 跳回 `/task-list.html` + toast"任务不存在"

### 10.2 段没有任何版本

理论不可能(任务创建时必为每段建至少 1 个 SubTask)。若发生,前端中栏显示"该段数据异常,请联系研发"。

### 10.3 用户长时间停留页面后操作

- 操作时返回 4010(token 失效)→ 跳登录页
- 操作时返回 4002(状态不一致,如重做时段已经在重做)→ toast 错误 + 强制刷新页面拿最新数据

### 10.4 轮询频率与停止

- 仅当存在 generating / regenerating / composing 状态的实体时启动轮询
- 全部完成 → 停止轮询
- 用户切换 tab(`document.visibilityState === 'hidden'`)→ 暂停轮询,回来再继续(节省带宽)

### 10.5 多 tab 同时操作

- 业务可能同时打开多个详情 tab
- 每个 tab 独立轮询,不会冲突
- 一个 tab 的写操作 → 其他 tab 通过下次轮询拿到新状态(不主动通知)

---

## 11. 非功能性要求

- **任务详情查询:** P95 ≤ 800ms
- **子任务轮询:** P99 ≤ 200ms(单个查询)
- **审核提交:** P99 ≤ 500ms
- **重做 / 重试:** P99 ≤ 800ms(包含数据库写 + 队列派发)
- **合成提交:** P99 ≤ 500ms(只创建记录 + 派发,不等合成完成)
- **轮询频率:** 5–10 秒/次。注意:用户离开 tab 时暂停

---

## 12. 开发者注意事项(原型 vs 真实)

### 12.1 原型 segData 写死

```javascript
// 原型(写死 6 段 + 各版本)
const segData = {
  1: { versions: [{ v: 1, status: 'satisfied', ... }] },
  3: { versions: [{ v: 1, status: 'unusable' }, { v: 2, status: 'usable' }, ...] },
  ...
};

// 真实开发
const task = await api.get(`/api/tasks/${taskId}`);
const segData = groupSubTasksBySeg(task.subtasks);  // 按段归组
```

### 12.2 已耗时实时滚动

原型用 `_elapsedTickStart = Date.now()` + 写死的 `elapsedBaseSec` 模拟。真实开发:

```javascript
function tickElapsed() {
  document.querySelectorAll('.elapsed-timer').forEach(el => {
    const startedAt = el.dataset.startedAt;     // ISO 字符串
    const elapsed = Math.floor((Date.now() - new Date(startedAt).getTime()) / 1000);
    el.textContent = formatElapsed(elapsed);
  });
}
setInterval(tickElapsed, 1000);
```

后端**必须**返回 `startedAt`(ISO 时间戳),不要返回相对秒数。

### 12.3 SubTaskID 的复制

原型 `copySubTaskId` 用 `navigator.clipboard.writeText`。真实开发保持即可,加 fallback 到 `document.execCommand('copy')` 兼容老浏览器。

### 12.4 写权限判断

```javascript
// 渲染按钮前
const canWrite = task.isMine;
<button disabled={!canWrite}>重做本段</button>
```

不要在前端硬编码当前用户对比,让后端返回 `isMine` 减少前端维护负担。

### 12.5 视频文件占位

原型用 CSS 渐变模拟视频(`v1` / `v2` / ... 不同色)。真实开发:

```html
<video src={subtask.videoUrl} controls poster={...} />
```

注意 `videoUrl` 需要支持公网访问(走对象存储签名 URL 或公开桶)。

### 12.6 长视频合成的 Composition 列表

原型在打开 Modal 时静态渲染 2 条 mock 数据。真实开发:

```javascript
async function openComposeModal(taskId) {
  const res = await api.get(`/api/tasks/${taskId}/compositions`);
  composedVersions = res.data.list;
  renderComposedList();
  // ... 启动轮询(若有 composing 状态)
}
```

### 12.7 视频 Lightbox 真实播放

原型显示模拟播放器(渐变 + 静态进度条)。真实开发:

```html
<video src={composition.videoUrl} controls autoplay style="width:100%; aspect-ratio:16/9;" />
```

业务点开就能直接播放真实视频。

---

## 13. 涉及的数据库表

| 表名 | 用途 |
|---|---|
| `tasks` | 任务主表(读) |
| `subtasks` | 段级版本(读 / 写,审核 / 重做 / 重试时新增或更新) |
| `compositions` | 长视频合成版本表(读 / 写) |
| `quota_usage` | 重做 / 重试时更新预占额度 |
| `users` | 校验创建者 + 显示名 |

**索引建议:**

- `subtasks (task_id, seg, v)` — 唯一约束(同段同任务不允许重复版本号)
- `subtasks (task_id, status)` — 状态聚合查询
- `subtasks (status, started_at)` — 后端定时扫描超时任务
- `compositions (task_id, created_at DESC)` — 列表查询

**关键约束:**

- `(task_id, seg, v)` 唯一
- 软删除推荐:`deleted_at` 字段。一期不开放删除接口,但留字段方便后续

---

## 14. 待确认

- [ ] 子任务轮询的具体频率 — 5 秒还是 10 秒?WebSocket 二期再考虑
- [ ] 视频文件下载是直链还是签名 URL?推荐**签名 URL**(30 分钟有效),防爬
- [ ] 失败的子任务保留多久?是否需要清理策略?
- [ ] 长视频合成超时的兜底 — 后端单次合成最长 5 分钟?超时如何处理?是否自动失败 + 业务可重试?
- [ ] 业务能否删除单个 SubTask 版本?目前 spec 不支持(版本只增不删)。如果需要,要补 DELETE 接口
- [ ] 业务能否对 satisfied 段做"取消满意"?目前是修改结论(只能改成另一档结论),不能取消打回 pending。要确认是否需要
- [ ] 商品图文资料的更新 — 任务创建后,商品图改了能不能影响任务里的素材?目前是**冗余存**(创建时快照),后续修改不同步
- [ ] 同事任务"看历史合成"是否允许下载文件?目前 spec 是**允许**,如果要严格隔离改为不允许
