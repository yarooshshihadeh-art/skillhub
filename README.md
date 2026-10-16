# SkillHub 复刻 · 云数据库驱动的 AI Skills 社区

一个「AI Skills 社区」的前后端复刻实现：**17 个静态页面 + 一套完全由云数据库驱动的管理后台（CMS）**。

前台所有文案、排行榜、企业卡片、发布区文案都从云端读取；后台改完即时生效，**无需重新部署**。

> ### ⚠️ 免责声明
> 本项目是**学习用途的界面复刻**，与 SkillHub、腾讯及任何官方产品**无任何关联**，不代表、也不使用其数据或服务。
> 仓库内的站点文案、榜单数据均为**演示数据**，不含任何真实用户信息或凭据。请勿用于商业用途。

---

## 特性

- **多页静态站点**：首页、技能库、技能详情、插件广场、企业专区、专家包、SOUL 广场、MCP 广场、教程、SkillPay、大赛、发布、登录等 17 个页面，纯 HTML/CSS/JS，无构建步骤。
- **云端 CMS**：站点内容存于 `site_content` 表（键值 + `schema` 字段描述表单结构），后台按 schema 动态生成表单，改完即时生效。
- **列表模板引擎**：容器标 `data-cmode="repeat"`，按数组重复渲染模板并用 `{{字段}}` 填充；技能榜直接从 `skills` 表取数。
- **完整后台**：工作台、内容运营（站点文案 / 首页区块 / 页面文案）、技能库、投稿审核、用户与权限、操作日志。
- **行级安全（RLS）**：前台匿名可读，写入只允许管理员；管理员成员表用 `SECURITY DEFINER` 函数判断，规避 RLS 自引用递归。
- **零依赖前端**：前台不引入框架，图标与动效均为原生实现。

---

## 架构

```
浏览器 ──► 静态页面（public/*.html）
             │
             ├─ js/content.js ──► 读取 site_content + skills ──► 填充 DOM
             │                     （按 data-ckey / data-cfield / data-cmode 定位）
             │
             └─ js/admin.js  ──► 后台：读 schema 生成表单 ──► 写回 site_content

数据层：PostgreSQL + PostgREST 风格 REST 接口 + RLS
```

**关键点：页面本身不认识数据库。** 页面只在元素上标 `data-ckey="s_hero" data-cfield="title"`，
`content.js` 启动时把整张 `site_content` 表拉下来，按标记填进去；缓存进 `localStorage` 实现秒开。

---

## 目录结构

```
public/                  站点与后台（部署时托管这个目录）
├── index.html           首页（102 处内容绑定 + 3 个列表容器）
├── admin.html           管理后台入口
├── skills.html 等       其余 15 个页面
├── assets/ css/         样式表
└── js/
    ├── cloud-config.js  云服务配置（改成你自己的，见下）
    ├── cloud.js         云服务客户端封装（登录 / 数据 / 文件）
    ├── content.js       内容引擎：DOM 绑定 + 列表模板渲染 + 字母头像兜底
    ├── admin.js         后台全部视图
    └── vendor/          DOMPurify（富文本净化）
sql/
├── schema.sql           建表 + 函数 + 权限 + RLS 策略
└── seed.sql             演示数据（25 条站点内容 + 16 条技能）
server.js                零依赖本地预览服务器
```

---

## 快速开始

### 1. 本地预览

```bash
node server.js          # 打开 http://localhost:8000
```

> 仓库自带一套**公开的演示云后端配置**（`public/js/cloud-config.js`），因此克隆后即可直接看到完整数据。
> 该配置只含前端可公开的信息（endpoint / 公钥），浏览器的任何人本来就能看到；写入受服务端 RLS 保护。

### 2. 接入你自己的后端

1. 准备一个 PostgreSQL（托管平台需提供 REST 接口与 `auth.uid()`；自建可用 PostgREST）。
2. 依次执行 `sql/schema.sql`、`sql/seed.sql`（注意：多数托管平台的迁移通道**单次只支持一条语句**，请按分号拆开执行）。
3. 修改 `public/js/cloud-config.js` 的四个值：

```js
window.SKILLHUB_PUBLIC_CONFIG = {
  resourceId: '你的 resourceId',
  endpoint: '你的服务端点',           // 必须与服务端校验的 Origin 完全一致
  oauthRelayBaseUrl: '你的 OAuth 中转地址',
  publishableKey: '你的公开密钥',
};
```

4. 部署 `public/` 目录（静态即可）：

```bash
python3 -m http.server $PORT --bind 0.0.0.0 --directory public
```

### 3. 进入后台

打开 `/admin.html` 登录。**首次登录的用户会被引导「认领超级管理员」** —— 这是设计好的引导机制：
当系统里一个管理员都没有时，RLS 策略 `admin_members_claim_first` 允许当前用户自封超管，否则新部署的后台会因「没人能审批第一个管理员」而永久锁死。

---

## 内容系统怎么工作

### 声明式绑定

```html
<!-- 文本 -->
<h2 data-ckey="s_ranking" data-cfield="title">发现当下值得使用的 Skill</h2>

<!-- 富文本（经 DOMPurify 净化）-->
<h2 data-ckey="s_enterprise" data-cfield="title" data-cmode="html"></h2>

<!-- 列表：从 skills 表按 home_rank_trending 排序取数 -->
<div data-cmode="repeat" data-csource="skills" data-clist="trending" data-climit="8"></div>
<template id="tpl-rank-row">…{{name}}…{{downloads}}…</template>
```

- `data-ckey` / `data-cfield`：定位到 `site_content.value` 里的某个字段，支持点路径与数组下标。
- `data-cmode="html"`：按富文本渲染（净化后）。
- `data-cmode="repeat"`：容器内按模板重复渲染，模板里用 `{{字段}}` 占位。
- 静态兜底：写死在 HTML 里的内容同时也是首屏兜底，云端取不到数据时页面依然可读。

### 后台字段类型

`schema` 里每个字段的 `type` 决定后台表单控件：

| type | 控件 | 存进 value 的类型 |
|---|---|---|
| `text` | 单行输入 | string |
| `textarea` | 多行输入 | string |
| `html` | 多行输入（允许 `<br>` 等标签） | string |
| `lines` | 多行输入 | **string[]**（每行一项，用于逐字动画的分段标题） |
| `number` | 数字输入 | number |
| `bool` | 开关 | boolean |
| `list` | 可增删的条目组（用 `item` 描述子字段，支持嵌套） | object[] |

### 安全

富文本一律经 `DOMPurify` 净化；净化库加载失败时**降级为完全转义**，绝不裸渲染。后台保存时按现有 value 合并字段（`Object.assign`）而非整体覆盖，避免 schema 之外的字段被静默删除。

---

## 已知限制

- **详情页未接入内容绑定**：`skill.html`、`plugin.html`、`package.html`、`enterprise-org.html` 的卡片仍是静态快照，尚未走 `repeat` 引擎接真实数据集。
- **列表页排序 / 筛选未动态化**：`skills.html` 的「近期飙升 / 下载量 / 最近上新」为静态展示。
- **登录与实名认证为演示实现**：原站为真实系统，此处仅保留交互闭环与权限骨架。
- 站点文案与榜单数字是复刻时的演示数据，不保证与原站一致或持续更新。

---

## 第三方

| 组件 | 用途 | 许可 |
|---|---|---|
| [DOMPurify](https://github.com/cure53/DOMPurify) | 富文本净化 | Apache-2.0 OR MPL-2.0 |

## 许可

[MIT](LICENSE)
