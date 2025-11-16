# 部署与配置详解

本文档详细分析 ColiVara 的部署配置、环境设置和外部服务集成。

## 1. 部署架构

### 1.1 服务组件

```
┌─────────────────────────────────────────────────┐
│              Load Balancer / Nginx              │
└─────────────────────┬───────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Django App  │ │  Django App  │ │  Django App  │
│  (Uvicorn)   │ │  (Uvicorn)   │ │  (Uvicorn)   │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  PostgreSQL  │ │  Gotenberg   │ │  Embeddings  │
│  + pgVector  │ │              │ │   Service    │
└──────────────┘ └──────────────┘ └──────────────┘
        │
        ▼
┌──────────────┐
│   AWS S3     │
└──────────────┘

External Services:
┌─────────┐ ┌─────────┐ ┌─────────┐
│ Stripe  │ │  Svix   │ │ Sentry  │
└─────────┘ └─────────┘ └─────────┘
```

### 1.2 Docker Compose 结构

#### 开发环境

**文件位置**: `/home/user/ColiVara/docker-compose.yml`

```yaml
services:
  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  gotenberg:
    image: gotenberg/gotenberg:8
    ports:
      - "3000:3000"

  web:
    build: ./web
    command: uvicorn config.asgi:application --host 0.0.0.0 --port 8000 --reload
    volumes:
      - ./web:/app
    ports:
      - "8000:8000"
    depends_on:
      - db
      - gotenberg
    environment:
      - DEBUG=True
      - DATABASE_URL=postgres://postgres:postgres@db/postgres
      - GOTENBERG_URL=http://gotenberg:3000
```

#### 生产环境

**文件位置**: `/home/user/ColiVara/docker-compose-prod.yml`

```yaml
services:
  db:
    # 同开发环境，但添加资源限制
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru

  web:
    build: ./web
    command: ./release.sh
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 1G
        reservations:
          memory: 512M
    environment:
      - DEBUG=False
      - SECURE_SSL_REDIRECT=True
```

## 2. 环境配置

### 2.1 配置文件

**文件位置**: `web/config/settings.py`

使用 `environs` 库管理环境变量：

```python
from environs import Env

env = Env()
env.read_env()  # 读取 .env 文件
```

### 2.2 核心配置项

#### 2.2.1 Django 基础配置

```python
# 安全密钥
SECRET_KEY = env("SECRET_KEY", default="django-insecure-dummy-key")

# 调试模式
DEBUG = env.bool("DEBUG", default=True)
LOCAL = env.bool("LOCAL", default=True)

# 允许的主机
ALLOWED_HOSTS = ["*"]  # 生产环境应限制

# 管理员
DJANGO_ADMINS = env.list(
    "DJANGO_ADMINS",
    default=["Dummy Name:dummy@example.com"]
)
ADMINS = [tuple(x.split(":")) for x in DJANGO_ADMINS]
# 示例: DJANGO_ADMINS=Blake:blake@example.com,Alice:alice@example.com
```

#### 2.2.2 安全配置

```python
# 生产环境必须启用
SECURE_SSL_REDIRECT = env.bool("DJANGO_SECURE_SSL_REDIRECT", default=False)
SECURE_HSTS_SECONDS = env.int("DJANGO_SECURE_HSTS_SECONDS", default=0)
SECURE_HSTS_INCLUDE_SUBDOMAINS = env.bool(
    "DJANGO_SECURE_HSTS_INCLUDE_SUBDOMAINS", default=False
)
SECURE_HSTS_PRELOAD = env.bool("DJANGO_SECURE_HSTS_PRELOAD", default=False)
SESSION_COOKIE_SECURE = env.bool("DJANGO_SESSION_COOKIE_SECURE", default=False)
CSRF_COOKIE_SECURE = env.bool("DJANGO_CSRF_COOKIE_SECURE", default=False)
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
```

**生产环境示例**:

```bash
DJANGO_SECURE_SSL_REDIRECT=True
DJANGO_SECURE_HSTS_SECONDS=2592000  # 30 days
DJANGO_SECURE_HSTS_INCLUDE_SUBDOMAINS=True
DJANGO_SECURE_HSTS_PRELOAD=True
DJANGO_SESSION_COOKIE_SECURE=True
DJANGO_CSRF_COOKIE_SECURE=True
```

#### 2.2.3 数据库配置

```python
DATABASES = {
    "default": env.dj_db_url(
        "DATABASE_URL",
        default="postgres://postgres@db/postgres"
    )
}
```

**环境变量格式**:

```bash
# PostgreSQL
DATABASE_URL=postgres://user:password@host:port/dbname

# 示例
DATABASE_URL=postgres://colivara:secret@db.example.com:5432/colivara_prod
```

#### 2.2.4 外部服务配置

```python
# 嵌入服务
EMBEDDINGS_URL = env("EMBEDDINGS_URL")
ALWAYS_ON_EMBEDDINGS_URL = env("ALWAYS_ON_EMBEDDINGS_URL")
EMBEDDINGS_URL_TOKEN = env("EMBEDDINGS_URL_TOKEN")

# Gotenberg
GOTENBERG_URL = env("GOTENBERG_URL", default="http://gotenberg:3000")

# 代理（可选）
PROXY_URL = env("PROXY_URL", default=None)

# AWS S3
AWS_ACCESS_KEY_ID = env("AWS_ACCESS_KEY_ID")
AWS_SECRET_ACCESS_KEY = env("AWS_SECRET_ACCESS_KEY")
AWS_STORAGE_BUCKET_NAME = env("AWS_STORAGE_BUCKET_NAME")
AWS_S3_REGION_NAME = env("AWS_S3_REGION_NAME", default="us-east-1")
AWS_S3_ENDPOINT_URL = env("AWS_S3_ENDPOINT_URL", default=None)

# Stripe
STRIPE_SECRET_KEY = env("STRIPE_SECRET_KEY", default="")
STRIPE_PUBLISHABLE_KEY = env("STRIPE_PUBLISHABLE_KEY", default="")

# Svix (Webhook)
SVIX_TOKEN = env("SVIX_TOKEN", default="")

# Sentry
SENTRY_DSN = env("SENTRY_DSN", default="")
```

### 2.3 .env 文件示例

```bash
# Django
SECRET_KEY=your-secret-key-here
DEBUG=False
ALLOWED_HOSTS=api.example.com,www.example.com

# Security
DJANGO_SECURE_SSL_REDIRECT=True
DJANGO_SECURE_HSTS_SECONDS=2592000

# Database
DATABASE_URL=postgres://colivara:password@db:5432/colivara

# Embeddings Service
EMBEDDINGS_URL=http://embeddings:8080/v1/embeddings
ALWAYS_ON_EMBEDDINGS_URL=http://embeddings:8080/v1/embeddings
EMBEDDINGS_URL_TOKEN=your-embeddings-token

# Gotenberg
GOTENBERG_URL=http://gotenberg:3000

# AWS S3
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_STORAGE_BUCKET_NAME=colivara-documents
AWS_S3_REGION_NAME=us-east-1

# Stripe
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...

# Svix
SVIX_TOKEN=svix_auth_token_...

# Sentry
SENTRY_DSN=https://...@sentry.io/...

# Email
EMAIL_CONSOLE=False
EMAIL_HOST=smtp.gmail.com
EMAIL_HOST_USER=noreply@example.com
EMAIL_HOST_PASSWORD=your-email-password
DEFAULT_FROM_EMAIL=noreply@example.com

# Admins
DJANGO_ADMINS=Admin:admin@example.com
```

## 3. 数据库配置

### 3.1 PostgreSQL + pgVector

#### 3.1.1 安装

```sql
-- 启用 pgvector 扩展
CREATE EXTENSION IF NOT EXISTS vector;
```

**自动化**（文件位置: `web/api/migrations/0001_enable_pgvector.py`）:

```python
from django.contrib.postgres.operations import CreateExtension

class Migration(migrations.Migration):
    operations = [
        CreateExtension("vector"),
    ]
```

#### 3.1.2 性能调优

**postgresql.conf**:

```ini
# 内存配置
shared_buffers = 4GB
effective_cache_size = 12GB
work_mem = 256MB
maintenance_work_mem = 1GB

# 连接
max_connections = 100

# WAL
wal_buffers = 16MB
checkpoint_completion_target = 0.9

# 查询规划
random_page_cost = 1.1  # SSD
effective_io_concurrency = 200
```

#### 3.1.3 向量索引优化

```sql
-- 创建 IVFFlat 索引
CREATE INDEX ON api_pageembedding
USING ivfflat (embedding halfvec_cosine_ops)
WITH (lists = 100);

-- lists 参数选择:
-- - 数据量 < 1M: lists = sqrt(rows)
-- - 数据量 > 1M: lists = rows / 1000
```

### 3.2 数据库迁移

```bash
# 应用迁移
python manage.py migrate

# 创建新迁移
python manage.py makemigrations

# 查看迁移状态
python manage.py showmigrations
```

## 4. 文件存储配置

### 4.1 AWS S3

**文件位置**: `web/config/settings.py:240-280`

```python
# S3 作为默认存储
DEFAULT_FILE_STORAGE = "storages.backends.s3boto3.S3Boto3Storage"

# S3 配置
AWS_ACCESS_KEY_ID = env("AWS_ACCESS_KEY_ID")
AWS_SECRET_ACCESS_KEY = env("AWS_SECRET_ACCESS_KEY")
AWS_STORAGE_BUCKET_NAME = env("AWS_STORAGE_BUCKET_NAME")
AWS_S3_REGION_NAME = env("AWS_S3_REGION_NAME", default="us-east-1")

# 自定义端点（S3 兼容服务）
AWS_S3_ENDPOINT_URL = env("AWS_S3_ENDPOINT_URL", default=None)

# 文件公开访问
AWS_DEFAULT_ACL = "public-read"
AWS_QUERYSTRING_AUTH = False

# 文件上传限制
DATA_UPLOAD_MAX_MEMORY_SIZE = 50 * 1024 * 1024  # 50MB
```

### 4.2 S3 兼容服务

#### MinIO

```bash
# .env
AWS_ACCESS_KEY_ID=minioadmin
AWS_SECRET_ACCESS_KEY=minioadmin
AWS_STORAGE_BUCKET_NAME=colivara
AWS_S3_ENDPOINT_URL=http://minio:9000
```

#### DigitalOcean Spaces

```bash
# .env
AWS_ACCESS_KEY_ID=your-spaces-key
AWS_SECRET_ACCESS_KEY=your-spaces-secret
AWS_STORAGE_BUCKET_NAME=your-space-name
AWS_S3_REGION_NAME=nyc3
AWS_S3_ENDPOINT_URL=https://nyc3.digitaloceanspaces.com
```

### 4.3 本地文件存储（开发）

```python
# settings.py
if DEBUG:
    DEFAULT_FILE_STORAGE = "django.core.files.storage.FileSystemStorage"
    MEDIA_ROOT = BASE_DIR / "media"
    MEDIA_URL = "/media/"
```

## 5. 外部服务集成

### 5.1 Embeddings Service（ColiVarE）

**部署选项**:

#### 选项 1: 使用官方服务

```bash
EMBEDDINGS_URL=https://embeddings.colivara.com/v1/embeddings
ALWAYS_ON_EMBEDDINGS_URL=https://embeddings.colivara.com/v1/embeddings
EMBEDDINGS_URL_TOKEN=your-api-token
```

#### 选项 2: 自托管

```bash
# Docker
docker run -d \
  --name colivare \
  --gpus all \
  -p 8080:8080 \
  -e MODEL_NAME=vidore/colpali-v1.2 \
  ghcr.io/tjmlabs/colivare:latest

# .env
EMBEDDINGS_URL=http://embeddings:8080/v1/embeddings
ALWAYS_ON_EMBEDDINGS_URL=http://embeddings:8080/v1/embeddings
EMBEDDINGS_URL_TOKEN=your-token
```

**硬件要求**:
- GPU: NVIDIA GPU with 8GB+ VRAM
- RAM: 16GB+
- Disk: 20GB+

### 5.2 Gotenberg

**Docker Compose**:

```yaml
gotenberg:
  image: gotenberg/gotenberg:8
  ports:
    - "3000:3000"
  command:
    - "gotenberg"
    - "--api-timeout=30s"
    - "--api-timeout-duration=30s"
```

**配置**:

```bash
GOTENBERG_URL=http://gotenberg:3000
```

### 5.3 Stripe

```python
# settings.py
import stripe

stripe.api_key = env("STRIPE_SECRET_KEY")
```

**Webhook 配置**:

```bash
# Stripe CLI (测试)
stripe listen --forward-to localhost:8000/stripe-webhook/

# 生产环境
# 在 Stripe Dashboard 配置 Webhook URL:
# https://api.example.com/stripe-webhook/
```

### 5.4 Svix (Webhook)

```python
# 创建应用
from svix.api import SvixAsync

svix = SvixAsync(settings.SVIX_TOKEN)
app = await svix.application.create(ApplicationIn(name="user@example.com"))

# 创建端点
endpoint = await svix.endpoint.create(
    app.id,
    EndpointIn(
        url="https://customer.example.com/webhooks/colivara",
        version=1,
    ),
)
```

### 5.5 Sentry

```python
# settings.py
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

if not DEBUG and SENTRY_DSN:
    sentry_sdk.init(
        dsn=SENTRY_DSN,
        integrations=[DjangoIntegration()],
        traces_sample_rate=0.1,
        send_default_pii=False,
    )
```

## 6. 中间件配置

### 6.1 CORS

```python
# settings.py
INSTALLED_APPS = [
    ...
    "corsheaders",
]

MIDDLEWARE = [
    ...
    "corsheaders.middleware.CorsMiddleware",
    ...
]

# CORS 配置
CORS_ALLOW_ALL_ORIGINS = True  # 开发环境

# 生产环境
CORS_ALLOWED_ORIGINS = [
    "https://app.example.com",
    "https://www.example.com",
]

CORS_ALLOW_METHODS = [
    "DELETE",
    "GET",
    "OPTIONS",
    "PATCH",
    "POST",
    "PUT",
]
```

### 6.2 自定义中间件

**文件位置**: `web/api/middleware.py`

```python
def add_slash(get_response):
    """自动添加尾部斜杠"""
    def middleware(request):
        if not request.path.endswith('/'):
            return HttpResponsePermanentRedirect(request.path + '/')
        return get_response(request)
    return middleware
```

**配置**:

```python
MIDDLEWARE = [
    ...
    "api.middleware.add_slash",
    ...
]
```

## 7. 认证配置

### 7.1 Django Allauth

```python
INSTALLED_APPS = [
    ...
    "allauth",
    "allauth.account",
    "allauth.socialaccount",
    "allauth.socialaccount.providers.google",
    "allauth.socialaccount.providers.github",
]

AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",
    "allauth.account.auth_backends.AuthenticationBackend",
]

# Allauth 配置
ACCOUNT_EMAIL_REQUIRED = True
ACCOUNT_USERNAME_REQUIRED = False
ACCOUNT_AUTHENTICATION_METHOD = "email"
SOCIALACCOUNT_AUTO_SIGNUP = True
```

### 7.2 社交登录

#### Google

```python
# 在 Google Cloud Console 创建 OAuth2 凭据
# 然后在 Django Admin 添加 Social App:

SOCIALACCOUNT_PROVIDERS = {
    "google": {
        "SCOPE": ["profile", "email"],
        "AUTH_PARAMS": {"access_type": "online"},
    }
}
```

#### GitHub

```python
SOCIALACCOUNT_PROVIDERS = {
    "github": {
        "SCOPE": ["user:email"],
    }
}
```

## 8. 静态文件配置

```python
# settings.py
STATIC_URL = "/static/"
STATICFILES_DIRS = [BASE_DIR / "static"]
STATIC_ROOT = BASE_DIR / "staticfiles"

# 生产环境使用 Servestatic
INSTALLED_APPS = [
    "servestatic.runserver_nostatic",
    ...
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "servestatic.middleware.ServeStaticMiddleware",
    ...
]

# Servestatic 配置
STATICFILES_STORAGE = "servestatic.storage.CompressedManifestStaticFilesStorage"
```

**收集静态文件**:

```bash
python manage.py collectstatic --noinput
```

## 9. 日志配置

```python
# settings.py
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "verbose": {
            "format": "{levelname} {asctime} {module} {message}",
            "style": "{",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "verbose",
        },
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "filename": "logs/django.log",
            "maxBytes": 10485760,  # 10MB
            "backupCount": 5,
            "formatter": "verbose",
        },
    },
    "root": {
        "handlers": ["console", "file"],
        "level": "INFO",
    },
    "loggers": {
        "django": {
            "handlers": ["console", "file"],
            "level": "INFO",
            "propagate": False,
        },
        "api": {
            "handlers": ["console", "file"],
            "level": "DEBUG",
            "propagate": False,
        },
    },
}
```

## 10. 部署检查清单

### 10.1 安全

- [ ] `DEBUG = False`
- [ ] `SECRET_KEY` 使用强随机值
- [ ] `ALLOWED_HOSTS` 配置正确
- [ ] 启用 HTTPS（`SECURE_SSL_REDIRECT = True`）
- [ ] 启用 HSTS
- [ ] `SESSION_COOKIE_SECURE = True`
- [ ] `CSRF_COOKIE_SECURE = True`
- [ ] 数据库密码强度足够
- [ ] AWS 密钥安全存储
- [ ] `.env` 文件不在版本控制中

### 10.2 性能

- [ ] 数据库连接池配置
- [ ] 静态文件压缩（Servestatic）
- [ ] 数据库索引已创建
- [ ] pgVector 索引已优化
- [ ] Redis 缓存配置（生产环境）
- [ ] 负载均衡器配置

### 10.3 监控

- [ ] Sentry 错误追踪
- [ ] 日志聚合（如 CloudWatch, Datadog）
- [ ] 性能监控（如 New Relic, DataDog）
- [ ] 数据库监控
- [ ] 磁盘空间监控

### 10.4 备份

- [ ] 数据库自动备份
- [ ] S3 版本控制
- [ ] 备份恢复测试
- [ ] 灾难恢复计划

## 11. 生产部署脚本

**文件位置**: `web/release.sh`

```bash
#!/bin/bash
set -e

echo "Starting release process..."

# 收集静态文件
echo "Collecting static files..."
python manage.py collectstatic --noinput

# 运行数据库迁移
echo "Running database migrations..."
python manage.py migrate --noinput

# 启动 Uvicorn
echo "Starting Uvicorn..."
uvicorn config.asgi:application \
    --host 0.0.0.0 \
    --port 8000 \
    --workers 4 \
    --log-level info
```

## 12. CI/CD 配置

**文件位置**: `.github/workflows/test.yml`

```yaml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: pgvector/pgvector:pg16
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      gotenberg:
        image: gotenberg/gotenberg:8

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          pip install -r web/requirements.txt

      - name: Run tests
        run: |
          cd web
          pytest --cov=api --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

## 总结

ColiVara 的部署配置展现了以下特点：

1. **环境隔离**: 开发、测试、生产环境分离
2. **配置管理**: 使用环境变量管理敏感信息
3. **容器化**: Docker Compose 简化部署
4. **安全性**: 完善的 HTTPS 和安全配置
5. **可扩展**: 支持水平扩展和负载均衡
6. **监控**: 完整的日志和错误追踪
7. **自动化**: CI/CD 流程自动化
8. **灵活性**: 支持多种存储后端和外部服务
