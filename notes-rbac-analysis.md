# Epic-Stack 笔记 RBAC 权限管控机制分析报告

## 一、权限与角色初始化的真实来源

### 1.1 初始化位置：数据库迁移文件

权限和角色的初始化**不在 `prisma/seed.ts`**，而是在 **`prisma/migrations/20250221233640_init/migration.sql`** 中通过 SQL INSERT 语句完成。

迁移文件中的注释明确说明了这一点：
```sql
-- The user Roles and Permissions are seeded here.
-- If you'd like to customise roles and permissions, you can edit and add the code below to your `prisma/seed.ts` file.
```

### 1.2 完整的权限数据

迁移文件中初始化了 **16 条 Permission 记录**（2 个实体 × 4 种操作 × 2 种访问范围）：

| id | action | entity | access | 说明 |
|----|--------|--------|--------|------|
| clnf2zvli0000pcou3zzzzome | create | user | own | 创建自己的用户 |
| clnf2zvll0001pcouly1310ku | create | user | any | 创建任意用户 |
| clnf2zvll0002pcouka7348re | read | user | own | 读取自己的用户信息 |
| clnf2zvlm0003pcouea4dee51 | read | user | any | 读取任意用户信息 |
| clnf2zvlm0004pcou2guvolx5 | update | user | own | 更新自己的用户信息 |
| clnf2zvln0005pcoun78ps5ap | update | user | any | 更新任意用户信息 |
| clnf2zvlo0006pcouyoptc5jp | delete | user | own | 删除自己的用户 |
| clnf2zvlo0007pcouw1yzoyam | delete | user | any | 删除任意用户 |
| **clnf2zvlp0008pcou9r0fhbm8** | **create** | **note** | **own** | **创建自己的笔记** |
| **clnf2zvlp0009pcouj3qib9q9** | **create** | **note** | **any** | **创建任意笔记** |
| **clnf2zvlq000apcouxnspejs9** | **read** | **note** | **own** | **读取自己的笔记** |
| **clnf2zvlr000bpcouf4cg3x72** | **read** | **note** | **any** | **读取任意笔记** |
| **clnf2zvlr000cpcouy1vp6oeg** | **update** | **note** | **own** | **更新自己的笔记** |
| **clnf2zvls000dpcouvzwjjzrq** | **update** | **note** | **any** | **更新任意笔记** |
| **clnf2zvls000epcou4ts5ui8f** | **delete** | **note** | **own** | **删除自己的笔记** |
| **clnf2zvlt000fpcouk29jbmxn** | **delete** | **note** | **any** | **删除任意笔记** |

### 1.3 角色数据

初始化了 **2 条 Role 记录**：

| id | name | 说明 |
|----|------|------|
| clnf2zvlw000gpcour6dyyuh6 | admin | 管理员角色 |
| clnf2zvlx000hpcou5dfrbegs | user | 普通用户角色 |

### 1.4 角色与权限的关联

通过 `_PermissionToRole` 关联表建立关系：

**Admin 角色**：关联所有 `access = 'any'` 的权限（共 8 条）
- `create:user:any`
- `read:user:any`
- `update:user:any`
- `delete:user:any`
- `create:note:any`
- `read:note:any`
- `update:note:any`
- `delete:note:any`

**User 角色**：关联所有 `access = 'own'` 的权限（共 8 条）
- `create:user:own`
- `read:user:own`
- `update:user:own`
- `delete:user:own`
- `create:note:own`
- `read:note:own`
- `update:note:own`
- `delete:note:own`

### 1.5 用户与角色的关联（seed.ts）

在 `prisma/seed.ts` 中，用户创建时连接角色：

```typescript
// 普通用户 - 只连接 user 角色
const user = await prisma.user.create({
  data: {
    ...userData,
    roles: { connect: { name: 'user' } },
  },
})

// 管理员用户 - 同时连接 admin 和 user 角色
const kody = await prisma.user.create({
  data: {
    email: 'kody@kcd.dev',
    username: 'kody',
    // ...
    roles: { connect: [{ name: 'admin' }, { name: 'user' }] },
  },
})
```

---

## 二、根路由权限数据下发到前端的完整链路

### 2.1 链路总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           数据库层                                          │
│  ┌──────────┐      ┌──────────────────┐      ┌──────────┐               │
│  │   User   │─────▶│   _RoleToUser    │─────▶│   Role   │               │
│  └──────────┘      └──────────────────┘      └──────────┘               │
│                                                  │                         │
│                                                  ▼                         │
│                                         ┌──────────────────┐              │
│                                         │ _PermissionToRole│              │
│                                         └──────────────────┘              │
│                                                  │                         │
│                                                  ▼                         │
│                                         ┌──────────────┐                  │
│                                         │  Permission  │                  │
│                                         │action:entity:│                  │
│                                         │    access    │                  │
│                                         └──────────────┘                  │
└─────────────────────────────────────────────────────────────────────────┘
                                                  │
                                                  ▼ 服务端查询
┌─────────────────────────────────────────────────────────────────────────┐
│                        Root Loader (app/root.tsx)                         │
│                                                                             │
│  prisma.user.findUnique({                                                 │
│    select: {                                                               │
│      id: true,                                                             │
│      name: true,                                                           │
│      username: true,                                                       │
│      roles: {                                                              │
│        select: {                                                           │
│          name: true,                                                       │
│          permissions: {                                                    │
│            select: { entity: true, action: true, access: true },        │
│          }                                                                 │
│        }                                                                   │
│      }                                                                     │
│    }                                                                       │
│  })                                                                        │
│                                                                             │
│  return data({ user, ... })  ← 包含完整的 roles.permissions 数据          │
└─────────────────────────────────────────────────────────────────────────┘
                                                  │
                                                  ▼ React Router 传递
┌─────────────────────────────────────────────────────────────────────────┐
│                      前端取数层 (app/utils/user.ts)                        │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  useOptionalUser()                                                 │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │ const data = useRouteLoaderData<typeof rootLoader>('root') │  │   │
│  │  │ return data?.user  ← 从根路由 loader 获取权限数据            │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  userHasPermission(user, permissionString)                        │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │ 解析 permissionString → { action, entity, access }         │  │   │
│  │  │ 遍历 user.roles，检查是否有角色包含所需权限                   │  │   │
│  │  │ return user.roles.some(role =>                              │  │   │
│  │  │   role.permissions.some(p =>                                │  │   │
│  │  │     p.entity === entity &&                                  │  │   │
│  │  │     p.action === action &&                                  │  │   │
│  │  │     (!access || access.includes(p.access))                  │  │   │
│  │  │   )                                                          │  │   │
│  │  │ )                                                            │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                                  │
                                                  ▼ 组件使用
┌─────────────────────────────────────────────────────────────────────────┐
│                           React 组件层                                      │
│                                                                             │
│  const user = useOptionalUser()                                            │
│  const canDelete = userHasPermission(                                      │
│    user,                                                                    │
│    isOwner ? 'delete:note:own' : 'delete:note:any'                        │
│  )                                                                          │
│                                                                             │
│  {canDelete ? <DeleteButton /> : null}  ← 根据权限控制 UI 显示             │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 服务端：Root Loader 权限查询

**文件**: `app/root.tsx`

```typescript
export async function loader({ request }: Route.LoaderArgs) {
  const timings = makeTimings('root loader')
  const userId = await time(() => getUserId(request), {
    timings,
    type: 'getUserId',
    desc: 'getUserId in root',
  })

  // 关键：查询用户时同时获取 roles 和 permissions
  const user = userId
    ? await time(
        () =>
          prisma.user.findUnique({
            select: {
              id: true,
              name: true,
              username: true,
              image: { select: { objectKey: true } },
              // 嵌套查询：角色及其包含的权限
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

  // 权限数据通过 loader 返回给前端
  return data(
    {
      user,  // user.roles[].permissions 包含完整权限数据
      requestInfo: { /* ... */ },
      ENV: getEnv(),
      toast,
      honeyProps,
    },
    // ...
  )
}
```

**返回的用户数据结构**：
```typescript
user: {
  id: 'clnf...',
  name: 'Kody',
  username: 'kody',
  image: { objectKey: 'user/kody.png' } | null,
  roles: [
    {
      name: 'admin',
      permissions: [
        { entity: 'user', action: 'create', access: 'any' },
        { entity: 'user', action: 'read', access: 'any' },
        { entity: 'user', action: 'update', access: 'any' },
        { entity: 'user', action: 'delete', access: 'any' },
        { entity: 'note', action: 'create', access: 'any' },
        { entity: 'note', action: 'read', access: 'any' },
        { entity: 'note', action: 'update', access: 'any' },
        { entity: 'note', action: 'delete', access: 'any' },
      ]
    },
    {
      name: 'user',
      permissions: [
        { entity: 'user', action: 'create', access: 'own' },
        { entity: 'user', action: 'read', access: 'own' },
        { entity: 'user', action: 'update', access: 'own' },
        { entity: 'user', action: 'delete', access: 'own' },
        { entity: 'note', action: 'create', access: 'own' },
        { entity: 'note', action: 'read', access: 'own' },
        { entity: 'note', action: 'update', access: 'own' },
        { entity: 'note', action: 'delete', access: 'own' },
      ]
    }
  ]
}
```

### 2.3 前端：取数层封装

**文件**: `app/utils/user.ts`

#### (1) 权限字符串类型定义

```typescript
type Action = 'create' | 'read' | 'update' | 'delete'
type Entity = 'user' | 'note'
type Access = 'own' | 'any' | 'own,any' | 'any,own'

export type PermissionString =
  | `${Action}:${Entity}`           // 例如: 'delete:note'
  | `${Action}:${Entity}:${Access}` // 例如: 'delete:note:own'
```

#### (2) 权限字符串解析

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

#### (3) 从 Root Loader 获取用户数据

```typescript
export function useOptionalUser() {
  // 使用 React Router 的 useRouteLoaderData 获取根路由数据
  // 'root' 是根路由的 id
  const data = useRouteLoaderData<typeof rootLoader>('root')
  if (!data || !isUser(data.user)) {
    return undefined
  }
  return data.user
}

export function useUser() {
  const maybeUser = useOptionalUser()
  if (!maybeUser) {
    throw new Error(
      'No user found in root loader, but user is required by useUser. ' +
      'If user is optional, try useOptionalUser instead.',
    )
  }
  return maybeUser
}
```

#### (4) 前端权限检查函数

```typescript
export function userHasPermission(
  user: Pick<ReturnType<typeof useUser>, 'roles'> | null | undefined,
  permission: PermissionString,
) {
  if (!user) return false
  
  // 解析权限字符串
  const { action, entity, access } = parsePermissionString(permission)
  
  // 遍历用户的所有角色，检查是否有一个角色包含所需权限
  return user.roles.some((role) =>
    role.permissions.some(
      (permission) =>
        // 实体匹配
        permission.entity === entity &&
        // 操作匹配
        permission.action === action &&
        // 访问范围匹配（无 access 要求 或 access 包含权限的 access）
        (!access || access.includes(permission.access)),
    ),
  )
}
```

### 2.4 组件使用示例

```typescript
// 在组件中使用
export default function NoteRoute({ loaderData }: Route.ComponentProps) {
  const user = useOptionalUser()
  const isOwner = user?.id === loaderData.note.ownerId
  
  // 使用权限检查
  const canDelete = userHasPermission(
    user,
    isOwner ? `delete:note:own` : `delete:note:any`,
  )

  return (
    <div>
      {/* 根据权限控制 UI 显示 */}
      {canDelete ? <DeleteButton /> : null}
    </div>
  )
}
```

---

## 三、三种笔记操作的鉴权路径分析

### 3.1 鉴权模式总览

| 操作 | 前端 UI 控制 | 服务端 Loader | 服务端 Action | 是否使用权限系统 | 一致性 |
|------|-------------|---------------|---------------|-----------------|--------|
| **创建** | `isOwner` 判断 | `requireUserId` | `requireUserId` | ❌ 否 | ⚠️ 部分一致 |
| **编辑** | `canDelete \|\| isOwner` | 直接检查 `ownerId` | 直接检查 `ownerId` | ❌ 否 | ❌ **不一致** |
| **删除** | `userHasPermission()` | 无独立 loader | `requireUserWithPermission()` | ✅ 是 | ✅ 一致 |

### 3.2 删除操作：完整且一致的鉴权

删除操作是**唯一完全使用 RBAC 权限系统**的操作，前后端逻辑一致。

#### (1) 前端 UI 控制

**文件**: `app/routes/users/$username/notes/$noteId.tsx:92-102`

```typescript
export default function NoteRoute({ loaderData, actionData }: Route.ComponentProps) {
  const user = useOptionalUser()
  const isOwner = user?.id === loaderData.note.ownerId
  
  // 关键：使用 userHasPermission 检查权限
  const canDelete = userHasPermission(
    user,
    isOwner ? `delete:note:own` : `delete:note:any`,
  )
  const displayBar = canDelete || isOwner

  return (
    <section>
      {displayBar ? (
        <div className={floatingToolbarClassName}>
          <div className="grid flex-1 grid-cols-2 justify-end gap-2">
            {/* 只有 canDelete 为 true 时才显示删除按钮 */}
            {canDelete ? (
              <DeleteNote id={loaderData.note.id} actionData={actionData} />
            ) : null}
            {/* 编辑按钮没有独立的权限检查 */}
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

**删除按钮显示逻辑**：
- 所有者 (`isOwner = true`)：需要 `delete:note:own` 权限（user 角色默认拥有）
- 非所有者：需要 `delete:note:any` 权限（admin 角色拥有）

#### (2) 服务端 Action 强制检查

**文件**: `app/routes/users/$username/notes/$noteId.tsx:56-90`

```typescript
export async function action({ request }: Route.ActionArgs) {
  const userId = await requireUserId(request)
  const formData = await request.formData()
  const submission = parseWithZod(formData, { schema: DeleteFormSchema })
  
  // ... 表单验证

  const { noteId } = submission.value

  // 1. 查询笔记获取 ownerId
  const note = await prisma.note.findFirst({
    select: { id: true, ownerId: true, owner: { select: { username: true } } },
    where: { id: noteId },
  })
  invariantResponse(note, 'Not found', { status: 404 })

  // 2. 判断是否为所有者
  const isOwner = note.ownerId === userId
  
  // 3. 使用权限系统检查（与前端逻辑一致）
  await requireUserWithPermission(
    request,
    isOwner ? `delete:note:own` : `delete:note:any`,
  )

  // 4. 执行删除
  await prisma.note.delete({ where: { id: note.id } })

  return redirectWithToast(...)
}
```

#### (3) 服务端权限检查函数

**文件**: `app/utils/permissions.server.ts`

```typescript
export async function requireUserWithPermission(
  request: Request,
  permission: PermissionString,
) {
  const userId = await requireUserId(request)
  const permissionData = parsePermissionString(permission)
  
  // 使用 Prisma 嵌套查询检查权限
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
                ? { in: permissionData.access }  // 对应前端的 access.includes()
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

#### (4) 删除操作鉴权流程图

```
                    用户点击删除按钮
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  前端：userHasPermission(user,                                 │
│         isOwner ? 'delete:note:own' : 'delete:note:any')    │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ 1. 解析权限字符串 → { action: 'delete', entity: 'note',  │ │
│  │                      access: ['own'] 或 ['any'] }        │ │
│  │ 2. 遍历 user.roles                                         │ │
│  │ 3. 检查是否有角色的 permissions 包含：                     │ │
│  │    p.entity === 'note' &&                                 │ │
│  │    p.action === 'delete' &&                               │ │
│  │    access.includes(p.access)                               │ │
│  └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
        canDelete = true         canDelete = false
              │                         │
              ▼                         ▼
        显示删除按钮                 隐藏删除按钮
              │
              ▼
        用户提交表单
              │
              ▼
┌──────────────────────────────────────────────────────────────┐
│  服务端：requireUserWithPermission(request,                   │
│         isOwner ? 'delete:note:own' : 'delete:note:any')    │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ 1. requireUserId(request) 确保已登录                      │ │
│  │ 2. parsePermissionString(permission) 解析权限字符串        │ │
│  │ 3. Prisma 查询：                                           │ │
│  │    prisma.user.findFirst({                                 │ │
│  │      where: {                                              │ │
│  │        id: userId,                                         │ │
│  │        roles: {                                            │ │
│  │          some: {                                           │ │
│  │            permissions: {                                  │ │
│  │              some: {                                       │ │
│  │                action: 'delete',                           │ │
│  │                entity: 'note',                             │ │
│  │                access: { in: ['own'] } 或 ['any']         │ │
│  │              }                                             │ │
│  │            }                                               │ │
│  │          }                                                 │ │
│  │        }                                                   │ │
│  │      }                                                     │ │
│  │    })                                                      │ │
│  └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
        有权限 → 执行删除            无权限 → 抛出 403
```

### 3.3 编辑操作：不一致的鉴权（存在问题）

编辑操作是**唯一存在前后端不一致**的操作。

#### (1) 前端入口显示逻辑

**文件**: `app/routes/users/$username/notes/$noteId.tsx:102-167`

```typescript
export default function NoteRoute({ loaderData, actionData }: Route.ComponentProps) {
  const user = useOptionalUser()
  const isOwner = user?.id === loaderData.note.ownerId
  
  const canDelete = userHasPermission(
    user,
    isOwner ? `delete:note:own` : `delete:note:any`,
  )
  
  // ⚠️ 关键问题：displayBar 的条件是 canDelete || isOwner
  const displayBar = canDelete || isOwner

  return (
    <section>
      {displayBar ? (
        <div className={floatingToolbarClassName}>
          <div className="grid flex-1 grid-cols-2 justify-end gap-2">
            {/* 删除按钮有 canDelete 检查 */}
            {canDelete ? <DeleteNote ... /> : null}
            
            {/* ⚠️ 编辑按钮没有独立的权限检查！ */}
            {/* 只要 displayBar 为 true 就显示 */}
            <Button asChild>
              <Link to="edit">
                <Icon name="pencil-1">Edit</Icon>
              </Link>
            </Button>
          </div>
        </div>
      ) : null}
    </section>
  )
}
```

**编辑按钮显示条件分析**：

`displayBar = canDelete || isOwner`

这意味着以下用户能看到编辑按钮：

| 用户类型 | isOwner | canDelete | displayBar | 能看到编辑按钮？ |
|---------|---------|-----------|------------|-----------------|
| 笔记所有者 | true | true（有 `delete:note:own`） | true | ✅ 能 |
| 管理员（非所有者） | false | true（有 `delete:note:any`） | true | ✅ **能看到！** |
| 普通用户（非所有者） | false | false | false | ❌ 不能 |

**问题**：管理员虽然不是所有者，但因为有 `delete:note:any` 权限（所以 `canDelete = true`），也能看到编辑按钮！

#### (2) 服务端 Loader 限制

**文件**: `app/routes/users/$username/notes/$noteId_.edit.tsx:10-32`

```typescript
export async function loader({ params, request }: Route.LoaderArgs) {
  const userId = await requireUserId(request)
  
  // ⚠️ 关键：直接使用 ownerId 限制，不使用权限系统！
  const note = await prisma.note.findFirst({
    select: {
      id: true,
      title: true,
      content: true,
      images: { select: { id: true, altText: true, objectKey: true } },
    },
    where: {
      id: params.noteId,
      ownerId: userId,  // 只允许所有者访问！
    },
  })
  
  // 如果不是所有者，note 为 null，抛出 404
  invariantResponse(note, 'Not found', { status: 404 })
  return { note }
}
```

#### (3) 服务端 Action 限制

**文件**: `app/routes/users/$username/notes/+shared/note-editor.server.tsx:34-48`

```typescript
export async function action({ request }: ActionFunctionArgs) {
  const userId = await requireUserId(request)
  const formData = await parseFormData(request, { maxFileSize: MAX_UPLOAD_SIZE })

  const submission = await parseWithZod(formData, {
    schema: NoteEditorSchema.superRefine(async (data, ctx) => {
      if (!data.id) return  // 新建笔记不检查

      // ⚠️ 更新时直接检查 ownerId，不使用权限系统！
      const note = await prisma.note.findUnique({
        select: { id: true },
        where: { id: data.id, ownerId: userId },  // 只允许所有者！
      })
      if (!note) {
        ctx.addIssue({
          code: z.ZodIssueCode.custom,
          message: 'Note not found',
        })
      }
    }).transform(...),
    async: true,
  })

  // ... 执行 upsert
}
```

#### (4) 编辑操作的不一致问题

| 层面 | 逻辑 | 结果 |
|-----|------|------|
| **前端入口** | `displayBar = canDelete \|\| isOwner` | 管理员能看到编辑按钮 |
| **服务端 Loader** | `where: { id: noteId, ownerId: userId }` | 只有所有者能访问编辑页 |
| **服务端 Action** | `where: { id: data.id, ownerId: userId }` | 只有所有者能提交更新 |

**实际场景演示**：

1. **用户 A** 是笔记所有者，角色：`user`
2. **用户 B** 是管理员，角色：`admin`（不是笔记所有者）

**用户 B 的体验**：
- 访问 `/users/userA/notes/note-1`
- 看到 **Edit 按钮**（因为 `canDelete = true`，有 `delete:note:any`）
- 点击 Edit 按钮，跳转到 `/users/userA/notes/note-1/edit`
- 服务端 Loader 查询：`where: { id: 'note-1', ownerId: 'userB-id' }`
- 查询结果为 `null`
- 看到 **404 错误页面**："No note with the id "note-1" exists"

**权限系统的设计意图**：

根据迁移文件中的权限配置：
- `admin` 角色拥有 `update:note:any` 权限
- 这意味着**设计上**管理员应该能够编辑任意笔记

**实际实现**：

编辑操作完全没有使用权限系统，直接检查 `ownerId`。结果：
- 管理员虽然有 `update:note:any` 权限，但无法使用
- 前端入口显示编辑按钮（误导）
- 服务端直接返回 404

#### (5) 编辑操作鉴权流程图

```
                    管理员访问他人笔记
                           │
                           ▼
              ┌────────────────────────┐
              │  前端：displayBar =     │
              │  canDelete || isOwner   │
              │  ┌────────────────────┐ │
              │  │ isOwner = false    │ │
              │  │ canDelete = true   │ │
              │  │   (有 delete:note: │ │
              │  │    any 权限)        │ │
              │  └────────────────────┘ │
              │  displayBar = true      │
              └────────────────────────┘
                           │
                           ▼
                    显示 Edit 按钮
                           │
                           ▼
                    管理员点击 Edit
                           │
                           ▼
              ┌────────────────────────┐
              │  服务端 Loader：        │
              │  prisma.note.findFirst({│
              │    where: {             │
              │      id: noteId,        │
              │      ownerId: userId    │
              │      // ⚠️ 不检查权限！  │
              │    }                     │
              │  })                      │
              │                         │
              │  结果：note = null       │
              └────────────────────────┘
                           │
                           ▼
                    抛出 404 错误
              "No note with the id ... exists"
```

### 3.4 创建操作：基于所有权隐式控制

创建操作**不使用权限系统**，而是基于页面上下文和 `ownerId` 隐式控制。

#### (1) 前端入口显示

**文件**: `app/routes/users/$username/notes/_layout.tsx:54-66`

```typescript
export default function NotesRoute({ loaderData }: Route.ComponentProps) {
  const user = useOptionalUser()
  const isOwner = user?.id === loaderData.owner.id

  return (
    <main>
      <div>
        <ul>
          {/* ⚠️ 只有 isOwner 为 true 时才显示 New Note 按钮 */}
          {isOwner ? (
            <li>
              <NavLink to="new">
                <Icon name="plus">New Note</Icon>
              </NavLink>
            </li>
          ) : null}
          {/* 笔记列表... */}
        </ul>
      </div>
    </main>
  )
}
```

**新建按钮显示逻辑**：
- 只有当前登录用户是当前页面的所有者时，才显示 "New Note" 按钮
- 这意味着：
  - 用户 A 访问 `/users/userA/notes`：能看到新建按钮
  - 用户 A 访问 `/users/userB/notes`：看不到新建按钮
  - 管理员访问 `/users/userA/notes`：看不到新建按钮

#### (2) 服务端 Loader

**文件**: `app/routes/users/$username/notes/new.tsx:7-10`

```typescript
export async function loader({ request }: Route.LoaderArgs) {
  await requireUserId(request)  // 只检查是否登录
  return {}
}
```

**问题**：任何人只要知道 URL `/users/xxx/notes/new`，都可以直接访问！

例如：
- 用户 B 直接访问 `/users/userA/notes/new`
- 服务端 loader 只检查 `requireUserId`
- 用户 B 能看到新建笔记表单

#### (3) 服务端 Action

**文件**: `app/routes/users/$username/notes/+shared/note-editor.server.tsx:27-130`

```typescript
export async function action({ request }: ActionFunctionArgs) {
  const userId = await requireUserId(request)  // 只检查是否登录

  const formData = await parseFormData(request, { maxFileSize: MAX_UPLOAD_SIZE })

  const submission = await parseWithZod(formData, {
    schema: NoteEditorSchema.superRefine(async (data, ctx) => {
      if (!data.id) return  // 新建时不检查 ownerId

      // 更新时检查 ownerId（前面已分析）
      const note = await prisma.note.findUnique({
        select: { id: true },
        where: { id: data.id, ownerId: userId },
      })
      // ...
    }).transform(...),
    async: true,
  })

  // ...

  const updatedNote = await prisma.note.upsert({
    select: { id: true, owner: { select: { username: true } } },
    where: { id: noteId },
    create: {
      id: noteId,
      ownerId: userId,  // ⚠️ 新建时自动设置为当前用户
      title,
      content,
      images: { create: newImages },
    },
    update: { /* ... */ },
  })

  return redirect(
    `/users/${updatedNote.owner.username}/notes/${updatedNote.id}`,
  )
}
```

**创建操作的实际行为**：

1. 用户 B 直接访问 `/users/userA/notes/new`
2. 能看到新建笔记表单
3. 填写内容并提交
4. 服务端创建笔记，`ownerId` 自动设置为 **用户 B 的 ID**（不是 userA）
5. 重定向到 `/users/userB/notes/new-note-id`

**结果**：
- 虽然用户 B 在 `/users/userA/notes/new` 页面创建笔记
- 但笔记的实际所有者是 **用户 B**
- 这是通过 `ownerId: userId` 隐式保证的

#### (4) 创建操作的权限系统对比

| 层面 | 实际实现 | 权限系统设计意图 |
|-----|---------|-----------------|
| `user` 角色 | 能创建笔记（通过 `ownerId: userId`） | 有 `create:note:own` 权限 |
| `admin` 角色 | 只能创建自己的笔记 | 有 `create:note:any` 权限（但无法使用） |

**问题**：
- `create:note:any` 权限在设计上存在
- 但实现中没有使用
- 管理员无法为他人创建笔记

#### (5) 创建操作鉴权流程图

```
              场景：用户 B 访问 /users/userA/notes/new
                           │
                           ▼
              ┌────────────────────────┐
              │  前端：_layout.tsx      │
              │  isOwner = (userB.id   │
              │           === userA.id) │
              │  isOwner = false        │
              │  不显示 New Note 按钮    │
              └────────────────────────┘
                           │
                           ▼
         用户 B 直接在地址栏输入 URL（绕过 UI）
                           │
                           ▼
              ┌────────────────────────┐
              │  服务端 Loader：        │
              │  await requireUserId()  │
              │  只检查是否登录          │
              │  用户 B 已登录 ✓        │
              └────────────────────────┘
                           │
                           ▼
                    显示新建笔记表单
                           │
                           ▼
                    用户 B 提交表单
                           │
                           ▼
              ┌────────────────────────┐
              │  服务端 Action：        │
              │  prisma.note.upsert({  │
              │    create: {            │
              │      ownerId: userId    │
              │      // = userB 的 ID   │
              │      // ⚠️ 不是 userA！ │
              │    }                    │
              │  })                      │
              └────────────────────────┘
                           │
                           ▼
              重定向到 /users/userB/notes/xxx
                           │
                           ▼
              笔记所有者实际上是用户 B
```

---

## 四、三种操作鉴权对比总结

### 4.1 鉴权方式对比表

| 维度 | 创建操作 | 编辑操作 | 删除操作 |
|-----|---------|---------|---------|
| **前端 UI 控制** | `isOwner` 判断 | `canDelete \|\| isOwner` | `userHasPermission()` |
| **服务端 Loader** | `requireUserId` | 直接检查 `ownerId` | 无独立 loader |
| **服务端 Action** | 隐式 `ownerId: userId` | 直接检查 `ownerId` | `requireUserWithPermission()` |
| **使用 RBAC 权限系统** | ❌ 否 | ❌ 否 | ✅ 是 |
| **前后端一致性** | ⚠️ 部分一致（UI 隐藏但 URL 可直接访问） | ❌ **不一致**（入口显示但服务端 404） | ✅ 一致 |

### 4.2 权限字符串使用情况

| 权限字符串 | 是否被实际使用 | 使用位置 |
|-----------|---------------|---------|
| `create:note:own` | ❌ 否 | 未使用 |
| `create:note:any` | ❌ 否 | 未使用 |
| `read:note:own` | ❌ 否 | 笔记公开可读 |
| `read:note:any` | ❌ 否 | 笔记公开可读 |
| `update:note:own` | ❌ 否 | 直接检查 `ownerId` |
| `update:note:any` | ❌ 否 | 直接检查 `ownerId` |
| `delete:note:own` | ✅ 是 | 删除操作 |
| `delete:note:any` | ✅ 是 | 删除操作 |

**结论**：16 个权限中，只有 2 个被实际使用！

### 4.3 不一致问题的根源

#### (1) 编辑操作的不一致

**问题表现**：
- 管理员能看到编辑按钮（因为有 `delete:note:any`）
- 但点击后看到 404

**根本原因**：
1. 编辑入口的显示逻辑 `displayBar = canDelete || isOwner` 有问题
2. 编辑操作的服务端完全没有使用权限系统
3. `update:note:any` 权限虽然存在，但从未被检查

#### (2) 创建操作的隐蔽问题

**问题表现**：
- 非所有者看不到新建按钮
- 但可以通过直接访问 URL 绕过

**根本原因**：
1. 新建按钮只检查 `isOwner`
2. 但服务端只检查 `requireUserId`
3. 最终通过 `ownerId: userId` 隐式保证了笔记属于创建者（这是对的）
4. 但 `create:note:any` 权限无法使用

### 4.4 设计意图 vs 实际实现

根据迁移文件中的权限设计，意图是：

| 角色 | 权限设计意图 | 实际能力 |
|-----|-------------|---------|
| **user** | `*:note:own` - 管理自己的笔记 | ✅ 能创建、编辑、删除自己的笔记 |
| **admin** | `*:note:any` - 管理任意笔记 | ⚠️ 只能删除任意笔记，不能编辑/创建 |

**实际效果**：
- 删除操作：完整实现了 RBAC
- 编辑操作：完全没有使用 RBAC，只有所有者能编辑
- 创建操作：完全没有使用 RBAC，只能创建自己的笔记

---

## 五、相关文件索引

| 文件路径 | 说明 |
|---------|------|
| `prisma/migrations/20250221233640_init/migration.sql` | 权限和角色的真实初始化位置（SQL INSERT） |
| `prisma/schema.prisma` | 数据模型定义 |
| `prisma/seed.ts` | 用户与角色的关联（connect role） |
| `app/root.tsx` | 根路由，权限数据下发的核心 |
| `app/utils/user.ts` | 前端权限检查函数（userHasPermission, parsePermissionString） |
| `app/utils/permissions.server.ts` | 服务端权限检查函数（requireUserWithPermission） |
| `app/routes/users/$username/notes/_layout.tsx` | 笔记布局，新建按钮的显示逻辑 |
| `app/routes/users/$username/notes/new.tsx` | 新建笔记页面 |
| `app/routes/users/$username/notes/$noteId.tsx` | 笔记详情页（删除操作、编辑入口） |
| `app/routes/users/$username/notes/$noteId_.edit.tsx` | 笔记编辑页 |
| `app/routes/users/$username/notes/+shared/note-editor.server.tsx` | 笔记创建/更新的服务端逻辑 |
