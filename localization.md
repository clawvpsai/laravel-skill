# Localization — Translations, Locales, Pluralization

## Setup

```php
// config/app.php
'locale' => 'en',           // current locale
'fallback_locale' => 'en', // when translation missing
'locales' => ['en', 'es', 'fr', 'de', 'hi', 'gu'],
```

## Translation Files

```php
// resources/lang/en/messages.php
return [
    'welcome' => 'Welcome :name',
    'new_posts' => '{0} No posts|{1} One post|[2,*] :count posts',
    'published_at' => 'Published on :date',
];
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
```

## Number/Date Formatting

```php
use Illuminate\Support\Number;

// Number formatting
Number::format(1234.56, 2); // "1,234.56"
Number::spell(42);          // "forty-two"
Number::currency(1234.56);  // "$1,234.56"

// Date formatting by locale
Carbon::parse($date)->locale('hi_IN')->format('d MMMM yyyy');
```

## Locale Detection

```php
// Detect from browser
app()->setLocale(request()->getPreferredLanguage());

// Force locale
URL::setLocale('es');
Route::get('{locale}/posts', ...)->name('posts.localized');

// Persisted in session
session(['locale' => 'es']);
```

## Common Mistakes

1. **`trans()` vs `__()`** — both work, `__()` is the newer helper
2. **Missing fallbacks** — always set `fallback_locale`
3. **Hardcoded strings** — extract every user-facing string to lang files
4. **Plural without trans_choice** — use `{0}|{1}|[2,*]` syntax for count-aware messages
5. **Not using Number::format** — locale-aware number formatting, not `number_format()`

## Updated from Research (2026-05)
- Laravel 13 improves Number formatting with locale-aware currency and percentage formatting
- Browser-based locale detection is available via `request()->getPreferredLanguage()`

Source: [Laravel Localization](https://laravel.com/docs/13.x/localization)
