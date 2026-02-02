# Agent Skills Reference

Quick reference showing which skills each agent has access to.

## Quick Matrix

| Skill # | Skill Name | Domain | Application | Infrastructure | API | Web | MCP |
|---------|------------|--------|-------------|----------------|-----|-----|-----|
| 01 | dotnet-clean-architecture | ✅ | ✅ | ✅ | ✅ | - | - |
| 02 | cqrs-command-generator | - | ✅ | - | - | - | - |
| 03 | cqrs-query-generator | - | ✅ | - | - | - | - |
| 04 | domain-entity-generator | ✅ | - | - | - | - | - |
| 05 | repository-pattern | ✅* | - | ✅ | - | - | - |
| 06 | ef-core-configuration | - | - | ✅ | - | - | - |
| 07 | **minimal-api-endpoints** | - | - | - | ✅ | - | - |
| 07 | legacy-api-controllers | - | - | - | ✅† | - | - |
| 08 | result-pattern | ✅ | ✅ | ✅ | ✅ | - | - |
| 09 | domain-events-generator | ✅ | ✅ | - | - | - | - |
| 10 | pipeline-behaviors | - | ✅ | - | - | - | - |
| 11 | fluent-validation | - | ✅ | - | - | - | - |
| 12 | jwt-authentication | - | - | ✅ | ✅ | - | - |
| 13 | permission-authorization | - | - | ✅ | ✅ | - | - |
| 14 | outbox-pattern | - | - | ✅ | - | - | - |
| 15 | quartz-background-jobs | - | - | ✅ | - | - | - |
| 16.1 | email-service-sendgrid | - | - ✅ | - | - | - |
| 16.2 | email-service-aws-ses | - | - | ✅ | - | - | - |
| 17 | health-checks | - | - | ✅ | - | - | - |
| 18 | audit-trail | - | - | ✅ | - | - | - |
| 19 | dapper-query-builder | - | - | ✅ | - | - | - |
| 20 | specification-pattern | ✅ | - | ✅ | - | - | - |
| 21 | unit-testing | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 22 | integration-testing | - | ✅ | ✅ | ✅ | - | - |
| 23 | postgresql-best-practices | - | - | ✅ | - | - | - |
| 24 | logging-configuration | - | ✅ | ✅ | - | - | - |
| 25 | options-pattern | - | ✅ | ✅ | - | - | - |
| 26 | rate-limiting | - | - | - | ✅ | - | - |
| 27 | server-sent-events | - | - | - | ✅ | - | - |

*Domain defines repository **interfaces**, Infrastructure implements them.
†Legacy controllers available as fallback only. **Prefer Minimal APIs (Microsoft recommendation)**.
‡CQRS skills now use **Kommand** (not MediatR) for commands/queries.

## By Agent

### Domain Agent (6 skills)
Core business logic and domain modeling.

- `04-domain-entity-generator` - Entities, value objects, aggregates
- `05-repository-pattern` - Repository interfaces (contracts only)
- `08-result-pattern` - Error handling
- `09-domain-events-generator` - Event definitions
- `20-specification-pattern` - Query specifications
- `21-unit-testing` - Domain tests

**Creates**: Entities, value objects, domain events, repository interfaces, error definitions, specifications

---

### Application Agent (9 skills)
Use case orchestration and workflows using **Kommand** CQRS library.

- `02-cqrs-command-generator` - Commands (Create, Update, Delete) with Kommand
- `03-cqrs-query-generator` - Queries (Get, List, Search) with Kommand
- `09-domain-events-generator` - Event handlers (INotification)
- `10-pipeline-behaviors` - Interceptors for validation, logging, performance
- `11-fluent-validation` - Input validation (or Kommand's IValidator<T>)
- `21-unit-testing` - Application tests
- `22-integration-testing` - Workflow tests
- `24-logging-configuration` - ILogger<T> usage in handlers
- `25-options-pattern` - Configuration in handlers

**Creates**: Commands, queries, handlers, validators, interceptors, event handlers, DTOs

---

### Infrastructure Agent (18 skills)
Technical implementations and external integrations.

- `05-repository-pattern` - Repository implementations
- `06-ef-core-configuration` - Entity mappings and migrations
- `08-result-pattern` - Infrastructure error handling
- `12-jwt-authentication` - JWT generation/validation
- `13-permission-authorization` - Permissions and roles
- `14-outbox-pattern` - Reliable event publishing
- `15-quartz-background-jobs` - Job scheduling
- `16.1-email-service-sendgrid` - SendGrid integration
- `16.2-email-service-aws-ses` - AWS SES integration
- `17-health-checks` - Service health monitoring
- `18-audit-trail` - Change tracking
- `19-dapper-query-builder` - Performance queries
- `20-specification-pattern` - Specification implementation
- `21-unit-testing` - Infrastructure tests
- `22-integration-testing` - Database tests
- `23-postgresql-best-practices` - PostgreSQL optimization + concurrency patterns
- `24-logging-configuration` - Serilog setup, ILoggerFactory
- `25-options-pattern` - IOptions, IOptionsSnapshot, IOptionsMonitor

**Creates**: Repository implementations, EF configurations, database migrations, background jobs, email services, auth setup, health checks, optimized PostgreSQL schemas, logging infrastructure, options configuration

---

### API Agent (9 skills)
HTTP concerns and RESTful endpoints using **Minimal APIs**.

- **`07-minimal-api-endpoints`** - Minimal APIs (Primary, Microsoft recommended)
- `07-legacy-api-controllers` - Controllers (Fallback for specific scenarios)
- `08-result-pattern` - Error responses
- `12-jwt-authentication` - Auth middleware
- `13-permission-authorization` - Authorization attributes
- `21-unit-testing` - API tests
- `22-integration-testing` - Endpoint tests
- `26-rate-limiting` - API rate limiting middleware
- `27-server-sent-events` - Real-time server-to-client streaming (SSE)

**Creates**: Minimal API endpoint handlers, request/response DTOs, middleware, authorization attributes, API versioning setup, rate limiting policies, SSE streaming endpoints

**Note**: Prefers Minimal APIs over controllers per Microsoft recommendation. Controllers available only for specific scenarios (JsonPatch, OData, etc.).

---

### Web Agent
Frontend and user interface.

- Modern frameworks (React, Vue, Angular, Blazor)
- CSS frameworks (Tailwind, Bootstrap)
- State management (Redux, Zustand)
- Form libraries
- `21-unit-testing` - Component tests

**Creates**: UI components, forms, state management, API integration, authentication flows

---

### MCP Agent
External integrations via Model Context Protocol.

- MCP protocol specification
- Server implementations
- Tool definitions
- `21-unit-testing` - Tool tests

**Creates**: MCP servers, tools, resources, integration code

---

## Shared Skills

**All Technical Agents** (Domain, Application, Infrastructure, API)
- `01-dotnet-clean-architecture` - Foundation
- `08-result-pattern` - Consistent error handling
- `21-unit-testing` - Testing

**Write-Side Agents** (Domain, Application, Infrastructure, API)
- `22-integration-testing` - Integration tests

**Auth Agents** (Infrastructure, API)
- `12-jwt-authentication`
- `13-permission-authorization`

**Data Agents** (Domain, Infrastructure)
- `05-repository-pattern` - Interface/Implementation split
- `20-specification-pattern` - Query specifications

**Event Agents** (Domain, Application)
- `09-domain-events-generator` - Raise/Handle split

---

## Common Skill Flows

### CRUD Feature (with Minimal APIs)
1. Domain: `04-domain-entity-generator` → Entity
2. Application: `02-cqrs-command-generator`, `03-cqrs-query-generator` → Commands/Queries
3. Infrastructure: `05-repository-pattern`, `06-ef-core-configuration` → Persistence
4. API: **`07-minimal-api-endpoints`** → Minimal API endpoints

### Outbox Pattern
1. Domain: `09-domain-events-generator` → Events
2. Infrastructure: `14-outbox-pattern`, `15-quartz-background-jobs` → Reliable publishing

### Authentication
1. Infrastructure: `12-jwt-authentication` → JWT service
2. API: `12-jwt-authentication`, `13-permission-authorization` → Middleware + authorization

### Complex Query
1. Domain: `20-specification-pattern` → Specification
2. Infrastructure: `19-dapper-query-builder` → Optimized query