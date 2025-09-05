# Mail Server Setup from Scratch for example.com

This guide walks you through setting up a clean mail server for `example.com` from scratch, including DNS cleanup, Postfix configuration, and DKIM signing. It is tailored for sending mail as `user@example.com` (not subdomain addresses), and includes recommended testing steps and considerations.

---

## 1. DNS Cleanup and Preparation

### **A. Remove Unnecessary Records**

- **Delete any TXT (SPF) records** for subdomains (`mail`, `www`, etc.) unless explicitly needed.
- **Keep only one SPF record for the root domain (@):**
  ```
  @  TXT  "v=spf1 ip4:YOUR_SERVER_IP -all"
  ```
- **DKIM and DMARC** records should be for the root domain.

### **B. Essential DNS Records**

| Name/Host             | Type | Value                                         |
|-----------------------|------|-----------------------------------------------|
| @                     | A    | YOUR_SERVER_IP                                |
| mail                  | A    | YOUR_SERVER_IP                                |
| @                     | MX   | mail.example.com (Priority 10)              |
| @                     | TXT  | v=spf1 ip4:YOUR_SERVER_IP -all                |
| default._domainkey    | TXT  | DKIM public key (see DKIM section below)      |
| _dmarc                | TXT  | v=DMARC1; p=quarantine; rua=mailto:postmaster@example.com |

---

## 2. Server Preparation

### **A. Remove Old Configurations**
```bash
sudo apt-get purge postfix opendkim opendkim-tools
sudo rm -rf /etc/postfix /etc/opendkim*
sudo apt-get update
```

### **B. Install Required Packages**
```bash
sudo apt-get install postfix opendkim opendkim-tools mail-utils
```

---

## 3. Postfix Configuration

### **A. Core Settings (`/etc/postfix/main.cf`)**
```conf
myhostname = mail.example.com
myorigin = /etc/mailname
inet_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
milter_default_action = accept
milter_protocol = 6
```
- Set `/etc/mailname` file contents to:
  ```
  example.com
  ```

### **B. Restart Postfix**
```bash
sudo systemctl restart postfix
```

---

## 4. DKIM Signing Setup

### **A. Generate DKIM Keys**
```bash
sudo mkdir -p /etc/opendkim/keys/example.com
sudo opendkim-genkey -b 2048 -d example.com -D /etc/opendkim/keys/example.com -s default -v
sudo chown -R opendkim:opendkim /etc/opendkim/keys/example.com
```
- `default.private` and `default.txt` will be created.

### **B. Configure OpenDKIM**

**Edit `/etc/opendkim.conf`:**
```conf
Syslog			yes
UMask			002
Canonicalization		relaxed/simple
Mode			sv
SubDomains		no
AutoRestart		yes
AutoRestartRate		10/1h
Background		yes
KeyTable		/etc/opendkim/key.table
SigningTable		/etc/opendkim/signing.table
ExternalIgnoreList	/etc/opendkim/trusted.hosts
InternalHosts		/etc/opendkim/trusted.hosts
Socket			inet:8891@localhost
```

**Create `/etc/opendkim/key.table`:**
```
default._domainkey.example.com example.com:default:/etc/opendkim/keys/example.com/default.private
```

**Create `/etc/opendkim/signing.table`:**
```
*@example.com default._domainkey.example.com
```

**Create `/etc/opendkim/trusted.hosts`:**
```
127.0.0.1
localhost
example.com
mail.example.com
```

### **C. Integrate OpenDKIM with Postfix**

**Check these lines in `/etc/postfix/main.cf`:**
```
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
milter_default_action = accept
milter_protocol = 6
```

### **D. Restart Services**
```bash
sudo systemctl restart opendkim postfix
```

---

## 5. Update DNS with DKIM Public Key

- Copy contents of `/etc/opendkim/keys/example.com/default.txt`
- Add as a TXT record in Hostinger:
  - **Name/Host:** `default._domainkey`
  - **Value:** Paste the DKIM public key

---

## 6. Testing

### **A. Send a Test Email**
```
Send an email as `user@example.com` to [https://www.mail-tester.com](https://www.mail-tester.com)
echo "This is a test email" | mail -s "working now" test-2e64rrsz8@srv1.mail-tester.com
```

### **B. What to Check**
- **SPF:** Should pass for your IP and domain.
- **DKIM:** Should show "signed" and pass.
- **DMARC:** Should show a policy is present.
- **From address:** Should be `user@example.com`.

### **C. Common Pitfalls**
- **DNS propagation:** Changes can take up to 48 hours. Use [MXToolbox](https://mxtoolbox.com/) or Google Dig to check records.
- **Typo in DNS records:** SPF, DKIM, and DMARC must match your domain exactly.
- **Permissions:** DKIM private keys must be readable by the `opendkim` user.

### **D. Troubleshooting Links**
- [Mail-tester.com](https://www.mail-tester.com)
- [MXToolbox DNS Lookup](https://mxtoolbox.com/)
- [Google Admin Toolbox Dig](https://toolbox.googleapps.com/apps/dig/)

---

## 7. Notes and Considerations

- Always use the root domain (`@`) for SPF unless subdomain mail is required.
- Keep your DNS TTL low before making changes for faster propagation.
- Only send mail as `user@example.com` for proper authentication.
- If you change IPs or keys, update DNS and config files accordingly.
- Regularly test with mail-tester.com to ensure ongoing deliverability.

---

**You now have a clean, authenticated mail server for example.com!**