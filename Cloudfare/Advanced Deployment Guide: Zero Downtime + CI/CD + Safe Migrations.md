# 🚀 Advanced Deployment Guide: Zero Downtime + CI/CD + Safe Migrations

## 🎯 Goal
Provide a **production-grade deployment strategy** with:
- Zero downtime deployments
- Automated CI/CD pipeline
- One-command deployment
- Safe database migration practices

---

# 🟦🟩 Zero Downtime Deployment (Blue-Green)

## ✅ Concept
Maintain two environments:
- Blue (current)
- Green (new)

Switch traffic instantly after validation.

---

## ✅ Laravel Example (Symlink Strategy)

### Folder Structure
```
/var/www/
  ├── blue/
  ├── green/
  └── current -> blue
```

### Deploy New Version
```bash
git clone repo /var/www/green
cd /var/www/green
composer install --no-dev --optimize-autoloader
php artisan migrate --force
php artisan config:cache
```

### Switch Release
```bash
ln -sfn /var/www/green /var/www/current
```

### Rollback
```bash
ln -sfn /var/www/blue /var/www/current
```

---

## ✅ Node / Next.js (Blue-Green with Ports)

```bash
pm2 start app.js --name app-blue -- --port=3000
pm2 start app.js --name app-green -- --port=3001
```

Switch Caddy:
```
reverse_proxy localhost:3001
```

---

# 🔁 Rolling Deployment (PM2)

```bash
pm2 reload app
```

Cluster mode:
```bash
pm2 start app.js -i max
pm2 reload app
```

---

# ⚙️ Production Deploy Script (One Command)

## ✅ deploy.sh

```bash
#!/bin/bash
set -e

APP_DIR=/var/www
NEW_DIR=$APP_DIR/green
CURRENT_LINK=$APP_DIR/current

# Pull latest code
rm -rf $NEW_DIR
git clone git@github.com:yourrepo.git $NEW_DIR
cd $NEW_DIR

# Install dependencies
composer install --no-dev --optimize-autoloader
npm install && npm run build || true

# Laravel optimizations
php artisan migrate --force
php artisan config:cache
php artisan route:cache

# Switch symlink
ln -sfn $NEW_DIR $CURRENT_LINK

# Reload services
sudo systemctl reload caddy

# Restart queue (if any)
sudo systemctl restart supervisor || true

echo "Deploy completed successfully ✅"
```

Run with:
```bash
bash deploy.sh
```

---

# 🔄 CI/CD Pipeline (GitHub Actions)

## ✅ .github/workflows/deploy.yml

```yaml
name: Deploy

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Deploy via SSH
      uses: appleboy/ssh-action@v0.1.5
      with:
        host: ${{ secrets.SERVER_HOST }}
        username: ${{ secrets.SERVER_USER }}
        key: ${{ secrets.SSH_PRIVATE_KEY }}
        script: |
          cd /var/www
          bash deploy.sh
```

---

# 🧠 Safe Database Migration Patterns

## ⚠️ Problem
Migrations can break running app if:
- Column removed while code still using it
- Schema changes incompatible

---

## ✅ Safe Migration Strategy

### 1. Backward-Compatible Changes

✅ Safe:
```php
Schema::table('users', function (Blueprint $table) {
    $table->string('new_column')->nullable();
});
```

❌ Unsafe:
```php
$table->dropColumn('old_column');
```

---

### 2. Two-Step Deployment

Step 1:
- Add new column
- Deploy code supporting both

Step 2:
- Remove old column later

---

### 3. Avoid Blocking Queries

```bash
php artisan migrate --force
```

Ensure migrations:
- Fast
- Non-locking

---

### 4. Zero Downtime DB Pattern

| Action | Strategy |
|------|---------|
| Add column | Safe ✅ |
| Rename column | Use duplicate then switch ✅ |
| Drop column | Delay ⏳ |

---

### 5. Feature Flags (Advanced)

Deploy:
- New schema first
- Enable feature later

---

# ✅ Final Deployment Checklist

- [ ] Code deployed to new version
- [ ] DB migrations safe and applied
- [ ] Health check passed
- [ ] Traffic switched
- [ ] Rollback plan ready

---

# 💡 Final Insight

> The key to zero downtime is **reversibility + compatibility**.

- Never break existing schema
- Always deploy forward-compatible code
- Always keep rollback ready

---

✅ This setup ensures safe, automated, and downtime-free deployments.
