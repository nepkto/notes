# Laravel Queue with Supervisor – Comprehensive Guide

This document provides instructions and troubleshooting tips for setting up and running Laravel queues in production using Supervisor. It covers common scenarios, configuration, monitoring, debugging stuck or pending jobs, and addresses important permission issues that can arise when running Supervisor.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Queue Setup](#queue-setup)
  - [1. Set the Queue Driver](#1-set-the-queue-driver)
  - [2. Configure the Queue Connection](#2-configure-the-queue-connection)
  - [3. Create Jobs](#3-create-jobs)
  - [4. Dispatch Jobs](#4-dispatch-jobs)
- [Supervisor Setup](#supervisor-setup)
  - [1. Install Supervisor](#1-install-supervisor)
  - [2. Supervisor Configuration Example](#2-supervisor-configuration-example)
  - [3. Reload and Start Supervisor](#3-reload-and-start-supervisor)
  - [4. Supervisor Permission Issues](#4-supervisor-permission-issues)
- [Monitoring and Troubleshooting](#monitoring-and-troubleshooting)
  - [1. Check Laravel Logs](#1-check-laravel-logs)
  - [2. Check Supervisor Logs](#2-check-supervisor-logs)
  - [3. Check Queue Status](#3-check-queue-status)
  - [4. Run Worker Manually](#4-run-worker-manually)
  - [5. Check Database Tables (Database Driver)](#5-check-database-tables-database-driver)
  - [6. Check Redis (Redis Driver)](#6-check-redis-redis-driver)
  - [7. Common Issues Checklist](#7-common-issues-checklist)
- [Best Practices](#best-practices)
- [References](#references)

---

## Prerequisites

- PHP >= 8.x
- Laravel >= 8.x
- Database (MySQL/PostgreSQL/SQLite) or Redis for queue driver
- Supervisor (process manager for Linux)

---

## Queue Setup

### 1. Set the Queue Driver

In your `.env` file, set the queue driver:

```ini
QUEUE_CONNECTION=database   # or 'redis', 'sqs', etc.
```

### 2. Configure the Queue Connection

For **database**:

```bash
php artisan queue:table
php artisan migrate
```

For **redis**:

- Install and configure Redis server.
- Set Redis credentials in `.env`.

### 3. Create Jobs

```bash
php artisan make:job ProcessSomething
```

Example job class:

```php
public function handle()
{
    // Job logic here
    \Log::info('Job started: ' . $this->job->getJobId());
}
```

### 4. Dispatch Jobs

In your code:

```php
ProcessSomething::dispatch($data);
```

---

## Supervisor Setup

### 1. Install Supervisor

On Ubuntu:

```bash
sudo apt-get install supervisor
```

### 2. Supervisor Configuration Example

Create a config file at `/etc/supervisor/conf.d/laravel-worker.conf`:

```
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /path/to/artisan queue:work database --sleep=3 --tries=3 --timeout=90
autostart=true
autorestart=true
user=www-data                ; Change to the web server user (www-data, nginx, apache, etc.)
numprocs=1
redirect_stderr=true
stdout_logfile=/var/log/supervisor/laravel-worker.log
```

- Adjust the `command`, `user`, and log file paths as needed.
- For Redis, use `queue:work redis ...`.

### 3. Reload and Start Supervisor

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*
```

### 4. Supervisor Permission Issues

**Supervisor** runs your Laravel worker processes under the user you specify (`user=www-data` in the example above). Permission issues can arise if this user does not have access to required files and directories.

#### Common Permission Problems

- **Logs and Storage:** Worker user (e.g., `www-data`) needs read/write access to:
  - `storage/`
  - `storage/logs/`
  - `bootstrap/cache/`
- **PHP Executable:** The user must be allowed to execute PHP.
- **Supervisor Config & Log:** Ensure Supervisor log directories and configured log files are accessible.

#### How to Fix Permission Issues

**Set directory permissions:**
```bash
sudo chown -R www-data:www-data /path/to/your-laravel-project
sudo chmod -R 775 /path/to/your-laravel-project/storage
sudo chmod -R 775 /path/to/your-laravel-project/bootstrap/cache
```

**Check your current directory and file permissions:**
```bash
ls -la storage
ls -la bootstrap/cache
```

**Restart Supervisor after adjusting permissions:**
```bash
sudo supervisorctl restart laravel-worker:*
```

**Note:** If permissions are wrong, you may see errors such as:
- `Permission denied` when writing logs or cache files
- No logs being written at all
- Jobs not processing with no clear errors

Always ensure the Supervisor worker user matches your web server and deployment configuration.

---

## Monitoring and Troubleshooting

### 1. Check Laravel Logs

```bash
tail -f storage/logs/laravel.log
```
Look for errors related to jobs or queue workers.

### 2. Check Supervisor Logs

```bash
tail -f /var/log/supervisor/laravel-worker.log
```
Look for process or PHP errors.

### 3. Check Queue Status

For **database** driver:

```sql
SELECT * FROM jobs;
SELECT * FROM failed_jobs;
```
- Check if jobs are stuck in `jobs` table or have moved to `failed_jobs`.

For **redis** driver:

```bash
redis-cli llen queues:default   # Replace 'default' with your queue name if needed
redis-cli lrange queues:default 0 -1
```

### 4. Run Worker Manually

To see immediate output/errors:

```bash
php artisan queue:work --verbose
```

### 5. Check Database Tables (Database Driver)

- Are jobs sitting in `jobs` table?
- Is the `attempts` column increasing?
- Are jobs appearing in `failed_jobs`?

### 6. Check Redis (Redis Driver)

- Is Redis running?  
  ```bash
  redis-cli ping
  ```
- Are jobs appearing in the queue?

### 7. Common Issues Checklist

- **QUEUE_CONNECTION** in `.env` does not match what the worker expects.
- Jobs dispatched to a specific queue but worker is listening to a different queue.
- Supervisor running as wrong user (should match web server user, e.g., `www-data`).
- Permissions on `storage` and `bootstrap/cache` directories.
- Worker is running but not processing jobs (restart it).
- Stale cache/config:  
  ```bash
  php artisan config:clear
  php artisan cache:clear
  php artisan queue:restart
  ```
- No errors, but jobs remain pending: try adding a log entry in the job’s `handle()` method to confirm execution.

---

## Best Practices

- Use `queue:work` (not `queue:listen`) with Supervisor in production.
- Use `php artisan queue:restart` after deploying new code.
- Monitor failed jobs and set up notifications.
- Use [Laravel Horizon](https://laravel.com/docs/horizon) for advanced queue monitoring (Redis driver).
- Regularly check Supervisor and Laravel logs.
- Ensure all environment variables are up to date for the worker process.
- Always verify permissions after code deployments or server changes.