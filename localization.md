# Localization — Translations, Locales, Pluralization

> Laravel's localization system handles multi-language content, pluralization, number formatting, and locale-aware date presentation.

## ⚠️ Security Warning: Laravel-Lang Packages (May 2026 Supply-Chain Attack)

The `laravel-lang/*` Composer packages (`laravel-lang/lang`, `laravel-lang/attributes`, `laravel-lang/http-statuses`, `laravel-lang/actions`) were compromised on **May 22, 2026** when an attacker rewrote every git tag across all four repos to point at malicious commits.

**Do NOT install these packages today** unless you can pin to a verified pre-2026-05-22 SHA. See [security.md](security.md#critical-laravel-lang-composer-supply-chain-attack-may-22-2026--no-cve) for full IoCs and remediation.

For new projects, use Laravel's built-in `resources/lang/` translation files — no third-party package needed for the vast majority of apps.


## Setup

```php
// config/app.php
'locale' => 'en',           // current locale
'fallback_locale' => 'en', // when translation missing
'locales' => ['en', 'es', 'fr', 'de', 'hi', 'gu'],
```

**Register locales in `bootstrap/app.php` (Laravel 12+):**
```php
->withRouting(
    locale: 'es', // force default locale in URL
)
```

## Translation Files

```php
// resources/lang/en/messages.php
return [
    'welcome' => 'Welcome :name',
    'new_posts' => '{0} No posts|{1} One post|[2,*] :count posts',
    'published_at' => 'Published on :date',
    'greeting' => 'Hello :first_name, you have :count messages',
];
```

**Organize by domain:**
```
resources/lang/
├── en/
│   ├── messages.php
│   ├── validation.php
│   └── passwords.php
├── es/
│   ├── messages.php
│   ├── validation.php
│   └── passwords.php
```

## Using Translations

```php
// Blade
@lang('messages.welcome', ['name' => $user->name])
{{ __('messages.welcome', ['name' => $user->name]) }}

// Code
trans('messages.welcome', ['name' => $user->name])
__("messages.new_posts", ['count' => $postCount])

// In validation messages
'custom' => [
    'email' => [
        'required' => 'We need to know your email address',
    ],
],
```

## Typed Translation Accessors (Laravel 13.15+)

`__()` and `trans()` historically return broad types (`array|string|null` for `__()`, `Translator|array|string` for `trans()`). That works fine in Blade but adds friction in strictly-typed code and static analysis. Laravel 13.15 adds two typed accessors that return concrete types:

```php
// Returns string|null — useful for single-value translations
public function label(): ?string
{
    return trans()->string('labels.' . $this->name);
}

// Returns array<string, mixed> — useful for grouped translations
public function options(): array
{
    return trans()->array($this->options_key);
}
```

**Mirror of existing typed helpers** — same shape as `config()->string()`, `request()->integer()`, `request()->enum()`, etc.

**When to use:**
- Eloquent model accessors that return translated text — improves IDE inference and PHPStan/Psalm accuracy
- Service methods that return translated strings as a known type
- Anywhere strict typing matters (typed properties, return type declarations)

**Edge case:** If the key is missing, `string()` returns `null` (same as `__()`). Use the null-safe operator `?->` or provide a default:
```php
return trans()->string('status.' . $this->code) ?? 'unknown';
```

Source: [Laravel News — Typed Translation Accessors](https://laravel-news.com/laravel-13-15-0) | [PR #60443](https://github.com/laravel/framework/pull/60443)

## Pluralization


```php
// {0} Zero|{1} One|[2,*] Many
echo trans_choice('messages.new_posts', 0); // "No posts"
echo trans_choice('messages.new_posts', 1); // "One post"
echo trans_choice('messages.new_posts', 5); // "5 posts"

// Named parameters
echo trans_choice('messages.greeting', $messageCount, [
    'first_name' => $user->first_name,
    'count' => $messageCount,
]);
```

**Pluralization intervals (Laravel 13 supports):**
```php
// resources/lang/en/messages.php
return [
    'items_sold' => '{0} No items sold|{1} 1 item sold|[2,10] :count items sold|[11,*] Over 10 items sold',
];
```

## Number/Date Formatting

```php
use Illuminate\Support\Number;

// Number formatting
Number::format(1234.56, 2); // "1,234.56"
Number::spell(42);          // "forty-two"
Number::currency(1234.56);  // "$1,234.56"
Number::percent(0.75);      // "75%"

// Locale-aware formatting
Number::format(1234.56, locale: 'de_DE'); // "1.234,56"

// Date formatting by locale
Carbon::parse($date)->locale('hi_IN')->format('d MMMM yyyy');

// In Blade
{{ Number::currency(1234.56, in: 'EUR', locale: 'de_DE') }}
```

## Locale Detection & Switching

```php
// Detect from browser
app()->setLocale(request()->getPreferredLanguage());

// From route parameter
Route::get('{locale}/posts', function (string $locale) {
    app()->setLocale($locale);
    return view('posts.index');
})->name('posts.localized');

// Force locale in URL
URL::setLocale('es');
Route::get('{locale}/posts', ...)->name('posts.localized');

// Persist in session
session(['locale' => 'es']);

// Middleware for locale switching
class LocaleMiddleware
{
    public function handle($request, $next)
    {
        if ($locale = session('locale')) {
            app()->setLocale($locale);
        } elseif ($request->has('lang')) {
            app()->setLocale($request->get('lang'));
            session(['locale' => $request->get('lang')]);
        }
        return $next($request);
    }
}
```

## Scoped Translations (Namespaces)

```php
// resources/lang/en/pagination.php
return [
    'next' => 'Next',
    'previous' => 'Previous',
];

// Use with dot notation
__('pagination.next');           // in code
{{ __('pagination.next') }}     // in Blade
```

## Inline Translations (Short Keys)

For frequently used strings, use short keys to avoid long translation chains:

```php
// resources/lang/en.json
{
    "Welcome back, :name": "Welcome back, :name",
    "Save": "Save",
    "Cancel": "Cancel",
    "Delete": "Delete"
}

// Use
__('Welcome back, :name', ['name' => $user->name])
__("Save")
```

## Validation Translations

```php
// resources/lang/en/validation.php
return [
    'required' => 'The :attribute field is required.',
    'email' => 'The :attribute must be a valid email address.',
    'min' => [
        'string' => 'The :attribute must be at least :min characters.',
    ],
    'custom' => [
        'age' => [
            'min' => 'You must be at least :min years old.',
        ],
    ],
    'attributes' => [
        'email' => 'email address',
        'password' => 'password',
    ],
];
```

## Translation Loading (Databases, APIs)

For dynamic/translated content (products, pages), use a database-backed approach:

```php
// Migration
Schema::create('translations', function (Blueprint $table) {
    $table->id();
    $table->string('key'); // e.g., "homepage.hero.title"
    $table->string('locale', 5);
    $table->text('value');
    $table->unique(['key', 'locale']);
});

// Model
class Translation extends Model
{
    public static function get(string $key, string $locale = null): ?string
    {
        return static::where('key', $key)
            ->where('locale', $locale ?? app()->getLocale())
            ->value('value');
    }
}

// Service for AI-generated or user-edited translations
class TranslationService
{
    public static function missing(string $locale): Collection
    {
        $en = static::where('locale', 'en')->pluck('key');
        $translated = static::where('locale', $locale)->pluck('key');
        return $en->diff($translated);
    }
}
```

## RTL (Right-to-Left) Languages

```php
// Middleware: Set direction based on locale
class SetDirectionMiddleware
{
    public function handle($request, $next)
    {
        $rtlLocales = ['ar', 'he', 'fa', 'ur'];
        $direction = in_array(app()->getLocale(), $rtlLocales) ? 'rtl' : 'ltr';
        view()->share('direction', $direction);
        return $next($request);
    }
}

// In Blade layouts
<html dir="{{ $direction ?? 'ltr' }}" lang="{{ app()->getLocale() }}">
```

## URL Routing Patterns for Locale Prefix

```php
// bootstrap/app.php (Laravel 11/12/13)
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        // Force all routes through the {locale} prefix
        then: function () {
            Route::middleware('web')
                ->prefix('{locale}')
                ->where(['locale' => '[a-z]{2}'])
                ->group(base_path('routes/web.php'));
        },
    )
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->web(append: [
            'App\\Http\\Middleware\\SetLocale',
        ]);
    });

// In SetLocale middleware
public function handle($request, Closure $next)
{
    $locale = $request->route('locale');
    if ($locale && in_array($locale, config('app.locales'))) {
        app()->setLocale($locale);
    } else {
        abort(404); // or redirect to default locale
    }
    return $next($request);
}
```

**Helper for generating localized URLs from inside templates:**

```php
// app/helpers.php
function localized_url(string $locale, ?string $path = null): string
{
    $path ??= request()->path();
    // strip any existing locale prefix
    $stripped = preg_replace('/^([a-z]{2})\\//', '', $path);
    return url("/{$locale}/{$stripped}");
}
```

```blade
{{-- In layout: hreflang tags for SEO --}}
@foreach (config('app.locales') as $loc)
    <link rel="alternate" hreflang="{{ $loc }}"
          href="{{ localized_url($loc) }}">
@endforeach
<link rel="alternate" hreflang="x-default" href="{{ localized_url(config('app.fallback_locale')) }}">
```

## Lazy Translation Loading

Loading every translation file for every locale on every request wastes memory and slows boot. Two patterns:

### Pattern 1: JSON-only translations (single file per locale)

```php
// resources/lang/en.json — all English strings in one file
{
  "Welcome back": "Welcome back",
  "Save": "Save",
  "Cancel": "Cancel"
}

// resources/lang/es.json
{
  "Welcome back": "Bienvenido de nuevo",
  "Save": "Guardar",
  "Cancel": "Cancelar"
}

// Loaded automatically — no per-domain file structure needed
__('Welcome back')
__('Save')
```

**Best for:** UI strings, button labels, error messages, anything short and global.

### Pattern 2: Lazy namespaces (load only when accessed)

```php
// config/app.php — register translator loaders with explicit namespaces
'translator' => [
    'loaders' => [
        'app' => fn() => new 'Illuminate\\Translation\\FileLoader'(app()['files'], resource_path('lang')),
    ],
],

// Use lazy namespace — translation file loaded only when first accessed
__('shop::cart.empty');   // loads resources/lang/en/shop/cart.php
__('auth::failed');       // loads resources/lang/en/auth.php
```

### Pattern 3: Database-backed translations (already shown above)

For dynamic content (CMS pages, product descriptions), use the DB-backed approach. Cache the translated value in Redis to avoid hitting the DB on every request:

```php
public static function get(string $key, string $locale = null): ?string
{
    $locale ??= app()->getLocale();
    return Cache::remember("trans:{$locale}:{$key}", 3600, fn() =>
        static::where('key', $key)->where('locale', $locale)->value('value')
    );
}
```

## Detecting Missing Translations in Staging

Run this in your CI or staging environment to catch untranslated keys before they hit production:

```php
// app/Console/Commands/CheckMissingTranslations.php
namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\File;

class CheckMissingTranslations extends Command
{
    protected $signature = 'i18n:check';
    protected $description = 'Find translation keys missing in non-default locales';

    public function handle(): int
    {
        $default = config('app.fallback_locale');
        $locales = config('app.locales');

        $defaultKeys = collect(File::allFiles(resource_path("lang/{$default}")))
            ->flatMap(fn($f) => $this->extractKeys($f));

        foreach ($locales as $locale) {
            if ($locale === $default) continue;
            $localeKeys = collect(File::allFiles(resource_path("lang/{$locale}")))
                ->flatMap(fn($f) => $this->extractKeys($f));

            $missing = $defaultKeys->diff($localeKeys);
            if ($missing->isNotEmpty()) {
                $this->warn("Missing in {$locale}:");
                $this->line($missing->implode("\n"));
                return self::FAILURE;
            }
        }
        $this->info('All translations complete');
        return self::SUCCESS;
    }

    private function extractKeys($file): array
    {
        $arr = include $file->getPathname();
        return is_array($arr) ? array_keys('' . collect($arr)->flatten()->keys()) : [];
    }
}
```

Add to your CI pipeline:
```yaml
- name: Check missing translations
  run: php artisan i18n:check
```

## Common Mistakes

1. **`trans()` vs `__()`** — both work, `__()` is the newer helper
2. **Missing fallbacks** — always set `fallback_locale`
3. **Hardcoded strings** — extract every user-facing string to lang files
4. **Plural without trans_choice** — use `{0}|{1}|[2,*]` syntax for count-aware messages
5. **Not using Number::format** — locale-aware number formatting, not `number_format()`
6. **Translation keys not namespaced** — use dot notation (`messages.greeting`) to avoid collisions
7. **Missing locale in URL routing** — SEO requires `hreflang` tags and locale-prefixed URLs
8. **No missing key fallback UI** — log or flag untranslated keys in staging
9. **Loading all translation files per request** — for large apps with 10+ domains, use lazy namespaces (`shop::cart.empty`) or split into JSON files to avoid loading thousands of keys on every request
10. **Installing laravel-lang/* packages post-2026-05-22** — supply-chain attack rewrote every git tag with backdoored commits. Pin to a verified pre-2026-05-22 SHA or skip the packages entirely
11. **Missing `trans()` typed return type in models** — model accessors returning translations should use `trans()->string()` / `trans()->array()` (Laravel 13.15+) for strict typing; otherwise return type is `array|string|null` and IDEs/PHPStan lose inference

## SEO — hreflang Tags

```php
// In your layout Blade file
@foreach (['en', 'es', 'fr'] as $locale)
    <link rel="alternate" hreflang="{{ $locale }}"
          href="{{ localized_url($locale, url()->current()) }}">
@endforeach
<link rel="alternate" hreflang="x-default" href="{{ localized_url('en', url()->current()) }}">
```

## Updated from Research (2026-06-29)

### Cycle 9 additions (2026-06-29)

- **URL routing for locale prefix** — `withRouting(then: ...)` + `prefix('{locale}')` + `SetLocale` middleware. Helper `localized_url($locale, $path)` for hreflang generation in templates.
- **Lazy translation loading** — three patterns: JSON-only translations (single file per locale), lazy namespaces (`shop::cart.empty` loads only on access), and DB-backed translations with Redis cache.
- **Detecting missing translations in staging** — `php artisan i18n:check` command diffs the fallback locale's keys against every other locale and exits non-zero on gaps. Wire into CI to fail builds on incomplete translations.

- Laravel 13 improves Number formatting with locale-aware currency, percentage, and spell-out formatting
- Browser-based locale detection via `request()->getPreferredLanguage()`
- Interval-based pluralization in `trans_choice()` (e.g., `[2,10]` range)
- Database-backed translations for dynamic content (products, CMS pages)
- RTL locale support requires direction-aware layouts

Sources: [Laravel Localization](https://laravel.com/docs/13.x/localization) | [Laravel Number Helper](https://laravel.com/docs/13.x/helpers#method-number)
