---
name: cqrs-query-generator
description: "Generates CQRS Queries with Handlers and Response DTOs for read operations. Uses Dapper for optimized read queries, bypassing the domain model for better performance."
version: 2.0.0
language: C#
framework: .NET 8+
dependencies: Kommand, Dapper
---

# CQRS Query Generator

## Overview

This skill generates Queries following the CQRS pattern using **Kommand** - a lightweight CQRS mediator for .NET 8+ with built-in OpenTelemetry support. Queries are read-only operations that return data without modifying state. Key principles:

- **Queries never modify state** - Read-only operations
- **Use Dapper for reads** - Bypass EF Core for performance
- **Return DTOs, not entities** - Projection to response models
- **Direct SQL queries** - Optimized for the specific use case

## Quick Reference

| Query Type | Use Case | Returns |
|------------|----------|---------|
| GetById | Single entity by ID | `EntityResponse` or `null` |
| GetAll | All entities (with optional filtering) | `IReadOnlyList<EntityResponse>` |
| GetPaged | Paginated list | `PagedList<EntityResponse>` |
| Search | Filtered/searched results | `IReadOnlyList<EntityResponse>` |
| Exists | Check if entity exists | `bool` |

---

## Query Structure

```
/Application/{Feature}/
├── Get{Entity}ById/
│   ├── Get{Entity}ByIdQuery.cs       # Query + Handler
│   ├── Get{Entity}ByIdQueryValidator.cs
│   └── {Entity}Response.cs            # Response DTO
├── GetAll{Entities}/
│   ├── GetAll{Entities}Query.cs
│   └── {Entity}ListResponse.cs
└── Get{Entities}ByOrganization/
    ├── Get{Entities}ByOrganizationQuery.cs
    └── {Entity}ByOrganizationResponse.cs
```

---

## Template: Get By ID Query

```csharp
// src/{name}.application/{Feature}/Get{Entity}ById/Get{Entity}ByIdQuery.cs
using System.Data;
using Dapper;
using Kommand;
using {name}.application.abstractions.data;

namespace {name}.application.{feature}.Get{Entity}ById;

// ═══════════════════════════════════════════════════════════════
// QUERY RECORD
// ═══════════════════════════════════════════════════════════════
public sealed record Get{Entity}ByIdQuery(Guid Id) : IQuery<{Entity}Response?>;

// ═══════════════════════════════════════════════════════════════
// HANDLER
// ═══════════════════════════════════════════════════════════════
internal sealed class Get{Entity}ByIdQueryHandler
    : IQueryHandler<Get{Entity}ByIdQuery, {Entity}Response?>
{
    private readonly ISqlConnectionFactory _sqlConnectionFactory;

    public Get{Entity}ByIdQueryHandler(ISqlConnectionFactory sqlConnectionFactory)
    {
        _sqlConnectionFactory = sqlConnectionFactory;
    }

    public async Task<{Entity}Response?> HandleAsync(
        Get{Entity}ByIdQuery query,
        CancellationToken cancellationToken)
    {
        using IDbConnection connection = _sqlConnectionFactory.CreateConnection();

        const string sql = """
            SELECT
                e.id AS Id,
                e.name AS Name,
                e.description AS Description,
                e.created_at AS CreatedAt,
                e.updated_at AS UpdatedAt
            FROM {table_name} e
            WHERE e.id = @Id
            """;

        var {entity} = await connection.QueryFirstOrDefaultAsync<{Entity}Response>(
            sql,
            new { query.Id });

        return {entity};
    }
}
```

### Query Validator

```csharp
// src/{name}.application/{Feature}/Get{Entity}ById/Get{Entity}ByIdQueryValidator.cs
using Kommand.Validation;

namespace {name}.application.{feature}.Get{Entity}ById;

internal sealed class Get{Entity}ByIdQueryValidator : IValidator<Get{Entity}ByIdQuery>
{
    public Task<ValidationResult> ValidateAsync(
        Get{Entity}ByIdQuery query,
        CancellationToken cancellationToken)
    {
        var errors = new List<ValidationError>();

        if (query.Id == Guid.Empty)
        {
            errors.Add(new ValidationError("Id", "{Entity} ID is required", "REQUIRED"));
        }

        return Task.FromResult(errors.Count > 0
            ? ValidationResult.Failure(errors)
            : ValidationResult.Success());
    }
}
```

### Response DTO

```csharp
// src/{name}.application/{Feature}/Get{Entity}ById/{Entity}Response.cs
namespace {name}.application.{feature}.Get{Entity}ById;

public sealed class {Entity}Response
{
    public Guid Id { get; init; }
    public required string Name { get; init; }
    public string? Description { get; init; }
    public DateTime CreatedAt { get; init; }
    public DateTime UpdatedAt { get; init; }
}
```

---

## Template: Get All Query

```csharp
// src/{name}.application/{Feature}/GetAll{Entities}/GetAll{Entities}Query.cs
using System.Data;
using Dapper;
using Kommand;
using {name}.application.abstractions.data;

namespace {name}.application.{feature}.GetAll{Entities};

public sealed record GetAll{Entities}Query : IQuery<IReadOnlyList<{Entity}ListResponse>>;

internal sealed class GetAll{Entities}QueryHandler
    : IQueryHandler<GetAll{Entities}Query, IReadOnlyList<{Entity}ListResponse>>
{
    private readonly ISqlConnectionFactory _sqlConnectionFactory;

    public GetAll{Entities}QueryHandler(ISqlConnectionFactory sqlConnectionFactory)
    {
        _sqlConnectionFactory = sqlConnectionFactory;
    }

    public async Task<IReadOnlyList<{Entity}ListResponse>> HandleAsync(
        GetAll{Entities}Query query,
        CancellationToken cancellationToken)
    {
        using IDbConnection connection = _sqlConnectionFactory.CreateConnection();

        const string sql = """
            SELECT
                e.id AS Id,
                e.name AS Name,
                e.description AS Description
            FROM {table_name} e
            ORDER BY e.name ASC
            """;

        var {entities} = await connection.QueryAsync<{Entity}ListResponse>(sql);

        return {entities}.ToList();
    }
}
```

---

## Template: Get By Parent ID Query

```csharp
// src/{name}.application/{Feature}/Get{Entities}ByOrganizationId/Get{Entities}ByOrganizationIdQuery.cs
using System.Data;
using Dapper;
using Kommand;
using {name}.application.abstractions.data;

namespace {name}.application.{feature}.Get{Entities}ByOrganizationId;

public sealed record Get{Entities}ByOrganizationIdQuery(
    Guid OrganizationId) : IQuery<IReadOnlyList<{Entity}Response>>;

internal sealed class Get{Entities}ByOrganizationIdQueryHandler
    : IQueryHandler<Get{Entities}ByOrganizationIdQuery, IReadOnlyList<{Entity}Response>>
{
    private readonly ISqlConnectionFactory _sqlConnectionFactory;

    public Get{Entities}ByOrganizationIdQueryHandler(ISqlConnectionFactory sqlConnectionFactory)
    {
        _sqlConnectionFactory = sqlConnectionFactory;
    }

    public async Task<IReadOnlyList<{Entity}Response>> HandleAsync(
        Get{Entities}ByOrganizationIdQuery query,
        CancellationToken cancellationToken)
    {
        using IDbConnection connection = _sqlConnectionFactory.CreateConnection();

        const string sql = """
            SELECT
                e.id AS Id,
                e.name AS Name,
                e.description AS Description,
                e.organization_id AS OrganizationId,
                o.name AS OrganizationName
            FROM {table_name} e
            INNER JOIN organization o ON e.organization_id = o.id
            WHERE e.organization_id = @OrganizationId
            ORDER BY e.name ASC
            """;

        var {entities} = await connection.QueryAsync<{Entity}Response>(
            sql,
            new { query.OrganizationId });

        return {entities}.ToList();
    }
}
```

### Query Validator

```csharp
// src/{name}.application/{Feature}/Get{Entities}ByOrganizationId/Get{Entities}ByOrganizationIdQueryValidator.cs
using Kommand.Validation;

namespace {name}.application.{feature}.Get{Entities}ByOrganizationId;

internal sealed class Get{Entities}ByOrganizationIdQueryValidator
    : IValidator<Get{Entities}ByOrganizationIdQuery>
{
    public Task<ValidationResult> ValidateAsync(
        Get{Entities}ByOrganizationIdQuery query,
        CancellationToken cancellationToken)
    {
        var errors = new List<ValidationError>();

        if (query.OrganizationId == Guid.Empty)
        {
            errors.Add(new ValidationError("OrganizationId", "Organization ID is required"));
        }

        return Task.FromResult(errors.Count > 0
            ? ValidationResult.Failure(errors)
            : ValidationResult.Success());
    }
}
```

---

## Template: Paginated Query

```csharp
// src/{name}.application/{Feature}/Get{Entities}Paged/Get{Entities}PagedQuery.cs
using System.Data;
using Dapper;
using Kommand;
using {name}.application.abstractions.data;

namespace {name}.application.{feature}.Get{Entities}Paged;

public sealed record Get{Entities}PagedQuery(
    int PageNumber,
    int PageSize,
    string? SearchTerm = null) : IQuery<PagedList<{Entity}Response>>;

internal sealed class Get{Entities}PagedQueryHandler
    : IQueryHandler<Get{Entities}PagedQuery, PagedList<{Entity}Response>>
{
    private readonly ISqlConnectionFactory _sqlConnectionFactory;

    public Get{Entities}PagedQueryHandler(ISqlConnectionFactory sqlConnectionFactory)
    {
        _sqlConnectionFactory = sqlConnectionFactory;
    }

    public async Task<PagedList<{Entity}Response>> HandleAsync(
        Get{Entities}PagedQuery query,
        CancellationToken cancellationToken)
    {
        using IDbConnection connection = _sqlConnectionFactory.CreateConnection();

        var offset = (query.PageNumber - 1) * query.PageSize;
        var searchPattern = query.SearchTerm is not null
            ? $"%{query.SearchTerm}%"
            : null;

        const string countSql = """
            SELECT COUNT(*)
            FROM {table_name} e
            WHERE (@SearchTerm IS NULL OR e.name ILIKE @SearchTerm)
            """;

        const string dataSql = """
            SELECT
                e.id AS Id,
                e.name AS Name,
                e.description AS Description,
                e.created_at AS CreatedAt
            FROM {table_name} e
            WHERE (@SearchTerm IS NULL OR e.name ILIKE @SearchTerm)
            ORDER BY e.created_at DESC
            OFFSET @Offset ROWS
            FETCH NEXT @PageSize ROWS ONLY
            """;

        var totalCount = await connection.ExecuteScalarAsync<int>(
            countSql,
            new { SearchTerm = searchPattern });

        var items = await connection.QueryAsync<{Entity}Response>(
            dataSql,
            new
            {
                SearchTerm = searchPattern,
                Offset = offset,
                query.PageSize
            });

        return new PagedList<{Entity}Response>(
            items.ToList(),
            query.PageNumber,
            query.PageSize,
            totalCount);
    }
}
```

### Paged Query Validator

```csharp
// src/{name}.application/{Feature}/Get{Entities}Paged/Get{Entities}PagedQueryValidator.cs
using Kommand.Validation;

namespace {name}.application.{feature}.Get{Entities}Paged;

internal sealed class Get{Entities}PagedQueryValidator : IValidator<Get{Entities}PagedQuery>
{
    public Task<ValidationResult> ValidateAsync(
        Get{Entities}PagedQuery query,
        CancellationToken cancellationToken)
    {
        var errors = new List<ValidationError>();

        if (query.PageNumber < 1)
        {
            errors.Add(new ValidationError("PageNumber", "Page number must be at least 1"));
        }

        if (query.PageSize < 1 || query.PageSize > 100)
        {
            errors.Add(new ValidationError("PageSize", "Page size must be between 1 and 100"));
        }

        return Task.FromResult(errors.Count > 0
            ? ValidationResult.Failure(errors)
            : ValidationResult.Success());
    }
}
```

### Shared PagedList Model

```csharp
// src/{name}.application/Abstractions/Pagination/PagedList.cs
namespace {name}.application.abstractions.pagination;

public sealed class PagedList<T>
{
    public IReadOnlyList<T> Items { get; }
    public int PageNumber { get; }
    public int PageSize { get; }
    public int TotalCount { get; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPreviousPage => PageNumber > 1;
    public bool HasNextPage => PageNumber < TotalPages;

    public PagedList(IReadOnlyList<T> items, int pageNumber, int pageSize, int totalCount)
    {
        Items = items;
        PageNumber = pageNumber;
        PageSize = pageSize;
        TotalCount = totalCount;
    }
}
```

---

## Template: Query with Multi-Mapping (Joins)

```csharp
// src/{name}.application/{Feature}/Get{Entity}WithDetails/Get{Entity}WithDetailsQuery.cs
using System.Data;
using Dapper;
using Kommand;
using {name}.application.abstractions.data;

namespace {name}.application.{feature}.Get{Entity}WithDetails;

public sealed record Get{Entity}WithDetailsQuery(
    Guid Id) : IQuery<{Entity}DetailResponse?>;

internal sealed class Get{Entity}WithDetailsQueryHandler
    : IQueryHandler<Get{Entity}WithDetailsQuery, {Entity}DetailResponse?>
{
    private readonly ISqlConnectionFactory _sqlConnectionFactory;

    public Get{Entity}WithDetailsQueryHandler(ISqlConnectionFactory sqlConnectionFactory)
    {
        _sqlConnectionFactory = sqlConnectionFactory;
    }

    public async Task<{Entity}DetailResponse?> HandleAsync(
        Get{Entity}WithDetailsQuery query,
        CancellationToken cancellationToken)
    {
        using IDbConnection connection = _sqlConnectionFactory.CreateConnection();

        const string sql = """
            SELECT
                e.id AS Id,
                e.name AS Name,
                e.description AS Description,
                c.id AS ChildId,
                c.name AS ChildName,
                c.sort_order AS ChildSortOrder
            FROM {table_name} e
            LEFT JOIN {child_table} c ON c.{entity}_id = e.id
            WHERE e.id = @Id
            ORDER BY c.sort_order ASC
            """;

        // Dictionary to track parent entity for multi-mapping
        Dictionary<Guid, {Entity}DetailResponse> entityDictionary = new();

        await connection.QueryAsync<{Entity}DetailResponse, {Child}Response, {Entity}DetailResponse>(
            sql,
            (entity, child) =>
            {
                if (!entityDictionary.TryGetValue(entity.Id, out var existingEntity))
                {
                    existingEntity = entity;
                    entityDictionary.Add(entity.Id, existingEntity);
                }

                if (child is not null)
                {
                    existingEntity.Children.Add(child);
                }

                return existingEntity;
            },
            new { query.Id },
            splitOn: "ChildId");

        return entityDictionary.Values.FirstOrDefault();
    }
}

// Response with nested children
public sealed class {Entity}DetailResponse
{
    public Guid Id { get; init; }
    public required string Name { get; init; }
    public string? Description { get; init; }
    public List<{Child}Response> Children { get; init; } = new();
}

public sealed class {Child}Response
{
    public Guid ChildId { get; init; }
    public required string ChildName { get; init; }
    public int ChildSortOrder { get; init; }
}
```

---

## Template: Query Interceptor (Query-Only)

```csharp
// src/{name}.application/Interceptors/QueryCachingInterceptor.cs
using Kommand;
using Kommand.Interceptors;
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

namespace {name}.application.interceptors;

/// <summary>
/// Caches query results - applies only to queries using IQueryInterceptor
/// </summary>
public sealed class QueryCachingInterceptor<TQuery, TResponse>
    : IQueryInterceptor<TQuery, TResponse>
    where TQuery : IQuery<TResponse>
{
    private readonly IDistributedCache _cache;

    public QueryCachingInterceptor(IDistributedCache cache)
    {
        _cache = cache;
    }

    public async Task<TResponse> HandleAsync(
        TQuery query,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        // Check if query implements ICacheable
        if (query is not ICacheableQuery cacheable)
        {
            return await next();
        }

        var cacheKey = cacheable.CacheKey;

        // Try get from cache
        var cachedValue = await _cache.GetStringAsync(cacheKey, cancellationToken);
        if (cachedValue is not null)
        {
            return JsonSerializer.Deserialize<TResponse>(cachedValue)!;
        }

        // Execute query
        var response = await next();

        // Cache response
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = cacheable.CacheDuration ?? TimeSpan.FromMinutes(5)
        };

        await _cache.SetStringAsync(
            cacheKey,
            JsonSerializer.Serialize(response),
            options,
            cancellationToken);

        return response;
    }
}

// Interface for cacheable queries
public interface ICacheableQuery
{
    string CacheKey { get; }
    TimeSpan? CacheDuration { get; }
}
```

---

## Template: API Endpoint with Kommand Query

```csharp
// src/{name}.api/Endpoints/{Entity}Endpoints.cs
using Kommand;
using Kommand.Validation;

public static class {Entity}Endpoints
{
    public static void Map{Entity}Endpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/{entities}")
            .WithTags("{Entities}")
            .RequireAuthorization();

        group.MapGet("/", GetAll{Entities});
        group.MapGet("/{id:guid}", Get{Entity}ById);
        group.MapGet("/paged", Get{Entities}Paged);
    }

    private static async Task<IResult> GetAll{Entities}(
        IMediator mediator,
        CancellationToken cancellationToken)
    {
        var result = await mediator.QueryAsync(
            new GetAll{Entities}Query(),
            cancellationToken);

        return Results.Ok(result);
    }

    private static async Task<IResult> Get{Entity}ById(
        Guid id,
        IMediator mediator,
        CancellationToken cancellationToken)
    {
        try
        {
            var result = await mediator.QueryAsync(
                new Get{Entity}ByIdQuery(id),
                cancellationToken);

            return result is not null
                ? Results.Ok(result)
                : Results.NotFound();
        }
        catch (ValidationException ex)
        {
            return Results.BadRequest(new { errors = ex.Errors });
        }
    }

    private static async Task<IResult> Get{Entities}Paged(
        [AsParameters] Get{Entities}PagedRequest request,
        IMediator mediator,
        CancellationToken cancellationToken)
    {
        try
        {
            var query = new Get{Entities}PagedQuery(
                request.PageNumber ?? 1,
                request.PageSize ?? 20,
                request.SearchTerm);

            var result = await mediator.QueryAsync(query, cancellationToken);

            return Results.Ok(result);
        }
        catch (ValidationException ex)
        {
            return Results.BadRequest(new { errors = ex.Errors });
        }
    }
}

// Request model for query string binding
public sealed record Get{Entities}PagedRequest(
    int? PageNumber,
    int? PageSize,
    string? SearchTerm);
```

---

## SQL Connection Factory

```csharp
// src/{name}.application/Abstractions/Data/ISqlConnectionFactory.cs
using System.Data;

namespace {name}.application.abstractions.data;

public interface ISqlConnectionFactory
{
    IDbConnection CreateConnection();
}

// src/{name}.infrastructure/Data/SqlConnectionFactory.cs
using System.Data;
using Npgsql;
using {name}.application.abstractions.data;

namespace {name}.infrastructure.data;

internal sealed class SqlConnectionFactory : ISqlConnectionFactory
{
    private readonly string _connectionString;

    public SqlConnectionFactory(string connectionString)
    {
        _connectionString = connectionString;
    }

    public IDbConnection CreateConnection()
    {
        var connection = new NpgsqlConnection(_connectionString);
        connection.Open();
        return connection;
    }
}
```

---

## SQL Best Practices

### Column Naming (Snake Case to PascalCase)

```sql
-- PostgreSQL with snake_case columns
SELECT
    e.id AS Id,                          -- Maps to Id property
    e.first_name AS FirstName,           -- Maps to FirstName property
    e.created_at AS CreatedAt,           -- Maps to CreatedAt property
    e.organization_id AS OrganizationId  -- Maps to OrganizationId property
FROM entity e
```

### Avoiding N+1 Queries

```sql
-- ❌ BAD: Separate queries for children
SELECT * FROM parent WHERE id = @Id;
-- Then for each parent:
SELECT * FROM child WHERE parent_id = @ParentId;

-- ✅ GOOD: Single query with JOIN
SELECT
    p.id AS Id, p.name AS Name,
    c.id AS ChildId, c.name AS ChildName
FROM parent p
LEFT JOIN child c ON c.parent_id = p.id
WHERE p.id = @Id
```

### Using CTEs for Complex Queries

```sql
WITH RankedItems AS (
    SELECT
        *,
        ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY score DESC) as rank
    FROM items
),
TopItems AS (
    SELECT * FROM RankedItems WHERE rank <= 3
)
SELECT * FROM TopItems ORDER BY category_id, rank;
```

---

## Critical Rules

1. **Queries never modify state** - Read-only operations
2. **Use Dapper for queries** - Better performance than EF Core for reads
3. **Return DTOs, not entities** - Don't expose domain models
4. **Use parameterized queries** - Prevent SQL injection
5. **Alias columns to match DTOs** - Use `AS PropertyName`
6. **Always close connections** - Use `using` statement
7. **Use multi-mapping for joins** - Avoid N+1 queries
8. **Validate query parameters** - Especially for pagination
9. **Use CTEs for complex logic** - More readable than nested queries
10. **Return null for not found** - Let API layer decide response

---

## Anti-Patterns to Avoid

```csharp
// ❌ WRONG: Using Handle instead of HandleAsync
public async Task<Response> Handle(Query query, CancellationToken ct)

// ✅ CORRECT: Kommand uses HandleAsync
public async Task<Response> HandleAsync(Query query, CancellationToken ct)

// ❌ WRONG: Using EF Core for read queries
public async Task<EntityResponse?> HandleAsync(...)
{
    var entity = await _dbContext.Entities
        .Include(e => e.Children)
        .FirstOrDefaultAsync(e => e.Id == query.Id);
    // Heavy, tracks changes unnecessarily
}

// ✅ CORRECT: Use Dapper
public async Task<EntityResponse?> HandleAsync(...)
{
    using var connection = _sqlConnectionFactory.CreateConnection();
    // Direct SQL, no tracking overhead
}

// ❌ WRONG: Returning domain entities
public sealed record GetEntityQuery(Guid Id) : IQuery<Entity>; // Exposes domain

// ✅ CORRECT: Return DTOs
public sealed record GetEntityQuery(Guid Id) : IQuery<EntityResponse?>;

// ❌ WRONG: String concatenation in SQL
var sql = $"SELECT * FROM entity WHERE name = '{query.Name}'"; // SQL injection!

// ✅ CORRECT: Parameterized queries
var sql = "SELECT * FROM entity WHERE name = @Name";
await connection.QueryAsync(sql, new { query.Name });
```

---

## Related Skills

- `02-cqrs-command-generator` - Generate write-side commands
- `04-domain-entity-generator` - Generate domain entities
- `06-ef-core-configuration` - EF Core for write operations
- `19-dapper-query-builder` - Advanced Dapper patterns
