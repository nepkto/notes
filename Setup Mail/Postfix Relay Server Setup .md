# Postfix Relay Server Setup for example.com

This guide will walk you through setting up Postfix as a relay (smart host) server for `example.com`, including DNS configuration, authentication, and DKIM signing. This is ideal if you want your server to send mail through an upstream SMTP relay (e.g., SendGrid, Gmail, ISP relay, etc.), while maintaining proper authentication and deliverability.

---

## 1. DNS Preparation and Cleanup

### **A. Clean Unnecessary Records**
- **SPF:** Only one record for root domain (`@`):
  ```
  @  TXT  "v=spf1 ip4:YOUR_SERVER_IP -all"
  ```
- **MX:** Points to your relay (if you receive mail) or your real mailbox provider.
- **A:** `mail.example.com` points to your server’s IP.
- **DKIM:** TXT for root domain.
- **DMARC:** TXT for root domain.

---

## 2. Postfix Installation

```bash
sudo apt-get update
sudo apt-get install postfix opendkim opendkim-tools
```

> When prompted for "General type of mail configuration", select **"Satellite system"** or **"Internet Site"** if you plan to relay later.

---

## 3. Postfix Relay Configuration

### **A. Set Relayhost in `/etc/postfix/main.cf`**

```conf
myhostname = mail.example.com
myorigin = /etc/mailname
relayhost = [RELAY_SERVER]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = may
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
```
- Replace `[RELAY_SERVER]:587` with your upstream SMTP relay, e.g. `[smtp.sendgrid.net]:587` or `[smtp.gmail.com]:587`
- Set `/etc/mailname` to:
  ```
  example.com
  ```

### **B. Add SASL Credentials**

**Create `/etc/postfix/sasl_passwd`:**
```
[RELAY_SERVER]:587 USERNAME:PASSWORD
```
- Replace with your relay’s login and password.

**Secure and hash the file:**
```bash
sudo postmap /etc/postfix/sasl_passwd
sudo chown root:root /etc/postfix/sasl_passwd*
sudo chmod 600 /etc/postfix/sasl_passwd*
```

---

## 4. DKIM Signing (Optional but Recommended)

**Follow the same DKIM steps as the main server setup:**

- Generate DKIM keys.
- Add public key as TXT record (`default._domainkey`) in Hostinger.
- Configure `/etc/opendkim.conf`, `/etc/opendkim/key.table`, `/etc/opendkim/signing.table`, and `/etc/opendkim/trusted.hosts` as per previous guide.
- Integrate with Postfix via `smtpd_milters = inet:localhost:8891`.

---

## 5. Restart Services

```bash
sudo systemctl restart opendkim postfix
```

---

## 6. Testing

### **A. Send a Test Email**
Send an email as `user@example.com` to [https://www.mail-tester.com](https://www.mail-tester.com) or your own mailbox.

### **B. What to Check**
- **SPF:** Should pass for your domain and server IP.
- **DKIM:** Should show "signed" and pass.
- **Relay:** Headers should show mail passed through your relay host.
- **From address:** Should be `user@example.com`.

### **C. Useful Testing Tools**
- [Mail-tester.com](https://www.mail-tester.com)
- [MXToolbox](https://mxtoolbox.com/)
- [Google Admin Toolbox Dig](https://toolbox.googleapps.com/apps/dig/)

---

## 7. Notes and Considerations

- **DNS changes take time to propagate (up to 24–48 hours).**
- Your relay host may require you to verify your domain and DKIM.
- If you change relay credentials, update `sasl_passwd` and re-run `postmap`.
- Always test with mail-tester.com for deliverability and authentication.
- For higher security, use TLS for relaying (port 587) and strong relay credentials.

---

**You now have a Postfix relay server ready for authenticated, DKIM-signed mail through your chosen SMTP relay!**