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
## `@class` and `@style` Directives (Laravel 9.18+)

These directives build a `class="..."` or `style="..."` attribute from a PHP array, applying a class/style only when its key evaluates to a truthy value. AI assistants still default to the verbose `class="{{ $isActive ? 'active' : '' }}"` ternary pattern, which is XSS-prone when `$isActive` is a user-controlled string and the class list comes from a config:

```blade
{{-- @class — emit each class only when the value is truthy --}}
<a href="{{ $post->url }}"
   @class([
       'nav-link',
       'nav-link--active' => $post->isActive(),
       'nav-link--featured' => $post->isFeatured(),
       'text-muted' => ! $post->published,
   ])>
    {{ $post->title }}
</a>

{{-- @style — same shape, for inline style strings --}}
<div @style([
       'background-color: ' . $user->color,
       'font-weight: bold' => $user->isAdmin(),
       'display: none' => ! $user->shouldShow(),
   ])>
    ...
</div>
```

**Why this matters:**

- **XSS safety:** `@class` / `@style` concatenate literal array keys (the CSS class names you wrote) — not the user-supplied values. The values only control *whether* a class is emitted, not *which* class. The verbose `{{ $isActive ? 'active' : '' }}` pattern will silently inject HTML content if `$isActive` is ever a non-string. Use `@class` for any class list that depends on more than one condition.
- **Tailwind / utility frameworks:** `@class(['bg-red-500' => $hasError])` collapses the `class="bg-red-500 {{ $hasError ? '' : 'hidden' }}"` mess into one place — easier to grep, easier to lint.
- **Pairing with `@checked` / `@selected`:** when you have a list of options with mixed dynamic class lists and checked/selected states, the directives compose cleanly — no inline concatenation, no string interpolation footguns.

**Common gotcha:** passing a string value as the *value* of an array entry (e.g. `@class(['btn-' . $variant => true])`) works but is a code smell — pre-compute the class list in PHP or use `@class(['btn', 'btn-' . $variant => true])` so the base classes always render. The directive never emits `class="false"` or `class="0"` because PHP coerces array values to boolean before checking.

## `<x-slot:foo>` Short-Form Slot Syntax (Laravel 9+)

Laravel 9 added a compact alternative to `<x-slot name="foo">...</x-slot>` — use `<x-slot:foo>...</x-slot>` instead. Verbosity drops, diff readability goes up:

```blade
{{-- Long form --}}
<x-card>
    <x-slot name="header">Post title</x-slot>
    <x-slot name="footer">
        <a href="/back">Back</a>
    </x-slot>
    Body content
</x-card>

{{-- Short form (Laravel 9+) — same render, no `name=` --}}
<x-card>
    <x-slot:header>Post title</x-slot:header>
    <x-slot:footer>
        <a href="/back">Back</a>
    </x-slot:footer>
    Body content
</x-card>
```

**Why this matters:** the long form's `<x-slot name="header">` requires the `name=` attribute as a separate string, which means IDE refactor tools won't always rename it consistently and the slot name is easy to typo. The short form treats the slot name as part of the element identifier, so IDEs (PhpStorm, VSCode + intelephense) and `grep` treat it as one token. For components with 3+ slots, this is a real readability win.

**Gotcha:** the short form is a Blade compile-time shortcut — it expands to the same `<x-slot name="...">` syntax internally. If you're rendering a component dynamically with `<x-dynamic-component>` and a config-driven slot name, the long form is the only option (you can't write `<x-slot:{name}>`). For static slot names, always prefer the short form.

## `@aware` Directive — Parent Data Without Explicit Props (Laravel 9+)

Child components can pull data from their parent component without having the parent pass it via props. This is the "context" / "implicit prop" pattern — used by component libraries like `laravel/breeze` for theming and menu nesting:

```blade
{{-- resources/views/components/menu.blade.php --}}
<div {{ $attributes->class(['menu']) }}>
    <h4>{{ $title }}</h4>
    {{ $slot }}
</div>

{{-- resources/views/components/menu/item.blade.php --}}
@aware(['title'])  {{-- pulls $title from the parent <x-menu> --}}
<a {{ $attributes->class(['menu-item']) }}>{{ $title }}: {{ $slot }}</a>

{{-- Usage — no explicit title pass to <x-menu.item> --}}
<x-menu title="Settings">
    <x-menu.item href="/profile">Profile</x-menu.item>
    <x-menu.item href="/logout">Logout</x-menu.item>
</x-menu>
```

**Why this matters:** the alternative is explicit prop threading — `<x-menu.item :title="$title" :href="..." />` for every item in every menu — which clutters the call site. `@aware` lets the parent context propagate to children automatically, mirroring how Vue's `provide/inject` or React's `Context` work.

**Gotchas:**

- **The parent must define the prop explicitly** — `@aware(['title'])` on a child requires the parent component to have `$title` in its constructor props or `@props(['title'])`. If the parent doesn't have it, the child sees `null` and silently renders "Settings: Profile" → "null: Logout" (worse than crashing — easy to miss in dev). Always verify the prop is defined on the parent.
- **No type safety** — `@aware` is compile-time, not runtime. There's no validation that the parent actually provides the right type. If the parent passes a string and the child expects an array, you get a TypeError at render. Use PHPStan/Psalm generics to catch this in CI.
- **Anti-pattern for deep nesting** — `@aware` is great for one level (parent menu → menu item). Three levels deep, the data flow becomes invisible and you can't tell from a single file what props it actually has. At that point, prefer explicit prop passing or move to a Slot-based composition.

## `<x-dynamic-component>` for Runtime-Resolved Component Names

When the component name comes from a config, DB column, plugin system, or A/B test, you can't use the static `<x-name />` syntax. Use `<x-dynamic-component>`:

```blade
{{-- Component name from a database column (CMS widget system) --}}
@php $widget = $page->widget; @endphp
<x-dynamic-component
    :component="$widget->view"
    :data="$widget->data"
/>

{{-- A/B-tested component (50/50 split) --}}
<x-dynamic-component
    :component="cache()->remember('hero-variant-' . $user->id, 3600, fn() => collect(['hero-a', 'hero-b'])->random())"
/>

{{-- Plugin system — components discovered from vendor/ packages --}}
@foreach($plugins as $plugin)
    <x-dynamic-component
        :component="$plugin->componentClass"
        :config="$plugin->config"
    />
@endforeach
```

**Why this matters:** AI assistants default to `@include($component->name, [...])` for this pattern, which works but bypasses the entire class-based component system — no `withSlot`, no `withAttributes`, no constructor injection, no `render()` method. `<x-dynamic-component>` preserves all the component-system benefits while accepting a variable name.

**Gotchas:**

- **No compile-time validation** — if `$widget->view` is `'nonexistent'`, you get a `ViewNotFoundException` at render time, not a `make:component` validation. For CMS widget systems, always validate the view exists first: `View::exists($widget->view)` (see next section).
- **Class-based vs anonymous components** — `<x-dynamic-component>` works with both, but anonymous components are looked up first in `resources/views/components/{name}.blade.php`, then class-based components in `App\View\Components\{Name}`. For multi-tenant plugin systems, prefer class-based components explicitly (`:component="\App\View\Components\Plugins\Chart::class"`) to avoid path-resolution confusion.
- **`@once` / `@pushOnce` identity** — covered above; the 13.x fix specifically affects dynamic components. Stacks track identity across `<x-dynamic-component>` includes correctly as of 13.16+.

## View Cache: `view:cache` / `view:clear` and the Production Build Pipeline

Blade compiles every `.blade.php` to plain PHP on first render and caches the compiled file to `storage/framework/views/`. In production, this first-render compilation adds 50–300ms per uncached view — a real problem when a deploy hits a cold worker pool or the first request after `view:clear` happens on the production load balancer.

**Build pipeline (production CI, post-`composer install`):**

```bash
# Pre-compile every .blade.php in the app
php artisan view:cache

# ALWAYS run this AFTER `composer install` and BEFORE the first request hits the worker pool
# Idempotent — safe to re-run on every deploy
```

**When to clear:**

```bash
# Local dev — Blade auto-recompiles on file change, but cached views can get out of sync
# after a Blade-compiler-bug fix or after `git stash`/`git pull` mtime changes
php artisan view:clear

# Production — never run on its own, only as part of a view-bug fix:
# 1. view:clear
# 2. deploy new code that changes the Blade templates
# 3. view:cache (to warm the new compiled views)
```

**Gotchas:**

- **`view:cache` is NOT automatic on deploy** — Laravel doesn't run it during `php artisan optimize` (which only handles `config:cache`, `route:cache`, `event:cache`). You must add it to your CI/CD deploy script explicitly. AI-generated deploy scripts (`github actions`, `forge`, `envoy`) miss this 90% of the time. Symptom: first request to a new view after deploy throws `ErrorException: include(): Filename cannot be empty` or similar — because the compiled view path resolved against the old `storage/framework/views/` file that no longer matches the new template.
- **Octane + view cache: stale compiled views across requests** — under Swoole/RoadRunner/FrankenPHP, the compiled view class is kept in memory between requests. After a deploy that changes a `.blade.php`, in-flight Octane workers still have the old compiled view in their process memory until they're recycled. Two fixes: (1) `php artisan octane:reload` after every deploy that touches `resources/views/`, (2) rely on the `octane.warm` config list to pre-load the most-used views at boot.
- **Cache file ownership** — `view:cache` writes files as the PHP-FPM / Octane worker user. After switching deploy users (e.g. from `forge` to `www-data`), `view:clear` may fail with permission errors. Run `chown -R $WEB_USER:$WEB_GROUP storage/framework/views` after user changes, or `view:clear && view:cache` as the new user.
- **`View::exists()`** for runtime safety — before rendering a dynamic view, check it exists: `if (View::exists($partial)) { return view($partial, $data); }`. Throws `ViewNotFoundException` otherwise. Critical for tenant-themable systems, plugin architectures, and CMS widget rendering where the view path comes from a config / DB.

## Common Mistakes

1. **`{!! !!}` with user-submitted content** — XSS vulnerability
2. **Using `{!!` for debug output** — debug tools that use `{!!` should only be for trusted internal data
3. **Not using `@csrf` in forms** — CSRF token missing, request rejected
4. **Forgetting `@method` for PUT/PATCH/DELETE** — wrong HTTP method, 419 error
5. **Old `{{$var}}` (no space)** — deprecated in Laravel 10, use `{{ $var }}`
6. **`new HtmlString($userInput)`** — `HtmlString` is the *class-based* equivalent of `{!! !!}` and bypasses escaping. CVE-2026-33080 (Filament) was a real-world instance of this — always `e($value)` before wrapping. Use `{{ }}` / `e()` first, then wrap in `HtmlString` only for known-safe HTML
7. **Class list with `{{ $isActive ? 'active' : '' }}` ternary** — XSS-prone if `$isActive` is user-controlled (template injection), and harder to read than `@class([...])`. Use `@class` for any class list with 2+ conditions
8. **`<x-slot name="header">` instead of `<x-slot:header>`** — the long form is harder to grep, harder to IDE-refactor, and 2x the keystrokes. Use the short form for static slot names
9. **`@aware` for 3+ levels of nesting** — works but invisible data flow. Prefer explicit props or slot composition past one level
10. **`<x-dynamic-component :component="$userInput">` without `View::exists()` check** — throws `ViewNotFoundException` at render. Always validate dynamic component names (and consider class-based components for plugin systems)
11. **Not running `php artisan view:cache` in production deploys** — first request after deploy compiles every Blade view on the spot (50–300ms per view). Add `view:cache` to your deploy script AFTER `composer install` and BEFORE the worker pool accepts traffic. Symptom: deploy succeeds, first few requests get `ErrorException: include(): Filename cannot be empty` until each view is compiled
12. **Octane + stale view cache after deploy** — Octane keeps compiled views in process memory across requests. After a `.blade.php` change, run `php artisan octane:reload` (or rely on `octane.warm` config). Without it, in-flight workers render the old template even after `view:clear`
13. **Unquoted attribute interpolation `data-foo={{ $value }}` (attribute injection)** — `{{ }}` escapes HTML entity characters but the surrounding attribute context is still subject to injection if the attribute isn't quoted. Always quote your attributes: `<div data-foo="{{ $value }}">` not `<div data-foo={{ $value }}>`. The unquoted form lets attackers break out with a space character
14. **N+1 in `@foreach` over a relationship without eager loading** — `<li>{{ $post->user->name }}</li>` triggers a query per iteration. Same fix as anywhere else: `Post::with('user')->get()`. Blade doesn't change the underlying query behavior; eager loading is the caller's job

## Updated from Research (2026-07-14, cycle 39)

- **`@class` and `@style` directives (Laravel 9.18+)** — added full section. The most-missed Blade directive for CSS class lists. Collapses `class="{{ $isActive ? 'active' : '' }}"` ternaries into a single array-based declaration, and is XSS-safe (only array keys become classes, not the values). Tailwind-friendly.
- **`<x-slot:foo>` short-form slot syntax (Laravel 9+)** — added full section. Drop the `name=` attribute for static slot names — IDEs and `grep` treat `<x-slot:header>` as a single token, which fixes a real refactor-rename problem. Long form only needed when the slot name is dynamic.
- **`@aware` directive (Laravel 9+)** — added full section. Pulls parent component data into a child without explicit prop threading — mirrors Vue's `provide/inject` and React's `Context`. Great for menu nesting and theme propagation, but watch: (1) the parent MUST have the prop defined or the child silently gets `null`, (2) no type safety, (3) past one level of nesting prefer explicit props for visibility.
- **`<x-dynamic-component>` for runtime-resolved component names** — added full section. The right answer for CMS widget systems, A/B-tested components, and plugin architectures. AI models default to `@include($component->name, [...])` for this pattern, which bypasses the entire component system. Always validate the view exists with `View::exists()` first.
- **View cache: `view:cache` / `view:clear` + Octane staleness** — added full section. `view:cache` is NOT part of `php artisan optimize` — you must add it to your CI/CD deploy script. Octane workers keep compiled views in process memory; after a deploy that touches `resources/views/`, run `octane:reload` (or rely on `octane.warm`). Most AI-generated deploy scripts miss both.
- **Common Mistakes list grew 6 → 14 entries** — added `@class` vs ternary, `<x-slot>` short form, `@aware` 3-level anti-pattern, `dynamic-component` without `View::exists`, missing `view:cache` in deploy, Octane stale view cache, unquoted attribute injection, and N+1 in `@foreach`.

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
