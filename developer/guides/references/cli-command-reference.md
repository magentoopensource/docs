---
title: "CLI Command Reference: Complete bin/magento Guide"
description: "Developer guide: CLI Command Reference: Complete bin/magento Guide"
type: "reference"
tier: 1
difficulty: "intermediate"
version: "2.4.7+"
estimated_time: "30 minutes"
topics:
  - cli-development
  - custom-commands
  - maintenance
  - deployment
last_updated: "2026-02-07"
---

# CLI Command Reference: Complete bin/magento Guide

Complete reference for all `bin/magento` CLI commands with syntax, options, examples, and best practices for Adobe Commerce 2.4.7+.

## Quick Reference Table

| Category | Key Commands | Use Case |
|----------|-------------|----------|
| **Cache** | `cache:clean`, `cache:flush`, `cache:enable` | Clear/manage cache types |
| **Indexer** | `indexer:reindex`, `indexer:status`, `indexer:set-mode` | Rebuild/configure indexes |
| **Setup** | `setup:upgrade`, `setup:di:compile`, `setup:static-content:deploy` | Deploy/upgrade code |
| **Module** | `module:enable`, `module:disable`, `module:status` | Control module state |
| **Config** | `config:set`, `config:show`, `app:config:dump` | Manage configuration |
| **Cron** | `cron:run`, `cron:install`, `cron:remove` | Execute scheduled tasks |
| **Deploy** | `deploy:mode:set`, `deploy:mode:show` | Switch deployment modes |
| **Maintenance** | `maintenance:enable`, `maintenance:disable` | Toggle maintenance mode |
| **Customer** | `customer:hash:upgrade` | Customer operations |
| **Dev** | `dev:tests:run`, `dev:query-log:enable` | Development tools |

## Command Categories

### Cache Commands

Cache management is critical for performance and seeing configuration changes take effect.

#### cache:clean

**Description:** Cleans cache types by removing all items from enabled cache types without flushing storage.

**Syntax:**
```bash
bin/magento cache:clean [types]
```

**Options:**
- `types` - Space-separated list of cache types. Omit to clean all.

**Available Cache Types:**

| Type | Code | Purpose |
|------|------|---------|
| Configuration | `config` | System configuration, modules, etc. |
| Layout | `layout` | Layout building instructions |
| Block HTML | `block_html` | Page block output |
| Collections | `collections` | Collection data |
| Reflection | `reflection` | API and webapi reflection |
| Database DDL | `db_ddl` | Database schema |
| Compiled Config | `compiled_config` | Compiled configuration |
| EAV | `eav` | Entity attribute values metadata |
| Customer Notification | `customer_notification` | Notifications in customer account |
| Config Integration | `config_integration` | Integration configuration |
| Config Integration API | `config_integration_api` | Integration API configuration |
| Full Page | `full_page` | Full page cache (FPC) |
| Config Webservice | `config_webservice` | Web services configuration |
| Translations | `translate` | Translation files |

**Examples:**

```bash
# Clean all cache types
bin/magento cache:clean

# Clean specific cache types
bin/magento cache:clean config layout

# Clean only full page cache
bin/magento cache:clean full_page

# Common combination after config changes
bin/magento cache:clean config compiled_config
```

**Expected Output:**
```
Cleaned cache types:
config
layout
```

**When to Use:**
- After changing system configuration
- After modifying layout XML files
- Before deploying to production (to ensure fresh cache)
- When troubleshooting frontend display issues

**Common Mistakes:**

```bash
# WRONG: Using cache type names instead of codes
bin/magento cache:clean "Block HTML"

# CORRECT: Use cache type codes
bin/magento cache:clean block_html

# WRONG: Forgetting to clean compiled_config after di.xml changes
bin/magento cache:clean config

# CORRECT: Clean both config and compiled_config
bin/magento cache:clean config compiled_config
```

#### cache:flush

**Description:** Flushes cache storage completely, removing all items from all cache types.

**Syntax:**
```bash
bin/magento cache:flush [types]
```

**Difference from cache:clean:**
- `cache:clean` - Removes items from Magento cache types only
- `cache:flush` - Empties the entire cache storage (Redis/filesystem)

**Examples:**

```bash
# Flush all cache storage
bin/magento cache:flush

# Flush specific types (same as cache:clean for most purposes)
bin/magento cache:flush config

# Nuclear option - flush everything including non-Magento cache
bin/magento cache:flush && redis-cli FLUSHALL
```

**When to Use:**
- When cache:clean doesn't resolve issues
- After major code changes (module install/remove)
- When cache becomes corrupted
- In development environments

**Performance Impact:** HIGH - Causes cache rebuild, slowing site until cache warms up. Use `cache:clean` in production.

#### cache:enable / cache:disable

**Description:** Enable or disable specific cache types.

**Syntax:**
```bash
bin/magento cache:enable [types]
bin/magento cache:disable [types]
```

**Examples:**

```bash
# Enable all caches
bin/magento cache:enable

# Enable specific caches
bin/magento cache:enable config layout

# Disable full page cache for development
bin/magento cache:disable full_page block_html

# Common dev setup: disable all except config/layout
bin/magento cache:disable
bin/magento cache:enable config layout
```

**Development vs Production:**

```bash
# Development: Disable most caches for faster iteration
bin/magento cache:disable full_page block_html collections

# Production: Enable all caches for performance
bin/magento cache:enable
```

#### cache:status

**Description:** Check status of all cache types.

**Syntax:**
```bash
bin/magento cache:status
```

**Example Output:**
```
Current status:
                        config: 1
                        layout: 1
                    block_html: 1
                   collections: 1
                    reflection: 1
                        db_ddl: 1
               compiled_config: 1
                           eav: 1
         customer_notification: 1
            config_integration: 1
        config_integration_api: 1
                     full_page: 1
             config_webservice: 1
                     translate: 1
```

**Legend:** `1` = enabled, `0` = disabled

### Indexer Commands

Indexes optimize database queries. Keeping indexes updated is critical for performance.

#### indexer:reindex

**Description:** Rebuild one or more indexes.

**Syntax:**
```bash
bin/magento indexer:reindex [indexers]
```

**Available Indexers:**

| Indexer | Code | Purpose |
|---------|------|---------|
| Design Config Grid | `design_config_grid` | Admin design configuration grid |
| Customer Grid | `customer_grid` | Admin customer grid |
| Category Products | `catalog_category_product` | Product assignments to categories |
| Product Categories | `catalog_product_category` | Category assignments to products |
| Catalog Product Rule | `catalogrule_rule` | Catalog price rules |
| Product Price | `catalog_product_price` | Product prices (with tier prices, rules) |
| Product EAV | `catalog_product_attribute` | Product attributes used in layered nav |
| Stock | `cataloginventory_stock` | Stock status |
| Catalog Search | `catalogsearch_fulltext` | Full-text search index |
| Sales Rule | `salesrule_rule` | Shopping cart price rules |

**Examples:**

```bash
# Reindex all
bin/magento indexer:reindex

# Reindex specific indexer
bin/magento indexer:reindex catalog_product_price

# Reindex multiple indexers
bin/magento indexer:reindex catalog_category_product catalog_product_category catalogsearch_fulltext

# Common combination after catalog changes
bin/magento indexer:reindex catalog_category_product catalog_product_price catalogsearch_fulltext
```

**Expected Output:**
```
Design Config Grid index has been rebuilt successfully in 00:00:01
Customer Grid index has been rebuilt successfully in 00:00:03
Category Products index has been rebuilt successfully in 00:00:45
Product Categories index has been rebuilt successfully in 00:00:30
...
```

**When to Use:**
- After importing products
- After mass attribute updates
- After changing catalog price rules
- When product data doesn't appear on frontend
- After manual database changes

**Performance Considerations:**

```bash
# Large catalogs: Reindex off-peak hours
# 10k products: ~2-5 minutes
# 100k products: ~15-30 minutes
# 1M+ products: ~1-3 hours

# Monitor progress in separate terminal
watch -n 5 'bin/magento indexer:status'

# Run in background and log output
nohup bin/magento indexer:reindex > reindex.log 2>&1 &
```

#### indexer:status

**Description:** Show status of all indexers.

**Syntax:**
```bash
bin/magento indexer:status [indexers]
```

**Example Output:**
```
Design Config Grid:                                Ready
Customer Grid:                                     Ready
Category Products:                                 Reindex required
Product Categories:                                Ready
Catalog Product Rule:                              Ready
Product Price:                                     Processing
Product EAV:                                       Ready
Stock:                                             Ready
Catalog Search:                                    Ready
```

**Status Values:**
- `Ready` - Index is up to date
- `Reindex required` - Data changed, index needs rebuild
- `Processing` - Reindex in progress
- `Working` - Update on Schedule mode, processing backlog

#### indexer:set-mode

**Description:** Set indexer mode to "realtime" or "schedule".

**Syntax:**
```bash
bin/magento indexer:set-mode {realtime|schedule} [indexers]
```

**Modes:**

| Mode | Behavior | Use Case |
|------|----------|----------|
| `realtime` | Index updates immediately on data change | Small catalogs, dev environments |
| `schedule` | Index updates via cron (default every 1 min) | Large catalogs, production |

**Examples:**

```bash
# Set all indexers to schedule mode (recommended for production)
bin/magento indexer:set-mode schedule

# Set specific indexers to realtime
bin/magento indexer:set-mode realtime catalog_product_price

# Production setup: Schedule mode + cron
bin/magento indexer:set-mode schedule
bin/magento cron:install
```

**Verification:**

```bash
bin/magento indexer:show-mode
```

**Expected Output:**
```
Design Config Grid:                                Update by Schedule
Customer Grid:                                     Update by Schedule
Category Products:                                 Update on Save
...
```

**Best Practice:**

```bash
# Production: Always use schedule mode
bin/magento indexer:set-mode schedule

# Development: Use realtime for faster feedback
bin/magento indexer:set-mode realtime

# Exception: catalogsearch_fulltext always schedule in production
bin/magento indexer:set-mode schedule catalogsearch_fulltext
```

#### indexer:reset

**Description:** Reset indexer status to invalid (forces reindex).

**Syntax:**
```bash
bin/magento indexer:reset [indexers]
```

**Examples:**

```bash
# Reset all indexers
bin/magento indexer:reset

# Reset specific indexer
bin/magento indexer:reset catalog_product_price

# Reset and reindex in one command
bin/magento indexer:reset && bin/magento indexer:reindex
```

**When to Use:**
- Indexer stuck in "Processing" state
- Suspected index corruption
- After restoring database backup

### Setup Commands

Setup commands handle deployment, upgrades, and compilation.

#### setup:upgrade

**Description:** Upgrade database schema and data (run after module changes).

**Syntax:**
```bash
bin/magento setup:upgrade [options]
```

**Options:**
- `--keep-generated` - Keep generated code (faster, but risky)
- `--safe-mode=1` - Safe installer with data backup during update
- `--data-restore=1` - Restore removed data
- `--dry-run=1` - Show SQL statements without executing

**Examples:**

```bash
# Standard upgrade
bin/magento setup:upgrade

# Dry run to preview changes
bin/magento setup:upgrade --dry-run=1

# Safe mode with backup
bin/magento setup:upgrade --safe-mode=1

# Fast upgrade keeping generated code (not recommended)
bin/magento setup:upgrade --keep-generated
```

**Expected Output:**
```
Cache cleared successfully
File system cleanup:
/var/www/html/var/cache
/var/www/html/var/page_cache
/var/www/html/generated/code
...
Updating modules:
Schema creation/updates:
Module 'Magento_Store':
Module 'Magento_AdvancedPricingImportExport':
...
Data install/update:
Module 'Magento_Store':
Module 'Magento_User':
...
```

**When to Use:**
- After installing new module
- After enabling/disabling module
- After updating module code
- After restoring database backup
- After running `composer update`

**Common Workflow:**

```bash
# Full deployment workflow
composer install                          # Install dependencies
bin/magento module:enable Vendor_Module   # Enable new module
bin/magento setup:upgrade                 # Upgrade database
bin/magento setup:di:compile              # Compile DI
bin/magento setup:static-content:deploy -f # Deploy static content
bin/magento cache:flush                   # Clear cache
```

#### setup:di:compile

**Description:** Generate dependency injection configuration and code.

**Syntax:**
```bash
bin/magento setup:di:compile
```

**What It Does:**
- Generates factories, proxies, and interceptors
- Creates `generated/metadata/` directory with DI metadata
- Validates DI configuration in all modules

**Examples:**

```bash
# Standard compilation
bin/magento setup:di:compile

# Compilation for multi-tenant setup
bin/magento setup:di:compile --ansi

# Parallel terminal: Monitor memory usage
watch -n 1 'ps aux | grep php | grep -v grep'
```

**Expected Output:**
```
Compilation was started.
Interception cache generation... 7/7 [============================] 100% 1 min
Generated code and dependency injection configuration successfully.
```

**When to Use:**
- After modifying `di.xml`
- After creating new classes with DI
- After running `setup:upgrade`
- Before deploying to production
- When getting "Class not found" errors in generated code

**Common Mistakes:**

```bash
# WRONG: Compiling before setup:upgrade
bin/magento setup:di:compile
bin/magento setup:upgrade

# CORRECT: Upgrade first, then compile
bin/magento setup:upgrade
bin/magento setup:di:compile

# WRONG: Compiling in developer mode (not needed)
bin/magento deploy:mode:set developer
bin/magento setup:di:compile  # Unnecessary

# CORRECT: Only compile in production/default mode
bin/magento deploy:mode:set production
bin/magento setup:di:compile
```

**Performance:**
- Small projects: 1-3 minutes
- Medium projects: 3-8 minutes
- Large projects: 8-20 minutes
- Memory required: 2-4 GB RAM

#### setup:static-content:deploy

**Description:** Deploy static view files (CSS, JS, images).

**Syntax:**
```bash
bin/magento setup:static-content:deploy [options] [languages]
```

**Options:**
- `-f, --force` - Force deployment (clears existing files)
- `--no-javascript` - Skip JavaScript bundling
- `--no-css` - Skip CSS deployment
- `--no-less` - Skip LESS compilation
- `--no-images` - Skip image optimization
- `--no-fonts` - Skip font copying
- `--theme=<theme>` - Deploy specific theme
- `--exclude-theme=<theme>` - Exclude theme
- `--area=<area>` - Deploy specific area (adminhtml, frontend)
- `--strategy=<strategy>` - Deployment strategy (standard, quick, compact)
- `-j, --jobs=<number>` - Number of parallel jobs (default: 4)

**Deployment Strategies:**

| Strategy | Description | Use Case |
|----------|-------------|----------|
| `standard` | Copy files for each theme/locale | Production (default) |
| `quick` | Minimize deployment time | Development |
| `compact` | Symlink parent theme files | Small servers |

**Examples:**

```bash
# Deploy all themes and locales
bin/magento setup:static-content:deploy

# Deploy specific locales
bin/magento setup:static-content:deploy en_US de_DE fr_FR

# Force deploy with parallel jobs
bin/magento setup:static-content:deploy -f -j 8 en_US

# Deploy frontend only
bin/magento setup:static-content:deploy --area=frontend

# Deploy specific theme
bin/magento setup:static-content:deploy --theme=Magento/luma

# Quick deployment for development
bin/magento setup:static-content:deploy --strategy=quick -f

# Production deployment with all optimizations
bin/magento setup:static-content:deploy -f --strategy=standard -j 8 en_US

# Skip JavaScript bundling (faster)
bin/magento setup:static-content:deploy --no-javascript
```

**Expected Output:**
```
Requested languages: en_US
=== frontend -> Magento/luma -> en_US ===
Successful: 2389 files; errors: 0
---
=== adminhtml -> Magento/backend -> en_US ===
Successful: 3214 files; errors: 0
---
Execution time: 342.123456789
```

**When to Use:**
- After modifying LESS/CSS files
- After modifying JavaScript files
- After adding/removing themes
- After changing static content configuration
- Before deploying to production

**Performance Optimization:**

```bash
# Use all CPU cores
CORES=$(nproc)
bin/magento setup:static-content:deploy -j $CORES

# Deploy only changed files (in CI/CD)
bin/magento setup:static-content:deploy --no-parent --strategy=compact

# Exclude unnecessary themes
bin/magento setup:static-content:deploy \
  --exclude-theme=Magento/blank \
  --theme=Magento/luma \
  en_US
```

**Common Issues:**

```bash
# Issue: Out of memory during deployment
# Solution: Increase PHP memory limit
php -d memory_limit=4G bin/magento setup:static-content:deploy

# Issue: Slow deployment
# Solution: Use more jobs and quick strategy
bin/magento setup:static-content:deploy -f --strategy=quick -j 8

# Issue: Missing files after deployment
# Solution: Clear pub/static and var/view_preprocessed first
rm -rf pub/static/* var/view_preprocessed/*
bin/magento setup:static-content:deploy -f
```

### Module Commands

Control module state (enabled/disabled).

#### module:enable

**Description:** Enable one or more modules.

**Syntax:**
```bash
bin/magento module:enable [options] [modules]
```

**Options:**
- `-c, --clear-static-content` - Clear generated static view files
- `-f, --force` - Bypass dependency checks (dangerous)
- `--all` - Enable all modules

**Examples:**

```bash
# Enable single module
bin/magento module:enable Vendor_Module

# Enable multiple modules
bin/magento module:enable Vendor_ModuleOne Vendor_ModuleTwo

# Enable all disabled modules
bin/magento module:enable --all

# Enable with static content cleanup
bin/magento module:enable Vendor_Module --clear-static-content

# Full workflow after enabling module
bin/magento module:enable Vendor_Module
bin/magento setup:upgrade
bin/magento setup:di:compile
bin/magento cache:flush
```

**Expected Output:**
```
The following modules have been enabled:
- Vendor_Module

To make sure that the enabled modules are properly registered, run 'setup:upgrade'.
Cache cleared successfully.
Generated classes cleared successfully.
```

#### module:disable

**Description:** Disable one or more modules.

**Syntax:**
```bash
bin/magento module:disable [options] [modules]
```

**Options:**
- `-c, --clear-static-content` - Clear generated static view files
- `-f, --force` - Bypass dependency checks
- `--all` - Disable all modules (dangerous, for troubleshooting)

**Examples:**

```bash
# Disable single module
bin/magento module:disable Vendor_Module

# Disable multiple modules
bin/magento module:disable Vendor_ModuleOne Vendor_ModuleTwo

# Disable with force (skip dependency check)
bin/magento module:disable -f Vendor_Module

# Troubleshooting: Disable all third-party modules
bin/magento module:status | grep -v "Magento_" | grep "enabled" | awk '{print $1}' | xargs bin/magento module:disable
```

**Warning:** Disabling modules that other modules depend on will break functionality. Use `--force` carefully.

#### module:status

**Description:** Display status of all modules.

**Syntax:**
```bash
bin/magento module:status
```

**Example Output:**
```
List of enabled modules:
Magento_Store
Magento_AdvancedPricingImportExport
...
Vendor_CustomModule

List of disabled modules:
Magento_Marketplace
Vendor_DisabledModule

Total number of modules: 276
```

**Useful Filters:**

```bash
# Show only enabled modules
bin/magento module:status | grep -A 200 "enabled"

# Show only disabled modules
bin/magento module:status | grep -A 200 "disabled"

# Show only custom modules
bin/magento module:status | grep -v "Magento_"

# Count enabled modules
bin/magento module:status | grep -A 200 "enabled" | grep -c "Magento_"
```

### Config Commands

Manage system configuration.

#### config:set

**Description:** Set configuration value in database.

**Syntax:**
```bash
bin/magento config:set [options] path value
```

**Options:**
- `--scope=<scope>` - Config scope (default, websites, stores)
- `--scope-code=<code>` - Scope code (website/store code)
- `--lock` - Lock value (can't be changed in Admin)
- `--lock-env` - Lock value, require env var to change
- `--lock-config` - Lock value, require config file to change

**Examples:**

```bash
# Set global configuration
bin/magento config:set web/unsecure/base_url "https://example.com/"

# Set for specific website
bin/magento config:set --scope=websites --scope-code=base web/secure/base_url "https://example.com/"

# Set for specific store view
bin/magento config:set --scope=stores --scope-code=default general/locale/timezone "America/New_York"

# Lock configuration value
bin/magento config:set --lock dev/debug/debug_logging 1

# Common settings
bin/magento config:set web/secure/use_in_frontend 1
bin/magento config:set web/secure/use_in_adminhtml 1
bin/magento config:set dev/static/sign 0
bin/magento config:set admin/security/session_lifetime 86400

# Disable modules via config
bin/magento config:set dev/debug/template_hints_storefront 1
bin/magento config:set dev/debug/template_hints_admin 1
```

**Important Paths:**

| Path | Description | Values |
|------|-------------|--------|
| `web/unsecure/base_url` | Base URL (HTTP) | URL |
| `web/secure/base_url` | Base URL (HTTPS) | URL |
| `web/secure/use_in_frontend` | Use HTTPS on frontend | 0, 1 |
| `web/secure/use_in_adminhtml` | Use HTTPS in admin | 0, 1 |
| `dev/static/sign` | Static file signing | 0, 1 |
| `dev/debug/debug_logging` | Debug logging | 0, 1 |
| `dev/debug/template_hints_storefront` | Template hints (frontend) | 0, 1 |
| `admin/security/session_lifetime` | Admin session lifetime | seconds |
| `catalog/search/engine` | Search engine | elasticsearch7, opensearch |

#### config:show

**Description:** Show configuration value from database.

**Syntax:**
```bash
bin/magento config:show [path]
```

**Examples:**

```bash
# Show all configuration
bin/magento config:show

# Show specific path
bin/magento config:show web/secure/base_url

# Show all web configuration
bin/magento config:show | grep "web/"

# Show all paths containing "base_url"
bin/magento config:show | grep base_url

# Check search engine config
bin/magento config:show catalog/search/engine
```

**Expected Output:**
```
web/unsecure/base_url - https://example.com/
web/secure/base_url - https://example.com/
```

#### app:config:dump

**Description:** Create application dump (export config to files).

**Syntax:**
```bash
bin/magento app:config:dump [types]
```

**Types:**
- `scopes` - Scope configuration (stores, websites)
- `system` - System configuration
- `themes` - Themes configuration
- `i18n` - Translation configuration

**Examples:**

```bash
# Dump all configuration
bin/magento app:config:dump

# Dump only system config
bin/magento app:config:dump system

# Dump scopes (for multi-store setup)
bin/magento app:config:dump scopes
```

**Output Files:**
- `app/etc/config.php` - Module list, scopes, themes
- `app/etc/env.php` - Environment-specific config (DB, cache, etc.)

**Use Case - Pipeline Deployment:**

```bash
# Step 1: On development, export config
bin/magento app:config:dump

# Step 2: Commit config.php to Git
git add app/etc/config.php
git commit -m "Export config"

# Step 3: On production, import config
git pull
bin/magento app:config:import
bin/magento cache:flush
```

### Cron Commands

Manage scheduled tasks.

#### cron:run

**Description:** Run cron jobs manually.

**Syntax:**
```bash
bin/magento cron:run [options]
```

**Options:**
- `--group=<group>` - Run specific cron group
- `--exclude-group=<group>` - Exclude cron group

**Cron Groups:**

| Group | Purpose | Frequency |
|-------|---------|-----------|
| `default` | Default tasks (indexing, cleanup) | Every 1 min |
| `index` | Index updates (schedule mode) | Every 1 min |
| `consumers` | Message queue consumers | Every 1 min |
| `catalog_event` | Catalog events | Every 1 min |

**Examples:**

```bash
# Run all cron groups
bin/magento cron:run

# Run specific group
bin/magento cron:run --group=index

# Run default group only
bin/magento cron:run --group=default

# Exclude group
bin/magento cron:run --exclude-group=catalog_event

# Run in verbose mode with logging
bin/magento cron:run 2>&1 | tee cron.log

# Monitor cron execution
tail -f var/log/cron.log
```

**Crontab Setup:**

```bash
# Install Magento cron
bin/magento cron:install

# Verify crontab entry
crontab -l

# Expected output:
# * * * * * /usr/bin/php /var/www/html/bin/magento cron:run 2>&1 | grep -v "Ran jobs by schedule" >> /var/www/html/var/log/magento.cron.log
# * * * * * /usr/bin/php /var/www/html/update/cron.php >> /var/www/html/var/log/update.cron.log
# Note: setup:cron:run does not exist. The correct command is cron:run (shown above).
```

#### cron:install

**Description:** Install cron jobs in system crontab.

**Syntax:**
```bash
bin/magento cron:install [options]
```

**Options:**
- `-f, --force` - Force install (replace existing)
- `-d <path>` - Path to Magento installation

**Examples:**

```bash
# Standard install
bin/magento cron:install

# Force reinstall
bin/magento cron:install --force

# Install with custom path
bin/magento cron:install -d /var/www/html
```

#### cron:remove

**Description:** Remove cron jobs from system crontab.

**Syntax:**
```bash
bin/magento cron:remove
```

**Example:**

```bash
# Remove Magento cron entries
bin/magento cron:remove

# Verify removal
crontab -l  # Should not show Magento entries
```

### Deploy Commands

Manage deployment mode.

#### deploy:mode:set

**Description:** Set application deployment mode.

**Syntax:**
```bash
bin/magento deploy:mode:set {developer|production|default} [options]
```

**Options:**
- `-s, --skip-compilation` - Skip code compilation

**Modes:**

| Mode | Static Files | Error Display | Performance | Use Case |
|------|-------------|---------------|-------------|----------|
| `developer` | Generated on-the-fly | Verbose errors | Slow | Local development |
| `production` | Pre-compiled | Generic errors | Fast | Live site |
| `default` | Pre-compiled | Some errors | Medium | Staging |

**Examples:**

```bash
# Switch to developer mode
bin/magento deploy:mode:set developer

# Switch to production mode
bin/magento deploy:mode:set production

# Switch to production, skip compilation
bin/magento deploy:mode:set production --skip-compilation

# Full production deployment
bin/magento deploy:mode:set production
bin/magento setup:di:compile
bin/magento setup:static-content:deploy -f
bin/magento cache:flush
```

**Expected Output:**
```
Enabled maintenance mode
Requested modes: production
Generated code and dependency injection configuration successfully.
Deployed static content version: 1234567890
Successfully deployed static content.
Disabled maintenance mode
```

**Mode Characteristics:**

```bash
# Developer mode
- Static files symlinked from source
- Exceptions displayed with stack trace
- Error logging: verbose
- Performance: slow (no compilation)
- Auto-compilation on-the-fly

# Production mode
- Static files deployed to pub/static
- Exceptions: generic error messages
- Error logging: minimal (logs only)
- Performance: fast (pre-compiled)
- Maintenance mode during switch

# Default mode
- Static files deployed to pub/static
- Exceptions: some details shown
- Error logging: balanced
- Performance: medium
- Hybrid mode (rarely used)
```

#### deploy:mode:show

**Description:** Show current deployment mode.

**Syntax:**
```bash
bin/magento deploy:mode:show
```

**Example Output:**
```
Current application mode: production
```

### Maintenance Commands

Control maintenance mode (displays "Service Temporarily Unavailable" to visitors).

#### maintenance:enable

**Description:** Enable maintenance mode.

**Syntax:**
```bash
bin/magento maintenance:enable [options]
```

**Options:**
- `--ip=<IP>` - Allow specific IP addresses (comma-separated)

**Examples:**

```bash
# Enable maintenance mode
bin/magento maintenance:enable

# Enable with IP whitelist
bin/magento maintenance:enable --ip=192.168.1.100

# Enable with multiple IPs
bin/magento maintenance:enable --ip=192.168.1.100,192.168.1.101

# Allow local development IP
bin/magento maintenance:enable --ip=127.0.0.1
```

**What Happens:**
- Creates `var/.maintenance.flag` file
- Displays maintenance page to all visitors (except whitelisted IPs)
- Admin panel remains accessible

**Custom Maintenance Page:**

```bash
# Create custom maintenance page
# pub/errors/503.phtml (customize this file)

# Enable maintenance
bin/magento maintenance:enable
```

#### maintenance:disable

**Description:** Disable maintenance mode.

**Syntax:**
```bash
bin/magento maintenance:disable
```

**Example:**

```bash
# Disable maintenance mode
bin/magento maintenance:disable
```

**Verification:**

```bash
# Check maintenance status
bin/magento maintenance:status

# Expected output when disabled:
# Status: maintenance mode is not active
```

#### maintenance:status

**Description:** Show current maintenance mode status.

**Syntax:**
```bash
bin/magento maintenance:status
```

**Example Output:**
```
Status: maintenance mode is active
List of exempt IP-addresses: 192.168.1.100
```

#### maintenance:allow-ips

**Description:** Set allowed IP addresses during maintenance mode.

**Syntax:**
```bash
bin/magento maintenance:allow-ips [options] [ip-addresses]
```

**Options:**
- `--none` - Clear IP whitelist

**Examples:**

```bash
# Add allowed IPs
bin/magento maintenance:allow-ips 192.168.1.100 192.168.1.101

# Clear all allowed IPs
bin/magento maintenance:allow-ips --none

# Add current IP (dynamic)
CURRENT_IP=$(curl -s ifconfig.me)
bin/magento maintenance:allow-ips $CURRENT_IP
```

### Customer Commands

Customer-related operations.

#### customer:hash:upgrade

**Description:** Upgrade customer password hashes to latest algorithm.

**Syntax:**
```bash
bin/magento customer:hash:upgrade
```

**When to Use:**
- After upgrading Magento (new hashing algorithm available)
- Improved security by upgrading to stronger hashing

**Example:**

```bash
# Upgrade all customer password hashes
bin/magento customer:hash:upgrade
```

**Expected Output:**
```
Finished updating password hashes.
```

### Developer Commands

Development and testing tools.

#### dev:tests:run

**Description:** Run tests (unit, integration, functional).

**Syntax:**
```bash
bin/magento dev:tests:run {unit|integration|functional} [options]
```

**Options:**
- `--filter=<pattern>` - Run tests matching pattern
- `--testsuite=<suite>` - Run specific test suite

**Examples:**

```bash
# Run unit tests
bin/magento dev:tests:run unit

# Run integration tests
bin/magento dev:tests:run integration

# Run specific test class
bin/magento dev:tests:run unit --filter="Vendor\\Module\\Test\\Unit\\Model\\ProductTest"

# Run with custom PHPUnit config
vendor/bin/phpunit -c dev/tests/unit/phpunit.xml.dist app/code/Vendor/Module/Test/Unit
```

#### dev:query-log:enable

**Description:** Enable database query logging.

**Syntax:**
```bash
bin/magento dev:query-log:enable [options]
```

**Options:**
- `--include-all-queries=<0|1>` - Log all queries (default: 1)
- `--query-time-threshold=<ms>` - Only log queries slower than threshold
- `--include-call-stack=<0|1>` - Include call stack in log

**Examples:**

```bash
# Enable query logging
bin/magento dev:query-log:enable

# Enable with call stack
bin/magento dev:query-log:enable --include-call-stack=1

# Log only slow queries (> 100ms)
bin/magento dev:query-log:enable --query-time-threshold=100

# View query log
tail -f var/debug/db.log
```

#### dev:query-log:disable

**Description:** Disable database query logging.

**Syntax:**
```bash
bin/magento dev:query-log:disable
```

#### dev:source-theme:deploy

**Description:** Deploy source theme files (for theme development).

**Syntax:**
```bash
bin/magento dev:source-theme:deploy [options] [themes]
```

**Options:**
- `--locale=<locale>` - Deploy specific locale
- `--area=<area>` - Deploy specific area

**Examples:**

```bash
# Deploy all themes
bin/magento dev:source-theme:deploy

# Deploy specific theme
bin/magento dev:source-theme:deploy Magento/luma

# Deploy theme for specific locale
bin/magento dev:source-theme:deploy --locale=en_US --area=frontend Vendor/custom-theme
```

#### dev:template-hints:enable / disable

**Description:** Enable/disable template hints (shows template file paths on frontend).

**Syntax:**
```bash
bin/magento dev:template-hints:enable [options]
bin/magento dev:template-hints:disable
```

**Options:**
- `--area=<area>` - Enable for specific area (frontend, adminhtml)

**Examples:**

```bash
# Enable template hints for frontend
bin/magento dev:template-hints:enable

# Enable for storefront only
bin/magento dev:template-hints:enable --area=frontend

# Disable template hints
bin/magento dev:template-hints:disable
```

**Equivalent Config:**

```bash
# Via config:set
bin/magento config:set dev/debug/template_hints_storefront 1
bin/magento config:set dev/debug/template_hints_blocks 1
```

### Other Useful Commands

#### admin:user:create

**Description:** Create admin user.

**Syntax:**
```bash
bin/magento admin:user:create [options]
```

**Options:**
- `--admin-user` - Admin username
- `--admin-password` - Admin password
- `--admin-email` - Admin email
- `--admin-firstname` - Admin first name
- `--admin-lastname` - Admin last name

**Example:**

```bash
bin/magento admin:user:create \
  --admin-user="john.doe" \
  --admin-password="SecurePass123!" \
  --admin-email="john.doe@example.com" \
  --admin-firstname="John" \
  --admin-lastname="Doe"
```

#### admin:user:unlock

**Description:** Unlock admin user account.

**Syntax:**
```bash
bin/magento admin:user:unlock <username>
```

**Example:**

```bash
# Unlock admin user
bin/magento admin:user:unlock admin

# Check if unlock was successful
mysql -u root -p magento -e "SELECT username, is_active, failures_num FROM admin_user WHERE username='admin';"
```

#### catalog:images:resize

**Description:** Resize product images (catalog image cache).

**Syntax:**
```bash
bin/magento catalog:images:resize
```

**Example:**

```bash
# Resize all product images
bin/magento catalog:images:resize

# Clear image cache first, then resize
rm -rf pub/media/catalog/product/cache/*
bin/magento catalog:images:resize
```

**When to Use:**
- After changing image size configuration
- After mass product import with new images
- After theme installation

#### catalog:product:attributes:cleanup

**Description:** Remove unused product attributes.

**Syntax:**
```bash
bin/magento catalog:product:attributes:cleanup
```

**Example:**

```bash
# Cleanup unused attributes
bin/magento catalog:product:attributes:cleanup
```

#### encryption:payment-data:update

**Description:** Re-encrypt payment data with new encryption key.

**Syntax:**
```bash
bin/magento encryption:payment-data:update
```

**When to Use:**
- After rotating encryption key in `app/etc/env.php`

**Example:**

```bash
# Backup env.php first
cp app/etc/env.php app/etc/env.php.backup

# Update encryption key in env.php
# Then re-encrypt payment data
bin/magento encryption:payment-data:update
```

#### store:list

**Description:** List all stores, websites, and store views.

**Syntax:**
```bash
bin/magento store:list
```

**Example Output:**
```
+----------+--------------+-----------+
| ID       | Website Code | Store Code |
+----------+--------------+-----------+
| 1        | base         | default   |
| 2        | base         | german    |
+----------+--------------+-----------+
```

#### store:website:list

**Description:** List all websites.

**Syntax:**
```bash
bin/magento store:website:list
```

## Creating Custom CLI Commands

Complete example of creating a custom CLI command.

### Step 1: Create Command Class

**File:** `app/code/Vendor/Module/Console/Command/CustomCommand.php`

```php
<?php

declare(strict_types=1);

namespace Vendor\Module\Console\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Magento\Framework\App\State;
use Psr\Log\LoggerInterface;

class CustomCommand extends Command
{
    private const COMMAND_NAME = 'vendor:custom:command';
    private const ARGUMENT_NAME = 'name';
    private const OPTION_ENABLE = 'enable';

    /**
     * @var State
     */
    private State $state;

    /**
     * @var LoggerInterface
     */
    private LoggerInterface $logger;

    /**
     * @param State $state
     * @param LoggerInterface $logger
     * @param string|null $name
     */
    public function __construct(
        State $state,
        LoggerInterface $logger,
        string $name = null
    ) {
        $this->state = $state;
        $this->logger = $logger;
        parent::__construct($name);
    }

    /**
     * Configure command
     */
    protected function configure(): void
    {
        $this->setName(self::COMMAND_NAME)
            ->setDescription('Custom command description')
            ->setDefinition([
                new InputArgument(
                    self::ARGUMENT_NAME,
                    InputArgument::REQUIRED,
                    'Name argument (required)'
                ),
                new InputOption(
                    self::OPTION_ENABLE,
                    '-e',
                    InputOption::VALUE_NONE,
                    'Enable option (flag)'
                )
            ]);

        parent::configure();
    }

    /**
     * Execute command
     *
     * @param InputInterface $input
     * @param OutputInterface $output
     * @return int
     */
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        try {
            // Set area code (required for some operations)
            $this->state->setAreaCode(\Magento\Framework\App\Area::AREA_ADMINHTML);

            // Get input values
            $name = $input->getArgument(self::ARGUMENT_NAME);
            $isEnabled = $input->getOption(self::OPTION_ENABLE);

            // Output
            $output->writeln("<info>Processing: {$name}</info>");

            if ($isEnabled) {
                $output->writeln("<comment>Enable option is set</comment>");
            }

            // Your business logic here
            $this->processCustomLogic($name, $isEnabled);

            $output->writeln("<info>Command completed successfully</info>");
            return Command::SUCCESS;

        } catch (\Exception $e) {
            $this->logger->error($e->getMessage());
            $output->writeln("<error>{$e->getMessage()}</error>");
            return Command::FAILURE;
        }
    }

    /**
     * Custom business logic
     *
     * @param string $name
     * @param bool $isEnabled
     * @return void
     */
    private function processCustomLogic(string $name, bool $isEnabled): void
    {
        // Implementation
    }
}
```

### Step 2: Register Command in DI

**File:** `app/code/Vendor/Module/etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Framework\Console\CommandList">
        <arguments>
            <argument name="commands" xsi:type="array">
                <item name="vendorCustomCommand" xsi:type="object">Vendor\Module\Console\Command\CustomCommand</item>
            </argument>
        </arguments>
    </type>
</config>
```

### Step 3: Usage

```bash
# Clear cache and regenerate DI
bin/magento cache:clean
bin/magento setup:di:compile

# Run custom command
bin/magento vendor:custom:command "test" --enable

# Get help
bin/magento vendor:custom:command --help

# Expected output:
# Description:
#   Custom command description
#
# Usage:
#   vendor:custom:command [options] [--] <name>
#
# Arguments:
#   name                  Name argument (required)
#
# Options:
#   -e, --enable          Enable option (flag)
```

### Advanced Features

**Progress Bar:**

```php
use Symfony\Component\Console\Helper\ProgressBar;

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $items = range(1, 100);

    $progressBar = new ProgressBar($output, count($items));
    $progressBar->start();

    foreach ($items as $item) {
        // Process item
        sleep(1);
        $progressBar->advance();
    }

    $progressBar->finish();
    $output->writeln('');

    return Command::SUCCESS;
}
```

**Table Output:**

```php
use Symfony\Component\Console\Helper\Table;

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $table = new Table($output);
    $table->setHeaders(['ID', 'Name', 'Status'])
        ->setRows([
            [1, 'Product A', 'Active'],
            [2, 'Product B', 'Inactive'],
        ]);
    $table->render();

    return Command::SUCCESS;
}
```

**Interactive Prompts:**

```php
use Symfony\Component\Console\Question\ConfirmationQuestion;

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $helper = $this->getHelper('question');
    $question = new ConfirmationQuestion('Continue with this action? (y/N) ', false);

    if (!$helper->ask($input, $output, $question)) {
        $output->writeln('Action cancelled');
        return Command::FAILURE;
    }

    // Continue with action
    return Command::SUCCESS;
}
```

## Command Cheat Sheet Table

| Task | Command |
|------|---------|
| Clear cache | `bin/magento cache:flush` |
| Reindex all | `bin/magento indexer:reindex` |
| Deploy static content | `bin/magento setup:static-content:deploy -f` |
| Switch to production | `bin/magento deploy:mode:set production` |
| Enable maintenance | `bin/magento maintenance:enable` |
| Disable maintenance | `bin/magento maintenance:disable` |
| Enable module | `bin/magento module:enable Vendor_Module` |
| Disable module | `bin/magento module:disable Vendor_Module` |
| Run setup upgrade | `bin/magento setup:upgrade` |
| Compile DI | `bin/magento setup:di:compile` |
| Set config value | `bin/magento config:set path/to/config value` |
| Show config value | `bin/magento config:show path/to/config` |
| Run cron manually | `bin/magento cron:run` |
| Install cron | `bin/magento cron:install` |
| Check indexer status | `bin/magento indexer:status` |
| Set indexer mode | `bin/magento indexer:set-mode schedule` |
| Create admin user | `bin/magento admin:user:create` |
| Unlock admin user | `bin/magento admin:user:unlock <username>` |
| Resize product images | `bin/magento catalog:images:resize` |
| List stores | `bin/magento store:list` |
| Enable template hints | `bin/magento dev:template-hints:enable` |
| Enable query logging | `bin/magento dev:query-log:enable` |
| Check Magento version | `bin/magento --version` |

## Common Command Workflows

### Full Deployment Workflow

```bash
# 1. Put site in maintenance mode
bin/magento maintenance:enable --ip=127.0.0.1

# 2. Pull latest code
git pull origin main

# 3. Update dependencies
composer install --no-dev --optimize-autoloader

# 4. Upgrade database
bin/magento setup:upgrade

# 5. Compile DI
bin/magento setup:di:compile

# 6. Deploy static content
bin/magento setup:static-content:deploy -f -j 8 en_US

# 7. Clear caches
bin/magento cache:flush

# 8. Reindex (if in schedule mode, this happens via cron)
bin/magento indexer:reindex

# 9. Disable maintenance mode
bin/magento maintenance:disable

# 10. Verify deployment
bin/magento deploy:mode:show
```

### Module Installation Workflow

```bash
# 1. Install module via Composer
composer require vendor/module-name

# 2. Enable module
bin/magento module:enable Vendor_ModuleName

# 3. Run setup upgrade
bin/magento setup:upgrade

# 4. Compile DI
bin/magento setup:di:compile

# 5. Deploy static content (if module has frontend assets)
bin/magento setup:static-content:deploy -f

# 6. Clear cache
bin/magento cache:flush

# 7. Verify module is enabled
bin/magento module:status | grep Vendor_ModuleName
```

### Troubleshooting Workflow

```bash
# 1. Enable developer mode for detailed errors
bin/magento deploy:mode:set developer

# 2. Disable all third-party modules
bin/magento module:status | grep -v "Magento_" | grep -v "List" | awk '{print $1}' | xargs bin/magento module:disable

# 3. Run setup upgrade
bin/magento setup:upgrade

# 4. Clear all caches and generated files
rm -rf var/cache/* var/page_cache/* generated/* pub/static/*
bin/magento cache:flush

# 5. Re-enable modules one by one
bin/magento module:enable Vendor_ModuleOne
bin/magento setup:upgrade
# Test site

# 6. Enable query logging to debug database issues
bin/magento dev:query-log:enable --include-call-stack=1

# 7. Monitor logs
tail -f var/log/system.log var/log/exception.log var/debug/db.log
```

### Performance Optimization Workflow

```bash
# 1. Set production mode
bin/magento deploy:mode:set production

# 2. Enable all caches
bin/magento cache:enable

# 3. Set indexers to schedule mode
bin/magento indexer:set-mode schedule

# 4. Install cron
bin/magento cron:install

# 5. Optimize images
bin/magento catalog:images:resize

# 6. Deploy static content with optimizations
bin/magento setup:static-content:deploy -f --strategy=compact -j 8 en_US

# 7. Verify cache status
bin/magento cache:status

# 8. Verify indexer status
bin/magento indexer:status
```

## Assumptions

**Magento/PHP versions:** Commands documented for Adobe Commerce 2.4.7/2.4.8 with PHP 8.2/8.3. Most commands work identically in 2.4.x series; some options differ in 2.3.x.

**Modules:** Assumes default Magento installation. Custom modules may add additional CLI commands via `etc/di.xml`.

**Areas:** CLI commands execute in `global` area by default. Some commands set specific areas (e.g., `setup:upgrade` uses `adminhtml` for certain operations).

**Scopes:** Configuration commands (`config:set`, `config:show`) operate on default scope unless `--scope` and `--scope-code` options are specified.

**Environment:** Assumes standard server environment with SSH access, cron capability, and sufficient PHP memory (2-4 GB for compilation/deployment). Cloud environments may require platform-specific commands (`ece-tools` on Adobe Commerce Cloud).

## Why This Approach

**Comprehensive command reference:** Developers need a single source for all CLI commands, including syntax, options, and real-world examples. Official docs are spread across multiple pages.

**Examples with output:** Showing expected output helps developers verify commands executed correctly and debug issues when output differs.

**Grouped by category:** Organizing commands by function (cache, indexer, setup, etc.) makes it easier to find the right command for a task vs alphabetical listing.

**Common workflows:** Real deployment/troubleshooting workflows show how commands are chained together, which is more valuable than isolated command documentation.

**Custom command guide:** Including complete example of creating custom CLI commands empowers developers to extend Magento's CLI capabilities.

**Alternatives considered:**
- Alphabetical listing: Harder to find related commands
- Man-page style: Too verbose, less practical
- Interactive tool: Not portable, requires separate tool

## Security Impact

**authZ/authN:** CLI commands require direct server access (SSH). No built-in authentication beyond OS-level access control. Secure SSH keys and `sudo` policies are critical.

**CSRF/form key:** Not applicable (CLI doesn't use web sessions).

**XSS escaping:** Not applicable (CLI operates in terminal, not browser).

**Secrets:** Be cautious with commands that output sensitive data:

```bash
# AVOID: Outputting passwords/keys to terminal or logs
bin/magento config:show payment/braintree/private_key

# BETTER: Store sensitive config in env.php or environment variables
```

**PII:** Commands like `customer:hash:upgrade` and `admin:user:create` handle PII. Ensure CLI execution logs (`var/log/*.log`) are not publicly accessible.

**Audit logging:** CLI commands are not logged in Magento's Admin Actions Log. Use OS-level audit tools (auditd, syslog) to track CLI command execution for compliance.

## Performance Impact

**Cacheability (FPC/Varnish):** `cache:flush` and `cache:clean` clear FPC, causing all pages to regenerate on next request. In production, clear cache during low-traffic periods or use cache warming.

**Redis tags:** `cache:flush` sends `FLUSHALL` to Redis (if configured), clearing all keys including session data. Users will be logged out. Use `cache:clean` for selective clearing.

**DB load:**
- `indexer:reindex` - HIGH load on large catalogs (full table scans, writes)
- `setup:upgrade` - MEDIUM load (DDL operations, data patches)
- `catalog:images:resize` - MEDIUM CPU/disk I/O

**CWV (Core Web Vitals):**
- `setup:static-content:deploy` - Regenerates CSS/JS bundles; can improve LCP/CLS if minification/bundling optimized
- `catalog:images:resize` - Optimizes image sizes; improves LCP

**Command execution times (typical production environment):**
- `cache:flush`: 1-5 seconds
- `setup:di:compile`: 3-10 minutes
- `setup:static-content:deploy`: 5-30 minutes
- `indexer:reindex`: 10 minutes - 2 hours (depends on catalog size)

**Mitigation:**
- Run intensive commands (`setup:di:compile`, `setup:static-content:deploy`, `indexer:reindex`) during maintenance windows
- Use `--jobs` option for parallel processing
- Use `indexer:set-mode schedule` and let cron handle reindexing

## Backward Compatibility

**API/DB schema changes:** CLI commands are part of Magento's public API. New commands added in minor versions (2.4.7, 2.4.8) are BC-safe. Command removal or option changes are rare and announced as breaking changes.

**Upgrade/migration notes:**
- **2.4.6 → 2.4.7:** No CLI command removals. New commands added: `customer:hash:upgrade` enhancements.
- **2.4.7 → 2.4.8:** Expected additions for OpenSearch config, no known deprecations.

**Custom commands:** Use Symfony Console component's semantic versioning. Always extend `Command` class from `Symfony\Component\Console\Command\Command`, not Magento's deprecated abstractions.

**Scripting compatibility:** CLI commands return exit codes (`0` = success, `1` = failure). Scripts should check `$?` for command success:

```bash
#!/bin/bash
bin/magento setup:upgrade
if [ $? -ne 0 ]; then
    echo "Upgrade failed"
    exit 1
fi
```

## Tests to Add

**Unit tests:**

```php
// Test custom command logic
class CustomCommandTest extends \PHPUnit\Framework\TestCase
{
    public function testExecuteReturnsSuccess()
    {
        $command = new CustomCommand($this->stateMock, $this->loggerMock);
        $input = new ArrayInput(['name' => 'test']);
        $output = new BufferedOutput();

        $result = $command->run($input, $output);
        $this->assertEquals(Command::SUCCESS, $result);
    }
}
```

**Integration tests:**

```php
// Test command execution in real environment
class CommandIntegrationTest extends \Magento\TestFramework\TestCase\AbstractController
{
    public function testSetupUpgradeCommand()
    {
        $command = $this->objectManager->create(\Magento\Setup\Console\Command\UpgradeCommand::class);
        $input = new ArrayInput([]);
        $output = new BufferedOutput();

        $command->run($input, $output);
        $this->assertStringContainsString('success', $output->fetch());
    }
}
```

**Functional tests (shell scripts):**

```bash
#!/bin/bash
# test_cli_commands.sh

# Test cache commands
bin/magento cache:flush
if [ $? -ne 0 ]; then
    echo "FAIL: cache:flush"
    exit 1
fi

# Test indexer commands
bin/magento indexer:status
if [ $? -ne 0 ]; then
    echo "FAIL: indexer:status"
    exit 1
fi

echo "PASS: All CLI commands executed successfully"
```

## Docs to Update

**README:** Add "CLI Commands" section with link to this guide. Include quick start for common tasks.

**CHANGELOG:** Document new CLI commands added in each release:

```markdown
## [2.4.7]
### Added
- `customer:hash:upgrade` - Upgrade customer password hashes
- Enhanced `dev:query-log:enable` with call stack support
```

**Admin user guide paths:**
1. System > Tools > Index Management
   - Screenshot: Indexer status grid
   - Note: CLI equivalent: `bin/magento indexer:status`
2. System > Tools > Cache Management
   - Screenshot: Cache types list
   - Note: CLI equivalent: `bin/magento cache:status`
3. Stores > Configuration > Advanced > Developer > Debug
   - Screenshot: Template hints settings
   - Note: CLI equivalent: `bin/magento dev:template-hints:enable`

**Screenshots to capture:**
- Terminal output of `bin/magento list` showing all commands
- `bin/magento indexer:status` output
- `bin/magento setup:static-content:deploy` progress
- Custom command help output (`bin/magento vendor:custom:command --help`)

---

**Related Resources:**
- [Magento DevDocs - Command-line Tool](https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/cli/config-cli.html)
- [Symfony Console Component](https://symfony.com/doc/current/components/console.html)
- [Magento Technical Guidelines - CLI](https://developer.adobe.com/commerce/php/development/cli-commands/)
- [Magento Performance Best Practices](https://experienceleague.adobe.com/docs/commerce-operations/performance-best-practices/overview.html)

## Related Documentation

### Related Guides

- [CI/CD Deployment Pipelines for Magento 2](../how-to/cicd-deployment.md)
- [Indexer System Deep Dive: Understanding Magento 2's Data Indexing Architecture](../explanations/indexer-system.md)
- [Cron Jobs Implementation: Building Reliable Scheduled Tasks in Magento 2](../how-to/cron-jobs.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Customer Overview](../../modules/customer/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
