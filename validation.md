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

## Common Mistakes

1. **`required` vs `filled`** — `required` fails on `""`, use `filled` when `0` or `false` are valid
2. **`unique` without ignoring self** — `unique:users,email,{$user->id}` for updates
3. **Not using Form Request** — keeps controller clean, easy to reuse
4. **No custom error messages** — always provide user-friendly messages in `messages()`
5. **Validating arrays incorrectly** — `tags.*` validates each element

## Updated from Research (2026-05)
- Laravel 13 enhances `required_if` behavior and adds `prohibits` rule
- `Rule::when()` method allows conditional rule application

Source: [Laravel Validation](https://laravel.com/docs/13.x/validation)
