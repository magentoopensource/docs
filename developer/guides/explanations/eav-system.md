---
title: "EAV System Architecture: Understanding Entity-Attribute-Value in Magento 2"
description: "Comprehensive explanation of Magento 2's EAV system: table structure, creating custom EAV entities and attributes, attribute sets and groups, flat tables for performance, when to use EAV vs flat tables, and optimization strategies"
type: "explanation"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "90 minutes"
topics:
  - eav
  - attributes
  - database-design
  - performance
  - flat-tables
  - catalog
  - customer
  - optimization
last_updated: "2026-02-07"
---

# EAV System Architecture: Understanding Entity-Attribute-Value in Magento 2

## Learning Objectives

By completing this guide, you will:

- Understand the EAV (Entity-Attribute-Value) pattern and why Magento uses it
- Master the EAV table structure and relationships
- Create custom EAV entities from scratch
- Implement custom product and customer attributes
- Work with attribute sets, groups, and options
- Understand flat table architecture and when to use it
- Optimize EAV queries for performance
- Choose between EAV and flat tables for custom entities
- Debug EAV-related performance issues

## Introduction

The **Entity-Attribute-Value (EAV)** model is a database design pattern that enables flexible, dynamic schema by storing attributes as data rather than columns. Magento 2 uses EAV extensively for products, customers, categories, and orders, allowing merchants to add custom attributes without database schema changes.

### Why EAV in Magento?

**Problem EAV solves:**
- E-commerce requires highly customizable entities (products with varying attributes)
- Traditional relational model: Adding attribute = schema change (ALTER TABLE)
- EAV model: Adding attribute = insert row in attribute table

**Example scenario:**
- Clothing store: Products need `size`, `color`, `material`
- Electronics store: Products need `warranty_period`, `voltage`, `battery_type`
- Traditional DB: Single `catalog_product_entity` table with 100+ columns (many NULL)
- EAV DB: Dynamic attributes stored in separate value tables

### EAV Trade-offs

**Advantages:**
- **Schema flexibility**: Add attributes without migrations
- **Sparse data efficiency**: NULL values don't consume space
- **Multi-tenant friendly**: Different stores/attribute sets share infrastructure
- **Admin UI integration**: Attributes automatically appear in admin forms

**Disadvantages:**
- **Query complexity**: Requires joins across multiple tables
- **Performance overhead**: 10-20x slower than flat table queries
- **Index limitations**: Cannot index attribute values directly
- **ORM abstraction**: Hides complexity but adds overhead

**When Magento uses EAV:**
- Products (`catalog_product`)
- Categories (`catalog_category`)
- Customers (`customer`)
- Customer addresses (`customer_address`)

**When Magento uses flat tables:**
- Orders (`sales_order`)
- Invoices (`sales_invoice`)
- Quotes (`quote`)
- CMS pages (`cms_page`)

## EAV Table Structure

### Core EAV Tables

Every EAV entity type has six table types:

```
1. Entity Table          : catalog_product_entity
2. Attribute Table       : eav_attribute (shared across all EAV entities)
3. Entity Type Table     : eav_entity_type (defines entity types)
4. Attribute Set Table   : eav_attribute_set (groups attributes)
5. Attribute Group Table : eav_attribute_group (organizes attributes in admin)
6. Value Tables (typed)  : catalog_product_entity_int, _varchar, _text, _decimal, _datetime
```

### 1. Entity Table

**Structure (`catalog_product_entity`):**

```sql
CREATE TABLE `catalog_product_entity` (
  `entity_id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `attribute_set_id` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `type_id` VARCHAR(32) NOT NULL DEFAULT 'simple',
  `sku` VARCHAR(64) DEFAULT NULL,
  `has_options` SMALLINT(6) NOT NULL DEFAULT 0,
  `required_options` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`entity_id`),
  UNIQUE KEY `CATALOG_PRODUCT_ENTITY_SKU` (`sku`),
  KEY `CATALOG_PRODUCT_ENTITY_ATTRIBUTE_SET_ID` (`attribute_set_id`),
  KEY `CATALOG_PRODUCT_ENTITY_TYPE_ID` (`type_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='Catalog Product Table';
```

**Key fields:**
- `entity_id`: Primary key, referenced by all value tables
- `attribute_set_id`: Groups product into template (e.g., "Default", "Clothing")
- `type_id`: Product type (simple, configurable, bundle, grouped)
- `sku`: Unique product identifier
- Static attributes: `created_at`, `updated_at` stored directly (not in EAV)

**Why static attributes?**
- Frequently queried fields stored in entity table for performance
- Avoids joins for common operations (SKU lookup, type filtering)

### 2. Attribute Table

**Structure (`eav_attribute`):**

```sql
CREATE TABLE `eav_attribute` (
  `attribute_id` SMALLINT(5) UNSIGNED NOT NULL AUTO_INCREMENT,
  `entity_type_id` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `attribute_code` VARCHAR(255) NOT NULL,
  `attribute_model` VARCHAR(255) DEFAULT NULL,
  `backend_model` VARCHAR(255) DEFAULT NULL,
  `backend_type` VARCHAR(8) NOT NULL DEFAULT 'static',
  `backend_table` VARCHAR(255) DEFAULT NULL,
  `frontend_model` VARCHAR(255) DEFAULT NULL,
  `frontend_input` VARCHAR(50) DEFAULT NULL,
  `frontend_label` VARCHAR(255) DEFAULT NULL,
  `frontend_class` VARCHAR(255) DEFAULT NULL,
  `source_model` VARCHAR(255) DEFAULT NULL,
  `is_required` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `is_user_defined` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `default_value` TEXT DEFAULT NULL,
  `is_unique` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `note` VARCHAR(255) DEFAULT NULL,
  PRIMARY KEY (`attribute_id`),
  UNIQUE KEY `EAV_ATTRIBUTE_ENTITY_TYPE_ID_ATTRIBUTE_CODE` (`entity_type_id`,`attribute_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='Eav Attribute';
```

**Critical fields:**
- `attribute_code`: Unique identifier (e.g., `color`, `manufacturer`)
- `backend_type`: Data type → determines value table (`int`, `varchar`, `text`, `decimal`, `datetime`)
- `frontend_input`: Input type (text, select, multiselect, date, boolean, etc.)
- `source_model`: Provides option values for select/multiselect
- `is_required`: Validation flag
- `is_user_defined`: Custom vs system attribute

**Additional product-specific attributes (`catalog_eav_attribute`):**

```sql
CREATE TABLE `catalog_eav_attribute` (
  `attribute_id` SMALLINT(5) UNSIGNED NOT NULL,
  `is_global` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 1,
  `is_visible` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 1,
  `is_searchable` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `is_filterable` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `is_comparable` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `is_visible_on_front` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `is_html_allowed_on_front` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `is_used_for_price_rules` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `is_filterable_in_search` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `used_in_product_listing` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `used_for_sort_by` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `apply_to` VARCHAR(255) DEFAULT NULL,
  `is_visible_in_advanced_search` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `position` INT(11) NOT NULL DEFAULT 0,
  `is_wysiwyg_enabled` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `is_used_for_promo_rules` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `is_required_in_admin_store` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `is_used_in_grid` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `is_visible_in_grid` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `is_filterable_in_grid` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `search_weight` FLOAT NOT NULL DEFAULT 1,
  PRIMARY KEY (`attribute_id`),
  CONSTRAINT `FK_CATALOG_EAV_ATTRIBUTE` FOREIGN KEY (`attribute_id`) REFERENCES `eav_attribute` (`attribute_id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**Scope flags:**
- `is_global`: Scope (0=store, 1=global, 2=website)
- `is_searchable`: Index for fulltext search
- `is_filterable`: Show in layered navigation
- `used_in_product_listing`: Load on category pages

### 3. Value Tables (Typed)

Each backend type has a dedicated value table:

**INT values (`catalog_product_entity_int`):**
```sql
CREATE TABLE `catalog_product_entity_int` (
  `value_id` INT(11) NOT NULL AUTO_INCREMENT,
  `attribute_id` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `store_id` SMALLINT(5) UNSIGNED NOT NULL DEFAULT 0,
  `entity_id` INT(10) UNSIGNED NOT NULL DEFAULT 0,
  `value` INT(11) DEFAULT NULL,
  PRIMARY KEY (`value_id`),
  UNIQUE KEY `UNQ_CATALOG_PRODUCT_ENTITY_INT` (`entity_id`,`attribute_id`,`store_id`),
  KEY `IDX_ATTRIBUTE_ID` (`attribute_id`),
  KEY `IDX_STORE_ID` (`store_id`),
  CONSTRAINT `FK_CATALOG_PRODUCT_ENTITY_INT_ENTITY` FOREIGN KEY (`entity_id`) REFERENCES `catalog_product_entity` (`entity_id`) ON DELETE CASCADE,
  CONSTRAINT `FK_CATALOG_PRODUCT_ENTITY_INT_ATTRIBUTE` FOREIGN KEY (`attribute_id`) REFERENCES `eav_attribute` (`attribute_id`) ON DELETE CASCADE,
  CONSTRAINT `FK_CATALOG_PRODUCT_ENTITY_INT_STORE` FOREIGN KEY (`store_id`) REFERENCES `store` (`store_id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**Key design elements:**
- Composite unique key: `(entity_id, attribute_id, store_id)` prevents duplicates
- `store_id`: Enables store-specific attribute values
- Foreign keys: Cascade deletes maintain referential integrity

**Value table types:**
- `_int`: Integers, boolean (0/1), select option IDs
- `_varchar`: Short strings (<255 chars), SKU references
- `_text`: Long text, HTML content, JSON
- `_decimal`: Prices, weights, dimensions
- `_datetime`: Dates, timestamps

### Query Example: Fetch Product with Attributes

**Without EAV awareness (incorrect):**
```php
// This ONLY gets static attributes (sku, created_at)
$product = $productRepository->get('SKU123');
echo $product->getName(); // NULL (not loaded)
```

**Correct approach:**
```php
// Load with specific attributes
$product = $productRepository->get('SKU123', false, null, true);
echo $product->getName(); // Loaded from catalog_product_entity_varchar

// Or load via collection
$collection = $productCollectionFactory->create();
$collection->addAttributeToSelect(['name', 'price', 'description']);
$collection->addFieldToFilter('sku', 'SKU123');
$product = $collection->getFirstItem();
```

**Generated SQL (simplified):**
```sql
SELECT
    e.entity_id,
    e.sku,
    name_attr.value AS name,
    price_attr.value AS price
FROM catalog_product_entity e
LEFT JOIN catalog_product_entity_varchar name_attr
    ON name_attr.entity_id = e.entity_id
    AND name_attr.attribute_id = (SELECT attribute_id FROM eav_attribute WHERE attribute_code = 'name' AND entity_type_id = 4)
    AND name_attr.store_id IN (0, 1) -- Global + current store
LEFT JOIN catalog_product_entity_decimal price_attr
    ON price_attr.entity_id = e.entity_id
    AND price_attr.attribute_id = (SELECT attribute_id FROM eav_attribute WHERE attribute_code = 'price' AND entity_type_id = 4)
    AND price_attr.store_id IN (0, 1)
WHERE e.sku = 'SKU123';
```

**Performance note:** Each attribute = 1 LEFT JOIN. Loading 20 attributes = 20 joins.

## Creating Custom EAV Attributes

### Product Attribute via Data Patch

**Setup/Patch/Data/AddWarrantyPeriodAttribute.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Catalog\Model\Product;
use Magento\Eav\Model\Entity\Attribute\ScopedAttributeInterface;
use Magento\Eav\Setup\EavSetup;
use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

/**
 * Add warranty_period product attribute
 */
class AddWarrantyPeriodAttribute implements DataPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
        private readonly EavSetupFactory $eavSetupFactory
    ) {}

    /**
     * @inheritdoc
     */
    public function apply(): void
    {
        /** @var EavSetup $eavSetup */
        $eavSetup = $this->eavSetupFactory->create(['setup' => $this->moduleDataSetup]);

        $eavSetup->addAttribute(
            Product::ENTITY,
            'warranty_period',
            [
                // Basic Configuration
                'type' => 'int',                          // Backend type: int, varchar, text, decimal, datetime
                'label' => 'Warranty Period (Months)',    // Frontend label
                'input' => 'text',                        // Input type: text, select, multiselect, date, boolean, textarea
                'required' => false,                      // Is required in admin
                'user_defined' => true,                   // Custom attribute (vs system)
                'default' => 12,                          // Default value

                // Scope
                'global' => ScopedAttributeInterface::SCOPE_GLOBAL, // SCOPE_GLOBAL, SCOPE_WEBSITE, SCOPE_STORE

                // Frontend Configuration
                'visible' => true,                        // Visible in admin
                'visible_on_front' => true,               // Show on product page
                'used_in_product_listing' => true,        // Load in category pages
                'searchable' => false,                    // Index for search
                'filterable' => false,                    // Show in layered navigation
                'filterable_in_search' => false,          // Show in search layered nav
                'comparable' => true,                     // Show in product comparison
                'used_for_sort_by' => false,              // Allow sorting by this attribute

                // Backend Configuration
                'backend' => '',                          // Backend model for processing
                'frontend' => '',                         // Frontend model for rendering
                'source' => '',                           // Source model for options
                'frontend_class' => 'validate-number validate-zero-or-greater', // CSS validation classes

                // Admin Configuration
                'is_used_in_grid' => true,                // Show in admin product grid
                'is_visible_in_grid' => false,            // Visible by default in grid
                'is_filterable_in_grid' => true,          // Filterable in grid
                'position' => 100,                        // Sort position in attribute group

                // Advanced
                'apply_to' => '',                         // Apply to product types: simple,configurable,bundle (empty = all)
                'group' => 'Product Details',             // Attribute group name
                'note' => 'Warranty period in months',    // Admin note
            ]
        );
    }

    /**
     * @inheritdoc
     */
    public static function getDependencies(): array
    {
        return [];
    }

    /**
     * @inheritdoc
     */
    public function getAliases(): array
    {
        return [];
    }
}
```

**Apply patch:**
```bash
bin/magento setup:upgrade
bin/magento cache:flush
```

### Select Attribute with Options

**Setup/Patch/Data/AddConditionAttribute.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Catalog\Model\Product;
use Magento\Eav\Model\Entity\Attribute\ScopedAttributeInterface;
use Magento\Eav\Setup\EavSetup;
use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class AddConditionAttribute implements DataPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
        private readonly EavSetupFactory $eavSetupFactory
    ) {}

    public function apply(): void
    {
        /** @var EavSetup $eavSetup */
        $eavSetup = $this->eavSetupFactory->create(['setup' => $this->moduleDataSetup]);

        $eavSetup->addAttribute(
            Product::ENTITY,
            'product_condition',
            [
                'type' => 'int',
                'label' => 'Product Condition',
                'input' => 'select',
                'source' => \Magento\Eav\Model\Entity\Attribute\Source\Table::class,
                'required' => true,
                'global' => ScopedAttributeInterface::SCOPE_GLOBAL,
                'visible_on_front' => true,
                'used_in_product_listing' => true,
                'filterable' => true,
                'user_defined' => true,
                'option' => [
                    'values' => [
                        'New',
                        'Refurbished',
                        'Used - Like New',
                        'Used - Good',
                        'Used - Acceptable',
                    ],
                ],
                'group' => 'Product Details',
            ]
        );
    }

    public static function getDependencies(): array
    {
        return [];
    }

    public function getAliases(): array
    {
        return [];
    }
}
```

**Option storage:**
- Options stored in `eav_attribute_option` table
- Option labels in `eav_attribute_option_value` (multi-language)
- Value table stores option ID (integer), not label text

### Customer Attribute

**Setup/Patch/Data/AddCustomerLoyaltyTierAttribute.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Customer\Model\Customer;
use Magento\Customer\Setup\CustomerSetup;
use Magento\Customer\Setup\CustomerSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class AddCustomerLoyaltyTierAttribute implements DataPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
        private readonly CustomerSetupFactory $customerSetupFactory
    ) {}

    public function apply(): void
    {
        /** @var CustomerSetup $customerSetup */
        $customerSetup = $this->customerSetupFactory->create(['setup' => $this->moduleDataSetup]);

        $customerSetup->addAttribute(
            Customer::ENTITY,
            'loyalty_tier',
            [
                'type' => 'varchar',
                'label' => 'Loyalty Tier',
                'input' => 'select',
                'source' => \Vendor\Module\Model\Customer\Attribute\Source\LoyaltyTier::class,
                'required' => false,
                'default' => 'bronze',
                'visible' => true,
                'user_defined' => true,
                'system' => false,
                'position' => 100,
                'sort_order' => 100,
            ]
        );

        // Make attribute visible in admin customer form
        $attribute = $customerSetup->getEavConfig()->getAttribute(Customer::ENTITY, 'loyalty_tier');
        $attribute->setData('used_in_forms', [
            'adminhtml_customer',      // Admin customer edit
            'customer_account_edit',   // Customer account edit
            'customer_account_create', // Customer registration
        ]);
        $attribute->save();
    }

    public static function getDependencies(): array
    {
        return [];
    }

    public function getAliases(): array
    {
        return [];
    }
}
```

**Source model for options:**

**Model/Customer/Attribute/Source/LoyaltyTier.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Customer\Attribute\Source;

use Magento\Eav\Model\Entity\Attribute\Source\AbstractSource;

class LoyaltyTier extends AbstractSource
{
    private const TIER_BRONZE = 'bronze';
    private const TIER_SILVER = 'silver';
    private const TIER_GOLD = 'gold';
    private const TIER_PLATINUM = 'platinum';

    /**
     * Get all options
     *
     * @return array
     */
    public function getAllOptions(): array
    {
        if ($this->_options === null) {
            $this->_options = [
                ['label' => __('Bronze'), 'value' => self::TIER_BRONZE],
                ['label' => __('Silver'), 'value' => self::TIER_SILVER],
                ['label' => __('Gold'), 'value' => self::TIER_GOLD],
                ['label' => __('Platinum'), 'value' => self::TIER_PLATINUM],
            ];
        }

        return $this->_options;
    }
}
```

## Attribute Sets and Groups

### Attribute Sets

**Attribute sets** are templates that define which attributes are available for a product type.

**Example sets:**
- **Default**: General products (name, price, description, weight)
- **Clothing**: Default + size, color, material, care_instructions
- **Electronics**: Default + warranty_period, voltage, battery_type

**Benefits:**
- Organize attributes by product category
- Reduce admin form complexity (show only relevant attributes)
- Enable different validation rules per product type

**Database tables:**
- `eav_attribute_set`: Set metadata (name, entity_type_id)
- `eav_entity_attribute`: Links attributes to sets

### Attribute Groups

**Attribute groups** organize attributes within a set into tabs/sections in the admin form.

**Example groups for "Clothing" set:**
- **Product Details**: name, price, sku
- **Content**: description, short_description
- **Images**: image, small_image, thumbnail
- **Clothing Attributes**: size, color, material
- **Inventory**: qty, stock_status

**Database table:** `eav_attribute_group`

### Creating Attribute Set Programmatically

**Setup/Patch/Data/CreateElectronicsAttributeSet.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Catalog\Model\Product;
use Magento\Eav\Api\AttributeSetRepositoryInterface;
use Magento\Eav\Api\Data\AttributeSetInterfaceFactory;
use Magento\Eav\Model\Entity\Attribute\SetFactory as AttributeSetFactory;
use Magento\Eav\Setup\EavSetup;
use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class CreateElectronicsAttributeSet implements DataPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
        private readonly EavSetupFactory $eavSetupFactory,
        private readonly AttributeSetFactory $attributeSetFactory,
        private readonly AttributeSetInterfaceFactory $attributeSetInterfaceFactory,
        private readonly AttributeSetRepositoryInterface $attributeSetRepository
    ) {}

    public function apply(): void
    {
        /** @var EavSetup $eavSetup */
        $eavSetup = $this->eavSetupFactory->create(['setup' => $this->moduleDataSetup]);

        $entityTypeId = $eavSetup->getEntityTypeId(Product::ENTITY);
        $defaultSetId = $eavSetup->getDefaultAttributeSetId($entityTypeId);

        // Create new attribute set based on Default set
        /** @var \Magento\Eav\Model\Entity\Attribute\Set $attributeSet */
        $attributeSet = $this->attributeSetFactory->create();
        $attributeSet->setEntityTypeId($entityTypeId);
        $attributeSet->setAttributeSetName('Electronics');
        $attributeSet->validate();
        $attributeSet->save();
        $attributeSet->initFromSkeleton($defaultSetId);
        $attributeSet->save();

        $attributeSetId = $attributeSet->getId();

        // Create custom attribute group
        $eavSetup->addAttributeGroup(
            Product::ENTITY,
            $attributeSetId,
            'Electronics Attributes',
            100 // Sort order
        );

        // Add custom attributes to the set
        $attributes = [
            'warranty_period',
            'voltage',
            'battery_type',
            'energy_rating',
        ];

        foreach ($attributes as $attributeCode) {
            $eavSetup->addAttributeToSet(
                Product::ENTITY,
                $attributeSetId,
                'Electronics Attributes', // Group name
                $attributeCode
            );
        }
    }

    public static function getDependencies(): array
    {
        return [
            AddWarrantyPeriodAttribute::class,
            // Other attribute patches
        ];
    }

    public function getAliases(): array
    {
        return [];
    }
}
```

## Flat Tables for Performance

### What Are Flat Tables?

**Flat tables** denormalize EAV data into single-table structures with one column per attribute.

**Example: `catalog_product_flat_1` (store 1):**
```sql
CREATE TABLE `catalog_product_flat_1` (
  `entity_id` INT(10) UNSIGNED NOT NULL,
  `sku` VARCHAR(64) DEFAULT NULL,
  `name` VARCHAR(255) DEFAULT NULL,
  `price` DECIMAL(12,4) DEFAULT NULL,
  `description` TEXT,
  `short_description` TEXT,
  `image` VARCHAR(255) DEFAULT NULL,
  `weight` DECIMAL(12,4) DEFAULT NULL,
  `color` INT(11) DEFAULT NULL,
  `size` INT(11) DEFAULT NULL,
  -- ... all other attributes
  PRIMARY KEY (`entity_id`),
  KEY `IDX_SKU` (`sku`),
  KEY `IDX_NAME` (`name`),
  KEY `IDX_PRICE` (`price`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**Query comparison:**

**EAV query (20 joins):**
```sql
SELECT e.entity_id, name.value, price.value, color.value
FROM catalog_product_entity e
LEFT JOIN catalog_product_entity_varchar name ON ...
LEFT JOIN catalog_product_entity_decimal price ON ...
LEFT JOIN catalog_product_entity_int color ON ...
WHERE e.entity_id = 123;
-- Execution time: 50ms
```

**Flat query (0 joins):**
```sql
SELECT entity_id, name, price, color
FROM catalog_product_flat_1
WHERE entity_id = 123;
-- Execution time: 2ms
```

**Performance gain:** 25x faster for attribute-heavy queries.

### Enabling Product Flat Tables

**Admin:**
Stores > Configuration > Catalog > Catalog > Storefront > Use Flat Catalog Product = Yes

**CLI:**
```bash
bin/magento config:set catalog/frontend/flat_catalog_product 1
bin/magento indexer:reindex catalog_product_flat
```

**Indexer:** Maintains flat tables from EAV source via `catalog_product_flat` indexer.

### When to Use Flat Tables

**Use flat tables when:**
- High read volume (category pages, search results)
- Many attributes loaded per product (20+)
- Simple attribute types (no complex backend models)

**Avoid flat tables when:**
- Few attributes loaded (< 5)
- Frequent attribute schema changes
- Custom backend models (not compatible with flat indexing)

**Magento 2.4+ note:** Product flat tables are **deprecated** and may be removed in future versions. Use layered navigation indexes instead.

## EAV vs Flat Tables for Custom Entities

### Decision Matrix

| Criteria | Use EAV | Use Flat Table |
|----------|---------|----------------|
| **Attributes** | 10+ dynamic attributes | Fixed schema, < 10 fields |
| **Schema changes** | Frequent (user-defined) | Rare (developer-controlled) |
| **Attribute types** | Mixed (text, select, date) | Mostly simple types |
| **Query pattern** | Load few attributes | Load all fields |
| **Performance priority** | Flexibility > speed | Speed > flexibility |
| **Admin UI** | Need attribute management | Static form |
| **Multi-store** | Different attributes per store | Same fields globally |

**Examples:**

**Use EAV:**
- Custom product types with user-defined attributes
- Customer profiles with dynamic fields
- Form builder with custom field types

**Use flat table:**
- Order line items (fixed fields)
- Shipment tracking (static schema)
- Custom reports (performance-critical)

## Creating Custom EAV Entity

Let's create a custom "Warranty Registration" entity with EAV architecture.

### Step 1: Define Entity Type

**Setup/Patch/Data/CreateWarrantyEntityType.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Eav\Model\Config;
use Magento\Eav\Setup\EavSetup;
use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class CreateWarrantyEntityType implements DataPatchInterface
{
    private const ENTITY_TYPE_CODE = 'warranty_registration';

    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
        private readonly EavSetupFactory $eavSetupFactory,
        private readonly Config $eavConfig
    ) {}

    public function apply(): void
    {
        /** @var EavSetup $eavSetup */
        $eavSetup = $this->eavSetupFactory->create(['setup' => $this->moduleDataSetup]);

        // Create entity type
        $eavSetup->addEntityType(
            self::ENTITY_TYPE_CODE,
            [
                'entity_model' => \Vendor\Module\Model\ResourceModel\Warranty::class,
                'table' => 'vendor_warranty_entity',
                'increment_model' => null,
                'increment_per_store' => false,
                'attribute_model' => \Magento\Eav\Model\Entity\Attribute::class,
                'entity_attribute_collection' => \Magento\Eav\Model\ResourceModel\Entity\Attribute\Collection::class,
            ]
        );

        $entityTypeId = $eavSetup->getEntityTypeId(self::ENTITY_TYPE_CODE);

        // Add attributes
        $eavSetup->addAttribute(
            self::ENTITY_TYPE_CODE,
            'customer_name',
            [
                'type' => 'varchar',
                'label' => 'Customer Name',
                'input' => 'text',
                'required' => true,
                'user_defined' => false,
            ]
        );

        $eavSetup->addAttribute(
            self::ENTITY_TYPE_CODE,
            'product_serial',
            [
                'type' => 'varchar',
                'label' => 'Product Serial Number',
                'input' => 'text',
                'required' => true,
                'user_defined' => false,
            ]
        );

        $eavSetup->addAttribute(
            self::ENTITY_TYPE_CODE,
            'purchase_date',
            [
                'type' => 'datetime',
                'label' => 'Purchase Date',
                'input' => 'date',
                'required' => true,
                'user_defined' => false,
            ]
        );

        $eavSetup->addAttribute(
            self::ENTITY_TYPE_CODE,
            'warranty_period',
            [
                'type' => 'int',
                'label' => 'Warranty Period (Months)',
                'input' => 'text',
                'required' => true,
                'default' => 12,
                'user_defined' => false,
            ]
        );
    }

    public static function getDependencies(): array
    {
        return [];
    }

    public function getAliases(): array
    {
        return [];
    }
}
```

### Step 2: Create Database Schema

**etc/db_schema.xml**

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">

    <!-- Entity Table -->
    <table name="vendor_warranty_entity" resource="default" engine="innodb" comment="Warranty Registration Entity">
        <column xsi:type="int" name="entity_id" unsigned="true" nullable="false" identity="true" comment="Entity ID"/>
        <column xsi:type="int" name="customer_id" unsigned="true" nullable="true" comment="Customer ID"/>
        <column xsi:type="int" name="product_id" unsigned="true" nullable="true" comment="Product ID"/>
        <column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP" comment="Created At"/>
        <column xsi:type="timestamp" name="updated_at" nullable="false" default="CURRENT_TIMESTAMP" on_update="true" comment="Updated At"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="entity_id"/>
        </constraint>

        <constraint xsi:type="foreign" referenceId="FK_WARRANTY_CUSTOMER"
                    table="vendor_warranty_entity" column="customer_id"
                    referenceTable="customer_entity" referenceColumn="entity_id"
                    onDelete="CASCADE"/>

        <constraint xsi:type="foreign" referenceId="FK_WARRANTY_PRODUCT"
                    table="vendor_warranty_entity" column="product_id"
                    referenceTable="catalog_product_entity" referenceColumn="entity_id"
                    onDelete="SET NULL"/>

        <index referenceId="IDX_CUSTOMER_ID" indexType="btree">
            <column name="customer_id"/>
        </index>
    </table>

    <!-- Value Tables -->
    <table name="vendor_warranty_entity_datetime" resource="default" engine="innodb" comment="Warranty Datetime Attributes">
        <column xsi:type="int" name="value_id" unsigned="true" nullable="false" identity="true" comment="Value ID"/>
        <column xsi:type="smallint" name="attribute_id" unsigned="true" nullable="false" comment="Attribute ID"/>
        <column xsi:type="int" name="entity_id" unsigned="true" nullable="false" comment="Entity ID"/>
        <column xsi:type="datetime" name="value" nullable="true" comment="Value"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="value_id"/>
        </constraint>

        <constraint xsi:type="unique" referenceId="UNQ_ENTITY_ATTRIBUTE">
            <column name="entity_id"/>
            <column name="attribute_id"/>
        </constraint>

        <constraint xsi:type="foreign" referenceId="FK_WARRANTY_DATETIME_ENTITY"
                    table="vendor_warranty_entity_datetime" column="entity_id"
                    referenceTable="vendor_warranty_entity" referenceColumn="entity_id"
                    onDelete="CASCADE"/>

        <constraint xsi:type="foreign" referenceId="FK_WARRANTY_DATETIME_ATTRIBUTE"
                    table="vendor_warranty_entity_datetime" column="attribute_id"
                    referenceTable="eav_attribute" referenceColumn="attribute_id"
                    onDelete="CASCADE"/>
    </table>

    <table name="vendor_warranty_entity_int" resource="default" engine="innodb" comment="Warranty Int Attributes">
        <column xsi:type="int" name="value_id" unsigned="true" nullable="false" identity="true" comment="Value ID"/>
        <column xsi:type="smallint" name="attribute_id" unsigned="true" nullable="false" comment="Attribute ID"/>
        <column xsi:type="int" name="entity_id" unsigned="true" nullable="false" comment="Entity ID"/>
        <column xsi:type="int" name="value" nullable="true" comment="Value"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="value_id"/>
        </constraint>

        <constraint xsi:type="unique" referenceId="UNQ_ENTITY_ATTRIBUTE">
            <column name="entity_id"/>
            <column name="attribute_id"/>
        </constraint>

        <constraint xsi:type="foreign" referenceId="FK_WARRANTY_INT_ENTITY"
                    table="vendor_warranty_entity_int" column="entity_id"
                    referenceTable="vendor_warranty_entity" referenceColumn="entity_id"
                    onDelete="CASCADE"/>

        <constraint xsi:type="foreign" referenceId="FK_WARRANTY_INT_ATTRIBUTE"
                    table="vendor_warranty_entity_int" column="attribute_id"
                    referenceTable="eav_attribute" referenceColumn="attribute_id"
                    onDelete="CASCADE"/>
    </table>

    <table name="vendor_warranty_entity_varchar" resource="default" engine="innodb" comment="Warranty Varchar Attributes">
        <column xsi:type="int" name="value_id" unsigned="true" nullable="false" identity="true" comment="Value ID"/>
        <column xsi:type="smallint" name="attribute_id" unsigned="true" nullable="false" comment="Attribute ID"/>
        <column xsi:type="int" name="entity_id" unsigned="true" nullable="false" comment="Entity ID"/>
        <column xsi:type="varchar" name="value" length="255" nullable="true" comment="Value"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="value_id"/>
        </constraint>

        <constraint xsi:type="unique" referenceId="UNQ_ENTITY_ATTRIBUTE">
            <column name="entity_id"/>
            <column name="attribute_id"/>
        </constraint>

        <constraint xsi:type="foreign" referenceId="FK_WARRANTY_VARCHAR_ENTITY"
                    table="vendor_warranty_entity_varchar" column="entity_id"
                    referenceTable="vendor_warranty_entity" referenceColumn="entity_id"
                    onDelete="CASCADE"/>

        <constraint xsi:type="foreign" referenceId="FK_WARRANTY_VARCHAR_ATTRIBUTE"
                    table="vendor_warranty_entity_varchar" column="attribute_id"
                    referenceTable="eav_attribute" referenceColumn="attribute_id"
                    onDelete="CASCADE"/>
    </table>

</schema>
```

### Step 3: Create Model and Resource Model

**Model/Warranty.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Framework\Model\AbstractModel;

class Warranty extends AbstractModel
{
    protected function _construct(): void
    {
        $this->_init(\Vendor\Module\Model\ResourceModel\Warranty::class);
    }
}
```

**Model/ResourceModel/Warranty.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\ResourceModel;

use Magento\Eav\Model\Entity\AbstractEntity;
use Magento\Eav\Model\Entity\Context;

class Warranty extends AbstractEntity
{
    public function __construct(
        Context $context,
        $data = []
    ) {
        parent::__construct($context, $data);
    }

    protected function _construct(): void
    {
        // No _init() call for EAV entities
    }

    public function getEntityType()
    {
        if (empty($this->_type)) {
            $this->setType('warranty_registration');
        }
        return parent::getEntityType();
    }
}
```

## EAV Performance Optimization

### 1. Select Only Required Attributes

**Bad (loads all attributes):**
```php
$collection = $productCollectionFactory->create();
$collection->addAttributeToSelect('*'); // 50+ joins
```

**Good (loads specific attributes):**
```php
$collection = $productCollectionFactory->create();
$collection->addAttributeToSelect(['name', 'price', 'image']); // 3 joins
```

**Performance impact:** 10-50x faster query execution.

### 2. Use Attribute Codes Instead of IDs

**Bad (requires attribute lookup):**
```php
$product->setData(73, 'New Product Name'); // Attribute ID 73
```

**Good (direct attribute code):**
```php
$product->setData('name', 'New Product Name');
```

### 3. Batch Load Attributes

**Bad (N+1 query problem):**
```php
foreach ($products as $product) {
    echo $product->getName(); // Separate query for each product
}
```

**Good (single query):**
```php
$collection->addAttributeToSelect('name'); // Loaded once for all products
foreach ($collection as $product) {
    echo $product->getName(); // Already loaded
}
```

### 4. Use Flat Catalog for Frontend

Enable flat tables for frequently accessed entities (products, categories).

### 5. Index Attribute Values

For frequently filtered attributes, ensure they're marked as `filterable` to be indexed:

```php
$attribute->setData('is_filterable', 1);
$attribute->save();
```

This creates entries in `catalog_product_index_eav` for fast filtering.

### 6. Use Direct SQL for Bulk Operations

**Bad (ORM overhead):**
```php
foreach ($productIds as $id) {
    $product = $productRepository->getById($id);
    $product->setStatus(1);
    $productRepository->save($product);
}
```

**Good (direct SQL):**
```php
$connection->update(
    'catalog_product_entity_int',
    ['value' => 1],
    [
        'entity_id IN (?)' => $productIds,
        'attribute_id = ?' => $statusAttributeId,
    ]
);
```

**Performance gain:** 100-1000x faster for bulk updates.

---

## Assumptions

- **Target versions**: Magento 2.4.7+, PHP 8.2+, MySQL 8.0+
- **Deployment**: Production environment with proper indexing
- **Catalog size**: Examples assume medium catalogs (10k-50k products)
- **Infrastructure**: Database optimization (indexes, query cache) in place

## Why This Approach

**EAV for flexibility**: Enables dynamic schema without migrations, essential for multi-tenant SaaS

**Data patches**: Declarative, idempotent attribute creation (safe for re-runs)

**Attribute sets**: Organize attributes by product type, reduce admin complexity

**Flat tables**: Trade storage for query performance in read-heavy scenarios

**Source models**: Centralize option management, enable programmatic option generation

## Security Impact

**Authorization:** Attribute management requires admin access with `Magento_Catalog::attributes` ACL

**Data validation:** Attribute values validated via frontend_class and backend_model

**SQL injection:** All EAV queries use parameter binding (safe)

**XSS prevention:** Attribute values escaped in frontend rendering

## Performance Impact

**EAV query overhead:**
- Loading 1 attribute: ~5ms overhead
- Loading 10 attributes: ~30ms overhead
- Loading 50 attributes: ~200ms overhead

**Flat table performance:**
- Category page with 50 products, 20 attributes: 2-5ms (flat) vs 200-500ms (EAV)

**Index impact:**
- Attribute changes trigger product reindex
- Large attribute updates can take 10-30 minutes for full reindex

**Core Web Vitals:**
- Optimized EAV queries improve LCP on category/product pages
- Flat catalog reduces TTFB by 50-80% on category pages

## Backward Compatibility

**API stability:**
- EAV interfaces stable across Magento 2.4.x
- Attribute management APIs stable
- Product flat tables deprecated (may be removed in 2.5+)

**Database schema:**
- EAV schema stable across versions
- Attribute migrations via data patches (safe upgrades)

**Upgrade path:**
- Magento 2.4.7 → 2.4.8: No breaking changes
- Magento 2.4.x → 2.4.9: Monitor flat table deprecation

## Tests to Add

**Unit tests:**
- Attribute source model option generation
- Attribute validation logic
- EAV entity save/load operations

**Integration tests:**
- Attribute creation via data patch
- Attribute value persistence
- Collection filtering by custom attributes
- Attribute set assignment

**Functional tests (MFTF):**
- Create product attribute in admin
- Assign attribute to attribute set
- Edit product with custom attribute
- Filter products by custom attribute in admin grid

## Documentation to Update

**Developer documentation:**
- `README.md`: EAV architecture overview
- `ATTRIBUTES.md`: Custom attribute creation guide
- `PERFORMANCE.md`: EAV optimization strategies
- `MIGRATION.md`: Converting flat tables to EAV (or vice versa)

**Admin user guide:**
- Creating product attributes
- Managing attribute sets
- Assigning attributes to products

## Related Documentation

### Related Guides

- [Declarative Schema & Data Patches: Modern Database Management in Magento 2](../tutorials/declarative-schema-data-patches.md)
- [Product Types Development](../tutorials/product-types.md)
- [Indexer System Deep Dive: Understanding Magento 2's Data Indexing Architecture](indexer-system.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Customer Overview](../../modules/customer/README.md)
