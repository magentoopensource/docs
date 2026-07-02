---
title: "Docker Development Environment Setup for Magento 2"
description: "Complete guide to building a production-ready Docker-based Magento 2.4.7+ development environment with PHP 8.2, MySQL 8, Elasticsearch, Redis, Xdebug, and Mailhog"
type: "tutorial"
tier: 2
difficulty: "intermediate"
version: "2.4.7+"
estimated_time: "90 minutes"
topics:
  - docker
  - development-environment
  - php-8.2
  - magento-2.4.7
  - xdebug
  - elasticsearch
  - redis
  - local-development
last_updated: "2026-02-07"
---

# Docker Development Environment Setup for Magento 2

## Learning Objectives

By completing this tutorial, you will:

- Build a complete Docker-based Magento 2.4.7+ development environment
- Configure PHP 8.2, MySQL 8, Elasticsearch 8, Redis 7, and supporting services
- Set up Xdebug 3.x for debugging in VS Code and PHPStorm
- Install Magento with sample data for realistic testing
- Understand Docker networking, volumes, and service orchestration
- Troubleshoot common Docker/Magento development issues
- Optimize container performance for local development

## Introduction

A proper local development environment is foundational to productive Magento 2 work. Docker provides consistency across team members, eliminates "works on my machine" problems, and closely mirrors production infrastructure. This guide builds a complete, production-grade development stack that balances performance with ease of use.

### Why Docker for Magento 2?

- **Consistency**: Identical environment across all developers and CI/CD
- **Isolation**: Multiple Magento projects without port/version conflicts
- **Speed**: Faster than VMs; leverages host resources efficiently
- **Reproducibility**: Infrastructure as code (docker-compose.yml)
- **Production parity**: Match PHP, MySQL, Elasticsearch versions exactly

### Architecture Overview

The environment consists of seven services:

1. **nginx**: Web server (port 80/443)
2. **php-fpm**: PHP 8.2 with Xdebug, Composer, required extensions
3. **mysql**: MySQL 8.0 database
4. **elasticsearch**: Elasticsearch 8.x for catalog search
5. **redis**: Redis 7 for cache and session storage
6. **mailhog**: Email capture for testing (SMTP on 1025, UI on 8025)
7. **rabbitmq**: Message queue for asynchronous operations (optional but recommended)

## Step 1: Project Structure Setup

Create the project directory structure:

```bash
mkdir -p ~/magento2-docker/{nginx,docker}
cd ~/magento2-docker
```

Directory layout:

```
magento2-docker/
├── docker-compose.yml
├── .env
├── nginx/
│   └── default.conf
├── docker/
│   ├── php/
│   │   ├── Dockerfile
│   │   └── php.ini
│   └── mysql/
│       └── my.cnf
└── src/                    # Magento codebase goes here
```

## Step 2: Docker Compose Configuration

Create `docker-compose.yml`:

```yaml
version: '3.9'

services:
  nginx:
    image: nginx:1.25-alpine
    container_name: magento2_nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./src:/var/www/html:delegated
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - php-fpm
    networks:
      - magento

  php-fpm:
    build:
      context: ./docker/php
      dockerfile: Dockerfile
    container_name: magento2_php
    volumes:
      - ./src:/var/www/html:delegated
      - ./docker/php/php.ini:/usr/local/etc/php/conf.d/custom.ini:ro
      - ~/.composer:/var/www/.composer:delegated
    environment:
      PHP_IDE_CONFIG: "serverName=magento2.local"
      XDEBUG_CONFIG: "client_host=host.docker.internal"
      COMPOSER_MEMORY_LIMIT: -1
    networks:
      - magento
    extra_hosts:
      - "host.docker.internal:host-gateway"

  mysql:
    image: mysql:8.0
    container_name: magento2_mysql
    command: --default-authentication-plugin=mysql_native_password
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: magento2
      MYSQL_USER: magento2
      MYSQL_PASSWORD: magento2
    volumes:
      - mysql_data:/var/lib/mysql
      - ./docker/mysql/my.cnf:/etc/mysql/conf.d/custom.cnf:ro
    networks:
      - magento

  elasticsearch:
    image: elasticsearch:8.11.0
    container_name: magento2_elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    networks:
      - magento
    ulimits:
      memlock:
        soft: -1
        hard: -1

  redis:
    image: redis:7.2-alpine
    container_name: magento2_redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - magento

  mailhog:
    image: mailhog/mailhog:latest
    container_name: magento2_mailhog
    ports:
      - "1025:1025"
      - "8025:8025"
    networks:
      - magento

  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    container_name: magento2_rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: magento2
      RABBITMQ_DEFAULT_PASS: magento2
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - magento

networks:
  magento:
    driver: bridge

volumes:
  mysql_data:
  elasticsearch_data:
  redis_data:
  rabbitmq_data:
```

## Step 3: PHP-FPM Docker Image

Create `docker/php/Dockerfile`:

```dockerfile
FROM php:8.2-fpm

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    libzip-dev \
    libicu-dev \
    libxslt1-dev \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libwebp-dev \
    libsodium-dev \
    mariadb-client \
    vim \
    unzip \
    && rm -rf /var/lib/apt/lists/*

# Install PHP extensions required by Magento 2.4.7
RUN docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp \
    && docker-php-ext-install -j$(nproc) \
    bcmath \
    gd \
    intl \
    mbstring \
    mysqli \
    pdo_mysql \
    soap \
    xsl \
    zip \
    sockets \
    opcache \
    sodium

# Install Xdebug 3.x for PHP 8.2
RUN pecl install xdebug-3.3.1 && docker-php-ext-enable xdebug

# Configure Xdebug for remote debugging
RUN echo "zend_extension=xdebug.so" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini \
    && echo "xdebug.mode=develop,debug" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini \
    && echo "xdebug.client_host=host.docker.internal" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini \
    && echo "xdebug.client_port=9003" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini \
    && echo "xdebug.start_with_request=yes" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini \
    && echo "xdebug.log=/tmp/xdebug.log" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini \
    && echo "xdebug.idekey=PHPSTORM" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini

# Install Composer 2.x
COPY --from=composer:2.6 /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www/html

# Create www-data user with UID 1000 (match host user)
RUN usermod -u 1000 www-data && groupmod -g 1000 www-data

# Switch to www-data for Composer operations
USER www-data
```

Create `docker/php/php.ini`:

```ini
; Memory limits
memory_limit = 4G
max_execution_time = 1800
max_input_time = 1800

; Upload limits
upload_max_filesize = 64M
post_max_size = 64M

; OPcache settings for development
opcache.enable = 1
opcache.enable_cli = 1
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 100000
opcache.validate_timestamps = 1
opcache.revalidate_freq = 0
opcache.fast_shutdown = 1

; Error reporting
display_errors = On
display_startup_errors = On
error_reporting = E_ALL

; Session settings
session.save_handler = redis
session.save_path = "tcp://redis:6379?database=0"

; Realpath cache (performance)
realpath_cache_size = 10M
realpath_cache_ttl = 7200

; Timezone
date.timezone = UTC
```

## Step 4: Nginx Configuration

Create `nginx/default.conf`:

```nginx
upstream fastcgi_backend {
    server php-fpm:9000;
}

server {
    listen 80;
    server_name magento2.local;

    set $MAGE_ROOT /var/www/html;
    set $MAGE_MODE developer;

    root $MAGE_ROOT/pub;

    index index.php;
    autoindex off;
    charset UTF-8;
    error_page 404 403 = /errors/404.php;

    # Deny access to sensitive files
    location ~* ^/(?:\.git|\.htaccess|\.user\.ini|\.php_cs|\.php_cs\.dist|auth\.json|composer\.json|composer\.lock|magento_version|php\.ini\.sample) {
        deny all;
    }

    # PHP entry point for setup application
    location ~* ^/setup($|/) {
        root $MAGE_ROOT;
        location ~ ^/setup/index.php {
            fastcgi_pass fastcgi_backend;
            fastcgi_param PHP_FLAG "session.auto_start=off \n suhosin.session.cryptua=off";
            fastcgi_param PHP_VALUE "memory_limit=756M \n max_execution_time=600";
            fastcgi_read_timeout 600s;
            fastcgi_connect_timeout 600s;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
        }

        location ~ ^/setup/(?!pub/). {
            deny all;
        }

        location ~ ^/setup/pub/ {
            add_header X-Frame-Options "SAMEORIGIN";
        }
    }

    # PHP entry point for update application
    location ~* ^/update($|/) {
        root $MAGE_ROOT;

        location ~ ^/update/index.php {
            fastcgi_split_path_info ^(/update/index.php)(/.+)$;
            fastcgi_pass fastcgi_backend;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
            include fastcgi_params;
        }

        location ~ ^/update/(?!pub/). {
            deny all;
        }

        location ~ ^/update/pub/ {
            add_header X-Frame-Options "SAMEORIGIN";
        }
    }

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location /pub/ {
        location ~ ^/pub/media/(downloadable|customer|import|custom_options|theme_customization/.*\.xml) {
            deny all;
        }
        alias $MAGE_ROOT/pub/;
        add_header X-Frame-Options "SAMEORIGIN";
    }

    location /static/ {
        expires max;

        location ~ ^/static/version\d*/ {
            rewrite ^/static/version\d*/(.*)$ /static/$1 last;
        }

        location ~* \.(ico|jpg|jpeg|png|gif|svg|svgz|webp|avif|avifs|js|css|eot|ttf|otf|woff|woff2|html|json|webmanifest)$ {
            add_header Cache-Control "public";
            add_header X-Frame-Options "SAMEORIGIN";
            expires +1y;

            if (!-f $request_filename) {
                rewrite ^/static/(version\d*/)?(.*)$ /static.php?resource=$2 last;
            }
        }

        location ~* \.(zip|gz|gzip|bz2|csv|xml)$ {
            add_header Cache-Control "no-store";
            add_header X-Frame-Options "SAMEORIGIN";
            expires off;

            if (!-f $request_filename) {
                rewrite ^/static/(version\d*/)?(.*)$ /static.php?resource=$2 last;
            }
        }

        if (!-f $request_filename) {
            rewrite ^/static/(version\d*/)?(.*)$ /static.php?resource=$2 last;
        }

        add_header X-Frame-Options "SAMEORIGIN";
    }

    location /media/ {
        try_files $uri $uri/ /get.php$is_args$args;

        location ~ ^/media/theme_customization/.*\.xml {
            deny all;
        }

        location ~* \.(ico|jpg|jpeg|png|gif|svg|svgz|webp|avif|avifs|js|css|eot|ttf|otf|woff|woff2)$ {
            add_header Cache-Control "public";
            add_header X-Frame-Options "SAMEORIGIN";
            expires +1y;
            try_files $uri $uri/ /get.php$is_args$args;
        }

        location ~* \.(zip|gz|gzip|bz2|csv|xml)$ {
            add_header Cache-Control "no-store";
            add_header X-Frame-Options "SAMEORIGIN";
            expires off;
            try_files $uri $uri/ /get.php$is_args$args;
        }

        add_header X-Frame-Options "SAMEORIGIN";
    }

    location /media/customer/ {
        deny all;
    }

    location /media/downloadable/ {
        deny all;
    }

    location /media/import/ {
        deny all;
    }

    location /media/custom_options/ {
        deny all;
    }

    location /errors/ {
        location ~* \.xml$ {
            deny all;
        }
    }

    # PHP entry point for main application
    location ~ ^/(index|get|static|errors/report|errors/404|errors/503|health_check)\.php$ {
        try_files $uri =404;
        fastcgi_pass fastcgi_backend;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;

        fastcgi_param PHP_FLAG "session.auto_start=off \n suhosin.session.cryptua=off";
        fastcgi_param PHP_VALUE "memory_limit=756M \n max_execution_time=18000";
        fastcgi_read_timeout 600s;
        fastcgi_connect_timeout 600s;

        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Deny everything else in .php
    location ~* \.php$ {
        deny all;
    }

    gzip on;
    gzip_disable "msie6";

    gzip_comp_level 6;
    gzip_min_length 1100;
    gzip_buffers 16 8k;
    gzip_proxied any;
    gzip_types
        text/plain
        text/css
        text/js
        text/xml
        text/javascript
        application/javascript
        application/x-javascript
        application/json
        application/xml
        application/xml+rss
        image/svg+xml;
    gzip_vary on;
}
```

## Step 5: MySQL Configuration

Create `docker/mysql/my.cnf`:

```ini
[mysqld]
# InnoDB Settings
innodb_buffer_pool_size = 2G
innodb_log_file_size = 512M
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
innodb_file_per_table = 1

# General Settings
max_allowed_packet = 64M
max_connections = 500
table_open_cache = 4000
table_definition_cache = 4000
tmp_table_size = 256M
max_heap_table_size = 256M

# Query Cache (disabled in MySQL 8)
query_cache_size = 0
query_cache_type = 0

# Logging
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

# Character Set
character_set_server = utf8mb4
collation_server = utf8mb4_unicode_ci

# Binary Logging (can disable for dev)
skip-log-bin
```

## Step 6: Start Docker Environment

Build and start all services:

```bash
docker-compose up -d --build
```

Verify all containers are running:

```bash
docker-compose ps
```

Expected output: All services should show "Up" status.

Check service logs:

```bash
docker-compose logs -f php-fpm
docker-compose logs -f nginx
docker-compose logs -f elasticsearch
```

## Step 7: Install Magento 2.4.7

Enter the PHP container:

```bash
docker exec -it magento2_php bash
```

Create Magento authentication file (inside container):

```bash
cd /var/www/html
composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition=2.4.7 .
```

Enter your Magento authentication keys when prompted:
- Username: Your public key
- Password: Your private key

Install Magento:

```bash
bin/magento setup:install \
    --base-url=http://magento2.local/ \
    --db-host=mysql \
    --db-name=magento2 \
    --db-user=magento2 \
    --db-password=magento2 \
    --admin-firstname=Admin \
    --admin-lastname=User \
    --admin-email=admin@example.com \
    --admin-user=admin \
    --admin-password=Admin123! \
    --language=en_US \
    --currency=USD \
    --timezone=America/Chicago \
    --use-rewrites=1 \
    --search-engine=elasticsearch8 \
    --elasticsearch-host=elasticsearch \
    --elasticsearch-port=9200 \
    --elasticsearch-index-prefix=magento2 \
    --session-save=redis \
    --session-save-redis-host=redis \
    --session-save-redis-port=6379 \
    --session-save-redis-db=2 \
    --cache-backend=redis \
    --cache-backend-redis-server=redis \
    --cache-backend-redis-port=6379 \
    --cache-backend-redis-db=0 \
    --page-cache=redis \
    --page-cache-redis-server=redis \
    --page-cache-redis-port=6379 \
    --page-cache-redis-db=1 \
    --amqp-host=rabbitmq \
    --amqp-port=5672 \
    --amqp-user=magento2 \
    --amqp-password=magento2 \
    --consumers-wait-for-messages=0
```

Configure developer mode:

```bash
bin/magento deploy:mode:set developer
bin/magento module:disable Magento_AdminAdobeImsTwoFactorAuth Magento_TwoFactorAuth
bin/magento cache:flush
```

## Step 8: Install Sample Data (Optional)

Inside the PHP container:

```bash
bin/magento sampledata:deploy
bin/magento setup:upgrade
bin/magento indexer:reindex
bin/magento cache:flush
```

Sample data installation takes 10-15 minutes.

## Step 9: Configure Host File

Add to `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows):

```
127.0.0.1 magento2.local
```

## Step 10: Xdebug Configuration

### VS Code Setup

Create `.vscode/launch.json` in project root:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Listen for Xdebug",
            "type": "php",
            "request": "launch",
            "port": 9003,
            "pathMappings": {
                "/var/www/html": "${workspaceFolder}/src"
            },
            "log": true,
            "xdebugSettings": {
                "max_data": 65535,
                "show_hidden": 1,
                "max_children": 100,
                "max_depth": 5
            }
        }
    ]
}
```

Install PHP Debug extension (xdebug.php-debug) in VS Code.

Set breakpoint in `src/pub/index.php`, start debugging (F5), refresh browser.

### PHPStorm Setup

1. Go to Settings > PHP > Servers
2. Add new server:
   - Name: `magento2.local`
   - Host: `magento2.local`
   - Port: `80`
   - Debugger: Xdebug
   - Use path mappings: Check
   - Map `<Project root>/src` to `/var/www/html`

3. Go to Run > Start Listening for PHP Debug Connections
4. Install Xdebug helper browser extension
5. Set breakpoint, enable debugging, refresh page

## Step 11: Verify Installation

Access frontend: http://magento2.local

Access admin: http://magento2.local/admin

- Username: `admin`
- Password: `Admin123!`

View Mailhog UI: http://localhost:8025

View RabbitMQ UI: http://localhost:15672 (magento2/magento2)

Test Elasticsearch:

```bash
curl http://localhost:9200/_cat/indices?v
```

## Common Troubleshooting

### Permission Issues

Container cannot write to directories:

```bash
# On host machine
cd ~/magento2-docker
sudo chown -R 1000:1000 src/
chmod -R 775 src/var src/generated src/pub/static src/pub/media
```

### Memory Issues

"Allowed memory size exhausted" errors:

1. Increase PHP memory in `docker/php/php.ini`: `memory_limit = 4G`
2. Increase Docker Desktop memory allocation (Settings > Resources > Memory: 8GB+)
3. Restart containers: `docker-compose restart php-fpm`

### Elasticsearch Fails to Start

Increase virtual memory on host:

```bash
# Linux
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

# Mac/Windows (Docker Desktop settings)
# Resources > Advanced > VM max_map_count: 262144
```

### Xdebug Not Connecting

1. Verify Xdebug is loaded:

```bash
docker exec -it magento2_php php -v
# Should show "with Xdebug v3.3.1"
```

2. Check Xdebug log:

```bash
docker exec -it magento2_php cat /tmp/xdebug.log
```

3. Verify IDE is listening on port 9003

4. On Windows/Mac, confirm `host.docker.internal` resolves:

```bash
docker exec -it magento2_php ping host.docker.internal
```

### Slow Performance on Mac

Use Docker Desktop 4.x+ with VirtioFS:

1. Settings > General > Enable VirtioFS
2. Restart Docker Desktop

Alternative: Use `:cached` or `:delegated` volume flags (already in compose file)

### Static Content Not Loading

Deploy static content:

```bash
docker exec -it magento2_php bin/magento setup:static-content:deploy -f
```

Check permissions:

```bash
docker exec -it magento2_php ls -la pub/static
# Should be owned by www-data (UID 1000)
```

### Database Connection Refused

Verify MySQL is running and accepting connections:

```bash
docker exec -it magento2_mysql mysql -umagento2 -pmagento2 -e "SHOW DATABASES;"
```

Check `app/etc/env.php` database configuration matches `docker-compose.yml`.

### Composer Authentication Failed

Store credentials permanently:

```bash
docker exec -it magento2_php composer config --global http-basic.repo.magento.com <public_key> <private_key>
```

## Security Considerations

### Development vs Production

This setup is for **local development only**. For production:

1. Remove Xdebug (significant performance impact)
2. Enable OPcache with `validate_timestamps = 0`
3. Use environment-specific secrets (never commit credentials)
4. Enable TLS/SSL with valid certificates
5. Harden Nginx (rate limiting, ModSecurity, etc.)
6. Use production-grade MySQL configuration (enable binary logs, replication)
7. Implement backup strategies for data volumes

### Secrets Management

Never commit:
- `auth.json` (Magento keys)
- `app/etc/env.php` (database credentials)
- `.env` files with secrets

Use `.gitignore`:

```
src/auth.json
src/app/etc/env.php
src/var/
src/generated/
src/pub/static/
.env
```

### Container Security

- Run containers as non-root users (PHP container uses www-data UID 1000)
- Keep base images updated: `docker-compose pull && docker-compose up -d`
- Scan images for vulnerabilities: `docker scan magento2_php`
- Limit container capabilities and resources in production

## Performance Optimization

### Host Resource Allocation

Minimum recommended for Docker Desktop:

- Memory: 8GB (16GB preferred)
- CPUs: 4 cores (8 preferred)
- Disk: 100GB available space (SSD strongly recommended)

### File Sync Optimization

On Mac/Windows, use Mutagen or docker-sync for faster file sync:

```bash
# Install Mutagen
brew install mutagen-io/mutagen/mutagen

# Create mutagen.yml
mutagen:
  sync:
    defaults:
      mode: "two-way-resolved"
      ignore:
        vcs: true
        paths:
          - "var/"
          - "generated/"
          - "pub/static/"
```

### OPcache Tuning

For development, `validate_timestamps = 1` (auto-reload on change).

For production, set `validate_timestamps = 0` and deploy with:

```bash
docker exec -it magento2_php service php8.2-fpm reload
```

### Redis Configuration

Split Redis into three databases for optimal cache isolation:

- DB 0: Default cache (app/etc/env.php `cache/frontend/default`)
- DB 1: Page cache (app/etc/env.php `cache/frontend/page_cache`)
- DB 2: Session storage (app/etc/env.php `session`)

Already configured in Step 7 installation command.

### MySQL Query Performance

Enable slow query log (already in my.cnf) and analyze:

```bash
docker exec -it magento2_mysql mysql -uroot -proot -e "SELECT * FROM mysql.slow_log ORDER BY query_time DESC LIMIT 10;"
```

## Additional Development Tools

### Debugging Tools

**Mailhog**: Capture all outbound emails

```bash
# Configure Magento to use Mailhog (already in docker-compose.yml)
# Access UI: http://localhost:8025
```

**Redis Commander**: GUI for Redis inspection

Add to `docker-compose.yml`:

```yaml
  redis-commander:
    image: rediscommander/redis-commander:latest
    container_name: magento2_redis_commander
    environment:
      REDIS_HOSTS: local:redis:6379
    ports:
      - "8081:8081"
    networks:
      - magento
```

**Portainer**: Docker container management UI

```bash
docker run -d -p 9000:9000 --name portainer --restart always -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer-ce
```

Access: http://localhost:9000

### CLI Aliases

Add to `~/.bashrc` or `~/.zshrc`:

```bash
alias m2='docker exec -it magento2_php bin/magento'
alias m2bash='docker exec -it magento2_php bash'
alias m2composer='docker exec -it magento2_php composer'
alias m2mysql='docker exec -it magento2_mysql mysql -umagento2 -pmagento2 magento2'
```

Usage:

```bash
m2 cache:flush
m2composer require vendor/package
m2mysql -e "SELECT * FROM core_config_data WHERE path LIKE '%base_url%';"
```

## Maintenance Tasks

### Start Environment

```bash
cd ~/magento2-docker
docker-compose up -d
```

### Stop Environment

```bash
docker-compose down
```

### Remove Everything (including volumes)

```bash
docker-compose down -v
```

WARNING: This deletes the database and all data.

### View Container Logs

```bash
docker-compose logs -f php-fpm nginx mysql
```

### Update Dependencies

```bash
docker exec -it magento2_php composer update --with-all-dependencies
docker exec -it magento2_php bin/magento setup:upgrade
docker exec -it magento2_php bin/magento cache:flush
```

### Rebuild Containers

After changing Dockerfile or docker-compose.yml:

```bash
docker-compose down
docker-compose build --no-cache
docker-compose up -d
```

## Assumptions

- Target: Adobe Commerce / Magento Open Source 2.4.7
- PHP: 8.2
- MySQL: 8.0
- Elasticsearch: 8.11
- Host: Linux/Mac/Windows with Docker Desktop 20.10+
- Environment: Local development (not cloud, not production)
- Scope: Single store, single website

## Why This Approach

- **Docker Compose**: Standardized orchestration; easier than raw Docker commands
- **Separate containers**: Each service isolated; easy to upgrade/replace individual components
- **Named volumes**: Persist data across container restarts; faster than bind mounts
- **Delegated volumes**: Optimize file sync performance on Mac/Windows (`:delegated` flag)
- **Xdebug 3.x**: Latest version; improved performance over Xdebug 2.x
- **Redis split**: Three databases for cache/session isolation reduces key collision
- **OPcache enabled**: Balances dev (validate_timestamps=1) and performance
- **Mailhog**: Zero-config email testing; no SMTP relay needed
- **RabbitMQ**: Asynchronous processing (cron, indexing) offloaded from web requests

Alternatives considered:
- **Vagrant**: Slower, heavier than Docker; deprecated by most teams
- **MAMP/WAMP**: Not portable; version management difficult
- **Native PHP**: Environment conflicts; hard to match production versions
- **Kubernetes**: Overkill for local dev; use for staging/prod only

## Security Impact

### Development Environment Risks

- **Xdebug enabled**: Remote code execution if exposed publicly (mitigated: only bind to localhost)
- **Default credentials**: MySQL root/root, admin Admin123! (acceptable for local dev; never use in production)
- **No TLS**: HTTP only (acceptable for local; production requires HTTPS)
- **Docker socket exposure**: Running as root in containers (mitigated: www-data UID 1000 for PHP)

### Mitigations

- Bind services to localhost only (ports not exposed to network)
- Use firewall rules to block external access to Docker ports
- Never commit `auth.json` or `env.php` to version control
- Regularly update base images: `docker-compose pull`

### Authentication/Authorization

- Admin panel uses Magento ACL (Access Control List)
- API authentication via OAuth/token (configure in `app/etc/env.php`)
- Two-factor authentication disabled for dev (enabled in production)

### CSRF/Form Keys

- Magento automatically generates form keys for admin/frontend forms
- Ensure `formkey` is rendered in custom forms: `<?= $block->getBlockHtml('formkey') ?>`

### XSS Escaping

- Use `escapeHtml()`, `escapeUrl()`, `escapeJs()` in templates
- Never use `{variable|raw}` in templates without sanitization
- Content Security Policy (CSP) enforcement in Magento 2.4+

### PII/GDPR

- Sample data includes fake customer PII (safe for dev)
- Production: Implement data export/deletion per GDPR requirements
- Never use production database dumps in local dev without anonymization

### Secrets Management

- Store Magento keys in `~/.composer/auth.json` (not in project)
- Use `.env` for environment-specific secrets (add to `.gitignore`)
- Production: Use Vault, AWS Secrets Manager, or Azure Key Vault

## Performance Impact

### Full Page Cache (FPC)

- Redis page cache (DB 1) pre-configured in installation command
- Developer mode disables FPC; enable in production mode
- Varnish not included (optional; add varnish service for production-like testing)

### Database Load

- MySQL 8.0 optimized for InnoDB (2GB buffer pool)
- Indexers run on-save in developer mode (switch to schedule for production)
- Slow query log captures queries >2s (analyze regularly)

### Core Web Vitals (CWV)

Not applicable to local dev (no CDN, no real network latency).

For production:
- Use CDN for static assets
- Enable HTTP/2 (nginx 1.25 supports it; add TLS config)
- Lazy-load images, defer non-critical JS
- Measure with Lighthouse: `npx lighthouse http://magento2.local`

### Cacheability

- Full page cache: Redis (page_cache)
- Block cache: Redis (default cache)
- Layout cache: Redis (default cache)
- Config cache: Redis (default cache)

Verify cache hits:

```bash
docker exec -it magento2_redis redis-cli INFO stats | grep hits
```

### Resource Usage

- Idle: ~4GB RAM, 10% CPU
- Under load (reindex, compile): ~8GB RAM, 80% CPU
- Elasticsearch: ~1GB RAM (ES_JAVA_OPTS="-Xms1g -Xmx1g")
- MySQL: ~2GB RAM (innodb_buffer_pool_size=2G)

## Backward Compatibility

### Magento Versions

- Works with Magento 2.4.6 - 2.4.8 (minor PHP/Elasticsearch version adjustments)
- For Magento 2.4.5 and below: Use Elasticsearch 7.x instead of 8.x
- For Magento 2.4.7+: PHP 8.2 or 8.3 required

### PHP Versions

- PHP 8.2: Current setup
- PHP 8.3: Change `FROM php:8.3-fpm` in Dockerfile; test extensions compatibility
- PHP 8.1: Downgrade to `FROM php:8.1-fpm`; Magento 2.4.7 supports 8.1-8.3

### Migration Path

From existing local environment:

1. Export database: `mysqldump magento2 > magento2.sql`
2. Copy codebase to `src/`
3. Start Docker environment
4. Import database: `docker exec -i magento2_mysql mysql -umagento2 -pmagento2 magento2 < magento2.sql`
5. Update `app/etc/env.php` with Docker service hostnames (mysql, redis, elasticsearch)
6. Run `bin/magento setup:upgrade && bin/magento cache:flush`

## Tests to Add

### Environment Verification Tests

After setup, verify:

1. **Service availability**: `curl http://magento2.local` returns 200
2. **Admin login**: Login to http://magento2.local/admin succeeds
3. **Elasticsearch**: `curl http://localhost:9200/_cat/health` returns green/yellow
4. **Redis**: `docker exec -it magento2_redis redis-cli PING` returns PONG
5. **Xdebug**: Set breakpoint, trigger request, verify IDE breaks
6. **Mailhog**: Trigger password reset email, verify in Mailhog UI
7. **RabbitMQ**: `bin/magento queue:consumers:list` shows consumers

### Automated Health Checks

Add to `docker-compose.yml`:

```yaml
  nginx:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health_check.php"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### Unit Tests

Run Magento unit tests:

```bash
docker exec -it magento2_php vendor/bin/phpunit -c dev/tests/unit/phpunit.xml.dist
```

### Integration Tests

Configure and run integration tests:

```bash
docker exec -it magento2_php cp dev/tests/integration/etc/install-config-mysql.php.dist dev/tests/integration/etc/install-config-mysql.php
# Edit install-config-mysql.php with Docker service hostnames
docker exec -it magento2_php vendor/bin/phpunit -c dev/tests/integration/phpunit.xml.dist
```

## Documentation to Update

### Project README

Add section:

```markdown
## Development Environment

This project uses Docker for local development. See this guide for complete setup instructions.

Quick start:
1. Clone repo
2. `docker-compose up -d`
3. `docker exec -it magento2_php bin/magento setup:install ...`
4. Access http://magento2.local
```

### Troubleshooting Guide

Create `TROUBLESHOOTING.md` with:
- Common error messages and solutions
- Log file locations
- Debug commands
- Performance tuning tips

### Team Onboarding

Add to `CONTRIBUTING.md`:
- Required tools (Docker Desktop version)
- Magento authentication key setup
- First-time setup checklist
- Git workflow (branches, commits, PRs)

### Screenshots to Capture

For documentation:
1. Docker Desktop dashboard showing all containers running
2. VS Code debugging session with breakpoint hit
3. PHPStorm debugging session with variables inspector
4. Magento admin panel dashboard
5. Mailhog UI with captured emails
6. RabbitMQ management UI showing queues
7. Successful `docker-compose ps` output
8. Elasticsearch health check response

## Related Documentation

### Related Guides

- [CI/CD Deployment Pipelines for Magento 2](../how-to/cicd-deployment.md)
- [Comprehensive Testing Strategies for Magento 2](../how-to/testing-strategies.md)
