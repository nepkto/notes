# Apache Cheat Sheet

A handy reference for common Apache commands and diagnostics.

---

## Commands Overview
All commands referenced in this sheet:
```
sudo systemctl restart apache2
sudo systemctl reload apache2
sudo service apache2 restart
sudo apache2ctl configtest
sudo a2enmod rewrite
sudo a2enmod headers
apache2ctl -t -D DUMP_VHOSTS
apache2ctl -t -D DUMP_MODULES
sudo apachectl configtest
sudo tail -100 /var/log/apache2/access.log
sudo tail -100 /var/log/apache2/error.log
sudo bash -c 'echo > /var/log/apache2/error.log'
sudo grep "GET" /var/log/apache2/error.log
sudo grep -i "GET" /var/log/apache2/error.log
sudo bash -c 'echo > /var/log/nginx/Admin-Bioelite.error.log'
```

---

## Table of Contents

- [Basic Service Management](#basic-service-management)
- [Configuration & Modules](#configuration--modules)
- [Logs](#logs)
- [Troubleshooting](#troubleshooting)
- [Other Useful Commands](#other-useful-commands)
- [Locations](#locations)
- [Quick References](#quick-references)

---

## [Basic Service Management](#basic-service-management)

| Task                      | Command                                  | Description                                  |
|---------------------------|-------------------------------------------|----------------------------------------------|
| Restart Apache            | `sudo systemctl restart apache2`         | Restart the Apache service.                  |
| Reload Apache Config      | `sudo systemctl reload apache2`          | Reload configuration without full restart.   |
| Restart (alternative)     | `sudo service apache2 restart`           | Alternative restart command.                 |

---

## [Configuration & Modules](#configuration--modules)

| Task                      | Command                                                    | Description                                    |
|---------------------------|------------------------------------------------------------|------------------------------------------------|
| Check Config Syntax       | `sudo apache2ctl configtest`                               | Tests Apache config files for syntax errors.   |
| Enable Rewrite Module     | `sudo a2enmod rewrite`                                     | Enables mod_rewrite for URL rewriting.         |
| Enable Headers Module     | `sudo a2enmod headers`                                     | Enables mod_headers (for Header directives).   |
| List Loaded Modules       | `apache2ctl -t -D DUMP_MODULES`                            | Lists active/loaded Apache modules.            |
| List Virtual Hosts        | `apache2ctl -t -D DUMP_VHOSTS`                             | Displays vhost config currently loaded.        |
| Check Config Again        | `sudo apachectl configtest`                                | Alternative configtest command.                |

---

## [Logs](#logs)

### Error Log

- **View last 100 error lines**
  - `sudo tail -100 /var/log/apache2/error.log`
- **Clear error log**
  - `sudo bash -c 'echo > /var/log/apache2/error.log'`
- **Search for term (case insensitive)**
  - `sudo grep -i "GET" /var/log/apache2/error.log`
- **Search for specific term**
  - `sudo grep "GET" /var/log/apache2/error.log`

### Access Log

- **View last 100 access lines**
  - `sudo tail -100 /var/log/apache2/access.log`

---

## [Troubleshooting](#troubleshooting)

| Issue/Message                                                                       | Solution/Command                         | Description                                             |
|-------------------------------------------------------------------------------------|------------------------------------------|---------------------------------------------------------|
| `Invalid command 'Header'` error                                                    | `sudo a2enmod headers`                   | Enables headers module needed for 'Header' directive.   |
| Request URL not found (mod_rewrite issues)                                          | `sudo a2enmod rewrite`                   | Enables rewrite module.                                 |
| Check if modules/vhost config loaded                                                | `apache2ctl -t -D DUMP_MODULES`          | See which modules are active.                           |
|                                                                                     | `apache2ctl -t -D DUMP_VHOSTS`           | See active vhost configs.                               |

---

## [Other Useful Commands](#other-useful-commands)

| Task                          | Command                                      | Description                            |
|-------------------------------|----------------------------------------------|----------------------------------------|
| Search for specific term      | `sudo grep "GET" /var/log/apache2/error.log` | Case sensitive search in error log.    |
| Search (case insensitive)     | `sudo grep -i "GET" /var/log/apache2/error.log` | Case insensitive search in error log.  |
| Clear Nginx error log         | `sudo bash -c 'echo > /var/log/nginx/Admin-Bioelite.error.log'` | Clear (if using Nginx alongside).      |

---

## [Locations](#locations)

- **Config Files**: `/etc/apache2/`
- **Error Log**: `/var/log/apache2/error.log`
- **Access Log**: `/var/log/apache2/access.log`

---

## [Quick References](#quick-references)

- **.htaccess** files are used for per-directory config overrides.
- Always reload/restart Apache after editing config files or enabling modules.
- Check error logs for hints about configuration problems.

---