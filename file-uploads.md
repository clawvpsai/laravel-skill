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

    // Laravel 13.x: temporaryUploadUrl() returns ['url' => $url, 'headers' => $headers]
    // Older Laravel returned just the URL string — destructure the array for both pieces.
    ['url' => $url, 'headers' => $headers] = Storage::disk('s3')->temporaryUploadUrl(
        $key,
        now()->addMinutes(15),
        ['Content-Type' => $request->content_type]  // forward any signed headers you need
    );

    return response()->json([
        'upload_url' => $url,
        'upload_headers' => $headers,   // MUST be sent with the PUT — server validates them
        'public_url'  => Storage::disk('s3')->url($key),
        'key'         => $key,
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

## File Visibility

Visibility is a per-file attribute that controls whether the file can be served publicly. On
cloud disks (S3/R2) it maps to the object's ACL/permission; on the `public` local disk it maps
to a `chmod 0644` vs `chmod 0600` on the stored file.

```php
// Set visibility when writing
Storage::disk('s3')->put('docs/report.pdf', $contents, 'public');   // bucket-public
Storage::disk('s3')->put('docs/report.pdf', $contents, 'private');  // bucket-private (default)

// Read visibility
$vis = Storage::disk('s3')->getVisibility('docs/report.pdf');  // 'public' | 'private'

// Flip existing object's visibility
Storage::disk('s3')->setVisibility('docs/report.pdf', 'public');

// Visibility-aware URLs:
//   public  → Storage::url() returns a permanent URL
//   private → Storage::url() returns a public CDN URL only if the disk is configured for it
//             otherwise use temporaryUrl() with an expiration
```

**Disk default visibility** — set in `config/filesystems.php` per disk with the `visibility` key
(overrides per-file visibility only if you don't pass one in `put()`).

**Common gotcha:** `setVisibility()` is a metadata-only operation on S3 (HEAD + COPY), so it does
**not** trigger a re-upload — but the file must already exist. If you store with `public` and later
flip to `private`, the **existing CDN/cached URL is still served** until you invalidate it (CloudFront
invalidation, Cloudflare purge). Plan the rotation, don't rely on the call being instant.

**Cloudflare R2** — visibility works the same way as S3 (it's the same API driver), but R2 is
private-by-default at the bucket level. You must enable public development URL or use a custom
domain. `Storage::disk('r2')->url()` only works if the bucket allows public access.

## Anti-Extension-Spoofing & Content Validation

**The classic attack:** `evil.php.jpg` is uploaded. Server checks `getClientOriginalExtension()`
and sees `jpg`, but the actual file is `evil.php` renamed. When served from a path Apache/Nginx
double-interprets (e.g., misconfigured `AddHandler`), it executes as PHP.

Laravel's defenses:

```php
// 1. NEVER trust getClientOriginalName() / getClientOriginalExtension() for security decisions
//    Use them only for display. For storage naming, use hashName() or Str::uuid().

// 2. Validate MIME by inspecting the actual file content, not the request header
$finfo = finfo_open(FILEINFO_MIME_TYPE);
$actualMime = finfo_file($finfo, $file->getRealPath());
finfo_close($finfo);

$allowed = ['image/jpeg', 'image/png', 'image/webp'];
abort_unless(in_array($actualMime, $allowed, true), 422, 'Invalid file content');

// 3. Laravel's 'mimes:jpg,png' rule checks the extension *and* the mime type via finfo
//    (since 9.x) — but only if 'image' or 'mimetypes' rule is also present, so the symfony
//    MIME guesser runs. Always pair: 'file|image|mimes:jpeg,png|max:2048'.

// 4. Disable PHP execution in your upload directory (belt + suspenders)
//    nginx:  location ~ ^/storage/uploads/.*\.php$ { deny all; return 403; }
//    Apache: <FilesMatch "\.php$"> Require all denied </FilesMatch> in storage dir .htaccess
//    (Also set open_basedir / disable exec() per security.md)

// 5. Strip EXIF metadata from user-uploaded images (privacy + orientation attacks)
//    composer require spatie/pdf-to-image  (or use intervention/image ^3.5)
//    $image = Image::read($file)->orientate()->save($path);   // orientate() fixes rotation; strip EXIF separately

// 6. For SVGs, sanitize with a strict allowlist (SVGs can contain inline JS)
//    composer require enshrined/svg-sanitize
//    $sanitized = (new Sanitizer())->sanitize($file->get());
//    Storage::put($path, $sanitized);
```

**Rule of thumb:** if user-uploaded files are ever served from a path your web server can
interpret as executable code, you have a config bug — fix that, don't rely on filename checks.

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

> **Third-party packages need separate hardening.** `spatie/laravel-medialibrary` < 11.23.0 has two high-severity issues — CVE-2026-48555 (SSRF via `addMediaFromUrl()`) and CVE-2026-48557 (upload restriction bypass via `defaultSanitizer()`). See `security.md` § "Critical: spatie/laravel-medialibrary SSRF + Upload Bypass" for affected versions and the upgrade command. Same for `plank/laravel-mediable` (CVE-2026-4809, CVSS 9.3, no upstream patch).
>
> **Common third-party upload gotchas:**
> - `spatie/laravel-medialibrary` `addMediaFromUrl()` — validate the URL host against an allowlist before fetching. Never trust user-supplied URLs (SSRF → cloud metadata `169.254.169.254`, internal services, `file://` schemes).
> - `livewire/livewire` `<livewire:upload>` + `withFileUploads` — always pair with a max-size rule in KB; default is unbounded until you set one.
> - `intervention/image` < 3.5 — image processing pipeline mishandles EXIF orientation on JPEG; pin to ^3.5.

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

---

## Updated from Research (2026-06-30, cycle 15)

**file-uploads.md** (3.25 days stale — second-oldest file) — three gaps that AI models keep getting
wrong, now documented:

- **`Storage::temporaryUploadUrl()` returns `['url' => $url, 'headers' => $headers]`** (Laravel 13.x).
  Older docs and StackOverflow answers show it returning a plain string URL — that's the legacy
  signature. Modern usage must destructure the array and **forward the headers** with the PUT,
  or S3 will reject the upload with a `SignatureDoesNotMatch` error. Updated the worked example.
- **File Visibility section** — the `public` / `private` visibility model and how to flip it.
  AI models frequently suggest `Storage::url()` for sensitive files without understanding that
  visibility on S3 is per-object ACL and not auto-updated when you rotate buckets. New section
  covers the gotcha: `setVisibility()` is a metadata operation, not a re-upload, so CDN-cached
  URLs remain served until you purge them.
- **Anti-extension-spoofing section** — the `evil.php.jpg` attack and the layered defense
  (never trust original filename → `hashName()`; validate with `finfo` on the actual file
  bytes; pair Laravel's `mimes:` with `file|image` to force symfony MIME guesser; disable PHP
  execution in upload dirs in nginx/Apache; sanitize SVGs; strip EXIF). This is the most
  common blind spot in code-review checklists.

**No new Laravel 13.x release** as of 2026-06-30 18:00 UTC. v13.17.0 (June 23, 2026) remains the
latest. v13.17.1 / v13.18.0 still pending.
