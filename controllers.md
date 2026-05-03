# Controllers & Routing

## Resourceful Routes

```bash
# Generates: index, create, store, show, edit, update, destroy
php artisan make:controller PostController --model=Post --requests

# Plain resource (no create/edit)
php artisan make:controller PostController --model=Post --api
```

**Routes file (`routes/web.php`):**
```php
Route::resource('posts', PostController::class);

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

**Middleware order (App\Http\Kernel):**
```
0. Handle Request (earliest)
1. \Illuminate\Http\Middleware\HandleCors
2. \Illuminate\Foundation\Http\Middleware\ValidatePostSize
3. TrimStrings / ConvertEmptyStringsToNull
4. Your custom global middleware
5. \Illuminate\Session\Middleware\StartSession
6. \Illuminate\View\Middleware\ShareErrorsFromSession
7. \Illuminate\Routing\Middleware\SubstituteBindings
8. Your route middleware
9. \Illuminate\Auth\Middleware\Authenticate (if auth required)
10. Controller executes
11. Response goes back OUT through the stack
```

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