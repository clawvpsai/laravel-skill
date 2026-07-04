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

## Laravel 13 Starter Kits — Passkey Login Out of the Box

The Laravel 13 starter kits (React, Vue, Svelte, Livewire) ship with **passkey login UI pre-wired** when you choose the "Passkey" option during `laravel new`. Under the hood it's `laravel/passkeys` + Fortify wired into the auth controllers, so you get:

- Registration form with "Add passkey" button (calls `Passkeys::generateRegistrationOptions()`).
- Login form with "Sign in with passkey" button (calls `Passkeys::generateAuthenticationOptions()`).
- Automatic credential storage on `$user->passkeys()`.
- Fallback to password + email verification if no passkey is registered.

**To enable on an existing app (Laravel 13+):**
```bash
# 1. Install the passkeys package
composer require laravel/passkeys
php artisan passkeys:install

# 2. Enable in Fortify (config/fortify.php)
'features' => [
    Features::registration(),
    Features::resetPasswords(),
    Features::passkeyAuthentication(), // ← enables passkey login flow
    Features::passkeyRegistration(),  // ← enables in-app passkey enrollment
],

# 3. Front-end: render the passkey script where needed
@include('passkeys::script')
# or in non-Blade — see laravel/passkeys README for Vue/React components
```

**Laravel 13 `Features::passkeyAuthentication()` / `Features::passkeyRegistration()`** are first-party Fortify feature flags that wire the routes, controllers, and Blade components for you. You don't write a single line of WebAuthn code unless you need custom authenticator selection logic.

**When to roll your own vs use Fortify:**
- Use Fortify + starter kit if you want the standard "device-bound biometric" flow on first-party web UIs.
- Roll your own (using `laravel/passkeys` directly) if you need:
  - Cross-platform roaming authenticator support with custom UI flows.
  - Passkey-only login on a SPA without Fortify (call `Passkeys::generateAuthenticationOptions()` from your Vue/React component and POST the assertion back).
  - Custom challenge storage (e.g., Redis instead of session) for distributed / multi-node setups.

**Passkey recovery:** passkeys are device-bound. If the user loses their device and has no other passkey registered, they MUST fall back to password + email verification. **Always require at least one verified email + password as a recovery path** — don't ship passkey-only auth on a critical app without a fallback.

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

## Laravel 13 Policy Registration — `#[UsePolicy]` Attribute

The classic `Gate::policy(Model::class, PolicyClass::class)` registration in `AppServiceProvider` still works in Laravel 13, but there's now a **declarative, model-side** alternative — the `#[UsePolicy]` PHP attribute on the model class itself. Laravel reads the attribute via reflection and wires the policy automatically; no service-provider boot code required.

```php
<?php

namespace App\Models;

use App\Policies\OrderPolicy;
use Illuminate\Database\Eloquent\Attributes\UsePolicy;
use Illuminate\Database\Eloquent\Model;

#[UsePolicy(OrderPolicy::class)]
class Order extends Model
{
    //
}
```

**Why prefer `#[UsePolicy]` over `Gate::policy()` in Laravel 13:**

- **Colocation** — the policy is declared next to the model, so a code reviewer scanning `app/Models/Order.php` sees exactly which policy gates it. No need to grep `AppServiceProvider` to find out which policy class applies.
- **No boot-time race** — the registration is part of class metadata, so it works correctly in every request lifecycle (including Octane / FrankenPHP long-lived workers where `boot()` may run once per worker respawn but the model class is loaded per request).
- **No service provider bloat** — `AppServiceProvider` stays focused on app-level concerns (Gate::define closures, observers, Route::pattern global constraints) instead of accumulating policy registration for every model.

**What this does NOT change:**

- Policy method signatures (`view`, `viewAny`, `create`, `update`, `delete`, custom methods) are identical.
- The `Gate::authorize()`, `$this->authorize()`, and `Gate::forUser($user)->allows()` call sites are identical.
- The `#[Authorize('action', 'routeParam')]` controller attribute (covered below) still works without modification — it just resolves the policy via the same auto-discovery.

**Edge cases:**

- `#[UsePolicy]` lives at `Illuminate\Database\Eloquent\Attributes\UsePolicy`. Don't confuse it with the controller-side `#[Authorize]` attribute (`Illuminate\Routing\Attributes\Controllers\Authorize`) — same family of "declarative auth" attributes, but they live in different namespaces and serve different purposes.
- If you have BOTH `#[UsePolicy]` on the model AND `Gate::policy(Model::class, PolicyClass::class)` in `AppServiceProvider`, the explicit `Gate::policy()` call wins (it runs later in the boot cycle and overrides). Strip the manual registration to avoid the drift.
- For Laravel 12 and earlier, this attribute doesn't exist — keep using `Gate::policy()` for those versions. The attribute is Laravel 13+ only.

## Laravel 13 Controller Authorization Attributes — `#[Authorize]` + `#[Middleware]`

Laravel 13 adds PHP attributes that let you put policy authorization and middleware directly on the controller — replacing the `__construct()` middleware assignments and `$this->authorize()` / `Gate::authorize()` calls inside action methods.

```php
use Illuminate\Routing\Attributes\Controllers\Authorize;
use Illuminate\Routing\Attributes\Controllers\Middleware;

#[Middleware('auth:sanctum')]                              // class-level — all methods
class CommentController
{
    #[Authorize('create', [Comment::class, 'post'])]       // requires Post model in route + create permission
    public function store(Post $post) { /* ... */ }

    #[Authorize('delete', 'comment')]                       // comment is the route parameter name
    public function destroy(Comment $comment) { /* ... */ }

    #[Middleware('throttle:api')]                           // method-level middleware addition
    public function index() { /* ... */ }
}
```

**Two-argument `#[Authorize(ability, target)]` semantics:**

| Second argument | What happens | Example |
|---|---|---|
| Route parameter name (string) | Resolves the bound model from the route and passes it to the policy method | `#[Authorize('update', 'post')]` → `PostPolicy::update($user, $post)` |
| Model class (FQCN) | Policy method called WITHOUT a model instance (e.g. `create`) | `#[Authorize('create', Comment::class)]` |
| `[Class::class, 'routeParam']` tuple | Pass the class + a different route param to the policy (for actions that authorize one model based on another) | `#[Authorize('create', [Comment::class, 'post'])]` → `CommentPolicy::create($user, $post)` |

**Why use `#[Authorize]` over `$this->authorize()` in the method body:**

- **No "did I forget the auth check?" audit problem** — the attribute is on the controller method, declarative, and visible in code review at a glance. With `$this->authorize()` you have to read the first 3 lines of every action method.
- **Hard fail on missing policy** — if the policy method doesn't exist on the resolved policy class, the route throws at registration time (when the attribute is parsed), not at request time when the user is already authenticated.
- **Survives copy-paste** — if you scaffold a `ResourceController` and copy-paste methods, the auth attribute goes with the method. With `$this->authorize()` it's easy to paste a method and forget the call.

**What this does NOT replace:**

- Route-level middleware (`Route::middleware('auth:web')->group(...)`) — still useful for grouping many endpoints.
- Inline `$request->user()->can('do-thing')` checks — fine inside Blade views for showing/hiding UI affordances.
- `Gate::define()` for ad-hoc permissions that aren't tied to a model.

See `controllers.md` (Laravel 13 Controller Attributes section) for the full `#[Middleware]` / `#[Authorize]` usage patterns including `only:` / `except:` filters and middleware parameter syntax.

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


### Laravel 13 PHP Attribute Authorization (cycle 24, 2026-07-04)

- **`#[UsePolicy]` on Eloquent models** — Laravel 13 auto-discovers a model's policy via the `Illuminate\Database\Eloquent\Attributes\UsePolicy` PHP attribute. Lets you colocate policy registration with the model instead of centralizing all `Gate::policy()` calls in `AppServiceProvider`. Compatible with the existing `Gate::policy()` registration (explicit call wins on conflict).
- **`#[Authorize]` controller attribute** — Laravel 13's declarative policy check on controller methods. Signature: `#[Authorize('ability', $target)]` where `$target` is a route parameter name, a model class FQCN, or `[Class::class, 'routeParam']`. Pairs with the existing `#[Middleware]` attribute on controllers. See `controllers.md` (Laravel 13 Controller Attributes section) for full usage.

### Laravel 13 Starter Kit Passkey Integration

- Laravel 13 starter kits (React/Vue/Svelte/Livewire) ship passkey login UI pre-wired when you choose "Passkey" during `laravel new`. Uses `laravel/passkeys` + Fortify under the hood.
- Enable in existing apps via `Features::passkeyAuthentication()` and `Features::passkeyRegistration()` flags in `config/fortify.php` — no WebAuthn boilerplate to write.
- Always require email + password recovery path on critical apps — passkeys are device-bound.
