---
title: "Product Types Development"
description: "Deep dive into Magento product type architecture, custom product type creation, price calculation, stock management, and admin/frontend rendering."
type: "tutorial"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "60 minutes"
topics:
  - product types
  - price calculation
  - stock management
  - admin ui
  - frontend rendering
last_updated: "2026-02-07"
---

# Product Types Development

## Overview

Magento's product type architecture provides a flexible framework for representing different kinds of products (simple, configurable, bundle, grouped, downloadable, virtual). Understanding this system is essential for creating custom product types that integrate seamlessly with catalog, cart, checkout, and order workflows.

**What you'll learn:**
- Product type architecture and the type registry system
- Creating custom product types with full lifecycle support
- Price calculation strategies and price models
- Stock management and salability checks
- Admin UI integration with product forms and data providers
- Frontend rendering and add-to-cart customization
- Cart item representation and order item handling

**Prerequisites:**
- Magento 2.4.7+ (Adobe Commerce or Open Source)
- Advanced understanding of Magento module structure
- Experience with dependency injection and service contracts
- Knowledge of UI components and catalog architecture
- Familiarity with EAV attributes and product data management

---

## Product Type Architecture

### Core Components

Magento's product type system consists of:

1. **Type Model**: `Magento\Catalog\Model\Product\Type\AbstractType` - Core logic for product type
2. **Price Model**: `Magento\Catalog\Model\Product\Type\Price` - Price calculation
3. **Type Registry**: `Magento\Catalog\Model\Product\Type` - Manages available types
4. **Type Configuration**: `catalog_attributes.xml` and `product_types.xml`
5. **UI Components**: Admin form configuration and data providers
6. **Frontend Blocks**: Product detail and list page rendering

### Product Type Lifecycle

```
[Product Creation] → [Attribute Assignment] → [Data Validation]
       ↓
[Price Calculation] → [Salability Check] → [Add to Cart]
       ↓
[Cart Item Representation] → [Checkout] → [Order Creation]
       ↓
[Order Item Processing] → [Fulfillment] → [Invoicing]
```

### Type Registration Flow

```
product_types.xml → Type Registry → Factory → Type Model Instance
       ↓
Price Model Assignment → Option Processing → Validation Rules
```

---

## Creating a Custom Product Type

### Use Case: Rental Product Type

We'll create a "rental" product type that supports time-based pricing and availability management.

### Step 1: Declare Product Type

**File**: `Vendor/RentalProduct/etc/product_types.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Catalog:etc/product_types.xsd">

    <type name="rental" label="Rental Product" modelInstance="Vendor\RentalProduct\Model\Product\Type\Rental"
          indexPriority="60" sortOrder="80" isQty="true">
        <!-- Composable type configuration -->
        <priceModel instance="Vendor\RentalProduct\Model\Product\Type\Rental\Price"/>

        <!-- Allowed selection types -->
        <allowedSelectionTypes>
            <type name="simple"/>
        </allowedSelectionTypes>

        <!-- Stock indexer configuration -->
        <stockIndexerModel>Magento\CatalogInventory\Model\Indexer\Stock\Processor</stockIndexerModel>

        <!-- Custom attributes used by this type -->
        <customAttributes>
            <attribute name="rental_duration" retrieve="true"/>
            <attribute name="rental_deposit" retrieve="true"/>
            <attribute name="rental_late_fee" retrieve="true"/>
        </customAttributes>
    </type>
</config>
```

### Step 2: Configure Attributes for Type

**File**: `Vendor/RentalProduct/etc/catalog_attributes.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Catalog:etc/catalog_attributes.xsd">

    <!-- Attributes applicable to rental products -->
    <group name="rental">
        <attribute name="name"/>
        <attribute name="sku"/>
        <attribute name="price"/>
        <attribute name="weight"/>
        <attribute name="status"/>
        <attribute name="visibility"/>
        <attribute name="description"/>
        <attribute name="short_description"/>
        <attribute name="rental_duration"/>
        <attribute name="rental_deposit"/>
        <attribute name="rental_late_fee"/>
        <attribute name="rental_max_days"/>
        <attribute name="rental_min_days"/>
    </group>
</config>
```

### Step 3: Create Type Model

**File**: `Vendor/RentalProduct/Model/Product/Type/Rental.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\RentalProduct\Model\Product\Type;

use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Catalog\Model\Product;
use Magento\Catalog\Model\Product\Type\AbstractType;
use Magento\Framework\DataObject;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Serialize\Serializer\Json;

class Rental extends AbstractType
{
    public const TYPE_ID = 'rental';

    /**
     * Delete data specific for rental product type
     */
    public function deleteTypeSpecificData(Product $product): void
    {
        // Clean up rental-specific tables if any
    }

    /**
     * Check if product is available for sale
     *
     * @param Product $product
     * @return bool
     */
    public function isSalable($product): bool
    {
        $salable = parent::isSalable($product);

        if (!$salable) {
            return false;
        }

        // Check rental-specific availability
        $rentalAvailability = $product->getData('rental_availability');
        if ($rentalAvailability === null) {
            return true;
        }

        return (bool)$rentalAvailability;
    }

    /**
     * Check if product has required options
     *
     * @param Product $product
     * @return bool
     */
    public function hasRequiredOptions($product): bool
    {
        // Rental products require start/end date selection
        return true;
    }

    /**
     * Prepare product for cart
     *
     * @param DataObject $buyRequest
     * @param Product $product
     * @param string $processMode
     * @return array|string
     * @throws LocalizedException
     */
    protected function _prepareProduct(DataObject $buyRequest, $product, $processMode)
    {
        $result = parent::_prepareProduct($buyRequest, $product, $processMode);

        if (is_string($result)) {
            return $result;
        }

        // Validate rental dates
        $rentalStart = $buyRequest->getData('rental_start_date');
        $rentalEnd = $buyRequest->getData('rental_end_date');

        if (!$rentalStart || !$rentalEnd) {
            return __('Please select rental start and end dates.')->render();
        }

        // Validate date range
        $startDate = new \DateTime($rentalStart);
        $endDate = new \DateTime($rentalEnd);
        $days = $endDate->diff($startDate)->days;

        if ($days < 1) {
            return __('Rental period must be at least 1 day.')->render();
        }

        $minDays = (int)$product->getData('rental_min_days') ?: 1;
        $maxDays = (int)$product->getData('rental_max_days') ?: 365;

        if ($days < $minDays) {
            return __('Minimum rental period is %1 days.', $minDays)->render();
        }

        if ($days > $maxDays) {
            return __('Maximum rental period is %1 days.', $maxDays)->render();
        }

        // Check availability for date range
        if (!$this->isAvailableForDateRange($product, $startDate, $endDate)) {
            return __('Product is not available for selected dates.')->render();
        }

        // Add rental info to product
        $product->addCustomOption('rental_start_date', $rentalStart);
        $product->addCustomOption('rental_end_date', $rentalEnd);
        $product->addCustomOption('rental_days', (string)$days);

        return $result;
    }

    /**
     * Check product availability for date range
     */
    private function isAvailableForDateRange(
        Product $product,
        \DateTime $startDate,
        \DateTime $endDate
    ): bool {
        // Implement availability calendar check
        // Query rental_availability table or calendar service
        return true; // Simplified for example
    }

    /**
     * Process product configuration
     *
     * @param Product $product
     * @return Product
     */
    public function processConfiguration($product): Product
    {
        // Process rental-specific configuration
        return $product;
    }

    /**
     * Get product options
     *
     * @param Product $product
     * @return array
     */
    public function getOrderOptions($product): array
    {
        $options = parent::getOrderOptions($product);

        $rentalOptions = [
            'rental_start_date' => $product->getCustomOption('rental_start_date'),
            'rental_end_date' => $product->getCustomOption('rental_end_date'),
            'rental_days' => $product->getCustomOption('rental_days'),
        ];

        $options['rental_options'] = array_filter($rentalOptions);

        return $options;
    }

    /**
     * Check if product can be configured
     *
     * @param Product $product
     * @return bool
     */
    public function canConfigure($product): bool
    {
        return true;
    }

    /**
     * Prepare selected options for rental product
     *
     * @param Product $product
     * @param DataObject $buyRequest
     * @return array
     */
    public function processBuyRequest($product, $buyRequest): array
    {
        $rentalOptions = [];

        if ($buyRequest->hasData('rental_start_date')) {
            $rentalOptions['rental_start_date'] = $buyRequest->getData('rental_start_date');
        }

        if ($buyRequest->hasData('rental_end_date')) {
            $rentalOptions['rental_end_date'] = $buyRequest->getData('rental_end_date');
        }

        return $rentalOptions;
    }

    /**
     * Check if product is a virtual product
     */
    public function isVirtual($product): bool
    {
        return false; // Rentals are physical products
    }

    /**
     * Get child products
     */
    public function getChildrenIds($productId, $required = true): array
    {
        return [[]]; // No children for rental products
    }

    /**
     * Retrieve products divided into groups required to purchase
     */
    public function getProductsToPurchaseByReqGroups($product): array
    {
        return [[$product]];
    }
}
```

---

## Price Calculation

### Custom Price Model

**File**: `Vendor/RentalProduct/Model/Product/Type/Rental/Price.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\RentalProduct\Model\Product\Type\Rental;

use Magento\Catalog\Model\Product;
use Magento\Catalog\Model\Product\Type\Price as BasePrice;
use Magento\Framework\Pricing\PriceCurrencyInterface;

class Price extends BasePrice
{
    public function __construct(
        \Magento\CatalogRule\Model\ResourceModel\RuleFactory $ruleFactory,
        \Magento\Store\Model\StoreManagerInterface $storeManager,
        \Magento\Framework\Stdlib\DateTime\TimezoneInterface $localeDate,
        \Magento\Customer\Model\Session $customerSession,
        \Magento\Framework\Event\ManagerInterface $eventManager,
        PriceCurrencyInterface $priceCurrency,
        \Magento\Customer\Api\GroupManagementInterface $groupManagement,
        \Magento\Catalog\Api\Data\ProductTierPriceInterfaceFactory $tierPriceFactory,
        \Magento\Framework\App\Config\ScopeConfigInterface $config,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {
        parent::__construct(
            $ruleFactory,
            $storeManager,
            $localeDate,
            $customerSession,
            $eventManager,
            $priceCurrency,
            $groupManagement,
            $tierPriceFactory,
            $config
        );
    }

    /**
     * Get product final price
     *
     * @param float|null $qty
     * @param Product $product
     * @return float
     */
    public function getFinalPrice($qty, $product): float
    {
        if ($qty === null && $product->getCalculatedFinalPrice() !== null) {
            return (float)$product->getCalculatedFinalPrice();
        }

        $basePrice = $this->getBasePrice($product, $qty);
        $finalPrice = $this->calculateRentalPrice($basePrice, $product);

        $product->setFinalPrice($finalPrice);

        $this->_eventManager->dispatch('catalog_product_get_final_price', ['product' => $product, 'qty' => $qty]);

        $finalPrice = (float)$product->getData('final_price');
        $finalPrice = $this->_applyOptionsPrice($product, $qty, $finalPrice);
        $finalPrice = max(0, $finalPrice);

        $product->setFinalPrice($finalPrice);

        return $finalPrice;
    }

    /**
     * Calculate rental price based on duration and deposit
     */
    private function calculateRentalPrice(float $basePrice, Product $product): float
    {
        $rentalDays = $this->getRentalDays($product);

        if ($rentalDays === null) {
            // No rental period selected yet, return daily rate
            return $basePrice;
        }

        // Calculate rental fee
        $rentalFee = $basePrice * $rentalDays;

        // Add deposit if configured
        $deposit = (float)$product->getData('rental_deposit');
        if ($deposit > 0) {
            $rentalFee += $deposit;
        }

        // Apply bulk discounts for longer rentals
        $discount = $this->calculateBulkDiscount($rentalDays);
        if ($discount > 0) {
            $rentalFee = $rentalFee * (1 - $discount);
        }

        return $rentalFee;
    }

    /**
     * Get rental days from product options
     */
    private function getRentalDays(Product $product): ?int
    {
        $customOption = $product->getCustomOption('rental_days');

        if ($customOption) {
            return (int)$customOption->getValue();
        }

        // Check if rental days in additional options
        $additionalOptions = $product->getCustomOption('additional_options');
        if ($additionalOptions) {
            $options = $this->serializer->unserialize($additionalOptions->getValue());
            if (isset($options['rental_days'])) {
                return (int)$options['rental_days'];
            }
        }

        return null;
    }

    /**
     * Calculate bulk discount based on rental duration
     */
    private function calculateBulkDiscount(int $days): float
    {
        // Example: 10% off for 7+ days, 20% off for 30+ days
        if ($days >= 30) {
            return 0.20;
        } elseif ($days >= 7) {
            return 0.10;
        }

        return 0.0;
    }

    /**
     * Get product tier price by qty
     */
    public function getTierPrice($qty, $product): float
    {
        // Rental products don't use traditional tier pricing
        return $this->getBasePrice($product, $qty);
    }

    /**
     * Calculate deposit amount
     */
    public function getDepositAmount(Product $product): float
    {
        return (float)$product->getData('rental_deposit');
    }

    /**
     * Calculate late fee per day
     */
    public function getLateFeePerDay(Product $product): float
    {
        return (float)$product->getData('rental_late_fee');
    }
}
```

---

## Stock Management

### Custom Stock Status

**File**: `Vendor/RentalProduct/Model/ResourceModel/Product/Indexer/Price/Rental.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\RentalProduct\Model\ResourceModel\Product\Indexer\Price;

use Magento\Catalog\Model\ResourceModel\Product\Indexer\Price\DefaultPrice;

class Rental extends DefaultPrice
{
    /**
     * Prepare rental product prices
     */
    protected function _prepareDefaultFinalPriceTable(): self
    {
        $this->_prepareWebsiteDateTable();

        $connection = $this->getConnection();
        $select = $connection->select()
            ->from(['e' => $this->getTable('catalog_product_entity')], ['entity_id'])
            ->join(
                ['cg' => $this->getTable('customer_group')],
                '',
                ['customer_group_id']
            )
            ->join(
                ['cw' => $this->getTable('store_website')],
                '',
                ['website_id']
            )
            ->join(
                ['cwd' => $this->_getWebsiteDateTable()],
                'cw.website_id = cwd.website_id',
                []
            )
            ->join(
                ['csg' => $this->getTable('store_group')],
                'csg.website_id = cw.website_id AND cw.default_group_id = csg.group_id',
                []
            )
            ->join(
                ['cs' => $this->getTable('store')],
                'csg.default_store_id = cs.store_id AND cs.store_id != 0',
                []
            )
            ->where('e.type_id = ?', 'rental');

        // Add price attribute
        $this->_addAttributeToSelect($select, 'price', 'e.entity_id', 'cs.store_id', 0);

        // Add rental deposit
        $this->_addAttributeToSelect($select, 'rental_deposit', 'e.entity_id', 'cs.store_id', 0, 'rental_deposit');

        $select->columns([
            'orig_price' => new \Zend_Db_Expr('price.value'),
            'price' => new \Zend_Db_Expr('price.value'),
            'min_price' => new \Zend_Db_Expr('price.value'),
            'max_price' => new \Zend_Db_Expr('price.value + COALESCE(rental_deposit.value, 0)'),
            'tier_price' => new \Zend_Db_Expr('NULL'),
        ]);

        $connection->query(
            $connection->insertFromSelect(
                $select,
                $this->_getDefaultFinalPriceTable(),
                [],
                \Magento\Framework\DB\Adapter\AdapterInterface::INSERT_ON_DUPLICATE
            )
        );

        return $this;
    }
}
```

### Salability Check

**File**: `Vendor/RentalProduct/Plugin/InventorySales/IsSalableExtend.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\RentalProduct\Plugin\InventorySales;

use Magento\InventorySalesApi\Api\IsProductSalableInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Vendor\RentalProduct\Model\RentalAvailabilityChecker;

class IsSalableExtend
{
    public function __construct(
        private readonly ProductRepositoryInterface $productRepository,
        private readonly RentalAvailabilityChecker $availabilityChecker
    ) {
    }

    /**
     * Add rental availability check
     */
    public function afterExecute(
        IsProductSalableInterface $subject,
        bool $result,
        string $sku,
        int $stockId
    ): bool {
        if (!$result) {
            return false;
        }

        try {
            $product = $this->productRepository->get($sku);

            if ($product->getTypeId() !== 'rental') {
                return $result;
            }

            // Check if rental has available inventory for any future dates
            return $this->availabilityChecker->hasAvailability($product);

        } catch (\Exception $e) {
            return $result;
        }
    }
}
```

---

## Admin UI Configuration

### Product Form UI Component

**File**: `Vendor/RentalProduct/view/adminhtml/ui_component/product_form.xml`

```xml
<?xml version="1.0"?>
<form xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Ui:etc/ui_configuration.xsd">

    <!-- Rental Information Section -->
    <fieldset name="rental-information" sortOrder="20">
        <settings>
            <label translate="true">Rental Information</label>
            <collapsible>true</collapsible>
            <opened>true</opened>
            <visible>true</visible>
        </settings>

        <field name="rental_duration" formElement="select" sortOrder="10">
            <settings>
                <validation>
                    <rule name="required-entry" xsi:type="boolean">true</rule>
                </validation>
                <dataType>text</dataType>
                <label translate="true">Rental Duration Type</label>
                <dataScope>rental_duration</dataScope>
            </settings>
            <formElements>
                <select>
                    <settings>
                        <options class="Vendor\RentalProduct\Model\Config\Source\DurationType"/>
                    </settings>
                </select>
            </formElements>
        </field>

        <field name="rental_deposit" formElement="input" sortOrder="20">
            <settings>
                <validation>
                    <rule name="validate-number" xsi:type="boolean">true</rule>
                    <rule name="validate-zero-or-greater" xsi:type="boolean">true</rule>
                </validation>
                <dataType>number</dataType>
                <label translate="true">Security Deposit</label>
                <dataScope>rental_deposit</dataScope>
            </settings>
        </field>

        <field name="rental_late_fee" formElement="input" sortOrder="30">
            <settings>
                <validation>
                    <rule name="validate-number" xsi:type="boolean">true</rule>
                    <rule name="validate-zero-or-greater" xsi:type="boolean">true</rule>
                </validation>
                <dataType>number</dataType>
                <label translate="true">Late Fee (per day)</label>
                <dataScope>rental_late_fee</dataScope>
                <tooltip>
                    <description translate="true">Fee charged per day when rental is returned late</description>
                </tooltip>
            </settings>
        </field>

        <field name="rental_min_days" formElement="input" sortOrder="40">
            <settings>
                <validation>
                    <rule name="validate-number" xsi:type="boolean">true</rule>
                    <rule name="validate-greater-than-zero" xsi:type="boolean">true</rule>
                </validation>
                <dataType>number</dataType>
                <label translate="true">Minimum Rental Days</label>
                <dataScope>rental_min_days</dataScope>
            </settings>
        </field>

        <field name="rental_max_days" formElement="input" sortOrder="50">
            <settings>
                <validation>
                    <rule name="validate-number" xsi:type="boolean">true</rule>
                    <rule name="validate-greater-than-zero" xsi:type="boolean">true</rule>
                </validation>
                <dataType>number</dataType>
                <label translate="true">Maximum Rental Days</label>
                <dataScope>rental_max_days</dataScope>
            </settings>
        </field>

        <!-- Rental Availability Calendar -->
        <container name="rental_availability_container" sortOrder="60">
            <htmlContent name="rental_availability_calendar">
                <block class="Vendor\RentalProduct\Block\Adminhtml\Product\Edit\RentalCalendar"
                       name="rental.calendar"
                       template="Vendor_RentalProduct::product/edit/rental_calendar.phtml"/>
            </htmlContent>
        </container>
    </fieldset>
</form>
```

### Data Provider Modifier

**File**: `Vendor/RentalProduct/Ui/DataProvider/Product/Form/Modifier/RentalData.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\RentalProduct\Ui\DataProvider\Product\Form\Modifier;

use Magento\Catalog\Ui\DataProvider\Product\Form\Modifier\AbstractModifier;
use Magento\Catalog\Model\Locator\LocatorInterface;

class RentalData extends AbstractModifier
{
    public function __construct(
        private readonly LocatorInterface $locator
    ) {
    }

    /**
     * Modify product data
     */
    public function modifyData(array $data): array
    {
        $product = $this->locator->getProduct();

        if ($product->getTypeId() !== 'rental') {
            return $data;
        }

        $productId = $product->getId();

        if (isset($data[$productId])) {
            // Add rental-specific data
            $data[$productId]['product']['rental_data'] = [
                'deposit' => $product->getData('rental_deposit'),
                'late_fee' => $product->getData('rental_late_fee'),
                'min_days' => $product->getData('rental_min_days'),
                'max_days' => $product->getData('rental_max_days'),
            ];
        }

        return $data;
    }

    /**
     * Modify product form meta
     */
    public function modifyMeta(array $meta): array
    {
        $product = $this->locator->getProduct();

        if ($product->getTypeId() !== 'rental') {
            return $meta;
        }

        // Show/hide fields based on rental type
        return $meta;
    }
}
```

---

## Frontend Rendering

### Product Detail Page

**File**: `Vendor/RentalProduct/view/frontend/templates/product/view/type/rental.phtml`

```php
<?php
/**
 * @var \Magento\Catalog\Block\Product\View $block
 * @var \Magento\Framework\Escaper $escaper
 */
$product = $block->getProduct();
?>

<div class="rental-options" id="rental-options-<?= (int)$product->getId() ?>">
    <div class="rental-dates">
        <div class="field rental-start-date">
            <label class="label" for="rental-start-date">
                <span><?= $escaper->escapeHtml(__('Start Date')) ?></span>
            </label>
            <div class="control">
                <input type="text"
                       id="rental-start-date"
                       name="rental_start_date"
                       class="input-text rental-datepicker"
                       data-validate='{"required":true}'
                       data-rental-min-days="<?= (int)$product->getData('rental_min_days') ?>"
                       data-rental-max-days="<?= (int)$product->getData('rental_max_days') ?>"/>
            </div>
        </div>

        <div class="field rental-end-date">
            <label class="label" for="rental-end-date">
                <span><?= $escaper->escapeHtml(__('End Date')) ?></span>
            </label>
            <div class="control">
                <input type="text"
                       id="rental-end-date"
                       name="rental_end_date"
                       class="input-text rental-datepicker"
                       data-validate='{"required":true}'/>
            </div>
        </div>
    </div>

    <div class="rental-pricing-breakdown">
        <div class="rental-daily-rate">
            <span class="label"><?= $escaper->escapeHtml(__('Daily Rate:')) ?></span>
            <span class="value" data-price-type="basePrice"><?= $block->formatCurrency($product->getFinalPrice()) ?></span>
        </div>

        <div class="rental-duration" style="display:none;">
            <span class="label"><?= $escaper->escapeHtml(__('Rental Duration:')) ?></span>
            <span class="value" id="rental-duration-display">0 days</span>
        </div>

        <div class="rental-subtotal" style="display:none;">
            <span class="label"><?= $escaper->escapeHtml(__('Rental Subtotal:')) ?></span>
            <span class="value" id="rental-subtotal-display"><?= $block->formatCurrency(0) ?></span>
        </div>

        <?php if ($product->getData('rental_deposit')): ?>
        <div class="rental-deposit">
            <span class="label"><?= $escaper->escapeHtml(__('Security Deposit:')) ?></span>
            <span class="value"><?= $block->formatCurrency($product->getData('rental_deposit')) ?></span>
        </div>
        <?php endif; ?>

        <div class="rental-total" style="display:none;">
            <span class="label"><?= $escaper->escapeHtml(__('Total Due Today:')) ?></span>
            <span class="value price" id="rental-total-display"><?= $block->formatCurrency(0) ?></span>
        </div>
    </div>
</div>

<script type="text/x-magento-init">
{
    "#rental-options-<?= (int)$product->getId() ?>": {
        "Vendor_RentalProduct/js/rental-calculator": {
            "productId": <?= (int)$product->getId() ?>,
            "dailyRate": <?= (float)$product->getFinalPrice() ?>,
            "deposit": <?= (float)$product->getData('rental_deposit') ?>,
            "minDays": <?= (int)$product->getData('rental_min_days') ?>,
            "maxDays": <?= (int)$product->getData('rental_max_days') ?>,
            "priceFormat": <?= $block->getPriceFormatJson() ?>
        }
    }
}
</script>
```

### Rental Calculator Widget

**File**: `Vendor/RentalProduct/view/frontend/web/js/rental-calculator.js`

```javascript
define([
    'jquery',
    'Magento_Catalog/js/price-utils',
    'mage/calendar',
    'mage/translate'
], function ($, priceUtils, calendar, $t) {
    'use strict';

    $.widget('vendor.rentalCalculator', {
        options: {
            productId: null,
            dailyRate: 0,
            deposit: 0,
            minDays: 1,
            maxDays: 365,
            priceFormat: {}
        },

        _create: function () {
            this._initDatepickers();
            this._bindEvents();
        },

        _initDatepickers: function () {
            var self = this;
            var minDate = new Date();
            minDate.setDate(minDate.getDate() + 1);

            this.element.find('.rental-datepicker').calendar({
                minDate: minDate,
                dateFormat: 'mm/dd/yy',
                showButtonPanel: true,
                beforeShowDay: function (date) {
                    return self._checkAvailability(date);
                }
            });
        },

        _bindEvents: function () {
            var self = this;

            this.element.find('#rental-start-date, #rental-end-date').on('change', function () {
                self._calculatePrice();
            });
        },

        _checkAvailability: function (date) {
            // AJAX call to check availability
            // Returns [true, ''] for available, [false, '', 'disabled'] for unavailable
            return [true, ''];
        },

        _calculatePrice: function () {
            var startDate = this.element.find('#rental-start-date').val();
            var endDate = this.element.find('#rental-end-date').val();

            if (!startDate || !endDate) {
                return;
            }

            var start = new Date(startDate);
            var end = new Date(endDate);
            var days = Math.ceil((end - start) / (1000 * 60 * 60 * 24));

            if (days < this.options.minDays) {
                this._showError($t('Minimum rental period is %1 days').replace('%1', this.options.minDays));
                return;
            }

            if (days > this.options.maxDays) {
                this._showError($t('Maximum rental period is %1 days').replace('%1', this.options.maxDays));
                return;
            }

            this._hideError();

            // Calculate pricing
            var rentalSubtotal = this.options.dailyRate * days;
            var discount = this._calculateDiscount(days);
            rentalSubtotal = rentalSubtotal * (1 - discount);

            var total = rentalSubtotal + this.options.deposit;

            // Update display
            this.element.find('#rental-duration-display').text(days + ' ' + $t('days'));
            this.element.find('#rental-subtotal-display').text(
                priceUtils.formatPrice(rentalSubtotal, this.options.priceFormat)
            );
            this.element.find('#rental-total-display').text(
                priceUtils.formatPrice(total, this.options.priceFormat)
            );

            this.element.find('.rental-duration, .rental-subtotal, .rental-total').show();
        },

        _calculateDiscount: function (days) {
            if (days >= 30) {
                return 0.20;
            } else if (days >= 7) {
                return 0.10;
            }
            return 0;
        },

        _showError: function (message) {
            // Show validation error
            console.error(message);
        },

        _hideError: function () {
            // Hide validation error
        }
    });

    return $.vendor.rentalCalculator;
});
```

---

## Cart and Checkout Integration

### Quote Item Processor

**File**: `Vendor/RentalProduct/Model/Quote/Item/Processor.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\RentalProduct\Model\Quote\Item;

use Magento\Quote\Model\Quote\Item;
use Magento\Framework\Serialize\Serializer\Json;

class Processor
{
    public function __construct(
        private readonly Json $serializer
    ) {
    }

    /**
     * Process rental options for cart item
     */
    public function processItem(Item $item): void
    {
        $product = $item->getProduct();

        if ($product->getTypeId() !== 'rental') {
            return;
        }

        // Get rental options
        $options = [];
        if ($rentalStart = $product->getCustomOption('rental_start_date')) {
            $options['rental_start_date'] = $rentalStart->getValue();
        }

        if ($rentalEnd = $product->getCustomOption('rental_end_date')) {
            $options['rental_end_date'] = $rentalEnd->getValue();
        }

        if ($rentalDays = $product->getCustomOption('rental_days')) {
            $options['rental_days'] = $rentalDays->getValue();
        }

        // Store in additional options for display
        if (!empty($options)) {
            $additionalOptions = [
                [
                    'label' => __('Rental Period'),
                    'value' => sprintf(
                        '%s to %s (%d days)',
                        $options['rental_start_date'],
                        $options['rental_end_date'],
                        $options['rental_days']
                    )
                ]
            ];

            $item->addOption([
                'code' => 'additional_options',
                'value' => $this->serializer->serialize($additionalOptions)
            ]);
        }
    }
}
```

---

## Testing

### Unit Test Example

**File**: `Vendor/RentalProduct/Test/Unit/Model/Product/Type/RentalTest.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\RentalProduct\Test\Unit\Model\Product\Type;

use PHPUnit\Framework\TestCase;
use Vendor\RentalProduct\Model\Product\Type\Rental;
use Magento\Catalog\Model\Product;
use Magento\Framework\DataObject;

class RentalTest extends TestCase
{
    private Rental $rentalType;

    protected function setUp(): void
    {
        $this->rentalType = $this->createPartialMock(Rental::class, []);
    }

    public function testHasRequiredOptions(): void
    {
        $product = $this->createMock(Product::class);
        $this->assertTrue($this->rentalType->hasRequiredOptions($product));
    }

    public function testPrepareProductWithMissingDates(): void
    {
        $product = $this->createMock(Product::class);
        $buyRequest = new DataObject([]);

        $result = $this->rentalType->_prepareProduct($buyRequest, $product, 'full');

        $this->assertIsString($result);
        $this->assertStringContainsString('select rental start and end dates', $result);
    }
}
```

---

## Summary

**Key Takeaways:**
- Product types extend AbstractType and require price model, type configuration
- Custom attributes must be declared in catalog_attributes.xml
- Price calculation happens in dedicated Price model with support for dynamic pricing
- Stock/salability checks integrate with MSI via plugins
- Admin UI uses UI components with data provider modifiers
- Frontend requires custom templates and JavaScript for user interaction
- Cart integration uses custom options and additional_options for display
- Order items preserve rental configuration through product options

---

## Assumptions

- **Magento Version:** 2.4.7+ (Adobe Commerce)
- **PHP Version:** 8.2+
- **MSI:** Multi-Source Inventory enabled
- **Database:** Custom tables for rental availability calendar
- **Frontend:** Luma or custom theme with jQuery and UI widgets

## Why This Approach

- **Type System:** Leverages Magento's native product type architecture
- **Price Flexibility:** Separate price model allows complex rental pricing logic
- **Admin Integration:** UI components provide native admin experience
- **Cart Compatibility:** Custom options ensure rental info flows through order workflow
- **Extensibility:** Plugins and events allow third-party customization

## Security Impact

- **Input Validation:** Date inputs validated on server side
- **Price Manipulation:** Price recalculated server-side, not trusted from client
- **Availability Checks:** Authorization required for admin availability management
- **PII:** Rental dates may be considered PII in some jurisdictions

## Performance Impact

- **FPC:** Product pages with rental calendar bypass FPC (use AJAX for calendar)
- **Price Indexing:** Custom price indexer adds overhead; optimize for rental products
- **Availability Queries:** Date range checks may be slow; use indexed availability table
- **Cart:** Minimal impact; rental options stored as serialized data

## Backward Compatibility

- **Type Registration:** New product type doesn't affect existing types
- **Database Schema:** Attributes added via setup scripts, no core modifications
- **API:** Custom type excluded from API unless explicitly added
- **Upgrade Path:** Module upgrades preserve rental product data

## Tests to Add

**Unit Tests:**
```php
testRentalTypeIsSalable()
testPriceCalculationWithDiscount()
testDateRangeValidation()
```

**Integration Tests:**
```php
testAddRentalProductToCart()
testRentalPriceIndexing()
testOrderCreationWithRentalProduct()
```

**MFTF:**
```xml
<test name="AdminCreateRentalProductTest">
<test name="StorefrontAddRentalToCartTest">
```

## Docs to Update

- **README.md:** Installation, rental type usage, configuration
- **docs/PRODUCT_TYPE.md:** Architecture diagram, extension points
- **docs/PRICING.md:** Price calculation formulas, discount rules
- **Admin User Guide:** Screenshots of rental product creation workflow

## Related Documentation

### Related Guides

- [EAV System Architecture: Understanding Entity-Attribute-Value in Magento 2](../explanations/eav-system.md)
- [Declarative Schema & Data Patches: Modern Database Management in Magento 2](declarative-schema-data-patches.md)
- [Import/Export System Deep Dive](../how-to/import-export.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
