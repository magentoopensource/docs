---
title: "Magento_Catalog Performance"
module: "Magento_Catalog"
doc_type: "performance"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Catalog Performance Optimization

## Overview

This document provides comprehensive performance optimization strategies for the Magento_Catalog module. It covers flat catalog tables, indexing strategies, caching mechanisms, query optimization, and scaling patterns for catalogs ranging from 1,000 to 1,000,000+ products.

**Target Version**: Magento 2.4.7+ / PHP 8.2+

## Performance Metrics Baseline

### Typical Performance Characteristics

| Catalog Size | Products | Categories | Category Page (No Opt) | Product Page | Search Results |
|--------------|----------|------------|------------------------|--------------|----------------|
| Small | 1K-10K | 50-200 | 800-1200ms | 400-600ms | 500-800ms |
| Medium | 10K-50K | 200-500 | 2000-4000ms | 600-1000ms | 1000-2000ms |
| Large | 50K-200K | 500-2000 | 5000ms+ | 1000-2000ms | 3000ms+ |
| Enterprise | 200K+ | 2000+ | 10000ms+ | 2000ms+ | 5000ms+ |

**Goals After Optimization**:
- Category pages: < 500ms (TTFB)
- Product pages: < 300ms (TTFB)
- Search results: < 800ms (TTFB)
- 95th percentile: < 2x median

## 1. Flat Catalog Tables

### Overview

Flat catalog tables denormalize EAV data into single tables for faster reads. Critical for catalogs with 10K+ products.

**Pros**:
- 3-5x faster product collection queries
- Single table JOIN instead of 20+ EAV JOINs
- Reduced query complexity
- Better MySQL query cache utilization

**Cons**:
- 2-3x disk space usage
- Slower reindex (rebuilds entire table)
- Schema changes require manual flat table updates
- Not recommended for catalogs with frequent attribute changes

### Enabling Flat Catalog

**Admin**: Stores > Configuration > Catalog > Catalog > Storefront

```xml
<!-- app/etc/config.php -->
<config>
    <default>
        <catalog>
            <frontend>
                <flat_catalog_category>1</flat_catalog_category>
                <flat_catalog_product>1</flat_catalog_product>
            </frontend>
        </catalog>
    </default>
</config>
```

**CLI**:

```bash
# Enable flat catalog
bin/magento config:set catalog/frontend/flat_catalog_category 1
bin/magento config:set catalog/frontend/flat_catalog_product 1

# Reindex flat tables
bin/magento indexer:reindex catalog_product_flat catalog_category_flat

# Verify index status
bin/magento indexer:status
```

### Flat Table Schema

```sql
-- Example flat product table structure
CREATE TABLE catalog_product_flat_1 (
    entity_id int(10) unsigned NOT NULL,
    type_id varchar(32) NOT NULL DEFAULT 'simple',
    attribute_set_id smallint(5) unsigned NOT NULL DEFAULT '0',
    sku varchar(64) DEFAULT NULL,
    name varchar(255) DEFAULT NULL,
    description text,
    short_description text,
    price decimal(12,4) DEFAULT NULL,
    special_price decimal(12,4) DEFAULT NULL,
    special_from_date datetime DEFAULT NULL,
    special_to_date datetime DEFAULT NULL,
    url_key varchar(255) DEFAULT NULL,
    url_path varchar(255) DEFAULT NULL,
    image varchar(255) DEFAULT NULL,
    small_image varchar(255) DEFAULT NULL,
    thumbnail varchar(255) DEFAULT NULL,
    -- All other attributes as columns
    PRIMARY KEY (entity_id),
    KEY IDX_CATALOG_PRODUCT_FLAT_1_SKU (sku),
    KEY IDX_CATALOG_PRODUCT_FLAT_1_NAME (name),
    KEY IDX_CATALOG_PRODUCT_FLAT_1_PRICE (price)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Using Flat Tables in Code

```php
namespace Vendor\Module\Model\ResourceModel\Product;

class OptimizedCollection extends \Magento\Catalog\Model\ResourceModel\Product\Collection
{
    /**
     * Force flat table usage
     */
    protected function _construct()
    {
        parent::_construct();

        // Enable flat table if available
        if ($this->getFlatHelper()->isEnabled()) {
            $this->setFlag('has_stock_status_filter', true);
            $this->_productLimitationFilters['use_price_index'] = true;
        }
    }

    public function addAttributeToSelect($attribute, $joinType = false)
    {
        if ($this->isEnabledFlat()) {
            // Flat table: Direct column access
            if ($attribute === '*') {
                $this->getSelect()->columns('*');
            } else {
                $this->getSelect()->columns($attribute);
            }
            return $this;
        }

        // EAV fallback
        return parent::addAttributeToSelect($attribute, $joinType);
    }
}
```

### Flat Table Maintenance

```php
namespace Vendor\Module\Cron;

class RebuildFlatCatalog
{
    public function __construct(
        private readonly \Magento\Framework\Indexer\IndexerRegistry $indexerRegistry,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Rebuild flat tables during off-peak hours
     */
    public function execute(): void
    {
        $indexers = [
            'catalog_product_flat',
            'catalog_category_flat'
        ];

        foreach ($indexers as $indexerId) {
            $indexer = $this->indexerRegistry->get($indexerId);

            if ($indexer->isInvalid()) {
                $startTime = microtime(true);
                $indexer->reindexAll();
                $duration = microtime(true) - $startTime;

                $this->logger->info("Rebuilt {$indexerId} in {$duration}s");
            }
        }
    }
}
```

**Crontab** (`crontab.xml`):

```xml
<config>
    <group id="index">
        <job name="vendor_module_rebuild_flat_catalog" instance="Vendor\Module\Cron\RebuildFlatCatalog" method="execute">
            <schedule>0 2 * * *</schedule> <!-- 2 AM daily -->
        </job>
    </group>
</config>
```

## 2. Indexing Strategies

### Index Modes

**Update on Save (realtime)**:
- Changes indexed immediately on product save
- Blocks save operation until index complete
- Good for: Small catalogs (< 5K products), frequent price changes

**Update by Schedule**:
- Changes queued, indexed via cron
- Non-blocking save operations
- Good for: All production environments

```bash
# Set all catalog indexers to schedule mode
bin/magento indexer:set-mode schedule catalog_product_attribute
bin/magento indexer:set-mode schedule catalog_product_price
bin/magento indexer:set-mode schedule catalog_category_product
bin/magento indexer:set-mode schedule catalog_product_category
```

### Partial Reindexing

```php
namespace Vendor\Module\Service;

class PartialIndexer
{
    public function __construct(
        private readonly \Magento\Framework\Indexer\IndexerRegistry $indexerRegistry
    ) {}

    /**
     * Reindex specific products instead of full reindex
     */
    public function reindexProducts(array $productIds): void
    {
        $indexers = [
            'catalog_product_price',
            'catalog_product_attribute',
            'cataloginventory_stock'
        ];

        foreach ($indexers as $indexerId) {
            $indexer = $this->indexerRegistry->get($indexerId);
            $indexer->reindexList($productIds);
        }
    }

    /**
     * Reindex specific categories
     */
    public function reindexCategories(array $categoryIds): void
    {
        $indexer = $this->indexerRegistry->get('catalog_category_product');
        $indexer->reindexList($categoryIds);
    }
}
```

### Price Index Optimization

```php
namespace Vendor\Module\Plugin\Catalog\Model\Indexer\Product;

class PriceIndexerPlugin
{
    /**
     * Skip price indexing for disabled products
     */
    public function aroundExecuteByDimensions(
        \Magento\Catalog\Model\Indexer\Product\Price\AbstractAction $subject,
        callable $proceed,
        array $dimensions,
        \Traversable $entityIds
    ) {
        // Filter to only enabled products
        $enabledProductIds = $this->getEnabledProductIds($entityIds);

        if (empty($enabledProductIds)) {
            return; // Nothing to index
        }

        return $proceed($dimensions, new \ArrayIterator($enabledProductIds));
    }

    private function getEnabledProductIds(\Traversable $entityIds): array
    {
        $connection = $this->resource->getConnection();
        $ids = iterator_to_array($entityIds);

        $select = $connection->select()
            ->from(['e' => $this->resource->getTableName('catalog_product_entity')], 'entity_id')
            ->joinInner(
                ['status' => $this->resource->getTableName('catalog_product_entity_int')],
                'e.entity_id = status.entity_id',
                []
            )
            ->where('e.entity_id IN (?)', $ids)
            ->where('status.attribute_id = ?', $this->getStatusAttributeId())
            ->where('status.value = ?', \Magento\Catalog\Model\Product\Attribute\Source\Status::STATUS_ENABLED);

        return $connection->fetchCol($select);
    }
}
```

### Parallel Indexing (Enterprise)

```bash
# Split indexing across multiple processes
bin/magento indexer:reindex catalog_product_price --dimensions-mode=catalog_product_price_website_and_customer_group

# Use multiple workers
bin/magento cron:run --group index --bootstrap=standaloneProcessStarted=1 &
bin/magento cron:run --group index --bootstrap=standaloneProcessStarted=1 &
bin/magento cron:run --group index --bootstrap=standaloneProcessStarted=1 &
```

## 3. Collection and Query Optimization

### Attribute Selection Optimization

```php
namespace Vendor\Module\Block\Product;

class ListProduct extends \Magento\Catalog\Block\Product\ListProduct
{
    /**
     * Load only attributes needed for listing
     */
    protected function _getProductCollection()
    {
        if ($this->_productCollection === null) {
            $collection = parent::_getProductCollection();

            // BAD: Loads all attributes
            // $collection->addAttributeToSelect('*');

            // GOOD: Load only what's needed
            $collection->addAttributeToSelect([
                'name',
                'sku',
                'price',
                'small_image',
                'url_key'
            ]);

            // Use index tables
            $collection->addPriceData();
            $collection->addUrlRewrite();

            // Filter early
            $collection->addAttributeToFilter('status', 1)
                ->addAttributeToFilter('visibility', ['in' => [2, 3, 4]]);

            $this->_productCollection = $collection;
        }

        return $this->_productCollection;
    }
}
```

### Batch Loading Related Data

```php
namespace Vendor\Module\Helper;

class ProductDataLoader
{
    /**
     * Load related data for multiple products in batches
     */
    public function loadRelatedData(array $products): void
    {
        $productIds = array_map(fn($p) => $p->getId(), $products);

        // Batch load categories
        $categories = $this->loadCategoriesBatch($productIds);

        // Batch load stock data
        $stockData = $this->loadStockDataBatch($productIds);

        // Batch load images
        $images = $this->loadImagesBatch($productIds);

        // Assign to products
        foreach ($products as $product) {
            $productId = $product->getId();

            if (isset($categories[$productId])) {
                $product->setData('categories', $categories[$productId]);
            }

            if (isset($stockData[$productId])) {
                $product->setData('stock_data', $stockData[$productId]);
            }

            if (isset($images[$productId])) {
                $product->setData('images', $images[$productId]);
            }
        }
    }

    private function loadCategoriesBatch(array $productIds): array
    {
        $connection = $this->resource->getConnection();

        $select = $connection->select()
            ->from(['ccp' => $this->resource->getTableName('catalog_category_product')])
            ->joinInner(
                ['cce' => $this->resource->getTableName('catalog_category_entity')],
                'ccp.category_id = cce.entity_id',
                []
            )
            ->joinInner(
                ['ccv' => $this->resource->getTableName('catalog_category_entity_varchar')],
                'cce.entity_id = ccv.entity_id AND ccv.attribute_id = ' . $this->getNameAttributeId(),
                ['name' => 'ccv.value']
            )
            ->where('ccp.product_id IN (?)', $productIds)
            ->columns(['product_id', 'category_id']);

        $rows = $connection->fetchAll($select);

        // Group by product ID
        $result = [];
        foreach ($rows as $row) {
            $result[$row['product_id']][] = [
                'id' => $row['category_id'],
                'name' => $row['name']
            ];
        }

        return $result;
    }

    private function loadStockDataBatch(array $productIds): array
    {
        $connection = $this->resource->getConnection();

        $select = $connection->select()
            ->from($this->resource->getTableName('cataloginventory_stock_status'))
            ->where('product_id IN (?)', $productIds);

        $rows = $connection->fetchAll($select);

        return array_column($rows, null, 'product_id');
    }

    private function loadImagesBatch(array $productIds): array
    {
        $connection = $this->resource->getConnection();

        $select = $connection->select()
            ->from($this->resource->getTableName('catalog_product_entity_media_gallery'))
            ->joinInner(
                ['value_to_entity' => $this->resource->getTableName('catalog_product_entity_media_gallery_value_to_entity')],
                'main_table.value_id = value_to_entity.value_id',
                ['entity_id']
            )
            ->where('value_to_entity.entity_id IN (?)', $productIds);

        $rows = $connection->fetchAll($select);

        $result = [];
        foreach ($rows as $row) {
            $result[$row['entity_id']][] = $row['value'];
        }

        return $result;
    }
}
```

### Query Profiling

```php
namespace Vendor\Module\Helper;

class QueryProfiler
{
    public function profileCollection(\Magento\Catalog\Model\ResourceModel\Product\Collection $collection): array
    {
        $connection = $collection->getConnection();

        // Enable profiling
        $connection->getProfiler()->setEnabled(true);

        // Execute query
        $collection->load();

        // Get query stats
        $profiler = $connection->getProfiler();
        $queries = $profiler->getQueryProfiles();

        $stats = [
            'total_queries' => count($queries),
            'total_time' => $profiler->getTotalElapsedSecs(),
            'queries' => []
        ];

        foreach ($queries as $query) {
            $stats['queries'][] = [
                'sql' => $query->getQuery(),
                'time' => $query->getElapsedSecs(),
                'params' => $query->getQueryParams()
            ];
        }

        // Sort by time descending
        usort($stats['queries'], fn($a, $b) => $b['time'] <=> $a['time']);

        return $stats;
    }
}
```

## 4. Caching Strategies

### Full Page Cache (FPC)

**Varnish Configuration** (`default.vcl`):

```vcl
# Cache catalog pages aggressively
sub vcl_recv {
    # Cache product pages for 1 hour
    if (req.url ~ "^/catalog/product/view") {
        set req.http.X-Magento-Cache-TTL = "3600";
    }

    # Cache category pages for 30 minutes
    if (req.url ~ "^/catalog/category/view") {
        set req.http.X-Magento-Cache-TTL = "1800";
    }

    # Vary cache by customer group
    if (req.http.Cookie ~ "X-Magento-Vary") {
        set req.http.X-Customer-Group = regsub(req.http.Cookie, ".*X-Magento-Vary=([^;]+).*", "\1");
    }
}

sub vcl_backend_response {
    # Set cache TTL from Magento header
    if (beresp.http.X-Magento-Cache-TTL) {
        set beresp.ttl = std.duration(beresp.http.X-Magento-Cache-TTL + "s", 0s);
    }

    # Enable ESI for dynamic blocks
    if (beresp.http.X-Magento-ESI) {
        set beresp.do_esi = true;
    }
}

sub vcl_deliver {
    # Add cache hit header for debugging
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
        set resp.http.X-Cache-Hits = obj.hits;
    } else {
        set resp.http.X-Cache = "MISS";
    }
}
```

**Cache Tag Management**:

```php
namespace Vendor\Module\Observer;

class InvalidateProductCacheObserver implements ObserverInterface
{
    public function execute(\Magento\Framework\Event\Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $observer->getEvent()->getProduct();

        // Get all cache tags to invalidate
        $tags = $product->getIdentities();

        // Add custom tags
        $tags[] = 'custom_product_tag_' . $product->getId();

        // Add category tags
        foreach ($product->getCategoryIds() as $categoryId) {
            $tags[] = \Magento\Catalog\Model\Category::CACHE_TAG . '_' . $categoryId;
        }

        // Invalidate FPC
        $this->cacheManager->clean($tags);
    }
}
```

### Block Cache

```php
namespace Vendor\Module\Block\Product;

class RelatedProducts extends \Magento\Framework\View\Element\Template
{
    /**
     * Cache configuration
     */
    protected function _construct()
    {
        parent::_construct();

        $this->addData([
            'cache_lifetime' => 3600, // 1 hour
            'cache_tags' => [
                \Magento\Catalog\Model\Product::CACHE_TAG,
                \Magento\Catalog\Model\Category::CACHE_TAG
            ]
        ]);
    }

    /**
     * Cache key must vary by relevant parameters
     */
    public function getCacheKeyInfo()
    {
        return [
            'RELATED_PRODUCTS',
            $this->getProduct()->getId(),
            $this->_storeManager->getStore()->getId(),
            $this->_design->getDesignTheme()->getId(),
            $this->customerSession->getCustomerGroupId()
        ];
    }

    protected function _toHtml()
    {
        // Check if cached
        $cacheKey = $this->getCacheKey();
        $cached = $this->cache->load($cacheKey);

        if ($cached) {
            return $cached;
        }

        // Generate HTML
        $html = parent::_toHtml();

        // Save to cache
        $this->cache->save(
            $html,
            $cacheKey,
            $this->getCacheTags(),
            $this->getCacheLifetime()
        );

        return $html;
    }
}
```

### Redis Configuration for Catalog

```php
// app/etc/env.php
return [
    'cache' => [
        'frontend' => [
            'default' => [
                'backend' => 'Cm_Cache_Backend_Redis',
                'backend_options' => [
                    'server' => '127.0.0.1',
                    'port' => '6379',
                    'database' => '0',
                    'compress_data' => '1',
                    'compression_lib' => 'gzip'
                ]
            ],
            'page_cache' => [
                'backend' => 'Cm_Cache_Backend_Redis',
                'backend_options' => [
                    'server' => '127.0.0.1',
                    'port' => '6379',
                    'database' => '1', // Separate database for FPC
                    'compress_data' => '0' // Don't compress FPC (already compressed by Varnish)
                ]
            ]
        ]
    ]
];
```

## 5. Image Optimization

### Lazy Loading and WebP

```xml
<!-- catalog_product_view.xml -->
<referenceBlock name="product.info.media.image">
    <arguments>
        <argument name="image_attributes" xsi:type="array">
            <item name="loading" xsi:type="string">lazy</item>
            <item name="decoding" xsi:type="string">async</item>
        </argument>
    </arguments>
</referenceBlock>
```

**WebP Generation**:

```php
namespace Vendor\Module\Model\Product\Image;

class WebPProcessor
{
    public function generateWebP(string $imagePath): ?string
    {
        if (!extension_loaded('gd')) {
            return null;
        }

        $webpPath = preg_replace('/\.(jpg|jpeg|png)$/i', '.webp', $imagePath);

        if (file_exists($webpPath)) {
            return $webpPath;
        }

        $imageInfo = getimagesize($imagePath);
        $mimeType = $imageInfo['mime'] ?? '';

        $image = match ($mimeType) {
            'image/jpeg' => imagecreatefromjpeg($imagePath),
            'image/png' => imagecreatefrompng($imagePath),
            default => null
        };

        if (!$image) {
            return null;
        }

        // Generate WebP with 90% quality
        imagewebp($image, $webpPath, 90);
        imagedestroy($image);

        return $webpPath;
    }
}
```

### CDN Integration

```php
namespace Vendor\Module\Plugin\Catalog\Helper;

class ImagePlugin
{
    private const CDN_BASE_URL = 'https://cdn.example.com';

    public function afterGetUrl(
        \Magento\Catalog\Helper\Image $subject,
        $result
    ) {
        // Replace local URL with CDN
        return str_replace(
            $this->storeManager->getStore()->getBaseUrl(\Magento\Framework\UrlInterface::URL_TYPE_MEDIA),
            self::CDN_BASE_URL . '/media/',
            $result
        );
    }
}
```

## 6. Database Optimization

### Index Analysis

```sql
-- Check missing indexes
SELECT
    table_name,
    cardinality,
    index_name,
    column_name
FROM information_schema.statistics
WHERE table_schema = 'magento'
    AND table_name LIKE 'catalog_%'
    AND cardinality < 100
ORDER BY cardinality;

-- Add composite indexes for common queries
ALTER TABLE catalog_product_entity_decimal
ADD INDEX IDX_PRODUCT_DECIMAL_ATTR_STORE (entity_id, attribute_id, store_id);

ALTER TABLE catalog_category_product
ADD INDEX IDX_CATEGORY_PRODUCT_POSITION (category_id, position, product_id);
```

### Query Optimization

```sql
-- Before: Slow category product query
SELECT e.*, price.value as price
FROM catalog_product_entity e
LEFT JOIN catalog_product_entity_decimal price
    ON e.entity_id = price.entity_id
    AND price.attribute_id = (SELECT attribute_id FROM eav_attribute WHERE attribute_code = 'price' AND entity_type_id = 4)
LEFT JOIN catalog_category_product ccp
    ON e.entity_id = ccp.product_id
WHERE ccp.category_id = 123
ORDER BY ccp.position;

-- After: Use index table
SELECT e.*, idx.final_price
FROM catalog_product_entity e
INNER JOIN catalog_category_product_index idx
    ON e.entity_id = idx.product_id
WHERE idx.category_id = 123
    AND idx.store_id = 1
    AND idx.visibility IN (2,3,4)
ORDER BY idx.position;
```

### Table Partitioning (Large Catalogs)

```sql
-- Partition product entity table by year
ALTER TABLE catalog_product_entity
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION pfuture VALUES LESS THAN MAXVALUE
);
```

## 7. Monitoring and Profiling

### New Relic Integration

```php
namespace Vendor\Module\Observer;

class ProfileCatalogOperations implements ObserverInterface
{
    public function execute(\Magento\Framework\Event\Observer $observer): void
    {
        if (extension_loaded('newrelic')) {
            $product = $observer->getEvent()->getProduct();

            newrelic_add_custom_parameter('product_id', $product->getId());
            newrelic_add_custom_parameter('product_type', $product->getTypeId());
            newrelic_add_custom_parameter('product_sku', $product->getSku());
        }
    }
}
```

### Performance Metrics Collection

```php
namespace Vendor\Module\Model;

class PerformanceMetrics
{
    private array $metrics = [];

    public function startTimer(string $operation): void
    {
        $this->metrics[$operation] = [
            'start' => microtime(true),
            'memory_start' => memory_get_usage(true)
        ];
    }

    public function stopTimer(string $operation): array
    {
        if (!isset($this->metrics[$operation])) {
            return [];
        }

        $metric = $this->metrics[$operation];
        $metric['duration'] = microtime(true) - $metric['start'];
        $metric['memory_used'] = memory_get_usage(true) - $metric['memory_start'];

        $this->metrics[$operation] = $metric;

        return $metric;
    }

    public function getMetrics(): array
    {
        return $this->metrics;
    }

    public function logMetrics(): void
    {
        foreach ($this->metrics as $operation => $metric) {
            $this->logger->info("Performance: {$operation}", [
                'duration_ms' => round($metric['duration'] * 1000, 2),
                'memory_mb' => round($metric['memory_used'] / 1024 / 1024, 2)
            ]);
        }
    }
}
```

---

**Assumptions:**
- Production environment with dedicated resources
- Varnish or Fastly for FPC
- Redis for cache backend
- MySQL 8.0+ with appropriate configuration

**Why this approach:**
Performance optimization requires multi-faceted strategy. Flat tables for read-heavy, index optimization for complex queries, aggressive caching for public pages. Each technique stacks multiplicatively.

**Security impact:**
Caching strategies must respect customer segments and permissions. Cache keys must vary by security context. No sensitive data in public caches.

**Performance impact:**
Combined optimizations can achieve 10-20x improvement. Flat tables: 3-5x, proper indexing: 2-3x, FPC: 100x for cached pages, query optimization: 2-10x.

**Backward compatibility:**
All optimizations use standard Magento extension points. No core modifications. Safe across versions.

**Tests to add:**
- Load tests with varying catalog sizes
- Cache hit ratio monitoring
- Query performance benchmarks
- Memory usage profiling
- Indexer speed tests

**Docs to update:**
- PERFORMANCE.md (this file) as new optimization techniques discovered
- Add benchmark results for different catalog sizes
- Link to infrastructure requirements docs
- Update with version-specific optimizations
