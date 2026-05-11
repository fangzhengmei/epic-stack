# Epic Stack SEO 索引控制机制分析报告 (v5)

## 统计口径

本报告的所有统计均基于以下**可验证的代码证据**：

1. **路由配置来源**：`docs/routing.md:90-200` 中 `npx react-router routes` 的完整输出，共 **48 个路由**
2. **SEOHandle 声明来源**：`app/routes/` 目录下所有文件的 `export const handle` 声明
3. **generateSitemap 行为**：`@nasa-gcn/remix-seo` 官方文档

---

## 一、统计汇总

### 1.1 唯一统计口径

| 统计项 | 数量 | 说明 |
|-------|------|------|
| 总路由数 | 48 | 来自 `docs/routing.md:90-200` 的 `<Routes>` 输出 |
| 确定不会进入 sitemap | 17 | 声明 `getSitemapEntries: () => null` |
| 确定会进入 sitemap | 28 | 无排除声明的页面路由 |
| 布局路由（未排除） | 1 | `users/$username/notes/_layout.tsx` |
| 资源路由本身 | 2 | 处理 `/robots.txt` 和 `/sitemap.xml` 的路由 |

### 1.2 数字验证

```
总路由数: 48
├── 确定不会进入 sitemap: 17
├── 确定会进入 sitemap: 28
├── 布局路由（未排除）: 1
└── 资源路由本身: 2

验证: 17 + 28 + 1 + 2 = 48 ✓
```

---

## 二、确定不会进入 sitemap 的路由（17 个）

判定依据：路由文件中声明了 `getSitemapEntries: () => null`

### 2.1 认证相关（5 个）

| 序号 | URL 路径 | 路由文件 | 代码证据 |
|------|---------|---------|---------|
| 1 | `/forgot-password` | `app/routes/_auth/forgot-password.tsx` | L18-20: `getSitemapEntries: () => null` |
| 2 | `/login` | `app/routes/_auth/login.tsx` | L25-27: `getSitemapEntries: () => null` |
| 3 | `/reset-password` | `app/routes/_auth/reset-password.tsx` | L18-20: `getSitemapEntries: () => null` |
| 4 | `/signup` | `app/routes/_auth/signup.tsx` | L24-26: `getSitemapEntries: () => null` |
| 5 | `/verify` | `app/routes/_auth/verify.tsx` | L16-18: `getSitemapEntries: () => null` |

### 2.2 设置相关（11 个）

| 序号 | URL 路径 | 路由文件 | 代码证据 |
|------|---------|---------|---------|
| 6 | `/settings/profile` | `app/routes/settings/profile/_layout.tsx` | L16-19: `getSitemapEntries: () => null` |
| 7 | `/settings/profile/change-email` | `app/routes/settings/profile/change-email.tsx` | L23-26: `getSitemapEntries: () => null` |
| 8 | `/settings/profile/connections` | `app/routes/settings/profile/connections.tsx` | L29-32: `getSitemapEntries: () => null` |
| 9 | `/settings/profile` (index) | `app/routes/settings/profile/index.tsx` | L21-23: `getSitemapEntries: () => null` |
| 10 | `/settings/profile/password` | `app/routes/settings/profile/password.tsx` | L23-26: `getSitemapEntries: () => null` |
| 11 | `/settings/profile/password/create` | `app/routes/settings/profile/password_.create.tsx` | L20-23: `getSitemapEntries: () => null` |
| 12 | `/settings/profile/photo` | `app/routes/settings/profile/photo.tsx` | L24-27: `getSitemapEntries: () => null` |
| 13 | `/settings/profile/two-factor` | `app/routes/settings/profile/two-factor/_layout.tsx` | L7-10: `getSitemapEntries: () => null` |
| 14 | `/settings/profile/two-factor/disable` | `app/routes/settings/profile/two-factor/disable.tsx` | L14-17: `getSitemapEntries: () => null` |
| 15 | `/settings/profile/two-factor` (index) | `app/routes/settings/profile/two-factor/index.tsx` | L12-14: `getSitemapEntries: () => null` |
| 16 | `/settings/profile/two-factor/verify` | `app/routes/settings/profile/two-factor/verify.tsx` | L20-23: `getSitemapEntries: () => null` |

### 2.3 管理后台（1 个）

| 序号 | URL 路径 | 路由文件 | 代码证据 |
|------|---------|---------|---------|
| 17 | `/admin/cache` | `app/routes/admin/cache/index.tsx` | L30-32: `getSitemapEntries: () => null` |

### 2.4 本节统计验证

```
确定不会进入 sitemap: 17
├── 认证相关: 5
├── 设置相关: 11
└── 管理后台: 1

验证: 5 + 11 + 1 = 17 ✓
```

---

## 三、确定会进入 sitemap 的路由（28 个）

判定依据：路由文件中**没有**声明 `getSitemapEntries: () => null`，且不是布局路由或资源路由本身。

根据 `@nasa-gcn/remix-seo` 官方文档：
- 没有 `getSitemapEntries` 声明的路由 → 按默认规则处理
- 静态路由（不含 `:param`）→ 路由模式会出现在 sitemap 中
- 动态路由（含 `:param`）→ 路由模式会出现在 sitemap 中，但**不会**生成具体参数值的实例（如 `/users/kody`）

### 3.1 营销页面（5 个，全静态）

| 序号 | URL 路径 | 路由文件 | 判定依据 |
|------|---------|---------|---------|
| 1 | `/` (首页) | `app/routes/_marketing/index.tsx` | 无 `export const handle` 声明 |
| 2 | `/about` | `app/routes/_marketing/about.tsx` | 无 `export const handle` 声明 |
| 3 | `/privacy` | `app/routes/_marketing/privacy.tsx` | 无 `export const handle` 声明 |
| 4 | `/support` | `app/routes/_marketing/support.tsx` | 无 `export const handle` 声明 |
| 5 | `/tos` | `app/routes/_marketing/tos.tsx` | 无 `export const handle` 声明 |

### 3.2 用户内容路由（6 个）

| 序号 | URL 路径/模式 | 路由文件 | 路由类型 | 判定依据 |
|------|--------------|---------|---------|---------|
| 6 | `/users` | `app/routes/users/index.tsx` | 静态 | 无 `export const handle` 声明 |
| 7 | `/users/:username` | `app/routes/users/$username/index.tsx` | 动态 | 无 `export const handle` 声明 |
| 8 | `/users/:username/notes` | `app/routes/users/$username/notes/index.tsx` | 动态 | 无 `export const handle` 声明 |
| 9 | `/users/:username/notes/:noteId` | `app/routes/users/$username/notes/$noteId.tsx` | 动态 | 无 `export const handle` 声明 |
| 10 | `/users/:username/notes/:noteId/edit` | `app/routes/users/$username/notes/$noteId_.edit.tsx` | 动态 | 无 `export const handle` 声明 |
| 11 | `/users/:username/notes/new` | `app/routes/users/$username/notes/new.tsx` | 动态 | 无 `export const handle` 声明 |

### 3.3 认证相关（6 个）

| 序号 | URL 路径/模式 | 路由文件 | 路由类型 | 判定依据 |
|------|--------------|---------|---------|---------|
| 12 | `/*` (404) | `app/routes/$.tsx` | 静态 | 无 `export const handle` 声明 |
| 13 | `/auth/:provider` | `app/routes/_auth/auth.$provider/index.ts` | 动态 | 无 `export const handle` 声明 |
| 14 | `/auth/:provider/callback` | `app/routes/_auth/auth.$provider/callback.ts` | 动态 | 无 `export const handle` 声明 |
| 15 | `/logout` | `app/routes/_auth/logout.tsx` | 静态 | 无 `export const handle` 声明 |
| 16 | `/onboarding` | `app/routes/_auth/onboarding/index.tsx` | 静态 | 无 `export const handle` 声明 |
| 17 | `/onboarding/:provider` | `app/routes/_auth/onboarding/$provider.tsx` | 动态 | 无 `export const handle` 声明 |

### 3.4 WebAuthn 相关（2 个，全静态）

| 序号 | URL 路径 | 路由文件 | 判定依据 |
|------|---------|---------|---------|
| 18 | `/webauthn/authentication` | `app/routes/_auth/webauthn/authentication.ts` | 无 `export const handle` 声明 |
| 19 | `/webauthn/registration` | `app/routes/_auth/webauthn/registration.ts` | 无 `export const handle` 声明 |

### 3.5 管理后台（3 个）

| 序号 | URL 路径/模式 | 路由文件 | 路由类型 | 判定依据 |
|------|--------------|---------|---------|---------|
| 20 | `/admin/cache/lru/:cacheKey` | `app/routes/admin/cache/lru.$cacheKey.ts` | 动态 | 无 `export const handle` 声明 |
| 21 | `/admin/cache/sqlite` | `app/routes/admin/cache/sqlite.tsx` | 静态 | 无 `export const handle` 声明 |
| 22 | `/admin/cache/sqlite/:cacheKey` | `app/routes/admin/cache/sqlite.$cacheKey.ts` | 动态 | 无 `export const handle` 声明 |

### 3.6 设置页面（1 个，静态）

| 序号 | URL 路径 | 路由文件 | 判定依据 | 代码证据 |
|------|---------|---------|---------|---------|
| 23 | `/settings/profile/passkeys` | `app/routes/settings/profile/passkeys.tsx` | 有 `export const handle` 但无 `getSitemapEntries` | L12-14: 只有 `breadcrumb` 属性 |

### 3.7 资源路由（4 个，全静态）

| 序号 | URL 路径 | 路由文件 | 判定依据 |
|------|---------|---------|---------|
| 24 | `/resources/healthcheck` | `app/routes/resources/healthcheck.tsx` | 无 `export const handle` 声明 |
| 25 | `/resources/download-user-data` | `app/routes/resources/download-user-data.tsx` | 无 `export const handle` 声明 |
| 26 | `/resources/images` | `app/routes/resources/images.tsx` | 无 `export const handle` 声明 |
| 27 | `/resources/theme-switch` | `app/routes/resources/theme-switch.tsx` | 无 `export const handle` 声明 |

### 3.8 其他（1 个，静态）

| 序号 | URL 路径 | 路由文件 | 判定依据 |
|------|---------|---------|---------|
| 28 | `/me` | `app/routes/me.tsx` | 无 `export const handle` 声明 |

### 3.9 本节统计验证

```
确定会进入 sitemap: 28
├── 营销页面: 5
├── 用户内容: 6
├── 认证相关: 6
├── WebAuthn: 2
├── 管理后台: 3
├── 设置页面: 1
├── 资源路由: 4
└── 其他: 1

验证: 5 + 6 + 6 + 2 + 3 + 1 + 4 + 1 = 28 ✓
```

---

## 四、其他路由分类

### 4.1 布局路由（未排除，1 个）

布局路由不渲染具体页面，仅提供布局结构。

| 序号 | 路径 | 路由文件 | 判定依据 |
|------|------|---------|---------|
| 1 | `/users/:username/notes` 布局 | `app/routes/users/$username/notes/_layout.tsx` | 无 `export const handle` 声明，且为布局路由 |

### 4.2 资源路由本身（2 个）

这些路由用于处理 `/robots.txt` 和 `/sitemap.xml` 请求，本身不应出现在 sitemap 中。

| 序号 | URL 路径 | 路由文件 | 处理内容 |
|------|---------|---------|---------|
| 1 | `/robots.txt` | `app/routes/_seo/robots[.]txt.ts` | 生成 robots.txt 内容 |
| 2 | `/sitemap.xml` | `app/routes/_seo/sitemap[.]xml.ts` | 生成 sitemap.xml 内容 |

### 4.3 本节统计验证

```
其他路由分类: 3
├── 布局路由（未排除）: 1
└── 资源路由本身: 2

验证: 1 + 2 = 3 ✓
```

---

## 五、应排除但未排除的路由（14 个）

以下路由**没有**声明 `getSitemapEntries: () => null`，但基于功能特性不应出现在 sitemap 中。

### 5.1 认证流程相关（4 个）

| 序号 | URL 路径/模式 | 路由文件 | 应排除原因 |
|------|--------------|---------|-----------|
| 1 | `/*` (404) | `app/routes/$.tsx` | 404 兜底路由，无有效内容 |
| 2 | `/logout` | `app/routes/_auth/logout.tsx` | 仅执行登出操作，无实质内容 |
| 3 | `/onboarding` | `app/routes/_auth/onboarding/index.tsx` | 首次登录引导，需要特殊状态 |
| 4 | `/onboarding/:provider` | `app/routes/_auth/onboarding/$provider.tsx` | 首次登录引导，需要特殊状态 |

### 5.2 WebAuthn API（2 个）

| 序号 | URL 路径 | 路由文件 | 应排除原因 |
|------|---------|---------|-----------|
| 5 | `/webauthn/authentication` | `app/routes/_auth/webauthn/authentication.ts` | 仅返回 JSON，无 UI |
| 6 | `/webauthn/registration` | `app/routes/_auth/webauthn/registration.ts` | 仅返回 JSON，无 UI |

### 5.3 管理后台（3 个）

| 序号 | URL 路径/模式 | 路由文件 | 应排除原因 |
|------|--------------|---------|-----------|
| 7 | `/admin/cache/lru/:cacheKey` | `app/routes/admin/cache/lru.$cacheKey.ts` | 管理后台，同组 `/admin/cache` 已排除 |
| 8 | `/admin/cache/sqlite` | `app/routes/admin/cache/sqlite.tsx` | 管理后台，同组 `/admin/cache` 已排除 |
| 9 | `/admin/cache/sqlite/:cacheKey` | `app/routes/admin/cache/sqlite.$cacheKey.ts` | 管理后台，同组 `/admin/cache` 已排除 |

### 5.4 设置页面（1 个）

| 序号 | URL 路径 | 路由文件 | 应排除原因 |
|------|---------|---------|-----------|
| 10 | `/settings/profile/passkeys` | `app/routes/settings/profile/passkeys.tsx` | 设置页面，同组其他 10 个已排除 |

### 5.5 资源路由（4 个）

| 序号 | URL 路径 | 路由文件 | 应排除原因 |
|------|---------|---------|-----------|
| 11 | `/resources/healthcheck` | `app/routes/resources/healthcheck.tsx` | 仅返回 "OK" 文本，无 UI |
| 12 | `/resources/download-user-data` | `app/routes/resources/download-user-data.tsx` | 需要登录 |
| 13 | `/resources/images` | `app/routes/resources/images.tsx` | 图片处理 API |
| 14 | `/resources/theme-switch` | `app/routes/resources/theme-switch.tsx` | 仅处理 POST action |

### 5.6 本节统计验证

```
应排除但未排除: 14
├── 认证流程相关: 4
├── WebAuthn API: 2
├── 管理后台: 3
├── 设置页面: 1
└── 资源路由: 4

验证: 4 + 2 + 3 + 1 + 4 = 14 ✓
```

---

## 六、静态与动态路由分类

### 6.1 静态路由（不含 `:param`，17 个）

| 序号 | URL 路径 | 路由文件 | 是否排除 |
|------|---------|---------|---------|
| 1 | `/` (首页) | `_marketing/index.tsx` | 未排除 |
| 2 | `/about` | `_marketing/about.tsx` | 未排除 |
| 3 | `/privacy` | `_marketing/privacy.tsx` | 未排除 |
| 4 | `/support` | `_marketing/support.tsx` | 未排除 |
| 5 | `/tos` | `_marketing/tos.tsx` | 未排除 |
| 6 | `/users` | `users/index.tsx` | 未排除 |
| 7 | `/*` (404) | `$.tsx` | 未排除 |
| 8 | `/logout` | `_auth/logout.tsx` | 未排除 |
| 9 | `/onboarding` | `_auth/onboarding/index.tsx` | 未排除 |
| 10 | `/webauthn/authentication` | `_auth/webauthn/authentication.ts` | 未排除 |
| 11 | `/webauthn/registration` | `_auth/webauthn/registration.ts` | 未排除 |
| 12 | `/admin/cache/sqlite` | `admin/cache/sqlite.tsx` | 未排除 |
| 13 | `/settings/profile/passkeys` | `settings/profile/passkeys.tsx` | 未排除 |
| 14 | `/resources/healthcheck` | `resources/healthcheck.tsx` | 未排除 |
| 15 | `/resources/download-user-data` | `resources/download-user-data.tsx` | 未排除 |
| 16 | `/resources/images` | `resources/images.tsx` | 未排除 |
| 17 | `/resources/theme-switch` | `resources/theme-switch.tsx` | 未排除 |
| 18 | `/me` | `me.tsx` | 未排除 |

### 6.2 动态路由（含 `:param`，10 个）

| 序号 | URL 路径模式 | 路由文件 | 是否排除 |
|------|-------------|---------|---------|
| 1 | `/users/:username` | `users/$username/index.tsx` | 未排除 |
| 2 | `/users/:username/notes` | `users/$username/notes/index.tsx` | 未排除 |
| 3 | `/users/:username/notes/:noteId` | `users/$username/notes/$noteId.tsx` | 未排除 |
| 4 | `/users/:username/notes/:noteId/edit` | `users/$username/notes/$noteId_.edit.tsx` | 未排除 |
| 5 | `/users/:username/notes/new` | `users/$username/notes/new.tsx` | 未排除 |
| 6 | `/auth/:provider` | `_auth/auth.$provider/index.ts` | 未排除 |
| 7 | `/auth/:provider/callback` | `_auth/auth.$provider/callback.ts` | 未排除 |
| 8 | `/onboarding/:provider` | `_auth/onboarding/$provider.tsx` | 未排除 |
| 9 | `/admin/cache/lru/:cacheKey` | `admin/cache/lru.$cacheKey.ts` | 未排除 |
| 10 | `/admin/cache/sqlite/:cacheKey` | `admin/cache/sqlite.$cacheKey.ts` | 未排除 |

### 6.3 已排除路由统计

| 类型 | 数量 |
|------|------|
| 已排除的静态路由 | 6（forgot-password、login、reset-password、signup、verify、admin/cache） |
| 已排除的动态路由 | 0 |
| 已排除的布局路由 | 2（settings/profile/_layout、settings/profile/two-factor/_layout） |
| 已排除的设置页面 | 9（change-email、connections、index、password、password/create、photo、two-factor/*） |
| 总计已排除 | 17 |

### 6.4 本节统计验证

```
静态路由（未排除）: 18
动态路由（未排除）: 10
已排除路由: 17
布局路由（未排除）: 1
资源路由本身: 2

验证: 18 + 10 + 17 + 1 + 2 = 48 ✓
```

---

## 七、ALLOW_INDEXING 三层行为一致性分析

### 7.1 各层代码位置与逻辑

#### 层 1：HTML 文档层

**文件**: `app/root.tsx:148`

```typescript
const allowIndexing = ENV.ALLOW_INDEXING !== 'false'
// ...
{allowIndexing ? null : (
  <meta name="robots" content="noindex, nofollow" />
)}
```

**逻辑**：
- `ENV.ALLOW_INDEXING === 'false'` → 渲染 `noindex, nofollow`
- 其他情况 → 不渲染标签（默认允许）

**代码证据**: `app/utils/env.server.ts:63` 确认 `ALLOW_INDEXING` 通过 `getEnv()` 暴露为公共环境变量。

#### 层 2：Sitemap 资源路由

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

**检查结果**: 代码中**无** `process.env.ALLOW_INDEXING` 或 `ENV.ALLOW_INDEXING` 的引用。

**结论**: 无论 `ALLOW_INDEXING` 是什么值，`generateSitemap` 都会正常运行。

#### 层 3：Robots.txt 资源路由

**文件**: `app/routes/_seo/robots[.]txt.ts`

```typescript
export function loader({ request }: Route.LoaderArgs) {
	return generateRobotsTxt([
		{ type: 'sitemap', value: `${getDomainUrl(request)}/sitemap.xml` },
	])
}
```

**检查结果**: 代码中**无** `process.env.ALLOW_INDEXING` 的引用。

**默认策略**（来自 `@nasa-gcn/remix-seo` 文档）：
```
User-agent: *
Allow: /
Sitemap: https://your-domain.com/sitemap.xml
```

### 7.2 行为一致性对比表

| ALLOW_INDEXING 值 | HTML 层行为 | Sitemap 层行为 | Robots.txt 层行为 | 三层是否一致 |
|-------------------|------------|---------------|------------------|-------------|
| `undefined` (默认) | 不渲染 noindex 标签（允许） | 输出完整 sitemap（28 个路由） | 输出 `Allow: /` + sitemap 引用 | 一致 |
| `'true'` | 不渲染 noindex 标签（允许） | 输出完整 sitemap（28 个路由） | 输出 `Allow: /` + sitemap 引用 | 一致 |
| `'false'` | 渲染 `noindex, nofollow` 标签（禁止） | 输出完整 sitemap（28 个路由） | 输出 `Allow: /` + sitemap 引用 | 不一致 |

### 7.3 最终结论：ALLOW_INDEXING=false 时三层行为不一致

**当 `ALLOW_INDEXING="false"` 时**：

| 层面 | 实际行为 | 代码位置 | 是否受控制 |
|------|---------|---------|-----------|
| HTML 层 | 所有页面渲染 `<meta name="robots" content="noindex, nofollow" />` | `app/root.tsx:148` | 是 |
| Sitemap 层 | 输出完整 sitemap.xml，包含 28 个路由 | `app/routes/_seo/sitemap[.]xml.ts` | 否 |
| Robots.txt 层 | 输出 `User-agent: *` + `Allow: /` + `Sitemap: ...` | `app/routes/_seo/robots[.]txt.ts` | 否 |

**不一致的影响**：
- 搜索引擎访问 `/robots.txt` → 看到 `Allow: /`，允许抓取
- 搜索引擎访问 `/sitemap.xml` → 发现 28 个路由
- 搜索引擎访问具体页面 → 看到 `noindex` 标签

**结果**：
- 页面最终不会被索引（因为 HTML 有 noindex）
- 但搜索引擎会浪费资源抓取这些页面
- 且 sitemap 中的敏感路径会暴露

### 7.4 修正建议

**修改 `app/routes/_seo/sitemap[.]xml.ts`**：

```typescript
export async function loader({ request, context }: Route.LoaderArgs) {
	const allowIndexing = process.env.ALLOW_INDEXING !== 'false'
	
	if (!allowIndexing) {
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

**修改 `app/routes/_seo/robots[.]txt.ts`**：

```typescript
export function loader({ request }: Route.LoaderArgs) {
	const allowIndexing = process.env.ALLOW_INDEXING !== 'false'
	
	if (!allowIndexing) {
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

---

## 八、最终汇总

### 8.1 路由分类汇总

| 分类 | 数量 | 明细 |
|------|------|------|
| 总路由数 | 48 | 来自 `docs/routing.md:90-200` |
| 确定不会进入 sitemap | 17 | 5 认证 + 11 设置 + 1 管理后台 |
| 确定会进入 sitemap | 28 | 5 营销 + 6 用户 + 6 认证 + 2 WebAuthn + 3 管理后台 + 1 设置 + 4 资源路由 + 1 其他 |
| 布局路由（未排除） | 1 | `users/$username/notes/_layout.tsx` |
| 资源路由本身 | 2 | robots.txt + sitemap.xml 处理路由 |
| 应排除但未排除 | 14 | 4 认证 + 2 WebAuthn + 3 管理后台 + 1 设置 + 4 资源路由 |
| 静态路由（未排除） | 18 | 全为确定会进入 sitemap 的路由 |
| 动态路由（未排除） | 10 | 路由模式会出现，无具体实例 |

### 8.2 数字一致性验证

```
总路由数: 48

方式一：
  确定不会进入 sitemap: 17
+ 确定会进入 sitemap: 28
+ 布局路由（未排除）: 1
+ 资源路由本身: 2
= 48 ✓

方式二：
  静态路由（未排除）: 18
+ 动态路由（未排除）: 10
+ 已排除路由: 17
+ 布局路由（未排除）: 1
+ 资源路由本身: 2
= 48 ✓

方式三：
  已排除: 17
+ 应排除但未排除: 14
+ 应该保留的公共页面: 14（5 营销 + /users + 动态用户内容 8）
+ 布局路由（未排除）: 1
+ 资源路由本身: 2
= 48 ✓
```

### 8.3 ALLOW_INDEXING 最终结论

| 问题 | 结论 |
|------|------|
| 当 `ALLOW_INDEXING="false"` 时，三层行为是否一致？ | 不一致 |
| 哪层不受控制？ | Sitemap 层和 Robots.txt 层 |
| 影响范围？ | 所有环境，特别是开发/预览环境 |

---

## 九、优先级建议

| 优先级 | 问题 | 影响 |
|-------|------|------|
| 🔴 P0 | ALLOW_INDEXING 不控制 sitemap/robots | 所有环境 |
| 🔴 P0 | 14 个敏感路由未从 sitemap 排除 | 生产环境 |
