# login.spec.md — 登录页规格

> 基于 `login.html` 原型,系统唯一入口,账号由研发统一在数据库开通,无注册功能。

**前置依赖:** [common.spec.md](common.spec.md)
**对应原型:** [login.html](../login.html)
**路由:** `/login.html`(本页),登录成功跳 `/task-list.html`

---

## 1. 页面定位

- 系统**唯一入口**,未登录用户访问任何页面都应被前端拦截 → 重定向到本页
- 账号由研发统一在数据库表分配开通,**前端不提供注册 / 第三方登录入口**
- 不需要 token 鉴权(本页本身就是发 token 的地方)
- 每个账号在 Quota 表中绑定每日各模型生成额度(详见 common.spec.md 第 7.2 节)

---

## 2. 数据模型

涉及实体:`User`(详见 common.spec.md 第 7.1 节)。

本页面无独立数据模型。

---

## 3. UI 字段规格

### 3.1 表单字段

| 字段 | 控件 | 类型 | 必填 | 约束 | 默认值 | 说明 |
|---|---|---|---|---|---|---|
| 账号 | `<input type="text">` | string | ✓ | 1–32 字符,匹配 `/^[a-zA-Z0-9_.]+$/` | (空) | 原型为演示填写了 `wang.yunying`,真实开发应为空 |
| 密码 | `<input type="password">` 带眼睛切换 | string | ✓ | 6–32 字符,UTF-8 任意可见字符 | (空) | 原型为演示填写了占位符,真实开发应为空 |
| 记住我 | `<input type="checkbox">` | boolean | — | — | `true` | 勾选后 token 写 localStorage,否则 sessionStorage |

### 3.2 显示文案

页面静态文案(从原型 HTML 中提取,真实开发可直接复用):

- 顶部标签:`✨ 跨境电商视觉域专家`
- 主标语:`让商品视频生产 提速 100 倍`(后半句墨绿渐变高亮)
- 副标:`输入店铺与 ASIN,自动拉取商品资料,LLM 生成分镜脚本,Seedance 一键生成视频片段,业务专注审核与质量把控。`
- 3 个特性勾选项:
  - `分镜脚本自动生成,业务可逐段调整`
  - `支持单段重做与失败重试,版本可追溯`
  - `每日额度按账号管控,过程透明可观测`
- 底部版权:`© 2026 智汇创想 · Powered by XUQIANG & YANGZHIQIANG`
- 右侧提示卡:`本系统为内部工具,账号由研发统一开通。每个账号绑定每日视频生成额度,如需开通账号或调整额度,请联系研发负责人。`
- 帮助文案:`忘记密码请联系 IT`(无对应链接,纯文字)
- 版本号:`v1.0.0`(右下角,从前端 build 信息读取)

---

## 4. 交互逻辑

### 4.1 密码显示/隐藏切换

- 右侧眼睛图标(`ti-eye-off` ↔ `ti-eye`)点击时切换 `<input type>` 在 `password ↔ text` 之间
- 不影响表单值

### 4.2 登录按钮点击流程

```
1. 前端校验
   - username 非空 → 否则显示错误"请输入账号"
   - password 非空 → 否则显示错误"请输入密码"
2. 校验通过 → 按钮变 loading 态:
   - disabled
   - 显示 spinner + "登录中…"
3. 调用 POST /api/auth/login
4. 成功(code: 0):
   - 根据 remember 字段写入 localStorage 或 sessionStorage:
     - token
     - user(displayName / username / role)
     - expiresAt(now + expiresIn * 1000)
   - 跳转 /task-list.html(若 URL 含 ?redirect=xxx 参数则跳 xxx)
5. 失败(code !== 0):
   - 按钮恢复正常态
   - 错误提示卡显示 message 文案,红色边框 + 红字
6. 网络异常:
   - 按钮恢复正常态
   - 错误提示卡显示"网络异常,请稍后重试"
```

### 4.3 错误提示卡

- 表单上方 `<div class="login-error">`
- 默认 `display: none`,触发错误时 `display: flex`
- 内容:`<i class="ti ti-alert-circle"></i><span>错误文案</span>`
- 颜色:`var(--danger-bg)` 底色 + `var(--danger)` 边色和字色
- 用户开始输入新内容时自动隐藏(`onChange` 触发)

### 4.4 回车提交

按下 `Enter` 时触发登录(标准 `<form onSubmit>` 行为)。

### 4.5 已登录用户访问本页

- 前端检测 `localStorage.token` 或 `sessionStorage.token` 存在且未过期
- 直接跳 `/task-list.html`(不展示登录表单)
- 这是为了防止用户已登录后误打开本页又重新登录

---

## 5. 状态机

无独立状态机。表单状态:

```
idle → loading(登录中) → success(跳转)
                       → error(错误提示) → idle
```

---

## 6. API 契约

### 6.1 登录

**Endpoint:** `POST /api/auth/login`

**鉴权:** 不需要

**Request:**

```typescript
interface LoginRequest {
  username: string;       // 必填,1-32 字符,/^[a-zA-Z0-9_.]+$/
  password: string;       // 必填,6-32 字符
  remember: boolean;      // 默认 false,前端控制
}
```

**Response 成功:**

```typescript
interface LoginSuccess {
  code: 0;
  data: {
    token: string;            // JWT,Base64 字符串
    expiresIn: number;        // 过期秒数,如 86400(24 小时)
    user: {
      username: string;        // 'wang.yunying'
      displayName: string;     // '王运营'
      role: 'admin' | 'operator' | 'reviewer';
    };
  };
}
```

**Response 错误:**

```typescript
// 账号或密码错误
{ code: 4001, message: "账号或密码错误" }

// 账号已停用
{ code: 4030, message: "账号已停用,请联系 IT" }

// 输入格式错误
{ code: 4000, message: "账号格式错误" }

// 速率限制(短时间内多次失败)
{ code: 4290, message: "尝试过于频繁,请稍后再试" }
```

**幂等性:** 不需要(登录本就是 idempotent 的查询型操作)

**速率限制:** 同一 IP / username 每分钟最多 5 次失败,超出返回 4290。

### 6.2 退出登录

**Endpoint:** `POST /api/auth/logout`

**鉴权:** 需要

**Request:** 空

**Response:**

```typescript
{ code: 0 }
```

**前端行为:**

- 调用此接口后清除 localStorage / sessionStorage 中的 token、user 等
- 跳转 `/login.html`
- 即使接口失败,前端也强制清除本地数据并跳转(后端 token 失效是幂等的)

### 6.3 当前用户信息

**Endpoint:** `GET /api/auth/me`

**鉴权:** 需要

**用途:** 用户访问其他页面时,前端校验 token 有效性 + 拿用户信息渲染顶部栏。

**Response 成功:**

```typescript
{
  code: 0,
  data: {
    username: string;
    displayName: string;
    role: string;
    quota: {
      // 简化的当前用户额度信息,用于顶部展示
      model: string;          // 当前选用的视频生成模型
      dailyLimit: number;     // 总额度(秒)
      used: number;           // 已用
      preempted: number;      // 预占
    };
  }
}
```

---

## 7. 边界 & 错误处理

### 7.1 表单提交时的本地校验

按 4.2 节流程,前端校验失败时**不调用**后端 API,直接显示错误提示。

### 7.2 token 过期 / 失效

- 任何 API 返回 `code: 4010` → 前端拦截器(axios interceptor 等)统一处理:
  1. 清除 token
  2. 跳 `/login.html?redirect={当前页面路径}`
- 登录成功后跳回 `redirect` 参数指向的页面

### 7.3 多 tab 同时登录

- 不强制单 tab 登录(允许多 tab)
- 一个 tab 退出 → 其他 tab 下次调 API 时会拿到 4010 → 自动跳登录页

### 7.4 浏览器禁用 cookie / localStorage

- 不主动检测,使用 try/catch 包装 storage 操作
- 失败时降级到内存存储(刷新页面就丢失,体验差但不崩溃)

---

## 8. 非功能性要求

- **响应时间:** 登录接口 P99 ≤ 500ms
- **安全:**
  - 密码必须用 bcrypt / argon2 哈希存储,**禁止明文 / MD5 / SHA1**
  - JWT 签名密钥定期轮换(或使用 RSA 非对称密钥)
  - 启用 HTTPS,登录页 HTTP 自动重定向到 HTTPS
- **日志:** 登录成功 / 失败都要记审计日志(username / IP / UserAgent / timestamp / 结果)

---

## 9. 开发者注意事项(原型 vs 真实)

### 9.1 原型登录无校验

原型 `handleLogin()` 函数中 `setTimeout` 1 秒后**无脑跳转**到 task-list,不调用任何 API。真实开发要替换为:

```javascript
// 原型代码(仅演示)
setTimeout(() => { window.location.href = 'task-list.html'; }, 1000);

// 真实开发
const res = await fetch('/api/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ username, password, remember })
});
const json = await res.json();
if (json.code === 0) {
  storage.setItem('token', json.data.token);
  storage.setItem('user', JSON.stringify(json.data.user));
  window.location.href = redirect || '/task-list.html';
} else {
  showError(json.message);
}
```

### 9.2 原型默认填写了演示账号

原型为方便演示,`<input value="wang.yunying">` 和 `<input value="●●●●●●●●">`。**真实开发要清空 value**,让浏览器自动填充(autocomplete)生效。

### 9.3 "记住我"的实现

- 勾选 → `localStorage`(关闭浏览器再开仍登录)
- 不勾选 → `sessionStorage`(关闭 tab 即失效)
- 不要在两边都写,前端封装一层 storage helper:

```typescript
const storage = remember ? localStorage : sessionStorage;
storage.setItem('token', token);
```

### 9.4 提示文案的"忘记密码请联系 IT"

原型是纯文字。真实开发可考虑改为弹小窗显示 IT 同事的联系方式(企微 / 邮箱),或挂联系页链接。**一期不强求**。

---

## 10. 涉及的数据库表

| 表名 | 用途 | 字段定义 |
|---|---|---|
| `users` | 账号信息 | 见 common.spec.md 第 7.1 节 |
| `audit_logs`(可选) | 登录审计日志 | `id / username / action / ip / user_agent / result / created_at` |

**users 表的种子数据**(研发上线时手动 INSERT):

```sql
INSERT INTO users (username, password_hash, display_name, role, status)
VALUES
  ('wang.yunying', '<bcrypt hash>', '王运营', 'operator', 'active'),
  ('zhao.shenhe', '<bcrypt hash>', '赵审核', 'reviewer', 'active'),
  ...;
```

---

## 11. 待确认

- [ ] JWT 还是 session-cookie?推荐 JWT(无状态,易扩展),但要求严格清理 token 失效列表
- [ ] 是否需要图形验证码 / 滑块验证码?业务体量小,**一期不需要**,二期视情况加
- [ ] 失败 5 次后是否锁定账号?**一期不锁定**,只做速率限制(IP 维度);二期再讨论是否做账号锁定
- [ ] 是否需要登录通知(邮件 / 企微)异常登录?**一期不需要**
- [ ] "v1.0.0" 版本号怎么生成?建议从 `package.json` 或 git tag 读取
