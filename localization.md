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
12. **English interval plural for non-English locales** — `[2,*] :count products` is ungrammatical in Russian (3 forms), Polish (3), Arabic (6), Czech (3), Slovenian (4). Use full CLDR forms OR `gboquizosanchez/icu-i18n` if you serve these markets.
13. **Trusting undocumented CLDR support in `trans_choice()`** — works today via Symfony's `PluralizationRules`, but Laravel's docs only document the interval form. No SLA, can break on Symfony upgrades. Snapshot your translation strings in CI with `php artisan i18n:check` (added in cycle 9) — if any Russian/Arabic form regresses to the interval fallback, you'll see it before users do.
14. **Trusting `Lang::has($key, $locale)` for locale-specific existence** — 10-year-old footgun: returns `true` whenever the fallback chain has the key, regardless of whether `$locale` itself has it. Use `Lang::hasForLocale($key, $locale)` for strict locale-only checks (RTL flip indicators, audit reports, locale-specific UI). `Lang::has()` without a locale is correct for "any translation including fallback".
15. **Concatenating translated phrases in Blade (`{{ __('Attach') }} {{ $name }}`)** — assumes English word order; German puts the verb at the end (`Beitrag anhängen`), Japanese uses honorific suffixes, Arabic/Hindi reorder differently. Extract to a single key with placeholder: `'attach_resource' => ':resource anhängen'`.
16. **`$date->format('D M Y')` expecting a localized string** — `format()` is PHP native and ALWAYS English. Use `isoFormat('DD MMM YYYY')` (Carbon's own CLDR translations, Docker/CI safe) for multi-locale production. `translatedFormat()` works only if the container has the OS locale installed (fragile).
17. **`spatie/laravel-translatable` translated column set to `string`** — silently truncates the JSON at the first comma and corrupts data. The column MUST be `json` (or `text` on DBs without JSON). Pair with MySQL JSON functional index or PostgreSQL GIN if you query by translated content.
18. **No missing-key handler in production** — Laravel returns the raw key string (`messages.checkout.totals.subtotal`) to users when a translation is missing. Add `Lang::handleMissingKeysUsing()` in `AppServiceProvider::boot()` (Laravel 10.33+) to log + return a safe placeholder. Without it, refactor-renamed keys ship silently.
19. **Forgetting `Carbon::setLocale()` after `App::setLocale()`** — the two are independent. `App::setLocale()` updates Laravel; `Carbon::setLocale()` updates Carbon. Setting only one means `$post->created_at->isoFormat()` still renders English weekdays/months.
20. **Calling `Translator::addPath()` per-request** — these calls happen at boot in a service provider. Calling them in middleware per-request rebuilds the lookup chain and slows every translation. Cache the path list at boot.

## SEO — hreflang Tags

```php
// In your layout Blade file
@foreach (['en', 'es', 'fr'] as $locale)
    <link rel="alternate" hreflang="{{ $locale }}"
          href="{{ localized_url($locale, url()->current()) }}">
@endforeach
<link rel="alternate" hreflang="x-default" href="{{ localized_url('en', url()->current()) }}">
```



## CLDR Plural Rules for Non-English Languages (Undocumented but Supported)

Laravel's official docs only show the `{0}|{1}|[2,*]` interval syntax for `trans_choice()`. But the underlying implementation reads the rules from `Symfony\Component\Translation\PluralizationRules`, which supports the full [CLDR plural rule set](https://cldr.unicode.org/index/cldr-spec/plural-rules). **Arabic (`ar`), Russian (`ru`), Polish (`pl`), Slovenian (`sl`), and similar languages work out of the box** — you just have to provide the right number of pipe-separated forms.

The exact number of forms varies by language. Laravel picks the right one based on `Symfony\Component\Translation\getPluralRules($locale)[$count]`:

### Arabic (6 plural forms — zero / one / two / few / many / other)

```php
// resources/lang/ar/messages.php
return [
    // count=0 zero, count=1 one, count=2 two, count=3..10 few, count=11..99 many, count=100+ other
    'cart_items' => '{0} لا توجد منتجات|{1} منتج واحد|{2} منتجان|[3,10] :count منتجات|[11,99] :count منتجاً|[100,*] :count منتج',
];

// In code
echo trans_choice('messages.cart_items', 0);   // "لا توجد منتجات" (zero)
echo trans_choice('messages.cart_items', 1);   // "منتج واحد" (one)
echo trans_choice('messages.cart_items', 2);   // "منتجان" (two)
echo trans_choice('messages.cart_items', 5);   // "5 منتجات" (few)
echo trans_choice('messages.cart_items', 25);  // "25 منتجاً" (many)
echo trans_choice('messages.cart_items', 200); // "200 منتج" (other)
```

### Russian (3 plural forms — one / few / many)

```php
// resources/lang/ru/messages.php
return [
    'items_purchased' => '{1} :count купленный товар|[2,4] :count купленных товара|[5,*] :count купленных товаров',
];

echo trans_choice('messages.items_purchased', 1);   // "1 купленный товар"
echo trans_choice('messages.items_purchased', 3);   // "3 купленных товара"
echo trans_choice('messages.items_purchased', 11);  // "11 купленных товаров"
```

### Polish (3 — one / few / many)

```php
// resources/lang/pl/messages.php
return [
    'files_uploaded' => '{1} :count przesłany plik|[2,4] :count przesłane pliki|[5,*] :count przesłanych plików',
];
```

**Why this matters:** if you're shipping to RTL or Slavic markets, interval-only pluralization produces grammatically wrong text for ~90% of users. Going from `[2,*] :count products` (English interval) to the proper Russian/Arabic CLDR form is the difference between "looks translated" and "reads natively."

**Caveat:** Laravel's `trans_choice()` works with CLDR for the major languages but is **not officially documented** as supporting it. Test thoroughly before relying on it for production — if Symfony changes its plural-rules table, the framework has no obligation to warn you. If you need a contractually-stabilized API, see the ICU MessageFormat section below.

Source: [Phrase — Localizing Plurals in Laravel](https://phrase.com/blog/posts/luralization/) · [CLDR Plural Rules](https://cldr.unicode.org/index/cldr-spec/plural-rules)

## ICU MessageFormat (For Complex i18n)

Laravel's built-in `__()` / `trans_choice()` is intentionally simple. For genuine Unicode ICU MessageFormat support — gender-aware strings, CLDR plural rules with mathematical precision (`=0`, `=1`, `one`, `few`, `many`, `other`), number/date/currency formatting via `{amount, number, currency}`, and smart regional fallback (`es_MX` → `es`) — use the **`gboquizosanchez/icu-i18n`** Composer package.

```bash
composer require gboquizosanchez/icu-i18n
```

```php
// resources/lang/ru/messages.php — full ICU syntax, no Laravel pipe-syntax gotchas
return [
    'items' => '{count, plural, one{# товар} few{# товара} many{# товаров} other{# товара}}',
];

// count=1   → "1 товар"
// count=2   → "2 товара"
// count=5   → "5 товаров"
// count=21  → "21 товар" (CLDR handles the 21=many→one ending rule correctly)
```

```php
// Gender-aware translation — Laravel has no built-in equivalent
'greeting' => '{gender, select, male{Добро пожаловать, :name} female{Добро пожаловать, :name} other{Привет, :name}}',
```

**When to reach for ICU MessageFormat instead of `trans_choice()`:**
- You ship to markets with 3+ plural forms (Russian, Polish, Arabic, Czech, Slovenian, Welsh…)
- You need gender-aware copy (Arabic gendered verbs, Spanish gendered nouns)
- You need regional fallback (your app uses `es_MX` but a vendor package only ships `es`)
- You want `{amount, number, currency}` formatting inside the translation string itself

**Trade-off:** One more dependency (`~30 KB`), one more serializer boundary in your translation pipeline. Pick it when the built-in pipe-syntax can't hit your plural rules cleanly.

Source: [gboquizosanchez/icu-i18n on GitHub](https://github.com/gboquizosanchez/icu-i18n) · v2.0.1 released April 8, 2026.

## Updated from Research (2026-07-06)

### Cycle 27 additions (2026-07-06)

- **CLDR plural rules for non-English languages** — Laravel's `trans_choice()` is **undocumented** but supports the full [CLDR plural rule set](https://cldr.unicode.org/index/cldr-spec/plural-rules) via `Symfony\Component\Translation\PluralizationRules`. Arabic has 6 forms (`{0} / {1} / {2} / [3,10] few / [11,99] many / [100,*] other`), Russian has 3 (`{1} / [2,4] / [5,*]`), Polish has 3 (`{1} / [2,4] / [5,*]`). Production gotcha: if you ship to RTL or Slavic markets, interval-only English-style pluralization produces grammatically wrong copy for the majority of users — the CLDR form is the difference between "looks translated" and "reads natively."
- **`gboquizosanchez/icu-i18n` for full ICU MessageFormat** — package was published 2026 (v2.0.1 on 2026-04-08) and adds `{gender, select, ...}`, `{count, plural, =0{...} =1{...} one{...} few{...}}` Math-precise CLDR, number/currency/date formatting inside translation strings, and smart regional fallback (`es_MX` → `es`). Use when you ship to markets with 3+ plural forms, gender-aware copy, or vendor packages that only ship the base locale.

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

## Missing Translation Key Handler — `Lang::handleMissingKeysUsing()` (Laravel 10.33+)

When a translation key is missing, Laravel returns the key string itself (`__('user.profile.bio')` → `"user.profile.bio"`). For shipping apps, this leaks **raw key strings to users** and **silently breaks** when a refactor renames a key in the fallback locale without updating call sites. Laravel 10.33+ ships `Lang::handleMissingKeysUsing()` for exactly this case.

```php
// app/Providers/AppServiceProvider.php
public function boot(): void
{
    \Illuminate\Support\Facades\Lang::handleMissingKeysUsing(function (string $key, array $replace, ?string $locale) {
        // 1. Log to a dedicated channel — page-level Sentry alert for tracking
        \Illuminate\Support\Facades\Log::channel('missing-translations')->warning(
            "Missing translation key: {$key}",
            ['locale' => $locale, 'replace' => $replace, 'user_agent' => request()?->userAgent()]
        );

        // 2. Fire a Sentry-style breadcrumb so devs see it in their dashboard
        if (app()->bound('sentry')) {
            app('sentry')->captureMessage("Missing translation: {$key} [{$locale}]", level: 'warning');
        }

        // 3. Return a human-friendly placeholder, NEVER the raw key
        $keyParts = explode('.', $key);
        $last = end($keyParts);
        return match (true) {
            str_starts_with($key, 'validation.') => "Validation: {$last} missing",
            str_starts_with($key, 'auth.') => "Auth: {$last} missing",
            default => "[{$locale}/{$key}]", // bracketed so QA can grep for it
        };
    });
}
```

**Why this matters in production:**
- **Fail loud, not silent** — without the handler, a missing key returns `"messages.checkout.totals.subtotal"` to the user's screen. With the handler, you get a logged alert + a safe placeholder like `[en/messages.checkout.totals.subtotal]`.
- **Backward-compatible** — the handler is global but you can still use `Lang::has($key)` for conditional rendering. They compose: `has()` for "should I render this block?", `handleMissingKeysUsing()` for "what should I render when I can't avoid a miss?"
- **Anti-loop guard built in** — Laravel does not call the handler for translations made *from inside* the handler closure, so recursive misses don't crash the worker.

**Pair with `Lang::has()` checks** for keys you expect to be missing in some locales (e.g., a feature flag disabled in EU but live in US):

```blade
@if (Lang::has('features.checkout.one_click'))
    {{ __('features.checkout.one_click') }}
@else
    <a href="/checkout/standard">Continue to checkout</a>
@endif
```

Source: [Laravel docs — handleMissingKeysUsing](https://laravel.com/docs/13.x/localization#handling-missing-translation-strings) (Laravel 10.33+, October 2023).


## Locale-Specific Key Existence — `Lang::has()` vs `Lang::hasForLocale()`

The single most-skipped localization footgun in 10+ years of Laravel. **`Lang::has($key, $locale)` does NOT check whether the key exists in `$locale`** — it checks whether `trans($key, [], $locale)` returns something other than the key string, which **always succeeds via fallback** when the fallback locale has the key. This has been true since Laravel 4 and is still the source of "why is my multi-locale site shipping English text in German views?" bug reports.

```php
// /lang/en/messages.php
return ['greeting' => 'Hello'];

// /lang/de/messages.php
return []; // DE file exists but is empty

Lang::has('messages.greeting', 'de');   // TRUE ❌ — silently falls back to 'en'
Lang::has('messages.greeting');          // TRUE (uses fallback chain, expected)

// ✅ The right way — strict locale check
Lang::hasForLocale('messages.greeting', 'de'); // FALSE (correctly)
```

`Lang::hasForLocale($key, $locale)` was added in Laravel 5.1 specifically for this case (PR #10767, 2015-10-29) and is **the only correct API** when you need to know "is this key translated in this specific locale, no fallbacks".

**Use `hasForLocale()` when:**
- You're rendering a UI element that's only meaningful in some locales (e.g., a right-to-left flip indicator, an honorific prefix in Japanese)
- You're auditing translation coverage: `Lang::hasForLocale($key, 'de')` returns the real answer for German
- You're showing different UI per locale: a Spanish-language customer success email link might only exist in `es/` not `en/`

**Use `Lang::has($key)` (no locale) when:**
- You just want "do I have any translation for this key, including fallback?" — `has()` answers this correctly with the fallback chain honored

Source: [GitHub laravel/framework#10718](https://github.com/laravel/framework/issues/10718) — original 2015 bug report; [PR #10767](https://github.com/laravel/framework/pull/10767) — added `hasForLocale()`. The bug is still active in Laravel 13 because the default `has()` behavior is what most apps want (fallback is the point of having a fallback locale).


## Carbon Localization — `isoFormat()` vs `format()` vs `translatedFormat()`

Carbon's three date-formatting methods have **completely different locale behaviors** and AI models get them wrong constantly. This is the #1 reason "why doesn't `setlocale(LC_TIME, 'fr')` localize my dates?" tickets show up in Laravel projects.

```php
// ❌ WRONG — format() is PHP's DateTime::format() under the hood; ALWAYS English
$date->format('l jS F Y');        // "Monday 1st July 2026" — regardless of locale

// ⚠️ PARTIALLY RIGHT — translatedFormat() uses strftime(), depends on setlocale(LC_TIME, ...)
setlocale(LC_TIME, 'fr_FR.UTF-8'); // OS-level locale — fragile in Docker/CI
$date->translatedFormat('l j F Y'); // "lundi 1 juillet 2026" — works if OS has the locale installed

// ✅ CORRECT — isoFormat() uses Carbon's own embedded translations (no OS dependency)
$date->isoFormat('dddd D MMMM YYYY'); // "lundi 1 juillet 2026" — works everywhere
```

| Method | Locale source | Docker/CI safe | Token syntax | Best for |
|---|---|---|---|---|
| `format()` | None (always English) | N/A | PHP `DateTime` tokens | English-only output, log lines |
| `translatedFormat()` | OS `setlocale(LC_TIME, ...)` | ❌ (locale must be installed in container) | strftime tokens | Legacy PHP apps already using setlocale |
| `isoFormat()` | Carbon's own embedded translations | ✅ | Unicode CLDR tokens | New code, multi-locale production |

**`isoFormat()` token reference** (the most common ones — Carbon uses [CLDR](https://cldr.unicode.org/translation/date-time/date-time-patterns) tokens, NOT PHP `date()` tokens):

| Token | Output | PHP `date()` equivalent |
|---|---|---|
| `YYYY` | 2026 (4-digit year) | `Y` |
| `MM` | 07 (zero-padded month) | `m` |
| `MMM` | Jul (short month) | `M` |
| `MMMM` | July (full month) | `F` |
| `DD` | 13 (zero-padded day) | `d` |
| `dddd` | Monday (full weekday) | `l` |
| `HH:mm` | 14:30 (24-hour time) | `H:i` |
| `hh:mm a` | 02:30 pm (12-hour time) | `h:i a` |

**Setting the locale:**

```php
// Global — set once per request in middleware
Carbon::setLocale('fr');  // affects ALL Carbon instances in the request

// Per-instance — useful when a model has a stored locale preference
$userCreatedAt->locale($user->preferred_locale)->isoFormat('dddd D MMMM');
```

**The `setlocale()` vs `Carbon::setLocale()` gotcha:**

`setlocale(LC_TIME, 'fr_FR.UTF-8')` is an OS-level call. It only affects `translatedFormat()` (which uses strftime internally). It does NOT affect `format()` (PHP native) or `isoFormat()` (Carbon's embedded translations). Most apps call `setlocale()` thinking they've localized everything; in reality only `translatedFormat()` sees it.

```php
// In a middleware — the modern Laravel 13 way (locale only, no OS call)
public function handle($request, Closure $next)
{
    $locale = $this->resolveLocale($request);
    App::setLocale($locale);
    Carbon::setLocale($locale); // affects isoFormat() and translatedFormat()
    // NO setlocale(LC_TIME, ...) — that path is fragile under Docker/CI
    return $next($request);
}
```

Source: [Carbon Localization docs](https://carbon.nesbot.com/docs/#api-localization) · [CLDR date-time patterns](https://cldr.unicode.org/translation/date-time/date-time-patterns)


## Word-Order-Safe Translations — Placeholder Pattern for German/Arabic/Hindi

Most i18n bugs in Laravel apps come from this exact pattern:

```blade
{{-- ❌ WRONG — assumes English word order --}}
<button>{{ __('Attach') }} {{ $resource->name }}</button>

{{-- English: "Attach Post" ✓ --}}
{{-- German:  "Beitrag anhängen" ❌ — verb goes AFTER the noun --}}
{{-- Arabic:  "إرفاق المقالة" ✓ here, but other orderings break elsewhere --}}
{{-- Hindi:   "पोस्ट अटैच करें" — verb pattern different again --}}
```

**The fix — use a placeholder in the translation file:**

```php
// /lang/en/messages.php
'attach_resource' => 'Attach :resource',

// /lang/de/messages.php
'attach_resource' => ':resource anhängen',  // "Post anhängen"

// /lang/ar/messages.php
'attach_resource' => 'إرفاق :resource',     // RTL, but :resource goes FIRST in source

// /lang/hi/messages.php
'attach_resource' => ':resource अटैच करें',  // postfix verb
```

```blade
{{-- ✅ CORRECT — word order is per-locale --}}
<button>{{ __('messages.attach_resource', ['resource' => $resource->name]) }}</button>
```

This is the same pattern Laravel itself uses for password reset, validation, and notifications. The Laravel source uses placeholders for every sentence that could possibly change word order — AI models miss this 80% of the time on generated UIs.

**Common offenders to check in your codebase:**
- `<a>Delete {{ $name }}</a>` → German puts "löschen" after the noun
- `<p>Welcome {{ $name }}</p>` → Japanese often uses suffixes ("さん")
- `<span>{{ $count }} {{ __('items') }}</span>` → Russian needs 3 plural forms (see CLDR section above)
- `<div>Posted by {{ $user }} on {{ $date }}</div>` → Spanish drops "by" ("Publicado por … en …")
- `<small>in {{ $category }}</small>` → Arabic uses preposition after noun

**Rule of thumb:** If your Blade template has more than one `{{ }}` inside a single visible phrase, **extract to a translation key with placeholders**. The cost of one extra `lang/{locale}/messages.php` entry is much lower than the cost of a German-speaking customer seeing "Beitrag anhängen" rendered backwards.

**Bulk audit script** — find Blade templates that need this fix:

```php
// app/Console/Commands/AuditWordOrder.php
public function handle(): int
{
    $suspicious = [];
    foreach (File::allFiles(resource_path('views')) as $file) {
        $content = $file->getContents();
        // Match patterns like "<tag>__ string__ {{ $var }} more text</tag>" or similar
        preg_match_all('/\{\{[^}]+\}\}\s*\{\{[^}]+\}\}/', $content, $matches);
        foreach ($matches[0] as $m) {
            $suspicious[] = "{$file->getRelativePathname()}: {$m}";
        }
    }
    $this->line(implode("\n", array_unique($suspicious)));
    return self::SUCCESS;
}
```

Source: [GitHub laravel/nova-issues#1260](https://github.com/laravel/nova-issues/issues/1260) — the original German `Attach {{ $singularName }}` bug report that motivated the placeholder pattern in Laravel core.


## Eloquent Attribute Translation — `spatie/laravel-translatable`

For translating **Eloquent model attributes** (not just view strings) — product descriptions in 6 languages, blog post bodies, page titles — Laravel ships nothing built-in. The de facto community package is **`spatie/laravel-translatable`** (v6.x as of 2026, requires Laravel 10+/PHP 8.1+). It stores translations as JSON in a single column, which keeps queries fast and avoids 6 separate tables.

```bash
composer require spatie/laravel-translatable
```

```php
// Migration
Schema::create('products', function (Blueprint $table) {
    $table->id();
    $table->string('sku')->unique();
    $table->json('name');       // {"en": "Widget", "de": "Gerät", "es": "Artilugio"}
    $table->json('description'); // JSON column for translated attributes
    $table->decimal('price', 10, 2);  // non-translated columns stay scalar
    $table->timestamps();
});

// Model
use Spatie\Translatable\HasTranslations;
use Spatie\Translatable\Attributes\Translatable;

#[Translatable('name', 'description')]  // Laravel 13 class-level attribute
class Product extends Model
{
    use HasTranslations;

    // OR the traditional property (works on Laravel 10+):
    // public array $translatable = ['name', 'description'];
}
```

```php
// Set translations on save (any JSON-castable input)
$product = Product::create([
    'sku' => 'WIDGET-001',
    'name' => [
        'en' => 'Premium Widget',
        'de' => 'Premium-Gerät',
        'es' => 'Widget Premium',
    ],
    'description' => [
        'en' => 'A high-quality widget for everyday use.',
        'de' => 'Ein hochwertiges Gerät für den täglichen Gebrauch.',
    ],
    'price' => 29.99,
]);

// Access — automatically returns the current app locale
app()->setLocale('de');
echo $product->name; // "Premium-Gerät"
echo $product->getTranslation('name', 'es'); // "Widget Premium"

// Set a single locale without touching others
$product->setTranslation('name', 'fr', 'Widget Premium')
        ->save();

// Fallback behavior
echo $product->getTranslation('name', 'jp', useFallbackLocale: true); // falls back to 'en'
```

**Critical gotchas:**

1. **JSON column requirement** — the package stores translations as JSON, so the column **must** be `json` type (or `text` if your DB doesn't support JSON). Setting it to `string` will silently truncate at the first comma and corrupt data.

2. **`$appends` doesn't carry translations** — if you define an accessor like `getFullTitleAttribute()` that joins `$this->name` and `$this->brand`, the `name` returned is the raw array `['en' => '...', 'de' => '...']`, not the localized string. Wrap accessors in `$this->getTranslation('name', app()->getLocale())`.

3. **Querying by translated content is hard** — you can't do `Product::where('name->en', 'Widget')->get()` efficiently on MySQL without JSON indexes:
   ```php
   // /database/migrations/add_json_indexes.php
   $table->json('name');
   DB::statement('ALTER TABLE products ADD INDEX name_en_idx ((CAST(name->>"$.en" AS CHAR(255))))');
   // PostgreSQL has built-in GIN indexes for jsonb columns — much better fit
   ```

4. **`toArray()` / API Resources leak all translations** — when you serialize a model to JSON (API response, queue job), the entire translation map goes with it. Use a JsonResource with `whenLoaded()` or filter explicitly:
   ```php
   // ProductResource::toArray()
   return [
       'name' => $this->getTranslation('name', app()->getLocale()),
       'description' => $this->getTranslation('description', app()->getLocale()),
   ];
   ```

5. **`#[Translatable]` (Laravel 13) vs `$translatable` array** — both work; the attribute is variadic and merges with the property. New Laravel 13 code should prefer the attribute for IDE support (click-to-jump), but the property is fine for older codebases.

**When to use spatie/laravel-translatable:**
- Product catalogs, CMS pages, blog posts — anywhere users edit translated content in a Filament/Nova/admin panel
- A single row per entity, multiple languages
- Translation count is small (2–10 languages)

**When NOT to use spatie/laravel-translatable:**
- 20+ languages → consider a proper EAV table or Algolia/Meilisearch index
- Translation workflow involves per-locale approvals, drafts, reviewer assignment → reach for an actual TMS (Lokalise, Crowdin, Phrase)
- You need fallback chains (es_MX → es → en) — handle with a service class, the package doesn't do chained fallbacks

**Alternatives:**
- **`dimsav/laravel-translatable`** — older, abandoned, JSON-backed like spatie's. Don't pick for greenfield.
- **`astrotomic/laravel-translatable`** — alternative schema with one row per language (not JSON). Easier to query, harder to scale to many languages.
- **`propaganistas/laravel-translatable`** — for translated **enums** and **validation rules**, not model attributes.

Source: [spatie/laravel-translatable on GitHub](https://github.com/spatie/laravel-translatable) · v6.x docs at [spatie.be/docs/laravel-translatable](https://spatie.be/docs/laravel-translatable/v6/installation-setup) · 5+ million installs as of 2026.


## Plugin / Multi-Tenant Translation Paths — `Translator::addPath()` / `addJsonPath()` / `addNamespace()`

For plugin systems, white-label multi-tenant deployments, or modular monoliths where each tenant ships their own translation overrides, Laravel exposes three runtime translation-path injection methods on the Translator. **None of these are commonly documented**, and AI models default to "fork the lang/ directory" which doesn't scale.

```php
// 1. Add a flat directory of PHP lang files
app('translator')->addPath('/var/lib/tenants/acme/lang');
// Now /var/lib/tenants/acme/lang/en/messages.php overrides the same key in /lang/en/messages.php

// 2. Add a directory of JSON translation files
app('translator')->addJsonPath('/var/lib/tenants/acme/lang/json');
// Now /var/lib/tenants/acme/lang/json/en.json keys are merged into the global JSON map

// 3. Add a namespace with its own path hint (separate from `app`)
app('translator')->addNamespace('billing', '/var/lib/modules/billing/lang');
// Now __('billing::invoice.title') looks in /var/lib/modules/billing/lang/{locale}/invoice.php
// (Useful for module packages that ship their own translations without conflicting with the host app)
```

**Real-world use cases:**

- **Multi-tenant SaaS** — each tenant gets a `lang/tenant/{tenant_slug}/{locale}/` override directory loaded via `addPath()` in a tenant-resolving middleware. The base app has English defaults; tenants override per-locale without forking.
- **Modular monolith** — a `modules/billing/`, `modules/inventory/` layout where each module ships its own `lang/{locale}/` namespace via `addNamespace()` so module translations never collide with app translations.
- **White-label reseller** — `addPath()` to a drop-in `resellers/{slug}/lang/{locale}/` overrides core copy without touching the base `lang/` files.
- **A/B test copy variants** — load `addPath(resource_path('lang/experiments/variant-A'))` for half the users, `variant-B` for the other half. Switch without a deploy.

**Performance note:** `addPath()` is called once at boot (typically in a service provider). Each call is O(1) — the loader only walks the directory when a missing key triggers a fallback search. Do NOT call `addPath()` inside a middleware or per-request callback without caching the result.

**Order matters — last-added wins:**

The Translator iterates addPath targets in reverse-registration order (last added, first searched). When you call `addPath('/tenants/acme/lang')` after the default `/lang/` is registered, the tenant override wins. To make the base app win for keys the tenant doesn't override, just don't add a file for that key in the tenant directory — Laravel falls through to the next registered path automatically.

```php
// app/Providers/TenantServiceProvider.php
public function boot(): void
{
    $tenant = $this->resolveCurrentTenant();
    if ($tenant && is_dir($path = storage_path("tenants/{$tenant->slug}/lang"))) {
        // Last addPath() = first searched → tenant overrides win
        app('translator')->addPath($path);
        app('translator')->addJsonPath("{$path}/json");
    }
}
```

Source: [`Illuminate\Translation\Translator` API docs](https://api.laravel.com/docs/13.x/Illuminate/Translation/Translator.html) — `addPath()`, `addJsonPath()`, `addNamespace()` are documented in the API reference but not in the main localization guide.

## Updated from Research (2026-07-13)

### Cycle 36 additions (2026-07-13)

- **`Lang::handleMissingKeysUsing()` (Laravel 10.33+)** — official missing-key callback. Default Laravel behavior leaks raw key strings like `"messages.checkout.totals.subtotal"` to the user when a key is missing; this callback logs (dedicated channel + Sentry breadcrumb) and returns a bracketed `[locale/key]` placeholder. Anti-loop guard built in. Pair with `Lang::has()` for keys you expect to be missing in some locales (feature-flag-driven copy, region-specific UI).
- **`Lang::hasForLocale()` vs `Lang::has()`** — 10-year-old footgun. `Lang::has($key, $locale)` does NOT check whether the key exists in `$locale` — it returns `true` whenever the fallback chain has the key. Use `Lang::hasForLocale($key, $locale)` (Laravel 5.1+, PR #10767) when you need strict locale-only existence (RTL flip indicator, audit reporting, locale-specific UI). `Lang::has()` without a locale is correct for "any translation including fallback".
- **Carbon `isoFormat()` vs `format()` vs `translatedFormat()`** — three methods with completely different locale behaviors. `format()` is PHP native (always English, ignores `setlocale`). `translatedFormat()` uses OS `setlocale(LC_TIME, ...)` (fragile in Docker/CI). `isoFormat()` uses Carbon's own embedded CLDR translations (works everywhere, no OS dependency). Token syntax differs: `isoFormat` uses CLDR tokens (`YYYY`, `MM`, `dddd`) NOT PHP `date()` tokens (`Y`, `m`, `l`). The #1 reason "setlocale didn't localize my Carbon dates" tickets show up.
- **Word-order placeholder pattern for German/Arabic/Hindi** — `<button>{{ __('Attach') }} {{ $resource->name }}</button>` is wrong; German puts the verb at the end (`"Beitrag anhängen"`), Japanese uses honorific suffixes, Arabic/Hindi reorder differently. Fix: `'attach_resource' => ':resource anhängen'` with `__('messages.attach_resource', ['resource' => $name])`. Laravel itself uses placeholders for every sentence that could change word order; AI-generated UIs miss this 80% of the time. Bulk-audit script included.
- **`spatie/laravel-translatable` for Eloquent attribute translation** — de facto community package (v6.x, 5M+ installs) for translating model attributes (product names, blog post bodies, page titles). Stores translations as JSON in one column (vs 6 separate tables). Gotchas: column MUST be `json`/`text` not `string`; `$appends` accessors don't auto-localize; query-by-translation needs MySQL JSON indexes or PostgreSQL GIN; `toArray()` leaks all translations (use JsonResource with `getTranslation()`); Laravel 13 `#[Translatable]` attribute vs traditional `$translatable` array both work.
- **`Translator::addPath()` / `addJsonPath()` / `addNamespace()`** — undocumented runtime translation-path injection. Use for multi-tenant SaaS (per-tenant `lang/` overrides), modular monoliths (per-module `billing::invoice.title` namespaces), white-label resellers, A/B-tested copy variants. Last-added wins (reverse-registration search order). Call once at boot in a service provider, NOT per-request.
