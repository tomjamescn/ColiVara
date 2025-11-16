# API 端点详解

本文档详细分析 ColiVara 的所有 API 端点，位于 `web/api/views.py`（1616 行代码）。

## 1. API 框架

### 1.1 Django Ninja

ColiVara 使用 **Django Ninja** 作为 API 框架，而非传统的 Django REST Framework (DRF)。

**优势**:
- FastAPI 风格的语法
- 基于 Pydantic 的自动验证
- 自动生成 OpenAPI 文档
- 更好的性能（基于类型提示）
- 异步支持

**路由器定义**（文件位置: `views.py:29`）:

```python
from ninja import Router

router = Router()
```

### 1.2 响应模式

所有端点使用类型化的响应：

```python
@router.post(
    "/endpoint/",
    response={
        200: SuccessSchema,
        400: GenericError,
        404: GenericError,
    }
)
async def endpoint(...) -> Tuple[int, Union[SuccessSchema, GenericError]]:
    return 200, SuccessSchema(...)
```

## 2. 认证系统

### 2.1 Bearer Token 认证

**文件位置**: `views.py:43-68`

```python
class Request(HttpRequest):
    auth: CustomUser  # 类型提示

class Bearer(HttpBearer):
    async def authenticate(
        self, request: HttpRequest, token: str
    ) -> Optional[CustomUser]:
        try:
            user = await CustomUser.objects.aget(token=token)
            return user
        except CustomUser.DoesNotExist:
            return None
```

**使用方式**:

```python
@router.post("/endpoint/", auth=Bearer())
async def endpoint(request: Request):
    user = request.auth  # 已认证的用户
```

**请求头格式**:

```
Authorization: Bearer <your-token-here>
```

## 3. 健康检查

### 3.1 GET /health/

**文件位置**: `views.py:35-37`

最简单的端点，用于健康检查：

```python
@router.get("/health/", tags=["health"])
async def health(request) -> Dict[str, str]:
    return {"status": "ok"}
```

**特点**:
- 无需认证
- 用于负载均衡器健康检查
- 用于监控系统

**响应**:

```json
{
  "status": "ok"
}
```

## 4. 集合管理 API

### 4.1 数据模式

**输入模式**（文件位置: `views.py:74-98`）:

```python
class CollectionIn(Schema):
    name: str
    metadata: Optional[dict] = Field(default_factory=lambda: {})

    @model_validator(mode="after")
    def validate_name(self) -> Self:
        if self.name.lower() == "all":
            raise ValueError("Collection name 'all' is not allowed.")
        return self

class PatchCollectionIn(Schema):
    name: Optional[str] = None
    metadata: Optional[dict] = None

    @model_validator(mode="after")
    def validate_name(self) -> Self:
        if self.name and self.name.lower() == "all":
            raise ValueError("Collection name 'all' is not allowed.")
        if not any([self.name, self.metadata]):
            raise ValueError("At least one field must be provided to update.")
        return self
```

**输出模式**（文件位置: `views.py:101-105`）:

```python
class CollectionOut(Schema):
    id: int
    name: str
    metadata: dict
    num_documents: int
```

### 4.2 POST /collections/

**文件位置**: `views.py:116-153`

创建新集合：

```python
@router.post(
    "/collections/",
    tags=["collections"],
    auth=Bearer(),
    response={201: CollectionOut, 409: GenericError},
)
async def create_collection(
    request: Request, payload: CollectionIn
) -> Tuple[int, CollectionOut] | Tuple[int, GenericError]:
```

**请求示例**:

```json
{
  "name": "my_collection",
  "metadata": {
    "description": "My document collection",
    "category": "research"
  }
}
```

**响应（201）**:

```json
{
  "id": 1,
  "name": "my_collection",
  "metadata": {
    "description": "My document collection",
    "category": "research"
  },
  "num_documents": 0
}
```

**错误（409）**:

```json
{
  "detail": "You already have a collection with this name, did you mean to update it? Use PATCH instead."
}
```

**实现逻辑**:

```python
try:
    collection = await Collection.objects.acreate(
        name=payload.name,
        owner=request.auth,
        metadata=payload.metadata
    )
    return 201, CollectionOut(
        id=collection.id,
        name=collection.name,
        metadata=collection.metadata,
        num_documents=0,
    )
except IntegrityError:
    return 409, GenericError(detail="You already have a collection with this name...")
```

### 4.3 GET /collections/

**文件位置**: `views.py:156-183`

列出所有集合：

```python
@router.get(
    "/collections/",
    response=List[CollectionOut],
    tags=["collections"],
    auth=Bearer()
)
async def list_collections(request: Request) -> List[CollectionOut]:
```

**响应**:

```json
[
  {
    "id": 1,
    "name": "collection1",
    "metadata": {},
    "num_documents": 5
  },
  {
    "id": 2,
    "name": "collection2",
    "metadata": {},
    "num_documents": 3
  }
]
```

**实现逻辑**:

```python
collections = [
    CollectionOut(
        id=c.id,
        name=c.name,
        metadata=c.metadata,
        num_documents=await c.document_count(),
    )
    async for c in Collection.objects.filter(owner=request.auth)
]
return collections
```

### 4.4 GET /collections/{collection_name}/

**文件位置**: `views.py:186-228`

获取单个集合详情：

```python
@router.get(
    "/collections/{collection_name}/",
    response={200: CollectionOut, 404: GenericError},
    tags=["collections"],
    auth=Bearer(),
)
async def get_collection(
    request: Request, collection_name: str
) -> Tuple[int, CollectionOut] | Tuple[int, GenericError]:
```

**请求**: `GET /collections/my_collection/`

**响应（200）**:

```json
{
  "id": 1,
  "name": "my_collection",
  "metadata": {},
  "num_documents": 5
}
```

**错误（404）**:

```json
{
  "detail": "Collection: my_collection doesn't exist"
}
```

### 4.5 PATCH /collections/{collection_name}/

**文件位置**: `views.py:231-274`

部分更新集合：

```python
@router.patch(
    "/collections/{collection_name}/",
    tags=["collections"],
    auth=Bearer(),
    response={200: CollectionOut, 404: GenericError},
)
async def partial_update_collection(
    request: Request, collection_name: str, payload: PatchCollectionIn
) -> Tuple[int, CollectionOut] | Tuple[int, GenericError]:
```

**请求示例**:

```json
{
  "name": "renamed_collection",
  "metadata": {
    "updated": true
  }
}
```

**实现逻辑**:

```python
collection = await Collection.objects.aget(
    name=collection_name, owner=request.auth
)

collection.name = payload.name or collection.name

if payload.metadata is not None:
    collection.metadata = payload.metadata

await collection.asave()
```

**注意**:
- `metadata: null` → 保持不变
- `metadata: {}` → 清空元数据
- `metadata: {...}` → 替换元数据

### 4.6 DELETE /collections/{collection_name}/

**文件位置**: `views.py:277-310`

删除集合（级联删除所有文档）：

```python
@router.delete(
    "/collections/{collection_name}/",
    tags=["collections"],
    auth=Bearer(),
    response={204: None, 404: GenericError},
)
async def delete_collection(
    request: Request, collection_name: str
) -> Tuple[int, None] | Tuple[int, GenericError]:
```

**响应（204）**: 无内容

**错误（404）**:

```json
{
  "detail": "Collection: my_collection doesn't exist"
}
```

## 5. 文档管理 API

### 5.1 数据模式

**输入模式**（文件位置: `views.py:316-392`）:

```python
class DocumentIn(Schema):
    name: str
    metadata: dict = Field(default_factory=dict)
    collection_name: str = "default_collection"
    url: Optional[str] = None
    base64: Optional[str] = None
    wait: Optional[bool] = False
    use_proxy: Optional[bool] = False

    @model_validator(mode="after")
    def base64_or_url(self) -> Self:
        if not self.url and not self.base64:
            raise ValueError("Either 'url' or 'base64' must be provided.")
        if self.url and self.base64:
            raise ValueError("Only one of 'url' or 'base64' should be provided.")
        # 验证 base64 和 URL 格式
        return self

class DocumentInPatch(Schema):
    name: Optional[str] = None
    metadata: Optional[dict] = Field(default_factory=lambda: {})
    collection_name: Optional[str] = "default_collection"
    url: Optional[str] = None
    base64: Optional[str] = None
    use_proxy: Optional[bool] = False

    @model_validator(mode="after")
    def at_least_one_field(self) -> Self:
        if not any([self.name, self.metadata, self.url, self.base64]):
            raise ValueError("At least one field must be provided to update.")
        return self
```

**输出模式**（文件位置: `views.py:365-372`）:

```python
class DocumentOut(Schema):
    id: int
    name: str
    metadata: dict = Field(default_factory=dict)
    url: Optional[str] = None
    num_pages: int
    collection_name: str
    pages: Optional[List[PageOut]] = None

class PageOut(Schema):
    document_name: Optional[str] = None
    img_base64: str
    page_number: int
```

### 5.2 POST /documents/upsert-document/

**文件位置**: `views.py:520-570`

创建或更新文档（Upsert 操作）：

```python
@router.post(
    "/documents/upsert-document/",
    tags=["documents"],
    auth=Bearer(),
    response={201: DocumentOut, 202: GenericMessage, 400: GenericError},
)
async def upsert_document(
    request: Request, payload: DocumentIn
) -> Tuple[int, DocumentOut] | Tuple[int, GenericError] | Tuple[int, GenericMessage]:
```

**请求示例（URL）**:

```json
{
  "name": "research_paper",
  "collection_name": "my_collection",
  "url": "https://example.com/paper.pdf",
  "metadata": {
    "author": "John Doe",
    "year": 2024
  },
  "wait": false
}
```

**请求示例（Base64）**:

```json
{
  "name": "contract",
  "collection_name": "legal",
  "base64": "JVBERi0xLjQKJeLjz9MKMSAwIG9iago8PC...",
  "metadata": {
    "type": "contract",
    "signed": true
  },
  "wait": true
}
```

**参数说明**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | ✓ | 文档名称 |
| `collection_name` | string | - | 集合名称（默认: default_collection） |
| `url` | string | * | 文档 URL |
| `base64` | string | * | Base64 编码的文档 |
| `metadata` | object | - | 自定义元数据 |
| `wait` | boolean | - | 是否等待处理完成（默认: false） |
| `use_proxy` | boolean | - | 是否使用代理获取 URL（默认: false） |

\* `url` 和 `base64` 必须提供一个

**响应（wait=true, 201）**:

```json
{
  "id": 1,
  "name": "research_paper",
  "url": "https://s3.amazonaws.com/bucket/documents/...",
  "metadata": {
    "author": "John Doe",
    "year": 2024
  },
  "num_pages": 10,
  "collection_name": "my_collection"
}
```

**响应（wait=false, 202）**:

```json
{
  "detail": "Document is being processed in the background."
}
```

**实现逻辑**:

```python
if payload.collection_name == "all":
    return 400, GenericError(
        detail="The collection name 'all' is not allowed when inserting..."
    )

if payload.wait:
    # 同步处理
    return await process_upsert_document(request, payload)
else:
    # 异步处理
    asyncio.create_task(process_upsert_document(request, payload))
    return 202, GenericMessage(
        detail="Document is being processed in the background."
    )
```

### 5.3 process_upsert_document（内部函数）

**文件位置**: `views.py:395-517`

文档上传的核心处理逻辑：

```python
async def process_upsert_document(
    request: Request, payload: DocumentIn
) -> Tuple[int, DocumentOut] | Tuple[int, GenericError]:
```

**处理流程**:

```python
# 1. 获取或创建集合
collection, _ = await Collection.objects.aget_or_create(
    name=payload.collection_name, owner=request.auth
)

# 2. 检查文档是否存在
exists = await Document.objects.filter(
    name=payload.name, collection=collection
).aexists()

if exists:
    # 更新现有文档
    document = await Document.objects.aget(
        name=payload.name, collection=collection
    )
    document.metadata = payload.metadata
    document.url = payload.url or ""
    await document.pages.all().adelete()  # 删除旧页面
else:
    # 创建新文档
    document = Document(
        name=payload.name,
        metadata=payload.metadata,
        collection=collection,
        url=payload.url or "",
    )

# 3. 如果提供了 base64，保存到 S3
if payload.base64:
    await document.save_base64_to_s3(payload.base64)

# 4. 嵌入文档（核心步骤）
await document.embed_document(payload.use_proxy)

# 5. 重新获取文档（包含页面计数）
document = (
    await Document.objects.select_related("collection")
    .annotate(num_pages=Count("pages"))
    .aget(id=document.id)
)

# 6. 发送 Webhook（如果配置）
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
                ...
            },
        ),
    )

return 201, DocumentOut(...)
```

**错误处理**:

```python
except Exception as e:
    logger.error(f"Error processing document: {str(e)}")

    # 发送失败 Webhook
    if not payload.wait and request.auth.svix_application_id:
        await svix.message.create(
            request.auth.svix_application_id,
            MessageIn(
                event_type="upsert.fail",
                payload={
                    "type": "upsert.fail",
                    "message": "There was an error processing your document",
                    "error": str(e),
                    ...
                },
            ),
        )

    # 发送失败邮件（如果没有 Webhook）
    elif not payload.wait:
        email = EmailMessage(
            subject="Document Upsertion Failed",
            body=f"There was an error processing your document: {str(e)}",
            to=[request.auth.email],
            bcc=[settings.ADMINS[0][1]],
            from_email=settings.DEFAULT_FROM_EMAIL,
        )
        email.send()

    return 400, GenericError(detail=str(e))
```

### 5.4 GET /documents/{document_name}/

**文件位置**: `views.py:573-647`

获取单个文档详情：

```python
@router.get(
    "documents/{document_name}/",
    tags=["documents"],
    auth=Bearer(),
    response={200: DocumentOut, 404: GenericError, 409: GenericError},
)
async def get_document(
    request: Request,
    document_name: str,
    collection_name: Optional[str] = "default_collection",
    expand: Optional[str] = None,
) -> Tuple[int, DocumentOut] | Tuple[int, GenericError]:
```

**请求示例**:

```
GET /documents/research_paper/?collection_name=my_collection&expand=pages
```

**参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `document_name` | string | 文档名称（路径参数） |
| `collection_name` | string | 集合名称（默认: default_collection） |
| `expand` | string | 扩展字段（逗号分隔，如: "pages"） |

**响应（expand=pages）**:

```json
{
  "id": 1,
  "name": "research_paper",
  "url": "https://...",
  "metadata": {},
  "num_pages": 10,
  "collection_name": "my_collection",
  "pages": [
    {
      "document_name": "research_paper",
      "page_number": 1,
      "img_base64": "iVBORw0KGgoAAAANSUhEUgAA..."
    },
    {
      "document_name": "research_paper",
      "page_number": 2,
      "img_base64": "iVBORw0KGgoAAAANSUhEUgAA..."
    }
  ]
}
```

**特殊用法**: `collection_name="all"` 查询所有集合

```python
if collection_name == "all":
    document = await query.aget(
        name=document_name, collection__owner=request.auth
    )
```

**错误（409）**:

```json
{
  "detail": "Multiple documents with the name: research_paper exist in your collections. Please specify a collection."
}
```

### 5.5 GET /documents/

**文件位置**: `views.py:650-723`

列出文档：

```python
@router.get(
    "/documents/",
    tags=["documents"],
    auth=Bearer(),
    response=List[DocumentOut],
)
async def list_documents(
    request: Request,
    collection_name: Optional[str] = "default_collection",
    expand: Optional[str] = None,
) -> List[DocumentOut]:
```

**请求示例**:

```
GET /documents/?collection_name=all&expand=pages
```

**响应**:

```json
[
  {
    "id": 1,
    "name": "doc1",
    "url": "https://...",
    "metadata": {},
    "num_pages": 5,
    "collection_name": "collection1",
    "pages": [...]
  },
  {
    "id": 2,
    "name": "doc2",
    "url": "https://...",
    "metadata": {},
    "num_pages": 3,
    "collection_name": "collection2",
    "pages": [...]
  }
]
```

**性能优化**:

```python
query = Document.objects.select_related("collection").annotate(
    num_pages=Count("pages")
)

if "pages" in expand_fields:
    # 预加载页面（减少查询次数）
    query = query.prefetch_related(
        Prefetch("pages", queryset=Page.objects.order_by("page_number"))
    )
```

### 5.6 PATCH /documents/{document_name}/

**文件位置**: `views.py:726-806`

部分更新文档：

```python
@router.patch(
    "documents/{document_name}/",
    tags=["documents"],
    auth=Bearer(),
    response={200: DocumentOut, 404: GenericError, 409: GenericError},
)
async def partial_update_document(
    request: Request, document_name: str, payload: DocumentInPatch
) -> Tuple[int, DocumentOut] | Tuple[int, GenericError]:
```

**请求示例**:

```json
{
  "metadata": {
    "reviewed": true,
    "reviewer": "Jane Doe"
  }
}
```

**实现逻辑**:

```python
if payload.url and payload.url != document.url:
    # URL 变化 → 重新嵌入
    document.url = payload.url
    document.metadata = payload.metadata or document.metadata
    document.name = payload.name or document.name
    await document.pages.all().adelete()
    await document.embed_document(payload.use_proxy)

elif payload.base64:
    # 提供新 base64 → 重新嵌入
    document.metadata = payload.metadata or document.metadata
    document.name = payload.name or document.name
    await document.save_base64_to_s3(payload.base64)
    await document.pages.all().adelete()
    await document.embed_document(payload.use_proxy)

else:
    # 仅更新元数据/名称
    document.name = payload.name or document.name
    document.metadata = payload.metadata or document.metadata
    await document.asave()
```

### 5.7 DELETE /documents/delete-document/{document_name}/

**文件位置**: `views.py:809-861`

删除文档：

```python
@router.delete(
    "/documents/delete-document/{document_name}/",
    tags=["documents"],
    auth=Bearer(),
    response={204: None, 404: GenericError, 409: GenericError},
)
async def delete_document(
    request: Request,
    document_name: str,
    collection_name: Optional[str] = "default_collection",
) -> Tuple[int, None] | Tuple[int, GenericError]:
```

**请求**:

```
DELETE /documents/delete-document/research_paper/?collection_name=my_collection
```

**响应（204）**: 无内容

## 6. 搜索 API

### 6.1 数据模式

**查询过滤器**（文件位置: `views.py:867-908`）:

```python
class QueryFilter(Schema):
    class onEnum(str, Enum):
        document = "document"
        collection = "collection"

    class lookupEnum(str, Enum):
        key_lookup = "key_lookup"
        contains = "contains"
        contained_by = "contained_by"
        has_key = "has_key"
        has_keys = "has_keys"
        has_any_keys = "has_any_keys"

    on: onEnum = onEnum.document
    key: Union[str, List[str]]
    value: Optional[Union[str, int, float, bool]] = None
    lookup: lookupEnum = lookupEnum.key_lookup
```

**查询输入**（文件位置: `views.py:911-936`）:

```python
class QueryIn(Schema):
    query: str
    collection_name: Optional[str] = "all"
    top_k: Optional[int] = 3
    query_filter: Optional[QueryFilter] = None

class SearchImageIn(Schema):
    img_base64: str
    collection_name: Optional[str] = "all"
    top_k: Optional[int] = 3
    query_filter: Optional[QueryFilter] = None
```

**查询输出**（文件位置: `views.py:939-958`）:

```python
class PageOutQuery(Schema):
    collection_name: str
    collection_id: int
    collection_metadata: Optional[dict] = {}
    document_name: str
    document_id: int
    document_metadata: Optional[dict] = {}
    page_number: int
    raw_score: float
    normalized_score: float
    img_base64: str

class QueryOut(Schema):
    query: str
    results: List[PageOutQuery]

class SearchImageOut(Schema):
    results: List[PageOutQuery]
```

### 6.2 POST /search/

**文件位置**: `views.py:961-1062`

文本搜索端点：

```python
@router.post(
    "/search/",
    tags=["search"],
    auth=Bearer(),
    response={200: QueryOut, 503: GenericError},
)
async def search(
    request: Request, payload: QueryIn
) -> Tuple[int, QueryOut] | Tuple[int, GenericError]:
```

**请求示例**:

```json
{
  "query": "machine learning algorithms",
  "collection_name": "all",
  "top_k": 5,
  "query_filter": {
    "on": "document",
    "key": "year",
    "value": 2024,
    "lookup": "key_lookup"
  }
}
```

**响应**:

```json
{
  "query": "machine learning algorithms",
  "results": [
    {
      "collection_name": "research",
      "collection_id": 1,
      "collection_metadata": {},
      "document_name": "ml_paper",
      "document_id": 5,
      "document_metadata": {"year": 2024},
      "page_number": 3,
      "raw_score": 0.8542,
      "normalized_score": 0.7123,
      "img_base64": "iVBORw0KGgoAAAANSUhEUgAA..."
    }
  ]
}
```

**实现逻辑**:

```python
# 1. 获取查询嵌入
query_embeddings = await get_query_embeddings(payload.query)
if not query_embeddings:
    return 503, GenericError(detail="Failed to get embeddings")

query_length = len(query_embeddings)  # 用于归一化

# 2. 转换为 HalfVector
casted_query_embeddings = [
    HalfVector(embedding).to_text() for embedding in query_embeddings
]

# 3. 构建基础查询（包含过滤）
base_query = await filter_query(payload, request.auth)

# 4. 添加最大相似度计算
pages_query = (
    base_query
    .annotate(page_embeddings=ArrayAgg("embeddings__embedding"))
    .annotate(max_sim=MaxSim("page_embeddings", casted_query_embeddings))
    .order_by("-max_sim")[: payload.top_k or 3]
)

# 5. 执行查询
results = pages_query.values(
    "id",
    "page_number",
    "img_base64",
    "document__id",
    "document__name",
    "document__metadata",
    "document__collection__id",
    "document__collection__name",
    "document__collection__metadata",
    "max_sim",
)

# 6. 格式化结果（包含归一化得分）
formatted_results = [
    PageOutQuery(
        collection_name=row["document__collection__name"],
        ...
        raw_score=row["max_sim"],
        normalized_score=row["max_sim"] / query_length,
        ...
    )
    async for row in results
]

return 200, QueryOut(query=payload.query, results=formatted_results)
```

### 6.3 POST /search-image/

**文件位置**: `views.py:1065-1166`

图像搜索端点：

```python
@router.post(
    "/search-image/",
    tags=["search"],
    auth=Bearer(),
    response={200: SearchImageOut, 503: GenericError},
)
async def search_image(
    request: Request, payload: SearchImageIn
) -> Tuple[int, SearchImageOut] | Tuple[int, GenericError]:
```

**请求示例**:

```json
{
  "img_base64": "iVBORw0KGgoAAAANSUhEUgAA...",
  "collection_name": "my_collection",
  "top_k": 3,
  "query_filter": {
    "on": "document",
    "key": "type",
    "value": "diagram",
    "lookup": "contains"
  }
}
```

**实现逻辑**: 与文本搜索类似，但使用 `get_image_embeddings()` 而非 `get_query_embeddings()`。

### 6.4 POST /filter/

**文件位置**: `views.py:1169-1249`

元数据过滤端点：

```python
@router.post(
    "/filter/",
    tags=["filter"],
    auth=Bearer(),
    response={200: Union[List[DocumentOut], List[CollectionOut]], 503: GenericError},
)
async def filter(
    request: Request,
    payload: QueryFilter,
    expand: Optional[str] = None,
) -> Union[
    Tuple[int, List[DocumentOut]],
    Tuple[int, List[CollectionOut]],
    Tuple[int, GenericError],
]:
```

**请求示例（文档过滤）**:

```json
{
  "on": "document",
  "key": "author",
  "value": "John Doe",
  "lookup": "contains"
}
```

**请求示例（集合过滤）**:

```json
{
  "on": "collection",
  "key": "category",
  "value": "research",
  "lookup": "key_lookup"
}
```

**lookup 类型说明**:

| Lookup | 说明 | 示例 |
|--------|------|------|
| `key_lookup` | 精确匹配 | `metadata.author = "John Doe"` |
| `contains` | 包含 | `metadata contains {"author": "John"}` |
| `contained_by` | 被包含 | `metadata contained by {...}` |
| `has_key` | 键存在 | `metadata has key "author"` |
| `has_keys` | 多键存在 | `metadata has keys ["author", "year"]` |
| `has_any_keys` | 任意键存在 | `metadata has any of ["author", "editor"]` |

## 7. 辅助函数

### 7.1 get_query_embeddings

**文件位置**: `views.py:1252-1274`

获取文本查询的嵌入向量：

```python
async def get_query_embeddings(query: str) -> List:
    EMBEDDINGS_URL = settings.ALWAYS_ON_EMBEDDINGS_URL
    embed_token = settings.EMBEDDINGS_URL_TOKEN
    headers = {"Authorization": f"Bearer {embed_token}"}
    payload = {
        "input": {
            "task": "query",
            "input_data": [query],
        }
    }
    async with aiohttp.ClientSession() as session:
        async with session.post(
            EMBEDDINGS_URL, json=payload, headers=headers
        ) as response:
            if response.status != 200:
                logger.error(f"Failed to get embeddings: {response.status}")
                return []
            out = await response.json()
            return out["output"]["data"][0]["embedding"]
```

### 7.2 get_image_embeddings

**文件位置**: `views.py:1277-1299`

获取图像的嵌入向量：

```python
async def get_image_embeddings(img_base64: str) -> List:
    # 与 get_query_embeddings 类似，但 task="image"
    payload = {
        "input": {
            "task": "image",
            "input_data": [img_base64],
        }
    }
    ...
```

### 7.3 filter_query

**文件位置**: `views.py:1302-1334`

构建带过滤的页面查询：

```python
async def filter_query(
    payload: Union[QueryIn, SearchImageIn], user: CustomUser
) -> QuerySet[Page]:
    base_query = Page.objects.select_related("document__collection")

    # 集合过滤
    if payload.collection_name == "all":
        base_query = base_query.filter(document__collection__owner=user)
    else:
        base_query = base_query.filter(
            document__collection__owner=user,
            document__collection__name=payload.collection_name,
        )

    # 元数据过滤
    if payload.query_filter:
        on = payload.query_filter.on
        key = payload.query_filter.key
        value = payload.query_filter.value
        lookup = payload.query_filter.lookup

        field_prefix = (
            "document__collection__metadata"
            if on == "collection"
            else "document__metadata"
        )

        lookup_operations = {
            "key_lookup": lambda k, v: {f"{field_prefix}__{k}": v},
            "contains": lambda k, v: {f"{field_prefix}__contains": {k: v}},
            "contained_by": lambda k, v: {f"{field_prefix}__contained_by": {k: v}},
            "has_key": lambda k, _: {f"{field_prefix}__has_key": k},
            "has_keys": lambda k, _: {f"{field_prefix}__has_keys": k},
            "has_any_keys": lambda k, _: {f"{field_prefix}__has_any_keys": k},
        }

        filter_params = lookup_operations[lookup](key, value)
        base_query = base_query.filter(**filter_params)

    return base_query
```

### 7.4 filter_documents

**文件位置**: `views.py:1337-1361`

过滤文档（用于 /filter/ 端点）：

```python
async def filter_documents(
    query_filter: QueryFilter, user: CustomUser
) -> QuerySet[Document]:
    base_query = Document.objects.select_related("collection")
    base_query = base_query.filter(collection__owner=user)

    field_prefix = "metadata"
    # ... 类似的 lookup_operations 逻辑

    return base_query
```

### 7.5 filter_collections

**文件位置**: `views.py:1364-1387`

过滤集合（用于 /filter/ 端点）：

```python
async def filter_collections(
    query_filter: QueryFilter, user: CustomUser
) -> QuerySet[Collection]:
    base_query = Collection.objects.all().filter(owner=user)

    field_prefix = "metadata"
    # ... 类似的 lookup_operations 逻辑

    return base_query
```

## 8. 工具 API

### 8.1 POST /helpers/file-to-imgbase64/

**文件位置**: `views.py:1398-1421`

将文件转换为图像 base64（用于测试和调试）：

```python
@router.post(
    "helpers/file-to-imgbase64/",
    tags=["helpers"],
    response=List[FileOut],
    auth=Bearer(),
)
async def file_to_imgbase64(request, file: UploadedFile = File(...)) -> List[FileOut]:
    document_data = file.read()
    document = Document()
    img_base64 = await document._prep_document(document_data)

    results = []
    for i, img in enumerate(img_base64):
        results.append(FileOut(img_base64=img, page_number=i + 1))

    return results
```

**响应**:

```json
[
  {
    "page_number": 1,
    "img_base64": "iVBORw0KGgoAAAANSUhEUgAA..."
  },
  {
    "page_number": 2,
    "img_base64": "iVBORw0KGgoAAAANSUhEUgAA..."
  }
]
```

### 8.2 POST /helpers/file-to-base64/

**文件位置**: `views.py:1424-1437`

将文件转换为 base64：

```python
@router.post("helpers/file-to-base64/", tags=["helpers"], auth=Bearer())
async def file_to_base64(request, file: UploadedFile = File(...)) -> Dict[str, str]:
    document_data = file.read()
    return {"data": base64.b64encode(document_data).decode()}
```

## 9. 嵌入 API

### 9.1 POST /embeddings/

**文件位置**: `views.py:1478-1525`

直接调用嵌入服务：

```python
@router.post(
    "/embeddings/",
    tags=["embeddings"],
    auth=Bearer(),
    response={200: EmbeddingsOut, 503: GenericError},
)
async def embeddings(
    request: Request, payload: EmbeddingsIn
) -> Tuple[int, EmbeddingsOut] | Tuple[int, GenericError]:
```

**请求示例（文本）**:

```json
{
  "task": "query",
  "input_data": ["machine learning", "neural networks"]
}
```

**请求示例（图像）**:

```json
{
  "task": "image",
  "input_data": ["iVBORw0KGgoAAAANSUhEUgAA...", "iVBORw0KGgoAAAANSUhEUgAA..."]
}
```

**响应**:

```json
{
  "_object": "list",
  "data": [
    {
      "object": "embedding",
      "index": 0,
      "embedding": [[0.1, 0.2, ..., 0.128]]
    },
    {
      "object": "embedding",
      "index": 1,
      "embedding": [[0.1, 0.2, ..., 0.128]]
    }
  ],
  "model": "vidore/colpali-v1.2",
  "usage": {
    "prompt_tokens": 2,
    "total_tokens": 2
  }
}
```

## 10. Webhook API

### 10.1 POST /webhook/

**文件位置**: `views.py:1541-1616`

配置 Webhook 端点：

```python
@router.post(
    "/webhook/",
    auth=Bearer(),
    tags=["webhook"],
    response={200: WebhookOut, 400: GenericError},
)
async def add_webhook(
    request: Request, payload: WebhookIn
) -> Tuple[int, GenericError] | Tuple[int, WebhookOut]:
```

**请求**:

```json
{
  "url": "https://myapp.com/webhooks/colivara"
}
```

**响应**:

```json
{
  "app_id": "app_2abc...",
  "endpoint_id": "ep_2abc...",
  "webhook_secret": "whsec_abc123..."
}
```

**实现逻辑**:

```python
svix = SvixAsync(settings.SVIX_TOKEN)

# 创建或获取应用
if not request.auth.svix_application_id:
    app = await svix.application.create(ApplicationIn(name=request.auth.email))
    app_id = app.id
else:
    app_id = request.auth.svix_application_id

# 创建或更新端点
if request.auth.svix_endpoint_id:
    # 更新现有端点
    endpoint_out = await svix.endpoint.update(
        app_id,
        request.auth.svix_endpoint_id,
        EndpointUpdate(url=payload.url),
    )
else:
    # 创建新端点
    endpoint_out = await svix.endpoint.create(
        app_id,
        EndpointIn(
            url=payload.url,
            version=1,
            description="User webhook",
        ),
    )

# 保存到用户模型
request.auth.svix_application_id = app_id
request.auth.svix_endpoint_id = endpoint_out.id
await request.auth.asave()

# 获取 webhook secret
endpoint_secret_out = await svix.endpoint.get_secret(app_id, endpoint_out.id)

return 200, WebhookOut(
    app_id=app_id,
    endpoint_id=endpoint_out.id,
    webhook_secret=endpoint_secret_out.key,
)
```

**Webhook 事件类型**:

| 事件 | 说明 | Payload |
|------|------|---------|
| `upsert.success` | 文档上传成功 | 文档详情 |
| `upsert.fail` | 文档上传失败 | 错误信息 |

**Webhook 验证**:

使用 `webhook_secret` 验证请求签名（Svix 标准）。

## 11. 错误处理

### 11.1 通用错误模式

```python
class GenericError(Schema):
    detail: str

class GenericMessage(Schema):
    detail: str
```

### 11.2 常见错误码

| 状态码 | 说明 |
|--------|------|
| 400 | 请求参数错误、验证失败 |
| 401 | 未认证（无效 token） |
| 404 | 资源不存在 |
| 409 | 冲突（如重复创建、多个匹配） |
| 503 | 外部服务不可用（如嵌入服务） |

## 12. 性能优化

### 12.1 异步操作

所有端点都使用 `async def`：

```python
@router.post("/endpoint/")
async def endpoint(...):
    await Model.objects.acreate(...)
    await instance.asave()
    await instance.adelete()
```

### 12.2 批量操作

```python
await PageEmbedding.objects.abulk_create(bulk_create_embeddings)
await document.pages.all().adelete()
```

### 12.3 查询优化

```python
# Select Related（减少 JOIN 查询）
Document.objects.select_related("collection", "collection__owner")

# Prefetch Related（优化一对多）
Document.objects.prefetch_related(
    Prefetch("pages", queryset=Page.objects.order_by("page_number"))
)

# Annotate（聚合计算）
Document.objects.annotate(num_pages=Count("pages"))
```

## 总结

ColiVara 的 API 设计体现了以下特点：

1. **RESTful 设计**: 标准 HTTP 方法和状态码
2. **类型安全**: Pydantic Schema 验证
3. **异步优先**: 全异步端点
4. **灵活查询**: 支持元数据过滤和向量搜索
5. **错误处理**: 详细的错误信息和 Webhook 通知
6. **性能优化**: 批量操作、查询优化、背景任务
7. **扩展性**: 支持自定义元数据和 Webhook
