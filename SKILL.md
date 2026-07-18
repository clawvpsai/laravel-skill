---
name: Laravel
slug: laravel-developer
version: 1.22.29
description: Production-grade Laravel development — ship robust apps without common pitfalls.
metadata:
  {"emoji":"🟠","requires":{"bins":["php","composer"]},"os":["linux","darwin","win32"]}
---

## Navigation

| Topic | File | When to Use |
|---|---|---|
| Models, relationships, query builder, migrations | `eloquent.md` + `migrations.md` | Any DB work |
| Routing, validation, request lifecycle | `controllers.md` | Building endpoints |
| Sanctum, policies, gates, CSRF | `auth.md` | User auth & permissions |
| Jobs, workers, retries, failed jobs | `queues.md` | Background processing |
| `incrementEachQuietly` / `decrementEachQuietly` Eloquent bulk helpers (Laravel 13.20+) | `eloquent.md` (Bulk Operations section) | Batch counter updates without firing per-model events |
| `QueueFake::beforePushing()` / `afterPushing()` lifecycle hooks (Laravel 13.20+) | `queues.md` (QueueFake Lifecycle Hooks section) | Inspect jobs entering/exiting the fake queue in tests |
| First-party `Illuminate\Image` facade — resize, crop, convert, store (Laravel 13.20+) | `file-uploads.md` (First-Party Image Processing section) | Replace third-party image packages for basic transforms |
| `#[WithoutMiddleware]` controller attribute (Laravel 13.20+) | `controllers.md` (Laravel 13 Controller Attributes section) | Exclude specific middleware from individual controller actions (PR #60709) |
| `Storage::assertEmpty()` (Laravel 13.20+) | `file-uploads.md` (Testing File Uploads section) | Assert a storage path has no files — inverse of `assertExists` |
| `Request::integer()` / `Request::string()` / `Request::boolean()` / `Request::float()` / `Request::array()` / `Request::date()` typed accessors (Laravel 12+) | `controllers.md` (Request Typed Accessors section) | Auto-cast + safe defaults — replace `?:` null-coalescing + `intval()` boilerplate |
| `Request::enum()` / `Request::enums()` typed enum accessors (Laravel 12+) | `controllers.md` (Request Typed Accessors section) | Read enum-typed input as a typed enum; pair with `Rule::enum()` for validation |
| `#[Scope]` Eloquent query scope attribute (Laravel 12+) | `eloquent.md` (Scopes section) | Replaces `scopeXxx()` method naming; IDE-discoverable, refactor-rename safe |
| Templates, components, XSS, slots | `blade.md` | Blade/frontend work |
| CLI commands, scheduling, tinker | `artisan.md` | DevOps & maintenance |
| REST APIs, JSON, rate limiting | `api.md` | API development |
| XSS, SQL injection, hardening | `security.md` | Security-critical code |
| Caching, indexing, eager loading | `performance.md` | Optimization |
| AI agents, tools, streaming, embeddings | `ai.md` | AI/LLM integration |
| PHPUnit, factories, feature tests | `testing.md` | Test-driven dev |
| Production deployment, env, configs | `deployment.md` | Going live |
| File uploads, S3, presigned URLs | `file-uploads.md` | Media handling |
| Model lifecycle, observers, events | `observers.md` | Decoupled logic |
| DB schema, seeding, index management | `migrations.md` | Schema management |
| Log channels, structured logging, monitoring | `logging.md` | Debugging & monitoring |
| Composer self-update, CVE-2026-45793 / 40176 / 40261, supply chain | `security.md` (Composer section) + `deployment.md` step 0 | Updating Composer in deploy pipeline |
| Translations, locales, pluralization | `localization.md` | i18n/l10n |
| Rules, Form Requests, custom validators | `validation.md` | Input validation |
| Prohibited rule family + `Rule::when()` / `Rule::unless()` | `validation.md` (Prohibited Family + Rule::when sections) | Either-or fields, conditional rule chains, "missing/empty" enforcement |
| `#[Boot]` / `#[Initialize]` model lifecycle attributes | `observers.md` (model attribute hooks section) | Per-instance defaults + class-level event listeners without `boot()` override |
| `JsonSchema::anyOf()` discriminated unions (Laravel 13.17+, PR #60509) | `ai.md` (Advanced JSON Schema section) | Single `enum` collapses type-specific fields per variant; `anyOf()` keeps them so the LLM returns the full matching shape and your handler branches on the discriminator |
| `JsonSchema::union(['string', 'number'])` multi-type unions (Laravel 13.17+, PR #60455) | `ai.md` (Advanced JSON Schema section) | Third-party MCP tool schemas routinely use `type: ['string', 'number', 'boolean']`; pre-13.17 `JsonSchema::fromArray()` threw on these — now round-trips cleanly. 12.x still throws; 13.17+ backport only |
| `#[FailOnUnknownFields]` Form Request attribute | `validation.md` (Failing on Unknown Fields section) | Reject unknown fields at validation time (defense in depth) |
| `Context` facade for log context | `logging.md` (Context facade section) | Request/job-scoped context that auto-attaches to logs and survives queue dispatch |
| `Password::toPasswordRulesString()` | `validation.md` (Password rules section) | Derive JS `passwordrules` HTML hint from server-side `Password::` rule |
| `Features::passkeyAuthentication()` / `passkeyRegistration()` | `auth.md` (Starter Kits section) | Laravel 13 starter-kit + Fortify passkey integration |
| `#[UsePolicy]` model attribute (Laravel 13) | `auth.md` (Policy Registration section) | Colocate policy with model — replace `Gate::policy()` boot code |
| `#[Authorize]` controller attribute (Laravel 13) | `auth.md` (Controller Authorization Attributes section) + `controllers.md` | Declarative policy check on controller methods, route-param resolution built-in |
| `assertInvalid()` / `assertValid()` / `assertOnlyInvalid()` | `testing.md` (Validation Assertions section) | Laravel 11+ generic validation assertions for JSON + session flows |
| `Exceptions` facade (`assertReported`, `assertNothingReported`, `throwFirstReported`) | `testing.md` (Exception Assertions section) | Laravel 11.5+ clean exception assertions without try/catch |
| Pest 3 architecture testing (`->arch()->preset()->laravel()`, `->toBeFinal()`, `->toHaveMethodsDocumented()`) | `testing.md` (Pest 3 section) | Enforce Laravel conventions + design rules in CI |
| Pest 3 mutation testing (`pest --mutate`) | `testing.md` (Pest 3 section) | Verify tests actually catch injected bugs |
| Laravel AI SDK vs Laravel MCP vs Laravel Boost | `ai.md` (matrix section) | Pick the right product: AI SDK = app calls AI; MCP = expose app as MCP server; Boost = dev-time AI assistant context |
| Single action controllers (`__invoke`) | `controllers.md` (Single Action Controllers section) | Webhook handlers, OIDC callbacks, billing redirects — actions that don't fit the 7-method resource model |
| `Route::pattern()` global constraints | `controllers.md` (Route Patterns section) | Force `{id}` to match `\d+` (or any regex) app-wide; watch 404 vs `ModelNotFoundException` in tests |
| HEAD request cache headers (PR #60589) | `controllers.md` (API Versioning section) | 13.18.0+ finally sets `Cache-Control` / `ETag` on HEAD; CDN cache-warming scripts broke before |
| `WorkerStopping` `jobsProcessed` + `lastJobProcessedAt` (PR #60592) | `queues.md` (WorkerStopping section) | Lifetime-job-count and graceful-shutdown-gap dashboards without dirty reflection hacks |
| `php artisan dev` priority-based registration (PR #60580) | `artisan.md` (Dev Orchestration section) | 13.18.0+ lets vendor packages register a deterministic order so critical dev processes (log tail, queue worker) come first |
| `JsonResource` Advanced Patterns (`when`/`whenLoaded`/`whenCounted`/`whenPivotLoaded`, `additional`, `wrap`/`withoutWrapping`, `withResponse` for HATEOAS) | `api.md` (API Resource Advanced Patterns section) | Conditional attributes, response-level metadata, data wrapper control, and HATEOAS self/related links — covers ~90% of production API resource patterns that AI models default to verbose `response()->json([...])` for |
| `JsonApiResource::toLinks()` / `toMeta()` | `api.md` (JsonApiResource toLinks section) | JSON:API spec compliance for `self`/`related`/pagination/domain links and collection metadata (totals, generated_at, request_id) |
| `php artisan dev --kill-others-on-fail` (PR #60606) | `artisan.md` (Dev Orchestration section) | 13.18.0+ tears down sibling dev processes on non-zero exit; use in CI / one-shot dev, leave off for normal local dev |
| TaggedCache `flexible()` lock/defer namespace fix (PR #60626) | `performance.md` (TaggedCache section) | 13.18.0+ namespaces `flex_lock:` and `flex_defer:` keys separately so a custom lockName can't collide with a defer label |
| Debounced-jobs cache-hit reduction (PR #60575) | `performance.md` (Cache debounce subsection) | 13.18.0+ skips lock acquisition when already inside the debounce window — pairs with the 13.17.0 `maxWait` fix |
| `Release` queue middleware (`Illuminate\Queue\Middleware\Release`) (PR #60630) | `queues.md` (Job Middleware section) | 13.18.1+ declarative `->middleware(new Release($delay))` so jobs release themselves without `$this->release()` calls scattered through `handle()` |
| `$this->input()` on console commands (PR #60607) | `artisan.md` (Defining Commands section) | 13.18.1+ typed `$this->input('email')` / `$this->input(['email', 'name'])` — cleaner than `$this->argument()` + `$this->option()` + manual casting |
| `assertDatabaseEmpty()` accepts iterables (PR #60621) | `testing.md` (Database Assertions section) | 13.18.1+ lets you write `$this->assertDatabaseEmpty(User::all())` or `[$tableA, $tableB]` — was string-only and threw on collections |
| `api` / `json` routes respect `php artisan down` (PR #60595) | `controllers.md` (Maintenance section) + `deployment.md` (Maintenance Mode section) | 13.18.1+ returns JSON `503` for API/JSON callers under maintenance with `secret` bypass — drop hand-rolled maintenance middleware for API routes |
| `on-demand` log stacks respect configured channel name (PR #60635) | `logging.md` (Stacked / On-Demand Channels section) | 13.18.1+ `Log::build(['driver' => 'errorlog', 'channel' => 'audit'])` writes to the `audit` channel — was silently falling through to parent in some callsites |
| Inspect delayed jobs on `Queue::fake()` (PR #60636) | `testing.md` (Queue Assertions section) | 13.18.1+ queue fake tracks `availableAt` so `assertPushed(Job::class)` followed by `->delay(60)` inspections work — was lost on dispatch |
| `Str::mask()` encoding-aware tail (PR #60646) | `validation.md` (Sanitization Helpers section) + `security.md` (Output Encoding section) | 13.18.1+ doesn't chop a multi-byte character at the mask boundary — was emitting partial UTF-8 characters on long-tail strings |
| `foreignUuid` / `foreignUlid` Blueprint return types (PR #60643) | `migrations.md` (Foreign Key Constraints section) | 13.18.1+ returns `ColumnType::Uuid` / `ColumnType::Ulid` matching `foreignId()` — schema-dumpers and migration generators handle them uniformly |
| Predis retry config accepts scalar values (PR #60642) | `deployment.md` (Redis Configuration section) | 13.18.1+ `redis.options.read_write_timeout` etc. can be scalars in `config/database.php` without breaking `config:cache` (Predis-only; PhpRedis unaffected) |
| `RateLimited` middleware `releaseAfter` `__sleep()` fix (PR #60609) | `queues.md` (RateLimited middleware section) | 13.18.0+ serializes throttle timestamps so `dispatch($job->afterCommit())` doesn't lose the throttle window |
| `schedule:work` graceful signal handling (PR #60616) | `artisan.md` (schedule:work section) | Long-lived scheduler in Docker/K8s/supervisord exits cleanly on SIGTERM, lets in-flight runs finish |
| Soft-delete `restore()` event gating (PR #60605) | `observers.md` (restored callback note) | Laravel 13.17+ only fires `restored` when the underlying save() actually succeeded — check the boolean return |
| Embedding caching (`Embeddings::for(...)->cache(seconds: ...)`) | `ai.md` (Embedding Caching section) | Skip duplicate API calls for repeated inputs (knowledge base, FAQs) |
| `->queue()` for long-running AI calls | `ai.md` (Queue Long-Running AI Calls section) | Audio transcription, image gen, large-doc summarization — don't block HTTP |
| Prompt caching (`providerOptions(['cache_control' => [...]])`) | `ai.md` (Prompt Caching section) | Anthropic / OpenAI 90% discount on cached input tokens for agents with fixed system prompts |
| `RemembersConversations` trait + `Conversational` interface | `ai.md` (Conversation Memory section) | Multi-turn chat that persists history automatically — required for any real chatbot / support agent / in-product assistant |
| Per-class `Agent::fake()` + `preventStrayPrompts()` + `assertPrompted()` | `ai.md` (Testing AI Agents section) | Zero-API-cost AI tests in CI; per-agent scoping is cleaner than facade-level `AI::fake()` |
| Per-resource fakes (`Image::fake`, `Transcription::fake`, `Embeddings::fake`, `Reranking::fake`, `Files::fake`) | `ai.md` (Testing AI Agents section) | Each AI resource has its own fake + assertions — `AI::fake()` only covers text completions |
| `HasStructuredOutput` interface + `schema()` auto-fake | `ai.md` (Testing AI Agents section) | Structured-output fakes auto-generate schema-matching fake data — no hand-written JSON |
| `php artisan i18n:check` (missing translation detector) | `localization.md` (Detecting Missing Translations section) | CI integration to fail builds on incomplete translations |
| Lazy translation loading (JSON-only, lazy namespaces, DB+cache) | `localization.md` (Lazy Translation Loading section) | Avoid loading thousands of keys on every request for large apps |
| `Model::preventLazyLoading()` / `preventSilentlyDiscardingAttributes()` / `preventAccessingMissingAttributes()` / `prohibitDestructiveCommands()` | `eloquent.md` (Eloquent Strictness Hardening section) | Turn silent Eloquent bugs into loud exceptions in dev — N+1, missing `$fillable`, typos, accidental mass-delete |
| Laravel 13 class-level model attributes (`#[Table]` / `#[Fillable]` / `#[Hidden]` / `#[Visible]` / `#[Casts]`) | `eloquent.md` (Class-Level Model Attributes section) | Replace `$fillable` / `$hidden` / `$casts` / `$table` with class-level PHP attributes — Laravel 13's headline Eloquent shift |
| `whereVectorSimilarTo()` + `pgvector` + AI SDK embeddings | `eloquent.md` (Vector Similarity Search section) + `ai.md` | First-party semantic search on PostgreSQL + `pgvector`; accepts a plain string, embeds via AI SDK, ranks by cosine similarity |
| `Model::automaticallyEagerLoadRelationships()` (Laravel 12.8+) | `eloquent.md` (Automatic Eager Loading section) | Auto-fix N+1s at runtime — pairs with `preventLazyLoading()` (dev: throw, prod: auto-fix) |
| `Collection::reduceInto()` (PR #60651) | `eloquent.md` (Collection Aggregation section) | 13.19.0+ drop-in faster `Collection::reduce()` when the reducer doesn't read the carry — ~10–20% faster, pairs with `reduceMany` (13.15) |
| `Str::counted()` / `Stringable::counted()` (PR #60649) | `validation.md` (Sanitization Helpers section) | 13.19.0+ character-count helper — `mb_strlen(Str::of($s))` becomes `Str::counted($s)`. Cleaner min/max-N-character validation |
| `Http::query()` (PR #60663) | `api.md` (HTTP Client section) | 13.19.0+ non-sending HTTP client method — build a request and inspect it (`->url()`, `->body()`, `->data()`) without dispatching. Pairs with `assertQuery()` for URL/query-string assertions |
| `query` / `queryJson` HTTP testing helpers (PR #60662) | `testing.md` (HTTP Client Testing section) | 13.19.0+ `$response->assertQuery([...])`, `assertQueryMissing(...)`, `assertQueryJson('items.0.name', 'foo')` — same shape as `assertJson` for query strings |
| `assertSoftDeleted()` / `assertNotSoftDeleted()` `deletedAtColumn` param (PR #60657) | `testing.md` (Soft-Delete Assertions section) | 13.19.0+ pass `deletedAtColumn: 'archived_at'` to soft-delete assertions on models with custom `getDeletedAtColumn()` |
| Bulk SQS jobs via `SendMessageBatch` (PR #60645) | `queues.md` (SQS section) | 13.19.0+ SQS driver groups up to 10 dispatches per `SendMessageBatch` call. Free 10x API reduction for `Bus::bulk()` / `Queue::bulk()` on SQS — no code change |
| Postgres `whereDate` / `whereTime` Expression fix (PR #60540) | `eloquent.md` (PostgreSQL Gotchas section) | 13.19.0+ (and 12.63.0) `whereDate('created_at', ...)` / `whereTime('created_at', ...)` no longer crash when the column is a `DB::raw()` / `Expression` |
| `DateRule::past()` / `future()` / `nowOrPast()` / `nowOrFuture()` (PR #60687) | `validation.md` (Date Rules section) | 13.19.0+ explicit helpers on the date rule — clearer intent than the generic `after` / `before` rules when the semantic is "is the date in the past / future" |
| Cloud-agent queue pop (PR #60659) | `queues.md` (Cloud Queue section) | 13.19.0+ Laravel Cloud queue driver polls the cloud agent endpoint instead of SQS directly — fewer duplicate polls, no SQS API cost in cloud deployments |
| `whereAll()` / `whereAny()` / `orWhereAll()` / `orWhereAny()` (Laravel 10.47+) | `eloquent.md` (Query Builder section) | Multi-column WHERE with AND/OR semantics — cleaner alternative to nested closures for flat column lists |
| `whereMorphRelation` (Laravel 12+) | `eloquent.md` (Query Builder section) | Relationship-scoped `whereHas` shortcut for polymorphic relations |
| `#[Fillable]` / `#[Casts]` known bug [framework#59270](https://github.com/laravel/framework/issues/59270) | `eloquent.md` (Class-Level Model Attributes — Pitfalls) | `Model::query()->create()` misses the attribute registration on some 13.x point releases; `new Model()` + `->save()` is always fine |
| `Number::forHumans` / `Number::abbreviate` / `Number::fileSize` INF/NaN guard (PR #60617 + #60625) | `performance.md` (Number OOM section) + `api.md` (Number DoS section) | 13.18.0+ stops these from recursing on `INF`/`NaN` and OOM-crashing the worker — treat as a remote-DoS vector on API responses that render user input |
| `Request::json()` top-level zero body fix (PR #60614) | `api.md` (Error Responses section) | 13.18.0+ preserves a literal `0` request body (was coerced to `[]`); `PUT /counters/abc/reset` endpoints with body `0` now work without a `?? 0` workaround |
| `@checked` / `@selected` / `@disabled` / `@readonly` / `@required` form-helper directives | `blade.md` (Form-Helper Directives section) | Collapse the verbose `{{ old(...) == $x ? 'checked' : '' }}` pattern into single conditional attributes; emits static attribute when truthy, nothing when falsy (also XSS-safer than hand-rolled concatenations) |
| `@verbatim` directive (Vue/Alpine/JS) | `blade.md` (`@verbatim` section) | Wrap a chunk of frontend-framework template syntax so Blade doesn't try to parse `{{ }}` and `@click`; can't be nested |
| `Blade::render()` inline string rendering | `blade.md` (Rendering Inline Blade Strings section) | AI-generated output, CMS user templates, DB-stored email templates; **NOT sandboxed** — user-supplied templates can RCE via `@php` or arbitrary method calls; pass `deleteCachedView: true` and ideally whitelist directives or use Twig/Mustache for user-facing templates |
| Inline component views via `<<<'blade'` heredoc | `blade.md` (Inline Component Views section) | Class-based components return the Blade template directly from `render()`; best for small components, >20 lines still want external view files for cache + IDE highlighting |
| `@class` and `@style` directives (Laravel 9.18+) | `blade.md` (`@class` and `@style` Directives section) | The most-missed Blade directive for CSS class lists — collapses `class="{{ $isActive ? 'active' : '' }}"` ternaries into a single array-based declaration, and is XSS-safe (only array keys become classes, not the values). Use for any class list with 2+ conditions; pairs cleanly with Tailwind |
| `<x-slot:foo>` short-form slot syntax (Laravel 9+) | `blade.md` (`<x-slot:foo>` Short-Form Slot Syntax section) | Drop the `name=` attribute for static slot names — IDEs and `grep` treat `<x-slot:header>` as a single token, fixing a real refactor-rename problem. Long form only needed when the slot name is dynamic |
| `@aware` directive for parent-to-child data flow (Laravel 9+) | `blade.md` (`@aware` Directive section) | Child components pull parent data without explicit prop threading — mirrors Vue's `provide/inject`. Watch: the parent MUST have the prop defined (child silently gets `null` otherwise), no type safety, and past 1 level of nesting prefer explicit props for visibility |
| `<x-dynamic-component>` for runtime-resolved component names | `blade.md` (`<x-dynamic-component>` section) | The right answer for CMS widget systems, A/B-tested components, and plugin architectures. AI models default to `@include($component->name, [...])` for this pattern, which bypasses the entire component system. Always validate the view exists with `View::exists()` first |
| `view:cache` / `view:clear` + Octane stale view cache | `blade.md` (View Cache section) | `view:cache` is NOT part of `php artisan optimize` — add it explicitly to CI/CD deploy scripts. Octane workers keep compiled views in process memory; after a deploy that touches `resources/views/`, run `octane:reload` (or rely on `octane.warm` config). Most AI-generated deploy scripts miss both |
| FrankenPHP + Octane (Caddy single-binary runtime) | `deployment.md` (FrankenPHP section) | Laravel Cloud's underlying runtime; HTTP/3 + auto HTTPS + Mercure + Brotli out of the box; preferred over Swoole/RoadRunner for new Octane deploys on PHP 8.3+ |
| OPcache preload + tracing JIT (PHP 8.3+) | `deployment.md` (OPcache + JIT section) | Single biggest free performance win
| `EXPLAIN` / index-pattern deep dive + cache stampede SWaR | `performance.md` (Index Patterns section + Cache Stampede section) | Pre-optimize slow queries with the equality→range→ORDER BY composite rule; prevent thundering herd with `Cache::lock()->block()` or `Cache::flexible()` |
 — preload the framework autoloader into shared memory at worker boot; pair with `validate_timestamps=0` + SAPI reload on deploy |
| FrankenPHP thread-pool split (slow-endpoint isolation) | `deployment.md` (FrankenPHP section) | Mirror of FPM `pm` pool split — keep `/api/reports/*` and `/api/exports/*` on a dedicated thread pool so they can't starve the main request pool |
| `LARAVEL_STORAGE_PATH` for embedded FrankenPHP binaries | `deployment.md` (FrankenPHP section) | Each new binary extraction lives in a different temp dir — set `LARAVEL_STORAGE_PATH=/var/lib/yourapp/storage` or call `Application::useStoragePath()` so uploads/logs/caches survive upgrades |


| `Storage::fake()` vs `Storage::persistentFake()` decision matrix | `file-uploads.md` (Testing File Uploads section) | `persistentFake()` keeps files on disk after the test — required for byte-level content assertions (resized image width, watermarked PDF content, etc.). `fake()` deletes everything at end-of-test |
| `putFile` / `putFileAs` / `writeStream` vs `store` / `storeAs` decision matrix | `file-uploads.md` (Streaming APIs section) | `putFileAs()` auto-UUIDs the filename and streams; `writeStream()` takes any PHP resource (`fopen('http://...', 'r')`, tmp file handle, S3 read stream) — both avoid loading full bytes into memory |
| S3 multipart upload for files >5 GB (`CreateMultipartUpload` / `UploadPart` / `CompleteMultipartUpload`) | `file-uploads.md` (S3 Multipart Uploads section) | S3 PUT hard-limits at 5 GB. Pair with `tus.io` for browser-resumable or `uppy-aws-s3-multipart` for guided JS uploads. Laravel has no one-line API — use the SDK passthrough |
| `temporaryUploadUrl()` driver restrictions (s3 + local only — R2/MinIO/Backblaze fall back to SDK) | `file-uploads.md` (Direct S3 Upload section) | R2/MinIO/Backblaze throw `RuntimeException("Driver ... does not support generating temporary upload URLs.")` — reach for `Storage::disk('r2')->getAdapter()->getClient()->createPresignedRequest(...)` directly |
| `finfo` resource hygiene on Octane (`finfo_open()` once, `finfo_close()` on `Octane::flush()`) | `file-uploads.md` (Common Mistakes #13) + `security.md` (Anti-Extension-Spoofing section) | Opening `finfo` per-request under Octane leaks resource handles across workers; bind it in `AppServiceProvider::register()` and close on flush |

| `Lang::handleMissingKeysUsing()` (Laravel 10.33+) — log + safe placeholder for missing keys | `localization.md` (Missing Translation Key Handler section) | Default behavior leaks raw key strings (`messages.checkout.totals.subtotal`) to the user; this callback logs to a dedicated channel + fires a Sentry breadcrumb and returns a bracketed `[locale/key]` placeholder. Anti-loop guard built in. Pair with `Lang::has()` for keys you expect to be missing in some locales |
| `Lang::hasForLocale()` vs `Lang::has()` (PR #10767, Laravel 5.1+) | `localization.md` (Locale-Specific Key Existence section) | 10-year-old footgun — `Lang::has($key, $locale)` returns `true` whenever the fallback chain has the key. Use `Lang::hasForLocale()` for RTL flip indicators, audit reports, locale-specific UI; `Lang::has()` (no locale) is correct for "any translation including fallback" |
| Carbon `isoFormat()` vs `format()` vs `translatedFormat()` | `localization.md` (Carbon Localization section) | `format()` is PHP native (always English); `translatedFormat()` needs `setlocale(LC_TIME, ...)` and breaks in Docker/CI; `isoFormat()` uses Carbon's embedded CLDR translations — works everywhere. CLDR token syntax (`YYYY`/`MM`/`dddd`) is NOT the same as PHP `date()` tokens |
| Word-order-safe translations (placeholder pattern for German/Arabic/Hindi/Japanese) | `localization.md` (Word-Order-Safe Translations section) | `<button>{{ __('Attach') }} {{ $resource->name }}</button>` is wrong; German puts the verb at the end (`Beitrag anhängen`), Japanese uses honorific suffixes. Extract to one key with placeholder: `'attach_resource' => ':resource anhängen'`. AI-generated UIs miss this 80% of the time; bulk-audit script included |
| `spatie/laravel-translatable` for Eloquent attribute translation (v6.x, 5M+ installs) | `localization.md` (Eloquent Attribute Translation section) | De facto community package for translating model attributes (product names, blog post bodies, page titles). Stores translations as JSON in one column. Column MUST be `json`/`text` not `string`; `$appends` accessors don't auto-localize; `toArray()` leaks all translations (use JsonResource with `getTranslation()`) |
| `Translator::addPath()` / `addJsonPath()` / `addNamespace()` — undocumented runtime translation-path injection | `localization.md` (Plugin / Multi-Tenant Translation Paths section) | Use for multi-tenant SaaS (per-tenant `lang/` overrides), modular monoliths (per-module `billing::invoice.title` namespaces), white-label resellers, A/B-tested copy variants. Last-added wins (reverse-registration search order). Call once at boot in a service provider, NOT per-request |

| `$this->components->bulletList()` + the styled output factory (`bulletList` / `twoColumnDetail` / `info` / `warn` / `error` / `confirm` / `ask` / `choice` / `secret`) | `artisan.md` (Console Output Components section) | AI assistants default to manual `echo "things:\n"` loops; the factory is what Laravel itself uses for `migrate`, `schedule:list`, `queue:listen` — terminal-width-aware, ANSI-styled, consistent across all your commands |
| `schedule:clear-cache` (NOT `schedule:clear`) — clears stuck `withoutOverlapping()` mutex locks | `artisan.md` (`schedule:clear-cache` section) | AI-default `schedule:clear` doesn't exist ("Command schedule:clear is not defined"); the correct command has `-cache` appended since PR #40135. Default TTL = 24 hours; pass `->withoutOverlapping(N)` to override. Clear all or target the cache key directly with `Cache::forget('framework/schedule-mutex-<signature>')` |
| `Artisan::call()` exit codes + Octane trap (state leak + Kernel reuse + `BufferedOutput` capture) | `artisan.md` (`Artisan::call()` Exit Codes section) | Three real footguns the official docs gloss over: (1) constructor-injected request-scoped state leaks across Octane worker requests, (2) `Mail::fake()` / facade mocks / `config()` mutations persist into the next request, (3) output is silently captured into `BufferedOutput` unless you pass `null` or omit the 3rd arg. For admin actions that don't need synchronous completion, prefer `Artisan::queue()` (dispatches to a worker with a fresh kernel) |
| `php artisan down` flag reference (`--retry=N` / `--refresh=N` alias / `--render=view` / `--redirect=/path` / `--secret=token`) | `artisan.md` (`php artisan down` / `php artisan up` section) | All five flags compose freely. `--render` views run BEFORE the app boots — no DB, no facades, embed CSS inline. 13.18.1+ returns JSON 503 for API/JSON routes when `--secret` bypass is active (PR #60595). See `deployment.md` Maintenance Mode section for the full workflow |

## Critical Rules (Never Forget)

- **`env()` only in config files** — returns null after `config:cache`
- **Eager load relationships** — `with('posts')` or N+1 queries hit on every loop
- **`Model::preventX()` strictness hooks** — every Laravel project should turn on `preventSilentlyDiscardingAttributes()`, `preventAccessingMissingAttributes()`, and `preventLazyLoading()` in `AppServiceProvider::boot()`. Default Laravel skeleton ships with none.
- **`$fillable` or `$guarded`** — unprotected mass assignment = security hole
- **`findOrFail()` over `find()`** — null returns silently, crashes are better than silent bugs
- **Queue jobs serialize models as IDs** — re-fetch on process, model may be stale/deleted
- **`DB::transaction()` rolls back on exceptions only** — `exit`/timeout bypasses it
- **`{!! !!}` skips escaping** — XSS vector, use `{{ }}` by default
- **`new HtmlString($userInput)` also bypasses escaping** — class-based `{!! !!}`. Always `e($value)` before wrapping (CVE-2026-33080 in Filament was a real-world instance of this pattern)
- **Middleware order matters** — earlier wraps later execution
- **`required` passes empty string** — use `required|filled` for actual content
- **`firstOrCreate` persists immediately** — `firstOrNew` lets you validate before saving
- **Route model binding defaults to `id`** — override `getRouteKeyName()` for slugs
- **`Route::metadata()` survives `route:cache`** (Laravel 13.17+) — first-class route metadata via dot notation; use for audit, feature flags, OpenAPI emitters
- **`Agent::fake()` belongs in every AI test** — never hit a real LLM in CI. Add `->preventStrayPrompts()` so any unmocked agent call throws (catches newly-added agent invocations before they leak API costs into CI).
- **Don't pair `messages()` with `RemembersConversations`** — defining `messages()` manually makes the trait silently no-op. Pick ONE: trait = automatic DB-backed history, `messages()` = you own storage (Redis, custom scoped table, etc.).
- **`schedule:work` now catches signals** (Laravel 13.17+, PR #60616) — `SIGINT`/`SIGTERM`/`SIGQUIT` stop new runs cleanly and let in-flight `schedule:run` finish; long-lived scheduler processes no longer need wrapper scripts
- **HEAD request cache headers** (Laravel 13.18.0+, PR #60589) — `SetCacheHeaders` middleware now applies to `HEAD`. Before: CDN cache-warm scripts got no `Cache-Control`/`ETag`. Force cache headers explicitly if stuck on 13.16
- **Discriminated unions: `anyOf()` not `enum()`** (Laravel 13.17+) — when an AI structured-output field has multiple shapes (article / video / podcast), never collapse to `$schema->string()->enum([...])`. Use `$schema->anyOf([...])` so each variant keeps its type-specific fields; branch on the discriminator in your response handler. See `ai.md` (Advanced JSON Schema section).
- **Multi-type unions: `JsonSchema::union()` not `string|null`** (Laravel 13.17+) — when a JSON field can be `string` / `number` / `boolean`, don't fall back to a permissive `string` cast. `JsonSchema::union(['string', 'number'])->nullable()` preserves all three types in the structured-output contract; `'type' => ['string', 'number']` via `fromArray()` was broken before 13.17 (PR #60455).

- **Missing translation keys leak raw strings** — `__('messages.checkout.totals.subtotal')` returns the literal `"messages.checkout.totals.subtotal"` to the user when the key is missing. Register `Lang::handleMissingKeysUsing(fn ($key, $replace, $locale) => ...)` in `AppServiceProvider::boot()` to log + return a safe placeholder. Without it, refactor-renamed keys ship silently. Laravel 10.33+.
- **`Lang::has($key, $locale)` is fallback-aware, not locale-specific** — returns `true` whenever the fallback chain has the key, regardless of whether `$locale` itself has it. Use `Lang::hasForLocale($key, $locale)` for RTL flip indicators, audit reports, locale-specific UI. `Lang::has()` without a locale is correct for "any translation including fallback".
- **`$date->format()` is always English** — `format()` is PHP `DateTime::format()` under the hood, ignores `setlocale()`. `translatedFormat()` needs the OS locale installed (fragile in Docker/CI). Use `$date->isoFormat('dddd D MMMM YYYY')` — Carbon's embedded CLDR translations work everywhere, no OS dependency.
- **`schedule:clear-cache` is the command, NOT `schedule:clear`** — AI assistants hallucinate the bare form and Laravel replies "Command schedule:clear is not defined." If a `withoutOverlapping()` task appears to skip after a worker crash, deploy, or cache flush, run `php artisan schedule:clear-cache` — default mutex TTL is 24 hours; pass `->withoutOverlapping(N)` to override.
- **`UploadedFile::hashName()` / `Str::uuid()` for storage names, never `pathinfo(PATHINFO_FILENAME)`** — the pathinfo-based "strip everything after the last dot" pattern is the root cause of `plank/laravel-mediable` CVE-2026-49972 (CVSS 8.8, RCE via `shell.php.jpg` on misconfigured Apache/nginx) and is what AI assistants default to. Always store uploads with `hashName()` (server-rendered hex + server-detected extension) and disable PHP execution in the upload directory at the web server level — see `security.md` § "Critical: plank/laravel-mediable" CVE-2026-49972 for the full defense-in-depth pattern.
- **`Artisan::call()` and Octane workers don't mix** — the called command reuses the current Kernel, so constructor-injected request-scoped state (tenant, auth user, request ID), facade mocks, and `config()` mutations leak across requests in the same Swoole/RoadRunner/FrankenPHP worker. Inject per-invocation state inside `handle()` (not the constructor) and prefer `Artisan::queue()` for any HTTP-triggered admin work — it dispatches to a worker with a clean kernel.
- **`$this->components->bulletList()` and friends, not raw `echo`** — every command output beyond a single line should go through the `components` factory (`bulletList`, `twoColumnDetail`, `info`, `warn`, `error`, `confirm`, `ask`, `choice`, `secret`). Same API Laravel itself uses for `migrate`, `schedule:list`, `queue:listen` — terminal-width-aware, ANSI-styled, consistent across all your commands.
- **`@class` / `@style` directives, not ternary class lists** — `class="{{ $isActive ? 'active' : '' }}"` is XSS-prone if `$isActive` is ever non-string, harder to grep, and harder to lint. Use `@class(['nav-link', 'nav-link--active' => $isActive])` — only array keys become classes, not the values. Laravel 9.18+.
- **Always run `php artisan view:cache` in production deploys** — `view:cache` is NOT part of `php artisan optimize` (which only handles `config:cache`, `route:cache`, `event:cache`). Add it explicitly to your CI/CD script AFTER `composer install` and BEFORE the worker pool accepts traffic. Symptom of missing it: first requests after deploy get `ErrorException: include(): Filename cannot be empty`.
- **Octane workers keep compiled views in process memory** — after a deploy that changes a `.blade.php`, in-flight Swoole/RoadRunner/FrankenPHP workers still render the OLD template. Run `php artisan octane:reload` after every deploy that touches `resources/views/`, or pre-load views via the `octane.warm` config list. Without it, `view:clear` clears disk but the workers still serve stale views.

## Version Defaults

- **Laravel 13** (latest — requires PHP 8.3+)
- PHP 8.3+ (8.2 minimum, optimized for 8.4)
- Composer 2.x
- SQLite for local dev (zero config)
- Vite for asset bundling (not Laravel Mix)
- Laravel Reverb for WebSocket/real-time

## Pro Tips

- Run `php artisan route:list --json` to get all routes as JSON for programmatic use
- Use `php artisan tinker` to test code in real Laravel context
- Use `php artisan about` to see full environment summary
- Use `php artisan optimize` after major changes in production
- Use `php artisan make:model Post -mcr` to generate Model + Migration + Controller in one shot

Start with `eloquent.md` for most common Laravel work.

| `ShouldQueue` on observers + retry/backoff + `ModelNotFoundException` gotcha | `observers.md` (ShouldQueue on Observers section) | Observer re-fetches model by PK after the request returns — if you delete the model in the same request, pass data via DTO constructor args or wrap in an event with `ShouldQueue` listeners |
| `ShouldHandleEventsAfterCommit` for queued observers/listeners (GitHub #52440 gotcha) | `observers.md` (ShouldHandleEventsAfterCommit section) | Side effects fire after transaction commits — without it, observer work runs even if the transaction rolls back (data inconsistency). Watch: `ShouldQueue` + `ShouldHandleEventsAfterCommit` early-returns on Redis/SQS — pick one |
| Octane + static observer-registration state leak (`Post::flushEventListeners()` + `Octane::flush()`) | `observers.md` (Octane & Static Observer Registration section) | Constructor-injected request-scoped state (tenant, auth user) leaks across requests in the same worker — inject via method body or reset in `Octane::flush()` |
| `withoutEvents()` vs `updateQuietly()` vs `saveQuietly()` decision matrix | `observers.md` (Quiet Patterns section) | Different scopes: instance-only vs closure-wide; only `withoutEvents()` blocks ALL models in the closure. `updateQuietly()`/`saveQuietly()` don't suppress related-model events |
| `Event::fake([Class])` vs `Event::fake()` vs `Event::fakeFor()` for observer tests | `observers.md` (Testing Observers section) | Targeted faking (`Event::fake([PostCreated::class])`) preserves downstream events; `Bus::fake()` catches queued observers' internal job dispatches |
| `setObservableEvents()` / `getObservableEvents()` for selective event firing | `observers.md` (Selective Event Firing section) | Per-instance or per-class narrowing of which model events fire — useful for read-only DTO models, audit-only observers, performance-constrained sync imports |
