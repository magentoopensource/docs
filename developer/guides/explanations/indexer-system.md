---
title: "Indexer System Deep Dive: Understanding Magento 2's Data Indexing Architecture"
description: "Comprehensive explanation of Magento 2's indexer system: Mview architecture, changelog tables, creating custom indexers, partial vs full reindex, optimization strategies, scheduled vs real-time indexing, and debugging index issues"
type: "explanation"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "90 minutes"
topics:
  - indexer
  - mview
  - changelog
  - performance
  - database
  - optimization
  - triggers
  - cron
last_updated: "2026-02-07"
---

# Indexer System Deep Dive: Understanding Magento 2's Data Indexing Architecture

## Learning Objectives

By completing this guide, you will:

- Understand Magento 2's indexer architecture and the Mview (Materialized View) framework
- Master changelog tables and MySQL triggers for incremental indexing
- Create custom indexers with proper dependency management
- Distinguish between full reindex, partial reindex, and update on save
- Optimize index performance for large catalogs
- Choose between scheduled and real-time (update on save) indexing modes
- Debug index-related issues and inconsistencies
- Monitor and maintain index health in production environments

## Introduction

Magento 2's indexer system is a **denormalization engine** that transforms normalized database data into optimized flat structures for fast read operations. Without indexing, every product list page would require hundreds of database queries across EAV tables, categories, inventory, and prices.

### Why Indexing is Critical

**Performance impact:**
- **Without indexes**: Product list query with 100 products = 500+ queries
- **With indexes**: Same query = 1-2 queries to flat index tables

**Real-world example:**
- Category page loads in 200ms with indexes
- Same page loads in 5-10 seconds without indexes (timeout)

### Core Indexers in Magento 2

| Indexer | Purpose | Index Tables |
|---------|---------|--------------|
| **catalog_product_attribute** | Product EAV attributes flattened for layered navigation | `catalog_product_index_eav_*` |
| **catalog_product_price** | Product final prices (tier pricing, special prices, customer groups) | `catalog_product_index_price*` |
| **catalog_category_product** | Product-category associations with position | `catalog_category_product_index*` |
| **cataloginventory_stock** | Stock status (in stock/out of stock) per source | `cataloginventory_stock_status` |
| **catalogsearch_fulltext** | Elasticsearch/OpenSearch product search index | External search engine |
| **catalog_product_flat** | All product attributes in single table (deprecated) | `catalog_product_flat_*` |
| **customer_grid** | Customer list for admin grid | `customer_grid_flat` |
| **design_config_grid** | Design configuration for admin | `design_config_grid_flat` |

## Indexer Architecture

### Components Overview

```
Source Data (EAV, Relations)
  ↓
Changelog Tables (track changes via triggers)
  ↓
Mview (Materialized View Framework)
  ↓
Indexer Action Classes (execute reindex logic)
  ↓
Index Tables (denormalized data)
  ↓
Frontend Queries (fast reads)
```

### Mview (Materialized View) Framework

**Mview** is Magento's framework for managing incremental index updates using changelog tables and database triggers.

**Key concepts:**
- **Changelog table**: Records which entity IDs changed since last reindex
- **Subscription**: Links a changelog to an index table
- **Version**: Tracks last processed changelog record
- **Trigger**: MySQL trigger that writes to changelog on data changes

**Mview flow:**
1. Entity changes (product save, stock update) → MySQL trigger fires
2. Trigger writes entity ID to changelog table (`*_cl` tables)
3. Cron job (`indexer_update_all_views`) processes changelog
4. Indexer reads changed IDs from changelog
5. Indexer reindexes only changed entities (partial reindex)
6. Version ID updated to mark changelog as processed

**Advantages:**
- **Incremental updates**: Only reindex changed entities
- **Decoupled**: Source data changes don't block index updates
- **Scheduled**: Index updates run via cron, not inline during admin saves

## Changelog Tables and Triggers

### Changelog Table Structure

Every Mview-enabled indexer has a changelog table with this structure:

```sql
CREATE TABLE `catalog_category_product_cl` (
  `version_id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'Version ID',
  `entity_id` INT(10) UNSIGNED NOT NULL DEFAULT 0 COMMENT 'Entity ID (product_id)',
  PRIMARY KEY (`version_id`),
  KEY `IDX_CATALOG_CATEGORY_PRODUCT_CL_ENTITY_ID` (`entity_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='catalog_category_product_cl';
```

**Fields:**
- `version_id`: Auto-increment primary key tracking changelog entry order
- `entity_id`: ID of changed entity (e.g., product_id, category_id)

**Key characteristics:**
- Lightweight (2 columns only)
- Fast inserts (triggered on every entity change)
- No foreign keys (performance optimization)

### MySQL Triggers

Magento creates triggers on source tables that write to changelog tables.

**Example trigger for `catalog_category_product`:**

```sql
CREATE TRIGGER trg_catalog_category_product_after_insert
AFTER INSERT ON catalog_category_product
FOR EACH ROW
BEGIN
    INSERT IGNORE INTO catalog_category_product_cl (entity_id) VALUES (NEW.product_id);
END;

CREATE TRIGGER trg_catalog_category_product_after_update
AFTER UPDATE ON catalog_category_product
FOR EACH ROW
BEGIN
    INSERT IGNORE INTO catalog_category_product_cl (entity_id) VALUES (NEW.product_id);
END;

CREATE TRIGGER trg_catalog_category_product_after_delete
AFTER DELETE ON catalog_category_product
FOR EACH ROW
BEGIN
    INSERT IGNORE INTO catalog_category_product_cl (entity_id) VALUES (OLD.product_id);
END;
```

**Trigger behavior:**
- `INSERT IGNORE`: Prevents duplicate entries (same entity ID already in changelog)
- Writes happen **after** main table modification (consistent state)
- Performance impact: <1ms per trigger (acceptable overhead)

### Viewing Changelog Data

```bash
# Count pending changelog entries
mysql> SELECT COUNT(*) FROM catalog_category_product_cl;

# View recent changes
mysql> SELECT * FROM catalog_category_product_cl ORDER BY version_id DESC LIMIT 100;

# Find all changelog tables
mysql> SHOW TABLES LIKE '%_cl';
```

## Indexer Modes

Magento indexers support two modes:

### 1. Update on Save (Real-time)

**Behavior:**
- Index updates **immediately** when entity is saved
- No changelog tables used
- Synchronous operation (blocks admin save)

**Configuration:**
```bash
bin/magento indexer:set-mode realtime catalog_product_price
```

**Use cases:**
- Small catalogs (<5,000 products)
- Real-time price/stock accuracy required
- Low admin save frequency

**Drawbacks:**
- Admin product save can take 5-10+ seconds
- Concurrent saves cause bottlenecks
- High database load during business hours

### 2. Update by Schedule (Scheduled)

**Behavior:**
- Changes tracked in changelog tables via triggers
- Index updates via cron job (`indexer_update_all_views`)
- Asynchronous operation (non-blocking saves)

**Configuration:**
```bash
bin/magento indexer:set-mode schedule catalog_product_price
```

**Cron job:**
```xml
<!-- vendor/magento/module-indexer/etc/crontab.xml -->
<job name="indexer_update_all_views" instance="Magento\Indexer\Cron\UpdateMview" method="execute">
    <schedule>* * * * *</schedule><!-- Every minute -->
</job>
```

**Use cases:**
- Medium to large catalogs (5,000+ products)
- High admin save frequency
- Production environments (recommended)

**Advantages:**
- Fast admin saves (<1 second)
- Controlled resource usage (cron handles load)
- Scales to millions of products

**Drawbacks:**
- Index updates delayed by cron frequency (up to 1 minute)
- Requires working cron jobs

### Mode Comparison

| Aspect | Update on Save | Update by Schedule |
|--------|----------------|---------------------|
| **Admin save time** | Slow (5-10s) | Fast (<1s) |
| **Index freshness** | Immediate | 1-minute delay |
| **Database load** | Spiky (during saves) | Steady (via cron) |
| **Scalability** | Poor | Excellent |
| **Production use** | Not recommended | Recommended |

**Best practice:** Use **scheduled mode** in production, **realtime mode** only for development/testing.

## Creating a Custom Indexer

Let's create a custom indexer that maintains a denormalized table of products with their final prices and stock status for fast API responses.

### Step 1: Define Indexer Configuration

**etc/indexer.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Indexer/etc/indexer.xsd">
    <indexer id="vendor_custom_product_data"
             view_id="vendor_custom_product_data"
             class="Vendor\Module\Model\Indexer\ProductData"
             primary="product_id">
        <title translate="true">Custom Product Data Index</title>
        <description translate="true">Maintains denormalized product data for fast API access</description>

        <!-- Dependencies: Reindex this indexer when these indexers run -->
        <depends>
            <indexer id="catalog_product_price"/>
            <indexer id="cataloginventory_stock"/>
        </depends>
    </indexer>
</config>
```

**Configuration fields:**
- `id`: Unique indexer identifier
- `view_id`: Mview subscription ID (links to mview.xml)
- `class`: Indexer action class implementing `ActionInterface`
- `primary`: Primary key column in index table
- `depends`: Indexers that should trigger this indexer's reindex

### Step 2: Define Mview Subscription

**etc/mview.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Mview/etc/mview.xsd">
    <view id="vendor_custom_product_data"
          class="Vendor\Module\Model\Indexer\ProductData"
          group="indexer">

        <!-- Changelog table -->
        <subscriptions>
            <!-- Track product changes -->
            <table name="catalog_product_entity" entity_column="entity_id"/>

            <!-- Track price changes -->
            <table name="catalog_product_entity_decimal" entity_column="entity_id">
                <column name="attribute_id"/><!-- Only track specific attributes -->
            </table>

            <!-- Track stock changes -->
            <table name="cataloginventory_stock_item" entity_column="product_id"/>

            <!-- Track category assignments -->
            <table name="catalog_category_product" entity_column="product_id"/>
        </subscriptions>
    </view>
</config>
```

**Subscription configuration:**
- `table name`: Source table to track changes on
- `entity_column`: Column containing the entity ID to write to changelog
- `column`: Filter triggers to specific columns (optional)

**Generated changelog table**: `vendor_custom_product_data_cl`

### Step 3: Create Index Table Schema

**etc/db_schema.xml**

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
    <table name="vendor_custom_product_data_index" resource="default" engine="innodb" comment="Custom Product Data Index">
        <column xsi:type="int" name="product_id" unsigned="true" nullable="false" comment="Product ID"/>
        <column xsi:type="varchar" name="sku" nullable="false" length="64" comment="SKU"/>
        <column xsi:type="varchar" name="name" nullable="false" length="255" comment="Product Name"/>
        <column xsi:type="decimal" name="price" scale="4" precision="20" unsigned="false" nullable="false" default="0" comment="Final Price"/>
        <column xsi:type="int" name="qty" unsigned="true" nullable="false" default="0" comment="Stock Quantity"/>
        <column xsi:type="smallint" name="is_in_stock" unsigned="true" nullable="false" default="0" comment="Stock Status"/>
        <column xsi:type="int" name="category_id" unsigned="true" nullable="true" comment="Primary Category ID"/>
        <column xsi:type="timestamp" name="updated_at" on_update="true" nullable="false" default="CURRENT_TIMESTAMP" comment="Last Updated"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="product_id"/>
        </constraint>

        <index referenceId="VENDOR_CUSTOM_PRODUCT_DATA_INDEX_SKU" indexType="btree">
            <column name="sku"/>
        </index>

        <index referenceId="VENDOR_CUSTOM_PRODUCT_DATA_INDEX_IS_IN_STOCK" indexType="btree">
            <column name="is_in_stock"/>
        </index>

        <index referenceId="VENDOR_CUSTOM_PRODUCT_DATA_INDEX_CATEGORY_ID" indexType="btree">
            <column name="category_id"/>
        </index>
    </table>
</schema>
```

**Index table design principles:**
- **Denormalized**: All data in single table (no joins required)
- **Primary key**: Entity ID for fast lookups
- **Indexes**: Cover query patterns (SKU lookup, stock filtering, category filtering)
- **Updated timestamp**: Track index freshness

### Step 4: Implement Indexer Action Class

**Model/Indexer/ProductData.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Indexer;

use Magento\Framework\App\ResourceConnection;
use Magento\Framework\DB\Adapter\AdapterInterface;
use Magento\Framework\Indexer\ActionInterface;
use Magento\Framework\Mview\ActionInterface as MviewActionInterface;

/**
 * Custom product data indexer
 */
class ProductData implements ActionInterface, MviewActionInterface
{
    private const INDEX_TABLE = 'vendor_custom_product_data_index';

    public function __construct(
        private readonly ResourceConnection $resourceConnection,
        private readonly ProductDataBuilder $dataBuilder
    ) {}

    /**
     * Execute full reindex
     *
     * @return void
     */
    public function executeFull(): void
    {
        $connection = $this->getConnection();

        // Truncate index table for clean rebuild
        $connection->truncateTable($this->getIndexTable());

        // Fetch all product data
        $data = $this->dataBuilder->buildAllData();

        // Insert in batches
        $this->insertBatch($connection, $data);
    }

    /**
     * Execute partial reindex for specific product IDs
     *
     * @param array $ids Product IDs to reindex
     * @return void
     */
    public function executeList(array $ids): void
    {
        if (empty($ids)) {
            return;
        }

        $connection = $this->getConnection();

        // Delete existing records for these IDs
        $connection->delete(
            $this->getIndexTable(),
            ['product_id IN (?)' => $ids]
        );

        // Fetch data for specified IDs
        $data = $this->dataBuilder->buildData($ids);

        // Insert updated data
        $this->insertBatch($connection, $data);
    }

    /**
     * Execute partial reindex for single product ID
     *
     * @param int $id Product ID
     * @return void
     */
    public function executeRow($id): void
    {
        $this->executeList([$id]);
    }

    /**
     * Execute incremental reindex based on changelog
     *
     * @param int[] $ids Changed entity IDs from changelog
     * @return void
     */
    public function execute($ids): void
    {
        $this->executeList($ids);
    }

    /**
     * Insert data in batches for memory efficiency
     */
    private function insertBatch(AdapterInterface $connection, array $data, int $batchSize = 1000): void
    {
        $batch = [];

        foreach ($data as $row) {
            $batch[] = $row;

            if (count($batch) >= $batchSize) {
                $connection->insertMultiple($this->getIndexTable(), $batch);
                $batch = [];
            }
        }

        if (!empty($batch)) {
            $connection->insertMultiple($this->getIndexTable(), $batch);
        }
    }

    /**
     * Get database connection
     */
    private function getConnection(): AdapterInterface
    {
        return $this->resourceConnection->getConnection();
    }

    /**
     * Get index table name
     */
    private function getIndexTable(): string
    {
        return $this->resourceConnection->getTableName(self::INDEX_TABLE);
    }
}
```

**Indexer interface requirements:**

- `ActionInterface::executeFull()`: Full reindex (all entities)
- `ActionInterface::executeList(array $ids)`: Partial reindex (specific IDs)
- `ActionInterface::executeRow($id)`: Reindex single entity
- `MviewActionInterface::execute($ids)`: Incremental reindex from changelog

### Step 5: Implement Data Builder

**Model/Indexer/ProductDataBuilder.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Indexer;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Catalog\Model\ResourceModel\Product\CollectionFactory;
use Magento\CatalogInventory\Api\StockRegistryInterface;
use Magento\Framework\Pricing\PriceCurrencyInterface;
use Magento\Store\Model\StoreManagerInterface;

/**
 * Build index data from source tables
 */
class ProductDataBuilder
{
    public function __construct(
        private readonly CollectionFactory $productCollectionFactory,
        private readonly StockRegistryInterface $stockRegistry,
        private readonly PriceCurrencyInterface $priceCurrency,
        private readonly StoreManagerInterface $storeManager
    ) {}

    /**
     * Build data for all products
     *
     * @return array
     */
    public function buildAllData(): array
    {
        $collection = $this->productCollectionFactory->create();
        $collection->addAttributeToSelect(['sku', 'name', 'price']);

        $data = [];

        foreach ($collection as $product) {
            $data[] = $this->buildProductData($product);
        }

        return $data;
    }

    /**
     * Build data for specific product IDs
     *
     * @param array $ids
     * @return array
     */
    public function buildData(array $ids): array
    {
        $collection = $this->productCollectionFactory->create();
        $collection->addAttributeToSelect(['sku', 'name', 'price']);
        $collection->addIdFilter($ids);

        $data = [];

        foreach ($collection as $product) {
            $data[] = $this->buildProductData($product);
        }

        return $data;
    }

    /**
     * Build index data for single product
     */
    private function buildProductData(ProductInterface $product): array
    {
        $stockItem = $this->stockRegistry->getStockItem($product->getId());

        return [
            'product_id' => $product->getId(),
            'sku' => $product->getSku(),
            'name' => $product->getName(),
            'price' => $product->getFinalPrice(),
            'qty' => $stockItem->getQty(),
            'is_in_stock' => $stockItem->getIsInStock() ? 1 : 0,
            'category_id' => $this->getPrimaryCategory($product),
            'updated_at' => date('Y-m-d H:i:s'),
        ];
    }

    /**
     * Get product's primary category ID
     */
    private function getPrimaryCategory(ProductInterface $product): ?int
    {
        $categoryIds = $product->getCategoryIds();

        return !empty($categoryIds) ? (int) $categoryIds[0] : null;
    }
}
```

**Data builder responsibilities:**
- Query source tables (EAV, stock, categories)
- Transform data into index table format
- Handle missing data gracefully
- Optimize queries (load collections, not individual products)

### Step 6: Run Indexer

```bash
# Reindex custom indexer
bin/magento indexer:reindex vendor_custom_product_data

# Check status
bin/magento indexer:status vendor_custom_product_data

# Set to scheduled mode
bin/magento indexer:set-mode schedule vendor_custom_product_data

# Reset indexer (clears version, forces full reindex on next run)
bin/magento indexer:reset vendor_custom_product_data
```

## Full Reindex vs Partial Reindex

### Full Reindex

**When it runs:**
- Manual: `bin/magento indexer:reindex`
- Indexer is invalid/reset: First cron run after reset
- Dependency triggers full reindex: When dependent indexer runs full reindex

**Process:**
1. Truncate index table
2. Rebuild from scratch (all entities)
3. Mark indexer as valid

**Duration:**
- Small catalogs (<10k products): 1-5 minutes
- Large catalogs (100k+ products): 10-60+ minutes

**Use cases:**
- After major data imports
- Index corruption detected
- Structural changes to index logic

### Partial Reindex

**When it runs:**
- Scheduled mode: Every cron run (processes changelog)
- Update on save: After entity save

**Process:**
1. Read changed entity IDs from changelog
2. Delete existing index records for those IDs
3. Rebuild index records for those IDs only
4. Update changelog version

**Duration:**
- Typically <1 second (processes 100-1000 entities)

**Use cases:**
- Normal operations (product saves, stock updates)
- Price changes
- Category assignments

### Performance Comparison

| Catalog Size | Full Reindex | Partial Reindex (100 products) |
|--------------|--------------|--------------------------------|
| 10,000 products | 2 minutes | <1 second |
| 50,000 products | 10 minutes | <1 second |
| 100,000 products | 30 minutes | 1-2 seconds |

**Key takeaway:** Partial reindex scales independently of catalog size (only reindexes changed entities).

## Index Optimization Strategies

### 1. Optimize Index Table Structure

**Use appropriate column types:**
```sql
-- GOOD: INT for IDs (4 bytes)
product_id INT UNSIGNED NOT NULL

-- BAD: VARCHAR for IDs (overhead)
product_id VARCHAR(255) NOT NULL
```

**Add covering indexes for common queries:**
```sql
-- Query: SELECT * FROM index WHERE category_id = 5 AND is_in_stock = 1
CREATE INDEX idx_category_stock ON vendor_custom_product_data_index(category_id, is_in_stock);
```

**Use composite indexes for multi-column filters:**
```sql
-- Covers: WHERE is_in_stock = 1 AND price BETWEEN 10 AND 50
CREATE INDEX idx_stock_price ON vendor_custom_product_data_index(is_in_stock, price);
```

### 2. Batch Processing

**Insert in batches (1000-5000 rows per batch):**
```php
private function insertBatch(array $data, int $batchSize = 1000): void
{
    $batches = array_chunk($data, $batchSize);

    foreach ($batches as $batch) {
        $this->getConnection()->insertMultiple($this->getIndexTable(), $batch);
    }
}
```

**Process changelog in chunks:**
```php
public function execute($ids): void
{
    // Process in chunks to avoid memory issues
    $chunks = array_chunk($ids, 1000);

    foreach ($chunks as $chunk) {
        $this->executeList($chunk);
    }
}
```

### 3. Optimize Queries

**Use direct SQL instead of collections:**
```php
// GOOD: Single query with joins
$select = $connection->select()
    ->from(['e' => 'catalog_product_entity'], ['entity_id', 'sku'])
    ->joinLeft(['v' => 'catalog_product_entity_varchar'], 'v.entity_id = e.entity_id AND v.attribute_id = :nameAttrId', ['value'])
    ->where('e.entity_id IN (?)', $ids);

$data = $connection->fetchAll($select);

// BAD: N+1 queries via collection iteration
foreach ($collection as $product) {
    $name = $product->getName(); // Loads attribute separately
}
```

**Use `INSERT ... ON DUPLICATE KEY UPDATE` for upserts:**
```php
$connection->insertOnDuplicate(
    $this->getIndexTable(),
    $data,
    ['price', 'qty', 'is_in_stock', 'updated_at'] // Update these columns on conflict
);
```

### 4. Parallel Indexing

For very large catalogs, process different stores/websites in parallel:

```php
// Split indexing by store
public function executeFull(): void
{
    $stores = $this->storeManager->getStores();

    foreach ($stores as $store) {
        $this->reindexStore($store->getId());
    }
}
```

### 5. Disable Indexes During Full Reindex

For MySQL, disable indexes during bulk inserts, then rebuild:

```php
public function executeFull(): void
{
    $connection = $this->getConnection();
    $table = $this->getIndexTable();

    // Disable keys for faster inserts
    $connection->disableTableKeys($table);

    // Insert data
    $this->insertBatch($connection, $data);

    // Re-enable keys (rebuilds indexes)
    $connection->enableTableKeys($table);
}
```

**Performance gain:** 50-70% faster bulk inserts on large datasets.

## Debugging Index Issues

### Common Issues and Solutions

#### 1. Indexer Stuck in "Processing" State

**Symptom:**
```bash
bin/magento indexer:status
# Shows: Processing
```

**Cause:** Indexer process killed or crashed mid-execution

**Solution:**
```bash
# Reset indexer state
bin/magento indexer:reset vendor_custom_product_data

# Reindex
bin/magento indexer:reindex vendor_custom_product_data
```

#### 2. Changelog Growing Indefinitely

**Symptom:**
```sql
SELECT COUNT(*) FROM vendor_custom_product_data_cl;
-- Returns millions of rows
```

**Cause:** Cron not running or indexer disabled

**Check cron:**
```bash
# View last cron execution
SELECT * FROM cron_schedule WHERE job_code = 'indexer_update_all_views' ORDER BY executed_at DESC LIMIT 10;
```

**Solution:**
- Ensure cron is running: `ps aux | grep cron`
- Check Magento cron schedule: `bin/magento cron:run` (or inspect the `cron_schedule` table directly; `cron:status` does not exist in core)
- Manually trigger indexer: `bin/magento indexer:reindex`

#### 3. Index Data Inconsistent with Source Data

**Symptom:** Frontend shows outdated prices or stock

**Cause:**
- Triggers not created (installation issue)
- Index table corrupted
- Partial reindex missing entities

**Check triggers:**
```sql
SHOW TRIGGERS LIKE 'catalog_product_entity';
```

**Solution:**
```bash
# Reset and full reindex
bin/magento indexer:reset vendor_custom_product_data
bin/magento indexer:reindex vendor_custom_product_data

# Recreate triggers if missing
bin/magento setup:upgrade
```

#### 4. Indexer Performance Degradation

**Symptom:** Reindex takes progressively longer over time

**Causes:**
- Changelog table fragmentation
- Missing indexes on source tables
- Inefficient queries

**Analyze changelog:**
```sql
ANALYZE TABLE vendor_custom_product_data_cl;
OPTIMIZE TABLE vendor_custom_product_data_cl;
```

**Add indexes to source tables:**
```sql
-- Check slow query log for missing indexes
SHOW INDEXES FROM catalog_product_entity;
```

**Profile indexer:**
```bash
# Enable query profiling
mysql> SET profiling = 1;

# Run reindex
bin/magento indexer:reindex vendor_custom_product_data

# View query times
mysql> SHOW PROFILES;
mysql> SHOW PROFILE FOR QUERY 5;
```

### Debugging Tools

**1. Indexer status table:**
```sql
SELECT * FROM indexer_state;
```

**Columns:**
- `state_id`: Indexer ID
- `status`: `valid`, `invalid`, `working`
- `updated`: Last reindex time

**2. Mview state table:**
```sql
SELECT * FROM mview_state;
```

**Columns:**
- `view_id`: Mview subscription ID
- `mode`: `enabled` or `disabled`
- `status`: `idle`, `working`, `suspended`
- `updated`: Last update time
- `version_id`: Last processed changelog version

**3. Changelog version tracking:**
```sql
-- Check last processed version
SELECT version_id FROM mview_state WHERE view_id = 'vendor_custom_product_data';

-- Check latest changelog version
SELECT MAX(version_id) FROM vendor_custom_product_data_cl;

-- Pending changelog entries
SELECT COUNT(*) FROM vendor_custom_product_data_cl WHERE version_id > (SELECT version_id FROM mview_state WHERE view_id = 'vendor_custom_product_data');
```

## Production Monitoring

### Key Metrics to Monitor

**1. Index validity:**
```bash
# Alert if any indexer is invalid for >5 minutes
bin/magento indexer:status | grep -i invalid
```

**2. Changelog size:**
```sql
-- Alert if changelog has >10,000 pending entries
SELECT
    cl.table_name,
    COUNT(*) as pending_entries
FROM information_schema.tables cl
WHERE cl.table_name LIKE '%_cl'
GROUP BY cl.table_name;
```

**3. Reindex duration:**
```sql
-- Track reindex duration via custom logging
SELECT
    indexer_id,
    AVG(duration_seconds) as avg_duration,
    MAX(duration_seconds) as max_duration
FROM custom_indexer_log
WHERE created_at > DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY indexer_id;
```

**4. Cron execution:**
```sql
-- Alert if indexer cron hasn't run in 5+ minutes
SELECT MAX(executed_at)
FROM cron_schedule
WHERE job_code = 'indexer_update_all_views'
  AND status = 'success';
```

### Index Maintenance Schedule

**Daily:**
- Monitor indexer status (all valid)
- Check changelog sizes (not growing indefinitely)
- Review slow query log for index-related queries

**Weekly:**
- Analyze/optimize index tables
- Review indexer performance metrics
- Check for orphaned changelog entries

**Monthly:**
- Full reindex during maintenance window
- Rebuild index table statistics
- Review and optimize index table structure

---

## Assumptions

- **Target versions**: Magento 2.4.7+, PHP 8.2+, MySQL 8.0+
- **Deployment**: Production environment with working cron jobs
- **Catalog size**: Examples assume medium to large catalogs (10k-100k products)
- **Infrastructure**: Dedicated database server with adequate resources

## Why This Approach

**Mview framework**: Standard Magento 2 approach, compatible with core indexers

**Changelog tables**: Decouple data changes from index updates, enable async processing

**Scheduled mode**: Production best practice, scales to large catalogs

**Batch processing**: Memory-efficient, handles large datasets without exhausting resources

**Direct SQL queries**: Faster than ORM for bulk operations

## Security Impact

**Authorization:** Indexer management requires CLI access or `Magento_Indexer::manage` ACL in admin

**Data exposure:** Index tables should not contain sensitive data (PII, payment info)

**SQL injection:** All queries use prepared statements and parameter binding

**Trigger security:** Triggers created by Magento setup, not user input

## Performance Impact

**Cacheability:**
- Index tables are read-heavy (cached in MySQL buffer pool)
- Index updates do not invalidate FPC (index data not directly exposed to frontend)
- Consider query cache for read-only replicas

**Database impact:**
- Triggers: <1ms overhead per write operation
- Changelog inserts: Minimal (2-column table, no foreign keys)
- Full reindex: High database load (run during maintenance windows)
- Partial reindex: Low database load (targeted updates)

**Indexer performance:**
- Partial reindex (1000 products): 1-2 seconds
- Full reindex (100k products): 30-60 minutes
- Optimized full reindex (parallel, batch): 10-20 minutes

**Core Web Vitals impact:**
- **Indirect improvement**: Faster product list queries improve LCP
- Direct impact: Reducing product list query time from 5s to 200ms improves TTFB and LCP

## Backward Compatibility

**API stability:**
- `ActionInterface` is stable across Magento 2.4.x
- `MviewActionInterface` is stable
- Indexer XML schema is stable

**Database schema:**
- Index tables use declarative schema (safe upgrades)
- Changelog tables created by Mview framework
- Triggers recreated on `setup:upgrade`

**Upgrade path:**
- Magento 2.4.7 → 2.4.8: Full reindex recommended after upgrade
- Magento 2.4.x → 2.4.9: Monitor for Mview framework changes

## Tests to Add

**Unit tests:**
- Indexer action methods (executeFull, executeList, executeRow)
- Data builder logic (buildData, buildAllData)
- Batch processing (insertBatch)

**Integration tests:**
- Full reindex creates correct data
- Partial reindex updates only changed entities
- Changelog triggers write correctly
- Mview processes changelog correctly
- Dependencies trigger reindex

**Functional tests (MFTF):**
- Admin reindex via System > Index Management
- Indexer mode change (realtime ↔ schedule)
- Index status display

**Performance tests:**
- Reindex duration for 10k, 50k, 100k products
- Partial reindex duration for 100, 1000 changed products
- Changelog processing rate (entities per second)

## Documentation to Update

**Developer documentation:**
- `README.md`: Custom indexer overview
- `ARCHITECTURE.md`: Indexer architecture diagram, Mview flow
- `PERFORMANCE.md`: Index optimization strategies
- `TROUBLESHOOTING.md`: Common index issues and solutions

**Operations documentation:**
- Index monitoring setup
- Cron job configuration
- Production reindex schedule
- Disaster recovery (index corruption)

**Code comments:**
- Inline documentation for complex SQL queries
- Indexer action class docblocks
- Data builder transformation logic

## Related Documentation

### Related Guides

- [EAV System Architecture: Understanding Entity-Attribute-Value in Magento 2](eav-system.md)
- [Cron Jobs Implementation: Building Reliable Scheduled Tasks in Magento 2](../how-to/cron-jobs.md)
- [Full Page Cache Strategy for High-Performance Magento](../how-to/full-page-cache-strategy.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
