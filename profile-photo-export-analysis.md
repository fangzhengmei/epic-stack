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
