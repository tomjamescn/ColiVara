# 搜索与向量检索机制

本文档详细分析 ColiVara 的向量搜索机制，包括 Late-Interaction 嵌入、MaxSim 相似度计算和元数据过滤。

## 1. 搜索架构概览

```
查询输入（文本或图像）
    │
    ▼
├─ 嵌入生成
│   ├─ 文本查询 → ColPali (query mode)
│   └─ 图像查询 → ColPali (image mode)
│
    ▼
├─ 返回查询嵌入向量 (N × 128)
│   └─ N = token 数量（动态）
│
    ▼
├─ 转换为 HalfVector
│
    ▼
├─ 构建数据库查询
│   ├─ 集合过滤（collection_name）
│   └─ 元数据过滤（query_filter）
│
    ▼
├─ 聚合页面嵌入
│   └─ ArrayAgg("embeddings__embedding")
│
    ▼
├─ MaxSim 相似度计算
│   ├─ 页面嵌入: M × 128
│   ├─ 查询嵌入: N × 128
│   └─ 计算所有 M×N 对的相似度
│       └─ 返回最大值
│
    ▼
├─ 排序和 Top-K
│   └─ ORDER BY -max_sim LIMIT top_k
│
    ▼
├─ 结果归一化
│   └─ normalized_score = raw_score / query_length
│
    ▼
└─ 返回结果
    └─ PageOutQuery[]
```

## 2. Late-Interaction 嵌入机制

### 2.1 什么是 Late-Interaction

传统向量搜索：
```
文档 → 单个嵌入向量 (1 × 768)
查询 → 单个嵌入向量 (1 × 768)
相似度 = dot(文档向量, 查询向量)
```

Late-Interaction（ColPali）：
```
文档页面 → 多个嵌入向量 (M × 128)
查询 → 多个嵌入向量 (N × 128)
相似度 = MaxSim(文档向量[], 查询向量[])
```

### 2.2 优势

| 特性 | 传统方法 | Late-Interaction |
|------|---------|-----------------|
| **粒度** | 文档级别 | Token 级别 |
| **灵活性** | 固定维度 | 动态 token 数 |
| **精度** | 整体匹配 | 局部最佳匹配 |
| **计算** | 简单点积 | MaxSim 聚合 |

### 2.3 嵌入向量结构

**查询嵌入**:

```python
query = "machine learning"
embeddings = await get_query_embeddings(query)

# 结构:
[
    [0.12, 0.34, ..., 0.89],  # Token 1: "machine"
    [0.45, 0.67, ..., 0.12],  # Token 2: "learning"
    [0.78, 0.90, ..., 0.34],  # Token 3: [CLS] 或其他特殊 token
]
# 每个 token 都是 128 维向量
```

**页面嵌入**（已存储在数据库）:

```python
Page 1 → [
    PageEmbedding 1: [0.11, 0.22, ..., 0.88],  # 图像 patch 1
    PageEmbedding 2: [0.33, 0.44, ..., 0.66],  # 图像 patch 2
    PageEmbedding 3: [0.55, 0.66, ..., 0.44],  # 图像 patch 3
    ...
    PageEmbedding M: [0.77, 0.88, ..., 0.22],  # 图像 patch M
]
```

## 3. MaxSim 相似度计算

### 3.1 SQL 函数定义

**文件位置**: `web/api/models.py:707-709`

```python
class MaxSim(Func):
    function = "max_sim"
    output_field = FloatField()
```

**PostgreSQL 函数**（数据库迁移中定义）:

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
            -- 使用余弦距离的负值（<#> 操作符）
            page_embedding <#> CAST(query_embedding AS halfvec) AS similarity
        FROM
            UNNEST(page_embeddings) AS page_embedding,
            UNNEST(query_embeddings) AS query_embedding
    ) AS similarities
$$
LANGUAGE SQL IMMUTABLE PARALLEL SAFE;
```

### 3.2 计算流程

**示例**:

```
页面嵌入 (3 个向量):
P1 = [0.1, 0.2, ..., 0.8]
P2 = [0.3, 0.4, ..., 0.6]
P3 = [0.5, 0.6, ..., 0.4]

查询嵌入 (2 个向量):
Q1 = [0.2, 0.3, ..., 0.7]
Q2 = [0.4, 0.5, ..., 0.5]

相似度矩阵 (3 × 2 = 6 次计算):
       Q1      Q2
P1   0.85    0.72
P2   0.78    0.91
P3   0.65    0.70

MaxSim = max(0.85, 0.72, 0.78, 0.91, 0.65, 0.70) = 0.91
```

**关键点**:
- 计算所有 M×N 对的相似度
- 使用余弦相似度（通过 pgvector 的 `<#>` 操作符）
- 返回最大值

### 3.3 为什么使用 MaxSim

**直觉**:
- 查询的某个 token 可能与文档的某个部分高度相关
- 我们关心"最佳匹配"而非"平均匹配"
- 类似于"找到最相关的段落"

**示例**:

```
查询: "machine learning algorithms"
文档页面包含:
- 大量无关文本
- 一个段落详细描述机器学习算法

MaxSim 会找到那个高度相关的段落，而平均相似度会被无关文本拉低。
```

## 4. 文本搜索实现

### 4.1 端点实现

**文件位置**: `web/api/views.py:961-1062`

```python
@router.post("/search/", auth=Bearer())
async def search(request: Request, payload: QueryIn):
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
            collection_id=row["document__collection__id"],
            collection_metadata=row["document__collection__metadata"] or {},
            document_name=row["document__name"],
            document_id=row["document__id"],
            document_metadata=row["document__metadata"] or {},
            page_number=row["page_number"],
            raw_score=row["max_sim"],
            normalized_score=row["max_sim"] / query_length,
            img_base64=row["img_base64"],
        )
        async for row in results
    ]

    return 200, QueryOut(query=payload.query, results=formatted_results)
```

### 4.2 查询嵌入生成

**文件位置**: `web/api/views.py:1252-1274`

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
            # 返回动态数量的嵌入向量
            return out["output"]["data"][0]["embedding"]
```

**请求示例**:

```json
{
  "input": {
    "task": "query",
    "input_data": ["machine learning algorithms"]
  }
}
```

**响应示例**:

```json
{
  "output": {
    "data": [
      {
        "embedding": [
          [0.12, 0.34, ..., 0.89],  // Token 1
          [0.45, 0.67, ..., 0.12],  // Token 2
          [0.78, 0.90, ..., 0.34]   // Token 3
        ],
        "index": 0,
        "object": "embedding"
      }
    ]
  }
}
```

### 4.3 HalfVector 转换

**文件位置**: `views.py:1008-1010`

```python
from pgvector.utils import HalfVector

casted_query_embeddings = [
    HalfVector(embedding).to_text() for embedding in query_embeddings
]
```

**为什么需要转换**:
- 数据库中的嵌入是 HalfVector（float16）
- 嵌入服务返回的是 float32
- 需要转换为相同格式以进行相似度计算

**转换示例**:

```python
# 原始嵌入 (float32)
embedding = [0.12345678, 0.87654321, ...]

# 转换为 HalfVector (float16)
half_vec = HalfVector(embedding)

# 转换为 PostgreSQL 文本格式
text_format = half_vec.to_text()
# 输出: "[0.1235,0.8765,...]"
```

### 4.4 数据库查询分析

**SQL 等价**（简化版）:

```sql
SELECT
    p.id,
    p.page_number,
    p.img_base64,
    d.id AS document__id,
    d.name AS document__name,
    d.metadata AS document__metadata,
    c.id AS document__collection__id,
    c.name AS document__collection__name,
    c.metadata AS document__collection__metadata,
    max_sim(
        ARRAY_AGG(pe.embedding),
        ARRAY['[0.12,...]', '[0.45,...]', '[0.78,...]']
    ) AS max_sim
FROM
    api_page p
    INNER JOIN api_document d ON p.document_id = d.id
    INNER JOIN api_collection c ON d.collection_id = c.id
    INNER JOIN api_pageembedding pe ON pe.page_id = p.id
WHERE
    c.owner_id = <user_id>
    AND c.name = 'my_collection'  -- 如果指定
    AND d.metadata @> '{"key": "value"}'  -- 如果有过滤
GROUP BY
    p.id, d.id, c.id
ORDER BY
    max_sim DESC
LIMIT 3;
```

## 5. 图像搜索实现

### 5.1 端点实现

**文件位置**: `web/api/views.py:1065-1166`

```python
@router.post("/search-image/", auth=Bearer())
async def search_image(request: Request, payload: SearchImageIn):
    # 与文本搜索类似，但使用图像嵌入
    image_embeddings = await get_image_embeddings(payload.img_base64)
    if not image_embeddings:
        return 503, GenericError(detail="Failed to get embeddings")

    # ... 其余逻辑与文本搜索相同
```

### 5.2 图像嵌入生成

**文件位置**: `web/api/views.py:1277-1299`

```python
async def get_image_embeddings(img_base64: str) -> List:
    EMBEDDINGS_URL = settings.ALWAYS_ON_EMBEDDINGS_URL
    embed_token = settings.EMBEDDINGS_URL_TOKEN

    headers = {"Authorization": f"Bearer {embed_token}"}
    payload = {
        "input": {
            "task": "image",  # 注意这里是 "image"
            "input_data": [img_base64],
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

### 5.3 图像搜索用例

**以图搜图**:

```json
{
  "img_base64": "iVBORw0KGgoAAAANSUhEUgAA...",
  "collection_name": "all",
  "top_k": 5
}
```

**用途**:
- 查找相似的图表
- 查找相似的页面布局
- 查找包含特定视觉元素的页面

## 6. 元数据过滤

### 6.1 过滤器结构

**文件位置**: `web/api/views.py:867-908`

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

### 6.2 Lookup 类型详解

#### 6.2.1 key_lookup（精确匹配）

```python
# 过滤器
{
  "on": "document",
  "key": "author",
  "value": "John Doe",
  "lookup": "key_lookup"
}

# SQL 等价
WHERE document.metadata->>'author' = 'John Doe'

# 匹配的元数据
{"author": "John Doe", "year": 2024}  ✓
{"author": "Jane Doe", "year": 2024}  ✗
```

#### 6.2.2 contains（包含）

```python
# 过滤器
{
  "on": "document",
  "key": "author",
  "value": "John",
  "lookup": "contains"
}

# SQL 等价
WHERE document.metadata @> '{"author": "John"}'

# 匹配的元数据
{"author": "John", "year": 2024}  ✓
{"author": "John Doe", "year": 2024}  ✗ (完整匹配)
```

#### 6.2.3 contained_by（被包含）

```python
# 过滤器
{
  "on": "document",
  "key": "tags",
  "value": ["ml", "ai"],
  "lookup": "contained_by"
}

# SQL 等价
WHERE document.metadata <@ '{"tags": ["ml", "ai"]}'

# 匹配的元数据
{"tags": ["ml"]}  ✓ (子集)
{"tags": ["ml", "ai"]}  ✓ (相等)
{"tags": ["ml", "ai", "dl"]}  ✗ (超集)
```

#### 6.2.4 has_key（键存在）

```python
# 过滤器
{
  "on": "document",
  "key": "reviewed",
  "lookup": "has_key"
}

# SQL 等价
WHERE document.metadata ? 'reviewed'

# 匹配的元数据
{"reviewed": true, "author": "John"}  ✓
{"reviewed": false}  ✓
{"author": "John"}  ✗
```

#### 6.2.5 has_keys（多键存在）

```python
# 过滤器
{
  "on": "document",
  "key": ["author", "year"],
  "lookup": "has_keys"
}

# SQL 等价
WHERE document.metadata ?& ARRAY['author', 'year']

# 匹配的元数据
{"author": "John", "year": 2024}  ✓
{"author": "John", "year": 2024, "reviewed": true}  ✓
{"author": "John"}  ✗ (缺少 year)
```

#### 6.2.6 has_any_keys（任意键存在）

```python
# 过滤器
{
  "on": "document",
  "key": ["author", "editor"],
  "lookup": "has_any_keys"
}

# SQL 等价
WHERE document.metadata ?| ARRAY['author', 'editor']

# 匹配的元数据
{"author": "John"}  ✓
{"editor": "Jane"}  ✓
{"author": "John", "editor": "Jane"}  ✓
{"year": 2024}  ✗
```

### 6.3 过滤实现

**文件位置**: `web/api/views.py:1302-1334`

```python
async def filter_query(
    payload: Union[QueryIn, SearchImageIn], user: CustomUser
) -> QuerySet[Page]:
    base_query = Page.objects.select_related("document__collection")

    # 1. 集合过滤
    if payload.collection_name == "all":
        base_query = base_query.filter(document__collection__owner=user)
    else:
        base_query = base_query.filter(
            document__collection__owner=user,
            document__collection__name=payload.collection_name,
        )

    # 2. 元数据过滤
    if payload.query_filter:
        on = payload.query_filter.on
        key = payload.query_filter.key
        value = payload.query_filter.value
        lookup = payload.query_filter.lookup

        # 确定字段前缀
        field_prefix = (
            "document__collection__metadata"
            if on == "collection"
            else "document__metadata"
        )

        # Lookup 操作映射
        lookup_operations = {
            "key_lookup": lambda k, v: {f"{field_prefix}__{k}": v},
            "contains": lambda k, v: {f"{field_prefix}__contains": {k: v}},
            "contained_by": lambda k, v: {f"{field_prefix}__contained_by": {k: v}},
            "has_key": lambda k, _: {f"{field_prefix}__has_key": k},
            "has_keys": lambda k, _: {f"{field_prefix}__has_keys": k},
            "has_any_keys": lambda k, _: {f"{field_prefix}__has_any_keys": k},
        }

        # 应用过滤
        filter_params = lookup_operations[lookup](key, value)
        base_query = base_query.filter(**filter_params)

    return base_query
```

### 6.4 组合查询示例

**复杂查询**:

```json
{
  "query": "neural networks",
  "collection_name": "research",
  "top_k": 5,
  "query_filter": {
    "on": "document",
    "key": "year",
    "value": 2024,
    "lookup": "key_lookup"
  }
}
```

**执行流程**:

1. 过滤集合: `collection.name = 'research' AND collection.owner = <user>`
2. 过滤文档: `document.metadata->>'year' = '2024'`
3. 获取所有符合条件的页面
4. 计算 MaxSim 相似度
5. 排序并返回 Top-5

## 7. 得分归一化

### 7.1 为什么需要归一化

**问题**: 不同查询的 token 数量不同，导致得分不可比。

```
查询 1: "ML" (1 token)
  → MaxSim 得分范围: [0, 1]

查询 2: "machine learning algorithms" (3 tokens)
  → MaxSim 得分范围: [0, 3]
```

### 7.2 归一化公式

```python
normalization_factor = query_length  # token 数量

normalized_score = raw_score / normalization_factor
```

**示例**:

```
查询: "machine learning" (2 tokens)
页面 A: raw_score = 1.8
  → normalized_score = 1.8 / 2 = 0.9

页面 B: raw_score = 1.5
  → normalized_score = 1.5 / 2 = 0.75
```

### 7.3 实现

**文件位置**: `views.py:1038-1058`

```python
# 获取查询长度
query_length = len(query_embeddings)

# ... 执行查询

# 格式化结果时归一化
formatted_results = [
    PageOutQuery(
        ...
        raw_score=row["max_sim"],
        normalized_score=row["max_sim"] / query_length,
        ...
    )
    async for row in results
]
```

### 7.4 使用建议

| 场景 | 使用得分 | 原因 |
|------|---------|------|
| 单次查询排序 | Raw Score | 相对顺序不变 |
| 跨查询比较 | Normalized Score | 消除 token 数差异 |
| 阈值过滤 | Normalized Score | 统一标准 |

## 8. 性能优化

### 8.1 数据库索引

```python
# PageEmbedding 模型
embedding = HalfVectorField(dimensions=128)
```

**pgvector 索引** (在数据库迁移中创建):

```sql
CREATE INDEX ON api_pageembedding
USING ivfflat (embedding halfvec_cosine_ops)
WITH (lists = 100);
```

**参数**:
- `ivfflat`: 近似最近邻（ANN）索引
- `halfvec_cosine_ops`: 余弦相似度操作符
- `lists = 100`: 聚类数量（权衡速度和精度）

### 8.2 查询优化

#### 8.2.1 Select Related

```python
base_query = Page.objects.select_related("document__collection")
```

**效果**: 减少数据库查询次数（JOIN 而非多次查询）。

#### 8.2.2 Limit 早期应用

```python
pages_query = (
    base_query
    .annotate(page_embeddings=ArrayAgg("embeddings__embedding"))
    .annotate(max_sim=MaxSim("page_embeddings", casted_query_embeddings))
    .order_by("-max_sim")[: payload.top_k or 3]  # LIMIT 在数据库层面
)
```

**效果**: 数据库只返回 Top-K 结果，减少数据传输。

#### 8.2.3 Values 优化

```python
results = pages_query.values(
    "id",
    "page_number",
    "img_base64",
    # ... 仅选择需要的字段
)
```

**效果**: 避免加载整个模型对象。

### 8.3 批量处理

对于大量查询，可以使用批量嵌入：

```python
# 批量获取嵌入
queries = ["query1", "query2", "query3"]
embeddings_response = await get_embeddings_batch(queries)
```

## 9. 搜索质量

### 9.1 影响因素

| 因素 | 影响 |
|------|------|
| **文档质量** | 清晰的图像产生更好的嵌入 |
| **查询精度** | 具体的查询比模糊的查询效果好 |
| **元数据** | 结合元数据过滤提高准确性 |
| **集合大小** | 较小的集合搜索更快更准确 |

### 9.2 最佳实践

#### 9.2.1 文本查询

```python
# 好的查询
"machine learning algorithms for image classification"
"revenue trends in Q4 2023"
"safety procedures for hazardous materials"

# 差的查询
"things"  # 太模糊
"a"  # 太短
"asdfghjkl"  # 无意义
```

#### 9.2.2 图像查询

```python
# 适合图像搜索的场景
- 查找相似的图表
- 查找包含特定布局的页面
- 查找包含特定视觉元素的页面

# 不适合的场景
- 需要精确文本匹配（使用文本搜索）
- 需要理解复杂语义（使用文本搜索）
```

#### 9.2.3 元数据过滤

```python
# 有效使用
{
  "query": "revenue",
  "query_filter": {
    "on": "document",
    "key": "year",
    "value": 2024,
    "lookup": "key_lookup"
  }
}
# 先过滤到相关文档，再搜索

# 低效使用
{
  "query": "revenue",
  "collection_name": "all"  # 搜索所有集合
}
# 搜索范围太大
```

### 9.3 调试技巧

#### 9.3.1 检查原始得分

```python
# 返回的结果包含两个得分
{
  "raw_score": 0.85,
  "normalized_score": 0.425
}

# 如果 raw_score 很低 (< 0.3)，可能是:
- 查询与文档不相关
- 嵌入质量问题
- 文档图像质量问题
```

#### 9.3.2 使用 top_k

```python
# 开始时使用较大的 top_k
{
  "query": "...",
  "top_k": 10
}

# 检查结果分布
- 前几个结果得分接近 → 好的查询
- 得分差异很大 → 可能需要调整查询
```

## 10. 与传统搜索的比较

| 特性 | 传统全文搜索 | ColiVara 向量搜索 |
|------|-------------|-----------------|
| **输入** | 文本 | 文本或图像 |
| **索引** | 倒排索引 | 向量索引 |
| **匹配** | 关键词匹配 | 语义相似度 |
| **格式** | 需要文本提取 | 直接处理图像 |
| **语言** | 依赖语言模型 | 跨语言 |
| **布局** | 忽略 | 考虑视觉布局 |
| **图表** | 无法搜索 | 可以搜索 |
| **性能** | O(log N) | O(N)（但有 ANN 优化） |

## 总结

ColiVara 的搜索机制展现了以下特点：

1. **Late-Interaction**: Token 级别的细粒度匹配
2. **MaxSim**: 局部最佳匹配策略
3. **多模态**: 支持文本和图像查询
4. **灵活过滤**: 丰富的元数据过滤选项
5. **得分归一化**: 跨查询可比性
6. **性能优化**: 数据库索引、查询优化、批量处理
7. **视觉理解**: 无需 OCR 的文档搜索
