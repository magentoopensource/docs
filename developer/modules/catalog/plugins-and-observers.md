---
title: "Magento_Catalog Plugins & Observers"
module: "Magento_Catalog"
doc_type: "plugins-and-observers"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Catalog Plugins and Observers

## Overview

This document catalogs all significant plugin points and observer events in the Magento_Catalog module. Understanding these extension points is critical for safely extending catalog functionality without modifying core code.

**Target Version**: Magento 2.4.7+ / PHP 8.2+

## Plugin Architecture

Plugins (interceptors) allow you to intercept public method calls and modify arguments, behavior, or results without inheritance.

### Plugin Types

- **before**: Modify method arguments before execution
- **after**: Modify method result after execution
- **around**: Full control - wrap entire method

### Best Practices

1. **Prefer plugins over preferences** (rewrites)
2. **Keep plugins lightweight** - avoid heavy computation
3. **Use specific method interception** - avoid generic `around` when `before`/`after` suffice
4. **Document plugin chain order** - `sortOrder` matters when multiple plugins target same method
5. **Never modify constructor signatures** - plugins cannot intercept `__construct()`

## Critical Plugin Points

### 1. ProductRepositoryInterface

#### save()

**Use Cases**: Validation, data enrichment, external system sync, logging

```php
namespace Vendor\Module\Plugin\Catalog\Api;

class ProductRepositoryPlugin
{
    public function __construct(
        private readonly \Psr\Log\LoggerInterface $logger,
        private readonly \Vendor\Module\Service\ProductValidator $validator
    ) {}

    /**
     * Validate product data before save
     */
    public function beforeSave(
        \Magento\Catalog\Api\ProductRepositoryInterface $subject,
        \Magento\Catalog\Api\Data\ProductInterface $product,
        $saveOptions = false
    ): array {
        // Custom validation
        if (!$this->validator->validate($product)) {
            throw new \Magento\Framework\Exception\ValidatorException(
                __('Product validation failed: %1', $this->validator->getErrors())
            );
        }

        // Enrich product data
        if (!$product->getMetaDescription()) {
            $product->setMetaDescription(substr($product->getDescription(), 0, 160));
        }

        return [$product, $saveOptions];
    }

    /**
     * Sync product to external system after save
     */
    public function afterSave(
        \Magento\Catalog\Api\ProductRepositoryInterface $subject,
        \Magento\Catalog\Api\Data\ProductInterface $result
    ): \Magento\Catalog\Api\Data\ProductInterface {
        try {
            $this->externalSystemSync->syncProduct($result);
        } catch (\Exception $e) {
            $this->logger->error('External sync failed: ' . $e->getMessage());
            // Don't break save operation
        }

        return $result;
    }

    /**
     * Add transaction logging
     */
    public function aroundSave(
        \Magento\Catalog\Api\ProductRepositoryInterface $subject,
        callable $proceed,
        \Magento\Catalog\Api\Data\ProductInterface $product,
        $saveOptions = false
    ): \Magento\Catalog\Api\Data\ProductInterface {
        $startTime = microtime(true);
        $originalSku = $product->getOrigData('sku') ?: 'new';

        try {
            $result = $proceed($product, $saveOptions);

            $this->logger->info('Product saved', [
                'sku' => $result->getSku(),
                'original_sku' => $originalSku,
                'duration_ms' => (microtime(true) - $startTime) * 1000
            ]);

            return $result;
        } catch (\Exception $e) {
            $this->logger->error('Product save failed', [
                'sku' => $product->getSku(),
                'error' => $e->getMessage()
            ]);
            throw $e;
        }
    }
}
```

**Configuration** (`di.xml`):

```xml
<type name="Magento\Catalog\Api\ProductRepositoryInterface">
    <plugin name="vendor_module_product_repository_plugin"
            type="Vendor\Module\Plugin\Catalog\Api\ProductRepositoryPlugin"
            sortOrder="10"/>
</type>
```

#### get() / getList()

**Use Cases**: Data enrichment, access control, custom filtering

```php
namespace Vendor\Module\Plugin\Catalog\Api;

class ProductRepositoryGetPlugin
{
    /**
     * Add extension attributes after get
     */
    public function afterGet(
        \Magento\Catalog\Api\ProductRepositoryInterface $subject,
        \Magento\Catalog\Api\Data\ProductInterface $result
    ): \Magento\Catalog\Api\Data\ProductInterface {
        $extensionAttributes = $result->getExtensionAttributes();

        if ($extensionAttributes === null) {
            $extensionAttributes = $this->extensionAttributesFactory->create();
        }

        // Add custom data
        $vendorInfo = $this->vendorRepository->getByProductId($result->getId());
        $extensionAttributes->setVendorInfo($vendorInfo);

        $result->setExtensionAttributes($extensionAttributes);

        return $result;
    }

    /**
     * Filter results by custom criteria
     */
    public function afterGetList(
        \Magento\Catalog\Api\ProductRepositoryInterface $subject,
        \Magento\Catalog\Api\Data\ProductSearchResultsInterface $searchResults
    ): \Magento\Catalog\Api\Data\ProductSearchResultsInterface {
        $items = $searchResults->getItems();

        // Apply custom business logic filtering
        $filteredItems = array_filter($items, function ($product) {
            return $this->accessControl->canViewProduct($product);
        });

        $searchResults->setItems($filteredItems);
        $searchResults->setTotalCount(count($filteredItems));

        return $searchResults;
    }
}
```

### 2. CategoryRepositoryInterface

#### save()

**Use Cases**: URL validation, automatic slug generation, hierarchy validation

```php
namespace Vendor\Module\Plugin\Catalog\Api;

class CategoryRepositoryPlugin
{
    /**
     * Validate category data before save
     */
    public function beforeSave(
        \Magento\Catalog\Api\CategoryRepositoryInterface $subject,
        \Magento\Catalog\Api\Data\CategoryInterface $category
    ): array {
        // Ensure URL key is set
        if (!$category->getUrlKey()) {
            $urlKey = $this->urlKeyGenerator->generate($category->getName());
            $category->setUrlKey($urlKey);
        }

        // Validate parent exists
        if ($category->getParentId()) {
            try {
                $parent = $subject->get($category->getParentId());
            } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
                throw new \Magento\Framework\Exception\ValidatorException(
                    __('Parent category does not exist')
                );
            }
        }

        return [$category];
    }

    /**
     * Update sitemap after category save
     */
    public function afterSave(
        \Magento\Catalog\Api\CategoryRepositoryInterface $subject,
        \Magento\Catalog\Api\Data\CategoryInterface $result
    ): \Magento\Catalog\Api\Data\CategoryInterface {
        $this->sitemapUpdater->addCategory($result);
        return $result;
    }
}
```

### 3. Product Model

#### getPrice() / getFinalPrice()

**Use Cases**: Custom pricing logic, B2B pricing, dynamic pricing

```php
namespace Vendor\Module\Plugin\Catalog\Model;

class ProductPricePlugin
{
    /**
     * Apply custom pricing for B2B customers
     */
    public function afterGetFinalPrice(
        \Magento\Catalog\Model\Product $subject,
        $result
    ): float {
        if ($this->customerSession->isLoggedIn()) {
            $customer = $this->customerSession->getCustomer();

            if ($customer->getCustomAttribute('is_b2b_customer')) {
                $b2bDiscount = (float) $customer->getCustomAttribute('discount_percentage')->getValue();
                $result = $result * (1 - $b2bDiscount / 100);
            }
        }

        return $result;
    }
}
```

### 4. Product Collection

#### addAttributeToSelect()

**Use Cases**: Optimize attribute loading, conditional attribute selection

```php
namespace Vendor\Module\Plugin\Catalog\Model\ResourceModel\Product;

class CollectionPlugin
{
    /**
     * Always load critical custom attributes
     */
    public function afterAddAttributeToSelect(
        \Magento\Catalog\Model\ResourceModel\Product\Collection $subject,
        $result,
        $attribute,
        $joinType = false
    ) {
        // Ensure critical attributes are always loaded
        $criticalAttributes = ['vendor_id', 'vendor_sku', 'sourcing_status'];

        foreach ($criticalAttributes as $attr) {
            if (!isset($subject->getSelect()->getPart(\Zend_Db_Select::COLUMNS)[$attr])) {
                $subject->addAttributeToSelect($attr, $joinType);
            }
        }

        return $result;
    }
}
```

### 5. URL Rewrite Management

#### getProductRequestPath()

**Use Cases**: Custom URL structure, multilingual URLs, shortened URLs

```php
namespace Vendor\Module\Plugin\CatalogUrlRewrite\Model;

class ProductUrlPathGeneratorPlugin
{
    /**
     * Customize product URL format
     */
    public function afterGetUrlPath(
        \Magento\CatalogUrlRewrite\Model\ProductUrlPathGenerator $subject,
        $result,
        $product,
        $category = null
    ): string {
        // Add category path only for specific attribute sets
        if ($category && in_array($product->getAttributeSetId(), [4, 5, 6])) {
            return $category->getUrlPath() . '/' . $product->getUrlKey();
        }

        // Short URLs for other products
        return $product->getUrlKey();
    }
}
```

## Observer Events

### Product Events

#### catalog_product_save_before

**Dispatched**: Before product is saved to database
**Event Object**: `product` (Magento\Catalog\Model\Product)

**Use Cases**: Validation, data normalization, logging

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

class ProductSaveBeforeObserver implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $observer->getEvent()->getProduct();

        // Auto-generate SKU if empty
        if (!$product->getSku()) {
            $product->setSku($this->skuGenerator->generate($product));
        }

        // Normalize data
        $product->setName(ucwords(strtolower($product->getName())));

        // Set default values
        if (!$product->getData('vendor_id')) {
            $product->setData('vendor_id', $this->config->getDefaultVendorId());
        }
    }
}
```

**Configuration** (`events.xml`):

```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="catalog_product_save_before">
        <observer name="vendor_module_product_save_before"
                  instance="Vendor\Module\Observer\ProductSaveBeforeObserver"/>
    </event>
</config>
```

#### catalog_product_save_after

**Dispatched**: After product is saved to database
**Event Object**: `product` (Magento\Catalog\Model\Product)

**Use Cases**: Indexing, cache clearing, external sync, notifications

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

class ProductSaveAfterObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Vendor\Module\Service\SearchIndexer $searchIndexer,
        private readonly \Vendor\Module\Service\NotificationService $notificationService,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    public function execute(Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $observer->getEvent()->getProduct();

        try {
            // Update custom search index
            $this->searchIndexer->indexProduct($product);

            // Notify relevant parties of changes
            if ($product->dataHasChangedFor('price')) {
                $this->notificationService->notifyPriceChange($product);
            }

            if ($product->dataHasChangedFor('qty')) {
                $this->notificationService->notifyStockChange($product);
            }
        } catch (\Exception $e) {
            $this->logger->error('Product save after observer failed: ' . $e->getMessage());
        }
    }
}
```

#### catalog_product_delete_before

**Dispatched**: Before product is deleted
**Event Object**: `product` (Magento\Catalog\Model\Product)

**Use Cases**: Cascade deletion, archive data, prevent deletion

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

class ProductDeleteBeforeObserver implements ObserverInterface
{
    /**
     * Archive product before deletion
     */
    public function execute(Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $observer->getEvent()->getProduct();

        // Archive product data
        $this->archiveService->archiveProduct($product);

        // Prevent deletion of products with active orders
        if ($this->orderChecker->hasActiveOrders($product)) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Cannot delete product with active orders. Disable it instead.')
            );
        }
    }
}
```

#### catalog_product_delete_after

**Dispatched**: After product is deleted
**Event Object**: `product` (Magento\Catalog\Model\Product)

**Use Cases**: Cleanup, external system updates

```php
namespace Vendor\Module\Observer;

class ProductDeleteAfterObserver implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $observer->getEvent()->getProduct();

        // Clean up related custom data
        $this->customDataCleaner->cleanupProductData($product->getId());

        // Remove from external systems
        $this->externalSystemSync->removeProduct($product->getSku());
    }
}
```

#### catalog_product_load_after

**Dispatched**: After product is loaded from database
**Event Object**: `product` (Magento\Catalog\Model\Product)

**Use Cases**: Data enrichment, dynamic attribute population

```php
namespace Vendor\Module\Observer;

class ProductLoadAfterObserver implements ObserverInterface
{
    /**
     * Enrich product with custom data after load
     */
    public function execute(Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $observer->getEvent()->getProduct();

        // Add real-time inventory from external warehouse
        if ($product->getTypeId() === 'simple') {
            $externalQty = $this->warehouseApi->getQuantity($product->getSku());
            $product->setData('warehouse_qty', $externalQty);
        }

        // Calculate dynamic shipping estimate
        $product->setData('estimated_delivery', $this->shippingEstimator->estimate($product));
    }
}
```

#### catalog_product_collection_load_after

**Dispatched**: After product collection is loaded
**Event Object**: `collection` (Magento\Catalog\Model\ResourceModel\Product\Collection)

**Use Cases**: Batch data enrichment, collection filtering

```php
namespace Vendor\Module\Observer;

class ProductCollectionLoadAfterObserver implements ObserverInterface
{
    /**
     * Enrich entire collection with custom data
     */
    public function execute(Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\ResourceModel\Product\Collection $collection */
        $collection = $observer->getEvent()->getCollection();

        // Batch load related custom data
        $productIds = $collection->getAllIds();
        $vendorData = $this->vendorRepository->getByProductIds($productIds);

        foreach ($collection as $product) {
            if (isset($vendorData[$product->getId()])) {
                $product->setData('vendor_info', $vendorData[$product->getId()]);
            }
        }
    }
}
```

#### catalog_product_get_final_price

**Dispatched**: During final price calculation
**Event Object**: `product` (Magento\Catalog\Model\Product), `qty`

**Use Cases**: Custom pricing logic, dynamic discounts

```php
namespace Vendor\Module\Observer;

class ProductGetFinalPriceObserver implements ObserverInterface
{
    /**
     * Apply custom pricing rules
     */
    public function execute(Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $observer->getEvent()->getProduct();
        $qty = $observer->getEvent()->getQty();

        $finalPrice = $product->getData('final_price');

        // Apply flash sale discount
        if ($this->flashSaleService->isActive($product)) {
            $discount = $this->flashSaleService->getDiscount($product);
            $finalPrice = $finalPrice * (1 - $discount / 100);
        }

        // Apply loyalty program discount
        if ($this->customerSession->isLoggedIn()) {
            $loyaltyDiscount = $this->loyaltyService->getProductDiscount(
                $product,
                $this->customerSession->getCustomerId()
            );
            $finalPrice = $finalPrice * (1 - $loyaltyDiscount / 100);
        }

        $product->setData('final_price', $finalPrice);
    }
}
```

### Category Events

#### catalog_category_save_before

**Dispatched**: Before category is saved
**Event Object**: `category` (Magento\Catalog\Model\Category)

**Use Cases**: Validation, URL key generation, hierarchy checks

```php
namespace Vendor\Module\Observer;

class CategorySaveBeforeObserver implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Category $category */
        $category = $observer->getEvent()->getCategory();

        // Generate URL key if missing
        if (!$category->getUrlKey()) {
            $urlKey = $this->urlKeyGenerator->generate($category->getName());
            $category->setUrlKey($urlKey);
        }

        // Validate max depth
        if ($category->getLevel() > $this->config->getMaxCategoryDepth()) {
            throw new \Magento\Framework\Exception\ValidatorException(
                __('Category depth exceeds maximum allowed level')
            );
        }
    }
}
```

#### catalog_category_save_after

**Dispatched**: After category is saved
**Event Object**: `category` (Magento\Catalog\Model\Category)

**Use Cases**: Cache invalidation, search reindex, sitemap update

```php
namespace Vendor\Module\Observer;

class CategorySaveAfterObserver implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Category $category */
        $category = $observer->getEvent()->getCategory();

        // Invalidate navigation cache
        $this->cacheManager->clean([
            \Magento\Catalog\Model\Category::CACHE_TAG,
            'catalog_navigation'
        ]);

        // Update sitemap
        $this->sitemapGenerator->regenerate();
    }
}
```

#### catalog_category_move_after

> **Note:** There is no `catalog_category_move_before` event in core Magento.
> Only `catalog_category_move_after` exists, dispatched by `CatalogUrlRewrite` module
> after a category tree move completes. To validate moves before they happen,
> use a plugin on `\Magento\Catalog\Model\ResourceModel\Category::changeParent()`.

**Dispatched**: After category is moved in tree
**Event Objects**: `category_id`, `prev_parent`, `parent_id`

### Attribute Events

#### catalog_entity_attribute_save_after

**Dispatched**: After product/category attribute is saved
**Event Object**: `attribute` (Magento\Catalog\Model\ResourceModel\Eav\Attribute)

**Use Cases**: Index updates, cache clearing

```php
namespace Vendor\Module\Observer;

class AttributeSaveAfterObserver implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\ResourceModel\Eav\Attribute $attribute */
        $attribute = $observer->getEvent()->getAttribute();

        // If attribute became searchable, reindex
        if ($attribute->dataHasChangedFor('is_searchable') && $attribute->getIsSearchable()) {
            $this->indexerRegistry->get('catalogsearch_fulltext')->invalidate();
        }

        // If attribute became filterable, update layered nav
        if ($attribute->dataHasChangedFor('is_filterable')) {
            $this->cacheManager->clean(['layered_navigation']);
        }
    }
}
```

### Import/Export Events

#### catalog_product_import_bunch_save_after

**Dispatched**: After a batch of products is imported
**Event Objects**: `bunch`, `adapter`

**Use Cases**: Post-import processing, data validation

```php
namespace Vendor\Module\Observer;

class ProductImportAfterObserver implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        $bunch = $observer->getEvent()->getBunch();
        $adapter = $observer->getEvent()->getAdapter();

        // Process imported products
        foreach ($bunch as $rowData) {
            if (isset($rowData['sku'])) {
                $this->postImportProcessor->process($rowData['sku']);
            }
        }
    }
}
```

## Complete Event Reference

### Product Events

| Event Name | When Dispatched | Event Data | Common Use Cases |
|------------|----------------|------------|------------------|
| `catalog_product_new_action` | Product created via admin | `product` | Welcome emails, notifications |
| `catalog_product_save_before` | Before save | `product` | Validation, data normalization |
| `catalog_product_save_after` | After save | `product` | Indexing, external sync |
| `catalog_product_save_commit_after` | After DB commit | `product` | Final cleanup, notifications |
| `catalog_product_delete_before` | Before deletion | `product` | Archive, validation |
| `catalog_product_delete_after` | After deletion | `product` | Cleanup |
| `catalog_product_delete_commit_after` | After DB commit | `product` | Final cleanup |
| `catalog_product_load_after` | After load | `product` | Data enrichment |
| `catalog_product_get_final_price` | Price calculation | `product`, `qty` | Custom pricing |
| `catalog_product_collection_load_after` | After collection load | `collection` | Batch enrichment |
| `catalog_product_attribute_update_before` | Mass attribute update | `attributes_data`, `product_ids` | Validation |

### Category Events

| Event Name | When Dispatched | Event Data | Common Use Cases |
|------------|----------------|------------|------------------|
| `catalog_category_save_before` | Before save | `category` | Validation, URL generation |
| `catalog_category_save_after` | After save | `category` | Cache clear, sitemap |
| `catalog_category_save_commit_after` | After DB commit | `category` | Final operations |
| `catalog_category_delete_before` | Before deletion | `category` | Validation |
| `catalog_category_delete_after` | After deletion | `category` | Cleanup |
| `catalog_category_move_after` | After tree move (CatalogUrlRewrite) | `category_id`, `prev_parent`, `parent_id` | URL regeneration |
| `catalog_category_tree_init_inactive_category_ids` | Building tree | `tree` | Custom filtering |

### Configuration

**events.xml** (Global scope):

```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="catalog_product_save_after">
        <observer name="vendor_module_product_observer"
                  instance="Vendor\Module\Observer\ProductSaveAfterObserver"/>
    </event>
</config>
```

**di.xml** (Plugin configuration):

```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Catalog\Api\ProductRepositoryInterface">
        <plugin name="vendor_module_product_plugin"
                type="Vendor\Module\Plugin\Catalog\Api\ProductRepositoryPlugin"
                sortOrder="10"
                disabled="false"/>
    </type>
</config>
```

## Plugin vs Observer Decision Matrix

| Scenario | Use Plugin | Use Observer | Reason |
|----------|-----------|-------------|--------|
| Modify method arguments | Yes | No | Plugins intercept methods |
| Modify method return value | Yes | No | Plugins have direct access |
| React to completed action | No | Yes | Observers handle notifications |
| Prevent method execution | Yes (around) | No | Plugins control flow |
| Multiple independent reactions | No | Yes | Observers decouple logic |
| Performance critical path | Yes | Maybe | Plugins are slightly faster |
| Need transaction rollback capability | No | Yes | Observers within transaction |

---

**Assumptions:**
- Magento 2.4.7+ with service contracts
- PSR-3 logging available
- Custom modules follow Magento coding standards

**Why this approach:**
Plugins provide type-safe method interception. Observers enable event-driven decoupling. Both preserve upgrade path vs preferences/rewrites.

**Security impact:**
Plugins on repositories should validate ACL if implementing authorization. Observers should not bypass security checks in product save flow.

**Performance impact:**
Each plugin adds method call overhead (~0.1ms). Observers in save flow execute synchronously - keep lightweight. Consider message queues for heavy processing.

**Backward compatibility:**
Plugins on service contracts (repositories) are BC-safe. Plugins on models may break if method signatures change. Observers are BC-safe as long as event data structure is maintained.

**Tests to add:**
- Unit tests for plugin methods with mocked subjects
- Integration tests verifying plugin execution in full flow
- Observer tests with event dispatching
- Performance tests measuring plugin overhead

**Docs to update:**
- PLUGINS_AND_OBSERVERS.md (this file) when adding significant new extension points
- Create visual plugin chain diagrams for complex scenarios
- Document breaking changes in VERSION_COMPATIBILITY.md
