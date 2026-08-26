# Complete Docker & SQL Server 2022 Setup Guide

## Table of Contents
1. [Docker Desktop vs WSL-only Installation](#1-docker-desktop-vs-wsl-only-installation)
2. [Installing Ubuntu in WSL](#2-installing-ubuntu-in-wsl)
3. [Setting Up Git and Docker in WSL](#3-setting-up-git-and-docker-in-wsl)
4. [SQL Server 2022 Container - Decision Guide](#4-sql-server-2022-container---decision-guide)
5. [Docker Compose Configuration](#5-docker-compose-configuration)
6. [Managing Your SQL Server Container](#6-managing-your-sql-server-container)
7. [Multi-Service Architecture Decisions](#7-multi-service-architecture-decisions)
8. [Best Practices & Recommendations](#8-best-practices--recommendations)
9. [Troubleshooting](#9-troubleshooting)
10. [Quick Reference Card](#10-quick-reference-card)

---

## 1. Docker Desktop vs WSL-only Installation

### Option 1: Docker Desktop (with WSL2 backend)
**Pros:**
- GUI interface for container management
- Easy installation and setup
- Built-in Kubernetes support
- Enterprise features (scanning, extensions)

**Cons:**
- Resource-heavy (background processes)
- Paid for commercial use in large enterprises
- Can have virtualization detection issues

**Installation choices:**
- **Per-user installation (Recommended):** Uses WSL2 backend, no admin privileges required.
- **All-users installation:** Requires admin password, allows Hyper-V/Windows Containers.

### Option 2: Docker Engine inside WSL (Recommended approach)
**Pros:**
- Lightweight - minimal resource usage
- Completely free with no licensing restrictions
- Better file I/O performance
- Native Linux experience
- No virtualization detection issues

**Cons:**
- No GUI (manage via command line)
- Requires manual setup

### Option 3: WSL Containers (wslc) - Microsoft's native solution
**Pros:**
- Fully native, no third-party software
- Extremely lightweight
- Built into WSL

**Cons:**
- Still in public preview
- Requires Windows Insider build
- Limited ecosystem support

---

## 2. Installing Ubuntu in WSL

### Recommended Method (Single Command)
```powershell
wsl --install
```
**This command:**
- Enables Windows Subsystem for Linux and Virtual Machine Platform
- Downloads and installs the latest Linux kernel
- Sets WSL 2 as default version
- Installs Ubuntu (latest LTS version)

### Alternative - Install Specific Version
```powershell
# List available distributions
wsl --list --online

# Install specific version
wsl --install -d Ubuntu-24.04
```

### After Installation - Initial Setup
1. Restart your computer.
2. Launch Ubuntu from the Start Menu.
3. Create a UNIX username.
4. Create a password (characters won't be visible while typing).

### Update Packages
```bash
sudo apt update && sudo apt upgrade -y
```

---

## 3. Setting Up Git and Docker in WSL

### Installing Git
```bash
sudo apt install git -y

# Configure Git
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Verify installation
git --version
```

### Installing Docker Engine
```bash
# Quick install script (recommended)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group (avoids using sudo)
sudo usermod -aG docker $USER

# Start Docker service
sudo service docker start
```

### Auto-start Docker on WSL Launch
Add to `~/.bashrc` or `~/.zshrc`:
```bash
echo 'sudo service docker start' >> ~/.bashrc
```

### Verify Installations
```bash
docker run hello-world
git --version
```

### SSH Setup for Git (Optional)
```bash
ssh-keygen -t ed25519 -C "your.email@example.com"
cat ~/.ssh/id_ed25519.pub  # Copy and add to GitHub/GitLab
```

---

## 4. SQL Server 2022 Container - Decision Guide

### Key Configuration Decisions

#### 1. Container Name
- **Purpose:** Easy identification
- **Recommendation:** `sqlserver2022`
- **Why:** Simpler than using container IDs

#### 2. Port Mapping
- **Option A:** `1433:1433` - Standard port (easily guessable)
- **Option B:** `14333:1433` - Non-standard port (security through obscurity)
- **Recommendation:** Use non-standard port for external exposure

#### 3. Environment Variables
| Variable | Purpose | Required | Recommended Value |
|---|---|---|---|
| `ACCEPT_EULA` | Accept license terms | Yes | `Y` |
| `MSSQL_SA_PASSWORD` | SA account password | Yes | Strong password (min 8 chars, 3 of 4 complexity) |
| `MSSQL_PID` | Product edition | Yes | `Developer` (free, full features) |
| `MSSQL_AGENT_ENABLED` | Enable SQL Agent | No | `false` (unless scheduled jobs needed) |
| `MSSQL_COLLATION` | Default collation | No | Optional, default is fine |
| `MSSQL_LCID` | Language/locale | No | Optional, `1033` for US English |

#### 4. Storage Options
| Option | Example | Best For |
|---|---|---|
| Named Volume | `sqlserver_data:/var/opt/mssql` | Docker manages storage, portable |
| Bind Mount | `./data:/var/opt/mssql` | Direct Windows access, backup |
- **Recommendation:** Named Volume for simplicity and portability.

#### 5. Restart Policies
| Policy | Behavior |
|---|---|
| `no` | Never restarts automatically |
| `always` | Restarts on any stop (including system reboot) |
| `unless-stopped` | Restarts unless manually stopped |
| `on-failure` | Restarts only on error exit codes |
- **Recommendation:** `unless-stopped` for production, `no` for development.

#### 6. Resource Limits
```yaml
mem_limit: 2g   # Limit RAM to 2GB
cpus: 2         # Limit to 2 CPU cores
```

---

## 5. Docker Compose Configuration

### Final `docker-compose.yml` File
```yaml
version: '3.8'

services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: sqlserver2022
    hostname: sqlserver2022
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=${SA_PASSWORD}  # Use .env file
      - MSSQL_PID=Developer
      - MSSQL_AGENT_ENABLED=false
    ports:
      - "14333:1433"
    volumes:
      - sqlserver_data:/var/opt/mssql
    restart: "no"
    mem_limit: 2g
    cpus: 2
    networks:
      - sqlserver_network
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P ${SA_PASSWORD} -Q 'SELECT 1' || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  sqlserver_data:
    name: sqlserver_data

networks:
  sqlserver_network:
    name: sqlserver_network
    driver: bridge
```

### `.env` File (Store sensitive data)
```text
SA_PASSWORD=YourStrongPassword@2024!
```

### How to Start
```bash
# Create directory and file
mkdir ~/sqlserver
cd ~/sqlserver
nano docker-compose.yml   # Paste the content above
nano .env                 # Add SA_PASSWORD

# Start container
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs -f
```

---

## 6. Managing Your SQL Server Container

### Commands Reference
| Action | Command |
|---|---|
| Start | `docker compose up -d` |
| Stop | `docker compose stop` |
| Restart | `docker compose restart` |
| Remove (keep data) | `docker compose down` |
| Remove + delete data | `docker compose down -v` |
| View logs | `docker compose logs -f` |
| Execute inside | `docker compose exec sqlserver bash` |

### Connecting to SQL Server

**From Windows (SSMS/Azure Data Studio):**
- **Server:** `localhost,14333`
- **Authentication:** SQL Server Authentication
- **Login:** `sa`
- **Password:** Your SA password

**From WSL Terminal:**
```bash
/opt/mssql-tools/bin/sqlcmd -S localhost,14333 -U sa -P 'YourPassword'
```

**From Another Container:**
```bash
sqlcmd -S sqlserver2022,1433 -U sa -P 'YourPassword'
```

### Backup and Restore

**Backup a database:**
```bash
# Create backup inside container
docker compose exec sqlserver /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -Q "BACKUP DATABASE [YourDB] TO DISK='/var/opt/mssql/YourDB.bak'"

# Copy backup to host
docker cp sqlserver2022:/var/opt/mssql/YourDB.bak ./YourDB.bak
```

**Backup entire volume:**
```bash
docker run --rm -v sqlserver_data:/source -v $(pwd):/backup alpine tar czf /backup/sqlserver_backup.tar.gz -C /source .
```

**Restore volume backup:**
```bash
docker run --rm -v sqlserver_data:/target -v $(pwd):/backup alpine tar xzf /backup/sqlserver_backup.tar.gz -C /target
```

---

## 7. Multi-Service Architecture Decisions

### Approach 1: Single YAML (All-in-One)
**Best for:** Development, testing, small teams  
**Example:** App + SQL Server in one file
```yaml
version: '3.8'
services:
  sqlserver:
    # ... SQL Server config
  
  myapp:
    build: .
    depends_on:
      sqlserver:
        condition: service_healthy
    environment:
      - ConnectionStrings__DefaultConnection=Server=sqlserver2022,1433;Database=MyAppDB;User Id=sa;Password=${SA_PASSWORD};TrustServerCertificate=true
```
- **Pros:** One-command setup, dependency management, reproducible
- **Cons:** Tight coupling, limited scaling

### Approach 2: Separate YAMLs (Modular)
**Best for:** Production, multiple teams, microservices  
**Structure:**
```text
infra/docker-compose.yml    # SQL Server, Redis, etc.
app1/docker-compose.yml     # Application 1
app2/docker-compose.yml     # Application 2
```
**Commands:**
```bash
# Start infrastructure
docker compose -f infra/docker-compose.yml up -d

# Start app
docker compose -f app1/docker-compose.yml up -d
```
- **Pros:** Independent scaling/lifecycle, team autonomy, modular
- **Cons:** More complex, multiple commands needed

### Approach 3: Hybrid with Include
**Best for:** Flexibility, best of both worlds
```yaml
# docker-compose.yml (master)
version: '3.8'
include:
  - infra/docker-compose.yml
  - app1/docker-compose.yml
```

### Decision Tree
```text
Another app uses SQL Server?
    ├── Local dev / Testing → Single YAML
    ├── Production / Multi-team → Separate YAMLs
    └── Both environments → Hybrid Approach
```

---

## 8. Best Practices & Recommendations

### Environment Variables
**DO NOT** hardcode passwords. Use `.env` file:
```yaml
# docker-compose.yml
environment:
  - MSSQL_SA_PASSWORD=${SA_PASSWORD}
```
```bash
# .env
SA_PASSWORD=YourStrongPassword@2024!
```

### Health Checks
Always add health checks for dependencies:
```yaml
healthcheck:
  test: ["CMD-SHELL", "/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P ${SA_PASSWORD} -Q 'SELECT 1' || exit 1"]
  interval: 10s
  timeout: 5s
  retries: 10
```

### Resource Limits
Prevent containers from consuming all resources:
```yaml
mem_limit: 2g
cpus: 2
```

### Version Control
- Store `docker-compose.yml` in Git.
- **NEVER** commit `.env` files (add to `.gitignore`).
- Use `.env.example` as a template.

### Regular Backups
```bash
# Schedule daily backup of volume (crontab example)
0 2 * * * docker run --rm -v sqlserver_data:/source -v /backup:/backup alpine tar czf /backup/sqlserver_backup_$(date +\%Y\%m\%d).tar.gz -C /source .
```

### Monitoring
```bash
# Check disk usage
docker system df

# Monitor container resources
docker stats sqlserver2022
```

---

## 9. Troubleshooting

### Common Issues

#### Issue 1: Virtualization not detected
**Error:** "Virtualization support not detected"  
**Solution:**
```powershell
# Enable WSL and Virtual Machine Platform
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# Set WSL 2 as default
wsl --set-default-version 2

# Download and install WSL kernel update
# https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi

# Restart PC
```

#### Issue 2: Container fails to start
**Solution:**
```bash
# Check logs
docker compose logs

# Check if port is already in use
netstat -ano | findstr 14333

# Try a different port (Change to "14334:1433" in docker-compose.yml)
```

#### Issue 3: Password complexity error
**Error:** "Password does not meet SQL Server password complexity requirements"  
**Solution:** Ensure password meets these criteria:
- Min 8 characters
- Must include at least 3 of:
  - Uppercase letters (A-Z)
  - Lowercase letters (a-z)
  - Digits (0-9)
  - Special characters (!@#$%^&*)

#### Issue 4: Unable to connect from Windows
**Solution:**
- Verify container is running: `docker compose ps`
- Check firewall settings
- Ensure using correct port: `localhost,14333`
- Test connection from WSL first

#### Issue 5: Volume data lost
**Solution:** Always use `docker compose down` (not `-v`) to keep data.
```bash
# This preserves data
docker compose down

# This deletes all data (CAUTION)
docker compose down -v
```

#### Issue 6: Docker not starting in WSL
**Solution:**
```bash
# Start Docker manually
sudo service docker start

# Check service status
sudo service docker status

# Add auto-start to .bashrc
echo 'sudo service docker start' >> ~/.bashrc
```

### Quick Diagnostic Commands
```bash
# Check WSL status
wsl --status

# List Docker containers
docker ps -a

# Check Docker service
sudo service docker status

# View container details
docker inspect sqlserver2022

# Check volume exists
docker volume ls

# View volume details
docker volume inspect sqlserver_data
```

---

## 10. Quick Reference Card

### Setup Commands
```bash
# Install WSL Ubuntu
wsl --install

# Update Ubuntu
sudo apt update && sudo apt upgrade -y

# Install Git
sudo apt install git -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh

# Start SQL Server
cd ~/sqlserver && docker compose up -d
```

### Docker Compose Commands
```bash
docker compose up -d               # Start
docker compose down                # Stop (keep data)
docker compose down -v             # Stop + delete data
docker compose logs -f             # View logs
docker compose ps                  # Status
docker compose restart             # Restart
docker compose exec sqlserver bash # Enter container
```

### SQL Server Connection Strings
```text
Windows:   localhost,14333
WSL:       localhost,14333
Container: sqlserver2022,1433
```