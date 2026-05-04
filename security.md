# Security — XSS, SQL Injection, Hardening

## XSS Prevention

```php
// Blade auto-escapes {{ }} — always use this for user content
{{ $user->comment }}

// For raw HTML (only when content is pre-sanitized):
{!! \Illuminate\Support\HtmlString($sanitizedHtml) !!}
// or
{!! clean($user->htmlContent) !!} // via HTMLPurifier package

// Never:
{!! $user->bio !!} // bio could contain <script> tags
```

**Sanitization:**
```bash
composer require mews/purifier
```

```php
// app/Helpers/sanitize.php
function cleanHtml(string $html): string
{
    return \Mews\Purifier\Facades\Purifier::clean($html);
}

// Use on input
$post->html_content = cleanHtml($request->input('html_content'));
```

## SQL Injection Prevention

```php
// SAFE — parameterized queries
User::where('email', $email)->first();
User::whereIn('id', [1, 2, 3])->get();

// SAFE — select + bindings
DB::select('SELECT * FROM users WHERE email = ?', [$email]);

// SAFE — query builder with bindings
User::whereRaw('email = ?', [$email])->first();

// DANGEROUS — string interpolation (never do this)
User::whereRaw("email = '$email'")->first(); // SQL injection
DB::raw("SELECT * FROM users WHERE email = '$email'"); // SQL injection

// DANGEROUS — direct interpolation even in select
DB::select("SELECT * FROM users WHERE email = '$email'"); // SQL injection
```

**Always use query builder bindings, never string interpolation.**

## Mass Assignment Protection

```php
// Always define fillable
protected $fillable = ['title', 'body', 'author_id'];
protected $guarded = []; // blocks all except fillable

// Never leave unguarded with user-submitted data
protected $guarded = []; // WRONG — allows all fields
protected $fillable = []; // WRONG — blocks all fields
```

## Password Security

```php
// Hashing
$hash = Hash::make($password);

// Verification
if (!Hash::check($password, $hash)) {
    abort(401, 'Invalid credentials');
}

// Bcrypt settings (already good by default)
Hash::make($password, ['rounds' => 12]); // override if needed
```

## Brute Force & Rate Limiting on Auth Endpoints

Laravel 13 Sanctum supports per-second rate limiting. Protect login/register/password-reset endpoints aggressively:

```php
// config/retres.php — define rate limiters
use Illuminate\Cache\RateLimiting\RateLimiter;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

RateLimiter::for('auth', function (Request $request) {
    return Limit::perMinute(10)->by($request->ip()); // 10 attempts/minute
});

RateLimiter::for('auth-otp', function (Request $request) {
    return Limit::perMinute(5)->by($request->ip()); // OTP requests
});

RateLimiter::for('api', function (Request $request) {
    return Limit::perSecond(60)->by($request->user()?->id ?: $request->ip());
});
```

```php
// routes/api.php or routes/web.php
Route::middleware('throttle:auth')->group(function () {
    Route::post('/login', [AuthController::class, 'login']);
    Route::post('/register', [AuthController::class, 'register']);
    Route::post('/password/email', [ForgotPasswordController::class, 'sendResetLinkEmail']);
    Route::post('/password/reset', [ResetPasswordController::class, 'reset']);
});

// OTP verification — stricter
Route::middleware('throttle:auth-otp')->group(function () {
    Route::post('/verify-otp', [AuthController::class, 'verifyOtp']);
    Route::post('/resend-otp', [AuthController::class, 'resendOtp']);
});
```

**Custom rate limit response:**
```php
RateLimiter::for('auth', function (Request $request) {
    return Limit::perMinute(10)
        ->by($request->ip())
        ->response(fn() => response()->json([
            'message' => 'Too many attempts. Try again in 60 seconds.',
            'retry_after' => 60,
        ], 429));
});
```

**Track failed login attempts:**
```php
// AppServiceProvider boot()
RateLimiter::for('login', function (Request $request) {
    $email = (string) $request->email;
    
    return Limit::perMinute(5)
        ->by($email . '|' . $request->ip())
        ->response(fn() => response()->json([
            'message' => 'Too many login attempts. Please try again later.',
        ], 429));
});
```

## File Upload Security

```php
// Validate file type and size
$request->validate([
    'avatar' => 'required|image|mimes:jpeg,png,gif,webp|max:2048',
]);

// Store with unique name — never trust user input for filename
$path = $request->file('avatar')->store('avatars', 'public');

// If you need the stored path:
$filename = $request->file('avatar')->hashName(); // cryptographically secure filename
$path = $request->file('avatar')->storeAs('avatars', $filename, 'public');
```

**Prevent PHP file uploads (MIME type validation):**
```php
// Force extension check — prevents double extension attacks (evil.jpg.php)
$request->validate([
    'file' => 'required|file|mimes:jpg,jpeg,png,gif,webp|max:5120',
]);
```

## Command Injection

```php
// NEVER pass user input to shell commands
shell_exec("git log --author={$userInput}"); // DANGEROUS
exec("ls {$directory}"); // DANGEROUS

// SAFE — use Symfony Console for git operations
use Symfony\Component\Process\Process;
$process = new Process(['git', 'log', '--author=' . $userInput]);
$process->run();

// Or validate input strictly before any shell call
if (!preg_match('/^[a-zA-Z0-9_.\s-]{1,100}$/', $userInput)) {
    abort(400, 'Invalid characters');
}
```

## HTTPS & Security Headers

```php
// Always force HTTPS in production (AppServiceProvider)
if (app()->environment('production')) {
    URL::forceScheme('https');
}
```

**Security headers via `bootstrap/app.php`:**
```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->append(function ($request, $next) {
        $response = $next($request);
        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('X-Frame-Options', 'DENY');
        $response->headers->set('X-XSS-Protection', '1; mode=block');
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
        $response->headers->set('Permissions-Policy', 'camera=(), microphone=(), geolocation=()');
        return $response;
    });
});
```

## CSRF Protection (Laravel 13 Enhanced)

Laravel 13 formalizes CSRF as `PreventRequestForgery` middleware with **origin-aware request verification** while preserving token-based CSRF compatibility:

```php
// The middleware automatically validates Origin/Referer headers
// Ensure config/app.php has correct APP_URL configured

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
Route::post('/webhook', [WebhookController::class, 'handle'])
    ->withoutMiddleware([\Illuminate\Foundation\Http\Middleware\PreventRequestForgery::class]);
```

**Laravel 13 CSRF origin validation — key config:**
```php
// config/app.php — must match the actual app URL for origin checking to work
'url' => env('APP_URL', 'https://yourapp.com'),
```

## API Authentication

```php
// Use Sanctum for token auth — never roll your own
// Sanctum handles token storage, expiration, revocation

// If you must use API keys:
$apiKey = $request->header('X-API-Key');
$user = User::where('api_key', hash('sha256', $apiKey))->first();
```

## Environment Security

```php
// .env never in version control
.env
.env.local
.env.production

// .gitignore must include:
.env
.env.*
*.key
storage/*.key
```

## Session & Cookie Security

```php
// config/session.php — production settings
'driver' => 'encrypted-cookies',
'same_site' => 'lax', // strict = only same-site, lax = cross-site GET
'secure' => true, // HTTPS only in production
'http_only' => true, // JS can't read session cookie
'encrypt' => true, // encrypt entire cookie
```

## Content Security Policy (CSP)

```php
// bootstrap/app.php — add CSP header
->withMiddleware(function (Middleware $middleware) {
    $middleware->append(function ($request, $next) {
        $response = $next($request);
        $response->headers->set(
            'Content-Security-Policy',
            "default-src 'self'; script-src 'self' 'nonce-" . $request->session()->token() . "'"
        );
        return $response;
    });
});
```

## Common Mistakes

1. **`{!! !!}` with user content** — XSS vulnerability, always sanitize first
2. **String interpolation in queries** — SQL injection, always use bindings
3. **No $fillable/$guarded** — mass assignment vulnerability
4. **No rate limiting on login** — brute force attacks succeed
5. **User input in shell commands** — command injection, validate or use Symfony Process
6. **Plaintext passwords** — always `Hash::make()`
7. **Exposing stack traces in production** — `APP_DEBUG=false` in production
8. **APP_URL misconfigured** — breaks CSRF origin validation and redirects
9. **SameSite=null on session cookies** — enables CSRF from malicious links
10. **File uploads without mimes validation** — PHP files executable on server


## Updated from Research (2026-05-04)

### Laravel 13 CSRF Enhancement

Laravel 13 formalizes CSRF protection as `PreventRequestForgery` middleware with **origin-aware request verification** while preserving token-based CSRF compatibility. The middleware validates Origin/Referer headers against `config/app.php`'s `url` setting.

### Brute Force Protection

Laravel 13's per-second rate limiting via `RateLimiter::for()` enables granular brute-force protection on auth endpoints. Key patterns:
- Login/OTP endpoints: 5-10 attempts/minute per IP
- Account lockout after failed attempts
- Custom JSON response for rate limit hits (no redirect to error page)

### Content Security Policy

Laravel 13 supports CSP headers via middleware. Nonce-based CSP is the modern approach for allowing inline scripts safely.

Source: [Laravel 13 Docs - CSRF](https://laravel.com/docs/13.x/csrf) | [Laravel 13 Docs - Rate Limiting](https://laravel.com/docs/13.x/rate-limiting)