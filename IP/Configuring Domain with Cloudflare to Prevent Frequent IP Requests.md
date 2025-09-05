# Configuring Domain with Cloudflare to Prevent Frequent IP Requests

This guide explains how to point your domain to Cloudflare and configure it to automatically prevent requests from frequently changing or abusive IPs.

---

## 1. Point Your Domain to Cloudflare

1. **Sign Up / Log In**  
   - Go to [Cloudflare](https://www.cloudflare.com/).  
   - Create an account or log in.

2. **Add Your Domain**  
   - In the dashboard, click **Add Site**.  
   - Enter your domain name and click **Add Site**.  

3. **Select a Plan**  
   - Choose a plan (Free, Pro, Business, or Enterprise).  
   - The **Free plan** is enough for basic protection.  

4. **Update DNS**  
   - Cloudflare will scan your DNS records.  
   - Verify them and ensure critical records (`A`, `CNAME`, `MX`) are **proxied** (orange cloud icon).  
   - Update your domain’s **nameservers** at your registrar to Cloudflare’s nameservers.  
   - Wait for DNS propagation (up to 24 hours).  

---

## 2. Enable Security Features

- **Rate Limiting**  
  Configure request limits to prevent abuse:  
  - Example: Allow 60 requests per minute per IP.  
  - Navigate to **Security → Bots → Rate Limiting Rules**.  

- **DDoS Protection**  
  Ensure DDoS protection is enabled under **Security → DDoS**.  

- **Web Application Firewall (WAF)**  
  - Enable WAF under **Security → WAF**.  
  - Apply managed rules to block common attack patterns.  

- **IP Access Rules**  
  - Go to **Security → Tools**.  
  - Block, challenge, or whitelist IP ranges or geolocations.  

---

## 3. Configure Bot Management

- Enable **Bot Fight Mode** under **Security → Bots**.  
- On Pro or higher plans, use **Super Bot Fight Mode** for advanced bot detection and mitigation.  

---

## 4. Create Firewall Rules

Set custom firewall rules under **Security → Firewall Rules**:  

- Block requests from suspicious user agents.  
- Challenge or block requests from certain countries.  
- Rate-limit endpoints (e.g., login, search).  

---

## 5. Automatic IP Blocking for Frequent Requests

Use **Rate Limiting** to block abusive IPs automatically:  

- Example Rule:  
  - If an IP exceeds **100 requests in 10 seconds**, block it for **30 minutes**.  
- Steps:  
  1. Go to **Rules → Rate Limiting**.  
  2. Create a new rule with thresholds and action (Block or Challenge).  

---

## 6. Monitor and Adjust

- Check **Analytics** and **Firewall Events** to track blocked requests.  
- Fine-tune rules regularly to balance security and user experience.  

---

## ✅ Summary

By pointing your domain to Cloudflare and enabling its **DDoS Protection, WAF, Rate Limiting, and Bot Management**, you can:  

- Protect against abusive traffic.  
- Automatically block frequently changing IP requests.  
- Keep your website secure while allowing legitimate visitors.  
