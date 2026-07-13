# Artisan — CLI Commands, Scheduling, Tinker

## Essential Commands

```bash
# Serve (dev)
php artisan serve --port=8080 --host=0.0.0.0

# Clear caches
php artisan config:clear
php artisan cache:clear
php artisan route:clear
php artisan view:clear
php artisan optimize:clear

# Config cache (production — do AFTER deploy)
php artisan config:cache      # compile config to bootstrap/cache/config.php
php artisan route:cache      # cache routes (no closures in routes!)
php artisan view:cache       # compile Blade views

# Only run these in deployment scripts, never in dev — they break without careful setup
php artisan optimize         # runs config:cache + route:cache + view:cache

# Rebuild
php artisan config:clear && php artisan optimize
```

**NEVER run `route:cache` with closures in routes — breaks silently.**

## Custom Commands

```bash
php artisan make:command ProcessPostsCommand
```

```php
// app/Console/Commands/ProcessPostsCommand.php
namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Models\Post;
use Illuminate\Support\Facades\Log;

class ProcessPostsCommand extends Command
{
    protected $signature = 'posts:process {--dry-run : Show what would be processed}';
    protected $description = 'Process pending posts';

    public function handle(): int
    {
        $query = Post::where('status', 'pending');

        if ($this->option('dry-run')) {
            $this->info("Would process {$query->count()} posts");
            return Command::SUCCESS;
        }

        // Laravel 13.18.1+ — typed CLI input reader (parallel to request()->input()):
        $email = $this->input('email');                  // single value
        [$email, $name] = $this->input(['email', 'name']); // multiple values, returns array
        $all = $this->input();                            // all args + options as array

        $count = 0;
        $query->chunk(100, function ($posts) use (&$count) {
            foreach ($posts as $post) {
                $post->update(['status' => 'processed']);
                $count++;
            }
        });

        $this->info("Processed {$count} posts");
        return Command::SUCCESS;
    }
}
```

**Register commands — in `AppServiceProvider` boot() or auto-loaded via `bootstrap/app.php`:**
```php
// Auto-load all Commands in app/Console/Commands
$this->commands([
    // Commands are auto-discovered in Laravel 11+
]);
```

## Scheduling (Cron)

In Laravel 11+, scheduling is defined in `routes/console.php` (no Kernel needed):

```php
// routes/console.php
use Illuminate\Support\Facades\Schedule;

Schedule::command('posts:process')->everyMinute();
Schedule::command('reports:generate')->hourly();
Schedule::command('backup:run')->dailyAt('00:00');
Schedule::command('cleanup:old')->weekly();
Schedule::command('invoices:close')->monthly();

// Run queue worker continuously (auto-restart)
Schedule::command('queue:work --stop-when-empty')
    ->everyMinute()
    ->withoutOverlapping()
    ->runInBackground();

// Closure (cron syntax)
Schedule::call(fn() => User::where('verified', false)->deleteOldUnverified())
    ->weekly();
```

**Enable scheduler — add to server crontab:**
```bash
* * * * * cd /path-to-project && php artisan schedule:run >> /dev/null 2>&1
```

**List scheduled tasks (Laravel 13.8+):**
```bash
# List all scheduled tasks
php artisan schedule:list

# Filter by environment
php artisan schedule:list --environment=production
```

### `Schedule::between()` / `unlessBetween()` Timezone Fix (Laravel 13.17+)

In 13.17, `between()` and `unlessBetween()` apply timezone conversion **per-value**, so the order of `->timezone()` calls in the chain no longer matters (previously, `->timezone()` had to come **before** `->between()` or it was silently ignored and the scheduler fell back to UTC). PR [#60518](https://github.com/laravel/framework/pull/60518).

```php
// Both of these now work identically in Laravel 13.17+:
Schedule::command('report')
    ->timezone('Europe/Rome')
    ->between('10:00', '12:00');        // before: only this order worked

Schedule::command('report')
    ->between('10:00', '12:00')
    ->timezone('Europe/Rome');           // before: silently used UTC
```

If you're upgrading a 12.x or pre-13.17 codebase, audit `routes/console.php` for this pattern — it's a common latent bug where evening reports fire at the wrong wall-clock hour.

## `schedule:work` — Graceful Shutdown on SIGINT / SIGTERM / SIGQUIT (Laravel 13.17+)

Before 13.17, running the scheduler as a long-lived process (`php artisan schedule:work` in a Docker container, a Kubernetes pod, or `supervisord`) was risky: on `SIGTERM` (pod eviction) the process got killed mid-task — any in-flight `schedule:run` could be torn apart, leaving the task half-done. As of PR #60616 (13.17+), `schedule:work` traps `SIGINT` / `SIGTERM` / `SIGQUIT`, stops starting new runs, waits for any in-flight `schedule:run` to finish, then exits cleanly — the same behavior `queue:work` has had since 5.x.

```bash
# In Docker / systemd / supervisord — long-lived scheduler
php artisan schedule:work

# Send a graceful-stop (e.g. on container shutdown):
kill -TERM <pid>   # exits with 0 once in-flight runs complete
```

**How it behaves:**
- Receives signal → flips internal `shouldQuit = true`
- Stops launching new `schedule:run` subprocesses
- Waits for every `schedule:run` already started to exit (sub-processes are `Symfony\Component\Process\Process` instances tracked in `$executions`)
- Returns `SUCCESS` once `$executions` is empty
- **In-flight schedules still complete** — important for billing / email / report jobs that must not be cut off mid-write

**Before 13.17 (workaround):** wrap `php artisan schedule:work` in a script that traps SIGTERM and runs `artisan schedule:list` last (no-op) so the kernel doesn't tear the process apart. With 13.17+ you can rely on native signal handling.

**Container recipe (unchanged, but now safe):**

```dockerfile
# Dockerfile
CMD ["php", "artisan", "schedule:work"]
```

Pair with a Kubernetes `terminationGracePeriodSeconds: 60` (or longer than your longest scheduled task) so the kubelet waits for in-flight runs.

Source: [PR #60616 — schedule:work catch signals](https://github.com/laravel/framework/pull/60616)

## Laravel AI / Boost MCP (Laravel 13+)

```bash
# Install the Boost MCP server for AI assistant integration
php artisan boost:install

# Interactive AI-assisted upgrade to latest Laravel
php artisan boost:upgrade
```

## `php artisan dev` — Dev Process Orchestration (Laravel 13.16+)

Laravel 13.16 ships a first-party `php artisan dev` command that runs your local dev processes (server, queue worker, log tail, Vite, etc.) concurrently — replaces per-project `composer dev` scripts. Configure processes in code via `Illuminate\Foundation\Console\DevCommands`:

```php
// In a service provider (e.g. AppServiceProvider::boot())
use Illuminate\Foundation\Console\DevCommands;

DevCommands::artisan('reverb:start');                            // php artisan reverb:start
DevCommands::artisan('queue:work --tries=1', 'queue')->green(); // named + colored
DevCommands::register('stripe listen --forward-to ' . config('app.url')); // arbitrary shell
DevCommands::npm('run dev', 'vite');                              // auto-detects npm/yarn/pnpm/bun
```

Then run from your project root:
```bash
php artisan dev
```

Default processes (matching `composer dev`): server, queue worker, log tail, Vite. You can override entirely.

**Key behaviors:**
- **Package-aware** — vendor packages don't auto-register dev commands; only your app code does
- **NodePackageManager detection** — generic commands like `npm run dev` resolve to the right runner based on lockfiles (npm, yarn, pnpm, bun)
- **Process names & colors** — second arg names the process (defaults to first segment before space); `->orange()`, `->green()`, etc. for color
- **Upgrade note** — always upgrade to **v13.16.1** (not 13.16.0) — fixes a bug where `DevCommands` could stop itself from registering

**When to use:**
- New Laravel 13.16+ projects — remove `composer dev` and `composer dev:install` scripts
- Teams that want process orchestration in code (versionable, reviewable) instead of `composer.json` scripts
- Replacing `npx concurrently` or similar tooling

Source: [Laravel News — artisan dev](https://laravel-news.com/laravel-13-16-0) | [PR #60412](https://github.com/laravel/framework/pull/60412)

## `php artisan dev:list` — Inspect Registered Dev Processes (Laravel 13.17+)

Companion to `php artisan dev`. Lists every dev process registered for the current project — including those from vendor packages (vendor packages are *not* auto-registered for `dev`, but they may register them via the `DevCommands` API and `dev:list` will surface them):

```bash
php artisan dev:list
```

**Sample output:**
```
  COMMANDS
  --------
  artisan reverb:start
  artisan queue:work --tries=1     [queue]
  artisan pail --timeout=0
  npm run dev                       [vite]
  stripe listen --forward-to https://app.test
```

**When to use:**
- Before running `php artisan dev`, verify what is going to start (catches typos, forgotten `DevCommands::register()` calls, or un-removed legacy `composer dev` scripts)
- During code review — `dev:list` is the canonical list of dev-only processes
- Debugging "why is my dev server doing X?" — `dev:list` reveals the registered commands
- Onboarding — new team members can see all dev processes in one shot

**Pairs with `php artisan dev`:**
- `dev:list` is read-only and safe to run in any environment
- `dev` actually runs the listed processes concurrently
- `dev:list` does NOT filter by environment — it shows everything registered regardless of `APP_ENV`

Source: [PR #60573](https://github.com/laravel/framework/pull/60573) | [Laravel 13.17 Release Notes](https://github.com/laravel/framework/releases/tag/v13.17.0)

## `Macroable` on `InvokedProcess` (Laravel 13.15+)

Laravel 13.15 adds the `Macroable` trait to `Illuminate\Process\InvokedProcess` so you can extend the running-subprocess object with your own helpers (timeouts, abort signals, custom output transformers, etc.) without subclassing:

```php
use Illuminate\Support\Facades\Process;
use Illuminate\Process\InvokedProcess;

// Register a macro once at boot time (e.g. AppServiceProvider)
InvokedProcess::macro('recordDuration', function (): float {
    /** @var InvokedProcess $this */
    return (microtime(true) - $this->startTime) * 1000;
});

// Use the macro on the result of ->run()
$process = Process::timeout(10)->run("ffmpeg -i input.mp4 output.mp4");
$durationMs = $process->recordDuration();
```

**Useful macros to register:**
- `recordDuration()` — wall-clock runtime of the subprocess
- `signal($sig)` — wrapper around `signal()` that pre-validates the signal name
- `lines($max)` — bounded line-buffer wrapper around the `lazy()` output iterator
- `abortIf($condition)` — call `signal(SIGTERM)` then `wait()` based on a condition

**Why this matters:** `InvokedProcess` was previously a sealed-shape object — the only way to add helpers was to wrap it. `Macroable` removes the need for a wrapper and keeps call sites clean.

- **`input()` on Console Commands (Laravel 13.18.1, PR #60607 by @stevebauman)** — `$this->input('name')` returns a single value (string|array|null), `$this->input(['a', 'b'])` returns an associative array of named values, `$this->input()` (no args) returns everything. Cleaner than `$this->argument()` + `$this->option()` + manual casting for every command. Works on `Illuminate\Console\Command` subclasses only.

Source: [PR #60392](https://github.com/laravel/framework/pull/60392) | [Laravel 13.15 Release Notes](https://github.com/laravel/framework/releases/tag/v13.15.0)

## Tinker


```bash
php artisan tinker
```

```php
// Test code in real Laravel context
>>> Post::find(1)->title
=> "My First Post"

>>> User::factory()->make(['name' => 'Test'])
=> User {#0000 name: "Test"}

>>> dispatch(new \App\Jobs\ProcessPostJob(5))

>>> Cache::put('key', 'value', 60)
```

**For programmatic use in scripts:**
```php
// In any PHP script (not tinker)
require 'vendor/autoload.php';
$app = require 'bootstrap/app.php';
$kernel = $app->make(Illuminate\Contracts\Console\Kernel::class);
$kernel->call('tinker');
// Note: In Laravel 13+, use: php artisan tinker directly
```

## Environment & Config

```bash
# Generate app key
php artisan key:generate

# List all routes
php artisan route:list

# List routes matching pattern
php artisan route:list --path=api

# Get routes as JSON (for tooling)
php artisan route:list --json

# Show environment
php artisan about

# List all config values
php artisan config:show app
```

## Common Mistakes

1. **`route:cache` with closures** — closures don't serialize, cache fails silently
2. **`config:cache` before `env()` cleanup** — env returns null if cached before env vars read
3. **`php artisan config:cache` in dev** — must clear cache after every config change
4. **No `--withoutOverlapping()` on queue:work** — duplicate workers fight over jobs
5. **Scheduling without cron entry** — `schedule:run` never fires without server cron
6. **`env()` outside config files** — after `config:cache`, env becomes null everywhere
7. **`schedule:work` killed mid-task on container shutdown** — on Laravel <13.17, `SIGTERM` tears the process apart during a `schedule:run`. Upgrade to 13.17+ (PR #60616) for graceful signal handling, or set `terminationGracePeriodSeconds: 60+` in Kubernetes so the kernel doesn't cut in-flight tasks.

## Updated from Research (2026-06-29)

- **`schedule:work` graceful signal handling (PR #60616, 13.17+)** — `SIGINT` / `SIGTERM` / `SIGQUIT` now stops new runs cleanly and waits for in-flight `schedule:run` subprocesses to finish before exiting. Long-lived scheduler processes in containers can finally rely on native signal handling instead of wrapper scripts.



---

## Console Output Components (the `components` helper)

Beyond `$this->line()` / `$this->info()` / `$this->error()` / `$this->warn()` / `$this->comment()` / `$this->newLine()` (which the section above covers), Laravel ships a much richer **`$this->components`** factory — accessible via `Illuminate\Console\View\Components\Factory` and instantiated for every command. AI assistants default to manual `echo "things:\n"` loops. The factory emits ANSI-styled, width-aware, consistently-formatted output that survives terminal-width changes (CI logs vs wide terminals). Use it for everything except single-line messages.

```php
// Bullet list — the most-used component
$this->components->bulletList([
    'First migrated row',
    'Second migrated row',
    'Third migrated row',
]);

// Two-column layout (label + value rows)
$this->components->twoColumnDetail('Total Posts', 142_857);
$this->components->twoColumnDetail('Active Users', 8_421);

// Info / error / warn — same colors as $this->info() etc, but routed through the factory
$this->components->info('All migrations complete.');     // green
$this->components->warn('Skipped 3 rows.');              // yellow
$this->components->error('Migration failed.');           // red
```

**Other components worth knowing:**

| Component | Use case |
|---|---|
| `bulletList(array $items)` | Render an array of strings as a bulleted list |
| `twoColumnDetail(string $first, string $second)` | Render key/value rows; aligns the column to terminal width |
| `info(string $string)` / `warn()` / `error()` | Styled ANSI-color messages, parallel to `$this->info()` |
| `confirm(string $question, bool $default = false)` | `yes / no` prompt returning bool |
| `ask(string $question, ?string $default = null)` | Free-text input, returns string |
| `choice(string $question, array $choices, mixed $default, ?int $attempts = null, bool $multiple = false)` | Single- or multi-select against a list of choices |
| `secret(string $question)` | Masked input (replacement for `password()` in non-Prompts codebases) |

**Why the factory exists:** plain `echo` / `$this->info()` doesn't auto-indent, doesn't respect terminal width, and doesn't guarantee consistent styling across all your commands. The `components` factory is what Laravel itself uses for `migrate`, `queue:listen`, `schedule:list`, etc. — every new command you ship should use it for the same reasons.

**Programmatic use in helper classes (not commands):**

```php
use Symfony\Component\Console\Output\ConsoleOutput;

$output = new ConsoleOutput();
\Illuminate\Console\View\Components\Factory::render($output)
    ->bulletList(['a', 'b', 'c']);
```

Outside a command, instantiate `Symfony\Component\Console\Output\ConsoleOutput` and pass it to `Factory::render()`. Inside a command, `$this->components` is already wired.

Source: [Laravel 13.x Artisan Docs — Output](https://laravel.com/docs/13.x/artisan#writing-output) | [API: Factory](https://api.laravel.com/docs/13.x/Illuminate/Console/View/Components/Factory.html) | [API: BulletList](https://api.laravel.com/docs/13.x/Illuminate/Console/View/Components/BulletList.html)

---

## `schedule:clear-cache` — Clear Stuck `withoutOverlapping` Mutexes

When you schedule a task with `->withoutOverlapping()`, Laravel stores a cache lock under the hood (key like `framework/schedule-mutex-<signature>`). If a task crashes mid-flight, the worker dies, or the cache driver hiccups, that lock can survive — and the next run is silently **skipped** because Laravel still thinks the previous one is running. The fix is `schedule:clear-cache`, NOT the nonexistent `schedule:clear` (which AI assistants routinely hallucinate).

```bash
# Clear ALL schedule mutex locks (every scheduled task)
php artisan schedule:clear-cache

# Clear a specific task's lock
# (not directly supported as a CLI flag; instead delete the cache key manually)
php artisan tinker
>>> Cache::forget('framework/schedule-mutex-ProcessPostsCommand')

# Re-enable / disable the scheduler without deleting the cron entry
# (Laravel 12+) — `schedule:list` shows the current state
php artisan schedule:list
php artisan schedule:list --environment=production
```

**When to run `schedule:clear-cache`:**

- A scheduled task with `withoutOverlapping()` is showing `running` in `schedule:list` but no actual worker is running it
- You deleted a queued job / migration mid-run and the lock was never released
- You killed a long-running task and the lock TTL is 24 hours
- The cache backend was wiped (Redis flush, cache driver swap from `file` to `redis`)
- Post-deploy: storm the mutexes if your deploy procedure involves killing `schedule:work` and you have a long `withoutOverlapping()` task that doesn't release its lock cleanly

**Three crucial gotchas:**

1. **`schedule:clear` doesn't exist.** AI assistants default to `schedule:clear` — it produces "Command schedule:clear is not defined." The correct command is `schedule:clear-cache`. Don't paper over this by writing a custom command; flag it back to the AI and use the real one.
2. **No per-task flag.** `schedule:clear-cache` clears **everything**. To clear a single stuck lock without nuking others, target the cache key directly with `Cache::forget(...)` or your cache backend's CLI (Redis: `DEL`; database: `DELETE FROM cache WHERE key LIKE 'framework/schedule-mutex-%'`).
3. **`withoutOverlapping()` lock TTL = 24 hours by default.** Pass `->withoutOverlapping(3600)` (in seconds) for a custom TTL — a task that's been stuck less than an hour will not be auto-released. Idempotent deployments during that window will appear to "skip" the task entirely.

**Same `Cache::forget()` pattern works for `mutexName()` custom locks:**

```php
Schedule::command('reports:generate')
    ->withoutOverlapping(3600)
    ->mutexName('generate-reports');  // custom lock key: 'framework/schedule-mutex-generate-reports'

// Manual unlock (e.g. admin endpoint):
Cache::forget('framework/schedule-mutex-generate-reports');
```

**Why this is still a 23-year-old footgun in 2026:** the lock mechanism was added in Laravel 5.x (PR [#40135](https://github.com/laravel/framework/pull/40135) shipped the `schedule:clear-cache` command to clear them) and the problem still ships — every team that runs `schedule:work` long-lived under supervisord / Octane / k8s hits this at least once. Document your recovery command in runbooks; do not let it become tribal knowledge.

Source: [Laravel 13.x Scheduling Docs — Preventing Task Overlaps](https://laravel.com/docs/13.x/scheduling#preventing-task-overlaps) | [PR #40135 — schedule:clear-cache](https://github.com/laravel/framework/pull/40135)

---

## `Artisan::call()` Exit Codes, Boot, and the Octane Trap

`Artisan::call()` runs a command in the current PHP process — no `proc_open`, no shell-out, no separate PHP boot. The exit code is returned as an `int`:

```php
use Illuminate\Support\Facades\Artisan;

$exitCode = Artisan::call('mail:send', [
    'user' => $user,
    '--queue' => 'default',
]);

if ($exitCode !== 0) {
    // handle failure — but see "Exception trap" below
}
```

Exit codes follow the standard convention: `0` = success, `1` = general failure, `2` = misuse (often used for invalid input). Commands can also **throw** instead of returning a non-zero — `Artisan::call()` will let the exception bubble up.

**Passing parameters — three forms:**

```php
Artisan::call('mail:send', ['user' => 1, '--queue' => 'default']); // safest — explicit array
Artisan::call('mail:send 1 --queue=default');                       // string form — parser-dependent
Artisan::call(MailSendCommand::class, ['user' => 1]);               // class-name form
```

Boolean flags: `Artisan::call('migrate:refresh', ['--force' => true])`. Array arguments: `Artisan::call('mail:send', ['--id' => [5, 13]])`.

**Three traps the official docs do not flag clearly:**

### Trap 1 — Octane state leak across requests

Under Octane (Swoole / RoadRunner / FrankenPHP), the app **stays booted** between requests. If you call `Artisan::call()` and the inner command constructor-injects a service that holds request-scoped state (tenant resolver, request ID, auth user), that state **leaks into the next request** unless explicitly reset. Symptom: HTTP request N+1 sees the tenant of request N's `Artisan::call()` invocation.

```php
// BAD — leaks under Octane because the tenant is set in the command constructor
class SendInvoiceEmailCommand extends Command
{
    public function __construct(private TenantResolver $tenants) { parent::__construct(); }
    public function handle(): int
    {
        $this->tenants->set(auth()->user()->tenant_id);  // gets stuck on the worker
    }
}

// GOOD — read per-invocation state inside handle()
public function handle(): int
{
    $tenantId = auth()->user()->tenant_id;  // fresh per call
    $this->tenants->set($tenantId);
}
```

Reset if you must keep the constructor: bind the command as `scoped` in a service provider, or wrap with `App::forgetInstance(...)` between calls. See `observers.md` (Octane & Static Observer Registration) for the same pattern with observers — same root cause, different surface.

### Trap 2 — `Artisan::call()` does NOT trigger `Kernel` reboots

Each `Artisan::call()` reuses the current Kernel. If your command (or any code it transitively calls) wrote to `config()`, set a `Mail::fake()`, or mocked a facade, **the state survives the `call()`**. Production code that calls `Artisan::call()` inside an HTTP route handler will leak those mutations into the next HTTP request handled by the same worker.

Workaround: wrap in `app()->forgetInstance(...)` for the bindings you mutated, or use `Artisan::call()` from a long-running queue worker (not an HTTP request) where one mutation cycle ends when the job ends.

### Trap 3 — `Artisan::call()` swallows output unless you redirect it

By default, output goes to the same place the calling command writes to. If you call `Artisan::call('migrate', [], $outputBuffer)` from a queue job or HTTP request, the migration's progress messages go to `$outputBuffer` (a `BufferedOutput`), not the caller. For nested command output to appear in the parent, omit the third argument — but then it leaks to the parent terminal:

```php
use Symfony\Component\Console\Output\BufferedOutput;

$buffer = new BufferedOutput();
$exit = Artisan::call('migrate', [], $buffer);   // captured — must echo or log to see
Log::info($buffer->fetch());                       // explicit — see all output as a string
```

Common bug: AI calls `Artisan::call()` and then complains "my migration looks like it didn't run" because the output was captured into a buffer that was discarded.

**When to prefer `Artisan::queue()` over `Artisan::call()`:**

```php
Artisan::queue(MailSendCommand::class, ['user' => 1, '--queue' => 'default']);
```

`Artisan::queue()` serializes the command and **dispatches it on the queue** — the worker that picks it up reboots the kernel cleanly. Use this for HTTP-triggered admin actions (rebuild cache, reindex search) that don't need to finish synchronously. It's the Octane-safe version of `Artisan::call()`.

Source: [Laravel 13.x Artisan Docs — Programmatically Executing Commands](https://laravel.com/docs/13.x/artisan#programmatically-executing-commands) | [Laravel 13.x Artisan Docs — Exit Codes](https://laravel.com/docs/13.x/artisan#exit-codes)

---

## `php artisan down` / `php artisan up` — Maintenance Mode Command Reference

Maintenance mode lives in `deployment.md` (full workflow: pre-rendered views, secret bypass, CI integration, Mautic / WordPress patterns). This section covers the **command flags** that AI assistants ask about most often. All five flags compose freely:

| Flag | Purpose | Example |
|---|---|---|
| `--retry=N` | Send `Retry-After: N` header + write `<meta http-equiv="refresh" content="N">` to the default 503 page | `php artisan down --retry=60` |
| `--refresh=N` | **Alias of `--retry`.** Both write the meta-refresh directive. | `php artisan down --refresh=15` |
| `--render=view` | Render a custom Blade view before the app boots (no DB / facades available inside) | `php artisan down --render="errors::503"` |
| `--redirect=/` | Redirect all maintenance requests to a path | `php artisan down --redirect=/` |
| `--secret=token` | Bypass token — visitors at `https://app.test/secret-token` get the live app | `php artisan down --secret="bypass-1630542a-4b66"` |

**Combine for the standard "deploy with refresh timer" pattern:**

```bash
php artisan down --retry=15 --secret="bypass-$(uuidgen)" --render="errors::503"
# Deploy...
php artisan up
```

**What `--render` can and cannot access:**

- ✅ Plain HTML, Blade syntax (`@if`, `@foreach`), `{{ config('app.name') }}` constants
- ✅ Static assets via `<link>` / `<script>` (must be CDN'd or pre-rendered; the public disk is still served via nginx but the Laravel-side view cache isn't)
- ❌ Database queries, facades (`Cache`, `Redis`, `Log`), Eloquent models, queue dispatchers
- ❌ Service-provider-booted services (DB connection isn't open yet)

Because `--render` is served **before Laravel boots**, the view must be self-contained. Embed all CSS inline; do not rely on `mix()` / `Vite::asset()` for stylesheets. For richer pages, deploy a static HTML file to public/ and `--render="errors::503"`-style reference it.

**Laravel 13.18.1+ flag:** when the `--secret` bypass is active, requests to `/api/*` or routes that set `Accept: application/json` now return JSON 503 instead of the HTML page. See `controllers.md` + `deployment.md` (Maintenance section) for the full flow. PR [#60595](https://github.com/laravel/framework/pull/60595).

**Quick reference — when to use which flag:**

- Deployment with 15-second auto-retry: `--retry=15`
- Soft launch / pre-announced outage with custom branding: `--render="errors::503"` (build the Blade at `resources/views/errors/503.blade.php`)
- Full redirect-to-status-page (Cloudflare / Statuspage): `--redirect=https://status.example.com`
- Deploy with staff bypass: `--secret="$(uuidgen)"`
- Blue/green DB migration: `--redirect=/maintenance-info` while a separate route serves a Blade view

**Gotcha — testing:** the default `file` maintenance driver does NOT isolate across parallel test processes. Use the `array` driver (Laravel 13.16+) for `phpunit --parallel`: set `MAINTENANCE_DRIVER=array` in `.env.testing`. Detail in `testing.md`.

Source: [Laravel 13.x Configuration — Maintenance Mode](https://laravel.com/docs/13.x/configuration#maintenance-mode) | [Laravel 13.18.1 PR #60595](https://github.com/laravel/framework/pull/60595)

---

## Updated from Research (2026-07-13)

- **`$this->components->bulletList()` + the `components` output factory** — long-standing (since Laravel 6) but routinely missed by AI-generated commands. Every command output other than a single line should go through the `components` factory (`bulletList`, `twoColumnDetail`, `info`, `warn`, `error`, `confirm`, `ask`, `choice`, `secret`) for terminal-width-aware, ANSI-styled, consistent formatting. Same API Laravel itself uses for `migrate`, `schedule:list`, `queue:listen`, etc.
- **`schedule:clear-cache` (NOT `schedule:clear`)** — clears stuck `withoutOverlapping()` mutex locks after a worker crash, lock TTL expiry, or cache-backend flush. The AI-default `schedule:clear` doesn't exist; the correct command has been `-cache` appended since PR #40135. Default mutex TTL = 24 hours; pass `->withoutOverlapping(N)` to override. No per-task flag — clear all or target the cache key directly with `Cache::forget('framework/schedule-mutex-<signature>')`.
- **`Artisan::call()` under Octane** — three real footguns the official docs gloss over: (1) constructor-injected request-scoped state leaks across requests under Swoole/RoadRunner/FrankenPHP workers, (2) the called command reuses the current Kernel so `Mail::fake()` / facade mocks / `config()` mutations persist into the next request, (3) output is silently captured into a `BufferedOutput` unless you pass `null` or omit the third argument. For admin actions that don't need synchronous completion, prefer `Artisan::queue()` which dispatches to a worker with a fresh kernel.
- **`php artisan down` flag reference** — `--retry=N` / `--refresh=N` (alias), `--render=view` (pre-boot, no facades), `--redirect=/path`, `--secret=token` (bypass). Compose freely. `--render` views run before the app boots — no DB, no facades, embed all CSS inline. Laravel 13.18.1+ returns JSON 503 for API/JSON routes when `--secret` bypass is active (PR #60595). See `deployment.md` Maintenance Mode section for the full workflow.
