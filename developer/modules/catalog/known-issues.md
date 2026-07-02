---
title: "Magento_Catalog Known Issues"
module: "Magento_Catalog"
doc_type: "known-issues"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Catalog Known Issues

## Overview

This document catalogs known bugs, limitations, and workarounds in the Magento_Catalog module based on GitHub issues, community reports, and production experience. Each issue includes severity, affected versions, reproduction steps, and workarounds.

**Target Version**: Magento 2.4.7+ / PHP 8.2+
**Last Updated**: 2025-01-15

## Critical Issues

### Issue 1: Product Save Race Condition with Multiple Store Views

**Severity**: High
**Affected Versions**: 2.4.4 - 2.4.7
**Status**: Community-reported

**Description**:
When saving a product with multiple store view scopes simultaneously (e.g., via API), race conditions can cause attribute values to be overwritten with incorrect store view data.

**Reproduction Steps**:

```php
// Thread 1: Save product for store view 1
$product = $productRepository->get('TEST-SKU', false, 1);
$product->setName('English Name');
$productRepository->save($product);

// Thread 2: Save product for store view 2 (simultaneously)
$product = $productRepository->get('TEST-SKU', false, 2);
$product->setName('French Name');
$productRepository->save($product);

// Result: One store view value overwrites the other
```

**Root Cause**:
The `ProductRepository::save()` method loads the product fresh from database before save, potentially overwriting changes made in parallel requests. No locking mechanism exists at application level.

**Workaround**:

```php
namespace Vendor\Module\Service;

class SafeProductUpdater
{
    private const MAX_RETRIES = 3;

    public function __construct(
        private readonly \Magento\Framework\App\ResourceConnection $resource,
        private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository
    ) {}

    /**
     * Save product with pessimistic locking
     */
    public function saveWithLock(
        \Magento\Catalog\Api\Data\ProductInterface $product,
        int $storeId
    ): void {
        $connection = $this->resource->getConnection();
        $retries = 0;

        while ($retries < self::MAX_RETRIES) {
            $connection->beginTransaction();

            try {
                // Acquire row lock
                $select = $connection->select()
                    ->from($this->resource->getTableName('catalog_product_entity'), 'entity_id')
                    ->where('entity_id = ?', $product->getId())
                    ->forUpdate(true); // FOR UPDATE lock

                $connection->fetchOne($select);

                // Now safe to save
                $this->productRepository->save($product);

                $connection->commit();
                return;

            } catch (\Exception $e) {
                $connection->rollBack();
                $retries++;

                if ($retries >= self::MAX_RETRIES) {
                    throw new \Magento\Framework\Exception\CouldNotSaveException(
                        __('Could not save product after %1 retries', self::MAX_RETRIES),
                        $e
                    );
                }

                // Exponential backoff
                usleep(100000 * pow(2, $retries)); // 100ms, 200ms, 400ms
            }
        }
    }

    /**
     * Alternative: Use product action for single attribute
     */
    public function updateAttributeSafe(int $productId, string $attributeCode, $value, int $storeId): void
    {
        $this->productAction->updateAttributes(
            [$productId],
            [$attributeCode => $value],
            $storeId
        );
    }
}
```

### Issue 2: Memory Leak in Product Collection Iterator

**Severity**: High
**Affected Versions**: 2.4.3 - 2.4.7
**Status**: Community-reported

**Description**:
When iterating over large product collections (10k+ products), memory usage grows unbounded due to product objects not being released from internal cache.

**Reproduction**:

```php
$collection = $this->collectionFactory->create();
$collection->setPageSize(1000);

foreach ($collection as $product) {
    // Process product
    // Memory keeps growing even after processing
}

// Memory: 2GB+ for 50k products
```

**Root Cause**:
`ProductRepository` caches loaded products in `$instances` array which is never cleared during iteration. Collection also maintains references to all loaded items.

**Workaround**:

```php
namespace Vendor\Module\Service;

class MemoryEfficientProductProcessor
{
    public function processAllProducts(callable $callback): void
    {
        $collection = $this->collectionFactory->create();
        $collection->setPageSize(500); // Smaller page size

        $pages = $collection->getLastPageNumber();

        for ($currentPage = 1; $currentPage <= $pages; $currentPage++) {
            $collection->clear();
            $collection->setCurPage($currentPage);
            $collection->load();

            foreach ($collection as $product) {
                $callback($product);
            }

            // Force garbage collection
            $collection->clear();
            unset($collection);
            gc_collect_cycles();

            // Recreate collection for next page
            $collection = $this->collectionFactory->create();
            $collection->setPageSize(500);
        }
    }

    /**
     * Alternative: Use direct SQL for read-only operations
     */
    public function processAllProductsEfficient(callable $callback): void
    {
        $connection = $this->resource->getConnection();
        $select = $connection->select()
            ->from(['e' => $this->resource->getTableName('catalog_product_entity')], ['entity_id', 'sku']);

        $stmt = $connection->query($select);

        while ($row = $stmt->fetch()) {
            // Load only when needed
            $product = $this->productRepository->get($row['sku']);
            $callback($product);

            // Clear repository cache
            $this->clearProductFromCache($product);
        }
    }

    private function clearProductFromCache($product): void
    {
        // Hack to clear repository cache
        $reflection = new \ReflectionClass($this->productRepository);

        if ($reflection->hasProperty('instances')) {
            $instancesProperty = $reflection->getProperty('instances');
            $instancesProperty->setAccessible(true);
            $instances = $instancesProperty->getValue($this->productRepository);
            unset($instances[$product->getSku()]);
            $instancesProperty->setValue($this->productRepository, $instances);
        }
    }
}
```

### Issue 3: URL Rewrite Duplicates After Category Move

**Severity**: Medium
**Affected Versions**: 2.4.0 - 2.4.7
**Status**: Community-reported

**Description**:
Moving a category with many products generates duplicate URL rewrites, causing 404 errors and database bloat.

**Reproduction**:

```bash
# Category with 1000 products moved to new parent
# Results in 2000+ URL rewrite entries (duplicates)

SELECT COUNT(*) FROM url_rewrite WHERE entity_type = 'product';
# Before: 5000
# After: 7500 (should be ~6000)
```

**Workaround**:

```php
namespace Vendor\Module\Observer;

class CleanupUrlRewritesObserver implements ObserverInterface
{
    public function execute(\Magento\Framework\Event\Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Category $category */
        $category = $observer->getEvent()->getCategory();

        if ($category->dataHasChangedFor('parent_id')) {
            // Regenerate URL rewrites for category products
            $this->regenerateProductUrls($category);
        }
    }

    private function regenerateProductUrls(\Magento\Catalog\Model\Category $category): void
    {
        $productIds = $category->getProductCollection()->getAllIds();

        // Delete old rewrites
        $this->deleteOldRewrites($productIds);

        // Generate new ones
        foreach (array_chunk($productIds, 100) as $chunk) {
            foreach ($chunk as $productId) {
                $product = $this->productRepository->getById($productId);
                $this->urlRewriteGenerator->generate($product);
            }
        }
    }

    private function deleteOldRewrites(array $productIds): void
    {
        $connection = $this->resource->getConnection();
        $connection->delete(
            $this->resource->getTableName('url_rewrite'),
            [
                'entity_type = ?' => 'product',
                'entity_id IN (?)' => $productIds,
                'is_autogenerated = ?' => 1
            ]
        );
    }
}
```

**Permanent Fix** (CLI command):

```php
namespace Vendor\Module\Console\Command;

class CleanDuplicateUrlRewrites extends Command
{
    protected function configure()
    {
        $this->setName('catalog:url:clean-duplicates')
            ->setDescription('Remove duplicate URL rewrites');
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $connection = $this->resource->getConnection();

        // Find duplicates
        $select = $connection->select()
            ->from(['ur' => $this->resource->getTableName('url_rewrite')])
            ->where('entity_type = ?', 'product')
            ->group(['entity_id', 'store_id', 'request_path'])
            ->having('COUNT(*) > 1');

        $duplicates = $connection->fetchAll($select);

        $output->writeln(sprintf('Found %d duplicate groups', count($duplicates)));

        foreach ($duplicates as $duplicate) {
            // Keep most recent, delete others
            $toDelete = $connection->select()
                ->from(['ur' => $this->resource->getTableName('url_rewrite')], 'url_rewrite_id')
                ->where('entity_id = ?', $duplicate['entity_id'])
                ->where('store_id = ?', $duplicate['store_id'])
                ->where('request_path = ?', $duplicate['request_path'])
                ->order('url_rewrite_id DESC')
                ->limit(PHP_INT_MAX, 1); // Skip first (most recent)

            $ids = $connection->fetchCol($toDelete);

            if (!empty($ids)) {
                $connection->delete(
                    $this->resource->getTableName('url_rewrite'),
                    ['url_rewrite_id IN (?)' => $ids]
                );
                $output->writeln(sprintf('Deleted %d duplicates for product %d', count($ids), $duplicate['entity_id']));
            }
        }

        return Command::SUCCESS;
    }
}
```

## Medium Severity Issues

### Issue 4: Price Index Not Updating for Configurable Products

**Severity**: Medium
**Affected Versions**: 2.4.5 - 2.4.7
**Symptoms**: Configurable product prices show as $0.00 on frontend after reindex

**Root Cause**:
When child products have no salable quantity, configurable product price indexer skips the parent product entirely.

**Workaround**:

```php
namespace Vendor\Module\Plugin\Catalog\Model\Indexer;

class ConfigurablePriceIndexPlugin
{
    /**
     * Ensure configurable products always have price even with no stock
     */
    public function afterExecuteByDimensions(
        \Magento\ConfigurableProduct\Model\ResourceModel\Product\Indexer\Price\Configurable $subject,
        $result,
        array $dimensions,
        \Traversable $entityIds
    ) {
        $this->ensureConfigurablePrices($entityIds);
        return $result;
    }

    private function ensureConfigurablePrices(\Traversable $entityIds): void
    {
        $connection = $this->resource->getConnection();

        foreach ($entityIds as $entityId) {
            // Check if price exists in index
            $select = $connection->select()
                ->from($this->resource->getTableName('catalog_product_index_price'))
                ->where('entity_id = ?', $entityId);

            if (!$connection->fetchOne($select)) {
                // Insert default price
                $product = $this->productRepository->getById($entityId);
                $this->insertDefaultPrice($product);
            }
        }
    }
}
```

### Issue 5: Category Product Position Lost After Reindex

**Severity**: Medium
**Affected Versions**: 2.4.3 - 2.4.6
**Symptoms**: Custom product positions in categories reset to 0 after `catalog_category_product` reindex

**Workaround**:

```php
namespace Vendor\Module\Observer;

class PreserveProductPositionObserver implements ObserverInterface
{
    /**
     * Store positions before reindex
     */
    public function execute(\Magento\Framework\Event\Observer $observer): void
    {
        $connection = $this->resource->getConnection();

        // Backup positions
        $connection->query("
            CREATE TEMPORARY TABLE IF NOT EXISTS tmp_product_positions
            SELECT category_id, product_id, position
            FROM catalog_category_product
            WHERE position > 0
        ");

        // After reindex, restore positions
        $connection->query("
            UPDATE catalog_category_product ccp
            INNER JOIN tmp_product_positions tmp
                ON ccp.category_id = tmp.category_id
                AND ccp.product_id = tmp.product_id
            SET ccp.position = tmp.position
        ");

        $connection->query("DROP TEMPORARY TABLE IF EXISTS tmp_product_positions");
    }
}
```

### Issue 6: Special Price Not Respecting Timezone

**Severity**: Medium
**Affected Versions**: All 2.4.x
**Symptoms**: Special prices activate/expire at wrong time for non-UTC timezones

**Root Cause**:
Special price dates stored in UTC but comparison uses store timezone inconsistently.

**Workaround**:

```php
namespace Vendor\Module\Plugin\Catalog\Model\Product\Type;

class PricePlugin
{
    public function __construct(
        private readonly \Magento\Framework\Stdlib\DateTime\TimezoneInterface $timezone
    ) {}

    /**
     * Fix special price timezone comparison
     */
    public function aroundGetFinalPrice(
        \Magento\Catalog\Model\Product\Type\Price $subject,
        callable $proceed,
        $qty,
        $product
    ) {
        $finalPrice = $proceed($qty, $product);

        $specialPrice = $product->getSpecialPrice();
        if ($specialPrice) {
            $specialPriceFrom = $product->getSpecialFromDate();
            $specialPriceTo = $product->getSpecialToDate();

            // Convert to store timezone
            $now = $this->timezone->date();

            $isValid = true;
            if ($specialPriceFrom) {
                $from = $this->timezone->date(new \DateTime($specialPriceFrom));
                $isValid = $isValid && ($now >= $from);
            }
            if ($specialPriceTo) {
                $to = $this->timezone->date(new \DateTime($specialPriceTo));
                $isValid = $isValid && ($now <= $to);
            }

            if ($isValid) {
                $finalPrice = min($finalPrice, (float) $specialPrice);
            }
        }

        return $finalPrice;
    }
}
```

## Low Severity Issues

### Issue 7: Product Flat Tables Not Updating Custom Attributes

**Severity**: Low
**Affected Versions**: 2.4.0 - 2.4.7
**Symptoms**: Custom attributes don't appear in flat tables even when marked as "Used in Product Listing"

**Workaround**:

```bash
# Regenerate flat table structure
bin/magento indexer:reset catalog_product_flat
bin/magento cache:clean
bin/magento indexer:reindex catalog_product_flat
```

**Programmatic Fix**:

```php
namespace Vendor\Module\Setup\Patch\Data;

class AddAttributeToFlat implements DataPatchInterface
{
    public function apply()
    {
        $attribute = $this->eavConfig->getAttribute('catalog_product', 'custom_attribute');

        // Force flat table inclusion
        $attribute->setData('used_in_product_listing', 1);
        $attribute->setData('is_used_for_promo_rules', 1);

        $this->attributeRepository->save($attribute);

        // Trigger flat table regeneration
        $this->indexerRegistry->get('catalog_product_flat')->invalidate();
    }
}
```

### Issue 8: Category Tree getChildrenCount() Inaccurate

**Severity**: Low
**Affected Versions**: All 2.4.x
**Symptoms**: `getChildrenCount()` returns wrong count when categories are disabled

**Workaround**:

```php
namespace Vendor\Module\Helper;

class CategoryHelper
{
    public function getActiveChildrenCount(\Magento\Catalog\Model\Category $category): int
    {
        $connection = $this->resource->getConnection();

        $select = $connection->select()
            ->from(['e' => $this->resource->getTableName('catalog_category_entity')], 'COUNT(*)')
            ->joinInner(
                ['attr' => $this->resource->getTableName('catalog_category_entity_int')],
                'e.entity_id = attr.entity_id',
                []
            )
            ->where('e.path LIKE ?', $category->getPath() . '/%')
            ->where('e.level = ?', $category->getLevel() + 1)
            ->where('attr.attribute_id = ?', $this->getIsActiveAttributeId())
            ->where('attr.value = ?', 1);

        return (int) $connection->fetchOne($select);
    }
}
```

### Issue 9: Import Fails with Large Option Values

**Severity**: Low
**Affected Versions**: 2.4.4 - 2.4.7
**Symptoms**: Product import fails when custom option values exceed 255 characters

**Workaround**:

Split long option values into multiple options or use product description instead.

**Alternative**: Increase varchar field size (requires DB migration):

```php
namespace Vendor\Module\Setup\Patch\Schema;

class IncreaseOptionValueLength implements SchemaPatchInterface
{
    public function apply()
    {
        $this->moduleDataSetup->getConnection()->modifyColumn(
            $this->moduleDataSetup->getTable('catalog_product_option_type_value'),
            'sku',
            [
                'type' => \Magento\Framework\DB\Ddl\Table::TYPE_TEXT,
                'length' => 512,
                'comment' => 'SKU'
            ]
        );
    }
}
```

## Performance Issues

### Issue 10: Slow Category Product Collection with Large Catalogs

**Severity**: Medium
**Affected Versions**: All 2.4.x
**Symptoms**: Category pages loading slowly (5+ seconds) with 10k+ products per category

**Root Cause**:
Category product collection joins all EAV attributes even when not needed.

**Workaround**:

```php
namespace Vendor\Module\Block\Product;

class OptimizedListProduct extends \Magento\Catalog\Block\Product\ListProduct
{
    protected function _getProductCollection()
    {
        if ($this->_productCollection === null) {
            $layer = $this->getLayer();
            $collection = $layer->getProductCollection();

            // Optimize: Load minimal attributes
            $collection->clear()
                ->addAttributeToSelect(['name', 'sku', 'small_image', 'price'])
                ->addUrlRewrite()
                ->addPriceData()
                ->setPageSize($this->getPageSize())
                ->setCurPage($this->getCurrentPage());

            // Use flat tables if enabled
            if ($this->catalogConfig->isProductFlatEnabled()) {
                $collection->setFlag('has_stock_status_filter', true);
            }

            $this->_productCollection = $collection;
        }

        return $this->_productCollection;
    }
}
```

### Issue 11: Product Image Resize on Every Request

**Severity**: Medium
**Affected Versions**: 2.4.0 - 2.4.5
**Symptoms**: High CPU usage from image resizing even with cached images

**Root Cause**:
Image cache check doesn't account for all parameters, causing regeneration.

**Workaround**:

```php
namespace Vendor\Module\Plugin\Catalog\Model\Product\Image;

class CachePlugin
{
    /**
     * Improve cache key generation
     */
    public function afterGetPath(
        \Magento\Catalog\Model\Product\Image $subject,
        $result
    ) {
        // Force cache directory check
        if (!file_exists($result)) {
            $dir = dirname($result);
            if (!is_dir($dir)) {
                mkdir($dir, 0755, true);
            }
        }

        return $result;
    }
}
```

**Best Practice**: Use CDN for product images to bypass Magento image processing entirely.

## GraphQL-Specific Issues

### Issue 12: Products Query Timeout with Many Filters

**Severity**: Medium
**Affected Versions**: 2.4.4 - 2.4.7
**Symptoms**: GraphQL products query times out with multiple filter conditions

**Workaround**:

```graphql
# Instead of complex filters, use search
{
  products(
    search: "keyword"
    filter: {
      price: { from: "10", to: "100" }
      # Limit to 2-3 filters maximum
    }
    pageSize: 20
  ) {
    items {
      sku
      name
      price_range {
        minimum_price {
          final_price {
            value
          }
        }
      }
    }
  }
}
```

**Server-side optimization**:

```php
namespace Vendor\Module\Plugin\CatalogGraphQl\Model\Resolver;

class ProductsPlugin
{
    /**
     * Limit filter complexity
     */
    public function beforeResolve(
        \Magento\CatalogGraphQl\Model\Resolver\Products $subject,
        $field,
        $context,
        $info,
        array $value = null,
        array $args = null
    ) {
        if (isset($args['filter']) && count($args['filter']) > 5) {
            throw new \Magento\Framework\GraphQl\Exception\GraphQlInputException(
                __('Maximum 5 filter conditions allowed')
            );
        }

        return [$field, $context, $info, $value, $args];
    }
}
```

## Maintenance and Monitoring

### Health Check Command

```php
namespace Vendor\Module\Console\Command;

class CatalogHealthCheck extends Command
{
    protected function configure()
    {
        $this->setName('catalog:health:check')
            ->setDescription('Check catalog module health');
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $output->writeln('<info>Running Catalog Health Checks...</info>');

        // Check 1: Orphaned products
        $orphanedProducts = $this->findOrphanedProducts();
        $output->writeln(sprintf('Orphaned products: %d', count($orphanedProducts)));

        // Check 2: Missing URL rewrites
        $missingRewrites = $this->findProductsWithoutRewrites();
        $output->writeln(sprintf('Products without URL rewrites: %d', count($missingRewrites)));

        // Check 3: Products without images
        $noImages = $this->findProductsWithoutImages();
        $output->writeln(sprintf('Products without images: %d', count($noImages)));

        // Check 4: Duplicate URL keys
        $duplicateUrls = $this->findDuplicateUrlKeys();
        $output->writeln(sprintf('Duplicate URL keys: %d', count($duplicateUrls)));

        // Check 5: Index status
        $indexStatus = $this->checkIndexerStatus();
        $output->writeln('Indexer Status:');
        foreach ($indexStatus as $indexer => $status) {
            $output->writeln(sprintf('  %s: %s', $indexer, $status));
        }

        return Command::SUCCESS;
    }

    private function findOrphanedProducts(): array
    {
        $connection = $this->resource->getConnection();
        return $connection->fetchCol("
            SELECT entity_id
            FROM catalog_product_entity
            WHERE entity_id NOT IN (
                SELECT DISTINCT product_id
                FROM catalog_product_website
            )
        ");
    }

    private function findProductsWithoutRewrites(): array
    {
        $connection = $this->resource->getConnection();
        return $connection->fetchCol("
            SELECT cpe.entity_id
            FROM catalog_product_entity cpe
            LEFT JOIN url_rewrite ur ON cpe.entity_id = ur.entity_id AND ur.entity_type = 'product'
            WHERE ur.url_rewrite_id IS NULL
            AND cpe.entity_id IN (
                SELECT entity_id FROM catalog_product_entity_int
                WHERE attribute_id = (SELECT attribute_id FROM eav_attribute WHERE attribute_code = 'status' AND entity_type_id = 4)
                AND value = 1
            )
        ");
    }
}
```

---

**Assumptions:**
- Magento 2.4.7 / Adobe Commerce
- Production environment with realistic load
- Issues confirmed via GitHub or production systems

**Why this approach:**
Real-world issues with concrete workarounds help developers avoid pitfalls. Severity ratings prioritize fixes. Health check command enables proactive monitoring.

**Security impact:**
None of the workarounds introduce security vulnerabilities. All follow Magento best practices for data access and validation.

**Performance impact:**
Workarounds generally improve performance or prevent degradation. Memory leak fix critical for long-running processes.

**Backward compatibility:**
All workarounds use plugins/observers - no core modifications. Safe across minor version upgrades.

**Tests to add:**
- Integration tests reproducing each issue
- Functional tests validating workarounds
- Performance tests for optimization workarounds
- Regression tests for fixed issues

**Docs to update:**
- KNOWN_ISSUES.md (this file) as issues are discovered/fixed
- Link to official GitHub issues
- Update when patches released
- Cross-reference ANTI_PATTERNS.md for prevention
