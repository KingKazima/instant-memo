# CLAUDE.md

即刻备忘 (InstantMemo)。整个应用 = 由html静态文件实现，零依赖、无构建。Supabase + Vercel。个人自用，MAU ≈ 1。

## 铁律

1. **`SUPABASE_URL` 和 `SUPABASE_ANON_KEY` 是生产环境真实值，别动** — `anon` public key 公开非敏感，写死是设计
2. **html静态文件实现，不加构建步骤、不 npm install。
4. **用户内容用 `textContent` 渲染** — 严禁 `innerHTML`。新增代码也必须遵守
5. **换域名 → Supabase Settings → API → Network Restrictions 加白名单** — 否则全站 403

## 设计原则：零摩擦

唯一标准：**用户能在 3 秒内完成"打开 → 敲字 → 回车"吗？**


## 交互规格

- 提交成功 → 输入框**立即清空 + 重新聚焦** → Toast "✓ 已记录" **1.6s** 消失
- 提交失败 → **保留输入框内容**，Toast 提示错误
- 移动端 → 断点 480px，键盘不遮挡输入框

## 参考文档

- `PRD.md` — 交互规格权威来源（比代码更权威），改 UI 前必读
- `SDD.md` — 数据库 Schema、RLS 策略、API 端点定义
- `deploy.md` — 可执行建表 SQL 在 2.2 节，排错表在末尾
- `README.md` — 不用看，对外的

## 架构

```
浏览器 ←→ Supabase (supabase-js v2 CDN)
              ├── auth.signInAnonymously() → 会话存 LocalStorage
              └── memos: id(UUID) | content(TEXT) | created_at(TIMESTAMPTZ) | user_id(UUID)
```