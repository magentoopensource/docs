---
title: "Magento_Quote Version Compatibility"
module: "Magento_Quote"
doc_type: "version-compatibility"
version: "2.4.7+"
last_updated: "2026-02-09"
---

# Magento_Quote Version Compatibility

## Overview

This document tracks verified API surface, deprecations, and compatibility information for the Magento_Quote module in version 2.4.7.

> **Note:** This document only contains facts verified against the Magento 2.4.7 source code. Version-to-version changelog entries that could not be verified against source have been removed.

**Module Version (2.4.7):** 101.2.7
**PHP Requirement:** `~8.1.0||~8.2.0||~8.3.0`

---

## Key Facts

### Interface Return Types

Magento 2.4.7 Quote API interfaces use **docblock `@return` annotations only** — they do NOT have PHP native return type declarations:

```php
// Actual CartRepositoryInterface in 2.4.7
// NO PHP return type declarations — only @return docblocks
public function get($cartId);
public function getList(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria);
public function getForCustomer($customerId, array $sharedStoreIds = []);
public function getActive($cartId, array $sharedStoreIds = []);
public function getActiveForCustomer($customerId, array $sharedStoreIds = []);
public function save(\Magento\Quote\Api\Data\CartInterface $quote);
public function delete(\Magento\Quote\Api\Data\CartInterface $quote);
```

Similarly, `CartManagementInterface` has NO PHP return type declarations:

```php
public function createEmptyCart();
public function createEmptyCartForCustomer($customerId);
public function getCartForCustomer($customerId);
public function assignCustomer($cartId, $customerId, $storeId);
public function placeOrder($cartId, \Magento\Quote\Api\Data\PaymentInterface $paymentMethod = null);
```

When implementing these interfaces, match the interface signature exactly — do NOT add PHP return types or typed parameters that don't exist in the interface.

### Quote Model Properties

The Quote model in 2.4.7 uses **traditional untyped properties** (e.g., `protected $_customer`, `protected $_addresses`). It does NOT use PHP typed property declarations.

Some newer classes in the module (e.g., `QuoteAddressValidator`) do use typed properties like `protected AddressRepositoryInterface $addressRepository`, but the core Quote model does not.

---

## Composer Dependencies (2.4.7)

| Dependency | Version Constraint |
|---|---|
| magento/framework | 103.0.* |
| magento/module-authorization | 100.4.* |
| magento/module-backend | 102.0.* |
| magento/module-catalog | 104.0.* |
| magento/module-catalog-inventory | 100.4.* |
| magento/module-checkout | 100.4.* |
| magento/module-customer | 103.0.* |
| magento/module-directory | 100.4.* |
| magento/module-eav | 102.1.* |
| magento/module-payment | 100.4.* |
| magento/module-sales | 103.0.* |
| magento/module-sales-sequence | 100.4.* |
| magento/module-shipping | 100.4.* |
| magento/module-store | 101.1.* |
| magento/module-tax | 100.4.* |

---

## API Interfaces

### Management Interfaces (Api/ — 21 interfaces)

- `BillingAddressManagementInterface`
- `CartItemRepositoryInterface`
- `CartManagementInterface`
- `CartRepositoryInterface`
- `CartTotalManagementInterface`
- `CartTotalRepositoryInterface`
- `ChangeQuoteControlInterface`
- `CouponManagementInterface`
- `GuestBillingAddressManagementInterface`
- `GuestCartItemRepositoryInterface`
- `GuestCartManagementInterface`
- `GuestCartRepositoryInterface`
- `GuestCartTotalManagementInterface`
- `GuestCartTotalRepositoryInterface`
- `GuestCouponManagementInterface`
- `GuestPaymentMethodManagementInterface`
- `GuestShipmentEstimationInterface`
- `GuestShippingMethodManagementInterface`
- `PaymentMethodManagementInterface`
- `ShipmentEstimationInterface`
- `ShippingMethodManagementInterface`

### Data Interfaces (Api/Data/ — 17 interfaces)

- `AddressAdditionalDataInterface`
- `AddressInterface`
- `CartInterface`
- `CartItemInterface`
- `CartSearchResultsInterface`
- `CurrencyInterface`
- `EstimateAddressInterface`
- `PaymentInterface`
- `PaymentMethodInterface`
- `ProductOptionInterface`
- `ShippingAssignmentInterface`
- `ShippingInterface`
- `ShippingMethodInterface`
- `TotalsAdditionalDataInterface`
- `TotalSegmentInterface`
- `TotalsInterface`
- `TotalsItemInterface`

---

## Verified Deprecations (from `@deprecated` annotations in source)

### Deprecated Interface Methods

| Interface | Method | Deprecated Since |
|---|---|---|
| `ShippingMethodManagementInterface` | `estimateByAddress()` | 100.0.7 |
| `GuestShippingMethodManagementInterface` | `estimateByAddress()` | 100.0.7 |

### Deprecated Class Methods/Properties

| Class | Member | Deprecated Since | Notes |
|---|---|---|---|
| `ShippingMethodManagement` | `getEstimatedRates()` | 100.1.6 | — |
| `ShippingMethodManagement` | `getDataObjectProcessor()` | 101.0.0 | — |
| `GuestShippingMethodManagement` | `getShipmentEstimationManagement()` | 100.0.7 | — |
| `Quote` | `loadByCustomer()` | 101.0.0 | — |
| `BillingAddressManagement` | `getShippingAddressAssignment()` | 101.0.0 | — |
| `QuoteRepository` | `addFilterGroupToCollection()` | 101.0.0 | — |
| `QuoteRepository` | `$quoteCollection` | 101.0.0 | — |
| `QuoteRepository` | `$quoteFactory` | 101.1.2 | — |
| `Quote\Item` | `$stockRegistry` | 101.0.0 | — |
| `Quote\TotalsCollector` | `_collectItemsQtys()` | (no version) | See `QuantityCollector` |
| `QuoteAddressValidator` | `$customerSession` | 101.1.1 | — |
| `ResourceModel\Quote` | `substractProductFromQuotes()` | 101.0.3 | Typo fix: use `subtractProductFromQuotes()` |

### Deprecated Classes

| Class | Deprecated Since | Notes |
|---|---|---|
| `Quote\Item\CartItemProcessorsPool` | 100.1.0 | — |
| `Quote\Validator\MinimumOrderAmount\ValidationMessage::$currency` | 101.0.3 | Property only |

---

## `@since` Version Tags

Most API interfaces are tagged `@since 100.0.2`. Key ones:

| Class/Interface | @since |
|---|---|
| `CartRepositoryInterface` | 100.0.2 |
| `CartManagementInterface` | 100.0.2 |
| `CartInterface` | 100.0.2 |
| `CartItemInterface` | 100.0.2 |
| `AddressInterface` | 100.0.2 |
| `PaymentInterface` | 100.0.2 |
| `TotalsInterface` | 100.0.2 |
| `TotalSegmentInterface` | 100.0.2 |
| `Quote\Address::$_addressRateFactory` | 101.0.0 |

---

## Upgrade Best Practices

### Use Service Contracts

```php
// Recommended — service contract (stable API)
public function __construct(
    private readonly \Magento\Quote\Api\CartRepositoryInterface $cartRepository
) {}

// Avoid — model direct (internal implementation)
public function __construct(
    private readonly \Magento\Quote\Model\QuoteFactory $quoteFactory
) {}
```

### Plugin on Interfaces, Not Classes

```xml
<!-- Recommended — stable plugin target -->
<plugin name="vendor_plugin"
        type="Vendor\Module\Plugin\CartRepositoryPlugin"
        for="Magento\Quote\Api\CartRepositoryInterface"/>

<!-- Avoid — may break on internal refactoring -->
<plugin name="vendor_plugin"
        type="Vendor\Module\Plugin\QuotePlugin"
        for="Magento\Quote\Model\Quote"/>
```

### Use Extension Attributes

```xml
<!-- etc/extension_attributes.xml -->
<extension_attributes for="Magento\Quote\Api\Data\CartInterface">
    <attribute code="custom_field" type="string"/>
</extension_attributes>
```

---

**Disclaimer:** This document covers the Magento_Quote module as found in `magento/product-community-edition:2.4.7`. Version-to-version change details could not be verified from source alone and have been removed. For version-specific changelogs, consult the official Adobe Commerce release notes.
