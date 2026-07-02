---
title: "Magento_Catalog Architecture"
module: "Magento_Catalog"
doc_type: "architecture"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Catalog Architecture

## Overview

The `Magento_Catalog` module implements a sophisticated multi-layered architecture combining Entity-Attribute-Value (EAV) flexibility with repository patterns, service contracts, and domain-driven design principles. This document details the architectural patterns, database schema, key classes, and design decisions that enable extensible product and category management.

**Target Version**: Magento 2.4.7+ / PHP 8.2+

## Architectural Layers

### 1. Service Contract Layer (API)

Service contracts define the public API surface and ensure backward compatibility across versions. All external integrations should use these interfaces.

#### Core Repository Interfaces

```php
namespace Magento\Catalog\Api;

interface ProductRepositoryInterface
{
    /**
     * @param \Magento\Catalog\Api\Data\ProductInterface $product
     * @param bool $saveOptions
     * @return \Magento\Catalog\Api\Data\ProductInterface
     * @throws \Magento\Framework\Exception\CouldNotSaveException
     */
    public function save(
        \Magento\Catalog\Api\Data\ProductInterface $product,
        $saveOptions = false
    );

    /**
     * @param string $sku
     * @param bool $editMode
     * @param int|null $storeId
     * @param bool $forceReload
     * @return \Magento\Catalog\Api\Data\ProductInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function get($sku, $editMode = false, $storeId = null, $forceReload = false);

    /**
     * @param \Magento\Catalog\Api\Data\ProductInterface $product
     * @return bool
     * @throws \Magento\Framework\Exception\StateException
     */
    public function delete(\Magento\Catalog\Api\Data\ProductInterface $product);

    /**
     * @param string $sku
     * @return bool
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     * @throws \Magento\Framework\Exception\StateException
     */
    public function deleteById($sku);

    /**
     * @param \Magento\Framework\Api\SearchCriteriaInterface $searchCriteria
     * @return \Magento\Catalog\Api\Data\ProductSearchResultsInterface
     */
    public function getList(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria);
}
```

#### Data Transfer Objects (DTOs)

```php
namespace Magento\Catalog\Api\Data;

interface ProductInterface extends \Magento\Framework\Api\CustomAttributesDataInterface
{
    const SKU = 'sku';
    const NAME = 'name';
    const PRICE = 'price';
    const TYPE_ID = 'type_id';
    const ATTRIBUTE_SET_ID = 'attribute_set_id';
    const STATUS = 'status';
    const VISIBILITY = 'visibility';

    /**
     * @return string
     */
    public function getSku();

    /**
     * @param string $sku
     * @return $this
     */
    public function setSku($sku);

    /**
     * @return string
     */
    public function getName();

    /**
     * @param string $name
     * @return $this
     */
    public function setName($name);

    /**
     * @return float
     */
    public function getPrice();

    /**
     * @param float $price
     * @return $this
     */
    public function setPrice($price);

    // ... additional getters/setters
}
```

#### Search Criteria Example

```php
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Api\FilterBuilder;
use Magento\Framework\Api\SortOrderBuilder;

class ProductSearchService
{
    public function __construct(
        private readonly ProductRepositoryInterface $productRepository,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder,
        private readonly FilterBuilder $filterBuilder,
        private readonly SortOrderBuilder $sortOrderBuilder
    ) {}

    public function findExpensiveProducts(float $minPrice): array
    {
        // Build price filter
        $priceFilter = $this->filterBuilder
            ->setField('price')
            ->setValue($minPrice)
            ->setConditionType('gteq')
            ->create();

        // Build status filter (enabled only)
        $statusFilter = $this->filterBuilder
            ->setField('status')
            ->setValue(\Magento\Catalog\Model\Product\Attribute\Source\Status::STATUS_ENABLED)
            ->setConditionType('eq')
            ->create();

        // Build sort order
        $sortOrder = $this->sortOrderBuilder
            ->setField('price')
            ->setDirection(\Magento\Framework\Api\SortOrder::SORT_DESC)
            ->create();

        // Build search criteria
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilters([$priceFilter])
            ->addFilters([$statusFilter])
            ->addSortOrder($sortOrder)
            ->setPageSize(20)
            ->setCurrentPage(1)
            ->create();

        $searchResults = $this->productRepository->getList($searchCriteria);
        return $searchResults->getItems();
    }
}
```

### 2. Domain Model Layer

Domain models encapsulate business logic and data access. The catalog module uses both active record patterns (Model) and repository patterns.

#### Product Model Architecture

```php
namespace Magento\Catalog\Model;

class Product extends \Magento\Catalog\Model\AbstractModel implements
    \Magento\Catalog\Api\Data\ProductInterface,
    \Magento\Framework\DataObject\IdentityInterface
{
    // Cache tags for FPC invalidation
    const CACHE_TAG = 'cat_p';

    protected $_eventPrefix = 'catalog_product';
    protected $_eventObject = 'product';
    protected $_cacheTag = false; // Not set by default; uses CACHE_TAG constant

    /**
     * Product type instance
     */
    protected $_typeInstance = null;

    /**
     * Product options
     */
    protected $_options = [];

    /**
     * Get product type instance
     */
    public function getTypeInstance()
    {
        if ($this->_typeInstance === null) {
            $this->_typeInstance = $this->_typeInstanceFactory->create($this);
        }
        return $this->_typeInstance;
    }

    /**
     * Get final price with catalog rules applied
     */
    public function getFinalPrice($qty = null)
    {
        return $this->getPriceInfo()
            ->getPrice('final_price')
            ->getAmount()
            ->getValue();
    }

    // EAV-specific methods
    protected function _construct()
    {
        $this->_init(\Magento\Catalog\Model\ResourceModel\Product::class);
    }

    public function getIdentities()
    {
        $identities = [self::CACHE_TAG . '_' . $this->getId()];

        if (!$this->getId()) {
            return $identities;
        }

        // Add category identities for cache invalidation
        foreach ($this->getCategoryIds() as $categoryId) {
            $identities[] = \Magento\Catalog\Model\Category::CACHE_TAG . '_' . $categoryId;
        }

        return $identities;
    }
}
```

#### Product Type Architecture

Product types follow a strategy pattern:

```php
namespace Magento\Catalog\Model\Product\Type;

abstract class AbstractType
{
    /**
     * Check if product is composite (has children)
     */
    abstract public function isComposite($product);

    /**
     * Prepare product for cart
     */
    abstract public function prepareForCart(\Magento\Framework\DataObject $buyRequest, $product);

    /**
     * Check if product can be configured
     */
    public function canConfigure($product)
    {
        return false;
    }

    /**
     * Process configuration for product
     */
    public function processConfiguration(
        \Magento\Framework\DataObject $buyRequest,
        $product,
        $processMode = self::PROCESS_MODE_LITE
    ) {
        return [];
    }
}
```

#### Category Model with MPTT (Nested Set)

```php
namespace Magento\Catalog\Model;

class Category extends \Magento\Catalog\Model\AbstractModel implements
    \Magento\Catalog\Api\Data\CategoryInterface,
    \Magento\Framework\DataObject\IdentityInterface
{
    const CACHE_TAG = 'cat_c';
    const TREE_ROOT_ID = 1;

    /**
     * Get category path (IDs from root to current)
     */
    public function getPath()
    {
        return $this->getData('path');
    }

    /**
     * Get parent category IDs
     */
    public function getParentIds()
    {
        return array_diff($this->getPathIds(), [$this->getId()]);
    }

    /**
     * Get all children IDs (recursive)
     */
    public function getAllChildren($asArray = false)
    {
        $children = $this->getResource()->getAllChildren($this);

        if ($asArray) {
            return explode(',', $children);
        }

        return $children;
    }

    /**
     * Move category to new parent
     */
    public function move($parentId, $afterCategoryId)
    {
        $parent = $this->categoryRepository->get($parentId);
        $this->getResource()->changeParent($this, $parent, $afterCategoryId);
        return $this;
    }
}
```

### 3. Resource Model Layer (Data Access)

Resource models handle database operations, including EAV complexity.

#### Product Resource Model

```php
namespace Magento\Catalog\Model\ResourceModel;

class Product extends \Magento\Catalog\Model\ResourceModel\AbstractResource
{
    /**
     * Main entity table
     */
    protected $_entityTable = 'catalog_product_entity';

    /**
     * Load product by SKU
     */
    public function getIdBySku($sku)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getEntityTable(), 'entity_id')
            ->where('sku = :sku');

        return $connection->fetchOne($select, ['sku' => $sku]);
    }

    /**
     * Get product websites
     */
    public function getWebsiteIds($product)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getTable('catalog_product_website'), 'website_id')
            ->where('product_id = ?', $product->getId());

        return $connection->fetchCol($select);
    }

    /**
     * Save product-category relations
     */
    public function saveCategoryIds($product)
    {
        $categoryIds = $product->getCategoryIds();
        $oldCategoryIds = $this->getCategoryIds($product);

        $insert = array_diff($categoryIds, $oldCategoryIds);
        $delete = array_diff($oldCategoryIds, $categoryIds);

        $connection = $this->getConnection();

        if (!empty($delete)) {
            $connection->delete(
                $this->getTable('catalog_category_product'),
                ['product_id = ?' => $product->getId(), 'category_id IN (?)' => $delete]
            );
        }

        if (!empty($insert)) {
            $data = [];
            foreach ($insert as $categoryId) {
                $data[] = [
                    'category_id' => $categoryId,
                    'product_id' => $product->getId(),
                    'position' => 0
                ];
            }
            $connection->insertMultiple($this->getTable('catalog_category_product'), $data);
        }

        return $this;
    }
}
```

#### Category Resource with Tree Operations

```php
namespace Magento\Catalog\Model\ResourceModel\Category;

class Tree extends \Magento\Catalog\Model\ResourceModel\Category
{
    /**
     * Move category in tree structure
     */
    public function changeParent($category, $newParent, $afterCategoryId = null)
    {
        $childrenIds = $this->getAllChildren($category);
        $connection = $this->getConnection();

        $connection->beginTransaction();

        try {
            // Update path for moved category and children
            $newPath = $newParent->getPath() . '/' . $category->getId();
            $oldPath = $category->getPath();

            // Update category
            $connection->update(
                $this->getEntityTable(),
                ['path' => $newPath, 'parent_id' => $newParent->getId()],
                ['entity_id = ?' => $category->getId()]
            );

            // Update children paths
            foreach ($childrenIds as $childId) {
                $childPath = $connection->fetchOne(
                    $connection->select()
                        ->from($this->getEntityTable(), 'path')
                        ->where('entity_id = ?', $childId)
                );

                $newChildPath = str_replace($oldPath, $newPath, $childPath);
                $connection->update(
                    $this->getEntityTable(),
                    ['path' => $newChildPath],
                    ['entity_id = ?' => $childId]
                );
            }

            $connection->commit();
        } catch (\Exception $e) {
            $connection->rollBack();
            throw $e;
        }

        return $this;
    }
}
```

### 4. Repository Implementation Layer

Repositories implement service contracts and provide caching, validation, and event dispatching.

```php
namespace Magento\Catalog\Model;

class ProductRepository implements \Magento\Catalog\Api\ProductRepositoryInterface
{
    private array $instances = [];
    private array $instancesById = [];

    public function __construct(
        private readonly ProductFactory $productFactory,
        private readonly Product\Initialization\Helper\ProductLinks $linkInitializer,
        private readonly Product\Gallery\Processor $mediaGalleryProcessor,
        private readonly \Magento\Store\Model\StoreManagerInterface $storeManager,
        private readonly \Magento\Framework\Api\SearchCriteriaBuilder $searchCriteriaBuilder,
        private readonly CollectionProcessorInterface $collectionProcessor,
        private readonly \Magento\Catalog\Model\ResourceModel\Product\CollectionFactory $collectionFactory
    ) {}

    public function get($sku, $editMode = false, $storeId = null, $forceReload = false)
    {
        $cacheKey = $this->getCacheKey([$editMode, $storeId]);

        if (!isset($this->instances[$sku][$cacheKey]) || $forceReload) {
            $product = $this->productFactory->create();

            $productId = $this->resourceModel->getIdBySku($sku);
            if (!$productId) {
                throw new NoSuchEntityException(
                    __("The product that was requested doesn't exist. Verify the product and try again.")
                );
            }

            if ($editMode) {
                $product->setData('_edit_mode', true);
            }

            if ($storeId !== null) {
                $product->setData('store_id', $storeId);
            } else {
                $storeId = $this->storeManager->getStore()->getId();
            }

            $product->load($productId);
            $this->instances[$sku][$cacheKey] = $product;
            $this->instancesById[$product->getId()][$cacheKey] = $product;
        }

        return $this->instances[$sku][$cacheKey];
    }

    public function save(\Magento\Catalog\Api\Data\ProductInterface $product, $saveOptions = false)
    {
        $tierPrices = $product->getData('tier_price');

        try {
            // Validate SKU uniqueness
            $existingProduct = $this->get($product->getSku());
            if ($existingProduct->getId() && $existingProduct->getId() != $product->getId()) {
                throw new \Magento\Framework\Exception\CouldNotSaveException(
                    __('A product with the SKU "%1" already exists.', $product->getSku())
                );
            }
        } catch (NoSuchEntityException $e) {
            // SKU is unique, proceed
        }

        $this->resourceModel->save($product);

        // Save tier prices
        if ($tierPrices !== null) {
            $product->setData('tier_price', $tierPrices);
            $this->tierPriceManagement->update($product);
        }

        // Clear cache
        unset($this->instances[$product->getSku()]);
        unset($this->instancesById[$product->getId()]);

        return $this->get($product->getSku(), false, $product->getStoreId());
    }

    public function getList(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria)
    {
        $collection = $this->collectionFactory->create();
        $this->collectionProcessor->process($searchCriteria, $collection);

        $searchResults = $this->searchResultsFactory->create();
        $searchResults->setSearchCriteria($searchCriteria);
        $searchResults->setItems($collection->getItems());
        $searchResults->setTotalCount($collection->getSize());

        return $searchResults;
    }

    private function getCacheKey(array $data): string
    {
        return sha1(serialize($data));
    }
}
```

## Database Schema

### Core Product Tables

#### catalog_product_entity (Main Product Table)

```sql
CREATE TABLE `catalog_product_entity` (
  `entity_id` int unsigned NOT NULL AUTO_INCREMENT,
  `attribute_set_id` smallint unsigned NOT NULL DEFAULT '0',
  `type_id` varchar(32) NOT NULL DEFAULT 'simple',
  `sku` varchar(64) DEFAULT NULL,
  `has_options` smallint NOT NULL DEFAULT '0',
  `required_options` smallint unsigned NOT NULL DEFAULT '0',
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`entity_id`),
  UNIQUE KEY `CATALOG_PRODUCT_ENTITY_SKU` (`sku`),
  KEY `CATALOG_PRODUCT_ENTITY_ATTRIBUTE_SET_ID` (`attribute_set_id`),
  KEY `CATALOG_PRODUCT_ENTITY_TYPE_ID` (`type_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### EAV Attribute Value Tables

Products use typed EAV tables for attribute storage:

- `catalog_product_entity_varchar`: Text values (< 255 chars)
- `catalog_product_entity_text`: Long text values
- `catalog_product_entity_int`: Integer values and select options
- `catalog_product_entity_decimal`: Decimal values (price, weight)
- `catalog_product_entity_datetime`: Date/time values

```sql
CREATE TABLE `catalog_product_entity_varchar` (
  `value_id` int NOT NULL AUTO_INCREMENT,
  `attribute_id` smallint unsigned NOT NULL,
  `store_id` smallint unsigned NOT NULL DEFAULT '0',
  `entity_id` int unsigned NOT NULL,
  `value` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`value_id`),
  UNIQUE KEY `CATALOG_PRODUCT_ENTITY_VARCHAR_ENTITY_ID_ATTRIBUTE_ID_STORE_ID`
    (`entity_id`,`attribute_id`,`store_id`),
  KEY `CATALOG_PRODUCT_ENTITY_VARCHAR_ATTRIBUTE_ID` (`attribute_id`),
  KEY `CATALOG_PRODUCT_ENTITY_VARCHAR_STORE_ID` (`store_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### catalog_product_website (Multi-Website Assignment)

```sql
CREATE TABLE `catalog_product_website` (
  `product_id` int unsigned NOT NULL,
  `website_id` smallint unsigned NOT NULL,
  PRIMARY KEY (`product_id`,`website_id`),
  KEY `CATALOG_PRODUCT_WEBSITE_WEBSITE_ID` (`website_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Category Tables

#### catalog_category_entity (Main Category Table with MPTT)

```sql
CREATE TABLE `catalog_category_entity` (
  `entity_id` int unsigned NOT NULL AUTO_INCREMENT,
  `attribute_set_id` smallint unsigned NOT NULL DEFAULT '0',
  `parent_id` int unsigned NOT NULL DEFAULT '0',
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `path` varchar(255) NOT NULL,
  `position` int NOT NULL,
  `level` int NOT NULL DEFAULT '0',
  `children_count` int NOT NULL,
  PRIMARY KEY (`entity_id`),
  KEY `CATALOG_CATEGORY_ENTITY_LEVEL` (`level`),
  KEY `CATALOG_CATEGORY_ENTITY_PATH` (`path`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

The `path` field implements Modified Preorder Tree Traversal:
- Format: `1/2/5/12` (IDs from root to current category)
- Enables efficient tree queries
- `level` indicates depth (root = 0, immediate children = 1)

#### catalog_category_product (Product-Category Relations)

```sql
CREATE TABLE `catalog_category_product` (
  `category_id` int unsigned NOT NULL DEFAULT '0',
  `product_id` int unsigned NOT NULL DEFAULT '0',
  `position` int NOT NULL DEFAULT '0',
  PRIMARY KEY (`category_id`,`product_id`),
  KEY `CATALOG_CATEGORY_PRODUCT_PRODUCT_ID` (`product_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Relationship Tables

#### catalog_product_link (Related, Upsell, Cross-sell)

```sql
CREATE TABLE `catalog_product_link` (
  `link_id` int unsigned NOT NULL AUTO_INCREMENT,
  `product_id` int unsigned NOT NULL DEFAULT '0',
  `linked_product_id` int unsigned NOT NULL DEFAULT '0',
  `link_type_id` smallint unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (`link_id`),
  UNIQUE KEY `CATALOG_PRODUCT_LINK_PRODUCT_ID_LINKED_PRODUCT_ID_LINK_TYPE_ID`
    (`product_id`,`linked_product_id`,`link_type_id`),
  KEY `CATALOG_PRODUCT_LINK_LINKED_PRODUCT_ID` (`linked_product_id`),
  KEY `CATALOG_PRODUCT_LINK_LINK_TYPE_ID` (`link_type_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Link types:
- `1`: Related Products
- `4`: Upsell Products
- `5`: Cross-sell Products

### Index Tables (Performance)

#### catalog_product_index_price (Pre-calculated Prices)

```sql
CREATE TABLE `catalog_product_index_price` (
  `entity_id` int unsigned NOT NULL,
  `customer_group_id` int unsigned NOT NULL,
  `website_id` smallint unsigned NOT NULL,
  `tax_class_id` smallint unsigned DEFAULT '0',
  `price` decimal(20,6) DEFAULT NULL,
  `final_price` decimal(20,6) DEFAULT NULL,
  `min_price` decimal(20,6) NOT NULL,
  `max_price` decimal(20,6) NOT NULL,
  `tier_price` decimal(20,6) DEFAULT NULL,
  PRIMARY KEY (`entity_id`,`customer_group_id`,`website_id`),
  KEY `CATALOG_PRODUCT_INDEX_PRICE_CUSTOMER_GROUP_ID` (`customer_group_id`),
  KEY `CATALOG_PRODUCT_INDEX_PRICE_WEBSITE_ID` (`website_id`),
  KEY `CATALOG_PRODUCT_INDEX_PRICE_MIN_PRICE` (`min_price`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Populated by `catalog_product_price` indexer for fast price queries with catalog rules applied.

#### catalog_category_product_index (Category-Product Denormalized)

```sql
CREATE TABLE `catalog_category_product_index` (
  `category_id` int unsigned NOT NULL DEFAULT '0',
  `product_id` int unsigned NOT NULL DEFAULT '0',
  `position` int DEFAULT NULL,
  `is_parent` smallint unsigned NOT NULL DEFAULT '0',
  `store_id` smallint unsigned NOT NULL DEFAULT '0',
  `visibility` smallint unsigned NOT NULL,
  PRIMARY KEY (`category_id`,`product_id`,`store_id`),
  KEY `CATALOG_CATEGORY_PRODUCT_INDEX_PRODUCT_ID_STORE_ID_CATEGORY_ID_VISIBILITY`
    (`product_id`,`store_id`,`category_id`,`visibility`),
  KEY `CATALOG_CATEGORY_PRODUCT_INDEX_STORE_ID` (`store_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Key Classes and Their Responsibilities

### Product Management

| Class | Responsibility | Layer |
|-------|---------------|-------|
| `Magento\Catalog\Api\ProductRepositoryInterface` | Product CRUD service contract | API |
| `Magento\Catalog\Model\ProductRepository` | Repository implementation with caching | Domain |
| `Magento\Catalog\Model\Product` | Product domain model | Domain |
| `Magento\Catalog\Model\ResourceModel\Product` | Database operations for products | Data Access |
| `Magento\Catalog\Model\Product\Type\AbstractType` | Product type strategy base | Domain |
| `Magento\Catalog\Model\Product\Type\Simple` | Simple product type implementation | Domain |

### Category Management

| Class | Responsibility | Layer |
|-------|---------------|-------|
| `Magento\Catalog\Api\CategoryRepositoryInterface` | Category CRUD service contract | API |
| `Magento\Catalog\Model\CategoryRepository` | Repository implementation | Domain |
| `Magento\Catalog\Model\Category` | Category domain model with tree logic | Domain |
| `Magento\Catalog\Model\ResourceModel\Category\Tree` | Tree structure operations | Data Access |

### Attribute Management

| Class | Responsibility | Layer |
|-------|---------------|-------|
| `Magento\Catalog\Api\ProductAttributeRepositoryInterface` | Attribute metadata service contract | API |
| `Magento\Catalog\Model\ResourceModel\Eav\Attribute` | Product attribute model | Domain |
| `Magento\Eav\Model\Entity\Attribute\Set` | Attribute set management | Domain |

### Pricing

| Class | Responsibility | Layer |
|-------|---------------|-------|
| `Magento\Catalog\Pricing\Price\FinalPrice` | Calculate final price with rules | Domain |
| `Magento\Catalog\Pricing\Price\RegularPrice` | Base price calculation | Domain |
| `Magento\Catalog\Pricing\Price\TierPrice` | Tier pricing calculation | Domain |
| `Magento\Catalog\Model\Product\PriceModifier\Composite` | Price modification orchestration | Domain |

### Media Gallery

| Class | Responsibility | Layer |
|-------|---------------|-------|
| `Magento\Catalog\Model\Product\Gallery\Processor` | Image processing and storage | Domain |
| `Magento\Catalog\Api\ProductAttributeMediaGalleryManagementInterface` | Gallery API contract | API |

## Dependency Graph

```
Magento_Catalog depends on:
├── Magento_Store (Multi-store scoping)
├── Magento_Eav (Attribute system)
├── Magento_Customer (Customer groups for pricing)
├── Magento_Backend (Admin UI)
├── Magento_Theme (Frontend rendering)
├── Magento_MediaStorage (Media handling)
├── Magento_Indexer (Index management)
├── Magento_UrlRewrite (SEO URLs)
├── Magento_Msrp (Manufacturer Suggested Retail Price)
└── Magento_CatalogUrlRewrite (URL generation)

Modules that depend on Magento_Catalog:
├── Magento_CatalogInventory (Stock management - deprecated)
├── Magento_InventoryCatalog (MSI integration)
├── Magento_CatalogRule (Promotional pricing)
├── Magento_CatalogSearch (Search layer)
├── Magento_Quote (Shopping cart)
├── Magento_Sales (Order processing)
├── Magento_ConfigurableProduct (Configurable products)
├── Magento_Bundle (Bundle products)
├── Magento_GroupedProduct (Grouped products)
├── Magento_Downloadable (Downloadable products)
└── Magento_GiftCard (Adobe Commerce only)
```

## Extension Points

### Plugins (Preferred)

Intercept public methods on repositories, models, and services:

```php
namespace Vendor\Module\Plugin;

class ProductRepositoryPlugin
{
    /**
     * Before save - validate custom business rules
     */
    public function beforeSave(
        \Magento\Catalog\Api\ProductRepositoryInterface $subject,
        \Magento\Catalog\Api\Data\ProductInterface $product,
        $saveOptions = false
    ) {
        // Custom validation logic
        if ($product->getCustomAttribute('vendor_id') === null) {
            throw new \Magento\Framework\Exception\ValidatorException(
                __('Vendor ID is required')
            );
        }

        return [$product, $saveOptions];
    }

    /**
     * After get - add custom data
     */
    public function afterGet(
        \Magento\Catalog\Api\ProductRepositoryInterface $subject,
        \Magento\Catalog\Api\Data\ProductInterface $result
    ) {
        // Enrich product with custom data
        $extensionAttributes = $result->getExtensionAttributes();
        $extensionAttributes->setVendorName($this->getVendorName($result));
        $result->setExtensionAttributes($extensionAttributes);

        return $result;
    }
}
```

### Observers (Event-Driven)

Key catalog events:

```php
// Product events
catalog_product_save_before
catalog_product_save_after
catalog_product_delete_before
catalog_product_delete_after
catalog_product_load_after
catalog_product_collection_load_after

// Category events
catalog_category_save_before
catalog_category_save_after
catalog_category_delete_before
catalog_category_delete_after
catalog_category_move_after  // dispatched by CatalogUrlRewrite, not Catalog; no _before event

// Attribute events
catalog_product_attribute_update_before
catalog_entity_attribute_save_after
```

Example observer:

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\ObserverInterface;

class ProductSaveAfterObserver implements ObserverInterface
{
    public function execute(\Magento\Framework\Event\Observer $observer)
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $observer->getEvent()->getProduct();

        // Custom logic after product save
        $this->logger->info('Product saved: ' . $product->getSku());
    }
}
```

### Custom Product Types

Extend `\Magento\Catalog\Model\Product\Type\AbstractType`:

```php
namespace Vendor\Module\Model\Product\Type;

class Subscription extends \Magento\Catalog\Model\Product\Type\Virtual
{
    const TYPE_CODE = 'subscription';

    public function deleteTypeSpecificData(\Magento\Catalog\Model\Product $product)
    {
        // Cleanup subscription-specific data
    }

    public function beforeSave($product)
    {
        parent::beforeSave($product);
        // Subscription-specific validation
        return $this;
    }
}
```

Register in `etc/product_types.xml`:

```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Catalog:etc/product_types.xsd">
    <type name="subscription" label="Subscription Product" modelInstance="Vendor\Module\Model\Product\Type\Subscription"
          indexPriority="50" sortOrder="100" isQty="false">
        <priceModel instance="Vendor\Module\Model\Product\Type\Subscription\Price"/>
    </type>
</config>
```

## Design Patterns Used

1. **Repository Pattern**: `ProductRepository`, `CategoryRepository`
2. **Factory Pattern**: `ProductFactory`, `CategoryFactory`
3. **Strategy Pattern**: Product types (`AbstractType` implementations)
4. **Observer Pattern**: Event system for extensibility
5. **Proxy Pattern**: Lazy loading of related entities
6. **Composite Pattern**: Price modifiers and calculations
7. **Builder Pattern**: `SearchCriteriaBuilder`, `FilterBuilder`
8. **Data Mapper Pattern**: Resource models for ORM

---

**Assumptions:**
- Magento 2.4.7+ with MSI enabled
- PHP 8.2+ with typed properties
- Elasticsearch/OpenSearch for catalog search
- Redis for cache backend

**Why this approach:**
Service contracts provide API stability across versions. Repository pattern with caching improves performance. EAV system enables flexible attribute management without schema migrations.

**Security impact:**
All repository methods validate ACL when called from admin context. Extension attributes provide safe data enrichment without modifying core contracts. Event observers should validate permissions before executing sensitive operations.

**Performance impact:**
Repository instances cache prevents duplicate loads. Index tables eliminate EAV joins on frontend. Category tree uses MPTT for efficient queries. Price index pre-calculates final prices with catalog rules applied.

**Backward compatibility:**
Service contracts (API interfaces) are BC-stable within major versions. Plugin points on public methods are safe. Direct model usage may break across versions—always use repositories.

**Tests to add:**
- Unit tests for repository implementations with mocked dependencies
- Integration tests for CRUD operations with fixtures
- API-functional tests for REST/GraphQL endpoints
- Database schema validation tests

**Docs to update:**
- ARCHITECTURE.md (this file) when significant structural changes occur
- Add sequence diagrams for complex flows in EXECUTION_FLOWS.md
- Update VERSION_COMPATIBILITY.md when API changes happen
