---
title: "Magento_Catalog Anti-Patterns"
module: "Magento_Catalog"
doc_type: "anti-patterns"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Catalog Anti-Patterns

## Overview

This document catalogs common mistakes, anti-patterns, and pitfalls when working with the Magento_Catalog module. Each anti-pattern includes the problem, why it's harmful, and the correct approach with code examples.

**Target Version**: Magento 2.4.7+ / PHP 8.2+

## Category 1: Data Access Anti-Patterns

### Anti-Pattern 1.1: Direct Model Loading Instead of Repository

**Problem Code**:

```php
namespace Vendor\Module\Model;

class BadProductLoader
{
    public function __construct(
        private readonly \Magento\Catalog\Model\ProductFactory $productFactory
    ) {}

    public function loadProduct(int $productId)
    {
        // ANTI-PATTERN: Direct model loading
        $product = $this->productFactory->create();
        $product->load($productId);
        return $product;
    }
}
```

**Why This Is Wrong**:

1. **No caching**: Models don't implement repository cache layer
2. **No validation**: Missing business logic validation in repository
3. **BC risk**: Direct model usage may break across versions
4. **No events**: Repository dispatches additional events for plugins
5. **Type safety**: Models don't enforce service contract interfaces

**Impact**: 3-5x slower due to missing cache; potential upgrade breaks

**Correct Approach**:

```php
namespace Vendor\Module\Model;

class GoodProductLoader
{
    public function __construct(
        private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository
    ) {}

    public function loadProduct(int $productId): \Magento\Catalog\Api\Data\ProductInterface
    {
        try {
            return $this->productRepository->getById($productId);
        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            throw new \InvalidArgumentException(
                sprintf('Product with ID %d does not exist', $productId)
            );
        }
    }

    public function loadProductBySku(string $sku): \Magento\Catalog\Api\Data\ProductInterface
    {
        return $this->productRepository->get($sku);
    }
}
```

### Anti-Pattern 1.2: Direct SQL Queries Instead of Collections

**Problem Code**:

```php
namespace Vendor\Module\Model;

class BadProductQuery
{
    public function __construct(
        private readonly \Magento\Framework\App\ResourceConnection $resource
    ) {}

    public function getExpensiveProducts(): array
    {
        // ANTI-PATTERN: Direct SQL bypasses EAV abstraction
        $connection = $this->resource->getConnection();
        $select = $connection->select()
            ->from('catalog_product_entity', ['entity_id', 'sku'])
            ->where('price > ?', 100);

        return $connection->fetchAll($select);
    }
}
```

**Why This Is Wrong**:

1. **EAV ignorance**: Price is in `catalog_product_entity_decimal`, not main table
2. **No scope handling**: Ignores store view, website scoping
3. **Index bypass**: Doesn't use price index table (slower)
4. **Breaks abstraction**: Hard to maintain when schema changes
5. **No attribute filtering**: Can't easily add filters on other attributes

**Correct Approach**:

```php
namespace Vendor\Module\Model;

class GoodProductQuery
{
    public function __construct(
        private readonly \Magento\Catalog\Model\ResourceModel\Product\CollectionFactory $collectionFactory
    ) {}

    public function getExpensiveProducts(float $minPrice = 100): array
    {
        $collection = $this->collectionFactory->create();

        $collection->addAttributeToSelect(['sku', 'name', 'price'])
            ->addAttributeToFilter('price', ['gteq' => $minPrice])
            ->addAttributeToFilter('status', \Magento\Catalog\Model\Product\Attribute\Source\Status::STATUS_ENABLED)
            ->setOrder('price', 'DESC')
            ->setPageSize(50);

        return $collection->getItems();
    }

    /**
     * Use SearchCriteria for API-level queries
     */
    public function getExpensiveProductsViaApi(float $minPrice = 100): array
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('price', $minPrice, 'gteq')
            ->addFilter('status', \Magento\Catalog\Model\Product\Attribute\Source\Status::STATUS_ENABLED)
            ->addSortOrder($this->sortOrderBuilder->setField('price')->setDirection('DESC')->create())
            ->setPageSize(50)
            ->create();

        return $this->productRepository->getList($searchCriteria)->getItems();
    }
}
```

### Anti-Pattern 1.3: N+1 Query Problem in Loops

**Problem Code**:

```php
namespace Vendor\Module\Block;

class BadProductList extends \Magento\Framework\View\Element\Template
{
    public function getProductPrices(): array
    {
        $productIds = [1, 2, 3, 4, 5]; // 100+ IDs in reality
        $prices = [];

        foreach ($productIds as $productId) {
            // ANTI-PATTERN: Loading product in loop = N+1 queries
            $product = $this->productRepository->getById($productId);
            $prices[$productId] = $product->getFinalPrice();
        }

        return $prices;
    }
}
```

**Why This Is Wrong**:

1. **Performance disaster**: 100 products = 100+ database queries
2. **Memory bloat**: Each product fully loaded with all attributes
3. **Slow page load**: Can add 1-2 seconds to page rendering
4. **Server strain**: High CPU and DB load under traffic

**Impact**: Page load time increases linearly with product count

**Correct Approach**:

```php
namespace Vendor\Module\Block;

class GoodProductList extends \Magento\Framework\View\Element\Template
{
    public function __construct(
        \Magento\Framework\View\Element\Template\Context $context,
        private readonly \Magento\Catalog\Model\ResourceModel\Product\CollectionFactory $collectionFactory,
        array $data = []
    ) {
        parent::__construct($context, $data);
    }

    public function getProductPrices(array $productIds): array
    {
        // Load all products in single query
        $collection = $this->collectionFactory->create();
        $collection->addIdFilter($productIds)
            ->addAttributeToSelect('price')
            ->addPriceData(); // Join price index table

        $prices = [];
        foreach ($collection as $product) {
            $prices[$product->getId()] = $product->getFinalPrice();
        }

        return $prices;
    }

    /**
     * Even better: Use price index directly
     */
    public function getProductPricesFromIndex(array $productIds): array
    {
        $connection = $this->resource->getConnection();
        $select = $connection->select()
            ->from($this->resource->getTableName('catalog_product_index_price'), ['entity_id', 'final_price'])
            ->where('entity_id IN (?)', $productIds)
            ->where('customer_group_id = ?', $this->customerSession->getCustomerGroupId())
            ->where('website_id = ?', $this->storeManager->getStore()->getWebsiteId());

        return $connection->fetchPairs($select);
    }
}
```

## Category 2: Save Operation Anti-Patterns

### Anti-Pattern 2.1: Saving Products in Loops Without Bulk API

**Problem Code**:

```php
namespace Vendor\Module\Model;

class BadBulkUpdater
{
    public function updatePrices(array $priceUpdates): void
    {
        foreach ($priceUpdates as $sku => $newPrice) {
            // ANTI-PATTERN: Individual saves in loop
            $product = $this->productRepository->get($sku);
            $product->setPrice($newPrice);
            $this->productRepository->save($product); // Triggers full save flow each time
        }
    }
}
```

**Why This Is Wrong**:

1. **Extremely slow**: Each save triggers indexing, cache clear, events
2. **Transaction overhead**: Each save is separate DB transaction
3. **Lock contention**: Multiple saves can cause table locks
4. **Memory leaks**: Products accumulate in memory
5. **Index thrashing**: Reindexes same data repeatedly

**Impact**: 1000 products can take 30+ minutes

**Correct Approach**:

```php
namespace Vendor\Module\Model;

class GoodBulkUpdater
{
    public function __construct(
        private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
        private readonly \Magento\Catalog\Model\Product\Action $productAction,
        private readonly \Magento\Catalog\Model\Indexer\Product\Price\Processor $priceIndexer
    ) {}

    /**
     * Method 1: Use mass action (fastest for simple attributes)
     */
    public function updatePricesBulk(array $priceUpdates): void
    {
        $productIds = [];
        $skuToId = [];

        // Get product IDs
        foreach (array_keys($priceUpdates) as $sku) {
            $product = $this->productRepository->get($sku);
            $productIds[] = $product->getId();
            $skuToId[$sku] = $product->getId();
        }

        // Single mass update query
        foreach ($priceUpdates as $sku => $newPrice) {
            $this->productAction->updateAttributes(
                [$skuToId[$sku]],
                ['price' => $newPrice],
                0 // Global scope
            );
        }

        // Reindex once at end
        $this->priceIndexer->reindexList($productIds);
    }

    /**
     * Method 2: Batch processing with memory management
     */
    public function updatePricesBatched(array $priceUpdates, int $batchSize = 100): void
    {
        $batches = array_chunk($priceUpdates, $batchSize, true);

        foreach ($batches as $batch) {
            $productIds = [];

            foreach ($batch as $sku => $newPrice) {
                $product = $this->productRepository->get($sku);
                $product->setPrice($newPrice);
                $this->productRepository->save($product);
                $productIds[] = $product->getId();
            }

            // Reindex batch
            $this->priceIndexer->reindexList($productIds);

            // Free memory
            gc_collect_cycles();
        }
    }

    /**
     * Method 3: Direct SQL for maximum performance (use with caution)
     */
    public function updatePricesDirect(array $priceUpdates): void
    {
        $connection = $this->resource->getConnection();
        $priceAttributeId = $this->eavConfig->getAttribute('catalog_product', 'price')->getId();

        $connection->beginTransaction();

        try {
            foreach ($priceUpdates as $sku => $newPrice) {
                $productId = $this->resource->getConnection()->fetchOne(
                    $connection->select()
                        ->from($this->resource->getTableName('catalog_product_entity'), 'entity_id')
                        ->where('sku = ?', $sku)
                );

                if ($productId) {
                    $connection->insertOnDuplicate(
                        $this->resource->getTableName('catalog_product_entity_decimal'),
                        [
                            'entity_id' => $productId,
                            'attribute_id' => $priceAttributeId,
                            'store_id' => 0,
                            'value' => $newPrice
                        ],
                        ['value']
                    );
                }
            }

            $connection->commit();

            // Reindex all at once
            $this->priceIndexer->reindexAll();
        } catch (\Exception $e) {
            $connection->rollBack();
            throw $e;
        }
    }
}
```

### Anti-Pattern 2.2: Not Using Transactions for Multi-Step Operations

**Problem Code**:

```php
namespace Vendor\Module\Model;

class BadProductCreator
{
    public function createProductWithRelations(array $data): void
    {
        // ANTI-PATTERN: No transaction wrapping multiple operations
        $product = $this->productFactory->create();
        $product->setData($data);
        $this->productRepository->save($product); // Can fail here

        // If this fails, product exists but has no images (inconsistent state)
        $this->mediaGallery->addImages($product, $data['images']);

        // If this fails, product and images exist but no stock (inconsistent)
        $this->stockRegistry->updateStockItemBySku($product->getSku(), $data['stock']);
    }
}
```

**Why This Is Wrong**:

1. **Data inconsistency**: Partial failures leave data in invalid state
2. **No rollback**: Can't undo partial operations
3. **Hard to debug**: Unclear which step failed
4. **Cleanup required**: Manual intervention to fix partial saves

**Correct Approach**:

```php
namespace Vendor\Module\Model;

class GoodProductCreator
{
    public function __construct(
        private readonly \Magento\Framework\App\ResourceConnection $resource,
        private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    public function createProductWithRelations(array $data): \Magento\Catalog\Api\Data\ProductInterface
    {
        $connection = $this->resource->getConnection();
        $connection->beginTransaction();

        try {
            // Step 1: Create product
            $product = $this->productFactory->create();
            $product->setData($data);
            $product = $this->productRepository->save($product);

            // Step 2: Add images
            if (!empty($data['images'])) {
                $this->mediaGallery->addImages($product, $data['images']);
                $product = $this->productRepository->save($product);
            }

            // Step 3: Update stock
            if (isset($data['stock'])) {
                $this->stockRegistry->updateStockItemBySku($product->getSku(), $data['stock']);
            }

            // Step 4: Add to categories
            if (!empty($data['category_ids'])) {
                $product->setCategoryIds($data['category_ids']);
                $product = $this->productRepository->save($product);
            }

            $connection->commit();
            return $product;

        } catch (\Exception $e) {
            $connection->rollBack();
            $this->logger->error('Product creation failed: ' . $e->getMessage(), [
                'data' => $data,
                'trace' => $e->getTraceAsString()
            ]);
            throw new \Magento\Framework\Exception\CouldNotSaveException(
                __('Failed to create product: %1', $e->getMessage()),
                $e
            );
        }
    }
}
```

## Category 3: Performance Anti-Patterns

### Anti-Pattern 3.1: Loading Full Product Objects for Listing Pages

**Problem Code**:

```php
namespace Vendor\Module\Block\Product;

class BadProductList extends \Magento\Framework\View\Element\Template
{
    public function getProducts()
    {
        $collection = $this->collectionFactory->create();

        // ANTI-PATTERN: Loading all attributes when only few are needed
        $collection->addAttributeToSelect('*') // Loads 100+ attributes!
            ->setPageSize(24);

        return $collection;
    }
}
```

**Why This Is Wrong**:

1. **Memory waste**: Loading megabytes of unused data
2. **Slow queries**: Multiple EAV table joins
3. **Cache bloat**: Caching huge objects
4. **Network overhead**: More data transferred

**Impact**: 5-10x slower collection loading

**Correct Approach**:

```php
namespace Vendor\Module\Block\Product;

class GoodProductList extends \Magento\Framework\View\Element\Template
{
    public function getProducts()
    {
        $collection = $this->collectionFactory->create();

        // Only load attributes needed for display
        $collection->addAttributeToSelect([
                'name',
                'sku',
                'small_image',
                'price',
                'url_key'
            ])
            ->addUrlRewrite()
            ->addPriceData() // Use price index
            ->addAttributeToFilter('status', \Magento\Catalog\Model\Product\Attribute\Source\Status::STATUS_ENABLED)
            ->setPageSize(24);

        return $collection;
    }

    /**
     * For even better performance, use flat tables if enabled
     */
    public function getProductsOptimized()
    {
        $collection = $this->collectionFactory->create();

        if ($this->catalogConfig->isProductFlatEnabled()) {
            // Flat table = single query, no EAV joins
            $collection->setFlag('has_stock_status_filter', true);
        }

        $collection->addAttributeToSelect(['name', 'sku', 'small_image', 'price', 'url_key'])
            ->setPageSize(24);

        return $collection;
    }
}
```

### Anti-Pattern 3.2: Not Using Indexers for Calculated Data

**Problem Code**:

```php
namespace Vendor\Module\Block\Product;

class BadPriceDisplay
{
    public function getProductFinalPrice(\Magento\Catalog\Model\Product $product): float
    {
        // ANTI-PATTERN: Calculating price on every page load
        $price = $product->getPrice();

        // Apply tier price
        $tierPrices = $product->getTierPrices();
        foreach ($tierPrices as $tierPrice) {
            if ($tierPrice['qty'] <= 1) {
                $price = min($price, $tierPrice['price']);
            }
        }

        // Apply catalog rules
        $rulePrice = $this->catalogRuleModel->calcProductPriceRule($product, $price);
        if ($rulePrice) {
            $price = $rulePrice;
        }

        // Apply special price
        if ($product->getSpecialPrice() && $this->isSpecialPriceActive($product)) {
            $price = min($price, $product->getSpecialPrice());
        }

        return $price;
    }
}
```

**Why This Is Wrong**:

1. **Repeated calculations**: Same price calculated thousands of times
2. **DB queries**: Tier prices, catalog rules queried per product
3. **No caching**: Results not cached
4. **Slow pages**: Price calculation adds 50-100ms per product

**Correct Approach**:

```php
namespace Vendor\Module\Block\Product;

class GoodPriceDisplay
{
    /**
     * Use price index (pre-calculated by indexer)
     */
    public function getProductFinalPrice(\Magento\Catalog\Model\Product $product): float
    {
        // Price already calculated by catalog_product_price indexer
        return $product->getPriceInfo()
            ->getPrice(\Magento\Catalog\Pricing\Price\FinalPrice::PRICE_CODE)
            ->getAmount()
            ->getValue();
    }

    /**
     * For collections, use price index join
     */
    public function addPriceToCollection($collection)
    {
        $collection->addPriceData(
            $this->customerSession->getCustomerGroupId(),
            $this->storeManager->getStore()->getWebsiteId()
        );

        return $collection;
    }
}
```

### Anti-Pattern 3.3: Category Tree Recursion Without Caching

**Problem Code**:

```php
namespace Vendor\Module\Helper;

class BadCategoryTree
{
    public function getCategoryTree($parentId = 2): array
    {
        // ANTI-PATTERN: Recursive loading without cache
        $category = $this->categoryRepository->get($parentId);
        $tree = [
            'id' => $category->getId(),
            'name' => $category->getName(),
            'children' => []
        ];

        $children = $category->getChildrenCategories();
        foreach ($children as $child) {
            // Recursive call = exponential queries
            $tree['children'][] = $this->getCategoryTree($child->getId());
        }

        return $tree;
    }
}
```

**Why This Is Wrong**:

1. **Exponential queries**: 5-level tree = 100+ queries
2. **Memory explosion**: Deep recursion causes memory issues
3. **No caching**: Tree rebuilt on every request
4. **Stack overflow risk**: Deep trees can exceed recursion limit

**Correct Approach**:

```php
namespace Vendor\Module\Helper;

class GoodCategoryTree
{
    private const CACHE_TAG = 'category_tree';
    private const CACHE_LIFETIME = 3600;

    public function __construct(
        private readonly \Magento\Catalog\Api\CategoryRepositoryInterface $categoryRepository,
        private readonly \Magento\Catalog\Model\ResourceModel\Category\Tree $categoryTree,
        private readonly \Magento\Framework\App\CacheInterface $cache,
        private readonly \Magento\Framework\Serialize\SerializerInterface $serializer
    ) {}

    public function getCategoryTree(int $parentId = 2, int $maxDepth = 3): array
    {
        $cacheKey = self::CACHE_TAG . '_' . $parentId . '_' . $maxDepth;

        // Check cache first
        if ($cached = $this->cache->load($cacheKey)) {
            return $this->serializer->unserialize($cached);
        }

        // Load entire tree in single query using MPTT (Modified Preorder Tree Traversal)
        $parentCategory = $this->categoryRepository->get($parentId);
        $tree = $this->categoryTree->getTree($parentCategory, $maxDepth);

        // Build array structure
        $result = $this->buildTreeArray($tree);

        // Cache result
        $this->cache->save(
            $this->serializer->serialize($result),
            $cacheKey,
            [self::CACHE_TAG, \Magento\Catalog\Model\Category::CACHE_TAG],
            self::CACHE_LIFETIME
        );

        return $result;
    }

    private function buildTreeArray($treeNode): array
    {
        $result = [
            'id' => $treeNode['entity_id'],
            'name' => $treeNode['name'],
            'level' => $treeNode['level'],
            'children' => []
        ];

        if (!empty($treeNode['children'])) {
            foreach ($treeNode['children'] as $child) {
                $result['children'][] = $this->buildTreeArray($child);
            }
        }

        return $result;
    }
}
```

## Category 4: Code Structure Anti-Patterns

### Anti-Pattern 4.1: Using Helpers Instead of Services

**Problem Code**:

```php
namespace Vendor\Module\Helper;

class BadProductHelper extends \Magento\Framework\App\Helper\AbstractHelper
{
    // ANTI-PATTERN: Helper doing business logic
    public function getRelatedProducts($product)
    {
        return $product->getRelatedProducts();
    }

    public function calculateDiscount($product)
    {
        // Business logic in helper
        return $product->getPrice() * 0.1;
    }
}
```

**Why This Is Wrong**:

1. **Helpers are deprecated**: Magento discourages helpers for business logic
2. **Hard to test**: Helpers often depend on global state
3. **No type safety**: Helper methods often return mixed types
4. **Violates SRP**: Helpers become dumping ground for logic

**Correct Approach**:

```php
namespace Vendor\Module\Service;

class ProductRelationService
{
    public function __construct(
        private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
        private readonly \Magento\Catalog\Model\Product\Link $productLink
    ) {}

    public function getRelatedProducts(
        \Magento\Catalog\Api\Data\ProductInterface $product
    ): array {
        return $this->productLink->getLinkedProductCollection($product)
            ->setLinkModel($this->productLink)
            ->addAttributeToSelect('*')
            ->setLinkTypeId(\Magento\Catalog\Model\Product\Link::LINK_TYPE_RELATED)
            ->getItems();
    }
}

namespace Vendor\Module\Service;

class DiscountCalculator
{
    public function calculateDiscount(
        \Magento\Catalog\Api\Data\ProductInterface $product,
        float $discountPercent
    ): float {
        $price = $product->getFinalPrice();
        return $price * ($discountPercent / 100);
    }
}
```

### Anti-Pattern 4.2: Using ObjectManager Directly

**Problem Code**:

```php
namespace Vendor\Module\Model;

class BadProductService
{
    public function getProduct(string $sku)
    {
        // ANTI-PATTERN: Direct ObjectManager usage
        $objectManager = \Magento\Framework\App\ObjectManager::getInstance();
        $productRepository = $objectManager->get(\Magento\Catalog\Api\ProductRepositoryInterface::class);

        return $productRepository->get($sku);
    }
}
```

**Why This Is Wrong**:

1. **Hides dependencies**: Hard to understand what class needs
2. **Breaks testing**: Can't inject mocks
3. **Service locator anti-pattern**: Considered bad practice
4. **No compile-time checks**: Errors only at runtime
5. **Violates DI principles**: Defeats purpose of dependency injection

**Correct Approach**:

```php
namespace Vendor\Module\Model;

class GoodProductService
{
    public function __construct(
        private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository
    ) {}

    public function getProduct(string $sku): \Magento\Catalog\Api\Data\ProductInterface
    {
        return $this->productRepository->get($sku);
    }
}
```

### Anti-Pattern 4.3: Modifying Core Files or Using Preferences

**Problem Code**:

```xml
<!-- ANTI-PATTERN: Rewriting core class -->
<config>
    <preference for="Magento\Catalog\Model\Product"
                type="Vendor\Module\Model\Product"/>
</config>
```

**Why This Is Wrong**:

1. **Breaks upgrades**: Core changes overwrite your modifications
2. **Conflicts**: Multiple modules can't prefer same class
3. **Hard to debug**: Non-obvious code flow
4. **BC breaks**: Internal changes break your code

**Correct Approach**:

```php
// Use plugin instead
namespace Vendor\Module\Plugin\Catalog\Model;

class ProductPlugin
{
    public function afterGetFinalPrice(
        \Magento\Catalog\Model\Product $subject,
        $result
    ) {
        // Modify behavior via plugin
        return $result * 0.9; // 10% discount
    }
}
```

```xml
<!-- di.xml -->
<config>
    <type name="Magento\Catalog\Model\Product">
        <plugin name="vendor_module_product_plugin"
                type="Vendor\Module\Plugin\Catalog\Model\ProductPlugin"/>
    </type>
</config>
```

## Category 5: Security Anti-Patterns

### Anti-Pattern 5.1: No Input Validation

**Problem Code**:

```php
namespace Vendor\Module\Controller\Adminhtml\Product;

class BadSave extends \Magento\Backend\App\Action
{
    public function execute()
    {
        // ANTI-PATTERN: Using raw POST data without validation
        $data = $this->getRequest()->getPostValue();

        $product = $this->productFactory->create();
        $product->setData($data); // Dangerous: allows mass assignment
        $this->productRepository->save($product);
    }
}
```

**Why This Is Wrong**:

1. **Mass assignment vulnerability**: Attacker can set any field
2. **XSS risk**: Unescaped data stored in database
3. **SQL injection**: If data reaches raw queries
4. **No type validation**: Wrong data types cause errors

**Correct Approach**:

```php
namespace Vendor\Module\Controller\Adminhtml\Product;

class GoodSave extends \Magento\Backend\App\Action
{
    private const ALLOWED_FIELDS = ['sku', 'name', 'price', 'description'];

    public function execute()
    {
        $data = $this->getRequest()->getPostValue();

        // Validate CSRF token
        if (!$this->formKeyValidator->validate($this->getRequest())) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Invalid Form Key. Please refresh the page.')
            );
        }

        // Whitelist allowed fields
        $data = array_intersect_key($data, array_flip(self::ALLOWED_FIELDS));

        // Validate and sanitize
        $this->validateProductData($data);

        $product = $this->productFactory->create();

        // Set fields individually (safer than mass assignment)
        if (isset($data['sku'])) {
            $product->setSku($this->escaper->escapeHtml($data['sku']));
        }
        if (isset($data['name'])) {
            $product->setName($this->escaper->escapeHtml($data['name']));
        }
        if (isset($data['price'])) {
            $product->setPrice((float) $data['price']);
        }

        $this->productRepository->save($product);
    }

    private function validateProductData(array $data): void
    {
        if (empty($data['sku'])) {
            throw new \Magento\Framework\Exception\ValidatorException(
                __('SKU is required')
            );
        }

        if (!preg_match('/^[a-zA-Z0-9_-]+$/', $data['sku'])) {
            throw new \Magento\Framework\Exception\ValidatorException(
                __('SKU contains invalid characters')
            );
        }

        if (isset($data['price']) && !is_numeric($data['price'])) {
            throw new \Magento\Framework\Exception\ValidatorException(
                __('Price must be numeric')
            );
        }
    }
}
```

### Anti-Pattern 5.2: Missing ACL Checks

**Problem Code**:

```php
namespace Vendor\Module\Controller\Adminhtml\Product;

class BadDelete extends \Magento\Backend\App\Action
{
    // ANTI-PATTERN: No ACL check
    public function execute()
    {
        $id = $this->getRequest()->getParam('id');
        $this->productRepository->deleteById($id);
    }
}
```

**Correct Approach**:

```php
namespace Vendor\Module\Controller\Adminhtml\Product;

class GoodDelete extends \Magento\Backend\App\Action
{
    const ADMIN_RESOURCE = 'Magento_Catalog::products';

    protected function _isAllowed()
    {
        return $this->_authorization->isAllowed(self::ADMIN_RESOURCE);
    }

    public function execute()
    {
        if (!$this->_isAllowed()) {
            return $this->resultRedirectFactory->create()->setPath('admin/denied');
        }

        $id = (int) $this->getRequest()->getParam('id');

        if (!$id) {
            throw new \InvalidArgumentException('Invalid product ID');
        }

        try {
            $this->productRepository->deleteById($id);
            $this->messageManager->addSuccessMessage(__('Product deleted successfully'));
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
        }

        return $this->resultRedirectFactory->create()->setPath('*/*/');
    }
}
```

## Summary Checklist

**Data Access**:
- [ ] Use repositories, not direct model loading
- [ ] Use collections/SearchCriteria, not raw SQL
- [ ] Avoid N+1 queries - batch load data
- [ ] Use proper scoping (store, website, global)

**Save Operations**:
- [ ] Use mass actions for bulk updates
- [ ] Wrap multi-step operations in transactions
- [ ] Reindex once after batch operations
- [ ] Handle exceptions and rollback properly

**Performance**:
- [ ] Load only needed attributes (avoid `*`)
- [ ] Use index tables for calculated data
- [ ] Cache expensive operations
- [ ] Use flat catalog for large datasets

**Code Structure**:
- [ ] Use services, not helpers, for business logic
- [ ] Inject dependencies, never use ObjectManager
- [ ] Use plugins, not preferences/rewrites
- [ ] Follow SOLID principles

**Security**:
- [ ] Validate and sanitize all inputs
- [ ] Use whitelisting for mass assignment
- [ ] Check ACL permissions
- [ ] Escape output to prevent XSS

---

**Assumptions:**
- Magento 2.4.7+ / PHP 8.2+
- Production environment with realistic traffic
- Standard Magento coding practices

**Why this approach:**
Anti-patterns documentation prevents common mistakes. Real code examples show both wrong and right approaches. Impact metrics help prioritize fixes.

**Security impact:**
Examples demonstrate input validation, ACL checks, and CSRF protection. Following these patterns prevents XSS, SQL injection, and unauthorized access.

**Performance impact:**
Correct patterns improve performance 5-10x in most cases. N+1 query fixes alone can reduce page load by 1-2 seconds.

**Backward compatibility:**
All correct examples use service contracts and proper extension points. No breaking changes across minor versions.

**Tests to add:**
- Unit tests demonstrating anti-pattern problems
- Integration tests showing correct approach performance
- Security tests validating input sanitization

**Docs to update:**
- ANTI_PATTERNS.md (this file) as new patterns emerge
- Add metrics for performance impact
- Link to VERSION_COMPATIBILITY.md for upgrade considerations
