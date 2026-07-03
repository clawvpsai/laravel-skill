# Validation — Rules, Form Requests, Custom Validators

## Validation Rules

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rules\Password;

$validator = Validator::make($data, [
    'name'     => 'required|string|max:255',
    'email'    => 'required|email:rfc,dns|unique:users,email',
    'password' => 'required|min:8|confirmed',
    'age'      => 'nullable|integer|min:18|max:150',
    'url'      => 'nullable|url',
    'phone'    => 'nullable|regex:/^[0-9+\-\s]+$/',
    'avatar'   => 'nullable|image|mimes:jpg,png,gif,webp|max:2048',
    'posts'    => 'nullable|array',
    'posts.*'  => 'integer|exists:posts,id', // each post must exist
    'role'     => 'in:admin,editor,author',  // whitelist
]);
```

## Form Request (Recommended)

```bash
php artisan make:request StorePostRequest
```

```php
class StorePostRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'title'   => 'required|string|max:255',
            'body'    => 'required|string|min:20',
            'tags'    => 'array',
            'tags.*'  => 'string|max:50',
            'status'  => 'in:draft,published,archived',
        ];
    }

    public function messages(): array
    {
        return [
            'title.required' => 'A title is required',
            'body.min'       => 'Posts must be at least 20 characters',
        ];
    }

    public function prepareForValidation(): void
    {
        $this->merge(['slug' => Str::slug($this->title)]);
    }

    public function withValidator(Validator $validator): void
    {
        $validator->after(function ($v) {
            if ($this->status === 'published' && !$this->body) {
                $v->errors()->add('published', 'Cannot publish empty post');
            }
        });
    }

    public function authorize(): bool
    {
        return $this->user()->can('create posts');
    }
}
```

**Use in controller:**
```php
public function store(StorePostRequest $request)
{
    // $request->validated() returns clean data
    Post::create($request->validated());
    return redirect()->route('posts.index');
}
```

## Custom Validation Rules

```php
// Rule
php artisan make:rule Uppercase

class Uppercase implements Rule
{
    public function passes($attribute, $value): bool
    {
        return strtoupper($value) === $value;
    }

    public function message(): string
    {
        return 'The :attribute must be uppercase.';
    }
}

// Use
'name' => ['required', new Uppercase()],
```

## Laravel 13 Validation Attributes

```php
// Auto-map validation attribute names
public function attributes(): array
{
    return [
        'card_number' => 'credit card number',
        'cvv' => 'security code',
    ];
}
```

## Rule Reference

| Rule | Description |
|---|---|
| `required` | Must be present (fails on empty string `""`) |
| `filled` | Must be present AND non-empty |
| `present` | Must be in request (even null/empty) |
| `nullable` | Can be null, skip other rules if null |
| `email:rfc,dns` | RFC validation + DNS check |
| `unique:table,column` | Check uniqueness in DB |
| `exists:table,column` | Check existence in DB |
| `in:a,b,c` | Must be one of these values |
| `not_in:a,b,c` | Must NOT be one of these |
| `image` | File must be image (jpg, png, gif, webp) |
| `mimes:jpg,png` | MIME type check |
| `dimensions:min_width=100` | Image dimension check |
| `regex:/pattern/` | Custom regex |
| `required_if:field,value` | Required if condition met |
| `required_unless:field,value` | Required unless condition met |
| `same:field` | Must match another field |
| `different:field` | Must differ from another field |
| `before:date` | Must be before this date |
| `after_or_equal:today` | Must be today or after |

## `#[FailOnUnknownFields]` Attribute (Laravel 13+)

Reject requests that include fields NOT declared in your `rules()` array. Useful defense-in-depth against mass-assignment probing and accidental field leakage:

```php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Foundation\Http\Attributes\FailOnUnknownFields;

#[FailOnUnknownFields]
class UpdatePostRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'body'  => ['required', 'string'],
            // anything else in the request payload → 422 with "field is not allowed"
        ];
    }
}
```

**Behaviour:**
- Without the attribute → unknown fields are silently ignored (current default).
- With `#[FailOnUnknownFields]` (no args) → fail on any unknown field.
- `#[FailOnUnknownFields(allow: ['_token', '_method'])]` → whitelist specific unknown fields (commonly needed for CSRF token + form method spoofing).

**When to use:**
- Public APIs that accept JSON payloads (no HTML form constraints).
- Admin endpoints where users might submit extra fields and get confused why "nothing happened".
- Defense-in-depth on top of `$fillable` — `$fillable` protects the model; `#[FailOnUnknownFields]` fails fast at validation so you don't even reach the model.

## `Password::toPasswordRulesString()` (Laravel 13.9+)

Convert any `Password` rule instance into the JS hint string consumed by the `passwordrules` HTML attribute, used by browsers' built-in password managers and the `zxcvbn` style validators:

```php
use Illuminate\Validation\Rules\Password;

$rules = Password::min(12)
    ->max(64)
    ->mixedCase()
    ->numbers()
    ->symbols()
    ->toPasswordRulesString();
// 'minlength: 12; maxlength: 64; required: lower; required: upper; required: digit; required: special;'
```

**Pair with the `passwordrules` attribute in your Blade form** (the new Laravel 13 starter kits do this automatically via `Password::defaults()`):
```blade
<input type="password" name="password" passwordrules="{{ Password::defaults()->toPasswordRulesString() }}">
```

**Why this matters:** Before 13.9, you had to hand-write the `passwordrules` string — easy to drift from your server-side `Password::` rule. Now you derive both from the same source. If you change `Password::min(12)` to `Password::min(14)`, the HTML hint updates automatically.

## Security: `date_equals` Validation Bypass Fix (Laravel 13.15+)

**Critical security fix** — the `date_equals` rule used loose comparison (`==`) which allowed invalid date strings to bypass validation against falsy reference dates:

```php
// BEFORE 13.15 — exploitable:
// $request->date_equals = "not-a-date";  // parses to null
// null == strtotime("1970-01-01")  // 0 → loose-equals true!
$validator->validate([
    'expires_at' => 'date_equals:2024-01-01',
]);
// An invalid date string could pass because `null == 0`

// AFTER 13.15 — strict equality for the equal operator:
// null === 0 → false → validation correctly fails
```

**Who is affected:** Any Laravel app exposing `date_equals` to user input (form submissions, API requests, scheduled-task inputs). **Upgrade to Laravel 13.15+** to mitigate.

**Loose comparison preserved for `DateTime` objects** — only the strict-vs-loose change applies to the equal operator. `DateTimeInterface` instances still compare correctly.

Source: [Laravel News — date_equals fix](https://laravel-news.com/laravel-13-15-0) | [PR #60393](https://github.com/laravel/framework/pull/60393)

## Prohibited Validation Rule Family (Laravel 13)

The "prohibited" family is the inverse of `required` — it forces a field to be **missing or empty**. Use it when you want to *forbid* a value rather than require one (either-or fields, "don't submit this in this state", "exactly one of these"):

```php
public function rules(): array
{
    return [
        // Field MUST be missing or empty (no null, no string, no array, no file).
        'legacy_id'         => 'prohibited',

        // Field MUST be missing/empty WHEN another field equals a value.
        // Pairs with `Rule::prohibitedIf(Closure)` for arbitrary boolean logic.
        'coupon_code'       => 'prohibited_if:payment_method,stripe',

        // Field MUST be missing/empty UNLESS another field equals a value.
        'gift_message'      => 'prohibited_unless:order_type,gift',

        // Field MUST be missing/empty WHEN another field is "accepted" (true/yes/1).
        'refund_reason'     => 'prohibited_if_accepted:is_refundable',

        // Field MUST be missing/empty WHEN another field is "declined" (false/no/0).
        'followup_required' => 'prohibited_if_declined:needs_followup',

        // REVERSE direction: if THIS field is present, the OTHER fields MUST be empty.
        // Classic "either primary_email OR backup_email, not both" pattern.
        'primary_email'     => 'prohibits:backup_email',
        'backup_email'      => 'prohibits:primary_email',
    ];
}
```

**"Empty" definition (per Laravel 13 docs) — applies uniformly across the prohibited family:**
A field is "empty" if it meets any of:
- The value is `null`
- The value is an empty string `""`
- The value is an empty array or empty `Countable` object
- The value is an uploaded file with an empty path

**Closure form for arbitrary boolean logic:**
```php
'use_real_money' => [
    Rule::prohibitedIf(fn () => $this->user()->isMinor() || $this->input('sandbox_mode')),
],
```

**Common use cases:**
- "Either coupon_code OR payment_method, never both" → `prohibits:payment_method` on `coupon_code` and vice versa
- "If is_refundable=true, don't submit a refund_reason" → `prohibited_if_accepted:is_refundable`
- "If order_type=gift, message is required; if not gift, no gift_message allowed" → `prohibited_unless:order_type,gift` + `required_unless:order_type,gift`
- "Block admin-only fields from non-admins" → `prohibits` chained with `Rule::when()` (see below)

## `Rule::when()` / `Rule::unless()` — Conditional Rule Builders

For form requests where a rule applies only under some condition (admin role, feature flag, partial draft), branching in `rules()` works but gets messy. `Rule::when()` / `Rule::unless()` collapses it into a single expression:

```php
use Illuminate\Validation\Rule;

public function rules(): array
{
    return [
        'internal_notes' => Rule::when(
            $this->user()->can('view-internal-notes'),
            ['required', 'string', 'max:1000'],
            ['nullable'],
        ),

        // Inverse: only apply when the condition is FALSE
        'beta_flag' => Rule::unless(
            $this->user()->hasFeature('beta-program'),
            ['prohibited'],
            ['nullable', 'boolean'],
        ),
    ];
}
```

**Why prefer `Rule::when()` over conditional arrays:**
- Keeps `rules()` as a flat, declarative array — easier to scan
- The branch and fallback are visible on one line
- Plays well with static analyzers and IDE autocompletion
- No risk of accidentally producing duplicate keys

**Pair with `Rule::prohibitedIf()` / `Rule::requiredIf()`** for the inverse direction. When you need to apply or skip multiple rules under a condition, use `Rule::when()`. When you need to invert a single field's required state based on a boolean, use `Rule::requiredIf()` / `Rule::prohibitedIf()`.

## Common Mistakes — `required` fails on `""`, use `filled` when `0` or `false` are valid
2. **`unique` without ignoring self** — `unique:users,email,{$user->id}` for updates
3. **Not using Form Request** — keeps controller clean, easy to reuse
4. **No custom error messages** — always provide user-friendly messages in `messages()`
5. **Validating arrays incorrectly** — `tags.*` validates each element
6. **`starts_with`/`ends_with` rejecting numeric values** — before Laravel 13.10, these rules rejected integer/float input. Now they accept them. If you have numeric validation logic that worked around this, simplify it (Laravel 13.10+).

## Updated from Research (2026-05-20)
- **starts_with/ends_with accept numeric values (Laravel 13.10+)** — fixed validation rule regression. Previously rejected integer/float values. Now accepts them alongside strings.
- **Line break injection prevention (Laravel 13.10+)** — email validation now rejects values containing line breaks, preventing email header injection attacks.
- Laravel 13 enhances `required_if` behavior and adds `prohibits` rule
- `Rule::when()` method allows conditional rule application

See full details below in **Prohibited Validation Rule Family (Laravel 13)** and **`Rule::when()` / `Rule::unless()`** sections — covering `prohibited`, `prohibited_if`, `prohibited_unless`, `prohibited_if_accepted`, `prohibited_if_declined`, and reverse-direction `prohibits` rules plus the conditional rule array builder.

Source: [Laravel Validation](https://laravel.com/docs/13.x/validation)


### `#[FailOnUnknownFields]` Form Request Attribute (Laravel 13+)

- `#[FailOnUnknownFields]` on a Form Request class fails validation when the payload contains fields NOT in `rules()`. Optional `allow:` argument whitelists specific unknown fields (`_token`, `_method`).
- Defense-in-depth against mass-assignment probing. Pairs with `$fillable` — `$fillable` protects the model; this fails fast at validation.

### `Password::toPasswordRulesString()` (Laravel 13.9+)

- Converts a `Password` rule instance to the JS hint string consumed by the `passwordrules` HTML attribute (browser password managers, zxcvbn-style validators).
- Single source of truth: change `Password::min(12)` → `Password::min(14)`, the HTML hint auto-updates.
- New Laravel 13 starter kits auto-render this attribute on new-password inputs via `Password::defaults()`.
