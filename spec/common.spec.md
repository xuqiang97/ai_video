# common.spec.md — 通用约定

> 全局约定,所有页面 spec 引用此文档。修改本文档前请评估对所有页面的影响。

---

## 1. 技术栈建议

本项目为内部工具,业务体量小(约 10 人,每天 50 个视频任务),建议:

- **前端:** React 18 + TypeScript + Vite + Tailwind CSS,或同等价位的 Vue 3 + TS。原型 HTML 的设计 tokens(`--brand`、`--text-1` 等)直接迁移为 Tailwind config 或 CSS variables。
- **后端:** Node.js (Nest.js / Express) 或 Python (FastAPI) 或 Go (Gin)。
- **数据库:** PostgreSQL(推荐,支持 JSONB 字段存提示词长文本)或 MySQL 8.0+。
- **对象存储:** S3 兼容(AWS S3 / 阿里 OSS / MinIO),用于存视频文件、商品图。
- **异步任务队列:** Redis + Bull / Celery,用于视频生成、长视频合成等长耗时任务。
- **部署:** 单实例即可,业务体量不需要分布式。

---

## 2. 全局 ID 生成规则

所有业务实体 ID 使用**前缀 + 日期 + 序号**形式,日期为创建当天:

| 实体 | 前缀 | 示例 | 说明 |
|---|---|---|---|
| Task(生成任务) | `T` | `T20260425_000007` | 同日下递增,6 位数字 |
| SubTask(子任务,段级版本) | `S` | `S20260425_000003` | 同日下递增,6 位数字。每段每次生成都是一个 SubTask |
| Composition(长视频合成版本) | `C` | `C20260425_000001` | 同日下递增,6 位数字 |
| User(用户) | — | `wang.yunying` | 直接使用 username,不用代理 ID |

**生成方式:**

```typescript
// 后端伪代码
function generateId(prefix: 'T' | 'S' | 'C'): string {
  const today = format(new Date(), 'yyyyMMdd');
  // 用 Redis INCR 或数据库行级锁拿当日序号
  const seq = await incrDailySequence(`${prefix}:${today}`);
  return `${prefix}${today}_${String(seq).padStart(6, '0')}`;
}
```

**注意:**

- ID 不复用,即使任务删除,序号也不回退
- 跨天序号重置(20260425 的序号和 20260426 的不连续)
- ID 唯一,可作为业务主键。数据库可额外用 UUID 作内部主键

---

## 3. 全局时间格式

- **API 传输 / 数据库存储:** ISO 8601(`2026-04-25T14:35:21.000Z`)或 epoch 毫秒
- **前端展示:** `YYYY-MM-DD HH:mm:ss`(精确到秒)
- **相对时间(如"已耗时"):** 由前端基于 `startedAt` 实时计算,不在后端返回相对时间字符串

---

## 4. 全局鉴权

### 4.1 Token 机制

- 登录成功后后端返回 `token`(JWT 推荐,有效期 24 小时,可配置)
- 前端将 token 写入 `localStorage`(勾"记住我")或 `sessionStorage`(不勾)
- 所有 API 请求(除 `/api/auth/login`)在 Header 携带:

```
Authorization: Bearer <token>
```

### 4.2 Token 失效处理

- 后端校验 token 失败 → 返回 `401 Unauthorized` + `code: 4010`
- 前端统一拦截 401:清除本地 token,跳转 `login.html`,并保留原页面 URL 作为 `redirect` 参数,登录成功后跳回

### 4.3 鉴权范围

- `login.html` 不需要 token
- 其他所有页面 / API **必须**携带有效 token

---

## 5. 全局错误码格式

所有 API 响应统一结构:

```typescript
interface ApiResponse<T = any> {
  code: number;       // 0 表示成功,非 0 表示错误
  message?: string;   // 错误时的可读提示(中文,可直接展示给用户)
  data?: T;           // 成功时的业务数据
}
```

**错误码分段:**

| 段 | 含义 | 举例 |
|---|---|---|
| 0 | 成功 | `code: 0` |
| 4000–4099 | 业务错误(用户输入 / 状态错误) | `4001 账号密码错误`、`4002 任务状态不允许此操作` |
| 4010 | Token 无效 / 过期 | 前端拦截 → 跳登录页 |
| 4030 | 权限不足 | `4030 此账号已停用` |
| 4290 | 速率限制 | `4290 请求过于频繁` |
| 5000+ | 服务端错误 | `5001 视频生成模型异常`、`5002 对象存储上传失败` |

**HTTP 状态码:**

- 业务成功(`code: 0`)→ HTTP 200
- 业务错误(`code: 4xxx`)→ HTTP 200(便于前端统一拦截)或 HTTP 4xx(标准 RESTful)。**项目内部统一选一种,推荐 HTTP 200 + body code**。
- 服务端错误 → HTTP 500

---

## 6. 全局分页约定

所有列表 API 统一参数:

```typescript
interface PageRequest {
  page: number;       // 1-indexed,默认 1
  pageSize: number;   // 默认 20,可选 10 / 20 / 50
  // 其他业务过滤字段...
}

interface PageResponse<T> {
  total: number;          // 总记录数
  page: number;           // 当前页
  pageSize: number;       // 当前每页数量
  list: T[];              // 当前页数据
}
```

---

## 7. 全局数据模型

以下表为多个页面共用,在此统一定义。各页面 spec 中只引用、不重复。

### 7.1 User(用户)

```typescript
interface User {
  username: string;       // 主键,1-32 字符,/^[a-zA-Z0-9_.]+$/
  passwordHash: string;   // bcrypt / argon2 哈希,不返回到前端
  displayName: string;    // 中文姓名,如 "王运营"
  role: 'admin' | 'operator' | 'reviewer';  // 一期所有人都是 operator
  status: 'active' | 'disabled';
  createdAt: string;
  updatedAt: string;
  lastLoginAt: string | null;
}
```

**关键说明:**

- 账号由研发**手工**在数据库 `INSERT`,无注册接口
- 一期不区分角色权限,所有 `operator` 都能做所有操作
- `status: disabled` 时登录返回 `4030`,前端展示"账号已停用,请联系 IT"

### 7.2 Quota(每日额度)

```typescript
interface Quota {
  id: number;
  username: string;       // 外键 → User.username
  model: string;          // 模型标识,如 'seedance-2.0'、'gpt-5.4'
  dailyLimit: number;     // 每日上限,单位:秒(视频生成)/ 次(LLM 调用)
  unit: 'seconds' | 'calls';
  createdAt: string;
  updatedAt: string;
}

interface QuotaUsage {
  id: number;
  username: string;
  model: string;
  date: string;           // YYYY-MM-DD
  used: number;           // 当日已用
  preempted: number;      // 当日已预占未结算
  // available = dailyLimit - used - preempted
}
```

**关键逻辑:**

- 提交视频生成 / 重做 / 重试时,**预占** `5s` 额度(`preempted += 5`)
- 生成成功 → `preempted -= 5; used += 5`(净增加 5)
- 生成失败 → `preempted -= 5`(返还,业务可重试不消耗额度)
- 业务每次提交前后端校验 `available >= 5`,否则返回 `4002 额度不足`

### 7.3 Shop(店铺)

```typescript
interface Shop {
  id: string;             // 例如 'Amazon-Z01097-US'
  platform: string;       // 'Amazon' | 'eBay' | ...
  account: string;        // 'Z01097'
  region: string;         // 'US' | 'UK' | 'JP' | ...
  createdAt: string;
}
```

**说明:** 一期店铺由业务在新建任务页**手动填写**(自由文本),不强制对应到 Shop 表。二期改为下拉选择并强校验。

---

## 8. 全局视觉约定

### 8.1 主色

- **品牌主色:** `#00B96B`(墨绿青),用于主按钮、链接、激活态、品牌区
- **辅助色:** info `#165DFF`、success `#00B42A`、warning `#FF7D00`、danger `#F53F3F`、purple `#722ED1`

### 8.2 字体

- **正文:** 系统字体栈,中文优先 PingFang SC / Microsoft YaHei
- **等宽:** SF Mono / Monaco / Consolas — 用于 ID、代码、时间戳等

### 8.3 图标

- 统一使用 [Tabler Icons](https://tabler-icons.io/),class 形式 `<i class="ti ti-xxx">`
- CDN: `https://cdn.jsdelivr.net/npm/@tabler/icons-webfont@2.47.0/tabler-icons.min.css`

### 8.4 状态色对应

| 业务状态 | 颜色变量 |
|---|---|
| satisfied(满意)/ success(合成成功) | `--success` 绿 |
| usable(可用) | `--info` 蓝 |
| unusable(不可用)/ failed(失败) | `--danger` 红 |
| pending(待审核) | `--text-3` 灰 |
| regenerating(重做中)/ generating(生成中)/ composing(合成中) | `--purple` 紫 |

各 spec 中所有状态枚举值的精确写法见对应章节。

---

## 9. 开发者注意事项(原型 vs 真实的差异)

### 9.1 原型 mock 数据 → 真实数据库查询

HTML 原型里硬编码了 `segData`、`composedVersions` 等 JS 变量。真实开发应:

- 页面初始化时调用对应 API 加载数据
- mock 数据中的字段名、字段含义保持一致(原型已经做了字段对齐)

### 9.2 原型时间字段 → 真实时间字段

原型里 "已耗时" 用 `elapsedBaseSec` + 页面加载时刻 模拟。真实开发:

- 后端返回每个版本 / 合成任务的 `startedAt` 时间戳
- 前端实时计算 `elapsedSec = (Date.now() - startedAt) / 1000`
- 不要从后端取相对秒数

### 9.3 原型 ID → 真实 ID

原型里所有 ID(TaskID / SubTaskID / CompositionID)都是写死的字符串。真实开发:

- 由后端按 `第 2 节` 规则生成
- 前端只展示,不生成

### 9.4 异步任务的状态轮询

视频生成、长视频合成都是异步任务。真实开发推荐:

- **轮询:** 前端每 5–10 秒调一次 `GET /api/subtask/{id}` 拿最新状态(简单,适合一期)
- **WebSocket:** 后端主动推送状态变更(体验更好,二期再考虑)
- 用户离开页面再回来时,自动重新拉取所有"进行中"任务的状态

### 9.5 视频文件存储

- 5s 视频片段:`/{taskId}/segment-{seg}-v{ver}-{subTaskId}.mp4`
- 长视频合成:`/{taskId}/composition-{compositionId}.mp4`
- 都存对象存储,数据库只存 URL / 文件 key

---

## 10. API 路由规划建议

按 RESTful 风格组织:

```
POST   /api/auth/login                       # 登录
POST   /api/auth/logout                      # 退出
GET    /api/auth/me                          # 当前用户信息

GET    /api/quota                            # 当前用户当日额度
GET    /api/shops?keyword=...                # 店铺列表(二期用,一期不实现)

GET    /api/tasks                            # 任务列表(分页 + 筛选)
POST   /api/tasks                            # 新建任务
GET    /api/tasks/{taskId}                   # 任务详情
PATCH  /api/tasks/{taskId}                   # 更新任务(如改标签)

POST   /api/tasks/{taskId}/script            # 生成总分镜脚本(LLM)
POST   /api/tasks/{taskId}/segments          # 提交所有段提示词,触发视频生成

GET    /api/subtasks/{subTaskId}             # 子任务详情(轮询用)
POST   /api/subtasks/{subTaskId}/review      # 段审核(满意/可用/不可用)
POST   /api/subtasks/{subTaskId}/redo        # 重做(已审核段)
POST   /api/subtasks/{subTaskId}/retry       # 重试(失败段)

GET    /api/tasks/{taskId}/compositions      # 该任务的所有长视频合成版本
POST   /api/tasks/{taskId}/compositions      # 提交长视频合成任务
GET    /api/compositions/{compositionId}     # 合成版本详情(轮询用)
POST   /api/compositions/{compositionId}/retry  # 重试失败的合成
GET    /api/compositions/{compositionId}/download  # 下载长视频(返回临时 URL)
```

具体的 request/response 结构在各页面 spec 中定义。

---

## 11. 待确认

以下事项 spec 写到此处仍未明确,等业务 / 研发讨论后补齐:

- [ ] LLM 调用模型(GPT-5.4 / Seed 2.0)的具体接入方式 — 自建网关还是直连?
- [ ] Seedance 2.0 的 API 形式 — 同步等待 / 异步 webhook / 异步轮询?
- [ ] 是否需要审计日志(操作人 / 时间 / 动作)?一期建议**做**,接入 SLS / ELK
- [ ] 每日额度重置时间 — 当地凌晨 00:00 还是 UTC 00:00?
- [ ] 用户离职后账号怎么处理 — 改 `status: disabled` 即可,数据保留
- [ ] 长视频合成超时的兜底 — 后端单次合成最长 5 分钟?超时如何处理?
