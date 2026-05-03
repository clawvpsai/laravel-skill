# Deployment — Production, Env, Configs

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

# 2. Install dependencies
composer install --optimize-autoloader --no-dev

# 3. Run migrations
php artisan migrate --force  # --force bypasses confirmation in production

# 4. Clear and rebuild caches
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache  # if using events

# 5. Restart queues
php artisan queue:restart  # tells workers to reload new code

# 6. If using Octane or Swoole
php artisan octane:reload
```

## Deployment Script Example

```bash
#!/bin/bash
set -e

echo "Deploying..."

cd /var/www/myapp

git pull origin main

composer install --optimize-autoloader --no-dev

php artisan migrate --force

php artisan config:cache
php artisan route:cache
php artisan view:cache

php artisan queue:restart

echo "Deployment complete"
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

## Nginx Config

```nginx
server {
    listen 80;
    server_name yourapp.com;
    return 301 https://yourapp.com$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourapp.com;

    root /var/www/myapp/public;
    index index.php;

    ssl_certificate /etc/letsencrypt/live/yourapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourapp.com/privkey.pem;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

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

## Health Check

```php
// routes/web.php
Route::get('/up', fn() => response()->json(['status' => 'ok', 'time' => now()->toIso8601String()]));
```

**Nginx health check:**
```nginx
location /up {
    access_log off;
    return 200 "ok";
    add_header Content-Type text/plain;
}
```

## Common Mistakes

1. **`APP_DEBUG=true` in production** — exposes stack traces, credentials
2. **`env()` outside config files** — returns null after `config:cache`
3. **No queue workers running** — jobs sit forever, never process
4. **`route:cache` with closures** — breaks silently, routes fail
5. **Not restarting queue workers** — old code keeps running until manual restart
6. **Missing `.env` in deployment** — app crashes, no config