---
title: "Magento_Checkout Version Compatibility"
module: "Magento_Checkout"
doc_type: "version-compatibility"
version: "2.4.7+"
last_updated: "2026-02-09"
---

# Magento_Checkout Version Compatibility

## Overview

This document tracks verified API surface, deprecations, and compatibility information for the Magento_Checkout module in version 2.4.7.

> **Note:** This document only contains facts verified against the Magento 2.4.7 source code. Version-to-version changelog entries that could not be verified against source have been removed.

**Module Version (2.4.7):** 100.4.7
**PHP Requirement:** `~8.1.0||~8.2.0||~8.3.0`

---

## Key Facts

### Interface Return Types

Magento 2.4.7 Checkout API interfaces use **docblock annotations only** — they do NOT have PHP native return type declarations:

```php
// Actual PaymentInformationManagementInterface in 2.4.7
// NO PHP return type declarations
public function savePaymentInformationAndPlaceOrder(
    $cartId,
    \Magento\Quote\Api\Data\PaymentInterface $paymentMethod,
    \Magento\Quote\Api\Data\AddressInterface $billingAddress = null
);

public function savePaymentInformation(
    $cartId,
    \Magento\Quote\Api\Data\PaymentInterface $paymentMethod,
    \Magento\Quote\Api\Data\AddressInterface $billingAddress = null
);

public function getPaymentInformation($cartId);
```

When implementing these interfaces, match the signature exactly — do NOT add PHP return types.

### Non-Existent Methods

The following do **NOT exist** in Magento_Checkout 2.4.7:

- `Cart::getCartData()` — does not exist. The `CustomerData\Cart` class only has `getSectionData()` and `isGuestCheckoutAllowed()`.
- `InventoryReservationManagementInterface` — does not exist.

### RequestInfoFilterInterface Is NOT Deprecated

`Magento\Checkout\Model\Cart\RequestInfoFilterInterface` is **not deprecated**. It has `@api` and `@since 100.1.2` annotations with no `@deprecated` tag.

---

## Composer Dependencies (2.4.7)

| Dependency | Version Constraint |
|---|---|
| magento/framework | 103.0.* |
| magento/module-captcha | 100.4.* |
| magento/module-catalog | 104.0.* |
| magento/module-catalog-inventory | 100.4.* |
| magento/module-config | 101.2.* |
| magento/module-customer | 103.0.* |
| magento/module-directory | 100.4.* |
| magento/module-eav | 102.1.* |
| magento/module-msrp | 100.4.* |
| magento/module-page-cache | 100.4.* |
| magento/module-payment | 100.4.* |
| magento/module-quote | 101.2.* |
| magento/module-sales | 103.0.* |
| magento/module-sales-rule | 101.2.* |
| magento/module-security | 100.4.* |
| magento/module-shipping | 100.4.* |
| magento/module-store | 101.1.* |
| magento/module-tax | 100.4.* |
| magento/module-theme | 101.1.* |
| magento/module-ui | 101.2.* |
| magento/module-authorization | 100.4.* |
| magento/module-csp | 100.4.* |

---

## API Interfaces

### Management Interfaces (Api/)

- `PaymentInformationManagementInterface` (@api, @since 100.0.2)
- `GuestPaymentInformationManagementInterface` (@api, @since 100.0.2)
- `ShippingInformationManagementInterface` (@api, @since 100.0.2)
- `GuestShippingInformationManagementInterface` (@api, @since 100.0.2)
- `TotalsInformationManagementInterface` (@api, @since 100.0.2)
- `GuestTotalsInformationManagementInterface` (@api, @since 100.0.2)
- `AgreementsValidatorInterface` (@api, @since 100.0.2)
- `PaymentProcessingRateLimiterInterface` (@api, no @since)
- `PaymentSavingRateLimiterInterface` (no @api, no @since)

### Data Interfaces (Api/Data/)

- `PaymentDetailsInterface` (@api, @since 100.0.2)
- `ShippingInformationInterface` (@api, @since 100.0.2)
- `TotalsInformationInterface` (@api, @since 100.0.2)

### Exception Classes

- `PaymentProcessingRateLimitExceededException` (@api)

### Other @api Classes

- `Controller\Cart` (abstract, @api)
- `Model\Cart\RequestInfoFilterInterface` (@api, @since 100.1.2)

---

## Verified Deprecations (from `@deprecated` annotations in source)

### Deprecated Classes

| Class | Deprecated Since | Replacement |
|---|---|---|
| `Controller\Account\Create` | 100.2.5 | `DelegateCreate` |
| `Block\Shipping\Price` | 100.1.0 | — |
| `Model\Cart\CartInterface` | 100.1.0 | `\Magento\Quote\Api\Data\CartInterface` |
| `Model\Cart` | 100.1.0 | `\Magento\Quote\Api\Data\CartInterface` |
| `Model\Sidebar` | 100.1.0 | — |

---

## REST API Endpoints

These REST endpoints are stable across 2.4.x:

```
POST   /rest/V1/carts/mine/shipping-information
POST   /rest/V1/carts/mine/payment-information
POST   /rest/V1/guest-carts/:cartId/shipping-information
POST   /rest/V1/guest-carts/:cartId/payment-information
GET    /rest/V1/carts/mine/totals
GET    /rest/V1/guest-carts/:cartId/totals
```

---

## Upgrade Best Practices

### Use Service Contracts

```php
// Recommended — uses service contract (stable across versions)
public function __construct(
    private readonly \Magento\Checkout\Api\PaymentInformationManagementInterface $paymentManagement
) {}

// Avoid — uses concrete implementation (may change)
public function __construct(
    private readonly \Magento\Checkout\Model\PaymentInformationManagement $paymentManagement
) {}
```

### Use Extension Attributes for Custom Data

```php
// Recommended — version-safe
$extensionAttributes = $quote->getExtensionAttributes();
$extensionAttributes->setCustomData($value);

// Avoid — triggers PHP 8.2 dynamic property deprecation
$quote->custom_data = $value;
```

---

**Disclaimer:** This document covers the Magento_Checkout module as found in `magento/product-community-edition:2.4.7`. Version-to-version change details could not be verified from source alone and have been removed. For version-specific changelogs, consult the official Adobe Commerce release notes.
