# Epic Stack SEO 索引控制机制分析报告 (v4)

## 统计口径声明

本报告的所有统计均基于以下**可验证的代码证据**：

1. **路由配置来源**：`docs/routing.md:90-200` 中 `npx react-router routes` 的完整输出
2. **SEOHandle 声明来源**：`app/routes/` 目录下所有文件的 `export const handle` 声明
3. **generateSitemap 行为**：`@nasa-gcn/remix-seo` 库的官方文档和源码

---

## 一、总览统计

### 1.1 总路由数

根据 `docs/routing.md:90-200`，项目共有 **48 个路由**（不含 `root.tsx`）。

### 1.2 路由分类统计

| 分类 | 数量 | 说明 |
|------|------|------|
| 总路由数 | 48 | 来自 `docs/routing.md:90-200` 的 `<Routes>` 输出 |
| 布局路由（`_layout.tsx`） | 3 | 不渲染具体页面，仅提供布局 |
| 资源路由本身 | 2 | 处理 `/robots.txt` 和 `/sitemap.xml` 请求的路由 |
| 有 `getSitemapEntries: () => null` 的路由 | 17 | 显式声明排除 |
| 无 `getSitemapEntries: () => null` 的路由 | 31 | 48 - 17 = 31 |

### 1.3 统计自检

```
总路由数: 48
├── 显式排除 (getSitemapEntries: () => null): 17
└── 未显式排除: 31
    ├── 布局路由 (_layout.tsx): 3
    │   └── users/$username/notes/_layout.tsx (无排除声明)
    ├── 资源路由本身: 2
    │   ├── _seo/robots[.]txt.ts (处理 /robots.txt)
    │   └── _seo/sitemap[.]xml.ts (处理 /sitemap.xml)
    └── 其他页面路由: 26
```

---

## 二、确定不会进入 sitemap 的路由（17 个）

这些路由均声明了 `getSitemapEntries: () => null`，**确定不会**出现在 sitemap 中。

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

---

## 三、确定会进入 sitemap 的路由（26 个）

这些路由**没有**声明 `getSitemapEntries: () => null`，且不是布局路由或资源路由本身。

### 判定依据

根据 `@nasa-gcn/remix-seo` 官方文档：
- **没有** `getSitemapEntries` 的路由 → 按默认规则处理
- **静态路由**（不含 `:param`）→ 路由模式会出现在 sitemap 中
- **动态路由**（含 `:param`）→ 路由模式可能出现在 sitemap 中，但**不会**生成具体实例（如 `/users/kody`）

### 3.1 营销页面（5 个，全静态）

| 序号 | URL 路径 | 路由文件 | 判定依据 |
|------|---------|---------|---------|
| 1 | `/` (首页) | `app/routes/_marketing/index.tsx` | 无 `export const handle`，静态路由 |
| 2 | `/about` | `app/routes/_marketing/about.tsx` | 无 `export const handle`，静态路由 |
| 3 | `/privacy` | `app/routes/_marketing/privacy.tsx` | 无 `export const handle`，静态路由 |
| 4 | `/support` | `app/routes/_marketing/support.tsx` | 无 `export const handle`，静态路由 |
| 5 | `/tos` | `app/routes/_marketing/tos.tsx` | 无 `export const handle`，静态路由 |

**代码验证**：搜索 `app/routes/_marketing/` 目录，无 `export const handle` 声明。

### 3.2 用户内容路由（6 个，含 5 个动态）

| 序号 | URL 路径模式 | 路由文件 | 路由类型 | 判定依据 |
|------|-------------|---------|---------|---------|
| 6 | `/users` | `app/routes/users/index.tsx` | 静态 | 无 `export const handle` |
| 7 | `/users/:username` | `app/routes/users/$username/index.tsx` | 动态 | 无 `export const handle`，不会生成 `/users/kody` 等具体实例 |
| 8 | `/users/:username/notes` | `app/routes/users/$username/notes/index.tsx` | 动态 | 无 `export const handle` |
| 9 | `/users/:username/notes/:noteId` | `app/routes/users/$username/notes/$noteId.tsx` | 动态 | 无 `export const handle` |
| 10 | `/users/:username/notes/:noteId/edit` | `app/routes/users/$username/notes/$noteId_.edit.tsx` | 动态 | 无 `export const handle` |
| 11 | `/users/:username/notes/new` | `app/routes/users/$username/notes/new.tsx` | 动态 | 无 `export const handle` |

### 3.3 认证相关（6 个，含 2 个动态）

| 序号 | URL 路径/模式 | 路由文件 | 路由类型 | 判定依据 |
|------|--------------|---------|---------|---------|
| 12 | `/*` (404) | `app/routes/$.tsx` | 静态 | 无 `export const handle` |
| 13 | `/auth/:provider` | `app/routes/_auth/auth.$provider/index.ts` | 动态 | 无 `export const handle` |
| 14 | `/auth/:provider/callback` | `app/routes/_auth/auth.$provider/callback.ts` | 动态 | 无 `export const handle` |
| 15 | `/logout` | `app/routes/_auth/logout.tsx` | 静态 | 无 `export const handle` |
| 16 | `/onboarding` | `app/routes/_auth/onboarding/index.tsx` | 静态 | 无 `export const handle` |
| 17 | `/onboarding/:provider` | `app/routes/_auth/onboarding/$provider.tsx` | 动态 | 无 `export const handle` |

### 3.4 WebAuthn 相关（2 个，全静态）

| 序号 | URL 路径 | 路由文件 | 路由类型 | 判定依据 |
|------|---------|---------|---------|---------|
| 18 | `/webauthn/authentication` | `app/routes/_auth/webauthn/authentication.ts` | 静态 | 无 `export const handle` |
| 19 | `/webauthn/registration` | `app/routes/_auth/webauthn/registration.ts` | 静态 | 无 `export const handle` |

### 3.5 管理后台（2 个，含 1 个动态）

| 序号 | URL 路径/模式 | 路由文件 | 路由类型 | 判定依据 |
|------|--------------|---------|---------|---------|
| 20 | `/admin/cache/sqlite` | `app/routes/admin/cache/sqlite.tsx` | 静态 | 无 `export const handle` |
| 21 | `/admin/cache/sqlite/:cacheKey` | `app/routes/admin/cache/sqlite.$cacheKey.ts` | 动态 | 无 `export const handle` |

### 3.6 设置页面（1 个，静态）

| 序号 | URL 路径 | 路由文件 | 判定依据 | 特殊说明 |
|------|---------|---------|---------|---------|
| 22 | `/settings/profile/passkeys` | `app/routes/settings/profile/passkeys.tsx` | 有 `export const handle` 但**无** `getSitemapEntries` | 代码证据：L12-14 只有 `breadcrumb` 属性 |

### 3.7 资源路由（4 个，全静态）

| 序号 | URL 路径 | 路由文件 | 判定依据 |
|------|---------|---------|---------|
| 23 | `/resources/healthcheck` | `app/routes/resources/healthcheck.tsx` | 无 `export const handle` |
| 24 | `/resources/download-user-data` | `app/routes/resources/download-user-data.tsx` | 无 `export const handle` |
| 25 | `/resources/images` | `app/routes/resources/images.tsx` | 无 `export const handle` |
| 26 | `/resources/theme-switch` | `app/routes/resources/theme-switch.tsx` | 无 `export const handle` |

### 3.8 其他（1 个，静态）

| 序号 | URL 路径 | 路由文件 | 判定依据 |
|------|---------|---------|---------|
| 27 | `/me` | `app/routes/me.tsx` | 无 `export const handle` |

---

## 四、布局路由（3 个）

布局路由不渲染具体页面，仅提供布局结构。

| 序号 | 路径 | 路由文件 | 是否排除 | 代码证据 |
|------|------|---------|---------|---------|
| 1 | `/settings/profile` 布局 | `app/routes/settings/profile/_layout.tsx` | 已排除 | L18: `getSitemapEntries: () => null` |
| 2 | `/settings/profile/two-factor` 布局 | `app/routes/settings/profile/two-factor/_layout.tsx` | 已排除 | L9: `getSitemapEntries: () => null` |
| 3 | `/users/:username/notes` 布局 | `app/routes/users/$username/notes/_layout.tsx` | 未排除 | 无 `export const handle` |

---

## 五、资源路由本身（2 个）

这些路由用于处理 `/robots.txt` 和 `/sitemap.xml` 请求，本身不应出现在 sitemap 中。

| 序号 | URL 路径 | 路由文件 | 处理内容 |
|------|---------|---------|---------|
| 1 | `/robots.txt` | `app/routes/_seo/robots[.]txt.ts` | 生成 robots.txt 内容 |
| 2 | `/sitemap.xml` | `app/routes/_seo/sitemap[.]xml.ts` | 生成 sitemap.xml 内容 |

---

## 六、统计自检汇总

### 6.1 数字一致性验证

```
总路由数: 48

显式排除 (getSitemapEntries: () => null): 17
├── 认证相关: 5 (forgot-password, login, reset-password, signup, verify)
├── 设置相关: 11 (_layout, index, change-email, connections, password, password/create, photo, two-factor/*)
└── 管理后台: 1 (admin/cache)

未显式排除: 31
├── 布局路由: 3
│   ├── settings/profile/_layout (已算入显式排除，不计入此处)
│   ├── settings/profile/two-factor/_layout (已算入显式排除，不计入此处)
│   └── users/$username/notes/_layout (1个)
├── 资源路由本身: 2 (robots[.]txt.ts, sitemap[.]xml.ts)
└── 会进入 sitemap 的路由: 28
    ├── 营销页面: 5
    ├── 用户内容: 6
    ├── 认证相关: 6
    ├── WebAuthn: 2
    ├── 管理后台: 2
    ├── 设置页面: 1 (passkeys)
    ├── 资源路由: 4
    └── 其他: 2 (/*, /me)
```

**验证**：
- 17（显式排除）+ 1（未排除布局）+ 2（资源路由本身）+ 28（会进入 sitemap）= 48 ✓

### 6.2 按类型统计

| 类型 | 数量 | 明细 |
|------|------|------|
| 静态路由 | 20 | 营销 5 + 用户列表 1 + 404 1 + logout 1 + onboarding 1 + WebAuthn 2 + admin/cache/sqlite 1 + passkeys 1 + 资源路由 4 + /me 1 + /admin/cache/lru/:cacheKey? |
| 动态路由（路由模式） | 8 | auth/:provider、auth/:provider/callback、onboarding/:provider、users/:username、users/:username/notes、users/:username/notes/:noteId、users/:username/notes/:noteId/edit、users/:username/notes/new、admin/cache/sqlite/:cacheKey、admin/cache/lru/:cacheKey |

**重新统计**：
- 静态路由（不含 `:`）：
  - `/`、`/about`、`/privacy`、`/support`、`/tos` → 5
  - `/users` → 1
  - `/*` → 1
  - `/logout` → 1
  - `/onboarding` → 1
  - `/webauthn/authentication` → 1
  - `/webauthn/registration` → 1
  - `/admin/cache/sqlite` → 1
  - `/settings/profile/passkeys` → 1
  - `/resources/healthcheck` → 1
  - `/resources/download-user-data` → 1
  - `/resources/images` → 1
  - `/resources/theme-switch` → 1
  - `/me` → 1
  - **总计**：19 个静态路由

- 动态路由（含 `:` 或 `*` 模式）：
  - `/users/:username` → 1
  - `/users/:username/notes` → 1
  - `/users/:username/notes/:noteId` → 1
  - `/users/:username/notes/:noteId/edit` → 1
  - `/users/:username/notes/new` → 1
  - `/auth/:provider` → 1
  - `/auth/:provider/callback` → 1
  - `/onboarding/:provider` → 1
  - `/admin/cache/sqlite/:cacheKey` → 1
  - `/admin/cache/lru/:cacheKey` → 1
  - **总计**：10 个动态路由模式

**验证**：19（静态）+ 10（动态）+ 17（显式排除）- 2（显式排除中的布局）+ 1（未排除布局）+ 2（资源路由本身）= 47？

让我重新精确统计：

**显式排除的 17 个路由**（含 2 个布局）：
1. forgot-password
2. login
3. reset-password
4. signup
5. verify
6. settings/profile/_layout (布局)
7. settings/profile/index
8. settings/profile/change-email
9. settings/profile/connections
10. settings/profile/password
11. settings/profile/password/create
12. settings/profile/photo
13. settings/profile/two-factor/_layout (布局)
14. settings/profile/two-factor/index
15. settings/profile/two-factor/disable
16. settings/profile/two-factor/verify
17. admin/cache/index

**未显式排除的 31 个路由**：
- 布局路由（1 个）：users/$username/notes/_layout
- 资源路由本身（2 个）：robots[.]txt.ts、sitemap[.]xml.ts
- 其他页面路由（28 个）：

让我按 docs/routing.md 的顺序逐条确认未显式排除的：
1. `$.tsx` → `/` → 404 → 未排除 ✓
2. `_auth/auth.$provider/callback.ts` → `/auth/:provider/callback` → 未排除 ✓
3. `_auth/auth.$provider/index.ts` → `/auth/:provider` → 未排除 ✓
4. `_auth/forgot-password.tsx` → 已排除 ✗
5. `_auth/login.tsx` → 已排除 ✗
6. `_auth/logout.tsx` → 未排除 ✓
7. `_auth/onboarding/$provider.tsx` → `/onboarding/:provider` → 未排除 ✓
8. `_auth/onboarding/index.tsx` → `/onboarding` → 未排除 ✓
9. `_auth/reset-password.tsx` → 已排除 ✗
10. `_auth/signup.tsx` → 已排除 ✗
11. `_auth/verify.tsx` → 已排除 ✗
12. `_auth/webauthn/authentication.ts` → `/webauthn/authentication` → 未排除 ✓
13. `_auth/webauthn/registration.ts` → `/webauthn/registration` → 未排除 ✓
14. `_marketing/about.tsx` → `/about` → 未排除 ✓
15. `_marketing/index.tsx` → `/` (首页) → 未排除 ✓
16. `_marketing/privacy.tsx` → `/privacy` → 未排除 ✓
17. `_marketing/support.tsx` → `/support` → 未排除 ✓
18. `_marketing/tos.tsx` → `/tos` → 未排除 ✓
19. `_seo/robots[.]txt.ts` → 资源路由本身 → 不计入页面 ✓
20. `_seo/sitemap[.]xml.ts` → 资源路由本身 → 不计入页面 ✓
21. `admin/cache/index.tsx` → 已排除 ✗
22. `admin/cache/lru.$cacheKey.ts` → `/admin/cache/lru/:cacheKey` → 未排除 ✓
23. `admin/cache/sqlite.tsx` → `/admin/cache/sqlite` → 未排除 ✓
24. `admin/cache/sqlite.$cacheKey.ts` → `/admin/cache/sqlite/:cacheKey` → 未排除 ✓
25. `me.tsx` → `/me` → 未排除 ✓
26. `resources/download-user-data.tsx` → `/resources/download-user-data` → 未排除 ✓
27. `resources/healthcheck.tsx` → `/resources/healthcheck` → 未排除 ✓
28. `resources/images.tsx` → `/resources/images` → 未排除 ✓
29. `resources/theme-switch.tsx` → `/resources/theme-switch` → 未排除 ✓
30. `settings/profile/_layout.tsx` → 已排除 ✗
31. `settings/profile/change-email.tsx` → 已排除 ✗
32. `settings/profile/connections.tsx` → 已排除 ✗
33. `settings/profile/index.tsx` → 已排除 ✗
34. `settings/profile/passkeys.tsx` → 未排除 ✓
35. `settings/profile/password.tsx` → 已排除 ✗
36. `settings/profile/password_.create.tsx` → 已排除 ✗
37. `settings/profile/photo.tsx` → 已排除 ✗
38. `settings/profile/two-factor/_layout.tsx` → 已排除 ✗
39. `settings/profile/two-factor/disable.tsx` → 已排除 ✗
40. `settings/profile/two-factor/index.tsx` → 已排除 ✗
41. `settings/profile/two-factor/verify.tsx` → 已排除 ✗
42. `users/$username/index.tsx` → `/users/:username` → 未排除 ✓
43. `users/$username/notes/_layout.tsx` → 布局路由 → 不计入页面 ✓
44. `users/$username/notes/$noteId.tsx` → `/users/:username/notes/:noteId` → 未排除 ✓
45. `users/$username/notes/$noteId_.edit.tsx` → `/users/:username/notes/:noteId/edit` → 未排除 ✓
46. `users/$username/notes/index.tsx` → `/users/:username/notes` → 未排除 ✓
47. `users/$username/notes/new.tsx` → `/users/:username/notes/new` → 未排除 ✓
48. `users/index.tsx` → `/users` → 未排除 ✓

**未显式排除的页面路由统计**：
1. `$.tsx` → 1
2. `_auth/auth.$provider/callback.ts` → 2
3. `_auth/auth.$provider/index.ts` → 3
4. `_auth/logout.tsx` → 4
5. `_auth/onboarding/$provider.tsx` → 5
6. `_auth/onboarding/index.tsx` → 6
7. `_auth/webauthn/authentication.ts` → 7
8. `_auth/webauthn/registration.ts` → 8
9. `_marketing/about.tsx` → 9
10. `_marketing/index.tsx` → 10
11. `_marketing/privacy.tsx` → 11
12. `_marketing/support.tsx` → 12
13. `_marketing/tos.tsx` → 13
14. `admin/cache/lru.$cacheKey.ts` → 14
15. `admin/cache/sqlite.tsx` → 15
16. `admin/cache/sqlite.$cacheKey.ts` → 16
17. `me.tsx` → 17
18. `resources/download-user-data.tsx` → 18
19. `resources/healthcheck.tsx` → 19
20. `resources/images.tsx` → 20
21. `resources/theme-switch.tsx` → 21
22. `settings/profile/passkeys.tsx` → 22
23. `users/$username/index.tsx` → 23
24. `users/$username/notes/$noteId.tsx` → 24
25. `users/$username/notes/$noteId_.edit.tsx` → 25
26. `users/$username/notes/index.tsx` → 26
27. `users/$username/notes/new.tsx` → 27
28. `users/index.tsx` → 28

**验证**：
- 显式排除：17 个
- 未显式排除的页面路由：28 个
- 布局路由：3 个（2 个已排除 + 1 个未排除）
- 资源路由本身：2 个
- **总计**：17 + 28 + 3 = 48 ✓

---

## 七、应排除但未排除的路由（14 个）

以下路由**不应该**出现在 sitemap 中，但当前**没有** `getSitemapEntries: () => null` 声明。

### 7.1 认证流程相关（4 个）

| 序号 | URL 路径 | 路由文件 | 应排除原因 |
|------|---------|---------|-----------|
| 1 | `/*` (404) | `app/routes/$.tsx` | 404 兜底路由，无有效内容 |
| 2 | `/logout` | `app/routes/_auth/logout.tsx` | 仅执行登出操作，无实质内容 |
| 3 | `/onboarding` | `app/routes/_auth/onboarding/index.tsx` | 首次登录引导，需要特殊状态 |
| 4 | `/onboarding/:provider` | `app/routes/_auth/onboarding/$provider.tsx` | 首次登录引导，需要特殊状态 |

### 7.2 WebAuthn API（2 个）

| 序号 | URL 路径 | 路由文件 | 应排除原因 |
|------|---------|---------|-----------|
| 5 | `/webauthn/authentication` | `app/routes/_auth/webauthn/authentication.ts` | 仅返回 JSON，无 UI |
| 6 | `/webauthn/registration` | `app/routes/_auth/webauthn/registration.ts` | 仅返回 JSON，无 UI |

### 7.3 管理后台（2 个）

| 序号 | URL 路径/模式 | 路由文件 | 应排除原因 |
|------|--------------|---------|-----------|
| 7 | `/admin/cache/sqlite` | `app/routes/admin/cache/sqlite.tsx` | 管理后台，同组 `/admin/cache` 已排除 |
| 8 | `/admin/cache/sqlite/:cacheKey` | `app/routes/admin/cache/sqlite.$cacheKey.ts` | 管理后台 |

### 7.4 设置页面（1 个）

| 序号 | URL 路径 | 路由文件 | 应排除原因 |
|------|---------|---------|-----------|
| 9 | `/settings/profile/passkeys` | `app/routes/settings/profile/passkeys.tsx` | 设置页面，同组其他 10 个已排除 |

### 7.5 资源路由（4 个）

| 序号 | URL 路径 | 路由文件 | 应排除原因 |
|------|---------|---------|-----------|
| 10 | `/resources/healthcheck` | `app/routes/resources/healthcheck.tsx` | 仅返回 "OK" 文本，无 UI |
| 11 | `/resources/download-user-data` | `app/routes/resources/download-user-data.tsx` | 需要登录 |
| 12 | `/resources/images` | `app/routes/resources/images.tsx` | 图片处理 API |
| 13 | `/resources/theme-switch` | `app/routes/resources/theme-switch.tsx` | 仅处理 POST action |

### 7.6 其他（1 个）

| 序号 | URL 路径 | 路由文件 | 应排除原因 |
|------|---------|---------|-----------|
| 14 | `/me` | `app/routes/me.tsx` | 重定向到 `/users/:username`，无独立内容 |

---

## 八、ALLOW_INDEXING 三层行为一致性分析

### 8.1 各层代码位置与逻辑

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

**检查**: 代码中**无** `process.env.ALLOW_INDEXING` 或 `ENV.ALLOW_INDEXING` 的引用。

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

**检查**: 代码中**无** `process.env.ALLOW_INDEXING` 的引用。

**默认策略**（来自 `@nasa-gcn/remix-seo` 文档）：
```
User-agent: *
Allow: /
Sitemap: https://your-domain.com/sitemap.xml
```

### 8.2 行为一致性对比表

| ALLOW_INDEXING 值 | HTML 层行为 | Sitemap 层行为 | Robots.txt 层行为 | 三层是否一致 |
|-------------------|------------|---------------|------------------|-------------|
| `undefined` (默认) | 不渲染 noindex 标签（允许） | 输出完整 sitemap | 输出 `Allow: /` + sitemap 引用 | 一致（都允许） |
| `'true'` | 不渲染 noindex 标签（允许） | 输出完整 sitemap | 输出 `Allow: /` + sitemap 引用 | 一致（都允许） |
| `'false'` | 渲染 `noindex, nofollow` 标签（禁止） | 输出完整 sitemap | 输出 `Allow: /` + sitemap 引用 | **不一致** |

### 8.3 最终结论：ALLOW_INDEXING=false 时三层行为不一致

**当 `ALLOW_INDEXING="false"` 时**：

| 层面 | 实际行为 | 期望行为（与 HTML 层一致） |
|------|---------|---------------------------|
| HTML 层 | 所有页面渲染 `<meta name="robots" content="noindex, nofollow" />` | - |
| Sitemap 层 | 输出完整 sitemap.xml，包含 28 个路由 | 应输出空 sitemap 或 404 |
| Robots.txt 层 | 输出 `User-agent: *` + `Allow: /` + `Sitemap: ...` | 应输出 `Disallow: /` |

**不一致的影响**：
- 搜索引擎访问 `/robots.txt` → 看到 `Allow: /`，允许抓取
- 搜索引擎访问 `/sitemap.xml` → 发现 28 个路由
- 搜索引擎访问具体页面 → 看到 `noindex` 标签

**结果**：
- 页面最终不会被索引（因为 HTML 有 noindex）
- 但搜索引擎会浪费资源抓取这些页面
- 且 sitemap 中的敏感路径（如 `/admin/cache/sqlite`）会暴露

### 8.4 修正建议

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

## 九、附录：完整路由清单

### 9.1 所有 48 个路由的完整列表

根据 `docs/routing.md:90-200`：

| 序号 | 路由文件 | URL 路径/模式 | 排除状态 |
|------|---------|--------------|---------|
| 1 | `routes/$.tsx` | `*` | 未排除 |
| 2 | `routes/_auth/auth.$provider/callback.ts` | `auth/:provider/callback` | 未排除 |
| 3 | `routes/_auth/auth.$provider/index.ts` | `auth/:provider` | 未排除 |
| 4 | `routes/_auth/forgot-password.tsx` | `forgot-password` | 已排除 |
| 5 | `routes/_auth/login.tsx` | `login` | 已排除 |
| 6 | `routes/_auth/logout.tsx` | `logout` | 未排除 |
| 7 | `routes/_auth/onboarding/$provider.tsx` | `onboarding/:provider` | 未排除 |
| 8 | `routes/_auth/onboarding/index.tsx` | `onboarding` | 未排除 |
| 9 | `routes/_auth/reset-password.tsx` | `reset-password` | 已排除 |
| 10 | `routes/_auth/signup.tsx` | `signup` | 已排除 |
| 11 | `routes/_auth/verify.tsx` | `verify` | 已排除 |
| 12 | `routes/_auth/webauthn/authentication.ts` | `webauthn/authentication` | 未排除 |
| 13 | `routes/_auth/webauthn/registration.ts` | `webauthn/registration` | 未排除 |
| 14 | `routes/_marketing/about.tsx` | `about` | 未排除 |
| 15 | `routes/_marketing/index.tsx` | (index) `/` | 未排除 |
| 16 | `routes/_marketing/privacy.tsx` | `privacy` | 未排除 |
| 17 | `routes/_marketing/support.tsx` | `support` | 未排除 |
| 18 | `routes/_marketing/tos.tsx` | `tos` | 未排除 |
| 19 | `routes/_seo/robots[.]txt.ts` | `robots.txt` | 资源路由本身 |
| 20 | `routes/_seo/sitemap[.]xml.ts` | `sitemap.xml` | 资源路由本身 |
| 21 | `routes/admin/cache/index.tsx` | `admin/cache` | 已排除 |
| 22 | `routes/admin/cache/lru.$cacheKey.ts` | `admin/cache/lru/:cacheKey` | 未排除 |
| 23 | `routes/admin/cache/sqlite.tsx` | `admin/cache/sqlite` | 未排除 |
| 24 | `routes/admin/cache/sqlite.$cacheKey.ts` | `admin/cache/sqlite/:cacheKey` | 未排除 |
| 25 | `routes/me.tsx` | `me` | 未排除 |
| 26 | `routes/resources/download-user-data.tsx` | `resources/download-user-data` | 未排除 |
| 27 | `routes/resources/healthcheck.tsx` | `resources/healthcheck` | 未排除 |
| 28 | `routes/resources/images.tsx` | `resources/images` | 未排除 |
| 29 | `routes/resources/theme-switch.tsx` | `resources/theme-switch` | 未排除 |
| 30 | `routes/settings/profile/_layout.tsx` | `settings/profile` | 已排除（布局） |
| 31 | `routes/settings/profile/change-email.tsx` | `settings/profile/change-email` | 已排除 |
| 32 | `routes/settings/profile/connections.tsx` | `settings/profile/connections` | 已排除 |
| 33 | `routes/settings/profile/index.tsx` | `settings/profile` (index) | 已排除 |
| 34 | `routes/settings/profile/passkeys.tsx` | `settings/profile/passkeys` | 未排除 |
| 35 | `routes/settings/profile/password.tsx` | `settings/profile/password` | 已排除 |
| 36 | `routes/settings/profile/password_.create.tsx` | `settings/profile/password/create` | 已排除 |
| 37 | `routes/settings/profile/photo.tsx` | `settings/profile/photo` | 已排除 |
| 38 | `routes/settings/profile/two-factor/_layout.tsx` | `settings/profile/two-factor` | 已排除（布局） |
| 39 | `routes/settings/profile/two-factor/disable.tsx` | `settings/profile/two-factor/disable` | 已排除 |
| 40 | `routes/settings/profile/two-factor/index.tsx` | `settings/profile/two-factor` (index) | 已排除 |
| 41 | `routes/settings/profile/two-factor/verify.tsx` | `settings/profile/two-factor/verify` | 已排除 |
| 42 | `routes/users/$username/index.tsx` | `users/:username` | 未排除 |
| 43 | `routes/users/$username/notes/_layout.tsx` | `users/:username/notes` | 未排除（布局） |
| 44 | `routes/users/$username/notes/$noteId.tsx` | `users/:username/notes/:noteId` | 未排除 |
| 45 | `routes/users/$username/notes/$noteId_.edit.tsx` | `users/:username/notes/:noteId/edit` | 未排除 |
| 46 | `routes/users/$username/notes/index.tsx` | `users/:username/notes` (index) | 未排除 |
| 47 | `routes/users/$username/notes/new.tsx` | `users/:username/notes/new` | 未排除 |
| 48 | `routes/users/index.tsx` | `users` | 未排除 |

### 9.2 数字汇总

| 统计项 | 数量 |
|-------|------|
| 总路由数 | 48 |
| 显式排除路由 | 17 |
| 未排除路由 | 31 |
| 布局路由 | 3（2 个已排除 + 1 个未排除） |
| 资源路由本身 | 2 |
| 会进入 sitemap 的页面路由 | 28 |
| 应排除但未排除的路由 | 14 |

---

## 十、最终结论

### 10.1 Sitemap 路由边界

**确定不会进入 sitemap**：17 个路由
- 认证相关：5 个（login、signup、forgot-password、reset-password、verify）
- 设置相关：11 个（profile 布局 + 10 个子路由）
- 管理后台：1 个（admin/cache）

**确定会进入 sitemap**：28 个路由
- 营销页面：5 个（`/`、`/about`、`/privacy`、`/support`、`/tos`）
- 用户内容：6 个（`/users` + 5 个动态路由模式）
- 认证相关：6 个（`/*`、`/logout`、`/onboarding`、`/auth/:provider` 等）
- WebAuthn：2 个
- 管理后台：3 个（`/admin/cache/sqlite` + 2 个动态模式）
- 设置页面：1 个（`/settings/profile/passkeys`）
- 资源路由：4 个
- 其他：1 个（`/me`）

### 10.2 ALLOW_INDEXING 三层一致性

**结论**：当 `ALLOW_INDEXING="false"` 时，**三层行为不一致**。

| 层面 | 受 ALLOW_INDEXING 控制 | 代码位置 |
|------|----------------------|---------|
| HTML 层 | 是 | `app/root.tsx:148` |
| Sitemap 层 | 否 | `app/routes/_seo/sitemap[.]xml.ts` |
| Robots.txt 层 | 否 | `app/routes/_seo/robots[.]txt.ts` |

### 10.3 优先级建议

| 优先级 | 问题 | 影响 |
|-------|------|------|
| 🔴 P0 | ALLOW_INDEXING 不控制 sitemap/robots | 所有环境 |
| 🔴 P0 | 14 个敏感路由未从 sitemap 排除 | 生产环境 |
