# Epic Stack 路由约定分析报告

## 目录

1. [概述](#概述)
2. [路由自动生成机制](#路由自动生成机制)
3. [文件命名与路径映射约定](#文件命名与路径映射约定)
4. [路由模块共置约定](#路由模块共置约定)
5. [特殊文件约定](#特殊文件约定)
6. [配置与忽略规则](#配置与忽略规则)
7. [实际项目示例分析](#实际项目示例分析)

---

## 概述

Epic Stack 使用 **基于文件系统的路由**（File-based Routing），通过 `react-router-auto-routes` 库自动从 `app/routes/` 目录生成路由配置。这种方式兼具：
- 代码与路由的共置能力
- 清晰的文件夹组织结构

核心配置文件：
- `react-router.config.ts` - React Router 全局配置
- `app/routes.ts` - 路由自动生成配置

---

## 路由自动生成机制

### 1. 技术栈演进

Epic Stack 经历了路由工具的演进：
- **旧方案**：`remix-flat-routes` - 使用 `+` 后缀的混合约定
- **新方案**：`react-router-auto-routes` - 更贴近 React Router 原生约定

决策文档：`docs/decisions/045-rr-auto-routes.md`

### 2. 配置结构

**`react-router.config.ts`（全局配置）：**

```typescript
export default {
  ssr: true,
  routeDiscovery: { mode: 'initial' },  // 路由发现模式
  // ...
}
```

**`app/routes.ts`（路由生成配置）：**

```typescript
import { autoRoutes } from 'react-router-auto-routes'

export default autoRoutes({
  ignoredRouteFiles: [
    '.*',
    '**/*.css',
    '**/*.test.{js,jsx,ts,tsx}',
    '**/__*.*',
    '**/*.server.*',
    '**/*.client.*',
  ],
})
```

### 3. 路由发现流程

1. 扫描 `app/routes/` 目录
2. 根据文件命名约定解析路径
3. 应用忽略规则过滤非路由文件
4. 生成路由树配置
5. 与 `root.tsx` 组合形成完整路由

**调试工具：**
```bash
npx react-router routes
```
此命令可输出 JSX 风格的路由树，便于验证路由映射。

---

## 文件命名与路径映射约定

### 1. 基础路由

| 文件名 | URL 路径 | 说明 |
|--------|----------|------|
| `about.tsx` | `/about` | 普通路由 |
| `index.tsx` | `/` | 索引路由（目录级） |
| `me.tsx` | `/me` | 根级路由 |

**目录结构示例：**
```
app/routes/
├── about.tsx          → /about
├── me.tsx             → /me
└── users/
    └── index.tsx      → /users
```

### 2. 动态路由（`$` 前缀）

使用 `$` 前缀命名文件或目录表示动态参数。

| 命名方式 | URL 路径 | 参数名 |
|----------|----------|--------|
| `$username.tsx` | `/users/:username` | `params.username` |
| `$noteId.tsx` | `/notes/:noteId` | `params.noteId` |

**示例：**
```
app/routes/users/
└── $username/
    ├── index.tsx      → /users/:username
    └── notes/
        └── $noteId.tsx → /users/:username/notes/:noteId
```

代码中访问参数：
```typescript
export async function loader({ params }: Route.LoaderArgs) {
  const username = params.username  // 类型安全
  const noteId = params.noteId
}
```

### 3. 路径组（`_` 前缀）

使用 `_` 前缀的目录表示**路径组**，该前缀**不会出现在 URL 中**，仅用于组织路由和共享布局。

| 目录名 | 用途 | URL 影响 |
|--------|------|----------|
| `_auth/` | 认证相关路由 | 不影响 URL |
| `_marketing/` | 营销页面路由 | 不影响 URL |
| `_seo/` | SEO 资源路由 | 不影响 URL |

**示例：**
```
app/routes/_auth/
├── login.tsx          → /login （而非 /_auth/login）
├── signup.tsx         → /signup
└── forgot-password.tsx → /forgot-password
```

### 4. 嵌套路由与布局

#### 4.1 目录结构嵌套

子目录自动创建嵌套路由层次：

```
app/routes/settings/
└── profile/
    ├── _layout.tsx    → /settings/profile（布局组件）
    ├── index.tsx      → /settings/profile（索引）
    ├── password.tsx   → /settings/profile/password
    └── two-factor/
        ├── _layout.tsx → /settings/profile/two-factor（子布局）
        └── index.tsx  → /settings/profile/two-factor
```

#### 4.2 `_layout.tsx` 布局文件

`_layout.tsx` 是当前目录级别的布局组件，通过 `<Outlet />` 渲染子路由：

```typescript
// app/routes/users/$username/notes/_layout.tsx
import { Outlet } from 'react-router'

export default function NotesLayout({ loaderData }) {
  return (
    <div className="layout">
      <nav>{/* 侧边栏导航 */}</nav>
      <main>
        <Outlet />  {/* 子路由内容渲染位置 */}
      </main>
    </div>
  )
}
```

#### 4.3 无布局嵌套（普通目录）

如果目录中没有 `_layout.tsx`，子路由仍会嵌套但没有中间布局层：

```
app/routes/admin/
└── cache/
    ├── index.tsx           → /admin/cache（索引）
    └── lru.$cacheKey.tsx   → /admin/cache/lru/:cacheKey
```

### 5. 转义字符（`[.]`）

使用 `[.]` 转义文件名中的点号，用于创建包含扩展名的路由：

| 文件名 | URL 路径 |
|--------|----------|
| `robots[.]txt.ts` | `/robots.txt` |
| `sitemap[.]xml.ts` | `/sitemap.xml` |

**示例：**
```typescript
// app/routes/_seo/robots[.]txt.ts
export async function loader() {
  return new Response('User-agent: *\nAllow: /', {
    headers: { 'Content-Type': 'text/plain' },
  })
}
```

### 6. 路径后缀（`_` 后缀）

使用 `_` 后缀可以创建 URL 中的额外路径段：

| 文件名 | URL 路径 |
|--------|----------|
| `password_.create.tsx` | `/password/create` |
| `$noteId_.edit.tsx` | `/:noteId/edit` |

**示例对比：**
```
app/routes/settings/profile/
├── password.tsx         → /settings/profile/password
└── password_.create.tsx → /settings/profile/password/create
```

### 7. 通配符路由（`$`）

根级的 `$.tsx` 作为通配符/404 路由：

```
app/routes/
└── $.tsx                → /*（兜底路由）
```

---

## 路由模块共置约定

Epic Stack 的核心设计理念之一是 **"路由与相关代码共置"**。以下是共置约定的详细说明：

### 1. `+` 前缀目录（共置资源目录）

以 `+` 开头的目录**不会被识别为路由**，用于存放与当前路由相关的共享代码、资源和类型。

| 目录名 | 用途 | 示例内容 |
|--------|------|----------|
| `+shared/` | 共享组件和逻辑 | 表单组件、工具函数 |
| `+logos/` | 静态资源集合 | SVG 图标、图片 |
| `+types/` | 类型定义 | React Router 自动生成的类型 |

#### 1.1 `+shared/` - 共享组件

存放多个路由共享的组件：

```
app/routes/users/$username/notes/
├── +shared/
│   ├── note-editor.tsx          # 笔记编辑器组件
│   └── note-editor.server.tsx   # 服务端 action 逻辑
├── $noteId.tsx                  # 查看笔记（使用 +shared 组件）
├── $noteId_.edit.tsx            # 编辑笔记（使用 +shared 组件）
└── new.tsx                      # 新建笔记（使用 +shared 组件）
```

**使用方式：**
```typescript
// app/routes/users/$username/notes/new.tsx
import { NoteEditor } from './+shared/note-editor.tsx'
import { action } from './+shared/note-editor.server.tsx'

export { action }
export default function NewNoteRoute() {
  return <NoteEditor />
}
```

#### 1.2 `+logos/` - 资源集合

存放与页面相关的静态资源：

```
app/routes/_marketing/
├── +logos/
│   ├── logos.ts        # 图标数据配置
│   ├── remix.svg
│   ├── tailwind.svg
│   └── stars.jpg
└── index.tsx           # 首页引用 +logos/logos.ts
```

**使用方式：**
```typescript
// app/routes/_marketing/index.tsx
import { logos, stars } from './+logos/logos.ts'
```

#### 1.3 `+types/` - 类型定义目录

React Router 自动生成的类型目录，提供类型安全的路由参数和加载器数据：

```typescript
// 自动生成类型，可在路由文件中引用
import { type Route } from './+types/$noteId_.edit.ts'

export async function loader({ params }: Route.LoaderArgs) {
  // params.noteId 具有正确的类型
}

export default function Component({ loaderData }: Route.ComponentProps) {
  // loaderData 具有正确的类型
}
```

**生成命令：**
```bash
npm run typecheck
# 或
npx react-router typegen
```

### 2. `.server.` 后缀（服务端专用模块）

文件名包含 `.server.` 的文件：
- **不会被识别为路由**
- 仅在服务端执行
- 用于存放服务端逻辑、数据库操作等

| 文件模式 | 用途 | 示例 |
|----------|------|------|
| `*.server.ts` | 纯服务端工具 | `verify.server.ts` |
| `*.server.tsx` | 服务端组件/逻辑 | `note-editor.server.tsx` |

**示例结构：**
```
app/routes/_auth/
├── verify.tsx              # 路由文件（客户端+服务端）
├── verify.server.ts        # 服务端验证逻辑（非路由）
└── webauthn/
    ├── authentication.ts   # 路由文件
    ├── registration.ts     # 路由文件
    └── utils.server.ts     # WebAuthn 服务端工具
```

**使用方式：**
```typescript
// app/routes/_auth/verify.tsx
import { validateRequest } from './verify.server.ts'  // 仅服务端可见

export async function action({ request }) {
  const formData = await request.formData()
  return validateRequest(request, formData)  // 调用服务端逻辑
}
```

### 3. `.client.` 后缀（客户端专用模块）

文件名包含 `.client.` 的文件：
- **不会被识别为路由**
- 仅在客户端执行
- 用于存放浏览器专用逻辑

虽然当前项目中未大量使用，但框架支持此约定。

### 4. 路由与服务端逻辑配对模式

对于复杂路由，常采用以下模式：

```
app/routes/_auth/onboarding/
├── index.tsx           # 路由组件 + loader
├── index.server.ts     # 服务端 action 逻辑（非路由）
├── $provider.tsx       # 动态路由组件
└── $provider.server.ts # 动态路由服务端逻辑（非路由）
```

这种模式的优势：
- 服务端敏感代码不会泄露到客户端
- 逻辑分离清晰
- 便于测试

---

## 特殊文件约定

### 1. 测试文件

```
**/*.test.{js,jsx,ts,tsx}
```

测试文件与路由文件共置，但不会被识别为路由：

```
app/routes/_auth/auth.$provider/
├── callback.ts
├── callback.test.ts     # 测试文件（忽略）
└── index.ts
```

### 2. 隐藏文件

```
.*
```

以 `.` 开头的文件被忽略。

### 3. 双下划线文件

```
**/__*.*
```

以 `__` 开头的文件被忽略，常用于内部工具。

### 4. CSS 文件

```
**/*.css
```

CSS 文件被忽略，样式应通过其他方式引入。

---

## 配置与忽略规则

### 1. 完整忽略规则（来自 `app/routes.ts`）

```typescript
ignoredRouteFiles: [
  '.*',                    // 隐藏文件
  '**/*.css',              // 样式文件
  '**/*.test.{js,jsx,ts,tsx}',  // 测试文件
  '**/__*.*',              // 双下划线文件
  '**/*.server.*',         // 服务端专用模块
  '**/*.client.*',         // 客户端专用模块
]
```

### 2. 转义 `server`/`client` 关键词

如果需要创建文件名包含 `server` 或 `client` 的路由，使用方括号转义：

```
my-route.[server].tsx    → /my-route/server
my-route.[client].tsx    → /my-route/client
```

---

## 实际项目示例分析

### 示例 1：完整的 Notes 路由结构

```
app/routes/users/$username/notes/
├── _layout.tsx              # 布局 → /users/:username/notes
├── index.tsx                # 列表 → /users/:username/notes
├── $noteId.tsx              # 详情 → /users/:username/notes/:noteId
├── $noteId_.edit.tsx        # 编辑 → /users/:username/notes/:noteId/edit
├── new.tsx                  # 新建 → /users/:username/notes/new
├── +shared/
│   ├── note-editor.tsx      # 共享编辑器组件
│   └── note-editor.server.tsx # 共享服务端逻辑
└── +types/                  # 自动生成的类型
    ├── _layout.ts
    ├── index.ts
    ├── $noteId.ts
    └── $noteId_.edit.ts
```

**生成的路由树：**
```tsx
<Route path="users/:username/notes" file=".../_layout.tsx">
  <Route index file=".../index.tsx" />
  <Route path=":noteId" file=".../$noteId.tsx" />
  <Route path=":noteId/edit" file=".../$noteId_.edit.tsx" />
  <Route path="new" file=".../new.tsx" />
</Route>
```

### 示例 2：认证路由组

```
app/routes/_auth/
├── login.tsx               # /login
├── logout.tsx              # /logout
├── signup.tsx              # /signup
├── verify.tsx              # /verify
├── verify.server.ts        # 服务端验证逻辑
├── auth.$provider/
│   ├── callback.ts         # /auth/:provider/callback
│   ├── callback.test.ts    # 测试（忽略）
│   └── index.ts            # /auth/:provider
├── onboarding/
│   ├── index.tsx           # /onboarding
│   ├── index.server.ts     # 服务端逻辑
│   ├── $provider.tsx       # /onboarding/:provider
│   └── $provider.server.ts # 服务端逻辑
└── webauthn/
    ├── authentication.ts   # /webauthn/authentication
    ├── registration.ts     # /webauthn/registration
    └── utils.server.ts     # WebAuthn 工具
```

### 示例 3：设置页面嵌套布局

```
app/routes/settings/profile/
├── _layout.tsx             # 主布局 → /settings/profile
├── index.tsx               # /settings/profile
├── password.tsx            # /settings/profile/password
├── password_.create.tsx    # /settings/profile/password/create
├── two-factor/
│   ├── _layout.tsx         # 子布局 → /settings/profile/two-factor
│   ├── index.tsx           # /settings/profile/two-factor
│   ├── enable.tsx          # /settings/profile/two-factor/enable
│   └── verify.tsx          # /settings/profile/two-factor/verify
└── +types/                 # 自动生成类型
```

---

## 总结

### 核心约定速查表

| 约定 | 符号 | 说明 | 示例 |
|------|------|------|------|
| 动态参数 | `$` | URL 动态段 | `$username`, `$noteId` |
| 路径组 | `_` 前缀 | 组织路由，不影响 URL | `_auth`, `_marketing` |
| 布局文件 | `_layout.tsx` | 目录级布局组件 | `settings/profile/_layout.tsx` |
| 共置目录 | `+` 前缀 | 非路由资源目录 | `+shared`, `+types`, `+logos` |
| 服务端模块 | `.server.` | 仅服务端，非路由 | `verify.server.ts` |
| 客户端模块 | `.client.` | 仅客户端，非路由 | `utils.client.ts` |
| 转义点号 | `[.]` | 文件名中的点 | `robots[.]txt.ts` |
| 路径后缀 | `_` 后缀 | 额外 URL 段 | `password_.create.tsx` |
| 通配符 | `$` | 根级兜底路由 | `$.tsx` |

### 设计优势

1. **类型安全**：自动生成 `+types/` 目录，提供完整的 TypeScript 类型支持
2. **代码共置**：路由与相关逻辑、组件、测试放在一起
3. **清晰组织**：路径组和布局系统支持大型应用的路由组织
4. **服务端安全**：`.server.` 文件确保敏感代码不会泄露到客户端
5. **与 React Router 兼容**：贴近原生约定，降低学习成本

### 调试与验证

```bash
# 查看生成的路由树
npx react-router routes

# 生成类型定义
npx react-router typegen

# 完整类型检查
npm run typecheck
```
