---
title: "Magento_Catalog Version Compatibility"
module: "Magento_Catalog"
doc_type: "version-compatibility"
version: "2.4.7+"
last_updated: "2026-02-09"
---

# Magento_Catalog Version Compatibility

## Overview

This document tracks verified API surface, deprecations, and compatibility information for the Magento_Catalog module in version 2.4.7.

> **Note:** This document only contains facts verified against the Magento 2.4.7 source code. Version-to-version changelog entries that could not be verified against source have been removed.

**Module Version (2.4.7):** 104.0.7
**PHP Requirement:** `~8.1.0||~8.2.0||~8.3.0`

---

## Key Facts

### CatalogInventory Module Status

The `Magento_CatalogInventory` module (100.4.7) **still exists** in Magento 2.4.7. It is marked as `"abandoned"` in its `composer.json` with a reference to `magento/inventory-metapackage`, but the module is still present and functional. The Catalog module depends on it: `"magento/module-catalog-inventory": "100.4.*"`.

**Do NOT remove CatalogInventory usage assuming it was removed in 2.4.7.** Plan migration to MSI as a gradual process.

### Flat Catalog Configuration

Flat catalog configuration paths **still exist** in 2.4.7 `etc/config.xml`:

```xml
<flat_catalog_category>0</flat_catalog_category>
<flat_catalog_product>0</flat_catalog_product>
```

Both are disabled by default. The flat product index has `max_index_count` set to 64.

### Interface Return Types

Magento 2.4.7 Catalog API interfaces use **docblock `@return` annotations only** — they do NOT have PHP native return type declarations. For example:

```php
// Actual ProductRepositoryInterface in 2.4.7
// NO `: ProductInterface` return type — only @return docblock
public function save(
    \Magento\Catalog\Api\Data\ProductInterface $product,
    $saveOptions = false
);

// NO `: ProductInterface` return type
public function get($sku, $editMode = false, $storeId = null, $forceReload = false);

// NO `: ProductInterface` return type
public function getById($productId, $editMode = false, $storeId = null, $forceReload = false);
```

When implementing these interfaces, you do NOT need PHP return type declarations — match the interface signature exactly.

---

## Composer Dependencies (2.4.7)

The Catalog module depends on these Magento modules:

| Dependency | Version Constraint |
|---|---|
| magento/framework | 103.0.* |
| magento/module-backend | 102.0.* |
| magento/module-catalog-inventory | 100.4.* |
| magento/module-catalog-rule | 101.2.* |
| magento/module-catalog-url-rewrite | 100.4.* |
| magento/module-checkout | 100.4.* |
| magento/module-cms | 104.0.* |
| magento/module-config | 101.2.* |
| magento/module-customer | 103.0.* |
| magento/module-directory | 100.4.* |
| magento/module-eav | 102.1.* |
| magento/module-indexer | 100.4.* |
| magento/module-media-storage | 100.4.* |
| magento/module-msrp | 100.4.* |
| magento/module-page-cache | 100.4.* |
| magento/module-product-alert | 100.4.* |
| magento/module-quote | 101.2.* |
| magento/module-store | 101.1.* |
| magento/module-tax | 100.4.* |
| magento/module-theme | 101.1.* |
| magento/module-ui | 101.2.* |
| magento/module-url-rewrite | 102.0.* |
| magento/module-widget | 101.2.* |
| magento/module-wishlist | 101.2.* |

---

## Verified Deprecations (from `@deprecated` annotations in source)

### Deprecated Interfaces

| Interface | Deprecated Since | Replacement |
|---|---|---|
| `ProductTierPriceManagementInterface` | 102.0.0 | `ScopedProductTierPriceManagementInterface` |

### Deprecated Classes

| Class | Deprecated Since | Notes |
|---|---|---|
| `Cron\SynchronizeWebsiteAttributes` | (no version) | See `Attribute\Backend\WebsiteSpecific\ValueSynchronizer` |
| `Model\ResourceModel\Category\Flat` | 100.0.2 | Flat catalog deprecated |

### Deprecated Product Model Methods (sample)

| Method | Deprecated Since | Reason |
|---|---|---|
| `getMediaAttributes()` | 102.0.6 | — |
| `getMediaAttributeValues()` | 102.0.6 | — |
| `isConfigurable()` | 102.0.6 | — |
| `getConfigurableAttributeCollection()` | 102.0.4 | — |
| `getGroupedLinkCollection()` | 102.0.6 | — |
| `getRelatedLinkCollection()` | 102.0.5 | — |
| `getStockItem()` | 102.0.0 | "Product model shouldn't be responsible for stock status" |
| `setStockItem()` | 102.0.0 | Same reason |

### Deprecated Controller Classes

| Class | Deprecated Since |
|---|---|
| `Controller\Adminhtml\Category` | 101.0.8 |

### Deprecated Helpers

| Class | Deprecated Since |
|---|---|
| Various `ProductLink\Repository` methods | 103.0.4 |

---

## `@since` Version Tags

The majority of Catalog API interfaces are tagged `@since 100.0.2` (foundational). Notable later additions:

| Class/Interface | @since |
|---|---|
| `ProductFrontendActionInterface` | 102.0.0 |
| `CategoryLinkInterface` | 102.0.0 |
| `BasePriceInterface` | 102.0.0 |
| `CostInterface` | 102.0.0 |
| `SpecialPriceInterface` | 102.0.0 |
| `ScopedProductTierPriceManagementInterface` | 102.0.0 |
| `CategoryListInterface` | 102.0.0 |

---

## Product Model Class Signature

The `Product` model in 2.4.7 extends and implements:

```php
class Product extends \Magento\Catalog\Model\AbstractModel implements
    IdentityInterface,
    SaleableInterface,
    ProductInterface,
    ResetAfterRequestInterface
```

Properties use **untyped** declarations (e.g., `protected $_cacheTag = 'cat_p'`), not PHP typed properties.

---

## Upgrade Best Practices

### Use Service Contracts

```php
// Recommended — uses interface (stable API)
public function __construct(
    private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository
) {}

// Avoid — uses concrete class (internal, may change)
public function __construct(
    private readonly \Magento\Catalog\Model\ProductRepository $productRepository
) {}
```

### Loading Products

```php
// Deprecated pattern (still works)
$product = $this->productFactory->create()->load($id);

// Recommended pattern
$product = $this->productRepository->getById($id);
```

---

**Disclaimer:** This document covers the Magento_Catalog module as found in `magento/product-community-edition:2.4.7`. Version-to-version change details (e.g., "added in 2.4.6") could not be verified from source alone and have been removed to prevent inaccuracies. For version-specific changelogs, consult the official Adobe Commerce release notes.
