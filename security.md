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
