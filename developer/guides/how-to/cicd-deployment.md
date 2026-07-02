---
title: "CI/CD Deployment Pipelines for Magento 2"
description: "Developer guide: CI/CD Deployment Pipelines for Magento 2"
type: "how-to"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "35 minutes"
topics:
  - ci/cd
  - deployment
  - static content deployment
  - zero-downtime
  - devops
  - automation
last_updated: "2026-02-07"
---

# CI/CD Deployment Pipelines for Magento 2

## Overview

Reliable CI/CD pipelines are essential for Magento deployments at scale. This guide provides production-proven patterns for build artifacts, static content deployment, zero-downtime releases, and rollback procedures across GitHub Actions, GitLab CI, and cloud platforms.

**What you'll learn:**
- Build artifact strategy vs build-on-server approaches
- Static content deployment strategies (quick, compact, standard)
- Zero-downtime deployment patterns (blue-green, rolling)
- GitHub Actions and GitLab CI pipeline examples
- Rollback procedures and health checks
- Performance optimization for deployment speed

**Prerequisites:**
- Magento architecture knowledge (FPC, indexers, cron)
- Basic CI/CD concepts (pipelines, stages, artifacts)
- Linux system administration (SSH, systemd, file permissions)
- Git workflows (branching, tagging, releases)

---

## Deployment Strategies: Overview

### Build Artifact vs Build-on-Server

| Approach | Build Artifact | Build-on-Server |
|----------|----------------|-----------------|
| **Where code is built** | CI server | Production server |
| **Composer install** | CI stage | During deployment |
| **Static content** | Pre-compiled | On-demand or pre-deploy |
| **Deploy speed** | Fast (2-5 min) | Slow (10-20 min) |
| **Rollback** | Instant (swap symlink) | Rebuild required |
| **Disk space** | 2-3x (artifacts) | 1x (single codebase) |
| **Best for** | Production, staging | Development only |

**Recommendation:** Use build artifact strategy for all non-development environments.

---

## Build Artifact Pipeline

### Architecture

```
[Git Push] → [CI Build] → [Run Tests] → [Compile Assets] → [Package Artifact] → [Deploy to Server] → [Activate Release]
```

### Stage 1: Build and Test

**GitHub Actions example:**

```yaml
name: Build and Deploy

on:
  push:
    branches: [main, staging, production]
  pull_request:
    branches: [main]

env:
  PHP_VERSION: '8.2'
  NODE_VERSION: '18'
  COMPOSER_VERSION: '2'

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.artifact.outputs.name }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          extensions: bcmath, ctype, curl, dom, gd, intl, mbstring, pdo_mysql, simplexml, soap, xsl, zip
          tools: composer:v2

      - name: Get composer cache directory
        id: composer-cache
        run: echo "dir=$(composer config cache-files-dir)" >> $GITHUB_OUTPUT

      - name: Cache composer dependencies
        uses: actions/cache@v3
        with:
          path: ${{ steps.composer-cache.outputs.dir }}
          key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
          restore-keys: ${{ runner.os }}-composer-

      - name: Install dependencies
        run: composer install --no-dev --optimize-autoloader --no-interaction --prefer-dist

      - name: Run static analysis
        run: |
          vendor/bin/phpstan analyse app/code --level 8
          vendor/bin/phpcs app/code --standard=Magento2

      - name: Run unit tests
        run: vendor/bin/phpunit -c dev/tests/unit/phpunit.xml.dist

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install Node dependencies
        run: npm ci

      - name: Build frontend assets
        run: npm run build

      - name: Set deployment mode to production
        run: bin/magento deploy:mode:set production --skip-compilation

      - name: Compile DI
        run: bin/magento setup:di:compile

      - name: Deploy static content (all locales and themes)
        run: |
          bin/magento setup:static-content:deploy -f \
            en_US en_GB de_DE fr_FR \
            --theme Magento/luma \
            --theme Magento/backend \
            --jobs 4

      - name: Create artifact directory structure
        run: |
          mkdir -p artifact/magento
          rsync -a --exclude='.git' --exclude='node_modules' --exclude='dev' --exclude='var/log' . artifact/magento/

      - name: Generate artifact name
        id: artifact
        run: |
          ARTIFACT_NAME="magento-${{ github.sha }}-$(date +%Y%m%d-%H%M%S).tar.gz"
          echo "name=$ARTIFACT_NAME" >> $GITHUB_OUTPUT
          echo "Artifact name: $ARTIFACT_NAME"

      - name: Package artifact
        run: |
          cd artifact
          tar -czf ${{ steps.artifact.outputs.name }} magento/
          sha256sum ${{ steps.artifact.outputs.name }} > ${{ steps.artifact.outputs.name }}.sha256

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: ${{ steps.artifact.outputs.name }}
          path: |
            artifact/${{ steps.artifact.outputs.name }}
            artifact/${{ steps.artifact.outputs.name }}.sha256
          retention-days: 30
```

### Stage 2: Deploy to Staging

```yaml
  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/staging'
    environment:
      name: staging
      url: https://staging.example.com

    steps:
      - name: Download artifact
        uses: actions/download-artifact@v3
        with:
          name: ${{ needs.build.outputs.artifact-name }}

      - name: Verify artifact checksum
        run: |
          sha256sum -c ${{ needs.build.outputs.artifact-name }}.sha256
          echo "Artifact verified successfully"

      - name: Deploy to staging servers
        uses: appleboy/ssh-action@v0.1.10
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            set -e

            # Configuration
            DEPLOY_USER="www-data"
            DEPLOY_PATH="/var/www/magento"
            RELEASES_PATH="$DEPLOY_PATH/releases"
            SHARED_PATH="$DEPLOY_PATH/shared"
            CURRENT_PATH="$DEPLOY_PATH/current"
            ARTIFACT_NAME="${{ needs.build.outputs.artifact-name }}"
            RELEASE_NAME="$(date +%Y%m%d-%H%M%S)"
            RELEASE_PATH="$RELEASES_PATH/$RELEASE_NAME"

            # Create release directory
            mkdir -p $RELEASE_PATH

            # Extract artifact
            tar -xzf /tmp/$ARTIFACT_NAME -C $RELEASE_PATH --strip-components=1

            # Link shared directories
            ln -nfs $SHARED_PATH/var $RELEASE_PATH/var
            ln -nfs $SHARED_PATH/pub/media $RELEASE_PATH/pub/media
            ln -nfs $SHARED_PATH/app/etc/env.php $RELEASE_PATH/app/etc/env.php

            # Set permissions
            chown -R $DEPLOY_USER:$DEPLOY_USER $RELEASE_PATH
            find $RELEASE_PATH -type d -exec chmod 755 {} \;
            find $RELEASE_PATH -type f -exec chmod 644 {} \;
            chmod +x $RELEASE_PATH/bin/magento

            # Maintenance mode ON
            if [ -L $CURRENT_PATH ]; then
              php $CURRENT_PATH/bin/magento maintenance:enable
            fi

            # Database migrations
            php $RELEASE_PATH/bin/magento setup:upgrade --keep-generated

            # Reindex
            php $RELEASE_PATH/bin/magento indexer:reindex

            # Clear cache
            php $RELEASE_PATH/bin/magento cache:flush

            # Atomic symlink swap
            ln -nfs $RELEASE_PATH $CURRENT_PATH

            # Maintenance mode OFF
            php $CURRENT_PATH/bin/magento maintenance:disable

            # Reload PHP-FPM
            sudo systemctl reload php8.2-fpm

            # Keep only last 5 releases
            cd $RELEASES_PATH && ls -t | tail -n +6 | xargs rm -rf

            echo "Deployment completed successfully"

      - name: Run smoke tests
        run: |
          curl -f https://staging.example.com/health || exit 1
          curl -f https://staging.example.com/ | grep -q "Magento" || exit 1

      - name: Notify deployment
        uses: 8398a7/action-slack@v3
        if: always()
        with:
          status: ${{ job.status }}
          text: 'Staging deployment ${{ job.status }}'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### Stage 3: Deploy to Production (Manual Approval)

```yaml
  deploy-production:
    needs: [build, deploy-staging]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/production'
    environment:
      name: production
      url: https://www.example.com

    steps:
      # Similar to staging, with additional steps:
      # - Blue-green deployment pattern
      # - Canary release (10% → 50% → 100%)
      # - Advanced health checks
      # - Automatic rollback on errors
```

---

## Static Content Deployment Strategies

### Strategy Comparison

| Strategy | Speed | File Size | Use Case |
|----------|-------|-----------|----------|
| **Standard** | Slow (10-15 min) | Smallest | Production, multi-locale |
| **Quick** | Medium (5-7 min) | Medium | Single-locale stores |
| **Compact** | Fast (3-5 min) | Largest | Development, staging |

### Standard Strategy (Production)

**Pros:**
- Smallest file size (minified, merged)
- Best performance (fewer HTTP requests)
- Optimal for CDN distribution

**Cons:**
- Slowest deployment
- Requires all locales/themes specified

**Command:**

```bash
bin/magento setup:static-content:deploy \
  en_US de_DE fr_FR \
  --theme Magento/luma \
  --theme Magento/backend \
  --exclude-theme Magento/blank \
  --jobs 4 \
  --max-execution-time 3600 \
  --no-html-minify  # Optional: skip HTML minification for speed
```

**Optimization:**

```bash
# Pre-generate RequireJS config (faster than on-demand)
bin/magento setup:static-content:deploy --strategy=compact

# Use parallel jobs (CPU cores - 1)
--jobs $(( $(nproc) - 1 ))
```

### Quick Strategy (Staging)

**Pros:**
- Faster than standard
- Production-like assets
- Good for pre-production testing

**Cons:**
- Slightly larger files
- Less aggressive optimization

**Command:**

```bash
bin/magento setup:static-content:deploy -f -s quick \
  en_US \
  --theme Magento/luma \
  --jobs 4
```

### Compact Strategy

**Pros:**
- Fastest deployment — compiles all locales in a single pass
- Reduces duplication across locale-specific files

**Cons:**
- Less granular output (harder to debug locale-specific issues)
- All locales compiled together, cannot deploy selectively

> **Note:** Symlink behavior (no copying, instant changes) is a feature of
> **developer mode**, not the compact strategy. Do not confuse the two.

**Command:**

```bash
bin/magento setup:static-content:deploy -f -s compact
```

**Use in developer mode:**

```bash
bin/magento deploy:mode:set developer
# Static files generated on-demand, no pre-deployment needed
```

### Optimizing Static Content Deployment

**1. Deploy only changed themes:**

```bash
# Deploy admin theme only (after backend changes)
bin/magento setup:static-content:deploy --theme Magento/backend

# Deploy specific locale
bin/magento setup:static-content:deploy en_US --theme Magento/luma
```

**2. Exclude unnecessary locales:**

```bash
# In app/etc/config.php, set allowed locales
'system' => [
    'default' => [
        'general' => [
            'locale' => [
                'code' => 'en_US'
            ]
        ]
    ]
]
```

**3. Use CDN for static assets:**

```bash
# app/etc/env.php
'system' => [
    'default' => [
        'web' => [
            'unsecure' => [
                'base_static_url' => 'https://cdn.example.com/static/'
            ],
            'secure' => [
                'base_static_url' => 'https://cdn.example.com/static/'
            ]
        ]
    ]
]
```

**4. Cache frontend dependencies:**

```bash
# CI pipeline: cache node_modules and composer packages
- name: Cache Node modules
  uses: actions/cache@v3
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
```

---

## Zero-Downtime Deployment Patterns

### Blue-Green Deployment

**Architecture:**

```
                    Load Balancer
                   /              \
              [Blue]              [Green]
          (Current: v1.0)     (New: v1.1)
```

**Process:**

1. Deploy new version to Green environment
2. Run smoke tests on Green
3. Switch traffic from Blue to Green
4. Monitor metrics (error rates, response times)
5. If issues, instant rollback to Blue
6. After validation, Blue becomes next Green

**Implementation:**

```bash
#!/bin/bash
# blue-green-deploy.sh

set -e

BLUE_PATH="/var/www/magento-blue"
GREEN_PATH="/var/www/magento-green"
CURRENT_LINK="/var/www/magento"
ARTIFACT_PATH="/tmp/magento-artifact.tar.gz"

# Determine current and target
if [ "$(readlink $CURRENT_LINK)" == "$BLUE_PATH" ]; then
    CURRENT_ENV="blue"
    TARGET_ENV="green"
    TARGET_PATH=$GREEN_PATH
else
    CURRENT_ENV="green"
    TARGET_ENV="blue"
    TARGET_PATH=$BLUE_PATH
fi

echo "Current environment: $CURRENT_ENV"
echo "Deploying to: $TARGET_ENV"

# Extract artifact to target
rm -rf $TARGET_PATH/*
tar -xzf $ARTIFACT_PATH -C $TARGET_PATH --strip-components=1

# Link shared resources
ln -nfs /var/www/shared/var $TARGET_PATH/var
ln -nfs /var/www/shared/pub/media $TARGET_PATH/pub/media
ln -nfs /var/www/shared/app/etc/env.php $TARGET_PATH/app/etc/env.php

# Set permissions
chown -R www-data:www-data $TARGET_PATH

# Database migrations (apply to shared DB)
php $TARGET_PATH/bin/magento setup:upgrade --keep-generated

# Reindex
php $TARGET_PATH/bin/magento indexer:reindex

# Clear cache
php $TARGET_PATH/bin/magento cache:flush

# Health check
php $TARGET_PATH/bin/magento app:config:status || { echo "Health check failed"; exit 1; }

# Atomic switch
ln -nfs $TARGET_PATH $CURRENT_LINK

# Reload PHP-FPM (graceful reload)
sudo systemctl reload php8.2-fpm

# Reload Nginx
sudo nginx -s reload

echo "Deployment to $TARGET_ENV completed. Traffic switched."
```

**Nginx configuration:**

```nginx
upstream magento_backend {
    server unix:/var/run/php/php8.2-fpm.sock;
}

server {
    listen 80;
    server_name example.com;

    # Current symlink points to blue or green
    root /var/www/magento/pub;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        fastcgi_pass magento_backend;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

### Rolling Deployment (Multi-Server)

**Architecture:**

```
Load Balancer
    ├── Server 1 (update → activate)
    ├── Server 2 (update → activate)
    ├── Server 3 (update → activate)
    └── Server 4 (update → activate)
```

**Process:**

1. Remove Server 1 from load balancer
2. Deploy new version to Server 1
3. Validate Server 1 health
4. Add Server 1 back to load balancer
5. Repeat for Server 2, 3, 4...

**Ansible playbook example:**

```yaml
# deploy-rolling.yml
---
- name: Rolling deployment to production servers
  hosts: webservers
  serial: 1  # Deploy one server at a time
  max_fail_percentage: 0

  vars:
    artifact_url: "https://artifacts.example.com/magento-{{ build_id }}.tar.gz"
    deploy_path: "/var/www/magento"
    release_path: "{{ deploy_path }}/releases/{{ ansible_date_time.epoch }}"
    current_path: "{{ deploy_path }}/current"
    shared_path: "{{ deploy_path }}/shared"

  tasks:
    - name: Remove server from load balancer
      uri:
        url: "https://lb.example.com/api/servers/{{ inventory_hostname }}/disable"
        method: POST
        headers:
          Authorization: "Bearer {{ lb_api_token }}"
      delegate_to: localhost

    - name: Wait for active connections to drain
      wait_for:
        timeout: 30

    - name: Download artifact
      get_url:
        url: "{{ artifact_url }}"
        dest: "/tmp/magento-artifact.tar.gz"
        checksum: "sha256:{{ artifact_checksum }}"

    - name: Create release directory
      file:
        path: "{{ release_path }}"
        state: directory
        owner: www-data
        group: www-data

    - name: Extract artifact
      unarchive:
        src: "/tmp/magento-artifact.tar.gz"
        dest: "{{ release_path }}"
        remote_src: yes
        extra_opts: [--strip-components=1]

    - name: Link shared directories
      file:
        src: "{{ shared_path }}/{{ item }}"
        dest: "{{ release_path }}/{{ item }}"
        state: link
      loop:
        - var
        - pub/media
        - app/etc/env.php

    - name: Run database migrations
      command: php {{ release_path }}/bin/magento setup:upgrade --keep-generated
      run_once: true  # Only run on first server

    - name: Reindex
      command: php {{ release_path }}/bin/magento indexer:reindex

    - name: Clear cache
      command: php {{ release_path }}/bin/magento cache:flush

    - name: Switch symlink
      file:
        src: "{{ release_path }}"
        dest: "{{ current_path }}"
        state: link
        force: yes

    - name: Reload PHP-FPM
      systemd:
        name: php8.2-fpm
        state: reloaded

    - name: Health check
      uri:
        url: "http://localhost/health"
        status_code: 200
      retries: 5
      delay: 5

    - name: Add server back to load balancer
      uri:
        url: "https://lb.example.com/api/servers/{{ inventory_hostname }}/enable"
        method: POST
        headers:
          Authorization: "Bearer {{ lb_api_token }}"
      delegate_to: localhost

    - name: Cleanup old releases
      shell: ls -t {{ deploy_path }}/releases | tail -n +6 | xargs rm -rf
      args:
        chdir: "{{ deploy_path }}/releases"
```

---

## Health Checks and Validation

### Application Health Endpoint

**Create custom health check module:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Health;

use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Framework\Controller\Result\JsonFactory;
use Magento\Framework\App\ResourceConnection;
use Magento\Framework\App\CacheInterface;

class Check implements HttpGetActionInterface
{
    public function __construct(
        private readonly JsonFactory $resultJsonFactory,
        private readonly ResourceConnection $resourceConnection,
        private readonly CacheInterface $cache
    ) {}

    public function execute()
    {
        $result = $this->resultJsonFactory->create();
        $health = [
            'status' => 'healthy',
            'checks' => []
        ];

        // Database check
        try {
            $connection = $this->resourceConnection->getConnection();
            $connection->fetchOne('SELECT 1');
            $health['checks']['database'] = 'ok';
        } catch (\Exception $e) {
            $health['checks']['database'] = 'error: ' . $e->getMessage();
            $health['status'] = 'unhealthy';
        }

        // Cache check
        try {
            $this->cache->save('health_check', 'health_check', [], 60);
            $cached = $this->cache->load('health_check');
            $health['checks']['cache'] = $cached === 'health_check' ? 'ok' : 'error';
        } catch (\Exception $e) {
            $health['checks']['cache'] = 'error: ' . $e->getMessage();
        }

        // Filesystem check
        $varPath = BP . '/var';
        $health['checks']['filesystem'] = is_writable($varPath) ? 'ok' : 'error: var/ not writable';

        $httpCode = $health['status'] === 'healthy' ? 200 : 503;
        return $result->setHttpResponseCode($httpCode)->setData($health);
    }
}
```

**Route configuration (routes.xml):**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="standard">
        <route id="health" frontName="health">
            <module name="Vendor_Module"/>
        </route>
    </router>
</config>
```

**Usage:**

```bash
curl -f https://example.com/health/check || exit 1
```

### Load Balancer Health Checks

**HAProxy configuration:**

```haproxy
backend magento_servers
    balance roundrobin
    option httpchk GET /health/check
    http-check expect status 200

    server web1 10.0.1.10:80 check inter 5s fall 3 rise 2
    server web2 10.0.1.11:80 check inter 5s fall 3 rise 2
    server web3 10.0.1.12:80 check inter 5s fall 3 rise 2
```

**AWS ALB Target Group:**

```json
{
  "HealthCheckEnabled": true,
  "HealthCheckIntervalSeconds": 30,
  "HealthCheckPath": "/health/check",
  "HealthCheckProtocol": "HTTP",
  "HealthCheckTimeoutSeconds": 5,
  "HealthyThresholdCount": 2,
  "UnhealthyThresholdCount": 3,
  "Matcher": {
    "HttpCode": "200"
  }
}
```

---

## Rollback Procedures

### Instant Rollback (Symlink-Based)

```bash
#!/bin/bash
# rollback.sh

set -e

RELEASES_PATH="/var/www/magento/releases"
CURRENT_PATH="/var/www/magento/current"

# Get current release
CURRENT_RELEASE=$(readlink $CURRENT_PATH)

# Get previous release
PREVIOUS_RELEASE=$(ls -t $RELEASES_PATH | sed -n '2p')

if [ -z "$PREVIOUS_RELEASE" ]; then
    echo "No previous release found"
    exit 1
fi

echo "Rolling back from $(basename $CURRENT_RELEASE) to $PREVIOUS_RELEASE"

# Maintenance mode ON
php $CURRENT_PATH/bin/magento maintenance:enable

# Atomic symlink swap
ln -nfs $RELEASES_PATH/$PREVIOUS_RELEASE $CURRENT_PATH

# Clear cache
php $CURRENT_PATH/bin/magento cache:flush

# Maintenance mode OFF
php $CURRENT_PATH/bin/magento maintenance:disable

# Reload services
sudo systemctl reload php8.2-fpm
sudo nginx -s reload

echo "Rollback completed successfully"
```

### Database Rollback (Migrations)

**Important:** Magento doesn't support automatic DB rollback. Strategies:

**1. Backup before deployment:**

```bash
# Pre-deployment backup
mysqldump -u root -p magento_prod > /backups/magento_prod_$(date +%Y%m%d-%H%M%S).sql

# Restore if needed
mysql -u root -p magento_prod < /backups/magento_prod_20260205-120000.sql
```

**2. Schema version tracking:**

```bash
# Check current schema version
bin/magento module:status

# Manually revert to previous version (requires custom migration scripts)
bin/magento setup:rollback --code-version=2.4.6
```

**3. Blue-Green with separate databases (complex):**

```
Blue: Database A (current)
Green: Database B (new schema)

After validation → switch application to Database B
Rollback → switch back to Database A
```

### Automated Rollback on Failure

**GitHub Actions example:**

```yaml
- name: Deploy to production
  id: deploy
  run: ./deploy.sh

- name: Run smoke tests
  id: smoke-test
  run: ./smoke-tests.sh
  continue-on-error: true

- name: Rollback on failure
  if: steps.smoke-test.outcome == 'failure'
  run: |
    echo "Smoke tests failed, initiating rollback"
    ./rollback.sh
    exit 1
```

---

## GitLab CI Complete Example

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - package
  - deploy-staging
  - deploy-production

variables:
  PHP_VERSION: "8.2"
  ARTIFACT_NAME: "magento-${CI_COMMIT_SHA}-${CI_PIPELINE_ID}.tar.gz"

build:
  stage: build
  image: php:${PHP_VERSION}-cli
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - vendor/
      - node_modules/
  script:
    - apt-get update && apt-get install -y git unzip nodejs npm
    - composer install --no-dev --optimize-autoloader
    - npm ci && npm run build
    - bin/magento deploy:mode:set production --skip-compilation
    - bin/magento setup:di:compile
    - bin/magento setup:static-content:deploy -f en_US --jobs 4
  artifacts:
    paths:
      - ./
    expire_in: 1 week

test:unit:
  stage: test
  image: php:${PHP_VERSION}-cli
  dependencies:
    - build
  script:
    - composer install --dev
    - vendor/bin/phpunit -c dev/tests/unit/phpunit.xml.dist

test:static:
  stage: test
  image: php:${PHP_VERSION}-cli
  dependencies:
    - build
  script:
    - composer install --dev
    - vendor/bin/phpstan analyse app/code --level 8
    - vendor/bin/phpcs app/code --standard=Magento2

package:
  stage: package
  image: alpine:latest
  dependencies:
    - build
  script:
    - apk add tar gzip
    - tar -czf ${ARTIFACT_NAME} --exclude='.git' --exclude='dev' --exclude='var/log' .
    - sha256sum ${ARTIFACT_NAME} > ${ARTIFACT_NAME}.sha256
  artifacts:
    paths:
      - ${ARTIFACT_NAME}
      - ${ARTIFACT_NAME}.sha256
    expire_in: 30 days

deploy:staging:
  stage: deploy-staging
  image: alpine:latest
  dependencies:
    - package
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - staging
  script:
    - apk add openssh-client rsync
    - eval $(ssh-agent -s)
    - echo "$STAGING_SSH_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - ssh-keyscan $STAGING_HOST >> ~/.ssh/known_hosts
    - scp ${ARTIFACT_NAME} ${STAGING_USER}@${STAGING_HOST}:/tmp/
    - ssh ${STAGING_USER}@${STAGING_HOST} "./deploy.sh /tmp/${ARTIFACT_NAME}"

deploy:production:
  stage: deploy-production
  image: alpine:latest
  dependencies:
    - package
  environment:
    name: production
    url: https://www.example.com
  only:
    - production
  when: manual  # Require manual approval
  script:
    - apk add openssh-client
    - eval $(ssh-agent -s)
    - echo "$PRODUCTION_SSH_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - ssh-keyscan $PRODUCTION_HOST >> ~/.ssh/known_hosts
    - scp ${ARTIFACT_NAME} ${PRODUCTION_USER}@${PRODUCTION_HOST}:/tmp/
    - ssh ${PRODUCTION_USER}@${PRODUCTION_HOST} "./deploy.sh /tmp/${ARTIFACT_NAME}"
  after_script:
    - echo "Running smoke tests"
    - curl -f https://www.example.com/health/check || exit 1
```

---

## Performance Optimization

### Parallel Builds

```yaml
# Split static content deployment by locale
deploy-static-en:
  script: bin/magento setup:static-content:deploy en_US --jobs 4

deploy-static-de:
  script: bin/magento setup:static-content:deploy de_DE --jobs 4

# Run in parallel
```

### Incremental Builds

```bash
# Note: setup:di:compile-multi-tenant does not exist in Magento 2.4.x.
# Use the standard DI compile command (it compiles all modules):
bin/magento setup:di:compile

# Skip unchanged static content
bin/magento setup:static-content:deploy --theme Magento/backend  # Admin only
```

### Caching in CI

```yaml
- name: Cache Composer packages
  uses: actions/cache@v3
  with:
    path: ~/.composer/cache
    key: composer-${{ hashFiles('composer.lock') }}

- name: Cache static content
  uses: actions/cache@v3
  with:
    path: |
      pub/static
      var/view_preprocessed
    key: static-${{ hashFiles('app/design/**/*') }}
```

---

## Assumptions

- **Magento version:** 2.4.7+ (PHP 8.2+, Composer 2)
- **Infrastructure:** Linux servers (Ubuntu 22.04 or RHEL 9)
- **Web server:** Nginx 1.24+ with PHP-FPM 8.2
- **CI platform:** GitHub Actions or GitLab CI
- **Deployment:** Symlink-based releases with shared resources
- **Database:** MySQL 8.0 or MariaDB 10.6

## Why This Approach

**Build artifact strategy:**
- Eliminates "works on my machine" issues (identical artifacts across environments)
- Faster deployments (no composer install on production)
- Instant rollback (symlink swap)
- Reproducible builds (composer.lock committed)

**Blue-green deployment:**
- Zero downtime (traffic switches instantly)
- Easy rollback (switch back to previous environment)
- Full validation before traffic switch
- No database locking during deployment

**Rolling deployment:**
- Zero downtime for multi-server setups
- Gradual rollout reduces blast radius
- Load balancer handles traffic distribution
- Automatic health check integration

**Static content pre-compilation:**
- No on-demand generation overhead
- Consistent performance (no cache warm-up)
- CDN-ready assets
- Predictable deployment times

## Security Impact

- **SSH keys:** Use dedicated deploy keys with restricted permissions (read-only access to artifact storage, write to deployment directories only)
- **Secrets:** Store in CI platform secrets manager (GitHub Secrets, GitLab CI/CD variables); never commit to VCS
- **File permissions:** `www-data:www-data` ownership, 755 for directories, 644 for files, +x only for bin/magento
- **Artifact integrity:** SHA256 checksums verify artifact hasn't been tampered with
- **Database credentials:** Stored in `app/etc/env.php` (symlinked from shared directory, excluded from artifact)
- **Maintenance mode:** Prevents race conditions during schema upgrades; returns 503 status to search engines (preserves SEO)

## Performance Impact

- **Deployment speed:** Build artifact: 2-5 min; Build-on-server: 10-20 min
- **Downtime:** Blue-green: 0s; Rolling: 0s; Traditional: 2-5 min
- **Static content:** Pre-compiled: 3-5 min (one-time); On-demand: 100-500ms per request
- **Rollback time:** Symlink swap: <10s; Full rebuild: 10-20 min
- **Cache warm-up:** Pre-warmed cache (during deployment) vs cold cache (after rollback)
- **Database migrations:** `--keep-generated` flag avoids full DI recompilation

## Backward Compatibility

- **Deployment scripts:** Shell scripts compatible with Bash 4+, POSIX-compliant
- **Magento CLI:** Commands stable across 2.4.x; deprecations noted in release notes
- **Artifact structure:** Consistent across Magento versions; `composer.lock` ensures dependency compatibility
- **Database migrations:** Forward-only; rollback requires manual intervention or backups
- **Static content:** Theme structure changes may require updated deployment commands

## Tests to Add

**Pre-deployment:**
- Unit tests (PHPUnit)
- Integration tests
- Static analysis (PHPStan, PHPCS)
- Security scanning (composer audit)

**Post-deployment:**
- Smoke tests (HTTP 200 on critical URLs)
- Health check endpoint validation
- Functional tests (checkout flow, admin login)
- Performance regression tests (response time baselines)

## Documentation to Update

- **README:** Add deployment instructions, required secrets, server prerequisites
- **RUNBOOK:** Document rollback procedures, emergency contacts, escalation paths
- **CHANGELOG:** Track deployment history, schema changes, breaking changes
- **Infrastructure diagrams:** Server topology, load balancer configuration, network flow
- **Screenshots:** CI pipeline stages, artifact storage, deployment logs
- **Monitoring dashboards:** Grafana/Datadog links, key metrics, alert thresholds

---

## Additional Resources

- [Magento DevDocs: Deployment](https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/deployment/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [Blue-Green Deployment Pattern](https://martinfowler.com/bliki/BlueGreenDeployment.html)
- [Ansible for Magento Automation](https://github.com/elfhosted/ansible-magento2)

## Related Documentation

### Related Guides

- [Docker Development Environment Setup for Magento 2](../tutorials/docker-development-environment.md)
- [Comprehensive Testing Strategies for Magento 2](testing-strategies.md)
- [Full Page Cache Strategy for High-Performance Magento](full-page-cache-strategy.md)
