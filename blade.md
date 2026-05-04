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

## View Composers (run on every render of a view)

```php
// AppServiceProvider boot()
View::composer('layouts.app', function (View $view) {
    $view->with('notifications', auth()->user()?->unreadNotifications);
});

// Wildcard
View::composer('partials.*', fn(View $view) => $view->with('theme', 'dark'));
```

## Common Mistakes

1. **`{!! !!}` with user-submitted content** — XSS vulnerability
2. **Using `{!!` for debug output** — debug tools that use `{!!` should only be for trusted internal data
3. **Not using `@csrf` in forms** — CSRF token missing, request rejected
4. **Forgetting `@method` for PUT/PATCH/DELETE** — wrong HTTP method, 419 error
5. **Old `{{$var}}` (no space)** — deprecated in Laravel 10, use `{{ $var }}`


## Updated from Research (2026-05)

- **r/laravel on Reddit: What are your honest thoughts about Livewire vs Inertia.js** (https://www.reddit.com/r/laravel/comments/s9za3a/what_are_your_honest_thoughts_about_livewire_vs/)
  I love Livewire and the author is a brilliant man. But if you want job portability, Inertia seems to be the way to go because you will use Vue along with it. If eventually you move to a different company where they don&#x27;t use the same stack (Laravel), so all your Livewire skills are going to waste.

- **Livewire vs Inertia: Which one Should I Choose for Laravel Development?** (https://www.scalablepath.com/php/livewire-vs-inertia)
  <strong>If you have a profile more oriented towards back-end development, you will definitely feel more comfortable using Livewire</strong>. However, if your team includes people who are already familiar with front-end frameworks like React, Vue, or Angular,

- **Inertia.js and Livewire: a high level comparison | Information Overload** (https://sebastiandedeyne.com/inertia-js-and-livewire-a-high-level-comparison/)
  Inertia replaces Blade views altogether by returning JavaScript components from controller actions. Those components can be built with your frontend framework of choice. <strong>Livewire is a Laravel library, while Inertia has adapters for several server and client frameworks</strong>.
