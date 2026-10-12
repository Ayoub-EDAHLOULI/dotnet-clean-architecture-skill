# CLAUDE.md — working agreement for this repo

## What this repo is

An open-source agent skill pack: `skills/dotnet-clean-architecture/SKILL.md`
plus reference files in `skills/dotnet-clean-architecture/references/`.
It teaches agents to build, extend and audit .NET 10 services using Clean
Architecture, CQRS, EF Core and multi-tenant SaaS patterns.

`SKILL.md` is the source of truth for rules, layout, workflows and the
reference index. Read it before any task.

## How we work

- One step at a time. Before writing or changing a file, state in a few lines
  what you will do and which files you will touch, then wait for my "go".
- One reference file (or one focused change) per step, sized to be one commit.
- **Never run `git commit`, `git push`, or any history-changing git command.**
  I make every commit myself. When a step is done, suggest a commit message
  in Conventional Commits style (`docs(references): add multi-tenancy guide`).
- Prefer honest, critical feedback over agreement. If one of my requests
  conflicts with `SKILL.md` or with current .NET practice, say so and explain.

## Content rules

- Target **.NET 10 / EF Core 10 / C# 14**. Flag anything version-specific.
- Default database provider is **SQL Server**. Where a rule differs on
  PostgreSQL, add a short "PostgreSQL:" note.
- Default endpoint style is **minimal APIs** with endpoint groups and
  `TypedResults`; controllers are a documented alternative.
- Examples use a **neutral domain only**: Orders, Products, Customers.
  Never HR, payroll, or any real company's business domain.
- The patterns come from a private production service. Never ask for its
  code, never name it or the company, never reproduce its business logic.
- No commercially licensed dependencies by default: Mediator
  (martinothamar, source-generated) instead of MediatR, explicit mapping
  instead of AutoMapper, AwesomeAssertions or Shouldly instead of
  FluentAssertions 8+.
- Verify package names, versions and APIs before stating them. If you can't
  verify, say so instead of guessing.
- Every rule in a reference file explains _why_ in one or two sentences,
  ideally with the failure it prevents.

## Reference file format

Each file in `references/` follows this shape:

1. Purpose: one paragraph on when to load the file.
2. Rules: numbered, each with a short rationale.
3. Code: minimal, compilable-looking C# in the Orders domain.
4. Anti-patterns: what not to do and why.
5. Checklist: items an agent can verify mechanically.

Keep each file focused and under ~400 lines. Load-on-demand only works if
the files stay small.

## Repo layout

```
skills/dotnet-clean-architecture/
  SKILL.md
  references/        one .md per topic, listed in SKILL.md's reference index
README.md  LICENSE  CLAUDE.md  .gitignore
```

## Planned order of reference files

1. multi-tenancy.md
2. persistence-efcore.md
3. error-handling.md
4. cqrs-slices.md
5. api-endpoints.md
6. testing.md
7. solution-structure.md
8. pipeline-behaviors.md
9. domain-modeling.md
10. audit-checklist.md
11. anti-patterns.md
12. mediatr-alternative.md
