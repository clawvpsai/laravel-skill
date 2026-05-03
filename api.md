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