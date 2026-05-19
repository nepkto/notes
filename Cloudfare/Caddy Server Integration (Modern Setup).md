---

# ⚡ Caddy Server Integration (Modern Setup)

## ✅ Why Use Caddy?

Caddy simplifies SSL management compared to Nginx/Apache:

- Automatic HTTPS (no Certbot needed)
- Auto-renewal of certificates
- Built-in HTTP → HTTPS redirect
- Minimal configuration

---

## 🔧 Basic Caddy Setup

### Example Caddyfile

```
example.com, www.example.com {
    reverse_proxy localhost:3000
}
```

✅ This automatically:
- Issues SSL for both domains
- Redirects HTTP → HTTPS
- Keeps certificate renewed

---

## 🌐 Architecture with Cloudflare + Caddy

```
Browser
   ↓ HTTPS
Cloudflare (SSL Edge)
   ↓ HTTPS
Caddy (Auto SSL)
   ↓
Application
```

---

## ⚠️ Important: Cloudflare Interaction

Even with Caddy, Cloudflare still controls part of SSL.

### ✅ Required Settings

- SSL Mode: **Full (Strict)**
- DNS:
```
A  @   → SERVER_IP   (proxied ✅)
A  www → SERVER_IP   (proxied ✅)
```

- Enable:
```
Always Use HTTPS
```

---

## ⚠️ SSL Generation Issue (Common)

If Cloudflare proxy is enabled:
- Caddy may fail to generate SSL (HTTP challenge blocked)

### ✅ Solution Options

#### Option 1: Temporary Disable Proxy
1. Set DNS to *DNS Only* (grey cloud)
2. Start Caddy (generate SSL)
3. Re-enable proxy

#### Option 2: DNS Challenge (Recommended)

Use Cloudflare API:

```
{
    acme_dns cloudflare {env.CLOUDFLARE_API_TOKEN}
}

example.com, www.example.com {
    reverse_proxy localhost:3000
}
```

✅ Works without disabling proxy

---

## ⚠️ Common Mistakes with Caddy

| Issue | Impact |
|------|------|
| Only configuring www | root domain fails |
| Cloudflare Flexible SSL | HTTPS breaks |
| Missing DNS proxy | SSL mismatch |

---

## ✅ Key Takeaways

- Caddy removes manual SSL complexity
- Still requires correct Cloudflare setup
- Always include both domains in config
- Prefer DNS challenge for production setups

---
