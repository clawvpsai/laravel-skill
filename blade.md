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

## Form-Helper Directives (`@checked` / `@selected` / `@disabled` / `@readonly` / `@required`)

Since Laravel 9, Blade ships form-helper directives that collapse the old verbose `{{ old(...) == $x ? 'checked' : '' }}` pattern into a single conditional attribute. **These are the most-missed modern Blade directives** — AI models still write the verbose version by default.

```blade
{{-- @checked — emits checked="checked" when the expression is truthy --}}
<input type="checkbox" name="active" value="1" @checked(old('active', $user->active)) />
<input type="radio" name="role" value="admin" @checked($user->role === 'admin') />

{{-- @selected — for <option selected="selected"> --}}
<select name="version">
    @foreach ($product->versions as $version)
        <option value="{{ $version }}" @selected(old('version', $product->version) == $version)>
            {{ $version }}
        </option>
    @endforeach
</select>

{{-- @disabled — emits disabled="disabled" when truthy --}}
<button type="submit" @disabled($errors->isNotEmpty())>Submit</button>
<input type="text" name="email" @disabled(! auth()->user()->can('update', $post)) />

{{-- @readonly — emits readonly="readonly" when truthy --}}
<input type="email" name="email" value="{{ $user->email }}" @readonly(! $user->isAdmin()) />

{{-- @required — emits required="required" when truthy (HTML form validation only) --}}
<input type="text" name="title" @required($user->isAdmin()) />
```

**Why this matters:**

- **XSS-safety:** The directives emit a static attribute string when the expression is truthy and *nothing at all* when it's falsy — they never concatenate user input, so they're safer than hand-rolled `{{ ... ? 'checked' : '' }}` patterns that some authors mistakenly pass `$userInput` into.
- **Readability:** `@checked(old('active', $user->active))` reads like English; the verbose version does not.
- **Accessibility:** `@required` + `@readonly` map directly to the same-named HTML attributes that screen readers announce — don't fake them with `aria-*` attributes.
- **Server-side validation is still required:** `@required` only adds the HTML5 attribute — clients can bypass it. Pair with `required` in your `validate()` rules.

**Common AI mistake to watch for:** never use `@required($value)` with raw user input — the directive only emits the literal HTML attribute when truthy, but a stale `$value` from a previously-failed form can leak through. Always go through `old()` or a validated model attribute.

## `@verbatim` Directive (Vue / Alpine / Vanilla JS Templates)

When a Blade template embeds a large chunk of a frontend framework's template syntax (Vue, Alpine, Svelte, vanilla `<script>` tags), every `{{ }}` and `@click` would otherwise be parsed by Blade. `@verbatim` tells Blade to leave the contents untouched:

```blade
@verbatim
    <div class="container" x-data="{ open: false }">
        <button @click="open = !open">Toggle</button>
        <p x-show="open">Hello, {{ user.name }}</p>
    </div>
@endverbatim
```

Inside `@verbatim`, `{{ user.name }}` is passed through verbatim to the browser, where Alpine evaluates it. The `@click` is also preserved (Blade would otherwise error on unknown directives).

**When to use:** any Blade view that includes a meaningful chunk of frontend-framework syntax. Don't use it for trivial cases — `@{{ jsVariable }}` (the `@` prefix) still works for one-off JS `{{ }}` interpolations.

**Gotcha:** `@verbatim` cannot be nested. If you need two separate verbatim blocks, close and reopen.

## Rendering Inline Blade Strings (`Blade::render`)

`Illuminate\Support\Facades\Blade::render()` compiles and renders a Blade template string to HTML. Critical for AI-generated content, CMS user templates, email templates stored in the DB, and dynamic theme/preview systems:

```php
use Illuminate\Support\Facades\Blade;

// Simple string render
$html = Blade::render('Hello, {{ $name }}', ['name' => 'Julian Bashir']);

// Full directive support — @if / @foreach / @include / components all work
$html = Blade::render(
    '@if($user->isAdmin()) <p>Admin</p> @else <p>User</p> @endif',
    ['user' => $user]
);

// Delete the cached view file after rendering (recommended for one-off / user-supplied templates)
$html = Blade::render(
    $userTemplate,
    ['order' => $order, 'customer' => $customer],
    deleteCachedView: true
);
```

**Behavior:**
- Templates are written to `storage/framework/views/` (the normal Blade cache) before rendering.
- Pass `deleteCachedView: true` if the template came from user input — this prevents cache-pollution attacks where a malicious template persists across requests.
- The second argument is the data context — variables not in the array are NOT available (unlike `view()`, you can't rely on `auth()->user()` being implicitly bound).
- **XSS / RCE warning:** `Blade::render()` of user-supplied templates does NOT sandbox. A user who can author a template string can execute arbitrary PHP via `@php` directives, call arbitrary static methods via `{{ App\Services\X::dangerous() }}`, or include arbitrary components via `<x-*>`. **Always sanitize user-supplied templates with a whitelist of allowed directives**, or use a dedicated templating engine (Twig, Handlebars, Mustache) for user-facing templates. The Laravel AI SDK does NOT use `Blade::render()` for untrusted output for exactly this reason.

**Common use cases:**
- **AI SDK output rendering** for trusted system templates — pass them through `Blade::render()` to get the final HTML.
- **CMS templates** stored in DB (email builder widgets, landing-page builders).
- **Webhook payloads** that embed HTML — render server-side, never trust the client.
- **Email templates** stored per-tenant in the database.

## Inline Component Views (`<<<'blade'` Heredoc)

Class-based components can return the Blade template directly from `render()` instead of pointing at a separate file. Use this for small components that don't deserve their own `.blade.php` file:

```php
<?php

namespace App\View\Components;

use Illuminate\View\Component;

class Alert extends Component
{
    public string $type;
    public string $message;

    public function __construct(string $type = 'info', string $message = '')
    {
        $this->type = $type;
        $this->message = $message;
    }

    public function render(): string
    {
        return <<<'blade'
            <div class="alert alert-{{ $type }}">
                {{ $message }}
            </div>
        blade;
    }
}
```

**Why `<<<'blade'` (nowdoc):** the leading single-quote identifier tells PHP to leave the contents completely literal — no variable interpolation, no escape-sequence processing. The `blade` tag name is just a convention; any identifier works.

**When to use:** small components where a separate `.blade.php` file would be overkill. For anything >20 lines or that includes sub-views, use a real view file — they cache better and are easier to syntax-highlight.

**Watch out:** IDE Blade syntax highlighting doesn't recognize inline heredoc components. If your team relies on tooling, prefer external view files for non-trivial components.


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

## Updated from Research (2026-07-02, cycle 19)

- **Form-helper directives (`@checked` / `@selected` / `@disabled` / `@readonly` / `@required`)** — added full section. These directives collapse the verbose `{{ old('foo') == $x ? 'checked' : '' }}` pattern that AI models still write by default. They emit a static attribute when truthy and nothing when falsy, so they're also safer than hand-rolled concatenations. `@required` adds only the HTML5 attribute — pair with server-side `required` validation rules.
- **`@verbatim` directive (Vue / Alpine / vanilla JS)** — added full section. Tells Blade to leave the contents untouched so frontend-framework template syntax (`{{ var }}`, `@click`) doesn't get parsed. Don't use it for trivial cases — `@{{ jsVar }}` still works for one-off JS interpolations.
- **`Blade::render()` — inline string rendering** — added full section. Critical for AI-generated output rendering, CMS user templates, DB-stored email templates. Watch the **RCE / XSS warning**: `Blade::render()` does NOT sandbox user-supplied templates — a malicious template can use `@php` directives or call arbitrary static methods via `{{ \App\Services\X::dangerous() }}`. Always pass `deleteCachedView: true` for untrusted templates, and ideally whitelist allowed directives or use a separate templating engine (Twig / Mustache) for user-facing templates.
- **Inline component views via `<<<'blade'` heredoc** — added full section. Class-based components can return the Blade template directly from `render()` instead of pointing at a separate `.blade.php` file. Best for small components; >20 lines should still use external view files for cache + IDE-highlight reasons.

## Updated from Research (2026-06-28, cycle 6)

- **`@once` / `@pushOnce` 13.x fix** — identity now persists across nested `@include` and `<x-dynamic-component>` (previously could re-render the block when tracking was reset by an intervening directive). Critical for shared admin partials.
- **Anonymous component props** — define props inline in the component view with `@props([...])`; auto-discovered from `resources/views/components/`. Skips the class file for one-off presentational components.
- **New Common Mistake #6** — `new HtmlString($userInput)` is the class-based equivalent of `{!! !!}` and skips escaping. Same root cause as CVE-2026-33080 in Filament. Always `e($value)` before wrapping.

## Updated from Research (2026-05)

- **@fonts Directive (Laravel 13.7+)** — `@fonts` generates `<link rel="preload">` tags for Vite-managed fonts, improving Core Web Vitals.
- **@props Directive** — pass all public constructor properties to a component view inline without `->with()`.

Source: [Laravel 13 Blade](https://laravel.com/docs/13.x/blade) | [Laravel 13 Components](https://laravel.com/docs/13.x/blade#components)
