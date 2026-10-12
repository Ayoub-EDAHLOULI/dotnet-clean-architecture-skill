# Multi-tenancy

## Purpose

Load this file for any task that resolves the current tenant, reads or writes
tenant-owned data, adds a tenant-owned entity, or audits tenant isolation. It
expands SKILL.md rules 3–6. Scope is v1: one shared database, a `TenantId`
column on every tenant-owned table, isolation enforced by EF Core 10 named
query filters and a `SaveChanges` interceptor. Database-per-tenant is out of
scope.

## Rules

1. **Shared database, `TenantId` on every tenant-owned table.** One schema and
   one migration path keep operations simple; the cost is that isolation is
   enforced in code, so the rules below are not optional.
2. **Tenant-owned entities implement `ITenantOwned`; `TenantId` has a private
   setter.** Only the interceptor sets it, through
   `entry.Property(nameof(ITenantOwned.TenantId)).CurrentValue`. A public
   setter lets a handler or mapper move a row into another tenant.
3. **`ITenantContext` is an Application port; hosts set it, nothing else
   does.** Persistence reads it, Api and Worker set it. Resolving the tenant
   inside Persistence or Infrastructure drags `HttpContext` into layers that
   the Worker also uses, where it is always null.
4. **Api: resolve the tenant from a claim, after `UseAuthentication()`.**
   Before authentication `HttpContext.User` is empty, so any tenant read there
   comes from unauthenticated input.
5. **A tenant header is accepted only if it matches the claim, or the caller
   passes the impersonation policy.** Every impersonated request is logged
   with the real user and the target tenant. An unchecked header lets any
   authenticated user read any tenant.
6. **Worker: the tenant comes from the message envelope, and every published
   message carries it.** A consumer has no principal; a message without a
   tenant cannot be processed safely, so it is rejected, not defaulted.
7. **Tenant and soft delete are separate named query filters, `Tenant` and
   `SoftDelete`, applied by convention to every `ITenantOwned` /
   `ISoftDeletable` entity** (EF Core 10). One combined filter cannot be
   partially disabled, so "show deleted items" would also disable isolation.
8. **No tenant means no rows (fail closed).** The filter compares against a
   nullable current tenant; with none resolved it matches nothing. A fallback
   tenant turns a resolution bug into a data leak.
9. **Cross-tenant reads use `IgnoreQueryFilters(["Tenant"])` only inside an
   audited `SystemScope`.** System jobs that write tenant data run one DI
   scope per tenant with that tenant set, so rule 10 still holds.
10. **The `SaveChanges` interceptor stamps `TenantId` on added entities and
    throws on a missing or mismatched tenant.** Filters only protect reads; a
    write with the wrong `TenantId` succeeds silently without this guard.
11. **Bulk operations and raw SQL have weaker guarantees.**
    `ExecuteUpdate`/`ExecuteDelete` respect the `Tenant` query filter but
    bypass the `SaveChanges` interceptor: no tenant guard, no audit stamps,
    and `ExecuteDelete` hard-deletes even soft-deletable rows. `ExecuteSql`
    bypasses both filter and interceptor and must filter by tenant explicitly.
12. **Uniqueness is a filtered unique index on `(TenantId, Key)`.** Two
    tenants may own the same SKU, and a soft-deleted row must not block
    re-creating it. A validator check races; the index does not.
13. **Filter both sides of a required navigation.** EF uses an `INNER JOIN`
    for required navigations, so a filtered principal silently removes its
    dependents from `Include` queries. The convention in this file filters
    every `ITenantOwned` type, which keeps both sides consistent.
14. **No DbContext pooling by default.** A pooled context is reused across
    requests, so a tenant captured at construction leaks into the next
    request. If pooling is required, use `AddPooledDbContextFactory` plus a
    scoped factory that sets the tenant on every lease.
15. **Everything outside the database is tenant-keyed:** cache keys,
    blob/file paths, and the logging scope (`TenantId` in every log entry).
    Query filters do not reach Redis or blob storage; an unprefixed cache key
    serves one tenant's data to another.

## Code

### Ports and markers (Domain, Application)

```csharp
// Domain
public interface ITenantOwned { Guid TenantId { get; } }
public interface ISoftDeletable { bool IsDeleted { get; } }

public sealed class Product : ITenantOwned, ISoftDeletable
{
    public Guid Id { get; private set; }         // EF value generation; never set in domain code
    public Guid TenantId { get; private set; }   // set only by TenantInterceptor
    public bool IsDeleted { get; private set; }
    public string Sku { get; private set; } = default!;
    public int Stock { get; private set; }
    // factory and intent methods omitted
}

// Application
public interface ITenantContext
{
    Guid? TenantId { get; }
}

public sealed class TenantContext : ITenantContext
{
    public Guid? TenantId { get; private set; }

    public void Set(Guid tenantId)
    {
        if (tenantId == Guid.Empty) throw new ArgumentException("Empty tenant.", nameof(tenantId));
        if (TenantId is not null && TenantId != tenantId)
            throw new InvalidOperationException("Tenant already set for this scope.");
        TenantId = tenantId;
    }
}
```

Register once per host: `services.AddScoped<TenantContext>()` and
`services.AddScoped<ITenantContext>(sp => sp.GetRequiredService<TenantContext>())`.

### Api: tenant resolution middleware

```csharp
public sealed class TenantResolutionMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(
        HttpContext http, TenantContext tenant,
        IAuthorizationService authz, ILogger<TenantResolutionMiddleware> logger)
    {
        if (Guid.TryParse(http.User.FindFirstValue(TenantClaims.TenantId), out var claimTenant))
        {
            var effective = claimTenant;
            if (http.Request.Headers.TryGetValue(TenantHeaders.TenantId, out var header))
            {
                if (!Guid.TryParse(header, out var requested))
                {
                    await Results.Problem(statusCode: 400, title: "Invalid tenant header.").ExecuteAsync(http);
                    return;
                }
                if (requested != claimTenant)
                {
                    var allowed = await authz.AuthorizeAsync(http.User, Policies.ImpersonateTenant);
                    if (!allowed.Succeeded)
                    {
                        await Results.Problem(statusCode: 403, title: "Tenant mismatch.").ExecuteAsync(http);
                        return;
                    }
                    logger.LogWarning("User {UserId} impersonating tenant {TargetTenantId}",
                        http.User.FindFirstValue(ClaimTypes.NameIdentifier), requested);
                    effective = requested;
                }
            }
            tenant.Set(effective);
        }
        // No claim: tenant stays unset; filters return nothing, writes throw.

        using (logger.BeginScope(new Dictionary<string, object?> { ["TenantId"] = tenant.TenantId }))
            await next(http);
    }
}

// Program.cs order matters
app.UseAuthentication();
app.UseMiddleware<TenantResolutionMiddleware>();
app.UseAuthorization();
```

### Worker: tenant from the message envelope

```csharp
public sealed record MessageEnvelope<T>(Guid MessageId, Guid TenantId, T Payload);
public interface IMessageHandler<in T> { Task HandleAsync(T message, CancellationToken ct); }

public sealed class TenantScopedConsumer<T>(
    IServiceScopeFactory scopes, ILogger<TenantScopedConsumer<T>> logger)
{
    public async Task ConsumeAsync(MessageEnvelope<T> envelope, CancellationToken ct)
    {
        if (envelope.TenantId == Guid.Empty)
            throw new InvalidOperationException($"Message {envelope.MessageId} has no tenant.");

        await using var scope = scopes.CreateAsyncScope();
        scope.ServiceProvider.GetRequiredService<TenantContext>().Set(envelope.TenantId);

        using (logger.BeginScope(new Dictionary<string, object?> { ["TenantId"] = envelope.TenantId }))
            await scope.ServiceProvider.GetRequiredService<IMessageHandler<T>>()
                .HandleAsync(envelope.Payload, ct);
    }
}
```

Wire `ConsumeAsync` into whichever transport the Worker uses; publishers build
the envelope from `ITenantContext.TenantId` and throw if it is null.

### DbContext: named filters by convention

```csharp
public static class QueryFilters
{
    public const string Tenant = "Tenant";
    public const string SoftDelete = "SoftDelete";
}

public sealed class AppDbContext : DbContext, IAppDbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options, ITenantContext tenant)
        : base(options) => CurrentTenantId = tenant.TenantId;

    // Read by the filters (parameterized per instance) and by TenantInterceptor.
    public Guid? CurrentTenantId { get; internal set; }

    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);

        foreach (var type in modelBuilder.Model.GetEntityTypes()
                     .Where(t => t.BaseType is null).Select(t => t.ClrType))
        {
            if (typeof(ITenantOwned).IsAssignableFrom(type))
                Invoke(nameof(ApplyTenantFilter), type, modelBuilder);
            if (typeof(ISoftDeletable).IsAssignableFrom(type))
                Invoke(nameof(ApplySoftDeleteFilter), type, modelBuilder);
        }
    }

    private void ApplyTenantFilter<T>(ModelBuilder b) where T : class, ITenantOwned =>
        b.Entity<T>().HasQueryFilter(QueryFilters.Tenant,
            e => EF.Property<Guid>(e, nameof(ITenantOwned.TenantId)) == CurrentTenantId);

    private static void ApplySoftDeleteFilter<T>(ModelBuilder b) where T : class, ISoftDeletable =>
        b.Entity<T>().HasQueryFilter(QueryFilters.SoftDelete,
            e => !EF.Property<bool>(e, nameof(ISoftDeletable.IsDeleted)));

    private void Invoke(string method, Type entity, ModelBuilder b) =>
        typeof(AppDbContext)
            .GetMethod(method, BindingFlags.NonPublic | BindingFlags.Instance | BindingFlags.Static)!
            .MakeGenericMethod(entity)
            .Invoke(this, [b]);
}
```

With `CurrentTenantId` null, the tenant predicate matches no row (rule 8).
Filters are defined on root types only; EF Core does not allow them on derived
types.

### TenantInterceptor

```csharp
public sealed class TenantInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData, InterceptionResult<int> result)
    { Guard(eventData.Context); return result; }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result, CancellationToken ct = default)
    { Guard(eventData.Context); return ValueTask.FromResult(result); }

    private static void Guard(DbContext? context)
    {
        if (context is not AppDbContext db) return;

        foreach (var entry in db.ChangeTracker.Entries<ITenantOwned>())
        {
            if (entry.State is EntityState.Unchanged or EntityState.Detached) continue;

            var current = db.CurrentTenantId ?? throw new InvalidOperationException(
                $"Cannot save {entry.Metadata.ClrType.Name} without a resolved tenant.");
            var tenantId = entry.Property(nameof(ITenantOwned.TenantId));

            if (entry.State == EntityState.Added && (Guid)tenantId.CurrentValue! == Guid.Empty)
            {
                tenantId.CurrentValue = current;
                continue;
            }

            if ((Guid)tenantId.CurrentValue! != current
                || (entry.State != EntityState.Added && (Guid)tenantId.OriginalValue! != current))
                throw new InvalidOperationException(
                    $"Cross-tenant write on {entry.Metadata.ClrType.Name} blocked.");
        }
    }
}

// AddPersistence
services.AddSingleton<TenantInterceptor>();
services.AddDbContext<AppDbContext>((sp, o) => o
    .UseSqlServer(connectionString)
    .AddInterceptors(sp.GetRequiredService<TenantInterceptor>()));
```

The interceptor is stateless and reads the tenant from the context instance,
so it works unchanged with the pooled factory from rule 14.

### Tenant-scoped filtered unique index

```csharp
public sealed class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.HasIndex(p => new { p.TenantId, p.Sku })
            .IsUnique()
            .HasFilter("[IsDeleted] = 0");
    }
}
```

PostgreSQL: `.HasFilter("\"IsDeleted\" = false")`.

### Bulk operations and raw SQL

```csharp
// Tenant filter applies; interceptor does not. Stamp audit fields yourself.
await db.Products
    .Where(p => p.Sku == sku)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.IsDeleted, true), ct);

// Neither filter nor interceptor applies: the tenant predicate is mandatory.
await db.Database.ExecuteSqlAsync(
    $"UPDATE [Products] SET [IsDeleted] = 1 WHERE [TenantId] = {tenantId} AND [Sku] = {sku}", ct);
```

### Audited system scope

```csharp
public sealed class SystemScope(IDisposable? logScope) : IDisposable
{
    public static SystemScope Begin(string reason, ILogger logger)
    {
        logger.LogWarning("Entering cross-tenant system scope: {Reason}", reason);
        return new(logger.BeginScope(new Dictionary<string, object?> { ["SystemScope"] = reason }));
    }

    public void Dispose() => logScope?.Dispose();
}

public static class TenantQueryableExtensions
{
    // Requiring the scope makes every cross-tenant read greppable and logged.
    public static IQueryable<T> AcrossTenants<T>(this IQueryable<T> query, SystemScope scope)
        where T : class, ITenantOwned
    {
        ArgumentNullException.ThrowIfNull(scope);
        return query.IgnoreQueryFilters([QueryFilters.Tenant]);
    }
}

// Usage: soft-delete filter still applies
using var scope = SystemScope.Begin("Nightly low-stock report", logger);
var lowStock = await db.Products.AcrossTenants(scope).Where(p => p.Stock < 5).ToListAsync(ct);
```

## Anti-patterns

- **Reading the tenant from a header, route value or query string alone.**
  Any authenticated user can then address any tenant.
- **Resolving the tenant before `UseAuthentication()`.** The principal is
  still empty, so the only available source is untrusted input.
- **Defaulting to `Guid.Empty` or a "default tenant".** Rows written with it
  are visible to every request that also fails to resolve a tenant.
- **A dev or fallback tenant switched on by environment name.** A
  misconfigured `ASPNETCORE_ENVIRONMENT` in production silently merges
  tenants.
- **`IgnoreQueryFilters()` with no filter names.** It disables tenant
  isolation along with soft delete.
- **Setting `TenantId` in handlers, mappers or constructors.** It spreads the
  guarantee across every call site; one miss is a cross-tenant row.
- **`IHttpContextAccessor` in Persistence or Infrastructure.** It is null in
  the Worker and couples data access to HTTP.
- **Compiled models (`dotnet ef dbcontext optimize`) with query filters.** The
  EF Core docs list global query filters as unsupported in compiled models;
  verify before enabling one.
- **Unprefixed cache keys or shared blob paths.** Query filters do not apply
  outside the database.

## Checklist

- [ ] Every tenant-owned entity implements `ITenantOwned`, and its `TenantId`
      setter is private.
- [ ] No code outside `TenantInterceptor` assigns `TenantId`.
- [ ] `TenantResolutionMiddleware` is registered after `UseAuthentication()`
      and before `UseAuthorization()`.
- [ ] A tenant header different from the claim requires the impersonation
      policy and produces a log entry.
- [ ] Every published message envelope has a non-empty `TenantId`; consumers
      reject empty ones.
- [ ] Every `ITenantOwned` root type has a `Tenant` named filter; every
      `ISoftDeletable` root type has a `SoftDelete` named filter.
- [ ] No `IgnoreQueryFilters()` call without filter names; every
      `IgnoreQueryFilters(["Tenant"])` sits inside a `SystemScope`.
- [ ] `TenantInterceptor` is registered on every `DbContext` that holds
      tenant-owned entities.
- [ ] Every `ExecuteSql`/`FromSql` touching a tenant-owned table includes a
      `TenantId` predicate.
- [ ] No `ExecuteDelete` on an `ISoftDeletable` entity.
- [ ] Every unique index on a tenant-owned table starts with `TenantId` and is
      filtered on `IsDeleted`.
- [ ] No `AddDbContextPool`; if pooling exists, it goes through a scoped
      factory that sets `CurrentTenantId`.
- [ ] No fallback tenant, and no tenant logic keyed on environment name.
- [ ] Cache keys and blob paths include the tenant; log scope includes
      `TenantId`.
- [ ] Every `ITenantOwned` entity is covered by the tenant-isolation test
      (template in `testing.md`).
