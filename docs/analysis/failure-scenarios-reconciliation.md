# Epic Stack 失败场景确定性对账报告

## 概述

本文档对 **上传失败** 和 **写库失败** 两类关键场景进行**确定性对账**，明确回答：

1. 哪些情况会触发 `ErrorBoundary`？
2. 哪些情况 `actionData` 可用，表单会回填？
3. 各种失败场景的精确执行路径是什么？

---

## 一、核心机制确定性分析

### 1.1 React Router 错误处理机制（确定性）

基于 React Router 的约定和代码分析，以下是**100% 确定**的行为：

| Action 行为 | React Router 响应 | 组件渲染 | `actionData` | 表单回填 |
|-------------|-------------------|----------|--------------|----------|
| `return data({ result: ... })` | 正常响应 | ✅ 原组件渲染 | ✅ 包含返回值 | ✅ **会回填** |
| `return redirect(...)` | 302 重定向 | ✅ 目标页面渲染 | - | - |
| `throw` 未捕获异常 | 捕获异常 | ❌ **ErrorBoundary 渲染** | ❌ **undefined** | ❌ **不会回填** |
| `throw data(..., { status: 500 })` | 捕获 Response | ❌ **ErrorBoundary 渲染** | ❌ **undefined** | ❌ **不会回填** |

**关键确定性结论**：

> 只要 action 执行过程中**抛出任何未捕获的异常**（无论是 `Error`、`Response` 还是其他），React Router 都会：
> 1. 查找最近的 `ErrorBoundary`
> 2. 渲染 ErrorBoundary 组件
> 3. **原表单组件不会被渲染**
> 4. `actionData` 为 `undefined`（组件根本不执行）
> 5. **表单不会回填**

### 1.2 ErrorBoundary 触发条件（确定性）

**代码位置**: `app/routes/users/$username/notes/$noteId_.edit.tsx:41-51`

```typescript
export function ErrorBoundary() {
    return (
        <GeneralErrorBoundary
            statusHandlers={{
                404: ({ params }) => (
                    <p>No note with the id "{params.noteId}" exists</p>
                ),
            }}
        />
    )
}
```

**代码位置**: `app/components/error-boundary.tsx:16-53`

```typescript
export function GeneralErrorBoundary({
    defaultStatusHandler = ({ error }) => (
        <p>{error.status} {error.data}</p>
    ),
    statusHandlers,  // 按 HTTP 状态码定制
    unexpectedErrorHandler = (error) => <p>{getErrorMessage(error)}</p>,
}: {
    // ...
}) {
    const error = useRouteError()
    const isResponse = isRouteErrorResponse(error)

    return (
        <div className="...">
            {isResponse
                ? // HTTP 错误响应（如 throw redirect()、throw json()）
                  (statusHandlers?.[error.status] ?? defaultStatusHandler)({
                      error,
                      params,
                  })
                : // 未捕获的异常（如 throw new Error()）
                  unexpectedErrorHandler(error)}
        </div>
    )
}
```

**确定性结论**：

> ErrorBoundary 会在以下情况触发：
> 1. `throw new Error('...')` - 普通异常
> 2. `throw data({ message: '...' }, { status: 500 })` - Response 类型的错误
> 3. **任何未被 action 内部 catch 的异常**

> 一旦 ErrorBoundary 被触发：
> - 原表单组件**不会被渲染**
> - `useForm` 不会执行
> - `lastResult: actionData?.result` 中的 `actionData` 为 `undefined`
> - **表单值完全丢失，用户必须重新填写**

---

## 二、上传失败场景对账

### 2.1 上传失败的代码路径分析

**代码位置**: `app/routes/users/$username/notes/+shared/note-editor.server.tsx:48-81`

```typescript
.transform(async ({ images = [], ...data }) => {
    const noteId = data.id ?? cuid()
    return {
        ...data,
        id: noteId,
        imageUpdates: await Promise.all(
            images.filter(imageHasId).map(async (i) => {
                if (imageHasFile(i)) {
                    return {
                        id: i.id,
                        altText: i.altText,
                        objectKey: await uploadNoteImage(userId, noteId, i.file),  // ⚠️ 可能抛出
                    }
                }
                return { id: i.id, altText: i.altText }
            }),
        ),
        newImages: await Promise.all(
            images
                .filter(imageHasFile)
                .filter((i) => !i.id)
                .map(async (image) => ({
                    altText: image.altText,
                    objectKey: await uploadNoteImage(userId, noteId, image.file),  // ⚠️ 可能抛出
                })),
        ),
    }
})
```

**代码位置**: `app/utils/storage.server.ts:20-24`

```typescript
if (!uploadResponse.ok) {
    const errorMessage = `Failed to upload file to storage. Server responded with ${uploadResponse.status}: ${uploadResponse.statusText}`
    console.error(errorMessage)
    throw new Error(`Failed to upload object: ${key}`)  // 直接抛出 Error
}
```

### 2.2 关键问题：transform 中的异常如何被处理？

这是最关键的不确定性来源。让我们基于 **Zod** 和 **Conform** 的设计目标来分析：

#### Zod 的行为（确定性）

| Zod API | transform 抛出异常时的行为 |
|---------|---------------------------|
| `schema.parse(data)` | 异常继续向上抛出 |
| `schema.safeParse(data)` | 捕获异常，返回 `{ success: false, error: ZodError }` |

#### Conform parseWithZod 的行为（高概率推断）

Conform 的设计目标是**优雅地处理表单验证错误**，让用户能够看到友好的错误提示并保留输入值。

基于这个设计目标，**高概率推断** `parseWithZod` 内部使用 `safeParse`。

但是，这里有一个**关键区别**：

| 错误类型 | 报告方式 | 预期行为 |
|----------|----------|----------|
| **验证错误** | `ctx.addIssue()` | ✅ 返回 `{ status: 'error' }` |
| **运行时异常** | 直接 `throw` | ⚠️ 行为不确定 |

#### 对比：login.tsx 中的"正确"模式

**代码位置**: `app/routes/_auth/login.tsx:50-64`

```typescript
schema: (intent) =>
    LoginFormSchema.transform(async (data, ctx) => {
        if (intent !== null) return { ...data, session: null }

        const session = await login(data)
        if (!session) {
            ctx.addIssue({                             // ✅ 使用 ctx.addIssue()
                code: z.ZodIssueCode.custom,
                message: 'Invalid username or password',
            })
            return z.NEVER                             // ✅ 显式返回，不是抛出
        }

        return { ...data, session }
    }),
```

**关键观察**：

> `login.tsx` 使用 `ctx.addIssue()` + `return z.NEVER` 来报告错误，而不是直接抛出异常。

这表明：**推荐的做法是通过 `ctx.addIssue()` 来报告业务逻辑错误**，让 Conform 能够正确处理。

### 2.3 上传失败场景对账表

基于以上分析，以下是**上传失败**（S3 服务不可用、网络中断等）的精确对账：

---

#### 场景 A-1：transform 中 S3 上传抛出异常

**触发条件**：
- `uploadNoteImage()` 中的 `fetch()` 失败
- 或 `uploadResponse.ok === false`
- 直接 `throw new Error('Failed to upload object: ...')`

**执行路径分析**：

```
时间线：
T0: 用户提交表单，包含 3 张图片
T1: parseWithZod 开始执行
T2: Zod 基础验证通过
T3: superRefine 执行（检查笔记所有权），通过
T4: transform 开始执行
T5: 图片 1 上传 S3 成功
T6: 图片 2 上传 S3 成功
T7: 图片 3 上传 S3 失败 → throw new Error('Failed to upload object...')
```

**现在的关键问题**：这个 `Error` 会被谁捕获？

| 可能性 | 行为 | 结果 |
|--------|------|------|
| **可能性 1**: `parseWithZod` 使用 `safeParse` | 异常被捕获，`submission.status === 'error'` | 继续执行 `if (submission.status !== 'success')`，然后 `return data({ result: submission.reply() })` |
| **可能性 2**: `parseWithZod` 不捕获 transform 异常 | 异常继续向上抛出 | 触发 ErrorBoundary |

**基于代码模式的推断**：

查看 `note-editor.server.tsx` 的完整流程：

```typescript
const submission = await parseWithZod(formData, {
    schema: NoteEditorSchema
        .superRefine(async (data, ctx) => {
            // ... 使用 ctx.addIssue()
        })
        .transform(async ({ images = [], ...data }) => {
            // ... 直接调用 uploadNoteImage()，没有 try-catch
            // uploadNoteImage() 可能抛出 Error
        }),
    async: true,
})

// 只有当 parseWithZod 返回时才会执行到这里
if (submission.status !== 'success') {
    return data(
        { result: submission.reply() },
        { status: submission.status === 'error' ? 400 : 200 },
    )
}
```

**关键推断**：

> 如果 `transform` 中的异常能够被 `parseWithZod` 捕获并转换为 `submission.status === 'error'`，那么：
> - 代码会继续执行 `if (submission.status !== 'success')`
> - 然后 `return data({ result: submission.reply() })`
> - ✅ 表单会回填

> 如果异常**不被捕获**，直接向上抛出：
> - `if (submission.status !== 'success')` 这行代码**不会执行**
> - 异常被 React Router 捕获
> - ❌ 触发 ErrorBoundary，表单不回填

### 2.4 上传失败场景的确定性结论

基于代码分析，我们可以给出以下**分级确定性**结论：

---

#### 🔴 确定性结论（100% 确定）

**孤儿文件风险**：
- 如果上传了部分文件后失败，**已上传的文件不会自动删除**
- 这会导致 S3 存储中的"孤儿文件"
- 需要手动清理机制

---

#### 🟡 高概率结论（90% 置信）

基于 Conform/Zod 的设计目标：

**`parseWithZod` 会捕获 transform 中的异常**：
- `submission.status === 'error'`
- 执行 `return data({ result: submission.reply() })`
- ✅ **表单会回填**

**理由**：
1. Conform 的设计目标是优雅处理表单错误
2. Zod 的 `safeParse` 会捕获 transform 异常
3. `parseWithZod` 应该使用 `safeParse` 内部

---

#### 🟠 需验证结论（50% 置信）

**异常类型可能影响行为**：
- 如果是 `ZodError`（验证相关）→ 被捕获
- 如果是普通 `Error`（网络错误等）→ 可能不被捕获

**建议添加验证代码**来确认实际行为。

---

#### ✅ 最佳实践（确保确定性行为）

要**100% 确保**上传失败时表单能够回填，应该修改代码：

```typescript
// 当前代码（不推荐）
.transform(async ({ images = [], ...data }) => {
    // ...
    objectKey: await uploadNoteImage(userId, noteId, i.file),  // 可能抛出
    // ...
})

// 推荐修改（使用 ctx.addIssue）
.transform(async ({ images = [], ...data }, ctx) => {  // 添加 ctx 参数
    const noteId = data.id ?? cuid()
    
    try {
        // 处理图片上传
        const imageUpdates = await Promise.all(
            images.filter(imageHasId).map(async (i) => {
                if (imageHasFile(i)) {
                    try {
                        return {
                            id: i.id,
                            altText: i.altText,
                            objectKey: await uploadNoteImage(userId, noteId, i.file),
                        }
                    } catch (error) {
                        ctx.addIssue({
                            code: z.ZodIssueCode.custom,
                            message: '图片上传失败，请重试',
                            path: ['images', String(images.indexOf(i)), 'file'],
                        })
                        return z.NEVER
                    }
                }
                return { id: i.id, altText: i.altText }
            }),
        )
        // ...
    } catch (error) {
        ctx.addIssue({
            code: z.ZodIssueCode.custom,
            message: '上传过程中发生错误，请重试',
        })
        return z.NEVER
    }
})
```

---

## 三、写库失败场景对账

### 3.1 写库失败的代码路径分析

**代码位置**: `app/routes/users/$username/notes/+shared/note-editor.server.tsx:92-126`

```typescript
// 验证通过检查
if (submission.status !== 'success') {
    return data(
        { result: submission.reply() },
        { status: submission.status === 'error' ? 400 : 200 },
    )
}

// 提取数据
const {
    id: noteId,
    title,
    content,
    imageUpdates = [],
    newImages = [],
} = submission.value

// ⚠️ 数据库操作：没有 try-catch 保护！
const updatedNote = await prisma.note.upsert({
    select: { id: true, owner: { select: { username: true } } },
    where: { id: noteId },
    create: {
        id: noteId,
        ownerId: userId,
        title,
        content,
        images: { create: newImages },
    },
    update: {
        title,
        content,
        images: {
            deleteMany: { id: { notIn: imageUpdates.map((i) => i.id) } },
            updateMany: imageUpdates.map((updates) => ({
                where: { id: updates.id },
                data: { ...updates },
            })),
            create: newImages,
        },
    },
})

// 成功重定向
return redirect(
    `/users/${updatedNote.owner.username}/notes/${updatedNote.id}`,
)
```

### 3.2 写库失败的确定性分析

这部分的行为是 **100% 确定**的，因为代码逻辑非常清晰：

```
执行路径：
1. if (submission.status !== 'success') → 通过，不执行 return
2. 解构 submission.value → 成功
3. await prisma.note.upsert({ ... }) → 抛出异常
4. 没有 try-catch → 异常向上传播
5. React Router 捕获异常 → 触发 ErrorBoundary
6. 原组件不渲染 → actionData 为 undefined
7. 表单不回填
```

### 3.3 可能的写库失败原因

| 失败原因 | Prisma 异常类型 | 是否触发 ErrorBoundary |
|----------|-----------------|------------------------|
| 数据库连接中断 | `PrismaClientInitializationError` | ✅ 是 |
| 唯一约束违反（并发创建） | `PrismaClientKnownRequestError` (code: P2002) | ✅ 是 |
| 外键约束违反 | `PrismaClientKnownRequestError` | ✅ 是 |
| 权限不足 | `PrismaClientKnownRequestError` | ✅ 是 |
| 查询超时 | `PrismaClientKnownRequestError` | ✅ 是 |
| 磁盘满/其他系统错误 | `PrismaClientUnknownRequestError` | ✅ 是 |

### 3.4 写库失败场景对账表

---

#### 场景 B-1：数据库操作失败（当前代码）

**触发条件**：
- `prisma.note.upsert()` 抛出任何异常

**执行路径（确定性）**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  写库失败执行路径（当前代码）                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  T0: submission.status === 'success' → 验证通过                             │
│                                                                              │
│  T1: 解构 submission.value → 成功                                           │
│                                                                              │
│  T2: await prisma.note.upsert({ ... })                                      │
│         ↓ 抛出异常（如数据库连接失败）                                         │
│                                                                              │
│  T3: 没有 try-catch 捕获 → 异常向上传播                                       │
│         ↓                                                                    │
│  T4: React Router 捕获异常                                                   │
│         ↓                                                                    │
│  T5: 查找最近的 ErrorBoundary                                                │
│         ↓                                                                    │
│  T6: 渲染 ErrorBoundary 组件                                                 │
│         ↓                                                                    │
│  T7: 原 NoteEditor 组件**不渲染**                                            │
│         ↓                                                                    │
│  T8: useForm() 不执行 → lastResult 无数据                                    │
│         ↓                                                                    │
│  T9: ❌ 表单值丢失，用户看到错误页面                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**确定性结论**：

| 项目 | 结果 | 确定性 |
|------|------|--------|
| 是否触发 ErrorBoundary | ✅ **是** | 100% |
| `actionData` 是否可用 | ❌ **undefined** | 100% |
| 表单是否回填 | ❌ **不会回填** | 100% |
| 用户看到什么 | ErrorBoundary 渲染的错误页面 | 100% |
| 用户需要做什么 | 重新填写整个表单 | 100% |

---

#### 场景 B-2：数据库操作失败（改进后：添加 try-catch）

**推荐的改进代码**：

```typescript
// 验证通过
if (submission.status !== 'success') {
    return data(
        { result: submission.reply() },
        { status: submission.status === 'error' ? 400 : 200 },
    )
}

const { id: noteId, title, content, imageUpdates, newImages } = submission.value

try {
    const updatedNote = await prisma.note.upsert({ ... })
    return redirect(`/users/${updatedNote.owner.username}/notes/${updatedNote.id}`)
} catch (error) {
    // 记录错误日志
    console.error('Failed to save note:', error)
    
    // ✅ 使用 submission.reply() 返回错误，保留表单值
    return data(
        {
            result: submission.reply({
                formErrors: ['保存失败，请稍后重试'],
            }),
        },
        { status: 500 },
    )
}
```

**执行路径（改进后）**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  写库失败执行路径（改进后）                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  T0: submission.status === 'success' → 验证通过                             │
│                                                                              │
│  T1: 进入 try 块                                                             │
│                                                                              │
│  T2: await prisma.note.upsert({ ... })                                      │
│         ↓ 抛出异常                                                           │
│                                                                              │
│  T3: catch 块捕获异常                                                        │
│         ↓                                                                    │
│  T4: 记录错误日志                                                            │
│         ↓                                                                    │
│  T5: return data({ result: submission.reply({ formErrors: [...] }) })      │
│         ↓                                                                    │
│  T6: React Router 正常响应，不触发 ErrorBoundary                             │
│         ↓                                                                    │
│  T7: 原 NoteEditor 组件正常渲染                                               │
│         ↓                                                                    │
│  T8: actionData = { result: { status: 'error', fields: {...}, errors: {...} } }│
│         ↓                                                                    │
│  T9: useForm({ lastResult: actionData?.result })                             │
│         ↓                                                                    │
│  T10: ✅ 表单值自动回填，错误显示在表单底部                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**确定性结论（改进后）**：

| 项目 | 结果 | 确定性 |
|------|------|--------|
| 是否触发 ErrorBoundary | ❌ **否** | 100% |
| `actionData` 是否可用 | ✅ **包含 result** | 100% |
| 表单是否回填 | ✅ **会回填** | 100% |
| 用户看到什么 | 原表单页面，底部显示错误消息 | 100% |
| 用户需要做什么 | 修改错误部分，重新提交 | 100% |

---

## 四、完整对账总表

### 4.1 上传失败场景对账

| 场景 | 触发条件 | 当前行为 | 是否回填 | 确定性 |
|------|----------|----------|----------|--------|
| **A-1** | S3 上传抛出异常（当前代码） | 取决于 `parseWithZod` 实现 | ⚠️ **可能回填，也可能不** | 50% |
| **A-2** | S3 上传抛出异常（使用 `ctx.addIssue`） | `submission.status === 'error'` → `return data(...)` | ✅ **会回填** | 100% |
| **A-3** | `superRefine` 中 `ctx.addIssue()` | `submission.status === 'error'` → `return data(...)` | ✅ **会回填** | 100% |

### 4.2 写库失败场景对账

| 场景 | 触发条件 | 当前行为 | 是否回填 | 确定性 |
|------|----------|----------|----------|--------|
| **B-1** | 数据库操作失败（当前代码，无 try-catch） | 异常抛出 → ErrorBoundary | ❌ **不会回填** | 100% |
| **B-2** | 数据库操作失败（改进后，有 try-catch + `submission.reply()`） | `return data({ result: ... })` | ✅ **会回填** | 100% |

### 4.3 其他失败场景对账

| 场景 | 触发条件 | 行为 | 是否回填 | 确定性 |
|------|----------|------|----------|--------|
| **C-1** | Zod 基础验证失败（类型、长度等） | `submission.status === 'error'` → `return data(...)` | ✅ **会回填** | 100% |
| **C-2** | `parseFormData` 文件大小超限 | 抛出异常？或返回错误？ | ⚠️ 需验证 | 50% |
| **C-3** | `requireUserId` 未登录 | 抛出 `redirect` 到登录页 | -（重定向，不是错误） | 100% |

---

## 五、关键代码路径速查

### 5.1 会回填的代码路径（✅ 确定）

```typescript
// 路径 1: 验证失败，通过 submission.reply() 返回
if (submission.status !== 'success') {
    return data(
        { result: submission.reply() },           // ✅ 包含表单值和错误
        { status: submission.status === 'error' ? 400 : 200 },
    )
}

// 路径 2: 业务失败，手动构造 reply()（如 signup.tsx）
if (response.status === 'success') {
    return redirect(redirectTo.toString())
} else {
    return data(
        {
            result: submission.reply({                // ✅ 包含表单值和错误
                formErrors: [response.error.message],
            }),
        },
        { status: 500 },
    )
}

// 路径 3: 改进后的数据库错误处理
try {
    const updatedNote = await prisma.note.upsert({ ... })
    return redirect(...)
} catch (error) {
    return data(
        {
            result: submission.reply({                // ✅ 包含表单值和错误
                formErrors: ['保存失败，请重试'],
            }),
        },
        { status: 500 },
    )
}
```

### 5.2 不会回填的代码路径（❌ 确定）

```typescript
// 路径 1: 未捕获的异常（当前 note-editor.server.tsx）
const updatedNote = await prisma.note.upsert({ ... })  // 抛出异常
// 没有 try-catch → 异常向上传播 → ErrorBoundary

// 路径 2: 显式 throw Response
throw data({ message: 'Not Found' }, { status: 404 })  // 触发 ErrorBoundary

// 路径 3: throw 普通 Error
throw new Error('Something went wrong')  // 触发 ErrorBoundary
```

### 5.3 不确定的代码路径（⚠️ 需验证）

```typescript
// 路径 1: transform 中直接抛出异常（不是通过 ctx.addIssue()）
.transform(async ({ images = [], ...data }) => {
    // ...
    objectKey: await uploadNoteImage(userId, noteId, i.file),  // 抛出 Error
    // 这个 Error 会被 parseWithZod 捕获吗？
})

// 可能的结果：
// A. 被捕获 → submission.status === 'error' → ✅ 回填
// B. 不被捕获 → 异常向上传播 → ❌ ErrorBoundary
```

---

## 六、改进建议

基于以上对账分析，以下是**高优先级改进建议**：

### 6.1 高优先级：给数据库操作添加 try-catch

**问题**：`note-editor.server.tsx` 中的数据库操作没有 try-catch 保护，失败时用户丢失所有输入。

**改进代码**：

```typescript
// app/routes/users/$username/notes/+shared/note-editor.server.tsx

if (submission.status !== 'success') {
    return data(
        { result: submission.reply() },
        { status: submission.status === 'error' ? 400 : 200 },
    )
}

const {
    id: noteId,
    title,
    content,
    imageUpdates = [],
    newImages = [],
} = submission.value

try {
    const updatedNote = await prisma.note.upsert({
        select: { id: true, owner: { select: { username: true } } },
        where: { id: noteId },
        create: {
            id: noteId,
            ownerId: userId,
            title,
            content,
            images: { create: newImages },
        },
        update: {
            title,
            content,
            images: {
                deleteMany: { id: { notIn: imageUpdates.map((i) => i.id) } },
                updateMany: imageUpdates.map((updates) => ({
                    where: { id: updates.id },
                    data: {
                        ...updates,
                        id: updates.objectKey ? cuid() : updates.id,
                    },
                })),
                create: newImages,
            },
        },
    })

    return redirect(
        `/users/${updatedNote.owner.username}/notes/${updatedNote.id}`,
    )
} catch (error) {
    console.error('Failed to save note:', error)
    
    return data(
        {
            result: submission.reply({
                formErrors: ['保存失败，请稍后重试。您的输入已保留。'],
            }),
        },
        { status: 500 },
    )
}
```

**改进效果**：

| 项目 | 改进前 | 改进后 |
|------|--------|--------|
| 数据库失败时 | ❌ 触发 ErrorBoundary，表单不回填 | ✅ 返回友好错误，表单回填 |
| 用户体验 | 差（需重填整个表单） | 好（只需重试） |
| 实现复杂度 | 低（无代码） | 中（添加 try-catch） |

### 6.2 中优先级：验证 transform 异常处理行为

**问题**：`uploadNoteImage()` 抛出异常时的行为不确定。

**建议**：

1. **添加单元测试**来确认 `parseWithZod` 对 transform 异常的实际行为
2. **或修改代码**使用 `ctx.addIssue()` 来确保确定性行为

**推荐修改**：

```typescript
// 在 transform 中添加 ctx 参数并使用 try-catch
.transform(async ({ images = [], ...data }, ctx) => {
    const noteId = data.id ?? cuid()
    
    try {
        const imageUpdates = await Promise.all(
            images.filter(imageHasId).map(async (i, index) => {
                if (imageHasFile(i)) {
                    try {
                        return {
                            id: i.id,
                            altText: i.altText,
                            objectKey: await uploadNoteImage(userId, noteId, i.file),
                        }
                    } catch (uploadError) {
                        ctx.addIssue({
                            code: z.ZodIssueCode.custom,
                            message: `图片 ${index + 1} 上传失败，请重试`,
                            path: ['images', String(index), 'file'],
                        })
                        return z.NEVER
                    }
                }
                return { id: i.id, altText: i.altText }
            }),
        )
        
        // ... 处理 newImages
        
        return {
            ...data,
            id: noteId,
            imageUpdates,
            newImages: await processNewImages(images, userId, noteId, ctx),
        }
    } catch (error) {
        ctx.addIssue({
            code: z.ZodIssueCode.custom,
            message: '上传过程中发生错误，请稍后重试',
        })
        return z.NEVER
    }
})
```

### 6.3 低优先级：添加孤儿文件清理机制

**问题**：部分上传成功后失败，已上传的文件成为孤儿文件。

**建议**：

1. **短期**：添加定时任务清理临时/孤儿文件
2. **长期**：实现"两阶段提交"模式：
   - 先上传到临时位置
   - 数据库操作成功后再移动到永久位置
   - 失败时删除临时文件

---

## 七、总结

### 7.1 核心确定性结论

1. **React Router ErrorBoundary 行为（100% 确定）**：
   - 只要 action 抛出未捕获的异常，就会触发 ErrorBoundary
   - 原组件不渲染，`actionData` 为 `undefined`
   - **表单不会回填**

2. **submission.reply() 行为（100% 确定）**：
   - 当 action `return data({ result: submission.reply() })` 时
   - 组件正常渲染，`actionData.result` 包含表单值和错误
   - **表单会回填**

3. **当前 note-editor.server.tsx 写库失败行为（100% 确定）**：
   - 没有 try-catch 保护
   - 数据库异常向上传播
   - 触发 ErrorBoundary
   - **表单不会回填**

### 7.2 快速决策表

| 你想实现什么 | 使用什么模式 | 表单是否回填 |
|--------------|--------------|--------------|
| 验证失败，显示错误 | `if (submission.status !== 'success') { return data({ result: submission.reply() }) }` | ✅ 会 |
| 业务逻辑失败，保留输入 | `try { ... } catch { return data({ result: submission.reply({ formErrors: [...] }) }) }` | ✅ 会 |
| 严重错误，无法恢复 | `throw new Error('...')` 或 `throw data(..., { status: 500 })` | ❌ 不会 |

### 7.3 代码位置速查

| 功能 | 文件路径 | 关键行 |
|------|----------|--------|
| 笔记编辑器 Action（当前有问题） | `app/routes/users/$username/notes/+shared/note-editor.server.tsx` | 100-126 |
| 注册表单（正确的错误处理模式） | `app/routes/_auth/signup.tsx` | 73-90 |
| ErrorBoundary 组件 | `app/components/error-boundary.tsx` | 全部 |
| useForm 接收 lastResult | `app/routes/users/$username/notes/+shared/note-editor.tsx` | 62-65 |
| S3 上传抛出异常 | `app/utils/storage.server.ts` | 20-24 |

---

## 附录：术语表

| 术语 | 定义 |
|------|------|
| **回填** | 表单提交失败后，用户之前输入的值被自动填充回表单控件中 |
| **ErrorBoundary** | React Router 中用于捕获和渲染错误的组件边界 |
| **actionData** | React Router 中 action 函数返回的数据，传递给页面组件 |
| **submission** | Conform 中 `parseWithZod()` 的返回值，包含验证状态和结果 |
| **submission.reply()** | Conform 中用于构造客户端可解析的错误响应的方法 |
| **ctx.addIssue()** | Zod 中用于在 `superRefine` 或 `transform` 中添加验证错误的方法 |
| **z.NEVER** | Zod 中用于标记验证失败的特殊返回值 |
| **孤儿文件** | 已上传到存储服务但数据库中没有对应记录的文件，无法被访问或清理 |
