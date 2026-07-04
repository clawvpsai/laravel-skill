# Deployment — Production, Env, Configs, Docker, Octane

## Environment Variables

```bash
# .env — NEVER commit this
APP_NAME=MyApp
APP_ENV=production
APP_KEY=base64:xxxx  # generate with php artisan key:generate
APP_DEBUG=false      # never true in production
APP_URL=https://yourapp.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=myapp
DB_USERNAME=myapp_user
DB_PASSWORD=secure_password

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=587
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS="hello@yourapp.com"
```

## Production Deployment Steps

```bash
# 0. Update Composer itself first — three CVEs (CVE-2026-45793 token leak + two Perforce command injections)
#    were disclosed in 2026 and fixed in 2.9.8 / 2.2.28 / 1.10.28. `composer install` runs the
#    binary you're updating, so do this OUTSIDE the deploy job that runs composer install — in a
#    separate step, a Docker build layer, or your AMI/image build.
composer self-update           # or: composer self-update --2 (LTS 2.2.x)
composer --version             # verify ≥ 2.9.8 or ≥ 2.2.28

# 1. Clone/pull code
git pull origin main

# 2. Install dependencies (optimized, no dev)
composer install --optimize-autoloader --no-dev

# 3. Run migrations
php artisan migrate --force  # --force bypasses confirmation

# 4. Clear and rebuild caches
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache  # if using events

# 5. Restart queue workers
php artisan queue:restart  # tells workers to reload new code

# 6. If using Octane/Swoole
php artisan octane:reload
```

## Zero-Downtime Deployment

The key: never serve old code while new code is being prepared.

### Blue-Green Deployment
```bash
# Deploy to a new directory, then swap symlink
ln -sfn /var/www/releases/v2 /var/www/current

# Nginx points to /var/www/current/public
# Old release stays on disk until you clean it up
```

### Deployment Script with Rollback
```bash
#!/bin/bash
set -e

APP_DIR="/var/www/myapp"
RELEASE_DIR="${APP_DIR}/releases/$(date +%Y%m%d%H%M%S)"
KEEP_RELEASES=3

echo "Deploying to ${RELEASE_DIR}..."

# Clone/pull to new release dir
git clone git@github.com:owner/repo.git "${RELEASE_DIR}"
cd "${RELEASE_DIR}"

# Install deps
composer install --optimize-autoloader --no-dev

# Copy .env
cp "${APP_DIR}/.env" "${RELEASE_DIR}/.env"

# Run migrations
php artisan migrate --force

# Cache configs
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Symlink storage
ln -sfn "${APP_DIR}/storage" "${RELEASE_DIR}/storage"

# Atomic swap
ln -sfn "${RELEASE_DIR}" "${APP_DIR}/current"

# Restart workers
php artisan queue:restart || true

# Prune old releases
cd "${APP_DIR}/releases"
ls -1t | tail -n +$((KEEP_RELEASES + 1)) | xargs rm -rf

echo "Deployment complete"
```

### Health Check After Deploy
```bash
# Wait for app to be ready before declaring success
for i in {1..30}; do
    if curl -sf "https://yourapp.com/up" > /dev/null; then
        echo "Health check passed"
        exit 0
    fi
    sleep 1
done
echo "Health check failed"
exit 1
```

## Docker

### Dockerfile
```dockerfile
FROM php:8.4-fpm-alpine

# Install system deps
RUN apk add --no-cache \
    nginx \
    supervisor \
    postgresql-dev \
    linux-headers \
    $PHPIZE_DEPS

# Install PHP extensions
RUN docker-php-ext-install pdo pdo_mysql pgsql pcntl bcmath

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# App user
RUN adduser -D -u 1000 app

WORKDIR /var/www

COPY --chown=app:app . .

RUN composer install --optimize-autoloader --no-dev

EXPOSE 8080

CMD ["php", "artisan", "octane:work", "--port=8080"]
# Or for FPM:
# CMD ["php-fpm"]
```

### Docker Compose (Laravel + Redis + Postgres)
```yaml
services:
  app:
    build: .
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - .:/var/www
      - ./storage:/var/www/storage
    environment:
      - APP_ENV=production
      - APP_DEBUG=false
      - DB_CONNECTION=pgsql
      - DB_HOST=db
      - DB_PORT=5432
      - DB_DATABASE=myapp
      - CACHE_DRIVER=redis
      - QUEUE_CONNECTION=redis
      - SESSION_DRIVER=redis
      - REDIS_HOST=redis
    depends_on:
      - db
      - redis

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - .:/var/www
      - ./storage:/var/www/storage
    depends_on:
      - app

  queue:
    build: .
    command: php artisan queue:work redis --queue=high-priority,default --tries=3
    restart: unless-stopped
    volumes:
      - .:/var/www
    depends_on:
      - redis
      - db

  scheduler:
    build: .
    command: sh -c "while true; do php artisan schedule:run; sleep 60; done"
    restart: unless-stopped
    volumes:
      - .:/var/www

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp_user
      POSTGRES_PASSWORD: secure_password
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

### Nginx Docker Config

```nginx
# Define rate-limit zones once in the http {} block (outside any server {})
limit_req_zone $binary_remote_addr zone=api_rl:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=api_conn:10m;

server {
    listen 80;
    server_name yourapp.com;
    return 301 https://yourapp.com$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourapp.com;

    root /var/www/public;
    index index.php;

    ssl_certificate /etc/letsencrypt/live/yourapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourapp.com/privkey.pem;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Gzip
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # ============================================================
    # Slow JSON Stream DoS mitigations (Laravel Tier 1, CVSS 7.5)
    # Disclosed 2026-06-27 — see security.md for full details
    # ============================================================
    # Hard ceiling: close any body that takes longer than 10 s to read
    client_body_timeout 10s;
    # Minimum average read rate (bytes/s); closes drip attacks at 1 B/s
    client_min_rate 1024;
    # Cap body size — defense in depth for large-payload attacks
    client_max_body_size 10m;
    # Limit concurrent connections per IP (must reference the zone above)
    limit_conn api_conn 10;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass app:8080;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;

        # Per-request fastcgi timeout — backstop in case nginx misses
        fastcgi_read_timeout 30s;
    }

    # API routes — add explicit rate limiting on top of body timeouts
    location /api/ {
        limit_req zone=api_rl burst=20 nodelay;
        try_files $uri /index.php?$query_string;
    }

    # Static assets (no PHP)
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```
## Postgres Transaction Pooler Support (Laravel 13.17+)



First-class support for PgBouncer / Postgres pooler **transaction mode** in `Illuminate\Database\PostgresConnection`. Important when running Laravel behind serverless / edge platforms (Neon, Supabase, AWS RDS Proxy in transaction mode) where **persistent connections are forbidden** — the pooler sits between Laravel and Postgres and multiplexes connections on every transaction.

**How it differs from session-mode pooling:**
- **Session mode** — connection pinned to a backend for the lifetime of the client session (Laravel's default; works fine for long-running workers)
- **Transaction mode** — connection assigned only for the duration of a transaction; released back to the pool on commit/rollback (required for serverless / edge; saves connections at scale)

**Laravel 13.17 changes:**
- `PostgresConnection` correctly resets connection-scoped state between transactions when running against a transaction pooler
- No application-level config required — the driver detects pooler behavior via standard Postgres connection attributes
- Works with `DB::transaction(...)` calls — commits/rollbacks release the connection cleanly

**When you MUST use this:**
- Neon serverless Postgres (no session pinning)
- Supabase's pooled connection string (port 6543, transaction mode)
- AWS RDS Proxy in transaction mode
- Any PgBouncer deployment configured with `pool_mode = transaction`

**When you can ignore this:**
- Direct Postgres connection (no pooler)
- PgBouncer in `session` mode (works as before)
- AWS RDS Proxy in `session` mode (still pin-compatible)

**Connection string example for Neon:**
```
# .env
DB_CONNECTION=pgsql
DB_HOST=ep-cool-darkness-123456.us-east-2.aws.neon.tech
DB_PORT=5432
DB_DATABASE=neondb
DB_USERNAME=neondb_owner
DB_PASSWORD=npg_xxx
# Neon also exposes a pooled endpoint on a different host for transaction pooling
```

**Connection string example for Supabase (transaction pooler):**
```
DB_CONNECTION=pgsql
DB_HOST=aws-0-us-east-1.pooler.supabase.com
DB_PORT=6543        # transaction pooler port
DB_DATABASE=postgres
DB_USERNAME=postgres.PROJECT_REF
DB_PASSWORD=xxx
```

**Gotchas (transaction mode limits):**
- `PREPARE` / `DEALLOCATE` statements don't persist across transactions — Laravel already re-prepares as needed
- `SET LOCAL` is the only `SET` allowed (it resets at transaction end) — Laravel's `SET search_path` / `SET statement_timeout` should be `SET LOCAL`
- Advisory locks (`pg_advisory_lock`) **do not work** in transaction mode — use a Redis lock instead
- `LISTEN` / `NOTIFY` don't work (session-scoped) — use Postgres triggers that write to a queue table instead
- **API/JSON Routes Respect `php artisan down` (Laravel 13.18.1, PR #60595 by @davidrushton)** — when the app is in maintenance mode with `secret` bypass (`php artisan down --secret=...`), requests to `/api/*` or routes that set `Accept: application/json` were not always gated correctly. Now the `Down` command handles both web and API/JSON routes uniformly — maintenance mode responses return JSON (`{"message": "Service Unavailable", "retry_after": N}`) for API/JSON callers instead of falling through to a 500. If you've hand-rolled a "maintenance mode" middleware for API routes because the built-in one didn't catch them, you can now delete it.

Source: [PR #60425](https://github.com/laravel/framework/pull/60425) | [Laravel 13.17 Release Notes](https://github.com/laravel/framework/releases/tag/v13.17.0) | [Neon connection pooling docs](https://neon.tech/docs/connect/connection-pooling)

## Laravel Octane (Swoole/Roadrunner)

Octane boots the app once and serves requests from memory — dramatically faster than FPM.

```bash
composer require laravel/octane

# Install with Swoole
php artisan octane:install --server=swoole

# Start
php artisan octane:start --port=8080 --workers=4

# Reload (zero-downtime deploy)
php artisan octane:reload

# Roadrunner alternative
php artisan octane:install --server=roadrunner
```

**Octane + Nginx config:**
```nginx
upstream laravel octane {
    server 127.0.0.1:8080 weight=4;
    server 127.0.0.1:8081 weight=1;  // spare for graceful reload
}

server {
    # ...
    location / {
        proxy_pass http://laravel;
        proxy_http_version 1.1;
        proxy_set_header Connection "upgrade";
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Octane caveats:**
- State is persistent across requests — don't store request-specific data in properties
- Use `Octane::tick()` for background tasks
- Database connections are pooled — close connections after long idle: `DB::purge()`

## Queue Worker (Supervisor)

```ini
# /etc/supervisor/conf.d/laravel-worker.conf
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/myapp/artisan queue:work redis --queue=high-priority,default --tries=3 --backoff=60
directory=/var/www/myapp
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=4
redirect_stderr=true
stdout_logfile=/var/log/laravel-worker.log
stopwaitsecs=3600
```

## Health Check Endpoint

Laravel 12+ has built-in `/up` health endpoint:

```php
// bootstrap/app.php — already included by default in Laravel 12+
->withRouting(
    health: '/up',
)
```

**Custom health check (returns component status):**
```php
Route::get('/up', function () {
    $checks = [
        'database' => tryPingingConnection('mysql'),
        'cache' => Cache::getStore() !== null,
        'queue' => true, // queue:work running implies healthy
    ];

    $allHealthy = !in_array(false, $checks, true);

    return response()->json([
        'status' => $allHealthy ? 'ok' : 'degraded',
        'checks' => $checks,
        'time' => now()->toIso8601String(),
    ], $allHealthy ? 200 : 503);
});
```

## PHP Config (php.ini)

```ini
; Production settings
display_errors = Off
log_errors = On
max_execution_time = 60
memory_limit = 256M
post_max_size = 64M
upload_max_filesize = 64M
```


## PHP-FPM Pool Hardening (Slow JSON Stream backstop — 2026-06-28)

The Slow JSON Stream attack (disclosed June 27, 2026 — Tier 1 for Laravel, CVSS 7.5) showed that PHP-FPM workers can be pinned for minutes by a single chunked JSON request with no body-read timeout. Even with nginx `client_body_timeout 10s` in front, you need a backstop at the FPM layer because nginx can be bypassed (Octane, Cloudflare passthrough, internal services).

```ini
; /etc/php/8.4/fpm/pool.d/www.conf
[www]
user = www-data
group = www-data

pm = dynamic
pm.max_children = 20            ; tune to RAM: ~100 MB per worker on a 2 GB host
pm.start_servers = 4
pm.min_spare_servers = 2
pm.max_spare_servers = 6

; Recycle workers after 500 requests — prevents unbounded RSS growth under
; sustained slow-body attacks where each worker accumulates buffer state
pm.max_requests = 500

; Hard kill for any request over 30 s — this is the PHP-level equivalent of
; nginx's client_body_timeout. Last line of defense if nginx is bypassed.
pm.request_terminate_timeout = 30s

; Track slow workers
slowlog = /var/log/php-fpm/www-slow.log
request_slowlog_timeout = 10s

; Tighten per-request resource limits
php_admin_value[memory_limit] = 256M
php_admin_value[post_max_size] = 10m
php_admin_value[upload_max_filesize] = 10m
php_admin_value[max_execution_time] = 30
```

**Quick verification after deploy:**

```bash
# Check the effective config
sudo php-fpm8.4 -tt

# Confirm workers recycle
sudo tail -f /var/log/php-fpm/www-slow.log

# Test the timeout yourself — should return 504 after 30 s
curl -X POST -H "Transfer-Encoding: chunked" -H "Content-Type: application/json" \
  --max-time 60 \
  --data-binary $'0\r\n\r\n' \
  https://yourapp.com/api/test
```

**Why `request_terminate_timeout` is the critical setting:** when nginx is bypassed (Laravel Octane/RoadRunner exposed directly, or internal services), there is no upstream body timeout — `request_terminate_timeout` is what closes the FPM worker so it can serve the next request. Without it, a single blocked request ties up a worker until OOM.


## OPcache + JIT Production Tuning (PHP 8.3+)

OPcache is the single biggest free performance win for Laravel. With `validate_timestamps=0` and a preload script, the framework autoloader, service container, and compiled config are loaded into shared memory once at worker startup — eliminating the per-request disk + parse cost on the bootstrap hot path.

```ini
; /etc/php/8.4/fpm/conf.d/10-opcache.ini
[opcache]
opcache.enable=1
opcache.enable_cli=0                   ; CLI workers can re-validate
opcache.memory_consumption=256         ; MB — bump if you see "interned string buffer full"
opcache.interned_strings_buffer=64
opcache.max_accelerated_files=20000    ; MUST exceed your project's PHP file count
opcache.max_wasted_percentage=10
opcache.validate_timestamps=0          ; PRODUCTION ONLY — clear opcache during deploys
opcache.revalidate_freq=0              ; unused when validate_timestamps=0
opcache.fast_shutdown=1
opcache.save_comments=1
opcache.preload=/var/www/html/preload.php
opcache.preload_user=www-data

; PHP 8.3+ JIT — only meaningful for CPU-bound code (image processing, math, JSON-heavy APIs)
; tracing JIT is generally the best starting point for Laravel apps
opcache.jit=tracing
opcache.jit_buffer_size=64M
```

**`preload.php` (place at project root, commit it):**
```php
<?php
// Preload the framework + compiled config so the autoloader is bypassed on every request.
// Run `php artisan config:cache route:cache event:cache view:cache` before generating this.
$app = require_once __DIR__.'/bootstrap/app.php';
$app->make(Illuminate\Contracts\Console\Kernel::class)->bootstrap();

$preload = function (string $path) use (&$preload): void {
    if (is_file($path)) {
        opcache_compile_file($path);
        return;
    }
    if (is_dir($path)) {
        foreach (new RecursiveIteratorIterator(new RecursiveDirectoryIterator($path)) as $file) {
            if ($file->isFile() && str_ends_with($file->getFilename(), '.php')) {
                opcache_compile_file($file->getPathname());
            }
        }
    }
};

// Preload the framework and your compiled config files
$preload(__DIR__.'/vendor/laravel/framework/src/Illuminate');
$preload(__DIR__.'/bootstrap/cache');
// Optionally preload your app code (larger preload = more shared memory, slower worker boot)
$preload(__DIR__.'/app');
```

**Critical deploy rule:** when `validate_timestamps=0`, you MUST invalidate OPcache on every deploy or stale code keeps running. Pick ONE:

```bash
# Option A — restart the SAPI (simplest, brief downtime)
sudo systemctl reload php8.4-fpm

# Option B — surgical cache flush (no downtime, less thorough)
php -r 'opcache_reset(); echo "opcache reset\n";'
# Then call it from a privileged script in your deploy pipeline.

# Option C — opcache.file_cache_fallback (PHP 7.4+) lets you ship a precompiled
# file cache to each host; opcache picks it up on the next request without restart.
```

**Monitoring (add to observability dashboard):**
```php
// Hit this endpoint from a Prometheus blackbox exporter
Route::get('/metrics/opcache', function () {
    $status = opcache_get_status(false);
    return response()->json([
        'memory_used_pct' => $status['memory_usage']['used_memory'] / $status['memory_usage']['free_memory'] * 100,
        'hit_rate' => $status['opcache_statistics']['opcache_hit_rate'] ?? 0,
        'cached_scripts' => $status['opcache_statistics']['num_cached_scripts'] ?? 0,
        'jit_buffer_used' => $status['jit']['buffer_size'] ?? 0,
    ]);
})->middleware('auth.basic');  // gate it — opcache stats can leak app surface
```

**When JIT helps vs. hurts:** tracing JIT shines on CPU-bound code (image processing, math, Eloquent's `whereIn(array_fill(0, 10000, 1))` style queries, JSON encode/decode loops). For typical CRUD Laravel apps the JIT gain is usually 2-5% — not worth the 64MB RAM cost on small hosts. Start with `tracing` + 64M on hosts with >1GB RAM per worker; back off to `function` mode or 32M if you don't see meaningful improvement in your blackbox p95.

**When NOT to preload the entire framework:** if you run hundreds of micro-services, each hosting a small Laravel app, the shared memory cost of preloading the full framework per worker is wasteful — switch to `opcache.preload` only on the heavy services (monolith, big API) and let the smaller services rely on plain `opcache.enable=1` + the autoloader.

## FrankenPHP (Laravel Cloud's Underlying Runtime)

FrankenPHP is the modern, single-binary PHP app server built on Caddy — used by [Laravel Cloud](https://cloud.laravel.com) as its primary runtime, and a first-class Laravel Octane driver. It runs PHP as a long-lived worker (no per-request boot), supports HTTP/1.1, HTTP/2, HTTP/3, automatic HTTPS (Let's Encrypt + zero-config), Mercure hub, and Brotli/Zstd compression out of the box.

```bash
# Install (Linux glibc build — avoid Alpine/musl in production per FrankenPHP docs)
curl -fsSL https://github.com/dunglas/frankenphp/releases/latest/download/frankenphp-linux-x86_64 -o /usr/local/bin/frankenphp
chmod +x /usr/local/bin/frankenphp

# Standalone Laravel with built-in Octane driver
composer require laravel/octane
php artisan octane:install --server=frankenphp

# Run via Octane (worker mode, auto-recycles every 500 requests)
php artisan octane:frankenphp --workers=4 --max-requests=500
```

**Caddyfile for a production deploy (zero-config TLS, HTTP/3, Brotli):**
```caddyfile
{
    frankenphp {
        num 4
    }
    order php_server before file_server
}

yourapp.com {
    root * /var/www/html/public
    encode zstd br gzip
    php_server {
        try_files {path} {path}/ /index.php?{query}
    }

    # Split the thread pool for slow endpoints (uploads, reports) so they
    # can't starve the main pool — exactly like an FPM "pm" pool split
    @slow path /api/reports/* /api/exports/*
    route @slow {
        php_server {
            worker /var/www/html/public/index.php 1 {
                num 1
                max_threads 4
            }
        }
    }

    log {
        output file /var/log/caddy/yourapp.log
    }
}
```

**Sizing formula:** `num_threads × memory_limit < available_memory`. With a 256M `memory_limit` and a 4 GB box, that's ~14 max threads. Start with `num = CPU cores` and tune up under load tests (k6, Gatling).

**Why `frankenphp` instead of Swoole or RoadRunner:**

| Aspect | Swoole | RoadRunner | FrankenPHP |
|---|---|---|---|
| TLS / HTTP/3 | manual | manual | built-in (Caddy) |
| Auto HTTPS | ❌ | ❌ | ✅ Let's Encrypt |
| Workers reload on file change | ❌ manual | ⚠️ config-based | ✅ `max-requests` flag |
| Mercure hub | ❌ | ❌ | ✅ built-in |
| Production readiness | high (PHP-China heavy) | high | high (Laravel Cloud bet) |
| Dockerfile base | custom build | custom build | official `dunglas/frankenphp` |
| PHP version | 8.0+ | 8.0+ | 8.3+ |
| Static binary | ❌ | ❌ | ✅ single binary deploy |

**Critical caveat — `LARAVEL_STORAGE_PATH` for embedded binaries:** if you ship a Laravel app as a standalone FrankenPHP binary (the `frankenphp build` / `frankenphp php-cli` flow), each new version is extracted into a different temp dir. Set `LARAVEL_STORAGE_PATH=/var/lib/yourapp/storage` in your environment, or call `Application::useStoragePath()` in `bootstrap/app.php`, so uploaded files, logs, and caches survive upgrades.

**FrankenPHP + Octane gotchas:**
- Static properties and class-level caches leak between requests just like Swoole/RoadRunner — same `Octane::tick()` and `DB::purge()` patterns apply.
- Caddy's automatic HTTPS requires the box to have public DNS + port 80/443 open. If you're behind a load balancer or in a private VPC, set `auto_https off` in the global options and terminate TLS at the LB.
- The official Docker image (`dunglas/frankenphp`) is Debian-based; use it directly for production rather than rolling your own multi-stage build.
- For the `artisan octane:frankenphp` command to work via a standalone binary, the binary MUST be named `frankenphp` and on `$PATH` — Octane shells out to it directly.

## Common Mistakes

1. **`APP_DEBUG=true` in production** — exposes stack traces, credentials
2. **`env()` outside config files** — returns null after `config:cache`
3. **No queue workers running** — jobs sit forever, never process
4. **`route:cache` with closures** — breaks silently, routes fail
5. **Not restarting queue workers** — old code keeps running
6. **Missing `.env` in deployment** — app crashes
7. **Running migrations on every deploy without check** — causes downtime during schema change
8. **No health check after deploy** — silent failures go unnoticed
9. **Octane with singleton request state** — data leaks between requests
10. **Docker without health checks** — containers marked healthy when app is down