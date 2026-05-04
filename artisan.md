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