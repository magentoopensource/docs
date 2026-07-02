---
title: "Custom Shipping Method Development: Building Flexible Shipping Solutions"
description: "Complete guide to developing custom shipping methods in Magento 2: carrier model implementation, rate calculation logic, tracking integration, multi-warehouse support, shipping restrictions, and testing strategies"
type: "tutorial"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "75 minutes"
topics:
  - shipping-method
  - carrier
  - rate-calculation
  - tracking
  - checkout
  - multi-warehouse
  - service-contracts
  - testing
last_updated: "2026-02-07"
---

# Custom Shipping Method Development: Building Flexible Shipping Solutions

## Learning Objectives

By completing this tutorial, you will:

- Understand Magento 2's shipping carrier architecture and rate request flow
- Implement custom carrier models with flexible rate calculation logic
- Integrate third-party shipping APIs for real-time rate quotes
- Build tracking number integration for shipment visibility
- Implement multi-warehouse and origin-based shipping calculations
- Handle shipping restrictions (weight, dimensions, destinations)
- Optimize shipping method performance and caching strategies
- Write comprehensive tests for shipping methods
- Deploy shipping methods to production with proper configuration

## Introduction

Shipping methods are critical components of e-commerce functionality. Magento 2's shipping architecture provides a flexible framework for implementing both simple table-based rates and complex API-driven shipping calculations.

### Shipping Architecture Overview

**Core components:**
1. **Carrier Model**: Implements rate calculation and tracking logic
2. **Rate Request**: Contains cart/quote data for rate calculation
3. **Rate Result**: Collection of shipping rates returned to customer
4. **Tracking**: Links tracking numbers to shipments
5. **Configuration**: Admin settings for carrier setup

**Shipping flow:**
```
Cart → Rate Request → Carrier::collectRates() → Rate Calculation → Rate Result → Customer Selection → Order → Shipment → Tracking
```

### When to Build a Custom Shipping Method

**Use cases for custom shipping methods:**
- Integration with regional/local carriers not supported by Magento
- Complex business rules (volume discounts, customer groups, product attributes)
- Multi-warehouse shipping with proximity-based rate calculation
- Custom handling fees, insurance, signature requirements
- Specialized freight or white-glove delivery services
- In-store pickup with location selection

**Alternatives to consider:**
- **Table Rates**: Use Magento's built-in table rate method for simple weight/price-based rates
- **Third-party extensions**: Evaluate marketplace extensions before building custom
- **Shipping aggregators**: Use services like ShipStation, Shippo for multi-carrier support

## Carrier Model Structure

### Module Structure

```
Vendor/Shipping/
├── registration.php
├── composer.json
├── etc/
│   ├── module.xml
│   ├── config.xml                    # Default carrier configuration
│   ├── adminhtml/
│   │   └── system.xml                # Admin configuration fields
│   └── di.xml                        # Service contracts, preferences
├── Model/
│   ├── Carrier/
│   │   └── CustomCarrier.php         # Main carrier implementation
│   ├── Config/
│   │   └── Source/
│   │       ├── Method.php            # Shipping method options
│   │       └── PackageSize.php       # Package size options
│   ├── RateCalculator.php            # Rate calculation service
│   ├── TrackingAdapter.php           # Tracking API adapter
│   └── ResourceModel/
│       └── Rate/
│           ├── Collection.php        # Rate table collection
│           └── Rate.php              # Rate table resource
├── Observer/
│   └── RestrictShippingObserver.php  # Shipping method restrictions
├── Plugin/
│   └── AddShippingInformationExtend.php  # Extend shipping info
└── view/
    ├── frontend/
    │   ├── layout/
    │   │   └── checkout_cart_index.xml
    │   └── templates/
    │       └── carrier/
    │           └── additional_info.phtml
    └── adminhtml/
        └── templates/
            └── system/
                └── config/
                    └── rate_table.phtml
```

### Module Registration and Dependencies

**registration.php**

```php
<?php
declare(strict_types=1);

use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Vendor_Shipping',
    __DIR__
);
```

**composer.json**

```json
{
    "name": "vendor/magento2-custom-shipping",
    "description": "Custom shipping carrier for Magento 2",
    "type": "magento2-module",
    "version": "1.0.0",
    "license": "proprietary",
    "require": {
        "php": "~8.2.0||~8.3.0",
        "magento/framework": "^103.0",
        "magento/module-shipping": "^100.4",
        "magento/module-quote": "^101.2",
        "magento/module-sales": "^103.0",
        "magento/module-catalog": "^104.0",
        "magento/module-store": "^101.1",
        "guzzlehttp/guzzle": "^7.5"
    },
    "autoload": {
        "files": [
            "registration.php"
        ],
        "psr-4": {
            "Vendor\\Shipping\\": ""
        }
    }
}
```

**etc/module.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Vendor_Shipping">
        <sequence>
            <module name="Magento_Shipping"/>
            <module name="Magento_Quote"/>
            <module name="Magento_Sales"/>
            <module name="Magento_Catalog"/>
        </sequence>
    </module>
</config>
```

### Carrier Configuration

**etc/config.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Store:etc/config.xsd">
    <default>
        <carriers>
            <customcarrier>
                <active>0</active>
                <model>Vendor\Shipping\Model\Carrier\CustomCarrier</model>
                <name>Custom Carrier</name>
                <title>Custom Shipping</title>
                <type>I</type><!-- I = Individual, O = Online -->
                <sallowspecific>0</sallowspecific>
                <specificerrmsg>This shipping method is not available. Please contact us for alternative options.</specificerrmsg>
                <sort_order>10</sort_order>

                <!-- Rate calculation settings -->
                <base_rate>5.00</base_rate>
                <per_item_rate>2.00</per_item_rate>
                <per_weight_rate>1.50</per_weight_rate>
                <handling_type>F</handling_type><!-- F = Fixed, P = Percent -->
                <handling_fee>0</handling_fee>

                <!-- API settings -->
                <api_endpoint>https://api.carrier.com/v1</api_endpoint>
                <api_key backend_model="Magento\Config\Model\Config\Backend\Encrypted"/>

                <!-- Restrictions -->
                <max_package_weight>50</max_package_weight>
                <free_shipping_threshold>100</free_shipping_threshold>

                <!-- Delivery time -->
                <estimated_delivery_days>3-5</estimated_delivery_days>
            </customcarrier>
        </carriers>
    </default>
</config>
```

**Configuration fields explained:**

- `model`: Carrier class implementing `\Magento\Shipping\Model\Carrier\CarrierInterface`
- `type`: `I` (individual rates) or `O` (online API rates)
- `sallowspecific`: Allow shipping to specific countries only
- `handling_type`: `F` (fixed fee) or `P` (percentage)
- `free_shipping_threshold`: Order subtotal for free shipping

### Admin Configuration

**etc/adminhtml/system.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system_file.xsd">
    <system>
        <section id="carriers">
            <group id="customcarrier" translate="label" type="text" sortOrder="100" showInDefault="1" showInWebsite="1" showInStore="1">
                <label>Custom Carrier</label>

                <field id="active" translate="label" type="select" sortOrder="10" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Enabled</label>
                    <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
                </field>

                <field id="title" translate="label" type="text" sortOrder="20" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Title</label>
                </field>

                <field id="name" translate="label" type="text" sortOrder="30" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Method Name</label>
                </field>

                <!-- Rate Calculation -->
                <field id="rate_calculation_mode" translate="label" type="select" sortOrder="40" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Rate Calculation Mode</label>
                    <source_model>Vendor\Shipping\Model\Config\Source\CalculationMode</source_model>
                    <comment>Choose between fixed rates, API-based rates, or custom table rates</comment>
                </field>

                <field id="base_rate" translate="label" type="text" sortOrder="50" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Base Rate</label>
                    <validate>validate-number validate-zero-or-greater</validate>
                    <depends>
                        <field id="rate_calculation_mode">fixed</field>
                    </depends>
                </field>

                <field id="per_item_rate" translate="label" type="text" sortOrder="60" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Rate Per Item</label>
                    <validate>validate-number validate-zero-or-greater</validate>
                    <depends>
                        <field id="rate_calculation_mode">fixed</field>
                    </depends>
                </field>

                <field id="per_weight_rate" translate="label" type="text" sortOrder="70" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Rate Per Weight Unit (lbs)</label>
                    <validate>validate-number validate-zero-or-greater</validate>
                    <depends>
                        <field id="rate_calculation_mode">fixed</field>
                    </depends>
                </field>

                <field id="handling_type" translate="label" type="select" sortOrder="80" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Handling Fee Type</label>
                    <source_model>Magento\Shipping\Model\Source\HandlingType</source_model>
                </field>

                <field id="handling_fee" translate="label" type="text" sortOrder="90" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Handling Fee</label>
                    <validate>validate-number validate-zero-or-greater</validate>
                </field>

                <!-- API Configuration -->
                <field id="api_endpoint" translate="label" type="select" sortOrder="100" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>API Endpoint</label>
                    <source_model>Vendor\Shipping\Model\Config\Source\Environment</source_model>
                    <depends>
                        <field id="rate_calculation_mode">api</field>
                    </depends>
                </field>

                <field id="api_key" translate="label" type="obscure" sortOrder="110" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>API Key</label>
                    <backend_model>Magento\Config\Model\Config\Backend\Encrypted</backend_model>
                    <depends>
                        <field id="rate_calculation_mode">api</field>
                    </depends>
                </field>

                <!-- Restrictions -->
                <field id="max_package_weight" translate="label" type="text" sortOrder="120" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Maximum Package Weight (lbs)</label>
                    <validate>validate-number validate-greater-than-zero</validate>
                    <comment>Orders exceeding this weight will not show this shipping method</comment>
                </field>

                <field id="free_shipping_threshold" translate="label" type="text" sortOrder="130" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Free Shipping Threshold</label>
                    <validate>validate-number validate-zero-or-greater</validate>
                    <comment>Order subtotal above which shipping is free (0 to disable)</comment>
                </field>

                <field id="estimated_delivery_days" translate="label" type="text" sortOrder="140" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Estimated Delivery Time</label>
                    <comment>Display estimated delivery time to customers (e.g., "3-5 business days")</comment>
                </field>

                <!-- Applicable Countries -->
                <field id="sallowspecific" translate="label" type="select" sortOrder="150" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Ship to Applicable Countries</label>
                    <frontend_class>shipping-applicable-country</frontend_class>
                    <source_model>Magento\Shipping\Model\Config\Source\Allspecificcountries</source_model>
                </field>

                <field id="specificcountry" translate="label" type="multiselect" sortOrder="160" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Ship to Specific Countries</label>
                    <source_model>Magento\Directory\Model\Config\Source\Country</source_model>
                    <can_be_empty>1</can_be_empty>
                    <depends>
                        <field id="sallowspecific">1</field>
                    </depends>
                </field>

                <field id="showmethod" translate="label" type="select" sortOrder="170" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Show Method if Not Applicable</label>
                    <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
                </field>

                <field id="specificerrmsg" translate="label" type="textarea" sortOrder="180" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Error Message</label>
                    <comment>Displayed when shipping method is not available</comment>
                </field>

                <field id="sort_order" translate="label" type="text" sortOrder="190" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Sort Order</label>
                </field>

            </group>
        </section>
    </system>
</config>
```

## Carrier Model Implementation

The carrier model is the heart of the shipping method, implementing rate collection and tracking functionality.

**Model/Carrier/CustomCarrier.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Shipping\Model\Carrier;

use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Framework\DataObject;
use Magento\Quote\Model\Quote\Address\RateRequest;
use Magento\Quote\Model\Quote\Address\RateResult\ErrorFactory;
use Magento\Quote\Model\Quote\Address\RateResult\Method;
use Magento\Quote\Model\Quote\Address\RateResult\MethodFactory;
use Magento\Shipping\Model\Carrier\AbstractCarrier;
use Magento\Shipping\Model\Carrier\CarrierInterface;
use Magento\Shipping\Model\Rate\Result;
use Magento\Shipping\Model\Rate\ResultFactory;
use Magento\Shipping\Model\Tracking\Result as TrackingResult;
use Magento\Shipping\Model\Tracking\Result\StatusFactory;
use Psr\Log\LoggerInterface;

/**
 * Custom shipping carrier
 */
class CustomCarrier extends AbstractCarrier implements CarrierInterface
{
    protected $_code = 'customcarrier';

    protected $_isFixed = true; // Set false for real-time API rates

    public function __construct(
        ScopeConfigInterface $scopeConfig,
        ErrorFactory $rateErrorFactory,
        LoggerInterface $logger,
        private readonly ResultFactory $rateResultFactory,
        private readonly MethodFactory $rateMethodFactory,
        private readonly StatusFactory $trackStatusFactory,
        array $data = []
    ) {
        parent::__construct($scopeConfig, $rateErrorFactory, $logger, $data);
    }

    /**
     * Collect shipping rates
     *
     * @param RateRequest $request
     * @return Result|bool
     */
    public function collectRates(RateRequest $request)
    {
        if (!$this->isActive()) {
            return false;
        }

        // Check if shipping to this destination is allowed
        if (!$this->isDestinationAllowed($request)) {
            return $this->getErrorMessage();
        }

        // Check weight restrictions
        if (!$this->isWeightAllowed($request)) {
            return $this->getErrorMessage();
        }

        /** @var Result $result */
        $result = $this->rateResultFactory->create();

        // Calculate shipping rate
        $shippingPrice = $this->calculateShippingPrice($request);

        if ($shippingPrice !== false) {
            $method = $this->createRateMethod($shippingPrice);
            $result->append($method);
        } else {
            return $this->getErrorMessage();
        }

        return $result;
    }

    /**
     * Check if carrier is active
     */
    private function isActive(): bool
    {
        return $this->getConfigFlag('active');
    }

    /**
     * Check if destination country is allowed
     */
    private function isDestinationAllowed(RateRequest $request): bool
    {
        $allowedCountries = $this->getConfigData('specificcountry');

        if ($this->getConfigData('sallowspecific') && $allowedCountries) {
            $allowedCountries = explode(',', $allowedCountries);
            return in_array($request->getDestCountryId(), $allowedCountries, true);
        }

        return true;
    }

    /**
     * Check if package weight is within limits
     */
    private function isWeightAllowed(RateRequest $request): bool
    {
        $maxWeight = (float) $this->getConfigData('max_package_weight');

        if ($maxWeight > 0 && $request->getPackageWeight() > $maxWeight) {
            return false;
        }

        return true;
    }

    /**
     * Calculate shipping price based on configuration
     */
    private function calculateShippingPrice(RateRequest $request): float|false
    {
        $calculationMode = $this->getConfigData('rate_calculation_mode') ?: 'fixed';

        return match ($calculationMode) {
            'fixed' => $this->calculateFixedRate($request),
            'api' => $this->calculateApiRate($request),
            'table' => $this->calculateTableRate($request),
            default => false,
        };
    }

    /**
     * Calculate fixed rate based on weight and item count
     */
    private function calculateFixedRate(RateRequest $request): float
    {
        $baseRate = (float) $this->getConfigData('base_rate');
        $perItemRate = (float) $this->getConfigData('per_item_rate');
        $perWeightRate = (float) $this->getConfigData('per_weight_rate');

        $itemCount = 0;
        if ($request->getAllItems()) {
            foreach ($request->getAllItems() as $item) {
                if ($item->getProduct()->isVirtual() || $item->getParentItem()) {
                    continue;
                }
                $itemCount += $item->getQty();
            }
        }

        $price = $baseRate
            + ($perItemRate * $itemCount)
            + ($perWeightRate * $request->getPackageWeight());

        // Apply handling fee
        $price = $this->applyHandlingFee($price);

        // Check for free shipping threshold
        $freeShippingThreshold = (float) $this->getConfigData('free_shipping_threshold');
        if ($freeShippingThreshold > 0 && $request->getPackageValue() >= $freeShippingThreshold) {
            return 0.00;
        }

        return max(0.00, $price);
    }

    /**
     * Calculate rate using external API
     */
    private function calculateApiRate(RateRequest $request): float|false
    {
        try {
            // This would call your carrier's API
            // Example: return $this->apiAdapter->getRates($request);
            return 10.00; // Placeholder
        } catch (\Exception $e) {
            $this->_logger->error('Shipping API error: ' . $e->getMessage());
            return false;
        }
    }

    /**
     * Calculate rate from custom table
     */
    private function calculateTableRate(RateRequest $request): float|false
    {
        // This would query custom rate table
        // Example: return $this->rateRepository->getRateForRequest($request);
        return 15.00; // Placeholder
    }

    /**
     * Apply handling fee to shipping price
     */
    private function applyHandlingFee(float $price): float
    {
        $handlingFee = (float) $this->getConfigData('handling_fee');

        if ($handlingFee <= 0) {
            return $price;
        }

        $handlingType = $this->getConfigData('handling_type');

        if ($handlingType === 'P') {
            // Percentage
            return $price + ($price * ($handlingFee / 100));
        }

        // Fixed
        return $price + $handlingFee;
    }

    /**
     * Create rate method object
     */
    private function createRateMethod(float $price): Method
    {
        /** @var Method $method */
        $method = $this->rateMethodFactory->create();

        $method->setCarrier($this->_code);
        $method->setCarrierTitle($this->getConfigData('title'));

        $method->setMethod($this->_code);
        $method->setMethodTitle($this->getConfigData('name'));

        $method->setPrice($price);
        $method->setCost($price);

        // Add estimated delivery time
        $deliveryTime = $this->getConfigData('estimated_delivery_days');
        if ($deliveryTime) {
            $method->setMethodDescription('Estimated delivery: ' . $deliveryTime);
        }

        return $method;
    }

    /**
     * Get error message when shipping is not available
     */
    private function getErrorMessage()
    {
        if (!$this->getConfigFlag('showmethod')) {
            return false;
        }

        $error = $this->_rateErrorFactory->create();
        $error->setCarrier($this->_code);
        $error->setCarrierTitle($this->getConfigData('title'));
        $error->setErrorMessage(
            $this->getConfigData('specificerrmsg') ?: __('This shipping method is not available.')
        );

        return $error;
    }

    /**
     * Get allowed shipping methods
     *
     * @return array
     */
    public function getAllowedMethods(): array
    {
        return [
            $this->_code => $this->getConfigData('name')
        ];
    }

    /**
     * Check if carrier has shipping tracking capability
     *
     * @return bool
     */
    public function isTrackingAvailable(): bool
    {
        return true;
    }

    /**
     * Get tracking information
     *
     * @param string $tracking
     * @return TrackingResult|string
     */
    public function getTrackingInfo($tracking)
    {
        $result = $this->trackStatusFactory->create();

        $result->setCarrier($this->_code);
        $result->setCarrierTitle($this->getConfigData('title'));
        $result->setTracking($tracking);

        // In production, fetch from carrier API
        // Example: $trackingData = $this->trackingAdapter->getTrackingInfo($tracking);

        // Mock tracking data
        $result->setTrackSummary('Package is in transit');
        $result->setUrl('https://tracking.carrier.com?tracking=' . $tracking);

        return $result;
    }
}
```

## Rate Calculation Service

For complex rate calculations, extract the logic into a dedicated service.

**Model/RateCalculator.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Shipping\Model;

use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Api\GroupManagementInterface;
use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Quote\Model\Quote\Address\RateRequest;
use Magento\Store\Model\ScopeInterface;

/**
 * Advanced shipping rate calculator
 */
class RateCalculator
{
    private const CONFIG_PATH_VOLUME_DISCOUNT = 'carriers/customcarrier/volume_discount_enabled';
    private const CONFIG_PATH_CUSTOMER_GROUP_RATES = 'carriers/customcarrier/customer_group_rates';

    public function __construct(
        private readonly ScopeConfigInterface $scopeConfig,
        private readonly ProductRepositoryInterface $productRepository,
        private readonly CustomerRepositoryInterface $customerRepository,
        private readonly GroupManagementInterface $groupManagement
    ) {}

    /**
     * Calculate rate with advanced rules
     */
    public function calculateRate(RateRequest $request): float
    {
        $baseRate = $this->calculateBaseRate($request);

        // Apply customer group discount
        $baseRate = $this->applyCustomerGroupDiscount($baseRate, $request);

        // Apply volume discount
        $baseRate = $this->applyVolumeDiscount($baseRate, $request);

        // Apply product attribute-based surcharges
        $baseRate = $this->applyProductSurcharges($baseRate, $request);

        // Apply destination-based adjustments
        $baseRate = $this->applyDestinationAdjustments($baseRate, $request);

        return max(0.00, $baseRate);
    }

    /**
     * Calculate base rate from weight and distance
     */
    private function calculateBaseRate(RateRequest $request): float
    {
        $weight = $request->getPackageWeight();
        $value = $request->getPackageValue();

        // Weight-based tier pricing
        $rate = match (true) {
            $weight <= 5 => 5.00,
            $weight <= 10 => 8.00,
            $weight <= 20 => 12.00,
            $weight <= 50 => 20.00,
            default => 30.00,
        };

        // Add insurance for high-value orders
        if ($value > 500) {
            $rate += ($value - 500) * 0.01; // 1% of value over $500
        }

        return $rate;
    }

    /**
     * Apply customer group-based discount
     */
    private function applyCustomerGroupDiscount(float $rate, RateRequest $request): float
    {
        $customerId = $request->getData('customer_id');

        if (!$customerId) {
            return $rate;
        }

        try {
            $customer = $this->customerRepository->getById($customerId);
            $groupId = $customer->getGroupId();

            // Fetch customer group discount from config
            $groupRates = $this->getCustomerGroupRates();

            if (isset($groupRates[$groupId])) {
                $discountPercent = (float) $groupRates[$groupId];
                $rate = $rate * (1 - ($discountPercent / 100));
            }
        } catch (\Exception $e) {
            // Customer not found or group not configured, no discount
        }

        return $rate;
    }

    /**
     * Apply volume discount based on item count or subtotal
     */
    private function applyVolumeDiscount(float $rate, RateRequest $request): float
    {
        if (!$this->isVolumeDiscountEnabled()) {
            return $rate;
        }

        $itemCount = 0;
        if ($request->getAllItems()) {
            foreach ($request->getAllItems() as $item) {
                if (!$item->getProduct()->isVirtual() && !$item->getParentItem()) {
                    $itemCount += $item->getQty();
                }
            }
        }

        // Volume discount tiers
        $discount = match (true) {
            $itemCount >= 20 => 0.25, // 25% off
            $itemCount >= 10 => 0.15, // 15% off
            $itemCount >= 5 => 0.10,  // 10% off
            default => 0,
        };

        return $rate * (1 - $discount);
    }

    /**
     * Apply surcharges based on product attributes
     */
    private function applyProductSurcharges(float $rate, RateRequest $request): float
    {
        $surcharge = 0;

        if ($request->getAllItems()) {
            foreach ($request->getAllItems() as $item) {
                if ($item->getProduct()->isVirtual() || $item->getParentItem()) {
                    continue;
                }

                // Check for fragile items
                if ($item->getProduct()->getData('is_fragile')) {
                    $surcharge += 3.00 * $item->getQty();
                }

                // Check for oversized items
                if ($item->getProduct()->getData('is_oversized')) {
                    $surcharge += 10.00 * $item->getQty();
                }

                // Check for hazmat
                if ($item->getProduct()->getData('requires_hazmat_shipping')) {
                    $surcharge += 25.00;
                    break; // Hazmat fee applies per shipment, not per item
                }
            }
        }

        return $rate + $surcharge;
    }

    /**
     * Apply destination-based adjustments (remote areas, express zones)
     */
    private function applyDestinationAdjustments(float $rate, RateRequest $request): float
    {
        $destCountry = $request->getDestCountryId();
        $destPostcode = $request->getDestPostcode();

        // Remote area surcharge
        if ($this->isRemoteArea($destCountry, $destPostcode)) {
            $rate += 15.00;
        }

        // International shipping
        if ($destCountry !== 'US') {
            $rate *= 1.5; // 50% surcharge for international
        }

        return $rate;
    }

    /**
     * Check if destination is in remote area
     */
    private function isRemoteArea(string $country, string $postcode): bool
    {
        // This would typically query a database or API
        // For US, check against remote zip code ranges

        $remoteZipRanges = [
            ['99500', '99999'], // Alaska
            ['96701', '96898'], // Hawaii
        ];

        foreach ($remoteZipRanges as [$min, $max]) {
            if ($postcode >= $min && $postcode <= $max) {
                return true;
            }
        }

        return false;
    }

    /**
     * Check if volume discount is enabled
     */
    private function isVolumeDiscountEnabled(): bool
    {
        return (bool) $this->scopeConfig->getValue(
            self::CONFIG_PATH_VOLUME_DISCOUNT,
            ScopeInterface::SCOPE_STORE
        );
    }

    /**
     * Get customer group rates from config
     */
    private function getCustomerGroupRates(): array
    {
        $config = $this->scopeConfig->getValue(
            self::CONFIG_PATH_CUSTOMER_GROUP_RATES,
            ScopeInterface::SCOPE_STORE
        );

        if (!$config) {
            return [];
        }

        // Parse serialized config: [{"group_id": "1", "discount": "10"}, ...]
        $rates = json_decode($config, true);

        if (!is_array($rates)) {
            return [];
        }

        $result = [];
        foreach ($rates as $rate) {
            if (isset($rate['group_id'], $rate['discount'])) {
                $result[(int) $rate['group_id']] = (float) $rate['discount'];
            }
        }

        return $result;
    }
}
```

**Register in di.xml:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Inject RateCalculator into Carrier -->
    <type name="Vendor\Shipping\Model\Carrier\CustomCarrier">
        <arguments>
            <argument name="rateCalculator" xsi:type="object">Vendor\Shipping\Model\RateCalculator</argument>
        </arguments>
    </type>

</config>
```

## Tracking Integration

Tracking integration allows customers to see shipment status in their account and admin to track shipments.

**Model/TrackingAdapter.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Shipping\Model;

use GuzzleHttp\Client;
use GuzzleHttp\Exception\GuzzleException;
use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Shipping\Model\Tracking\Result\Status;
use Magento\Shipping\Model\Tracking\Result\StatusFactory;
use Magento\Store\Model\ScopeInterface;
use Psr\Log\LoggerInterface;

/**
 * Tracking API adapter
 */
class TrackingAdapter
{
    private const CONFIG_PATH_API_ENDPOINT = 'carriers/customcarrier/api_endpoint';
    private const CONFIG_PATH_API_KEY = 'carriers/customcarrier/api_key';

    public function __construct(
        private readonly ScopeConfigInterface $scopeConfig,
        private readonly StatusFactory $statusFactory,
        private readonly Client $httpClient,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Get tracking information from carrier API
     *
     * @param string $trackingNumber
     * @return Status
     */
    public function getTrackingInfo(string $trackingNumber): Status
    {
        /** @var Status $status */
        $status = $this->statusFactory->create();

        try {
            $trackingData = $this->fetchTrackingData($trackingNumber);

            $status->setTracking($trackingNumber);
            $status->setCarrier('customcarrier');
            $status->setCarrierTitle($this->getCarrierTitle());
            $status->setUrl($trackingData['tracking_url'] ?? null);

            // Set tracking summary
            $summary = $this->formatTrackingSummary($trackingData);
            $status->setTrackSummary($summary);

            // Set detailed tracking events
            if (isset($trackingData['events'])) {
                $progressDetail = $this->formatTrackingEvents($trackingData['events']);
                $status->setProgressDetail($progressDetail);
            }

            // Set delivery information
            if (isset($trackingData['delivered_at'])) {
                $status->setDeliveryDate($trackingData['delivered_at']);
                $status->setDeliveryTime($trackingData['delivered_at']);
                $status->setSignedBy($trackingData['signed_by'] ?? 'N/A');
            }

        } catch (\Exception $e) {
            $this->logger->error('Tracking API error: ' . $e->getMessage(), [
                'tracking_number' => $trackingNumber,
            ]);

            $status->setTracking($trackingNumber);
            $status->setCarrier('customcarrier');
            $status->setCarrierTitle($this->getCarrierTitle());
            $status->setTrackSummary('Tracking information is currently unavailable. Please try again later.');
        }

        return $status;
    }

    /**
     * Fetch tracking data from carrier API
     */
    private function fetchTrackingData(string $trackingNumber): array
    {
        $apiEndpoint = $this->scopeConfig->getValue(
            self::CONFIG_PATH_API_ENDPOINT,
            ScopeInterface::SCOPE_STORE
        );

        $apiKey = $this->scopeConfig->getValue(
            self::CONFIG_PATH_API_KEY,
            ScopeInterface::SCOPE_STORE
        );

        try {
            $response = $this->httpClient->get(
                $apiEndpoint . '/tracking/' . $trackingNumber,
                [
                    'headers' => [
                        'Authorization' => 'Bearer ' . $apiKey,
                        'Content-Type' => 'application/json',
                    ],
                    'timeout' => 10,
                ]
            );

            $body = (string) $response->getBody();
            $data = json_decode($body, true);

            if (json_last_error() !== JSON_ERROR_NONE) {
                throw new \RuntimeException('Invalid JSON response from tracking API');
            }

            return $data;

        } catch (GuzzleException $e) {
            throw new \RuntimeException('Failed to fetch tracking data: ' . $e->getMessage());
        }
    }

    /**
     * Format tracking summary text
     */
    private function formatTrackingSummary(array $trackingData): string
    {
        $status = $trackingData['status'] ?? 'unknown';

        return match ($status) {
            'in_transit' => 'Package is in transit to destination',
            'out_for_delivery' => 'Package is out for delivery',
            'delivered' => 'Package has been delivered',
            'exception' => 'Delivery exception: ' . ($trackingData['exception_message'] ?? 'Unknown issue'),
            'returned' => 'Package has been returned to sender',
            default => 'Tracking status unknown',
        };
    }

    /**
     * Format tracking events for progress detail
     */
    private function formatTrackingEvents(array $events): array
    {
        $progressDetail = [];

        foreach ($events as $event) {
            $progressDetail[] = [
                'activity' => $event['description'] ?? '',
                'deliverydate' => $event['date'] ?? '',
                'deliverytime' => $event['time'] ?? '',
                'deliverylocation' => $event['location'] ?? '',
            ];
        }

        return $progressDetail;
    }

    /**
     * Get carrier title from config
     */
    private function getCarrierTitle(): string
    {
        return (string) $this->scopeConfig->getValue(
            'carriers/customcarrier/title',
            ScopeInterface::SCOPE_STORE
        );
    }
}
```

**Update carrier to use tracking adapter:**

```php
// In CustomCarrier.php constructor, inject TrackingAdapter:
public function __construct(
    // ... other dependencies
    private readonly TrackingAdapter $trackingAdapter,
    // ...
) {
    parent::__construct($scopeConfig, $rateErrorFactory, $logger, $data);
}

// Update getTrackingInfo method:
public function getTrackingInfo($tracking)
{
    return $this->trackingAdapter->getTrackingInfo($tracking);
}
```

## Multi-Warehouse Shipping

For businesses with multiple fulfillment centers, calculate shipping from the optimal warehouse.

**Model/WarehouseSelector.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Shipping\Model;

use Magento\Framework\Exception\NoSuchEntityException;
use Magento\InventoryApi\Api\GetSourcesAssignedToStockOrderedByPriorityInterface;
use Magento\InventoryApi\Api\SourceRepositoryInterface;
use Magento\Quote\Model\Quote\Address\RateRequest;
use Magento\Store\Model\StoreManagerInterface;

/**
 * Select optimal warehouse for shipping calculation
 */
class WarehouseSelector
{
    public function __construct(
        private readonly StoreManagerInterface $storeManager,
        private readonly GetSourcesAssignedToStockOrderedByPriorityInterface $getSourcesAssignedToStock,
        private readonly SourceRepositoryInterface $sourceRepository,
        private readonly DistanceCalculator $distanceCalculator
    ) {}

    /**
     * Select best warehouse based on inventory and proximity
     *
     * @param RateRequest $request
     * @return array|null Warehouse data with origin address
     */
    public function selectWarehouse(RateRequest $request): ?array
    {
        $availableWarehouses = $this->getAvailableWarehouses($request);

        if (empty($availableWarehouses)) {
            return null;
        }

        // If only one warehouse, return it
        if (count($availableWarehouses) === 1) {
            return reset($availableWarehouses);
        }

        // Calculate distance from each warehouse to destination
        $destinationLat = $request->getData('dest_latitude');
        $destinationLng = $request->getData('dest_longitude');

        if (!$destinationLat || !$destinationLng) {
            // Fallback: use first warehouse if geocoding unavailable
            return reset($availableWarehouses);
        }

        $closest = null;
        $closestDistance = PHP_FLOAT_MAX;

        foreach ($availableWarehouses as $warehouse) {
            $distance = $this->distanceCalculator->calculate(
                $warehouse['latitude'],
                $warehouse['longitude'],
                $destinationLat,
                $destinationLng
            );

            if ($distance < $closestDistance) {
                $closestDistance = $distance;
                $closest = $warehouse;
            }
        }

        return $closest;
    }

    /**
     * Get warehouses that have inventory for requested items
     */
    private function getAvailableWarehouses(RateRequest $request): array
    {
        // Get stock for current store
        try {
            $store = $this->storeManager->getStore();
            $stockId = (int) $store->getWebsite()->getCode(); // Simplified
        } catch (NoSuchEntityException $e) {
            return [];
        }

        // Get sources (warehouses) assigned to this stock
        $sources = $this->getSourcesAssignedToStock->execute($stockId);

        $warehouses = [];

        foreach ($sources as $source) {
            if (!$source->isEnabled()) {
                continue;
            }

            // Check if source has inventory for requested items
            if (!$this->sourceHasInventory($source->getSourceCode(), $request)) {
                continue;
            }

            try {
                $sourceData = $this->sourceRepository->get($source->getSourceCode());

                $warehouses[] = [
                    'source_code' => $sourceData->getSourceCode(),
                    'name' => $sourceData->getName(),
                    'country' => $sourceData->getCountryId(),
                    'region' => $sourceData->getRegion(),
                    'postcode' => $sourceData->getPostcode(),
                    'city' => $sourceData->getCity(),
                    'street' => $sourceData->getStreet(),
                    'latitude' => $sourceData->getLatitude(),
                    'longitude' => $sourceData->getLongitude(),
                ];
            } catch (NoSuchEntityException $e) {
                continue;
            }
        }

        return $warehouses;
    }

    /**
     * Check if source has inventory for all requested items
     */
    private function sourceHasInventory(string $sourceCode, RateRequest $request): bool
    {
        // This would check MSI inventory levels
        // Simplified for example
        return true;
    }
}
```

**Model/DistanceCalculator.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Shipping\Model;

/**
 * Calculate distance between two geographic coordinates
 */
class DistanceCalculator
{
    private const EARTH_RADIUS_MILES = 3959;
    private const EARTH_RADIUS_KM = 6371;

    /**
     * Calculate distance using Haversine formula
     *
     * @param float $lat1 Origin latitude
     * @param float $lng1 Origin longitude
     * @param float $lat2 Destination latitude
     * @param float $lng2 Destination longitude
     * @param string $unit 'mi' for miles, 'km' for kilometers
     * @return float Distance in specified unit
     */
    public function calculate(
        float $lat1,
        float $lng1,
        float $lat2,
        float $lng2,
        string $unit = 'mi'
    ): float {
        $latDelta = deg2rad($lat2 - $lat1);
        $lngDelta = deg2rad($lng2 - $lng1);

        $a = sin($latDelta / 2) * sin($latDelta / 2) +
            cos(deg2rad($lat1)) * cos(deg2rad($lat2)) *
            sin($lngDelta / 2) * sin($lngDelta / 2);

        $c = 2 * atan2(sqrt($a), sqrt(1 - $a));

        $radius = $unit === 'km' ? self::EARTH_RADIUS_KM : self::EARTH_RADIUS_MILES;

        return $radius * $c;
    }
}
```

## Shipping Restrictions

Implement observers and validators to restrict shipping methods based on business rules.

**Observer/RestrictShippingObserver.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Shipping\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

/**
 * Restrict shipping methods based on custom rules
 */
class RestrictShippingObserver implements ObserverInterface
{
    /**
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var \Magento\Shipping\Model\Rate\Result $result */
        $result = $observer->getEvent()->getResult();

        /** @var Quote $quote */
        $quote = $observer->getEvent()->getRequest()->getData('quote');

        if (!$quote) {
            return;
        }

        // Restrict based on product attributes
        if ($this->quoteContainsRestrictedProducts($quote)) {
            $this->removeCustomCarrier($result);
        }

        // Restrict based on customer group
        if ($this->isRestrictedCustomerGroup($quote)) {
            $this->removeCustomCarrier($result);
        }

        // Restrict based on order value
        if ($this->isBelowMinimumOrderValue($quote)) {
            $this->removeCustomCarrier($result);
        }
    }

    /**
     * Check if quote contains products with shipping restrictions
     */
    private function quoteContainsRestrictedProducts(Quote $quote): bool
    {
        foreach ($quote->getAllVisibleItems() as $item) {
            if ($item->getProduct()->getData('no_custom_shipping')) {
                return true;
            }

            // Check for dangerous goods
            if ($item->getProduct()->getData('is_dangerous_goods')) {
                return true;
            }
        }

        return false;
    }

    /**
     * Check if customer group is restricted
     */
    private function isRestrictedCustomerGroup(Quote $quote): bool
    {
        $restrictedGroups = [3]; // Example: Wholesale group

        return in_array($quote->getCustomerGroupId(), $restrictedGroups, true);
    }

    /**
     * Check if order value is below minimum
     */
    private function isBelowMinimumOrderValue(Quote $quote): bool
    {
        $minimumOrderValue = 50.00;

        return $quote->getSubtotal() < $minimumOrderValue;
    }

    /**
     * Remove custom carrier from result
     */
    private function removeCustomCarrier(\Magento\Shipping\Model\Rate\Result $result): void
    {
        $rates = $result->getAllRates();

        foreach ($rates as $key => $rate) {
            if ($rate->getCarrier() === 'customcarrier') {
                unset($rates[$key]);
            }
        }

        $result->reset();
        foreach ($rates as $rate) {
            $result->append($rate);
        }
    }
}
```

**Register observer in events.xml:**

Note: There is no `shipping_collect_rates_after` event in core Magento. To restrict shipping methods, use a plugin on `Magento\Shipping\Model\Shipping::collectRates` instead.

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:di:etc/di.xsd">
    <type name="Magento\Shipping\Model\Shipping">
        <plugin name="vendor_shipping_restrict"
                type="Vendor\Shipping\Plugin\RestrictShippingPlugin"/>
    </type>
</config>
```

## Testing Shipping Methods

**Test/Unit/Model/Carrier/CustomCarrierTest.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Shipping\Test\Unit\Model\Carrier;

use Magento\Quote\Model\Quote\Address\RateRequest;
use Magento\Quote\Model\Quote\Address\RateResult\MethodFactory;
use Magento\Shipping\Model\Rate\ResultFactory;
use PHPUnit\Framework\MockObject\MockObject;
use PHPUnit\Framework\TestCase;
use Vendor\Shipping\Model\Carrier\CustomCarrier;

class CustomCarrierTest extends TestCase
{
    private CustomCarrier $carrier;
    private ResultFactory|MockObject $rateResultFactory;
    private MethodFactory|MockObject $rateMethodFactory;

    protected function setUp(): void
    {
        $this->rateResultFactory = $this->createMock(ResultFactory::class);
        $this->rateMethodFactory = $this->createMock(MethodFactory::class);

        // Create carrier with mocked dependencies
        // (Full constructor omitted for brevity)
    }

    public function testCollectRatesReturnsMethodForValidRequest(): void
    {
        $request = $this->createMock(RateRequest::class);
        $request->method('getDestCountryId')->willReturn('US');
        $request->method('getPackageWeight')->willReturn(10.0);
        $request->method('getPackageValue')->willReturn(50.00);

        $result = $this->carrier->collectRates($request);

        $this->assertInstanceOf(\Magento\Shipping\Model\Rate\Result::class, $result);
        $this->assertNotEmpty($result->getAllRates());
    }

    public function testCollectRatesReturnsFalseWhenInactive(): void
    {
        // Mock carrier as inactive
        $request = $this->createMock(RateRequest::class);

        $result = $this->carrier->collectRates($request);

        $this->assertFalse($result);
    }

    public function testCollectRatesReturnsErrorWhenWeightExceedsLimit(): void
    {
        $request = $this->createMock(RateRequest::class);
        $request->method('getPackageWeight')->willReturn(100.0); // Over limit

        $result = $this->carrier->collectRates($request);

        $this->assertInstanceOf(\Magento\Quote\Model\Quote\Address\RateResult\Error::class, $result);
    }

    public function testFreeShippingAppliedAboveThreshold(): void
    {
        $request = $this->createMock(RateRequest::class);
        $request->method('getPackageValue')->willReturn(150.00); // Above $100 threshold

        $result = $this->carrier->collectRates($request);
        $rates = $result->getAllRates();

        $this->assertEquals(0.00, $rates[0]->getPrice());
    }
}
```

---

## Assumptions

- **Target versions**: Magento 2.4.7+, PHP 8.2+
- **Deployment**: Single store or multi-store with shared shipping configuration
- **Multi-warehouse**: Uses Magento MSI (Multi-Source Inventory)
- **API integration**: Carrier provides REST API with rate and tracking endpoints

## Why This Approach

**AbstractCarrier inheritance**: Provides standard configuration, logging, and error handling

**RateCalculator service**: Separates complex business logic from carrier for testability

**TrackingAdapter**: Abstracts API client for easy testing and API version changes

**WarehouseSelector**: Implements proximity-based warehouse selection for optimal shipping costs

**Observer pattern**: Enables flexible restriction rules without modifying carrier

## Security Impact

**API credentials**: Encrypted in database, never exposed in frontend

**Input validation**: All rate request data validated (weight, destinations, values)

**Rate manipulation prevention**: Rates calculated server-side, never trusted from client

**Authorization**: Shipping configuration requires admin access with `Magento_Shipping::carriers` ACL

## Performance Impact

**Cacheability:**
- Shipping rates are **not cached** (cart-specific, dynamic)
- Carrier configuration **is cached** (config cache type)
- API responses can be cached short-term (60-300 seconds) via custom cache

**Database impact:**
- Rate requests query: quote items, product attributes, inventory levels
- Properly indexed tables ensure fast lookups
- For high traffic, consider Redis for rate caching

**API latency:**
- External carrier APIs: 500-3000ms per rate request
- Mitigation: Request rates in parallel for multiple carriers
- Timeout shipping API calls at 10 seconds to prevent checkout delays

**Core Web Vitals impact:**
- Shipping rates load during checkout step (does not impact homepage/category)
- Use AJAX to fetch rates asynchronously after address entry
- Display loading indicator during rate calculation

## Backward Compatibility

**API stability:**
- `CarrierInterface` is stable across Magento 2.4.x
- `RateRequest` structure is stable
- Configuration XML structure is stable

**Database schema:**
- No custom tables required for basic shipping (uses core tables)
- Custom rate tables use declarative schema for safe upgrades

**Upgrade path:**
- Magento 2.4.7 → 2.4.8: No breaking changes
- Magento 2.4.x → 2.4.9: Monitor deprecation notices in shipping interfaces

## Tests to Add

**Unit tests:**
- Rate calculation logic (fixed, API, table)
- Handling fee calculation (fixed, percentage)
- Weight/destination restrictions
- Free shipping threshold
- Error message generation

**Integration tests:**
- Carrier availability in checkout
- Rate request with real quote
- Tracking information retrieval
- Multi-warehouse selection

**Functional tests (MFTF):**
- Complete checkout with custom shipping method
- Admin shipment creation with tracking
- Shipping restrictions (weight, country)
- Free shipping threshold

## Documentation to Update

**Merchant documentation:**
- `README.md`: Installation, configuration
- `CONFIGURATION.md`: Rate calculation modes, restrictions
- `TROUBLESHOOTING.md`: Common issues (rates not showing, tracking not working)

**Developer documentation:**
- `ARCHITECTURE.md`: Carrier architecture, extension points
- `API.md`: Carrier API integration guide
- `TESTING.md`: How to test shipping methods

**Admin user guide:**
- How to configure shipping method
- How to create shipments with tracking
- Troubleshooting missing rates

## Related Documentation

### Related Guides

- [Custom Payment Method Development: Building Secure Payment Integrations](custom-payment-method.md)
- [Service Contracts vs Repositories in Magento 2](../explanations/service-contracts-repositories.md)
- [Comprehensive Testing Strategies for Magento 2](../how-to/testing-strategies.md)

### Related Module Documentation

- [Magento_Checkout Overview](../../modules/checkout/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
- [Magento_Quote Overview](../../modules/quote/README.md)
