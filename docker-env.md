# Docker Development Environment

A collection of Dockerfiles and docker-compose configurations for local development and production.

---

## Common Commands

```bash
# Build and start
# -d        : detached mode, run containers in background
# --build   : force rebuild images before starting
docker compose up -d --build

# Or separately
docker compose build   # build images only
docker compose up -d   # start containers (uses existing images)

# View logs
# -f : follow/stream logs in real time (ctrl+c to exit)
docker compose logs -f
docker compose logs -f node   # single service only

# List services
docker compose ps

# Access container shell (use service name, not container name)
docker compose exec node sh
docker compose exec php bash
docker compose exec python bash

# Run one-off command
docker compose exec php composer install
docker compose exec node npm install

# Single service — rebuild or restart one container without touching others
docker compose up -d --build node     # rebuild and restart node only
docker compose restart node           # restart without rebuilding

# Stop
docker compose down

# Stop and reset database volumes
docker compose down -v
rm -rf mysql-data postgres-data

# Remove dangling/unused images (middle ground — does not affect running containers)
docker image prune

# Reset entire project environment (scoped to current folder only)
# -v                : remove named volumes (wipes database data)
# --rmi all         : remove images built for this project
# --remove-orphans  : remove containers no longer defined in compose file
docker compose down -v --rmi all --remove-orphans
# Then rebuild fresh
docker compose up -d --build
```

---

## Table of Contents

- [Docker Development Environment](#docker-development-environment)
  - [Common Commands](#common-commands)
  - [Table of Contents](#table-of-contents)
  - [Node.js](#nodejs)
  - [Python](#python)
  - [PHP / Laravel](#php--laravel)
  - [Databases](#databases)
    - [SQLite](#sqlite)
    - [MySQL](#mysql)
    - [PostgreSQL](#postgresql)
    - [Redis](#redis)
  - [Environment Variables (.env)](#environment-variables-env)
  - [Production](#production)
  - [Notes](#notes)

---

## Node.js

**Dockerfile.node**

```dockerfile
FROM node:22-slim

RUN npm install -g npm@latest

WORKDIR /app

USER node
```

**docker-compose.yml**

```yaml
services:
  node:
    build:
      context: .
      dockerfile: Dockerfile.node
    container_name: node-container
    working_dir: /app
    volumes:
      - .:/app
    ports:
      - "3000:3000"
    tty: true
    # command: sh -c "npm run dev"
    networks:
      - app_network

networks:
  app_network:
    external: true
```

Access: `http://localhost:3000`

---

## Python

**Dockerfile.python**

```dockerfile
FROM python:3.13-slim

RUN useradd -u 1000 -m python

RUN pip install flask flask-cors

WORKDIR /app

USER python
```

**docker-compose.yml**

```yaml
services:
  python:
    build:
      context: .
      dockerfile: Dockerfile.python
    container_name: python-container
    working_dir: /app
    volumes:
      - .:/app
    ports:
      - "4000:4000"
    tty: true
    # command: sh -c "python3 index.py"
    networks:
      - app_network

networks:
  app_network:
    external: true
```

Access: `http://localhost:4000`

---

## PHP / Laravel

> This setup is optimized for Laravel but works for vanilla PHP too.
> For vanilla PHP, just change `root /app/public` to your document root.

**Dockerfile.php**

```dockerfile
FROM php:8.3-cli

RUN apt-get update && apt-get install -y \
    git curl zip unzip \
    libpng-dev libonig-dev libxml2-dev libzip-dev \
    libcurl4-openssl-dev libssl-dev libsqlite3-dev \
    && rm -rf /var/lib/apt/lists/*

# PHP extensions for Laravel
RUN docker-php-ext-install \
    pdo pdo_mysql pdo_sqlite \
    mysqli \
    mbstring \
    exif \
    pcntl \
    bcmath \
    gd \
    zip \
    xml \
    curl

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Install Node.js (for Laravel Vite/Mix)
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get update \
    && apt-get install -y nodejs \
    && npm install -g npm@latest \
    && rm -rf /var/lib/apt/lists/*

RUN useradd -u 1000 -m php

WORKDIR /app

USER php
```

**docker-compose.yml**

```yaml
services:
  php:
    build:
      context: .
      dockerfile: Dockerfile.php
    container_name: php-container
    working_dir: /app
    volumes:
      - .:/app
    ports:
      - "8080:8080"
    tty: true
    # command: sh -c "php artisan serve --host=0.0.0.0 --port=8080"
    # command: sh -c "php -S 0.0.0.0:8080"
    networks:
      - app_network

networks:
  app_network:
    external: true
```

Access: `http://localhost:8080`

---

## Databases

### SQLite

> SQLite is a file-based database - no Docker container needed.
> The database is just a file in your project directory.

**Setup:**

```bash
# Create database file
touch database/database.sqlite
```

**Connection examples:**

```php
// PHP / Laravel (.env)
DB_CONNECTION=sqlite
DB_DATABASE=/app/database/database.sqlite
```

```javascript
// Node.js (better-sqlite3)
const Database = require('better-sqlite3');
const db = new Database('./database/database.sqlite');
```

```python
# Python
import sqlite3
conn = sqlite3.connect('./database/database.sqlite')
```

> SQLite support (`pdo_sqlite`) is already included in the PHP Dockerfiles above.

| Pros | Cons |
|------|------|
| No container needed | Not for high concurrency |
| Zero configuration | Single file = harder to scale |
| Great for development | No network access (local only) |

---

### MySQL

**docker-compose.yml**

```yaml
services:
  mysql:
    image: mysql:8.0
    container_name: mysql-container
    environment:
      MYSQL_ROOT_PASSWORD: secret
    ports:
      - "3306:3306"
    volumes:
      - ./mysql-data:/var/lib/mysql
    restart: unless-stopped
    networks:
      - app_network

networks:
  app_network:
    driver: bridge
```

| Field | From Container | From Host |
|-------|----------------|-----------|
| Host | `mysql` | `localhost` |
| Port | `3306` | `3306` |
| User | `root` | `root` |
| Password | `secret` | `secret` |

---

### PostgreSQL

**docker-compose.yml**

```yaml
services:
  postgres:
    image: postgres:16
    container_name: postgres-container
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - app_network

networks:
  app_network:
    driver: bridge
```

| Field | From Container | From Host |
|-------|----------------|-----------|
| Host | `postgres` | `localhost` |
| Port | `5432` | `5432` |
| User | `postgres` | `postgres` |
| Password | `secret` | `secret` |

---

### Redis

**docker-compose.yml**

```yaml
services:
  redis:
    image: redis:alpine
    container_name: redis-container
    ports:
      - "6379:6379"
    volumes:
      - ./redis-data:/data
    restart: unless-stopped
    networks:
      - app_network

networks:
  app_network:
    driver: bridge
```

| Field | From Container | From Host |
|-------|----------------|-----------|
| Host | `redis` | `localhost` |
| Port | `6379` | `6379` |

**Connection examples:**

```php
// Laravel (.env)
REDIS_HOST=redis
REDIS_PORT=6379
```

```javascript
// Node.js (ioredis)
const Redis = require('ioredis')
const redis = new Redis({ host: 'redis', port: 6379 })
```

```python
# Python
import redis
r = redis.Redis(host='redis', port=6379)
```

> Add `redis-data` to `.gitignore`.

---

## Environment Variables (.env)

Docker Compose automatically loads a `.env` file from the project root. Use it to pass variables into your containers without hardcoding values.

**.env**

```env
APP_PORT=3000
DB_PASSWORD=secret
NODE_ENV=development
```

**docker-compose.yml**

```yaml
services:
  node:
    environment:
      - NODE_ENV=${NODE_ENV}
      - DB_PASSWORD=${DB_PASSWORD}
    ports:
      - "${APP_PORT}:${APP_PORT}"
```

> Never commit `.env` to git — add it to `.gitignore`. Commit a `.env.example` instead with placeholder values.

---

## Production

In production, run Nginx as a **separate container** in front of your app containers. It handles SSL, domain routing, and acts as a reverse proxy — your apps stay unchanged.

```
Internet → Nginx (80/443) → app containers (internal ports)
```

**docker-compose.yml**

```yaml
services:
  nginx:
    image: nginx:alpine
    container_name: nginx-container
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - /etc/letsencrypt:/etc/letsencrypt:ro
    networks:
      - app_network
    depends_on:
      - node
      - python
      - php

networks:
  app_network:
    driver: bridge
```

**nginx/nginx.conf**

```nginx
events {}

http {
    # Node.js app
    server {
        listen 443 ssl;
        server_name node.yourdomain.com;

        ssl_certificate /etc/letsencrypt/live/node.yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/node.yourdomain.com/privkey.pem;

        location / {
            proxy_pass http://node:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }

    # Python app
    server {
        listen 443 ssl;
        server_name python.yourdomain.com;

        ssl_certificate /etc/letsencrypt/live/python.yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/python.yourdomain.com/privkey.pem;

        location / {
            proxy_pass http://python:4000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }

    # PHP/Laravel app
    server {
        listen 443 ssl;
        server_name php.yourdomain.com;

        ssl_certificate /etc/letsencrypt/live/php.yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/php.yourdomain.com/privkey.pem;

        root /app/public;
        index index.php;

        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }

        location ~ \.php$ {
            fastcgi_pass php:9000;
            fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
            include fastcgi_params;
        }
    }

    # Redirect HTTP to HTTPS
    server {
        listen 80;
        server_name _;
        return 301 https://$host$request_uri;
    }
}
```

> SSL certificates can be obtained free via [Let's Encrypt](https://letsencrypt.org/) using Certbot.
> PHP-FPM speaks FastCGI (not HTTP), so it uses `fastcgi_pass` instead of `proxy_pass`.

---

## Notes

- Database data is persisted in `mysql-data` and `postgres-data` directories
- Add these directories to `.gitignore`
- Modify `command` field to match your project structure
- For production, use the Nginx setup above — do not use dev servers (`artisan serve`, Flask dev, `npm run dev`) in production
