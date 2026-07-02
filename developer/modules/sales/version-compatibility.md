---
title: "Magento_Sales Version Compatibility"
module: "Magento_Sales"
doc_type: "version-compatibility"
version: "2.4.7+"
last_updated: "2026-02-09"
---

# Magento_Sales Version Compatibility

## Overview

This document tracks verified API surface, deprecations, and compatibility information for the Magento_Sales module in version 2.4.7.

> **Note:** This document only contains facts verified against the Magento 2.4.7 source code. Version-to-version changelog entries that could not be verified against source have been removed.

**Module Version (2.4.7):** 103.0.7
**PHP Requirement:** `~8.1.0||~8.2.0||~8.3.0`

---

## Key Facts

### Interface Return Types

Magento 2.4.7 Sales API interfaces use **docblock `@return` annotations only** — they do NOT have PHP native return type declarations:

```php
// Actual OrderRepositoryInterface in 2.4.7
// NO PHP return type declarations
public function getList(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria);
public function get($id);
public function delete(\Magento\Sales\Api\Data\OrderInterface $entity);
public function save(\Magento\Sales\Api\Data\OrderInterface $entity);
```

Similarly, `OrderManagementInterface` has NO PHP return type declarations:

```php
public function cancel($id);
public function getCommentsList($id);
public function addComment($id, \Magento\Sales\Api\Data\OrderStatusHistoryInterface $statusHistory);
public function notify($id);
public function getStatus($id);
public function hold($id);
public function unHold($id);
public function place(\Magento\Sales\Api\Data\OrderInterface $order);
```

### Non-Existent Interfaces

The following interfaces do **NOT exist** in Magento_Sales 2.4.7 — do not reference them:

- `BulkOrderManagementInterface` — does not exist
- `getListWithAdvancedFilters()` method — does not exist on `OrderRepositoryInterface`

### Order Grid Sync

The sales order grid is synced via `GridSyncInsertObserver` on `sales_order_save_after`, **NOT** via mview subscriptions.

---

## Composer Dependencies (2.4.7)

| Dependency | Version Constraint |
|---|---|
| magento/framework | 103.0.* |
| magento/module-authorization | 100.4.* |
| magento/module-backend | 102.0.* |
| magento/module-catalog | 104.0.* |
| magento/module-bundle | 101.0.* |
| magento/module-catalog-inventory | 100.4.* |
| magento/module-checkout | 100.4.* |
| magento/module-config | 101.2.* |
| magento/module-customer | 103.0.* |
| magento/module-directory | 100.4.* |
| magento/module-eav | 102.1.* |
| magento/module-gift-message | 100.4.* |
| magento/module-media-storage | 100.4.* |
| magento/module-payment | 100.4.* |
| magento/module-quote | 101.2.* |
| magento/module-reports | 100.4.* |
| magento/module-sales-rule | 101.2.* |
| magento/module-sales-sequence | 100.4.* |
| magento/module-shipping | 100.4.* |
| magento/module-store | 101.1.* |
| magento/module-tax | 100.4.* |
| magento/module-theme | 101.1.* |
| magento/module-ui | 101.2.* |
| magento/module-widget | 101.2.* |
| magento/module-wishlist | 101.2.* |

---

## API Interfaces

The Sales module has approximately 82 `@api`-annotated interfaces. Key ones include:

### Management Interfaces

- `OrderRepositoryInterface` (@since 100.0.2)
- `OrderManagementInterface` (@since 100.0.2)
- `InvoiceOrderInterface`
- `InvoiceManagementInterface`
- `RefundOrderInterface`
- `RefundInvoiceInterface`
- `ShipOrderInterface`
- `ShipmentManagementInterface`
- `CreditmemoManagementInterface`
- `PaymentFailuresInterface`

### Exception Interfaces

- `CouldNotInvoiceExceptionInterface` (@since 100.1.2)
- `CouldNotShipExceptionInterface` (@since 100.1.2)
- `DocumentValidationExceptionInterface` (@since 100.1.2)
- `CouldNotRefundExceptionInterface` (@since 100.1.3)

---

## Verified Deprecations (from `@deprecated` annotations in source)

### Deprecated Classes

| Class | Deprecated Since | Replacement |
|---|---|---|
| `Model\Increment` | 101.0.0 | — |
| `Controller\Adminhtml\Order\AbstractMassAction` | 101.0.0 | — |
| `Model\ResourceModel\Order\Creditmemo\Relation\Refund` | 100.1.3 | — |
| `Model\Order\Email\Sender\ShipmentSender` | 102.1.0 | `Order\Shipment\Sender\EmailSender` |
| `Block\Adminhtml\Order\Totalbar` | 101.0.6 | — |

### Deprecated Methods

| Class | Method | Deprecated Since | Notes |
|---|---|---|---|
| `Order` | `addStatusHistoryComment()` | 101.0.5 | — |
| `CreditmemoService` | `getRefundAdapter()` | 100.1.3 | — |
| `CreditmemoService` | `getResourceConnection()` | 100.1.3 | — |
| `CreditmemoService` | `getOrderRepository()` | 100.1.3 | — |
| `CreditmemoService` | `getInvoiceRepository()` | 100.1.3 | — |
| `OrderRepository` | `setSearchResultAppliedRules()` | 101.0.0 | — |
| `AdminOrder\Create` | `deleteCustomOption()` | 101.0.0 | — |
| `AdminOrder\Create` | `deleteUseQuoteAddress()` | 101.0.0 | — |
| `Order\Payment` | `getCcSsIssue()` | 100.1.0 | Unused |
| `Order\Payment` | `getCcSsStartMonth()` | 100.1.0 | Unused |
| `Order\Payment` | `getCcSsStartYear()` | 100.1.0 | Unused |
| `Order\Config` | `getStatusFrontendLabel()` | 102.0.1 | See `StatusLabel` |
| `Block\Order\History` | `getOrderCollectionFactory()` | 100.1.1 | — |
| `Block\Order\History` | `getViewOrderUrl()` | 102.0.3 | — |
| `Block\Order\Recent` | `getViewOrderUrl()` | 102.0.3 | — |

### Deprecated Event Argument

In all email sender classes (OrderSender, InvoiceSender, ShipmentSender, CreditmemoSender, etc.), the event argument `transport` is deprecated — use `transportObject` instead.

### Deprecated Payment Interface Constants

| Constant | Notes |
|---|---|
| `OrderPaymentInterface::CC_SS_START_YEAR` | Unused |
| `OrderPaymentInterface::CC_SS_START_MONTH` | Unused |
| `OrderPaymentInterface::CC_SS_ISSUE` | Unused |

---

## Upgrade Best Practices

### Use Repository Pattern

```php
// Recommended — uses service contract
$searchCriteria = $this->searchCriteriaBuilder
    ->addFilter('status', 'processing')
    ->create();
$orders = $this->orderRepository->getList($searchCriteria);

// Deprecated pattern (still works)
$collection = $this->orderCollectionFactory->create();
$collection->addFieldToFilter('status', 'processing');
```

### Use Interface Type Hints

```php
// Recommended — interface (stable API)
public function __construct(
    private readonly \Magento\Sales\Api\OrderRepositoryInterface $orderRepository
) {}

// Avoid — concrete class (may change)
public function __construct(
    private readonly \Magento\Sales\Model\OrderRepository $orderRepository
) {}
```

---

**Disclaimer:** This document covers the Magento_Sales module as found in `magento/product-community-edition:2.4.7`. Version-to-version change details could not be verified from source alone and have been removed. For version-specific changelogs, consult the official Adobe Commerce release notes.
