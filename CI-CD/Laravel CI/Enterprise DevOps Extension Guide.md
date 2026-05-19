# 🏗 Enterprise DevOps Extension Guide

## 🚀 Extensions Added
- 🐳 Docker + Docker Compose setup
- ☸️ Kubernetes (Enterprise-level)
- 📊 Monitoring & Logging (Prometheus, Grafana, ELK)
- 🔐 Secrets Management (Vault / ENV strategy)

---

# 🐳 Docker + Docker Compose Setup

## ✅ Example: Laravel + Caddy + MySQL

### Dockerfile (Laravel)
```Dockerfile
FROM php:8.2-fpm
WORKDIR /var/www
COPY . .
RUN docker-php-ext-install pdo pdo_mysql
```

### docker-compose.yml
```yaml
version: '3.8'

services:
  app:
    build: .
    container_name: laravel_app
    volumes:
      - .:/var/www

  caddy:
    image: caddy:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile

  mysql:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: secret
```

---

# ☸️ Kubernetes (Enterprise Setup)

## ✅ Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 3000
```

## ✅ Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: app
  ports:
    - port: 80
      targetPort: 3000
```

## ✅ Ingress (Caddy / Nginx)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

---

# 📊 Monitoring & Logging

## ✅ Prometheus + Grafana

- Prometheus → collects metrics
- Grafana → visualizes them

### Example scrape config
```yaml
scrape_configs:
  - job_name: 'app'
    static_configs:
      - targets: ['localhost:3000']
```

---

## ✅ ELK Stack

- Elasticsearch → store logs
- Logstash → process logs
- Kibana → visualize logs

---

# 🔐 Secrets Management

## ✅ Option 1: .env (basic)

```
DB_PASSWORD=secret
API_KEY=123
```

---

## ✅ Option 2: Vault (recommended)

- Store secrets securely
- Dynamic secrets

Example workflow:

```bash
vault kv put secret/app DB_PASSWORD=secret
vault kv get secret/app
```

---

## ✅ Option 3: Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_PASSWORD: c2VjcmV0  # base64
```

---

# 🧠 Best Practices

- Never store secrets in repo
- Use environment-specific configs
- Encrypt secrets at rest

---

# 🎯 Final Insight

> Modern DevOps = Automation + Observability + Security

- Docker → portability
- Kubernetes → scalability
- Monitoring → visibility
- Secrets → security

---

✅ You now have a complete enterprise-ready stack.