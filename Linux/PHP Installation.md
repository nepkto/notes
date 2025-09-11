# PHP Removal, Reset, and Installation Guide

This README provides step-by-step instructions for:
1. **Complete removal/reset of PHP and its configuration files**
2. **Installation of PHP and PHP-FPM of a specific version**  
   - Using the regular package manager process  
   - Manually compiling PHP 8.2 from source (recommended for unsupported or custom installations)

---

## 1. Complete Removal/Reset of PHP and Configuration

### **A. Remove All PHP Packages**

```bash
sudo apt remove --purge 'php*'
sudo apt autoremove
```

### **B. Remove Configuration Files**

```bash
sudo rm -rf /etc/php/
sudo rm -rf /usr/local/php*
sudo rm -rf /var/lib/php/
sudo rm -rf /var/log/php*
sudo rm -rf /var/run/php/
```

### **C. Clean APT Cache**

```bash
sudo apt clean
```

### **D. Remove Third-Party PPAs (if any)**

List PPAs:
```bash
ls /etc/apt/sources.list.d/
```
Remove PHP-related PPAs:
```bash
sudo rm /etc/apt/sources.list.d/ondrej-*.list
```

---

## 2. Install PHP and PHP-FPM of Specific Version

### **A. Regular Process (Using APT Repository)**

> **Note:** The available PHP version depends on your OS and its repositories.  
> For Ubuntu, newer PHP versions are provided by the [Ondřej Surý PPA](https://launchpad.net/~ondrej/+archive/ubuntu/php).

#### **For Ubuntu/Debian (Supported PHP Versions)**

**Add PPA (Ubuntu example):**
```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt update
```

**Install Specific PHP Version (e.g., PHP 8.1, PHP 8.1-fpm):**
```bash
sudo apt install php8.2 libapache2-mod-php8.2 php8.2-mysql php8.2-mbstring php8.2-dom php8.2-gd php8.2-zip php8.2-curl -y
```

**Verify Installation:**
```bash
php -v
systemctl status php8.1-fpm
```

---

### **B. Manual Process: Compile PHP 8.2 from Source**

> Use this if your desired PHP version is not available via APT.

#### **Step 1: Install Build Dependencies**

```bash
sudo apt update
sudo apt install -y build-essential autoconf bison re2c libxml2-dev \
libsqlite3-dev libssl-dev libcurl4-openssl-dev libjpeg-dev libpng-dev \
libwebp-dev libfreetype6-dev libzip-dev libonig-dev libxslt1-dev libffi-dev \
libmysqlclient-dev libreadline-dev pkg-config
```

#### **Step 2: Download PHP Source**

Find the latest PHP 8.2 tarball at [php.net/downloads](https://www.php.net/downloads.php):

```bash
wget https://www.php.net/distributions/php-8.2.14.tar.gz
tar -xvf php-8.2.14.tar.gz
cd php-8.2.14
```

#### **Step 3: Configure Build Options**

```bash
./configure --prefix=/usr/local/php8.2 \
--with-openssl \
--with-zlib \
--enable-mbstring \
--enable-zip \
--enable-soap \
--enable-intl \
--with-curl \
--with-pdo-mysql \
--with-readline \
--enable-fpm \
--with-fpm-user=www-data \
--with-fpm-group=www-data
```

#### **Step 4: Compile and Install**

```bash
make -j$(nproc)
sudo make install
```

#### **Step 5: Configure PHP-FPM**

Copy the default configs:
```bash
sudo cp /usr/local/php8.2/etc/php-fpm.conf.default /usr/local/php8.2/etc/php-fpm.conf
sudo cp /usr/local/php8.2/etc/php-fpm.d/www.conf.default /usr/local/php8.2/etc/php-fpm.d/www.conf
```

#### **Step 6: Add PHP 8.2 to PATH**

Add to `~/.bashrc` or `/etc/profile.d/php8.2.sh`:
```bash
export PATH="/usr/local/php8.2/bin:/usr/local/php8.2/sbin:$PATH"
```
Reload your shell:
```bash
source ~/.bashrc
```

#### **Step 7: Verify Installation**

```bash
php -v
php-fpm -v
which php
```

---

## **Troubleshooting**

- Always backup your data before removal or upgrades.
- For manual installs, ensure all dependencies are met and configuration paths are correct.
- Use official documentation for advanced configuration: [PHP Manual](https://www.php.net/manual/en/)

---

## **References**

- [PHP Downloads](https://www.php.net/downloads.php)
- [Ondřej Surý PHP PPA](https://launchpad.net/~ondrej/+archive/ubuntu/php)
- [PHP Manual Installation](https://www.php.net/manual/en/install.unix.php)