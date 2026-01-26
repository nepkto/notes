# Composition Over Inheritance in PHP/Laravel (Multi‑Tenant + Reporting Example)

## Table of Contents
- [1. Inheritance vs Composition (quick comparison)](#1-inheritance-vs-composition-quick-comparison)
- [2. The Laravel use‑case we’ll build](#2-the-laravel-use-case-well-build)
- [3. Central DB vs Tenant DB](#3-central-db-vs-tenant-db)
- [4. Composition design: the tenancy building blocks](#4-composition-design-the-tenancy-building-blocks)
- [5. TenantContext (shared state, explicit and testable)](#5-tenantcontext-shared-state-explicit-and-testable)
- [6. Resolve tenant from subdomain (composition instead of “magic base controller”)](#6-resolve-tenant-from-subdomain-composition-instead-of-magic-base-controller)
- [7. TenantConnectionManager (runtime DB switching)](#7-tenantconnectionmanager-runtime-db-switching)
- [8. Middleware that composes everything together](#8-middleware-that-composes-everything-together)
- [9. Tenant models: “use tenant connection” via trait (composition)](#9-tenant-models-use-tenant-connection-via-trait-composition)
- [10. Admin cross‑tenant reporting (why inheritance breaks down)](#10-admin-cross-tenant-reporting-why-inheritance-breaks-down)
- [11. RunInTenant helper (composition for safe switching)](#11-runintenant-helper-composition-for-safe-switching)
- [12. CrossTenantRevenueReport (compose runner + central tenant list + query)](#12-crosstenantrevenuereport-compose-runner--central-tenant-list--query)
- [13. Excel export (composition: exporter is separate from report)](#13-excel-export-composition-exporter-is-separate-from-report)
- [14. Controller usage (thin controller; compose services)](#14-controller-usage-thin-controller-compose-services)
- [15. Why composition is especially important for database‑per‑tenant](#15-why-composition-is-especially-important-for-database-per-tenant)
- [16. When inheritance is still okay in Laravel](#16-when-inheritance-is-still-okay-in-laravel)
- [17. Summary (mental model)](#17-summary-mental-model)
- [18. Suggested folder structure (optional but practical)](#18-suggested-folder-structure-optional-but-practical)
- [19. Next improvements (real-world production ideas)](#19-next-improvements-real-world-production-ideas)
- [20. Senior/Seasoned Engineer Q&A (Interview‑style)](#20-seniorseasoned-engineer-qa-interview-style)

---

**Composition over inheritance** is an object‑oriented design guideline:

> Prefer **building classes by combining smaller objects** (composition) rather than creating large, deep **class hierarchies** (inheritance).

In Laravel, composition is especially natural because Laravel encourages:
- **Dependency Injection (DI)** via the Service Container
- **Interfaces / contracts**
- **Service Providers** to bind abstractions to implementations
- Small, testable services instead of “god” base classes

This note explains the concept and then walks through a realistic example:

- **Multi‑tenant app**
- **Database‑per‑tenant**
- **Tenant resolved via subdomain**
- **Admin cross‑tenant reporting**
- **Excel export**

The goal is for you to understand not only *what* composition is, but *why it maps so well to Laravel*.

---

## 1. Inheritance vs Composition (quick comparison)

### Inheritance (“is‑a”)
Inheritance is appropriate when a child class truly **is a specialized form of** its parent and the parent/child relationship is stable.

Example idea:
- `JsonReport extends BaseReport`
- `RevenueReport extends BaseReport`
- `CrossTenantRevenueReport extends RevenueReport`

**Common problems in real projects**
- **Tight coupling**: child classes depend on parent internals.
- **Fragile base class**: a change in `BaseReport` can break many subclasses.
- **Subclass explosion**: “report + caching + logging + export format” becomes a combinatorial nightmare.
- Harder to test in isolation.

### Composition (“has‑a / uses‑a”)
Composition means your class **has** collaborators:
- A report “has a” query builder and “uses” a tenant runner
- An exporter “uses” a collection of rows

**Benefits**
- Swap implementations without changing the main class
- Smaller classes, clearer responsibilities
- Easier unit testing
- More flexibility (especially important in multi‑tenant systems)

---

## 2. The Laravel use‑case we’ll build

We want a Laravel architecture where:

### Tenant side
- Each tenant has its **own database**
- Tenant is resolved from **subdomain** (e.g., `acme.example.com`)
- App automatically switches DB connection to the current tenant DB

### Admin side
- Admin wants a **cross‑tenant revenue report**
- That means: loop across all tenants, connect to each tenant DB, query revenue, aggregate, export to **Excel**

This is a great scenario for composition because:
- Tenant resolution is one concern
- DB connection switching is another concern
- Reporting logic is another concern
- Excel exporting is another concern

Each concern can be expressed as a small composable service.

---

## 3. Central DB vs Tenant DB

A typical database‑per‑tenant setup uses:

### Central database (shared)
Holds tenant registry:
- `tenants` table: `id`, `slug`, `db_host`, `db_name`, `db_username`, `db_password`, ...

### Tenant databases (one per tenant)
Holds tenant application tables:
- `invoices`, `customers`, `users`, ...

In `config/database.php`, you often define:
- `mysql` (or `central`) → central connection
- `tenant` → a “dynamic” connection; we replace its database credentials at runtime per request/job

---

## 4. Composition design: the tenancy building blocks

We compose tenancy using four pieces:

1. **TenantContext**: stores the currently active tenant (for this request/job)
2. **SubdomainTenantResolver**: figures out which tenant is being requested
3. **TenantConnectionManager**: switches the `tenant` connection dynamically
4. **InitializeTenant middleware**: orchestrates (resolve → set context → connect)

Instead of building a large inheritance base like `BaseTenantController` or `TenantModel` and forcing everything through that hierarchy, we keep small services and wire them together.

---

## 5. TenantContext (shared state, explicit and testable)

**Why it exists:**  
We need a single place to store “current tenant” so we don’t rely on ad‑hoc globals or duplicated logic.

```php
<?php
// app/Tenancy/TenantContext.php
namespace App\Tenancy;

use App\Models\Tenant;

final class TenantContext
{
    private ?Tenant $tenant = null;

    public function set(Tenant $tenant): void
    {
        $this->tenant = $tenant;
    }

    public function get(): ?Tenant
    {
        return $this->tenant;
    }

    public function require(): Tenant
    {
        return $this->tenant ?? throw new \RuntimeException('Tenant not set in context.');
    }

    public function clear(): void
    {
        $this->tenant = null;
    }
}
```

This class has a single responsibility and is easy to mock or replace.

---

## 6. Resolve tenant from subdomain (composition instead of “magic base controller”)

**Goal:** `acme.example.com` → find tenant with slug `acme`.

Important: This resolver must query the **central DB**, because we haven’t switched to the tenant DB yet.

```php
<?php
// app/Tenancy/SubdomainTenantResolver.php
namespace App\Tenancy;

use App\Models\Tenant;
use Illuminate\Http\Request;

final class SubdomainTenantResolver
{
    public function __construct(private Request $request) {}

    public function resolve(): ?Tenant
    {
        $host = $this->request->getHost(); // acme.example.com
        $sub  = explode('.', $host)[0] ?? null;

        if (!$sub || $sub === 'www') return null;

        return Tenant::on('mysql') // central connection
            ->where('slug', $sub)
            ->first();
    }
}
```

If later you switch from subdomain to `X-Tenant` header, you can introduce `HeaderTenantResolver` and swap it—without touching reporting logic or models.

---

## 7. TenantConnectionManager (runtime DB switching)

**Goal:** update `database.connections.tenant.*` and reconnect.

```php
<?php
// app/Tenancy/TenantConnectionManager.php
namespace App\Tenancy;

use App\Models\Tenant;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\DB;

final class TenantConnectionManager
{
    public function connectTo(Tenant $tenant): void
    {
        // Note: Config::set() works even if you ran `php artisan config:cache`.
        // The set is in-memory for the current process/request; it does NOT rewrite cached config files.
        Config::set('database.connections.tenant.host', $tenant->db_host);
        Config::set('database.connections.tenant.database', $tenant->db_name);
        Config::set('database.connections.tenant.username', $tenant->db_username);
        Config::set('database.connections.tenant.password', $tenant->db_password);

        // Important: the connection object caches settings; purge forces re-creation using updated config
        DB::purge('tenant');
        DB::reconnect('tenant');
    }

    public function disconnect(): void
    {
        DB::disconnect('tenant');
    }
}
```

**Config cache impact (short)**
- `config:cache` changes *how Laravel loads the initial config*, but `Config::set()` / `Config::get()` still work normally.
- Changes are in-memory and apply only to the current request/process.
- In long-lived workers (Octane/queues), you must clear/reset to avoid tenant leakage.

---

## 8. Middleware that composes everything together

Rather than inheritance (e.g., `TenantBaseController` that every controller must extend), a middleware is the correct Laravel composition point.

```php
<?php
// app/Http/Middleware/InitializeTenant.php
namespace App\Http\Middleware;

use App\Tenancy\SubdomainTenantResolver;
use App\Tenancy\TenantConnectionManager;
use App\Tenancy\TenantContext;
use Closure;

final class InitializeTenant
{
    public function __construct(
        private SubdomainTenantResolver $resolver,
        private TenantContext $context,
        private TenantConnectionManager $connections,
    ) {}

    public function handle($request, Closure $next)
    {
        $tenant = $this->resolver->resolve();
        if (!$tenant) {
            abort(404, 'Tenant not found.');
        }

        $this->context->set($tenant);
        $this->connections->connectTo($tenant);

        try {
            return $next($request);
        } finally {
            // Important for Octane / workers: avoid tenant data leaking
            $this->context->clear();
            $this->connections->disconnect();
        }
    }
}
```

**Why composition matters here:**  
The middleware doesn’t “inherit” tenancy behavior; it *uses* it by delegating to small services.

---

## 9. Tenant models: “use tenant connection” via trait (composition)

In database‑per‑tenant, the model must use the `tenant` connection.

Instead of `class TenantModel extends Model` and forcing every model to extend it, a small trait is a composable approach.

```php
<?php
// app/Models/Concerns/UsesTenantConnection.php
namespace App\Models\Concerns;

trait UsesTenantConnection
{
    public function getConnectionName()
    {
        return 'tenant';
    }
}
```

Example tenant model:

```php
<?php
namespace App\Models\Tenant;

use Illuminate\Database\Eloquent\Model;
use App\Models\Concerns\UsesTenantConnection;

final class Invoice extends Model
{
    use UsesTenantConnection;

    protected $table = 'invoices';
}
```

---

## 10. Admin cross‑tenant reporting (why inheritance breaks down)

For admin reports across many tenant databases:
- You cannot rely on a single global DB connection
- You must **iterate tenants**
- Each iteration requires switching to that tenant DB

If you tried to solve this with inheritance (“BaseReport knows tenancy”), you’d mix too many responsibilities into a base class.

Instead we compose:
- A **runner** that safely “executes code in a tenant”
- A **report** that uses the runner to gather rows
- An **exporter** that converts rows into Excel output

---

## 11. RunInTenant helper (composition for safe switching)

This encapsulates the tricky “switch → run → cleanup” sequence in one place.

```php
<?php
// app/Tenancy/RunInTenant.php
namespace App\Tenancy;

use App\Models\Tenant;

final class RunInTenant
{
    public function __construct(
        private TenantConnectionManager $connections,
        private TenantContext $context,
    ) {}

    /** @return mixed */
    public function run(Tenant $tenant, callable $callback)
    {
        $this->context->set($tenant);
        $this->connections->connectTo($tenant);

        try {
            return $callback($tenant);
        } finally {
            $this->context->clear();
            $this->connections->disconnect();
        }
    }
}
```

Now any service (reporting, maintenance tasks, migrations, etc.) can reuse this *without inheritance*.

---

## 12. CrossTenantRevenueReport (compose runner + central tenant list + query)

**What it does**
- Fetch all tenants from central DB
- For each tenant:
  - connect to tenant DB
  - query invoices revenue
  - return normalized rows
- Flatten into one combined collection

```php
<?php
// app/Reports/Admin/CrossTenantRevenueReport.php
namespace App\Reports\Admin;

use App\Models\Tenant;
use App\Tenancy\RunInTenant;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\DB;

final class CrossTenantRevenueReport
{
    public function __construct(private RunInTenant $runner) {}

    public function rows(array $params): Collection
    {
        $tenants = Tenant::on('mysql')->get(); // central

        return $tenants->flatMap(function (Tenant $tenant) use ($params) {
            return $this->runner->run($tenant, function () use ($tenant, $params) {
                $from = $params['from'] ?? now()->subMonth()->toDateString();
                $to   = $params['to'] ?? now()->toDateString();

                $rows = DB::connection('tenant')
                    ->table('invoices')
                    ->whereDate('paid_at', '>=', $from)
                    ->whereDate('paid_at', '<=', $to)
                    ->selectRaw(
                        '? as tenant_id, ? as tenant_slug, date(paid_at) as day, sum(total_cents) as revenue_cents',
                        [$tenant->id, $tenant->slug]
                    )
                    ->groupBy('day')
                    ->orderBy('day')
                    ->get();

                return collect($rows)->map(fn($r) => [
                    'tenant_id' => $r->tenant_id,
                    'tenant_slug' => $r->tenant_slug,
                    'day' => $r->day,
                    'revenue' => $r->revenue_cents / 100,
                ]);
            });
        });
    }
}
```

### Why this is composition
`CrossTenantRevenueReport` does not “extend” a base report to get tenant switching.  
It *has a* `RunInTenant` collaborator and *uses* it.

This makes it:
- easier to test (you can stub `RunInTenant`)
- easier to change switching strategy (e.g., use read replicas, caching, sharding)
- safer in long‑running processes

---

## 13. Excel export (composition: exporter is separate from report)

We keep reporting and exporting separate:
- Report returns `Collection<array>`
- Exporter turns that into `.xlsx`

Most Laravel apps use **maatwebsite/excel** for Excel exports.

### Export class
```php
<?php
// app/Reports/Exporters/ArrayCollectionExport.php
namespace App\Reports\Exporters;

use Illuminate\Support\Collection;
use Maatwebsite\Excel\Concerns\FromCollection;
use Maatwebsite\Excel\Concerns\WithHeadings;

final class ArrayCollectionExport implements FromCollection, WithHeadings
{
    public function __construct(private Collection $rows) {}

    public function collection(): Collection
    {
        return $this->rows;
    }

    public function headings(): array
    {
        if ($this->rows->isEmpty()) return [];
        return array_keys($this->rows->first());
    }
}
```

### ExcelExporter service
```php
<?php
// app/Reports/Exporters/ExcelExporter.php
namespace App\Reports\Exporters;

use Illuminate\Support\Collection;
use Maatwebsite\Excel\Facades\Excel;
use Symfony\Component\HttpFoundation\BinaryFileResponse;

final class ExcelExporter
{
    public function download(Collection $rows, string $filenameBase): BinaryFileResponse
    {
        return Excel::download(new ArrayCollectionExport($rows), $filenameBase . '.xlsx');
    }
}
```

---

## 14. Controller usage (thin controller; compose services)

```php
<?php
namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use App\Reports\Admin\CrossTenantRevenueReport;
use App\Reports\Exporters\ExcelExporter;

final class AdminReportsController extends Controller
{
    public function revenue(CrossTenantRevenueReport $report, ExcelExporter $excel)
    {
        $rows = $report->rows(request()->all());
        return $excel->download($rows, 'cross-tenant-revenue');
    }
}
```

This is classic Laravel composition:
- Controller does not contain logic
- It orchestrates collaborators

---

## 15. Why composition is especially important for database‑per‑tenant

### (A) You must prevent tenant leakage
In Octane / queue workers, a process may handle multiple requests/jobs.  
If you don’t clear context and reconnect properly, one tenant can “leak” into another.

Our middleware and `RunInTenant` explicitly:
- set tenant
- connect
- run
- clear tenant
- disconnect

### (B) Admin cross‑tenant reporting is inherently “N connections”
You will likely query 10s/100s/1000s of tenant DBs.
Composition makes it easier to:
- run per tenant in a safe wrapper
- add batching, retry, and timeouts
- move reporting to async jobs when needed

### (C) Export format should not affect data logic
Excel/CSV/PDF are output concerns.
Keeping `ExcelExporter` separate means:
- your report stays pure (“give me rows”)
- exporters can be swapped or added without changing report logic

---

## 16. When inheritance is still okay in Laravel

Composition over inheritance is a guideline, not a ban.

Laravel uses inheritance well where it is stable and intentional:
- `FormRequest extends Request`
- `JsonResource extends Resource`
- `Exception` hierarchies

Use inheritance when the relationship is truly “is‑a” and stable.
Use composition when you want flexibility and reuse across concerns.

---

## 17. Summary (mental model)

### Prefer composition in these cases
- Multi‑tenant behavior (resolution, switching, context)
- Reporting pipelines (filters, query building, aggregation)
- Exporting (Excel/CSV/PDF) as plug‑in output formats
- Cross‑cutting concerns (logging, caching, retry)

### In our example, composition looks like:
- Middleware **uses** resolver + connection manager + context
- Report **uses** tenant runner + DB queries
- Excel exporter **uses** rows and converts them to `.xlsx`

No deep base classes, minimal coupling, easy to extend.

---

## 18. Suggested folder structure (optional but practical)

- `app/Tenancy/`
  - `TenantContext.php`
  - `SubdomainTenantResolver.php`
  - `TenantConnectionManager.php`
  - `RunInTenant.php`
- `app/Http/Middleware/`
  - `InitializeTenant.php`
- `app/Models/Concerns/`
  - `UsesTenantConnection.php`
- `app/Reports/Admin/`
  - `CrossTenantRevenueReport.php`
- `app/Reports/Exporters/`
  - `ExcelExporter.php`
  - `ArrayCollectionExport.php`

---

## 19. Next improvements (real-world production ideas)

- **Queue the report** for large tenant counts, store the Excel file (S3), and notify admin when ready.
- Add **retry/backoff** in `RunInTenant` to handle transient DB errors.
- Add **chunking** over tenants to control memory.
- Add **report parameters validation** via `FormRequest`.
- Encrypt tenant DB credentials in the central database (encrypted casts / KMS).

---

## 20. Senior/Seasoned Engineer Q&A (Interview‑style)

These questions and answers are tailored to the exact scenario above (Laravel, database‑per‑tenant, subdomain resolution, cross‑tenant reporting, Excel export) and focus on tradeoffs, failure modes, and architecture decisions.

### Q1) What does “composition over inheritance” mean in practical Laravel code?
**A:** It means preferring to build features by wiring small services together (via DI / container bindings / middleware / decorators) rather than creating “base classes” and extending them everywhere.  
In this example:
- Tenancy is built from `SubdomainTenantResolver` + `TenantConnectionManager` + `TenantContext` + middleware.
- Reporting is built from `RunInTenant` + `CrossTenantRevenueReport` + `ExcelExporter`.  
None of these require deep class hierarchies.

---

### Q2) When would you still choose inheritance in Laravel?
**A:** When there is a stable “is‑a” relationship and framework conventions make it natural, such as:
- `FormRequest` subclasses for validation/authorization
- `JsonResource` subclasses for presentation
- `Exception` hierarchies  
But I avoid inheritance when it’s primarily for code reuse across unrelated responsibilities.

---

### Q3) Why is inheritance risky for multi‑tenant database switching?
**A:** Because DB switching is a cross‑cutting concern that touches controllers, jobs, reports, commands, etc. Putting it into a base class (e.g., `BaseTenantController`, `TenantModel`, `BaseReport`) leads to:
- implicit coupling (“this class works only if you extended the right base class”)
- hidden side effects (connection changes in constructors/boot methods)
- brittleness when different flows require different switching behavior (tenant vs admin vs jobs)

Composition keeps switching explicit and reusable in a dedicated service (`RunInTenant`, middleware).

---

### Q4) How does `php artisan config:cache` affect `Config::set()`/`Config::get()` in dynamic tenant connection switching?
**A:** It doesn’t prevent them from working. With config cache, Laravel loads config from a compiled array (`bootstrap/cache/config.php`).  
`Config::set()` still mutates the in‑memory config repository for the current process/request.

Key nuance: the mutation is **not persisted** to disk, and in long‑running processes you must reset/cleanup to avoid leakage.

---

### Q5) Why do you need `DB::purge('tenant')` and not just `Config::set()` + `DB::reconnect()`?
**A:** The connection manager can cache an existing connection instance. `Config::set()` changes values, but the already-created PDO connection may still point to the previous database.  
`DB::purge('tenant')` removes the connection from the manager so the next connect uses the updated config. `DB::reconnect()` then ensures a fresh connection is created.

---

### Q6) What’s the biggest operational risk of runtime `Config::set()` in tenancy?
**A:** Tenant leakage in long‑running processes (Octane, queue workers).  
If you don’t clear context and disconnect/purge after handling a tenant, the next request/job may reuse the same process with stale config, causing cross-tenant data exposure.

Mitigation: always cleanup in `finally { ... }`, and consider Octane-specific flushing hooks if used.

---

### Q7) How would you run an admin cross‑tenant report without bringing down the system?
**A:** The naive approach loops all tenants synchronously and connects to each DB. For a small number of tenants, that may be fine.  
At scale:
- Run it as a queued job
- Batch tenants (chunking) and stream results into a file
- Store the Excel file in S3 and return a “report ready” link
- Add timeouts/retries per tenant so one bad tenant DB doesn’t fail the whole report
Composition helps because the per-tenant execution is isolated in `RunInTenant`.

---

### Q8) What about transactions when switching tenants?
**A:** A transaction must not span across tenant switches. The correct boundary is:
- connect to tenant
- run query/transaction
- commit/rollback
- disconnect
- move to next tenant  
This is naturally enforced when the “unit of work” is inside the callback passed to `RunInTenant`.

---

### Q9) Would you use Eloquent models or Query Builder for admin cross‑tenant reports?
**A:** For cross‑tenant aggregation, I often prefer `DB::connection('tenant')->table(...)` because:
- it avoids accidental model boot events/scopes that might add overhead
- it keeps report queries explicit and performant  
Eloquent is fine for tenant-local reports, but admin cross-tenant often benefits from tighter query control.

---

### Q10) How do you handle schema differences between tenant databases over time?
**A:** This is a real risk with database‑per‑tenant:
- ensure consistent migrations across all tenant DBs
- maintain a “tenant migration status” tracker in central DB
- in cross‑tenant reports, guard against missing columns/tables and report per-tenant failures cleanly  
Composition helps by isolating per-tenant execution and allowing “skip + record error” strategies.

---

### Q11) How would you test this design?
**A:** I would test at multiple levels:
- Unit test `SubdomainTenantResolver` by mocking the request host.
- Unit test `TenantConnectionManager` by asserting config keys are set and `DB::purge`/`reconnect` are called (or integration test with actual DB).
- Unit test `CrossTenantRevenueReport` by stubbing `RunInTenant` to return sample rows per tenant.
- Feature test middleware that a tenant route switches to the correct DB.  
Composition makes stubbing/mocking far easier than inheritance-heavy approaches.

---

### Q12) Is the `TenantContext` necessary? Why not just resolve tenant each time?
**A:** Resolving tenant repeatedly can:
- duplicate work and query central DB multiple times
- create inconsistencies if resolution logic changes mid-request
- make it harder to pass tenant into jobs/commands  
`TenantContext` centralizes “current tenant” and makes it explicit, which also improves observability and logging.

---

### Q13) What is the clean boundary between “central” and “tenant” concerns?
**A:** Anything about:
- tenant discovery, credentials, lifecycle → **central**
- tenant business data (invoices, customers) → **tenant**  
In code: central models use central connection, tenant models use `tenant` connection. Reports decide explicitly whether they operate per-tenant or cross-tenant.

---

### Q14) If you needed caching, where would you put it without changing business logic?
**A:** Add caching as a decorator around report generation or row retrieval:
- e.g., `CachedReport` that wraps a `Report` and caches `rows($params)` by key
- or cache per-tenant partial results  
This is a classic composition win: add behavior by wrapping, not subclassing everything.

---

### Q15) What metrics/logging would you add for reliability?
**A:** At minimum:
- log tenant slug/id and report name per execution
- timer per tenant query duration
- count failures/timeouts per tenant
- export generation time and file size
- warnings if tenant context is missing in a tenant-only route  
These can be added via middleware/decorators without modifying core business logic (composition again).