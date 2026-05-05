# Epic-Stack 笔记 RBAC 权限管控机制分析报告

## 一、权限系统架构概述

Epic-Stack 采用了经典的 **RBAC (Role-Based Access Control)** 模型，并创新性地结合了 **所有权检查** 和 **权限字符串** 机制，实现了前后端一致的权限管控。

### 1.1 数据模型设计

核心数据模型位于 `prisma/schema.prisma`，采用三表结构：

#### (1) Permission 表 - 权限定义

```prisma
model Permission {
  id          String @id @default(cuid())
  action      String // e.g. create, read, update, delete
  entity      String // e.g. note, user, etc.
  access      String // e.g. own or any
  description String @default("")

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  roles Role[]

  @@unique([action, entity, access])
}
```

**关键字段说明**：
- `action`: 操作类型，如 `create`、`read`、`update`、`delete`
- `entity`: 实体类型，如 `note`、`user`
- `access`: 访问范围，`own` 表示仅自己的，`any` 表示任意的

#### (2) Role 表 - 角色定义

```prisma
model Role {
  id          String @id @default(cuid())
  name        String @unique
  description String @default("")

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  users       User[]
  permissions Permission[]
}
```

#### (3) User 表 - 用户与角色关联

```prisma
model User {
  id       String  @id @default(cuid())
  // ... 其他字段

  roles       Role[]      // 多对多关系
  notes       Note[]      // 用户拥有的笔记
}
```

#### (4) Note 表 - 笔记与所有者

```prisma
model Note {
  id      String @id @default(cuid())
  title   String
  content String

  owner   User   @relation(fields: [ownerId], references: [id], onDelete: Cascade, onUpdate: Cascade)
  ownerId String  // 关键：明确的所有权字段
}
```

---

### 1.2 权限字符串格式定义

权限字符串定义于 `app/utils/user.ts`，采用 `action:entity[:access]` 格式：

```typescript
type Action = 'create' | 'read' | 'update' | 'delete'
type Entity = 'user' | 'note'
type Access = 'own' | 'any' | 'own,any' | 'any,own'

export type PermissionString =
  | `${Action}:${Entity}`
  | `${Action}:${Entity}:${Access}`
```

**示例**：
| 权限字符串 | 含义 |
|-----------|------|
| `delete:note:own` | 删除自己的笔记 |
| `delete:note:any` | 删除任意笔记（管理员权限） |
| `update:note:own` | 更新自己的笔记 |
| `read:note:any` | 读取任意笔记 |

---

## 二、服务端权限检查机制

服务端权限检查核心位于 `app/utils/permissions.server.ts`。

### 2.1 `requireUserWithPermission` 函数

这是服务端权限检查的核心函数：

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
    throw data(
      {
        error: 'Unauthorized',
        requiredPermission: permissionData,
        message: `Unauthorized: required permissions: ${permission}`,
      },
      { status: 403 },
    )
  }
  return user.id
}
```

**检查逻辑**：
1. 先通过 `requireUserId` 确保用户已登录
2. 解析权限字符串为 `{ action, entity, access }`
3. 通过 Prisma 的嵌套查询检查用户角色是否包含所需权限
4. 使用 `roles.some.permissions.some` 进行多对多关系查询
5. `access` 字段使用 `{ in: permissionData.access }` 支持多个值（如 `own,any`）

### 2.2 `requireUserWithRole` 函数

直接检查用户是否有特定角色：

```typescript
export async function requireUserWithRole(request: Request, name: string) {
  const userId = await requireUserId(request)
  const user = await prisma.user.findFirst({
    select: { id: true },
    where: { id: userId, roles: { some: { name } } },
  })
  // ... 同上的错误处理
}
```

---

## 三、前端权限检查机制

前端权限检查核心位于 `app/utils/user.ts`。

### 3.1 权限字符串解析函数

```typescript
export function parsePermissionString(permissionString: PermissionString) {
  const [action, entity, access] = permissionString.split(':') as [
    Action,
    Entity,
    Access | undefined,
  ]
  return {
    action,
    entity,
    access: access ? (access.split(',') as Array<Access>) : undefined,
  }
}
```

### 3.2 `userHasPermission` 函数

前端组件中用于检查权限的核心函数：

```typescript
export function userHasPermission(
  user: Pick<ReturnType<typeof useUser>, 'roles'> | null | undefined,
  permission: PermissionString,
) {
  if (!user) return false
  const { action, entity, access } = parsePermissionString(permission)
  return user.roles.some((role) =>
    role.permissions.some(
      (permission) =>
        permission.entity === entity &&
        permission.action === action &&
        (!access || access.includes(permission.access)),
    ),
  )
}
```

**检查逻辑**：
1. 如果用户为 null/undefined，直接返回 false
2. 解析权限字符串
3. 遍历用户的所有角色，检查是否有一个角色包含所需权限
4. 检查条件：`entity` 匹配 + `action` 匹配 + (无 access 要求 或 access 包含权限的 access)

### 3.3 `userHasRole` 函数

检查用户是否有特定角色：

```typescript
export function userHasRole(
  user: Pick<ReturnType<typeof useUser>, 'roles'> | null,
  role: string,
) {
  if (!user) return false
  return user.roles.some((r) => r.name === role)
}
```

---

## 四、根路由权限数据下发机制

### 4.1 Root Loader 权限查询

根路由的 loader 位于 `app/root.tsx`，负责在每次请求时获取用户的完整权限数据：

```typescript
export async function loader({ request }: Route.LoaderArgs) {
  const timings = makeTimings('root loader')
  const userId = await time(() => getUserId(request), {
    timings,
    type: 'getUserId',
    desc: 'getUserId in root',
  })

  const user = userId
    ? await time(
        () =>
          prisma.user.findUnique({
            select: {
              id: true,
              name: true,
              username: true,
              image: { select: { objectKey: true } },
              // 关键：同时查询角色和权限
              roles: {
                select: {
                  name: true,
                  permissions: {
                    select: { entity: true, action: true, access: true },
                  },
                },
              },
            },
            where: { id: userId },
          }),
        { timings, type: 'find user', desc: 'find user in root' },
      )
    : null

  // ... 其他处理

  return data(
    {
      user,  // 包含 roles.permissions 数据
      requestInfo: { /* ... */ },
      ENV: getEnv(),
      toast,
      honeyProps,
    },
    // ...
  )
}
```

**权限数据结构**：
```typescript
user: {
  id: string,
  name: string | null,
  username: string,
  image: { objectKey: string } | null,
  roles: [
    {
      name: string,  // 如 'admin', 'user'
      permissions: [
        { entity: 'note', action: 'create', access: 'own' },
        { entity: 'note', action: 'read', access: 'own' },
        { entity: 'note', action: 'update', access: 'own' },
        { entity: 'note', action: 'delete', access: 'own' },
        // ... 更多权限
      ]
    }
  ]
}
```

### 4.2 前端获取用户数据

通过 `useOptionalUser` 和 `useUser` hooks 获取：

```typescript
// app/utils/user.ts
export function useOptionalUser() {
  const data = useRouteLoaderData<typeof rootLoader>('root')
  if (!data || !isUser(data.user)) {
    return undefined
  }
  return data.user
}

export function useUser() {
  const maybeUser = useOptionalUser()
  if (!maybeUser) {
    throw new Error('No user found in root loader...')
  }
  return maybeUser
}
```

**关键机制**：
- 使用 React Router 的 `useRouteLoaderData('root')` 获取根路由 loader 数据
- 由于根路由是所有子路由的父级，其 loader 数据在所有页面都可访问
- 这确保了权限数据在前端全局可用，且每次请求都会刷新

---

## 五、笔记读写删操作的权限实现

### 5.1 所有权检查与权限字符串的结合模式

Epic-Stack 的核心设计模式是：**先判断是否为所有者，再根据结果选择不同的权限字符串进行检查**。

**核心模式**：
```typescript
const isOwner = note.ownerId === userId
await requireUserWithPermission(
  request,
  isOwner ? `delete:note:own` : `delete:note:any`,
)
```

这种模式的优点：
1. **普通用户**：只能操作自己的笔记（需要 `:own` 权限）
2. **管理员**：可以操作任意笔记（需要 `:any` 权限）
3. **安全性**：服务端强制检查，无法绕过

### 5.2 删除操作 - 完整示例

#### (1) 服务端 Action 检查

文件：`app/routes/users/$username/notes/$noteId.tsx`

```typescript
export async function action({ request }: Route.ActionArgs) {
  const userId = await requireUserId(request)
  const formData = await request.formData()
  const submission = parseWithZod(formData, {
    schema: DeleteFormSchema,
  })
  // ... 表单验证

  const { noteId } = submission.value

  // 1. 先查询笔记获取 ownerId
  const note = await prisma.note.findFirst({
    select: { id: true, ownerId: true, owner: { select: { username: true } } },
    where: { id: noteId },
  })
  invariantResponse(note, 'Not found', { status: 404 })

  // 2. 判断是否为所有者
  const isOwner = note.ownerId === userId
  
  // 3. 根据所有权选择不同的权限字符串
  await requireUserWithPermission(
    request,
    isOwner ? `delete:note:own` : `delete:note:any`,
  )

  // 4. 执行删除
  await prisma.note.delete({ where: { id: note.id } })

  return redirectWithToast(`/users/${note.owner.username}/notes`, {
    type: 'success',
    title: 'Success',
    description: 'Your note has been deleted.',
  })
}
```

#### (2) 前端组件控制

同样的逻辑用于控制删除按钮的显示：

```typescript
export default function NoteRoute({
  loaderData,
  actionData,
}: Route.ComponentProps) {
  const user = useOptionalUser()
  const isOwner = user?.id === loaderData.note.ownerId
  
  // 前端使用相同的权限检查逻辑
  const canDelete = userHasPermission(
    user,
    isOwner ? `delete:note:own` : `delete:note:any`,
  )
  const displayBar = canDelete || isOwner

  return (
    <section>
      {/* ... 笔记内容 ... */}
      
      {displayBar ? (
        <div className={floatingToolbarClassName}>
          <div className="grid flex-1 grid-cols-2 justify-end gap-2">
            {/* 只有 canDelete 为 true 时才显示删除按钮 */}
            {canDelete ? (
              <DeleteNote id={loaderData.note.id} actionData={actionData} />
            ) : null}
            <Button asChild>
              <Link to="edit">Edit</Link>
            </Button>
          </div>
        </div>
      ) : null}
    </section>
  )
}
```

### 5.3 编辑/创建操作

#### (1) 编辑操作 - 基于所有权的查询

文件：`app/routes/users/$username/notes/$noteId_.edit.tsx`

```typescript
export async function loader({ params, request }: Route.LoaderArgs) {
  const userId = await requireUserId(request)
  
  // 直接在查询条件中加入 ownerId，实现所有权检查
  const note = await prisma.note.findFirst({
    select: { id: true, title: true, content: true, images: { ... } },
    where: {
      id: params.noteId,
      ownerId: userId,  // 关键：只允许编辑自己的笔记
    },
  })
  invariantResponse(note, 'Not found', { status: 404 })
  return { note }
}
```

**说明**：编辑操作采用了更直接的方式 - 在查询条件中直接加入 `ownerId: userId`。如果用户不是所有者，查询会返回 `null`，从而触发 404 错误。

#### (2) 创建操作 - 隐式的权限检查

文件：`app/routes/users/$username/notes/+shared/note-editor.server.tsx`

```typescript
export async function action({ request }: ActionFunctionArgs) {
  const userId = await requireUserId(request)  // 先要求登录

  const formData = await parseFormData(request, { maxFileSize: MAX_UPLOAD_SIZE })

  const submission = await parseWithZod(formData, {
    schema: NoteEditorSchema.superRefine(async (data, ctx) => {
      if (!data.id) return  // 新建笔记不检查

      // 更新时检查：必须是自己的笔记
      const note = await prisma.note.findUnique({
        select: { id: true },
        where: { id: data.id, ownerId: userId },
      })
      if (!note) {
        ctx.addIssue({
          code: z.ZodIssueCode.custom,
          message: 'Note not found',
        })
      }
    }).transform(async ({ images = [], ...data }) => {
      // ... 图片处理
    }),
    async: true,
  })

  // ... 验证通过后执行 upsert
  const updatedNote = await prisma.note.upsert({
    select: { id: true, owner: { select: { username: true } } },
    where: { id: noteId },
    create: {
      id: noteId,
      ownerId: userId,  // 新建时自动设置 ownerId
      title,
      content,
      images: { create: newImages },
    },
    update: {
      title,
      content,
      images: { /* ... */ },
    },
  })

  return redirect(...)
}
```

**创建操作的权限逻辑**：
1. 只要求用户已登录（`requireUserId`）
2. 新建笔记时自动设置 `ownerId: userId`，确保用户成为所有者
3. 更新时在 `superRefine` 中检查 `ownerId: userId`

### 5.4 读取操作 - 公开访问设计

从代码分析来看，笔记的读取操作（`$noteId.tsx` 的 loader）没有显式的权限检查：

```typescript
export async function loader({ params }: Route.LoaderArgs) {
  const note = await prisma.note.findUnique({
    where: { id: params.noteId },
    select: { id: true, title: true, content: true, ownerId: true, ... },
  })
  invariantResponse(note, 'Not found', { status: 404 })

  const date = new Date(note.updatedAt)
  const timeAgo = formatDistanceToNow(date)

  return { note, timeAgo }
}
```

**设计意图**：
- 笔记采用公开可读的设计（类似于博客系统）
- 任何人都可以通过链接访问笔记（只要知道 ID）
- 但编辑和删除受严格的权限控制

### 5.5 新建按钮的显示控制

在 `_layout.tsx` 中，新建按钮只对所有者显示：

```typescript
export default function NotesRoute({ loaderData }: Route.ComponentProps) {
  const user = useOptionalUser()
  const isOwner = user?.id === loaderData.owner.id
  // ...

  return (
    <main>
      <div>
        <ul>
          {/* 只有所有者才能看到 "New Note" 按钮 */}
          {isOwner ? (
            <li>
              <NavLink to="new">
                <Icon name="plus">New Note</Icon>
              </NavLink>
            </li>
          ) : null}
          {/* ... 笔记列表 ... */}
        </ul>
      </div>
    </main>
  )
}
```

---

## 六、前后端权限一致性保证

### 6.1 一致性机制

Epic-Stack 通过以下机制确保前后端权限一致：

| 层级 | 机制 | 实现位置 |
|-----|------|---------|
| **数据来源一致** | 权限数据都来自数据库的同一套 Role/Permission 表 | `prisma/schema.prisma` |
| **权限字符串一致** | 前后端使用相同的 `action:entity:access` 格式 | `app/utils/user.ts` |
| **解析逻辑一致** | 前后端使用相同的 `parsePermissionString` 函数 | `app/utils/user.ts` |
| **检查逻辑一致** | `userHasPermission` 与 `requireUserWithPermission` 的检查条件相同 | 对比见下文 |

### 6.2 前后端检查逻辑对比

**前端** `userHasPermission`：
```typescript
return user.roles.some((role) =>
  role.permissions.some(
    (permission) =>
      permission.entity === entity &&
      permission.action === action &&
      (!access || access.includes(permission.access)),
  ),
)
```

**服务端** `requireUserWithPermission` 的 Prisma 查询条件：
```typescript
where: {
  id: userId,
  roles: {
    some: {
      permissions: {
        some: {
          ...permissionData,  // action, entity 直接匹配
          access: permissionData.access
            ? { in: permissionData.access }  // access 使用 in 查询
            : undefined,
        },
      },
    },
  },
}
```

**逻辑等价性**：
- `roles.some` → Prisma 的 `roles: { some: { ... } }`
- `permissions.some` → Prisma 的 `permissions: { some: { ... } }`
- `entity === entity` → 直接匹配
- `action === action` → 直接匹配
- `access.includes(permission.access)` → Prisma 的 `{ in: accessArray }`

### 6.3 双重防护设计

Epic-Stack 采用了 **前端显示控制 + 服务端强制校验** 的双重防护：

```
┌─────────────────────────────────────────────────────────────┐
│                        用户操作                                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  第一层：前端控制                                              │
│  • userHasPermission() 检查权限                              │
│  • 控制按钮/链接的显示/隐藏                                    │
│  • 改善用户体验，避免无效请求                                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ (用户通过某种方式绕过前端)
┌─────────────────────────────────────────────────────────────┐
│  第二层：服务端强制校验                                        │
│  • requireUserWithPermission() 强制检查                      │
│  • 不通过则抛出 403 错误                                      │
│  • 这是真正的安全防线                                          │
└─────────────────────────────────────────────────────────────┘
```

**关键原则**：前端控制只是为了用户体验，服务端才是真正的安全边界。

---

## 七、权限初始化与种子数据

### 7.1 角色和权限的初始化

从 `prisma/seed.ts` 可以看到用户角色的分配：

```typescript
// 普通用户 - 连接到 'user' 角色
const user = await prisma.user.create({
  select: { id: true },
  data: {
    ...userData,
    password: { create: createPassword(userData.username) },
    roles: { connect: { name: 'user' } },  // 只有 user 角色
  },
})

// 管理员用户 - 同时连接到 'admin' 和 'user' 角色
const kody = await prisma.user.create({
  select: { id: true },
  data: {
    email: 'kody@kcd.dev',
    username: 'kody',
    // ...
    roles: { connect: [{ name: 'admin' }, { name: 'user' }] },  // 双角色
  },
})
```

### 7.2 权限配置（推断）

虽然 seed.ts 没有直接创建权限数据，但从决策文档 `docs/decisions/028-permissions-rbac.md` 和代码逻辑可以推断出标准的权限配置：

**User 角色权限**：
- `create:note:own` - 创建自己的笔记
- `read:note:own` - 读取自己的笔记
- `update:note:own` - 更新自己的笔记
- `delete:note:own` - 删除自己的笔记

**Admin 角色权限**：
- `create:note:any` - 创建任意笔记
- `read:note:any` - 读取任意笔记
- `update:note:any` - 更新任意笔记
- `delete:note:any` - 删除任意笔记

---

## 八、总结与架构图

### 8.1 核心设计模式总结

| 设计要点 | 实现方式 | 代码位置 |
|---------|---------|---------|
| **权限模型** | RBAC + Entity-Action-Access | `prisma/schema.prisma` |
| **权限字符串** | `action:entity:access` 格式 | `app/utils/user.ts` |
| **所有权检查** | 先判断 `isOwner = ownerId === userId` | 各路由文件 |
| **权限选择** | 根据 isOwner 选择 `:own` 或 `:any` 权限 | 各路由文件 |
| **服务端检查** | `requireUserWithPermission()` 强制校验 | `app/utils/permissions.server.ts` |
| **前端检查** | `userHasPermission()` 控制显示 | `app/utils/user.ts` |
| **权限下发** | Root Loader 查询 roles.permissions | `app/root.tsx` |

### 8.2 权限数据流图

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Database   │────▶│  Root Loader │────▶│  React Router│
│(Role/Permission)    │ (root.tsx)   │     │ useRouteLoaderData│
└──────────────┘     └──────────────┘     └──────────────┘
        ▲                    │                      │
        │                    ▼                      ▼
        │           ┌──────────────┐     ┌──────────────┐
        │           │  服务端权限   │     │  前端权限     │
        │           │  检查函数    │     │  检查函数    │
        │           │requireUser   │     │userHas       │
        │           │WithPermission│     │Permission    │
        │           └──────────────┘     └──────────────┘
        │                    │                      │
        └────────────────────┴──────────────────────┘
                              │
                              ▼
                    ┌──────────────┐
                    │  笔记操作     │
                    │  读写删       │
                    └──────────────┘
```

### 8.3 权限检查决策流程

```
                    用户发起操作
                         │
                         ▼
              ┌──────────────────────┐
              │  前端：userHasPermission │
              │  检查是否有权限       │
              └──────────────────────┘
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
         有权限                  无权限
              │                     │
              ▼                     ▼
        显示操作按钮             隐藏操作按钮
        (如删除按钮)              (改善用户体验)
              │
              ▼
        用户点击按钮
              │
              ▼
    ┌─────────────────────┐
    │ 服务端：requireUser  │
    │   WithPermission    │
    │   强制权限检查       │
    └─────────────────────┘
              │
    ┌─────────┴─────────┐
    │                   │
    ▼                   ▼
  有权限              无权限
    │                   │
    ▼                   ▼
执行操作            抛出 403 错误
(如删除笔记)        (真正的安全防线)
```

### 8.4 关键设计亮点

1. **前后端逻辑复用**：权限字符串、解析函数、检查逻辑在前后端保持一致
2. **双重防护**：前端控制体验，服务端保障安全
3. **灵活的权限模型**：`own` vs `any` 设计区分了普通用户和管理员
4. **所有权感知**：先判断所有权再选择权限字符串的模式简洁有效
5. **全局权限数据**：通过 Root Loader 下发，所有页面都能访问
6. **类型安全**：TypeScript 类型定义确保权限字符串的正确性

---

## 九、相关文件索引

| 文件路径 | 说明 |
|---------|------|
| `prisma/schema.prisma` | 数据模型定义（User, Role, Permission, Note） |
| `app/root.tsx` | 根路由，权限数据下发的核心 |
| `app/utils/user.ts` | 前端权限检查函数（userHasPermission, parsePermissionString） |
| `app/utils/permissions.server.ts` | 服务端权限检查函数（requireUserWithPermission） |
| `app/routes/users/$username/notes/$noteId.tsx` | 笔记详情页（删除操作的权限示例） |
| `app/routes/users/$username/notes/$noteId_.edit.tsx` | 笔记编辑页 |
| `app/routes/users/$username/notes/+shared/note-editor.server.tsx` | 笔记创建/更新的服务端逻辑 |
| `prisma/seed.ts` | 数据库种子数据（角色分配示例） |
| `docs/decisions/028-permissions-rbac.md` | RBAC 设计决策文档 |
