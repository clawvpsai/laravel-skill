# Laravel Version-Specific Requirements

## Active Versions

- **Laravel 11** — **End of life** (security support ended March 12, 2026). Upgrade to 12 or 13.
- **Laravel 12** — Active development (v12.62.0 as of June 2026; bug fixes until Aug 13, 2026, security until Feb 24, 2027)
- **Laravel 13** — Current latest (v13.17.0 as of June 24, 2026; bug fixes until Q3 2027, security until Mar 17, 2028)

## Version Selector Prompt

When working with Laravel, start by determining the version. Ask the user or check `composer.json`:
```bash
grep '"laravel/framework"' composer.json | grep -oP '\d+\.\d+'
```

Then load the relevant sections below.

---

## Laravel 13 (Latest — June 2026, v13.17.0)

### New in Laravel 13

- **PHP 8.3 minimum required** (8.3, 8.4, 8.5 supported — 8.2 dropped vs Laravel 12)
- **Security fixes until March 17, 2028** (24-month security window per Laravel's policy)
- **Bug fixes until Q3 2027** (~18 months)
- **Laravel AI / Boost MCP** — first-party MCP server for AI assistants
  - `/upgrade-laravel-v13` slash command for automated upgrades
  - Interactive HTML responses in Claude/VS Code Copilot
- **Improved Eloquent** — better attribute casting, faster queries
- **Faster routing** — optimized route resolution
- **Native TypeScript support** — improved scaffolding
- **Laravel Reverb** — first-party WebSocket server (production-ready)
- **New PHP Attributes for Controllers:** `#[Middleware]` and `#[Authorize]`
- **New PHP Attributes for Testing:** `#[Group]`, `#[TestProperty]`, `#[UnitTest]`
- **New PHP Attributes for Queues:** `#[Job]`, `#[Job\Backoff()]`, `#[Job\MaxAttempts()]`, `#[Job\Timeout()]`, `#[Job\FailOnTimeout]`
- **Queue Routing** — `Queue::route()` for centralized queue/connection routing by job class

### New in Laravel 13.17 (June 23, 2026, v13.17.0)

- **Route metadata support** — `Route::metadata([...])` lets you attach arbitrary structured metadata to a route (audit info, feature flags, doc links, internal ownership). Query it back via `Route::getMetadata()` for tooling, dashboard generation, OpenAPI emitters, or to build route inventory reports. PR #60530 by @benbjurstrom.
- **PostgreSQL transaction pooler support** — first-class support for PgBouncer/Postgres pooler transaction mode in `Illuminate\Database\PostgresConnection`. Important when running Laravel behind serverless/edge platforms (Neon, Supabase, AWS RDS Proxy in transaction mode) where persistent connections are forbidden. PR #60425 by @DGarbs51.
- **`ShouldNotRetry` exception handler** — new `Illuminate\Contracts\Queue\ShouldNotRetry` interface — when an exception implements it, the queue worker will **fail** the job instead of retrying it, regardless of `$tries` / `maxExceptions`. Use for permanent failures (validation errors, missing records, 4xx upstream errors) that retrying cannot fix. PR #60552 by @alexbowers.
- **`php artisan dev:list` command** — companion to `php artisan dev` (added in 13.16). Lists every dev command registered for the current project (including vendor packages) so you can verify what's going to run before you run `dev`. PR #60573 by @joetannenbaum.
- **`--without-migration-data` flag on `DumpCommand`** — `php artisan schema:dump --without-migration-data` now emits **schema only** (no `INSERT` statements) even when migration data is included. Useful for greenfield DB provisioning or shipping schema files to read replicas. PR #60570 by @jackbayliss.
- **Cache debounce `maxWait` performance fix** — `Cache::flexible()` / `Cache::remember()` with `->debounce(...)->maxWait(...)` no longer re-checks the underlying source on every call when inside the debounce window. Significantly reduces cache hits / DB load for high-frequency debounced keys (rate limiters, hot idempotency keys). PR #60559 by @jackbayliss.
- **`between()` / `unlessBetween()` timezone-independence fix** — these Eloquent query helpers previously depended on the order of `->timezone()` calls. Now timezone conversion is applied per-value, so the order no longer matters. PR #60518 by @ManicardiFrancesco.
- **Clear transaction manager state on disconnect** — fixes leaked transactions / open savepoints that survived a DB disconnect. Prevents phantom "current transaction is aborted, commands ignored until end of transaction block" errors after a failover. PR #60574 by @lazerg.
- **`nightly` framework-install workflow** — new GitHub Actions workflow verifies the framework actually installs cleanly every night (catches regressions in `composer create-project`). PR #60532 by @jackbayliss.

**13.x hardening/cleanup in 13.17:**
- Tightened array-shape typehints where the first parameter is optional (#60553)
- Improved `InteractsWithData::when*` return types (#60536)
- `validateBoolean` / `validateNumeric` now accept `string|int|float` for the validated field type (#60549)
- Missing `@throws \ReflectionException` / `@throws \InvalidArgumentException` annotations on several helpers (#60535, #60546)
- `FileStore` cache deserialization fix for short timestamps (#60543)
- `DevCommands` vendor registration check no longer skips userland frames (#60538)
- `brick/math` 0.18 allowed (#60560)

**12.x backports included in 13.17:**
- **Multi-type union support in `Illuminate\JsonSchema`** — pair with the `fromArray()` deserializer from 13.15 (#60462)
- **Ability to refresh cache locks** — `Cache::restoreLock()` lets you re-acquire a previously-released lock by token (#58349, bytestream)
- **Guard `JsonSchema` deserializer against unbounded `$ref` expansion** — security fix that prevents attackers from crafting a JSON Schema that expands recursively and DoSes the validator. (Security fix — backported to both 12.x and 13.x.)
- **New error messages for detecting lost connections** — `Connection::disconnected()` now throws a `LostConnectionException` with the actual driver error, not a generic "gone away" message (#60472)
- **Postgres `whereDate()` / `whereTime()` crash when column is `Expression`** — fixed (#60540)

**Migration from 13.16.1 → 13.17.0:** no breaking changes. Standard `composer update laravel/framework` is safe.

**Where each 13.17 feature lives in this skill (cross-references):**
- `Route::metadata()` / `Route::getMetadata()` → see `controllers.md` (Route Metadata section)
- `php artisan dev:list` → see `artisan.md` (Inspect Registered Dev Processes section)
- `ShouldNotRetry` exception handler → see `queues.md` (ShouldNotRetry Exception Handler section)
- Postgres transaction pooler support (PgBouncer / Neon / Supabase / RDS Proxy) → see `deployment.md` (Postgres Transaction Pooler Support section)
- `--without-migration-data` flag on `schema:dump` → see `migrations.md` (Running Migrations section)
- `Cache::flexible()` / `Cache::remember()` debounce `maxWait` performance fix → see `performance.md` (Cache debounce subsection)

### New in Laravel 13.16 (June 16, 2026, v13.16.0 — patch v13.16.1)

- **`php artisan dev` command** — new first-party dev orchestration command. Runs server, queue worker, log tailing, and Vite concurrently — replaces per-project `composer dev` scripts. Configure via `Illuminate\Foundation\Console\DevCommands`. Package-aware (vendor packages are skipped). Auto-detects Node package manager (npm/yarn/pnpm/bun). **Upgrade to v13.16.1** which fixes a registration bug.
- **`whenFilledEnum()` request helper** — `Request` now has `$request->whenFilledEnum('status', Status::class, fn(Status $s) => ...)` — combines `whenFilled()` + `tryFrom()` + null guard in one call. Invalid enum values silently skip the callback (no throw). Optional default callback runs when the primary callback doesn't.
- **`withCookies()` on all responses** — promoted from `RedirectResponse` to `ResponseTrait`. Attach multiple cookies to any response type (`JsonResponse`, standard `Response`) in a single call: `response()->json($data)->withCookies([$cookieA, $cookieB])`. Additive, non-breaking.
- **`array` maintenance mode driver** — new `array` driver joins `file` and `cache`. Designed for **parallel testing** where file driver + `Cache` facade mocking can break tests that call `php artisan up`/`down`. Set `APP_MAINTENANCE_DRIVER=array` in `.env.testing`.
- **`anyOf` JSON Schema support** — `JsonSchema` now supports `anyOf` and multi-type unions. Deserializer is hardened against unbounded `$ref` expansion (prevents recursion attacks).
- **Enums in `broadcastAs()`** — broadcast events can return an enum from `broadcastAs()` for the event name; pairs with typed SDK generators so client/server event names stay in sync.
- **Queue attributes on traits** — `#[Job]`, `#[Tries]`, `#[Backoff]`, `#[Timeout]`, `#[FailOnTimeout]`, `#[DebounceFor]` attributes now work on traits used by jobs (not just on job classes directly).
- **`Batchable::batching` finished-batch fix** — fixed crash when accessing `Batchable::batching` after a batch has finished.
- **HTTP client serialization hardening** — request and fake-response serialization is now hardened against malicious input.
- **Improved return types** — generic types added to `WorkerIdle`, `WorkerLooping`, `listenForSignals`, model scope callbacks, connection callbacks, and `DatabaseTransactionsManager` getters for better static analysis.

**Patch highlights (v13.16.1):** Fixes a bug where `DevCommands` could stop itself from registering when packages attempted to register commands. Always upgrade to 13.16.1 over 13.16.0.

### New in Laravel 13.15 (June 9, 2026, v13.15.0)

- **Typed translation accessors** — `trans()->string($key)` and `trans()->array($key)` return concrete `string` / `array` types instead of `array|string|null`. Mirrors the typed helpers like `config()` and `request()` — improves static analysis and IDE inference. Available on both `trans()` and `__()` helpers.
- **`JsonSchema::fromArray()` deserializer** — turn a raw JSON Schema array back into `Type` objects (inverse of serialization). Pairs with multi-type union support in the schema builder.
- **Dedicated Cloud queue driver (deepened)** — Laravel Cloud's managed queue driver is now a first-class connection. Several changes land together:
  - Managed queues boot **before** service providers (jobs ready before app boots)
  - Throws `ManagedQueueNotFoundException` when a configured managed queue is missing
  - FIFO queue name normalization corrected
  - Request ID header renamed `X-Request-ID` → `Cloud-Request-ID` and now emitted into logs for distributed tracing
- **Enums in `Queue::route()`** — pass enum cases for both queue and connection when routing jobs: `Queue::route(Job::class, connection: MyQueue::Redis, queue: QueueName::Emails)` instead of strings.
- **`Prohibitable` on `cache:clear` and `queue:flush`** — both commands now respect the `Prohibitable` interface (already added to `queue:clear`, `config:clear`, `key:generate`).
- **`Macroable` on `InvokedProcess`** — add macros to `InvokedProcess` for custom subprocess behavior.
- **`Cache::rememberWithWarmth()`** — `Cache::remember()`'s stateful twin. Returns `array{value, bool}` where the bool indicates whether the value came from the cache. Lets you log cache-hit/miss, set `X-Cache` headers, or skip expensive post-processing on a cache hit — without a separate `Cache::has()` round trip. `Cache::remember()` is now a thin wrapper around this. (PR #60385)
- **`queue:failed` real class name** — fixed bug where `php artisan queue:failed` displayed the wrapped/obfuscated class name instead of the real job class.
- **Compiled Blade views not left expired** — fixed bug where unchanged compiled Blade views were unnecessarily treated as expired.
- **Number helper fixes** — `Number::fileSize()` handles negative byte values; `Number::trim()` no longer returns null for `INF`/`NAN`; `Number::pairs()` no longer infinite-loops on negative step and throws on zero step.
- **Generics on queue routes** — `QueueRoutes::all()` is now properly typed.

**Security fixes in v13.15.0:**
- **`date_equals` validation bypass** — the `date_equals` rule used loose comparison, so invalid date strings parsing to `null` could pass against reference dates parsing to falsy values (e.g., `1970-01-01`). Fixed by using strict equality for the equal operator while keeping loose comparison for legitimate `DateTime` objects. **Upgrade critical for apps exposing `date_equals` to untrusted input.**
- **Restricted route unserialization** — route caching/resolution unserialization now restricts accepted classes, reducing object-injection surface during route loading.

**Other fixes:** Infinite recursion when a model scope is defined with a private attribute; infinite recursion when a middleware group references itself; FIFO normalization in Cloud managed queues.

### New in Laravel 13.14 (June 2026, v13.14.0)


- **Lazy refresh hook on all connections** — database connections now register a lazy refresh hook that resets stale connections automatically on use, improving reliability in long-running workers
- **Cache falsy JSON payloads in HTTP client** — falsy values (null, false, 0, "") are now cached in HTTP client JSON responses. Previously `null` responses were not cached at all
- **`Request::createFromBase()` Symfony 8.1 compatibility** — fixed compatibility issue when creating Request objects from Symfony 8.1 Request instances
- **`Message::embed` data attachment fix** — fixed handling of data attachment URLs in mail message embedding, correcting email rendering issues
- **Cloud logging formatter namespaced** — CloudLogFormatter is now properly namespaced, fixing cloud deployment logging issues
- **Child queue properties over inherited attributes** — when a job extends another job class, child job properties now correctly take precedence over parent class attributes (e.g., $tries, $timeout) instead of being overwritten
- **JSON Schema array deserializer** — new `arrayDeserializer` for JSON Schema validation, allowing proper deserialization of array-typed fields from JSON Schema definitions
- **`InspectedJob` queue name** — InspectedJob now carries the queue name, improving job observability and debugging in cloud queue workers
- **`#[DebounceFor]` units parameter** — the `#[DebounceFor]` queue attribute now supports explicit time units: `#[DebounceFor(30, 'seconds')]`, `#[DebounceFor(2, 'minutes')]`, etc.
- **Null header handling fix** — fixed crash when HTTP headers contain null values in the Header class

**Patch release highlights:** v13.14.0 is primarily a bug-fix and compatibility release:
- Child queue job property inheritance fix (queues using base classes)
- Falsy HTTP response caching (reduces redundant API calls)
- Lazy refresh hook (fewer stale DB connection errors in workers)

### New in Laravel 13.13 (June 2026, v13.13.0)

- **MariaDB Vector Index support** — new vector index capability for AI/embeddings workloads directly in migrations: `$table->vectorIndex(['embedding'], 'vector_index', 'vector_length:1536)` (requires MariaDB with vector support)
- **`Bus::bulk()`** — new method for dispatching multiple jobs at once with a single call. Useful for batch processing: `Bus::bulk([new ProcessItem(1), new ProcessItem(2), new ProcessItem(3)])`
- **`Cache` attribute memoization** — `#[Cache(ttl: 300)]` now supports memoization — the callback is only executed once and the result is cached for subsequent calls within the same request
- **`Http::asPsrClient()`** — HTTP facade now implements `Psr\Http\Client\ClientInterface`, allowing Laravel's Http client to be used as a drop-in PSR-18 client
- **`attachFromStorage` for Mailables** — `MailMessage::attachFromStorage()` convenience method for attaching files from Laravel storage disks to notifications/mails
- **`InspectedJob` payload** — queue jobs can now carry an arbitrary payload array via `InspectedJob`, useful for observability and debugging
- **Cloud managed queue FIFO fix** — fixed FIFO name normalization in Cloud managed queues (backported from 13.11 fix)
- **SQL Server unique constraint error handling** — `isUniqueConstraintError()` now catches SQL Server error 2627
- **`whereDate`/`whereTime` crash fix** — fixed crash when `$column` is a `Query\Expression`
- **`assertJsonPathsCanonicalizing`** — new `TestResponse` assertion: `$response->assertJsonPathsCanonicalizing($paths)` for order-independent JSON comparison
- **`Str::studly()`/`Str::pascal()` normalize parameter** — both methods now accept `normalize: true` to strip non-alphanumeric chars before casing: `Str::studly('hello-world_test', normalize: true)` → `HelloWorldTest`
- **Image dimension validation operator fix** — fixed inverted ratio comparison in dimension validation rules
- **Scheduler opt-out of pause/interrupt cache** — scheduled tasks can now opt out of worker pause and interrupt cache checks via attribute
- **Event skipping indication** — events can now signal they were intentionally skipped in listeners
- **`UniqueFor` hint unit** — improved type hinting for `UniqueFor` validation rule

### New in Laravel 13.12 (May 2026, v13.12.0)

- **Worker restart on lost connection opt-out** — queue workers can now be configured to NOT restart on lost database/Redis connection via `Queue::preventWorkerRestartOnLostConnection()`
- **Scheduler attributes** — `Schedule` now supports custom attributes for grouping/organizing scheduled tasks: `->attributes(['group' => 'critical'])`
- **`Str::studly()`/`Str::pascal()` normalize parameter** — see v13.13 entry above (same feature, noted in both releases)
- **`assertJsonPathsCanonicalizing`** — see v13.13 entry above
- **`queue:clear` default param null** — `queue:clear` command now defaults `driver` param to `null` instead of requiring explicit driver argument
- **Scheduler callback parameter resolution** — scheduled event callbacks are now resolved by type rather than parameter name, fixing edge cases with DI
- **`compact()` removal** — all remaining `compact()` calls in framework replaced with explicit arrays (compatibility with PHP 8.4)
- **SQLite `file:` prefix URI support** — SQLite connections now support `file:` prefix in DSN: `sqlite://file:/path/to/database`
- **ClearCommand prohibitable** — `queue:clear`, `cache:clear`, `config:clear` now implement `Prohibitable` interface
- **Auto-discovered listeners opt-out** — event listeners can opt out of auto-discovery via `#[Discoverable(enabled: false)]`
- **`Up`/`Down` commands exception reporting** — `php artisan up` and `php artisan down` now properly report exceptions instead of silently failing
- **`Number::spell()` type fix** — fixed incorrect `@return` types in `Number::spell()`, `ordinal()`, and `spellOrdinal()`
- **Async HTTP retries with array backoff** — async HTTP client retries now work correctly with array backoff values like `[1, 5, 10]`
- **PSR-7 multipart array handling** — fixed multipart/form-data handling for PSR-7 responses
- **`Optional::offsetUnset()` docblock** — fixed incorrect docblock type hint
- **`factory()->pivot()` stub** — new `factory()->pivot()` method generates pivot table model factories
- **`KeyGenerateCommand` prohibited** — `php artisan key:generate` now respects the `Prohibitable` interface

### New in Laravel 13.11 (May 2026, v13.11.2)

- **Dedicated Cloud Queue** — new `Illuminate\Foundation\Cloud\Queue` decorator wraps any queue driver (Redis, SQS, etc.) with cloud-native instrumentation. Tracks currently processing job, job start timestamps, and dispatches cloud-specific events. Use via the `cloud` queue connection.
- **Cloud-Request-ID in logs** — `Cloud-Request-ID` header value is automatically written to log entries for request tracing across services.
- **Boot managed queues before service providers** — cloud queue service providers now boot before application service providers, ensuring queue workers are ready before the app starts processing jobs.
- **Lifecycle deferred event method fixes** — fixed edge cases in event deferral for cloud queue lifecycle events.

### New in Laravel 13.10 (May 2026, v13.10.0)

- **WorkerIdle event** — new `WorkerIdle` event dispatched when the queue worker has no jobs to process. Useful for cleanup, metrics, or scaling down workers. Receives `WorkerOptions`.
- **assertPushedOnce()** — new `Queue::assertPushedOnce()` assertion to verify a job was pushed exactly once (alias for `assertPushed($job)` with exactly 1 call count).
- **starts_with/ends_with accept numeric values** — `starts_with` and `ends_with` validation rules now correctly accept numeric (integer/float) input values.
- **Validate line breaks in emails** — email validation now rejects values containing line breaks (potential email header injection).
- **storage store artisan command** — new `php artisan storage:store` command to write files to storage disks programmatically.
- **URL-encode paths** — Laravel now properly URL-encodes paths in routing to prevent traversal issues.
- **Schedule::group() lifecycle callbacks** — `Schedule::group()` now supports `before()`, `after()`, `onFailure()`, and `onSuccess()` callbacks.
- **Delimit aggregate aliases** — aggregate query aliases (COUNT, SUM, AVG) are now properly delimited in the query builder.
- **SQS overflow store flush on queue:clear** — `php artisan queue:clear` now optionally flushes the SQS FIFO overflow store.
- **queue:list --json output** — `php artisan queue:work --stop-when-empty` option for CI/CD scripts.
- **WorkerLooping event now receives WorkerOptions** — `WorkerLooping` event listeners can now access `WorkerOptions` for context-aware handling.
- **Cloud-Request-ID header** — request tracking header renamed from `X-Request-ID` to `Cloud-Request-ID` for clarity in cloud deployments.

### New in Laravel 13.9 (May 2026)

- **PreparesForDispatch interface for Jobs** — new `PreparesForDispatch` interface lets jobs execute preparation logic before being dispatched to the queue. Useful for validation, enrichment, or state checks that run synchronously at dispatch time rather than async in handle().
- **PendingDispatch conditionable** — `PendingDispatch` now supports `->when()` and `->unless()` conditionals for cleaner conditional dispatch logic.
- **Migration events with name** — `MigrationStarted` and `MigrationEnded` events now carry the migration name for more granular event handling.
- **Generic return types on Builder paginate** — `paginate()`, `simplePaginate()`, `cursorPaginate()` now return properly typed `LengthAwarePaginator`/`Paginator` instances.
- **Queue:pause error display** — `php artisan queue:pause` now shows a clear error when the Worker is not pausable.
- **ThrottlesExceptions Closure support** — `ThrottlesExceptions` middleware now accepts a Closure for dynamic, exception-type-aware rate limiting.
- **foreignUuidFor schema helper** — `$table->foreignUuidFor(Model::class)` collapses UUID FK + index + constraint into one call.
- Internal: `mt_rand()` replaces `rand()`, `preg_split` replaces `mb_split`, PSR-7 multipart array handling fixed.

### New in Laravel 13.8 (May 2026)

- **DatabaseLock excludes expired locks** — `DatabaseLock::isLock` now properly excludes expired locks when checking ownership, fixing race conditions in lock cleanup
- **Worker Pausing/Resuming events** — new `WorkerPausing` and `WorkerResuming` events dispatched by queue workers
- **assertSessionMissingInput** — new TestResponse assertion to verify old input is NOT present in session after form submission
- **assertPushedOn enum support for QueueFake** — `QueueFake::assertPushedOn()` now accepts an enum as the queue argument
- **Custom ON DELETE/UPDATE for foreign keys** — `foreignId()->constrained()->onDelete()->onUpdate()` now supports fully custom constraint actions
- **LocalScope private recursion fix** — `LocalScope` now properly handles private methods without infinite recursion
- **schedule:list environment filter** — `php artisan schedule:list --environment=production` filters output by environment
- **Collection min/max generic result type** — `Collection::min()`/`max()` now return `T|null` instead of `mixed`
- **All queue inspection methods** — new `allPushed()`, `allNotPushed()`, `allPushedOn()` methods on QueueFake for batch assertions
- **SortDirection enum** — `Illuminate\Database\Query\SortDirection` provides type-safe `Ascending`/`Descending` for `orderBy()` instead of string `'asc'`/`'desc'`

### New in Laravel 13.7 (April 2026)

- **Debounceable Queued Jobs** — `#[DebounceFor]` attribute keeps only the last dispatch within a time window. Eliminates redundant processing from bursty workloads (e.g., user edits same doc 10x in 30s → 1 rebuild instead of 10).
- **JSON support for health route** — `/up` health endpoint now supports returning JSON response data for richer health status
- **JsonFormatter** — native `Monolog\Formatter\JsonFormatter` support for structured JSON log output
- **Cloudflare Email Service support** — new mail driver integration for Cloudflare's email routing
- **@fonts Blade directive** — `@fonts` generates `<link rel="preload">` tags for Vite-managed fonts, improving Core Web Vitals

### New in Laravel 13.5 (Late April 2026)

- Queue infrastructure maturity improvements across v13.3 through v13.6 releases

### Breaking Changes from 12

- PHP 8.2 minimum (8.3 recommended)
- `assertStatus()` fully removed (use `assertSuccessful()` or `assertStatus(200)`)
- Some deprecated helpers removed
- `config/app.php` structure may need verification

### Migration to Laravel 13

```bash
composer require laravel/framework:^13.0 --with-all-dependencies
php artisan --version  # verify
```

### Laravel 13 AI Features

```bash
# Install Boost MCP server
php artisan boost:install

# Use AI-assisted upgrade
/upgrade-laravel-v13
```

---

## Laravel 12 (February 2025, v12.62.0)

### New in Laravel 12

- **Application Starter Kits** — React, Svelte, Vue, and Livewire with Shadcn components
- **PHP 8.2 minimum required** — supports 8.2, 8.3, 8.4, 8.5
- **Bootstrap 5 by default** — no more Bootstrap 4
- **Vite as default bundler** — Laravel Mix officially deprecated
- **Health endpoint at `/up`** — built into framework, no route needed
- **`php artisan serve` accepts `--host` and `--port`** natively
- **`Queue::after()` and `Queue::failure()` methods** — cleaner queue events
- **Per-second rate limiting** — `RateLimiter::for('api', ...)` supports `perSecond()`
- **`Str::stripTags()`** — better than `strip_tags()`
- **Laravel Reverb** — first-party WebSocket server

### Breaking Changes from 11

- `bootstrap/app.php` no longer registers all service providers by default
- `config/app.php` no longer has `aliases` array
- Route middleware registered in `bootstrap/app.php` via `$middleware->`
- `App\Http\Kernel` and `App\Console\Kernel` fully removed

### Laravel 12 Latest Patch (v12.62.0 — June 9, 2026)

v12.62.0 is a **patch release** — contains bug fixes and features backported from Laravel 13, no new breaking features:
- **`JsonSchema::fromArray()` deserializer (backport from 13.15)** — turn a raw JSON Schema array back into `Type` objects. Pairs with the new multi-type union support.
- **Multi-type union support in `JsonSchema` (backport from 13.15)** — `Illuminate\JsonSchema` now supports `anyOf` and multi-type unions.
- **PostgreSQL `compileColumns()` performance fix (backport from 13.15)** — skips `pg_collation` lookup on PostgreSQL < 9.1.
- **Cloud queue support (backport from 13.11.2)** — managed queues boot before service providers; `ManagedQueueNotFoundException` thrown when missing; FIFO name normalization corrected.
- **HTTP attach empty contents preserved (backport from 13.15)** — `GrahamCampbell` fix that prevents empty attachment bodies from being dropped.
- **`Number` helper fixes (backport from 13.15)** — `Number::trim()` no longer returns null for INF/NAN; `Number::pairs()` no longer infinite-loops when `$by <= 0`; `Number::fileSize()` handles negative byte values.
- **LocalFilesystemAdapter path separators (backport from 13.15)** — separators are no longer encoded in the Local adapter.
- **Env parser regex fix (backport from 13.15)** — `Env::addVariableToEnvContents` quoting fix.
- **Deprecation logging fix (backport from 13.15)** — `config` is now bound before logging the deprecation notice.
- **queue:failed real class name (backport from 13.15)** — `clementmas` fix so the command shows the actual job class instead of an alias.
- **SQS / Cloud queue name normalization** — FIFO queue names normalized correctly.
- **No new features beyond backports** — purely maintenance

---

## Laravel 11 (2024)

### New in Laravel 11

- **Application skeleton cleaned** — far less boilerplate
- **`bootstrap/app.php`** replaces `Kernel.php` for middleware registration
- **`config/app.php`** much shorter — providers and aliases removed
- **Per-second rate limiting** via `RateLimiter::for()` with `->perSecond()`
- **Health check endpoint** — `Route::get('/up', fn() => 'ok')`
- **`php artisan about`** shows comprehensive environment summary
- **Password rehashing** — automatic on login
- **`assertSuccessful()`** on responses

### Breaking Changes from 10

- `App\Http\Kernel` removed — use `bootstrap/app.php`
- `App\Console\Kernel` removed — use `routes/console.php`

---

## Laravel 10 (2023 — LTS, Security until late 2026)

### New in Laravel 10

- **PHP 8.1 minimum required**
- **`Illuminate/Validation/Rules/ProhibitedBy`**
- **Native rate limiting** — `RateLimiter` in `RouteServiceProvider`
- **`RateLimiter::for()`** accepts callable returning `Limit` objects
- **Scheduled tasks on one server** — `onOneServer()` method
- **`Model::preventsAccessingMissingAttributes()`** — lazy-loading guard

---

## Version-Agnostic Best Practices (Always Apply)

### Always Use

- PHP 8.3+ (Laravel 13 requires 8.3+, optimized for 8.4)
- Composer 2.x
- Vite (not Mix — Mix deprecated in 11+)
- SQLite for local dev (zero config)
- `php artisan key:generate` on fresh install
- `assertSuccessful()` not `assertStatus(200)` in tests

### Deprecation Timeline

| Feature | Deprecated In | Removed In |
|---|---|---|
| Laravel Mix | 11 (use Vite) | 12 |
| `assertStatus()` | 10 | 13 |
| `php artisan make:model -mcr` shorthand | Still works | — |
| `App\Http\Kernel` removed | 11 | 12 |
| `App\Console\Kernel` removed | 11 | 12 |
| `config/app.php` aliases array | 11 | 12 |
| PHP 8.0/8.1/8.2 support | 13 | — |
| Bootstrap 4 default | 12 | — |

### Version Detection

```bash
# Check Laravel version
php artisan --version
# Laravel 13.14.0 (June 2026)

# Check PHP version
php -v
# PHP 8.4.x

# Check composer
composer --version
# Composer 2.x
```

---

## Agent Checklist Per Task

Before working on any Laravel task:
- [ ] Identify Laravel version (`php artisan --version`)
- [ ] Check PHP version compatibility (8.3+ for Laravel 13)
- [ ] Load version-specific rules above
- [ ] Check if feature is version-specific
- [ ] Apply version-specific patterns, not generic ones

---

Sources: [Laravel 13 Release Notes](https://laravel.com/docs/13.x/releases) | [GitHub Releases](https://github.com/laravel/framework/releases) | [Packagist](https://packagist.org/packages/laravel/framework)

---

## Ecosystem Update (2026-06-28, cycle 5)

### Slow JSON Stream DoS Disclosure — Tier 1 for Laravel

PHP/Laravel is now flagged as **Tier 1 (CVSS 7.5 HIGH)** in the cr0hn "Slow JSON Stream" DoS research (June 27, 2026). No framework version update mitigates this — the fix is at the deployment layer (nginx `client_body_timeout` + `client_min_rate` + PHP-FPM `request_terminate_timeout`). See `security.md` for the full mitigation playbook.

### No New Framework Version (Laravel 13.17.0 still latest)

As of June 28, 2026, Laravel framework **v13.17.0** (June 23, 2026) remains the latest. Laravel 13.18 has not been released yet. Track [github.com/laravel/framework/releases](https://github.com/laravel/framework/releases) for the next tag.

