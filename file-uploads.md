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

> **`temporaryUploadUrl()` is driver-restricted.** Only `s3` and `local` drivers implement it
> natively. On R2 / MinIO / Backblaze / DigitalOcean Spaces, it throws
> `RuntimeException("Driver ... does not support generating temporary upload URLs.")`. For those,
> reach for the underlying AWS SDK directly (`Storage::disk('r2')->getAdapter()->getClient()->createPresignedRequest(...)`),
> or use `mnapoli/laravel-local-temporary-upload-url` to add local parity.

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

> **CVE-2026-49972 cross-reference (plank/laravel-mediable, 2026-07-13):** The `shell.php.jpg` anti-pattern is now a formally-disclosed CVE in `plank/laravel-mediable` (CVSS 8.8, fixed in 7.0.0 / commit `49e3583`). Root cause is identical to the bullet above: `pathinfo(PATHINFO_FILENAME)` preserves the inner `.php` while the sanitizer + Laravel's `mimes:` rule check only the outer `.jpg`. On misconfigured Apache/nginx where any filename containing `.php` is interpreted as PHP, the file executes on next request. Defense layers: (1) **use `UploadedFile::hashName()` / `Str::uuid()` for storage names — never `pathinfo(PATHINFO_FILENAME)`**; (2) enable `php_admin_flag engine off` in the uploads pool (FrankenPHP/LiteSpeed), or `location ~ ^/storage/uploads/.*\.php$ { deny all; return 403; }` (nginx), or `<FilesMatch "\.php$"> Require all denied </FilesMatch>` (Apache); (3) validate server-side MIME via `image` rule, not `mimetypes` alone. Full CVE writeup + defense-in-depth pattern in `security.md` § "Critical: plank/laravel-mediable — CVE-2026-4809 + CVE-2026-49969 + CVE-2026-49970 + CVE-2026-49972".

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

> **Third-party packages need separate hardening.** `spatie/laravel-medialibrary` < 11.23.0 has two high-severity issues — CVE-2026-48555 (SSRF via `addMediaFromUrl()`) and CVE-2026-48557 (upload restriction bypass via `defaultSanitizer()`). See `security.md` § "Critical: spatie/laravel-medialibrary SSRF + Upload Bypass" for affected versions and the upgrade command. Same for `plank/laravel-mediable` — **four CVEs now**, all addressed in 7.0.0 (2026-07-12/13): CVE-2026-4809 (CVSS 9.3 Critical — client-MIME trust, partial 7.0.0 fix), CVE-2026-49969 (CVSS 7.4 High — SSRF via `RemoteUrlAdapter`), CVE-2026-49970 (CVSS 5.4 Medium — path traversal via `File::sanitizePath()`), and **CVE-2026-49972 (CVSS 8.8 High — double-extension RCE via `pathinfo(PATHINFO_FILENAME)`)**. See `security.md` § plank/laravel-mediable for upgrade command + defense-in-depth (pathinfo → hashName, server MIME, disable PHP execution in upload dir).
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
9. **Calling `temporaryUploadUrl()` on R2 / MinIO / Backblaze without an SDK fallback** — only `s3` and `local` drivers implement it natively; for R2/MinIO, reach for `Storage::disk('r2')->getAdapter()->getClient()->createPresignedRequest(...)` (AWS SDK passthrough), or use `mnapoli/laravel-local-temporary-upload-url` for local dev parity
10. **Assuming `Storage::fake('s3')` exercises the S3 driver** — it doesn't. Faked disks write to `storage/framework/testing/disks/s3/` locally; no network call is made. Add a real-S3 integration test (or MinIO container) for the SDK-level layer
11. **Using `Storage::fake()` when the test needs to inspect the uploaded bytes after the fact** — `fake()` deletes everything in the temp dir at end-of-test. For byte-level content assertions (`Image::make($bytes)->width()`), use `Storage::persistentFake($disk)` instead
12. **Assuming `putFileAs()` returns `false` on out-of-disk space** — legacy Flysystem local-adapter bug (framework #25288): it creates a 0-byte file and returns the success path. Verify `Storage::size($path) > 0` after the call, especially in Docker/CI ephemeral disks
13. **Opening `finfo` in a controller method body and forgetting `finfo_close()`** — leaks the resource handle across Octane requests. Hoist `$finfo = finfo_open(...)` into `AppServiceProvider::register()` (or wherever you own the singleton) and close in `Octane::flush()`

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
latest. v13.18.0 shipped 2026-06-30; relevant fixes are in it.

## Streaming APIs: `putFile` vs `storeAs` vs `writeStream`

Three streaming-native APIs that all avoid `file_get_contents()`-style full-file load, with
different ergonomics. Pick by intent:

| API | Input | Auto-filename | Returns | Use when |
|---|---|---|---|---|
| `$file->store($path)` | `UploadedFile` | UUID (`hashName()`) | path string | Most "save the upload" cases |
| `$file->storeAs($path, $name)` | `UploadedFile` | explicit | path string | You need a deterministic name |
| `Storage::putFile($path, $file)` | `UploadedFile` / `File` | UUID | path string | **Storage-facade-first code** — same as `$file->store()` but no need to call it on the file |
| `Storage::putFileAs($path, $file, $name)` | `UploadedFile` / `File` | explicit | path string | Storage-facade-first + custom name |
| `Storage::put($path, $contents, $options)` | string | explicit | bool | Inline content / already-loaded bytes |
| `Storage::putStream($path, $resource, $options)` | resource (`fopen(...)`) | explicit | bool | Streaming from PHP open file/URL handles |
| `Storage::writeStream($path, $resource, $options)` | resource | explicit | bool | Alias of `putStream` on most drivers |

```php
use Illuminate\Http\File;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

// putFile(): auto-UUID filename, storage-facade-first style
$path = Storage::disk('s3')->putFile('photos', $request->file('avatar'));

// putFileAs(): explicit name (e.g. user-id.jpg for a deterministic path)
$path = Storage::disk('s3')->putFileAs(
    "photos/{$user->id}",
    $request->file('avatar'),
    "{$user->id}.jpg"
);

// writeStream(): write from any PHP resource — direct stream, never loads full bytes
$resource = fopen('https://example.com/large-video.mp4', 'r');
$path = 'videos/import.mp4';
Storage::disk('s3')->putStream($path, $resource, ['StorageClass' => 'STANDARD_IA']);
if (is_resource($resource)) fclose($resource);
```

**All three stream the file** — they `fopen($file->getRealPath(), 'r')` and `stream_copy_to_stream`
to the driver's put-stream. Constant memory regardless of file size.

**`putFileAs()` return value caveat** — returns the **path string** on success or **`false`**
on failure. But the legacy Flysystem local-adapter bug ([laravel/framework#25288](https://github.com/laravel/framework/issues/25288))
is still reproducible: on out-of-disk space, it creates a 0-byte file and returns the success
path. **Always verify** with `Storage::size($path) > 0` (and ideally `->exists()`) after the
call, especially on local ephemeral disks (Docker containers, CI runners).

## S3 Multipart Uploads (Files >5 GB)

S3 **PUT** hard-limits objects at **5 GB**. Files larger than that require multipart upload,
which is a 3-step dance per part:

1. **`CreateMultipartUpload`** — tell S3 "I'm starting a multipart upload to this key" → returns
   an `UploadId`.
2. **Per-part `UploadPart`** — sign a PUT URL for each 5 MB–5 GB chunk, upload in parallel from
   the client. Each part returns an `{PartNumber, ETag}` from S3.
3. **`CompleteMultipartUpload`** — hand S3 the full part list with ETags → S3 stitches them.

```php
// Server: open a multipart upload and hand the client a signed URL per part
use Aws\S3\MultipartUploader;
use Illuminate\Support\Facades\Storage;

public function startMultipart(Request $request): JsonResponse
{
    $key = "uploads/{$request->user()->id}/" . Str::uuid() . '/' . $request->filename;

    // flysystem-aws-s3-v3 ships MultipartUploader; for the simpler "presigned per part" flow:
    $client = Storage::disk('s3')->getAdapter()->getClient();

    $result = $client->createMultipartUpload([
        'Bucket' => config('filesystems.disks.s3.bucket'),
        'Key'    => $key,
    ]);

    // Store UploadId keyed to the user's session/job; the client polls it after each part
    Cache::put("upload:{$result['UploadId']}", ['key' => $key, 'parts' => []], now()->addHour());

    return response()->json([
        'upload_id' => $result['UploadId'],
        'key'       => $key,
        'chunk_size' => 10 * 1024 * 1024, // 10 MB chunks
    ]);
}

// Client-side: each chunk PUTs directly to a per-part presigned URL, reports ETag back
// GET /api/uploads/{uploadId}/part-url?partNumber=3 → returns one UploadPart URL
// POST /api/uploads/{uploadId}/complete  → finalises with the ETag list
```

**Practical alternatives for browser uploads:**
- **`tus.io` resumable uploads** (see the **Chunked Uploads** section above) — handles the
  multipart-on-client-side transparently and only needs ~10 lines of frontend + the
  `tus-php-server` package. Best for "user drops a 20 GB video into the browser."
- **`Uppy` / `FilePond` / `uppy-aws-s3-multipart` plugin** — JS plugins that speak S3 multipart
  natively. Pair with the server-side chunk-signing endpoint above (15 lines of routes).
- **`S3 Transfer Acceleration`** — for cross-region uploads, leave multipart to the client lib.

**Don't roll your own chunk-and-reassemble** — you'd have to manage partial files, cleanup,
and parallel transfer, all of which the AWS SDK already does. Use the SDK or a wrapper.

## Testing File Uploads

```php
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

public function test_user_can_upload_avatar(): void
{
    // fake() deletes everything in the temp dir at the end of the test.
    // Use this when assertions are about the FILE EXISTING, not the contents.
    Storage::fake('avatars');

    $response = $this->post('/avatar', [
        'avatar' => UploadedFile::fake()->image('me.jpg', 200, 200),
    ]);

    $response->assertOk();
    Storage::disk('avatars')->assertExists('me.jpg'); // ✅
    // Storage::disk('avatars')->get('me.jpg')  →  throws FileNotFoundException (deleted)
}

public function test_user_uploaded_image_was_resized_correctly(): void
{
    // persistentFake() keeps files on disk after the test so you can read them back
    // for byte-level or content-level assertions. Required for:
    //   - "the resized image is 300px wide"
    //   - "the watermarked PDF has the user_id text in the right place"
    //   - any test that introspects the upload's content
    Storage::persistentFake('avatars');

    $response = $this->post('/avatar', [
        'avatar' => UploadedFile::fake()->image('me.jpg', 1000, 1000),
    ]);

    $response->assertOk();

    // Read bytes that survived the test
    $bytes = Storage::disk('avatars')->get('avatars/' . $response->json('filename'));
    $image = \Intervention\Image\Laravel\Facades\Image::make($bytes);
    $this->assertEquals(300, $image->width());

    // Files persist between test runs until you call:
    //   Storage::disk('avatars')->deleteDirectory('/');  // or a one-shot after the assertion
}
```

**Where the faked disk actually lives on disk:**

```
storage/framework/testing/disks/{disk_name}/
```

Even if you call `Storage::fake('s3')`, **S3 is not contacted.** The fake swaps the disk for a
local temp dir at `storage/framework/testing/disks/s3/`. This is the #1 source of "but my
presigned URL was never called!" confusion — faked tests prove your code talks to the **filesystem
adapter correctly**, not that it talks to S3. Add a real-S3 integration test outside the suite
(or under `tests/Integration/` with a separate CI tag) for that layer.

**`Storage::fake()` gotchas:**
- Calling `Storage::fake()` without an argument fakes the `local` default disk, not the `public`
  one. If your controller writes to `Storage::disk('public')`, call `Storage::fake('public')`.
- `Storage::fake()` does NOT fake URL generation via `temporaryUrl()` — those return real
  strings (signature isn't validated against a real S3 call). For real presigned-URL testing,
  you need a MinIO container or S3 mock (see `aws-sdk-php-mock`).
- Multiple `Storage::fake()` calls in one test → the second call **wipes the first** (the
  underlying swap is single-slot). Use `Storage::fake('disk-a')` + `Storage::fake('disk-b')`
  if you need both, in that order; or call `Storage::fake()` once and stack other disks onto
  your test setup.

**Three other useful asserters:**

```php
Storage::disk('photos')->assertExists('photo1.jpg');
Storage::disk('photos')->assertExists(['photo1.jpg', 'photo2.jpg']);   // multiple
Storage::disk('photos')->assertMissing('deleted.jpg');
Storage::disk('photos')->assertDirectoryEmpty('/wallpapers');
Storage::disk('photos')->assertCount('/wallpapers', 2);
```

Sources: [Laravel 13 Filesystem — Testing](https://laravel.com/docs/13.x/filesystem#testing) |
[Laravel 13 Filesystem — Automatic Streaming](https://laravel.com/docs/13.x/filesystem#automatic-streaming) |
[Laravel 13 Filesystem — Temporary Upload URLs](https://laravel.com/docs/13.x/filesystem#temporary-upload-urls) |
[mnapoli/laravel-local-temporary-upload-url](https://github.com/mnapoli/laravel-local-temporary-upload-url) |
[laravel/framework#25288](https://github.com/laravel/framework/issues/25288) |
[AWS S3 Multipart Upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)


---

## Updated from Research (2026-07-11, cycle 34)

**file-uploads.md** (10 days stale — oldest untouched file since cycle 15) — three fresh gaps that
AI models keep getting wrong, plus two minor clarifications on existing sections.

- **`Storage::persistentFake()` for inspecting uploaded files in tests** — Laravel 11+ ships
  `Storage::persistentFake($disk)` as the documented alternative to `Storage::fake($disk)`. The
  default `fake()` method **deletes all files in its temporary directory at the end of the test**;
  `persistentFake()` keeps them on disk so you can `file_get_contents()` them, render them in
  `assertJson` body assertions, or eyeball them in `storage/framework/testing/disks/{name}/`. AI
  models default to `fake()` and then write workarounds like saving the file path to a global;
  `persistentFake()` is the answer. See the new **"Testing File Uploads"** section.
- **`putFile()` / `putFileAs()` vs `store()` / `storeAs()` vs `writeStream()`** — three streaming
  APIs that all avoid `file_get_contents()`-style full-file load, but with different ergonomics.
  AI models default to `storeAs()` and never mention `putFileAs()`, which **auto-generates a UUID
  filename and streams the file** (no path concatenation, no UUID line). `writeStream()` takes a
  PHP resource (`fopen('http://...', 'r')`, a tmp file handle, an S3 read stream) and writes it
  directly. New **"Streaming APIs: putFile vs storeAs vs writeStream"** subsection.
- **`putFileAs` silent-zero-byte failure on out-of-disk** (Laravel framework issue #25288) — the
  legacy Flysystem local-adapter bug from Laravel 5.6 is still reproducible when `putFileAs()`
  hits a disk-full condition: it creates a 0-byte file and returns the success path instead of
  `false`. AI models assume `putFileAs` returns `false` on failure. Documented in the
  `Common Mistakes` list with the workaround (`Storage::size($path) > 0` check after upload).
- **S3 multipart upload for files >5 GB** — the answer to the cycle-15 watch-list item. S3 PUT
  hard-limits objects at 5 GB; larger files require multipart (`CreateMultipartUpload` →
  individual `UploadPart` signed URLs → `CompleteMultipartUpload`). Laravel does not ship a
  one-line API for this; the recommended pattern is `tus.io` (large/unreliable connections) or a
  hand-rolled multipart driver. New **"S3 Multipart Uploads (Files >5 GB)"** subsection.
- **`temporaryUploadUrl()` driver restrictions** — per the Laravel docs, only `s3` and `local`
  drivers implement `temporaryUploadUrl()`. AI models frequently reach for it on the `r2` or
  custom S3-compatible disks; it falls back to `RuntimeException("Driver ... does not support
  generating temporary upload URLs.")`. For R2/MinIO/Backblaze, sign the URL via the underlying
  AWS SDK directly (`Storage::disk('r2')->getAdapter()->getClient()->createPresignedRequest(...)`).
  Documented inline in the **"Direct S3 Upload (Presigned URL — No Server Involvement)"** section.

**Common Mistakes list in `file-uploads.md` grew 8 → 13 entries** — added "Calling
`temporaryUploadUrl()` on R2 / MinIO / Backblaze without an SDK fallback", "Assuming
`Storage::fake('s3')` exercises the S3 driver (it writes to local temp — fake disk lives at
`storage/framework/testing/disks/s3/`)", "Using `Storage::fake()` when you need to inspect the
uploaded bytes after the test (use `persistentFake()` instead)", "Assuming `putFileAs()` returns
`false` on out-of-disk space (legacy Flysystem local-adapter bug → 0-byte file, see framework
#25288)", and "Putting the `finfo` resource handle in a controller method body without
`finfo_close()` (resource leak across Octane requests)".

**SKILL.md** bumped `1.22.19 → 1.22.20`; 5 new cross-reference rows added (Storage::fake vs
persistentFake, putFile/putFileAs/writeStream decision matrix, S3 multipart >5 GB,
temporaryUploadUrl driver restrictions, finfo close hygiene).

**No version-stamp change to "Active Versions"** — `v13.19.0` / `v12.63.0` unchanged from
cycle 33. No Laravel 13.x release in the last 6 hours (cycle 34 trigger time: 2026-07-11
06:09 UTC). No file other than file-uploads.md was touched this cycle.

**Watch list for cycle 35:**
- **`localization.md`** — touched in cycle 27 (14 days stale), next gap-fill candidate. Likely
  targets: Laravel 13 `php artisan translatable:generate` (community / 13-starter-kit), ICU plural
  variants in Blade components (vs full `MessageFormat`), the Translation Memoization handler for
  Carbon dates, and the new `Locale::canonicalize()` helper.
- **`artisan.md`** — touched in cycle 4 (37 days stale, oldest file in the entire skill). Likely
  targets: `php artisan db:show --read --write`, scheduler `everyTwoHours()->between('...', '...')`,
  `Artisan::call()` exit-code propagation under Octane, and the new `$this->components->bulletList()`.
- **v13.19.1** — likely mid-to-late July 2026 once 5–10 post-13.19 PRs accumulate. Watch
  [github.com/laravel/framework/releases](https://github.com/laravel/framework/releases).
- **v13.20.0** — first minor after 13.19, likely late July / early August 2026.
- **Laravel 12 EOL** — bug fixes end **August 13, 2026** (33 days from cycle time). Plan
  migrations off 12.x accordingly.

---

## First-Party Image Processing (Laravel 13.20+, `Illuminate\Image`)

Laravel 13.20 ships a first-class `Image` facade backed by **Intervention Image v4** (GD and Imagick drivers). It handles resizing, cropping, format conversion, and storage without needing a third-party wrapper.

**Install the driver:**
```bash
composer require intervention/image
```

**Configure driver (optional — GD is default):**
```php
// .env
IMGIX_DRIVER=imagick
```

Or create `config/image.php`:
```php
<?php
return [
    'driver' => env('IMAGE_DRIVER', 'gd'),
];
```

### API Reference

The `Image` facade is **immutable** — every transformation returns a new `Image` instance.

```php
use Illuminate\Support\Facades\Image;

// ── Input sources ──────────────────────────────────────────
Image::read('photos/hero.jpg');                        // local file path
Image::read($request->file('photo'));                 // UploadedFile
Image::read(Storage::disk('s3')->path('uploads/x.png')); // from storage
Image::read('https://example.com/remote.jpg');         // remote URL
Image::read($bytes);                                   // raw binary string
Image::read(fopen('photo.jpg', 'r'));                 // PHP resource

// ── Resize ────────────────────────────────────────────────
// Constrain to max width (proportional height)
Image::read($file)->resize(maxWidth: 800)->save();

// Constrain to max height (proportional width)
Image::read($file)->resize(maxHeight: 600)->save();

// Exact dimensions (center-crop to fit)
Image::read($file)->resize(width: 400, height: 400)->save();

// Resize only if larger (don't upscale)
Image::read($file)->resize(maxWidth: 1200, height: 1200, keepAspectRatio: true)->save();

// ── Crop ──────────────────────────────────────────────────
Image::read($file)->crop(width: 200, height: 200)->save();

// Crop from specific coordinates (x, y, width, height)
Image::read($file)->crop(x: 50, y: 50, width: 300, height: 300)->save();

// ── Format conversion ─────────────────────────────────────
Image::read('photo.jpg')->toJpeg(quality: 85)->save('photo.jpg');
Image::read('photo.jpg')->toWebp(quality: 80)->save('photo.webp');
Image::read('photo.png')->toPng()->save('photo.png');
Image::read('photo.jpg')->toGif()->save('photo.gif');

// ── Effects ───────────────────────────────────────────────
Image::read($file)->blur(amount: 5)->save();          // 1–100
Image::read($file)->sharpen(amount: 80)->save();       // 1–100
Image::read($file)->brightness(20)->save();           // -100 to 100
Image::read($file)->contrast(20)->save();             // -100 to 100
Image::read($file)->flip('h')->save();                // 'h' or 'v'
Image::read($file)->rotate(90)->save();

// ── Save options ───────────────────────────────────────────
Image::read($file)->resize(800)->save();                     // overwrite source
Image::read($file)->resize(800)->save('output.jpg');          // new path
Image::read($file)->resize(800)->toJpeg(quality: 85)->save(); // convert + save

// ── Inline manipulation (modify in memory, no file I/O) ───
$img = Image::read($file)->resize(400)->blur(2);
$img2 = $img->sharpen(80);  // $img is unchanged (immutable)

// ── Stream directly to storage (no local temp file) ─────────
Image::read($request->file('banner'))
    ->resize(maxHeight: 300)
    ->toJpeg(quality: 85)
    ->store(path: 'banners/' . Str::uuid() . '.jpg', disk: 's3');

// Or store to local disk
Image::read($file)->resize(200)->store(path: 'thumbs/x.jpg');
```

### Driver Capabilities

| Operation      | GD  | Imagick |
|----------------|-----|---------|
| `resize`       | ✅  | ✅      |
| `crop`         | ✅  | ✅      |
| `toJpeg`       | ✅  | ✅      |
| `toWebp`       | ✅  | ✅      |
| `toPng`        | ✅  | ✅      |
| `toGif`        | ✅  | ✅      |
| `blur`         | ✅  | ✅      |
| `sharpen`      | ✅  | ✅      |
| `brightness`   | ✅  | ✅      |
| `contrast`     | ✅  | ✅      |
| `flip`         | ✅  | ✅      |
| `rotate`       | ✅  | ✅      |
| `toGreyscale`  | ✅  | ✅      |

### Common Patterns

**Avatar resize (square, center-crop):**
```php
$avatar = Image::read($request->file('avatar'))
    ->resize(width: 256, height: 256)
    ->crop(width: 256, height: 256)
    ->toJpeg(quality: 90)
    ->store(path: 'avatars/' . $user->id . '.jpg', disk: 'public');
```

**Blog hero image (max width, strip EXIF):**
```php
$path = Image::read($request->file('hero'))
    ->resize(maxWidth: 1200)
    ->toWebp(quality: 85)
    ->store(path: 'blog/heroes/' . Str::uuid() . '.webp');
```

**Thumbnail generation (batch):**
```php
foreach ($request->file('gallery') as $photo) {
    $uuid = Str::uuid();
    Image::read($photo)
        ->resize(maxWidth: 400)
        ->toJpeg(quality: 80)
        ->store(path: "thumbs/{$uuid}.jpg");
    Image::read($photo)
        ->resize(maxWidth: 800)
        ->toJpeg(quality: 85)
        ->store(path: "medium/{$uuid}.jpg");
}
```

### Testing

```php
use Illuminate\Support\Facades\Image;

// Fake for testing (avoids actual image processing in tests)
Image::fake($imageResource);
$image->assertWidth(800);
$image->assertHeight(600);

// Fake a specific image size
Image::fake(1024, 768);
// Then:
Image::read(...)
```

### Common Mistakes

- **`->save()` without args overwrites the source file** — always pass a path or use `->store()` when you don't want to modify the original
- **Intervention Image v4 required, not v3** — the facade interface targets v4 API. `composer require intervention/image:^4.0`
- **Memory limits on large images** — GD driver is single-threaded and memory-intensive. For server-side image processing at scale (batch resize thousands of photos), prefer Imagick and set `memory_limit: -1` in PHP or use a queue job
- **Format conversion drops EXIF orientation** — if you need to preserve EXIF data (camera orientation, GPS), use Imagick and preserve metadata explicitly

> **Sources:** [Laravel 13.20.0 release notes](https://github.com/laravel/framework/releases/tag/v13.20.0) | [Laravel News: First-Party Image Processing](https://laravel-news.com/laravel-13-20-0) | [Intervention Image v4 docs](https://image.intervention.io/v4/)
