# 软件设计文档 (SDD - Software Design Document)

**项目名称**：即刻备忘 (InstantMemo)  
**版本**：V1.0.0  
**文档状态**：定稿  
**架构风格**：Serverless BaaS + 纯静态单页应用 (SPA)

### 1. 系统整体架构 (System Architecture)

采用 **C/S 直连云BaaS** 的极简拓扑，放弃中间应用服务器层。

```mermaid
graph LR
    Client[多端浏览器 (PC/Phone/Pad)] -->|HTTPS + REST API| Supabase[Supabase 后端即服务]
    Supabase -->|认证| Auth[Supabase Auth (匿名登录)]
    Supabase -->|数据操作| PostgreSQL[(PostgreSQL 数据库)]
    Client -->|静态托管| CDN[Vercel / CDN 全球分发]
```

### 2. 技术选型与版本

| 层级 | 技术栈 | 选型理由 |
| :--- | :--- | :--- |
| **前端** | 原生 HTML5 + CSS3 + Vanilla JS (ES6) | 零依赖，单HTML文件体积 < 20KB，无需构建工具，极致轻量。 |
| **云 BaaS** | Supabase (v2) | 提供实时 PostgreSQL API 和匿名认证；免费额度 500MB 数据库，个人使用绰绰有余。 |
| **部署托管** | Vercel / Netlify | 支持 Git 自动部署，自带全球 CDN 和强制 HTTPS，自定义域名友好。 |
| **版本控制** | Git (GitHub) | 单文件变更，便于回滚。 |

### 3. 数据库设计 (Schema Design)

**表名**：`memos`  
**RLS (行级安全策略)**：启用，强制用户只能操作 `user_id` 匹配自身 `auth.uid()` 的行。

| 字段名 | 数据类型 | 约束 | 描述 |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY`, `DEFAULT gen_random_uuid()` | 主键，全局唯一标识符 |
| `content` | `TEXT` | `NOT NULL` | 备忘的具体文本内容 |
| `created_at` | `TIMESTAMPTZ` | `DEFAULT now()` | 记录创建时间（带时区），用于列表排序 |
| `user_id` | `UUID` | `NOT NULL` | 关联 `auth.users`，用于数据隔离与安全策略 |

**索引策略**：在 `user_id` 和 `created_at` 上建立复合索引，优化列表查询速度。
```sql
CREATE INDEX idx_memos_user_created ON memos(user_id, created_at DESC);
```

### 4. 接口定义 (API Specs)

无需编写后端代码，直接调用 Supabase 自动生成的 RESTful API（基于 OpenAPI）。

| 功能 | 方法 | 端点 | 请求体/参数 |
| :--- | :--- | :--- | :--- |
| **匿名登录** | `POST` | `/auth/v1/signup` | `{ "gotrue_meta_security": {} }` (SDK封装) |
| **新增备忘** | `POST` | `/rest/v1/memos` | `{ "content": "string", "user_id": "uuid" }` |
| **查询全部** | `GET` | `/rest/v1/memos` | Query: `user_id=eq.{id}&order=created_at.desc` |

### 5. 前端核心模块设计 (Frontend Module)

单 HTML 文件内通过 IIFE（立即执行函数）隔离作用域，核心模块划分如下：

- **认证模块 (`initAuth`)**：封装 `supabase.auth.signInAnonymously()`，管理 `currentUserId` 状态，处理会话恢复。
- **数据访问模块 (`addMemo`, `fetchMemos`)**：封装 Supabase 数据操作，统一处理异常与乐观更新（UI 先变化，后端确认后补反馈）。
- **UI 渲染引擎 (`renderList`)**：使用 `document.createDocumentFragment()` 批量构建 DOM，替换 `innerHTML` 以提升列表渲染性能。
- **路由控制器 (`showPage`)**：通过 `display: none/flex` 控制双页面切换，维护浏览器历史状态（可借助 `hash` 路由简化）。

### 6. 数据流与时序 (Data Flow)

1. **录入流**：用户输入内容 → 回车触发 `addMemo` → 调用 Supabase API `insert` → 成功回调清空 Input + Toast 提示。
2. **同步流**：用户点击“查看全部” → 调用 `fetchMemos` → 获得 JSON 数组 → 遍历构建 `li` 列表项 → 一次追加至 DOM。
3. **跨设备流**：新设备打开 URL → `initAuth` 产生新的匿名 `user_id` → 查询该 ID 下的数据 → 返回（因为新设备无数据，展示空态；原设备数据保留）。

### 7. 部署拓扑与环境配置

- **环境变量**：`SUPABASE_URL` 和 `SUPABASE_ANON_KEY` 在构建时注入（本方案写死在前端，因匿名公钥属公开非敏感信息，但建议通过环境变量管理）。
- **部署步骤**：
  1. 将 `index.html` 推送到 GitHub 仓库。
  2. 在 Vercel 中导入该仓库，使用默认静态配置。
  3. 绑定个人域名（如 `memo.yourdomain.com`）以规避浏览器跨域和第三方 Cookie 限制。

### 8. 安全性设计 (Security)

- **Row Level Security (RLS)**：这是最核心的安全防线。即使前端伪造 `user_id`，数据库底层 RLS 会强制校验 `auth.uid()`，杜绝越权访问。
- **CORS 策略**：Supabase 控制台仅允许配置的白名单域名访问，防止 API 被恶意第三方盗用。
- **输入净化**：前端仅做基础 `trim()` 去空；因数据不涉及 HTML 渲染（直接 `textContent` 赋值），XSS 风险极低。

### 9. 已知限制与未来扩展 (Trade-offs & Future)

| 维度 | V1.0 妥协方案 | V2.0 可扩展方向 |
| :--- | :--- | :--- |
| **跨设备用户识别** | 依赖浏览器的 LocalStorage 存储 Session，若清除缓存则匿名账户丢失（但数据残留在库中）。 | 升级为 **邮箱/密码注册** 或 **手机号 OTP**，实现真正的永久跨设备登录。 |
| **数据管理** | 无法删除或修改错记内容。 | 增加左滑删除 + 长按编辑（但需权衡“简洁”原则，可能做成隐藏手势）。 |
| **离线支持** | 无网络完全不可用。 | 增加 Service Worker 做 PWA 离线缓存，提交失败时存入 IndexedDB 延迟同步。 |
| **提醒功能** | 仅存储 `content`，无时间触发。 | 表新增 `remind_at` 字段，后端配合 pg_cron 或 Edge Functions 推送 Web Push 通知。 |