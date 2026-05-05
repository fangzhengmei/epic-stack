# Epic Stack 资产存储分析报告

## 概述

本报告详细分析 Epic Stack 中用户上传资产（主要是图片）时，文件如何从前端流转到远端对象存储（S3 兼容服务，如 Tigris、S3 或 R2）的完整流程。重点分析上传请求、存储服务调用和 Signed URL 生成这几个环节是如何跨前端、后端和存储服务协作的。

## 架构概览

Epic Stack 采用了一种**混合存储架构**：

1. **元数据存储**：图片的引用信息（objectKey、所有权、时间戳等）存储在 SQLite 数据库中
2. **二进制数据存储**：实际的图片文件存储在 S3 兼容的对象存储服务（Tigris）中
3. **访问代理**：图片通过本地 `/resources/images` 路由进行代理访问，而不是直接暴露存储服务 URL

这种架构的优势：
- 数据库保持轻量，备份高效
- 利用对象存储的可扩展性和 CDN 能力
- 统一的访问控制和图片优化入口

## 环境配置

存储服务通过环境变量进行配置，使用 AWS S3 兼容的 API：

```typescript
// app/utils/env.server.ts:23-28
// Tigris Object Storage Configuration
AWS_ACCESS_KEY_ID: z.string(),
AWS_SECRET_ACCESS_KEY: z.string(),
AWS_REGION: z.string(),
AWS_ENDPOINT_URL_S3: z.string().url(),
BUCKET_NAME: z.string(),
```

**关键配置项说明：**
- `AWS_ENDPOINT_URL_S3`: S3 兼容服务的端点 URL（如 Tigris 的 `https://fly.storage.tigris.dev`）
- `BUCKET_NAME`: 存储桶名称
- `AWS_REGION`: 区域（Tigris 使用 "auto"）
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`: 认证凭证

## 核心模块分析

### 1. 存储服务核心 (`storage.server.ts`)

这是整个资产存储系统的核心，包含了所有与存储服务交互的逻辑。

#### 1.1 上传函数

系统提供了两个主要的上传函数：

**用户头像上传：**
```typescript
// app/utils/storage.server.ts:29-38
export async function uploadProfileImage(
	userId: string,
	file: File | FileUpload,
) {
	const fileId = createId()
	const fileExtension = file.name.split('.').pop() || ''
	const timestamp = Date.now()
	const key = `users/${userId}/profile-images/${timestamp}-${fileId}.${fileExtension}`
	return uploadToStorage(file, key)
}
```

**笔记图片上传：**
```typescript
// app/utils/storage.server.ts:40-50
export async function uploadNoteImage(
	userId: string,
	noteId: string,
	file: File | FileUpload,
) {
	const fileId = createId()
	const fileExtension = file.name.split('.').pop() || ''
	const timestamp = Date.now()
	const key = `users/${userId}/notes/${noteId}/images/${timestamp}-${fileId}.${fileExtension}`
	return uploadToStorage(file, key)
}
```

**对象键（Object Key）命名策略：**
- 用户头像：`users/{userId}/profile-images/{timestamp}-{fileId}.{extension}`
- 笔记图片：`users/{userId}/notes/{noteId}/images/{timestamp}-{fileId}.{extension}`

这种命名策略的优势：
1. **层级结构清晰**：按用户、资源类型、时间戳组织
2. **唯一性保证**：使用 cuid2 生成的 fileId 确保唯一性
3. **时间戳缓存 busting**：每次上传都生成新的 key，避免缓存问题

#### 1.2 核心上传逻辑

```typescript
// app/utils/storage.server.ts:11-27
async function uploadToStorage(file: File | FileUpload, key: string) {
	const { url, headers } = getSignedPutRequestInfo(file, key)

	const uploadResponse = await fetch(url, {
		method: 'PUT',
		headers,
		body: file instanceof File ? file : (file as FileUpload).stream(),
	})

	if (!uploadResponse.ok) {
		const errorMessage = `Failed to upload file to storage. Server responded with ${uploadResponse.status}: ${uploadResponse.statusText}`
		console.error(errorMessage)
		throw new Error(`Failed to upload object: ${key}`)
	}

	return key
}
```

**关键要点：**
1. **服务端直传**：文件从应用服务器直接上传到存储服务，而不是通过客户端
2. **流式上传**：支持 `File` 和 `FileUpload` 两种类型，后者可以流式传输
3. **Signed URL 模式**：使用预签名的 URL 和请求头进行认证

#### 1.3 Signed URL 生成（AWS Signature V4 实现）

Epic Stack 没有使用 AWS SDK，而是**手动实现了 AWS Signature Version 4** 签名算法。这是一个关键的设计决策，使得代码更轻量且不依赖大型 SDK。

**签名密钥生成：**
```typescript
// app/utils/storage.server.ts:52-75
function hmacSha256(key: string | Buffer, message: string) {
	const hmac = createHmac('sha256', key)
	hmac.update(message)
	return hmac.digest()
}

function sha256(message: string) {
	const hash = createHash('sha256')
	hash.update(message)
	return hash.digest('hex')
}

function getSignatureKey(
	key: string,
	dateStamp: string,
	regionName: string,
	serviceName: string,
) {
	const kDate = hmacSha256(`AWS4${key}`, dateStamp)
	const kRegion = hmacSha256(kDate, regionName)
	const kService = hmacSha256(kRegion, serviceName)
	const kSigning = hmacSha256(kService, 'aws4_request')
	return kSigning
}
```

**基础签名请求信息生成：**
```typescript
// app/utils/storage.server.ts:77-148
function getBaseSignedRequestInfo({
	method,
	key,
	contentType,
	uploadDate,
}: {
	method: 'GET' | 'PUT'
	key: string
	contentType?: string
	uploadDate?: string
}) {
	const url = `${STORAGE_ENDPOINT}/${STORAGE_BUCKET}/${key}`
	const endpoint = new URL(url)

	// 准备日期字符串
	const amzDate = new Date().toISOString().replace(/[:-]|\.\d{3}/g, '')
	const dateStamp = amzDate.slice(0, 8)

	// 构建请求头数组
	const headers = [
		...(contentType ? [`content-type:${contentType}`] : []),
		`host:${endpoint.host}`,
		`x-amz-content-sha256:UNSIGNED-PAYLOAD`,
		`x-amz-date:${amzDate}`,
		...(uploadDate ? [`x-amz-meta-upload-date:${uploadDate}`] : []),
	]

	const canonicalHeaders = headers.join('\n') + '\n'
	const signedHeaders = headers.map((h) => h.split(':')[0]).join(';')

	// 构建规范请求（Canonical Request）
	const canonicalRequest = [
		method,
		`/${STORAGE_BUCKET}/${key}`,
		'', // canonicalQueryString
		canonicalHeaders,
		signedHeaders,
		'UNSIGNED-PAYLOAD',
	].join('\n')

	// 准备待签名字符串（String to Sign）
	const algorithm = 'AWS4-HMAC-SHA256'
	const credentialScope = `${dateStamp}/${STORAGE_REGION}/s3/aws4_request`
	const stringToSign = [
		algorithm,
		amzDate,
		credentialScope,
		sha256(canonicalRequest),
	].join('\n')

	// 计算签名
	const signingKey = getSignatureKey(
		STORAGE_SECRET_KEY,
		dateStamp,
		STORAGE_REGION,
		's3',
	)
	const signature = createHmac('sha256', signingKey)
		.update(stringToSign)
		.digest('hex')

	// 构建基础请求头
	const baseHeaders = {
		'X-Amz-Date': amzDate,
		'X-Amz-Content-SHA256': 'UNSIGNED-PAYLOAD',
		Authorization: [
			`${algorithm} Credential=${STORAGE_ACCESS_KEY}/${credentialScope}`,
			`SignedHeaders=${signedHeaders}`,
			`Signature=${signature}`,
		].join(', '),
	}

	return { url, baseHeaders }
}
```

**AWS Signature V4 签名流程详解：**

1. **日期格式化**：
   - `amzDate`: ISO 格式日期，去除特殊字符（如 `20260505T123456Z`）
   - `dateStamp`: 仅日期部分（如 `20260505`）

2. **构建规范请求（Canonical Request）**：
   ```
   <HTTPMethod>\n
   <CanonicalURI>\n
   <CanonicalQueryString>\n
   <CanonicalHeaders>\n
   <SignedHeaders>\n
   <PayloadHash>
   ```
   - 使用 `UNSIGNED-PAYLOAD` 表示不计算请求体哈希（更高效）

3. **构建待签名字符串（String to Sign）**：
   ```
   <Algorithm>\n
   <RequestDate>\n
   <CredentialScope>\n
   <HashedCanonicalRequest>
   ```

4. **派生签名密钥**：
   - `kDate = HMAC("AWS4" + SecretKey, Date)`
   - `kRegion = HMAC(kDate, Region)`
   - `kService = HMAC(kRegion, Service)`
   - `kSigning = HMAC(kService, "aws4_request")`

5. **计算签名**：
   - `Signature = HMAC(kSigning, StringToSign)`

6. **构建 Authorization 头**：
   ```
   AWS4-HMAC-SHA256 
   Credential=AKIAIOSFODNN7EXAMPLE/20130524/us-east-1/s3/aws4_request, 
   SignedHeaders=content-type;host;x-amz-date, 
   Signature=76d97...
   ```

**PUT 请求签名信息：**
```typescript
// app/utils/storage.server.ts:150-167
function getSignedPutRequestInfo(file: File | FileUpload, key: string) {
	const uploadDate = new Date().toISOString()
	const { url, baseHeaders } = getBaseSignedRequestInfo({
		method: 'PUT',
		key,
		contentType: file.type,
		uploadDate,
	})

	return {
		url,
		headers: {
			...baseHeaders,
			'Content-Type': file.type,
			'X-Amz-Meta-Upload-Date': uploadDate,
		},
	}
}
```

**GET 请求签名信息（用于读取图片）：**
```typescript
// app/utils/storage.server.ts:169-179
export function getSignedGetRequestInfo(key: string) {
	const { url, baseHeaders } = getBaseSignedRequestInfo({
		method: 'GET',
		key,
	})

	return {
		url,
		headers: baseHeaders,
	}
}
```

### 2. 前端上传实现

#### 2.1 用户头像上传组件 (`photo.tsx`)

**表单配置：**
```typescript
// app/routes/settings/profile/photo.tsx:135-141
<Form
	method="POST"
	encType="multipart/form-data"
	className="flex flex-col items-center justify-center gap-10"
	onReset={() => setNewImageSrc(null)}
	{...getFormProps(form)}
>
```

**文件输入配置：**
```typescript
// app/routes/settings/profile/photo.tsx:160-176
<input
	{...getInputProps(fields.photoFile, { type: 'file' })}
	accept="image/*"
	className="peer sr-only"
	required
	tabIndex={newImageSrc ? -1 : 0}
	onChange={(e) => {
		const file = e.currentTarget.files?.[0]
		if (file) {
			const reader = new FileReader()
			reader.onload = (event) => {
				setNewImageSrc(event.target?.result?.toString() ?? null)
			}
			reader.readAsDataURL(file)
		}
	}}
/>
```

**关键特性：**
1. **渐进式增强**：使用 CSS 控制按钮显示，无 JavaScript 也能工作
2. **图片预览**：使用 `FileReader` 读取本地文件进行预览
3. **文件类型限制**：`accept="image/*"` 只接受图片文件

**Schema 验证：**
```typescript
// app/routes/settings/profile/photo.tsx:29-44
const MAX_SIZE = 1024 * 1024 * 3 // 3MB

const NewImageSchema = z.object({
	intent: z.literal('submit'),
	photoFile: z
		.instanceof(File)
		.refine((file) => file.size > 0, 'Image is required')
		.refine(
			(file) => file.size <= MAX_SIZE,
			'Image size must be less than 3MB',
		),
})
```

### 3. 后端 Action 处理

#### 3.1 用户头像上传 Action

```typescript
// app/routes/settings/profile/photo.tsx:66-107
export async function action({ request }: Route.ActionArgs) {
	const userId = await requireUserId(request)

	const formData = await parseFormData(request, { maxFileSize: MAX_SIZE })
	const submission = await parseWithZod(formData, {
		schema: PhotoFormSchema.transform(async (data) => {
			if (data.intent === 'delete') return { intent: 'delete' }
			if (data.photoFile.size <= 0) return z.NEVER
			return {
				intent: data.intent,
				image: {
					objectKey: await uploadProfileImage(userId, data.photoFile),
				},
			}
		}),
		async: true,
	})

	if (submission.status !== 'success') {
		return data(
			{ result: submission.reply() },
			{ status: submission.status === 'error' ? 400 : 200 },
		)
	}

	const { image, intent } = submission.value

	if (intent === 'delete') {
		await prisma.userImage.deleteMany({ where: { userId } })
		return redirect('/settings/profile')
	}

	// 事务处理：删除旧图片，创建新图片
	await prisma.$transaction(async ($prisma) => {
		await $prisma.userImage.deleteMany({ where: { userId } })
		await $prisma.user.update({
			where: { id: userId },
			data: { image: { create: image } },
		})
	})

	return redirect('/settings/profile')
}
```

**处理流程：**
1. **认证检查**：`requireUserId(request)` 确保用户已登录
2. **表单解析**：`parseFormData` 解析 multipart/form-data
3. **Schema 转换**：在 Zod transform 中调用 `uploadProfileImage` 上传文件
4. **事务处理**：使用数据库事务确保原子性
5. **重定向**：操作完成后重定向到配置页面

#### 3.2 笔记图片上传 Action

```typescript
// app/routes/users/$username/notes/+shared/note-editor.server.tsx:27-131
export async function action({ request }: ActionFunctionArgs) {
	const userId = await requireUserId(request)

	const formData = await parseFormData(request, {
		maxFileSize: MAX_UPLOAD_SIZE,
	})

	const submission = await parseWithZod(formData, {
		schema: NoteEditorSchema.superRefine(async (data, ctx) => {
			// 验证笔记归属
			if (!data.id) return
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
			const noteId = data.id ?? cuid()
			return {
				...data,
				id: noteId,
				// 处理现有图片更新
				imageUpdates: await Promise.all(
					images.filter(imageHasId).map(async (i) => {
						if (imageHasFile(i)) {
							return {
								id: i.id,
								altText: i.altText,
								objectKey: await uploadNoteImage(userId, noteId, i.file),
							}
						} else {
							return {
								id: i.id,
								altText: i.altText,
							}
						}
					}),
				),
				// 处理新图片上传
				newImages: await Promise.all(
					images
						.filter(imageHasFile)
						.filter((i) => !i.id)
						.map(async (image) => {
							return {
								altText: image.altText,
								objectKey: await uploadNoteImage(userId, noteId, image.file),
							}
						}),
				),
			}
		}),
		async: true,
	})

	// ... 数据库操作
}
```

**关键特性：**
1. **多图片支持**：同时处理多个图片上传
2. **分类处理**：区分"新图片"和"更新图片"
3. **并行上传**：使用 `Promise.all` 并行处理多个上传
4. **权限验证**：验证笔记属于当前用户

### 4. 图片读取与代理服务

#### 4.1 图片资源路由 (`images.tsx`)

```typescript
// app/routes/resources/images.tsx:28-80
export async function loader({ request }: Route.LoaderArgs) {
	const url = new URL(request.url)
	const searchParams = url.searchParams

	const headers = new Headers()
	headers.set('Cache-Control', 'public, max-age=31536000, immutable')

	const objectKey = searchParams.get('objectKey')

	return getImgResponse(request, {
		headers,
		allowlistedOrigins: [
			getDomainUrl(request),
			process.env.AWS_ENDPOINT_URL_S3,
		].filter(Boolean),
		cacheFolder: await getCacheDir(),
		getImgSource: () => {
			if (objectKey) {
				const { url: signedUrl, headers: signedHeaders } =
					getSignedGetRequestInfo(objectKey)
				return {
					type: 'fetch',
					url: signedUrl,
					headers: signedHeaders,
				}
			}

			const src = searchParams.get('src')
			invariantResponse(src, 'src query parameter is required', { status: 400 })

			if (URL.canParse(src)) {
				// 从外部 URL 获取图片
				return {
					type: 'fetch',
					url: src,
				}
			}
			// 从文件系统获取图片
			if (src.startsWith('/assets')) {
				return {
					type: 'fs',
					path: '.' + src,
				}
			}
			return {
				type: 'fs',
				path: './public' + src,
			}
		},
	})
}
```

**代理服务的作用：**
1. **统一入口**：所有图片访问都通过 `/resources/images` 路由
2. **缓存控制**：设置 `max-age=31536000, immutable` 进行强缓存
3. **多种来源支持**：
   - 对象存储（通过 `objectKey` 参数）
   - 外部 URL（通过 `src` 参数）
   - 本地文件系统（`/assets` 或 `public` 目录）
4. **图片优化**：使用 `openimg` 库进行图片格式转换、缩放等优化

#### 4.2 图片 URL 生成函数

```typescript
// app/utils/misc.tsx:8-16
export function getUserImgSrc(objectKey?: string | null) {
	return objectKey
		? `/resources/images?objectKey=${encodeURIComponent(objectKey)}`
		: '/img/user.png'
}

export function getNoteImgSrc(objectKey: string) {
	return `/resources/images?objectKey=${encodeURIComponent(objectKey)}`
}
```

**设计要点：**
1. **URL 编码**：使用 `encodeURIComponent` 确保 objectKey 中的特殊字符正确传递
2. **默认值处理**：`getUserImgSrc` 在 objectKey 为空时返回默认头像
3. **统一格式**：所有图片 URL 都指向本地代理路由，不暴露存储服务细节

## 完整文件流转流程

### 1. 上传流程详解

#### 阶段 1：前端表单提交

```
用户选择图片 → 前端预览 → 提交表单
     ↓
Form 组件 (encType="multipart/form-data")
     ↓
POST /settings/profile/photo
```

**关键步骤：**
1. 用户通过文件选择器选择图片
2. `FileReader` 读取文件并在前端进行预览
3. 用户点击 "Save Photo" 提交表单
4. 浏览器发送 `multipart/form-data` POST 请求

#### 阶段 2：后端 Action 处理

```
Request → requireUserId (认证) → parseFormData (解析)
     ↓
Zod Schema 验证 + Transform
     ↓
uploadProfileImage(userId, file) 被调用
```

**关键步骤：**
1. `requireUserId` 验证用户是否登录
2. `parseFormData` 解析 multipart 表单数据（最大 3MB）
3. Zod Schema 进行验证，包括文件大小检查
4. 在 `transform` 回调中调用上传函数

#### 阶段 3：存储服务上传

```
uploadProfileImage → 生成 objectKey
     ↓
getSignedPutRequestInfo (生成签名 URL 和请求头)
     ↓
fetch(url, { method: 'PUT', headers, body: file })
     ↓
存储服务 (Tigris/S3/R2)
```

**关键步骤：**
1. 生成唯一的 objectKey（包含 userId、时间戳、fileId）
2. 调用 `getSignedPutRequestInfo`：
   - 构建 AWS Signature V4 签名
   - 生成签名的 URL 和请求头
3. 使用 `fetch` 直接将文件 PUT 到存储服务
4. 存储服务验证签名并存储文件

#### 阶段 4：元数据持久化

```
上传成功 → 返回 objectKey
     ↓
数据库事务
     ↓
1. 删除旧的 userImage 记录
2. 创建新的 userImage 记录（关联 objectKey）
     ↓
重定向到配置页面
```

**关键步骤：**
1. 上传成功后，`uploadProfileImage` 返回 objectKey
2. 使用 `prisma.$transaction` 确保原子性：
   - 删除用户所有旧的图片记录
   - 创建新的图片记录，包含 objectKey
3. 重定向用户到配置页面

### 2. 读取流程详解

#### 阶段 1：图片 URL 构建

```
组件渲染 → getUserImgSrc(objectKey)
     ↓
生成 URL: /resources/images?objectKey=xxx
     ↓
<img src="..." />
```

**关键步骤：**
1. 组件从数据库获取 userImage 记录（包含 objectKey）
2. 调用 `getUserImgSrc` 构建图片 URL
3. 渲染 `<img>` 标签，浏览器发起 GET 请求

#### 阶段 2：代理路由处理

```
GET /resources/images?objectKey=xxx
     ↓
getImgResponse (openimg 库)
     ↓
getImgSource 回调
     ↓
getSignedGetRequestInfo(objectKey)
     ↓
生成签名的 GET URL 和请求头
```

**关键步骤：**
1. `images.tsx` 的 loader 接收请求
2. 解析 `objectKey` 查询参数
3. 调用 `getSignedGetRequestInfo` 生成签名信息
4. 返回给 `openimg` 库，由其进行实际的获取和优化

#### 阶段 3：图片获取与优化

```
openimg 使用 signed URL 从存储服务获取图片
     ↓
进行图片优化（格式转换、缩放、缓存）
     ↓
返回优化后的图片给浏览器
     ↓
浏览器缓存图片 (max-age=31536000)
```

**关键步骤：**
1. `openimg` 使用签名的 URL 和请求头从存储服务获取原始图片
2. 根据请求参数进行优化（如 `w=200&h=200&format=webp`）
3. 本地缓存优化后的图片（在 `/data/images` 或测试目录）
4. 设置 `Cache-Control: public, max-age=31536000, immutable`
5. 返回优化后的图片响应

## 数据库模型

### UserImage 模型

```prisma
model UserImage {
  id          String   @id @default(cuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  objectKey   String   // 存储服务中的对象键
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@index([userId])
}
```

### NoteImage 模型

```prisma
model NoteImage {
  id          String   @id @default(cuid())
  noteId      String
  note        Note     @relation(fields: [noteId], references: [id], onDelete: Cascade)
  objectKey   String   // 存储服务中的对象键
  altText     String?  // 替代文本
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@index([noteId])
}
```

**设计要点：**
1. **只存储引用**：数据库中只存储 `objectKey`，不存储二进制数据
2. **级联删除**：`onDelete: Cascade` 确保用户/笔记删除时，图片记录也被删除
3. **索引优化**：对 `userId` 和 `noteId` 建立索引，加速查询

## 关键设计决策分析

### 1. 服务端直传 vs 客户端直传

**当前实现：服务端直传**

```
客户端 → 应用服务器 → 存储服务
```

**优点：**
- 更安全：不在前端暴露存储服务凭证
- 更好的控制：可以在上传前后执行业务逻辑（验证、转换等）
- 统一的错误处理
- 更简单的前端实现

**缺点：**
- 应用服务器成为瓶颈：大文件上传会占用服务器资源
- 延迟增加：数据需要经过应用服务器中转

**替代方案：客户端直传（Presigned URL 模式）**

```
客户端 → 应用服务器 (获取 presigned URL)
客户端 → 存储服务 (使用 presigned URL 直传)
```

### 2. 手动实现 AWS Signature V4 vs 使用 AWS SDK

**当前实现：手动实现**

**优点：**
- 零依赖：不需要安装庞大的 AWS SDK
- 更轻量：代码体积小，启动快
- 完全控制：可以精确控制签名逻辑
- S3 兼容：可以与任何 S3 兼容的服务配合使用（Tigris、MinIO、R2 等）

**缺点：**
- 维护成本：需要自己维护签名逻辑
- 功能受限：只实现了必要的功能（PUT、GET），不支持高级特性
- 测试复杂：需要确保签名实现完全正确

### 3. 代理服务访问 vs 直接访问存储服务

**当前实现：代理服务**

**优点：**
- 统一入口：所有图片访问都经过应用服务器
- 图片优化：可以使用 `openimg` 进行实时优化
- 访问控制：可以在代理层实现权限检查
- URL 美化：不暴露存储服务的内部 URL
- 缓存控制：统一设置缓存策略

**缺点：**
- 服务器负载：所有图片流量都经过应用服务器
- 延迟增加：多了一层中转

## 安全考虑

### 1. 凭证安全

- **存储凭证**：`AWS_SECRET_ACCESS_KEY` 只在服务端使用，不会暴露给前端
- **环境变量**：凭证通过环境变量注入，不提交到代码仓库
- **签名机制**：使用 AWS Signature V4，不在请求中直接传输密钥

### 2. 访问控制

- **认证检查**：所有上传操作都需要先通过 `requireUserId` 认证
- **权限验证**：笔记图片上传时验证笔记属于当前用户
- **对象键隔离**：每个用户的文件存储在独立的 `users/{userId}/` 前缀下

### 3. 文件验证

- **大小限制**：头像最大 3MB，笔记图片有 `MAX_UPLOAD_SIZE` 限制
- **类型限制**：前端使用 `accept="image/*"` 限制文件类型
- **Schema 验证**：后端使用 Zod 进行严格的验证

## 性能优化

### 1. 缓存策略

- **浏览器缓存**：`Cache-Control: public, max-age=31536000, immutable`
- **服务端缓存**：`openimg` 在本地缓存优化后的图片
- **对象键版本化**：每次上传生成新的 objectKey，自然避免缓存问题

### 2. 并行处理

- 笔记图片上传使用 `Promise.all` 并行处理多个文件
- 不阻塞其他请求处理

### 3. 流式上传

- 支持 `FileUpload` 类型，可以流式传输大文件
- 不需要完全加载到内存中

## 扩展性分析

### 1. 支持更多文件类型

当前实现主要针对图片，但架构可以扩展到其他文件类型：

1. 添加新的上传函数（如 `uploadDocument`、`uploadVideo`）
2. 更新 Schema 验证规则
3. 添加新的数据库模型（或扩展现有模型）

### 2. 切换存储服务

由于使用了 S3 兼容的 API，切换存储服务非常简单：

1. 更新环境变量：
   - `AWS_ENDPOINT_URL_S3`
   - `AWS_REGION`
   - `BUCKET_NAME`
   - 凭证信息

2. 不需要修改代码（只要新服务支持 S3 API）

**支持的存储服务：**
- AWS S3
- Cloudflare R2
- Tigris（当前使用）
- MinIO（自托管）
- Backblaze B2
- 任何其他 S3 兼容的对象存储

### 3. 实现客户端直传

如果未来需要支持大文件上传或减轻服务器负载，可以考虑实现客户端直传：

1. 创建一个 API 端点，返回 presigned URL
2. 前端获取 presigned URL 后直接上传到存储服务
3. 上传完成后通知后端更新数据库

## 代码位置索引

| 功能 | 文件路径 | 关键函数/组件 |
|------|----------|--------------|
| 存储服务核心 | `app/utils/storage.server.ts` | `uploadProfileImage`, `uploadNoteImage`, `getSignedPutRequestInfo`, `getBaseSignedRequestInfo` |
| 用户头像上传 | `app/routes/settings/profile/photo.tsx` | `action`, `PhotoRoute` 组件 |
| 笔记图片上传 | `app/routes/users/$username/notes/+shared/note-editor.server.tsx` | `action` |
| 图片资源路由 | `app/routes/resources/images.tsx` | `loader` |
| URL 生成工具 | `app/utils/misc.tsx` | `getUserImgSrc`, `getNoteImgSrc` |
| 环境配置 | `app/utils/env.server.ts` | 存储相关环境变量 Schema |
| 文档 | `docs/image-storage.md` | 整体架构说明 |
| 决策记录 | `docs/decisions/040-tigris-image-storage.md` | 架构决策背景 |

## 总结

Epic Stack 的资产存储系统采用了一种**简洁但功能完整**的架构设计：

1. **混合存储**：元数据在 SQLite，二进制数据在对象存储
2. **服务端直传**：文件经过应用服务器中转，提供更好的控制和安全性
3. **手动签名**：不依赖 AWS SDK，手动实现 AWS Signature V4，保持轻量
4. **代理访问**：通过 `/resources/images` 路由统一处理图片访问和优化
5. **S3 兼容**：可以轻松切换到任何 S3 兼容的存储服务

这种设计非常适合中小型应用，提供了：
- ✅ 良好的安全性
- ✅ 清晰的代码结构
- ✅ 灵活的扩展性
- ✅ 较低的维护成本

对于需要处理超大文件或极高并发的场景，可以考虑演进到客户端直传模式，但当前实现已经为大多数应用场景提供了优秀的基础。
