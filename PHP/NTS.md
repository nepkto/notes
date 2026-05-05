# PHP Thread Safe (TS) vs Non-Thread Safe (NTS): A Practical Guide (Windows & Linux)
 
_Last updated: 2026-04-27_
 
## 1) What Thread Safe (TS) and Non-Thread Safe (NTS) mean in PHP
 
### Thread Safe (TS)
- A **Thread Safe (TS)** PHP build is compiled with **Zend Thread Safety (ZTS)** enabled.
- ZTS adds internal protections so PHP can run safely in environments where **multiple threads may execute PHP code inside the same process**.
- Typical use case: **Apache `mod_php` on Windows** (and sometimes other threaded, in-process SAPI setups).
 
### Non-Thread Safe (NTS)
- A **Non-Thread Safe (NTS)** PHP build is compiled **without ZTS**.
- It assumes PHP requests are handled in a model where **each request runs in its own process** (or otherwise not sharing PHP runtime state across threads).
- Typical use case: **FastCGI / PHP-FPM** (common on both Windows and Linux).
 
---
 
## 2) Differences between TS and NTS (architecture, performance, memory)
 
### Architecture differences
- **TS (ZTS enabled)**
  - Uses thread-aware data structures and access patterns.
  - Helps prevent race conditions when PHP is hosted **in a threaded web server module**.
- **NTS (ZTS disabled)**
  - Simpler runtime assumptions.
  - Best for **process-based** request handling (FastCGI/FPM), where isolation is provided by processes.
 
### Performance considerations (general guidance)
- **NTS is typically slightly faster** than TS in many scenarios because it avoids some thread-safety overhead.
- Real-world performance depends more on:
  - OPcache configuration
  - I/O, database, network latency
  - application code and framework behavior
  - process manager tuning (FPM / FastCGI settings)
 
### Memory considerations
- **TS** may have a small additional memory overhead due to thread-safety mechanisms.
- **NTS** often has slightly less overhead per worker, but overall memory use depends heavily on:
  - number of worker processes (FPM/FastCGI)
  - loaded extensions
  - OPcache settings
 
---
 
## 3) PHP on Windows with IIS (FastCGI) – why NTS is required
 
### IIS + FastCGI execution model
- On Windows, PHP with IIS is most commonly run via **FastCGI** (`php-cgi.exe`) managed by IIS.
- With FastCGI, PHP runs **out-of-process** (separate worker processes), and IIS communicates with PHP over FastCGI.
 
### Why NTS is the correct choice for IIS
- The FastCGI model does **not** require PHP to be thread-safe in the same way as an in-process, threaded module.
- The recommended and commonly supported deployment for **IIS + FastCGI is PHP NTS**:
  - better performance characteristics (less overhead than ZTS)
  - aligns with Microsoft/IIS FastCGI process isolation design
  - matches how most Windows PHP distributions are documented for IIS
 
> Note: You may find TS builds available for Windows, but for **IIS FastCGI**, NTS is the standard, recommended choice.
 
---
 
## 4) PHP 8.3 changes regarding CGI on Windows
 
The most notable Windows+CGI-related change affecting PHP 8.3 deployments is security hardening in response to a CGI argument injection RCE vulnerability (CVE-2024-4577).
 
**PHP 8.3 Changes Regarding CGI on Windows: CVE-2024-4577**
 
In June 2024, a major security flaw was disclosed affecting PHP in CGI mode on Windows, including PHP 8.3. This vulnerability is tracked as **CVE-2024-4577**, and it allows for Remote Code Execution (RCE) through argument injection in certain locales (Traditional Chinese, Simplified Chinese, or Japanese code pages).
 
**What changed/what's the issue?**
- When using Apache and PHP in CGI mode on Windows (php.exe or php-cgi.exe exposed or mapped by AddHandler/Action or ScriptAlias), special handling by the Windows "Best-Fit" feature could replace certain characters (most notably, the soft hyphen, 0xAD) with a normal hyphen (0x2D) in command line arguments.
- PHP-CGI interprets those as new options/flags, letting attackers inject arguments (e.g., `-d` to change settings or even run code).
- This is a **regression/variant** of an old 2012 bug (CVE-2012-1823) thought to have been patched.
- The vulnerability existed in PHP 8.3.* before 8.3.8, as well as 8.2.* before 8.2.20, and 8.1.* before 8.1.29[[1]](https://nvd.nist.gov/vuln/detail/CVE-2024-4577)[[2]](https://infosecwriteups.com/cve-2024-4577-php-cgi-argument-injection-remote-code-execution-294ed4758e4f)[[3]](https://ikhaleelkhan.medium.com/exploiting-php-cgi-argument-injection-cve-2024-4577-a51aa78bd536)[[4]](https://www.sonicwall.com/blog/windows-php-servers-in-cgi-mode-vulnerable-to-exploitation-cve-2024-4577).
 
**Impact and Exploitation**
- Allows unauthenticated attackers to execute arbitrary PHP code.
- Can be triggered on default XAMPP installs on Windows.
- Exploit code is publicly available and was observed being used in the wild shortly after disclosure.
 
**Required configuration for vulnerability:**
- Windows OS in a vulnerable locale (Japanese/Chinese).
- Apache configured with PHP in CGI mode, with php.exe or php-cgi.exe accessible.
 
**What changed in PHP 8.3 (and other branches) to fix this?**
- The fix is present in **PHP 8.3.8** and later. Administrators must upgrade to at least 8.3.8 (or 8.2.20/8.1.29 for respective branches) to patch the vulnerability. Patched versions add more robust argument sanitization and further protections against command-line argument injection.
 
**Mitigation steps if you cannot upgrade immediately:**
- Block public access to php.exe/php-cgi.exe.
- Avoid exposing your server to the vulnerable locales if possible.
- Disable CGI mode for PHP if it’s not required, or use PHP as a module (FPM, etc.) instead of CGI[[5]](https://docs.indusface.com/en/article/cve-2024-4577-php-cgi-rce-exploitation-in-windows-servers)[[4]](https://www.sonicwall.com/blog/windows-php-servers-in-cgi-mode-vulnerable-to-exploitation-cve-2024-4577).
 
**In summary**:  
If you are running PHP 8.3 on Windows in CGI mode, it is critical to upgrade to version **8.3.8 or newer** to resolve CVE-2024-4577. This fixes a serious issue where attackers could inject PHP arguments and execute code due to Windows' command-line encoding quirks with certain locales. Immediate update and auditing of your server configuration is strongly recommended.
 
**References:**
- [PHP official changelog](https://www.php.net/ChangeLog-8.php)
- [Detailed writeups and exploit details](https://nvd.nist.gov/vuln/detail/CVE-2024-4577)[[1]](https://nvd.nist.gov/vuln/detail/CVE-2024-4577)[[2]](https://infosecwriteups.com/cve-2024-4577-php-cgi-argument-injection-remote-code-execution-294ed4758e4f)[[3]](https://ikhaleelkhan.medium.com/exploiting-php-cgi-argument-injection-cve-2024-4577-a51aa78bd536)[[4]](https://www.sonicwall.com/blog/windows-php-servers-in-cgi-mode-vulnerable-to-exploitation-cve-2024-4577)
 
---
 
1. [NVD - CVE-2024-4577](https://nvd.nist.gov/vuln/detail/CVE-2024-4577)
2. [CVE-2024–4577 — PHP CGI Argument Injection Remote Code Execution](https://infosecwriteups.com/cve-2024-4577-php-cgi-argument-injection-remote-code-execution-294ed4758e4f)
3. [Exploiting PHP CGI Argument Injection: CVE-2024–4577](https://ikhaleelkhan.medium.com/exploiting-php-cgi-argument-injection-cve-2024-4577-a51aa78bd536)
4. [Windows PHP Servers in CGI Mode Vulnerable to Exploitation ... - SonicWall](https://www.sonicwall.com/blog/windows-php-servers-in-cgi-mode-vulnerable-to-exploitation-cve-2024-4577)
5. [CVE-2024-4577 - PHP-CGI RCE Exploitation in Windows Servers](https://docs.indusface.com/en/article/cve-2024-4577-php-cgi-rce-exploitation-in-windows-servers)
 
---
 
## 5) PHP on Linux with Apache
 
### mod_php (TS)
- **What it is:** PHP runs as an **Apache module** inside Apache worker processes.
- **Thread-safety relevance:**
  - On Linux, Apache can run different Multi-Processing Modules (MPMs):
    - **prefork**: process-based (no threads per process)
    - **worker/event**: threaded
  - Historically, `mod_php` is most compatible with **prefork**.
- **Which build is used:**
  - Many Linux distributions package PHP for `mod_php` as part of their repo and handle the correct build internally.
  - The TS vs NTS distinction is most prominent for **Windows binaries**; on Linux, you usually select a *SAPI* (module vs FPM) via packages rather than downloading TS/NTS zip builds.
 
### PHP-FPM / FastCGI (NTS)
- **What it is:** PHP runs as a separate service (`php-fpm`) managing worker **processes**.
- **How Apache uses it:** Apache proxies requests to FPM using:
  - `mod_proxy_fcgi` (common)
  - or other FastCGI connectors
- **Why it’s preferred:**
  - works well with Apache threaded MPMs (`event` / `worker`)
  - isolates PHP crashes/leaks from Apache
  - easier scaling and tuning (pool config, number of workers, etc.)
  - common standard for modern deployments
 
---
 
## 6) How to check whether PHP is TS or NTS on Windows and Linux
 
### From the command line
Run:
```sh
php -i | findstr /i "Thread Safety"
Exploiting PHP CGI Argument Injection: CVE-2024–4577
Exploiting PHP CGI Argument Injection: CVE-2024–4577 Introduction: PHP, one of the most widely used scripting languages, has always been a target for attackers due to its extensive deployment in …
 