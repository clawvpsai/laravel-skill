# Blade — Templates, Components, XSS

## Golden Rules

### NEVER use `{!! !!}` unless you fully control the HTML

```html
<!-- DANGEROUS — renders raw HTML, XSS vector -->
{!! $post->html_content !!}

<!-- SAFE — escapes all HTML -->
{{ $post->html_content }}

<!-- Only use {!! when you know the content is sanitized HTML -->
@if($post->html_content)
    {!! clean($post->html_content) !!}  <!-- via HTMLPurifier or similar -->
@endif
```

### Blade `{{ }}` auto-escapes by default

```php
{{ '<script>alert(1)</script>' }}  
<!-- Renders as: &lt;script&gt;alert(1)&lt;/script&gt; — safe -->
```

## Components

```bash
# Anonymous component (just a view)
php artisan make:component Alert

# Class-based component
php artisan make:component PostCard --view
```

**Component class:**
```php
<?php
// app/View/Components/PostCard.php
namespace App\View\Components;

use Illuminate\View\Component;

class PostCard extends Component
{
    public function __construct(
        public string $title,
        public string $body = '',
        public bool $featured = false,
    ) {}

    public function render()
    {
        return view('components.post-card');
    }
}
```

**Usage:**
```html
<x-post-card title="Hello" body="Content here" :featured="$isFeatured" />
```

### @props Directive (pass all public properties to view)

For class-based components, use `@props` in the view to auto-pass all public constructor properties without explicitly calling `$this->with()`:

```php
// app/View/Components/StatsCard.php
class StatsCard extends Component
{
    public function __construct(
        public int $views = 0,
        public int $likes = 0,
        public string $label = 'Stats',
    ) {}

    public function render()
    {
        return view('components.stats-card');
    }
}
```

```blade
{{-- resources/views/components/stats-card.blade.php --}}
@props(['views', 'likes', 'label'])

<div class="stats-card">
    <h3>{{ $label }}</h3>
    <p>👁 {{ $views }} · ❤️ {{ $likes }}</p>
</div>
```

Then call it simply:
```html
<x-stats-card :views="$post->views" :likes="$post->likes" label="Popular Post" />
```

**Without `@props`, you must explicitly pass data:**
```php
// Instead of relying on @props, you can also use:
public function render()
{
    return view('components.stats-card')->with([
        'views' => $this->views,
        'likes' => $this->likes,
        'label' => $this->label,
    ]);
}
```

## Slots

```php
<!-- layout.blade.php -->
<div class="card">
    <div class="card-header">@yield('title')</div>
    <div class="card-body">
        {{ $slot }}
    </div>
</div>

<!-- page.blade.php -->
<x-layout>
    <x-slot name="title">My Page Title</x-slot>
    <p>Page content</p>
</x-layout>
```

## Directives

```html
<!-- If/else -->
@if($user)
    <p>Hello {{ $user->name }}</p>
@elseif($user->isAdmin())
    <p>Welcome admin</p>
@else
    <p>Guest</p>
@endif

<!-- Auth -->
@auth
    <p>Logged in as {{ auth()->user()->name }}</p>
@endauth

@guest
    <a href="/login">Login</a>
@endguest

<!-- Loop -->
@forelse($posts as $post)
    <li>{{ $post->title }}</li>
@empty
    <li>No posts found</li>
@endforelse

<!-- PHP -->
@php
    $counter = $counter + 1;
@endphp

<!-- Include -->
@include('partials.header', ['title' => 'About'])

<!-- Components -->
@component('alert', ['type' => 'error'])
    @slot('title')
        Error
    @endslot
    Something went wrong.
@endcomponent
```

## CSRF in Forms

```html
<form method="POST" action="/posts">
    @csrf
    @method('PUT')
    <input name="title" value="{{ old('title') }}">
    @error('title')
        <span class="error">{{ $message }}</span>
    @enderror
</form>
```

## Asset URLs

```html
<!-- Mix (Vite build output) -->
<link href="{{ mix('css/app.css') }}">

<!-- URL -->
<img src="{{ asset('images/logo.png') }}">
<img src="{{ storage_path('app/public/avatars/' . $user->avatar) }}">
```

## @fonts Directive (Laravel 13.7+)

The `@fonts` directive preloads Vite-managed font files for better performance:

```blade
{{-- Load custom fonts from Vite bundle --}}
@fonts

{{-- Works with <link rel="preload"> for font files in your Vite manifest --}}
```

This generates `<link rel="preload">` tags for fonts managed by Vite, improving Core Web Vitals (FCP, LCP) by reducing font render-blocking.

**Typical usage in layout:**
```blade
<head>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
    @fonts
</head>
```

## View Composers (run on every render of a view)

```php
// AppServiceProvider boot()
View::composer('layouts.app', function (View $view) {
    $view->with('notifications', auth()->user()?->unreadNotifications);
});

// Wildcard
View::composer('partials.*', fn(View $view) => $view->with('theme', 'dark'));
```



## `@once` / `@pushOnce` (Stacks Without Duplicates, Laravel 13.x Improved)

Use `@once` to emit a block of HTML/JS/CSS exactly once per render cycle (e.g., a one-time modal include, an analytics snippet). `@pushOnce` does the same for `@stack` content:

```blade
{{-- Renders the script tag exactly once, even if included 10 times via @include --}}
@once
    @push('scripts')
        <script src="{{ asset('js/charts.js') }}"></script>
    @endpush
@endonce

{{-- In your layout: --}}
<head>
    @stack('scripts')
</head>
```

**Laravel 13.x improvement:** `@once` and `@pushOnce` now correctly track identity across `@include` chains, partial components, and `<x-dynamic-component>` — previously nested includes could each render the block if the tracking was reset by an intervening directive. This matters for admin layouts that include the same partial from multiple sub-views.

**When NOT to use:** if you actually want the block to render on every include (e.g., a per-row form input), use `@push` (not `@pushOnce`).

## Anonymous Component Props (Laravel 13.x)

For one-off components (e.g., a single alert in one view), anonymous components let you define the view AND its props in one file:

```blade
{{-- resources/views/components/alert.blade.php --}}
@props(['type' => 'info', 'message'])

<div class="alert alert-{{ $type }}">
    {{ $message }}
</div>
```

Used as:
```html
<x-alert type="success" :message="$post->title" />
<x-alert message="Default info type" />
```

Anonymous components are auto-discovered from `resources/views/components/`. Use them when you don't need a class-based component with logic — keeps the file count down for tiny presentational pieces.
## Common Mistakes

1. **`{!! !!}` with user-submitted content** — XSS vulnerability
2. **Using `{!!` for debug output** — debug tools that use `{!!` should only be for trusted internal data
3. **Not using `@csrf` in forms** — CSRF token missing, request rejected
4. **Forgetting `@method` for PUT/PATCH/DELETE** — wrong HTTP method, 419 error
5. **Old `{{$var}}` (no space)** — deprecated in Laravel 10, use `{{ $var }}`
6. **`new HtmlString($userInput)`** — `HtmlString` is the *class-based* equivalent of `{!! !!}` and bypasses escaping. CVE-2026-33080 (Filament) was a real-world instance of this — always `e($value)` before wrapping. Use `{{ }}` / `e()` first, then wrap in `HtmlString` only for known-safe HTML

## Updated from Research (2026-06-28, cycle 6)

- **`@once` / `@pushOnce` 13.x fix** — identity now persists across nested `@include` and `<x-dynamic-component>` (previously could re-render the block when tracking was reset by an intervening directive). Critical for shared admin partials.
- **Anonymous component props** — define props inline in the component view with `@props([...])`; auto-discovered from `resources/views/components/`. Skips the class file for one-off presentational components.
- **New Common Mistake #6** — `new HtmlString($userInput)` is the class-based equivalent of `{!! !!}` and skips escaping. Same root cause as CVE-2026-33080 in Filament. Always `e($value)` before wrapping.

## Updated from Research (2026-05)

- **@fonts Directive (Laravel 13.7+)** — `@fonts` generates `<link rel="preload">` tags for Vite-managed fonts, improving Core Web Vitals.
- **@props Directive** — pass all public constructor properties to a component view inline without `->with()`.

Source: [Laravel 13 Blade](https://laravel.com/docs/13.x/blade) | [Laravel 13 Components](https://laravel.com/docs/13.x/blade#components)
