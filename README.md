# AI 商品视频系统(原型)

> 跨境电商视觉域内部工具:输入店铺与 ASIN,自动拉取商品资料,LLM 生成分镜脚本,Seedance 一键生成视频片段,业务专注审核与质量把控。

🌐 **在线 Demo:** https://xuqiang97.github.io/ai_video

📖 **产品文档(语雀):** https://www.yuque.com/johnny97pm/zhcx/ai_video?singleDoc

---

## 一句话讲产品

业务每天要为 50+ 商品生成宣传视频,传统拍摄一条要 1 天,本系统让业务**输入 ASIN → 5 分钟拿到一条 30 秒视频**,把视频生产端到端时间从「天」压缩到「分钟」。

业务的精力从「拍摄/剪辑」转移到「审核/把控」:每段 5 秒视频片段独立审核(满意 / 可用 / 不可用),不满意可单段重做,最后挑选已审核片段合成长视频。

---

## 主链路

```
登录 → 任务列表 → 新建生成任务(3 步 wizard) → 任务详情/审核 → 合成长视频
                                                    ↓
                                              单段重做 / 失败重试
```

详细操作流程见 [在线 Demo](https://xuqiang97.github.io/ai_video) 和 [产品文档](https://www.yuque.com/johnny97pm/zhcx/ai_video?singleDoc)。

---

## 关键设计

### 异步状态机(段 / 版本级)

每段每次生成都是一个独立子任务(SubTask),状态机:

```
generating → pending → satisfied / usable / unusable
                ↓
              redo → regenerating → ...

failed → retry → regenerating → ...
```

每个版本对应一个 SubTaskID(`S20260425_NNNNNN`),按创建顺序递增,方便研发排查。

### 段以版本为最小单位

同一段经历多次重做后会有 v1 / v2 / v3 ...,每个版本独立记录状态、提示词、子任务 ID。业务可以在版本切换 Tab 看到全部历史,不会被覆盖。

### 失败 / 重做的额度管理

- **生成成功**:扣除预占的 5s 额度
- **生成失败**:返还预占额度,业务可不限次数重试
- **已审核重做**:消耗 5s 额度(算新一次创建)

### 长视频合成(灵活组合)

合成的最小单位不是「段」而是「**已审核版本**」。同段多个已审核版本都可同时勾选(支持复用 / 扩展时长),顺序自由调:

```
拼接顺序: (1) 第1段 v1 → (2) 第3段 v3 → (3) 第3段 v2 → (4) 第4段 v1
```

业务可以基于一个 30s 任务合成多个长视频版本(15s / 20s / 30s / 45s 等)并存,各自下载。

### 异步合成任务

合成长视频是异步流水线(后端 FFmpeg + 对象存储,典型 30s–几分钟),前端展示 `合成中 / 合成成功 / 合成失败` 三档状态,完成后业务可在线预览或下载。

---

## 一期范围 / 二期范围

### 一期(已完成,2026-04 上线)

- 账号 + 每日额度管理(账号由研发统一开通,绑定每日各模型生成秒数额度)
- 任务管理:列表 / 筛选(快速 chip / 时间 / 模型 / 状态) / 分页
- 新建任务 wizard:商品图文资料 → 总分镜脚本配置 → 逐段提示词确认
- 任务审核:段级三档结论 / 单段重做 / 失败重试 / 版本管理
- 长视频合成简易版:同任务内灵活组合 / 异步合成 / 在线预览 / 下载

### 二期(规划中)

- **工程化重构**:CSS / shell.js 抽离共用,4 个老页面改造
- **长视频合成完整模块**:同商品跨任务选段 / 独立 list / create / detail 三页
- **字幕合成**:独立模块
- **音频合成**:独立模块
- **风格模板沉淀**:支持业务保存自己的「我的模板」(prompt 配置)

---

## 文件结构

```
ai_video/
├── README.md
├── index.html              # 根入口,重定向到 login
├── login.html              # 登录页
├── task-list.html          # 任务列表
├── task-create.html        # 新建任务 wizard(3 步)
├── task-detail.html        # 任务详情/审核 + 长视频合成
└── spec/
    ├── login.spec.md
    ├── task-list.spec.md
    ├── task-create.spec.md
    └── task-detail.spec.md
```

---

## 本地运行

纯静态 HTML/CSS/JS,无构建,无依赖。

**方式一:双击打开**

直接双击任意 `.html` 文件,浏览器打开即可。

**方式二:起本地 server(推荐,跨页跳转更稳)**

```bash
cd ai_video
python3 -m http.server 8000
```

然后打开 http://localhost:8000

---

## 部署

通过 GitHub Pages 部署:

1. 仓库 Settings → Pages
2. Source 选 `main` 分支 + `/ (root)`
3. 等几分钟,访问 `https://<你的用户名>.github.io/ai_video`

根路径访问会自动跳转到 `login.html`。

---

## 设计文档

- [login.html 规格](spec/login.spec.md) — 登录页
- [task-list.html 规格](spec/task-list.spec.md) — 任务列表
- [task-create.html 规格](spec/task-create.spec.md) — 新建任务 wizard
- [task-detail.html 规格](spec/task-detail.spec.md) — 任务详情 / 审核 / 合成

---

## 技术栈

- 纯静态 HTML / CSS / JS,无构建无依赖
- Tabler Icons 通过 CDN 引入
- 设计 tokens 内嵌在每个页面顶部 `<style>` 块(二期工程化时抽离)

---

Powered by XUQIANG & YANGZHIQIANG
