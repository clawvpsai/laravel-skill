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
    'r2' => [ // Cloudflare R2
        'driver' => 's3',
        'endpoint' => env('R2_ENDPOINT'),
        'key' => env('R2_ACCESS_KEY_ID'),
        'secret' => env('R2_SECRET_ACCESS_KEY'),
        'bucket' => env('R2_BUCKET'),
        'region' => 'auto',
    ],
],
```

## Uploading Files

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

public function store(Request $request)
{
    // Single file
    $path = $request->file('avatar')->store('avatars', 'public');

    // With custom filename (UUID-based)
    $filename = Str::uuid() . '.' . $request->file('avatar')->getClientOriginalExtension();
    $path = $request->file('avatar')->storeAs('avatars', $filename, 'public');

    // Multiple files
    $paths = collect($request->file('images'))->map(fn($f) => $f->store('images', 's3'));

    // Store from base64
    $data = base64_decode($request->input('image_data'));
    $path = Storage::disk('public')->put('avatars/' . $filename, $data);
}
```

## Direct S3 Upload (Presigned URL — No Server Involvement)

Best for large files — user uploads directly to S3/R2, server only gets the final URL.

```php
// Controller — generate presigned upload URL
use Illuminate\Support\Facades\Storage;

public function uploadUrl(Request $request): JsonResponse
{
    $request->validate(['filename' => 'required|string', 'content_type' => 'required|string']);

    $key = 'uploads/' . Str::uuid() . '/' . $request->filename;
    $url = Storage::disk('s3')->temporaryUploadUrl($key, now()->addMinutes(15), [
        'Content-Type' => $request->content_type,
    ]);

    return response()->json([
        'upload_url' => $url,
        'public_url' => Storage::disk('s3')->url($key),
        'key' => $key,
    ]);
}

// Frontend (JavaScript) — PUT directly to S3 presigned URL
fetch(presignedUploadUrl, {
    method: 'PUT',
    headers: { 'Content-Type': file.type },
    body: file,
}).then(() => saveToDatabase(publicUrl));
```

## Streaming Uploads (Large Files)

For files too large to hold in memory:

```php
// Stream upload directly to storage (constant memory regardless of file size)
public function streamUpload(Request $request)
{
    $file = $request->file('video');
    $path = 'videos/' . Str::uuid() . '.' . $file->getClientOriginalExtension();

    // Stream file contents to storage (doesn't load entire file into memory)
    Storage::disk('s3')->put(
        $path,
        fopen($file->getRealPath(), 'r'),
        ['StorageClass' => 'STANDARD_IA']
    );

    return response()->json(['path' => $path]);
}

// Or using the file's temporary stream
$handle = $request->file('large_file')->openFile();
Storage::disk('local')->put($path, $handle);
```

## Chunked Uploads (Resumable — tus protocol)

For very large files or unreliable connections:

```php
// Install: composer require aliuly/laravel-tus
// Uses tus.io resumable upload protocol

// config/tus.php
return [
    'upload_dir' => storage_path('app/tus-uploads'),
    'uri' => '/api/upload',
];

// Route
Route::match(['get', 'post'], '/api/upload', [UploadController::class, 'handle'])
    ->middleware('throttle:10,1');
```

## Retrieving Files

```php
// URL (public files)
$url = Storage::disk('public')->url($path);

// Download
return Storage::disk('s3')->download($path, $name, ['Content-Type' => 'application/pdf']);

// Stream response (doesn't load into memory)
return Storage::response('public/' . $path);

// Temporary URL (private files)
$url = Storage::disk('s3')->temporaryUrl($path, now()->addMinutes(5));

// R2/Cloudflare: custom URL format
$url = Storage::disk('r2')->temporaryUrl($path, now()->addMinutes(60), [
    'ResponseContentDisposition' => 'attachment; filename="document.pdf"',
]);
```

## File Deletion

```php
// Single file
Storage::disk('public')->delete($path);

// Multiple files
Storage::delete(['avatars/1.jpg', 'avatars/2.jpg']);

// All files in a directory
Storage::disk('public')->delete(Storage::disk('public')->files('avatars'));

// On model delete (Observer)
class PostObserver
{
    public function deleted(Post $post)
    {
        foreach ($post->attachments as $attachment) {
            Storage::disk('s3')->delete($attachment->path);
        }
    }
}
```

## Image Validation

```php
$request->validate([
    'photo' => [
        'required',
        'file',
        'image',
        'mimes:jpeg,png,gif,webp',
        'max:2048', // KB
        'dimensions:min_width=100,min_height=100,max_width=4096,max_height=4096',
    ],
    'document' => 'required|file|mimes:pdf,doc,docx|max:10240', // 10MB
]);
```

## File Upload Security Checklist

1. **Never trust the filename** — `$file->getClientOriginalName()` is user-controlled
2. **Generate new filename** — use `Str::uuid()` or hash the original
   ```php
   $name = Str::uuid() . '.' . $file->getClientOriginalExtension();
   ```
3. **Don't store in web-accessible path without hashing** — or disable direct access
4. **Validate MIME type server-side** — don't trust `mimes` alone
   ```php
   $mime = $file->getMimeType(); // application/pdf, image/png
   $allowed = ['image/jpeg', 'image/png', 'image/webp'];
   if (!in_array($mime, $allowed)) abort(422);
   ```
5. **Set disk to non-public for sensitive files** — private S3 buckets with signed URLs
6. **Scan uploaded files for malware** — use `clamav` for user-uploaded content
7. **Limit file size** — `max:2048` (KB) and `upload_max_filesize` in php.ini
8. **Disable `exec()` in PHP** — prevents PHP upload exploits
9. **Use presigned URLs for large uploads** — don't proxy through your server
10. **Store files outside webroot** — only serve via Storage URLs or signed URLs

## Storage Disk Selection Guide

| Use Case | Disk |
|---|---|
| User avatars, public images | `public` + CloudFront/CDN |
| Large uploads, videos | `s3` with `STANDARD_IA` |
| Sensitive documents | `s3` private + presigned URL |
| Backups, exports | `s3` with lifecycle policy |
| Local dev only | `local` |

## Common Mistakes

1. **Using original filename** — XSS vector if served as-is; use `$file->hashName()` or UUID
2. **`storage/app/public` needs `php artisan storage:link`** — creates symlink in `public/`
3. **S3 bucket publicly accessible** — should be private with presigned URLs
4. **Large file uploads timing out** — set `client_max_body_size` in nginx, `max_execution_time` in php.ini
5. **Not cleaning up on model delete** — use model events or `FileCleanupJob`
6. **Serving private files via public URL** — use `temporaryUrl()` or signed URLs
7. **Loading large files into memory** — use `fopen()` streaming instead of `file_get_contents()`
8. **No disk space monitoring** — uploads fail silently when disk is full

## Updated from Research (2026-05)
- Laravel 13 file validation supports `dimensions:min_width=,min_height=` rules
- S3 presigned URLs are the recommended way to serve private files
- Direct-to-S3 upload pattern eliminates server bottleneck for large files
- Cloudflare R2 works with S3-compatible API (same driver, different endpoint)

Sources: [Laravel Filesystem](https://laravel.com/docs/13.x/filesystem) | [Laravel Storage Presigned URLs](https://laravel.com/docs/13.x/filesystem#temporary-urls)
