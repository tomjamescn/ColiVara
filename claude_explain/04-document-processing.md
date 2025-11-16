# 文档处理流程详解

本文档详细分析 ColiVara 的文档处理机制，包括文档上传、格式转换、嵌入生成等核心流程。

## 1. 处理流程概览

```
用户上传文档
    │
    ▼
├─ URL 或 Base64 验证
│
    ▼
├─ 创建/更新 Document 对象
│
    ▼
├─ Base64 → S3 存储（如果提供 base64）
│
    ▼
├─ 文档预处理 (_prep_document)
│   │
│   ├─ 获取文档数据
│   │   ├─ 从 S3 读取
│   │   ├─ 从 URL 下载
│   │   └─ 从参数获取
│   │
│   ├─ 文件格式检测（python-magic）
│   │
│   ├─ 大小和格式验证
│   │
│   ├─ 格式转换
│   │   ├─ 图像 → 直接 Base64
│   │   ├─ PDF → pdf2image → Base64
│   │   ├─ Office 文档 → Gotenberg → PDF → Base64
│   │   └─ 网页 → Gotenberg (Chromium) → PDF → Base64
│   │
│   └─ 返回 base64_images[]
│
    ▼
├─ 嵌入生成 (embed_document)
│   │
│   ├─ 分批（每批 3 个页面）
│   │
│   ├─ 发送到嵌入服务（ColPali）
│   │   ├─ 重试机制（最多 3 次）
│   │   ├─ 批次间延迟（1 秒）
│   │   └─ 返回嵌入向量
│   │
│   ├─ 保存 Page 对象
│   │   ├─ page_number
│   │   └─ img_base64
│   │
│   └─ 批量创建 PageEmbedding 对象
│       └─ embedding: HalfVectorField(128)
│
    ▼
├─ 发送 Webhook 通知（如果配置）
│   ├─ upsert.success
│   └─ upsert.fail
│
    ▼
└─ 返回 DocumentOut 响应
```

## 2. 文档预处理 (_prep_document)

**文件位置**: `web/api/models.py:341-585`

### 2.1 支持的文件格式

#### 2.1.1 图像格式（直接处理）

**文件位置**: `models.py:353-360`

```python
IMAGE_EXTENSIONS = [
    "png",
    "jpg",
    "jpeg",
    "tiff",
    "bmp",
    "gif",
]
```

**处理方式**: 直接转换为 base64，无需格式转换。

#### 2.1.2 Office 文档（需转换）

```python
# Word
"doc", "docx", "docm", "dot", "dotm", "dotx"

# Excel
"xls", "xlsx", "xlsm", "xlsb", "xlt", "xltm", "xltx"

# PowerPoint
"ppt", "pptx", "pptm", "pps", "pot", "potm", "potx"
```

**处理方式**: Gotenberg (LibreOffice) → PDF → 图像

#### 2.1.3 开源办公格式

```python
# OpenDocument
"odt",  # Text
"ods",  # Spreadsheet
"odp",  # Presentation
"odg",  # Graphics
"odf",  # Formula

# Legacy formats
"fodg", "fodp", "fods", "fodt"
```

#### 2.1.4 文本和标记语言

```python
"txt", "rtf", "csv", "xml", "html", "htm", "xhtml"
```

#### 2.1.5 其他格式

```python
# Adobe
"pdf", "eps", "psd"

# 矢量图形
"svg", "wmf", "emf", "cgm", "dxf"

# 出版物
"epub", "pub"

# 更多...（总计 100+ 格式）
```

### 2.2 文档来源

#### 2.2.1 从参数获取

**文件位置**: `models.py:492-498`

```python
if document_data:
    logger.info("Document data provided.")
    # 使用 python-magic 检测 MIME 类型
    mime = magic.Magic(mime=True)
    mime_type = mime.from_buffer(document_data)
    extension = get_extension_from_mime(mime_type).lstrip(".")
    filename = f"document.{extension}"
```

**用例**: 用于 `helpers/file-to-imgbase64/` 端点。

#### 2.2.2 从 S3 获取

**文件位置**: `models.py:501-506`

```python
elif self.s3_file:
    logger.info(f"Fetching document from S3: {self.s3_file.name}")
    with self.s3_file.open("rb") as f:
        document_data = f.read()
    filename = os.path.basename(self.s3_file.name)
    logger.info(f"Document filename: {filename}")
```

**用例**: 用户上传 base64 后，文档已保存到 S3。

#### 2.2.3 从 URL 获取

**文件位置**: `models.py:508-528`

```python
elif self.url:
    content_type, filename, document_data = await self._fetch_document(use_proxy)

    if "text/html" in content_type:
        logger.info("Document is a webpage.")
        # 网页转 PDF
        document_data = await self._convert_url_to_pdf(self.url)
        logger.info("Successfully converted URL to PDF.")
        filename = f"{filename}.pdf"
    else:
        # 普通文件
        logger.info(f"Fetching document from URL: {self.url}")
        if "application/pdf" in content_type:
            extension = "pdf"
        else:
            extension = get_extension_from_mime(content_type).lstrip(".")
        name = os.path.splitext(filename)[0]
        filename = f"{name}.{extension}"
```

**特殊处理**:
- **网页** (text/html): 使用 Gotenberg Chromium 模式转换
- **普通文件**: 直接下载

### 2.3 URL 获取逻辑

**文件位置**: `models.py:587-617`

```python
async def _fetch_document(self, use_proxy: Optional[bool] = False):
    proxy = None
    if use_proxy:
        proxy = settings.PROXY_URL
        # 代理通常不支持 HTTPS
        self.url = self.url.replace("https://", "http://")
        logger.info("Using proxy to fetch document.")

    MAX_SIZE_BYTES = 50 * 1024 * 1024  # 50 MB

    async with aiohttp.ClientSession() as session:
        async with session.get(self.url, proxy=proxy) as response:
            if response.status != 200:
                raise ValidationError(
                    "Failed to fetch document from URL. "
                    "Some documents are protected by anti-scraping measures. "
                    "We recommend you download them and send us base64."
                )

            # 提取元数据
            content_type = response.headers.get("Content-Type", "").lower()
            content_disposition = response.headers.get("Content-Disposition", "")
            content_length = response.headers.get("Content-Length")

            # 检查大小
            if content_length and int(content_length) > MAX_SIZE_BYTES:
                raise ValidationError("Document exceeds maximum size of 50MB.")

            # 提取文件名
            filename_match = re.findall('filename="(.+)"', content_disposition)
            filename = (
                filename_match[0]
                if filename_match
                else os.path.basename(urllib.parse.urlparse(self.url).path)
            )

            if not filename:
                filename = "downloaded_file"

            return content_type, filename, await response.read()
```

**关键点**:
- 支持代理（`use_proxy` 参数）
- 50MB 大小限制
- 从 `Content-Disposition` 或 URL 提取文件名
- 处理反爬虫保护（返回友好错误信息）

### 2.4 文档验证

**文件位置**: `models.py:534-545`

```python
# 确保数据和文件名存在
assert document_data, "Document data should be set"
assert filename, "Filename should be set"

if not extension:
    extension = os.path.splitext(filename)[1].lstrip(".")

# 大小验证
if len(document_data) > MAX_SIZE_BYTES:
    raise ValidationError("Document exceeds maximum size of 50MB.")

# 格式验证
if extension not in ALLOWED_EXTENSIONS:
    raise ValidationError(f"File extension .{extension} is not allowed.")
```

### 2.5 格式转换

#### 2.5.1 图像处理

**文件位置**: `models.py:548-562`

```python
is_image = extension in IMAGE_EXTENSIONS

if is_image:
    logger.info("Document is an image. Converting to base64.")
    img_base64 = base64.b64encode(document_data).decode("utf-8")
    return [img_base64]
```

**流程**: 图像 → Base64 → 完成

#### 2.5.2 PDF 处理

**文件位置**: `models.py:555-572`

```python
is_pdf = extension == "pdf"

if is_pdf:
    logger.info("Document is already a PDF.")
    pdf_data = document_data

# PDF 转图像
try:
    images = convert_from_bytes(pdf_data)
except Exception:
    raise ValidationError(
        "Failed to convert PDF to images. "
        "The PDF may be corrupted, which sometimes happens with URLs. "
        "Try downloading the document and sending us the base64."
    )
logger.info(f"Successfully converted PDF to {len(images)} images.")
```

**流程**: PDF → pdf2image → PIL Images

#### 2.5.3 Office 文档处理

**文件位置**: `models.py:551-554`

```python
if not is_image and not is_pdf:
    logger.info(f"Converting document to PDF. Extension: {extension}")
    # 使用 Gotenberg 转换为 PDF
    pdf_data = await self._convert_to_pdf(document_data, filename)
```

**流程**: Office → Gotenberg (LibreOffice) → PDF → pdf2image → Images

#### 2.5.4 网页处理

**文件位置**: `models.py:512-517`

```python
if "text/html" in content_type:
    logger.info("Document is a webpage.")
    # 网页转 PDF
    document_data = await self._convert_url_to_pdf(self.url)
    logger.info("Successfully converted URL to PDF.")
    filename = f"{filename}.pdf"
```

**流程**: URL → Gotenberg (Chromium) → PDF → pdf2image → Images

### 2.6 图像转 Base64

**文件位置**: `models.py:574-583`

```python
# Step 4: Turn the images into base64 strings
base64_images = []
for image in images:
    img_io = BytesIO()
    image.save(img_io, "PNG")  # 统一保存为 PNG
    img_data = img_io.getvalue()
    img_base64 = base64.b64encode(img_data).decode("utf-8")
    base64_images.append(img_base64)

# Step 5: returning the base64 images
return base64_images
```

**注意**: 所有页面统一转换为 PNG 格式。

## 3. Gotenberg 集成

### 3.1 LibreOffice 转换

**文件位置**: `models.py:619-654`

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_fixed(2),
    reraise=True,
    retry=retry_if_exception_type(aiohttp.ClientError),
)
async def _convert_to_pdf(self, document_data: bytes, filename: str) -> bytes:
    """使用 Gotenberg LibreOffice 模块转换文档为 PDF"""
    Gotenberg_URL = settings.GOTENBERG_URL
    endpoint = "/forms/libreoffice/convert"
    url = Gotenberg_URL + endpoint

    # 准备表单数据
    form = aiohttp.FormData()
    form.add_field(
        "files",
        document_data,
        filename=filename,
    )

    headers = {
        "Accept": "application/pdf",
    }

    async with aiohttp.ClientSession() as session:
        async with session.post(url, data=form, headers=headers) as response:
            if response.status != 200:
                error_message = await response.text()
                raise ValidationError(
                    f"Failed to convert document to PDF via Gotenberg: {error_message}"
                )
            pdf_data = await response.read()

    return pdf_data
```

**支持的格式**:
- Office 文档（DOC, DOCX, XLS, XLSX, PPT, PPTX）
- OpenDocument（ODT, ODS, ODP）
- 其他 LibreOffice 支持的格式

**重试策略**:
- 最多 3 次
- 每次等待 2 秒
- 仅在网络错误时重试

### 3.2 Chromium 网页转换

**文件位置**: `models.py:656-682`

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_fixed(2),
    reraise=True,
    retry=retry_if_exception_type(aiohttp.ClientError),
)
async def _convert_url_to_pdf(self, url: str) -> bytes:
    """使用 Gotenberg Chromium 模块将网页转换为 PDF"""
    Gotenberg_URL = settings.GOTENBERG_URL
    endpoint = "/forms/chromium/convert/url"
    gotenberg_url = Gotenberg_URL + endpoint

    logger.info(f"Converting URL to PDF as a Webpage: {url}")

    # 准备表单数据
    form = aiohttp.FormData()
    form.add_field("url", url, content_type="text/plain")

    async with aiohttp.ClientSession() as session:
        async with session.post(gotenberg_url, data=form) as response:
            if response.status != 200:
                error_message = await response.text()
                raise ValidationError(
                    f"Failed to convert URL to PDF via Gotenberg: {error_message}"
                )
            pdf_data = await response.read()

    return pdf_data
```

**特点**:
- 使用 Headless Chrome 渲染
- 支持 JavaScript
- 支持 CSS 样式
- 类似于"打印页面"功能

## 4. 嵌入生成 (embed_document)

**文件位置**: `models.py:184-339`

### 4.1 配置参数

```python
EMBEDDINGS_URL = settings.EMBEDDINGS_URL
EMBEDDINGS_BATCH_SIZE = 3           # 每批 3 个页面
DELAY_BETWEEN_BATCHES = 1           # 批次间延迟 1 秒
```

### 4.2 批量发送函数

**文件位置**: `models.py:210-260`

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_fixed(5),
    reraise=True,
    retry=retry_if_exception_type(aiohttp.ClientError),
)
async def send_batch(
    session: aiohttp.ClientSession, images: List[str]
) -> List[Dict[str, Any]]:
    """发送一批图像到嵌入服务"""
    payload = {"input": {"task": "image", "input_data": images}}
    headers = {"Authorization": f"Bearer {settings.EMBEDDINGS_URL_TOKEN}"}

    async with session.post(
        EMBEDDINGS_URL, json=payload, headers=headers
    ) as response:
        if response.status != 200:
            logger.error(
                f"Failed to get embeddings from the embeddings service. "
                f"Status code: {response.status}"
            )
            raise ValidationError(
                "Failed to get embeddings from the embeddings service."
            )

        out = await response.json()

        if "output" not in out or "data" not in out["output"]:
            raise ValidationError(
                f"Failed to get embeddings from the embeddings service. "
                f"Response: {out}"
            )

        logger.info(f"Got embeddings for batch of {len(images)} images.")

    return out["output"]["data"]
```

**请求格式**:

```json
{
  "input": {
    "task": "image",
    "input_data": [
      "base64_image_1",
      "base64_image_2",
      "base64_image_3"
    ]
  }
}
```

**响应格式**:

```json
{
  "output": {
    "data": [
      {
        "embedding": [[0.1, 0.2, ..., 0.128], [0.1, 0.2, ...]],
        "index": 0,
        "object": "embedding"
      },
      ...
    ]
  }
}
```

**重试策略**:
- 最多 3 次
- 每次等待 5 秒
- 仅在网络错误时重试

### 4.3 主处理流程

**文件位置**: `models.py:262-338`

```python
async def embed_document(self, use_proxy: Optional[bool] = False) -> None:
    # 1. 准备文档（转换为 base64 图像）
    base64_images = await self._prep_document(use_proxy=use_proxy)
    logger.info(f"Successfully prepped document {self.name}")

    # 2. 分批
    batches = [
        base64_images[i : i + EMBEDDINGS_BATCH_SIZE]
        for i in range(0, len(base64_images), EMBEDDINGS_BATCH_SIZE)
    ]
    logger.info(
        f"Split document {self.name} into {len(batches)} batches for embedding"
    )

    try:
        # 3. 保存文档
        await self.asave()
        logger.info(f"Starting the process to save document {self.name}")

        # 4. 逐批处理
        async with aiohttp.ClientSession() as session:
            embedding_results = []
            for i, batch in enumerate(batches):
                # 处理批次
                batch_result = await send_batch(session, batch)
                embedding_results.extend(batch_result)

                logger.info(
                    f"Processed batch {i+1}/{len(batches)} for document {self.name}"
                )

                # 批次间延迟（避免过载嵌入服务）
                if i < len(batches) - 1:
                    logger.info(
                        f"Waiting {DELAY_BETWEEN_BATCHES} seconds before next batch"
                    )
                    await asyncio.sleep(DELAY_BETWEEN_BATCHES)

        logger.info(
            f"Successfully got embeddings for all pages in document {self.name}"
        )

        # 5. 保存页面和嵌入
        for i, embedding_obj in enumerate(embedding_results):
            # 验证嵌入格式
            assert (
                isinstance(embedding_obj["embedding"], list)
                and isinstance(embedding_obj["embedding"][0], list)
                and len(embedding_obj["embedding"][0]) == 128
            ), "Embedding is not a list of a list of 128 floats"

            # 创建页面
            page = Page(
                document=self,
                page_number=i + 1,
                img_base64=base64_images[i],
            )
            await page.asave()

            # 批量创建嵌入向量
            bulk_create_embeddings = [
                PageEmbedding(page=page, embedding=embedding)
                for embedding in embedding_obj["embedding"]
            ]
            await PageEmbedding.objects.abulk_create(bulk_create_embeddings)

            logger.info(
                f"Successfully saved page {page.page_number} in document {self.name}"
            )

        logger.info(f"Successfully saved all pages in document {self.name}")

    except Exception as e:
        # 如果出错，删除文档（级联删除页面）
        if self.pk:
            await self.adelete()
        raise ValidationError(f"Failed to save pages: {str(e)}")
```

### 4.4 嵌入向量结构

每个页面有多个嵌入向量（Late-Interaction）：

```python
Page 1 (3 tokens)
    ├── PageEmbedding 1: [128 floats]
    ├── PageEmbedding 2: [128 floats]
    └── PageEmbedding 3: [128 floats]

Page 2 (5 tokens)
    ├── PageEmbedding 1: [128 floats]
    ├── PageEmbedding 2: [128 floats]
    ├── PageEmbedding 3: [128 floats]
    ├── PageEmbedding 4: [128 floats]
    └── PageEmbedding 5: [128 floats]
```

**特点**:
- 每个页面的嵌入向量数量不固定（取决于 token 数量）
- 每个嵌入向量都是 128 维
- 使用 HalfVector（float16）存储

## 5. Base64 到 S3

**文件位置**: `models.py:143-171`

```python
async def save_base64_to_s3(self, base64_content: str) -> None:
    """将 base64 编码的文件保存到 S3"""
    try:
        # 1. 删除旧文件（如果存在）
        await self.delete_s3_file()

        # 2. 解码 base64
        file_content = base64.b64decode(base64_content)

        # 3. 检测 MIME 类型
        mime = magic.Magic(mime=True)
        mime_type = mime.from_buffer(file_content)

        # 4. 获取适当的文件扩展名
        extension = get_extension_from_mime(mime_type)

        # 5. 生成文件名
        safe_name = self.name.replace(" ", "_")
        name_without_extension, _ = os.path.splitext(safe_name)
        filename = f"{name_without_extension}{extension}"

        # 6. 保存到 S3
        await sync_to_async(self.s3_file.save)(
            filename, ContentFile(file_content), save=True
        )

    except Exception as e:
        raise ValidationError(f"Failed to save file to S3: {str(e)}")
```

**文件路径示例**:

```
生产环境: documents/user_at_example.com/research_paper.pdf
开发环境: dev-documents/user_at_example.com/research_paper.pdf
```

## 6. 错误处理和回滚

### 6.1 事务性处理

```python
try:
    await self.asave()
    # ... 处理嵌入
    # ... 保存页面
except Exception as e:
    # 如果出错，删除文档
    if self.pk:
        await self.adelete()  # 级联删除所有页面
    raise ValidationError(f"Failed to save pages: {str(e)}")
```

### 6.2 级联删除

```
Document 删除
    ↓
所有 Page 删除（CASCADE）
    ↓
所有 PageEmbedding 删除（CASCADE）
    ↓
S3 文件删除（django-cleanup）
```

### 6.3 Webhook 通知

**成功通知**（文件位置: `views.py:442-464`）:

```python
if not payload.wait and request.auth.svix_application_id:
    svix = SvixAsync(settings.SVIX_TOKEN)
    await svix.message.create(
        request.auth.svix_application_id,
        MessageIn(
            event_type="upsert.success",
            payload={
                "type": "upsert.success",
                "message": "Document upserted successfully",
                "id": document.id,
                "name": document.name,
                "metadata": document.metadata,
                "url": await document.get_url(),
                "num_pages": document.num_pages,
                "collection_name": document.collection.name,
            },
        ),
    )
```

**失败通知**（文件位置: `views.py:478-498`）:

```python
except Exception as e:
    if not payload.wait and request.auth.svix_application_id:
        svix = SvixAsync(settings.SVIX_TOKEN)
        await svix.message.create(
            request.auth.svix_application_id,
            MessageIn(
                event_type="upsert.fail",
                payload={
                    "type": "upsert.fail",
                    "message": "There was an error processing your document",
                    "name": payload.name,
                    "metadata": payload.metadata,
                    "collection_name": payload.collection_name,
                    "error": str(e),
                },
            ),
        )
```

### 6.4 邮件通知

**文件位置**: `views.py:499-516`

```python
elif not payload.wait:  # 异步模式且没有 Webhook
    user_email = request.auth.email
    admin_email = settings.ADMINS[0][1]
    from_email = settings.DEFAULT_FROM_EMAIL

    email = EmailMessage(
        subject="Document Upsertion Failed",
        body=f"There was an error processing your document: {str(e)}",
        to=[user_email],
        bcc=[admin_email],
        from_email=from_email,
    )
    email.content_subtype = "html"
    email.send()
```

## 7. 性能考虑

### 7.1 批量处理

```python
# 批量创建嵌入（而非逐个创建）
bulk_create_embeddings = [
    PageEmbedding(page=page, embedding=embedding)
    for embedding in embedding_obj["embedding"]
]
await PageEmbedding.objects.abulk_create(bulk_create_embeddings)
```

### 7.2 批次控制

```python
EMBEDDINGS_BATCH_SIZE = 3           # 平衡并发和负载
DELAY_BETWEEN_BATCHES = 1           # 避免过载嵌入服务
```

**权衡**:
- **批次太大**: 嵌入服务可能超时或内存不足
- **批次太小**: 处理时间过长（网络开销）
- **无延迟**: 可能过载嵌入服务
- **延迟太长**: 总处理时间过长

### 7.3 异步处理

```python
# 异步模式（默认）
if not payload.wait:
    asyncio.create_task(process_upsert_document(request, payload))
    return 202, "Processing in background"

# 同步模式
if payload.wait:
    return await process_upsert_document(request, payload)
```

**推荐**: 使用异步模式（`wait=false`），通过 Webhook 获取结果。

### 7.4 平均处理时间

根据文档说明（`views.py:530`）:

```
Average latency: 7 seconds per page
```

**示例**:
- 10 页文档 ≈ 70 秒
- 50 页文档 ≈ 350 秒（约 6 分钟）
- 100 页文档 ≈ 700 秒（约 12 分钟）

## 8. 限制和约束

### 8.1 文件大小

```python
MAX_SIZE_BYTES = 50 * 1024 * 1024  # 50 MB
```

### 8.2 文件格式

支持 100+ 格式，但不包括：
- 视频文件
- 音频文件
- 压缩文件（ZIP, RAR）
- 可执行文件

### 8.3 URL 限制

- 必须是有效的 HTTP/HTTPS URL
- 需要公开访问（或通过代理）
- 可能被反爬虫保护阻止

### 8.4 Base64 限制

- 必须是有效的 base64 编码
- 解码后不超过 50MB

## 9. 最佳实践

### 9.1 选择上传方式

| 场景 | 推荐方式 | 原因 |
|------|---------|------|
| 本地文件 | Base64 | 避免上传到临时服务器 |
| 公开 URL | URL | 节省带宽 |
| 需要爬虫的网页 | URL + proxy | 绕过反爬虫 |
| 大文件 | URL | 避免 base64 编码开销 |

### 9.2 元数据设计

```json
{
  "author": "John Doe",
  "year": 2024,
  "category": "research",
  "tags": ["machine-learning", "nlp"],
  "reviewed": true,
  "version": "1.0"
}
```

**建议**:
- 使用扁平结构（便于查询）
- 使用标准类型（string, number, boolean）
- 避免嵌套过深

### 9.3 集合组织

```
用户文档结构:
├── research_papers/
│   ├── ml_paper_1
│   ├── ml_paper_2
│   └── nlp_paper_1
├── contracts/
│   ├── contract_2024_01
│   └── contract_2024_02
└── receipts/
    ├── receipt_jan
    └── receipt_feb
```

### 9.4 错误处理

```python
# 推荐: 异步模式 + Webhook
{
  "wait": false,
  "url": "https://example.com/doc.pdf"
}

# Webhook 端点处理
@app.post("/webhooks/colivara")
def handle_webhook(event):
    if event["type"] == "upsert.success":
        # 处理成功
        process_document(event["id"])
    elif event["type"] == "upsert.fail":
        # 处理失败
        log_error(event["error"])
```

## 总结

ColiVara 的文档处理流程展现了以下特点：

1. **格式通用性**: 支持 100+ 文件格式
2. **灵活输入**: URL、Base64、S3 三种来源
3. **智能转换**: 自动检测格式并选择合适的转换器
4. **批量优化**: 批次处理和延迟控制
5. **错误恢复**: 事务性处理和级联删除
6. **异步支持**: 后台处理 + Webhook 通知
7. **重试机制**: 网络错误自动重试
8. **日志完善**: 每个步骤都有详细日志
