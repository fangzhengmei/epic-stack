# Epic Stack SEO 索引控制机制分析报告 (v3)

## 分析方法声明

本报告的所有结论均基于以下 **可验证的代码证据**：

1. **@nasa-gcn/remix-seo 库源码**：通过 npm 文档和上游仓库确认
2. **项目路由文件**：逐条检查 `app/routes/` 下所有文件的 `export const handle` 声明
3. **React Router 7 路由配置**：`docs/routing.md:89-201` 中 `npx react-router routes` 的完整输出
4. **环境变量配置**：`app/utils/env.server.ts`、`app/entry.server.tsx`、`app/root.tsx`

---

## 一、generateSitemap 的判断逻辑（已通过源码确认）

### 1.1 决策流程（来自上游库源码和文档）

根据 `@nasa-gcn/remix-seo` 的实现逻辑（源自 `balavishnuvj/remix-seo`）：

```
路由是否会出现在 sitemap.xml 中？

输入: context.serverBuild.routes (编译时路由模式列表)

对于每个路由:

  步骤 1: 检查是否有 SEOHandle
    ├── 有 handle.getSitemapEntries
    │   ├── 返回 null ──► 排除 (确定不会出现)
    │   └── 返回数组 ──► 包含这些自定义条目 (确定会出现)
    │
    └── 无 handle.getSitemapEntries (或无 handle)
        ├── 是布局路由 (_layout) ──► 排除 (不渲染具体页面)
        ├── 是动态路由 (含 :param) ──► 路由模式可能出现，但无具体实例
        └── 是静态路由 ──► 路由模式会出现在 sitemap 中
```

### 1.2 关键判断依据（库文档引用）

来自 [@nasa-gcn/remix-seo npm 文档](https://www.npmjs.com/package/@nasa-gcn/remix-seo)：

> **To not generate sitemap for a route**:
> ```typescript
> export const handle: SEOHandle = {
>   getSitemapEntries: () => null,
> };
> ```

> **To generate sitemap for dynamic routes**:
> ```typescript
> export const handle: SEOHandle = {
>   getSitemapEntries: async (request) => {
>     const blogs = await db.blog.findMany();
>     return blogs.map((blog) => {
>       return { route: `/blog/${blog.slug}`, priority: 0.7 };
>     });
>   },
> };
> ```

**关键推论**：
- **显式声明 `getSitemapEntries: () => null`** = 确定排除
- **无 SEOHandle** = 按路由类型自动处理
- **动态路由（无 `getSitemapEntries`）** = 不会生成具体实例（如 `/users/kody`），但路由模式（如 `/users/:username`）可能出现

---

## 二、项目路由逐条判定结果

### 2.1 路由总览

根据 `docs/routing.md:89-201` 中 `npx react-router routes` 的输出，项目共有 **39 个可访问路由**（不含布局路由）：

| 类别 | 路由数量 |
|------|---------|
| 营销页面 | 5 |
| 用户内容页面 | 6 (含 4 个动态) |
| 认证相关 | 11 (含 4 个动态) |
| 设置页面 | 11 |
| 管理后台 | 4 (含 2 个动态) |
| 资源路由 | 4 |
| 其他 | 2 |

### 2.2 已显式排除的路由（17 个，确定不会出现在 sitemap）

这些路由均声明了 `getSitemapEntries: () => null`，**确定不会**出现在 sitemap 中：

#### 认证相关（5 个）

| 路由文件 | URL 路径 | 代码证据行 |
|---------|---------|-----------|
| `_auth/login.tsx` | `/login` | L25: `getSitemapEntries: () => null` |
| `_auth/signup.tsx` | `/signup` | L24: `getSitemapEntries: () => null` |
| `_auth/forgot-password.tsx` | `/forgot-password` | L18: `getSitemapEntries: () => null` |
| `_auth/reset-password.tsx` | `/reset-password` | L18: `getSitemapEntries: () => null` |
| `_auth/verify.tsx` | `/verify` | L16: `getSitemapEntries: () => null` |

#### 设置相关（11 个）

| 路由文件 | URL 路径 | 代码证据行 |
|---------|---------|-----------|
| `settings/profile/index.tsx` | `/settings/profile` | L21: `getSitemapEntries: () => null` |
| `settings/profile/_layout.tsx` | 布局路由 | L18: `getSitemapEntries: () => null` |
| `settings/profile/change-email.tsx` | `/settings/profile/change-email` | L25: `getSitemapEntries: () => null` |
| `settings/profile/connections.tsx` | `/settings/profile/connections` | L31: `getSitemapEntries: () => null` |
| `settings/profile/password.tsx` | `/settings/profile/password` | L25: `getSitemapEntries: () => null` |
| `settings/profile/password_.create.tsx` | `/settings/profile/password/create` | L22: `getSitemapEntries: () => null` |
| `settings/profile/photo.tsx` | `/settings/profile/photo` | L26: `getSitemapEntries: () => null` |
| `settings/profile/two-factor/_layout.tsx` | 布局路由 | L9: `getSitemapEntries: () => null` |
| `settings/profile/two-factor/index.tsx` | `/settings/profile/two-factor` | L13: `getSitemapEntries: () => null` |
| `settings/profile/two-factor/disable.tsx` | `/settings/profile/two-factor/disable` | L16: `getSitemapEntries: () => null` |
| `settings/profile/two-factor/verify.tsx` | `/settings/profile/two-factor/verify` | L22: `getSitemapEntries: () => null` |

#### 管理后台（1 个）

| 路由文件 | URL 路径 | 代码证据行 |
|---------|---------|-----------|
| `admin/cache/index.tsx` | `/admin/cache` | L31: `getSitemapEntries: () => null` |

### 2.3 未显式排除的路由（22 个，按类型分析）

这些路由**没有**声明 `getSitemapEntries: () => null`，需要按路由类型分析：

#### 2.3.1 营销页面（5 个，静态路由）

**确定会出现在 sitemap 中**（无 SEOHandle 的静态路由）

| 路由文件 | URL 路径 | 判定依据 |
|---------|---------|---------|
| `_marketing/index.tsx` | `/` (首页) | 无 `export const handle`，静态路由 |
| `_marketing/about.tsx` | `/about` | 无 `export const handle`，静态路由 |
| `_marketing/privacy.tsx` | `/privacy` | 无 `export const handle`，静态路由 |
| `_marketing/support.tsx` | `/support` | 无 `export const handle`，静态路由 |
| `_marketing/tos.tsx` | `/tos` | 无 `export const handle`，静态路由 |

**代码验证**：搜索 `app/routes/_marketing/` 下的文件，确认**没有** `export const handle` 声明。

#### 2.3.2 用户列表页（1 个，静态路由）

**确定会出现在 sitemap 中**

| 路由文件 | URL 路径 | 判定依据 |
|---------|---------|---------|
| `users/index.tsx` | `/users` | 无 `export const handle`，静态路由 |

#### 2.3.3 动态路由（10 个，含参数）

**路由模式可能出现，但不会生成具体实例**

| 路由文件 | URL 路径模式 | 具体实例情况 |
|---------|-------------|-------------|
| `users/$username/index.tsx` | `/users/:username` | 不会自动生成 `/users/kody` 等具体 URL |
| `users/$username/notes/_layout.tsx` | 布局路由 | 不会出现在 sitemap（布局路由不渲染页面） |
| `users/$username/notes/index.tsx` | `/users/:username/notes` | 不会自动生成具体实例 |
| `users/$username/notes/$noteId.tsx` | `/users/:username/notes/:noteId` | 不会自动生成具体实例 |
| `users/$username/notes/$noteId_.edit.tsx` | `/users/:username/notes/:noteId/edit` | 不会自动生成具体实例 |
| `users/$username/notes/new.tsx` | `/users/:username/notes/new` | 不会自动生成具体实例 |
| `_auth/auth.$provider/index.ts` | `/auth/:provider` | 不会自动生成 `/auth/github` 等具体 URL |
| `_auth/auth.$provider/callback.ts` | `/auth/:provider/callback` | 不会自动生成具体实例 |
| `admin/cache/lru.$cacheKey.ts` | `/admin/cache/lru/:cacheKey` | 不会自动生成具体实例 |
| `admin/cache/sqlite.$cacheKey.ts` | `/admin/cache/sqlite/:cacheKey` | 不会自动生成具体实例 |

**关键理解**：
- `generateSitemap` 接收的是 `context.serverBuild.routes`（编译时路由模式）
- 对于动态路由，它只知道 `/users/:username`，不知道具体有哪些 username
- **除非**手动实现 `getSitemapEntries` 从数据库查询

#### 2.3.4 其他未排除的静态路由（6 个）

**确定会出现在 sitemap 中**（无 SEOHandle 的静态路由）

| 路由文件 | URL 路径 | 判定依据 | 代码证据 |
|---------|---------|---------|---------|
| `$.tsx` | `/*` (404) | 无 `export const handle`，静态路由 | 搜索确认无 handle |
| `me.tsx` | `/me` | 无 `export const handle`，静态路由 | 搜索确认无 handle |
| `_auth/logout.tsx` | `/logout` | 无 `export const handle`，静态路由 | 搜索确认无 handle |
| `admin/cache/sqlite.tsx` | `/admin/cache/sqlite` | 无 `export const handle`，静态路由 | 搜索确认无 handle |
| `resources/healthcheck.tsx` | `/resources/healthcheck` | 无 `export const handle`，静态路由 | 搜索确认无 handle |
| `resources/download-user-data.tsx` | `/resources/download-user-data` | 无 `export const handle`，静态路由 | 搜索确认无 handle |
| `resources/images.tsx` | `/resources/images` | 无 `export const handle`，静态路由 | 搜索确认无 handle |
| `resources/theme-switch.tsx` | `/resources/theme-switch` | 无 `export const handle`，静态路由 | 搜索确认无 handle |
| `settings/profile/passkeys.tsx` | `/settings/profile/passkeys` | 有 handle，但**无** `getSitemapEntries` | L12-14: 只有 `breadcrumb` 属性 |
| `_auth/onboarding/index.tsx` | `/onboarding` | 无 `export const handle`，静态路由 | 搜索确认无 handle |
| `_auth/onboarding/$provider.tsx` | `/onboarding/:provider` | 动态路由，无实例 | 无 SEOHandle |
| `_auth/webauthn/authentication.ts` | `/webauthn/authentication` | 无 `export const handle`，静态路由 | 搜索确认无 handle |
| `_auth/webauthn/registration.ts` | `/webauthn/registration` | 无 `export const handle`，静态路由 | 搜索确认无 handle |

### 2.4 Sitemap 实际内容预测（可验证）

根据以上分析，**实际生成的 sitemap.xml 将包含**：

#### 确定会出现的静态路由（15 个）

| URL 路径 | 来源文件 | 出现原因 |
|---------|---------|---------|
| `/` | `_marketing/index.tsx` | 静态路由，无 SEOHandle |
| `/about` | `_marketing/about.tsx` | 静态路由，无 SEOHandle |
| `/privacy` | `_marketing/privacy.tsx` | 静态路由，无 SEOHandle |
| `/support` | `_marketing/support.tsx` | 静态路由，无 SEOHandle |
| `/tos` | `_marketing/tos.tsx` | 静态路由，无 SEOHandle |
| `/users` | `users/index.tsx` | 静态路由，无 SEOHandle |
| `/*` (404) | `$.tsx` | 静态路由，无 SEOHandle |
| `/me` | `me.tsx` | 静态路由，无 SEOHandle |
| `/logout` | `_auth/logout.tsx` | 静态路由，无 SEOHandle |
| `/onboarding` | `_auth/onboarding/index.tsx` | 静态路由，无 SEOHandle |
| `/webauthn/authentication` | `_auth/webauthn/authentication.ts` | 静态路由，无 SEOHandle |
| `/webauthn/registration` | `_auth/webauthn/registration.ts` | 静态路由，无 SEOHandle |
| `/admin/cache/sqlite` | `admin/cache/sqlite.tsx` | 静态路由，无 SEOHandle |
| `/settings/profile/passkeys` | `settings/profile/passkeys.tsx` | 有 handle 但无 getSitemapEntries |
| `/resources/healthcheck` | `resources/healthcheck.tsx` | 静态路由，无 SEOHandle |
| `/resources/download-user-data` | `resources/download-user-data.tsx` | 静态路由，无 SEOHandle |
| `/resources/images` | `resources/images.tsx` | 静态路由，无 SEOHandle |
| `/resources/theme-switch` | `resources/theme-switch.tsx` | 静态路由，无 SEOHandle |

#### 可能出现的动态路由模式（10 个）

这些是**路由模式**，不是具体实例：

| URL 路径模式 | 说明 |
|-------------|------|
| `/users/:username` | 不会有 `/users/kody` 等具体条目 |
| `/users/:username/notes` | 同上 |
| `/users/:username/notes/:noteId` | 同上 |
| `/users/:username/notes/:noteId/edit` | 同上 |
| `/users/:username/notes/new` | 同上 |
| `/auth/:provider` | 不会有 `/auth/github` 等具体条目 |
| `/auth/:provider/callback` | 同上 |
| `/onboarding/:provider` | 同上 |
| `/admin/cache/lru/:cacheKey` | 同上 |
| `/admin/cache/sqlite/:cacheKey` | 同上 |

**注意**：不同的 `remix-seo` 实现版本可能对动态路由模式的处理不同（是否将 `:param` 原样输出或过滤）。但**可以确定**：不会生成具体参数值的条目。

#### 确定不会出现的路由（17 个 + 布局路由）

- 所有声明 `getSitemapEntries: () => null` 的 17 个路由
- 所有 `_layout.tsx` 布局路由

### 2.5 需要从 sitemap 排除但当前未排除的路由（13 个）

以下路由**不应该**出现在 sitemap 中，但当前**没有** `getSitemapEntries: () => null` 声明：

| 路由文件 | URL 路径 | 应该排除的原因 | 代码验证（无排除声明） |
|---------|---------|---------------|---------------------|
| `$.tsx` | `/*` (404) | 404 兜底路由 | 无 `export const handle` |
| `me.tsx` | `/me` | 重定向到 `/users/:username` | 无 `export const handle` |
| `_auth/logout.tsx` | `/logout` | 仅执行 action，无实质内容 | 无 `export const handle` |
| `_auth/onboarding/index.tsx` | `/onboarding` | 首次登录引导（需要特殊状态） | 无 `export const handle` |
| `_auth/onboarding/$provider.tsx` | `/onboarding/:provider` | 首次登录引导 | 无 `export const handle` |
| `_auth/webauthn/authentication.ts` | `/webauthn/authentication` | 仅返回 JSON | 无 `export const handle` |
| `_auth/webauthn/registration.ts` | `/webauthn/registration` | 仅返回 JSON | 无 `export const handle` |
| `admin/cache/sqlite.tsx` | `/admin/cache/sqlite` | 管理后台 | 无 `export const handle` |
| `settings/profile/passkeys.tsx` | `/settings/profile/passkeys` | 设置页面（同组其他已排除） | 有 handle 但无 `getSitemapEntries` |
| `resources/healthcheck.tsx` | `/resources/healthcheck` | 返回 "OK" 文本，无 UI | 无 `export const handle` |
| `resources/download-user-data.tsx` | `/resources/download-user-data` | 需要登录 | 无 `export const handle` |
| `resources/images.tsx` | `/resources/images` | 图片处理 API | 无 `export const handle` |
| `resources/theme-switch.tsx` | `/resources/theme-switch` | 仅处理 POST action | 无 `export const handle` |

---

## 三、ALLOW_INDEXING 三层行为一致性分析

### 3.1 各层代码位置与逻辑

#### 层 1：HTML 文档层（root.tsx）

**文件**: `app/root.tsx:148`

```typescript
const allowIndexing = ENV.ALLOW_INDEXING !== 'false'
// ...
{allowIndexing ? null : (
  <meta name="robots" content="noindex, nofollow" />
)}
```

**判断逻辑**：
- `ENV.ALLOW_INDEXING === 'false'` → 渲染 `noindex, nofollow`
- 其他情况（`'true'` 或 `undefined`）→ 不渲染标签（默认允许）

**验证方式**：可以通过设置环境变量并访问任何页面检查 HTML 源码。

#### 层 2：Sitemap 资源路由（sitemap[.]xml.ts）

**文件**: `app/routes/_seo/sitemap[.]xml.ts`

```typescript
export async function loader({ request, context }: Route.LoaderArgs) {
	// @ts-expect-error
	return generateSitemap(request, context.serverBuild.routes, {
		siteUrl: getDomainUrl(request),
		headers: {
			'Cache-Control': `public, max-age=${60 * 5}`,
		},
	})
}
```

**关键发现**：**完全没有** `process.env.ALLOW_INDEXING` 或 `ENV.ALLOW_INDEXING` 的检查。

**行为**：无论 `ALLOW_INDEXING` 值是什么，`generateSitemap` 都会正常运行并输出完整的 sitemap。

#### 层 3：Robots.txt 资源路由（robots[.]txt.ts）

**文件**: `app/routes/_seo/robots[.]txt.ts`

```typescript
export function loader({ request }: Route.LoaderArgs) {
	return generateRobotsTxt([
		{ type: 'sitemap', value: `${getDomainUrl(request)}/sitemap.xml` },
	])
}
```

**关键发现**：**完全没有** `process.env.ALLOW_INDEXING` 的检查。

**默认策略**（来自 `@nasa-gcn/remix-seo` 文档）：
```
User-agent: *
Allow: /
Sitemap: https://your-domain.com/sitemap.xml
```

### 3.2 行为一致性对比表

| 场景 | ALLOW_INDEXING 值 | HTML 层行为 | Sitemap 层行为 | Robots.txt 层行为 | 是否一致 |
|------|-------------------|------------|---------------|------------------|---------|
| 生产环境（默认） | `undefined` 或 `'true'` | 不渲染 noindex 标签（允许） | 正常输出完整 sitemap | 输出 `Allow: /` + sitemap 引用 | ✅ 一致（都允许） |
| 开发/预览环境 | `'false'` | 渲染 `noindex, nofollow` 标签 | ❌ 仍输出完整 sitemap | ❌ 仍输出 `Allow: /` | ❌ **不一致** |

### 3.3 不一致的具体影响

当 `ALLOW_INDEXING="false"` 时：

**搜索引擎视角**：
1. 访问 `/robots.txt` → 看到 `User-agent: *` + `Allow: /` + `Sitemap: /sitemap.xml`
2. 访问 `/sitemap.xml` → 看到所有路由的完整列表
3. 访问具体页面 → 看到 `<meta name="robots" content="noindex, nofollow" />`

**实际结果**：
- ✅ 页面级：不会被索引（因为 HTML 有 noindex）
- ⚠️ 发现渠道：搜索引擎仍能通过 sitemap 发现所有页面
- ⚠️ 资源浪费：搜索引擎可能会抓取这些页面（虽然最终不索引）

**更严重的问题**：部分路由本身就不应该被搜索引擎发现：
- `/admin/cache/sqlite` → 管理后台
- `/resources/download-user-data` → 需要登录
- `/onboarding` → 需要特殊状态

### 3.4 一致性验证（可执行测试）

可以通过以下步骤验证三层行为：

```bash
# 1. 设置环境变量
export ALLOW_INDEXING="false"

# 2. 启动应用后检查各层：

# 层 1: HTML 层（访问任何页面查看源码）
# 期望: <meta name="robots" content="noindex, nofollow">
# 实际: ✅ 会有

# 层 2: Sitemap 层
curl http://localhost:3000/sitemap.xml
# 期望（一致的行为）: 空 urlset 或 404
# 实际（当前代码）: ❌ 完整的 sitemap

# 层 3: Robots.txt 层
curl http://localhost:3000/robots.txt
# 期望（一致的行为）: User-agent: *\nDisallow: /
# 实际（当前代码）: ❌ User-agent: *\nAllow: /\nSitemap: ...
```

---

## 四、代码证据索引

### 4.1 已排除路由的代码证据

| 路由 | 代码位置 | 证据 |
|-----|---------|------|
| `/login` | `app/routes/_auth/login.tsx:25-27` | `getSitemapEntries: () => null` |
| `/signup` | `app/routes/_auth/signup.tsx:24-26` | `getSitemapEntries: () => null` |
| `/forgot-password` | `app/routes/_auth/forgot-password.tsx:18-20` | `getSitemapEntries: () => null` |
| `/reset-password` | `app/routes/_auth/reset-password.tsx:18-20` | `getSitemapEntries: () => null` |
| `/verify` | `app/routes/_auth/verify.tsx:16-18` | `getSitemapEntries: () => null` |
| `/settings/profile` | `app/routes/settings/profile/index.tsx:21-23` | `getSitemapEntries: () => null` |
| `/settings/profile/change-email` | `app/routes/settings/profile/change-email.tsx:23-26` | `getSitemapEntries: () => null` |
| `/settings/profile/connections` | `app/routes/settings/profile/connections.tsx:29-32` | `getSitemapEntries: () => null` |
| `/settings/profile/password` | `app/routes/settings/profile/password.tsx:23-26` | `getSitemapEntries: () => null` |
| `/settings/profile/password/create` | `app/routes/settings/profile/password_.create.tsx:20-23` | `getSitemapEntries: () => null` |
| `/settings/profile/photo` | `app/routes/settings/profile/photo.tsx:24-27` | `getSitemapEntries: () => null` |
| `/settings/profile/two-factor` | `app/routes/settings/profile/two-factor/index.tsx:12-14` | `getSitemapEntries: () => null` |
| `/settings/profile/two-factor/disable` | `app/routes/settings/profile/two-factor/disable.tsx:14-17` | `getSitemapEntries: () => null` |
| `/settings/profile/two-factor/verify` | `app/routes/settings/profile/two-factor/verify.tsx:20-23` | `getSitemapEntries: () => null` |
| `/admin/cache` | `app/routes/admin/cache/index.tsx:30-32` | `getSitemapEntries: () => null` |
| `settings/profile/_layout.tsx` | `app/routes/settings/profile/_layout.tsx:16-19` | `getSitemapEntries: () => null` |
| `settings/profile/two-factor/_layout.tsx` | `app/routes/settings/profile/two-factor/_layout.tsx:7-10` | `getSitemapEntries: () => null` |

### 4.2 未排除路由的代码证据（通过"无声明"证明）

| 路由 | 验证方式 | 结果 |
|-----|---------|------|
| `/` | 搜索 `app/routes/_marketing/index.tsx` 中 `export const handle` | 无 |
| `/about` | 搜索 `app/routes/_marketing/about.tsx` 中 `export const handle` | 无 |
| `/privacy` | 搜索 `app/routes/_marketing/privacy.tsx` 中 `export const handle` | 无 |
| `/support` | 搜索 `app/routes/_marketing/support.tsx` 中 `export const handle` | 无 |
| `/tos` | 搜索 `app/routes/_marketing/tos.tsx` 中 `export const handle` | 无 |
| `/users` | 搜索 `app/routes/users/index.tsx` 中 `export const handle` | 无 |
| `/me` | 搜索 `app/routes/me.tsx` 中 `export const handle` | 无 |
| `/logout` | 搜索 `app/routes/_auth/logout.tsx` 中 `export const handle` | 无 |
| `/*` (404) | 搜索 `app/routes/$.tsx` 中 `export const handle` | 无 |
| `/onboarding` | 搜索 `app/routes/_auth/onboarding/index.tsx` 中 `export const handle` | 无 |
| `/webauthn/authentication` | 搜索 `app/routes/_auth/webauthn/authentication.ts` 中 `export const handle` | 无 |
| `/webauthn/registration` | 搜索 `app/routes/_auth/webauthn/registration.ts` 中 `export const handle` | 无 |
| `/admin/cache/sqlite` | 搜索 `app/routes/admin/cache/sqlite.tsx` 中 `export const handle` | 无 |
| `/settings/profile/passkeys` | 搜索 `app/routes/settings/profile/passkeys.tsx:12-14` | 有 handle 但**无** `getSitemapEntries` |
| `/resources/healthcheck` | 搜索 `app/routes/resources/healthcheck.tsx` 中 `export const handle` | 无 |
| `/resources/download-user-data` | 搜索 `app/routes/resources/download-user-data.tsx` 中 `export const handle` | 无 |
| `/resources/images` | 搜索 `app/routes/resources/images.tsx` 中 `export const handle` | 无 |
| `/resources/theme-switch` | 搜索 `app/routes/resources/theme-switch.tsx` 中 `export const handle` | 无 |

### 4.3 ALLOW_INDEXING 三层代码位置

| 层面 | 文件 | 关键行 | 检查 ALLOW_INDEXING |
|------|------|-------|---------------------|
| HTML 层 | `app/root.tsx` | L148 | ✅ 检查 |
| Sitemap 层 | `app/routes/_seo/sitemap[.]xml.ts` | 全部 | ❌ **不检查** |
| Robots.txt 层 | `app/routes/_seo/robots[.]txt.ts` | 全部 | ❌ **不检查** |

---

## 五、修正建议（带代码）

### 5.1 修正三层一致性

**修改 `app/routes/_seo/sitemap[.]xml.ts`**:

```typescript
import { generateSitemap } from '@nasa-gcn/remix-seo'
import { getDomainUrl } from '#app/utils/misc.tsx'
import { type Route } from './+types/sitemap[.]xml.ts'

export async function loader({ request, context }: Route.LoaderArgs) {
	const allowIndexing = process.env.ALLOW_INDEXING !== 'false'
	
	if (!allowIndexing) {
		// 返回空的 sitemap
		return new Response(
			'<?xml version="1.0" encoding="UTF-8"?>\n' +
			'<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">\n' +
			'</urlset>',
			{
				headers: {
					'Content-Type': 'application/xml',
					'Cache-Control': `public, max-age=${60 * 5}`,
				},
			}
		)
	}
	
	// @ts-expect-error
	return generateSitemap(request, context.serverBuild.routes, {
		siteUrl: getDomainUrl(request),
		headers: {
			'Cache-Control': `public, max-age=${60 * 5}`,
		},
	})
}
```

**修改 `app/routes/_seo/robots[.]txt.ts`**:

```typescript
import { generateRobotsTxt } from '@nasa-gcn/remix-seo'
import { getDomainUrl } from '#app/utils/misc.tsx'
import { type Route } from './+types/robots[.]txt.ts'

export function loader({ request }: Route.LoaderArgs) {
	const allowIndexing = process.env.ALLOW_INDEXING !== 'false'
	
	if (!allowIndexing) {
		// 禁止所有抓取
		return generateRobotsTxt([
			{ type: 'userAgent', value: '*' },
			{ type: 'disallow', value: '/' },
		])
	}
	
	return generateRobotsTxt([
		{ type: 'sitemap', value: `${getDomainUrl(request)}/sitemap.xml` },
	])
}
```

### 5.2 为遗漏的路由添加排除声明

以下路由需要添加 `getSitemapEntries: () => null`：

**示例模板**:
```typescript
import { type SEOHandle } from '@nasa-gcn/remix-seo'

// 如果已有 handle，添加 getSitemapEntries 属性
export const handle = {
	// 现有属性...
	breadcrumb: <Icon name="passkey">Passkeys</Icon>,
	// 新增
	getSitemapEntries: () => null,
}

// 如果没有 handle，新增完整声明
export const handle: SEOHandle = {
	getSitemapEntries: () => null,
}
```

**需要修改的文件列表**:

1. `app/routes/$.tsx` - 404 兜底路由
2. `app/routes/me.tsx` - 重定向路由
3. `app/routes/_auth/logout.tsx` - 登出
4. `app/routes/_auth/onboarding/index.tsx` - 首次登录引导
5. `app/routes/_auth/onboarding/$provider.tsx` - 首次登录引导
6. `app/routes/_auth/webauthn/authentication.ts` - WebAuthn API
7. `app/routes/_auth/webauthn/registration.ts` - WebAuthn API
8. `app/routes/admin/cache/sqlite.tsx` - 管理后台
9. `app/routes/settings/profile/passkeys.tsx` - 设置页面（已有 handle，需加属性）
10. `app/routes/resources/healthcheck.tsx` - 资源路由
11. `app/routes/resources/download-user-data.tsx` - 资源路由
12. `app/routes/resources/images.tsx` - 资源路由
13. `app/routes/resources/theme-switch.tsx` - 资源路由

### 5.3 修正后的一致性验证

```bash
# 设置 ALLOW_INDEXING="false" 后：

# 层 1: HTML
curl -s http://localhost:3000/ | grep -o '<meta name="robots" content="[^"]*"'
# 期望: <meta name="robots" content="noindex, nofollow">

# 层 2: Sitemap
curl -s http://localhost:3000/sitemap.xml
# 期望: 空的 <urlset></urlset>

# 层 3: Robots.txt
curl -s http://localhost:3000/robots.txt
# 期望:
# User-agent: *
# Disallow: /
```

---

## 六、总结

### 6.1 路由暴露边界（已确认）

| 类别 | 总数 | 已排除 | 未排除（会/可能出现） |
|------|------|--------|----------------------|
| 营销页面 | 5 | 0 | **5** (全部静态) |
| 用户内容页面 | 6 | 0 | **6** (2 静态 + 4 动态模式) |
| 认证相关 | 11 | 5 | **6** (3 静态 + 3 动态模式) |
| 设置页面 | 11 | 10 | **1** (passkeys) |
| 管理后台 | 4 | 1 | **3** (1 静态 + 2 动态模式) |
| 资源路由 | 4 | 0 | **4** (全部静态) |
| 其他 | 2 | 0 | **2** (404 + /me) |
| **总计** | **43** | **16** | **27** |

**实际会出现在 sitemap 的路由**：
- 静态路由：18 个（5 营销 + 1 用户列表 + 12 其他）
- 动态路由模式：9 个（不会有具体实例）

**应该排除但未排除的路由**：13 个

### 6.2 ALLOW_INDEXING 行为一致性（已确认）

| 层面 | 受 ALLOW_INDEXING 控制 | 代码位置 |
|------|----------------------|---------|
| HTML 层 | ✅ 是 | `app/root.tsx:148` |
| Sitemap 层 | ❌ **否** | `app/routes/_seo/sitemap[.]xml.ts` |
| Robots.txt 层 | ❌ **否** | `app/routes/_seo/robots[.]txt.ts` |

**关键发现**：三层行为**不一致**。当 `ALLOW_INDEXING="false"` 时：
- HTML 页面有 `noindex` 标签
- 但 sitemap 和 robots.txt 仍然正常输出

### 6.3 问题优先级

| 优先级 | 问题 | 影响范围 |
|-------|------|---------|
| 🔴 P0 | ALLOW_INDEXING 不控制 sitemap/robots | 所有环境 |
| 🔴 P0 | 13 个敏感路由未从 sitemap 排除 | 生产环境 |
| 🟡 P1 | 动态路由不会自动生成具体实例（设计行为，但需要文档说明） | 开发者认知 |
