# 🌐 Cloudflare + SSL Troubleshooting Guide

## 🧩 Use Case Overview
- **Environment:** Ubuntu VPS, Nginx/Apache, Cloudflare (Free Plan)
- **Topic:** SSL issues after server migration + Cloudflare setup

---

## 📚 Scenario

A software engineer migrates a website to a new VPS and integrates Cloudflare.

### ✅ Initial State (Working)
- https://example.com
- https://www.example.com
- SSL generated via Certbot

### 🔄 Changes
- VPS migrated (IP changed)
- Cloudflare proxy enabled
- DNS imported
- SSL regenerated

---

## ❗ Problem

| URL | Result |
|-----|-------|
| https://www.example.com | ✅ Works |
| https://example.com | ❌ Falls back to HTTP |

---

## ❓ Key Question

Why does **www work but root domain fails** after Cloudflare + SSL changes?

---

## 🔍 Architecture Diagram

```
Browser
   ↓
Cloudflare (SSL Edge)
   ↓
Origin Server (VPS)
```

### SSL Layers
1. Browser ↔ Cloudflare
2. Cloudflare ↔ VPS

---

## 🔍 Root Causes

| Area | Issue | Effect |
|------|------|------|
| SSL Mode | Flexible | HTTPS breaks |
| Certificate | Missing root domain | example.com fails |
| DNS | Wrong config | mismatch |
| Redirect | Only www handled | HTTP fallback |

---

## 🛠 Fix Steps

### ✅ 1. Cloudflare Settings

Set SSL Mode:

```
Full (Strict)
```

Enable:

```
Always Use HTTPS
Automatic HTTPS Rewrites
```

---

### ✅ 2. Verify SSL Certificate

```
sudo certbot certificates
```

If missing root domain:

```
sudo certbot --nginx -d example.com -d www.example.com
```

---

### ✅ 3. DNS Setup

```
A   @     → SERVER_IP   (proxied ✅)
A   www   → SERVER_IP   (proxied ✅)
```

---

## ⚙️ Server Configuration

### ✅ Nginx

```
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
}
```

---

### ✅ Apache

```
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    Redirect permanent / https://example.com/
</VirtualHost>

<VirtualHost *:443>
    ServerName example.com
    ServerAlias www.example.com

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem
</VirtualHost>
```

---

## 🔄 Flow Diagrams

### ❌ Incorrect (Flexible)

```
Browser (HTTPS)
   ↓
Cloudflare
   ↓ (HTTP)
Server → Redirect issue
```

### ✅ Correct (Strict)

```
Browser (HTTPS)
   ↓
Cloudflare (HTTPS)
   ↓
Server (HTTPS valid cert)
```

---

## 🎤 Interview Q&A

**Q: Why www works but root fails?**  
A: Certificate or DNS mismatch.

**Q: Problem with Flexible SSL?**  
A: HTTPS → HTTP downgrade.

**Q: Best SSL mode?**  
A: Full (Strict).

**Q: Why disable proxy for Certbot?**  
A: Required for validation.

---

## 🧠 Key Takeaways

- SSL works in layers with Cloudflare
- Always include both domains in certificate
- DNS + SSL + redirect must align
- Use Full (Strict)

---

✅ Useful for DevOps troubleshooting and interviews
