---
name: cqrs-command-generator
description: "Generates CQRS Commands with Handlers, Validators, and Request DTOs following Clean Architecture patterns. Commands represent actions that modify state and return Result types for proper error handling."
version: 2.0.0
language: C#
framework: .NET 8+
dependencies: Kommand
---

# CQRS Command Generator

## Overview

This skill generates Commands following the CQRS (Command Query Responsibility Segregation) pattern using **Kommand** - a lightweight, production-ready CQRS mediator for .NET 8+ with built-in OpenTelemetry support. Commands represent intentions to change system state. Each command has:

- **Command Record** - Immutable data structure with request parameters
- **Validator** - Input validation using Kommand's IValidator<T>
- **Handler** - Business logic implementation
- **Request DTO** (optional) - API layer request model

## Quick Reference

| Command Type | Returns | Use Case |
|--------------|---------|----------|
| `ICommand` | `Unit` | Operations without return value (Update, Delete) |
| `ICommand<T>` | `T` | Operations returning data (Create returns Id) |

---

## Kommand vs MediatR

| Feature | Kommand | MediatR |
|---------|---------|---------|
| **Package Size** | <50KB | ~500KB |
| **Dependencies** | Zero (DI abstractions only) | Several |
| **OpenTelemetry** | Built-in | Requires setup |
| **Validation** | Built-in IValidator<T> | Requires FluentValidation |
| **Handler Lifetime** | Scoped (default) | Transient |
| **Handler Method** | `HandleAsync` | `Handle` |
| **Interceptors** | 3 types (All, Command, Query) | Single type |

---

## Command Structure

```
/Application/{Feature}/
├── Create{Entity}/
│   ├── Create{Entity}Command.cs        # Record + Handler
│   ├── Create{Entity}CommandValidator.cs
│   └── Create{Entity}Request.cs        # Optional API DTO
├── Update{Entity}/
│   ├── Update{Entity}Command.cs
│   ├── Update{Entity}CommandValidator.cs
│   └── Update{Entity}Request.cs
└── Delete{Entity}/
    ├── Delete{Entity}Command.cs
    └── Delete{Entity}CommandValidator.cs
```

---

## Template: Command with Return Value (Create)

Use for operations that return data (typically entity ID after creation).

```csharp
// src/{name}.application/{Feature}/Create{Entity}/Create{Entity}Command.cs
using Kommand;
using {name}.application.abstractions.clock;
using {name}.domain.abstractions;
using {name}.domain.{entities};

namespace {name}.application.{feature}.Create{Entity};

// ═══════════════════════════════════════════════════════════════
// COMMAND RECORD
// ═══════════════════════════════════════════════════════════════
public sealed record Create{Entity}Command(
    string Name,
    string? Description,
    Guid? ParentId) : ICommand<Guid>;

// ═══════════════════════════════════════════════════════════════
// HANDLER
// ═══════════════════════════════════════════════════════════════
internal sealed class Create{Entity}CommandHandler
    : ICommandHandler<Create{Entity}Command, Guid>
{
    private readonly I{Entity}Repository _{entity}Repository;
    private readonly IDateTimeProvider _dateTimeProvider;
    private readonly IUnitOfWork _unitOfWork;

    public Create{Entity}CommandHandler(
        I{Entity}Repository {entity}Repository,
        IDateTimeProvider dateTimeProvider,
        IUnitOfWork unitOfWork)
    {
        _{entity}Repository = {entity}Repository;
        _dateTimeProvider = dateTimeProvider;
        _unitOfWork = unitOfWork;
    }

    public async Task<Guid> HandleAsync(
        Create{Entity}Command command,
        CancellationToken cancellationToken)
    {
        // 1. Validate business rules
        var existingEntity = await _{entity}Repository
            .GetByNameAsync(command.Name, cancellationToken);

        if (existingEntity is not null)
        {
            throw new DomainException({Entity}Errors.AlreadyExists);
        }

        // 2. Create domain entity using factory method
        var {entity} = {Entity}.Create(
            command.Name,
            command.Description,
            _dateTimeProvider.UtcNow);

        // 3. Persist to repository
        _{entity}Repository.Add({entity});

        // 4. Save changes (via Unit of Work)
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        // 5. Return created entity ID
        return {entity}.Id;
    }
}
```

---

## Template: Command Validator (Kommand Native)

```csharp
// src/{name}.application/{Feature}/Create{Entity}/Create{Entity}CommandValidator.cs
using Kommand.Validation;

namespace {name}.application.{feature}.Create{Entity};

// ═══════════════════════════════════════════════════════════════
// VALIDATOR - Uses Kommand's built-in validation
// ═══════════════════════════════════════════════════════════════
internal sealed class Create{Entity}CommandValidator : IValidator<Create{Entity}Command>
{
    private readonly I{Entity}Repository _{entity}Repository;

    public Create{Entity}CommandValidator(I{Entity}Repository {entity}Repository)
    {
        _{entity}Repository = {entity}Repository;
    }

    public async Task<ValidationResult> ValidateAsync(
        Create{Entity}Command command,
        CancellationToken cancellationToken)
    {
        var errors = new List<ValidationError>();

        // ═══════════════════════════════════════════════════════════════
        // SYNCHRONOUS VALIDATION
        // ═══════════════════════════════════════════════════════════════
        if (string.IsNullOrWhiteSpace(command.Name))
        {
            errors.Add(new ValidationError("Name", "{Entity} name is required", "REQUIRED"));
        }
        else if (command.Name.Length > 100)
        {
            errors.Add(new ValidationError("Name", "{Entity} name must not exceed 100 characters", "MAX_LENGTH"));
        }

        if (command.Description?.Length > 500)
        {
            errors.Add(new ValidationError("Description", "Description must not exceed 500 characters", "MAX_LENGTH"));
        }

        // ═══════════════════════════════════════════════════════════════
        // ASYNC VALIDATION - Database checks
        // ═══════════════════════════════════════════════════════════════
        if (!string.IsNullOrWhiteSpace(command.Name))
        {
            var existing = await _{entity}Repository.GetByNameAsync(command.Name, cancellationToken);
            if (existing is not null)
            {
                errors.Add(new ValidationError("Name", "Name already exists", "DUPLICATE"));
            }
        }

        return errors.Count > 0
            ? ValidationResult.Failure(errors)
            : ValidationResult.Success();
    }
}
```

---

## Template: Command without Return Value (Update)

Use for operations that don't return data.

```csharp
// src/{name}.application/{Feature}/Update{Entity}/Update{Entity}Command.cs
using Kommand;
using {name}.application.abstractions.clock;
using {name}.domain.abstractions;
using {name}.domain.{entities};

namespace {name}.application.{feature}.Update{Entity};

// ═══════════════════════════════════════════════════════════════
// COMMAND RECORD - Returns Unit for void operations
// ═══════════════════════════════════════════════════════════════
public sealed record Update{Entity}Command(
    Guid Id,
    string Name,
    string? Description) : ICommand;  // ICommand = ICommand<Unit>

// ═══════════════════════════════════════════════════════════════
// HANDLER
// ═══════════════════════════════════════════════════════════════
internal sealed class Update{Entity}CommandHandler
    : ICommandHandler<Update{Entity}Command, Unit>
{
    private readonly I{Entity}Repository _{entity}Repository;
    private readonly IDateTimeProvider _dateTimeProvider;
    private readonly IUnitOfWork _unitOfWork;

    public Update{Entity}CommandHandler(
        I{Entity}Repository {entity}Repository,
        IDateTimeProvider dateTimeProvider,
        IUnitOfWork unitOfWork)
    {
        _{entity}Repository = {entity}Repository;
        _dateTimeProvider = dateTimeProvider;
        _unitOfWork = unitOfWork;
    }

    public async Task<Unit> HandleAsync(
        Update{Entity}Command command,
        CancellationToken cancellationToken)
    {
        // 1. Retrieve existing entity
        var {entity} = await _{entity}Repository
            .GetByIdAsync(command.Id, cancellationToken);

        if ({entity} is null)
        {
            throw new EntityNotFoundException(typeof({Entity}), command.Id);
        }

        // 2. Call domain method to update
        {entity}.Update(
            command.Name,
            command.Description,
            _dateTimeProvider.UtcNow);

        // 3. Save changes
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        return Unit.Value;
    }
}
```

---

## Template: Delete Command

```csharp
// src/{name}.application/{Feature}/Delete{Entity}/Delete{Entity}Command.cs
using Kommand;
using {name}.domain.abstractions;
using {name}.domain.{entities};

namespace {name}.application.{feature}.Delete{Entity};

public sealed record Delete{Entity}Command(Guid Id) : ICommand;

internal sealed class Delete{Entity}CommandHandler
    : ICommandHandler<Delete{Entity}Command, Unit>
{
    private readonly I{Entity}Repository _{entity}Repository;
    private readonly IUnitOfWork _unitOfWork;

    public Delete{Entity}CommandHandler(
        I{Entity}Repository {entity}Repository,
        IUnitOfWork unitOfWork)
    {
        _{entity}Repository = {entity}Repository;
        _unitOfWork = unitOfWork;
    }

    public async Task<Unit> HandleAsync(
        Delete{Entity}Command command,
        CancellationToken cancellationToken)
    {
        var {entity} = await _{entity}Repository
            .GetByIdAsync(command.Id, cancellationToken);

        if ({entity} is null)
        {
            throw new EntityNotFoundException(typeof({Entity}), command.Id);
        }

        // Check business rules before deletion
        if ({entity}.HasActiveRelationships())
        {
            throw new DomainException({Entity}Errors.CannotDeleteWithActiveRelationships);
        }

        _{entity}Repository.Remove({entity});

        await _unitOfWork.SaveChangesAsync(cancellationToken);

        return Unit.Value;
    }
}
```

---

## Template: Delete Command Validator

```csharp
// src/{name}.application/{Feature}/Delete{Entity}/Delete{Entity}CommandValidator.cs
using Kommand.Validation;

namespace {name}.application.{feature}.Delete{Entity};

internal sealed class Delete{Entity}CommandValidator : IValidator<Delete{Entity}Command>
{
    public Task<ValidationResult> ValidateAsync(
        Delete{Entity}Command command,
        CancellationToken cancellationToken)
    {
        var errors = new List<ValidationError>();

        if (command.Id == Guid.Empty)
        {
            errors.Add(new ValidationError("Id", "{Entity} ID is required", "REQUIRED"));
        }

        return Task.FromResult(errors.Count > 0
            ? ValidationResult.Failure(errors)
            : ValidationResult.Success());
    }
}
```

---

## Template: Kommand Registration

```csharp
// src/{name}.application/DependencyInjection.cs
using Microsoft.Extensions.DependencyInjection;
using Kommand;

namespace {name}.application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddKommand(config =>
        {
            // Register handlers from assembly
            config.RegisterHandlersFromAssembly(typeof(DependencyInjection).Assembly);

            // Enable validation (auto-discovers IValidator<T> implementations)
            config.WithValidation();

            // Add custom interceptors (order matters: first = outermost)
            config.AddInterceptor<LoggingInterceptor>();
            config.AddInterceptor<PerformanceInterceptor>();
        });

        return services;
    }
}
```

---

## Template: Command Interceptor (Logging)

```csharp
// src/{name}.application/Interceptors/LoggingInterceptor.cs
using Kommand;
using Kommand.Interceptors;
using Microsoft.Extensions.Logging;

namespace {name}.application.interceptors;

/// <summary>
/// Logs all commands (not queries) - use ICommandInterceptor
/// </summary>
public sealed class LoggingInterceptor<TCommand, TResponse>
    : ICommandInterceptor<TCommand, TResponse>
    where TCommand : ICommand<TResponse>
{
    private readonly ILogger<LoggingInterceptor<TCommand, TResponse>> _logger;

    public LoggingInterceptor(ILogger<LoggingInterceptor<TCommand, TResponse>> logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> HandleAsync(
        TCommand command,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var commandName = typeof(TCommand).Name;
        var startTime = DateTime.UtcNow;

        _logger.LogInformation(
            "Executing command {CommandName}",
            commandName);

        try
        {
            var response = await next();

            var duration = DateTime.UtcNow - startTime;
            _logger.LogInformation(
                "Command {CommandName} completed in {Duration}ms",
                commandName,
                duration.TotalMilliseconds);

            return response;
        }
        catch (Exception ex)
        {
            var duration = DateTime.UtcNow - startTime;
            _logger.LogError(
                ex,
                "Command {CommandName} failed after {Duration}ms",
                commandName,
                duration.TotalMilliseconds);
            throw;
        }
    }
}
```

---

## Template: API Endpoint with Kommand

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

        group.MapPost("/", Create{Entity});
        group.MapPut("/{id:guid}", Update{Entity});
        group.MapDelete("/{id:guid}", Delete{Entity});
    }

    private static async Task<IResult> Create{Entity}(
        Create{Entity}Request request,
        IMediator mediator,
        CancellationToken cancellationToken)
    {
        try
        {
            var command = new Create{Entity}Command(
                request.Name,
                request.Description,
                request.ParentId);

            var id = await mediator.SendAsync(command, cancellationToken);

            return Results.Created($"/api/{entities}/{id}", new { id });
        }
        catch (ValidationException ex)
        {
            return Results.BadRequest(new
            {
                type = "validation_error",
                errors = ex.Errors.Select(e => new
                {
                    property = e.PropertyName,
                    message = e.ErrorMessage,
                    code = e.ErrorCode
                })
            });
        }
    }

    private static async Task<IResult> Update{Entity}(
        Guid id,
        Update{Entity}Request request,
        IMediator mediator,
        CancellationToken cancellationToken)
    {
        try
        {
            var command = new Update{Entity}Command(id, request.Name, request.Description);
            await mediator.SendAsync(command, cancellationToken);
            return Results.NoContent();
        }
        catch (ValidationException ex)
        {
            return Results.BadRequest(new { errors = ex.Errors });
        }
        catch (EntityNotFoundException)
        {
            return Results.NotFound();
        }
    }

    private static async Task<IResult> Delete{Entity}(
        Guid id,
        IMediator mediator,
        CancellationToken cancellationToken)
    {
        try
        {
            await mediator.SendAsync(new Delete{Entity}Command(id), cancellationToken);
            return Results.NoContent();
        }
        catch (EntityNotFoundException)
        {
            return Results.NotFound();
        }
    }
}
```

---

## Validation Patterns

### Pattern 1: Simple Synchronous Validation

```csharp
public Task<ValidationResult> ValidateAsync(
    CreateCommand command,
    CancellationToken cancellationToken)
{
    var errors = new List<ValidationError>();

    if (string.IsNullOrWhiteSpace(command.Email))
        errors.Add(new ValidationError("Email", "Email is required", "REQUIRED"));
    else if (!command.Email.Contains('@'))
        errors.Add(new ValidationError("Email", "Invalid email format", "INVALID_FORMAT"));

    if (command.Age < 18)
        errors.Add(new ValidationError("Age", "Must be at least 18", "MIN_VALUE"));

    return Task.FromResult(errors.Count > 0
        ? ValidationResult.Failure(errors)
        : ValidationResult.Success());
}
```

### Pattern 2: Async Validation with Database Check

```csharp
public async Task<ValidationResult> ValidateAsync(
    CreateCommand command,
    CancellationToken cancellationToken)
{
    var errors = new List<ValidationError>();

    // Sync validation first (fast)
    if (string.IsNullOrWhiteSpace(command.Email))
    {
        errors.Add(new ValidationError("Email", "Email is required"));
    }

    // Async validation only if sync passed
    if (errors.Count == 0)
    {
        var exists = await _repository.EmailExistsAsync(command.Email, cancellationToken);
        if (exists)
        {
            errors.Add(new ValidationError("Email", "Email already in use", "DUPLICATE"));
        }
    }

    return errors.Count > 0
        ? ValidationResult.Failure(errors)
        : ValidationResult.Success();
}
```

### Pattern 3: Nested Object Validation

```csharp
public async Task<ValidationResult> ValidateAsync(
    CreateOrderCommand command,
    CancellationToken cancellationToken)
{
    var errors = new List<ValidationError>();

    // Validate order
    if (command.Items.Count == 0)
        errors.Add(new ValidationError("Items", "At least one item required"));

    // Validate each item
    for (int i = 0; i < command.Items.Count; i++)
    {
        var item = command.Items[i];
        if (item.Quantity <= 0)
            errors.Add(new ValidationError($"Items[{i}].Quantity", "Quantity must be positive"));
        if (item.ProductId == Guid.Empty)
            errors.Add(new ValidationError($"Items[{i}].ProductId", "Product ID required"));
    }

    return errors.Count > 0
        ? ValidationResult.Failure(errors)
        : ValidationResult.Success();
}
```

---

## Handler Patterns

### Pattern 1: Single Entity Operation

```csharp
public async Task<Guid> HandleAsync(CreateCommand command, CancellationToken ct)
{
    var entity = Entity.Create(command.Name);
    _repository.Add(entity);
    await _unitOfWork.SaveChangesAsync(ct);
    return entity.Id;
}
```

### Pattern 2: With Domain Events (Notifications)

```csharp
public async Task<Guid> HandleAsync(CreateUserCommand command, CancellationToken ct)
{
    var user = User.Create(command.Email, command.Name);
    _repository.Add(user);
    await _unitOfWork.SaveChangesAsync(ct);

    // Publish domain event (fire-and-forget)
    await _mediator.PublishAsync(
        new UserCreatedNotification(user.Id, user.Email),
        ct);

    return user.Id;
}
```

### Pattern 3: Batch Operations

```csharp
public async Task<Unit> HandleAsync(CreateBatchCommand command, CancellationToken ct)
{
    var entities = command.Items
        .Select(item => Entity.Create(item.Name))
        .ToList();

    _repository.AddRange(entities);
    await _unitOfWork.SaveChangesAsync(ct);

    return Unit.Value;
}
```

---

## Critical Rules

1. **Commands are records** - Immutable, value equality
2. **One handler per command** - No shared handlers
3. **Handlers return `HandleAsync`** - Not `Handle` (Kommand convention)
4. **Use `Unit.Value` for void operations** - Kommand's void type
5. **Validators are internal** - Not exposed outside Application layer
6. **Inject IUnitOfWork** - Don't call SaveChanges in repository
7. **Always use CancellationToken** - Pass through all async calls
8. **Domain logic in Domain** - Handler orchestrates, doesn't contain business rules
9. **Return IDs from Create** - Use `ICommand<Guid>` for creation
10. **Validators collect all errors** - Don't fail on first error

---

## Anti-Patterns to Avoid

```csharp
// ❌ WRONG: Using Handle instead of HandleAsync
public async Task<Guid> Handle(CreateCommand command, CancellationToken ct)

// ✅ CORRECT: Kommand uses HandleAsync
public async Task<Guid> HandleAsync(CreateCommand command, CancellationToken ct)

// ❌ WRONG: Returning null for void operations
public async Task<object?> HandleAsync(UpdateCommand command, CancellationToken ct)
{
    // ...
    return null;
}

// ✅ CORRECT: Return Unit for void operations
public async Task<Unit> HandleAsync(UpdateCommand command, CancellationToken ct)
{
    // ...
    return Unit.Value;
}

// ❌ WRONG: Throwing in validator (fail-fast)
if (string.IsNullOrEmpty(command.Name))
    throw new ValidationException("Name required");

// ✅ CORRECT: Collect all errors
if (string.IsNullOrEmpty(command.Name))
    errors.Add(new ValidationError("Name", "Name required"));
// Continue checking other fields...

// ❌ WRONG: Business logic in handler
if (request.Amount > 1000 && user.Level < 5)
    throw new Exception("Insufficient level");

// ✅ CORRECT: Business logic in domain
entity.ProcessOrder(request.Amount, user);  // Throws if invalid
```

---

## OpenTelemetry Integration

Kommand has built-in OpenTelemetry support:

```csharp
// Program.cs
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService("MyApp"))
    .WithTracing(tracing => tracing
        .AddSource("Kommand")  // Built-in activity source
        .AddConsoleExporter())
    .WithMetrics(metrics => metrics
        .AddMeter("Kommand")  // Built-in metrics
        .AddConsoleExporter());
```

**Metrics Collected:**
- `kommand.requests` - Total requests processed
- `kommand.requests.failed` - Failed requests
- `kommand.request.duration` - Request duration histogram

**Traces:**
- Activity name: `Command.{CommandName}` or `Query.{QueryName}`
- Tags: request type, response type, success/failure

---

## Related Skills

- `03-cqrs-query-generator` - Generate read-side queries
- `04-domain-entity-generator` - Generate domain entities with factory methods
- `08-result-pattern` - Complete Result pattern implementation
- `10-pipeline-behaviors` - Interceptors for cross-cutting concerns
