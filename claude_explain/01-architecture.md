# ColiVara 架构概览

## 1. 项目简介

**ColiVara** 是一个基于 ColPali 视觉语言模型的先进文档检索系统。它采用视觉嵌入（visual embeddings）而非传统文本解析来实现高精度的文档搜索和 RAG（检索增强生成）功能。

### 核心特性
- 基于视觉的文档理解（支持 100+ 文件格式）
- Late-Interaction 嵌入机制（延迟交互式嵌入）
- PostgreSQL + pgVector 向量存储
- RESTful API + Python/TypeScript SDK
- 完整的 CRUD 操作
- 灵活的元数据过滤（集合和文档级别）
- Webhook 事件通知
- 异步文档处理

## 2. 技术栈

### 后端框架
| 技术 | 版本 | 用途 |
|------|------|------|
| **Django** | 5.1.4 | Web 框架核心 |
| **Django Ninja** | 1.3.0 | RESTful API 框架（类似 FastAPI 的语法） |
| **Uvicorn** | 0.30.6 | ASGI 服务器 |
| **Python** | 3.12 | 编程语言 |

### 数据库与向量存储
| 技术 | 版本 | 用途 |
|------|------|------|
| **PostgreSQL** | 16 | 主数据库 |
| **pgVector** | 0.3.4 | 向量相似度搜索扩展 |
| **psycopg** | 3.2.3 | PostgreSQL 驱动（支持异步） |

### 核心依赖库
```
aiohttp[speedups]==3.10.5      # 异步 HTTP 客户端
httpx==0.27.2                  # 同步 HTTP 客户端
pdf2image==1.17.0              # PDF 转图像
python-magic==0.4.27           # 文件类型检测
tenacity==9.0.0                # 重试机制
django-cors-headers==4.4.0     # CORS 支持
django-storages[s3]==1.14.4    # S3 存储后端
stripe==10.12.0                # 支付集成
svix==1.40.0                   # Webhook 服务
sentry-sdk[django]==2.16.0     # 错误监控
```

## 3. 系统架构

### 3.1 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Layer                         │
│  (Python SDK / TypeScript SDK / HTTP API)                   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    Django Web Application                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   accounts   │  │     api      │  │    config    │      │
│  │  (用户管理)   │  │  (核心API)    │  │  (配置管理)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                  │                  │             │
│         └──────────────────┼──────────────────┘             │
└────────────────────────────┼────────────────────────────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          ▼                  ▼                  ▼
┌──────────────────┐ ┌──────────────┐ ┌────────────────┐
│   PostgreSQL     │ │   Embeddings │ │   Gotenberg    │
│   + pgVector     │ │    Service   │ │ (文档转换服务) │
│  (数据&向量存储)  │ │  (ColPali模型)│ │                │
└──────────────────┘ └──────────────┘ └────────────────┘
          │
          ▼
┌──────────────────┐
│     AWS S3       │
│   (文件存储)      │
└──────────────────┘

External Services:
┌─────────┐ ┌─────────┐ ┌─────────┐
│ Stripe  │ │  Svix   │ │ Sentry  │
│ (支付)  │ │(Webhook)│ │(监控)   │
└─────────┘ └─────────┘ └─────────┘
```

### 3.2 应用层级结构

```
web/
├── api/                    # 核心 API 应用
│   ├── models.py          # 数据模型（709 行）
│   ├── views.py           # API 端点（1616 行）
│   ├── middleware.py      # 自定义中间件
│   ├── migrations/        # 数据库迁移（26 个文件）
│   ├── management/        # Django 管理命令
│   └── tests/             # 测试套件（99%+ 覆盖率）
│
├── accounts/              # 用户认证应用
│   ├── models.py          # 自定义用户模型
│   ├── admin.py           # 管理界面配置
│   └── migrations/        # 用户应用迁移
│
└── config/                # Django 配置
    ├── settings.py        # 项目配置（320+ 行）
    ├── urls.py            # URL 路由
    ├── asgi.py            # ASGI 入口
    └── wsgi.py            # WSGI 入口
```

## 4. 核心概念

### 4.1 数据模型层级

```
CustomUser（用户）
    │
    └── Collection（集合）
            │
            ├── metadata: JSONField
            └── Document（文档）
                    │
                    ├── metadata: JSONField
                    ├── url: URLField
                    ├── s3_file: FileField
                    └── Page（页面）
                            │
                            ├── page_number: int
                            ├── img_base64: TextField
                            └── PageEmbedding（嵌入向量）
                                    │
                                    └── embedding: HalfVectorField(128)
```

### 4.2 请求处理流程

```
HTTP Request
    │
    ▼
Bearer Token 认证
    │
    ▼
URL 中间件（自动添加尾部斜杠）
    │
    ▼
Django Ninja Router
    │
    ├─→ Collections API
    ├─→ Documents API
    ├─→ Search API
    ├─→ Embeddings API
    └─→ Webhook API
    │
    ▼
数据库操作（异步）
    │
    ▼
JSON Response
```

## 5. 关键设计模式

### 5.1 异步优先（Async-First）

所有数据库操作和外部服务调用都使用异步方式：

```python
# 异步数据库操作
await Collection.objects.acreate(...)
await Document.objects.aget(...)
await document.asave()
await document.adelete()

# 异步外部服务调用
async with aiohttp.ClientSession() as session:
    async with session.post(url, ...) as response:
        data = await response.json()
```

### 5.2 批量处理与并发控制

文档嵌入处理采用批量+延迟策略：

```python
EMBEDDINGS_BATCH_SIZE = 3          # 每批 3 个页面
DELAY_BETWEEN_BATCHES = 1          # 批次间延迟 1 秒
```

### 5.3 重试机制

使用 tenacity 库实现自动重试：

```python
@retry(
    stop=stop_after_attempt(3),     # 最多重试 3 次
    wait=wait_fixed(5),             # 每次等待 5 秒
    reraise=True,
    retry=retry_if_exception_type(aiohttp.ClientError)
)
async def send_batch(...):
    # 发送批次到嵌入服务
```

### 5.4 背景任务处理

支持同步和异步两种文档处理模式：

```python
# 同步模式（wait=true）
if payload.wait:
    return await process_upsert_document(request, payload)

# 异步模式（wait=false，默认）
else:
    asyncio.create_task(process_upsert_document(request, payload))
    return 202, "Document is being processed in the background."
```

## 6. 安全特性

### 6.1 认证机制

- **Bearer Token 认证**：每个用户自动生成唯一令牌
- **OAuth2 社交登录**：支持 Google 和 GitHub
- **Django Session 认证**：用于管理界面

### 6.2 数据隔离

- 用户只能访问自己的集合和文档
- 所有查询都通过 `owner=request.auth` 过滤
- 数据库级别的唯一性约束

### 6.3 输入验证

使用 Pydantic Schema 进行严格的输入验证：

```python
class DocumentIn(Schema):
    name: str
    metadata: dict = Field(default_factory=dict)
    collection_name: str = "default_collection"
    url: Optional[str] = None
    base64: Optional[str] = None

    @model_validator(mode="after")
    def base64_or_url(self) -> Self:
        # 验证逻辑
```

## 7. 性能优化

### 7.1 数据库优化

- **索引**：在 name + owner、name + collection 等字段上创建索引
- **Select Related**：使用 `select_related()` 减少查询次数
- **Prefetch Related**：使用 `prefetch_related()` 优化一对多关系
- **HalfVector**：使用半精度浮点数减少存储空间（128 维 → 256 字节）

### 7.2 文件存储优化

- AWS S3 存储文档文件
- 按用户邮箱组织文件路径
- 自动清理孤立文件（django-cleanup）
- 最大文件大小限制：50MB

### 7.3 向量搜索优化

使用 PostgreSQL 自定义函数 `max_sim` 实现最大相似度搜索：

```sql
CREATE OR REPLACE FUNCTION max_sim(page_embeddings halfvec[], query_embeddings text[])
RETURNS float
AS $$
    -- MaxSim 实现
$$
LANGUAGE SQL IMMUTABLE PARALLEL SAFE;
```

## 8. 可扩展性设计

### 8.1 元数据系统

Collection 和 Document 都支持任意 JSON 元数据：

```python
metadata = JSONField(default=dict)
```

支持的查询操作：
- `key_lookup`：精确匹配
- `contains`：包含查询
- `contained_by`：被包含查询
- `has_key`：键存在查询
- `has_keys`：多键存在查询
- `has_any_keys`：任意键存在查询

### 8.2 Webhook 事件系统

使用 Svix 提供 Webhook 功能：

```python
Event Types:
- upsert.success: 文档上传成功
- upsert.fail: 文档上传失败
```

### 8.3 多格式支持

支持 100+ 文件格式，包括：
- 文档：PDF, DOCX, XLSX, PPTX, TXT, RTF...
- 图像：PNG, JPG, GIF, TIFF, BMP...
- 网页：HTML, EPUB, XHTML...

## 9. 监控与日志

### 9.1 日志系统

```python
logger = logging.getLogger(__name__)

logger.info(f"Document {self.name} processed successfully.")
logger.error(f"Failed to get embeddings: {response.status}")
```

### 9.2 错误监控

集成 Sentry 进行错误追踪：

```python
sentry_sdk.init(
    dsn=settings.SENTRY_DSN,
    integrations=[DjangoIntegration()],
)
```

### 9.3 健康检查

```python
@router.get("/health/")
async def health(request) -> Dict[str, str]:
    return {"status": "ok"}
```

## 10. 部署架构

### 10.1 容器化

使用 Docker Compose 编排多个服务：

```yaml
services:
  - db: PostgreSQL 16 + pgVector
  - web: Django + Uvicorn
  - gotenberg: 文档转换服务
  - redis: 缓存和队列（生产环境）
```

### 10.2 环境配置

通过环境变量管理配置：

```python
DEBUG = env.bool("DEBUG", default=True)
DATABASE_URL = env("DATABASE_URL")
EMBEDDINGS_URL = env("EMBEDDINGS_URL")
AWS_ACCESS_KEY_ID = env("AWS_ACCESS_KEY_ID")
STRIPE_SECRET_KEY = env("STRIPE_SECRET_KEY")
```

## 11. API 设计原则

### 11.1 RESTful 设计

- 使用标准 HTTP 方法（GET, POST, PATCH, DELETE）
- 资源化的 URL 设计
- 明确的状态码（200, 201, 204, 400, 404, 409, 503）

### 11.2 响应格式

统一的响应格式：

```python
# 成功响应
return 200, DocumentOut(...)

# 错误响应
return 404, GenericError(detail="Document not found")
```

### 11.3 版本控制

通过 URL 路径进行版本控制（当前为 v1）：

```python
# config/urls.py
path("api/", api.urls)
```

## 12. 代码质量保证

### 12.1 类型检查

使用 mypy 进行静态类型检查：

```ini
[mypy]
python_version = 3.12
warn_return_any = True
warn_unused_configs = True
```

### 12.2 测试覆盖

```python
pytest-django==4.8.0
pytest-cov==5.0.0
coverage >= 99%
```

### 12.3 代码格式化

使用 Ruff 进行代码格式化和 linting：

```yaml
# .pre-commit-config.yaml
- repo: https://github.com/astral-sh/ruff-pre-commit
  hooks:
    - id: ruff
    - id: ruff-format
```

## 总结

ColiVara 采用现代化的 Python 异步架构，结合 Django 生态系统的成熟性和 Django Ninja 的高性能，构建了一个完整的文档检索系统。其核心优势在于：

1. **视觉优先**：使用 ColPali 模型处理文档，无需 OCR
2. **异步高效**：全异步设计，支持高并发
3. **灵活扩展**：元数据系统和 Webhook 支持多样化需求
4. **开发友好**：99%+ 测试覆盖率，完善的类型提示
5. **生产就绪**：容器化部署，完善的监控和日志系统
