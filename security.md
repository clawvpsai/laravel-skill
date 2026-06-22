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

## Security Fixes from Laravel 13.15+

Two notable security hardenings landed in Laravel 13.15 (June 2026). Apps on Laravel 11/12/13 prior to 13.15 should plan an upgrade.

### 1. `date_equals` Validation Bypass (CVE-class)

The `date_equals` validation rule used loose comparison (`==`). Invalid date strings parse to `null`, and `null == 0` evaluates to `true` in PHP. Result: an invalid date could pass validation against a reference date that parses to a falsy value (e.g., `1970-01-01 00:00:00`).

**Affected versions:** Laravel ≤ 13.14.x  
**Fixed in:** Laravel 13.15.0  
**Severity:** Medium — exploitable via any user-controlled input validated by `date_equals`

```php
// Exploit example (pre-13.15)
// Request: POST /api/subscribe with { "expires_at": "not-a-date", "renewal_date": "1970-01-01" }
$validated = $request->validate([
    'expires_at' => 'date_equals:renewal_date', // passes when it shouldn't
]);
```

**Mitigation without upgrade:** Cast dates explicitly in `prepareForValidation()`:
```php
protected function prepareForValidation(): void
{
    $this->merge([
        'expires_at' => $this->expires_at ?: null,
    ]);
}
```

### 2. Restricted Route Unserialization

Route caching and resolution use PHP's `unserialize()` internally. Laravel 13.15 restricts the classes accepted during unserialization, reducing the object-injection surface for attackers who can influence cached route data.

**Affected versions:** All prior Laravel versions (long-standing hardening improvement)  
**Fixed in:** Laravel 13.15.0  
**Severity:** Low–Medium — only relevant if an attacker can poison the route cache (e.g., write access to `bootstrap/cache/`)

**Practical impact:** Most apps benefit automatically by upgrading. No code changes required — the framework now rejects disallowed classes during unserialization.

**For applications that custom-serialize routes:** Audit any custom `Serializable` implementations on route classes. The new restrictions may require explicit registration via the framework's allowed-classes list.

Source: [Laravel News 13.15](https://laravel-news.com/laravel-13-15-0) | [PR #60391](https://github.com/laravel/framework/pull/60391) | [PR #60393](https://github.com/laravel/framework/pull/60393)

## Critical: Laravel Passport CVE-2026-39976 (CVSS 7.1 — High)

**Authentication bypass in `client_credentials` tokens** affecting Laravel Passport **13.0.0 → 13.7.0**. Any machine-to-machine token can inadvertently authenticate as a real user.

**Root cause:** The `league/oauth2-server` library sets the JWT `sub` claim to the **client identifier** (since there's no user in a `client_credentials` flow). The Passport token guard then passes this value to `retrieveById()` without validating it's actually a user identifier — so a client_id that happens to match a user ID resolves to that real user.

**Affected:** `laravel/passport` from `13.0.0` to before `13.7.1`  
**Fixed in:** `laravel/passport` **13.7.1**  
**Disclosed:** June 2, 2026  
**Severity:** CVSS 7.1 (High) — full authentication bypass

### Who is affected

Any app using `laravel/passport` ≥ 13.0.0 and exposing `client_credentials` grants to:
- Internal service-to-service auth
- Public OAuth client flows
- Machine-to-machine API tokens

### How to fix

1. **Upgrade immediately:**
   ```bash
   composer require laravel/passport:^13.7.1
   php artisan passport:install
   ```

2. **Audit for exploit evidence** — look for `client_credentials` token usage where the resolved user isn't expected. Any token issued to a `client_credentials` client that resolved to a real user is a confirmed exploit.

3. **Defense in depth** — even on 13.7.1+, do not reuse user IDs as client IDs. Use UUIDs or prefixed identifiers (`cli_xxx` vs `usr_xxx`) so a `client_id` cannot collide with a `user_id`.

### Mitigation if you cannot upgrade immediately

Add a guard in your `AuthServiceProvider` to reject `client_credentials` token resolution to a non-`null` user:

```php
// app/Providers/AuthServiceProvider.php
public function boot(): void
{
    Auth::viaRequest('passport-client-credentials', function ($request) {
        // Force client_credentials flow to never resolve a user
        $psr = $request->attributes->get('psr_request');
        $grantType = data_get($psr->getParsedBody(), 'grant_type');
        if ($grantType === 'client_credentials') {
            return null; // never authenticate as a user
        }
        // ... existing passport token user resolution
    });
}
```

**Note:** This mitigation must be carefully tested — it changes the resolution contract for all `client_credentials` flows.

Source: [CVE-2026-39976 on OpenCVE](https://app.opencve.io/cve/?vendor=laravel) | [Laravel Passport security advisory](https://github.com/laravel/passport/security/advisories)

## Critical: Laravel Reverb RCE — CVE-2026-23524 (CVSS 9.8 — Critical)

**Insecure deserialization (CWE-502)** in Laravel Reverb when horizontal scaling is enabled. Reverb passed data from the Redis PubSub channel directly into PHP's `unserialize()` without restricting which classes could be instantiated — giving any actor who can publish to Redis full **Remote Code Execution**.

**Affected:** `laravel/reverb` **≤ 1.6.3** with `REVERB_SCALING_ENABLED=true`  
**Fixed in:** `laravel/reverb` **1.7.0**  
**Disclosed:** January 21, 2026  
**Severity:** CVSS 9.8 (Critical) — unauthenticated RCE in scaled deployments

### Why it's dangerous
- Redis servers are commonly deployed **without authentication**
- Many self-hosted setups bind Redis to a public interface or shared VPC
- A single malicious publish to the Reverb PubSub channel yields code execution on every Reverb node
- Only affects multi-node deployments (`REVERB_SCALING_ENABLED=true`) — single-node installs are safe

### How to fix
1. **Upgrade immediately:**
   ```bash
   composer require laravel/reverb:^1.7.0
   ```
2. **Verify your scaling config** — even on 1.7.0+, run Redis on a private network with `requirepass` set.

### Mitigation if you cannot upgrade immediately
```bash
# Option A: disable horizontal scaling (single-node only)
REVERB_SCALING_ENABLED=false

# Option B: lock down Redis
requirepass <strong-password>
# bind Redis to private IP / localhost only
# Use a firewall to block external Redis access (port 6379)
```

**For single-node Reverb installs** (the default for most apps), this CVE does not apply — `REVERB_SCALING_ENABLED` defaults to `false`.

Source: [CVE-2026-23524 on NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-23524) | [GHSA-m27r-m6rx-mhm4](https://github.com/laravel/reverb/security/advisories/GHSA-m27r-m6rx-mhm4)

## Critical: Laravel CRLF Injection — CVE-2026-48019 (CVSS High)

**CRLF injection in Laravel's mail handling** allows attackers who control an email address (e.g., via a contact form) to inject `\r\n` sequences and manipulate outbound email — adding BCC recipients, modifying subject lines, or smuggling extra MIME parts. Underlying root cause: insufficient stripping of CRLF characters before the value reaches Symfony Mailer / Symfony Mime.

**Affected:** All Laravel versions **≤ 13.9.0** and **≤ 12.59.0**  
**Fixed in:** Laravel **13.10.0** and **12.60.0**  
**Severity:** High — pre-auth email hijacking on apps that accept user-supplied email addresses

### Who is affected
Any app that sends mail using a user-supplied email address without further sanitization:
- Contact forms ("your email" + "recipient")
- Invite / share-by-email flows
- Password reset to arbitrary addresses
- User-to-user messaging

### How to fix
1. **Upgrade Laravel:**
   ```bash
   composer update laravel/framework  # to 13.10.0+ or 12.60.0+
   ```
2. **Defense in depth — validate inputs server-side:**
   ```php
   $request->validate(['email' => 'required|email:rfc']);
   // Use Laravel's built-in email validation; Laravel 10+ strips CRLF on validate.
   ```
3. **For Laravel 10 apps that can't upgrade** — strip CRLF manually before passing to mail:
   ```php
   $email = str_replace(["\r", "\n"], '', $request->input('email'));
   if (! filter_var($email, FILTER_VALIDATE_EMAIL)) {
       abort(422, 'Invalid email');
   }
   ```

**Apps on Laravel 13.10.0+ or 12.60.0+ are safe.** No code changes required.

Source: [CVE-2026-48019 on Feedly](https://feedly.com/cve/CVE-2026-48019) | [NVD entry](https://nvd.nist.gov/vuln/detail/CVE-2026-48019)

## Critical: Laravel-Lang Composer Supply-Chain Attack (May 22, 2026 — no CVE)

**Tag-rewrite supply-chain attack** on the Laravel-Lang GitHub organization. An attacker with push access rewrote **every git tag** across four popular Composer packages to point at malicious commits that exfiltrate credentials.

**Disclosed:** May 22, 2026 (rewrites between 22:32 UTC and 23:24 UTC)  
**Severity:** Critical — RCE backdoor in `src/helpers.php`, exfiltrates CI/cloud secrets  
**Coverage:** Mend.io MSC-2026-5762, Snyk advisory, GitHub Security Advisory

### Affected packages
- `laravel-lang/lang`
- `laravel-lang/attributes`
- `laravel-lang/http-statuses`
- `laravel-lang/actions`

Every git tag across all four packages was rewritten. **There is no safe version to pin to** unless you can verify the SHA against a pre-2026-05-22 commit.

### How the backdoor works
The malicious commits make a two-file change:
1. Add `"files": ["src/helpers.php"]` to `composer.json` (auto-load trigger)
2. Inject `src/helpers.php` containing decoy functions plus an anonymous closure that:
   - Fingerprints the host
   - Fetches a PHP credential stealer from `https://flipboxstudio[.]info/payload`
   - Executes it in the background
   - Drops artifacts in `<sys_get_temp_dir>/.laravel_locale/`

### Indicators of Compromise (IoCs)
- C2 domain: `flipboxstudio[.]info`
- Stage-one URL: `hxxps://flipboxstudio[.]info/payload`
- Stage-two exfil URL: `hxxps://flipboxstudio[.]info/exfil`
- Commit author: `Your Name <you@example.com>`
- Commit timestamps: 2026-05-22 22:32 UTC → 2026-05-23 00:00 UTC
- Modified files (always exactly these two): `composer.json`, `src/helpers.php`

### How to detect & remediate
1. **Audit `composer.lock`** — for any of the four affected packages, look up the pinned SHA in the upstream repo and check the commit author. Locked pre-2026-05-22 SHAs are safe.
2. **Check `vendor/laravel-lang/*/composer.json`** — confirm it does **not** contain `"files": ["src/helpers.php"]`.
3. **Check `vendor/laravel-lang/*/src/helpers.php`** — this file should not exist in any of the four affected packages.
4. **Search for IoCs on every host** — run `grep -r "flipboxstudio" /` and check for `<sys_get_temp_dir>/.laravel_locale/` artifacts.
5. **Rotate credentials** — anything the affected servers had access to (AWS, GitHub, npm, DB, etc.) should be considered compromised.
6. **Pin to verified SHAs** — replace `composer require laravel-lang/lang` with a manually-vetted commit SHA from before 2026-05-22.

### Defense in depth
- Use **Composer audit** in CI: `composer audit` (or `composer require --dev roave/security-advisories`).
- Pin all dependencies to **commit SHAs** in `composer.lock` for any third-party package.
- Run Composer in CI inside a sandboxed runner without access to long-lived secrets.
- Monitor outbound traffic from CI runners to unknown domains (`flipboxstudio[.]info`).

Source: [StepSecurity writeup](https://www.stepsecurity.io/blog/laravel-lang-supply-chain-attack) | [SecurityWeek](https://www.securityweek.com/laravel-lang-packages-poisoned-for-malware-delivery/) | [Mend.io analysis](https://www.mend.io/blog/laravel-lang-composer-tag-rewrite-supply-chain-attack/) | [Snyk advisory](https://snyk.io/blog/laravel-lang-supply-chain-advisory/)


## Critical: plank/laravel-mediable Arbitrary File Upload — CVE-2026-4809 (CVSS 9.3 — Critical, **No Patch**)

### Who is affected

Any Laravel app using `plank/laravel-mediable` through **version 6.4.0** that accepts (or prefers) a **client-supplied MIME type** during file upload handling. The package itself does not enforce a server-side MIME check; the application code decides whether to trust the client's `Content-Type` / `$_FILES[...]['type']`. Apps that rely on the client-supplied MIME for "image" uploads are vulnerable.

### Why it's dangerous

A remote attacker can submit a file containing **executable PHP code** while declaring a benign image MIME type (e.g. `image/jpeg`). If the file is stored in a **web-accessible and executable** location (which is the common `public/uploads` pattern), this leads directly to **remote code execution**. CVSS 9.3.

### How to fix

- **Do not rely on client-supplied MIME.** Re-validate using a server-side check (file extension allowlist + `Symfony\Component\HttpFoundation\File\UploadedFile::getMimeType()` or `guessExtension()` via `symfony/mime`).
- Move uploads **out of the public web root** if possible; serve them through a controller that enforces auth + content-type.
- Pin a patched version once `plank/laravel-mediable` ships one (none available at time of writing — vendor had not responded to coordinated disclosure attempts).
- Consider replacing `plank/laravel-mediable` with `spatie/laravel-medialibrary` (proactive MIME validation + VirusTotal integration in recent versions) until upstream patches.

### Mitigation if you cannot upgrade / replace immediately

```php
// App\Http\Requests\UploadRequest.php — example server-side check
public function rules(): array
{
    return [
        'file' => ['required', 'file', 'mimes:jpg,jpeg,png,webp', 'max:5120'],
    ];
}

// Storage path: NEVER serve uploads directly from public/.
// Use a route + controller that sets Content-Disposition: attachment
// and validates the authenticated user has access.
```

Source: [Feedly CVE feed for Laravel](https://feedly.com/cve/vendors/laravel) | [Snyk advisory DB](https://snyk.io/vuln/CVE-2026-4809)

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
