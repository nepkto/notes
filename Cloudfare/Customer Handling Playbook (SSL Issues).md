

---

# 👥 Customer Handling Playbook (SSL Issues)

## 🎯 Goal
Resolve SSL/domain issues **without requiring any action from end users**.

---

## ✅ Guiding Principle

> Always assume the issue is on the system side and fix it centrally.

---

## 🛠 Internal Troubleshooting Steps (Do NOT involve user)

### 1. Verify Issue Across Environments

```bash
curl -I https://example.com
curl -I https://www.example.com
```

- Test from different networks / machines
- Use monitoring or SSL checker tools

---

### 2. Fix Root Cause (Not Symptoms)

#### ✅ Cloudflare
- Set SSL mode: **Full (Strict)**
- Enable:
  - Always Use HTTPS
  - Automatic HTTPS Rewrites

#### ✅ Certificate Coverage

```bash
sudo certbot -d example.com -d www.example.com
```

#### ✅ DNS
Ensure:

```
A   @     → SERVER_IP   (proxied ✅)
A   www   → SERVER_IP   (proxied ✅)
```

#### ✅ Redirect Fix

**Nginx**
```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}
```

**Apache**
```apache
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    Redirect permanent / https://example.com/
</VirtualHost>
```

---

### 3. Clear Platform Cache (NOT user cache)

- Cloudflare → *Caching → Purge Everything*

---

## 🗣️ Recommended User Communication

### ✅ Correct Response

> "We identified a configuration issue related to SSL/HTTPS and have now resolved it. Please try accessing the site again — no action is required from your side."

---

### ⚠️ If propagation delay is possible

> "The issue has been fixed. In rare cases, it may take a short time to reflect globally due to network propagation."

---

## ❌ What NOT to Say

- "Clear your browser cache"
- "Try incognito mode"
- "It works on my machine"

---

## 🧠 Best Practices

- Fix issues at **Cloudflare or server level**
- Ensure **end-to-end HTTPS consistency**
- Avoid pushing debugging responsibility to users
- Validate fix using **independent tools**

---

## 💡 Pro Insight

> If SSL and redirect configuration are correct, users should never experience inconsistencies.

---

