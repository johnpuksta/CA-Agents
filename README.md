# .NET Clean Architecture Agent System

## What This Is

A coordinated multi-agent system for .NET Clean Architecture development. The **Orchestrator Skill** analyzes your request and coordinates specialized **Subagents** to deliver production-ready code following Clean Architecture principles.

## How to Use It

### Invoke the Orchestrator Skill

```
/orchestrator Create a Product entity with CRUD operations
```

The orchestrator analyzes the request and coordinates the appropriate subagents automatically.

### Architecture

```
┌─────────────────────────────────┐
│   Orchestrator (Skill)          │  ← Call this via /orchestrator
│   Runs in main conversation     │
└────────────┬────────────────────┘
             │ coordinates
    ┌────────┴────────┬────────┬────────┬────────┐
    ▼                 ▼        ▼        ▼        ▼
domain-agent   application-  infra-   api-   web-agent
(Subagent)      agent        agent   agent   (Subagent)
               (Subagent)  (Subagent)(Subagent)
```

**Key Difference**: The orchestrator is a **skill** (runs in main context, can invoke subagents), while the layer agents are **subagents** (run in isolated contexts).

## What It Understands

The orchestrator dynamically decides which agents to activate based on your request:

| You Ask For | It Activates | Why |
|-------------|--------------|-----|
| "Create Order entity" | Domain Agent | Pure domain modeling |
| "Add CreateOrder command" | Application Agent | CQRS command with handler |
| "Setup email notifications" | Infrastructure Agent | External service integration |
| "Create Products API" | Domain → Application → Infrastructure → API | Full CRUD feature |
| "Build user dashboard" | Application → API → Web | Read-side + UI |
| "Implement JWT auth" | Infrastructure → API | Cross-layer security |
| "Add outbox pattern" | Infrastructure Agent | Messaging infrastructure |

Simple tasks get one agent. Complex features get coordinated multi-agent execution.

## The Subagents

The orchestrator coordinates these specialized subagents:

- **domain-agent** - Entities, value objects, domain events, business rules
- **application-agent** - CQRS commands/queries, handlers, validators, pipeline behaviors
- **infrastructure-agent** - Repositories, EF Core, Dapper, auth, email, jobs, outbox
- **api-agent** - Minimal API endpoints, versioning, authorization, middleware
- **web-agent** - UI components, forms, state management, API integration
- **mcp-agent** - MCP servers, tools, external integrations

Each subagent has access to specific skills preloaded in their context (see `SKILLS-MAPPING.md` for details).

You can invoke subagents directly if you know which layer you need:

```
Use the domain-agent to create an Order entity
Use the application-agent to add a CreateOrderCommand
Use the infrastructure-agent to implement OrderRepository
```

However, using `/orchestrator` is recommended as it analyzes and coordinates automatically.

## What It Enforces

Every agent automatically follows Clean Architecture principles:

✅ **Dependencies point inward** - Domain → Application → Infrastructure → API → Web
✅ **Domain is pure** - Zero infrastructure dependencies
✅ **DDD patterns** - Entities, aggregates, value objects, domain events
✅ **CQRS** - Commands modify state, queries read data
✅ **Result pattern** - Business errors return Result, not exceptions
✅ **Repository per aggregate** - One repo per aggregate root
✅ **Proper encapsulation** - Private setters, factory methods, protected invariants

## Example Requests

### Simple
```
/orchestrator Create a Category entity with name and description
```

### Medium
```
/orchestrator Add order approval workflow with these rules:
- Orders over $10k require manager approval
- Approved orders send email notification
- Rejected orders include reason
```

### Complex
```
/orchestrator Build a complete survey system with:
- Survey aggregate with questions and responses
- CRUD operations for surveys
- Submit response workflow
- Analytics query for results
- Admin API endpoints
- Survey UI for respondents
```

## Pro Tips

### Be Specific
❌ "Add validation"
✅ "Add validation: email must be unique, password min 8 chars with special char"

### Include Business Rules
❌ "Create Order entity"
✅ "Create Order entity that can only be approved if total > 0 and user has approval permission"

### Mention Relationships
❌ "Create Product"
✅ "Create Product entity with Category reference and Inventory child entity"

### State Technical Requirements
❌ "Add authentication"
✅ "Add JWT authentication with refresh tokens and role-based authorization"

## What Gets Generated

For a typical CRUD feature, you get:

**Domain Layer**
- Entity with private setters, factory methods, validation
- Value objects (if needed)
- Domain events (Created, Updated, Deleted)
- Repository interface
- Error definitions
- Specifications (for complex queries)

**Application Layer**
- Create/Update/Delete commands with handlers
- Get/List/Search queries with handlers
- FluentValidation validators
- Domain event handlers
- Response DTOs

**Infrastructure Layer**
- Repository implementation
- EF Core entity configuration
- Database migration (if needed)
- Dapper queries (for performance-critical reads)

**API Layer**
- Minimal API endpoints (Microsoft's recommended approach)
- Request/Response DTOs
- Authorization attributes
- TypedResults with proper HTTP status codes

**Plus**
- Unit tests for each layer
- Integration tests for workflows
- XML documentation
- Proper async/await with CancellationToken

## Advanced Usage

### If You Know Which Layer
You can invoke subagents directly (but orchestrator is usually better):

```
Use the domain-agent to create Order aggregate with OrderItem children
Use the application-agent to add ProcessRefund command
Use the infrastructure-agent to setup Quartz job to process pending orders
Use the api-agent to create Orders endpoints with custom search
```

### Reference Material
- `SKILLS-MAPPING.md` - Which skills each subagent has access to
- `~/.claude/skills/00-orchestrator/` - Orchestrator skill documentation
- `~/.claude/skills/` - Individual skill documentation with detailed patterns
- `~/.claude/agents/` - Subagent configuration files

## Technical Architecture

### How It Works

1. **Orchestrator Skill** (`~/.claude/skills/00-orchestrator/`)
   - Runs in main conversation context
   - Analyzes requests and creates execution plans
   - Coordinates subagent invocation sequentially
   - Passes context between subagents

2. **Specialized Subagents** (`~/.claude/agents/`)
   - Run in isolated contexts
   - Have specific tool access and preloaded skills
   - Return results to main conversation
   - Cannot spawn other subagents (limitation by design)

3. **Skills** (`~/.claude/skills/`)
   - Loaded into subagent contexts at startup
   - Provide domain knowledge and patterns
   - Referenced in subagent YAML frontmatter

### Clean Architecture Dependency Rule

```
Domain ← Application ← Infrastructure ← API ← Web
```

The orchestrator enforces this dependency rule across all generated code.

## That's It

You now have an intelligent assistant that understands .NET Clean Architecture patterns, coordinates specialist agents, and generates production-ready code.

**Just call `/orchestrator` and describe what you need.**

---

**Questions?** The orchestrator can explain its decisions and suggest alternatives. Just ask.
