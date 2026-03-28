---
name: api-design
description: REST API 设计模式，包括资源命名、状态码、分页、过滤、错误响应、版本控制和速率限制，适用于生产环境 API。
origin: ECC
---

# API 设计模式

设计一致、对开发者友好的 REST API 的约定和最佳实践。

## 何时启用

- 设计新的 API 端点
- 审查现有的 API 契约
- 添加分页、过滤或排序
- 为 API 实现错误处理
- 规划 API 版本控制策略
- 构建面向公众或合作伙伴的 API

## 资源设计

### URL 结构

```
# Resources are nouns, plural, lowercase, kebab-case
GET    /api/v1/users
GET    /api/v1/users/:id
POST   /api/v1/users
PUT    /api/v1/users/:id
PATCH  /api/v1/users/:id
DELETE /api/v1/users/:id

# Sub-resources for relationships
GET    /api/v1/users/:id/orders
POST   /api/v1/users/:id/orders

# Actions that don't map to CRUD (use verbs sparingly)
POST   /api/v1/orders/:id/cancel
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
```

### 命名规则

```
# GOOD
/api/v1/team-members          # kebab-case for multi-word resources
/api/v1/orders?status=active  # query params for filtering
/api/v1/users/123/orders      # nested resources for ownership

# BAD
/api/v1/getUsers              # verb in URL
/api/v1/user                  # singular (use plural)
/api/v1/team_members          # snake_case in URLs
/api/v1/users/123/getOrders   # verb in nested resource
```

## HTTP 方法和状态码

### 方法语义

| Method | Idempotent | Safe | Use For |
|--------|-----------|------|---------|
| GET | Yes | Yes | 检索资源 |
| POST | No | No | 创建资源、触发动作 |
| PUT | Yes | No | 完整替换资源 |
| PATCH | No* | No | 部分更新资源 |
| DELETE | Yes | No | 删除资源 |

*PATCH 可以通过适当的实现实现幂等

### 状态码参考

```
# Success
200 OK                    — GET, PUT, PATCH (with response body)
201 Created               — POST (include Location header)
204 No Content            — DELETE, PUT (no response body)

# Client Errors
400 Bad Request           — 验证失败、格式错误的 JSON
401 Unauthorized          — 缺少或无效的身份验证
403 Forbidden             — 已验证但未授权
404 Not Found             — 资源不存在
409 Conflict              — 重复条目、状态冲突
422 Unprocessable Entity  — 语义无效（有效的 JSON，但数据错误）
429 Too Many Requests     — 超过速率限制

# Server Errors
500 Internal Server Error — 意外失败（绝不暴露细节）
502 Bad Gateway           — 上游服务失败
503 Service Unavailable   — 临时过载，包含 Retry-After
```

### 常见错误

```
# BAD: 对所有情况返回 200
{ "status": 200, "success": false, "error": "Not found" }

# GOOD: 语义化使用 HTTP 状态码
HTTP/1.1 404 Not Found
{ "error": { "code": "not_found", "message": "User not found" } }

# BAD: 对验证错误返回 500
# GOOD: 返回 400 或 422 并包含字段级详细信息

# BAD: 对创建的资源返回 200
# GOOD: 返回 201 并包含 Location header
HTTP/1.1 201 Created
Location: /api/v1/users/abc-123
```

## 响应格式

### 成功响应

```json
{
  "data": {
    "id": "abc-123",
    "email": "alice@example.com",
    "name": "Alice",
    "created_at": "2025-01-15T10:30:00Z"
  }
}
```

### 集合响应（带分页）

```json
{
  "data": [
    { "id": "abc-123", "name": "Alice" },
    { "id": "def-456", "name": "Bob" }
  ],
  "meta": {
    "total": 142,
    "page": 1,
    "per_page": 20,
    "total_pages": 8
  },
  "links": {
    "self": "/api/v1/users?page=1&per_page=20",
    "next": "/api/v1/users?page=2&per_page=20",
    "last": "/api/v1/users?page=8&per_page=20"
  }
}
```

### 错误响应

```json
{
  "error": {
    "code": "validation_error",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address",
        "code": "invalid_format"
      },
      {
        "field": "age",
        "message": "Must be between 0 and 150",
        "code": "out_of_range"
      }
    ]
  }
}
```

### 响应包装变体

```typescript
// Option A: 带数据包装的包装器（推荐用于公共 API）
interface ApiResponse<T> {
  data: T;
  meta?: PaginationMeta;
  links?: PaginationLinks;
}

interface ApiError {
  error: {
    code: string;
    message: string;
    details?: FieldError[];
  };
}

// Option B: 扁平响应（更简单，常见于内部 API）
// Success: 直接返回资源
// Error: 返回错误对象
// 通过 HTTP 状态码区分
```

## 分页

### 基于偏移（简单）

```
GET /api/v1/users?page=2&per_page=20

# Implementation
SELECT * FROM users
ORDER BY created_at DESC
LIMIT 20 OFFSET 20;
```

**Pros:** 易于实现，支持"跳转到第 N 页"
**Cons:** 在大偏移时慢（OFFSET 100000），并发插入时不一致

### 基于游标（可扩展）

```
GET /api/v1/users?cursor=eyJpZCI6MTIzfQ&limit=20

# Implementation
SELECT * FROM users
WHERE id > :cursor_id
ORDER BY id ASC
LIMIT 21;  -- 额外获取一条以确定 has_next
```

```json
{
  "data": [...],
  "meta": {
    "has_next": true,
    "next_cursor": "eyJpZCI6MTQzfQ"
  }
}
```

**Pros:** 无论位置如何性能一致，并发插入稳定
**Cons:** 无法跳转到任意页面，游标是不透明的

### 何时使用哪种分页

| Use Case | Pagination Type |
|----------|----------------|
| Admin dashboards, small datasets (<10K) | Offset |
| Infinite scroll, feeds, large datasets | Cursor |
| Public APIs | Cursor (default) with offset (optional) |
| Search results | Offset (users expect page numbers) |

## 过滤、排序和搜索

### 过滤

```
# Simple equality
GET /api/v1/orders?status=active&customer_id=abc-123

# Comparison operators (use bracket notation)
GET /api/v1/products?price[gte]=10&price[lte]=100
GET /api/v1/orders?created_at[after]=2025-01-01

# Multiple values (comma-separated)
GET /api/v1/products?category=electronics,clothing

# Nested fields (dot notation)
GET /api/v1/orders?customer.country=US
```

### 排序

```
# Single field (prefix - for descending)
GET /api/v1/products?sort=-created_at

# Multiple fields (comma-separated)
GET /api/v1/products?sort=-featured,price,-created_at
```

### 全文搜索

```
# Search query parameter
GET /api/v1/products?q=wireless+headphones

# Field-specific search
GET /api/v1/users?email=alice
```

### 稀疏字段集

```
# Return only specified fields (reduces payload)
GET /api/v1/users?fields=id,name,email
GET /api/v1/orders?fields=id,total,status&include=customer.name
```

## 身份验证和授权

### 基于 Token 的身份验证

```
# Bearer token in Authorization header
GET /api/v1/users
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

# API key (for server-to-server)
GET /api/v1/data
X-API-Key: sk_live_abc123
```

### 授权模式

```typescript
// Resource-level: check ownership
app.get("/api/v1/orders/:id", async (req, res) => {
  const order = await Order.findById(req.params.id);
  if (!order) return res.status(404).json({ error: { code: "not_found" } });
  if (order.userId !== req.user.id) return res.status(403).json({ error: { code: "forbidden" } });
  return res.json({ data: order });
});

// Role-based: check permissions
app.delete("/api/v1/users/:id", requireRole("admin"), async (req, res) => {
  await User.delete(req.params.id);
  return res.status(204).send();
});
```

## 速率限制

### 响应头

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640000000

# When exceeded
HTTP/1.1 429 Too Many Requests
Retry-After: 60
{
  "error": {
    "code": "rate_limit_exceeded",
    "message": "Rate limit exceeded. Try again in 60 seconds."
  }
}
```

### 速率限制层级

| Tier | Limit | Window | Use Case |
|------|-------|--------|----------|
| Anonymous | 30/min | Per IP | Public endpoints |
| Authenticated | 100/min | Per user | Standard API access |
| Premium | 1000/min | Per API key | Paid API plans |
| Internal | 10000/min | Per service | Service-to-service |

## 版本控制

### URL 路径版本控制（推荐）

```
/api/v1/users
/api/v2/users
```

**Pros:** 显式、易于路由、可缓存
**Cons:** 版本之间 URL 变化

### 请求头版本控制

```
GET /api/users
Accept: application/vnd.myapp.v2+json
```

**Pros:** 干净的 URL
**Cons:** 更难测试，容易忘记

### 版本控制策略

```
1. 从 /api/v1/ 开始 — 直到需要时才版本化
2. 最多维护 2 个活动版本（当前 + 前一个）
3. 弃用时间表：
   - 宣布弃用（公共 API 提前 6 个月通知）
   - 添加 Sunset header: Sunset: Sat, 01 Jan 2026 00:00:00 GMT
   - 在弃用日期后返回 410 Gone
4. 非破坏性更改不需要新版本：
   - 向响应添加新字段
   - 添加新的可选查询参数
   - 添加新的端点
5. 破坏性更改需要新版本：
   - 删除或重命名字段
   - 更改字段类型
   - 更改 URL 结构
   - 更改身份验证方法
```

## 实现模式

### TypeScript (Next.js API Route)

```typescript
import { z } from "zod";
import { NextRequest, NextResponse } from "next/server";

const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
});

export async function POST(req: NextRequest) {
  const body = await req.json();
  const parsed = createUserSchema.safeParse(body);

  if (!parsed.success) {
    return NextResponse.json({
      error: {
        code: "validation_error",
        message: "Request validation failed",
        details: parsed.error.issues.map(i => ({
          field: i.path.join("."),
          message: i.message,
          code: i.code,
        })),
      },
    }, { status: 422 });
  }

  const user = await createUser(parsed.data);

  return NextResponse.json(
    { data: user },
    {
      status: 201,
      headers: { Location: `/api/v1/users/${user.id}` },
    },
  );
}
```

### Python (Django REST Framework)

```python
from rest_framework import serializers, viewsets, status
from rest_framework.response import Response

class CreateUserSerializer(serializers.Serializer):
    email = serializers.EmailField()
    name = serializers.CharField(max_length=100)

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["id", "email", "name", "created_at"]

class UserViewSet(viewsets.ModelViewSet):
    serializer_class = UserSerializer
    permission_classes = [IsAuthenticated]

    def get_serializer_class(self):
        if self.action == "create":
            return CreateUserSerializer
        return UserSerializer

    def create(self, request):
        serializer = CreateUserSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = UserService.create(**serializer.validated_data)
        return Response(
            {"data": UserSerializer(user).data},
            status=status.HTTP_201_CREATED,
            headers={"Location": f"/api/v1/users/{user.id}"},
        )
```

### Go (net/http)

```go
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "invalid_json", "Invalid request body")
        return
    }

    if err := req.Validate(); err != nil {
        writeError(w, http.StatusUnprocessableEntity, "validation_error", err.Error())
        return
    }

    user, err := h.service.Create(r.Context(), req)
    if err != nil {
        switch {
        case errors.Is(err, domain.ErrEmailTaken):
            writeError(w, http.StatusConflict, "email_taken", "Email already registered")
        default:
            writeError(w, http.StatusInternalServerError, "internal_error", "Internal error")
        }
        return
    }

    w.Header().Set("Location", fmt.Sprintf("/api/v1/users/%s", user.ID))
    writeJSON(w, http.StatusCreated, map[string]any{"data": user})
}
```

## API 设计检查清单

在发布新端点之前：

- [ ] Resource URL 遵循命名约定（复数、kebab-case、无动词）
- [ ] 使用正确的 HTTP 方法（GET 用于读取，POST 用于创建等）
- [ ] 返回适当的状态码（不是对所有情况返回 200）
- [ ] 使用 schema 验证输入（Zod、Pydantic、Bean Validation）
- [ ] 错误响应遵循标准格式，包含代码和消息
- [ ] 列表端点实现分页（游标或偏移）
- [ ] 需要身份验证（或明确标记为公共）
- [ ] 检查授权（用户只能访问自己的资源）
- [ ] 配置速率限制
- [ ] 响应不泄露内部细节（堆栈跟踪、SQL 错误）
- [ ] 与现有端点命名一致（camelCase vs snake_case）
- [ ] 已文档化（OpenAPI/Swagger 规范已更新）
