---
title: "Magento_Quote Anti-Patterns"
module: "Magento_Quote"
doc_type: "anti-patterns"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Quote Anti-Patterns

## Overview

This document catalogs common mistakes, anti-patterns, and problematic practices when working with the Magento_Quote module. Each anti-pattern includes the problem description, why it's harmful, correct alternatives, and migration strategies for fixing existing code.

**Target Version:** Magento 2.4.7+ | Adobe Commerce & Open Source
**PHP Version:** 8.2+

## Anti-Pattern Categories

1. Direct Model Manipulation
2. Totals Calculation Errors
3. Quote State Management Issues
4. Performance Anti-Patterns
5. Security Vulnerabilities
6. Data Integrity Problems

---

## 1. Direct Model Manipulation

### Anti-Pattern 1.1: Direct save() on Quote Model

**Problem:**

```php
<?php
// WRONG: Direct save bypasses repository plugins and validation
$quote = $this->quoteFactory->create();
$quote->load($quoteId);
$quote->setGrandTotal(100.00);
$quote->save(); // Anti-pattern
```

**Why It's Harmful:**

- Bypasses repository plugins that may enforce business rules
- Skips validation logic in repository layer
- Doesn't trigger proper cache invalidation
- Breaks extension points for third-party modules
- Makes code harder to test (tightly coupled to model)

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;

class QuoteUpdateService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository
    ) {}

    public function updateQuote(int $quoteId): void
    {
        // CORRECT: Use repository for all persistence operations
        $quote = $this->cartRepository->get($quoteId);
        $quote->setGrandTotal(100.00);
        $this->cartRepository->save($quote);
    }
}
```

**Migration Strategy:**

1. Search codebase for `$quote->save()` calls
2. Replace with repository pattern
3. Add deprecation warnings if maintaining BC required
4. Run integration tests to verify plugins still execute

---

### Anti-Pattern 1.2: Direct Database Queries

**Problem:**

```php
<?php
// WRONG: Direct SQL query bypasses models and validations
$connection = $this->resource->getConnection();
$connection->update(
    'quote',
    ['grand_total' => 100.00],
    ['entity_id = ?' => $quoteId]
);
```

**Why It's Harmful:**

- Bypasses all business logic and validation
- Skips event dispatch (no observers execute)
- Doesn't update cached data
- Breaks data encapsulation
- Creates maintenance burden (schema changes break code)
- Violates Magento architecture principles

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;

class QuoteTotalService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository
    ) {}

    public function updateTotal(int $quoteId, float $total): void
    {
        // CORRECT: Use service contracts and models
        $quote = $this->cartRepository->get($quoteId);
        $quote->setGrandTotal($total);
        $quote->collectTotals(); // Recalculate dependent totals
        $this->cartRepository->save($quote);
    }
}
```

---

### Anti-Pattern 1.3: Modifying Quote Without collectTotals()

**Problem:**

```php
<?php
// WRONG: Changing items without recalculating totals
$quote = $this->cartRepository->get($quoteId);
$item = $quote->getItemById($itemId);
$item->setQty(10); // Changed quantity
$this->cartRepository->save($quote); // Saved without collectTotals()
```

**Why It's Harmful:**

- Quote totals become stale and incorrect
- Grand total doesn't reflect item changes
- Tax, shipping, discounts not recalculated
- Customer sees wrong prices in cart
- Order placement fails validation

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;

class UpdateItemQuantityService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository
    ) {}

    public function updateQuantity(int $quoteId, int $itemId, float $qty): void
    {
        $quote = $this->cartRepository->get($quoteId);
        $item = $quote->getItemById($itemId);

        if (!$item) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Item not found in cart.')
            );
        }

        $item->setQty($qty);

        // CORRECT: Always recalculate totals after modifications
        $quote->collectTotals();

        $this->cartRepository->save($quote);
    }
}
```

---

## 2. Totals Calculation Errors

### Anti-Pattern 2.1: Custom Totals Collector Without sort_order

**Problem:**

```xml
<!-- WRONG: No sort_order specified -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Sales:etc/sales.xsd">
    <section name="quote">
        <group name="totals">
            <item name="custom_fee" instance="Vendor\Module\Model\Quote\Address\Total\CustomFee"/>
        </group>
    </section>
</config>
```

**Why It's Harmful:**

- Unpredictable execution order
- May execute before required totals (e.g., before subtotal calculated)
- Different behavior across Magento versions
- Hard to debug totals issues
- Breaks in multi-module scenarios

**Correct Approach:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Sales:etc/sales.xsd">
    <section name="quote">
        <group name="totals">
            <!--
            CORRECT: Explicit sort_order
            Standard order:
            100 - Subtotal
            200 - Shipping
            300 - Tax
            400 - Discount
            500 - Grand Total
            -->
            <item name="custom_fee"
                  instance="Vendor\Module\Model\Quote\Address\Total\CustomFee"
                  sort_order="250"/>
        </group>
    </section>
</config>
```

---

### Anti-Pattern 2.2: Modifying Grand Total in Collector

**Problem:**

```php
<?php
// WRONG: Custom collector modifies grand_total directly
class CustomFee extends AbstractTotal
{
    public function collect(Quote $quote, ShippingAssignmentInterface $shippingAssignment, Total $total): self
    {
        $fee = 10.00;

        // WRONG: Don't modify grand_total directly
        $total->setGrandTotal($total->getGrandTotal() + $fee);

        return $this;
    }
}
```

**Why It's Harmful:**

- Grand total collector executes last and overwrites changes
- Breaks totals consistency
- Fee not included in proper total components
- Doesn't display correctly in cart/checkout
- Order totals mismatch quote totals

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Quote\Address\Total;

use Magento\Quote\Model\Quote;
use Magento\Quote\Api\Data\ShippingAssignmentInterface;
use Magento\Quote\Model\Quote\Address\Total;
use Magento\Quote\Model\Quote\Address\Total\AbstractTotal;

/**
 * Custom fee collector - CORRECT implementation
 */
class CustomFee extends AbstractTotal
{
    protected string $_code = 'custom_fee';

    public function collect(
        Quote $quote,
        ShippingAssignmentInterface $shippingAssignment,
        Total $total
    ): self {
        parent::collect($quote, $shippingAssignment, $total);

        $fee = 10.00;

        // CORRECT: Use setTotalAmount() - grand total collector will sum automatically
        $total->setTotalAmount($this->getCode(), $fee);
        $total->setBaseTotalAmount($this->getCode(), $fee);

        // Store for display
        $total->setCustomFeeAmount($fee);
        $total->setBaseCustomFeeAmount($fee);

        return $this;
    }

    /**
     * Fetch for display in cart
     *
     * @param Quote $quote
     * @param Total $total
     * @return array
     */
    public function fetch(Quote $quote, Total $total): array
    {
        return [
            'code' => $this->getCode(),
            'title' => __('Custom Fee'),
            'value' => $total->getCustomFeeAmount()
        ];
    }
}
```

---

### Anti-Pattern 2.3: Multiple collectTotals() Calls

**Problem:**

```php
<?php
// WRONG: Calling collectTotals() multiple times unnecessarily
$quote = $this->cartRepository->get($quoteId);

$quote->addProduct($product1);
$quote->collectTotals(); // Wasteful

$quote->addProduct($product2);
$quote->collectTotals(); // Wasteful

$quote->setCouponCode('SAVE10');
$quote->collectTotals(); // Wasteful

$this->cartRepository->save($quote);
```

**Why It's Harmful:**

- Totals collection is expensive (5-15ms per call)
- Executes all collectors multiple times unnecessarily
- Scales poorly with large carts
- Delays user experience
- Wastes server resources

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;

class BatchCartUpdateService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly ProductRepositoryInterface $productRepository
    ) {}

    public function addMultipleProducts(int $quoteId, array $productData): void
    {
        $quote = $this->cartRepository->get($quoteId);

        // Add all products first
        foreach ($productData as $data) {
            $product = $this->productRepository->getById($data['product_id']);
            $quote->addProduct($product, $data['qty']);
        }

        // Apply coupon if provided
        if (isset($productData['coupon_code'])) {
            $quote->setCouponCode($productData['coupon_code']);
        }

        // CORRECT: Collect totals once after all changes
        $quote->collectTotals();

        $this->cartRepository->save($quote);
    }
}
```

---

## 3. Quote State Management Issues

### Anti-Pattern 3.1: Not Validating Quote Ownership

**Problem:**

```php
<?php
// WRONG: Loading quote without validating customer ownership
public function updateCart(int $quoteId, int $itemId, float $qty): void
{
    $quote = $this->cartRepository->get($quoteId); // No ownership check
    $item = $quote->getItemById($itemId);
    $item->setQty($qty);
    $quote->collectTotals();
    $this->cartRepository->save($quote);
}
```

**Why It's Harmful:**

- Security vulnerability (customer can modify other customers' carts)
- CRITICAL: Allows cart manipulation attacks
- Violates data privacy
- PCI DSS compliance issue
- GDPR violation (accessing other customers' data)

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Customer\Model\Session as CustomerSession;
use Magento\Framework\Exception\LocalizedException;

class SecureCartUpdateService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly CustomerSession $customerSession
    ) {}

    public function updateCart(int $quoteId, int $itemId, float $qty): void
    {
        $quote = $this->cartRepository->get($quoteId);

        // CORRECT: Validate customer owns this quote
        $customerId = (int)$this->customerSession->getCustomerId();
        if ($quote->getCustomerId() !== $customerId) {
            throw new LocalizedException(__('Access denied.'));
        }

        $item = $quote->getItemById($itemId);
        if (!$item) {
            throw new LocalizedException(__('Item not found.'));
        }

        $item->setQty($qty);
        $quote->collectTotals();
        $this->cartRepository->save($quote);
    }
}
```

---

### Anti-Pattern 3.2: Reusing Converted Quotes

**Problem:**

```php
<?php
// WRONG: Trying to use quote after order placement
$quote = $this->cartRepository->get($quoteId);
$orderId = $this->cartManagement->placeOrder($quoteId);

// Quote is now inactive (is_active = 0)
$quote->addProduct($product); // Will fail or create issues
$quote->collectTotals();
$this->cartRepository->save($quote);
```

**Why It's Harmful:**

- Quote is marked inactive after order placement
- Reserved order ID already used
- Can create duplicate orders
- Data integrity issues
- Customer sees wrong cart state

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\CartManagementInterface;

class OrderPlacementService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly CartManagementInterface $cartManagement
    ) {}

    public function placeOrder(int $customerId): int
    {
        // Get quote before placing order
        $quote = $this->cartRepository->getForCustomer($customerId);
        $quoteId = $quote->getId();

        // Place order (quote becomes inactive)
        $orderId = $this->cartManagement->placeOrder($quoteId);

        // CORRECT: Create new cart for customer after order placement
        $newQuoteId = $this->cartManagement->createEmptyCartForCustomer($customerId);

        return $orderId;
    }
}
```

---

### Anti-Pattern 3.3: Not Handling Quote Merge Conflicts

**Problem:**

```php
<?php
// WRONG: Merging without checking for conflicts
public function mergeGuestCart(int $customerId, int $guestQuoteId): void
{
    $customerQuote = $this->cartRepository->getForCustomer($customerId);
    $guestQuote = $this->cartRepository->get($guestQuoteId);

    // WRONG: Blind merge without conflict resolution
    $customerQuote->merge($guestQuote);
    $this->cartRepository->save($customerQuote);
}
```

**Why It's Harmful:**

- May exceed cart item limits
- Duplicate items not handled properly
- Store mismatch issues
- Customer confusion
- Data loss on merge

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Framework\Exception\LocalizedException;

class CartMergeService
{
    private const MAX_ITEMS = 100;

    public function __construct(
        private readonly CartRepositoryInterface $cartRepository
    ) {}

    public function mergeGuestCart(int $customerId, int $guestQuoteId): void
    {
        $customerQuote = $this->cartRepository->getForCustomer($customerId);
        $guestQuote = $this->cartRepository->get($guestQuoteId);

        // CORRECT: Validate merge compatibility
        $this->validateMerge($customerQuote, $guestQuote);

        // Merge with conflict resolution
        $customerQuote->merge($guestQuote);
        $customerQuote->collectTotals();
        $this->cartRepository->save($customerQuote);

        // Delete guest quote
        $this->cartRepository->delete($guestQuote);
    }

    private function validateMerge($customerQuote, $guestQuote): void
    {
        // Check total items won't exceed limit
        $totalItems = $customerQuote->getItemsCount() + $guestQuote->getItemsCount();
        if ($totalItems > self::MAX_ITEMS) {
            throw new LocalizedException(
                __('Cannot merge carts: would exceed maximum of %1 items.', self::MAX_ITEMS)
            );
        }

        // Check stores match
        if ($customerQuote->getStoreId() !== $guestQuote->getStoreId()) {
            throw new LocalizedException(
                __('Cannot merge carts from different stores.')
            );
        }
    }
}
```

---

## 4. Performance Anti-Patterns

### Anti-Pattern 4.1: Loading Full Quote Collection

**Problem:**

```php
<?php
// WRONG: Loading all quotes to find one quote
$quoteCollection = $this->quoteCollectionFactory->create();
$quoteCollection->addFieldToFilter('reserved_order_id', $orderId);
$quotes = $quoteCollection->getItems(); // Loads ALL data
```

**Why It's Harmful:**

- Loads unnecessary data (items, addresses, payment)
- Slow query (missing indexes)
- High memory usage
- Scales poorly
- Database performance impact

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Model\ResourceModel\Quote\CollectionFactory;

class QuoteLookupService
{
    public function __construct(
        private readonly CollectionFactory $quoteCollectionFactory
    ) {}

    public function findByReservedOrderId(string $orderId): ?int
    {
        // CORRECT: Select only required fields, use limit
        $collection = $this->quoteCollectionFactory->create();
        $collection->addFieldToFilter('reserved_order_id', $orderId);
        $collection->addFieldToSelect('entity_id'); // Only load ID
        $collection->setPageSize(1); // Limit to 1 result

        $quote = $collection->getFirstItem();
        return $quote->getId() ?: null;
    }
}
```

---

### Anti-Pattern 4.2: N+1 Query Problem with Quote Items

**Problem:**

```php
<?php
// WRONG: Loading products in loop (N+1 queries)
$quote = $this->cartRepository->get($quoteId);
foreach ($quote->getAllItems() as $item) {
    // Each iteration triggers a product load query
    $product = $this->productRepository->getById($item->getProductId());
    $stockStatus = $product->getStockStatus();
}
```

**Why It's Harmful:**

- N+1 database queries (1 for quote + N for products)
- Severe performance degradation with large carts
- Timeout issues
- High database load
- Poor scalability

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Catalog\Model\ResourceModel\Product\CollectionFactory;

class QuoteProductStockService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly CollectionFactory $productCollectionFactory
    ) {}

    public function getStockStatus(int $quoteId): array
    {
        $quote = $this->cartRepository->get($quoteId);

        // Collect all product IDs
        $productIds = [];
        foreach ($quote->getAllItems() as $item) {
            $productIds[] = $item->getProductId();
        }

        // CORRECT: Load all products in single query
        $productCollection = $this->productCollectionFactory->create();
        $productCollection->addFieldToFilter('entity_id', ['in' => $productIds]);
        $productCollection->addAttributeToSelect('stock_status');

        // Build stock status map
        $stockStatus = [];
        foreach ($productCollection as $product) {
            $stockStatus[$product->getId()] = $product->getStockStatus();
        }

        return $stockStatus;
    }
}
```

---

### Anti-Pattern 4.3: Not Using Quote Item Collection Cache

**Problem:**

```php
<?php
// WRONG: Calling getAllItems() multiple times
$quote = $this->cartRepository->get($quoteId);

foreach ($quote->getAllItems() as $item) {
    // Process item
}

// Later in same request...
foreach ($quote->getAllItems() as $item) {
    // Process item again - reloads collection
}
```

**Why It's Harmful:**

- Multiple database queries for same data
- Unnecessary memory allocation
- Slower request processing
- Inefficient resource usage

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;

class QuoteItemProcessorService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository
    ) {}

    public function processItems(int $quoteId): void
    {
        $quote = $this->cartRepository->get($quoteId);

        // CORRECT: Load items once, store in variable
        $items = $quote->getAllItems();

        // First pass
        foreach ($items as $item) {
            $this->validateItem($item);
        }

        // Second pass - reuses same collection
        foreach ($items as $item) {
            $this->processItem($item);
        }
    }
}
```

---

## 5. Security Vulnerabilities

### Anti-Pattern 5.1: Trusting Client-Provided Prices

**Problem:**

```php
<?php
// WRONG: Accepting price from client request
public function addToCart(int $productId, float $qty, float $price): void
{
    $quote = $this->getCustomerQuote();
    $product = $this->productRepository->getById($productId);

    // CRITICAL VULNERABILITY: Never trust client price
    $product->setPrice($price);

    $quote->addProduct($product, $qty);
    $quote->collectTotals();
    $this->cartRepository->save($quote);
}
```

**Why It's Harmful:**

- CRITICAL security vulnerability
- Customers can set any price ($0.01 for expensive items)
- Revenue loss
- Fraud vector
- PCI DSS violation

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Quote\Api\CartManagementInterface;
use Magento\Customer\Model\Session;

class SecureAddToCartService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly ProductRepositoryInterface $productRepository,
        private readonly CartManagementInterface $cartManagement,
        private readonly Session $customerSession
    ) {}

    public function addToCart(int $productId, float $qty): void
    {
        $customerId = (int)$this->customerSession->getCustomerId();
        $quote = $this->cartManagement->getCartForCustomer($customerId);

        // CORRECT: Load product with server-side pricing
        $product = $this->productRepository->getById($productId, false, $quote->getStoreId());

        // Price comes from catalog, not client
        // Customer group pricing, special price, catalog rules all apply
        $quote->addProduct($product, $qty);
        $quote->collectTotals();
        $this->cartRepository->save($quote);
    }
}
```

---

### Anti-Pattern 5.2: Missing CSRF Protection

**Problem:**

```php
<?php
// WRONG: Controller without form key validation
class UpdateCart extends \Magento\Framework\App\Action\Action
{
    public function execute()
    {
        $cartData = $this->getRequest()->getParam('cart');

        // VULNERABILITY: No CSRF protection
        $this->updateCart($cartData);

        return $this->resultRedirectFactory->create()->setPath('checkout/cart');
    }
}
```

**Why It's Harmful:**

- CSRF (Cross-Site Request Forgery) vulnerability
- Attackers can trigger cart modifications via hidden forms
- Security compliance violation
- User account compromise

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Cart;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\App\CsrfAwareActionInterface;
use Magento\Framework\App\Request\InvalidRequestException;
use Magento\Framework\App\RequestInterface;

/**
 * CORRECT: Implement CsrfAwareActionInterface for API endpoints
 * OR use FormKey validator for form submissions
 */
class UpdateCart extends \Magento\Framework\App\Action\Action implements
    HttpPostActionInterface,
    CsrfAwareActionInterface
{
    public function __construct(
        \Magento\Framework\App\Action\Context $context,
        private readonly \Magento\Framework\Data\Form\FormKey\Validator $formKeyValidator
    ) {
        parent::__construct($context);
    }

    public function execute()
    {
        // CORRECT: Validate form key
        if (!$this->formKeyValidator->validate($this->getRequest())) {
            $this->messageManager->addErrorMessage(__('Invalid form key.'));
            return $this->resultRedirectFactory->create()->setPath('checkout/cart');
        }

        $cartData = $this->getRequest()->getParam('cart');
        $this->updateCart($cartData);

        return $this->resultRedirectFactory->create()->setPath('checkout/cart');
    }

    /**
     * Create CSRF validation exception
     */
    public function createCsrfValidationException(RequestInterface $request): ?InvalidRequestException
    {
        return new InvalidRequestException(
            $this->resultRedirectFactory->create()->setPath('checkout/cart'),
            [__('Invalid form key.')]
        );
    }

    /**
     * Validate for CSRF
     */
    public function validateForCsrf(RequestInterface $request): ?bool
    {
        return $this->formKeyValidator->validate($request);
    }
}
```

---

### Anti-Pattern 5.3: Exposing Internal Quote IDs

**Problem:**

```php
<?php
// WRONG: Using internal quote ID in URLs for guest carts
$checkoutUrl = $this->urlBuilder->getUrl('checkout/cart', [
    'quote_id' => $quote->getId() // Exposes internal ID
]);
```

**Why It's Harmful:**

- Quote ID enumeration attack vector
- Guests can access other guests' carts
- Security vulnerability
- Privacy violation
- Predictable IDs enable attacks

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Model\QuoteIdMaskFactory;
use Magento\Framework\UrlInterface;

class GuestCartUrlService
{
    public function __construct(
        private readonly QuoteIdMaskFactory $quoteIdMaskFactory,
        private readonly UrlInterface $urlBuilder
    ) {}

    public function getCheckoutUrl(int $quoteId): string
    {
        // CORRECT: Use masked ID for guest carts
        $quoteIdMask = $this->quoteIdMaskFactory->create();
        $quoteIdMask->load($quoteId, 'quote_id');

        if (!$quoteIdMask->getMaskedId()) {
            // Create masked ID if doesn't exist
            $maskedId = $quoteIdMask->createMaskedId($quoteId);
        } else {
            $maskedId = $quoteIdMask->getMaskedId();
        }

        // Use UUID instead of internal ID
        return $this->urlBuilder->getUrl('checkout/cart', [
            'cart_id' => $maskedId // UUID like "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
        ]);
    }
}
```

---

## 6. Data Integrity Problems

### Anti-Pattern 6.1: Not Validating Product Options

**Problem:**

```php
<?php
// WRONG: Adding configurable product without validating options
$product = $this->productRepository->getById($productId);
$quote->addProduct($product, $qty); // Missing required options
```

**Why It's Harmful:**

- Creates invalid quote items
- Order placement fails
- Customer frustration
- Data inconsistency
- Can't convert quote to order

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Framework\Exception\LocalizedException;

class AddConfigurableToCartService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly ProductRepositoryInterface $productRepository
    ) {}

    public function addConfigurable(
        int $quoteId,
        int $productId,
        float $qty,
        array $options
    ): void {
        $quote = $this->cartRepository->get($quoteId);
        $product = $this->productRepository->getById($productId);

        // CORRECT: Validate product type and required options
        if ($product->getTypeId() === 'configurable') {
            $this->validateConfigurableOptions($product, $options);
        }

        // Build buy request with options
        $buyRequest = new \Magento\Framework\DataObject([
            'product' => $productId,
            'qty' => $qty,
            'super_attribute' => $options
        ]);

        $result = $quote->addProduct($product, $buyRequest);

        // Check for error message
        if (is_string($result)) {
            throw new LocalizedException(__($result));
        }

        $quote->collectTotals();
        $this->cartRepository->save($quote);
    }

    private function validateConfigurableOptions($product, array $options): void
    {
        $configurableAttributes = $product->getTypeInstance()
            ->getConfigurableAttributes($product);

        foreach ($configurableAttributes as $attribute) {
            $attributeId = $attribute->getAttributeId();
            if (!isset($options[$attributeId])) {
                throw new LocalizedException(
                    __('Please specify required option: %1', $attribute->getLabel())
                );
            }
        }
    }
}
```

---

### Anti-Pattern 6.2: Not Handling Out of Stock Items

**Problem:**

```php
<?php
// WRONG: Adding items without stock validation
foreach ($productIds as $productId) {
    $product = $this->productRepository->getById($productId);
    $quote->addProduct($product, 1); // No stock check
}
```

**Why It's Harmful:**

- Customer adds out-of-stock items to cart
- Order placement fails
- Overselling issues
- Poor customer experience
- Inventory discrepancies

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\CatalogInventory\Api\StockRegistryInterface;
use Magento\Framework\Exception\LocalizedException;

class StockAwareAddToCartService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly ProductRepositoryInterface $productRepository,
        private readonly StockRegistryInterface $stockRegistry
    ) {}

    public function addProducts(int $quoteId, array $productData): void
    {
        $quote = $this->cartRepository->get($quoteId);

        foreach ($productData as $data) {
            $product = $this->productRepository->getById($data['product_id']);

            // CORRECT: Validate stock before adding
            $this->validateStock($product, $data['qty']);

            $result = $quote->addProduct($product, $data['qty']);

            if (is_string($result)) {
                throw new LocalizedException(__($result));
            }
        }

        $quote->collectTotals();
        $this->cartRepository->save($quote);
    }

    private function validateStock($product, float $qty): void
    {
        $stockItem = $this->stockRegistry->getStockItem($product->getId());

        // Check if in stock
        if (!$stockItem->getIsInStock()) {
            throw new LocalizedException(
                __('Product "%1" is out of stock.', $product->getName())
            );
        }

        // Check quantity availability
        if ($stockItem->getManageStock() && $qty > $stockItem->getQty()) {
            throw new LocalizedException(
                __(
                    'The requested qty of "%1" is not available. Available: %2',
                    $product->getName(),
                    $stockItem->getQty()
                )
            );
        }
    }
}
```

---

## Detection and Remediation

### Automated Detection

```bash
# Find direct save() calls on quote models
grep -r '\$quote->save()' app/code/

# Find direct database queries on quote table
grep -r "->update('quote'" app/code/

# Find missing collectTotals() calls
grep -B5 -A2 'cartRepository->save' app/code/ | grep -v collectTotals

# Find controllers without form key validation
grep -r "class.*extends.*Action" app/code/ | xargs grep -L "formKeyValidator"
```

### PHPStan Rules for Quote Anti-Patterns

```php
<?php
// Custom PHPStan rule to detect direct quote save

namespace Vendor\Module\PHPStan\Rules;

use PhpParser\Node;
use PHPStan\Analyser\Scope;
use PHPStan\Rules\Rule;

/**
 * Detect direct save() on Quote model
 *
 * @implements Rule<Node\Expr\MethodCall>
 */
class NoDirectQuoteSaveRule implements Rule
{
    public function getNodeType(): string
    {
        return Node\Expr\MethodCall::class;
    }

    public function processNode(Node $node, Scope $scope): array
    {
        if (!$node->name instanceof Node\Identifier) {
            return [];
        }

        if ($node->name->toString() !== 'save') {
            return [];
        }

        $callerType = $scope->getType($node->var);
        if ($callerType->getObjectClassNames() === ['Magento\\Quote\\Model\\Quote']) {
            return [
                'Direct save() on Quote model is not allowed. Use CartRepositoryInterface::save() instead.'
            ];
        }

        return [];
    }
}
```

---

**Assumptions:**
- Magento 2.4.7+ with PHP 8.2+
- Service contracts used for quote operations
- Form key validation on all POST requests
- Stock validation enabled

**Why This Approach:**
Catalogs real-world mistakes with concrete fixes. Provides detection strategies for existing codebases. Emphasizes security and data integrity alongside architectural concerns.

**Security Impact:**
- Missing CSRF protection enables attack vectors
- Trusting client prices is critical vulnerability
- Exposed internal IDs enable enumeration attacks
- Quote ownership validation prevents unauthorized access

**Performance Impact:**
- Multiple collectTotals() calls significantly degrade performance
- N+1 queries in loops cause severe slowdowns
- Full collection loads waste memory and database resources
- Proper optimization can improve cart operations by 5-10x

**Backward Compatibility:**
Fixing anti-patterns may require BC breaks if public APIs exposed problematic behavior. Use deprecation warnings and provide migration paths for major changes.

**Tests to Add:**
- Security tests for CSRF protection
- Integration tests for quote ownership validation
- Performance tests detecting N+1 queries
- Static analysis rules to prevent anti-patterns

**Docs to Update:**
- Best practices guide with anti-pattern examples
- Security checklist for quote operations
- Performance optimization guide
- Code review checklist
