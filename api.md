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

## Common Mistakes

1. **Returning arrays directly** — always wrap in `response()->json()` with proper status codes
2. **Not handling 404** — return 404 for missing resources, not empty array
3. **Rate limit not set** — API gets hammered, service degrades
4. **Wrong status codes** — 200 for success, 201 for created, 400 for bad request, 401 for unauthenticated, 403 for forbidden, 404 for not found, 422 for validation, 500 for server error
5. **No versioning** — `api/v1/` routes let you break changes without killing existing clients
6. **Exposing internal errors** — never return `exception->getMessage()` to API clients in production
7. **Mixing JsonApiResource and JsonResource** — pick one approach per API; mixing makes client code harder
8. **Missing `type` in JSON:API responses** — JsonApiResource handles this automatically; don't hand-roll responses that omit it

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
