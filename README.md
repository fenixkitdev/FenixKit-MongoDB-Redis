# RAPID by FenixKit — Repository · API · Persistence · Instant · .NET

<p align="center">
  <a href="https://fenixkit.dev/kits/rapid/">
    <img src="https://fenixkit.dev/assets/kits/banners/rapid.png" alt="RAPID by FenixKit" width="100%" />
  </a>
</p>
<p align="center">
  <img src="https://fenixkit.dev/assets/kits/logos/rapid.png" alt="RAPID" width="200" />
</p>
<h3 align="center">
  <a href="https://fenixkit.dev/kits/rapid/">fenixkit.dev/kits/rapid/</a>
</h3>

> **RAPID — Repository · API · Persistence · Instant · .NET**
> A MongoDB + Redis cache-aside .NET Minimal API template — tag-based invalidation, FailOpen resilience, dual pagination, hook-based repository — production-ready from day one.

## What's Inside

| Feature | Details |
|---|---|
| **Architecture** | .NET 8 / .NET 10 Minimal API — no controllers, faster startup |
| **Database** | MongoDB with a full abstraction layer |
| **Cache** | Redis cache-aside, tag-based invalidation, 3 control levels |
| **Error handling** | ErrorOr v2 result pattern + RFC 7807 ProblemDetails |
| **Pagination** | Offset-based and cursor-based, both included |
| **Repository** | `BaseRepository` with 7 overridable hooks + automatic cache integration |
| **Observability** | Health checks (MongoDB + Redis), X-Response-Time header |
| **API Docs** | Full Swagger/OpenAPI with XML doc comments |
| **Infrastructure** | Multi-stage Dockerfile + Docker Compose (api + mongodb + redis) |

---

## Why Not Start from Scratch?

Every new .NET API project faces the same decisions: how to structure routes, how to handle validation errors consistently, how to abstract the database, how to paginate efficiently, when to cache and how to keep it consistent. Getting these wrong early means painful refactors later.

| Starting from scratch | Using FenixKit |
|---|---|
| 2–5 days wiring project structure | Configure connection strings, run |
| Roll your own error-handling strategy | ErrorOr v2 result pattern wired in from day 1 |
| Write MongoDB boilerplate per collection | Generic `IDBRepository` — one interface, all collections |
| Implement pagination, debug edge cases | Offset + cursor pagination included |
| Copy-paste logic across repositories | 7 virtual hooks — extend CRUD without rewriting it |
| Redis cache from scratch — keys, tags, TTLs, fallback | 3-level tag cache wired into `BaseRepository` automatically |
| Swagger setup as an afterthought | Full OpenAPI with XML doc comments from the start |
| Docker setup varies per developer | Dockerfile + Docker Compose (api + mongo + redis) ready to go |

---

## Project Structure

```
MyApi/
├── Cache/
│   ├── ICacheService.cs          # Get / Set / Invalidate / InvalidateByTag(s)
│   ├── CacheOptions.cs           # Enabled, TTL, OperationTimeoutMs, ErrorBehavior (FailOpen / FailClosed)
│   ├── RedisCacheService.cs      # STRING values + SET tag indexes in Redis
│   ├── NullCacheService.cs       # No-op — injected when Cache:Enabled = false
│   └── Extensions/
│       └── CacheExtensions.cs    # AddRedisCache() DI extension
├── Common/
│   └── Models/                   # PagedResult<T>, CursorPagedResult<T>
├── Database/
│   └── Persistence/              # IDBRepository + MongoRepository
├── Domain/
│   ├── Entities/                 # BaseEntity, ICollectionEntity, Product
│   ├── Requests/                 # ICreateRequest<T>, IUpdateRequest<T>, DTOs
│   └── Responses/                # SummaryResponse, DetailResponse
├── Endpoints/                    # ProductEndpoints (Minimal API route groups)
├── Errors/                       # CacheErrors, MongoErrors, PageErrors, ProductErrors
├── HealthChecks/                 # MongoHealthCheck, RedisHealthCheck
├── Repositories/
│   ├── Base/                     # IBaseRepository, BaseRepository (7 hooks + cache)
│   └── ProductRepository
├── Program.cs
├── docker-compose.yml
└── Dockerfile
```

---

## Architecture

### Minimal API — No Controllers

Routes are declared as static methods grouped under `MapGroup`, keeping all routes for a resource co-located in one file. Less ceremony, faster startup, zero reflection overhead from controller discovery.

```csharp
public static class ProductEndpoints
{
    public static void MapProductEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/products").WithTags("Products");

        group.MapPost("list/",       GetAll);        // offset pagination
        group.MapPost("listcursor/", GetAllCursor);  // cursor pagination
        group.MapGet("/{id}",        GetById);
        group.MapPost("/",           Create);
        group.MapPut("/",            Update);
        group.MapDelete("/{id}",     Delete);
    }
}
```

### MongoDB Abstraction Layer

`IDBRepository` is the single interface all repositories talk to. It hides the MongoDB driver completely.

| Method | Description |
|---|---|
| `GetByIdAsync<T>` | Fetch a single document by ObjectId |
| `GetPagedAsync<T>` | Offset pagination — `TotalCount`, `TotalPages` |
| `GetPagedByCursorAsync<T>` | Cursor pagination using the `_id` index — O(log n) |
| `FindAsync<T>` | Filter documents by any predicate expression |
| `ExistsAsync<T>` | Check existence without fetching the full document |
| `CreateAsync<T>` | Insert and return the created entity |
| `UpdateAsync<T>` | Replace an existing document by Id |
| `DeleteAsync<T>` | Remove a document by Id |

---

## Redis Cache Layer

The cache follows the **cache-aside pattern**: the repository checks Redis before querying MongoDB, and populates Redis after a miss. Invalidation is tag-based — every cached entry is registered under one or more tags, and writing an entity wipes all entries under its tags.

### Three Levels of Control

| Level | Mechanism | Who uses it |
|---|---|---|
| **3 — Automatic** | `BaseRepository` calls `GetInvalidationTags` and wipes tags after every write | No code needed — always on |
| **2 — Tag-based** | `_Cache.InvalidateByTagAsync("product:category:Electronics")` | Derived repository — for custom domain queries |
| **1 — Manual** | `_Cache.InvalidateAsync("product:abc123")` | Derived repository — surgical single-key removal |

### Storage Layout in Redis

```
myapi:product:abc123              STRING   JSON of ProductDetailResponse   TTL 5 min
myapi:product:paged:p1:s20        STRING   JSON of PagedResult<...>         TTL 5 min
myapi:product:cursor:start:20:fwd STRING   JSON of CursorPagedResult<...>   TTL 5 min
myapi:product:category:Gaming     STRING   JSON of List<ProductSummary>     TTL 5 min

myapi:tag:product                 SET      { all paged + cursor keys }        no TTL
myapi:tag:product:abc123          SET      { "myapi:product:abc123" }         no TTL
myapi:tag:product:category:Gaming SET      { "myapi:product:category:..." }   no TTL
```

Tag Sets have no TTL by design — they are deleted when `InvalidateByTagAsync` runs, leaving no orphaned entries.

### ErrorBehavior Options

| Mode | Redis unavailable | Recommended when |
|---|---|---|
| `FailOpen` (default) | Cache treated as miss — falls through to MongoDB, request succeeds | Redis is a performance layer |
| `FailClosed` | Returns `Error.Unexpected` — request fails with 500 | Cache correctness is required |

### Cache Is Optional

Set `Cache:Enabled = false` to run without Redis. A `NullCacheService` no-op is registered in place of `RedisCacheService`. `IConnectionMultiplexer` is never registered, so the Redis health check is also omitted. No code changes required.

### Overriding Cache Behaviour

Every cache key and invalidation tag is controlled by virtual hooks on `BaseRepository`:

```csharp
// ProductRepository.cs

protected override string GetCacheKey(string id)
    => $"product:{id}";

protected override IEnumerable<string> GetInvalidationTags(Product entity)
{
    yield return "product";                              // clears all paged + cursor pages
    yield return GetCacheKey(entity.Id);                // clears this product's by-ID entry
    yield return $"product:category:{entity.Category}"; // clears the category-filtered list
}
```

On an update that changes `Category`, `BaseRepository` automatically unions the tags from both the original and the updated entity — so both the old and new category caches are invalidated.

---

## The ErrorOr Result Pattern

Every repository method returns `ErrorOr<T>` instead of throwing exceptions for domain errors. The caller always receives either a value or a list of typed errors — no try/catch needed.

```csharp
// In the endpoint handler — one line, all cases handled
var result = await repo.GetByIdAsync(id, ct);

return result.Match(
    product => Results.Ok(product),
    errors  => errors.ToResponse()); // → RFC 7807 ProblemDetails
```

Domain errors are typed and centralised:

```csharp
public static class ProductErrors
{
    public static Error NotFound(string id) =>
        Error.NotFound("Product.NotFound", $"No product with id '{id}' was found.");

    public static Error NameConflict(string name) =>
        Error.Conflict("Product.NameConflict", $"A product named '{name}' already exists.");

    public static readonly Error InvalidPrice =
        Error.Validation("Product.InvalidPrice", "Price must be greater than zero.");
}
```

**HTTP mapping is automatic:**

| ErrorOr type | HTTP Status |
|---|---|
| `Error.Validation` | 422 Unprocessable Entity |
| `Error.NotFound` | 404 Not Found |
| `Error.Conflict` | 409 Conflict |
| `Error.Unexpected` | 500 Internal Server Error |

---

## The BaseRepository Hook System

`BaseRepository` is the centrepiece of the kit. It implements all CRUD operations and exposes **7 virtual hook methods** that you override to inject domain logic — without touching the CRUD algorithm itself.

> **Template Method Pattern** — the base class defines the algorithm: validate → map → persist → project → cache. You override only the steps your domain needs. You never rewrite CRUD.

### The Seven Domain Hooks

| Hook | Called by | Purpose |
|---|---|---|
| `OnValidateCreateAsync` | `CreateAsync` | Validate before touching the database |
| `OnValidateUpdateAsync` | `UpdateAsync` | Validate after enrichment — sees final entity |
| `OnValidateDeleteAsync` | `DeleteAsync` | Block deletion if business rules require it |
| `OnMapCreateAsync` | `CreateAsync` | Set server-computed fields; abort before DB insert |
| `OnMapUpdateAsync` | `UpdateAsync` | Recompute derived fields; abort before validation |
| `OnMapToSummaryAsync` | `GetPagedAsync`, `GetPagedByCursorAsync` | Project entity → lightweight list DTO |
| `OnMapToDetailAsync` | `GetByIdAsync`, `CreateAsync` | Project entity → full detail DTO |

### The Four Cache Hooks

| Hook | Purpose |
|---|---|
| `GetCacheKey(id)` | Key for by-ID detail entries |
| `GetPagedCacheKey(input)` | Key for offset-paged results |
| `GetCursorCacheKey(input)` | Key for cursor-paged results |
| `GetInvalidationTags(entity)` | Tags to wipe on create / update / delete |

### Hook Execution Order — CreateAsync

```
1. OnValidateCreateAsync  — price > 0, category not empty, name unique
2. request.ToDBEntity()   — base field mapping
3. OnMapCreateAsync       — server computes Slug; abort here to skip DB insert
4. IDBRepository.CreateAsync  — MongoDB insertOne
5. OnMapToDetailAsync     — map to ProductDetailResponse, add FormattedPrice
6. Cache invalidation     — wipe tags from GetInvalidationTags(created)
7. 201 Created returned to client
```

### Real-World Hook Examples

**Server-side computed field — Slug**

```csharp
protected override Task<ErrorOr<Product>> OnMapCreateAsync(
    ProductCreateRequest request, Product entity, CancellationToken ct = default)
{
    entity.Slug = ToSlug(request.Name);  // never accepted from the client
    return Task.FromResult<ErrorOr<Product>>(entity);
}

protected override Task<ErrorOr<Product>> OnMapUpdateAsync(
    ProductUpdateRequest request, Product original, Product entity, CancellationToken ct = default)
{
    entity.Slug = request.Name != original.Name
        ? ToSlug(request.Name)   // recompute if name changed
        : original.Slug;         // preserve if name unchanged
    return Task.FromResult<ErrorOr<Product>>(entity);
}
```

**Presentation-only computed field — FormattedPrice**

```csharp
protected override Task<ErrorOr<ProductDetailResponse>> OnMapToDetailAsync(
    Product entity, CancellationToken ct = default)
{
    var response = ProductDetailResponse.From(entity);
    response.FormattedPrice = $"€{entity.Price:F2}";  // not stored in MongoDB
    return Task.FromResult<ErrorOr<ProductDetailResponse>>(response);
}
```

Because `FormattedPrice` is never stored, changing currency or locale requires no database migration — just update the hook.

---

## Summary / Detail Response Split

Returning the same large DTO for both list and single-item endpoints wastes bandwidth. `BaseRepository` enforces a two-projection architecture:

| Endpoint | Response Type | Fields |
|---|---|---|
| `POST /api/products/list/` | `ProductSummaryResponse` | Id, Name, Category, Price |
| `POST /api/products/listcursor/` | `ProductSummaryResponse` | Id, Name, Category, Price |
| `GET /api/products/{id}` | `ProductDetailResponse` | All fields + Slug + FormattedPrice |
| `POST /api/products/` | `ProductDetailResponse` | All fields + Slug + FormattedPrice |

Both DTOs are fully decoupled from the MongoDB document. Internal schema changes don't break your API contract — only the `From()` factory method changes.

---

## Pagination

### Offset-Based

Classic `page + pageSize` pagination. Returns `TotalCount` and `TotalPages` so the client can render numbered page navigation. Best for admin dashboards and backoffice UIs.

```json
// POST /api/products/list/
{ "page": 2, "pageSize": 20 }

// Response
{
  "items": [...],
  "page": 2,
  "pageSize": 20,
  "totalCount": 87,
  "totalPages": 5,
  "hasNext": true,
  "hasPrev": true
}
```

### Cursor-Based

Uses the MongoDB `_id` B-tree index directly. Instead of `Skip(N)`, the query filters `_id > cursor` — an **O(log n) indexed range scan** at any collection size. No duplicate items when documents are inserted between page loads.

```json
{ "cursor": "6641f3a2b1c2d3e4f5a6b7c8", "pageSize": 20, "forward": true }

// Response
{
  "items": [...],
  "nextCursor": "6641f3a2b1c2d3e4f5a6b7c8",
  "prevCursor": "6641f3a2b1c2d3e4f5a6b7c1",
  "hasNext": true,
  "hasPrev": false
}
```

| | Offset | Cursor |
|---|---|---|
| Jump to any page | Yes | No |
| TotalCount available | Yes | No |
| Consistent under concurrent writes | No | Yes |
| Performance at page 1 | O(1) | O(log n) |
| Performance at page 50 000 | O(n) skip | O(log n) always |
| Best for | Admin UIs, numbered navigation | Feeds, infinite scroll, large datasets |

Both are included — pick the right one per endpoint, or expose both.

---

## Error Handling

### RFC 7807 Problem Details on every error

All error responses are serialised as standard `ProblemDetails` JSON. Clients receive a consistent `type`, `title`, `status`, and `detail` on every error:

```json
// 404 Not Found
{
  "status": 404,
  "title": "Product.NotFound",
  "detail": "No product with id '6638f1a2b3c4d5e6f7a8b9c0' was found.",
  "traceId": "0HN5KQVJQVJQVJQV"
}

// 422 Validation — all errors returned at once
{
  "status": 422,
  "errors": {
    "Product.InvalidPrice": ["Price must be greater than zero."],
    "Product.InvalidCategory": ["Category cannot be empty."]
  }
}
```

### Global Exception Handler

Unhandled exceptions are caught and serialised as `500 Internal Server Error ProblemDetails` — clients always receive consistent JSON, never an HTML error page or a raw stack trace.

---

## Observability & Infrastructure

### Health Checks

```
GET /health/live   → Liveness probe  — is the process running?
GET /health/ready  → Readiness probe — is MongoDB reachable? Is Redis reachable?
```

Redis is excluded from `/health/ready` when `Cache:Enabled = false`. Both probes return structured JSON:

```json
{
  "status": "Healthy",
  "duration": "00:00:00.0042310",
  "entries": {
    "mongodb": { "status": "Healthy", "duration": "00:00:00.0041120" },
    "redis":   { "status": "Healthy", "duration": "00:00:00.0008540" }
  }
}
```

### X-Response-Time Header


### Docker

```bash
docker compose up --build

# API      → http://localhost:8081
# Swagger  → http://localhost:8081/swagger
# MongoDB  → localhost:27018
# Redis    → localhost:6379
```

---

## Adding a New Entity

Adding a new resource follows the same pattern every time. `BaseRepository` is never modified.

```
1. Domain/Entities/Order.cs          — inherits BaseEntity, implements ICollectionEntity
2. Domain/Requests/Orders/           — OrderCreateRequest, OrderUpdateRequest
3. Domain/Responses/Orders/          — OrderSummaryResponse, OrderDetailResponse
4. Repositories/IOrderRepository.cs  — extends IBaseRepository<Order, ...>
5. Repositories/OrderRepository.cs   — extends BaseRepository<Order, ...>, override hooks
6. Endpoints/OrderEndpoints.cs       — Minimal API route group
7. Errors/OrderErrors.cs             — typed domain errors for Order
8. Program.cs                        — AddSingleton<IOrderRepository, OrderRepository>()
```

Custom domain queries are added directly to the concrete repository:

```csharp
public async Task<ErrorOr<List<OrderSummaryResponse>>> GetByCustomerAsync(
    string customerId, CancellationToken ct = default)
{
    var cacheKey = $"order:customer:{customerId}";
    if (_Cache is not null)
    {
        var cached = await _Cache.GetAsync<List<OrderSummaryResponse>>(cacheKey, ct);
        if (cached.IsError) return cached.Errors;
        if (cached.Value is not null) return cached.Value;
    }

    var results = await _Repository.FindAsync<Order>(o => o.CustomerId == customerId, ct);
    if (results.IsError) return results.Errors;

    var summaries = new List<OrderSummaryResponse>();
    foreach (var entity in results.Value)
    {
        var mapped = await OnMapToSummaryAsync(entity, ct);
        if (mapped.IsError) return mapped.Errors;
        summaries.Add(mapped.Value);
    }

    if (_Cache is not null)
    {
        await _Cache.SetAsync(cacheKey, summaries,
            ["order", $"order:customer:{customerId}"], ct: ct);
    }

    return summaries;
}
```

---

## Getting Started

### Prerequisites

| Requirement | Minimum |
|---|---|
| .NET SDK | 8.0 LTS or 10.0 |
| MongoDB | 6.x+ (or use Docker Compose) |
| Redis | 8.x · or Valkey 7.2+ (or use Docker Compose) |
| Docker Desktop | 4.x (optional) |

### Setup

```bash
# 1. Unzip the template
# 2. Copy .env.example to .env and fill in credentials
cp .env.example .env

# 3. Restore packages
dotnet restore

# 4. Run with Docker (recommended)
docker compose up --build

# — or run locally —
dotnet run
```

### First Run Checklist

- `GET /health/live` → `{ "status": "Healthy" }`
- `GET /health/ready` → Healthy with both `mongodb` and `redis` entries
- `POST /api/products/` with valid body → `FormattedPrice` and `Slug` in response
- `POST /api/products/list/` → paged results with `totalCount`
- `POST /api/products/listcursor/` → cursor results with `nextCursor`
- `POST /api/products/list/` called twice → second `X-Response-Time` is noticeably lower
- `PUT /api/products/` → next `list` call shows higher `X-Response-Time` again (cache busted)
- `GET /api/products/{id}` with unknown id → `404 ProblemDetails`
- `POST /api/products/` with duplicate name → `409 ProblemDetails`

Once the checklist passes, rename or delete the Product files and replace them with your own domain entities.

---

## Technologies

| Package | Version | Role |
|---|---|---|
| .NET 8 LTS (C# 12) · .NET 10 (C# 14) | — | Runtime, SDK and language |
| MongoDB.Driver | — | Official MongoDB .NET driver |
| Redis 8 / Valkey 7.2+ | — | Cache server — compatible with both (`docker-compose.valkey.yml` included) |
| StackExchange.Redis | — | Redis client — `IConnectionMultiplexer` singleton |
| ErrorOr | — | Result pattern — no exceptions for domain errors |
| Swashbuckle.AspNetCore | — | Swagger UI and OpenAPI spec generation |
| DotNetEnv | — | `.env` file loading at startup |
| Steeltoe.Configuration.Placeholder | — | `${VAR}` placeholder resolution in config |
| Microsoft.Extensions.Diagnostics.HealthChecks | (built-in) | Liveness and readiness probe infrastructure |
| Docker + Docker Compose | — | api + mongodb + redis in one command |

---

## License

FenixKit is a commercial product. Each purchase grants a lifetime licence for unlimited personal and commercial projects.

👉 **[fenixkit.dev](https://fenixkit.dev)**


