# 🚀 Production Setup Guide: Laravel / Node / Next.js + Caddy + Cloudflare

## 🎯 Goal
Set up a modern production-ready architecture using:

- ✅ Cloudflare (DNS + CDN + Edge SSL)
- ✅ Caddy (Reverse proxy + Auto HTTPS)
- ✅ App Layer (Laravel / Node.js / Next.js)

---

# 🧠 Architecture Overview

```
User Browser
     ↓ HTTPS
Cloudflare (CDN + SSL Edge)
     ↓ HTTPS
Caddy (Reverse Proxy + SSL)
     ↓
Application Layer
     ↓
Database / Cache
```

---

# 🌐 Cloudflare Setup

## ✅ DNS Configuration

```
A   @     → SERVER_IP   (✅ proxied)
A   www   → SERVER_IP   (✅ proxied)
```

## ✅ SSL Settings

- SSL Mode: **Full (Strict)**
- Enable:
  - Always Use HTTPS
  - Automatic HTTPS Rewrites

---

# ⚡ Caddy Installation

```bash
sudo apt update
sudo apt install caddy
```

---

# 🟢 Laravel Production Setup

## ✅ Install PHP-FPM

```bash
sudo apt install php8.2-fpm php8.2-mysql
```

## ✅ Project Structure

```
/var/www/laravel
```

## ✅ Caddyfile

```
example.com, www.example.com {
    root * /var/www/laravel/public
    php_fastcgi unix//run/php/php8.2-fpm.sock
    file_server
}
```

## ✅ Flow

```
User → Cloudflare → Caddy → PHP-FPM → Laravel
```

---

# 🟢 Node.js Production Setup

## ✅ Run App (PM2 Recommended)

```bash
npm install -g pm2
pm2 start app.js --name myapp
pm2 startup
pm2 save
```

## ✅ Runs on

```
localhost:3000
```

## ✅ Caddyfile

```
example-node.com {
    reverse_proxy localhost:3000
}
```

## ✅ Flow

```
User → Cloudflare → Caddy → Node App
```

---

# 🟢 Next.js Production Setup

## ✅ Build & Start

```bash
npm run build
npm start
```

(usually runs on port 3000)

## ✅ Or Use PM2

```bash
pm2 start npm --name nextjs -- start
```

## ✅ Caddyfile

```
example-next.com {
    reverse_proxy localhost:3000
}
```

---

# 🔄 Multi-App / Multi-Domain Setup

```
example.com, www.example.com {
    root * /var/www/laravel/public
    php_fastcgi unix//run/php/php8.2-fpm.sock
    file_server
}

api.example.com {
    reverse_proxy localhost:4000
}

app.example.com {
    reverse_proxy localhost:3000
}
```

✅ Each domain handled independently
✅ SSL auto-managed per domain

---

# 🔐 Security Best Practices

Add headers in Caddy:

```
header {
    Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
    X-Frame-Options DENY
    X-Content-Type-Options nosniff
}
```

---

# ⚠️ Common Mistakes

| Issue | Fix |
|------|-----|
| Using Flexible SSL | Use Full (Strict) |
| Missing root domain | Add @ record |
| Certbot with Caddy | Not needed |
| Using artisan serve | Use PHP-FPM |

---

# ✅ Deployment Checklist

- [ ] DNS configured in Cloudflare
- [ ] SSL mode set to Full (Strict)
- [ ] Caddy running
- [ ] App running locally
- [ ] Ports 80/443 open

---

# 💡 Final Insight

> Cloudflare handles edge traffic, Caddy handles application routing and SSL, and your app handles business logic.

---

✅ This setup is production-ready and scales well for modern applications.