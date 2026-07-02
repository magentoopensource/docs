---
title: "Upgrade Guide: Adobe Commerce 2.4.6 to 2.4.7/2.4.8"
description: "Developer guide: Upgrade Guide: Adobe Commerce 2.4.6 to 2.4.7/2.4.8"
type: "reference"
tier: 1
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "25 minutes"
topics:
  - backward-compatibility
  - php-compatibility
  - database-migration
  - composer-dependencies
last_updated: "2026-02-07"
---

# Upgrade Guide: Adobe Commerce 2.4.6 to 2.4.7/2.4.8

This comprehensive guide covers the upgrade path from Adobe Commerce 2.4.6 to 2.4.7 and 2.4.8, including breaking changes, deprecated APIs, infrastructure requirements, and rollback strategies.

## Quick Reference Table

| Component | 2.4.6 | 2.4.7 | 2.4.8 | Action Required |
|-----------|-------|-------|-------|-----------------|
| PHP | 8.1, 8.2 | 8.2, 8.3 | 8.2, 8.3 | Update to 8.2+ before upgrade |
| MySQL | 8.0 | 8.0 | 8.0 | Verify MySQL 8.0.30+ |
| Elasticsearch | 7.17, 8.x | 7.17, 8.x, OpenSearch 1.2/2.x | OpenSearch 2.x (preferred) | Migrate to OpenSearch |
| Composer | 2.2+ | 2.2+ | 2.7+ | Update Composer to 2.7+ for 2.4.8 |
| Redis | 6.2, 7.0 | 7.0, 7.2 | 7.0, 7.2 | Update Redis to 7.0+ |
| Varnish | 7.1 | 7.3, 7.4 | 7.4 | Update Varnish |
| RabbitMQ | 3.9, 3.11 | 3.11, 3.13 | 3.13 | Update RabbitMQ |
| Node.js | 16, 18 | 18, 20 | 18, 20 | Update to Node 18+ |

## Version Compatibility Matrix

### Adobe Commerce 2.4.7

**Release Date:** April 2024

**Key Changes:**
- PHP 8.3 support added
- OpenSearch 2.x support
- Security enhancements (CVE patches)
- Performance improvements in indexing
- Enhanced GraphQL capabilities
- New Page Builder features

**Supported Technology Stack:**

```yaml
php:
  supported: [8.2, 8.3]
  recommended: 8.3
  deprecated: [8.1]

database:
  mysql: 8.0.30+
  mariadb: 10.6+ (not officially supported, use at own risk)

search:
  elasticsearch: [7.17, 8.x]
  opensearch: [1.2, 2.x]
  recommended: opensearch-2.x

cache:
  redis: [7.0, 7.2]

message_queue:
  rabbitmq: [3.11, 3.13]

web_server:
  nginx: 1.24+
  apache: 2.4+
```

### Adobe Commerce 2.4.8

**Release Date:** Expected Q2 2026 (Beta available)

**Key Changes:**
- Composer 2.7+ requirement
- Stricter type enforcement in core APIs
- Deprecated methods removed from 2.4.6
- Enhanced security defaults
- GraphQL schema improvements
- New Admin UI components

**Supported Technology Stack:**

```yaml
php:
  supported: [8.2, 8.3]
  recommended: 8.3
  minimum: 8.2

database:
  mysql: 8.0.35+

search:
  opensearch: [2.x]
  elasticsearch: deprecated (still works but not recommended)

composer:
  minimum: 2.7.0
```

## Breaking Changes in 2.4.7

### 1. PHP 8.1 Deprecation

**Impact:** HIGH

**Description:** PHP 8.1 is no longer supported. All custom code must be compatible with PHP 8.2/8.3.

**Action Required:**

```bash
# Before upgrade, verify PHP version
php -v

# Update PHP to 8.2 or 8.3
# On Ubuntu/Debian
sudo apt-get update
sudo apt-get install php8.3 php8.3-{bcmath,common,curl,fpm,gd,intl,mbstring,mysql,soap,xml,xsl,zip,cli}

# Update Composer PHP version requirement
composer config platform.php 8.3
```

**Code Changes Required:**

```php
// BREAKING: Dynamic properties deprecated in PHP 8.2
// OLD (will throw deprecation notices in 8.2, fatal error in 8.3)
class MyClass
{
    public function __construct()
    {
        $this->dynamicProperty = 'value'; // Deprecated
    }
}

// NEW: Declare all properties or use #[AllowDynamicProperties]
class MyClass
{
    private string $dynamicProperty; // Declare explicitly

    public function __construct()
    {
        $this->dynamicProperty = 'value';
    }
}

// OR use attribute (not recommended for new code)
#[\AllowDynamicProperties]
class MyClass
{
    public function __construct()
    {
        $this->dynamicProperty = 'value';
    }
}
```

### 2. Service Contract Interface Changes

**Impact:** MEDIUM

**Description:** Several service contract interfaces have new methods. Classes implementing these interfaces must add the new methods.

> **Note:** The core repository interfaces (`ProductRepositoryInterface`, `OrderRepositoryInterface`, `CustomerRepositoryInterface`) did not add new required methods in 2.4.7. However, when upgrading, always check for changes to interfaces your modules implement.

**Migration Path:**

```bash
# Find custom implementations of core interfaces
grep -r "implements.*RepositoryInterface" app/code/Vendor/Module/

# Review the Magento 2.4.7 release notes for any interface changes
# https://experienceleague.adobe.com/docs/commerce-operations/release/notes/open-source/2-4-7.html
```

### 3. Database Schema Changes

**Impact:** LOW-MEDIUM

**Description:** Minor schema changes may affect custom modules that query core tables directly. Always use the repository pattern or service contracts rather than direct table queries.

**Action Required:**

```php
// AVOID: Direct table queries that may break across versions
$connection->select()->from('catalog_product_entity', ['entity_id', 'sku']);

// PREFERRED: Use repository/service contracts
$product = $this->productRepository->get($sku);
```

### 4. Removed Deprecated Code

**Impact:** HIGH

**Description:** Code deprecated in 2.4.4-2.4.6 has been removed in 2.4.7.

**Removed Classes:**

```php
// REMOVED: Old payment methods
Magento\Authorizenet\* // Authorize.net Direct Post removed
Magento\Cybersource\* // Cybersource removed

// REMOVED: Legacy layout handles
// app/design/frontend/Vendor/theme/Magento_Checkout/layout/checkout_cart_configure.xml
// Use checkout_cart_item_configure.xml instead

// REMOVED: Deprecated JavaScript components
define(['Magento_Ui/js/form/element/abstract'], ...); // Still exists
// But several mixins removed - check console for warnings
```

**Migration for Payment Methods:**

```xml
<!-- OLD config.xml -->
<payment>
    <authorizenet_directpost>
        <active>1</active>
    </authorizenet_directpost>
</payment>

<!-- NEW: Migrate to Braintree or other payment provider -->
<payment>
    <braintree>
        <active>1</active>
        <model>BraintreeFacade</model>
    </braintree>
</payment>
```

### 5. Elasticsearch to OpenSearch Migration

**Impact:** HIGH (if using Elasticsearch)

**Description:** While Elasticsearch still works in 2.4.7, OpenSearch 2.x is the recommended search engine.

**Migration Steps:**

```bash
# 1. Install OpenSearch 2.x
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring
echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | sudo tee /etc/apt/sources.list.d/opensearch-2.list
sudo apt-get update
sudo apt-get install opensearch

# 2. Configure OpenSearch
sudo systemctl start opensearch
sudo systemctl enable opensearch

# 3. Update env.php
```

```php
// app/etc/env.php
'system' => [
    'default' => [
        'catalog' => [
            'search' => [
                'engine' => 'opensearch', // Changed from 'elasticsearch7'
                'opensearch_server_hostname' => 'localhost',
                'opensearch_server_port' => '9200',
                'opensearch_index_prefix' => 'magento2',
                'opensearch_enable_auth' => '0',
                'opensearch_server_timeout' => '15',
            ]
        ]
    ]
]
```

```bash
# 4. Reindex all indexes
bin/magento indexer:reindex catalogsearch_fulltext

# 5. Verify search works (test via storefront or API; no CLI search test command exists in core)
# curl -s https://your-store.test/catalogsearch/result/?q=search+term | head -20
```

## Breaking Changes in 2.4.8

### 1. Composer 2.7+ Required

**Impact:** HIGH

**Description:** Composer must be version 2.7.0 or higher due to new dependency resolution features.

**Action Required:**

```bash
# Check current version
composer --version

# Update Composer globally
sudo composer self-update

# Or download latest version
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
sudo mv composer.phar /usr/local/bin/composer
php -r "unlink('composer-setup.php');"

# Verify version
composer --version  # Should show 2.7.0+
```

### 2. Stricter Type Enforcement

**Impact:** MEDIUM

**Description:** Core APIs now enforce strict types. Passing wrong types will throw TypeErrors.

**Examples:**

```php
// BREAKING: Loose type coercion no longer works
$productRepository->get('123'); // Throws TypeError if expecting int

// OLD (worked in 2.4.6)
$product = $this->productRepository->getById('123'); // String accepted

// NEW (2.4.8 - strict types)
$product = $this->productRepository->getById((int)'123'); // Must cast to int

// Interface changes
interface ProductRepositoryInterface
{
    public function getById(int $id, bool $editMode = false, ?int $storeId = null); // int enforced
}
```

**Fix Custom Code:**

```bash
# Enable strict types in all PHP files
grep -l "<?php" app/code/Vendor/Module/**/*.php | while read file; do
    if ! grep -q "declare(strict_types=1);" "$file"; then
        sed -i '1s/^<?php/<?php\n\ndeclare(strict_types=1);/' "$file"
    fi
done
```

### 3. Removed Legacy Admin Widgets

**Impact:** MEDIUM

**Description:** Several deprecated Admin UI widgets removed. Must use UI components.

**Removed:**

```php
// REMOVED: Legacy grid widget
Magento\Backend\Block\Widget\Grid
Magento\Backend\Block\Widget\Grid\Column

// REMOVED: Legacy form widget
Magento\Backend\Block\Widget\Form

// Use UI components instead
```

**Migration Example:**

```xml
<!-- OLD: app/code/Vendor/Module/view/adminhtml/layout/vendor_module_index.xml -->
<block class="Vendor\Module\Block\Adminhtml\Grid" name="vendor.module.grid"/>

<!-- NEW: Use UI component -->
<uiComponent name="vendor_module_listing"/>
```

```xml
<!-- app/code/Vendor/Module/view/adminhtml/ui_component/vendor_module_listing.xml -->
<listing xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <dataSource name="vendor_module_listing_data_source">
        <argument name="dataProvider" xsi:type="configurableObject">
            <argument name="class" xsi:type="string">Vendor\Module\Ui\DataProvider\Listing</argument>
        </argument>
    </dataSource>
    <columns name="vendor_module_columns">
        <column name="id">
            <argument name="data" xsi:type="array">
                <item name="config" xsi:type="array">
                    <item name="label" xsi:type="string" translate="true">ID</item>
                </item>
            </argument>
        </column>
    </columns>
</listing>
```

## PHP 8.2/8.3 Compatibility Notes

### Key PHP 8.2 Changes

```php
// 1. Dynamic properties deprecated
#[\AllowDynamicProperties] // Use this attribute if needed
class MyClass {}

// 2. ${} string interpolation deprecated
$name = 'world';
echo "Hello ${name}"; // Deprecated
echo "Hello {$name}"; // Use this instead
echo "Hello $name";   // Or this

// 3. utf8_encode/utf8_decode deprecated
// OLD
$encoded = utf8_encode($string);

// NEW
$encoded = mb_convert_encoding($string, 'UTF-8', 'ISO-8859-1');
```

### Key PHP 8.3 Changes

```php
// 1. Typed class constants
class MyClass {
    public const string API_VERSION = 'v1'; // Type enforcement
}

// 2. #[\Override] attribute
class ChildClass extends ParentClass {
    #[\Override]
    public function method(): void {} // Ensures parent method exists
}

// 3. json_validate() function
if (json_validate($jsonString)) {
    $data = json_decode($jsonString);
}
```

### Common Compatibility Issues

```php
// Issue 1: Null coalescing in concatenation
// PHP 8.0+: Must use parentheses
$value = $array['key'] ?? 'default' . ' suffix'; // Deprecated syntax
$value = ($array['key'] ?? 'default') . ' suffix'; // Correct

// Issue 2: Return type declarations required
class MyRepository implements RepositoryInterface
{
    // OLD (works in 2.4.6)
    public function get($id) {}

    // NEW (required in 2.4.8)
    public function get(int $id): MyInterface {}
}

// Issue 3: Mixed return types
// Use 'mixed' for flexible returns
public function getValue(): mixed
{
    return $this->config->isEnabled() ? 'enabled' : null;
}
```

## MySQL 8.0 Migration Requirements

### Minimum Version

Adobe Commerce 2.4.7+ requires **MySQL 8.0.30 or higher**.

### Pre-Migration Checks

```bash
# Check current MySQL version
mysql --version

# Check for incompatible SQL modes
mysql -u root -p -e "SELECT @@GLOBAL.sql_mode;"

# Check for deprecated features
mysql -u root -p magento -e "
  SELECT TABLE_NAME, ENGINE
  FROM information_schema.TABLES
  WHERE TABLE_SCHEMA = 'magento'
  AND ENGINE = 'MyISAM';"
```

### Migration Steps

```bash
# 1. Backup database
mysqldump -u root -p magento > magento_backup_$(date +%Y%m%d).sql

# 2. Upgrade MySQL to 8.0.30+
# Ubuntu/Debian
wget https://dev.mysql.com/get/mysql-apt-config_0.8.24-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.24-1_all.deb
sudo apt-get update
sudo apt-get install mysql-server

# 3. Update MySQL configuration
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

```ini
# /etc/mysql/mysql.conf.d/mysqld.cnf
[mysqld]
# Required for Magento 2.4.7+
default-authentication-plugin=mysql_native_password
sql_mode="NO_ENGINE_SUBSTITUTION"
max_allowed_packet=256M
innodb_buffer_pool_size=2G
innodb_log_file_size=512M
```

```bash
# 4. Restart MySQL
sudo systemctl restart mysql

# 5. Run mysql_upgrade
sudo mysql_upgrade -u root -p

# 6. Verify Magento database
mysql -u magento_user -p magento -e "SHOW TABLES;"
```

### Common Issues

```sql
-- Issue 1: Reserved keywords (new in MySQL 8.0)
-- Table name 'rank' is now reserved
ALTER TABLE `rank` RENAME TO `product_rank`;

-- Issue 2: GROUP BY strictness
-- OLD (works in 5.7 with some sql_modes)
SELECT entity_id, MAX(created_at)
FROM sales_order
GROUP BY entity_id; -- Missing columns in GROUP BY

-- NEW (MySQL 8.0 strict mode)
SELECT entity_id, MAX(created_at) as max_created
FROM sales_order
GROUP BY entity_id;
```

## Composer Update Steps

### Step 1: Pre-Upgrade Preparation

```bash
# 1. Create a maintenance backup
bin/magento maintenance:enable
mysqldump -u root -p magento > pre_upgrade_$(date +%Y%m%d).sql
tar -czf magento_backup_$(date +%Y%m%d).tar.gz . --exclude='./var/cache' --exclude='./var/page_cache'

# 2. Clear caches and generated code
rm -rf var/cache/* var/page_cache/* var/view_preprocessed/* pub/static/* generated/*

# 3. Document current versions
bin/magento --version > pre_upgrade_version.txt
composer show magento/* > pre_upgrade_packages.txt
```

### Step 2: Update composer.json

```bash
# For 2.4.7 upgrade
composer require magento/product-community-edition=2.4.7 --no-update
composer require --dev phpunit/phpunit:^9 --no-update

# For 2.4.8 upgrade (when available)
composer require magento/product-community-edition=2.4.8 --no-update

# Update PHP version constraint
composer config platform.php 8.3
```

### Step 3: Update Dependencies

```bash
# Update all Magento packages
composer update magento/* --with-all-dependencies

# If you encounter conflicts, remove composer.lock and try again
rm composer.lock
composer update

# For specific conflicts, you may need to update other packages
composer update symfony/* --with-all-dependencies
composer update laminas/* --with-all-dependencies
```

### Step 4: Run Upgrade Commands

```bash
# 1. Run setup upgrade
bin/magento setup:upgrade

# 2. Compile DI
bin/magento setup:di:compile

# 3. Deploy static content
bin/magento setup:static-content:deploy -f

# 4. Reindex all
bin/magento indexer:reindex

# 5. Clear cache
bin/magento cache:clean
bin/magento cache:flush

# 6. Verify deployment
bin/magento deploy:mode:show
bin/magento module:status
```

### Step 5: Post-Upgrade Validation

```bash
# Note: upgrade:check is not a core Magento CLI command.
# Use the Upgrade Compatibility Tool (UCT) separately:
# bin/uct upgrade:check app/code/ --coming-version=2.4.8

# Check for deprecated code usage
vendor/bin/phpstan analyse app/code --level=8

# Test critical paths
bin/magento catalog:product:attributes:cleanup
bin/magento cron:run

# Verify version
bin/magento --version

# Disable maintenance mode
bin/magento maintenance:disable
```

## Common Upgrade Issues and Solutions

### Issue 1: Composer Memory Limit

**Symptom:**
```
Fatal error: Allowed memory size exhausted
```

**Solution:**
```bash
# Increase PHP memory limit for Composer
php -d memory_limit=4G $(which composer) update

# Or set it globally in php.ini
sudo nano /etc/php/8.3/cli/php.ini
# memory_limit = 4G
```

### Issue 2: Plugin Incompatibility

**Symptom:**
```
Area code is not set
Fatal error in plugin class
```

**Solution:**
```bash
# Disable third-party modules temporarily
bin/magento module:disable Vendor_Module

# Run upgrade
bin/magento setup:upgrade

# Re-enable modules one by one
bin/magento module:enable Vendor_Module

# Test after each enable
bin/magento cache:flush
```

**Prevention:**
```php
// In custom plugins, always check parent method exists
class ProductRepositoryExtend
{
    public function afterSave(
        \Magento\Catalog\Api\ProductRepositoryInterface $subject,
        $result
    ) {
        // Add defensive checks
        if (!method_exists($subject, 'save')) {
            return $result;
        }
        // Your logic
        return $result;
    }
}
```

### Issue 3: Database Schema Mismatch

**Symptom:**
```
Column not found: 1054 Unknown column 'some_column'
```

**Solution:**
```bash
# Check for pending schema updates
bin/magento setup:db:status

# If status is "Upgrade is required", run:
bin/magento setup:upgrade

# If specific table is missing columns, check db_schema.xml
cat app/code/Vendor/Module/etc/db_schema.xml

# Regenerate db_schema whitelist
bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_Module
```

### Issue 4: Static Content Deployment Fails

**Symptom:**
```
Error: Could not compile less file
Cannot find module '@magento/pagebuilder'
```

**Solution:**
```bash
# 1. Clear all static content
rm -rf pub/static/* var/view_preprocessed/*

# 2. Reinstall frontend dependencies
npm install --prefix vendor/magento/module-pagebuilder/view/adminhtml/web
npm install --prefix vendor/magento/theme-frontend-luma/web

# 3. Deploy with verbose output
bin/magento setup:static-content:deploy en_US -f --strategy=compact -vvv

# 4. If specific theme fails, deploy themes separately
bin/magento setup:static-content:deploy -f --theme=Magento/luma
```

### Issue 5: Cache Corruption

**Symptom:**
```
Frontend shows 404 errors
Admin panel not loading
```

**Solution:**
```bash
# Nuclear option - clear everything
rm -rf var/cache/* var/page_cache/* var/view_preprocessed/* pub/static/*
redis-cli FLUSHALL  # If using Redis
bin/magento cache:clean
bin/magento cache:flush

# Redeploy
bin/magento setup:di:compile
bin/magento setup:static-content:deploy -f

# Restart services
sudo systemctl restart php8.3-fpm
sudo systemctl restart nginx
sudo systemctl restart redis
```

## Post-Upgrade Validation Checklist

### Automated Tests

```bash
# 1. Run unit tests
vendor/bin/phpunit -c dev/tests/unit/phpunit.xml.dist

# 2. Run integration tests (sample)
vendor/bin/phpunit -c dev/tests/integration/phpunit.xml.dist app/code/Vendor/Module/Test/Integration

# 3. Static code analysis
vendor/bin/phpcs --standard=Magento2 app/code/Vendor/Module
vendor/bin/phpstan analyse app/code --level=8

# 4. Check for deprecated code
grep -r "@deprecated" vendor/magento/ | grep "2.4.6"
```

### Manual Validation

**Frontend Checklist:**
- [ ] Homepage loads without errors
- [ ] Category pages display products
- [ ] Product pages show correct data
- [ ] Search returns results
- [ ] Add to cart works
- [ ] Checkout flow completes
- [ ] Customer registration works
- [ ] Customer login/logout works
- [ ] Layered navigation filters work
- [ ] Product images display correctly

**Admin Checklist:**
- [ ] Admin login successful
- [ ] Dashboard loads
- [ ] Product grid displays
- [ ] Product save works
- [ ] Order grid displays
- [ ] Order view shows correct data
- [ ] Customer grid displays
- [ ] Configuration pages load
- [ ] Cache management works
- [ ] Index management works

**API Validation:**

```bash
# Test REST API
curl -X GET "https://your-site.com/rest/V1/products/24-MB01" \
  -H "Authorization: Bearer YOUR_TOKEN"

# Test GraphQL
curl -X POST "https://your-site.com/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ products(filter: {sku: {eq: \"24-MB01\"}}) { items { name sku } } }"}'
```

### Performance Validation

```bash
# Check index status
bin/magento indexer:status

# Verify cron is running
bin/magento cron:run
tail -f var/log/cron.log

# Check cache hit rates (Redis)
redis-cli INFO stats | grep hit

# Monitor slow queries
mysql -u root -p -e "SELECT * FROM mysql.slow_log ORDER BY query_time DESC LIMIT 10;"
```

## Rollback Strategies

### Pre-Rollback Preparation

Always prepare rollback plan BEFORE upgrading:

```bash
# 1. Full backup script (save as backup.sh)
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/magento"

# Database
mysqldump -u root -p magento > $BACKUP_DIR/db_$DATE.sql

# Files
tar -czf $BACKUP_DIR/files_$DATE.tar.gz \
  --exclude='./var/cache' \
  --exclude='./var/page_cache' \
  --exclude='./pub/static' \
  -C /var/www/magento .

# composer.lock and env.php
cp composer.lock $BACKUP_DIR/composer.lock.$DATE
cp app/etc/env.php $BACKUP_DIR/env.php.$DATE

echo "Backup completed: $DATE"
```

### Rollback Method 1: Git Revert (Recommended)

```bash
# If upgrade is in Git branch
git log --oneline -10  # Find commit before upgrade
git revert <upgrade-commit-hash>

# Restore database
mysql -u root -p magento < backup_db.sql

# Restore composer dependencies
git checkout HEAD~1 composer.lock
composer install

# Clear caches and regenerate
rm -rf generated/* var/cache/* var/page_cache/*
bin/magento setup:upgrade
bin/magento setup:di:compile
bin/magento cache:flush
```

### Rollback Method 2: Manual Restore

```bash
# 1. Enable maintenance mode
bin/magento maintenance:enable

# 2. Restore files
cd /var/www
rm -rf magento
tar -xzf /backups/magento/files_YYYYMMDD.tar.gz -C magento/

# 3. Restore database
mysql -u root -p magento < /backups/magento/db_YYYYMMDD.sql

# 4. Restore environment config
cp /backups/magento/env.php.YYYYMMDD magento/app/etc/env.php

# 5. Downgrade Composer packages
cd magento
composer install

# 6. Clear caches
rm -rf var/cache/* var/page_cache/* generated/*
bin/magento cache:clean

# 7. Disable maintenance
bin/magento maintenance:disable
```

### Rollback Method 3: Blue-Green Deployment

**Setup (before upgrade):**

```bash
# Keep two parallel installations
/var/www/magento-blue/   # Current production (2.4.6)
/var/www/magento-green/  # Upgrade target (2.4.7)

# Nginx config - switch upstream
upstream magento {
    server unix:/var/run/php/magento-blue.sock;  # Current
    # server unix:/var/run/php/magento-green.sock;  # Swap to this after upgrade
}
```

**Rollback:**

```nginx
# Simply switch back in nginx config
upstream magento {
    # server unix:/var/run/php/magento-green.sock;  # Failed upgrade
    server unix:/var/run/php/magento-blue.sock;  # Revert to old version
}
```

```bash
# Reload nginx
sudo nginx -t
sudo systemctl reload nginx

# Database: restore from backup if schema changed
mysql -u root -p magento < pre_upgrade_backup.sql
```

## Upgrade Troubleshooting Reference

### Debug Commands

```bash
# Enable developer mode for detailed errors
bin/magento deploy:mode:set developer

# Check logs in real-time
tail -f var/log/system.log
tail -f var/log/exception.log
tail -f var/log/debug.log

# Enable database query logging
mysql -u root -p -e "SET GLOBAL general_log = 'ON';"
tail -f /var/log/mysql/query.log

# PHP error log
tail -f /var/log/php8.3-fpm.log

# Web server logs
tail -f /var/log/nginx/error.log
```

### Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| `Area code is not set` | Plugin execution before area initialization | Wrap plugin code in try-catch, check area |
| `Service class does not exist` | Service contract not compiled | Run `setup:di:compile` |
| `Class not found` | Autoloader not updated | `composer dump-autoload` |
| `Column not found` | Database schema not upgraded | `setup:upgrade`, check `db_schema.xml` |
| `Memory size exhausted` | Insufficient PHP memory | Increase `memory_limit` in php.ini |
| `Segmentation fault` | PHP extension conflict | Check PHP modules, update extensions |

## Assumptions

**Magento/PHP versions:** This guide covers Adobe Commerce 2.4.6 → 2.4.7 → 2.4.8 upgrades. Targets PHP 8.2 and 8.3. For earlier versions (2.4.4, 2.4.5), additional intermediate upgrade steps may be required.

**Modules:** Assumes production environment with typical third-party modules. Custom/marketplace extensions must be verified for compatibility before upgrade.

**Areas:** Applies to all areas (frontend, adminhtml, API). Breaking changes affect custom modules in all scopes.

**Scopes:** Configuration changes affect global, website, and store scopes. Review scope-specific configs after upgrade.

**Environment:** Production server with SSH access, MySQL/MariaDB, Redis, Elasticsearch/OpenSearch, and standard LEMP/LAMP stack. Cloud environments (Adobe Commerce Cloud) have different upgrade procedures via `ece-tools`.

## Why This Approach

**Comprehensive coverage:** This guide includes infrastructure (PHP, MySQL, search engine), code-level changes (APIs, types, deprecated methods), and operational aspects (rollback, validation). Developers need all layers to successfully upgrade.

**Version-specific breakdown:** Separating 2.4.7 and 2.4.8 changes prevents confusion. Each version has distinct breaking changes that must be addressed differently.

**Real code examples:** Every breaking change includes OLD vs NEW code samples, making it easy to identify and fix incompatible patterns in custom code.

**Rollback strategies:** Production upgrades must have rollback plans. Three methods (Git revert, manual restore, blue-green) cover different deployment scenarios.

**Alternatives considered:**
- Official Magento upgrade docs: High-level, lacks code examples
- Composer-only guide: Misses infrastructure and code compatibility
- Code-only guide: Ignores database, search engine, and service requirements

## Security Impact

**authZ/authN:** No changes to core authentication mechanisms in 2.4.7/2.4.8, but custom implementations of `CustomerRepositoryInterface` must implement new methods correctly to avoid auth bypass.

**CSRF/form key:** Enhanced validation in 2.4.7. Custom forms must include form keys:

```php
// In custom forms
$formKey = $this->formKey->getFormKey();
```

**XSS escaping:** Stricter output escaping in Admin UI components. Use `escapeHtml()` in all custom templates:

```php
echo $block->escapeHtml($product->getName());
```

**Secrets:** After upgrade, rotate API tokens and integration keys. New installations use stronger hashing algorithms (bcrypt cost factor increased).

**PII:** Database schema changes add columns that may contain PII. Update GDPR export/delete logic if custom tables join with `customer_entity`.

## Performance Impact

**Cacheability (FPC/Varnish):** OpenSearch 2.x improves search performance by 15-20% vs Elasticsearch 7.17. FPC tags unchanged, but Varnish VCL may need updates for 2.4.7 (new health check endpoint).

**Redis tags:** No changes to cache tag structure. Flush all caches post-upgrade to prevent stale tag issues.

**DB load:** First-time reindexing after upgrade may take 2-4x longer than normal due to schema changes and data migration patches.

**CWV (Core Web Vitals):** Enhanced JS bundling in 2.4.7 reduces JS bundle size by ~10%. LCP and FID improvements of 5-10% in default Luma theme. Custom themes must be re-built to benefit.

**Impact during upgrade:**
- `setup:upgrade`: 5-15 minutes (depends on DB size)
- `setup:di:compile`: 3-5 minutes
- `setup:static-content:deploy`: 10-30 minutes (depends on locales/themes)
- Reindexing: 30 minutes - 2 hours (depends on catalog size)

**Mitigation:** Schedule upgrades during low-traffic windows. Use maintenance mode or blue-green deployment for zero-downtime.

## Backward Compatibility

**API/DB schema changes:**
- **Service Contracts:** New methods in repositories (BC break for implementations). Interfaces are extended, not modified, to preserve BC for method consumers.
- **Database schema:** New columns added with `DEFAULT NULL`, existing columns unchanged. Custom modules querying tables directly must update `SELECT` lists to handle new columns gracefully.
- **GraphQL schema:** New fields added to existing types (BC-safe). No field removals in 2.4.7. In 2.4.8, deprecated fields from 2.4.5 will be removed.

**Upgrade/migration notes:**
- **2.4.6 → 2.4.7:** Direct upgrade supported. No intermediate steps required.
- **2.4.5 or earlier → 2.4.7:** Must upgrade to 2.4.6 first, then to 2.4.7.
- **2.4.7 → 2.4.8:** Direct upgrade. Code deprecated in 2.4.6 will be removed; fix deprecation warnings before upgrading to 2.4.8.

**Extension compatibility:** Third-party modules must declare compatibility via `composer.json`:

```json
{
  "require": {
    "magento/framework": "~103.0.6"
  }
}
```

Check marketplace extensions for 2.4.7/2.4.8 compatibility before upgrading. Use `composer require --dry-run` to detect conflicts before committing.

## Tests to Add

**Unit tests:**

```php
// Test new repository methods
// Test that core repository methods still work after upgrade
public function testProductRepositoryGetByIdWorks()
{
    $product = $this->productRepository->getById(1);
    $this->assertInstanceOf(\Magento\Catalog\Api\Data\ProductInterface::class, $product);
}

// Test search criteria still works
public function testProductRepositoryGetListWorks()
{
    $searchCriteria = $this->searchCriteriaBuilder->create();
    $results = $this->productRepository->getList($searchCriteria);
    $this->assertGreaterThan(0, $results->getTotalCount());
}
```

**MFTF (Functional):**

```xml
<!-- Test checkout flow post-upgrade -->
<test name="StorefrontCheckoutWithNewPaymentMethodTest">
    <actionGroup ref="AddSimpleProductToCartActionGroup" stepKey="addProduct"/>
    <actionGroup ref="GoToCheckoutFromMinicartActionGroup" stepKey="goToCheckout"/>
    <actionGroup ref="CheckoutSelectFlatRateShippingMethodActionGroup" stepKey="selectShipping"/>
    <actionGroup ref="StorefrontCheckoutClickNextButtonActionGroup" stepKey="clickNext"/>
    <actionGroup ref="CheckoutSelectCheckMoneyOrderPaymentActionGroup" stepKey="selectPayment"/>
    <actionGroup ref="CheckoutPlaceOrderActionGroup" stepKey="placeOrder"/>
</test>
```

**Mutation testing:** For critical payment/checkout logic, use Infection PHP to verify test coverage:

```bash
composer require --dev infection/infection
vendor/bin/infection --test-framework=phpunit --threads=4
```

## Docs to Update

**README:** Update version compatibility table, add upgrade section linking to this guide.

**CHANGELOG:** Document all breaking changes, new features, deprecated code per version:

```markdown
## [2.4.7] - 2024-04-XX
### Breaking Changes
- PHP 8.1 support dropped
- Various interface signature refinements (check custom implementations)

### Deprecated
- Elasticsearch 7.x (use OpenSearch 2.x)

### Added
- OpenSearch 2.x support
- PHP 8.3 compatibility
```

**Admin user guide paths:**
1. Admin Panel > Stores > Configuration > Catalog > Catalog > Catalog Search
   - Screenshot: New "Search Engine: OpenSearch" dropdown
2. Admin Panel > System > Index Management
   - Screenshot: Index Management page
3. Admin Panel > System > Tools > Upgrade
   - Screenshot: Post-upgrade validation report

**Screenshots to capture:**
- OpenSearch configuration in Admin (Stores > Configuration > Catalog > Catalog Search)
- Index Management showing new indexes (System > Index Management)
- Performance monitoring dashboard showing improved metrics post-upgrade
- Composer version check (`composer --version` showing 2.7+)

---

**Related Resources:**
- [Official Adobe Commerce Release Notes](https://experienceleague.adobe.com/docs/commerce-operations/release/notes/overview.html)
- [PHP 8.2 Migration Guide](https://www.php.net/manual/en/migration82.php)
- [PHP 8.3 Migration Guide](https://www.php.net/manual/en/migration83.php)
- [OpenSearch Migration Guide](https://opensearch.org/docs/latest/migration/)
- [Magento Technical Guidelines - Backward Compatibility](https://developer.adobe.com/commerce/php/development/backward-compatibility-policy/)

## Related Documentation

### Related Guides

- [Declarative Schema & Data Patches: Modern Database Management in Magento 2](../tutorials/declarative-schema-data-patches.md)
- [CLI Command Reference: Complete bin/magento Guide](cli-command-reference.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Customer Overview](../../modules/customer/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
- [Magento_Checkout Overview](../../modules/checkout/README.md)
- [Magento_Quote Overview](../../modules/quote/README.md)
