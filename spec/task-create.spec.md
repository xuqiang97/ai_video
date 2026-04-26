# task-create.spec.md — 新建生成任务页规格

> 业务从店铺 + ASIN 出发,经过 3 步 wizard 生成一个新的视频任务:商品图文资料 → 总分镜脚本配置 → 逐段提示词确认 + 提交。

**前置依赖:** [common.spec.md](common.spec.md)
**对应原型:** [task-create.html](../task-create.html)
**路由:** `/task-create.html`
**鉴权:** 需要

---

## 1. 页面定位

- 业务**新建一个视频生成任务**的 wizard 页(3 步)
- 每个步骤独立校验,只有当前步骤数据完整才能进入下一步
- 提交后**异步**调用视频生成模型,业务被跳转回 `/task-list.html`,稍后在列表里看到新任务的状态变化
- **任务的"创建人"自动绑定为当前登录用户**(后端从 token 解析,前端不传)

---

## 2. Wizard 总览

```
[步骤 1: 商品图文资料] → [步骤 2: 生成总分镜脚本配置] → [步骤 3: 逐段提示词确认 + 提交]
```

每个步骤是独立的**配置阶段**,数据本地保存在前端状态里。**只有最后一步点"提交任务"时**才一次性把整个任务提交给后端创建 Task 实体并触发视频生成。

### 2.1 步骤切换规则

- **顶部 step 圆点可点击**回退到任意已访问步骤
- 前进:必须当前步骤校验通过
- 回退:不校验,直接跳
- 用户离开页面(关闭 tab / 跳转其他页):**所有数据丢失**,无草稿保存(一期不做草稿)

### 2.2 异常退出

- 若用户在步骤 2 / 3 中调用了"生成脚本"接口拿到了脚本,但没提交任务就关闭了 tab,**这次 LLM 调用产生的额度仍然消耗**(已经发生的 API 调用无法撤销)
- 若用户在步骤 3 提交任务时点了"取消",**视频生成尚未触发**,无额度消耗

---

## 3. 数据模型

### 3.1 Task(创建任务时落库的主表)

详见 [task-list.spec.md](task-list.spec.md) 第 2.1 节。新建任务时落库的字段:

```typescript
interface CreateTaskInput {
  shop: string;                     // 步骤 1 输入
  asin: string;                     // 步骤 1 输入
  productInfo: ProductInfo;         // 步骤 1 输入(商品图文资料)
  duration: number;                 // 步骤 2 选择(秒,5-60,5 倍数)
  scriptModel: string;              // 步骤 2 选择
  styleTemplate: 'aggressive' | 'balanced' | 'conservative';
  styleTemplateContent: string;     // 步骤 2 中可被用户修改的实际软要求文本
  videoModel: string;               // 步骤 3 选择
  segments: SegmentInput[];         // 步骤 3 各段最终提示词
}

interface ProductInfo {
  title: string;                    // 商品标题
  category: string;                 // 类目,自由文本(亚马逊层级,如 'Toys & Games >> ...')
  size: { length: number; width: number; height: number; unit: 'cm' | 'in'; };
  sellingPoints: string[];          // 卖点,每行一条
  images: string[];                 // 商品图 URL 数组,1-5 张
}

interface SegmentInput {
  seg: number;                      // 1-indexed
  prompt: string;                   // 分镜脚本(用户编辑后的最终内容)
}
```

### 3.2 ProductLibraryItem(商品中台一项 — 用于步骤 1 自动查询)

```typescript
interface ProductLibraryItem {
  shop: string;
  asin: string;
  title: string;
  category: string;
  size: { length: number; width: number; height: number; unit: 'cm' | 'in'; };
  sellingPoints: string[];
  defaultImages: string[];          // 多角度白底图,默认填到表单
}
```

商品中台是**外部依赖**,可能是公司内已有的产品数据库 / 接口。一期前端通过 `GET /api/products?shop=&asin=` 查,后端代理到中台。

### 3.3 风格模板的初始内容

3 个内置模板,作为前端默认值。后端不存储:

```typescript
const styleTemplates = {
  aggressive: '分镜设计优先级:在保证商品一致性和基础可用率的前提下,尽量把视频做得更像真正的商品宣传片...',
  balanced: '分镜设计优先级:在保证商品一致性和可用率的前提下,适度提升宣传片质感...',
  conservative: '分镜设计优先级:首要保证商品一致性、清晰度和可用率,镜头语言克制稳定...',
};
```

完整文案见原型 `task-create.html` 中 `softTemplates` JS 对象。**真实开发可以直接复制**。

### 3.4 固定硬规则(LLM system prompt)

固定不变,作为 LLM 调用的 system prompt 一部分。完整文案见原型 task-create.html 步骤 2 左侧"固定硬规则"卡。**真实开发可以直接复制**。后端在调 LLM 时拼接:

```
{固定硬规则} + {风格模板内容(用户可能改过)} + {商品图文资料(JSON)} + {视频时长}
```

---

## 4. 步骤 1:商品图文资料

### 4.1 UI 字段(从上到下)

| 字段 | 控件 | 必填 | 约束 | 说明 |
|---|---|---|---|---|
| 店铺 | input | ✓ | 自由文本 | 一期手动填写,二期改为下拉(对应 Shop 表) |
| ASIN | input(font-mono) | ✓ | 自由文本,通常 10 位字母数字 | 一期仅支持 ASIN,二期支持 SKU |
| **[查询]** 主色按钮 | — | — | — | 从商品中台查询自动填充下方字段 |
| 标题 | textarea(2 行) | ✓ | maxLength 200 | |
| 类目 | textarea(2 行) | ✓ | 自由文本 | 亚马逊层级,如 `Toys & Games(玩具和游戏)>>Novelty & Gag Toys(创意玩具)>>Money Banks(存钱罐)` |
| 尺寸 | 3 个 input(长 / 宽 / 高)+ 单位选择 | ✓ | 数字 + 单位 cm/in | 用 flex 布局合并,带连接符 |
| 卖点 | textarea(6 行) | ✓ | maxLength 2500,每行一条 | 实时显示字符计数 `XXX / 2500` |
| 商品图 | 图片上传画廊 | ✓ | 1–5 张,jpg/png/webp,单张 ≤ 10MB | 默认从产品库取多角度白底图 |

### 4.2 查询商品资料

**用户操作:** 输入店铺 + ASIN → 点 `[查询]`

**前端流程:**

```
1. 校验 shop, asin 非空
2. 查询按钮变 loading 态
3. GET /api/products?shop={shop}&asin={asin}
4. 成功 → 自动填充表单字段
5. 失败(404 等) → 显示警告卡:"商品中台未找到该 ASIN,请检查后重试,或手动录入下方字段"
   - 用户仍可手动填写所有字段后继续
6. 网络错误 → 同 5 处理,警告内容改为"网络异常,请重试或手动录入"
```

### 4.3 卖点字符计数

实时统计 textarea 字符数,显示格式 `123 / 2500`。超出 2500 字符时:
- 字符计数变红
- 阻止再输入(textarea `maxLength` 属性)

### 4.4 商品图上传

**前端实现:**

- 图片画廊网格,5 个槽位
- 默认是空槽,显示 `📷 点击上传 / 拖拽 / 粘贴,已上传 0 / 5 张`
- 上传方式:
  - 点击空槽 → 选择文件
  - 拖拽到槽内 → 自动上传
  - 复制图片 → 粘贴到页面任意位置 → 自动上传
- 上传中显示 spinner 占位
- 上传完成显示缩略图 + hover 删除按钮

**调用接口:** `POST /api/upload` 返回图片 URL,前端把 URL 存入 `productInfo.images` 数组。

### 4.5 校验进入步骤 2

点击"下一步"时校验:
- 所有必填字段非空
- 商品图至少 1 张
- 尺寸字段都是数字
- 卖点字符数在 [1, 2500]

任一不通过 → 显示 toast / inline error,不切换步骤。

---

## 5. 步骤 2:生成总分镜脚本配置

### 5.1 UI 结构

```
┌─ 视频时长 + 脚本生成模型(同行)─────────────┐
│ [视频时长 ▼] [脚本生成模型 ▼]                  │
└──────────────────────────────────────────────┘

┌─ 左:固定硬规则(只读) ──┬─ 右:风格模板(可编辑) ──┐
│                            │  [激进版][平衡版][保守版]   │
│  [大段 LLM system prompt]  │  [我的模板] [帮助文档]     │
│  [scrollable, max-height:  │                              │
│   480px]                    │  [textarea 软要求文本]      │
│                            │                              │
└────────────────────────────┴────────────────────────────┘

┌─ "我的模板"二期占位 + 提示文案 ─────────────┐
│ ⓘ 二期会支持保存为"我的模板"沉淀常用配置...   │
└──────────────────────────────────────────────┘

[← 上一步]                  [✨ 生成总分镜脚本 →]
```

### 5.2 字段规格

| 字段 | 控件 | 必填 | 默认值 | 选项 |
|---|---|---|---|---|
| 视频时长 | select | ✓ | `30 秒(6 段)` | `5 秒(1 段)` 到 `60 秒(12 段)`,5 的整数倍 |
| 脚本生成模型 | select | ✓ | `GPT-5.4` | `GPT-5.4` / `Seed 2.0`(二期可扩展) |
| 风格模板 | Tab(三选一)+ textarea | ✓ | `激进版` | `激进版`(推荐) / `平衡版` / `保守版`,Tab 切换更新 textarea 内容 |

### 5.3 帮助文档链接

风格模板 Tab 行右侧:`[❓ 帮助文档]` → 新窗口打开 `https://www.yuque.com/johnny97pm/zhcx/ai_video?singleDoc`

### 5.4 风格模板切换

- 点击 Tab 切换 → 立即覆盖 `<textarea id="soft-textarea">` 的 value 为对应模板内容
- 用户编辑 textarea → 不影响 Tab 高亮(但实际发给 LLM 的是当前 textarea 内容)
- **一期不持久化**用户的修改:下次新建任务又是默认模板。提示文案:`本次修改仅对当前任务生效,不影响下次新建`
- **二期**支持"我的模板"——用户可保存自定义为命名模板

### 5.5 生成总分镜脚本

点击 `[✨ 生成总分镜脚本]` 按钮:

```
1. 切换 Loading Modal 显示:
   - "正在生成总分镜脚本"
   - 副文案:"使用 GPT-5.4 · 激进版风格,根据商品图文资料生成 6 段分镜……"
   - "预计 15–30 秒,请耐心等待"
   - 取消按钮(用户可中断)
2. POST /api/scripts/generate(详见 7.2 节)
3. 成功 → 关闭 Loading,自动跳到步骤 3,把 segments 数据填进去
4. 失败 → 关闭 Loading,toast 错误,留在步骤 2
5. 取消 → 调用 cancel 接口或前端 abort fetch,留在步骤 2
```

**注意:** LLM 是同步调用(15-30 秒),不是异步任务。这跟视频生成不一样。

---

## 6. 步骤 3:逐段提示词确认 + 提交

### 6.1 UI 结构

```
┌─ 顶部成功提示 ────────────────────────────────┐
│ ✓ 已为商品 B08DNRKJX7 生成 6 段分镜提示词,    │
│   使用 GPT-5.4 · 激进版,耗时 22s              │
│                              [↺ 返回上一步重新生成] │
└──────────────────────────────────────────────┘

┌─ 最终提示词模板(只读) ───────────────────────┐
│ 📋 最终提示词模板  [只读]                       │
│ 下方每段编辑的内容会替换 {{5s分镜脚本}} 后,    │
│ 整体作为最终提示词发给视频生成模型             │
│                                                │
│ 【亚马逊美国站点商品卖点介绍视频5s分镜脚本】:  │
│ {{5s分镜脚本}}                                 │
│                                                │
│ 【要求】:虽然分镜脚本是中文,但视频画面中禁止... │
└──────────────────────────────────────────────┘

┌─ 提示条 + 帮助文档 ───────────────────────────┐
│ ⓘ 按时间顺序展示 6 段分镜脚本,可整体浏览节奏  │
│   点击任意一段卡片右上角的 ✏ "编辑" 进行微调  │
│                                  [❓ 帮助文档] │
└──────────────────────────────────────────────┘

[第 1 段卡][第 2 段卡][第 3 段卡]    ← 段卡片网格(2-3 列)
[第 4 段卡][第 5 段卡][第 6 段卡]

┌─ 提交区:视频生成模型 ────────────────────────┐
│ 视频生成模型: [Seedance 2.0 ▼]                │
│ 用于将每段分镜脚本生成 5 秒片段...            │
└──────────────────────────────────────────────┘

[← 上一步]                       [⮕ 提交任务]
```

### 6.2 段卡片(每段一个)

```
┌─ 第 N 段 · X–Ys ────────[✏ 编辑]─┐
│                                      │
│ {分镜脚本文本,自由换行}              │
│                                      │
└────────────────────────────────────┘
```

**编辑模式(点击 [✏ 编辑] 后):**

```
┌─ 第 N 段 · X–Ys ───[取消][保存修改]─┐
│ ┌──────────────────────────────────┐│
│ │ {textarea,可编辑}                  ││
│ │                                    ││
│ └──────────────────────────────────┘│
│ 字符数:98     ⓘ 修改后系统会自动暂存,│
│              提交时一起生效         │
└────────────────────────────────────┘
```

- 点击"取消" → 恢复原文本,退出编辑
- 点击"保存修改" → 更新本地 segments 数据,显示绿色 `已修改` 标签提示用户

### 6.3 提交任务

点击 `[⮕ 提交任务]`:

```
1. 弹出"提交任务"确认 Modal:
   - 列出任务概要(店铺 / ASIN / 时长 / 模型 / 风格 / 段数)
   - 提示"将预占 N×5 秒视频生成额度,生成失败的会自动返还"
   - 显示当前剩余额度
2. 用户确认 → 调用 POST /api/tasks(详见 7.3 节)
3. 成功 →
   - toast"任务 T20260425_NNNNNN 已提交,正在生成视频片段..."
   - 跳转 /task-list.html
   - 列表中能看到新任务,所有段状态为 generating
4. 失败 →
   - 4002 额度不足 → 显示"额度不足,无法提交。请等明日额度恢复或联系研发调整"
   - 其他 → toast 错误,留在步骤 3
```

### 6.4 重新生成的二次确认

点击 `[↺ 返回上一步重新生成]`:

```
弹出确认 Modal:
  "确认重新生成?"
  "将丢失你刚才对 6 段分镜脚本的所有修改"
  [取消] [确认重新生成]
```

确认后回到步骤 2(不立即重新调用 LLM,让用户先看一眼是否要改时长 / 模型 / 风格)。

---

## 7. API 契约

### 7.1 商品中台查询

**Endpoint:** `GET /api/products`

**鉴权:** 需要

**Query Params:**

```typescript
{
  shop: string;       // 例如 'Amazon-Z01097-US'
  asin: string;       // 例如 'B08DNRKJX7'
}
```

**Response 成功:**

```typescript
{
  code: 0,
  data: ProductLibraryItem  // 见第 3.2 节
}
```

**Response 找不到:**

```typescript
{ code: 4040, message: "商品中台未找到该 ASIN" }
```

### 7.2 生成总分镜脚本

**Endpoint:** `POST /api/scripts/generate`

**鉴权:** 需要

**用途:** 调用 LLM 生成总分镜脚本(同步,15-30 秒)

**Request:**

```typescript
{
  productInfo: ProductInfo;             // 见第 3.1 节
  duration: number;                      // 5 / 10 / 15 / ... / 60
  scriptModel: string;                   // 'GPT-5.4'
  styleTemplate: 'aggressive' | 'balanced' | 'conservative';
  styleTemplateContent: string;          // 用户实际编辑后的软要求文本
}
```

**Response 成功:**

```typescript
{
  code: 0,
  data: {
    segments: [
      { seg: 1, prompt: '暖金色光影中,金色陶瓷生肖马摆件...' },
      { seg: 2, prompt: '镜头切到侧前方中近景,缓慢环绕马头...' },
      // ...
    ];
    elapsedSec: 22;                      // 实际耗时,前端展示用
  }
}
```

**Response 失败:**

```typescript
// LLM 服务异常
{ code: 5001, message: "脚本生成失败,LLM 服务异常,请稍后重试" }

// LLM 输出不合 JSON 格式(prompt 工程问题)
{ code: 5002, message: "脚本生成结果格式异常,请重试" }

// 额度不足
{ code: 4002, message: "今日 LLM 调用额度已耗尽" }
```

**幂等性:** 不需要(LLM 调用本就不幂等,每次结果可能不同)

**速率限制:** 单用户每分钟最多 5 次调用(防止用户暴力刷脚本)

### 7.3 提交任务(创建 Task + 触发视频生成)

**Endpoint:** `POST /api/tasks`

**鉴权:** 需要

**Request:**

```typescript
CreateTaskInput   // 见第 3.1 节
```

**Response 成功:**

```typescript
{
  code: 0,
  data: {
    taskId: string;             // 'T20260425_000007'
    subTaskIds: string[];        // 各段对应的 SubTaskID,按段顺序
  }
}
```

**Response 失败:**

```typescript
// 额度不足
{ code: 4002, message: "视频生成额度不足,需要 30 秒,当前剩余 25 秒" }

// 字段校验失败
{ code: 4000, message: "段数与视频时长不匹配" }
```

**后端做的事:**

1. 校验额度(总秒数 = `segments.length * 5`,需 `available >= 总秒数`)
2. 创建 Task 记录(`createdBy = 当前 token 用户`)
3. 为每段创建 SubTask 记录(状态 `generating`)
4. **预占额度** `quotaUsage.preempted += 总秒数`
5. 异步派发视频生成任务到队列(每段独立)
6. 返回 taskId + subTaskIds

**幂等性:** 推荐前端在 Header 加 `Idempotency-Key`(防双击),后端短期缓存避免重复创建。

### 7.4 上传图片

**Endpoint:** `POST /api/upload`

**鉴权:** 需要

**Content-Type:** `multipart/form-data`

**Request:**

- `file`: 图片文件(jpg / png / webp,≤ 10MB)
- `purpose`: `'product_image'`(用于商品图)

**Response:**

```typescript
{
  code: 0,
  data: {
    url: string;        // 公网可访问 URL,可用于后续接口
    key: string;        // 对象存储 key,后端记录用
    size: number;       // 文件字节数
    mimeType: string;
  }
}
```

---

## 8. 状态机

无独立状态机。Wizard 是线性流转,前端用一个 `currentStep` 状态控制即可:

```typescript
let currentStep: 1 | 2 | 3 = 1;
let formData = {
  step1: { /* ... */ },
  step2: { /* ... */ },
  step3: { /* ... */ }
};
```

---

## 9. 边界 & 错误处理

### 9.1 用户在步骤 1 跳到步骤 2 后又回退修改

允许。回退时步骤 2 的字段保留,但**不**自动触发重新生成脚本。用户必须重新点 `[✨ 生成总分镜脚本]`。

### 9.2 商品图上传失败

单张失败 toast 提示,其他图不受影响。用户可重新上传那一张。

### 9.3 LLM 生成结果不符合 JSON 格式

后端做兜底:
- 调用 LLM 后强制解析 JSON
- 解析失败 → 重试一次
- 仍失败 → 返回 5002 错误,提示用户重试

### 9.4 段数校验

后端在 `POST /api/tasks` 时校验 `segments.length === duration / 5`,不一致返回 4000 错误。

### 9.5 用户已在步骤 3 编辑过段,又点"返回上一步重新生成"

按 6.4 节做二次确认。一旦确认,所有编辑丢失。

### 9.6 提交时浏览器关闭 / 网络断

由于 `POST /api/tasks` 是在用户点提交那一刻触发的同步请求,网络断会失败。建议:
- 前端做请求重试(最多 3 次)
- 用户提交后立即跳转列表页 — 即使列表加载失败,也能从其他设备 / 重新打开页面看到任务状态

---

## 10. 非功能性要求

- **生成脚本响应时间:** P95 ≤ 30 秒(取决于 LLM 服务性能)
- **创建任务响应时间:** P99 ≤ 1 秒(只创建数据库记录 + 派发队列,不等视频生成)
- **图片上传:** 单张 P95 ≤ 5 秒(取决于网络和文件大小)
- **额度检查:** 创建任务前严格校验,不允许超额(避免业务任务跑一半失败)

---

## 11. 开发者注意事项(原型 vs 真实)

### 11.1 原型 mock 数据

原型中"查询商品"按钮点击后,直接 1 秒延迟自动填充演示商品(金色生肖马存钱罐 ASIN B08DNRKJX7)。真实开发应:

```javascript
// 原型(写死)
setTimeout(() => fillFormWithDemo(), 1000);

// 真实开发
const res = await api.get('/api/products', { params: { shop, asin } });
if (res.code === 0) fillForm(res.data);
else showWarning(res.message);
```

### 11.2 原型 LLM 生成是 1.5 秒假 loading

```javascript
// 原型
setTimeout(() => {
  showStep(3);
  fillSegments(mockSegments);
}, 1500);

// 真实开发
const res = await api.post('/api/scripts/generate', input);
hideLoading();
if (res.code === 0) {
  showStep(3);
  fillSegments(res.data.segments);
} else {
  showError(res.message);
}
```

### 11.3 原型提交是 alert + 跳转

```javascript
// 原型
alert('任务已提交');
window.location.href = 'task-list.html';

// 真实开发
const res = await api.post('/api/tasks', formData);
if (res.code === 0) {
  toast.success(`任务 ${res.data.taskId} 已提交`);
  router.push('/task-list');
}
```

### 11.4 商品图上传的真实逻辑

原型只是 placeholder。真实开发要实现:
- `<input type="file" multiple>` 选择
- `dragover / drop` 拖拽
- `paste` 事件粘贴
- 每张图独立 `POST /api/upload`,显示进度
- 上传完后填回 URL 数组

### 11.5 LLM Prompt 拼接

后端组装 LLM 调用的 prompt 时:

```
system_prompt = 固定硬规则 + 风格模板内容(用户编辑后)

user_prompt = JSON.stringify({
  shop: ...,
  asin: ...,
  product: { title, category, size, sellingPoints, images },
  duration: 30
})
```

具体格式建议跟 LLM 团队对齐。

### 11.6 创建人字段不需要前端传

- 后端从 token 解析 `currentUser.username`,自动填到 `Task.createdBy`
- 前端**不**在 `CreateTaskInput` 中传 `createdBy` 字段(避免伪造)

---

## 12. 涉及的数据库表

| 表名 | 用途 |
|---|---|
| `tasks` | 任务主表(创建一行) |
| `subtasks` | 段级子任务(创建 N 行,N = segCount) |
| `quota_usage` | 预占额度变更(`preempted += N * 5`) |
| `users` | 校验当前用户 |

**事务建议:**

`POST /api/tasks` 应在单个事务内:
1. 检查额度
2. 创建 Task
3. 创建 N 个 SubTask
4. 更新 QuotaUsage
5. 派发 N 个视频生成任务到队列(队列推送可在事务外,失败时重试)

---

## 13. 待确认

- [ ] 步骤 1 的"查询商品"接口是直连产品库 / 公司中台 / 自建副本?**需要研发对齐**
- [ ] LLM 调用接入哪些模型?GPT-5.4 是 OpenAI 还是国内代理?
- [ ] LLM 失败的重试策略(自动重试一次?多次?)?**待研发讨论**
- [ ] 草稿保存:一期**不做**,业务确认是否真的不需要(关闭 tab 数据丢失)
- [ ] 商品图上传的 OSS / S3 配置?最大单张大小 10MB 是否合理?
- [ ] 可以 cancel 已经发起的 LLM 调用吗?后端要不要支持中断?(简单实现:前端 abort fetch,后端继续跑完不影响业务)
- [ ] 段数 12 时段卡是否会换 3 行?目前 CSS 是 grid 自动换行,手机 / 小屏可能压缩到 2 列
