# Authentication — Sanctum, Policies, Gates

## Sanctum (Laravel's API auth system)

```bash
composer require laravel/sanctum
php artisan install:api
```

**Setup:**
```php
// User model
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, Notifiable;
    
    public function tokens(): BelongsToMany
    {
        return $this->belongsToMany(Token::class);
    }
}
```

**Token creation:**
```php
// Create token with abilities (scopes)
$token = $user->createToken('api-token', ['posts:read', 'posts:write']);

// Use in request
// Authorization: Bearer <token>

// Revoke
$user->tokens()->delete(); // all tokens
$user->tokens()->where('id', $tokenId)->delete(); // specific token
```

**Middleware:**
```php
// routes/api.php
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/posts', fn() => Post::all());
    Route::post('/posts', fn() => Post::create(request()->all()));
});

// SPA (cookie-based, no token storage)
Route::middleware('auth:sanctum')->group(function () {
    // Uses session cookie, not Bearer token
});
```

## Policies

```bash
php artisan make:policy PostPolicy --model=Post
```

**Register in `AuthServiceProvider`:**
```php
protected $policies = [
    Post::class => PostPolicy::class,
];

// Or auto-discover
Gate::policy(Post::class, PostPolicy::class);
```

**Policy methods:**
```php
class PostPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, Post $post): bool { return true; }
    public function create(User $user): bool { return $user->can('create posts'); }
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->author_id || $user->isAdmin();
    }
    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->author_id || $user->isAdmin();
    }
}
```

**Use in Controller:**
```php
// Automatic policy resolution via route model binding
public function update(Request $request, Post $post)
{
    $this->authorize('update', $post); // throws 403 if not allowed
    // or
    Gate::authorize('update', $post);
}

// Manual
if (Gate::denies('update', $post)) {
    abort(403);
}
```

## Gates

```php
// AuthServiceProvider boot()
Gate::define('manage-settings', function (User $user) {
    return $user->isAdmin();
});

Gate::define('access-reports', fn(User $user) => $user->role === 'analyst');

// Use
if (Gate::allows('manage-settings')) { /* show admin UI */ }
if (Gate::forUser($user)->denies('access-reports')) { /* hide reports */ }
```

## CSRF Protection

Laravel auto-generates CSRF tokens for all session-based requests.

**Form (Blade):**
```html
<form method="POST" action="/posts">
    @csrf
    @method('PUT')
    ...
</form>
```

**JS fetch with CSRF:**
```js
// Add to meta tag
<meta name="csrf-token" content="{{ csrf_token() }}">

// Fetch
fetch('/posts', {
    method: 'POST',
    headers: {
        'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content,
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({ title: 'Hello' }),
});
```

**Exempt from CSRF** (only for external webhooks, never for your own forms):
```php
// Route
Route::post('/webhook', [WebhookController::class, 'handle'])
    ->withoutMiddleware([\Illuminate\Foundation\Http\Middleware\VerifyCsrfToken::class]);
```

## Session & Cookie Security

```php
// Config: config/session.php
'driver' => 'encrypted-cookies', // encrypts cookie content
'same_site' => 'lax', // strict = only same-site, lax = cross-site GET
'secure' => true, // only over HTTPS (set to false in local dev)

'http_only' => true, // JS can't read session cookie
'encrypt' => true, // encrypt entire cookie
```

## Password Hashing

```php
// Hash
$hash = Hash::make($password); // bcrypt
Hash::check($password, $hash); // verify — returns bool

// Custom bcrypt cost (if needed)
Hash::make($password, ['rounds' => 12]);
```

## Two-Factor Authentication

Laravel Fortify provides 2FA out of the box. Use it.

```bash
composer require laravel/fortify
php artisan fortify:install
```

## Common Mistakes

1. **Token abilities not checked** — always validate `$request->user()->tokenCan('action')`
2. **Policy not registered** — Gate doesn't auto-discover unregistered policies
3. **CSRF exemption for internal routes** — makes you vulnerable to CSRF attacks
4. **Plaintext passwords stored** — always use `Hash::make()`
5. **Session cookie not encrypted** — sensitive data visible in browser DevTools
6. **SameSite=nil** — allows CSRF from malicious links on other sites