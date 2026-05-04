# 用户头像处理与数据导出分析

## 一、用户头像上传、替换、删除流程

### 1.1 Multipart 解析

**核心文件**: `app/routes/settings/profile/photo.tsx`

使用 `@mjackson/form-data-parser` 库进行 multipart/form-data 解析：

```typescript
// 第69行
const formData = await parseFormData(request, { maxFileSize: MAX_SIZE })
```

- **MAX_SIZE**: 限制为 3MB (`1024 * 1024 * 3`)
- **表单验证**: 使用 Zod schema (`PhotoFormSchema`) 进行 discriminated union 验证
  - `intent: 'submit'`: 上传/替换头像
  - `intent: 'delete'`: 删除头像

### 1.2 对象存储（S3/MinIO 兼容）

**核心文件**: `app/utils/storage.server.ts`

#### 上传流程 (`uploadProfileImage`)

```typescript
// 第29-38行
export async function uploadProfileImage(
	userId: string,
	file: File | FileUpload,
) {
	const fileId = createId() // 生成唯一ID
	const fileExtension = file.name.split('.').pop() || ''
	const timestamp = Date.now()
	const key = `users/${userId}/profile-images/${timestamp}-${fileId}.${fileExtension}`
	return uploadToStorage(file, key)
}
```

**对象键命名规则**: `users/{userId}/profile-images/{timestamp}-{fileId}.{extension}`

#### AWS Signature V4 签名机制

存储服务实现了完整的 AWS S3 兼容签名：

- **签名算法**: AWS4-HMAC-SHA256
- **关键函数**:
  - `hmacSha256()`: HMAC-SHA256 哈希
  - `sha256()`: SHA256 哈希
  - `getSignatureKey()`: 生成签名密钥
  - `getBaseSignedRequestInfo()`: 构建基础签名信息
  - `getSignedPutRequestInfo()`: 生成 PUT 请求签名

**请求头包含**:
- `X-Amz-Date`: 日期时间
- `X-Amz-Content-SHA256`: UNSIGNED-PAYLOAD
- `Authorization`: 完整签名凭证

#### 上传执行 (`uploadToStorage`)

```typescript
// 第11-27行
async function uploadToStorage(file: File | FileUpload, key: string) {
	const { url, headers } = getSignedPutRequestInfo(file, key)
	const uploadResponse = await fetch(url, {
		method: 'PUT',
		headers,
		body: file instanceof File ? file : (file as FileUpload).stream(),
	})
	// ... 错误处理
	return key
}
```

### 1.3 数据库事务

**核心文件**: `app/routes/settings/profile/photo.tsx` 第98-104行

#### 上传/替换头像的事务处理

```typescript
await prisma.$transaction(async ($prisma) => {
	// 1. 删除用户所有旧头像记录
	await $prisma.userImage.deleteMany({ where: { userId } })
	// 2. 创建新头像记录并关联到用户
	await $prisma.user.update({
		where: { id: userId },
		data: { image: { create: image } },
	})
})
```

**事务特性**:
- **原子性**: 删除旧头像和创建新头像在同一事务中
- **先删后创**: 确保用户始终只有一个头像
- **Prisma Client**: 使用 `$prisma` 事务客户端

#### 删除头像的处理

```typescript
// 第93-95行
if (intent === 'delete') {
	await prisma.userImage.deleteMany({ where: { userId } })
	return redirect('/settings/profile')
}
```

### 1.4 完整流程图

```
上传/替换流程:
1. 前端表单提交 (multipart/form-data)
   ↓
2. parseFormData() 解析 (maxFileSize: 3MB)
   ↓
3. Zod schema 验证 (PhotoFormSchema)
   ↓
4. uploadProfileImage() 上传到 S3/MinIO
   ├── 生成唯一 objectKey
   ├── AWS Signature V4 签名
   └── fetch PUT 上传
   ↓
5. Prisma 事务:
   ├── deleteMany(旧头像)
   └── create(新头像) + update(用户关联)
   ↓
6. 重定向到个人资料页

删除流程:
1. 前端 intent=delete 提交
   ↓
2. parseFormData() 解析
   ↓
3. Zod 验证通过
   ↓
4. prisma.userImage.deleteMany({ userId })
   ↓
5. 重定向
```

### 1.5 对象存储与数据库一致性风险分析

#### 场景一：上传成功但事务失败

**当前代码顺序**（`photo.tsx:66-106`）：
```
1. uploadProfileImage() 上传到 S3/MinIO → 成功，文件已存在
2. Prisma 事务 → 失败（网络问题、数据库宕机、约束冲突等）
```

**会发生什么？**

| 影响项 | 具体后果 |
|-------|---------|
| **存储泄漏** | S3/MinIO 中存在孤立文件，无法通过数据库追踪 |
| **用户体验** | 用户看到上传失败，但存储空间被占用 |
| **成本累积** | 大量失败上传导致存储费用增加 |
| **不可恢复** | 没有 objectKey 记录，无法定位和清理这些文件 |

**对象键命名特点加剧问题**：
```
users/{userId}/profile-images/{timestamp}-{fileId}.{extension}
```
- `timestamp` + `fileId` 完全随机
- 无法通过用户 ID 反向推导失败上传的文件
- 即使扫描 `users/{userId}/profile-images/` 目录，也无法区分"有效但旧的文件"和"失败上传的文件"

---

#### 场景二：删除头像只删数据库不删对象

**当前代码**（`photo.tsx:93-106`）：

**删除操作**：
```typescript
if (intent === 'delete') {
  await prisma.userImage.deleteMany({ where: { userId } })
  // ❌ 没有调用 S3 DeleteObject
  return redirect('/settings/profile')
}
```

**替换操作（事务中）**：
```typescript
await prisma.$transaction(async ($prisma) => {
  await $prisma.userImage.deleteMany({ where: { userId } })
  // ❌ 事务中只删数据库，没有删旧的 S3 对象
  await $prisma.user.update({ data: { image: { create: image } } })
})
```

**会发生什么影响？**

| 影响项 | 具体后果 |
|-------|---------|
| **存储泄漏** | 每次替换头像，旧文件遗留在 S3/MinIO 中 |
| **成本累积** | 用户频繁更换头像 → 存储空间线性增长 |
| **隐私风险** | 旧头像文件仍可访问（如果知道 URL），用户以为已删除 |
| **数据残留** | 不符合 GDPR/CCPA 等"被遗忘权"要求 |

**量化示例**：
- 用户每月换 1 次头像，每次 2MB
- 1 年后：12 个孤立文件 = 24MB 浪费
- 10 万用户：2.4TB 存储浪费

---

#### 可执行的补救策略

##### 策略 A：补偿式删除（推荐，改动最小）

**上传失败场景**：在事务失败时尝试删除已上传的文件

```typescript
// 修改后的 action 逻辑
export async function action({ request }: Route.ActionArgs) {
  const userId = await requireUserId(request)
  const formData = await parseFormData(request, { maxFileSize: MAX_SIZE })
  
  let uploadedObjectKey: string | null = null
  
  try {
    const submission = await parseWithZod(formData, {
      schema: PhotoFormSchema.transform(async (data) => {
        if (data.intent === 'delete') return { intent: 'delete' }
        uploadedObjectKey = await uploadProfileImage(userId, data.photoFile)
        return {
          intent: data.intent,
          image: { objectKey: uploadedObjectKey },
        }
      }),
      async: true,
    })
    
    // ... 验证和事务处理
  } catch (error) {
    // 补偿：事务失败时尝试删除已上传的文件
    if (uploadedObjectKey) {
      try {
        await deleteFromStorage(uploadedObjectKey)
      } catch (deleteError) {
        console.error('Failed to clean up uploaded file:', deleteError)
        // 记录日志，后续由定时任务兜底
      }
    }
    throw error
  }
}
```

**删除/替换场景**：先获取旧 objectKey，事务成功后删除

```typescript
// 替换头像的改进版本
await prisma.$transaction(async ($prisma) => {
  // 1. 先查询旧头像的 objectKey
  const oldImage = await $prisma.userImage.findUnique({
    where: { userId },
    select: { objectKey: true },
  })
  
  // 2. 删除旧记录
  await $prisma.userImage.deleteMany({ where: { userId } })
  
  // 3. 创建新记录
  await $prisma.user.update({
    where: { id: userId },
    data: { image: { create: image } },
  })
  
  // 4. 返回旧 objectKey 供事务外删除
  return oldImage?.objectKey
})
.then(async (oldObjectKey) => {
  // 事务成功后，异步删除旧文件
  if (oldObjectKey) {
    try {
      await deleteFromStorage(oldObjectKey)
    } catch (error) {
      console.error('Failed to delete old profile image:', error)
    }
  }
})
```

---

##### 策略 B：定时任务兜底（必要补充）

即使有补偿删除，也可能因为：
- 补偿删除本身失败（网络问题）
- 进程崩溃（没机会执行 catch 块）
- 并发场景导致竞态

**需要实现的清理任务**：

```typescript
// 定期扫描的清理逻辑
async function cleanupOrphanedImages() {
  // 1. 从数据库获取所有有效的 objectKey
  const validKeys = new Set(
    (await prisma.userImage.findMany({ select: { objectKey: true } }))
      .map(img => img.objectKey)
  )
  
  // 2. 扫描 S3/MinIO 中的 profile-images 目录
  const s3Objects = await listStorageObjects('users/*/profile-images/')
  
  // 3. 找出不在数据库中的孤立文件
  const orphanedKeys = s3Objects.filter(obj => !validKeys.has(obj.Key))
  
  // 4. 删除孤立文件（或先标记，人工确认后再删）
  for (const key of orphanedKeys) {
    await deleteFromStorage(key)
    console.log(`Deleted orphaned image: ${key}`)
  }
}
```

**执行频率建议**：
- 开发环境：每天一次
- 生产环境：每周一次（或每月，视存储成本压力）

---

##### 策略 C：软删除 + 延迟清理（最安全）

**核心思想**：不立即删除，标记为待删除，一段时间后再清理

```typescript
// 方案：UserImage 表增加 status 字段
model UserImage {
  id        String   @id @default(cuid())
  objectKey String
  status    String   @default("active") // active | pending_deletion
  deletedAt DateTime?
  
  // ... 其他字段
}
```

**工作流**：
```
用户删除/替换头像
    ↓
更新数据库 status = 'pending_deletion', deletedAt = NOW()
    ↓
定时任务扫描：status='pending_deletion' AND deletedAt < NOW()-7天
    ↓
确认文件不再被引用 → 同时删除数据库记录 + S3 对象
```

**优势**：
- 可恢复：7 天内用户后悔可以恢复
- 安全：给缓存、CDN 足够的失效时间
- 可审计：有删除时间记录

---

#### 存储删除功能实现

当前 `storage.server.ts` 只有上传和获取签名，缺少删除函数：

```typescript
// 需要新增的函数
export async function deleteFromStorage(key: string) {
  const { url, headers } = getSignedDeleteRequestInfo(key)
  
  const response = await fetch(url, {
    method: 'DELETE',
    headers,
  })
  
  if (!response.ok) {
    throw new Error(`Failed to delete object: ${key}, status: ${response.status}`)
  }
}

function getSignedDeleteRequestInfo(key: string) {
  return getBaseSignedRequestInfo({
    method: 'DELETE',
    key,
  })
}
```

---

## 二、用户数据导出的敏感字段裁剪

**核心文件**: `app/routes/resources/download-user-data.tsx`

### 2.1 数据查询策略

使用 Prisma 的 `include` + `select` 组合进行精细控制：

```typescript
const user = await prisma.user.findUniqueOrThrow({
	where: { id: userId },
	include: {
		image: {
			select: {
				id: true,
				createdAt: true,
				updatedAt: true,
				objectKey: true,
			},
		},
		notes: {
			include: {
				images: {
					select: {
						id: true,
						createdAt: true,
						updatedAt: true,
						objectKey: true,
					},
				},
			},
		},
		password: false, // <-- 显式排除敏感字段
		sessions: true,
		roles: true,
	},
})
```

### 2.2 敏感字段裁剪机制

#### 方式一: 显式排除 (`password: false`)

```typescript
password: false, // 显式设置为 false 排除密码字段
```

这是最直接的敏感字段保护方式。即使 `include` 会默认包含所有字段，`password: false` 强制排除。

#### 方式二: 白名单选择 (`select`)

对于关联模型（如 `image`、`notes.images`），使用 `select` 明确指定返回字段：

```typescript
image: {
	select: {
		id: true,
		createdAt: true,
		updatedAt: true,
		objectKey: true,
	},
},
```

**优势**: 避免意外暴露敏感字段，只返回业务需要的字段。

#### 方式三: 图片 URL 转换（而非返回 Blob）

```typescript
return Response.json({
	user: {
		...user,
		image: user.image
			? {
					...user.image,
					url: domain + getUserImgSrc(user.image.objectKey),
				}
			: null,
		notes: user.notes.map((note) => ({
			...note,
			images: note.images.map((image) => ({
				...image,
				url: domain + getNoteImgSrc(image.objectKey),
			})),
		})),
	},
})
```

**设计考量**:
- 不返回图片二进制数据（避免导出文件过大）
- 转换为可访问的 URL
- 用户可通过 URL 下载原始图片

### 2.3 导出数据结构示例

```json
{
  "user": {
    "id": "cuid",
    "email": "user@example.com",
    "username": "username",
    "name": "User Name",
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-01T00:00:00Z",
    "image": {
      "id": "cuid",
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-01T00:00:00Z",
      "objectKey": "users/xxx/profile-images/xxx",
      "url": "https://domain.com/resources/user-images/xxx"
    },
    "notes": [
      {
        "id": "cuid",
        "title": "Note Title",
        "content": "Note Content",
        "createdAt": "2024-01-01T00:00:00Z",
        "updatedAt": "2024-01-01T00:00:00Z",
        "images": [
          {
            "id": "cuid",
            "objectKey": "users/xxx/notes/xxx/images/xxx",
            "url": "https://domain.com/resources/note-images/xxx"
          }
        ]
      }
    ],
    "sessions": [...],
    "roles": [...]
    // password 字段已被排除
  }
}
```

### 2.4 安全设计要点

| 安全措施 | 实现方式 | 文件位置 |
|---------|---------|---------|
| 密码排除 | `password: false` | download-user-data.tsx:36 |
| 图片字段白名单 | `select: { id, createdAt, updatedAt, objectKey }` | download-user-data.tsx:17-22 |
| 权限验证 | `requireUserId(request)` | download-user-data.tsx:7 |
| URL 而非 Blob | `getUserImgSrc()` / `getNoteImgSrc()` | download-user-data.tsx:50, 57 |

---

## 三、关键代码位置汇总

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| Multipart 解析 | `app/routes/settings/profile/photo.tsx` | L69 |
| 头像上传存储 | `app/utils/storage.server.ts` | L29-L38 |
| AWS 签名实现 | `app/utils/storage.server.ts` | L52-L148 |
| 头像事务处理 | `app/routes/settings/profile/photo.tsx` | L98-L104 |
| 数据导出路由 | `app/routes/resources/download-user-data.tsx` | 全文 |
| 密码字段排除 | `app/routes/resources/download-user-data.tsx` | L36 |
| 图片字段白名单 | `app/routes/resources/download-user-data.tsx` | L16-L33 |
