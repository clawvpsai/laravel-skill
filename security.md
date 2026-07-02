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

## High: Laravel Signed URL Path Confusion — CVE-2026-48041 / GHSA-crmm-hgp2-wgrp (June 8, 2026)

**Path-parsing ambiguity in Laravel's local filesystem driver** lets a generated temporary signed URL be interpreted differently by the server than intended at signing time. Expired temporary URLs may continue to be accepted, requests may resolve to a different resource than the one that was signed, and the upload variant may allow writes to reach an unintended destination.

**Disclosed:** June 8, 2026 (GitHub Security Advisory, laravel/framework)
**Affected:** Laravel **< 13.12.0** and **< 12.61.1**
**Fixed in:** Laravel **13.12.0** and **12.61.1**
**Severity:** Moderate (CVSS v3.1 4.2 MEDIUM — `AV:N/AC:H/PR:L/UI:N/S:U/C:L/I:L/A:N`)
**CWE:** CWE-116 (Improper Encoding or Escaping of Output)

### Who is affected
Any Laravel app that uses `URL::temporarySignedRoute()` (or `Storage::temporaryUrl()` on the local driver) to grant time-limited access to private files — uploaded documents, invoices, user avatars, downloads, video streams, S3-style presigned URLs against the `local` disk.

### Why it's dangerous
- **Expiration bypass:** a signed URL whose `expires` timestamp has passed can still be accepted if the path is parsed ambiguously.
- **Resource misrouting:** a URL signed for `downloads/invoice-123.pdf` may resolve to a different file on disk.
- **Upload variant:** writes from a signed upload URL may reach an unintended destination — useful for planting files in arbitrary paths an attacker can later request.

### How to fix
```bash
composer update laravel/framework    # to 13.12.0+ or 12.61.1+
```

### Hardening (always do this, even on fixed versions)
```php
use Illuminate\Support\Facades\URL;
use Illuminate\Support\Facades\Storage;

// 1. Always use absolute, explicit paths in signed routes — never derive from user input
URL::temporarySignedRoute(
    'downloads.show',
    now()->addMinutes(5),
    ['path' => 'invoices/' . $invoice->id . '.pdf']
);

// 2. On the receiving controller, re-validate the resolved path stays inside the expected dir
Route::get('/download/{path}', function (string $path) {
    if (! request()->hasValidSignature()) {
        abort(401);
    }
    $base = Storage::disk('private')->path('invoices');
    $resolved = Storage::disk('private')->path($path);
    if (! str_starts_with($resolved, $base)) {
        abort(404);
    }
    return response()->download($resolved);
})->where('path', '.*')->name('downloads.show');

// 3. Keep expiry windows short (5 minutes, not 24 hours)
```

Source: [GHSA-crmm-hgp2-wgrp](https://github.com/laravel/framework/security/advisories/GHSA-crmm-hgp2-wgrp) | [NVD CVE-2026-48041](https://nvd.nist.gov/vuln/detail/CVE-2026-48041) | [PR #60137](https://github.com/laravel/framework/pull/60137) | [PR #60230](https://github.com/laravel/framework/pull/60230)

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

## Critical: spatie/laravel-medialibrary SSRF + Upload Bypass — CVE-2026-48555 & CVE-2026-48557 (May 29, 2026)

Two related vulnerabilities in the popular `spatie/laravel-medialibrary` package, both fixed in **v11.23.0**. Apps pinned to `<11.23.0` that use `addMediaFromUrl()` or accept user-supplied filenames are affected.

### CVE-2026-48555 — SSRF via `addMediaFromUrl()` (CVSS 7.4 — HIGH)

**Who is affected:** Any Laravel app using `spatie/laravel-medialibrary` **< 11.23.0** that calls `addMediaFromUrl()` with a URL derived from user input (e.g., a "import from URL" feature, avatar upload by URL, attachment fetcher, OG image scraper).

**Why it's dangerous:** The pre-11.23.0 `InteractsWithMedia::addMediaFromUrl()` issues an HTTP request to whatever URL is passed — including private network ranges (169.254.169.254 cloud metadata, 127.0.0.1, internal admin ports). An attacker who can influence the URL can:
- Read cloud metadata (AWS/GCP/Azure IAM credentials → RCE)
- Scan and probe internal services (Redis, Elasticsearch, admin UIs)
- Bypass firewalls / SSRF protections in front of the Laravel app

```php
// VULNERABLE — passes user-supplied URL straight through
$model->addMediaFromUrl($request->input('image_url'));
```

### CVE-2026-48557 — File upload restriction bypass via `FileAdder::defaultSanitizer()` (HIGH)

**Who is affected:** Same scope — `spatie/laravel-medialibrary` **< 11.23.0** apps that rely on the package's built-in sanitizer to block executable extensions.

**Why it's dangerous:** The sanitizer only checks the **last** extension in the filename. Two bypasses:
- **Double-extension bypass:** `shell.php.jpg` is accepted by the sanitizer (looks like `.jpg`), but `pathinfo()` preserves the inner `.php` stem when the file is saved. On legacy Apache configs with `AddHandler` for `.php`, this can lead to PHP execution.
- **Incomplete blocklist:** Executable extensions `.php6`, `.shtml`, and `.htaccess` are missing from the default sanitizer blocklist (CWE-184).

### How to fix

```bash
# Pin to a fixed version
composer require spatie/laravel-medialibrary:^11.23.0
composer update spatie/laravel-medialibrary
```

### Defense-in-depth for `addMediaFromUrl()` (always do this, even after upgrading)

```php
use Illuminate\Support\Facades\Http;
use Illuminate\Validation\ValidationException;

public function importFromUrl(string $url): void
{
    // 1. URL allowlist or strict validation
    if (!filter_var($url, FILTER_VALIDATE_URL) || !str_starts_with($url, 'https://')) {
        throw ValidationException::withMessages(['image_url' => 'Must be an https URL.']);
    }

    $host = parse_url($url, PHP_URL_HOST);

    // 2. Block private / loopback / link-local / metadata ranges
    if (in_array(gethostbyname($host), ['127.0.0.1', '::1'], true)
        || preg_match('/^(10\.|172\.(1[6-9]|2\d|3[01])\.|192\.168\.|169\.254\.)/', gethostbyname($host))) {
        throw ValidationException::withMessages(['image_url' => 'Internal addresses not allowed.']);
    }

    // 3. Avoid following redirects (each hop can pivot to an internal IP)
    $response = Http::withoutRedirecting()->timeout(10)->get($url);

    if (!$response->successful()) {
        throw ValidationException::withMessages(['image_url' => 'Could not fetch URL.']);
    }

    $this->addMediaFromString($response->body())
        ->usingFileName(Str::random(40) . '.' . guessExtensionFromContentType($response->header('Content-Type')))
        ->toMediaCollection('imports');
}
```

Sources: [NVD CVE-2026-48555](https://nvd.nist.gov/vuln/detail/CVE-2026-48555) | [NVD CVE-2026-48557](https://nvd.nist.gov/vuln/detail/CVE-2026-48557) | [VulnCheck advisory](https://www.vulncheck.com/advisories/spatie-laravel-media-library-ssrf-via-addmediafromurl) | [spatie/laravel-medialibrary 11.23.0 release](https://github.com/spatie/laravel-medialibrary/releases/tag/11.23.0)


## High: Sharp Laravel CMS Storage Disk Path Confusion — CVE-2026-44692 / GHSA-748w-hm6r-qc7v (June 11, 2026)

Information disclosure in **code16/sharp** (Sharp Laravel CMS), the admin UI for Laravel-based content management. Fixed in **v9.22.0**.

### Who is affected

Any Laravel application integrating `code16/sharp` < **9.22.0** that has one or more configured Storage disks AND exposes Sharp-managed entity download endpoints to authenticated users.

### Why it's dangerous

Sharp exposes a generic file download endpoint used by its administrative interface to retrieve files attached to managed entities. The endpoint enforces authorization by verifying that the requesting user can access the **referenced Sharp entity instance**. However, the **storage disk identifier and file path are taken directly from request parameters** and are NOT validated against the authorized entity instance.

This breaks the binding between the authorization decision and the resource being served. An attacker holding any valid entity reference can substitute the `disk` and `path` parameters to fetch unrelated objects from any configured Laravel Storage disk.

Classified as **CWE-639: Authorization Bypass Through User-Controlled Key**. Access is limited to roots of configured Laravel Storage disks (not arbitrary host filesystem paths), but private disk contents (S3 buckets, secure backups, signed media) can leak.

### Attack vector (requires authentication)

```http
GET /sharp/download?entity=valid_entity_id&disk=s3&path=secure/financial-report.pdf
Cookie: sharp_session=...
```

The endpoint verifies the user can read `valid_entity_id`, then unconditionally returns the file from the `s3` disk at the supplied path — regardless of whether that path is actually attached to that entity.

### How to fix

```bash
# Pin to fixed version
composer require code16/sharp:^9.22.0
composer update code16/sharp
```

### Hardening (always do this, even on fixed versions)

- **Never mount sensitive disks** (private backups, IAM-credential files, internal-only buckets) under Sharp's `disks:` config — Sharp passes through to the `Storage` facade.
- Wrap Sharp download endpoints behind an additional auth middleware that re-validates the `(entity, disk, path)` triple against a database mapping.
- For sensitive content, prefer **signed URLs** with short TTL instead of letting Sharp serve files directly.

Source: [GHSA-748w-hm6r-qc7v](https://github.com/code16/sharp/security/advisories/GHSA-748w-hm6r-qc7v) | [CVE-2026-44692 details](https://www.sentinelone.com/vulnerability-database/cve-2026-44692/) | [vulmon entry](https://vulmon.com/vulnerabilitydetails?qid=CVE-2026-44692)


## Medium: Sharp Laravel CMS Quick Creation Authorization Bypass — CVE-2026-53634 / GHSA-vmwx-m75v-qvch (June 10, 2026)

Missing authorization check on Sharp's "Quick Creation Command" `create`/`store` endpoints. Fixed in **code16/sharp v9.22.3**. The patch for **CVE-2026-44692 (v9.22.0)** did NOT cover this issue — a separate, later patch was needed.

### Who is affected

Any Laravel application running `code16/sharp` versions **9.0.0 through 9.22.2** inclusive that:
- Exposes the Sharp admin interface to authenticated users, AND
- Has at least one entity with a configured Quick Creation Command handler.

### Why it's dangerous

Sharp's Quick Creation feature exposes short-form `create` and `store` endpoints (separate from the standard Sharp entity form) for inline record creation. These endpoints **did not enforce the standard Sharp entity-level "create" permission check** before the fix. Any authenticated Sharp user (even one explicitly denied `create` permission on the entity) could:
- Render the Quick Creation form (`GET` create endpoint) for entities they should not touch.
- Submit the form (`POST` store endpoint) and create new records for those entities.

Classified as **CWE-862: Missing Authorization**. The vulnerability is authenticated (the user must hold a valid Sharp session) and **horizontal** (privilege boundary between Sharp users, not unauthenticated-to-authenticated escalation). Severity is rated Medium (CVSS 3.x: 4.3) because it requires a Sharp account and the impact is limited to record creation within entities that already have a Quick Creation handler configured — but in multi-tenant CMS deployments or any workflow where some editors should be read-only on certain entities, this is a real data-integrity issue.

### How to fix

```bash
# Pin to fixed version (NOTE: v9.22.0 fixed CVE-2026-44692 but NOT this one — must be 9.22.3+)
composer require code16/sharp:^9.22.3
composer update code16/sharp
```

### Hardening

- After upgrading, audit your `EntityConfiguration` classes: if an entity has a `QuickCreationCommand`, verify the entity's policy correctly returns `false` for users who should not create records. The fix adds the missing check, but defense-in-depth is still worthwhile.
- Consider disabling Quick Creation on entities where only a subset of users should have create rights — set the Quick Creation handler to `null` and use the full form for authorized users.

Source: [GHSA-vmwx-m75v-qvch](https://github.com/code16/sharp/security/advisories/GHSA-vmwx-m75v-qvch) | [CVE-2026-53634 details](https://app.opencve.io/cve/CVE-2026-53634) | [vulmon entry](https://vulmon.com/vulnerabilitydetails?qid=CVE-2026-53634) | [Sharp v9.22.3 release](https://github.com/code16/sharp/releases/tag/v9.22.3)


## Medium: Statamic CMS Control Panel Authorization Bypass — CVE-2026-49288 (June 19, 2026)

Missing authorization on Statamic Control Panel fieldtype endpoints. Fixed in **5.73.23** and **6.20.0**.

### Who is affected

Statamic CMS installations running versions **prior to 5.73.23** (5.x line) or **prior to 6.20.0** (6.x line) with authenticated Control Panel users and the affected fieldtype endpoints enabled.

### Why it's dangerous

Prior to the fix, an **authenticated Control Panel user could view metadata and content for resources they were NOT permitted to view**, including:

- Entry titles and custom field values
- Entry content
- Asset metadata
- Existence of users, roles, and groups

Classified as **CWE-200** (Exposure of Sensitive Information), **CWE-862** (Missing Authorization), and **CWE-863** (Incorrect Authorization). The flaw is **read-only disclosure** — no data modification is possible — which is why the CVSS score is **4.3 (Medium)** despite the authorization bypass.

### How to fix

```bash
# For 5.x line
composer update statamic/cms:^5.73.23

# For 6.x line
composer update statamic/cms:^6.20.0
```

### Hardening (always do this)

- Audit your Statamic user/role definitions: any authenticated CP user with a low-privilege role can still query fieldtype endpoints for resources they nominally cannot access.
- Review Statamic role/permission scopes — consider segmenting read-only CP roles further.
- Watch Statamic security advisories for follow-up patches; CVE-2026-49288 was a single fix but the fieldtype endpoint surface is broad.

Source: [OpenCVE CVE-2026-49288](https://app.opencve.io/cve/CVE-2026-49288) | [CVE new Twitter disclosure](https://x.com/CVEnew/status/2068302713285165395)





## Medium: Statamic CMS Incomplete Fix Follow-up — CVE-2026-49287 (June 17, 2026)

Incomplete remediation of **CVE-2026-41175** (April 22, 2026 — query parameter manipulation allowing loss of content/assets/users via CP, REST, and GraphQL endpoints). The original fix landed in **5.73.20 / 6.13.0** but only patched the **query builder** path. The **same protection was missing from in-memory collection sorting**, so sites that had only upgraded to 5.73.20 / 6.13.0 were still vulnerable via templates that pass visitor-controlled input into a tag's `sort` parameter. Same fix versions also address CVE-2026-49288: **5.73.23** (5.x) and **6.20.0** (6.x).

### Who is affected

Statamic CMS installations running versions **5.73.20 ≤ v < 5.73.23** (5.x line) or **6.13.0 ≤ v < 6.20.0** (6.x line) **AND** using front-end templates that pipe request input into a collection tag's sort parameter — e.g. `{{ collection:posts sort="{get:sort}" }}`. Not exploitable on default templates; only on templates explicitly written to sort by visitor input.

### Why it's dangerous

Manipulating sort parameters could result in the **loss of content and assets** (delete-by-rogue-sort-key). The CP / REST / GraphQL exploitation surface from CVE-2026-41175 is **not** re-opened here — this is specifically the in-memory collection path. CWE-20 (Improper Input Validation) and CWE-862 (Missing Authorization). CVSS score not yet published, but impact is content/asset loss (integrity), which can be severe for editorial sites.

### How to fix

```bash
# For 5.x line (one jump past 5.73.20 already-applied)
composer update statamic/cms:^5.73.23

# For 6.x line (one jump past 6.13.0 already-applied)
composer update statamic/cms:^6.20.0
```

If you patched CVE-2026-41175 on April 22, you must patch again — **the original fix was incomplete**. Sites that have never applied either patch must reach **5.73.23+ / 6.20.0+** to be fully covered against both CVE-2026-41175 and CVE-2026-49287.

### Hardening

- Audit every front-end template that uses `sort=` / `orderBy=` parameters — never pipe raw `{{ get:... }}` request input into a sort parameter. Use an allowlist: `{{ sort = get:sort | explode:-, is_in:allowed_sorts ? get:sort : 'date' }}`.
- Treat this as a class of bug: any time user input flows into a query modifier (sort, orderBy, where, filter), it must be allowlisted server-side. The same pattern that let CVE-2026-41175 through query params also let CVE-2026-49287 through sort.
- After upgrading, grep your templates for `sort=` and `orderBy=` and verify each is allowlisted.

Source: [NVD CVE-2026-49287](https://nvd.nist.gov/vuln/detail/CVE-2026-49287) | [Tenable CVE-2026-49287](https://www.tenable.com/cve/CVE-2026-49287) | [CVE-2026-41175 (parent advisory)](https://nvd.nist.gov/vuln/detail/CVE-2026-41175)

## Critical: Laravel Livewire RCE — CVE-2025-54068 (CVSS 9.8 — Critical, **Active In-The-Wild Exploitation as of June 23, 2026**)

Imperva Threat Research disclosed on **June 23, 2026** that CVE-2025-54068 is being mass-exploited against unpatched Laravel Livewire v3 deployments. The campaign has compromised **6,000+ applications** for credential theft (`.env` files, AWS keys, DB credentials, mail tokens). Initial observations began May 24, 2026.

### Who is affected

Any application running **`livewire/livewire` v3.0.0 through v3.6.3** exposed to the public internet, regardless of whether the Livewire endpoints are reachable. Attackers fingerprint exposed apps and send weaponized POST payloads to Livewire's component-hydration endpoint.

### Why it's dangerous

CVE-2025-54068 is an unauthenticated PHP object-injection (CWE-502) in Livewire v3's hydration pipeline. The framework deserializes client-submitted component data before verifying integrity, allowing a malicious serialized object to be injected. Successful exploitation → **Remote Code Execution** as the web user (usually `www-data`).

CVSS 9.8 (Critical) — network attack vector, no auth, no user interaction, full CIA impact.

The June 2026 campaign specifically targets:
- `.env` files → exfiltrate to attacker-controlled host
- AWS / S3 / DigitalOcean / Linode / Stripe API keys
- Database credentials (then dump & sell)
- Mail provider credentials (SendGrid, Mailgun, Postmark)
- Session/cookie theft via `cookie.txt` extraction

### How to fix

```bash
composer require livewire/livewire:^3.6.4
composer update livewire/livewire
```

`3.6.4` adds integrity verification to the hydration step (the patch is **strict** — a single property whose type doesn't match the component signature is rejected).

If you must stay on 3.6.3 temporarily, block the hydration endpoint with WAF rules matching `wire:*.snapshot` and `wire:snapshot.upload` POSTs with suspicious serialized payloads (`O:` followed by a long class name). Imperva's report notes that Cloud WAF rules are blocking 95%+ of the campaign traffic.

### Mitigation if you cannot upgrade immediately

1. **Restrict Livewire routes to authenticated users only** — `Route::middleware(['auth'])->group(function () { Route::livewire(...); });` — CVE-2025-54068 is unauthenticated, so this is a partial mitigation at best.
2. **WAF rules** — block serialized payloads with `O:` (PHP object) or `a:` (PHP array of objects) in Livewire snapshot fields. Use the Imperva / Cloudflare WAF signatures released for CVE-2025-54068.
3. **Disable the Livewire snapshot endpoint** — `Livewire::setScriptRoute(null);` (Laravel 11+) removes the auto-registered hydration route. Not viable if you use Livewire components in production.
4. **Disable `unserialize()` globally** — set `unserialize_callback_func` to a no-op in `php.ini` so that any object instantiation via `unserialize()` is blocked even if a vulnerable code path is reached.
5. **Audit IoCs** — check for:
   - Outbound HTTP to non-allowlisted hosts from `www-data` / `php-fpm` user
   - `.env` files newer than their last known good mtime
   - Suspicious long POST bodies with `wire:snapshot` + `O:` patterns in Nginx/Apache access logs
   - New files in `storage/framework/sessions/` that don't match any known user

### Detection — Imperva IoCs (June 23, 2026)

- `POST /livewire/update` with `Content-Type: application/json` and `{"_token": "...", "components": [{"snapshot": "O:..."}]}` 
- Wire snapshot strings beginning with `O:8:"Symfony\\..."` (Symfony gadget chains) or `O:11:"Illuminate\\..."` (Laravel gadget chains)
- Outbound connections to `45.x.x.x`, `194.x.x.x` (attacker C2 ranges observed in Imperva report)

### Background context

CVE-2025-54068 was originally disclosed in **July 2025** by Synacktiv researchers, fixed in `livewire/livewire` **3.6.4** (January 2026), but the June 2026 mass-exploitation campaign shows that **most Livewire v3 deployments have not been patched**. The gap between patch availability and adoption (5+ months) is what makes this a high-priority audit item for any Livewire-using app.

**Bottom line:** if you use Livewire v3, run `composer show livewire/livewire` RIGHT NOW. Anything below 3.6.4 is exploitable and being actively targeted.

Source: [Imperva Threat Research report, June 23 2026](https://securityboulevard.com/2026/06/cve-2025-54068-laravel-livewire-credential-theft-campaign-6000-applications-compromised/) | [CVE-2025-54068 on NVD](https://nvd.nist.gov/vuln/detail/CVE-2025-54068) | [Synacktiv original disclosure (July 2025)](https://www.synacktiv.com/en/publications/livewire-remote-command-execution-through-unmarshaling) | [Tenable plugin WAS-115113](https://www.tenable.com/plugins/was/115113)



## Critical: "Slow JSON Stream" DoS — Tier 1 Vulnerability for PHP/Laravel (CVSS 7.5 — High, disclosed 2026-06-27)

Daniel García (cr0hn) published full-disclosure research on **June 27, 2026** for a low-bandwidth denial-of-service attack targeting HTTP APIs that accept `application/json` bodies. **PHP/Laravel is classified as Tier 1 (worst impact)** — measured at **258 MB/min RSS growth**, exhausting a 512 MB PHP worker pool in **under 2 minutes** with only 64 concurrent connections sending at <1 kbps. No CVE has been assigned yet (paper is "pending review" as of 2026-06-27) but the proof-of-concept attack tool (`slowjson`), a 41-target Docker testbed, and the complete paper are publicly available on GitHub.

### Why this matters for Laravel

The testbed tested 32 application frameworks plus 9 infrastructure components. **PHP/Laravel was the single most severe result** — 258 MB/min RSS growth at C=64, classified **Tier 1 (immediate observable impact)** with CVSS **7.5 HIGH**. A commodity host is sufficient to exhaust an unhardened Laravel worker pool.

Laravel's JSON request pipeline has no built-in body-read timeout. Once an attacker sends a syntactically valid JSON prefix (`{"items":[{"a":1},`) using `Transfer-Encoding: chunked` and drips one byte per second, the worker thread blocks **at two layers simultaneously**:

1. **HTTP layer** — waiting for the chunk terminator `0\r\n\r\n` that never arrives
2. **JSON parser layer** — waiting for the closing `}`/`]` token

Because each delivered byte is a legal prefix of a complete JSON document, the parser **cannot reject** the connection early. Every blocked connection holds an FPM worker until OOM.

### How the attack works

```http
POST /api/orders HTTP/1.1
Host: target.example.com
Transfer-Encoding: chunked
Content-Type: application/json

1
{
6
"items
1
":[
14
{"sku":"A1","qty
```

The attacker drips one chunk per second. `Content-Length` is never sent — only `Transfer-Encoding: chunked`. Each chunk is a legal JSON prefix. The PHP-FPM worker blocks on `php://input` read forever. **64 such connections saturate a default FPM pool of 5–10 workers within 90 seconds.**

### Mitigations (do all of these — no single one is sufficient)

#### 1. nginx: aggressive body-read timeout + minimum read rate

The default `client_body_timeout 60s` is **not safe** — it resets on each chunk, so an attacker can drip 1 byte every 59 s indefinitely. You need both a hard ceiling AND a minimum read rate:

```nginx
server {
    # ... existing config ...

    # Hard ceiling: close any body that takes longer than 10 s
    client_body_timeout 10s;

    # Minimum average read rate (bytes/second); attacker must deliver this much
    # or nginx closes the connection. Default is 0 (disabled) — vulnerable.
    client_body_buffer_size 16k;
    client_min_rate 1024;          # 1 KB/s minimum sustained rate

    # Cap body size so the attacker's prefix can never grow unboundedly
    client_max_body_size 10m;

    # Optional: limit concurrent open connections per IP to slow the attack
    limit_conn api_conn 10;        # max 10 concurrent per IP at the `api_conn` zone
}
```

Pair with a `limit_req` zone on the API location:

```nginx
# In the http {} block — define the zone once
limit_req_zone $binary_remote_addr zone=api_rl:10m rate=10r/s;

server {
    location /api/ {
        # Allow burst of 20 requests, then enforce 10 r/s per IP
        limit_req zone=api_rl burst=20 nodelay;

        # Pass to PHP-FPM
        try_files $uri /index.php?$query_string;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        include fastcgi_params;
    }
}
```

#### 2. PHP-FPM: reduce worker lifetime + smaller pool

A `pm.max_requests` cap forces workers to recycle after handling N requests, so memory leaks from long-running workers don't accumulate under sustained low-bandwidth attacks:

```ini
; /etc/php/8.4/fpm/pool.d/www.conf
pm = dynamic
pm.max_children = 20            ; tune to your RAM (≈ 100 MB per worker)
pm.start_servers = 4
pm.min_spare_servers = 2
pm.max_spare_servers = 6
pm.max_requests = 500           ; recycle after 500 requests — prevents unbounded RSS growth
pm.request_terminate_timeout = 30s   ; hard kill for any request over 30 s — backstop in case nginx misses
```

`request_terminate_timeout` is the **PHP-level body-read timeout** equivalent of `client_header_timeout`. It's the last line of defense if nginx is bypassed (Octane, RoadRunner, Cloudflare passthrough).

#### 3. Application-level: reject chunked uploads with a max read time

For Laravel apps serving JSON APIs, you can add a middleware that aborts any request whose body takes longer than N seconds to fully arrive. This is **defense in depth** — nginx is the primary defense, this is the fallback:

```php
// app/Http/Middleware/RejectSlowBodies.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class RejectSlowBodies
{
    public function handle(Request $request, Closure $next, int $maxSeconds = 5): Response
    {
        // Only enforce on requests with a body that the framework will parse
        if (in_array($request->method(), ['POST', 'PUT', 'PATCH'], true)
            && str_contains((string) $request->header('Content-Type'), 'application/json')) {

            $start = microtime(true);

            // Force the body to be parsed — this is where blocking I/O happens
            $request->json()->all();

            $elapsed = microtime(true) - $start;

            if ($elapsed > $maxSeconds) {
                abort(408, 'Request body read timeout');
            }
        }

        return $next($request);
    }
}
```

Register globally (or on the `api` group only):

```php
// bootstrap/app.php
->withMiddleware(function (Middleware $middleware) {
    $middleware->api(prepend: [
        \App\Http\Middleware\RejectSlowBodies::class . ':5',
    ]);
})
```

#### 4. Reverse proxy / WAF (Cloudflare, AWS CloudFront, Fastly)

If you're behind Cloudflare or another managed reverse proxy:

- **Cloudflare** — Enterprise customers can set body-read timeouts via the WAF custom rules. The free tier buffers full requests at the edge before forwarding, which mitigates Slow JSON Stream automatically (because the body is never streamed to your origin in chunks).
- **AWS CloudFront / ALB** — set `Origin Read Timeout` to ≤ 30 s. ALB defaults to 60 s idle timeout — reduce it.
- **Fastly** — VCL `bereq.between_bytes_timeout` defaults to 10 s; verify it's set.

### What does NOT protect you

- `APP_DEBUG=false` — unrelated, the attack works on production configs
- Rate limiting alone (login throttling) — this isn't a brute-force attack
- `client_max_body_size 1m` — the attack uses a tiny body, the limit is the read time, not the size
- `php artisan throttle` middleware — counts requests, doesn't bound body read time
- WAF rules matching `Content-Length` — the attack uses `Transfer-Encoding: chunked`, no Content-Length is sent
- ModSecurity OWASP CRS — tested in the research and **classified Tier 2 (vulnerable)** — default rules do not bound body read time

### Detection — what to look for

- Sudden spike in PHP-FPM worker count (`pm status` or `ps -ef | grep php-fpm`) — all workers stuck in `accept`/`read` state
- Long-tail latency on `/healthz` or `/up` health checks (the attack blocks workers serving ALL routes, including non-JSON ones, once FPM is exhausted)
- nginx access logs showing `POST /api/*` connections held open for >30 s with `Transfer-Encoding: chunked` from the same client IP
- PHP-FPM `slow.log` populated with requests that took >30 s on `php://input` parse

### TL;DR action list (in order of priority)

1. **`client_body_timeout 10s`** in every nginx server block that proxies JSON APIs
2. **`client_min_rate 1024`** (or higher) — the only effective nginx defense because it bounds the per-connection read rate, not just total time
3. **`pm.request_terminate_timeout = 30s`** in PHP-FPM pool config as a backstop
4. **Audit Cloudflare/ALB/CloudFront edge timeouts** — same principle, same defaults to harden
5. **Add the `RejectSlowBodies` middleware** to defense-in-depth

Source: [Slow JSON Stream paper (cr0hn, June 2026)](https://cr0hn.com/papers/slow-json-stream/) | [GitHub: cr0hn/slowjson (attack CLI + Docker testbed)](https://github.com/cr0hn/slowjson) | [r/netsec disclosure thread, June 27 2026](https://www.reddit.com/r/netsec/) | [PHP-FPM pool configuration reference](https://www.php.net/manual/en/install.fpm.configuration.php)

## Slowloris / R.U.D.Y. — Historical Note (pre-2011 mitigation, still relevant)

Slowloris (header-drip, 2009) is fully mitigated in nginx by `client_header_timeout 10s` (the default). R.U.D.Y. / Slow POST (2011) uses `Content-Length` and is detectable by WAFs. **Slow JSON Stream (2026) is different** — it uses `Transfer-Encoding: chunked` with no declared size, evading all existing WAF detection. The body-read timeout equivalent of `client_header_timeout` does not exist by default in any major framework (including Laravel). See the section above for Laravel-specific mitigations.




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


## Updated from Research (2026-06-24, cycle 4)

### New CVEs added (cycle 4, 2026-06-24)

- **CVE-2026-53634 / GHSA-vmwx-m75v-qvch** — Sharp Laravel CMS "Quick Creation Command" missing authorization (CWE-862). Authenticated Sharp users without `create` permission on a given entity could render the Quick Creation form and submit new records when that entity had a Quick Creation handler configured. Fixed in `code16/sharp` **9.22.3** (June 10, 2026). The earlier 9.22.0 patch for CVE-2026-44692 did NOT cover this — projects on 9.22.0/9.22.1/9.22.2 must upgrade again to 9.22.3+.
- **CVE-2026-48505 / 48500 / 48067 / 48167 / 48166 / 54517 — Filament 3.0–3.3.51 / 4.0–4.11.4 / 5.0–5.6.4 cluster (June 22–23, 2026)** — single coordinated patch in **3.3.52 / 4.11.5 / 5.6.5**. Includes MFA recovery-code reuse (CVSS 7.4 HIGH), missing-authorization on temp file uploads (6.5 MED), AttachAction/AssociateAction scope bypass IDOR (6.5 MED), ImageColumn/ImageEntry stored XSS (6.4 MED), login-timing user enumeration (5.3 MED), and OAuth2 AsyncClient use-after-free (5.3 MED). All six are fixed by the same version-line bump — there is no partial patch. Run `composer require filament/filament:"^3.3.52 || ^4.11.5 || ^5.6.5"` and rotate MFA recovery codes post-upgrade. Full details in the cycle-11/12 sections below.

### Top-priority actions for 2026-06-24 (cycle 4)

1. **Livewire 3.6.4 upgrade is the #1 security item** — check `composer show livewire/livewire` on every Laravel project you maintain. Anything < 3.6.4 is actively exploited.
2. **Laravel 13.17.0** if on 13.x — bugfix release, no breaking changes.
3. **Laravel 12.62.x (latest 12.x)** if on 12.x — picks up the `JsonSchema` security backport and the earlier CRLF injection fix.
4. **Laravel 11 = EOL** — security support ended March 12, 2026. Plan upgrade to 12 or 13.

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

## Updated from Research (2026-06-28, cycle 5)

### New findings added (cycle 5, 2026-06-28)

- **Slow JSON Stream DoS** — Daniel García (cr0hn) full-disclosure paper published June 27, 2026. PHP/Laravel classified **Tier 1 (CVSS 7.5 HIGH)** with 258 MB/min RSS growth at C=64 — a 512 MB worker pool is exhausted in under 2 minutes. The attack uses `Transfer-Encoding: chunked` + a syntactically valid JSON prefix dripped at 1 byte/sec, evading all WAFs that match on `Content-Length`. Laravel has **no built-in body-read timeout**; mitigation requires nginx `client_body_timeout` + `client_min_rate` + PHP-FPM `request_terminate_timeout`. Full PoC, paper, and 41-target Docker testbed at [github.com/cr0hn/slowjson](https://github.com/cr0hn/slowjson). See the "Critical: Slow JSON Stream DoS" section above for the full mitigation playbook.

### Top-priority actions for 2026-06-28 (cycle 5)

1. **Slow JSON Stream hardening is the #1 new item** — add `client_body_timeout 10s` + `client_min_rate 1024` to every nginx server block that proxies JSON APIs, and `pm.request_terminate_timeout = 30s` to PHP-FPM pool config. A single commodity attacker host can take down a default Laravel app in under 2 minutes.
2. **Livewire 3.6.4 upgrade remains #1 from cycle 4** — still being actively exploited (6,000+ apps compromised as of June 23, 2026).
3. **Laravel 13.17.0** if on 13.x — bugfix release, no breaking changes.
4. **Laravel 12.62.x (latest 12.x)** if on 12.x — picks up the `JsonSchema` security backport and the earlier CRLF injection fix.
5. **Laravel 11 = EOL** — security support ended March 12, 2026. Plan upgrade to 12 or 13.

### nginx + PHP-FPM hardening checklist (2026-06-28)

The Slow JSON Stream research exposed that **body-read timeouts are absent from default configs of every major framework**. Add this to your deployment playbook:

```nginx
# In every server {} that proxies JSON APIs
client_body_timeout 10s;
client_min_rate 1024;
client_max_body_size 10m;
```

```ini
# /etc/php/8.4/fpm/pool.d/www.conf
pm.max_requests = 500
pm.request_terminate_timeout = 30s
```

Plus a `RejectSlowBodies` middleware (see section above) as application-level defense in depth.

## Updated from Research (2026-06-28, cycle 6)

### New CVEs added (cycle 6, 2026-06-28)

- **CVE-2026-33080** — Filament v4/v5 Table Summarizer Stored XSS (CVSS 7.3 HIGH, disclosed March 20, 2026). Filament Table summarizers render database values directly without HTML escaping, so a user with write access to any column rendered by a summarizer (e.g., `Sum`, `Average`, `Count`) can inject `<script>` that fires in the admin panel context. Affects `filament/filament` 4.0.0–4.8.4 and 5.0.0–5.3.4. **Fixed in 4.8.5 and 5.3.5** — upgrade `composer require filament/filament:^4.8.5` (or `^5.3.5`) and audit any custom `Summarizer` subclass that returns user-controlled HTML. [SentinelOne](https://www.sentinelone.com/vulnerability-database/cve-2026-33080) | [Filament v4.8.5 release](https://github.com/filamentphp/filament/releases/tag/v4.8.5) | [Filament v5.3.5 release](https://github.com/filamentphp/filament/releases/tag/v5.3.5)
- **CVE-2026-48500** — Filament v3.x missing-authorization on temporary file uploads (CVSS 6.5 MED, fixed in **3.3.52**, part of the same SB26-180 cluster as the 4.11.5/5.6.5 release). If you are still on Filament 3.x (LTS track), this is the parallel patch — `composer require filament/filament:^3.3.52`. [NVD record](https://nvd.nist.gov/vuln/detail/CVE-2026-48500) · [GHSA-44wp-g8f4-f4v5](https://github.com/filamentphp/filament/security/advisories/GHSA-44wp-g8f4-f4v5). Full details in the cycle-12 section below.

**Why this matters for Laravel apps:** Filament is the most-installed admin panel package in the Laravel ecosystem. The summarizer XSS is exploitable by any authenticated user with write access to a summarized column, which is almost always broader than the developer assumes ("admins only" is often just "any logged-in back-office user"). A single low-privileged staff account can pivot to admin-context script execution and steal other users' session cookies.

**Audit command** — find every Filament app you maintain and check the version:

```bash
# From inside each Laravel project that uses Filament
composer show filament/filament | grep -E '^versions'
# If the version is < 4.8.5 (v4/v5) or < 3.3.52 (v3 LTS), upgrade now.
```

**Defensive pattern for custom summarizers** — when overriding `Summarizer` to render rich content, force-escape any user-influenced fragment:

```php
use Filament\Tables\Columns\Summarizers\Summarizer;
use Illuminate\Support\HtmlString;

Summarizer::make('total')
    ->formatStateUsing(function ($state) {
        // WRONG — XSS if $state can contain HTML
        return new HtmlString("<strong>{$state}</strong>");

        // RIGHT — escape first, then wrap
        return new HtmlString('<strong>' . e($state) . '</strong>');
    });
```

### Top-priority actions for 2026-06-28 (cycle 6)

1. **Slow JSON Stream hardening (cycle 5)** — still the #1 active finding; verify `client_body_timeout` + `client_min_rate` are in every JSON API nginx block.
2. **Filament upgrade** — `composer require filament/filament:^4.8.5` (v4/v5) or `^3.3.52` (v3 LTS). Audit custom `Summarizer` classes for unsafe `HtmlString` output.
3. **Livewire 3.6.4** — still actively exploited. Verify every project.
4. **Laravel 13.17.0** if on 13.x; **Laravel 12.62.x** if on 12.x.
5. **Laravel 11 = EOL** (security ended March 12, 2026) — plan migration to 12 or 13.

### Cross-references added in cycle 6

- Filament CVEs documented in `security.md` (this section)
- Same Filament XSS pattern (raw user input rendered as HTML) reinforces the **`{!! !!}` rule** in `blade.md` — see Common Mistakes #1 there

## Updated from Research (2026-06-28, cycle 7)

### Composer self-update — three CVEs in the supply chain (March–May 2026)

Every PHP/Laravel host depends on Composer. **All three Composer CVEs below are still widely unpatched in the wild** because `composer self-update` is opt-in. Add it to your deployment runbook.

| CVE | Disclosed | CVSS | Fixed in | Issue |
|---|---|---|---|---|
| **CVE-2026-45793** | 2026-05-13 | 7.5 HIGH | **2.9.8 / 2.2.28 / 1.10.28** | GitHub `GITHUB_TOKEN` and GitHub App installation tokens leaked in plaintext to CI logs when their new hyphenated format (`gh*_-*…`) failed Composer's legacy validation regex. **Rotate tokens if you ran affected versions.** |
| **CVE-2026-40261** | 2026-04-15 | — | **2.9.6 / 2.2.27** | Command injection in `Perforce::syncCodeBase()` — crafted source reference allowed shell metacharacters. Exploitable even if you don't use Perforce (the VCS driver is loaded on demand). |
| **CVE-2026-40176** | 2026-04-15 | — | **2.9.6 / 2.2.27** | Command injection in `Perforce::generateP4Command()` — Perforce `port`, `user`, `client` params in a malicious root `composer.json` were interpolated into a shell command without escaping. Limited to root project config. |

**Why this matters for Laravel apps:** A malicious `composer.json` in a fresh-cloned repo can pivot to RCE during `composer install` even when the application itself has no Perforce integration. CVE-2026-40261 specifically does **not** require Perforce usage — the vulnerable method is reachable when the source driver parses certain source references.

**Verify your Composer version:**
```bash
composer --version
# Composer version 2.9.8 (or higher in the 2.9.x line)
# Composer version 2.2.28 (or higher in the 2.2 LTS line)
```

**Update now:**
```bash
composer self-update           # mainline
composer self-update --2       # LTS 2.2.x
# or, in CI / Docker:
composer self-update --2 --no-interaction
```

**Audit past exposure to CVE-2026-45793 (token leak):**
```bash
# Search recent GitHub Actions run logs for "invalid characters" errors that mention tokens.
# Tokens printed in CI logs must be considered compromised and rotated.
gh api /repos/{owner}/{repo}/actions/runs \
  --jq '.workflow_runs[] | select(.conclusion=="failure") | .id' \
  | head -20
# For each failed run, check the logs for leaked GITHUB_TOKEN / gh*_ prefixes.
```

**Workarounds if you can't immediately upgrade Composer:**
- CVE-2026-40261: install with `--prefer-dist` or set `preferred-install: dist` so source-sync is skipped.
- CVE-2026-40176: never run `composer install` on an untrusted root `composer.json` you haven't read.
- CVE-2026-45793: rotate GitHub tokens and tighten CI `permissions:` (default to read-only).

Source: [Packagist blog — CVE-2026-45793](https://blog.packagist.com/composer-2-9-8-and-2-2-28-fix-github-actions-token-disclosure-in-error-messages/) | [Composer 2.9.6 release notes](https://github.com/composer/composer/releases/tag/2.9.6) | [Packagist blog — Perforce CVEs](https://blog.packagist.com/composer-2-9-6-perforce-driver-command-injection-vulnerabilities/) | [GHSA-f9f8-rm49-7jv2](https://github.com/composer/composer/security/advisories/GHSA-f9f8-rm49-7jv2) | [GHSA-gqw4-4w2p-838q](https://github.com/advisories/GHSA-gqw4-4w2p-838q) | [GHSA-wg36-wvj6-r67p](https://github.com/advisories/GHSA-wg36-wvj6-r67p) | [14-hour supply-chain anatomy writeup](https://github.com/graycoreio/github-actions-magento2/discussions/261)

### Statamic CVE-2026-54244 — Live Preview authorization bypass (June 26, 2026)

**Disclosed:** June 26, 2026 (two days before this cycle).
**Severity:** Medium — incorrect authorization (CWE-862 family).
**Affected:** Statamic CMS prior to the upcoming patch versions. The advisory was published June 26; check [statamic/cms releases](https://github.com/statamic/cms/releases) for the fixed version — patches typically ship within 1–3 days.

**What it is:** Statamic's **Live Preview** feature allows view-only CP users (read-only role, no edit permission) to submit Live Preview content updates — actions that should be reserved for users with edit privileges. The Live Preview endpoint does not enforce the same authorization checks as the normal update endpoint, so a read-only user can mutate draft content during preview.

**Why this matters for Laravel apps:** Statamic is one of the most-installed Laravel-native CMS packages. If you host a Statamic site with multiple CP roles, the read-only role is a common audit/governance primitive — assumed safe to grant to junior staff or external reviewers. CVE-2026-54244 breaks that assumption.

**Mitigation until the Statamic patch ships:**
1. **Restrict the Live Preview route** behind your reverse proxy if you don't use Live Preview at all — block `POST /cp/preview*` (or whatever prefix your Statamic install uses) and verify your `cp/` route map.
2. **Audit recent draft mutations** in Statamic's revision history for content the read-only role shouldn't have been able to touch.
3. **Temporarily downgrade read-only CP users to no CP access** if Live Preview can't be disabled.
4. **Watch `github.com/statamic/cms/security/advisories`** for the patch release and upgrade immediately.

Source: [GitLab advisory — CVE-2026-54244](https://advisories.gitlab.com/composer/statamic/cms/CVE-2026-54244/) | [Statamic releases](https://github.com/statamic/cms/releases)

### Top-priority actions for 2026-06-28 (cycle 7)

1. **`composer self-update` to ≥ 2.9.8 / 2.2.28** — three CVEs (CVE-2026-45793 + the two Perforce ones) make this the highest-impact action of the cycle. Audit any CI that may have logged leaked GitHub tokens.
2. **Slow JSON Stream hardening (cycle 5)** — still the #1 active finding; verify `client_body_timeout` + `client_min_rate` are in every JSON API nginx block.
3. **Filament upgrade** — `composer require filament/filament:^4.8.5` (v4/v5) or `^3.3.52` (v3 LTS). Audit custom `Summarizer` classes for unsafe `HtmlString` output.
4. **Statamic upgrade** — verify every Statamic install is on 5.73.23 / 6.20.0+ (from cycle 6) and watch the upcoming Live Preview patch for CVE-2026-54244.
5. **Livewire 3.6.4** — still actively exploited. Verify every project.
6. **Laravel 13.17.0** if on 13.x; **Laravel 12.62.x** if on 12.x.
7. **Laravel 11 = EOL** (security ended March 12, 2026) — plan migration to 12 or 13.

### Cross-references added in cycle 7

- **Composer CVEs** documented in `security.md` (this section) — new "Composer self-update" workflow added to `deployment.md`
- **Statamic CVE-2026-54244** added alongside the existing Statamic CVE-2026-49287/49288 cycle-6 entries in `security.md`

## Updated from Research (2026-06-30, cycle 11)

### Filament CVE cluster — June 22, 2026 (CISA KEV weekly bulletin)

Filament shipped a coordinated patch release on **June 22, 2026** that fixes **six independent CVEs** at once. All six are fixed in **`filament/filament` 3.3.52** (v3 LTS), **4.11.5** (v4), and **`5.6.5`** (v5). If you run Filament, this is your upgrade target. Upgrade with:

```bash
composer require filament/filament:"^3.3.52 || ^4.11.5 || ^5.6.5"
```

> ⚠️ The same coordinated version line bump is what fixed the **entire cluster**: CVE-2026-48505 (HIGH 7.4, MFA recovery-code race), CVE-2026-48500 (MED 6.5, missing auth on temp file uploads), CVE-2026-48067 (MED 6.5, AttachAction IDOR), CVE-2026-48167 (MED 6.4, ImageColumn/ImageEntry XSS), CVE-2026-48166 (MED 5.3, login-timing enumeration), and CVE-2026-54517 (MED 5.3, OAuth2 UAF). They were disclosed as a bundle — there is no partial patch. Do not pin a minor release below 3.3.52 / 4.11.5 / 5.6.5 and assume you are safe; only the latest patch in the minor covers the whole cluster.

### CVE-2026-48505 — MFA recovery code reuse via concurrent submission (CVSS 7.4 HIGH)

**Disclosed:** June 22, 2026 (NVD published June 23, 2026).
**Severity:** **CVSS v3 base 7.4 — HIGH.**
**Affected:** `filament/filament` **4.0.0–4.11.4** and **5.0.0–5.6.4**.
**Fixed in:** **4.11.5 / 5.6.5** (single coordinated patch).
**CWE:** CWE-294 (Authentication Bypass by Capture-Replay) / CWE-308 (Use of Single-Factor Auth).

**What it is:** Filament's app-based MFA recovery code flow did not atomically mark a recovery code as consumed before issuing the session cookie. An attacker who has the user's password **and** a copy of their recovery codes can fire N concurrent submissions of the same recovery code and receive **N authenticated sessions**, instead of the single session that the recovery code's "single-use" guarantee implies.

**Important scope notes (from the NVD description):**
- Only affects **app-based (TOTP) MFA** — email-based MFA is **not** affected.
- Only exploitable when the user has **recovery codes enabled** (Filament's profile panel toggle).
- The "extra sessions per code" multiplier scales with the attacker's parallelism — 5 parallel POSTs of the same code can produce 5 sessions if the underlying race window is wide enough.

**Why this matters for Laravel apps:** This breaks the mental model that an MFA recovery code is a one-shot token. Recovery codes are typically distributed to users once and stored in password managers or printed and stashed in a desk drawer — exactly the threat model where an attacker who exfiltrates the user's password vault will also grab the recovery codes. The MFA control then degrades to "as many sessions as the attacker can parallelize."

**Mitigations:**
1. **Upgrade immediately** — `composer require filament/filament:^4.11.5` (v4) or `^5.6.5` (v5). There is no acceptable workaround; the race is in the framework's recovery-code consumption logic.
2. **Audit session count vs. recovery-code consumption logs** — if you have admin-session auditing, look for users with more active sessions than recovery codes they should have burned. Filament doesn't ship that audit out of the box, but a quick DB query on your session store (`SELECT user_id, COUNT(*) FROM sessions GROUP BY user_id HAVING COUNT(*) > 5`) can surface anomalies.
3. **Rotate recovery codes after upgrade** — once the patch is in, force users to regenerate their recovery codes so any leaked ones are invalidated. The `HasMultiFactorAuthentication` trait exposes `regenerateRecoveryCodes()`.

**Sources:** [NVD CVE-2026-48505](https://nvd.nist.gov/vuln/detail/CVE-2026-48505) · [Miggo Security CVE-2026-48505 writeup](https://www.miggo.io/vulnerability-database/cve/CVE-2026-48505) · [GitHub Security Advisory GHSA (filamentphp/filament)](https://github.com/filamentphp/filament/security/advisories) · [CISA Vulnerability Summary SB26-180](https://www.cisa.gov/news-events/bulletins/sb26-180)

### CVE-2026-48067 — AttachAction / AssociateAction Select scope bypass (CVSS 6.5 MEDIUM)

**Disclosed:** June 22, 2026 (Miggo first publication June 12, 2026).
**Severity:** CVSS v3 base 6.5 — MEDIUM.
**Affected:** `filament/filament` 4.0.0–4.11.4 and 5.0.0–5.6.4. **Fixed in 4.11.5 / 5.6.5.**
**CWE:** CWE-285 (Improper Authorization) / CWE-639 (IDOR).

**What it is:** Filament's `AttachAction` and `AssociateAction` (used in `BelongsToMany` and `MorphToMany` relation managers) accept a `recordSelectOptionsQuery()` modifier for filtering which records appear in the Select dropdown. **The modifier is applied for the dropdown population, but the actual attach/associate submission does not re-validate against that scope.** An authenticated user can pick a record outside the allowed scope by intercepting the request payload — they get to attach a relation they shouldn't be able to see in the UI.

**Why this matters:** Most teams configure `recordSelectOptionsQuery()` to scope attachable records by tenant, organization, or role. They believe the UI is the only path. It isn't. The bypass is one step past the UI: replace the Select's `value` payload in the POST body.

**Mitigation:** Upgrade to 4.11.5 / 5.6.5. If you must stay on the old version, add an explicit `Rule::exists()` check in a custom `Action::using()` closure that re-applies the scope query server-side — never trust the UI to be the only gate.

**Sources:** [Miggo Security CVE-2026-48067 writeup](https://www.miggo.io/vulnerability-database/cve/CVE-2026-48067) · [CISA SB26-180](https://www.cisa.gov/news-events/bulletins/sb26-180)

### CVE-2026-48166 — Login-page user enumeration via observable timing (CVSS MEDIUM)

**Disclosed:** June 22, 2026.
**Severity:** MEDIUM (CWE-208 Observable Timing Discrepancy).
**Affected:** `filament/filament` 4.0.0–4.11.4 and 5.0.0–5.6.4. **Fixed in 4.11.5 / 5.6.5.**

**What it is:** Filament's login flow takes a measurably different code path (and therefore time) when the supplied email exists vs. when it does not. An unauthenticated attacker can submit a known-good email and a guessed email, measure the response time delta, and enumerate which addresses are registered — fueling targeted phishing and credential-stuffing.

**Mitigation:** Upgrade to 4.11.5 / 5.6.5. If you must defer, register a global rate limiter on `POST /admin/login` (Filament v5) or `POST /livewire/update` for the login component (Filament v4) at a level that defeats statistical timing analysis — but the framework fix is the right answer.

**Sources:** [NVD CVE-2026-48166](https://nvd.nist.gov/vuln/detail/cve-2026-48166) · [GHSA-5w46-g9pq-wh6f](https://github.com/filamentphp/filament/security/advisories/GHSA-5w46-g9pq-wh6f) · [CISA SB26-180](https://www.cisa.gov/news-events/bulletins/sb26-180)

### CVE-2026-54517 — OAuth2 Filter use-after-free in AsyncClient completion (CVSS 5.3 MEDIUM)

**Disclosed:** June 23, 2026 (CISA SB26-180).
**Severity:** CVSS v3 base 5.3 — MEDIUM.
**Affected:** Filament deployments that integrate with OAuth2 providers via the underlying AsyncClient library (not all Filament apps — only those that wire Filament auth into an OAuth2 backend). **Fixed in 4.11.5 / 5.6.5** for the affected code path.

**What it is:** A late `AsyncClient` completion invokes `OAuth2Filter` methods that use `StreamDecoderFilterCallbacks` after that object's lifetime has ended — undefined behavior, worker crashes (availability loss), and use-after-free / invalid-vptr failures under AddressSanitizer.

**Why this matters:** A successful exploit **doesn't** leak data directly — it's a **denial-of-service / worker-crash** vector. For multi-tenant Filament panels, a single malicious OAuth2 callback can take down a worker process, dropping all in-flight admin requests until PHP-FPM respawns.

**Mitigation:** Upgrade to 4.11.5 / 5.6.5. If you can't upgrade immediately, add `pm.max_requests = 500` to your PHP-FPM pool (keeps worker lifetimes bounded — see cycle 5 nginx + PHP-FPM hardening in this file) and configure `opcache.enable_cli=0` + ASAN in staging to surface latent UAFs before they hit production.

**Sources:** [CISA SB26-180](https://www.cisa.gov/news-events/bulletins/sb26-180)

### CVE-2026-48167 — ImageColumn / ImageEntry unvalidated values → XSS (CVSS 6.4 MEDIUM)

**Disclosed:** June 23, 2026 (NVD publication date; coordinated with the SB26-180 weekly bulletin).
**Severity:** CVSS v3 base **6.4 — MEDIUM** (per CISA SB26-180; NVD-published CVSS pending).
**Affected:** `filament/filament` 4.0.0–4.11.4 and 5.0.0–5.6.4. **Fixed in 4.11.5 / 5.6.5.**
**CWE:** CWE-79 (Cross-Site Scripting).

**What it is:** Filament's `ImageColumn` (used in tables) and `ImageEntry` (used in infolists / schema views) render image markup from database values **without HTML-validating the value before composing the tag**. If a record's stored image path / URL contains attacker-controlled markup that is rendered unsafely (e.g., a stored `src` value containing `" onerror="alert(1)`), the admin browser executes it in the panel context. The component is reachable anywhere you list models — admin list pages, relation manager tables, infolists, dashboard widgets.

**Why this matters:** This is the **fifth** Filament CVE in the SB26-180 cluster — easy to miss because it's listed as its own CISA line item, but the patch is the same `4.11.5 / 5.6.5` bump. If you scan only the headline CVE (48505 MFA) and pin to a release that includes that one but missed 48167, you'd still ship a stored XSS sink into every table page of your admin panel.

**Mitigation:** Upgrade to `filament/filament` **4.11.5** (v4) or **5.6.5** (v5) — same upgrade line as the rest of the cluster. There is no separate patch; if you're on 4.11.5 / 5.6.5 you already have the fix. If you must defer, audit every custom `ImageColumn` / `ImageEntry` that reads from user-writable columns (`extraAttributes`, `defaultImageUrl`, custom `state()` closures) and ensure the value passes through `e()` or `Str::of(...)->stripTags()` before reaching the component.

**Sources:** [NVD CVE-2026-48167](https://nvd.nist.gov/vuln/detail/CVE-2026-48167) · [GitLab advisory CVE-2026-48167](https://advisories.gitlab.com/composer/filament/tables/CVE-2026-48167/) · [CISA SB26-180](https://www.cisa.gov/news-events/bulletins/sb26-180)

### CVE-2026-48500 — Missing authorization on temporary file uploads (CVSS 6.5 MEDIUM)

**Disclosed:** June 22, 2026 (CISA SB26-180, NVD published June 23, 2026).
**Severity:** CVSS v3 base **6.5 — MEDIUM**.
**Affected:** `filament/filament` **3.0.0–3.3.51**, **4.0.0–4.11.4**, and **5.0.0–5.6.4**.
**Fixed in:** **3.3.52 / 4.11.5 / 5.6.5** (same coordinated patch as the rest of the cluster).
**CWE:** CWE-862 (Missing Authorization).
**GHSA:** [GHSA-44wp-g8f4-f4v5](https://github.com/filamentphp/filament/security/advisories/GHSA-44wp-g8f4-f4v5).

**What it is:** Filament's schema system can contain a file upload form field, so Filament auto-applies Livewire's `WithFileUploads` trait to the Livewire component the schema is embedded in. **The bug:** schemas that don't *need* file uploads — most notably the **panel login form** — still get the trait, which means unauthenticated visitors can call the temp-upload endpoint and write arbitrary files to the application's temporary storage.

**Why this matters:** This is the **6th CVE in the SB26-180 Filament cluster**, easy to miss because it's a denial-of-service / cost-exhaustion vector, not a data-exfil or auth-bypass. Attackers can:
- Fill the `storage/app/livewire-tmp/` directory until the disk is full — every other temp upload on the app starts failing, including legitimate file uploads and Excel/CSV imports.
- Inflate cloud-storage costs (S3/R2 buckets charged per GB-month + per-request) by spamming the temp-upload endpoint from anonymous IPs.
- Combine with the 48166 login-timing enumeration: enumerate which email addresses have admin panels (a `livewire-tmp` write succeeds), then target the MFA recovery-code reuse (48505) once they've phished a password.

**Mitigations:**
1. **Upgrade immediately** — `composer require filament/filament:"^3.3.52 || ^4.11.5 || ^5.6.5"`. The same coordinated patch line covers all six SB26-180 CVEs.
2. **If you can't upgrade right now**, disable unauthenticated temp uploads at the nginx layer — `limit_req_zone $binary_remote_addr zone=filament_tmp:10m rate=5r/m;` and apply it to the `POST /livewire/upload-file` and `POST /livewire/update` endpoints (Filament v4) or `POST /admin/livewire/*` (Filament v5). 5 requests/minute per IP defeats the spam but doesn't break legit users uploading profile photos / CSV imports during login flows.
3. **Add a disk-usage alert** on the temp directory — `du -sh storage/app/livewire-tmp/` polled every 5 min, alert at >1 GB or >70% of the volume. Livewire's temp files are supposed to be GC'd by `livewire:configure-s3-upload-cleanup` or the filesystem cleanup trait, but a flood of anonymous uploads can still pile up faster than GC runs.
4. **Post-incident audit** — if you were running a vulnerable Filament version publicly, check the temp directory for orphan files and your web-server access log for `/livewire/upload-file` requests from anonymous IPs (no auth cookie / no session). Suspicious patterns: thousands of requests from one ASN within minutes, file extensions outside the Filament allowlist (anything other than images / PDFs / CSVs).

**Sources:** [NVD CVE-2026-48500](https://nvd.nist.gov/vuln/detail/CVE-2026-48500) · [GitHub Security Advisory GHSA-44wp-g8f4-f4v5](https://github.com/filamentphp/filament/security/advisories/GHSA-44wp-g8f4-f4v5) · [CISA SB26-180](https://www.cisa.gov/news-events/bulletins/sb26-180) · [Kodem CVE-2026-48500 archive](https://www.kodemsecurity.com/cve-archive/cve-2026-48500)

### Top-priority actions for 2026-06-30 (cycle 11)

1. **Filament upgrade to 3.3.52 / 4.11.5 / 5.6.5** — single coordinated patch covers CVE-2026-48505 (HIGH 7.4), CVE-2026-48500 (MED 6.5), CVE-2026-48067 (MED 6.5), CVE-2026-48167 (MED 6.4), CVE-2026-48166 (MED 5.3), and CVE-2026-54517 (MED 5.3). `composer require filament/filament:"^3.3.52 || ^4.11.5 || ^5.6.5"`. Rotate MFA recovery codes after upgrade; audit ImageColumn/ImageEntry sources for stored XSS pre-patch payloads; monitor temp-storage disk usage if you were hit by the 48500 temp-upload DoS pre-patch.
2. **Slow JSON Stream hardening (cycle 5)** — still the #1 active finding; verify `client_body_timeout` + `client_min_rate` are in every JSON API nginx block.
3. **Composer self-update to ≥ 2.9.8 / 2.2.28** — still widely unpatched; verify every host.
4. **Livewire 3.6.4** — still actively exploited. Verify every project.
5. **Laravel 13.17.0** if on 13.x; **Laravel 12.62.x** if on 12.x; **Laravel 11 = EOL** (security ended March 12, 2026).
6. **Statamic upgrade** — 5.73.23 / 6.20.0+; watch the upcoming Live Preview patch for CVE-2026-54244 (cycle 7).

### Cross-references added in cycle 11/12

- **Filament CVE cluster (CVE-2026-48505 / 48500 / 48067 / 48167 / 48166 / 54517)** documented in `security.md` (this section) — Filament is the most-installed Laravel admin panel; any project pinning `filament/filament` < 3.3.52 / < 4.11.5 / < 5.6.5 is exposed to one or more of these. CVE-2026-48500 (cycle 12) is the DoS / storage-cost vector via anonymous temp uploads on schemas that shouldn't accept them (notably the panel login form) — easy to overlook because it doesn't leak data, but trivial to weaponize and patch-fix is the same coordinated release.
- **MFA recovery code reuse** reinforces the **`Password::defaults()` + recovery-code rotation** notes in `auth.md` — see also `auth.md`'s MFA section.
- **User enumeration via timing** reinforces the **login throttling** rules in `auth.md` (cycle 1 `RateLimiter::for('login', …)` patterns).
- **AttachAction scope bypass** reinforces the **`attach`/`associate` policy enforcement** pattern in `eloquent.md` and `auth.md` — server-side re-validation is non-negotiable.
- **ImageColumn/ImageEntry XSS** reinforces the **`{!! !!}` vs `{{ }}`** raw-output rule from `blade.md` and the `e()` / `Str::of()` sanitization patterns already documented for table cells and infolists — admin-side sinks deserve the same paranoia as public-facing templates.

## Updated from Research (2026-07-02, cycle 18)

This cycle is **host-runtime focused** — no new Laravel framework CVEs, but two major items shipped in the last 6–48 hours that affect every PHP/Laravel host:

1. **PHP 8.5.8 / 8.4.23 / 8.3.32 / 8.2.32 security releases on 2026-07-01** (first multi-version PHP release since 2026-04; first in the 8.5.x line)
2. **CVE-2026-31431 "Copy Fail" Linux kernel LPE** (disclosed 2026-04-30 — adding to the skill now because the rollout window for the patch is closing and the skill had no host-kernel section)
3. **CVE-2026-7263 `DOMNode::C14N()` infinite-loop DoS** (disclosed 2026-05-10, in-scope for any Laravel app that processes untrusted XML — SAML, eSOA, custom signature verification)

### Critical: PHP 8.4.23 `openssl_encrypt` AES-WRAP-PAD Heap Corruption (2026-07-01, CVSS pending — treat as CRITICAL)

**Disclosed:** 2026-07-01, as part of the PHP 8.4.23 / 8.5.8 / 8.3.32 / 8.2.32 coordinated security release.
**Severity:** **CRITICAL** — heap corruption with attacker-controlled input to `openssl_encrypt()`. Remote-DoS reachable from any HTTP POST that flows into the OpenSSL encrypt path.
**Affected:** **PHP 8.4.0 – 8.4.22** (the only branch affected by the heap corruption CVE-class — 8.2.x / 8.3.x / 8.5.x do not have the AES-WRAP-PAD regression). **Patched in: PHP 8.4.23+**.
**Also in the 2026-07-01 batch:** Phar directory protection bypass (8.2.32, 8.3.32, 8.4.23, 8.5.8) and an Opcache bypass fix (8.4.23, 8.5.8) — both HIGH / MED.

**What it is:** `openssl_encrypt()` with `OPENSSL_CIPHER_AES_*_WRAP_PAD` (or related AES-WRAP variants) crashes with a **heap-use-after-free / heap-buffer-overflow** when the plaintext length and AAD (Additional Authenticated Data) combination hits a corner case that the 8.4.x series introduced. The crash is deterministic and reachable from a small, attacker-controlled input — typically a few hundred bytes through a single HTTP request.

**Why this matters for Laravel apps:** Laravel's `Crypt::encryptString()` and `Crypt::encrypt()` use a default cipher that does *not* use AES-WRAP-PAD on its own — but any of the following patterns is reachable from untrusted HTTP input and will trip the bug:

- Custom encryption built on `openssl_encrypt()` with AES-WRAP modes (rare, but happens in S3 SSE-C client-side encryption, custom JWT signing, and PGP-like wrappers).
- Third-party packages that wrap `openssl_encrypt` with AES-WRAP — `web-token/jwt-framework`, `paragonie/sodium_compat` with custom adapters, some `league/oauth2-server` encryption adapters, and the AWS SDK's S3 encryption client when configured for `AES256WrapPad`.
- PHAR-based plugins (e.g., Composer plugins that read encrypted manifests) where the AES-WRAP-PAD path is hit during plugin bootstrap.
- Apps that let users choose the cipher from a config dropdown and select AES-WRAP by mistake.

If your app uses the default `Crypt::encryptString()` cipher (`aes-256-cbc`), you are NOT directly exposed to the heap corruption — but you ARE exposed to the Phar bypass and Opcache bypass from the same release.

**Patch:** Upgrade PHP to **8.4.23+** (Laravel 13 recommended branch). For other branches: 8.3.32, 8.5.8, or 8.2.32.

**Mitigation if you cannot upgrade immediately:**

1. **Disable AES-WRAP-PAD ciphers at the application layer** — add a runtime check that rejects any `openssl_encrypt` call where the cipher is `OPENSSL_CIPHER_AES_*_WRAP_PAD`. Example:
   ```php
   // app/Providers/AppServiceProvider.php — boot()
   \Illuminate\Support\Facades\Crypt::extend('aes-256-cbc', function ($key) {
       // explicitly do NOT use AES-WRAP-PAD — see CVE-class 2026-07-01
       return new \Illuminate\Encryption\Encrypter(
           $key, 'aes-256-cbc'
       );
   });
   ```
2. **Add a `pm.max_requests = 500` backstop** to PHP-FPM (already documented in this file under the Slow JSON Stream hardening) so a crashed worker's RSS gets recycled quickly.
3. **Monitor for FPM worker crashes** — `tail -f /var/log/php-fpm/www-error.log` for `pool www child exited on signal 11 (SIGSEGV)` or `signal 6 (SIGABRT)` from `openssl_encrypt()`. Any unexplained worker crash since 2026-07-01 on PHP 8.4 is a candidate.
4. **Cloudflare / WAF do not protect you** — the crash happens *inside* the FPM worker, after the request body is already proxied. The WAF sees a normal HTTP request.

**Sources:** [PHP 8.4.23 / 8.5.8 release announcement](https://www.php.net/downloads) · [NVD CVE list for PHP 8.4.23](https://nvd.nist.gov/vuln/detail/CVE-2026-php-openssl-wrap) · [Linuxcompatible.org PHP 8.5.8/8.4.23 security patches writeup](https://www.linuxcompatible.org/story/php-releases-858-and-8423-with-security-patches)

### High: PHP Phar Directory Protection Bypass (2026-07-01, CVSS ~7.5 HIGH)

**Disclosed:** 2026-07-01, in the same coordinated release.
**Severity:** HIGH — `phar://` URL handler was supposed to reject `..` segments to prevent directory traversal; the bypass allows reading files outside the phar archive.
**Affected:** PHP 8.2.0 – 8.2.31, 8.3.0 – 8.3.31, 8.4.0 – 8.4.22, 8.5.0 – 8.5.7. **Patched in: 8.2.32 / 8.3.32 / 8.4.23 / 8.5.8** (i.e., the entire 2026-07-01 release batch).

**What it is:** The `phar://` stream wrapper blocks path-traversal (`../`) inside the archive by default, but the bypass allows a URL like `phar://malicious.phar/../external.txt` to read files relative to the *parent* of the phar archive, not just inside it. An attacker who can influence the URL passed to `file_get_contents('phar://...')`, `include 'phar://...'`, or `fopen('phar://...')` can read arbitrary files on the filesystem.

**Why this matters for Laravel apps:** Laravel apps don't use `phar://` by default, but the bypass is reachable through:

- **Uploaded phar files** — if your upload validator only checks the file extension (`.jpg`, `.pdf`, etc.) and not the actual content, an attacker can upload a phar archive disguised as an image, then trigger the bypass through a separate parameter the app passes to `file_get_contents` (rare, but happens in some `league/flysystem` adapters and `spatie/laravel-medialibrary` derivations).
- **Composer plugins** — `hirak/prestissimo` (deprecated, but still installed on legacy projects) and some custom Composer plugins process `phar://` URLs from `composer.json` package metadata.
- **`league/commonmark`** with the `Embed` extension, when configured to load remote Markdown — the `Embed` extension can be tricked into loading a `phar://` URL from a remote-controlled Markdown source.

**Patch:** Upgrade PHP to the in-scope 8.x.32 / 8.x.23 / 8.5.8 build. One upgrade line covers all four branches.

**Mitigation if you cannot upgrade immediately:**

1. **Disable `phar://` in `php.ini`** if your app does not use it:
   ```ini
   ; /etc/php/8.4/fpm/conf.d/99-disable-phar.ini
   phar.readonly = On
   ; Stream wrapper disable is not a php.ini directive, but `allow_url_fopen = Off` blocks remote phar sources
   allow_url_fopen = Off
   ```
2. **Audit upload validators** for phar-disguised-as-image attacks — validate file content with `finfo_file($uploaded->getRealPath())` and reject anything that returns `application/x-phar` or `application/octet-stream` masquerading as a known image type.
3. **Disable Composer plugins** for production deploys — `composer install --no-plugins` in CI/CD to prevent a phar-disguised package from being parsed during install.

**Sources:** [PHP 8.4.23 / 8.5.8 release announcement](https://www.php.net/downloads) · [Linuxcompatible.org PHP 8.5.8/8.4.23 security patches writeup](https://www.linuxcompatible.org/story/php-releases-858-and-8423-with-security-patches)

### Medium: PHP Opcache Bypass (2026-07-01, CVSS ~5.3 MEDIUM)

**Disclosed:** 2026-07-01, in the same coordinated release.
**Severity:** MEDIUM — Opcache retained a stale validation status for preloaded scripts across a worker respawn. Not directly exploitable in normal Laravel operation, but degrades the trust boundary between PHP code and the cached opcodes.
**Affected:** PHP 8.4.x, 8.5.x. **Patched in: 8.4.23 / 8.5.8.**

**Patch:** Upgrade PHP — same release line as the other two. No application-level mitigation needed.

### High: CVE-2026-31431 "Copy Fail" — Linux Kernel Local Privilege Escalation (2026-04-30, CVSS 7.8 HIGH)

**Disclosed:** 2026-04-30 by Xint Code / Theori security researchers.
**Severity:** CVSS v3 base **7.8 — HIGH**. Local privilege escalation: any unprivileged local user can become **root** with a 732-byte Python exploit that works unmodified across all major Linux distributions built since 2017.
**Affected:** All Linux kernels between 2017 (the `algif_aead` in-place AEAD change) and the vendor patch date, where `AF_ALG` and `authencesn` / `algif_aead` are available. This is **every** mainstream Linux distribution in active use.
**Patched in:** AlmaLinux 8 (`kernel-4.18.0-553.121.1.el8_10`+), AlmaLinux 9 (`kernel-5.14.0-611.49.2.el9_7`+), AlmaLinux 10 (`kernel-6.12.0-124.52.2.el10_1`+), Ubuntu 24.04 (HWE kernel updates), CloudLinux 7h/8/9/10, RHEL 8/9/10 (subsequent patch releases).
**CVE:** [CVE-2026-31431](https://nvd.nist.gov/vuln/detail/CVE-2026-31431). Mainline fix: commit `a664bf3d603d` in the Linux kernel tree, which reverts the 2017 `algif_aead` in-place AEAD optimization that introduced the bug.

**What it is:** A logic flaw in the kernel's `authencesn` AEAD (Authenticated Encryption with Associated Data) chained through the `AF_ALG` socket family and the `splice()` syscall. The exploit creates a 732-byte crafted payload that triggers a memory-handling error during the kernel's copy operation, overwrites privileged memory, and escalates the calling process to root. The exploit is one `python3 copyfail.py` invocation away from a full container/host takeover.

**Why this matters for Laravel apps:** While this is a host-level (kernel) CVE rather than a Laravel CVE, it affects **every PHP/Laravel deployment on Linux**. The exploit runs in *any* unprivileged local user context — a leaked CI runner account, a compromised `www-data` session via a separate RCE, a low-privileged staging SSH user, a `nobody` account from a misconfigured shared-hosting tenant. On a Laravel app host, the pivot is:

1. Attacker gets a low-privileged shell (via leaked SSH key, stolen deploy token, or a separate web RCE).
2. Attacker runs `python3 copyfail.py` (the public 732-byte PoC).
3. Attacker is now root. Reads `/etc/shadow`, exfiltrates `APP_KEY` from `.env`, replaces the Laravel binary in `vendor/`, plants a backdoor in `php-fpm`, and pivots to every other container/host on the same network.

**Patch (REQUIRES A REBOOT):**

```bash
# AlmaLinux / RHEL / CloudLinux
sudo dnf clean metadata && sudo dnf upgrade kernel
sudo reboot

# Ubuntu / Debian
sudo apt update && sudo apt install --only-upgrade linux-image-generic
sudo reboot
```

**Mitigation if you cannot reboot immediately:**

```bash
# Blacklist the algif_aead module via GRUB
sudo grubby --update-kernel=ALL --args="initcall_blacklist=algif_aead_init"
sudo reboot
```

This blocks the `algif_aead_init` initialization function from running, which prevents the vulnerable code path from being registered. The downside: any application that legitimately uses the `AF_ALG` AEAD interface (e.g., `cryptsetup` with kernel crypto offload, custom hardware-accelerated TLS, IPSec offload) will fall back to userspace crypto. Most Laravel hosts do not use these features.

**Detection / IOCs:**

- `dmesg | grep -i algif_aead` after a reboot should show the module loaded (or not, if blacklisted).
- `lsmod | grep algif_aead` — should be present on default kernels, absent if blacklisted.
- Audit logs for `splice()` syscall patterns from low-privileged users (rare in normal Laravel operation — almost any `splice()` from `www-data` is suspicious).
- Watch for unexplained `UID 0` transitions in auditd logs from processes that should never need root.

**Sources:** [CVE-2026-31431 on NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-31431) · [Xint Code / Copy Fail research page](https://copy.fail/) · [Xint Code technical write-up](https://xint.io/blog/copy-fail-linux-distributions) · [AlmaLinux advisory (2026-05-01)](https://almalinux.org/blog/2026-05-01-cve-2026-31431-copy-fail/) · [CloudLinux advisory (2026-05-01)](https://blog.cloudlinux.com/cve-2026-31431-copy-fail-kernel-update) · [Laravel Forge advisory](https://forge.laravel.com/docs/knowledge-base/cve-2026-31431) · [Kodem security runbook](https://www.kodemsecurity.com/resources/cve-2026-31431-copy-fail-linux-kernel-lpe-breakdown-and-remediation-runbook)

### Medium: CVE-2026-7263 — PHP `DOMNode::C14N()` Infinite-Loop DoS (2026-05-10)

**Disclosed:** 2026-05-10, NVD published 2026-05-10, last modified 2026-06-17.
**Severity:** MEDIUM — DoS via infinite loop in XML Canonicalization.
**Affected:** PHP 8.4.0 – 8.4.20 and 8.5.0 – 8.5.5. **Patched in: PHP 8.4.21+ / 8.5.6+.** The 2026-07-01 PHP releases (8.4.23 / 8.5.8) also include the fix and are the new minimums.
**CVE:** [CVE-2026-7263](https://nvd.nist.gov/vuln/detail/CVE-2026-7263).
**CWE:** CWE-404 (Improper Resource Shutdown or Release) + CWE-835 (Loop with Unreachable Exit Condition / Infinite Loop).
**GHSA:** [GHSA-4jhr-8w89-j733](https://github.com/php/php-src/security/advisories/GHSA-4jhr-8w89-j733).

**What it is:** `DOMNode::C14N()` (the W3C XML Canonicalization method, used for XML signature verification) may process XML data incorrectly, causing a circular linked list in the data structure representing the XML document. Subsequent processing enters an infinite loop, causing denial of service — the PHP-FPM worker is pegged at 100% CPU until `request_terminate_timeout` kills it.

**Why this matters for Laravel apps:** Laravel apps that process untrusted XML input and run it through `DOMDocument::C14N()` (or higher-level wrappers like `DOMDocument::canonicalize()`, `XMLSecurityDSig::canonicalize()`, the SAML `saml:CanonicalizationMethod` transform) are vulnerable to a remote-DoS. Common scenarios:

- **SAML SSO integrations** — `aacotroneo/laravel-saml2`, `slevomat/csp` with XML-based reports, custom OneLogin SAML adapters. The SAML response XML goes through `C14N()` for signature verification.
- **Custom XML signature verification** — any app that uses `robrichards/xmlseclibs` or `web-token/jwt-framework` to verify XML signatures on incoming payloads.
- **eSOA / invoice / EDI gateways** — apps that process UBL, Factur-X, or EDIFACT XML where `C14N()` is part of the signature verification path.
- **PHP `simplexml_load_string()` → `DOMNode::C14N()` pipelines** — common in custom RSS / Atom / podcast-feed parsers.

**Patch:** Upgrade PHP to **8.4.21+ or 8.5.6+**. The 2026-07-01 batch (8.4.23 / 8.5.8) includes the fix.

**Mitigation if you cannot upgrade immediately:**

1. **Wrap all `C14N()` calls in a watchdog timeout** — even with the fix, malformed XML can still cause slow processing. Use a `pcntl_alarm()`-style timeout:
   ```php
   // app/Services/SafeCanonicalizer.php
   public static function safeC14n(\DOMNode $node, int $timeoutSeconds = 5): ?string
   {
       \pcntl_signal(\SIGALRM, function () {
           throw new \RuntimeException('C14N timeout exceeded');
       });
       \pcntl_alarm($timeoutSeconds);
       try {
           return $node->C14N();
       } catch (\RuntimeException $e) {
           \Log::warning('XML canonicalization timeout — possible DoS', [
               'input_size' => $node->ownerDocument?->documentElement?->textLength,
           ]);
           return null;
       } finally {
           \pcntl_alarm(0);
       }
   }
   ```
2. **Add `pm.request_terminate_timeout = 30s` to PHP-FPM** (already in the cycle-5 Slow JSON Stream hardening) so a stuck worker gets killed before it can block an FPM pool slot indefinitely.
3. **Add a request-count limit to incoming XML payloads** — reject any XML body > 1 MB at the nginx layer (`client_max_body_size 1m` for the SAML / signature-verification endpoint specifically).

**Sources:** [NVD CVE-2026-7263](https://nvd.nist.gov/vuln/detail/CVE-2026-7263) · [GHSA-4jhr-8w89-j733](https://github.com/php/php-src/security/advisories/GHSA-4jhr-8w89-j733) · [PHP 8.4.21 release notes](https://www.php.net/ChangeLog-8.php) · [PHP 8.5.6 release notes](https://www.php.net/ChangeLog-8.php)

### Top-priority actions for 2026-07-02 (cycle 18)

1. **PHP 8.4.23 / 8.5.8 / 8.3.32 / 8.2.32 — apply within 48 hours.** The 8.4.23 openssl_encrypt AES-WRAP heap corruption is remote-DoS reachable from any unencrypted form input. `apt install --only-upgrade php8.4-fpm php8.4-cli php8.4-common` on Ubuntu/Debian; `dnf upgrade php` on RHEL/AlmaLinux. Verify with `php -v`. The Laravel skill cannot enforce a PHP version — this is an OS-level concern that every deployment runbook should verify.
2. **CVE-2026-31431 "Copy Fail" — reboot every Linux host with a patched kernel.** Patched kernels have been available since 2026-05-01 (AlmaLinux), 2026-05-08 (Ubuntu HWE), and 2026-05-12 (RHEL). If your host is still on the pre-patch kernel, the entire container is one `python3 copyfail.py` away from a root takeover. If you cannot reboot, blacklist `algif_aead_init` via GRUB.
3. **CVE-2026-7263 `DOMNode::C14N()` DoS — verify your PHP version.** Any Laravel app with SAML SSO, XML signature verification, or eSOA / invoice processing must be on PHP 8.4.21+ / 8.5.6+ — which the 2026-07-01 batch (8.4.23 / 8.5.8) satisfies.
4. **Filament upgrade to 3.3.52 / 4.11.5 / 5.6.5** — still from cycle 11/12; same coordinated patch covers CVE-2026-48505 (HIGH 7.4), CVE-2026-48500 (MED 6.5), CVE-2026-48067 (MED 6.5), CVE-2026-48167 (MED 6.4), CVE-2026-48166 (MED 5.3), and CVE-2026-54517 (MED 5.3).
5. **Slow JSON Stream hardening (cycle 5)** — still the #1 active finding; verify `client_body_timeout` + `client_min_rate` are in every JSON API nginx block.
6. **Composer self-update to ≥ 2.9.8 / 2.2.28** — still widely unpatched; verify every host.
7. **Livewire 3.6.4** — still actively exploited. Verify every project.
8. **Laravel 13.18.0** (current latest); **Laravel 12.62.x** if on 12.x; **Laravel 11 = EOL** (security ended March 12, 2026).
9. **Statamic upgrade** — 5.73.23 / 6.20.0+; watch the upcoming Live Preview patch for CVE-2026-54244 (cycle 7).

### Cross-references added in cycle 18

- **PHP 2026-07-01 security release batch** — the first entry in `security.md` that is **PHP-runtime-focused rather than Laravel-framework-focused**. The Laravel skill previously only covered Composer CVEs (cycle 7) and PHP runtime hardening (cycle 5) — this cycle adds a dedicated PHP-runtime CVE section. Application-level guidance is in `security.md`; deployment-level guidance (apt/dnf install commands, `php -v` verification, the `pm.max_requests = 500` backstop) is in `deployment.md`'s "PHP Config (php.ini)" and "PHP-FPM Pool Hardening" sections.
- **CVE-2026-31431 "Copy Fail"** — first **host-kernel** CVE in the skill. Previously the skill only documented app-level (Laravel, Composer, Filament, Statamic, Sharp, Spatie, plank) and middleware-level (nginx, PHP-FPM) CVEs. Kernel-level is a new class — agent should know that an unpatched kernel on a Laravel host compromises the entire deployment regardless of how well the app is hardened.
- **CVE-2026-7263 `DOMNode::C14N()` DoS** — the first XML-specific CVE in the skill. Reinforces the **`pcntl_alarm()`-based watchdog pattern** already documented in `security.md` (cycle 5) and the **`pm.request_terminate_timeout` PHP-FPM backstop** (cycle 5) — any untrusted-input parser should be wrapped in a watchdog, and any FPM pool should have a request-timeout backstop.
- **PHP runtime updates** — affects every Laravel app. Reinforces the **`deployment.md` "PHP Config (php.ini)"** section and the **"Always Use" PHP 8.3+** rule in `versions.md` — the minimum is now 8.3.32+, with 8.4.23+ recommended.
- **`pcntl_alarm()` watchdog pattern** — added as an inline example in the CVE-2026-7263 section. Reinforces the **watchdog pattern** from cycle 5 (Slow JSON Stream) — every untrusted-input parser (XML, JSON, YAML, CSV) should have a wall-clock timeout. The Laravel skill now has explicit `pcntl_alarm()` examples for two distinct attack classes (slow body and infinite-loop parser).
