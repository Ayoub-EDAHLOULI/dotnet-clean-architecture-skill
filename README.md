# dotnet-clean-architecture-skill

An agent skill for building, extending and reviewing **.NET 10** services with
Clean Architecture, CQRS, EF Core and multi-tenant SaaS patterns.

> Status: work in progress (v0.1). The `SKILL.md` is drafted; reference files
> are being added one at a time.

## What it does

- **Scaffold** a layered .NET 10 solution with sensible, secure defaults.
- **Add feature slices**: command or query, handler, validator, endpoint, tests.
- **Audit** an existing codebase and report layering, tenancy, persistence and
  API issues by severity.

## Opinions, briefly

- Dependencies point inward; hosts are the only composition roots.
- Tenant identity comes from the authenticated principal, never an unchecked header.
- Tenant isolation and soft delete are separate EF Core 10 named query filters.
- Application returns `Result<T>`; the API speaks `ProblemDetails`.
- No commercially licensed dependencies by default (Mediator instead of MediatR,
  explicit mapping instead of AutoMapper).

See [`skills/dotnet-clean-architecture/SKILL.md`](skills/dotnet-clean-architecture/SKILL.md)
for the full rule set.

## Layout

```
skills/
  dotnet-clean-architecture/
    SKILL.md        entry point: rules, workflows, reference index
    references/     detailed guides, loaded on demand
```

## Installation

Instructions for Claude Code and other agents will be added once the first
reference files land.

## License

[MIT](LICENSE)
