---
name: dotnet-clean-architecture
description: Build, extend, or review .NET 10 services that use Clean Architecture, CQRS (Mediator), EF Core and multi-tenant SaaS patterns. Use when scaffolding a new ASP.NET Core solution, adding a command, query or endpoint, adding tenant isolation, or auditing an existing .NET codebase for layering, tenancy, persistence and API bugs.
---

# .NET Clean Architecture (CQRS, EF Core, multi-tenant)

Opinionated guidance for building production .NET 10 services with Clean
Architecture and CQRS. Every rule below exists because its absence caused a real
bug in a real codebase.

## When to use

- Scaffolding a new service or solution.
- Adding a feature slice: command or query, handler, validator, endpoint, tests.
- Adding or reviewing multi-tenancy, soft delete, auditing or concurrency.
- Auditing an existing ASP.NET Core codebase and producing a findings report.

## When NOT to use

- Small CRUD apps, prototypes or scripts. A single project with minimal APIs is
  the better choice there; say so instead of applying this structure.
- Non-.NET stacks.

## Non-negotiable rules

Apply these by default. Deviate only when the user asks, and state the trade-off.

**Architecture**
1. **Dependencies point inward.** Domain references nothing. Application
   references Domain (and Contracts if it publishes integration events).
   Persistence and Infrastructure reference Application. Only hosts (Api,
   Worker) reference Persistence and Infrastructure. Hosts never reference each
   other.
2. **Each layer registers itself.** Every project exposes one `AddX()`
   extension (`AddApplication`, `AddPersistence`, `AddInfrastructure`). Both
   hosts call the same methods. Assemblies are located with
   `typeof(Marker).Assembly`, never `Assembly.Load("Name")`.

**Multi-tenancy**
3. **Tenant identity comes from a trusted source.** In HTTP: a claim on the
   authenticated principal, resolved after `UseAuthentication()`. A tenant
   header is accepted only if it matches the claim or the caller holds an
   explicit impersonation permission. In workers: the message envelope.
4. **Tenant and soft delete are separate named query filters** (EF Core 10).
   Never call `IgnoreQueryFilters()` without filter names on tenant-owned data.
5. **A write without a tenant throws.** Never default to `Guid.Empty`.
6. **Uniqueness is tenant-scoped and soft-delete aware**: a unique index on
   `(TenantId, Key)` filtered on `IsDeleted = 0`. Validator checks are a
   courtesy message, not the guarantee.

**Application and API**
7. **Application returns `Result<T>`, never HTTP concepts.** No status codes in
   handlers. Endpoints map `Error.Type` to `ProblemDetails` (RFC 9457). One
   error shape for the whole API.
8. **Secure by default.** Configure a fallback authorization policy that
   requires an authenticated user. Never put `[AllowAnonymous]` on a controller
   or group that has protected actions; it overrides every `[Authorize]` beneath it.
9. **No unbounded reads.** Lists take a clamped `PageRequest` and end their
   sort with a unique tiebreaker (`ThenBy(x => x.Id)`). Queries project with
   `Select` into DTOs.
10. **State changes go through aggregate methods** (`Order.Create`,
    `order.Confirm()`). No mapper writes into entities. `PUT` replaces fully;
    partial updates are an explicit `PATCH`.

**Persistence**
11. **Cross-cutting fields are stamped by interceptors**, not handlers:
    tenant, audit, soft delete, domain-event-to-outbox. Use `TimeProvider`,
    not `DateTime.UtcNow`. Use `Guid.CreateVersion7()` for keys.
12. **Optimistic concurrency reaches the client.** Expose the row version as an
    `ETag`; require `If-Match` on `PUT`/`DELETE`.
13. **Startup never migrates in production.** Use a migration bundle or a
    deployment job. Development-only auto-migrate must fail loudly.

**Testing**
14. **Handlers that touch the database are tested against a real database**
    (Testcontainers). Never mock `DbSet`. Every tenant-owned entity is covered
    by the tenant-isolation test template.

## Solution layout

```
Orders.slnx
global.json                  pin SDK 10.0.x
Directory.Build.props        net10.0, Nullable, TreatWarningsAsErrors, analyzers
Directory.Packages.props     central package management
src/
  Orders.Domain              aggregates, value objects, domain events, errors
  Orders.Application         use-case slices, ports, behaviors, Result
  Orders.Contracts           integration events, public DTOs (no deps)
  Orders.Persistence         DbContext, configurations, interceptors, migrations
  Orders.Infrastructure      messaging, tenant provider, external clients
  Orders.Api                 endpoints, composition root
  Orders.Worker              consumers, outbox relay, composition root
tests/
  Orders.UnitTests           domain logic, validators, pure services
  Orders.IntegrationTests    handlers + API against real SQL (Testcontainers)
  Orders.ArchitectureTests   enforces rule 1 and naming conventions
```

Full details: `references/solution-structure.md`.

## Workflows

### Scaffold a new solution
1. Confirm the service name, database provider and whether it is multi-tenant.
2. Create the layout above. Wire `AddX()` methods and both hosts.
3. Add the cross-cutting baseline: `Result`/`Error`, validation behavior,
   `ProblemDetails`, interceptors, named filters, fallback auth policy, health
   checks (liveness and readiness).
4. Add the architecture tests and the tenant-isolation test before any feature.

### Add a feature slice
1. Read `references/cqrs-slices.md`.
2. Create one folder per use case: `Application/<Feature>/<UseCase>/` holding
   the command or query, its handler and its validator.
3. Change state through an aggregate method; add one if missing.
4. Add any uniqueness as a filtered unique index in the entity configuration.
5. Add the endpoint; map `Result` to HTTP with the shared helper.
6. Add an integration test for the happy path, one failure path, and tenant
   isolation if the entity is tenant-owned.

### Audit an existing codebase
1. Read `references/audit-checklist.md`.
2. Inspect, in order: project references, composition roots, tenant resolution,
   query filters and `SaveChanges`, entity indexes, one full slice, error
   handling, endpoints/authorization, tests.
3. Report findings grouped by severity (Critical, High, Medium), each with the
   evidence (file and line), the impact, and the fix. Lead with security and
   data-integrity issues. Note what is done well.

## Decision points

| Decision | Default | Alternative (when) |
|---|---|---|
| Mediator library | `Mediator` (source-generated, MIT) | MediatR, if already licensed: `references/mediatr-alternative.md` |
| Data access in handlers | `IAppDbContext` with `DbSet`s | Repositories, for rich aggregates with complex loading |
| Domain model | Aggregates with intent methods | Anemic entities, for pure reference data |
| Errors | `Result<T>` + `ProblemDetails` | Exceptions only for truly exceptional failures |
| Mapping | Explicit (`Select`, factory methods) | Mapster, for large read-only DTOs |
| Paging | Offset with clamped `PageRequest` | Keyset, for large tables or infinite scroll |
| Tenant storage | Shared database, `TenantId` column | Database per tenant, for strict isolation requirements |

## Reference index

Load only the file the current task needs.

| File | Load when |
|---|---|
| `references/solution-structure.md` | Scaffolding, or fixing project references |
| `references/domain-modeling.md` | Writing aggregates, value objects, domain events |
| `references/cqrs-slices.md` | Adding any command or query |
| `references/pipeline-behaviors.md` | Validation, logging, transaction behaviors |
| `references/error-handling.md` | `Result`, `Error`, `ProblemDetails` mapping |
| `references/persistence-efcore.md` | DbContext, configurations, interceptors, indexes, migrations |
| `references/multi-tenancy.md` | Anything touching tenant resolution or tenant-owned data |
| `references/api-endpoints.md` | Endpoints, versioning, OpenAPI, authorization |
| `references/testing.md` | Unit, integration, architecture and tenant-isolation tests |
| `references/audit-checklist.md` | Reviewing an existing codebase |
| `references/anti-patterns.md` | Explaining why a pattern is rejected |
| `references/mediatr-alternative.md` | The project uses or requires MediatR |

## Naming conventions

| Item | Convention | Example |
|---|---|---|
| Command | `<Verb><Noun>Command` | `CreateOrderCommand` |
| Query | `Get<Noun(s)>Query` | `GetOrdersQuery` |
| Handler | `<Request>Handler` | `CreateOrderHandler` |
| Validator | `<Request>Validator` | `CreateOrderValidator` |
| Response DTO | `<Noun>Response` / `<Noun>Summary` | `OrderResponse` |
| Error codes | `<Aggregate>.<Reason>` | `Order.NotFound` |
| Entities | Domain name, no `Entity` suffix | `Order`, not `OrderEntity` |
