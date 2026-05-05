# Epic Stack 权限控制链路分析报告（第二轮）

> 分析日期：2026-05-05
> 分析范围：编辑入口越权风险核实、角色变更后权限刷新机制、服务端拒绝链路验证

---

## 修正声明

**第一轮报告结论需要修正**：

第一轮报告声称"数据库、后端、前端三者在权限控制逻辑上保持高度一致"，这一结论**过于笼统且不准确**。

经过第二轮深入分析，发现：
- ✅ **删除操作**：三者确实保持一致（使用显式 RBAC 权限检查）
- ❌ **编辑操作**：三者存在**不一致**（后端使用隐式 ownerId 检查，绕过了 RBAC 系统）
- ⚠️ **角色变更场景**：存在"前端权限数据过期，但后端实时校验"的时间窗口

---

## 第一部分：编辑入口越权风险核实

### 1.1 删除操作（基准对照）

删除操作使用**显式的 RBAC 权限检查**，是正确的实现方式。

#### 后端校验代码

**文件**: `app/routes/users/$username/notes/$noteId.tsx:56-90`

```typescript
export async function action({ request }: Route.ActionArgs) {
  const userId = await requireUserId(request)
  // ... 解析表单数据 ...
  
  const note = await prisma.note.findFirst({
    select: { id: true, ownerId: true, owner: { select: { username: true } } },
    where: { id: noteId },
  })
  
  const isOwner = note.ownerId === userId  // 1. 判断所有权
  
  await requireUserWithPermission(  // 2. 显式 RBAC 权限检查
    request,
    isOwner ? `delete:note:own` : `delete:note:any`,  // 关键：根据所有权选择权限
  )
  
  await prisma.note.delete({ where: { id: note.id } })
  // ...
}
```

#### 权限检查函数

**文件**: `app/utils/permissions.server.ts:6-41`

```typescript
export async function requireUserWithPermission(
  request: Request,
  permission: PermissionString,
) {
  const userId = await requireUserId(request)
  const permissionData = parsePermissionString(permission)
  const user = await prisma.user.findFirst({
    select: { id: true },
    where: {
      id: userId,
      roles: {
        some: {
          permissions: {
            some: {
              ...permissionData,
              access: permissionData.access
                ? { in: permissionData.access }
                : undefined,
            },
          },
        },
      },
    },
  })
  if (!user) {
    throw data({ error: 'Unauthorized', ... }, { status: 403 })
  }
  return user.id
}
```

#### 前端 UI 检查

**文件**: `app/routes/users/$username/notes/$noteId.tsx:96-101`

```typescript
const user = useOptionalUser()
const isOwner = user?.id === loaderData.note.ownerId
const canDelete = userHasPermission(  // 与后端相同的权限检查逻辑
  user,
  isOwner ? `delete:note:own` : `delete:note:any`,
)
```

#### 删除操作结论

| 层级 | 实现方式 | 一致性 |
|------|----------|--------|
| 数据库 | `Permission` 表包含 `delete:note:own` 和 `delete:note:any` | ✅ |
| 后端 | `requireUserWithPermission` 显式检查 RBAC 权限 | ✅ 一致 |
| 前端 | `userHasPermission` 相同逻辑检查 | ✅ 一致 |

---

### 1.2 编辑操作（问题发现）

编辑操作使用**隐式的 ownerId 检查**，绕过了 RBAC 权限系统。

#### 编辑路由 Loader

**文件**: `app/routes/users/$username/notes/$noteId_.edit.tsx:10-32`

```typescript
export async function loader({ params, request }: Route.LoaderArgs) {
  const userId = await requireUserId(request)
  const note = await prisma.note.findFirst({
    select: {
      id: true,
      title: true,
      content: true,
      images: { ... },
    },
    where: {
      id: params.noteId,
      ownerId: userId,  // ⚠️ 关键问题：直接检查 ownerId，不是 RBAC 权限
    },
  })
  invariantResponse(note, 'Not found', { status: 404 })
  return { note }
}
```

#### 共享 Editor Action

**文件**: `app/routes/users/$username/notes/+shared/note-editor.server.tsx:27-83`

```typescript
export async function action({ request }: ActionFunctionArgs) {
  const userId = await requireUserId(request)
  
  const submission = await parseWithZod(formData, {
    schema: NoteEditorSchema.superRefine(async (data, ctx) => {
      if (!data.id) return
      
      const note = await prisma.note.findUnique({
        select: { id: true },
        where: { 
          id: data.id, 
          ownerId: userId  // ⚠️ 同样：直接检查 ownerId，不是 RBAC 权限
        },
      })
      if (!note) {
        ctx.addIssue({
          code: z.ZodIssueCode.custom,
          message: 'Note not found',  // ⚠️ 甚至返回 404 而不是 403
        })
      }
    }),
    // ...
  })
  // ...
}
```

#### 新建笔记 Loader

**文件**: `app/routes/users/$username/notes/new.tsx:7-10`

```typescript
export async function loader({ request }: Route.LoaderArgs) {
  await requireUserId(request)  // ⚠️ 只检查登录，不检查 create:note:own 权限
  return {}
}
```

#### 前端编辑按钮显示逻辑

**文件**: `app/routes/users/$username/notes/$noteId.tsx:102, 155-164`

```typescript
const displayBar = canDelete || isOwner  // ⚠️ 编辑按钮显示不检查权限

// ... 编辑按钮始终在 displayBar 为 true 时显示：
<Button asChild>
  <Link to="edit">
    <Icon name="pencil-1">
      <span className="max-md:hidden">Edit</span>
    </Icon>
  </Link>
</Button>
```

#### 编辑操作结论

| 层级 | 实现方式 | 一致性 |
|------|----------|--------|
| 数据库 | `Permission` 表包含 `update:note:own` 和 `update:note:any` | ✅ 定义了权限 |
| 后端 Loader | `where: { ownerId: userId }` - 隐式检查 | ❌ **不一致**，绕过 RBAC |
| 后端 Action | `superRefine` 中 `ownerId: userId` - 隐式检查 | ❌ **不一致**，绕过 RBAC |
| 前端 | 编辑按钮不检查 `update:note` 权限 | ❌ **不一致** |

---

### 1.3 编辑权限不一致的影响分析

#### 场景 1：管理员编辑他人笔记

| 组件 | 行为 | 结果 |
|------|------|------|
| 数据库权限 | 管理员有 `update:note:any` 权限 | ✅ 有权限 |
| 前端 UI | 编辑按钮显示条件：`isOwner \|\| canDelete` | ⚠️ 如果不是 owner 但有 delete:any，编辑按钮会显示 |
| 点击编辑链接 | 导航到 `/$noteId_.edit` | |
| 编辑 Loader | `where: { id: noteId, ownerId: userId }` | ❌ **返回 404，拒绝访问** |

**结论**：管理员虽然有 `update:note:any` 权限，但无法编辑他人的笔记。

#### 场景 2：普通用户编辑自己的笔记

| 组件 | 行为 | 结果 |
|------|------|------|
| 数据库权限 | 用户有 `update:note:own` 权限 | ✅ 有权限 |
| 前端 UI | `isOwner` 为 true，编辑按钮显示 | ✅ 显示 |
| 编辑 Loader | `ownerId === userId` 匹配 | ✅ 允许访问 |
| 编辑 Action | `ownerId === userId` 匹配 | ✅ 允许提交 |

**结论**：普通用户编辑自己的笔记，功能正常，但**实现方式绕过了 RBAC 系统**。

#### 场景 3：权限被撤销的用户

假设管理员通过后台移除了某用户的 `update:note:own` 权限：

| 组件 | 行为 | 结果 |
|------|------|------|
| 数据库权限 | 用户不再有 `update:note:own` 权限 | ❌ 无权限 |
| 编辑 Loader | `ownerId === userId` 仍为 true | ⚠️ **仍允许访问** |
| 编辑 Action | `ownerId === userId` 仍为 true | ⚠️ **仍允许提交** |

**结论**：由于使用的是 `ownerId` 检查而非 RBAC 权限检查，**权限撤销无效**。

---

## 第二部分：角色变更后权限刷新机制分析

### 2.1 前端权限数据来源

#### Root Loader 查询

**文件**: `app/root.tsx:71-133`

```typescript
export async function loader({ request }: Route.LoaderArgs) {
  const userId = await getUserId(request)
  
  const user = userId
    ? await prisma.user.findUnique({
        select: {
          id: true,
          name: true,
          username: true,
          image: { ... },
          roles: {
            select: {
              name: true,
              permissions: {  // 权限数据在这里查询
                select: { entity: true, action: true, access: true },
              },
            },
          },
        },
        where: { id: userId },
      })
    : null
  
  return data({ user, ... })
}
```

#### 前端获取方式

**文件**: `app/utils/user.ts:10-16`

```typescript
export function useOptionalUser() {
  const data = useRouteLoaderData<typeof rootLoader>('root')
  if (!data || !isUser(data.user)) {
    return undefined
  }
  return data.user  // 返回的是 root loader 缓存的数据
}
```

---

### 2.2 React Router Loader 执行时机

#### 默认行为

根据 React Router 文档和代码分析：

| 触发方式 | root loader 是否重新执行 | 权限数据是否刷新 |
|----------|-------------------------|-----------------|
| 点击 `<Link>` 导航 | ✅ 是 | ✅ 刷新 |
| 调用 `navigate()` | ✅ 是 | ✅ 刷新 |
| 使用 `useFetcher` 提交 | ❌ 否 | ❌ **不刷新** |
| 浏览器刷新页面 | ✅ 是 | ✅ 刷新 |
| 手动调用 `revalidate()` | ✅ 是 | ✅ 刷新 |

#### 项目中的 revalidate 使用

**文件**: `app/utils/client-hints.tsx:41-46`

```typescript
export function ClientHintCheck({ nonce }: { nonce: string }) {
  const { revalidate } = useRevalidator()
  React.useEffect(
    () => subscribeToSchemeChange(() => revalidate()),
    [revalidate],
  )
  // ...
}
```

这是唯一使用 `useRevalidator` 的地方，用于主题切换时刷新，**不是用于权限刷新**。

---

### 2.3 角色变更场景分析

#### 场景：管理员在后台修改用户角色

假设：
1. 用户 A 登录系统，当前角色为 `user`
2. 用户 A 打开笔记详情页，前端获取到的权限数据包含 `delete:note:own`
3. 管理员通过后台操作，将用户 A 的角色从 `user` 移除（或添加到 `admin`）
4. 用户 A 没有刷新页面，继续操作

**时间窗口分析**：

```
时间点 T1: 用户 A 登录，root loader 执行
         → 前端权限数据: { roles: [{ name: 'user', permissions: [...] }] }
         → 前端认为自己有 delete:note:own 权限

时间点 T2: 管理员修改用户 A 的角色（移除 user 角色）
         → 数据库已更新
         → 前端不知情，仍使用缓存的旧数据

时间点 T3: 用户 A 点击删除按钮
         → 前端: canDelete = true（基于旧数据）
         → 表单提交到 action
         → 后端 requireUserWithPermission 执行
         → 后端查询数据库实时权限: 用户 A 已无 user 角色
         → 后端 throw 403 错误
         → ErrorBoundary 显示 "You are not allowed to do that"
```

#### 证据链：服务端拒绝流程

| 步骤 | 文件位置 | 代码 |
|------|----------|------|
| 1. 前端显示按钮 | `$noteId.tsx:98-101` | `userHasPermission(user, isOwner ? 'delete:note:own' : 'delete:note:any')` |
| 2. 用户点击删除 | `$noteId.tsx:152-154` | 条件渲染 `{canDelete ? <DeleteNote ... /> : null}` |
| 3. 表单提交 | `$noteId.tsx:186-204` | `<Form method="POST">` 提交到 action |
| 4. 后端权限检查 | `permissions.server.ts:6-41` | `requireUserWithPermission` 查询数据库 **实时** 权限 |
| 5. 无权限抛出错误 | `permissions.server.ts:31-39` | `throw data({ error: 'Unauthorized' }, { status: 403 })` |
| 6. 错误被捕获 | `$noteId.tsx:226-237` | `ErrorBoundary` 处理 403 状态 |
| 7. 显示错误信息 | `$noteId.tsx:230` | `<p>You are not allowed to do that</p>` |

**关键证据**：

1. **前端权限来源**：`useRouteLoaderData('root')` - 来自 root loader 的**缓存数据**
   - 文件：`app/utils/user.ts:11`

2. **后端权限检查**：`requireUserWithPermission` 每次都**实时查询数据库**
   - 文件：`app/utils/permissions.server.ts:12-29`
   - 关键代码：`prisma.user.findFirst({ where: { id: userId, roles: { some: { permissions: { ... } } } } })`

3. **403 错误处理**：
   - 抛出：`app/utils/permissions.server.ts:37`
   - 捕获：`app/routes/users/$username/notes/$noteId.tsx:230`

---

### 2.4 权限刷新机制结论

| 问题 | 答案 | 证据 |
|------|------|------|
| 前端权限数据是否可能过期？ | ✅ 是 | 使用 `useRouteLoaderData` 获取缓存数据 |
| 什么情况下会过期？ | 角色变更后未重新导航 | React Router loader 缓存机制 |
| 过期数据是否会导致安全问题？ | ❌ 不会 | 后端每次都实时查询数据库 |
| 用户体验如何？ | 可能困惑 | 看到按钮但点击后被拒绝 |

**这是合理的设计**：
- 前端：使用缓存数据优化体验（可能过期）
- 后端：实时校验确保安全（始终最新）

---

## 第三部分：完整一致性对照表

### 3.1 按操作类型对比

| 操作 | 数据库权限 | 后端校验方式 | 前端检查方式 | 一致性状态 |
|------|-----------|-------------|-------------|-----------|
| **删除笔记** | `delete:note:own/any` | `requireUserWithPermission` (RBAC) | `userHasPermission` (相同逻辑) | ✅ **完全一致** |
| **编辑笔记** | `update:note:own/any` | `ownerId === userId` (隐式) | 不检查权限，仅 `isOwner` | ❌ **不一致** |
| **创建笔记** | `create:note:own` | `requireUserId` (仅登录) | 无检查（通过导航） | ❌ **不一致** |
| **读取笔记** | `read:note:own/any` | 未统一实现 | 未统一实现 | ⚠️ 需确认 |

### 3.2 角色权限定义与实际使用

#### 数据库定义的权限

**文件**: `prisma/migrations/20250221233640_init/migration.sql:240-255`

```sql
-- user 权限 (access='own')
INSERT INTO Permission VALUES(...,'create','user','own',...);
INSERT INTO Permission VALUES(...,'read','user','own',...);
INSERT INTO Permission VALUES(...,'update','user','own',...);
INSERT INTO Permission VALUES(...,'delete','user','own',...);
INSERT INTO Permission VALUES(...,'create','note','own',...);
INSERT INTO Permission VALUES(...,'read','note','own',...);
INSERT INTO Permission VALUES(...,'update','note','own',...);
INSERT INTO Permission VALUES(...,'delete','note','own',...);

-- admin 权限 (access='any')
INSERT INTO Permission VALUES(...,'create','user','any',...);
INSERT INTO Permission VALUES(...,'read','user','any',...);
INSERT INTO Permission VALUES(...,'update','user','any',...);
INSERT INTO Permission VALUES(...,'delete','user','any',...);
INSERT INTO Permission VALUES(...,'create','note','any',...);
INSERT INTO Permission VALUES(...,'read','note','any',...);
INSERT INTO Permission VALUES(...,'update','note','any',...);
INSERT INTO Permission VALUES(...,'delete','note','any',...);
```

#### 实际使用的权限

| 权限字符串 | 定义位置 | 使用位置 | 是否被使用 |
|-----------|---------|---------|-----------|
| `delete:note:own` | 迁移文件 | `$noteId.tsx:80` | ✅ 使用 |
| `delete:note:any` | 迁移文件 | `$noteId.tsx:80` | ✅ 使用 |
| `update:note:own` | 迁移文件 | 无 | ❌ **未使用** |
| `update:note:any` | 迁移文件 | 无 | ❌ **未使用** |
| `create:note:own` | 迁移文件 | 无 | ❌ **未使用** |
| `create:note:any` | 迁移文件 | 无 | ❌ **未使用** |
| `read:note:own` | 迁移文件 | 无 | ❌ **未使用** |
| `read:note:any` | 迁移文件 | 无 | ❌ **未使用** |

---

## 第四部分：问题汇总与建议

### 4.1 已确认的问题

#### 问题 1：编辑权限绕过 RBAC 系统

**严重程度**: 🔴 高

**证据**:
- `app/routes/users/$username/notes/$noteId_.edit.tsx:27` - `where: { ownerId: userId }`
- `app/routes/users/$username/notes/+shared/note-editor.server.tsx:40` - `where: { ownerId: userId }`

**影响**:
- 管理员无法编辑他人的笔记（尽管有 `update:note:any` 权限）
- 权限撤销无法生效（即使移除 `update:note:own`，用户仍可编辑）

#### 问题 2：创建权限未检查

**严重程度**: 🟡 中

**证据**:
- `app/routes/users/$username/notes/new.tsx:8` - 仅 `requireUserId(request)`

**影响**:
- 任何登录用户都可以创建笔记，无法通过 RBAC 控制

#### 问题 3：前端编辑按钮不检查权限

**严重程度**: 🟡 中

**证据**:
- `app/routes/users/$username/notes/$noteId.tsx:102` - `displayBar = canDelete || isOwner`
- 编辑按钮没有对应的 `canEdit` 检查

**影响**:
- 与后端实现方式不一致（虽然后端也不检查 RBAC）

### 4.2 设计特性（非问题）

#### 特性：前端权限数据可能过期

**状态**: ✅ 合理设计

**说明**:
- 前端使用 `useRouteLoaderData` 获取缓存的权限数据
- 后端 `requireUserWithPermission` 每次都实时查询数据库
- 这是"前端优化体验，后端保障安全"的合理设计

**证据**:
- 前端：`app/utils/user.ts:11` - `useRouteLoaderData('root')`
- 后端：`app/utils/permissions.server.ts:12-29` - 每次都 `prisma.user.findFirst()`

### 4.3 修复建议

#### 建议 1：统一编辑权限检查

将编辑路由改为与删除路由相同的实现方式：

```typescript
// 修改前
where: { id: noteId, ownerId: userId }

// 修改后
const isOwner = note.ownerId === userId
await requireUserWithPermission(
  request,
  isOwner ? 'update:note:own' : 'update:note:any'
)
```

#### 建议 2：添加创建权限检查

```typescript
// new.tsx loader
export async function loader({ request }: Route.LoaderArgs) {
  await requireUserWithPermission(request, 'create:note:own')
  return {}
}
```

#### 建议 3：前端添加 canEdit 检查

```typescript
// $noteId.tsx
const canEdit = userHasPermission(
  user,
  isOwner ? 'update:note:own' : 'update:note:any'
)
const displayBar = canDelete || canEdit  // 修改条件

// 条件渲染
{canEdit && <Link to="edit">Edit</Link>}
```

---

## 第五部分：可复核的证据索引

### 5.1 删除操作（正确实现）

| 文件路径 | 行号 | 说明 |
|----------|------|------|
| `app/utils/permissions.server.ts` | 6-41 | `requireUserWithPermission` 实现 |
| `app/utils/user.ts` | 48-62 | `userHasPermission` 实现 |
| `app/routes/.../$noteId.tsx` | 77-81 | 删除 action 中的权限选择 |
| `app/routes/.../$noteId.tsx` | 98-101 | 前端 `canDelete` 计算 |
| `app/routes/.../$noteId.tsx` | 152-154 | 前端删除按钮条件渲染 |
| `app/utils/permissions.server.ts` | 37 | 403 错误抛出 |
| `app/routes/.../$noteId.tsx` | 230 | 403 错误处理 |

### 5.2 编辑操作（问题实现）

| 文件路径 | 行号 | 说明 |
|----------|------|------|
| `app/routes/.../$noteId_.edit.tsx` | 25-28 | Loader 使用 `ownerId: userId` |
| `app/routes/.../note-editor.server.tsx` | 40 | Action 使用 `ownerId: userId` |
| `app/routes/.../$noteId.tsx` | 102 | 编辑按钮条件 `canDelete \|\| isOwner` |
| `app/routes/.../$noteId.tsx` | 155-164 | 编辑按钮无条件渲染 |
| `app/routes/.../new.tsx` | 8 | 创建仅检查 `requireUserId` |

### 5.3 权限模型定义

| 文件路径 | 行号 | 说明 |
|----------|------|------|
| `prisma/schema.prisma` | 99-124 | Permission 和 Role 模型 |
| `prisma/migrations/.../migration.sql` | 240-255 | 权限初始化数据 |
| `prisma/migrations/.../migration.sql` | 260-275 | 角色权限分配 |

### 5.4 数据刷新机制

| 文件路径 | 行号 | 说明 |
|----------|------|------|
| `app/root.tsx` | 79-101 | Root loader 查询用户权限 |
| `app/utils/user.ts` | 10-16 | `useOptionalUser` 使用 `useRouteLoaderData` |
| `app/utils/client-hints.tsx` | 42-45 | 唯一使用 `revalidate` 的地方 |

---

## 最终结论

### 修正后的一致性判断

| 声明 | 真实性 | 证据 |
|------|--------|------|
| 删除操作三者一致 | ✅ 真实 | 删除使用 `requireUserWithPermission` 和 `userHasPermission`，逻辑相同 |
| 编辑操作三者一致 | ❌ **虚假** | 编辑使用 `ownerId === userId`，绕过 RBAC；前端也不检查权限 |
| 角色变更后前端可能显示过期权限 | ✅ 真实 | 前端使用 `useRouteLoaderData` 缓存数据 |
| 过期权限会导致安全问题 | ❌ **虚假** | 后端每次都实时查询数据库，前端仅控制 UI 显示 |

### 核心发现

1. **删除操作**：实现正确，数据库、后端、前端三者完全一致
2. **编辑操作**：实现有缺陷，绕过 RBAC 系统，导致 `update:note` 权限形同虚设
3. **创建操作**：同样有缺陷，仅检查登录，不检查 `create:note` 权限
4. **权限刷新**：设计合理，前端缓存数据但后端实时校验，无安全风险

### 安全评估

| 风险项 | 风险等级 | 说明 |
|--------|---------|------|
| 编辑权限绕过 RBAC | 🔴 高 | 权限定义与实际执行不一致，管理员功能受限 |
| 创建权限未检查 | 🟡 中 | 任何登录用户都可创建，无法细粒度控制 |
| 前端权限数据过期 | 🟢 低 | 仅影响 UX，不影响安全，后端有防护 |
