# REST APIs — JSON, Rate Limiting, API Resources

## API Routes

```php
// routes/api.php (separate from web.php)
use App\Http\Controllers\Api\PostController;

Route::apiResource('posts', PostController::class);
```

**Register API routes in `bootstrap/app.php`:**
```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    api: __DIR__.'/../routes/api.php',  // API routes here
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
)
```

## API Resource Classes

```bash
php artisan make:resource PostResource
php artisan make:resource PostCollection
```

**Resource (single item):**
```php
// app/Http/Resources/PostResource.php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'body' => $this->body,
            'author' => new UserResource($this->whenLoaded('author')),
            'created_at' => $this->created_at->toIso8601String(),
            'updated_at' => $this->updated_at->toIso8601String(),
        ];
    }
}
```

**Collection:**
```php
// app/Http/Resources/PostCollection.php
class PostCollection extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection->map(fn($post) => [
                'id' => $post->id,
                'title' => $post->title,
            ]),
            'meta' => [
                'total' => $this->total(),
                'per_page' => $this->perPage(),
            ],
        ];
    }
}
```

**Controller response:**
```php
public function index()
{
    $posts = Post::with('author')->paginate(20);
    return PostResource::collection($posts);
}

public function show(Post $post)
{
    return new PostResource($post->load('author'));
}

## API Resource Advanced Patterns

Once you have the basic `JsonResource` shape down, four families of methods cover ~90% of the production API patterns: **conditional attributes**, **response-level metadata**, **data wrapping**, and **response hooks**. AI models default to verbose `response()->json([...])` calls when these helpers do the work in one line and survive collection wrapping, caching, and Octane — so this section exists mostly to make sure they get used.

### Conditional Attributes — `when()`, `whenAppends()`, `whenCounted()`, `whenPivotLoaded()`

```php
use App\Http\Resources\UserResource;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'         => $this->id,
            'title'      => $this->title,

            // Field appears only when truthy (e.g., user is admin)
            'draft_body' => $this->when($request->user()?->isAdmin, fn () => $this->draft_body),

            // Field appears only when the user was authenticated (check + value)
            'viewer_pinned' => $this->when($request->user(), fn ($u) => $this->pinnedFor($u)),

            // Field appears only when the relationship was eager-loaded (avoids N+1 in resources)
            'author'     => UserResource::make($this->whenLoaded('author')),
            'tags_count' => $this->whenCounted('tags'),           // needs ->withCount('tags')
            'tags'       => TagResource::collection($this->whenLoaded('tags')),

            // Pivot-only fields (BelongsToMany / HasManyThrough / etc.)
            'joined_at'  => $this->whenPivotLoaded('group_user', fn () => $this->pivot->joined_at),
            'role'       => $this->whenPivotLoadedAs('group_user', 'subscription', fn () => $this->pivot->role),
        ];
    }
}
```

**`mergeWhen()` / `merge()`** — merge multiple fields at once without flattening into the parent shape:

```php
public function toArray(Request $request): array
{
    return [
        'id' => $this->id,
        $this->mergeWhen($request->user()?->isAdmin, [
            'internal_notes'   => $this->internal_notes,
            'moderation_flags' => $this->moderation_flags,
        ]),
        $this->merge([
            // Always present, computed at runtime
            'comment_count' => $this->comments()->count(),
        ]),
    ];
}
```

> **Pitfall — `when()` evaluates the condition at JSON-serialization time**, not at controller time. For expensive conditions (DB queries, API calls), compute the value once in the controller and pass it via the resource constructor or `additional()` metadata instead.

### Response-Level Metadata — `additional()` / `with()`

For collection-level metadata (totals, generated-at timestamps, server info), use `additional()` rather than hand-rolling the wrapper:

```php
// Controller
public function index()
{
    $posts = Post::with('author')->paginate(20);

    return PostResource::collection($posts)
        ->additional([
            'meta' => [
                'generated_at' => now()->toIso8601String(),
                'server_id'    => gethostname(),
                'request_id'   => request()->header('X-Request-Id'),
            ],
        ]);
}
```

**Response:**
```json
{
  "data": [ ... ],
  "links": { "first": "...", "last": "...", "prev": null, "next": "..." },
  "meta": {
    "current_page": 1,
    "per_page": 20,
    "total": 142,
    "generated_at": "2026-07-06T18:00:00+00:00",
    "server_id": "web-01",
    "request_id": "abc-123"
  }
}
```

> **Nested `meta` vs flat `meta`** — Laravel merges `additional()` directly into the response root. Wrap your keys (`'meta' => [...]`) explicitly if you want a nested meta block — otherwise you pollute the top level.

### Data Wrapping — `wrap()` / `withoutWrapping()`

By default Laravel wraps the outermost resource in `{"data": ...}`. Override per-resource, per-collection, or globally:

```php
// 1. Per-resource
class PostResource extends JsonResource
{
    // Default: 'data'. Use 'post' for single-resource endpoints that should NOT use 'data'
    public static $wrap = 'post';
}

// 2. Per-collection (the wrapper key for a collection response)
class PostCollection extends JsonResource
{
    public static $wrap = 'posts';
}

// 3. Globally — kill the wrapper for ALL resources in this app
// app/Providers/AppServiceProvider.php
public function boot(): void
{
    JsonResource::withoutWrapping();   // applies to outermost response only
}

// 4. Per-controller, single call
return PostResource::make($post)->withoutWrapping();   // {'id': 1, 'title': '...'} (no data wrapper)
```

**Gotchas:**
- `JsonResource::withoutWrapping()` only affects the **outermost** response. Inner resource collections in a tree still wrap themselves.
- **Paginated collections always re-add `data`** even when `withoutWrapping()` is set — Laravel needs somewhere to put the items so pagination links can sit alongside. Strip with a custom resource collection if you must.
- **Octane + `withoutWrapping()` is global**: the underlying property is static. Once set in one request, every subsequent request in the same Octane worker inherits it (laravel/framework#42777). Don't conditionally flip it per-request; use `additional()` + a custom response macro if you need mixed wrapping.

### Response Hooks — `withResponse()` (HATEOAS, Custom Headers, ETags)

The `withResponse()` method fires when the resource is the **outermost** resource in a response. Use it for self/related links (HATEOAS), cache headers, ETag generation, or rate-limit headers:

```php
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'    => $this->id,
            'title' => $this->title,
            'body'  => $this->body,
        ];
    }

    public function withResponse(Request $request, \Illuminate\Http\JsonResponse $response): void
    {
        $response->header('X-Resource-Id', (string) $this->id);

        // HATEOAS self + related links
        $response->setData($response->getData(true) + [
            'links' => [
                'self'    => route('api.posts.show', $this->id),
                'author'  => route('api.users.show', $this->author_id),
                'comments' => route('api.posts.comments.index', $this->id),
            ],
        ]);

        // ETag for client-side caching (returns 304 Not Modified on If-None-Match)
        $etag = md5($response->getContent());
        $response->setEtag($etag);
        $response->headers->set('Cache-Control', 'private, must-revalidate');
    }
}
```

**For collections** the hook runs once on the outermost `JsonResponse`, not per-item — perfect for collection-level ETag (one hash for the whole page).

**Why this beats `response()->json([...])`** — keeps the transformation colocated with the resource, survives Octane/FrankenPHP re-use, works inside `JsonResource::collection()` and `JsonApiResource`, and lets you unit-test the headers via `assertJsonPath('links.self', ...)`.

### Resource Collections — `JsonResource::collection()` vs Dedicated `ResourceCollection` Class

```php
// 1. Inline collection (most cases)
return PostResource::collection(Post::paginate(20));

// 2. Dedicated collection class — needed when you want:
//    - Custom $wrap key ('posts' instead of 'data')
//    - Custom meta in the response
//    - Override toArray() to reshape items beyond the per-resource pattern
class PostCollection extends JsonResource
{
    public static $wrap = 'posts';

    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'meta' => [
                'totals' => [
                    'published' => Post::where('status', 'published')->count(),
                    'drafts'    => Post::where('status', 'draft')->count(),
                ],
            ],
        ];
    }
}

// In the controller:
return new PostCollection(Post::paginate(20));
```

**Decision tree:**
- **Inline `PostResource::collection()`** — default, covers 80% of cases
- **Dedicated `PostCollection` class** — when you need collection-only meta, a non-`data` wrap key, or to nest the collection under a key (`'posts'`, `'results'`, etc.)
- **Custom `toArray()`** with `'data' => $this->collection` (the pattern shown above) — when the per-item resource class doesn't have the shape you want and you'd rather reshape inline

### Resource Link Generation (`route()` Inside Resources)

`route()` works inside `toArray()` and `withResponse()` but **stale-URLs in cached responses** are a known pitfall — `route()` reads the URL state at request time. If you cache the resource output:

```php
// BAD — cached response returns stale URLs after a host change
'url' => route('api.posts.show', $this->id),

// GOOD — store the route name + params, resolve at serialization time
'url' => [
    'name'   => 'api.posts.show',
    'params' => $this->id,
],

// Or, in withResponse() (fires at response time, after cache read):
public function withResponse(Request $request, JsonResponse $response): void
{
    $data = $response->getData(true);
    $data['url'] = route('api.posts.show', $this->id);
    $response->setData($data);
}
```

### When to Pick Which API Resource Family

| Need | Use |
|------|-----|
| Internal/admin/BFF API where you control both ends | `JsonResource` + `additional()` / `withResponse()` |
| Public mobile/3rd-party API, want spec compliance | `JsonApiResource` (Laravel 13 first-party) |
| Flat (non-wrapped) single-resource response | `JsonResource::make($x)->withoutWrapping()` |
| Collection with custom wrapper (`'posts'`, `'results'`) | Dedicated `PostCollection extends JsonResource` with `$wrap` |
| HATEOAS-style self/related links | `withResponse()` hook |
| Sparse fieldsets / `include` query params | `JsonApiResource` + `JsonApiQueryParameters` |
| Conditional attributes per request | `when()`, `mergeWhen()`, `whenLoaded()`, `whenCounted()` |
```

## JSON:API Resources (Laravel 13 — Spec-Compliant APIs)

Laravel 13 ships first-party `JsonApiResource` for building APIs that comply with the [JSON:API specification](https://jsonapi.org/). Clients (mobile apps, third-party consumers) get a predictable, well-documented response structure without you having to hand-roll it.

```bash
php artisan make:json-api-resource PostResource
```

**Resource (single item):**
```php
// app/JsonApi/Resources/PostResource.php
namespace App\JsonApi\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\JsonApiResources\JsonApiResource;

class PostResource extends JsonApiResource
{
    public function toAttributes(Request $request): array
    {
        return [
            'title' => $this->title,
            'body' => $this->body,
            'created_at' => $this->created_at->toIso8601String(),
            'updated_at' => $this->updated_at->toIso8601String(),
        ];
    }

    public function toRelationships(Request $request): array
    {
        return [
            'author' => fn() => UserResource::make($this->whenLoaded('author')),
            'comments' => fn() => CommentResource::collection($this->whenLoaded('comments')),
        ];
    }
}
```

**Relationship resource:**
```php
// app/JsonApi/Resources/UserResource.php
class UserResource extends JsonApiResource
{
    public function toAttributes(Request $request): array
    {
        return [
            'name' => $this->name,
            'email' => $this->email,
        ];
    }
}
```

**Controller — automatic JSON:API response structure:**
```php
public function show(Post $post): JsonApiResource
{
    return PostResource::make($post->load('author', 'comments'));
}

public function index(): JsonApiResource
{
    return PostResource::collection(Post::with('author')->paginate(20));
}
```

**Generated response structure (what clients get):**
```json
{
  "data": {
    "type": "posts",
    "id": "42",
    "attributes": {
      "title": "My Post",
      "body": "Post content...",
      "created_at": "2026-05-15T10:00:00+00:00"
    },
    "relationships": {
      "author": {
        "data": { "type": "users", "id": "1" }
      },
      "comments": {
        "data": [
          { "type": "comments", "id": "7" }
        ]
      }
    },
    "links": {
      "self": "https://api.example.com/posts/42"
    }
  },
  "included": [
    {
      "type": "users",
      "id": "1",
      "attributes": { "name": "Adarsh", "email": "adarsh@example.com" }
    }
  ]
}
```

**Sparse Fieldsets (client asks for only specific fields):**
```php
// Client requests: GET /posts?fields[posts]=title,body&fields[users]=name
// Only those fields appear in response — reduces payload size
```

**Filtering, Sorting, Pagination (use Laravel 13's JsonApiQueryParameters):**
```php
use Illuminate\Http\JsonApiQueryParameters;

public function index(JsonApiQueryParameters $params): JsonApiResource
{
    $query = Post::query();

    // $params->filter() — extract filters
    // $params->sort() — extract sort fields
    // $params->page() — extract pagination

    return PostResource::collection($query->paginate(20));
}
```

**When to use JsonApiResource vs JsonResource:**
- **JsonApiResource** — public APIs consumed by mobile apps, third-party developers, or any client that benefits from a standardized spec. The structure is rigid but predictable.
- **JsonResource** — internal APIs, admin panels, B2B integrations where you control both client and server and want full flexibility over the response shape.

## JsonApiResource — `toLinks()` for Self/Related Links (JSON:API Spec Compliance)

`JsonApiResource` (Laravel 13 first-party) auto-emits `links.self` for each resource, but JSON:API also expects `related`, `first`/`prev`/`next`/`last` pagination links, and custom domain links. Override `toLinks()`:

```php
use Illuminate\Http\Request;
use Illuminate\Http\JsonApiResources\JsonApiResource;

class PostResource extends JsonApiResource
{
    public function toAttributes(Request $request): array
    {
        return [
            'title' => $this->title,
            'body'  => $this->body,
        ];
    }

    public function toLinks(Request $request): array
    {
        return [
            'self'    => route('api.v1.posts.show', $this->id),
            'related' => [
                'author'   => route('api.v1.users.show', $this->author_id),
                'comments' => route('api.v1.posts.comments.index', $this->id),
            ],
        ];
    }

    public function toRelationships(Request $request): array
    {
        return [
            'author' => fn () => UserResource::make($this->whenLoaded('author')),
        ];
    }
}
```

**Generated response:**
```json
{
  "data": {
    "type": "posts",
    "id": "42",
    "attributes": { "title": "...", "body": "..." },
    "links": {
      "self": "https://api.example.com/v1/posts/42",
      "related": {
        "author": "https://api.example.com/v1/users/7",
        "comments": "https://api.example.com/v1/posts/42/comments"
      }
    },
    "relationships": { ... }
  }
}
```

**Pagination links in JsonApiResource collections** are added automatically by the framework based on the paginator's `url()` / `nextPageUrl()` / `previousPageUrl()` outputs — no extra wiring needed.

### Custom Resource Collection Pagination Meta (`toMeta()`)

For JSON:API spec compliance, collections also accept a `toMeta()` hook for non-Links metadata (totals, generated-at, etc.):

```php
class PostCollection extends JsonResource
{
    public function toMeta(Request $request): array
    {
        return [
            'generated_at' => now()->toIso8601String(),
            'request_id'   => $request->header('X-Request-Id'),
        ];
    }
}
```

## Pagination

```php
// Cursor pagination (faster for large datasets)
Post::cursorPaginate(20);

// Number pagination
Post::paginate(20);

// JSON response with meta
return response()->json([
    'data' => PostResource::collection($posts),
    'meta' => [
        'current_page' => $posts->currentPage(),
        'last_page' => $posts->lastPage(),
        'per_page' => $posts->perPage(),
        'total' => $posts->total(),
    ],
    'links' => [
        'first' => $posts->url(1),
        'last' => $posts->url($posts->lastPage()),
        'prev' => $posts->previousPageUrl(),
        'next' => $posts->nextPageUrl(),
    ],
], 200);
```

## Rate Limiting

```php
// routes/api.php
use Illuminate\Routing\Middleware\ThrottleRequests;

Route::middleware('throttle:60,1')->group(function () {
    // 60 requests per minute
    Route::get('/posts', [PostController::class, 'index']);
});

// Named throttle
Route::middleware('throttle:posts:100,2')->group(function () {
    // 100 per 2 minutes
});

// Custom rate limiter (config/retres.php)
use Illuminate\Cache\RateLimiting\RateLimiter;

RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(100)->by($request->user()?->id ?: $request->ip());
});

RateLimiter::for('posts', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip())
        ->response(fn() => response()->json(['error' => 'Slow down'], 429));
});
```

**Throttle response:**
```json
{"message": "Too many requests.", "retry_after": 60}
```

## ThrottlesExceptions with Closure (Laravel 13.9+)

`ThrottlesExceptions` middleware (for rate-limiting exception/thrown error responses) now accepts a Closure for dynamic limit configuration — useful for adjusting limits based on exception type or request context:

```php
use Illuminate\Routing\Middleware\ThrottlesExceptions;

// Static limit
Route::middleware(ThrottlesExceptions::class)->group(function () {
    Route::post('/webhook', [WebhookController::class, 'handle']);
});

// Dynamic limit via Closure (Laravel 13.9+) — determine limit at runtime
Route::middleware(ThrottlesExceptions::using(function (Request $request, \Throwable $e) {
    // Stricter limit for auth-related exceptions
    if ($e instanceof \Illuminate\Auth\AuthenticationException) {
        return 3; // only 3 attempts for auth errors
    }
    // More lenient for other exceptions
    return 10;
}))->group(function () {
    Route::post('/login', [AuthController::class, 'login']);
});
```

**Common use cases:**
- Tighter throttling on authentication endpoints to prevent brute-force
- Different limits per exception type (e.g., 3 for auth, 10 for general validation)
- Context-aware limits based on user role or request origin

## Request Helpers — `whenFilledEnum()` (Laravel 13.16+)

The `Request`/`InteractsWithData` trait gains `whenFilledEnum()`, which combines the filled-check, `tryFrom()` enum cast, and null guard into a single call. Saves a lot of boilerplate when filtering by typed enum values:

```php
use App\Enums\Status;
use Illuminate\Http\Request;

public function index(Request $request)
{
    return Post::query()
        ->whenFilledEnum('status', Status::class, function (Status $status, $query) {
            $query->where('status', $status);
        })
        ->get();
}
```

**Optional default callback** runs when the primary callback doesn't (e.g., invalid enum value, empty input):

```php
Post::query()
    ->whenFilledEnum('status', Status::class,
        fn(Status $s, $q) => $q->where('status', $s),
        fn($q) => $q->where('status', Status::Active), // default if no valid value
    )
    ->get();
```

**Behavior:**
- Callback runs only when: the key is filled AND the class is a backed enum AND the value casts to a valid case via `tryFrom()`
- Invalid enum values silently skip the callback — no exception thrown
- Perfect for filter endpoints that accept `?status=active`

## `withCookies()` on All Responses (Laravel 13.16+)

The `withCookies()` method moved from `RedirectResponse` to `ResponseTrait`, so you can attach multiple cookies to any response type in a single call — including `JsonResponse`:

```php
// Multiple cookies in one call (no chaining)
return response()->json($data)->withCookies([$cookieA, $cookieB, $cookieC]);

// Equivalent pre-13.16 — had to chain
return response()->json($data)
    ->withCookie($cookieA)
    ->withCookie($cookieB)
    ->withCookie($cookieC);
```

**Why it matters:** Previously, JSON/API endpoints had to either chain `withCookie()` or fall back to cookies-as-arrays on the response. `withCookies()` is additive (non-breaking) and dramatically cleans up multi-cookie scenarios like OAuth callback endpoints or auth responses.

## HTTP Client — `Http::query()` Non-Sending Method (Laravel 13.19+)

Laravel 13.19 adds `Http::query()` — a non-sending method on the HTTP client that builds a request and returns a `Request` object you can inspect, **without actually dispatching the call**. Pairs with the new `assertQuery()` / `assertQueryMissing()` / `assertQueryJson()` test helpers (covered in `testing.md`).

```php
use Illuminate\Support\Facades\Http;

// 13.19+: build a GET request, inspect it, do NOT send
$request = Http::query('GET', 'https://api.github.com/users/octocat', ['per_page' => 50]);

$request->url();             // https://api.github.com/users/octocat?per_page=50
$request->method();          // GET
$request->data();            // ['per_page' => 50]
$request->body();            // raw body (for POST/PUT)
$request->header('Accept');  // application/json (default)
$request->isJson();          // true

// 13.19+: chains with the existing HTTP client config
$request = Http::withToken($token)
    ->acceptJson()
    ->query('POST', 'https://api.stripe.com/v1/charges', ['amount' => 1000, 'currency' => 'usd']);

// In a unit test — no Http::fake() needed
$service = new StripeService();
$request = $service->buildChargeRequest(1000, 'usd');
$this->assertEquals('https://api.stripe.com/v1/charges', $request->url());
$this->assertEquals(1000, $request->data()['amount']);
```

**When to use `Http::query()` vs `Http::fake()`:**

| Use case | Method |
|---|---|
| Unit test: "did I build the right URL/params/headers?" | `Http::query()` (no send) |
| Integration test: "does the response handler work?" | `Http::fake()` (returns mock response) |
| Both (most common) | `Http::fake()` + `Http::assertSent()` callback that uses `assertQuery` helpers |

**For 13.18 and earlier:** Use `Http::fake()` and inspect the `Request` via the `assertSent` callback:

```php
// 13.18 and earlier — equivalent pattern, requires Http::fake()
Http::fake();
Http::get('https://api.example.com/test', ['a' => 1]);
Http::assertSent(fn($request) => $request->url() === 'https://api.example.com/test?a=1');
```

`Http::query()` removes the need to fake-and-dispatch just to inspect request construction. Most useful in unit tests where you don't care about the response — you just want to verify the request shape.


## Error Responses


```php
// Validation errors (422)
return response()->json([
    'message' => 'Validation failed',
    'errors' => [
        'email' => ['The email field is required.'],
        'password' => ['The password must be at least 8 characters.'],
    ],
], 422);

// Not found (404)
return response()->json(['message' => 'Post not found'], 404);

// Generic error (500)
return response()->json(['message' => 'Internal error'], 500);

// Custom error handler (app/Exceptions/Handler.php)
$this->renderable(function (\Illuminate\Validation\ValidationException $e, $request) {
    if ($request->expectsJson()) {
        return response()->json([
            'message' => 'Validation failed',
            'errors' => $e->errors(),
        ], 422);
    }
});
```

**`Request::json()` top-level zero fix (Laravel 13.18.0+) — PR #60614:** Before 13.18.0, a `PUT`/`PATCH` request with a literal `0` body was coerced to `[]` when read via `Request::json()` or `Request::all()`. PR #60614 (commit 1c0c8fb5, 2026-06-28) preserves the zero. If your code branches on `$request->json('value') === 0` for counter resets, idempotency keys, or feature flags, you can drop the `?? 0` workaround. Watch the edge case: numeric strings (`"0"`) still come through correctly on 13.18.0+ — the fix only affected numeric `0`.

## Route Metadata for OpenAPI Generation (Laravel 13.17+)

`Route::metadata()` is the natural building block for auto-generated OpenAPI / Stoplight specs — attach `summary`, `description`, `tags`, `operationId`, and `security` to each route, then walk `Route::getRoutes()` to emit spec fragments. Survives `route:cache` serialization, so production-built apps still emit specs from cache.

```php
// routes/api.php
Route::get('/posts', [PostController::class, 'index'])
    ->metadata([
        'openapi' => [
            'summary'     => 'List posts',
            'description' => 'Returns paginated posts. Supports filter via ?status=',
            'tags'        => ['Posts'],
            'operationId' => 'posts.index',
            'security'    => [['sanctum' => []]],
        ],
    ]);

Route::post('/posts', [PostController::class, 'store'])
    ->metadata(['openapi' => ['summary' => 'Create post', 'tags' => ['Posts'], 'operationId' => 'posts.store']]);

// Group-level metadata cascades to children — perfect for setting common tags/security
Route::prefix('v1')->metadata(['openapi' => ['servers' => [['url' => 'https://api.example.com/v1']]]])
    ->group(__DIR__.'/api-v1.php');
```

```php
// Emit OpenAPI spec from route metadata (sketch)
$openapi = ['openapi' => '3.1.0', 'paths' => []];
foreach (Route::getRoutes() as $route) {
    $meta = $route->getMetadata('openapi');
    if (! $meta) continue;
    $openapi['paths'][$route->uri()] ??= [];
    $openapi['paths'][$route->uri()][strtolower($route->methods()[0])] = $meta;
}
file_put_contents(public_path('openapi.json'), json_encode($openapi, JSON_PRETTY_PRINT));
```

**Why it is the right primitive:** previously you had to annotate routes via PHP attributes, custom comments parsed at runtime, or hardcode specs in YAML/JSON that drifts out of sync. Route metadata is colocated with the route, survives route caching, and can drive docs, audit dashboards, feature flags, and ownership reports from a single source of truth.

See `controllers.md` (Route Metadata section) for the full API + group-cascade behavior.

## CORS

```php
// bootstrap/app.php
->withMiddleware(function (Middleware $middleware) {
    $middleware->api(prepend: [
        \Illuminate\Http\Middleware\HandleCors::class,
    ]);
})

// config/cors.php
'paths' => ['api/*'],
'allowed_methods' => ['*'],
'allowed_origins' => ['https://your-frontend.com'],
'allowed_headers' => ['Content-Type', 'Authorization'],
'exposed_headers' => ['X-RateLimit-Remaining'],
'max_age' => 86400,
'supports_credentials' => false,
```

## `Number::forHumans` / `Number::abbreviate` / `Number::fileSize` DoS on Non-Finite Input (Laravel 13.18.0+)

If any API response field is rendered through `Number::forHumans()`, `Number::abbreviate()`, or `Number::fileSize()` and the value can be influenced by a query string, header, or upstream API, you have a remote-DoS hole on Laravel <13.18.0. Calling these methods with `INF`, `-INF`, or `NaN` recursed into `Number::summarize()` with no base case, exhausted PHP memory, and aborted the request (or the whole worker on a long-lived process). On PHP 8.5, `NaN` also triggered a `floor(log10(NAN))` -> int deprecation warning before the OOM.

13.18.0 (PR #60617, commit 3a3c1d2b, 2026-06-27; PR #60625, commit 7e9d4aa1, 2026-06-29) short-circuits to `Number::format()` and returns the locale-aware `INF` / `-INF` / `NaN` symbol.

```php
use Illuminate\Support\Number;

// In a resource — pre-13.18.0, ?views=1e400 crashes the request
public function toArray(Request $request): array {
    return [
        'views'  => Number::forHumans((float) $this->views_count),  // safe on 13.18.0+
        'size'   => Number::fileSize($this->attachment_bytes),      // safe on 13.18.0+
        'rate'   => Number::abbreviate($this->computeRate()),      // safe on 13.18.0+
    ];
}
```

**Why this matters for API code specifically:**
- `Request::json()` (PR #60614, also 13.17.1) now preserves top-level zero bodies; pair with input-shape validation to reject non-finite numeric fields up front rather than letting them reach a `Number::` helper.
- `Request::validate(['views' => 'numeric'])` does **not** reject `INF` or `NaN` — `is_numeric(INF)` returns `true` on PHP 8.5. Add a custom rule or pre-filter:

```php
// app/Rules/FiniteNumber.php
class FiniteNumber implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (! is_finite($value)) {
            $fail("The {$attribute} must be a finite number.");
        }
    }
}
```

- For JSON request bodies, parse and pre-validate *before* any `Number::` call. JSON parsers will happily surface `1e400` as `INF` in PHP.

**Audit checklist:**
- `grep -rn "Number::forHumans\|Number::abbreviate\|Number::fileSize" app/Http/Resources/ app/Http/Controllers/`
- For every hit, confirm the input path cannot be `INF` / `NaN` (database casts, int math, validated `numeric` input with the `FiniteNumber` rule).
- If you can't upgrade to 13.18.0 immediately, wrap the call sites:

```php
function safeNumber(mixed $value, callable $fn, string $fallback = 'N/A'): string {
    if (! is_finite($value ?? 0)) {
        return $fallback;
    }
    return $fn($value);
}

// In the resource:
'views' => safeNumber($this->views_count, fn($v) => Number::forHumans($v)),
```

**Cross-references:** full performance-side discussion in `performance.md` (Number INF/NaN OOM section). Related 13.18.0 fix: `Request::json()` now preserves top-level zero bodies (PR #60614) — see the Error Responses section below for the test pattern.

Source: [PR #60617 — Fix Number::forHumans and Number::abbreviate crashing on INF/NAN](https://github.com/laravel/framework/pull/60617) | [PR #60625 — Fix Number::fileSize wrong unit suffix for non-finite inputs](https://github.com/laravel/framework/pull/60625)

## Common Mistakes

1. **Returning arrays directly** — always wrap in `response()->json()` with proper status codes
2. **Not handling 404** — return 404 for missing resources, not empty array
3. **Rate limit not set** — API gets hammered, service degrades
4. **Wrong status codes** — 200 for success, 201 for created, 400 for bad request, 401 for unauthenticated, 403 for forbidden, 404 for not found, 422 for validation, 500 for server error
5. **No versioning** — `api/v1/` routes let you break changes without killing existing clients
6. **Exposing internal errors** — never return `exception->getMessage()` to API clients in production
7. **Mixing JsonApiResource and JsonResource** — pick one approach per API; mixing makes client code harder
8. **Missing `type` in JSON:API responses** — JsonApiResource handles this automatically; don't hand-roll responses that omit it
9. **Calling `route()` inside cached `toArray()`** — returns stale URLs after a host/domain change; move to `withResponse()` or store the route name + params and resolve at serialization time
10. **Setting `JsonResource::withoutWrapping()` per-request under Octane** — the underlying property is static; once flipped in one request, every subsequent request in the worker inherits it (laravel/framework#42777). Set globally in `AppServiceProvider::boot()` only.
11. **Building HATEOAS links in the controller instead of `withResponse()`** — keeps the transformation colocated with the resource and works under `JsonResource::collection()` / Octane / route caching
12. **Hand-rolling `meta` blocks** when `additional()` already injects them at the response level — saves the `response()->json([...])` wrap and integrates with pagination meta
13. **Using `whenLoaded('author')` outside a resource** — it's a JsonResource method, not a model method; call it inside `toArray()` where `$this` is the resource proxy
14. **Using `mergeWhen()` outside a resource** — same restriction; `mergeWhen()` only exists on the resource's `toArray()` context

## Updated from Research (2026-07-06, cycle 29)

- **API Resource Advanced Patterns** — `when()` / `whenAppends()` / `whenCounted()` / `whenPivotLoaded()` / `whenPivotLoadedAs()` for conditional attributes, `additional()` / `with()` for response-level metadata, `wrap()` / `withoutWrapping()` for data wrapper control, `withResponse()` for HATEOAS self/related links and custom headers/ETags. Decision tree for choosing between inline `JsonResource::collection()`, dedicated `ResourceCollection`, and `JsonApiResource`. Notes the Octane + `withoutWrapping()` static-property gotcha (laravel/framework#42777) and the `route()` + cached-resource staleness pitfall. Common Mistakes list grew 8 → 14 entries.
- **JsonApiResource `toLinks()` / `toMeta()`** — JSON:API spec compliance for `self` / `related` / pagination / domain-specific links, plus `toMeta()` for collection-level metadata (totals, generated_at, request_id). Pairs with the existing JSON:API Resources section above.

---

## Updated from Research (2026-06-26, cycle 5)

- **Route Metadata for OpenAPI Generation (Laravel 13.17+)** — `Route::metadata()` is the natural primitive for auto-generated OpenAPI / Stoplight specs. Attach `summary`, `description`, `tags`, `operationId`, `security` per route (survives `route:cache`). Group-level metadata cascades to children for shared `servers` / `securitySchemes`. Walk `Route::getRoutes()` and read `Route::getMetadata('openapi')` to emit spec fragments — replaces attribute-based or comment-based route annotation systems that drift out of sync. See the detailed section above.

---

## Updated from Research (2026-05-18)

### ThrottlesExceptions Closure Support (Laravel 13.9+)

- `ThrottlesExceptions` middleware now accepts a Closure as its limit parameter for dynamic, runtime-determined throttling limits
- Enables exception-type-aware throttling (e.g., stricter limits on auth exceptions)
- Use case: protect login endpoints from brute-force with per-exception-type limits

### JSON:API Resources (Laravel 13 — Major Feature)

- **First-party JSON:API spec support** — `JsonApiResource` and `JsonApiQueryParameters` ship in Laravel 13 core
- **Spec-compliant responses** — `data`, `attributes`, `relationships`, `included`, `links` all auto-generated
- **Sparse fieldsets** — clients request only specific fields, reducing payload
- **When to use** — public/mobile/third-party APIs benefit most; internal/admin APIs may prefer flexible `JsonResource`
- **Install command** — `php artisan make:json-api-resource PostResource`

Sources: [Laravel 13 Docs - JSON:API Resources](https://laravel.com/docs/13.x/eloquent-resources) | [Laravel 13 Docs - Rate Limiting](https://laravel.com/docs/13.x/rate-limiting)
