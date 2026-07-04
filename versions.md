# Laravel Version-Specific Requirements

## Active Versions

- **Laravel 11** — **End of life** (security support ended March 12, 2026). Upgrade to 12 or 13.
- **Laravel 12** — Active development (v12.62.0 as of June 9, 2026; bug fixes until Aug 13, 2026, security until Feb 24, 2027)
- **Laravel 13** — Current latest (v13.18.1 as of July 2, 2026; bug fixes until Q3 2027, security until Mar 17, 2028)

## Version Selector Prompt

When working with Laravel, start by determining the version. Ask the user or check `composer.json`:
```bash
grep '"laravel/framework"' composer.json | grep -oP '\d+\.\d+'
```

Then load the relevant sections below.

---

## Laravel 13 (Latest — July 2026, v13.18.1)

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

### New in Laravel 13.18 (June 30, 2026, v13.18.0)

Tagged 2026-06-30 12:55 UTC. **All post-v13.17.0 fixes originally tracked in the previous "Late-June 2026 Bug Fixes" section below shipped together in this release** — there was no v13.17.1. `composer update laravel/framework` is the safe upgrade path; no breaking changes.

**Bug fixes (16 PRs):**

- **`schedule:work` graceful signal handling** — `SIGINT`/`SIGTERM`/`SIGQUIT` now stop new runs cleanly and wait for in-flight `schedule:run` subprocesses to finish (was hard-killed mid-task before). Long-lived scheduler processes in Docker/K8s/supervisord no longer need wrapper scripts. PR #60616 (commit ffa562df, 2026-06-28). Covered in `artisan.md` (schedule:work section).
- **`WorkerStopping` event payload** — new public `\$jobsProcessed` (int) and `\$lastJobProcessedAt` (?Carbon) properties; previously only `\$status`/`\$workerOptions`/`\$reason` were public. Enables lifetime-job-count dashboards and graceful-shutdown-gap tracking without reflection hacks. PR #60592 (commit fde9dd43, 2026-06-26) + PR #60608 (ensures `lastJobProcessedAt` is null if nothing was processed). Covered in `queues.md` (WorkerStopping section).
- **Soft-delete `restore()` event gating** — the `restored` model event now fires **only** when the underlying `save()` returns `true`; before, it fired unconditionally even when a `saving` listener cancelled the restore. Sibling change: `restoring` is now gated the same way. PR #60605 (commit eb473d3c, 2026-06-26). Covered in `observers.md`.
- **`HEAD` request cache headers** — `SetCacheHeaders` middleware now applies to `HEAD` (was a 13.16 regression — CDN cache-warming scripts and `link rel="preload"` audits were silently missing `Cache-Control`/`ETag`). PR #60589 (commit a1ecec74, 2026-06-25). Covered in `controllers.md`.
- **`RateLimited` middleware `releaseAfter` `__sleep` fix** — rate-limit release timestamps now serialize through `__sleep()`, so jobs that serialize/unserialize (e.g. via `dispatch($job->afterCommit())`) don't lose their throttle window. PR #60609 (commit a679b90b, 2026-06-27). Listed for awareness — main API unchanged.
- **Conditional return types on several methods** — `Paginator::fragment()`, `Route::domain()`, `Password/File/Email` rule `defaults()`, `__()`, `Lottery::`, `Arr::`, `ComponentAttributeBag::` all narrow return types when the first arg is non-null. PHPStan/Psalm ergonomics. PR #60586 (commit 4eef2ca8, 2026-06-25).
- **Sync getter return types with property generics** — collection / model / builder getters now expose their actual generic types so static analysis can follow `.pluck()` / `->map()` chains into the right element type. PR #60591.
- **`Request::json()` top-level zero body fix** — a literal `0` body was previously coerced to `[]` when read via `Request::json()` or `Request::all()`. Now preserved. PR #60614 (commit 1c0c8fb5, 2026-06-28). Watch the edge case: numeric strings (`"0"`) still come through correctly — the fix only affected numeric `0`. Covered in `api.md` (Error Responses section).
- **`Number::forHumans` / `Number::abbreviate` OOM on `INF` / `NaN`** — these methods previously recursed into `Number::summarize()` without a base case for non-finite inputs and exhausted PHP memory (the process aborted silently — remote-DoS hole on any API response that renders user input through these helpers). Now they delegate to `Number::format()` and emit the locale-aware `∞` / `-∞` / `NaN` symbols. PR #60617 (commit 3a3c1d2b, 2026-06-27). Covered in `performance.md` and `api.md`.
- **`Number::fileSize` wrong unit suffix for non-finite inputs** — sibling fix to the above; `Number::fileSize(INF)` no longer returns a unit-suffixed string for an unbounded value. PR #60625 (commit 7e9d4aa1, 2026-06-29). Same fix pattern, same coverage.
- **`php artisan dev` priority-based registration** — vendor package dev commands can now register a priority so they appear in a deterministic order in `php artisan dev` / `dev:list` output (no breaking change to existing registrations — they just default to a lower priority). PR #60580 (commit 03a7c1a2, 2026-06-23). Useful when a package's "log tail" or "queue worker" should be the first thing the user sees.
- **`php artisan dev --kill-others-on-fail`** — companion flag added in PR #60606. If any registered dev process exits non-zero, sibling processes (server, queue worker, log tail, Vite) are killed immediately instead of leaving zombie workers around. Set this in CI / one-shot dev boxes; leave off for normal local dev where one component dying shouldn't tear everything down.
- **TaggedCache `flexible()` lock + defer label collision fix** — `Cache::flexible()` with a custom `lockName` colliding with a `defer()` label could previously cause the lock to be released under the wrong key. Now both namespaces are namespaced (`flex_lock:` vs `flex_defer:`). PR #60626 (commit 1f95687c, 2026-06-29).
- **Debounced jobs cache-hit reduction** — `Cache::flexible()` / `Cache::remember()` with `->debounce(...)->maxWait(...)` now skips the lock acquisition when already inside the debounce window. Pairs with the 13.17 fix (PR #60559) — together they eliminate a class of "thundering herd on hot debounce keys" symptoms. PR #60575.

**Minor / housekeeping:**
- GitHub Actions group bump (dependabot) — PR #60622.

**Upgrade notes:** No breaking changes vs 13.17. Standard `composer update laravel/framework`. The four most operationally important fixes for production:
1. **`Number::forHumans` / `Number::abbreviate` / `Number::fileSize` INF/NaN guard** (PR #60617 + #60625) — remote-DoS hole on any endpoint rendering user-controllable numerics. **If you can't upgrade yet, see `api.md` DoS section for input-clamping wrap.**
2. **`Request::json()` zero body** (PR #60614) — for `PUT /counters/.../reset` style endpoints with body `0`, drop the `?? 0` workaround after upgrade.
3. **Soft-delete event gating** (PR #60605) — code that assumed `restored` always fired after `restore()` may need to check the boolean return.
4. **HEAD cache headers** (PR #60589) — CDN cache-warming scripts finally get `Cache-Control` / `ETag` on HEAD probes.

**Where each 13.18 feature lives in this skill (cross-references):**
- `schedule:work` signal handling → `artisan.md` (schedule:work section)
- `WorkerStopping` payload + graceful shutdown → `queues.md` (WorkerStopping section)
- Soft-delete `restore()` / `restoring` event gating → `observers.md` (restored callback note)
- HEAD cache headers → `controllers.md` (API Versioning section)
- `Number::forHumans` / `abbreviate` / `fileSize` INF/NaN guard → `performance.md` (Number OOM section) + `api.md` (Number DoS section)
- `Request::json()` top-level zero → `api.md` (Error Responses section)
- `php artisan dev` priority + `--kill-others-on-fail` → `artisan.md` (Dev Orchestration section)
- TaggedCache `flexible()` lock/defer namespace fix → `performance.md` (TaggedCache subsection)

### New in Laravel 13.18.1 (July 2, 2026, v13.18.1)

Tagged 2026-07-02 18:36 UTC (two days after 13.18.0). Pure patch release — `composer update laravel/framework` is safe; no breaking changes. Mostly bug fixes and developer-experience improvements, plus one genuinely useful new feature (`Release` queue middleware) and one JSON parsing hardening (`api`/`json` routes respect `Down` maintenance).

**New features (2 PRs):**

- **`Release` queue middleware (`Illuminate\Queue\Middleware\Release`)** — declarative companion to `$this->release()` / `$this->release($delay)`. Attach via `->middleware(new Release($delayInSeconds))` so a job automatically releases itself back to the queue after the middleware runs (without your `handle()` needing to call `release()` manually). Use for "try, and if it would conflict, hand off to another worker" patterns without scattering release calls across every job class. PR #60630 by @mgcodeur. Covered in `queues.md` (Job Middleware section).

- **`input()` method on console commands** — first-class CLI input reader that returns a typed array, parallel to `request()->input()` for HTTP. `$this->input('email')`, `$this->input(['email', 'name'])`, `$this->input()` (all). Cleaner than `$this->argument()` + `$this->option()` + manual casting in every command. PR #60607 by @stevebauman. Covered in `artisan.md` (Defining Commands section).

**Bug fixes (6 PRs):**

- **`assertDatabaseEmpty()` accepts iterables** — previously only worked with table-name strings; passing an Eloquent collection or Builder threw `Argument #1 must be of type string`. Now accepts `string|iterable`. Lets you write `$this->assertDatabaseEmpty(User::all())` or `$this->assertDatabaseEmpty([$tableA, $tableB])`. PR #60621 by @jackbayliss. Covered in `testing.md`.

- **`api` / `json` routes respect `php artisan down` (Maintenance)** — when the app is in maintenance mode with `secret` bypass, `php artisan down --secret=...`, requests to `/api/*` or routes that set `Accept: application/json` were not always gated correctly. Now the `Down` command handles both web and API/JSON routes uniformly — maintenance mode responses return JSON (`{"message": "Service Unavailable", "retry_after": N}`) for API/JSON callers instead of falling through to a 500. PR #60595 by @davidrushton. Covered in `controllers.md` + `deployment.md` (Maintenance section).

- **Channel name respected in `on-demand` log stacks** — `Log::build([...])`/`Log::stack([...])` with `driver: 'errorlog'` or `driver: 'monolog'` on an `on-demand` channel was previously ignoring the configured `channel` name in some callsites (logs landed under the parent's channel). Each channel now writes to its own configured name. PR #60635 by @maltf0. Covered in `logging.md` (Stacked / On-Demand Channels section).

- **Inspect delayed jobs on `Queue::fake()`** — `$this->artisan(...)` test helper plus `Queue::fake()` previously lost delayed-job metadata when inspecting the queue fake after a dispatch (`assertPushed` worked, but `assertReleased`/`assertDelayed` couldn't see delays). Now the queue fake tracks the `availableAt` timestamp so you can assert `$queue->assertPushed(Job::class)->delay(60)`-style inspections. PR #60636 by @jackbayliss. Covered in `testing.md` (Queue Assertions section).

- **`Str::mask()` respects encoding at the end of the string** — when the mask would truncate a multi-byte character at the tail, `Str::mask()` previously chopped the encoding and emitted a partial character (rare display bugs on UTF-8 inputs that hit the mask boundary). Now pads with the encoding-aware character instead. PR #60646 by @iammcoding. Covered in `validation.md` + `security.md` (output encoding section).

- **Predis retry config supports scalars for `config:cache`** — `config/database.php` `redis.options.read_write_timeout` and similar Predis options historically required arrays (so `php artisan config:cache` blew up when you set them to scalar values). Now scalars are accepted and passed through. PR #60642 by @crynobone. Niche — affects Predis users only; PhpRedis unaffected.

**Docblock / type-only fixes (1 PR):**

- **`foreignUuid` / `foreignUlid` Blueprint return types** — `foreignUuid()` / `foreignUlid()` on Blueprint now declare `ColumnType::Uuid` / `ColumnType::Ulid` to match `foreignId()`, so schema-dumpers and migration generators handle them uniformly. No runtime behavior change. PR #60643 by @LiddleDev. Covered in `migrations.md` (Foreign Key Constraints section).

**Upgrade notes:** No breaking changes vs 13.18.0. Drop-in safe to upgrade. The two most useful additions to actually pull from 13.18.1 in your codebase:

1. **`Release` middleware** (PR #60630) — lets you delete a handful of `$this->release(...)` calls scattered across job classes and centralize the release policy in `withMiddleware()` / job default middleware. Worth doing the next time you touch a job that releases itself.
2. **`api`/`json` routes respect `Down`** (PR #60595) — if you've hand-rolled a "maintenance mode" middleware for API routes because the built-in one didn't catch them, you can now delete it.

**Where each 13.18.1 change lives in this skill (cross-references):**
- `Release` queue middleware → `queues.md` (Job Middleware section)
- `input()` on console commands → `artisan.md` (Defining Commands section)
- `assertDatabaseEmpty()` iterable → `testing.md` (Database Assertions section)
- `api`/`json` routes respect `Down` → `controllers.md` (Maintenance section) + `deployment.md` (Maintenance Mode section)
- `on-demand` log stack channel name → `logging.md` (Stacked / On-Demand Channels section)
- Inspect delayed jobs on queue fake → `testing.md` (Queue Assertions section)
- `Str::mask()` encoding-aware tail → `validation.md` (Sanitization Helpers section) + `security.md` (Output Encoding section)
- `foreignUuid`/`foreignUlid` return types → `migrations.md` (Foreign Key Constraints section)
- Predis retry scalar config → `deployment.md` (Redis Configuration section)
### Late-June 2026 Bug Fixes — Now Shipped in v13.18.0 (History Note)

> **Status as of 2026-06-30:** every item originally listed in this section shipped in v13.18.0 on 2026-06-30 12:55 UTC. See the **"New in Laravel 13.18"** section above for the canonical description of each fix. This section is preserved as a historical trail showing the cycle-by-cycle tracking — useful when triaging incidents on older patch versions.

The PR numbers / commit SHAs / dates below are unchanged from when these were first merged into the `13.x` branch pre-release.

- **`schedule:work` graceful shutdown** — PR #60616 (commit ffa562df, 2026-06-28) → 13.18.0
- **`WorkerStopping` event payload** — PR #60592 (commit fde9dd43, 2026-06-26) → 13.18.0
- **Soft-delete `restore()` event gating** — PR #60605 (commit eb473d3c, 2026-06-26) → 13.18.0 (also gates `restoring`)
- **HEAD requests now set cache headers** — PR #60589 (commit a1ecec74, 2026-06-25) → 13.18.0
- **`RateLimited` middleware `releaseAfter` `__sleep` fix** — PR #60609 (commit a679b90b, 2026-06-27) → 13.18.0
- **Conditional return types on several methods** — PR #60586 (commit 4eef2ca8, 2026-06-25) → 13.18.0
- **`Json` parsing for top-level zero bodies** — PR #60614 (commit 1c0c8fb5, 2026-06-28) → 13.18.0
- **`Number::forHumans` / `Number::abbreviate` OOM on `INF` / `NaN`** — PR #60617 (commit 3a3c1d2b, 2026-06-27) → 13.18.0 (treated as remote-DoS vector on API responses that render user input)
- **`Number::fileSize` wrong unit suffix for non-finite inputs** — PR #60625 (commit 7e9d4aa1, 2026-06-29) → 13.18.0
- **`artisan dev` priority-based registration** — PR #60580 (commit 03a7c1a2, 2026-06-23) → 13.18.0
- **`artisan dev --kill-others-on-fail`** — PR #60606 (2026-06-30) → 13.18.0
- **TaggedCache `flexible()` lock + defer label collision fix** — PR #60626 (commit 1f95687c, 2026-06-29) → 13.18.0
- **Debounced jobs cache-hit reduction** — PR #60575 (2026-06-30) → 13.18.0

Watch [github.com/laravel/framework/releases](https://github.com/laravel/framework/releases) for **v13.18.1** (next patch — likely late July 2026 once enough post-13.18 fixes accumulate) or **v13.19.0** (next minor).

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


### Ecosystem Update (2026-06-28, cycle 7) — HISTORY (Laravel 13.17.0 was latest at this point)

#### No new Laravel framework version

Laravel framework **v13.17.0** (June 23, 2026) remains the latest stable as of 2026-06-28 12:00 UTC. v13.18 has not been tagged yet. Track [github.com/laravel/framework/releases](https://github.com/laravel/framework/releases) for the next release.

#### New CVEs in the Laravel ecosystem (cycle 7)

- **CVE-2026-45793** (Composer, May 13, 2026, CVSS 7.5 HIGH) — GitHub `GITHUB_TOKEN` and GitHub App installation tokens leaked in plaintext to CI logs when their new hyphenated format failed Composer's legacy validation regex. Fixed in **Composer 2.9.8 / 2.2.28 / 1.10.28**. If you ran affected versions in CI, **rotate tokens immediately**. See `security.md` for the full mitigation playbook.
- **CVE-2026-40261** (Composer Perforce, April 15, 2026) — command injection in `Perforce::syncCodeBase()`. Exploitable even when not using Perforce (source-driver loaded on demand). Fixed in Composer 2.9.6 / 2.2.27.
- **CVE-2026-40176** (Composer Perforce, April 15, 2026) — command injection in `Perforce::generateP4Command()` via malicious root `composer.json` Perforce params. Fixed in Composer 2.9.6 / 2.2.27.
- **CVE-2026-54244** (Statamic, June 26, 2026) — Live Preview authorization bypass allowing read-only CP users to submit content updates. Patch pending at time of writing; watch the [Statamic releases feed](https://github.com/statamic/cms/releases). See `security.md`.

#### Composer version recommendations (updated cycle 7)

- **Composer 2.9.8+** (mainline) — required for CVE-2026-45793, CVE-2026-40261, CVE-2026-40176 fixes
- **Composer 2.2.28+** (LTS line) — required for same fixes on LTS track
- **Composer 1.10.28+** — final patch on the Composer 1.x line (consider migrating to 2.x; 1.x is EOL for new features)


### Ecosystem Update (2026-06-28, cycle 8) — HISTORY (Laravel 13.17.0 was latest at this point)

#### No new Laravel framework version

Laravel framework **v13.17.0** (June 23, 2026) remains the latest stable as of 2026-06-28 18:00 UTC. v13.18 has not been tagged yet. Track [github.com/laravel/framework/releases](https://github.com/laravel/framework/releases) for the next release.

#### Skill maintenance pass — oldest topic files refreshed

No new CVEs found this cycle (the four cycle-7 CVEs — Composer 45793/40261/40176 + Statamic 54244 — remain the most pressing). Instead, refreshed four topic files that hadn't been touched since May 25 (34+ days), filling documented-but-skipped feature gaps:

- **`observers.md`** (cycle 5 → cycle 8) — added `#[Boot]` and `#[Initialize]` model lifecycle attribute section. These PHP attributes have shipped since Laravel 11 but were missing from the skill. Lets you hang lifecycle logic off individual model methods without overriding `boot()` or wrapping in an observer. Also added `#[Scope]` for inline local-scope methods.
- **`validation.md`** (cycle 5 → cycle 8) — added `#[FailOnUnknownFields]` attribute for Form Requests (Laravel 13+, defense-in-depth against mass-assignment probing / unknown-field leakage). Added `Password::toPasswordRulesString()` (Laravel 13.9+) for the JS hint string consumed by browser password managers via the `passwordrules` HTML attribute.
- **`logging.md`** (cycle 5 → cycle 8) — added full `Context` facade section. Replaces the older `Log::shareContext()` pattern with the documented public API: `Context::add()`, `addIf()`, `push()`, `all()`, `forget()`, `flush()`, plus `addHidden()` for context that survives the queue boundary but is excluded from log output (PII, tokens).
- **`auth.md`** (cycle 5 → cycle 8) — added Laravel 13 starter-kit + Fortify integration section showing how to enable passkeys in a single feature flag (`Features::passkeyAuthentication()` / `Features::passkeyRegistration()`) without writing WebAuthn boilerplate. Includes passkey recovery guidance (always keep email + password fallback).

SKILL.md bumped to **v1.19.1** (cycle-8 maintenance, no new framework version).

### Ecosystem Update (2026-06-30, cycle 15)

#### No new Laravel framework version (at cycle start)

At the start of the cycle-15 window (~18:00 UTC 2026-06-30), Laravel framework **v13.17.0** (June 23, 2026) was still the latest stable. **However, v13.18.0 was tagged at 2026-06-30 12:55 UTC** — five hours before this cycle ran. The cycle-15 entry was written before that tag was noticed; the v13.18.0 work is now incorporated in the **"New in Laravel 13.18"** section above (and superseded by cycle 16 which does the full version-refresh).

#### No new framework CVEs

CISA SB26-180 (week of June 22, 2026) and the Laravel-specific OpenCVE feed were both rechecked at 18:00 UTC. No new framework-level CVEs in the last 6 hours. The Filament MFA recovery-code reuse (SB26-180) and CVE-2026-39976 Laravel Passport (already documented in `security.md`) remain the highest-priority Laravel-ecosystem items.

#### Skill maintenance pass — oldest untouched files refreshed

No new framework version at cycle-time, no new CVEs. Targeted the two oldest topic files (both 3+ days stale) with feature gaps that AI models keep getting wrong:

- **`migrations.md`** (cycle 5 → cycle 15) — added `storedAs()` / `virtualAs()` generated columns (MySQL/MariaDB support both STORED and VIRTUAL; PostgreSQL/SQLite STORED only), PostgreSQL `generatedAs()` identity columns (SQL-standard `BIGINT GENERATED ALWAYS AS IDENTITY`, the modern replacement for `SERIAL`), `->comment('text')` column modifier, and the full index type matrix (`fullText()` with optional `->language('english')` on PG, `spatialIndex()` on MySQL/MariaDB). Critical because AI models frequently misuse `generatedAs()` for string concatenation (it only works on `smallint`/`integer`/`bigint` sequences) — clarified the gotcha.
- **`file-uploads.md`** (cycle 5 → cycle 15) — fixed the `Storage::temporaryUploadUrl()` example to use the Laravel 13.x array-destructure return (`['url' => $url, 'headers' => $headers]`) and document the mandatory header-forwarding; added a File Visibility section covering the `public` / `private` model, the `setVisibility()` metadata-vs-re-upload gotcha, and R2 specifics; added an anti-extension-spoofing section covering the `evil.php.jpg` attack and the layered defense (never trust original filename, validate with `finfo` on actual file bytes, pair Laravel's `mimes:` with `file|image`, disable PHP execution in upload dirs, sanitize SVGs, strip EXIF).

SKILL.md bumped to **v1.22.3** (cycle-15 maintenance, no new framework version). 15 cycles in last 3 days.

### Ecosystem Update (2026-07-01, cycle 16)

#### Laravel 13.18.0 released 2026-06-30 12:55 UTC — incorporated in this cycle

v13.18.0 was tagged ~11 hours before this cycle ran (June 30, 12:55 UTC vs July 1, 00:03 UTC). Cycle 15 missed it because it ran six hours earlier. Cycle 16 fully incorporates it:

- **`versions.md`** — added "New in Laravel 13.18 (June 30, 2026, v13.18.0)" section between 13.17 and the "Late-June Bug Fixes" history note (which was reframed from "pending fixes" → "shipped fixes — history trail"); updated the active versions line at the top; replaced the cycle-15 stale "v13.18.0 has not been tagged yet" claim with a corrected entry.
- **`api.md`** — updated `Number::forHumans` / `Number::abbreviate` / `Number::fileSize` DoS section to reference **Laravel 13.18.0+** (was incorrectly labelled `13.17.1+` — there was no v13.17.1, those fixes shipped in 13.18.0); same for `Request::json()` zero-body section.
- **`controllers.md`** — HEAD cache-headers section now references **Laravel 13.18.0+** (was `13.17.1+`).
- **`performance.md`** — Number OOM section now references **Laravel 13.18.0+** (was `13.17.1+`).
- **`migrations.md`, `file-uploads.md`** — removed the stale "v13.17.1 / v13.18.0 still pending" note (now resolved).
- **`SKILL.md`** — bumped v1.22.3 → v1.22.4; updated three nav-table entries and two critical-rules bullets to reference **13.18.0+** instead of `13.17.1+`.
- **`README.md`** — refreshed version support table + "Last research cycle" block.

#### Why `13.17.1+` was wrong

When cycle-15 ran (18:00 UTC 2026-06-30), those fixes were tracked as "post-v13.17.0, will ship in v13.17.1+". v13.18.0 was tagged **at 12:55 UTC the same day** — five hours before cycle 15 ran — but the cycle's web search didn't pick up the new release tag (the GitHub API for `releases/latest` was still cached as 13.17.0 at the time). All `13.17.1+` references in `api.md` / `performance.md` / `controllers.md` / `SKILL.md` are corrected to `13.18.0+` in this cycle.

#### No new framework CVEs

CISA KEV catalog and the Laravel OpenCVE feed rechecked at 00:03 UTC 2026-07-01. No new framework-level CVEs in the last 6 hours. The Filament MFA recovery-code reuse (SB26-180) and CVE-2026-39976 Laravel Passport (already in `security.md`) remain the highest-priority Laravel-ecosystem items.

SKILL.md bumped to **v1.22.4** (cycle-16 release-notes incorporation for Laravel 13.18.0). 16 cycles in last 3 days.

### Ecosystem Update (2026-07-01, cycle 17)

#### No new framework version, no new CVEs at cycle time

Latest Laravel framework release remains **v13.18.0** (June 30, 2026). GitHub `releases/latest` endpoint + Releasebot.io + Laravel News all confirm 13.18.0 is still the head of the 13.x line — no 13.18.1, 13.19.0, or 13.17.x patch has been tagged in the ~18 hours since the previous cycle. `laravel/laravel` skeleton at v13.8.0 (May 27), `laravel/passport` at v13.7.5 (Apr 21), `laravel/reverb` at v1.10.2 (May 13) — all unchanged.

GitHub Security Advisories API for `laravel/framework` rechecked at 18:17 UTC 2026-07-01 — no new advisories since the previous one (`GHSA-crmm-hgp2-wgrp` / CVE-2026-48041, June 8, 2026), which is already documented in `security.md`. CVE-2026-39976 (Passport, fixed 13.7.1), CVE-2026-23524 (Reverb RCE, fixed 1.7.0), CVE-2026-48019 (CRLF email injection, fixed 13.10.0/12.60.0), and CVE-2025-54068 (Livewire RCE, KEV-listed) — all already in `security.md`.

#### Targeted feature-gap fix: `observers.md` + `ShouldBeDiscovered` opt-out

Targeted the oldest untouched file from the previous batch (`observers.md`, last touched 2026-06-29). The Laravel 13.12 release added an opt-out mechanism for **automatic event discovery** — when `Event::shouldDiscoverEvents()` is on in `bootstrap/app.php`, every `ShouldQueue` listener gets auto-registered. AI models keep writing listeners that get unintended side effects from discovery (e.g., a `ShouldQueue` "send alert" listener gets queued on every `OrderShipped` event in the app, not just the one specific dispatch site the author intended). The opt-out contract (static `$shouldDiscover = false` property OR marker interface under `Illuminate\Contracts\Events\`) was previously missing from the skill.

- **`observers.md`** — added new `## ShouldBeDiscovered Opt-Out (Laravel 13.12+)` section between the `#[Boot]`/`#[Initialize]` block and the existing "Updated from Research" entry. Includes the `public static bool $shouldDiscover = false` opt-out pattern, the contract-name caveat (the exact marker-interface name varies between 13.12 patch levels — verify against `vendor/laravel/framework/src/Illuminate/Contracts/Events/` before relying on it), and explicit when-to-use / when-NOT-to-use guidance. Added a corresponding bullet to the `#[Boot]` / `#[Initialize]` summary section.

SKILL.md bumped to **v1.22.5** (cycle-17 gap-fill for `ShouldBeDiscovered` opt-out in `observers.md`). 17 cycles in last 3 days.

### Ecosystem Update (2026-07-02, cycle 18)

#### No new Laravel framework version

Laravel framework **v13.18.0** (June 30, 2026) remains the latest stable as of 2026-07-02 00:14 UTC. GitHub `releases/latest` endpoint, Releasebot.io, and Laravel News all confirm 13.18.0 is still the head of the 13.x line — no 13.18.1, 13.19.0, or 13.17.x patch has been tagged in the ~30 hours since the previous cycle. `laravel/laravel` skeleton at v13.8.0 (May 27), `laravel/passport` at v13.7.5 (Apr 21), `laravel/reverb` at v1.10.2 (May 13) — all unchanged.

GitHub Security Advisories API for `laravel/framework` rechecked at 00:10 UTC 2026-07-02 — no new advisories since the previous one (`GHSA-crmm-hgp2-wgrp` / CVE-2026-48041, June 8, 2026), which is already documented in `security.md`.

#### PHP runtime security releases — 2026-07-01 batch (CRITICAL for Laravel hosts)

The PHP project released **four simultaneous security patches on 2026-07-01** that every Laravel app needs to apply (Laravel 13 requires PHP 8.3+, so 8.3.32 / 8.4.23 / 8.5.8 are in-scope for all current Laravel 13 deployments; 8.2.32 is relevant for projects still on Laravel 12). This is the first multi-version PHP security release since 2026-04 and the first one in the 8.5.x line.

| PHP branch | Release | Severity | Key fixes |
|---|---|---|---|
| 8.2 (Laravel 12) | **8.2.32** | MED-HIGH | Phar directory protection bypass, Opcache fix |
| 8.3 (Laravel 13 min) | **8.3.32** | MED-HIGH | Phar directory protection bypass, Opcache fix |
| 8.4 (Laravel 13 recommended) | **8.4.23** | **CRITICAL** | **openssl_encrypt AES-WRAP-PAD heap corruption** + Phar bypass + Opcache fix |
| 8.5 (Laravel 13 supported) | **8.5.8** | MED-HIGH | Phar directory protection bypass, Opcache fix |

**Key vulnerabilities (full details in `security.md`):**

- **PHP 8.4.23 — `openssl_encrypt` AES-WRAP-PAD heap corruption (CRITICAL):** The OpenSSL extension's `openssl_encrypt()` crashes with a heap corruption fault when called with `OPENSSL_CIPHER_AES_256_WRAP_PAD` (or related AES-WRAP variants) and attacker-controlled input. This is reachable from Laravel code that uses `Crypt::encryptString()` (which uses `encrypter.php` defaults), any custom encryption built on `openssl_encrypt`, or any third-party package that wraps OpenSSL (e.g., custom JWT signing, S3 SSE-C client-side encryption). A remote attacker can trigger the crash with a small POST body — this is a remote-DoS on every Laravel 13 host running PHP 8.4 with the default encryption path exposed to user input. Patch: upgrade to **PHP 8.4.23+** (already in apt for Ubuntu 24.04 / Debian 13 by end of week).
- **PHP 8.5.8 / 8.4.23 / 8.3.32 / 8.2.32 — Phar directory protection bypass:** The phar wrapper historically blocked `phar://` URLs with `..` segments to prevent directory traversal. The bypass lets an attacker who can influence a `phar://` URL used in a `file_get_contents()`, `include`, or `fopen()` call (e.g., via an upload-metadata field or a config value) read or include files outside the phar archive. Laravel apps that accept phar uploads, use `League\CommonMark` with a `phar://` source, or run PHAR-based plugins (e.g., `hirak/prestissimo` historically) are at risk. Patch: upgrade PHP to the in-scope 8.x.32 / 8.x.23 / 8.5.8 build.
- **PHP 8.5.8 / 8.4.23 — Opcache bypass fix:** The Opcache bypass allowed cached preloaded scripts to retain a stale validation status across a worker respawn. Not directly exploitable, but degrades the security boundary between PHP code and the cached opcodes. Patch: included in the same release line.
- **CVE-2026-7263 — `DOMNode::C14N()` infinite-loop DoS** (disclosed 2026-05-10, **NOT in skill until now**): PHP versions 8.4.* before 8.4.21 and 8.5.* before 8.5.6 had a flaw in `DOMNode::C14N()` (the XML Canonicalization method) that creates a circular linked list and enters an infinite loop on attacker-controlled XML input. CWE-404 + CWE-835. Laravel apps that accept untrusted XML input and run it through `DOMDocument::C14N()` (rare, but happens in SAML/SSO integrations, e.g., `aacotroneo/laravel-saml2`, custom XML signature verification, eSOA gateways) are vulnerable to a remote-DoS that pegs a PHP-FPM worker until `request_terminate_timeout` kills it. Patch: PHP 8.4.21+ or 8.5.6+ — but **8.4.23 / 8.5.8 from 2026-07-01 also include this fix** (and all subsequent patches), so the upgrade is automatic if you apply the 2026-07-01 batch.

**Why this matters for Laravel apps:** The 2026-07-01 PHP release batch is the most consequential PHP-level security event for Laravel 13 deployments since the 2026-04 AES-WRAP issue. The openssl_encrypt CRITICAL is a remote-DoS reachable from any unencrypted form input that flows into `Crypt::encryptString()`. Apply the PHP patch (or use the OS package manager — `apt install --only-upgrade php8.4` / `dnf upgrade php`) within the next 48 hours. If your app uses Cloudflare or another CDN in front of nginx, the CDN does not protect you — the openssl_encrypt crash happens *inside* the FPM worker, after the request body is already proxied through.

**Recommended action:** Add `apt install --only-upgrade php8.4-fpm php8.4-cli php8.4-common` (or your distro's equivalent) to your standard patching runbook, and add a `php -v` check to your monitoring that alerts when the running PHP version is older than 30 days. The Laravel skill can't enforce a PHP version — that's an OS-level concern — but every Laravel deployment pipeline should verify the PHP patch level is current before allowing a deploy to proceed.

#### CVE-2026-31431 "Copy Fail" — Linux kernel LPE (host-level, every Laravel host)

Disclosed 2026-04-29 by Xint Code / Theori. CVSS **7.8 HIGH**. Local privilege escalation in the kernel's `authencesn` / `AF_ALG` / `splice()` path. Any unprivileged local user can become root with a **732-byte Python exploit** that works unmodified across all major Linux distributions built since 2017. While this is a host-level (kernel) CVE rather than a Laravel CVE, it affects **every PHP/Laravel deployment** on Linux — the exploit is one `python3 copyfail.py` away from a full container/host takeover, and on a Laravel app host where `php-fpm` runs as `www-data`, a single compromised local user (a CI runner account, a leaked deploy key, a low-privileged staging user) can pivot to root. **Fix:** upgrade the kernel (AlmaLinux 8/9/10, Ubuntu 24.04, CloudLinux 8/9/10 — all have patched kernels as of early May 2026) and **reboot**. Temporary mitigation if you can't reboot: blacklist the `algif_aead` module via `initcall_blacklist=algif_aead_init` in GRUB. Full details and patch-version table in `security.md`.

#### Summary of new ecosystem items added this cycle

1. **PHP 8.5.8 / 8.4.23 / 8.3.32 / 8.2.32** security release (2026-07-01) — openssl_encrypt AES-WRAP heap corruption (CRITICAL, PHP 8.4.23), Phar directory bypass (HIGH, all four branches), Opcache fix (MED, PHP 8.5.8 / 8.4.23). Affects every PHP/Laravel host.
2. **CVE-2026-31431 "Copy Fail"** Linux kernel LPE (2026-04-30, CVSS 7.8) — host-level, affects every Linux-running Laravel app. Kernels since 2017 are vulnerable. Patched in AlmaLinux 8 (4.18.0-553.121.1+), AlmaLinux 9 (5.14.0-611.49.2+), AlmaLinux 10 (6.12.0-124.52.2+), Ubuntu 24.04 (HWE), and equivalents.
3. **CVE-2026-7263 `DOMNode::C14N()` infinite-loop DoS** (2026-05-10) — affects PHP 8.4 < 8.4.21 and 8.5 < 8.5.6. Patched in 8.4.21+ / 8.5.6+; **8.4.23 / 8.5.8 (2026-07-01) include the fix and are the new minimums**.

SKILL.md bumped to **v1.22.6** (cycle-18 host-runtime security update — PHP 2026-07-01 batch + Copy Fail + DOMNode::C14N()). 18 cycles in last 3 days.

### Ecosystem Update (2026-07-02, cycle 19)

#### No new Laravel framework version

Laravel framework **v13.18.0** (June 30, 2026) remains the latest stable as of 2026-07-02 12:00 UTC — ~36 hours since the previous cycle. GitHub `releases/latest` endpoint, Releasebot.io, and Laravel News all confirm 13.18.0 is still the head of the 13.x line. `laravel/laravel` skeleton at v13.8.0, `laravel/passport` at v13.7.5, `laravel/reverb` at v1.10.2 — all unchanged.

GitHub Security Advisories API for `laravel/framework` rechecked at 12:00 UTC — no new framework-level advisories since `GHSA-crmm-hgp2-wgrp` (CVE-2026-48041, June 8, 2026), already documented in `security.md`.

Filament CVE-2026-55409 ("Disabled RichEditor field state can be used for XSS", June 17, 2026) is **NOT yet in `security.md`** but is a Filament-specific issue, not a Laravel framework CVE. It only affects apps using the Filament RichEditor with disabled state — defer to a future Filament-cluster cycle unless we get reports of exploitation.

#### Targeted feature-gap fix: `blade.md` (oldest untouched file at cycle start)

At cycle start, `blade.md` had not been touched since 2026-06-28 (cycle 6) — **4 days stale**, the oldest untouched file in the skill. The most-common AI-model miss in Blade work is the **form-helper directives** (`@checked` / `@selected` / `@disabled` / `@readonly` / `@required`) — added in Laravel 9.0 and stable for 4 years, but AI models still write the verbose `{{ old(...) == $x ? 'checked' : '' }}` pattern by default. Also missing: `@verbatim` (for Vue/Alpine templates inline) and `Blade::render()` for AI-generated / CMS-user templates — both critical for Laravel 13's AI SDK output rendering and modern landing-page builders.

- **`blade.md`** — added four new sections:
  1. **Form-Helper Directives (`@checked` / `@selected` / `@disabled` / `@readonly` / `@required`)** — full examples for each directive, plus the XSS-safety angle (the directives emit a static attribute when truthy and nothing when falsy, so they're safer than hand-rolled concatenations that some authors mistakenly pass `$userInput` into). Includes the accessibility note (`@required` only adds the HTML5 attribute; pair with `required` in `validate()`).
  2. **`@verbatim` Directive** — for Vue / Alpine / vanilla JS templates where `{{ }}` and `@click` would otherwise be parsed by Blade. Documented the non-nestable gotcha.
  3. **`Blade::render()` — Inline String Rendering** — critical for AI-generated templates, CMS user templates, DB-stored email templates. Documented the `deleteCachedView: true` cache-pollution guard AND the **RCE / XSS warning** — `Blade::render()` does NOT sandbox user-supplied templates; a malicious template can execute arbitrary PHP via `@php` directives, call arbitrary static methods via `{{ \\App\\Services\\X::dangerous() }}`, or include arbitrary components via `<x-*>`. The Laravel AI SDK deliberately does NOT use `Blade::render()` for untrusted output for exactly this reason. Recommendation: whitelist allowed directives or use a separate templating engine (Twig / Handlebars / Mustache) for user-facing templates.
  4. **Inline Component Views via `<<<'blade'` Heredoc** — class-based components can return the Blade template directly from `render()` instead of pointing at a separate `.blade.php` file. Best for small components; >20 lines should still use external view files for cache + IDE-highlight reasons. Documented the nowdoc (single-quote identifier) syntax.

- **`SKILL.md`** — bumped v1.22.6 → v1.22.7; added four new nav-table entries for the new Blade sections.

SKILL.md bumped to **v1.22.7** (cycle-19 Blade gap-fill — form-helper directives + `@verbatim` + `Blade::render()` + inline component views). 19 cycles in last 3 days.

---

## Auto-Updater Cycle Notes

### Cycle 23 (2026-07-04 06:00 UTC) — Cross-Reference Completion for 13.18.1

Cycle 22 (00:05 UTC) added the full Laravel 13.18.1 release notes to this file but only
updated cross-references (e.g. "Covered in `queues.md`") — the actual content in the
topic files was never written. Cycle 23 fills the gap by adding the missing content to:

- `queues.md` — `Release` middleware (PR #60630) + `Queue::fake()` delayed-job inspection (PR #60636)
- `artisan.md` — `input()` method on console commands (PR #60607)
- `testing.md` — `assertDatabaseEmpty()` accepts iterables (PR #60621) + `assertPushed()->delay()` for queue fake (PR #60636)
- `logging.md` — channel name respected in `on-demand` log stacks (PR #60635)
- `validation.md` — `Str::mask()` UTF-8 boundary fix (PR #60646)
- `deployment.md` — `php artisan down` intercepts `/api/*` and JSON routes (PR #60595)
- `controllers.md` — maintenance-mode JSON response for API routes (PR #60595)
- `migrations.md` — `foreignUuid()` / `foreignUlid()` Blueprint return types (PR #60643)

Items deliberately not added to topic files (too narrow to justify a section):
- Predis scalar retry config (PR #60642) — niche, Predis-only, no API-surface change


---

### Cycle 24 (2026-07-04 12:00 UTC) — auth.md Gap Fill (Laravel 13 PHP Attribute Authorization)

Cycle 23 (06:00 UTC) finished cross-reference completion for the 13.18.1 release notes. No new Laravel framework release since v13.18.1 (tagged 2026-07-02 18:36 UTC) — GitHub `releases/latest` still resolves to 13.18.1 at cycle time. GitHub Security Advisories API for `laravel/framework` rechecked at 12:00 UTC — no new advisories since `GHSA-crmm-hgp2-wgrp` (CVE-2026-48041, June 8, 2026), already documented in `security.md`. No new PHP batch, no new ecosystem CVEs.

`auth.md` was the **oldest untouched topic file** at cycle start (last modified 2026-06-28 — **6 days stale**), and was missing two of Laravel 13's headline authorization features:

1. **`#[UsePolicy]` PHP attribute on Eloquent models** (`Illuminate\Database\Eloquent\Attributes\UsePolicy`) — declarative model-side policy registration that replaces `Gate::policy(Model::class, PolicyClass::class)` in `AppServiceProvider`. Reviewed-by-reflection auto-registration. AI models writing fresh Laravel 13 code default to the verbose `AppServiceProvider` boot pattern instead of the attribute, because the attribute shipped in Laravel 13 and most training data is pre-13.
2. **`#[Authorize]` controller attribute** (`Illuminate\Routing\Attributes\Controllers\Authorize`) — declarative policy check on controller methods. Signature: `#[Authorize('ability', $target)]` where `$target` is a route parameter name, a model class FQCN, or `[Class::class, 'routeParam']`. Eliminates the `$this->authorize()` first-line-of-method pattern that's easy to forget on copy-paste. The attribute was already documented in `controllers.md` (Laravel 13 Controller Attributes section, line 84+) — `auth.md` just didn't cross-link to it.

- **`auth.md`** — added two new sections between "Policies" and "Gates":
  1. **Laravel 13 Policy Registration — `#[UsePolicy]` Attribute** — full example, three-reason preference rationale (colocation, no boot-time race in Octane/FrankenPHP, no service-provider bloat), explicit non-changes (policy method signatures and call sites identical), and three edge cases (namespace confusion with the controller-side `#[Authorize]`, manual `Gate::policy()` override rule, Laravel 12 backward-compat note).
  2. **Laravel 13 Controller Authorization Attributes — `#[Authorize]` + `#[Middleware]`** — full example, three-row table covering the three `#[Authorize]` second-argument forms (route param name, model class FQCN, `[Class::class, 'routeParam']` tuple), three reasons to prefer attribute over `$this->authorize()` (audit visibility, hard-fail on missing policy at registration time, copy-paste safety), explicit non-replacements (route-level middleware, Blade `can()` checks, ad-hoc `Gate::define()`), and cross-link to `controllers.md`.
  - Updated the "Updated from Research" section at the bottom with a `### Laravel 13 PHP Attribute Authorization (cycle 24, 2026-07-04)` entry.

- **`SKILL.md`** — bumped v1.22.10 → v1.22.11 (cycle 24).
- **`README.md`** — bumped version stamp + research-cycle marker to reflect cycle 24.
- **`versions.md`** — no change to the "New in Laravel 13.18.1" section (no new release); added the cycle 24 auto-updater note above.

SKILL.md bumped to **v1.22.11** (cycle-24 auth.md gap fill — `#[UsePolicy]` model attribute + `#[Authorize]` controller attribute + cross-link to `controllers.md`). 24 cycles in 4 days.

### Cycle 25 (2026-07-04 18:00 UTC) — CVE-2026-12184 (PHP TLS Setup-Failure Remote DoS) Addition to `security.md`

Cycle 24 (12:00 UTC) finished the auth.md gap-fill for `#[UsePolicy]` and `#[Authorize]`. No new Laravel framework release since v13.18.1 (tagged 2026-07-02 18:36 UTC) — GitHub `releases/latest` still resolves to 13.18.1 at cycle time. GitHub Security Advisories API for `laravel/framework` rechecked at 18:00 UTC — no new advisories since `GHSA-crmm-hgp2-wgrp` (CVE-2026-48041, June 8, 2026), already documented in `security.md`. No new framework releases, no new Laravel CVEs.

**New finding:** The `php/php-src` Security Advisories API revealed a **NEW PHP CVE published 2026-07-02 17:01:28 UTC — `GHSA-mhmq-mmqj-2v39` / CVE-2026-12184** ("Failure to setup TLS with a remote server can result in a remote DoS", severity HIGH). The advisory was published ~28 hours *after* the 2026-07-01 PHP release batch (8.2.32 / 8.3.32 / 8.4.23 / 8.5.8) shipped — and the upstream fix (PR #21031) was already merged into those binaries. Anyone who applied the cycle 18 patch is automatically protected; the cycle 25 value is CVE database completeness for the "wait-for-CVE-before-patching" operators who are exposed during the 1–3 day disclosure-lag window.

**What the CVE is:** A NULL-pointer dereference in `php_stream_url_wrap_http_ex` (PHP's internal HTTP/HTTPS stream wrapper). When TLS crypto setup fails — e.g., the remote server has an expired certificate, a hostname mismatch, or an aborted TLS handshake — the stream is closed and reset to `NULL`, but the subsequent peer-name cleanup block unconditionally tries to reset the peer name on the now-NULL stream. The result is a SIGSEGV in the PHP process. Triggers via a reproducer ([php/php-src#21468](https://github.com/php/php-src/issues/21468)) against **any** HTTPS endpoint with an expired certificate — no special protocol, no malicious server. The reproducer crashes the entire PHP-FPM pool because every request-handling worker that tries to call the bad endpoint dies.

**Why this matters for Laravel apps specifically:** Outbound HTTPS is everywhere in modern Laravel code paths:

- `Http::get()` / `Http::post()` / `Http::pool()` (the Laravel HTTP client wraps Guzzle, which uses PHP's stream handler by default — `GuzzleHttp\Psr7\Http\Stream` delegates to `php_stream_*` for HTTPS)
- `Mail::send()` with SMTP+STARTTLS (`Illuminate\Mail\Transport\SmtpTransport`)
- `Storage::disk('s3')->put()` and any S3 / R2 / GCS adapter (`league/flysystem-aws-s3-v3` → Guzzle → PHP stream)
- `Notification::route('slack', ...)` (Slack webhooks), `Notification::route('mailgun', ...)`, Twilio, SendGrid, Stripe SDKs (all use Guzzle → PHP stream)
- `OAuth` token refresh calls (`laravel/socialite` → Guzzle → PHP stream)
- Health-check pings, webhook deliveries, scheduled task runners
- Even `file_get_contents('https://...')` in any third-party package

The realistic production scenario is a payment gateway, OAuth provider, or webhook receiver whose certificate expires overnight. Every Laravel app that integrates with that service starts crashing on the next outbound call. The crash signature is a worker-level SIGSEGV, but with `pm.max_children` = 50 and 1k req/s incoming, the entire pool can be exhausted in seconds.

**Affected versions and patch levels:**

| PHP branch | Affected (pre-patch) | Patched in (already shipped) |
|---|---|---|
| 8.2 (Laravel 12) | < 8.2.32 | 8.2.32 (released 2026-07-01) |
| 8.3 (Laravel 13 minimum) | < 8.3.32 | 8.3.32 (released 2026-07-01) |
| 8.4 (Laravel 13 recommended) | < 8.4.21 | 8.4.23 (released 2026-07-01) — note the patch threshold is 8.4.21, so 8.4.21+ are all safe, 8.4.23 is the current latest |
| 8.5 (Laravel 13 supported) | < 8.5.6 | 8.5.8 (released 2026-07-01) — same logic, 8.5.6+ are all safe |

**Key insight for the Laravel skill — the "fix-before-CVE" pattern:** PHP's release process ships security fixes in binary releases, but publishes the public advisory 1–3 days *after* the binary release. This means:

- The **2026-07-01 batch was effectively a "fix-before-CVE" event** for CVE-2026-12184. Anyone who upgraded proactively on 2026-07-01 (cycle 18 of this skill) was safe; anyone who waited for the CVE to drop on 2026-07-02 17:01 UTC was exposed during the gap.
- **Agents and operators should treat PHP release announcements (every 4–6 weeks) as a hard maintenance window**, not a "wait-and-see-if-CVE-drops" event. The cycle 18 emphasis on "apply the 2026-07-01 batch within 48 hours" is validated retroactively by this CVE.
- This is the **second PHP-runtime CVE in the skill** (after cycle 18's `openssl_encrypt` AES-WRAP heap corruption). The pattern is: every PHP release batch deserves the same urgency as a Laravel Security Advisory, because the patch-vs-CVE lag means the upgrade window IS the protection window.

**Skill files changed in cycle 25:**

- **`security.md`** — appended a new "Updated from Research (2026-07-04, cycle 25)" section (~92 lines) with the full CVE-2026-12184 entry:
  - Affected versions table with the 8.4.21 / 8.5.6 patch threshold (older than the 8.4.23 / 8.5.8 from cycle 18, which is why the 2026-07-01 batch already contains the fix).
  - Detailed "Why this matters for Laravel apps" callout covering the 5 realistic production scenarios (Http::, Mail, S3, OAuth, file_get_contents).
  - Process-level DoS explanation (worker SIGSEGV scaling to full pool exhaustion).
  - 5 mitigations: outbound HTTPS audit, circuit-breaker `Http::safeGet()` macro, FPM `pm.max_requests=500` + systemd `Restart=always`, SIGSEGV monitoring, egress filtering.
  - 5 cross-references to existing skill sections (circuit-breaker, FPM Restart policy, cert-expiry monitoring, PHP release batch as fix-before-CVE signal, CVSS-pending rule).
  - Updated top-priority actions list (CVE-2026-12184 now first, with "already fixed by cycle 18 batch" note for those who applied it).
- **`SKILL.md`** — bumped v1.22.11 → v1.22.12 (cycle 25).
- **`README.md`** — bumped version stamp + research-cycle marker to reflect cycle 25.
- **`versions.md`** — no change to the "New in Laravel 13.18.1" section (no new Laravel release); added this cycle 25 auto-updater note.

**No changes to other files in cycle 25** — the targeted topic files (`ai.md`, `localization.md`, `eloquent.md`, `performance.md`, `file-uploads.md`, `api.md`, `observers.md`, `blade.md`, `artisan.md`, `controllers.md`, `deployment.md`, `logging.md`, `migrations.md`, `queues.md`, `testing.md`, `validation.md`, `SKILL.md`, `README.md`) are all within the 5–7 day staleness window and are still well-served by their cycle 4–24 content. Cycle 26 (2026-07-05 00:00 UTC) will target the next-stale file.

SKILL.md bumped to **v1.22.12** (cycle-25 PHP-runtime CVE addition — CVE-2026-12184 / GHSA-mhmq-mmqj-2v39 TLS setup-failure remote DoS, severity HIGH, fix already shipped in 2026-07-01 PHP batch). 25 cycles in 4 days.
