# Authentication — Sanctum, Policies, Gates, Passkeys

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

**Token creation with abilities (scopes):**
```php
// Create token with abilities
$token = $user->createToken('api-token', ['posts:read', 'posts:write']);

// Use in request
// Authorization: Bearer <token>

// Revoke
$user->tokens()->delete(); // all tokens
$user->tokens()->where('id', $tokenId)->delete(); // specific token

// Check abilities in request
if ($request->user()->tokenCan('posts:write')) {
    // user can write posts
}
```

**SPA (cookie-based, no token storage):**
```php
// routes/api.php — uses session cookie, not Bearer token
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/user', fn() => auth()->user());
});
```

## Laravel 13 Token Abilities (Enhanced)

Laravel 13 Sanctum enhances token abilities with richer semantics:

```php
// Create scoped token with fine-grained abilities
$token = $user->createToken('editor', [
    'posts:read',
    'posts:create',
    'posts:update',
    'comments:moderate',
]);

// Middleware can check abilities
Route::middleware('auth:sanctum')->group(function () {
    Route::post('/posts', fn() => /* create post */)
        ->middleware('ability:posts:create');
    
    Route::put('/posts/{post}', fn() => /* update post */)
        ->middleware('ability:posts:update');
});
```

## Passkeys (Laravel 13 — Passwordless WebAuthn)

Passkeys provide phishing-resistant, passwordless authentication using the WebAuthn standard. Users register a device (laptop biometric, phone) as a passkey and use it to sign in without passwords.

```bash
# Install the passkeys package
composer require laravel/passkeys
php artisan passkeys:install
```

**Register a passkey for a user:**
```php
use Laravel\Passkeys\Facades\Passkeys;

// Generate registration options (call from your registration/ settings page)
$options = Passkeys::generateRegistrationOptions(
    user: $user,
    authenticatorAttachment: 'platform', // platform = device-bound (phone/laptop), cross-platform = roaming key
);

// Store options temporarily and redirect user to browser's passkey prompt
session(['passkey_challenge' => $options->challenge]);
return response()->json(['options' => $options]);
```

**Verify and store passkey:**
```php
use Laravel\Passkeys\Facades\Passkeys;

$passkeys = Passkeys::retrieveAuthenticatorAttestationResponse(request());
$credential = Passkeys::createCredentialFromAttestation(
    $user,
    $passkeys,
    request()->session()->pull('passkey_challenge'),
);

// Store credential for future logins
$user->passkeys()->create([
    'credential_id' => $credential->credentialId,
    'public_key' => $credential->publicKey,
    'device_type' => $credential->deviceType,
    'counter' => $credential->counter,
]);
```

**Authenticate with passkey:**
```php
// Generate authentication options (from login page)
$options = Passkeys::generateAuthenticationOptions(
    allowCredentials: $user->passkeys()->get()->map(fn($pk) => [
        'id' => $pk->credential_id,
        'type' => 'public-key',
    ])->toArray(),
);
session(['passkey_auth_challenge' => $options->challenge]);
return response()->json(['options' => $options]);

// Verify authentication response
$passkeys = Passkeys::retrieveAssertionResponse(request());
$credential = Passkeys::authenticateCredential(
    $passkeys,
    $user->passkeys()->get()->map(fn($pk) => [
        'id' => $pk->credential_id,
        'publicKey' => $pk->public_key,
        'counter' => $pk->counter,
    ])->toArray(),
    request()->session()->pull('passkey_auth_challenge'),
);

// Login successful — credential->counter updated
$user->passkeys()->where('credential_id', $credential->credentialId)
    ->update(['counter' => $credential->counter]);
```

**Passkey routes (auto-registered by `passkeys:install`):**
- `GET /passkeys` — list user's passkeys
- `POST /passkeys` — register new passkey
- `DELETE /passkeys/{id}` — remove passkey

**Middleware for passkey-only routes:**
```php
Route::middleware('passkey')->group(function () {
    Route::get('/account/security', fn() => view('account.security'));
    Route::delete('/passkeys/{id}', fn() => /* delete */);
});
```

**Key Passkey concepts:**
- **Registration** — user proves they control a device; stores credential public key server-side
- **Authentication** — user proves possession of the private key via biometric/PIN; server verifies assertion signature
- **Credential ID** — unique identifier for a passkey on a specific device
- **Counter** — replay attack protection; must increment on each authentication
- **Platform authenticator** — device-bound (Touch ID, Windows Hello, phone unlock)
- **Cross-platform authenticenticator** — roaming key (security key, phone as FIDO2 key)

## Policies

```bash
php artisan make:policy PostPolicy --model=Post
```

**Register in `AppServiceProvider` (Laravel 11+):**
```php
use App\Models\Post;
use App\Policies\PostPolicy;
use Illuminate\Support\Facades\Gate;

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
// In AppServiceProvider boot() (Laravel 11+)
Gate::define('manage-settings', function (User $user) {
    return $user->isAdmin();
});

Gate::define('access-reports', fn(User $user) => $user->role === 'analyst');

// Use
if (Gate::allows('manage-settings')) { /* show admin UI */ }
if (Gate::forUser($user)->denies('access-reports')) { /* hide reports */ }
```

## CSRF Protection (Laravel 13 Enhanced)

Laravel 13 formalizes CSRF as `PreventRequestForgery` middleware with origin-aware verification:

```php
// The middleware automatically validates Origin/Referer headers
// Ensure config/app.php has correct URL configured

// Form (Blade) — standard CSRF token
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
    ->withoutMiddleware([\Illuminate\Foundation\Http\Middleware\PreventRequestForgery::class]);
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
4. **Plaintext passwords stored** — always `Hash::make()`
5. **Session cookie not encrypted** — sensitive data visible in browser DevTools
6. **SameSite=nil** — allows CSRF from malicious links on other sites
7. **Passkey counter not updated** — replay attack vulnerability if counter not incremented on auth
8. **Passkey challenge not cleared after use** — challenges are single-use; always remove from session after verification

## Updated from Research (2026-05)

- **Laravel 13 Token Abilities** — Sanctum tokens support fine-grained abilities that map to `tokenCan()` checks and `ability:` middleware for API authorization.
- **Laravel 13 CSRF** — CSRF middleware is now formalized as `PreventRequestForgery` with enhanced origin-aware request verification.
- **Passkeys (WebAuthn)** — Laravel 13 introduces first-party passkey support via `laravel/passkeys`. Passwordless, phishing-resistant authentication using asymmetric cryptography.

Source: [Laravel 13 Docs - Sanctum](https://laravel.com/docs/13.x/sanctum) | [Laravel Passkeys](https://laravel.com/docs/13.x/passkeys) | [CSRF](https://laravel.com/docs/13.x/csrf)