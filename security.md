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

// DANGEROUS — string interpolation (never do this)
User::whereRaw("email = '$email'")->first(); // SQL injection
DB::raw("SELECT * FROM users WHERE email = '$email'"); // SQL injection
```

**Always use query builder bindings, never string interpolation.**

## Mass Assignment Protection

```php
// Always define fillable
protected $fillable = ['title', 'body', 'author_id'];
protected $guarded = []; // blocks all except fillable

// Never leave unguarded
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

## File Upload Security

```php
// Validate file type and size
$request->validate([
    'avatar' => 'required|image|mimes:jpeg,png,gif,webp|max:2048',
]);

// Store with unique name
$path = $request->file('avatar')->store('avatars', 'public');

// Never trust user-provided file extensions
$extension = $request->file('avatar')->getClientOriginalExtension(); // NOT safe
$path = $request->file('avatar')->storeAs('avatars', bin2hex(random_bytes(16)) . '.jpg', 'public');
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

// Or validate input strictly
if (!preg_match('/^[a-zA-Z0-9_.-]+$/', $userInput)) {
    abort(400, 'Invalid characters');
}
```

## HTTPS & Headers

```php
// Always force HTTPS in production (AppServiceProvider)
if (app()->environment('production')) {
    URL::forceScheme('https');
}

// Security headers (bootstrap/app.php or middleware)
$middleware->append(function ($request, $next) {
    $response = $next($request);
    $response->headers->set('X-Content-Type-Options', 'nosniff');
    $response->headers->set('X-Frame-Options', 'DENY');
    $response->headers->set('X-XSS-Protection', '1; mode=block');
    $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
    return $response;
});
```

## Environment Security

```php
// .env never in version control
.env
.env.local
.env.production

// Ensure .gitignore has:
.env
.env.*
*.key
```

## API Authentication

```php
// Use Sanctum for token auth, never roll your own
// Sanctum handles token storage, expiration, revocation

// If you must use API keys:
$apiKey = $request->header('X-API-Key');
$user = User::where('api_key', hash('sha256', $apiKey))->first();
```

## Common Mistakes

1. **`{!! !!}` with user content** — XSS vulnerability, always sanitize first
2. **String interpolation in queries** — SQL injection, always use bindings
3. **No $fillable/$guarded** — mass assignment vulnerability
4. **User input in shell commands** — command injection, validate or use Symfony Process
5. **Plaintext passwords** — always `Hash::make()`
6. **Exposing stack traces in production** — debug=false in production .env