# Controllers & Routing

## Resourceful Routes

```bash
# Generates: index, create, store, show, edit, update, destroy
php artisan make:controller PostController --model=Post --requests

# Plain resource (no create/edit)
php artisan make:controller PostController --model=Post --api

# Single-action (invokable) controller — generates __invoke()
php artisan make:controller ProvisionServer --invokable
```

**Routes file (`routes/web.php`):**
```php
Route::resource('posts', PostController::class);

// Register an invokable controller — pass class, no method name
Route::post('/server', ProvisionServer::class);

// Nested (posts.comments)
Route::resource('posts.comments', CommentController::class);

// Partial
Route::resource('posts', PostController::class)->only(['index', 'show']);
Route::resource('posts', PostController::class)->except(['destroy']);
```

## Controller Structure

```php
class PostController extends Controller
{
    public function index() { /* GET /posts */ }
    public function create() { /* GET /posts/create */ }
    public function store(Request $request) { /* POST /posts */ }
    public function show(Post $post) { /* GET /posts/{post} */ }
    public function edit(Post $post) { /* GET /posts/{post}/edit */ }
    public function update(Request $request, Post $post) { /* PUT/PATCH /posts/{post} */ }
    public function destroy(Post $post) { /* DELETE /posts/{post} */ }
}
```

## Single Action Controllers (`__invoke`)

Use one controller class per non-CRUD endpoint when the action is complex enough to deserve its own class — webhook handlers, OAuth callbacks, billing webhooks, action endpoints that don't fit the 7-method resource model:

```php
class ProvisionServer extends Controller
{
    public function __invoke(ProvisionServerRequest $request)
    {
        $server = $this->provisioner->provision($request->validated());
        return response()->json(['id' => $server->id], 201);
    }
}
```

```php
// Route definition — pass class, no method
Route::post('/servers', ProvisionServer::class)->middleware('auth');
Route::match(['get', 'post'], '/webhooks/stripe', StripeWebhook::class);
```

**Why `__invoke` over a named method:**
- Each action can have its own typed FormRequest (each webhook can validate a different payload)
- Each action can have its own constructor with narrow dependencies (`StripeWebhook` injects only `StripeClient`, not the full `BillingService`)
- Easy to extract into a package without dragging in 6 unrelated sibling actions
- Plays nice with `Route::macro()` for ASP.NET-style action grouping

**Combine with `#[Middleware]` / `#[Authorize]` attributes:**

```php
#[Middleware('auth:sanctum')]
#[Authorize('create', Server::class)]
class ProvisionServer extends Controller
{
    public function __invoke(ProvisionServerRequest $request) { /* ... */ }
}
```

## Laravel 13 Controller Attributes

Laravel 13 introduces PHP attributes for middleware and authorization directly on controllers. **Laravel 13.20+** added the matching `#[WithoutMiddleware]` attribute to *exclude* middleware from specific actions — the missing complement to `#[Middleware]`:

```php
use Illuminate\Routing\Attributes\Controllers\Authorize;
use Illuminate\Routing\Attributes\Controllers\Middleware;
use App\Models\Comment;
use App\Models\Post;

// Apply middleware to entire controller
#[Middleware('auth')]
class PostController extends Controller
{
    // Apply middleware to specific method
    #[Middleware('subscribed')]
    #[Authorize('create', [Comment::class, 'post'])]
    public function store(Post $post)
    {
        // Method-level middleware and policy authorization
    }
}
```

// Laravel 13.20+ — exclude specific middleware from individual methods
use Illuminate\Routing\Attributes\Controllers\WithoutMiddleware;
use App\Http\Middleware\EnsureEmailIsVerified;

#[Middleware('auth')]
class UserController extends Controller
{
    // All methods get 'auth', but...

    #[WithoutMiddleware(EnsureEmailIsVerified::class)]
    public function profile()
    {
        // ...this method is also exempt from the email-verified check.
    }
}
```

**`#[WithoutMiddleware]` gotchas (PR #60709):**
- It can only **remove route-level middleware**, not global middleware. A class-level `#[Middleware('auth')]` is still overridable; a `$middleware->append(EnsureJsonResponse::class)` global is not.
- Matching uses `ReflectionAttribute::IS_INSTANCEOF` — exclude the parent middleware class and all subclasses are excluded too. Exclude only the leaf and you get the inverse.
- Class-level `#[WithoutMiddleware]` is inherited by child controllers (subclasses). Re-declare at the subclass if you want to undo.
- Use the `only` / `except` array args to scope a class-level attribute to particular methods: `#[WithoutMiddleware('throttle:api', except: ['show'])]`.
- The route-level `->withoutMiddleware([...])` and `Route::resource()->withoutMiddlewareFor([...], ...)` chain methods still work — the attribute is just the typed, IDE-discoverable alternative.

**Key attributes:**
- `#[Middleware('name')]` — apply middleware to controller or method
- `#[Authorize('action', [Model::class, 'relation])` — authorize with policy
- `#[WithoutMiddleware('name')]` *(13.20+)* — exclude middleware from a controller method

## Request Typed Accessors (Laravel 12+)

`Illuminate\Http\Request` ships typed accessors that auto-cast and return safe defaults — eliminating `?:` null-coalescing ladders and `intval()` / `strtolower()` boilerplate:

```php
public function update(StorePostRequest $request, Post $post)
{
    $title       = $request->string('title')->trim();        // Stringable — chains Str methods
    $perPage     = $request->integer('per_page', 25);         // int; default 25 if missing/invalid
    $archived    = $request->boolean('archived');             // accepts 1, "1", true, "true", "on", "yes"
    $tags        = $request->array('tags') ?? [];             // array, [] if missing
    $status      = $request->enum('status', PostStatus::class, PostStatus::Draft); // enum or default
    $products    = $request->enums('products', Product::class); // array of enums (skips invalid)
    $publishedAt = $request->date('published_at');           // Carbon; null if missing
    $amount      = $request->float('amount', 0.0);            // float, default
}
```

**Decision matrix — when to use which:**

| Method | Returns | Default behavior | When to use |
|---|---|---|---|
| `->input('key')` | `mixed` | `null` | Need raw, untyped payload |
| `->string('key')` | `Stringable` | empty `Stringable` | Trimming, regex, chaining `Str` |
| `->integer('key', 0)` | `int` | the second arg | Page sizes, counts, IDs from query |
| `->float('key', 0.0)` | `float` | the second arg | Money, percentages, ratios |
| `->boolean('key')` | `bool` | `false` | Checkboxes, feature flags from form |
| `->array('key', [])` | `array` | the second arg | Tag lists, multi-selects |
| `->date('key')` | `?\Carbon` | `null` | Date inputs that need Carbon math |
| `->enum('key', E::class, $default)` | `?E` | the third arg (or `null`) | Validated enum from request body |
| `->enums('key', E::class)` | `array<E>` | `[]` | Array of enums, invalid entries skipped |

**`Request::enum()` vs `Rule::enum()` (the most-confused pair):**
- **`Request::enum($key, E::class, $default)`** — used at *read time* in the controller. Returns the enum instance (or default). Doesn't validate; assumes input is already valid.
- **`Rule::enum(E::class)`** — used in a validator (`'status' => [Rule::enum(E::class)]`). Validates the input, returns a 422 on bad values.
- **Correct pairing:** `Rule::enum()` in the FormRequest's `rules()` for validation, then `$request->enum('status', E::class)` in the controller for the typed read. Skip the rule when you have a default fallback (`Request::enum($key, E::class, E::Pending)`).
- **For a form field you accept from the user:** use both. `Rule::enum()` enforces, `Request::enum()` reads.
- **For a field you set server-side with a known-safe default:** `Request::enum($key, E::class, E::Pending)` only.

**`boolean()` truthy values (canonical list, do not deviate):**
Returns `true` for: `1`, `"1"`, `true`, `"true"`, `"on"`, `"yes"` (case-insensitive).
Returns `false` for: literally everything else, including `"0"`, `"false"`, `"off"`, `""`, `null`, and the string `"no"` *if you mess up the casing*. If your HTML form sends `"NO"` (uppercase), it returns `false` — check your `<input>` value attribute matches one of the canonical strings.

**`string()` returns `Stringable`, not `string`:**
```php
$name = $request->string('name')->trim()->lower()->title();
// For raw string coercion in a JSON response:
$name = (string) $request->string('name');
// For comparisons:
if ($request->string('email')->lower() === 'admin@example.com') { /* ... */ }
```
Forgetting the `(string)` cast when stuffing into a JSON array is the #1 typo with this helper — `Stringable` serializes fine to JSON but blows up on `===` against a plain string.

Source: [Laravel 13 Docs - HTTP Requests](https://laravel.com/docs/13.x/requests#retrieving-input) | [Laravel 13 Docs - Validation](https://laravel.com/docs/13.x/validation#rule-enum)

## Route Patterns (Global Constraints)

```php
// Apply globally — every {id} in the entire app must now match \d+
Route::pattern('id', '[0-9]+');
Route::pattern('slug', '[a-z0-9-]+');
Route::pattern('uuid', '[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}');

// Numeric alternative to missing implicit binding
Route::pattern('year', '[0-9]{4}');
Route::get('/archive/{year}', function (int $year) { /* ... */ })->whereYear('year');
```

**Where this bites:**
- After declaring `Route::pattern('id', '\\d+')` globally, `posts/{post}` with implicit binding works, but `/posts/notanid` will now 404 instead of triggering a `ModelNotFoundException` — make sure tests expect 404, not 500.
- Place `Route::pattern()` calls in `bootstrap/app.php` or a service provider's `boot()` — they only need to be registered once per process.
- Don't combine with `Route::where()` on the same parameter — the per-route `where()` already wins for that route.

## Route Metadata (Laravel 13.17+)

Attach structured metadata to any route — survives `route:cache` serialization and cascades through route groups. Read it back via dot notation for tooling, dashboards, OpenAPI emitters, or feature-flag/audit gates:

```php
use Illuminate\Support\Facades\Route;

// Attach metadata to a single route
Route::get("/users", [UserController::class, "index"])
    ->metadata([
        "head" => ["title" => "Users"],
        "feature_flag" => "user_directory_v2",
        "owner" => "identity-team",
        "audit" => ["tier" => "pii", "retention_days" => 90],
    ]);

// Cascade through route groups
Route::middleware("auth")->group(function () {
    Route::get("/billing", BillingController::class)
        ->metadata(["tier" => "restricted"]);
});

// Read it back with dot notation
$route = Route::getRoutes()->match(request());
$title = $route->getMetadata("head.title", "Default Title");
$tier = $route->getMetadata("audit.tier");
```

**Key behaviors:**
- Metadata persists through `route:cache` (serialized with the route definition)
- Group cascading — child routes inherit parent metadata; explicit child metadata wins on conflict
- `getMetadata($key, $default)` — dot notation lookup with optional default
- Useful for: per-route audit info, feature flags, internal ownership, doc links, dashboard generation, OpenAPI metadata emitters

**Use cases:**
- Audit dashboards — list every route + tier + retention policy in one query
- Feature-flag enforcement — middleware reads metadata to decide if the route is gated
- Auto-generated OpenAPI / Stoplight specs — pull title, description, owner from metadata
- Multi-team ownership — route ownership/team metadata for incident routing

Source: [Laravel News — Route Metadata](https://laravel-news.com/laravel-13-17-0) | [PR #60530](https://github.com/laravel/framework/pull/60530)

## API Versioning & Route Group Conventions

```php
// routes/api.php
Route::prefix('v1')->name('api.v1.')->middleware(['auth:sanctum'])->group(function () {
    Route::apiResource('posts', PostController::class);
    Route::post('/posts/{post}/publish', PublishPost::class)
        ->metadata(['feature_flag' => 'publish_v2']);
});

Route::prefix('v2')->name('api.v2.')->middleware(['auth:sanctum'])->group(function () {
    // ...
});
```

**Convention checklist:**
- **Prefix with version** (`/v1`, `/v2`) + name with `api.v1.*` so URL helpers generate stable names
- **Keep middleware ordered** (auth:sanctum → throttle → binding → controller), don't repeat per route
- **Bind one API resource per version group** — never cross-link between versions
- **Use `Route::apiResource()`** instead of `Route::resource()` for JSON-only endpoints (skips `create` / `edit`)
- **Sunset version**: when deprecating `/v1`, add a `Sunset` HTTP header via middleware that reads `Route::getMetadata('sunset')` — return the date customers must migrate by

**HEAD request cache headers (Laravel 13.18.0+):**

Before 13.18.0, the `SetCacheHeaders` middleware did not set `Cache-Control` / `ETag` on `HEAD` requests — a real gotcha for CDN cache-warming scripts and `link rel="preload"` audits. As of PR #60589 (13.18.0+) the middleware applies to HEAD as expected. If you can't upgrade yet, force cache headers explicitly:

```php
// Workaround for 13.16 and earlier
public function show(Post $post)
{
    $response = response()->json($post);
    if (request()->isMethod('HEAD')) {
        $response->setCache(['public', 'max_age' => 60]);
    }
    return $response;
}
```

Source: [PR #60589 — fix cache headers not set on HEAD requests](https://github.com/laravel/framework/pull/60589)

## Validation

```php
// Option A: Form Request (recommended for complex validation)
class StorePostRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'title' => 'required|string|max:255',
            'body' => 'required|string|min:20',
            'tags' => 'array',
            'tags.*' => 'integer|exists:tags,id',
            'published_at' => 'nullable|date|after:now',
        ];
    }

    public function messages(): array
    {
        return [
            'body.min' => 'Post content must be at least 20 characters.',
        ];
    }
}

// Controller
public function store(StorePostRequest $request)
{
    // $request->validated() gives clean data
    Post::create($request->validated());
    return redirect()->route('posts.index');
}
```

```php
// Option B: Inline validation (simple cases)
$validated = $request->validate([
    'title' => 'required|string|max:255',
    'email' => 'required|email|unique:users,email',
]);
```

## Dependency Injection

```php
// Type-hint interfaces for testability
public function __construct(private readonly PostRepository $posts)
{}

public function update(StorePostRequest $request, Post $post)
{
    $this->posts->update($post, $request->validated());
    return redirect()->route('posts.show', $post);
}
```

## Middleware

```php
// Apply to routes
Route::middleware(['auth', 'verified'])->group(function () {
    Route::resource('posts', PostController::class)->except(['index', 'show']);
});

// Controller constructor
$this->middleware('auth')->only(['create', 'store', 'edit', 'update', 'destroy']);
$this->middleware('cache.headers:no-store')->only(['create', 'edit']);
```

**Middleware order matters — earlier wraps later execution:**
- Laravel 10 and earlier: middleware order configured in `App\Http\Kernel`
- Laravel 11+: middleware registered in `bootstrap/app.php` via `$middleware->` chain

**Always put early-return middleware BEFORE expensive operations.**

## Request Lifecycle

```
Request → Middleware → Route → Controller → Model/DB → View/Response → Middleware → Client
```

**Never do heavy computation before authentication checks.**

## Response Helpers

```php
// JSON
return response()->json(['posts' => $posts], 200, [], JSON_PRETTY_PRINT);

// Redirect
return redirect()->route('posts.show', $post);
return redirect()->back()->withInput()->withErrors($errors);

// File download
return response()->download($path, 'export.csv', ['Content-Type' => 'text/csv']);

// Stream (large files)
return response()->streamDownload(fn() => print('data'), 200, $headers);
```

## Error Handling

```php
// Manual abort
abort(404, 'Post not found');

// With response
return response()->view('errors.404', [], 404);

// JSON API error
return response()->json(['error' => 'Not found'], 404);

// Try-catch for recoverable errors
try {
    $post = Post::findOrFail($id);
} catch (ModelNotFoundException $e) {
    return response()->json(['error' => 'Not found'], 404);
}
```

## Common Mistakes

1. **Using `required` rule for empty string** — it passes `''`. Use `required|filled`
2. **Not validating arrays properly** — `tags.*` rules for each item
3. **Skipping `authorize()` in FormRequest** — add policy checks there
4. **Wrong HTTP method** — PUT/PATCH for update, POST for create, DELETE for destroy
5. **Not using route model binding** — manual `Post::find($id)` bypasses middleware
6. **Validation errors not flashed** — ensure `Back` or `withErrors` is called
7. **Using a 7-method resource for one-off actions** — webhooks, OIDC callbacks, billing redirects are not CRUD. Use `__invoke()` controllers instead.
8. **Forgetting `Route::pattern()` after upgrading** — global constraints override implicit binding. Tests that expected `ModelNotFoundException` will now see a 404 and start failing.
9. **Comparing `Request::string('key')` directly to a `string`** — `$request->string('email')` returns a `Stringable`, not a string. `$request->string('email') === 'a@b.com'` is always false. Cast with `(string)` or use `->lower()` for case-insensitive comparison. The Stringable serializes correctly to JSON, which is why the bug only shows up in `===` / type-hint checks.
10. **Using `Request::enum()` to validate** — `Request::enum($key, E::class)` *does not* validate. It returns `null` (or your default) when the value isn't a valid enum case, but accepts whatever the user sent. Pair with `Rule::enum(E::class)` in the FormRequest when the value comes from user input.
11. **Trying `#[WithoutMiddleware]` on global middleware** — it can only strip route-level middleware (class-level `#[Middleware]` and `Route::middleware()`). For a global middleware (`$middleware->append(ForceJson::class)`), use `Route::withoutMiddleware()` or move the bypass logic into the middleware itself.
12. **Forgetting `(bool)` around `Request::boolean()` in `array_merge`** — `array_merge($validated, ['archived' => $request->boolean('archived')])` works because PHP coerces, but `$row['archived'] ?? false` after a JSON roundtrip will give you `true`/`false` reliably only if the value was cast. The HTML checkbox "0" / "1" / "on" / "off" trap hits people who skip the helper and do `$request->input('archived') === 'on'` instead.


## Updated from Research (2026-06-29)

- **Single Action Controllers (`__invoke`)** — One controller per non-CRUD endpoint. Combine with `#[Middleware]` / `#[Authorize]` for typed, narrowly-scoped action classes.
- **Maintenance Mode Returns JSON for API/JSON Routes (Laravel 13.18.1, PR #60595)** — `php artisan down` now correctly intercepts requests to API routes (`/api/*`) and routes that set `Accept: application/json`, returning a proper JSON `{"message": "Service Unavailable", "retry_after": N}` body instead of falling through to a 500. Use the `secret` bypass flag (`--secret=...`) for staff bypass during deploys. See `deployment.md` for the full maintenance-mode workflow.
- **`Route::pattern()` global constraints** — `Route::pattern('id', '\d+')` applies to every route in the app. Watch for 404 vs `ModelNotFoundException` after applying.
- **HEAD request cache headers fix (PR #60589)** — `SetCacheHeaders` middleware now applies to `HEAD` requests as of 13.18.0+. Fixes CDN cache-warming scripts that previously saw no `Cache-Control` / `ETag` on HEAD probes.
- **API versioning convention** — `prefix('vN')->name('api.vN.')` keeps URL helpers stable; use `Route::apiResource()` instead of `Route::resource()` for JSON-only endpoints; add `Sunset` HTTP header for deprecated versions via metadata.

## Updated from Research (2026-07-18, cycle 42) — v13.20 + typed accessors

- **`#[WithoutMiddleware]` controller attribute (Laravel 13.20+, PR #60709 by @JurianArie)** — typed, IDE-discoverable complement to `#[Middleware]`. Excludes a middleware from a single method or an entire controller. Subclass-aware (uses `ReflectionAttribute::IS_INSTANCEOF`). Does **not** strip global middleware. Class-level attribute is inherited by subclasses — re-declare to opt back in.
- **`Request::enum()` vs `Rule::enum()` (the most-confused pair)** — `Request::enum($key, E::class, $default)` is for *reading* the validated input as a typed enum at controller time (no validation). `Rule::enum(E::class)` is for *validating* in a FormRequest. Pair them for user-submitted enum fields. Use `Request::enum()` with a default when the field is server-set.
- **Request typed accessors (`Request::integer()` / `Request::string()` / `Request::boolean()` / `Request::float()` / `Request::array()` / `Request::date()` / `Request::enum()` / `Request::enums()`)** — Laravel 12+ helpers that auto-cast and accept a default. Replace `?:` null-coalescing chains and `intval()` / `strtolower()` boilerplate. Watch out: `string()` returns a `Stringable` (cast to `string` for `===` and JSON `array_merge`); `boolean()` is `true` only for `1`/`"1"`/`true`/`"true"`/`"on"`/`"yes"` — case-insensitive; no `boolean:strict` needed.
- **`#[Scope]` attribute on Eloquent query scopes** — replaces `scopeXxx()` method naming. `#[Scope] protected function published(Builder $query): void` instead of `public function scopePublished($query)`. Method must be `protected` (not `public`). IDE-discoverable, refactor-rename safe, plays nicely with static analysis. Cross-ref: `eloquent.md` (Scopes section).
- **`Route::redirect()` / `Route::view()` shortcut improvements** — both accept a third status-code / data arg respectively. `Route::redirect('/here', '/there', 301)` for permanent SEO redirects; `Route::view('/welcome', 'welcome', ['name' => 'Taylor'])` for view-only endpoints. Still doesn't run middleware per request — keep them for static redirects/views only.

Source: [Laravel News — Laravel 13.20](https://laravel-news.com/laravel-13-20-0) | [Laravel 13 Docs - Controllers](https://laravel.com/docs/13.x/controllers) | [Laravel 13 Docs - HTTP Requests](https://laravel.com/docs/13.x/requests) | [PR #60709 — `#[WithoutMiddleware]`](https://github.com/laravel/framework/pull/60709) | [Eloquent Docs — Query Scopes](https://laravel.com/docs/13.x/eloquent#query-scopes)
