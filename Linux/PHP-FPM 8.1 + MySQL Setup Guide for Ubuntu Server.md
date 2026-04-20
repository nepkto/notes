# Apache2 Setup Guide on Ubuntu Server 24.04 LTS

A complete step-by-step guide to install, configure, and secure Apache2 web server on a newly created Ubuntu Server 24.04 LTS.

---

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Update the System](#2-update-the-system)
3. [Install Apache2](#3-install-apache2)
4. [Manage the Apache2 Service](#4-manage-the-apache2-service)
5. [Configure the Firewall (UFW)](#5-configure-the-firewall-ufw)
6. [Verify Installation](#6-verify-installation)
7. [Understand Apache Directory Structure](#7-understand-apache-directory-structure)
8. [Set Up Virtual Hosts](#8-set-up-virtual-hosts)
9. [Enable Essential Apache Modules](#9-enable-essential-apache-modules)
10. [Secure Apache with HTTPS (Let's Encrypt)](#10-secure-apache-with-https-lets-encrypt)
11. [Harden Apache Security](#11-harden-apache-security)
12. [Common Troubleshooting](#12-common-troubleshooting)
13. [Useful Commands Cheat Sheet](#13-useful-commands-cheat-sheet)

---

## 1. Prerequisites

Before you begin, ensure you have:

- A fresh **Ubuntu Server 24.04 LTS** installation
- A non-root user with `sudo` privileges
- SSH access to the server
- A registered domain name (optional, required for HTTPS)
- Ports **80** (HTTP) and **443** (HTTPS) available

Log in to your server:

```bash
ssh your_user@your_server_ip
```

---

## 2. Update the System

Always start by updating the package index and upgrading installed packages:

```bash
sudo apt update && sudo apt upgrade -y
```

Reboot if the kernel was upgraded:

```bash
sudo reboot
```

---

## 3. Install Apache2

Install Apache2 from the default Ubuntu repositories:

```bash
sudo apt install apache2 -y
```

Verify the version:

```bash
apache2 -v
```

Expected output (example):
```
Server version: Apache/2.4.58 (Ubuntu)
```

---

## 4. Manage the Apache2 Service

Apache2 starts automatically after installation. Common service commands:

```bash
# Check status
sudo systemctl status apache2

# Start the service
sudo systemctl start apache2

# Stop the service
sudo systemctl stop apache2

# Restart the service
sudo systemctl restart apache2

# Reload config without downtime
sudo systemctl reload apache2

# Enable on boot
sudo systemctl enable apache2

# Disable on boot
sudo systemctl disable apache2
```

---

## 5. Configure the Firewall (UFW)

Ubuntu 24.04 ships with UFW (Uncomplicated Firewall). Apache registers application profiles:

```bash
sudo ufw app list
```

You'll see:
```
Apache
Apache Full
Apache Secure
```

| Profile | Ports |
|---------|-------|
| Apache | 80 (HTTP) |
| Apache Secure | 443 (HTTPS) |
| Apache Full | 80 and 443 |

Allow HTTP and HTTPS traffic, and enable UFW:

```bash
sudo ufw allow 'Apache Full'
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

---

## 6. Verify Installation

Get your server's public IP:

```bash
curl -4 ifconfig.me
```

Open a web browser and navigate to:

```
http://your_server_ip
```

You should see the **"Apache2 Ubuntu Default Page"**.

---

## 7. Understand Apache Directory Structure

| Path | Purpose |
|------|---------|
| `/etc/apache2/` | Main configuration directory |
| `/etc/apache2/apache2.conf` | Main configuration file |
| `/etc/apache2/ports.conf` | Listening ports configuration |
| `/etc/apache2/sites-available/` | Available virtual host configs |
| `/etc/apache2/sites-enabled/` | Enabled virtual host configs (symlinks) |
| `/etc/apache2/mods-available/` | Available modules |
| `/etc/apache2/mods-enabled/` | Enabled modules |
| `/var/www/html/` | Default web root directory |
| `/var/log/apache2/access.log` | Access log |
| `/var/log/apache2/error.log` | Error log |

---

## 8. Set Up Virtual Hosts

Virtual hosts let you host multiple sites on one server. Replace `example.com` with your domain.

### 8.1 Create the Document Root

```bash
sudo mkdir -p /var/www/example.com/html
sudo chown -R $USER:$USER /var/www/example.com/html
sudo chmod -R 755 /var/www/example.com
```

### 8.2 Create a Sample Index Page

```bash
nano /var/www/example.com/html/index.html
```

Paste:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Welcome to example.com!</title>
  </head>
  <body>
    <h1>Success! The example.com virtual host is working!</h1>
  </body>
</html>
```

### 8.3 Create the Virtual Host Config

```bash
sudo nano /etc/apache2/sites-available/example.com.conf
```

Add:

```apache
<VirtualHost *:80>
    ServerAdmin admin@example.com
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/example.com/html

    <Directory /var/www/example.com/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/example.com-error.log
    CustomLog ${APACHE_LOG_DIR}/example.com-access.log combined
</VirtualHost>
```

### 8.4 Enable the Site and Disable the Default

```bash
sudo a2ensite example.com.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Expected configtest output: `Syntax OK`

---

## 9. Enable Essential Apache Modules

Enable commonly needed modules:

```bash
sudo a2enmod rewrite
sudo a2enmod headers
sudo a2enmod ssl
sudo a2enmod http2
sudo systemctl restart apache2
```

List enabled modules:

```bash
apache2ctl -M
```

---

## 10. Secure Apache with HTTPS (Let's Encrypt)

Install Certbot via `snap` (recommended on Ubuntu 24.04):

```bash
sudo apt install snapd -y
sudo snap install core && sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

Obtain and install an SSL certificate:

```bash
sudo certbot --apache -d example.com -d www.example.com
```

Follow the prompts. Certbot will automatically configure HTTPS redirection.

Test automatic renewal:

```bash
sudo certbot renew --dry-run
```

Renewals run automatically via a systemd timer. View the timer:

```bash
sudo systemctl list-timers | grep certbot
```

---

## 11. Harden Apache Security

### 11.1 Hide Apache Version Info

Edit:

```bash
sudo nano /etc/apache2/conf-available/security.conf
```

Set:

```apache
ServerTokens Prod
ServerSignature Off
TraceEnable Off
```

### 11.2 Disable Directory Listing

In your virtual host, change:

```apache
Options Indexes FollowSymLinks
```

to:

```apache
Options -Indexes +FollowSymLinks
```

### 11.3 Add Security Headers

Create a headers file:

```bash
sudo nano /etc/apache2/conf-available/security-headers.conf
```

Add:

```apache
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "SAMEORIGIN"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"
```

Enable and reload:

```bash
sudo a2enconf security-headers
sudo systemctl reload apache2
```

### 11.4 Install Fail2Ban (Optional)

```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
```

---

## 12. Common Troubleshooting

### Check Apache status and errors

```bash
sudo systemctl status apache2
sudo journalctl -xeu apache2
sudo tail -f /var/log/apache2/error.log
```

### Test configuration syntax

```bash
sudo apache2ctl configtest
```

### Port already in use

```bash
sudo ss -tulpn | grep ':80'
```

### Permissions issues

Ensure the web root is owned correctly:

```bash
sudo chown -R www-data:www-data /var/www/example.com/html
sudo find /var/www/example.com -type d -exec chmod 755 {} \;
sudo find /var/www/example.com -type f -exec chmod 644 {} \;
```

---

## 13. Useful Commands Cheat Sheet

```bash
# Service control
sudo systemctl {start|stop|restart|reload|status} apache2

# Site management
sudo a2ensite <site>.conf          # Enable a site
sudo a2dissite <site>.conf         # Disable a site

# Module management
sudo a2enmod <module>              # Enable a module
sudo a2dismod <module>             # Disable a module

# Config management
sudo a2enconf <conf>               # Enable a config snippet
sudo a2disconf <conf>              # Disable a config snippet

# Testing
sudo apache2ctl configtest         # Check syntax
apache2ctl -M                      # List loaded modules
apache2ctl -S                      # List virtual hosts

# Logs
sudo tail -f /var/log/apache2/access.log
sudo tail -f /var/log/apache2/error.log
```

---

## Conclusion

You now have a fully functional, secure Apache2 web server on Ubuntu 24.04 LTS with:
- ✅ Apache2 installed and running
- ✅ Firewall configured
- ✅ Virtual hosts set up
- ✅ HTTPS via Let's Encrypt
- ✅ Basic security hardening

**Next steps:** Install PHP, MySQL/MariaDB (to create a LAMP stack), or deploy your web application.

---