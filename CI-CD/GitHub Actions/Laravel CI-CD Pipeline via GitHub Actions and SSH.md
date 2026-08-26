# Laravel CI/CD Pipeline via GitHub Actions & SSH Deployment

This guide covers robust, secure deployment for Laravel applications using automated testing with GitHub Actions, SSH best practices, and correct Linux file permissions. Troubleshooting and full procedures for SSH authentication, permissions, and CI/CD pipeline setup are included.

---

## Table of Contents

- [Overview](#overview)
- [SSH Setup & GitHub Authentication](#ssh-setup--github-authentication)
- [SSH Directory Structure & Description](#ssh-directory-structure--description)
- [Cloning and Ownership Issues](#cloning-and-ownership-issues)
- [Laravel Directory Permissions](#laravel-directory-permissions)
- [Linux Permissions Side-By-Side Explanation](#linux-permissions-side-by-side-explanation)
- [Best Practice for CI/CD Laravel Deployment](#best-practice-for-cicd-laravel-deployment)
- [CI/CD SSH Key Setup for GitHub Actions](#cicd-ssh-key-setup-for-github-actions)
- [GitHub Actions Workflow](#github-actions-workflow)
- [Enhanced & Fixed Deployment Steps](#enhanced--fixed-deployment-steps)
- [Troubleshooting & FAQ](#troubleshooting--faq)
- [References](#references)

---

## Overview

This repository helps you automate secure Laravel deployments by integrating:
- **GitHub Actions** for CI/CD workflows
- **SSH key authentication** for cloning, server login, and pipeline deployment
- **Linux permissions** for safe, shared editing and web server access

---

## SSH Setup & GitHub Authentication

1. **Generate SSH Key**
    ```sh
    ssh-keygen -t ed25519 -C "your_email@example.com"
    eval "$(ssh-agent -s)"
    ssh-add ~/.ssh/id_ed25519
    cat ~/.ssh/id_ed25519.pub
    ```
2. **Add SSH Key to GitHub**
    - Go to: **GitHub → Settings → SSH and GPG Keys → New SSH Key**
    - Paste the public key and save.

3. **Test SSH Connection**
    ```sh
    ssh -T git@github.com
    ```
    If successful:
    ```
    Hi username! You've successfully authenticated…
    ```

4. **Clone your repository**
    ```sh
    git clone git@github.com:your_user/your_repo.git
    ```

---

## SSH Directory Structure & Description

```
~/.ssh/
├── id_rsa              # Private RSA key (600): Used for SSH authentication; must be kept secret.
├── id_rsa.pub          # Public RSA key (644): Share this with remote services/servers.
├── id_ed25519          # Private Ed25519 key (600): More secure and faster; also private and secret.
├── id_ed25519.pub      # Public Ed25519 key (644): Used for authentication like above.
├── known_hosts         # Trusted remote hosts (644): Contains fingerprints of hosts you've connected to.
├── authorized_keys     # Allowed public keys (600): Server-side; controls which keys/users may log in.
├── config              # SSH client config (600/644): Custom rules for different hosts/keys.
└── control/            # Connection multiplexing sockets: Reuse SSH sessions for speed.
```

---

## Cloning and Ownership Issues

**Problems:**
- `sudo git clone` uses root’s SSH keys, not your own.
- Cloning non-root in `/var/www` might fail due to lack of write permissions.

**Solutions:**
- Clone as your deploy user!  
- Set permissions:
    ```sh
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/id_rsa ~/.ssh/id_ed25519
    sudo usermod -aG www-data your_deploy_user
    sudo chown -R your_deploy_user:www-data /var/www
    sudo chmod -R 775 /var/www
    ```

---

## Laravel Directory Permissions

**For fresh Laravel setup:**
```sh
sudo usermod -aG www-data your_deploy_user
sudo chown -R your_deploy_user:www-data /var/www/test
sudo chmod -R ug+rwx /var/www/test/storage /var/www/test/bootstrap/cache
sudo find /var/www/test -type d -exec chmod 755 {} \;
sudo find /var/www/test -type f -exec chmod 644 {} \;
```
- **Owner:** `your_deploy_user` (pipeline/SSH/deployment user)
- **Group:** `www-data` (web server group: Nginx/Apache)
- **Writable Folders:** `storage` & `bootstrap/cache` as `775` (`ug+rwx`)

---

## Linux Permissions Side-By-Side Explanation

| Command | What it does |
|---------|--------------|
| `sudo usermod -aG www-data your_deploy_user` | Adds the user `your_deploy_user` to the `www-data` group. <br>**Why:** The web server group (`www-data`) can access folders, and your deploy user can work on files owned by this group, allowing safe shared editing and web server access. |
| `sudo chown -R your_deploy_user:www-data /var/www` | Changes ownership of all files and folders within `/var/www`.<br>`your_deploy_user` becomes the owner; `www-data` becomes the group.<br>**Why:** Your deploy user can modify and deploy code; the web server can read/write where group permissions allow. This is best for CI/CD and web deployment security and convenience. |

---

## Best Practice for CI/CD Laravel Deployment

**Preferred project ownership:**
```sh
sudo chown -R your_deploy_user:www-data /var/www/test
sudo chmod -R 755 /var/www/test
sudo chmod -R ug+rwx /var/www/test/storage /var/www/test/bootstrap/cache
```
- Lets CI/CD pipeline use your deploy user
- Allows web server to write required files

---

## CI/CD SSH Key Setup for GitHub Actions

**Best Practice:**
- Generate keys locally (never on your remote server).
- Store your private key in GitHub Secrets, never in your repository.
- CI/CD (GitHub Actions) connects to your server; server never "pulls" from CI.

**Step-by-step Process:**
1. **Generate SSH Key on Local Machine**
    ```sh
    ssh-keygen -t ed25519 -C "github-actions-deploy-lisasweep" -f ~/.ssh/lisasweep_deploy
    ```
    (Press Enter for passphrase)

2. **Copy Public Key to Server**
    - Automated:
      ```sh
      ssh-copy-id -i ~/.ssh/lisasweep_deploy.pub your_username@your_server_ip
      ```
    - Manual:
      ```sh
      cat ~/.ssh/lisasweep_deploy.pub
      # Paste output into ~/.ssh/authorized_keys on the server
      nano ~/.ssh/authorized_keys
      ```

3. **Test SSH Connection**
    ```sh
    ssh -i ~/.ssh/lisasweep_deploy your_username@your_server_ip
    ```
    You should log in without a password.

4. **Add Private Key to GitHub Secrets**
    - Display private key:
      ```sh
      cat ~/.ssh/lisasweep_deploy
      ```
    - Copy it all. Go to your repository: **Settings → Secrets → New Repository Secret**
      - Name: `SSH_PRIVATE_KEY`
      - Value: Paste the entire private key (beginning with `-----BEGIN OPENSSH PRIVATE KEY-----`)

---

## GitHub Actions Workflow

> See full workflow example above (expand for details!).

---

## Enhanced & Fixed Deployment Steps

- Fix common errors with Laravel writable directories:
    ```sh
    sudo chgrp -R www-data storage bootstrap/cache
    sudo chmod -R ug+rwx storage bootstrap/cache
    ```
- Always check post-deployment:
    ```sh
    ls -l storage/logs/
    ls -ld storage bootstrap/cache
    ```

---

## Troubleshooting & FAQ

- **Q: Why not use `sudo` when cloning?**  
  Root uses a different SSH key, causing failures.
- **Q: Should `/var/www` be owned by `www-data:www-data`?**  
  No; your deploy user should be owner for pipeline/deploy access. Use group `www-data` for server write access.
- **Q: Why not `chmod 777`?**  
  Unsafe — only use `775` for storage/cache and `755` for directories.
- **Q: Logs/cache folders unwritable?**  
  Ensure group owner is `www-data` and permissions allow group write: `ug+rwx`.

---

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Laravel Deployment Guide](https://laravel.com/docs/deployment)
- [GitHub SSH Key Best Practices](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
- [Appleboy SSH Action](https://github.com/appleboy/ssh-action)

---

## Final Checklist

- [ ] Deployment user in `www-data` group
- [ ] Ownership: `your_deploy_user:www-data`
- [ ] Permissions: `755` default, `ug+rwx` for writable folders
- [ ] SSH keys configured and tested before cloning
- [ ] CI/CD SSH private key added to GitHub secrets
- [ ] No private keys ever stored in your repository!
- [ ] Pipeline and branch logic verified