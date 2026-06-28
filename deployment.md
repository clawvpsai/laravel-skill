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