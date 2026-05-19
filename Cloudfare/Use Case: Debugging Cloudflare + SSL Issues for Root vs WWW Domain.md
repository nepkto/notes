# 🌐 Use Case: Debugging Cloudflare + SSL Issues for Root vs WWW Domain

## 🧩 Scenario
An engineer hosts a website on a VPS using **Ubuntu + Nginx + Certbot SSL**.

### Initial Setup (Working ✅)
- Server hosted on VPS
- SSL generated using Certbot
- Website works for:
  - https://example.com
  - https://www.example.com

---

## 🔄 Change Introduced
After one year, the engineer:
1. Moves server to a new VPS
2. Uses **Cloudflare (Free plan)**
3. Imports DNS records
4. Enables proxy (orange cloud)
5. Regenerates SSL using Certbot (after disabling proxy temporarily)

---

## ❗ Problem Statement
After setup:

- ✅ https://www.example.com → Works fine
- ❌ https://example.com → Redirects to HTTP and shows **Not Secure**

---

## ❓ Key Question

> Why does `www.example.com` work with HTTPS but `example.com` fails after moving to Cloudflare and regenerating SSL?

---

# 🔍 Investigation & Troubleshooting

## ✅ Step 1: Understand Cloudflare SSL Flow

```text
Browser → Cloudflare → VPS
```

There are **two SSL layers**:

1. Browser ↔ Cloudflare (handled by Cloudflare)
2. Cloudflare ↔ Server (depends on SSL mode + server cert)

---

## ✅ Step 2: Check Cloudflare SSL Mode

### Possible issue:
- SSL Mode = Flexible ❌

### Effect:
- Browser uses HTTPS
- Cloudflare connects via HTTP to server
- Causes redirect loops or insecure fallback

✅ Fix:

```
Cloudflare → SSL/TLS → Full (Strict)
```

---

## ✅ Step 3: Verify SSL Certificate Coverage

Run:

```bash
sudo certbot certificates
```

### Possible issue:
The certificate only includes:

```
www.example.com
```

But NOT:

```
example.com
```

### Effect:
- www works ✅
- root domain fails ❌

✅ Fix:

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

---

## ✅ Step 4: Check DNS Configuration in Cloudflare

Ensure both records exist:

```
A   @     → SERVER_IP   ✅ proxied
A   www   → SERVER_IP   ✅ proxied
```

### Possible issue:
- Root domain not proxied
- Misconfigured IP

---

## ✅ Step 5: Enforce HTTPS at Cloudflare Level

Go to:

Cloudflare → SSL/TLS → Edge Certificates

Enable:

```
✅ Always Use HTTPS
✅ Automatic HTTPS Rewrites
```

---

## ✅ Step 6: Check Server Redirect Configuration

### Nginx Example:

```nginx
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

### Possible issue:
- Redirect configured only for `www`
- Root domain missing

---

## ✅ Step 7: Clear Cache

### Browser:
- Hard refresh (Ctrl + Shift + R)
- Or use Incognito

### Cloudflare:

```
Caching → Purge Everything
```

---

# 🎯 Root Cause Summary

| Issue | Effect |
|------|------|
| Missing root domain in SSL cert | example.com fails |
| SSL mode = Flexible | HTTPS breaks |
| DNS misconfiguration | Domain mismatch |
| Missing redirect rules | HTTP fallback |

---

# ✅ Final Resolution

✔ Regenerate SSL including both domains  
✔ Set Cloudflare SSL mode to **Full (Strict)**  
✔ Ensure DNS entries are proxied  
✔ Enable HTTPS enforcement in Cloudflare  
✔ Fix server redirect rules  

---

# 💡 Key Learning

> When using Cloudflare, SSL is **not just your server configuration** — it is a **combined system** of:

- DNS
- Proxy behavior
- SSL mode
- Certificate coverage
- Server redirects

Misalignment in any one layer can break HTTPS.

---

✅ This scenario helps understand real-world debugging of HTTPS issues when integrating Cloudflare with a custom VPS setup.