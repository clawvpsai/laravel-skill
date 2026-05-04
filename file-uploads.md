# File Uploads & Storage

## File Storage (Local + S3 + Cloud)

**Config — `config/filesystems.php`:**
```php
'disks' => [
    'local' => [
        'driver' => 'local',
        'root' => storage_path('app'),
        'throw' => false,
    ],
    'public' => [
        'driver' => 'local',
        'root' => storage_path('app/public'),
        'url' => env('APP_URL').'/storage',
        'visibility' => 'public',
    ],
    's3' => [
        'driver' => 's3',
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'region' => env('AWS_DEFAULT_REGION'),
        'bucket' => env('AWS_BUCKET'),
        'url' => env('AWS_URL'),
        'throw' => false,
    ],
],
```

## Uploading Files

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Storage;

public function store(Request $request)
{
    // Single file
    $path = $request->file('avatar')->store('avatars', 'public'); // returns path
    // or with custom filename
    $path = $request->file('avatar')->storeAs('avatars', $user->id.'.jpg', 'public');

    // Multiple files
    $paths = $request->file('images')->map(fn($f) => $f->store('images', 's3'));

    // Store from base64
    $data = base64_decode($request->input('image_data'));
    $path = Storage::disk('public')->put('avatars/'.$filename, $data);
}
```

## Retrieving Files

```php
// URL (public files)
$url = Storage::disk('public')->url($path);

// Download
return Storage::disk('s3')->download($path, $name, $headers);

// Stream response
return Storage::response('public/'.$path);
```

## File Deletion

```php
Storage::disk('public')->delete($path);
// or multiple
Storage::delete($paths);
```

## Temporary URLs (S3 Presigned)

```php
$url = Storage::disk('s3')->temporaryUrl($path, now()->addMinutes(5));
```

## Image Validation

```php
$request->validate([
    'photo' => 'required|image|mimes:jpeg,png,gif,webp|max:2048|dimensions:min_width=100,min_height=100',
]);
```

## File Upload Security Checklist

1. **Never trust the filename** — `$file->getClientOriginalName()` is user-controlled
2. **Generate new filename** — use `Str::uuid()` or hash the original
   ```php
   $name = Str::uuid().'.'.$file->getClientOriginalExtension();
   ```
3. **Don't store in web-accessible path without hashing** — or disable direct access
4. **Validate MIME type server-side** — don't trust `mimes` alone, check with File facade
   ```php
   $mime = $file->getMimeType(); // application/pdf, image/png, etc.
   ```
5. **Set disk to non-public for sensitive files** — private S3 buckets, signed URLs
6. **Scan uploaded files for malware** — use `clamav` for user-uploaded content
7. **Limit file size** — `max:2048` (KB) or configure `upload_max_filesize` in php.ini
8. **Disable `exec()` in PHP** — prevents PHP upload exploits

## Common Mistakes

1. **Using original filename** — XSS vector if served as-is; use `$file->hashName()`
2. **`storage/app/public` needs `php artisan storage:link`** — creates symlink in `public/`
3. **S3 bucket publicly accessible** — should be private with presigned URLs
4. **Large file uploads timing out** — set `client_max_body_size` in nginx, `max_execution_time` in php.ini
5. **Not cleaning up on model delete** — use model events or `FileCleanupJob`

## Updated from Research (2026-05)
- Laravel 13 file validation supports `dimensions:min_width=,min_height=` rules
- S3 presigned URLs are the recommended way to serve private files

Source: [Laravel Filesystem](https://laravel.com/docs/13.x/filesystem)
