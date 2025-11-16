# 数据库模型详解

本文档详细分析 ColiVara 的数据库模型设计，所有模型定义位于 `web/api/models.py`（709 行代码）。

## 1. 模型层级关系

```
CustomUser (accounts.models)
    │
    └── Collection (api.models)
            │
            ├── name: CharField(255)
            ├── metadata: JSONField
            ├── created_at: DateTimeField
            └── updated_at: DateTimeField
            │
            └── Document (api.models)
                    │
                    ├── name: CharField(255)
                    ├── url: URLField
                    ├── s3_file: FileField
                    ├── metadata: JSONField
                    ├── created_at: DateTimeField
                    └── updated_at: DateTimeField
                    │
                    └── Page (api.models)
                            │
                            ├── page_number: IntegerField
                            ├── content: TextField (暂未使用)
                            ├── img_base64: TextField
                            ├── created_at: DateTimeField
                            └── updated_at: DateTimeField
                            │
                            └── PageEmbedding (api.models)
                                    │
                                    └── embedding: HalfVectorField(128)
```

## 2. CustomUser 模型

**文件位置**: `web/accounts/models.py:12-31`

### 2.1 模型定义

```python
class CustomUser(AbstractUser):
    subscribe_to_emails = models.BooleanField(default=False)
    tier = models.CharField(max_length=255, default="free")
    stripe_customer_id = models.CharField(max_length=255, blank=True)
    stripe_subscription_id = models.CharField(max_length=255, blank=True)
    svix_application_id = models.CharField(max_length=255, blank=True)
    svix_endpoint_id = models.CharField(max_length=255, blank=True)
    token = models.CharField(max_length=255, unique=True, editable=False)
```

### 2.2 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `subscribe_to_emails` | BooleanField | 是否订阅邮件通知 |
| `tier` | CharField | 用户等级（free/starter/professional） |
| `stripe_customer_id` | CharField | Stripe 客户 ID |
| `stripe_subscription_id` | CharField | Stripe 订阅 ID |
| `svix_application_id` | CharField | Svix 应用 ID（Webhook） |
| `svix_endpoint_id` | CharField | Svix 端点 ID（Webhook） |
| `token` | CharField | API 认证令牌（自动生成，唯一） |

### 2.3 特殊功能

**自动生成 API Token**:

```python
def save(self, *args, **kwargs):
    if not self.token:
        self.token = secrets.token_urlsafe(32)
    super().save(*args, **kwargs)
```

**继承字段**（来自 AbstractUser）:
- `username`: 用户名
- `email`: 邮箱
- `password`: 密码（哈希存储）
- `first_name`, `last_name`: 姓名
- `is_active`, `is_staff`, `is_superuser`: 权限标志
- `date_joined`, `last_login`: 时间戳

## 3. Collection 模型

**文件位置**: `web/api/models.py:83-112`

### 3.1 模型定义

```python
class Collection(models.Model):
    name = models.CharField(max_length=255, db_index=True)
    owner = models.ForeignKey(
        CustomUser,
        on_delete=models.CASCADE,
        related_name="collections"
    )
    metadata = JSONField(default=dict)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

### 3.2 字段说明

| 字段 | 类型 | 索引 | 说明 |
|------|------|------|------|
| `name` | CharField(255) | ✓ | 集合名称 |
| `owner` | ForeignKey(CustomUser) | ✓ | 所属用户（级联删除） |
| `metadata` | JSONField | - | 自定义元数据（任意 JSON） |
| `created_at` | DateTimeField | - | 创建时间（自动） |
| `updated_at` | DateTimeField | - | 更新时间（自动） |

### 3.3 约束条件

**唯一性约束**（文件位置: `models.py:100-102`）:

```python
models.UniqueConstraint(
    fields=["name", "owner"],
    name="unique_collection_per_user"
)
```

**检查约束**（文件位置: `models.py:104-106`）:

```python
models.CheckConstraint(
    condition=~Q(name="all"),
    name="collection_name_not_all"
)
```

禁止使用 "all" 作为集合名，因为 "all" 是保留关键字，用于查询所有集合。

### 3.4 复合索引

```python
models.Index(
    fields=["name", "owner"],
    name="collection_name_owner_idx"
)
```

优化查询性能：`Collection.objects.filter(name=..., owner=...)`

### 3.5 方法

**异步文档计数**（文件位置: `models.py:95-96`）:

```python
async def document_count(self) -> int:
    return await self.documents.acount()
```

## 4. Document 模型

**文件位置**: `web/api/models.py:114-683`

### 4.1 模型定义

```python
class Document(models.Model):
    collection = models.ForeignKey(
        Collection,
        on_delete=models.CASCADE,
        related_name="documents"
    )
    name = models.CharField(max_length=255, db_index=True)
    url = models.URLField(blank=True)
    s3_file = models.FileField(upload_to=get_upload_path, blank=True)
    metadata = JSONField(default=dict)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

### 4.2 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `collection` | ForeignKey(Collection) | 所属集合（级联删除） |
| `name` | CharField(255) | 文档名称 |
| `url` | URLField | 外部 URL（可选） |
| `s3_file` | FileField | S3 存储文件（可选） |
| `metadata` | JSONField | 自定义元数据 |
| `created_at` | DateTimeField | 创建时间 |
| `updated_at` | DateTimeField | 更新时间 |

**注意**: `url` 和 `s3_file` 至少有一个必须提供。

### 4.3 文件存储路径

**路径生成函数**（文件位置: `models.py:28-51`）:

```python
def get_upload_path(instance, filename):
    """
    Path format: documents/<owner_email>/<filename>
    Dev: dev-documents/<owner_email>/<filename>
    """
    owner_email = instance.collection.owner.email
    safe_email = owner_email.replace("@", "_at_")

    if settings.DEBUG:
        upload_path = f"dev-documents/{safe_email}/{filename}"
    else:
        upload_path = f"documents/{safe_email}/{filename}"

    # 路径长度限制: 116 字符
    MAX_UPLOAD_PATH_LENGTH = 116
    if len(upload_path) > MAX_UPLOAD_PATH_LENGTH:
        extension = os.path.splitext(upload_path)[1]
        trimmed_upload_path = upload_path[:MAX_UPLOAD_PATH_LENGTH - len(extension)] + extension
        return trimmed_upload_path

    return upload_path
```

### 4.4 约束条件

**唯一性约束**:

```python
models.UniqueConstraint(
    fields=["name", "collection"],
    name="unique_document_per_collection"
)
```

**复合索引**:

```python
models.Index(
    fields=["name", "collection"],
    name="document_name_collection_idx"
)
```

### 4.5 核心方法

#### 4.5.1 save_base64_to_s3

**文件位置**: `models.py:143-171`

将 base64 编码的文件保存到 S3：

```python
async def save_base64_to_s3(self, base64_content: str) -> None:
    # 1. 删除旧文件
    await self.delete_s3_file()

    # 2. 解码 base64
    file_content = base64.b64decode(base64_content)

    # 3. 检测 MIME 类型
    mime = magic.Magic(mime=True)
    mime_type = mime.from_buffer(file_content)

    # 4. 获取文件扩展名
    extension = get_extension_from_mime(mime_type)

    # 5. 生成文件名
    safe_name = self.name.replace(" ", "_")
    name_without_extension, _ = os.path.splitext(safe_name)
    filename = f"{name_without_extension}{extension}"

    # 6. 保存到 S3
    await sync_to_async(self.s3_file.save)(
        filename, ContentFile(file_content), save=True
    )
```

#### 4.5.2 embed_document

**文件位置**: `models.py:184-339`

文档嵌入的核心方法，执行完整的文档处理流程：

```python
async def embed_document(self, use_proxy: Optional[bool] = False) -> None:
    """
    处理流程:
    1. 准备文档（转换为 base64 图像）
    2. 分批处理
    3. 发送到嵌入服务
    4. 保存页面和嵌入向量
    """
```

**关键常量**:

```python
EMBEDDINGS_URL = settings.EMBEDDINGS_URL
EMBEDDINGS_BATCH_SIZE = 3           # 每批 3 个页面
DELAY_BETWEEN_BATCHES = 1           # 批次间延迟 1 秒
```

**批量发送函数**（文件位置: `models.py:210-260`）:

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_fixed(5),
    reraise=True,
    retry=retry_if_exception_type(aiohttp.ClientError),
)
async def send_batch(session: aiohttp.ClientSession, images: List[str]):
    payload = {"input": {"task": "image", "input_data": images}}
    headers = {"Authorization": f"Bearer {settings.EMBEDDINGS_URL_TOKEN}"}

    async with session.post(EMBEDDINGS_URL, json=payload, headers=headers) as response:
        if response.status != 200:
            raise ValidationError("Failed to get embeddings")

        out = await response.json()
        return out["output"]["data"]
```

**处理流程**:

```python
# 1. 准备文档（转换为图像）
base64_images = await self._prep_document(use_proxy=use_proxy)

# 2. 分批
batches = [
    base64_images[i : i + EMBEDDINGS_BATCH_SIZE]
    for i in range(0, len(base64_images), EMBEDDINGS_BATCH_SIZE)
]

# 3. 保存文档
await self.asave()

# 4. 逐批处理
async with aiohttp.ClientSession() as session:
    embedding_results = []
    for i, batch in enumerate(batches):
        batch_result = await send_batch(session, batch)
        embedding_results.extend(batch_result)

        if i < len(batches) - 1:
            await asyncio.sleep(DELAY_BETWEEN_BATCHES)

# 5. 保存页面和嵌入
for i, embedding_obj in enumerate(embedding_results):
    page = Page(
        document=self,
        page_number=i + 1,
        img_base64=base64_images[i],
    )
    await page.asave()

    bulk_create_embeddings = [
        PageEmbedding(page=page, embedding=embedding)
        for embedding in embedding_obj["embedding"]
    ]
    await PageEmbedding.objects.abulk_create(bulk_create_embeddings)
```

**错误处理**:

```python
except Exception as e:
    # 如果出错，删除文档（级联删除页面）
    if self.pk:
        await self.adelete()
    raise ValidationError(f"Failed to save pages: {str(e)}")
```

#### 4.5.3 _prep_document

**文件位置**: `models.py:341-585`

文档预处理方法，将任意格式的文档转换为 base64 图像列表：

**支持的文件格式**（文件位置: `models.py:353-485`）:

```python
IMAGE_EXTENSIONS = ["png", "jpg", "jpeg", "tiff", "bmp", "gif"]

ALLOWED_EXTENSIONS = [
    # Office 文档
    "doc", "docx", "docm", "dot", "dotm", "dotx",
    "xls", "xlsx", "xlsm", "xlsb", "xlt", "xltm", "xltx",
    "ppt", "pptx", "pptm", "pps", "pot", "potm", "potx",

    # PDF
    "pdf",

    # 文本和标记
    "txt", "rtf", "csv", "xml", "html", "htm", "xhtml",

    # 开源文档
    "odt", "ods", "odp", "odg", "odf",

    # 图像（100+ 格式）
    ...
]
```

**处理流程**:

```python
async def _prep_document(self, document_data=None, use_proxy=False) -> List[str]:
    """
    步骤:
    1. 验证文档（大小、类型）
    2. 如果不是图像或 PDF，通过 Gotenberg 转换为 PDF
    3. 将 PDF 转换为图像（pdf2image）
    4. 将图像转换为 base64 字符串
    """
```

**1. 获取文档数据**:

```python
if document_data:
    # 从参数获取
    mime_type = magic.from_buffer(document_data)
    extension = get_extension_from_mime(mime_type).lstrip(".")
    filename = f"document.{extension}"

elif self.s3_file:
    # 从 S3 获取
    with self.s3_file.open("rb") as f:
        document_data = f.read()
    filename = os.path.basename(self.s3_file.name)

elif self.url:
    # 从 URL 获取
    content_type, filename, document_data = await self._fetch_document(use_proxy)

    if "text/html" in content_type:
        # 网页转 PDF
        document_data = await self._convert_url_to_pdf(self.url)
        filename = f"{filename}.pdf"
```

**2. 验证**:

```python
MAX_SIZE_BYTES = 50 * 1024 * 1024  # 50 MB

if len(document_data) > MAX_SIZE_BYTES:
    raise ValidationError("Document exceeds maximum size of 50MB.")

if extension not in ALLOWED_EXTENSIONS:
    raise ValidationError(f"File extension .{extension} is not allowed.")
```

**3. 格式转换**:

```python
is_image = extension in IMAGE_EXTENSIONS
is_pdf = extension == "pdf"

if not is_image and not is_pdf:
    # 使用 Gotenberg 转换为 PDF
    pdf_data = await self._convert_to_pdf(document_data, filename)
elif is_pdf:
    pdf_data = document_data
else:
    # 图像直接转 base64
    img_base64 = base64.b64encode(document_data).decode("utf-8")
    return [img_base64]
```

**4. PDF 转图像**:

```python
try:
    images = convert_from_bytes(pdf_data)
except Exception:
    raise ValidationError("Failed to convert PDF to images.")
```

**5. 图像转 base64**:

```python
base64_images = []
for image in images:
    img_io = BytesIO()
    image.save(img_io, "PNG")
    img_data = img_io.getvalue()
    img_base64 = base64.b64encode(img_data).decode("utf-8")
    base64_images.append(img_base64)

return base64_images
```

#### 4.5.4 _fetch_document

**文件位置**: `models.py:587-617`

从 URL 获取文档：

```python
async def _fetch_document(self, use_proxy: Optional[bool] = False):
    proxy = None
    if use_proxy:
        proxy = settings.PROXY_URL
        self.url = self.url.replace("https://", "http://")

    MAX_SIZE_BYTES = 50 * 1024 * 1024

    async with aiohttp.ClientSession() as session:
        async with session.get(self.url, proxy=proxy) as response:
            if response.status != 200:
                raise ValidationError("Failed to fetch document from URL")

            content_type = response.headers.get("Content-Type", "").lower()
            content_disposition = response.headers.get("Content-Disposition", "")
            content_length = response.headers.get("Content-Length")

            if content_length and int(content_length) > MAX_SIZE_BYTES:
                raise ValidationError("Document exceeds maximum size of 50MB.")

            # 从 Content-Disposition 或 URL 提取文件名
            filename_match = re.findall('filename="(.+)"', content_disposition)
            filename = (
                filename_match[0]
                if filename_match
                else os.path.basename(urllib.parse.urlparse(self.url).path)
            )

            return content_type, filename, await response.read()
```

#### 4.5.5 _convert_to_pdf

**文件位置**: `models.py:619-654`

使用 Gotenberg 将文档转换为 PDF：

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_fixed(2),
    reraise=True,
    retry=retry_if_exception_type(aiohttp.ClientError),
)
async def _convert_to_pdf(self, document_data: bytes, filename: str) -> bytes:
    Gotenberg_URL = settings.GOTENBERG_URL
    endpoint = "/forms/libreoffice/convert"
    url = Gotenberg_URL + endpoint

    form = aiohttp.FormData()
    form.add_field("files", document_data, filename=filename)

    headers = {"Accept": "application/pdf"}

    async with aiohttp.ClientSession() as session:
        async with session.post(url, data=form, headers=headers) as response:
            if response.status != 200:
                error_message = await response.text()
                raise ValidationError(f"Failed to convert document to PDF: {error_message}")

            pdf_data = await response.read()

    return pdf_data
```

#### 4.5.6 _convert_url_to_pdf

**文件位置**: `models.py:656-682`

使用 Gotenberg 将网页转换为 PDF：

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_fixed(2),
    reraise=True,
    retry=retry_if_exception_type(aiohttp.ClientError),
)
async def _convert_url_to_pdf(self, url: str) -> bytes:
    Gotenberg_URL = settings.GOTENBERG_URL
    endpoint = "/forms/chromium/convert/url"
    gotenberg_url = Gotenberg_URL + endpoint

    form = aiohttp.FormData()
    form.add_field("url", url, content_type="text/plain")

    async with aiohttp.ClientSession() as session:
        async with session.post(gotenberg_url, data=form) as response:
            if response.status != 200:
                error_message = await response.text()
                raise ValidationError(f"Failed to convert URL to PDF: {error_message}")

            pdf_data = await response.read()

    return pdf_data
```

## 5. Page 模型

**文件位置**: `web/api/models.py:685-697`

### 5.1 模型定义

```python
class Page(models.Model):
    document = models.ForeignKey(
        Document,
        on_delete=models.CASCADE,
        related_name="pages"
    )
    page_number = models.IntegerField()
    content = models.TextField(blank=True)  # 预留字段，暂未使用
    img_base64 = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

### 5.2 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `document` | ForeignKey(Document) | 所属文档（级联删除） |
| `page_number` | IntegerField | 页码（1-indexed） |
| `content` | TextField | OCR 文本内容（预留，暂未使用） |
| `img_base64` | TextField | 页面图像的 base64 编码 |
| `created_at` | DateTimeField | 创建时间 |
| `updated_at` | DateTimeField | 更新时间 |

### 5.3 设计说明

- 每个页面存储一张图像的 base64 编码
- `content` 字段预留用于未来的 OCR 功能
- 页码从 1 开始（1-indexed）
- 级联删除：删除文档时自动删除所有页面

## 6. PageEmbedding 模型

**文件位置**: `web/api/models.py:699-702`

### 6.1 模型定义

```python
class PageEmbedding(models.Model):
    page = models.ForeignKey(
        Page,
        on_delete=models.CASCADE,
        related_name="embeddings"
    )
    embedding = HalfVectorField(dimensions=128)
```

### 6.2 字段说明

| 字段 | 类型 | 维度 | 存储大小 |
|------|------|------|---------|
| `page` | ForeignKey(Page) | - | - |
| `embedding` | HalfVectorField | 128 | 256 字节 |

### 6.3 HalfVectorField 详解

**来源**: `pgvector.django.HalfVectorField`

**特点**:
- 使用半精度浮点数（float16）
- 128 维向量 = 128 × 2 字节 = 256 字节
- 比 VectorField（float32）节省 50% 存储空间
- 精度损失可接受（对于相似度搜索）

**嵌入结构**:

每个页面有多个嵌入向量（Late-Interaction 机制）：

```
Page 1
    ├── Embedding 1: [128 floats]
    ├── Embedding 2: [128 floats]
    ├── ...
    └── Embedding N: [128 floats]
```

**批量创建**（文件位置: `models.py:324-328`）:

```python
bulk_create_embeddings = [
    PageEmbedding(page=page, embedding=embedding)
    for embedding in embedding_obj["embedding"]
]
await PageEmbedding.objects.abulk_create(bulk_create_embeddings)
```

## 7. MaxSim 函数

**文件位置**: `web/api/models.py:707-709`

### 7.1 定义

```python
class MaxSim(Func):
    function = "max_sim"
    output_field = FloatField()
```

### 7.2 SQL 函数

这是一个 PostgreSQL 自定义函数，用于计算最大相似度：

```sql
CREATE OR REPLACE FUNCTION max_sim(
    page_embeddings halfvec[],
    query_embeddings text[]
)
RETURNS float
AS $$
    SELECT MAX(similarity)
    FROM (
        SELECT
            page_embedding <#> CAST(query_embedding AS halfvec) AS similarity
        FROM
            UNNEST(page_embeddings) AS page_embedding,
            UNNEST(query_embeddings) AS query_embedding
    ) AS similarities
$$
LANGUAGE SQL IMMUTABLE PARALLEL SAFE;
```

### 7.3 Late-Interaction 机制

MaxSim 实现了 Late-Interaction 搜索：

1. 每个页面有 N 个嵌入向量
2. 每个查询有 M 个嵌入向量
3. 计算所有 N×M 对之间的相似度
4. 返回最大值

**使用示例**（文件位置: `views.py:1019-1022`）:

```python
pages_query = (
    base_query
    .annotate(page_embeddings=ArrayAgg("embeddings__embedding"))
    .annotate(max_sim=MaxSim("page_embeddings", casted_query_embeddings))
    .order_by("-max_sim")[:top_k]
)
```

## 8. 工具函数

### 8.1 get_extension_from_mime

**文件位置**: `models.py:54-80`

从 MIME 类型获取文件扩展名：

```python
def get_extension_from_mime(mime_type):
    # 硬编码常见类型
    hardcode_mimetypes = {
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document": ".docx",
        "application/vnd.openxmlformats-officedocument.presentationml.presentation": ".pptx",
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet": ".xlsx",
        "application/msword": ".doc",
        "application/pdf": ".pdf",
        "image/jpeg": ".jpg",
        "image/png": ".png",
        ...
    }

    if mime_type in hardcode_mimetypes:
        return hardcode_mimetypes[mime_type]

    # 使用 mimetypes 库猜测
    extension = mimetypes.guess_extension(mime_type)
    if extension:
        return extension

    # 默认 .bin
    return ".bin"
```

## 9. 数据库迁移

**文件位置**: `web/api/migrations/`

### 9.1 关键迁移

| 迁移文件 | 说明 |
|---------|------|
| `0001_enable_pgvector.py` | 启用 pgvector 扩展 |
| `0002_initial.py` | 创建初始模型 |
| `0003_*.py` | 添加元数据字段 |
| `0010_*.py` | 从 VectorField 迁移到 HalfVectorField |
| `0015_*.py` | 添加 S3 文件字段 |
| `0020_*.py` | 添加 Webhook 相关字段 |

### 9.2 pgvector 启用

```python
# 0001_enable_pgvector.py
from django.contrib.postgres.operations import CreateExtension

class Migration(migrations.Migration):
    operations = [
        CreateExtension("vector"),
    ]
```

## 10. 性能考虑

### 10.1 索引策略

```python
# Collection
indexes = [
    models.Index(fields=["name", "owner"])
]

# Document
indexes = [
    models.Index(fields=["name", "collection"])
]
```

### 10.2 查询优化

**Select Related**:

```python
Document.objects.select_related("collection", "collection__owner")
```

**Prefetch Related**:

```python
Document.objects.prefetch_related(
    Prefetch("pages", queryset=Page.objects.order_by("page_number"))
)
```

### 10.3 批量操作

**批量创建**:

```python
await PageEmbedding.objects.abulk_create(bulk_create_embeddings)
```

**批量删除**:

```python
await document.pages.all().adelete()
```

## 11. 数据完整性

### 11.1 级联删除

```
CustomUser 删除 → 所有 Collections 删除
Collection 删除 → 所有 Documents 删除
Document 删除 → 所有 Pages 删除
Page 删除 → 所有 PageEmbeddings 删除
```

### 11.2 唯一性保证

- (name, owner) 在 Collection 中唯一
- (name, collection) 在 Document 中唯一
- token 在 CustomUser 中唯一

### 11.3 文件清理

使用 `django-cleanup` 自动清理孤立文件：

```python
INSTALLED_APPS = [
    ...
    "django_cleanup.apps.CleanupConfig",
]
```

## 总结

ColiVara 的数据库模型设计体现了以下特点：

1. **层级清晰**: User → Collection → Document → Page → PageEmbedding
2. **异步优先**: 所有操作都支持异步
3. **灵活元数据**: JSONField 支持任意结构
4. **向量优化**: HalfVectorField 节省 50% 存储
5. **完整性保护**: 级联删除和唯一性约束
6. **文件管理**: S3 存储 + 自动清理
7. **错误处理**: 事务性操作，失败时回滚
