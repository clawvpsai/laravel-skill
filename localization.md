# Localization — Translations, Locales, Pluralization

> Laravel's localization system handles multi-language content, pluralization, number formatting, and locale-aware date presentation.

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

## Common Mistakes

1. **`trans()` vs `__()`** — both work, `__()` is the newer helper
2. **Missing fallbacks** — always set `fallback_locale`
3. **Hardcoded strings** — extract every user-facing string to lang files
4. **Plural without trans_choice** — use `{0}|{1}|[2,*]` syntax for count-aware messages
5. **Not using Number::format** — locale-aware number formatting, not `number_format()`
6. **Translation keys not namespaced** — use dot notation (`messages.greeting`) to avoid collisions
7. **Missing locale in URL routing** — SEO requires `hreflang` tags and locale-prefixed URLs
8. **No missing key fallback UI** — log or flag untranslated keys in staging

## SEO — hreflang Tags

```php
// In your layout Blade file
@foreach (['en', 'es', 'fr'] as $locale)
    <link rel="alternate" hreflang="{{ $locale }}"
          href="{{ localized_url($locale, url()->current()) }}">
@endforeach
<link rel="alternate" hreflang="x-default" href="{{ localized_url('en', url()->current()) }}">
```

## Updated from Research (2026-05)
- Laravel 13 improves Number formatting with locale-aware currency, percentage, and spell-out formatting
- Browser-based locale detection via `request()->getPreferredLanguage()`
- Interval-based pluralization in `trans_choice()` (e.g., `[2,10]` range)
- Database-backed translations for dynamic content (products, CMS pages)
- RTL locale support requires direction-aware layouts

Sources: [Laravel Localization](https://laravel.com/docs/13.x/localization) | [Laravel Number Helper](https://laravel.com/docs/13.x/helpers#method-number)
