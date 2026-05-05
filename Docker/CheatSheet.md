# 🐳 Docker Debug & Diagnostics Cheat Sheet
### Tech Stack: Laravel · Redis · Node · Queue · Supervisor · PHP · Nginx · Apache · Docker

---

## 📦 DOCKER — Core Debugging Commands

### Container Management
```bash
# List all running containers
docker ps

# List all containers (including stopped)
docker ps -a

# List containers with resource usage
docker stats

# Inspect a container (full config, network, mounts)
docker inspect <container_name_or_id>

# View real-time logs
docker logs -f <container_name>

# View last N lines of logs
docker logs --tail=100 <container_name>

# View logs with timestamps
docker logs -f --timestamps <container_name>

# Execute a command inside a running container
docker exec -it <container_name> bash
docker exec -it <container_name> sh   # if bash not available

# Copy files from container to host
docker cp <container_name>:/path/to/file ./local-path

# Check container resource usage (CPU, Memory, Net I/O)
docker stats <container_name>
```

### Image & Volume Debugging
```bash
# List images
docker images

# Inspect an image
docker inspect <image_name>

# List volumes
docker volume ls

# Inspect a volume
docker volume inspect <volume_name>

# Remove unused volumes
docker volume prune

# Remove all stopped containers, unused networks, dangling images
docker system prune -a

# Check disk usage
docker system df
```

### Network Debugging
```bash
# List networks
docker network ls

# Inspect a network
docker network inspect <network_name>

# Test connectivity between containers
docker exec -it <container_name> ping <other_container_name>

# Check open ports inside a container
docker exec -it <container_name> netstat -tulnp
docker exec -it <container_name> ss -tulnp
```

### Docker Compose
```bash
# Start services
docker compose up -d

# Rebuild and start services
docker compose up -d --build

# Stop services
docker compose down

# Stop and remove volumes
docker compose down -v

# View logs for all services
docker compose logs -f

# View logs for a specific service
docker compose logs -f <service_name>

# Check status of compose services
docker compose ps

# Restart a specific service
docker compose restart <service_name>

# Run a one-off command in a service
docker compose exec <service_name> bash

# Validate docker-compose.yml syntax
docker compose config
```

---

## 🐘 PHP — Debugging Commands

```bash
# Enter PHP container
docker exec -it <php_container> bash

# Check PHP version
php -v

# Check loaded PHP modules/extensions
php -m

# Check PHP configuration
php --ini
php -i | grep -i "error_log"
php -i | grep -i "memory_limit"
php -i | grep -i "upload_max"

# Check PHP-FPM status
php-fpm -t                          # test PHP-FPM config
php-fpm -i | grep -i "max_children"

# Check PHP error log
tail -f /var/log/php/error.log
tail -f /var/log/php-fpm/www-error.log

# Run a PHP script
php /var/www/html/artisan --version

# Check OPcache status
php -r "print_r(opcache_get_status());"
```

---

## 🟥 LARAVEL — Debugging Commands

```bash
# Enter Laravel app container
docker exec -it <app_container> bash

# Check Laravel version
php artisan --version

# Clear all caches
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan event:clear

# Cache all (production)
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# Check application environment and config
php artisan env
php artisan about

# Run database migrations
php artisan migrate
php artisan migrate:status
php artisan migrate:rollback

# Seed the database
php artisan db:seed

# List all routes
php artisan route:list
php artisan route:list --path=api

# Check queue status
php artisan queue:monitor
php artisan queue:failed
php artisan queue:retry all
php artisan queue:flush          # delete all failed jobs

# Tinker (interactive Laravel REPL)
php artisan tinker

# Debug config values in Tinker
>>> config('database.default')
>>> config('cache.default')
>>> env('APP_ENV')

# Storage symlink
php artisan storage:link

# Check scheduled tasks
php artisan schedule:list
php artisan schedule:run --verbose

# Optimize for production
php artisan optimize
php artisan optimize:clear

# View Laravel logs
tail -f /var/www/html/storage/logs/laravel.log
```

---

## 🔴 REDIS — Debugging Commands

```bash
# Enter Redis container
docker exec -it <redis_container> bash
# OR connect directly
docker exec -it <redis_container> redis-cli

# Ping Redis (check connectivity)
redis-cli ping                    # should return PONG

# Connect to Redis with auth
redis-cli -h <host> -p 6379 -a <password>

# Monitor all commands in real-time
redis-cli monitor

# Check Redis info & stats
redis-cli info
redis-cli info server
redis-cli info memory
redis-cli info clients
redis-cli info stats
redis-cli info replication

# List all keys
redis-cli keys "*"

# Count all keys
redis-cli dbsize

# Get value of a key
redis-cli get <key>

# Delete a key
redis-cli del <key>

# Flush all keys in current database
redis-cli flushdb

# Flush all databases (CAUTION!)
redis-cli flushall

# Check TTL of a key
redis-cli ttl <key>

# Check memory usage of a key
redis-cli memory usage <key>

# Slow log (queries slower than threshold)
redis-cli slowlog get 10

# Check connected clients
redis-cli client list

# Check Redis config
redis-cli config get maxmemory
redis-cli config get maxmemory-policy
redis-cli config get save

# From Laravel container — test Redis connection
php artisan tinker
>>> Redis::ping()
>>> Cache::store('redis')->put('test', 'hello', 10)
>>> Cache::store('redis')->get('test')
```

---

## 🟢 NODE.JS — Debugging Commands

```bash
# Enter Node container
docker exec -it <node_container> bash

# Check Node and npm version
node -v
npm -v
yarn -v

# Check running Node processes
ps aux | grep node

# Check environment variables
env | grep NODE

# Run app in debug mode
node --inspect=0.0.0.0:9229 app.js

# Check installed packages
npm list
npm list --depth=0

# Check for outdated packages
npm outdated

# View npm logs
npm config get cache
cat ~/.npm/_logs/*.log

# Check Node memory usage
node -e "console.log(process.memoryUsage())"

# Kill a Node process by port
kill $(lsof -t -i:3000)

# Test HTTP endpoint from within container
curl -v http://localhost:3000/health
wget -qO- http://localhost:3000/health
```

---

## ⚙️ QUEUE (Laravel Queue Worker) — Debugging

```bash
# Run queue worker (foreground)
php artisan queue:work

# Run with specific connection and queue
php artisan queue:work redis --queue=default,high

# Run with verbose output
php artisan queue:work --verbose

# Limit worker memory and timeout
php artisan queue:work --memory=512 --timeout=60

# Run queue listener (re-reads code on each job — for dev)
php artisan queue:listen

# Process only one job then stop
php artisan queue:work --once

# List failed jobs
php artisan queue:failed

# Retry a specific failed job
php artisan queue:retry <job_id>

# Retry all failed jobs
php artisan queue:retry all

# Delete a failed job
php artisan queue:forget <job_id>

# Flush all failed jobs
php artisan queue:flush

# Check queue size in Redis
redis-cli llen laravel_database_queues:default
redis-cli llen laravel_database_queues:high

# Monitor queues
php artisan queue:monitor redis:default,redis:high --max=100

# Horizon (if using Laravel Horizon)
php artisan horizon
php artisan horizon:status
php artisan horizon:pause
php artisan horizon:continue
php artisan horizon:terminate
```

---

## 🦺 SUPERVISOR — Debugging Commands

```bash
# Enter container with Supervisor
docker exec -it <container_name> bash

# Check Supervisor status
supervisorctl status

# Start all processes
supervisorctl start all

# Stop all processes
supervisorctl stop all

# Restart all processes
supervisorctl restart all

# Start a specific process
supervisorctl start laravel-worker:*

# Stop a specific process
supervisorctl stop laravel-worker:*

# Restart a specific process
supervisorctl restart laravel-worker:*

# Reload config without restarting supervisor
supervisorctl reread
supervisorctl update

# View Supervisor logs
tail -f /var/log/supervisor/supervisord.log
tail -f /var/log/supervisor/laravel-worker.log
tail -f /var/log/supervisor/laravel-worker-stderr.log

# Check Supervisor config
cat /etc/supervisor/conf.d/laravel-worker.conf

# Restart supervisord itself
service supervisor restart
# OR
supervisord -c /etc/supervisor/supervisord.conf
```

### Example Supervisor Config for Laravel Queue
```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/html/artisan queue:work redis --sleep=3 --tries=3 --timeout=90
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=4
redirect_stderr=true
stdout_logfile=/var/log/supervisor/laravel-worker.log
stopwaitsecs=3600
```

---

## 🌐 NGINX — Debugging Commands

```bash
# Enter Nginx container
docker exec -it <nginx_container> bash

# Check Nginx version
nginx -v

# Test Nginx configuration
nginx -t

# Reload Nginx config (graceful)
nginx -s reload

# View Nginx logs
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# Watch both logs simultaneously
tail -f /var/log/nginx/access.log /var/log/nginx/error.log

# Check Nginx process
ps aux | grep nginx

# Check what ports Nginx is listening on
netstat -tulnp | grep nginx
ss -tulnp | grep nginx

# Check Nginx config location
nginx -V 2>&1 | grep "conf-path"

# Display full Nginx config
nginx -T

# Debug upstream connection (from Nginx container)
curl -v http://php-fpm:9000
curl -v http://app:8000

# Check active Nginx connections
cat /proc/$(pgrep nginx | head -1)/net/sockstat
```

### Common Nginx Vhost Debug
```bash
# Check if site config is loaded
ls /etc/nginx/sites-enabled/
ls /etc/nginx/conf.d/

# Check PHP-FPM socket/port
grep -r "fastcgi_pass" /etc/nginx/
```

---

## 🪶 APACHE — Debugging Commands

```bash
# Enter Apache container
docker exec -it <apache_container> bash

# Check Apache version
apache2 -v
httpd -v

# Test Apache configuration
apache2ctl configtest
httpd -t

# Reload Apache config (graceful)
apache2ctl graceful
service apache2 reload

# Restart Apache
service apache2 restart
apache2ctl restart

# View Apache logs
tail -f /var/log/apache2/access.log
tail -f /var/log/apache2/error.log

# List enabled modules
apache2ctl -M
apachectl -M | grep -i rewrite    # check mod_rewrite

# Check enabled sites and configs
ls /etc/apache2/sites-enabled/
ls /etc/apache2/conf-enabled/

# Check Apache listening ports
netstat -tulnp | grep apache
ss -tulnp | grep apache2

# Enable/disable modules (inside container)
a2enmod rewrite
a2enmod headers
a2dismod status

# Full Apache config dump
apache2ctl -S

# Check .htaccess processing
grep -r "AllowOverride" /etc/apache2/
```

---

## 🔗 CONNECTIVITY & NETWORK DIAGNOSTICS

```bash
# Test connectivity between containers (inside any container)
ping <service_name>
curl -v http://<service_name>:<port>
wget -qO- http://<service_name>:<port>

# DNS resolution inside container
nslookup <service_name>
dig <service_name>

# Check all open ports inside a container
netstat -tulnp
ss -tulnp

# Port scan from one container to another
nc -zv <service_name> <port>

# Trace route inside container
traceroute <service_name>

# Check environment variables (for service URLs, credentials)
env | grep -i db
env | grep -i redis
env | grep -i app
printenv
```

---

## 🗄️ DATABASE (MySQL/PostgreSQL) — Quick Diagnostics

```bash
# MySQL
docker exec -it <mysql_container> mysql -u root -p
SHOW DATABASES;
SHOW TABLES;
SHOW PROCESSLIST;
SHOW STATUS LIKE 'Threads_connected';

# PostgreSQL
docker exec -it <postgres_container> psql -U postgres
\l        -- list databases
\dt       -- list tables
SELECT * FROM pg_stat_activity;

# From Laravel container
php artisan db
php artisan tinker
>>> DB::connection()->getPdo()
>>> DB::select('SELECT 1')
```

---

## 📋 QUICK HEALTH CHECK SCRIPT

```bash
#!/bin/bash
echo "=== Docker Containers ===" && docker ps
echo "=== Nginx ===" && docker exec nginx nginx -t
echo "=== PHP-FPM ===" && docker exec php php-fpm -t
echo "=== Redis ===" && docker exec redis redis-cli ping
echo "=== Supervisor ===" && docker exec app supervisorctl status
echo "=== Laravel ===" && docker exec app php artisan about
echo "=== Queue Failed Jobs ===" && docker exec app php artisan queue:failed
echo "=== Laravel Logs (last 20) ===" && docker exec app tail -20 /var/www/html/storage/logs/laravel.log
```

---

## 🚨 COMMON ISSUES & QUICK FIXES

| Issue | Diagnosis | Fix |
|-------|-----------|-----|
| 502 Bad Gateway | `docker logs nginx` → upstream failed | Restart PHP-FPM: `docker restart php` |
| Queue not processing | `supervisorctl status` → stopped | `supervisorctl restart laravel-worker:*` |
| Redis connection refused | `redis-cli ping` fails | Check Redis container running & port |
| Permission denied on storage | `ls -la storage/` | `chmod -R 775 storage bootstrap/cache` |
| .env not loaded | `php artisan env` wrong | `php artisan config:clear` |
| Node port not accessible | `netstat -tulnp` | Check EXPOSE in Dockerfile |
| DB migration fails | `php artisan migrate:status` | Check DB env vars & connectivity |
| OOM / container killed | `docker stats` | Increase memory limit in compose |
| Nginx 413 (payload too large) | Check error.log | Set `client_max_body_size 50M;` in Nginx |
| PHP timeout | Check PHP error log | Increase `max_execution_time` in php.ini |

---

## 📁 KEY LOG FILE Locations

| Service | Log Path |
|---------|----------|
| Laravel | `/var/www/html/storage/logs/laravel.log` |
| Nginx Access | `/var/log/nginx/access.log` |
| Nginx Error | `/var/log/nginx/error.log` |
| Apache Access | `/var/log/apache2/access.log` |
| Apache Error | `/var/log/apache2/error.log` |
| PHP-FPM | `/var/log/php-fpm/www-error.log` |
| Supervisor | `/var/log/supervisor/supervisord.log` |
| Redis | `/var/log/redis/redis-server.log` |
| MySQL | `/var/log/mysql/error.log` |

---

> 💡 **Tip:** Always use `docker compose logs -f <service>` first for quick diagnosis, then exec into the container for deeper investigation.
> 
> 🔐 **Security:** Never run `flushall` on Redis in production. Always double-check the container name before destructive commands.