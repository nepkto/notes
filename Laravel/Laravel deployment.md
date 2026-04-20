# Laravel Deployment Guide: CI/CD, HTTPS & Automated Backups

A complete production-ready guide to deploy a **Laravel** application on **Ubuntu Server 24.04 LTS** with **GitHub Actions CI/CD**, **Let's Encrypt HTTPS**, **Supervisor-managed queue workers**, and **automated MySQL + file backups**.

> **Assumes** Apache2, PHP-FPM 8.1, and MySQL are already installed (see previous guides).

---

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Install Composer](#2-install-composer)
3. [Install Node.js & npm](#3-install-nodejs--npm)
4. [Prepare Deployment User & Directory](#4-prepare-deployment-user--directory)
5. [Deploy the Laravel Application](#5-deploy-the-laravel-application)
6. [Configure Environment Variables](#6-configure-environment-variables)
7. [Configure Apache Virtual Host for Laravel](#7-configure-apache-virtual-host-for-laravel)
8. [Set Up HTTPS with Let's Encrypt](#8-set-up-https-with-lets-encrypt)
9. [Configure Laravel Scheduler & Queue Worker with Supervisor](#9-configure-laravel-scheduler--queue-worker-with-supervisor)
10. [Set Up GitHub Actions CI/CD](#10-set-up-github-actions-cicd)
11. [Zero-Downtime Deployment Script](#11-zero-downtime-deployment-script)
12. [Configure Automated Backups](#12-configure-automated-backups)
13. [Monitoring & Log Rotation](#13-monitoring--log-rotation)
14. [Security Hardening](#14-security-hardening)
15. [Troubleshooting](#15-troubleshooting)

---

## 1. Prerequisites

- Ubuntu Server 24.04 LTS with Apache2, PHP-FPM 8.1, MySQL installed
- A domain name pointing to your server (`A` record → server IP)
- SSH access with a `sudo` user
- A GitHub repository containing your Laravel app
- Your Laravel app's MySQL database created (e.g., `app_db`, `app_user`)

---

## 2. Install Composer

```bash
cd ~
curl -sS https://getcomposer.org/installer -o /tmp/composer-setup.php
HASH=$(curl -sS https://composer.github.io/installer.sig)
php -r "if (hash_file('sha384', '/tmp/composer-setup.php') === '$HASH') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('/tmp/composer-setup.php'); } echo PHP_EOL;"
sudo php /tmp/composer-setup.php --install-dir=/usr/local/bin --filename=composer
composer --version
```

---

## 3. Install Node.js & npm

For building frontend assets (Vite / Mix):

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v && npm -v
```

---

## 4. Prepare Deployment User & Directory

Create a dedicated deploy user (recommended):

```bash
sudo adduser --disabled-password --gecos "" deploy
sudo usermod -aG www-data deploy
```

Create the web directory:

```bash
sudo mkdir -p /var/www/example.com
sudo chown -R deploy:www-data /var/www/example.com
sudo chmod -R 2775 /var/www/example.com
```

Generate SSH key for the deploy user (used by GitHub Actions):

```bash
sudo -u deploy ssh-keygen -t ed25519 -C "deploy@example.com" -f /home/deploy/.ssh/id_ed25519 -N ""
sudo cat /home/deploy/.ssh/id_ed25519.pub >> /home/deploy/.ssh/authorized_keys
or (in case of permission)
sudo sh -c "cat /home/deploy/.ssh/id_ed25519.pub >> /home/deploy/.ssh/authorized_keys
sudo chown deploy:deploy /home/deploy/.ssh/authorized_keys
sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

Copy the **private key** (`/home/deploy/.ssh/id_ed25519`) — you'll add it to GitHub secrets later.

---

## 5. Deploy the Laravel Application

Clone your repo as the `deploy` user:

```bash
sudo -u deploy git clone https://github.com/your-org/your-laravel-app.git /var/www/example.com/current
cd /var/www/example.com/current
sudo -u deploy composer install --no-dev --optimize-autoloader
sudo -u deploy npm ci && sudo -u deploy npm run build
```

Set proper permissions on storage & cache:

```bash
sudo chown -R deploy:www-data /var/www/example.com/current
sudo find /var/www/example.com/current -type d -exec chmod 2775 {} \;
sudo find /var/www/example.com/current -type f -exec chmod 0664 {} \;
sudo chmod -R ug+rwx /var/www/example.com/current/storage /var/www/example.com/current/bootstrap/cache
```

---

## 6. Configure Environment Variables

```bash
sudo -u deploy cp /var/www/example.com/current/.env.example /var/www/example.com/current/.env
sudo -u deploy nano /var/www/example.com/current/.env
```

Example `.env`:

```ini
APP_NAME=Laravel
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=https://example.com

LOG_CHANNEL=stack
LOG_LEVEL=warning

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=app_db
DB_USERNAME=app_user
DB_PASSWORD=StrongUserPassword!

CACHE_DRIVER=file
QUEUE_CONNECTION=database
SESSION_DRIVER=file
```

Generate application key and run migrations:

```bash
cd /var/www/example.com/current
sudo -u deploy php artisan key:generate
sudo -u deploy php artisan migrate --force
sudo -u deploy php artisan storage:link
sudo -u deploy php artisan config:cache
sudo -u deploy php artisan route:cache
sudo -u deploy php artisan view:cache
```

---

## 7. Configure Apache Virtual Host for Laravel

```bash
sudo nano /etc/apache2/sites-available/example.com.conf
```

```apache
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/example.com/current/public

    <Directory /var/www/example.com/current/public>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.1-fpm.sock|fcgi://localhost"
    </FilesMatch>

    ErrorLog ${APACHE_LOG_DIR}/example.com-error.log
    CustomLog ${APACHE_LOG_DIR}/example.com-access.log combined
</VirtualHost>
```

Enable site and required modules:

```bash
sudo a2ensite example.com.conf
sudo a2enmod rewrite headers proxy_fcgi setenvif
sudo apache2ctl configtest
sudo systemctl reload apache2
```

---

## 8. Set Up HTTPS with Let's Encrypt

Install Certbot (snap method):

```bash
sudo snap install core && sudo snap refresh core
sudo snap install --classic certbot
sudo ln -sf /snap/bin/certbot /usr/bin/certbot
```

Obtain and install certificate:

```bash
sudo certbot --apache -d example.com -d www.example.com --non-interactive --agree-tos -m admin@example.com --redirect
```

Verify automatic renewal:

```bash
sudo certbot renew --dry-run
sudo systemctl list-timers | grep certbot
```

Update `.env` to enforce HTTPS:

```ini
APP_URL=https://example.com
```

In `app/Providers/AppServiceProvider.php`:

```php
use Illuminate\Support\Facades\URL;

public function boot(): void
{
    if (config('app.env') === 'production') {
        URL::forceScheme('https');
    }
}
```

---

## 9. Configure Laravel Scheduler & Queue Worker with Supervisor

### 9.1 Laravel Scheduler (Cron)

```bash
sudo crontab -u deploy -e
```

Add:

```cron
* * * * * cd /var/www/example.com/current && php artisan schedule:run >> /dev/null 2>&1
```

### 9.2 Install Supervisor

[Supervisor](http://supervisord.org/) is a process monitor that will automatically restart your `queue:work` processes if they fail. This is the approach **officially recommended by the Laravel documentation**.

```bash
sudo apt update
sudo apt install -y supervisor
sudo systemctl enable --now supervisor
sudo systemctl status supervisor
```

### 9.3 Create the Supervisor Configuration for Laravel Queue

Configuration files live in `/etc/supervisor/conf.d/`.

```bash
sudo nano /etc/supervisor/conf.d/laravel-worker.conf
```

Paste the following:

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=/usr/bin/php /var/www/example.com/current/artisan queue:work --sleep=3 --tries=3 --max-time=3600 --backoff=5
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=deploy
numprocs=2
redirect_stderr=true
stdout_logfile=/var/log/supervisor/laravel-worker.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=10
stopwaitsecs=3600
```

**Key options explained:**

| Option | Meaning |
|--------|---------|
| `numprocs=2` | Runs 2 parallel worker processes. Increase for heavier workloads. |
| `autorestart=true` | Restarts a worker if it crashes. |
| `user=deploy` | Run workers as the `deploy` user. |
| `--max-time=3600` | Each worker exits after 1h to prevent memory leaks (Supervisor restarts it). |
| `stopwaitsecs=3600` | Give workers up to 1h to finish current job on shutdown (must exceed your longest job). |
| `stopasgroup` / `killasgroup` | Ensures child processes are also terminated. |

### 9.4 (Optional) Separate Workers per Queue

To prioritize queues, define multiple programs. Example — a high-priority notifications queue and a default queue:

```ini
[program:laravel-worker-high]
process_name=%(program_name)s_%(process_num)02d
command=/usr/bin/php /var/www/example.com/current/artisan queue:work --queue=high,default --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
user=deploy
numprocs=3
redirect_stderr=true
stdout_logfile=/var/log/supervisor/laravel-worker-high.log
stopwaitsecs=3600

[program:laravel-worker-default]
process_name=%(program_name)s_%(process_num)02d
command=/usr/bin/php /var/www/example.com/current/artisan queue:work --queue=default --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
user=deploy
numprocs=2
redirect_stderr=true
stdout_logfile=/var/log/supervisor/laravel-worker-default.log
stopwaitsecs=3600
```

### 9.5 Load & Start the Workers

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*
```

Check the status:

```bash
sudo supervisorctl status
```

Expected output:

```
laravel-worker:laravel-worker_00   RUNNING   pid 12345, uptime 0:00:10
laravel-worker:laravel-worker_01   RUNNING   pid 12346, uptime 0:00:10
```

### 9.6 Managing Workers

```bash
# View running programs
sudo supervisorctl status

# Restart workers after a deploy (graceful — lets current jobs finish)
sudo supervisorctl restart laravel-worker:*

# Stop / Start specific group
sudo supervisorctl stop laravel-worker:*
sudo supervisorctl start laravel-worker:*

# Tail logs
sudo tail -f /var/log/supervisor/laravel-worker.log

# Interactive shell
sudo supervisorctl
```

### 9.7 Gracefully Restart Workers After Deploy

After pushing new code, tell existing workers to exit after finishing the current job. Laravel's built-in command handles this cleanly:

```bash
php artisan queue:restart
```

Supervisor will then auto-restart them with the new code. The deploy script below includes this step.

### 9.8 Allow `deploy` User to Control Supervisor (No Password)

```bash
sudo visudo -f /etc/sudoers.d/deploy
```

Ensure it contains:

```
deploy ALL=(ALL) NOPASSWD: /bin/systemctl reload php8.1-fpm, /bin/systemctl reload apache2, /usr/bin/supervisorctl
```

---

## 10. Set Up GitHub Actions CI/CD

### 10.1 Add GitHub Secrets

In your GitHub repo → **Settings → Secrets and variables → Actions**, add:

| Secret | Value |
|--------|-------|
| `SSH_HOST` | Your server IP or domain |
| `SSH_USER` | `deploy` |
| `SSH_PRIVATE_KEY` | Contents of `/home/deploy/.ssh/id_ed25519` |
| `SSH_PORT` | `22` (or custom) |

### 10.2 Create the Workflow

Create `.github/workflows/deploy.yml` in your Laravel repo:

```yaml
name: Laravel CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    name: Test Laravel App
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP 8.1
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
          extensions: mbstring, xml, bcmath, mysql, curl, gd, zip, intl
          coverage: none

      - name: Cache Composer packages
        uses: actions/cache@v4
        with:
          path: vendor
          key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}

      - name: Install Composer dependencies
        run: composer install --no-interaction --prefer-dist --optimize-autoloader

      - name: Copy .env
        run: cp .env.example .env

      - name: Generate app key
        run: php artisan key:generate

      - name: Set up database
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: root
        run: |
          php artisan config:clear
          php artisan migrate --force

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install npm & build
        run: |
          npm ci
          npm run build

      - name: Run tests
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: root
        run: php artisan test

  deploy:
    name: Deploy to Production
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: ${{ secrets.SSH_PORT }}
          script: |
            bash /var/www/example.com/deploy.sh
```

---

## 11. Zero-Downtime Deployment Script

Create `/var/www/example.com/deploy.sh`:

```bash
sudo -u deploy nano /var/www/example.com/deploy.sh
```

```bash
#!/usr/bin/env bash
set -e

APP_DIR="/var/www/example.com/current"
RELEASES_DIR="/var/www/example.com/releases"
SHARED_DIR="/var/www/example.com/shared"
BRANCH="main"
KEEP_RELEASES=5

TIMESTAMP=$(date +%Y%m%d%H%M%S)
NEW_RELEASE="$RELEASES_DIR/$TIMESTAMP"

mkdir -p "$RELEASES_DIR" "$SHARED_DIR/storage" "$SHARED_DIR"

echo "➤ Cloning repo..."
git clone --depth 1 --branch "$BRANCH" https://github.com/your-org/your-laravel-app.git "$NEW_RELEASE"

echo "➤ Linking shared .env and storage..."
ln -sfn "$SHARED_DIR/.env" "$NEW_RELEASE/.env"
rm -rf "$NEW_RELEASE/storage"
ln -sfn "$SHARED_DIR/storage" "$NEW_RELEASE/storage"

echo "➤ Installing dependencies..."
cd "$NEW_RELEASE"
composer install --no-dev --optimize-autoloader --no-interaction
npm ci
npm run build

echo "➤ Running migrations & cache..."
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

echo "➤ Switching symlink..."
ln -sfn "$NEW_RELEASE" "$APP_DIR"

echo "➤ Reloading PHP-FPM..."
sudo systemctl reload php8.1-fpm

echo "➤ Gracefully restarting queue workers..."
php artisan queue:restart
sudo supervisorctl restart laravel-worker:*

echo "➤ Cleaning old releases..."
cd "$RELEASES_DIR"
ls -1dt */ | tail -n +$((KEEP_RELEASES + 1)) | xargs -r rm -rf

echo "✅ Deployment complete: $TIMESTAMP"
```

Make executable and move shared files:

```bash
sudo chmod +x /var/www/example.com/deploy.sh
sudo -u deploy mv /var/www/example.com/current/.env /var/www/example.com/shared/.env
sudo -u deploy cp -r /var/www/example.com/current/storage /var/www/example.com/shared/
```

---

## 12. Configure Automated Backups

### 12.1 MySQL + Files Backup Script

```bash
sudo mkdir -p /var/backups/laravel /etc/laravel-backup
sudo nano /usr/local/bin/laravel-backup.sh
```

```bash
#!/usr/bin/env bash
set -e

# ===== CONFIG =====
APP_DIR="/var/www/example.com/current"
BACKUP_DIR="/var/backups/laravel"
DB_NAME="app_db"
DB_PASS_FILE="/etc/laravel-backup/.my.cnf"
RETENTION_DAYS=14
DATE=$(date +%Y%m%d_%H%M%S)

# Optional: S3 / remote upload
S3_BUCKET=""   # e.g., s3://my-bucket/laravel-backups

mkdir -p "$BACKUP_DIR"

echo "➤ Backing up MySQL database: $DB_NAME"
mysqldump --defaults-extra-file="$DB_PASS_FILE" \
  --single-transaction --quick --lock-tables=false \
  "$DB_NAME" | gzip > "$BACKUP_DIR/db_${DB_NAME}_${DATE}.sql.gz"

echo "➤ Backing up storage directory"
tar -czf "$BACKUP_DIR/storage_${DATE}.tar.gz" -C "$APP_DIR" storage

echo "➤ Backing up .env"
cp "$APP_DIR/.env" "$BACKUP_DIR/env_${DATE}.env"

if [ -n "$S3_BUCKET" ]; then
  echo "➤ Uploading to S3"
  aws s3 cp "$BACKUP_DIR/db_${DB_NAME}_${DATE}.sql.gz" "$S3_BUCKET/"
  aws s3 cp "$BACKUP_DIR/storage_${DATE}.tar.gz" "$S3_BUCKET/"
fi

echo "➤ Cleaning old backups (>$RETENTION_DAYS days)"
find "$BACKUP_DIR" -type f -mtime +$RETENTION_DAYS -delete

echo "✅ Backup complete: $DATE"
```

```bash
sudo chmod +x /usr/local/bin/laravel-backup.sh
```

### 12.2 Create a Read-Only MySQL Backup User

```bash
sudo mysql -u root -p
```

```sql
CREATE USER 'backup_user'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'StrongBackupPass!';
GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER, PROCESS, RELOAD ON *.* TO 'backup_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Store credentials securely:

```bash
sudo nano /etc/laravel-backup/.my.cnf
```

```ini
[client]
user=backup_user
password=StrongBackupPass!
```

```bash
sudo chmod 600 /etc/laravel-backup/.my.cnf
sudo chown root:root /etc/laravel-backup/.my.cnf
```

### 12.3 Schedule Backup with Cron

```bash
sudo crontab -e
```

Add (runs daily at 2 AM):

```cron
0 2 * * * /usr/local/bin/laravel-backup.sh >> /var/log/laravel-backup.log 2>&1
```

### 12.4 (Optional) Use `spatie/laravel-backup` Package

```bash
composer require spatie/laravel-backup
php artisan vendor:publish --provider="Spatie\Backup\BackupServiceProvider"
```

Schedule in `app/Console/Kernel.php`:

```php
protected function schedule(Schedule $schedule): void
{
    $schedule->command('backup:clean')->daily()->at('01:00');
    $schedule->command('backup:run')->daily()->at('02:00');
    $schedule->command('backup:monitor')->daily()->at('03:00');
}
```

---

## 13. Monitoring & Log Rotation

### 13.1 Log Rotation

```bash
sudo nano /etc/logrotate.d/laravel
```

```
/var/www/example.com/shared/storage/logs/*.log
/var/log/supervisor/laravel-worker*.log
/var/log/laravel-backup.log
{
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    copytruncate
    su deploy www-data
}
```

Test:

```bash
sudo logrotate -d /etc/logrotate.d/laravel
```

### 13.2 Health Check Endpoint

Add to `routes/web.php`:

```php
Route::get('/health', fn () => response()->json([
    'status' => 'ok',
    'time'   => now()->toIso8601String(),
]));
```

### 13.3 Supervisor Web UI (Optional)

Enable Supervisor's built-in HTTP dashboard for monitoring workers:

```bash
sudo nano /etc/supervisor/supervisord.conf
```

Uncomment / add:

```ini
[inet_http_server]
port=127.0.0.1:9001
username=admin
password=StrongSupervisorPass!
```

Reload:

```bash
sudo systemctl restart supervisor
```

Access via SSH tunnel: `ssh -L 9001:127.0.0.1:9001 deploy@your_server_ip`

---

## 14. Security Hardening

- **Protect `.env`**: It lives outside the `public/` DocumentRoot — already safe.
- **Disable directory listing**: Use `Options -Indexes`.
- **Set `APP_DEBUG=false`** in production.
- **Firewall**: Allow only `Apache Full`, `OpenSSH`.
- **Fail2Ban**:
  ```bash
  sudo apt install fail2ban -y
  sudo systemctl enable --now fail2ban
  ```
- **Automatic security updates**:
  ```bash
  sudo apt install unattended-upgrades -y
  sudo dpkg-reconfigure --priority=low unattended-upgrades
  ```

---

## 15. Troubleshooting

### Queue worker not processing jobs
```bash
sudo supervisorctl status
sudo tail -f /var/log/supervisor/laravel-worker.log
```

### Supervisor shows `FATAL` or `BACKOFF`
- Check the logfile listed in `stdout_logfile`.
- Common causes: wrong `user`, wrong PHP path, broken `.env`, missing `vendor/` folder.
- Test the command manually:
  ```bash
  sudo -u deploy /usr/bin/php /var/www/example.com/current/artisan queue:work --once
  ```

### Workers don't pick up new code after deploy
```bash
php artisan queue:restart
sudo supervisorctl restart laravel-worker:*
```

### Permission errors on storage/cache
```bash
sudo chown -R deploy:www-data /var/www/example.com/shared/storage
sudo chmod -R ug+rwx /var/www/example.com/shared/storage
```

### GitHub Actions SSH fails
- Verify secret `SSH_PRIVATE_KEY` includes `BEGIN`/`END` lines.
- Confirm key is in `/home/deploy/.ssh/authorized_keys`.
- Test manually: `ssh -i id_ed25519 deploy@your_server_ip`

### 500 error after deploy
```bash
tail -f /var/www/example.com/shared/storage/logs/laravel.log
tail -f /var/log/apache2/example.com-error.log
php artisan optimize:clear
```

### Certbot renewal fails
```bash
sudo certbot renew --dry-run
sudo tail -f /var/log/letsencrypt/letsencrypt.log
```

### Restore from backup
```bash
# Database
gunzip < /var/backups/laravel/db_app_db_20260419_020000.sql.gz | mysql -u root -p app_db

# Storage
tar -xzf /var/backups/laravel/storage_20260419_020000.tar.gz -C /var/www/example.com/shared/
```

---

## Conclusion

You now have a production-grade Laravel deployment with:

- ✅ **Composer + Node.js** build pipeline
- ✅ **Apache2 + PHP-FPM 8.1 + MySQL** integration
- ✅ **HTTPS** via Let's Encrypt with auto-renewal
- ✅ **GitHub Actions CI/CD** with tests & zero-downtime deploy
- ✅ **Symlinked release directory** (easy rollback)
- ✅ **Queue workers managed by Supervisor** (Laravel-recommended)
- ✅ **Automated daily backups** with retention
- ✅ **Log rotation**, **Fail2Ban**, and security hardening

### Quick Rollback

```bash
cd /var/www/example.com/releases
ls -1dt */ | head -n 5                  # list recent releases
sudo -u deploy ln -sfn /var/www/example.com/releases/<PREVIOUS_TIMESTAMP> /var/www/example.com/current
sudo systemctl reload php8.1-fpm
sudo supervisorctl restart laravel-worker:*
```

---

#!/usr/bin/env bash
set -e

# Usage: deploy.sh <branch>   (defaults to main)
BRANCH="${1:-main}"

# Environment-specific paths (override in server-specific copy if needed)
APP_ROOT="$(dirname "$(readlink -f "$0")")"          # e.g., /var/www/example.com
APP_DIR="$APP_ROOT/current"
RELEASES_DIR="$APP_ROOT/releases"
SHARED_DIR="$APP_ROOT/shared"
REPO_URL="https://github.com/your-org/your-laravel-app.git"
KEEP_RELEASES=5

TIMESTAMP=$(date +%Y%m%d%H%M%S)
NEW_RELEASE="$RELEASES_DIR/$TIMESTAMP"

mkdir -p "$RELEASES_DIR" "$SHARED_DIR/storage"

echo "➤ Deploying branch '$BRANCH' to $APP_ROOT"

echo "➤ Cloning repo..."
git clone --depth 1 --branch "$BRANCH" "$REPO_URL" "$NEW_RELEASE"

echo "➤ Linking shared .env and storage..."
ln -sfn "$SHARED_DIR/.env" "$NEW_RELEASE/.env"
rm -rf "$NEW_RELEASE/storage"
ln -sfn "$SHARED_DIR/storage" "$NEW_RELEASE/storage"

echo "➤ Installing dependencies..."
cd "$NEW_RELEASE"
composer install --no-dev --optimize-autoloader --no-interaction
npm ci
npm run build

echo "➤ Running migrations & cache..."
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

echo "➤ Switching symlink..."
ln -sfn "$NEW_RELEASE" "$APP_DIR"

echo "➤ Reloading PHP-FPM..."
sudo systemctl reload php8.1-fpm

echo "➤ Gracefully restarting queue workers..."
php artisan queue:restart
sudo supervisorctl restart laravel-worker:* || true

echo "➤ Cleaning old releases..."
cd "$RELEASES_DIR"
ls -1dt */ | tail -n +$((KEEP_RELEASES + 1)) | xargs -r rm -rf

echo "✅ Deployment complete: branch=$BRANCH release=$TIMESTAMP"