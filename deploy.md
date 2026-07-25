# 部署文档：即刻备忘 (InstantMemo) V1.0

**文档版本**：v1.0
**适用环境**：生产环境 (Production)
**目标**：将单 HTML 页面部署至公网，实现 PC/平板/手机全端访问，数据全量云同步。

---

## 1. 前置准备 (Prerequisites)

在开始之前，请确保你已拥有以下三个账号（全部免费）：

- **GitHub 账号**：用于代码托管。
- **Supabase 账号**：用于云数据库和鉴权（推荐直接用 GitHub OAuth 登录）。
- **Vercel 账号**：用于部署网页（推荐直接用 GitHub OAuth 登录）。

---

## 2. 第一步：云数据库与鉴权配置 (Supabase)

这是**最核心**的一步。后端逻辑全部由 Supabase 托管，配置错一个字符都会导致网页报错。

### 2.1 创建项目
1. 登录 [Supabase Dashboard](https://app.supabase.com)。
2. 点击 **"New project"**。
   - **Name**：`instant-memo`（随意）。
   - **Database Password**：生成一个强密码（务必记下来，虽然本方案不直接连数据库，但留着备用）。
   - **Region**：选择距离你最近的区域（如 `Singapore` 或 `Tokyo`，国内访问延迟较低）。
   - 点击创建，等待 2-3 分钟初始化完成。

### 2.2 执行建表 SQL（最关键）
1. 进入项目后台，点击左侧菜单 **"SQL Editor"**。
2. 点击 **"New query"**，将以下**完整 SQL 脚本**复制进去，点击 **"Run"** 执行：

```sql
-- 1. 创建备忘表
CREATE TABLE memos (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  content TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  user_id UUID NOT NULL
);

-- 2. 创建索引（优化列表查询速度）
CREATE INDEX idx_memos_user_created ON memos(user_id, created_at DESC);

-- 3. 开启行级安全锁 (RLS)
ALTER TABLE memos ENABLE ROW LEVEL SECURITY;

-- 4. 创建安全策略：用户只能操作自己的数据 (重点)
CREATE POLICY "用户只能操作自己的备忘" ON memos
  FOR ALL
  USING (auth.uid() = user_id);
```

> **容错检查**：如果报错 `relation "memos" already exists`，说明之前建过，直接跳过建表，只执行后面的 `ALTER` 和 `CREATE POLICY` 即可。

### 2.3 获取 API 密钥（匿名公钥）
1. 左侧菜单点击 **"Settings"** -> **"API"**。
2. 在 **"Project URL"** 处复制你的 Supabase 链接（格式：`https://xxxxx.supabase.co`）。
3. 在 **"Project API keys"** 处，复制 **`anon public`** 密钥（**注意：不要复制 `service_role` 密钥**，那个是管理员权限，泄露会导致数据库被删）。

---

## 3. 第二步：代码配置与环境变量注入

### 3.1 修改本地 HTML 文件
打开我之前提供的 `index.html`，找到文件顶部的 JavaScript 配置区（约第 90 行）：

```javascript
// --------------------------------------------------------------
// 1. 配置你的 Supabase (免费注册，获取 URL 和 anon key)
// --------------------------------------------------------------
const SUPABASE_URL = 'https://你的项目ID.supabase.co';       // 替换这里
const SUPABASE_ANON_KEY = '你的anon公钥';                   // 替换这里
```

将其替换为你刚从 Supabase 复制的值。

### 3.2 （进阶优化）利用 Vercel 环境变量
由于直接写死在代码里虽然能用，但不够“工程化”。如果你想将密钥隐藏在 Vercel 服务端环境变量中，可以这样做（可选，跳过则维持上一步）：

1. 将代码中的密钥占位符改为从 `window.env` 读取。
2. 在 Vercel 项目设置中添加 `SUPABASE_URL` 和 `SUPABASE_ANON_KEY`。
*（注：个人工具场景下，直接写死 `anon` 公钥是安全的，因为它仅用于限制匿名访问，不包含数据库读写权限，泄露后最多被他人调用 API 读走你一个人的数据，风险可控）*

---

## 4. 第三步：部署静态网页 (Vercel)

### 方式 A：极速 CLI 部署（推荐，最省事）
如果你本地装有 Node.js 和 Git：

```bash
# 1. 全局安装 Vercel CLI
npm install -g vercel

# 2. 进入你的 index.html 所在目录
cd /path/to/your/project

# 3. 登录并部署 (首次需授权)
vercel login
vercel --prod
```

终端会交互式询问配置，**直接全部回车使用默认值**即可。部署完成后，终端会输出一个公网地址（如 `https://instant-memo-xxx.vercel.app`）。

### 方式 B：Git 关联自动部署（适合后续迭代）
1. 在 GitHub 新建一个仓库（如 `instant-memo`）。
2. 将 `index.html` 推送到仓库。
3. 登录 Vercel，点击 **"Add New"** -> **"Project"**，导入该 GitHub 仓库。
4. Build Command 留空（因为是纯静态），Output Directory 留空，直接点击 **"Deploy"**。

---

## 5. 第四步：⚠️ 致命细节——Supabase 域名白名单 (CORS & Network)

**如果你跳过这一步，手机访问网页时一定会报 `403` 或 `CORS policy` 错误。** 因为 Supabase 默认只允许本地请求。

### 5.1 配置 Network Restrictions
1. 回到 Supabase 后台，左侧菜单进入 **"Settings"** -> **"API"**。
2. 滚动到页面下方的 **"Network Restrictions"** 区域。
3. 点击 **"Add allowed domain"**，输入你刚刚部署得到的 Vercel 域名（**注意：不要带 `https://` 前缀，也不要有末尾斜杠**，例如输入 `instant-memo-xxx.vercel.app`）。
4. 如果你想在本地调试，还需要额外添加 `localhost`。
5. 点击保存，等待约 1-2 分钟生效。

> **检查清单**：如果网页报错 `Request failed with status code 403`，99% 是因为这一步没做。

---

## 6. 第五步：验证与端到端测试 (E2E Test)

部署完成后，请按以下流程验证系统是否完好：

1. **桌面端测试**：打开电脑 Chrome，访问 Vercel 网址，页面应显示“即刻备忘”标题，输入框自动聚焦（光标闪烁）。
2. **录入测试**：随意输入“Hello World”，按下回车，底部闪现绿色“✓ 已记录”，输入框自动清空。
3. **列表测试**：点击“查看全部”，列表中按时间倒序显示刚才的条目。
4. **跨设备同步测试**：
   - 将 Vercel 网址用手机（关闭 WiFi，使用 4G/5G 网络）打开。
   - **关键观察**：手机上应显示与电脑上**完全相同**的数据列表（因为同步依赖 Supabase，不依赖设备本地存储）。
5. **匿名持久化测试**：关闭手机浏览器，重新打开链接，数据依然存在（除非你手动清除浏览器缓存或 Supabase 会话过期，届时 Supabase SDK 会自动重新生成匿名用户，数据还是那批数据，因为数据绑的是 `user_id`）。

---

## 7. 配置自定义域名（Optional）

如果你不喜欢 Vercel 的二级域名，可以绑定自己的域名：

1. 在 Vercel 项目设置中找到 **"Domains"**，输入你的域名（如 `memo.yourdomain.com`）。
2. 前往你的域名 DNS 服务商（如 Cloudflare），添加一条 **CNAME** 记录，指向 `cname.vercel-dns.com`。
3. 等待 SSL 证书自动签发（Vercel 自动支持 HTTPS）。
4. **千万别忘**：去 Supabase 的 Network Restrictions 中，把 `memo.yourdomain.com` 也加入白名单！

---

## 8. 常见故障排查 (Troubleshooting)

| 报错现象 | 可能原因 | 解决方案 |
| :--- | :--- | :--- |
| **白屏 / `Failed to fetch`** | Supabase URL 填错，或网络不通（被墙）。 | 检查 `SUPABASE_URL` 是否写了 `https://`；尝试切换 Supabase 地区或使用代理。 |
| **`403 Forbidden`** | Supabase Network 白名单未包含当前域名。 | 前往 Supabase Settings -> API -> Network Restrictions，将 Vercel 域名加入允许列表，等待 1 分钟。 |
| **`JWT expired` / 登录失败** | 浏览器缓存了旧的无效会话。 | 清除浏览器 LocalStorage（F12 -> Application -> Storage -> Clear site data）。 |
| **列表页显示空，但电脑上有数据** | 手机端匿名登录生成了新的 `user_id`，因 RLS 策略不同导致数据隔离。 | **这是预期行为**。如果你想多端共享数据，则需要升级为“邮箱密码登录”（V2.0），当前 V1.0 匿名模式本质上是每台设备一个独立的小房间。 |

---

## 9. 运维与监控建议

- **数据备份**：Supabase 自带每日自动备份（免费版保留 7 天），你可以去 Dashboard 的 **"Backups"** 手动下载 SQL 导出。
- **数据库查询**：如果你不小心记了脏数据，可以直接在 Supabase 的 **"Table Editor"** 中手写 SQL 或直接删除行（必要时可临时禁用 RLS）。
- **升级提醒**：当你修改了 `index.html` 推送到 GitHub 时，Vercel 会自动重新部署，无需手动触发。