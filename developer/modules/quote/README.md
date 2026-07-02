---
title: "Magento_Quote Overview"
module: "Magento_Quote"
doc_type: "overview"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Quote Module

## Overview

The `Magento_Quote` module is the foundational component of Adobe Commerce and Magento Open Source that manages shopping cart functionality, quote lifecycle, and the transition from cart to order. It provides the service contracts, data models, and business logic for cart operations, totals calculation, address management, payment/shipping method selection, and quote-to-order conversion.

**Target Version:** Magento 2.4.7+ | Adobe Commerce & Open Source
**PHP Version:** 8.2+
**Module Type:** Core Commerce Module

## Module Purpose

The Quote module serves as the central hub for pre-order business logic:

1. **Cart Management** - Create, retrieve, update, and delete quotes (shopping carts)
2. **Item Management** - Add products to cart, update quantities, remove items, configure options
3. **Address Management** - Handle billing/shipping addresses with validation
4. **Totals Calculation** - Orchestrate complex totals calculation through collectors
5. **Quote Lifecycle** - Manage quote states: active, merged, converted, expired
6. **Checkout Preparation** - Validate quote readiness for order placement
7. **Quote-to-Order Conversion** - Convert validated quotes into orders
8. **Guest Cart Handling** - Anonymous shopping with cart merging on login
9. **Multi-address Checkout** - Support shipping to multiple addresses

## Key Features

### Cart Operations

- **Add to Cart** - Product addition with configurable/bundle/grouped/simple support
- **Update Cart** - Quantity changes, option modifications, item removal
- **Cart Validation** - Stock validation, price verification, product availability
- **Cart Merge** - Merge guest cart into customer cart on login
- **Cart Persistence** - Save inactive carts, restore abandoned carts
- **Mini Cart** - Real-time cart summary with AJAX updates

### Totals Management

- **Modular Collectors** - Subtotal, tax, shipping, discount, grand total
- **Collector Priority** - Ordered execution with dependency resolution
- **Address-level Totals** - Separate totals for billing/shipping addresses
- **Custom Totals** - Extension points for custom fees/adjustments
- **Totals Caching** - Performance optimization for complex calculations

### Quote Addresses

- **Address Validation** - Required fields, format validation, region/country checks
- **Default Addresses** - Customer default address integration
- **Address Estimation** - Shipping cost estimation without full address
- **Multi-address Support** - Different shipping addresses per item group

### Payment and Shipping

- **Method Selection** - Choose payment/shipping methods
- **Method Availability** - Filter methods by cart contents, customer group, address
- **Method Validation** - Verify method applicability before order placement
- **Additional Data** - Store method-specific configuration (PO numbers, delivery instructions)

## Module Structure

```
Magento/Quote/
├── Api/                          # Service contracts
│   ├── CartManagementInterface.php
│   ├── CartRepositoryInterface.php
│   ├── CartItemRepositoryInterface.php
│   ├── CartTotalRepositoryInterface.php
│   ├── GuestCartManagementInterface.php
│   ├── PaymentMethodManagementInterface.php
│   └── ShipmentEstimationInterface.php
├── Model/
│   ├── Quote.php                 # Core quote entity
│   ├── Quote/
│   │   ├── Address.php           # Quote address entity
│   │   ├── Item.php              # Quote item entity
│   │   ├── Payment.php           # Payment information
│   │   ├── TotalsCollector.php   # Totals orchestrator
│   │   ├── Address/
│   │   │   ├── Total.php         # Address total model
│   │   │   └── Total/Collector.php
│   │   └── Item/
│   │       ├── AbstractItem.php
│   │       ├── Option.php        # Item options/configuration
│   │       └── Processor.php     # Item processing logic
│   ├── QuoteManagement.php       # Cart management implementation
│   ├── QuoteRepository.php       # Quote persistence
│   ├── GuestCart/                # Guest cart services
│   │   ├── GuestCartManagement.php
│   │   └── GuestCartRepository.php
│   └── ResourceModel/
│       ├── Quote.php             # Quote resource model
│       └── Quote/
│           ├── Collection.php
│           ├── Address.php
│           └── Item.php
├── Observer/                     # Event observers
│   ├── SubmitObserver.php        # Quote-to-order conversion
│   ├── Frontend/
│   │   └── QuoteLoadObserver.php # Load customer quote
│   └── Backend/
│       └── CustomerQuoteObserver.php
└── Setup/
    ├── Patch/
    │   └── Data/                 # Data patches
    └── db_schema.xml             # Declarative schema
```

## Core Service Contracts

### CartRepositoryInterface

Primary interface for quote persistence operations.

```php
<?php
namespace Magento\Quote\Api;

/**
 * Cart repository interface (actual 2.4.7 signatures — no PHP return types)
 *
 * @api
 * @since 100.0.2
 */
interface CartRepositoryInterface
{
    /**
     * @param int $cartId
     * @return \Magento\Quote\Api\Data\CartInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function get($cartId);

    /**
     * @param \Magento\Framework\Api\SearchCriteriaInterface $searchCriteria
     * @return \Magento\Quote\Api\Data\CartSearchResultsInterface
     */
    public function getList(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria);

    /**
     * @param int $customerId
     * @param int[] $sharedStoreIds
     * @return \Magento\Quote\Api\Data\CartInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function getForCustomer($customerId, array $sharedStoreIds = []);

    /**
     * @param int $cartId
     * @param int[] $sharedStoreIds
     * @return \Magento\Quote\Api\Data\CartInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function getActive($cartId, array $sharedStoreIds = []);

    /**
     * @param int $customerId
     * @param int[] $sharedStoreIds
     * @return \Magento\Quote\Api\Data\CartInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function getActiveForCustomer($customerId, array $sharedStoreIds = []);

    /**
     * @param \Magento\Quote\Api\Data\CartInterface $quote
     * @return void
     */
    public function save(\Magento\Quote\Api\Data\CartInterface $quote);

    /**
     * @param \Magento\Quote\Api\Data\CartInterface $quote
     * @return void
     */
    public function delete(\Magento\Quote\Api\Data\CartInterface $quote);
}
```

### CartManagementInterface

Manages cart lifecycle and operations.

```php
<?php
namespace Magento\Quote\Api;

/**
 * Cart management interface (actual 2.4.7 signatures — no PHP return types)
 *
 * @api
 * @since 100.0.2
 */
interface CartManagementInterface
{
    /** @return int Cart ID */
    public function createEmptyCart();

    /** @return int Cart ID */
    public function createEmptyCartForCustomer($customerId);

    /** @return \Magento\Quote\Api\Data\CartInterface */
    public function getCartForCustomer($customerId);

    /** @return bool */
    public function assignCustomer($cartId, $customerId, $storeId);

    /** @return int Order ID */
    public function placeOrder($cartId, \Magento\Quote\Api\Data\PaymentInterface $paymentMethod = null);
}
```

### CartItemRepositoryInterface

Manages items within the cart.

```php
<?php
namespace Magento\Quote\Api;

/**
 * Cart item repository interface (actual 2.4.7 signatures — no PHP return types)
 *
 * @api
 * @since 100.0.2
 */
interface CartItemRepositoryInterface
{
    /** @return \Magento\Quote\Api\Data\CartItemInterface[] */
    public function getList($cartId);

    /** @return \Magento\Quote\Api\Data\CartItemInterface */
    public function save(\Magento\Quote\Api\Data\CartItemInterface $cartItem);

    /** @return bool */
    public function deleteById($cartId, $itemId);
}
```

## Data Models

### Quote (CartInterface)

```php
<?php
namespace Magento\Quote\Api\Data;

/**
 * Cart data interface (actual 2.4.7 — no PHP return types, only @return docblocks)
 *
 * @api
 * @since 100.0.2
 */
interface CartInterface extends \Magento\Framework\Api\ExtensibleDataInterface
{
    // Identity
    /** @return int|null */
    public function getId();
    /** @return $this */
    public function setId($id);

    // Customer association
    /** @return int|null */
    public function getCustomerId();
    /** @return $this */
    public function setCustomerId($customerId);
    /** @return string|null */
    public function getCustomerEmail();
    /** @return $this */
    public function setCustomerEmail($customerEmail);
    /** @return bool|null */
    public function getCustomerIsGuest();
    /** @return $this */
    public function setCustomerIsGuest($isGuest);

    // Store context
    /** @return int */
    public function getStoreId();
    /** @return $this */
    public function setStoreId($storeId);

    // Quote state
    /** @return bool|null */
    public function getIsActive();
    /** @return $this */
    public function setIsActive($isActive);
    /** @return bool */
    public function getIsVirtual();

    // Items
    /** @return \Magento\Quote\Api\Data\CartItemInterface[]|null */
    public function getItems();
    /** @return $this */
    public function setItems(array $items);
    /** @return int */
    public function getItemsCount();
    /** @return float */
    public function getItemsQty();

    // Totals
    /** @return float|null */
    public function getGrandTotal();
    /** @return float|null */
    public function getBaseGrandTotal();
    /** @return float|null */
    public function getSubtotal();
    /** @return float|null */
    public function getBaseSubtotal();

    // Currency
    /** @return string|null */
    public function getBaseCurrencyCode();
    /** @return string|null */
    public function getQuoteCurrencyCode();
    /** @return float|null */
    public function getStoreToBaseRate();
    /** @return float|null */
    public function getStoreToQuoteRate();

    // Addresses
    /** @return \Magento\Quote\Api\Data\AddressInterface|null */
    public function getBillingAddress();
    /** @return $this */
    public function setBillingAddress(\Magento\Quote\Api\Data\AddressInterface $billingAddress = null);

    // Dates
    /** @return string|null */
    public function getCreatedAt();
    /** @return string|null */
    public function getUpdatedAt();

    // Reserved order ID
    /** @return string|null */
    public function getReservedOrderId();
    /** @return $this */
    public function setReservedOrderId($reservedOrderId);
}
```

### Quote Item (CartItemInterface)

```php
<?php
namespace Magento\Quote\Api\Data;

/**
 * Cart item data interface (actual 2.4.7 — no PHP return types)
 *
 * @api
 * @since 100.0.2
 */
interface CartItemInterface extends \Magento\Framework\Api\ExtensibleDataInterface
{
    /** @return int|null */
    public function getItemId();
    /** @return $this */
    public function setItemId($itemId);

    /** @return string */
    public function getSku();
    /** @return $this */
    public function setSku($sku);

    /** @return string */
    public function getQuoteId();
    /** @return $this */
    public function setQuoteId($quoteId);

    /** @return string|null */
    public function getProductType();
    /** @return $this */
    public function setProductType($productType);

    /** @return float */
    public function getQty();
    /** @return $this */
    public function setQty($qty);

    /** @return float */
    public function getPrice();
    /** @return $this */
    public function setPrice($price);

    /** @return string|null */
    public function getName();
    /** @return $this */
    public function setName($name);
}
```

## Common Use Cases

### Add Product to Cart

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\CartManagementInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Framework\Exception\LocalizedException;

class AddToCartService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly CartManagementInterface $cartManagement,
        private readonly ProductRepositoryInterface $productRepository
    ) {}

    /**
     * Add product to customer cart
     *
     * @param int $customerId
     * @param int $productId
     * @param float $qty
     * @param array $options
     * @return int Item ID
     * @throws LocalizedException
     */
    public function addToCart(
        int $customerId,
        int $productId,
        float $qty,
        array $options = []
    ): int {
        // Get or create cart
        $quote = $this->cartManagement->getCartForCustomer($customerId);

        // Load product
        $product = $this->productRepository->getById($productId);

        // Prepare buy request
        $buyRequest = new \Magento\Framework\DataObject([
            'qty' => $qty,
            'product' => $productId,
            'options' => $options
        ]);

        // Add product to quote
        $item = $quote->addProduct($product, $buyRequest);

        if (is_string($item)) {
            throw new LocalizedException(__($item));
        }

        // Save quote and recalculate totals
        $quote->collectTotals();
        $this->cartRepository->save($quote);

        return (int)$item->getId();
    }
}
```

### Update Item Quantity

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\CartItemRepositoryInterface;
use Magento\Framework\Exception\LocalizedException;

class UpdateCartService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly CartItemRepositoryInterface $itemRepository
    ) {}

    /**
     * Update item quantity
     *
     * @param int $cartId
     * @param int $itemId
     * @param float $qty
     * @return void
     * @throws LocalizedException
     */
    public function updateQuantity(int $cartId, int $itemId, float $qty): void
    {
        $quote = $this->cartRepository->get($cartId);
        $item = $quote->getItemById($itemId);

        if (!$item) {
            throw new LocalizedException(__('Item not found in cart.'));
        }

        if ($qty <= 0) {
            $quote->removeItem($itemId);
        } else {
            $item->setQty($qty);
        }

        $quote->collectTotals();
        $this->cartRepository->save($quote);
    }
}
```

### Apply Coupon Code

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\CouponManagementInterface;
use Magento\Framework\Exception\LocalizedException;

class CouponService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly CouponManagementInterface $couponManagement
    ) {}

    /**
     * Apply coupon code to cart
     *
     * @param int $cartId
     * @param string $couponCode
     * @return bool
     * @throws LocalizedException
     */
    public function applyCoupon(int $cartId, string $couponCode): bool
    {
        $this->couponManagement->set($cartId, $couponCode);

        $quote = $this->cartRepository->get($cartId);
        $quote->collectTotals();
        $this->cartRepository->save($quote);

        // Verify coupon was applied
        if ($quote->getCouponCode() !== $couponCode) {
            throw new LocalizedException(__('Coupon code "%1" is not valid.', $couponCode));
        }

        return true;
    }
}
```

## Extension Points

### Custom Totals Collector

Add custom fees or adjustments to cart totals.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Quote\Address\Total;

use Magento\Quote\Model\Quote;
use Magento\Quote\Api\Data\ShippingAssignmentInterface;
use Magento\Quote\Model\Quote\Address\Total;
use Magento\Quote\Model\Quote\Address\Total\AbstractTotal;

class CustomFee extends AbstractTotal
{
    /**
     * Collect custom fee total
     *
     * @param Quote $quote
     * @param ShippingAssignmentInterface $shippingAssignment
     * @param Total $total
     * @return $this
     */
    public function collect(
        Quote $quote,
        ShippingAssignmentInterface $shippingAssignment,
        Total $total
    ): self {
        parent::collect($quote, $shippingAssignment, $total);

        $address = $shippingAssignment->getShipping()->getAddress();

        // Calculate custom fee (example: 5% of subtotal)
        $subtotal = $total->getSubtotal();
        $customFee = $subtotal * 0.05;

        // Add to totals
        $total->setTotalAmount('custom_fee', $customFee);
        $total->setBaseTotalAmount('custom_fee', $customFee);
        $total->setCustomFeeAmount($customFee);
        $total->setBaseCustomFeeAmount($customFee);

        return $this;
    }

    /**
     * Fetch totals for display
     *
     * @param Quote $quote
     * @param Total $total
     * @return array
     */
    public function fetch(Quote $quote, Total $total): array
    {
        return [
            'code' => 'custom_fee',
            'title' => __('Custom Fee'),
            'value' => $total->getCustomFeeAmount()
        ];
    }
}
```

Register in `etc/sales.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Sales:etc/sales.xsd">
    <section name="quote">
        <group name="totals">
            <item name="custom_fee" instance="Vendor\Module\Model\Quote\Address\Total\CustomFee" sort_order="450"/>
        </group>
    </section>
</config>
```

## Database Schema

### Core Tables

- **`quote`** - Main quote entity (cart header)
- **`quote_item`** - Items in cart
- **`quote_address`** - Billing and shipping addresses
- **`quote_address_item`** - Address-item associations (multi-address)
- **`quote_item_option`** - Configurable/bundle product options
- **`quote_payment`** - Payment method information
- **`quote_shipping_rate`** - Available shipping rates
- **`quote_id_mask`** - Guest cart masked IDs

### Key Relationships

```
quote (1) ----< (many) quote_item
quote (1) ----< (many) quote_address
quote (1) ---- (1) quote_payment
quote_item (1) ----< (many) quote_item_option
quote_address (1) ----< (many) quote_shipping_rate
```

## Events

### Cart Events

- **`checkout_cart_add_product_complete`** - After product added to cart
- **`checkout_cart_update_items_after`** - After cart items updated
- **`checkout_cart_product_add_before`** - Before product added to cart
- **`sales_quote_remove_item`** - When item removed from cart
- **`sales_quote_collect_totals_before`** - Before totals collection
- **`sales_quote_collect_totals_after`** - After totals collection

### Quote Events

- **`sales_quote_save_before`** - Before quote saved
- **`sales_quote_save_after`** - After quote saved
- **`sales_quote_load_after`** - After quote loaded
- **`sales_quote_merge_before`** - Before merging quotes
- **`sales_quote_merge_after`** - After merging quotes

## Configuration

### System Configuration Paths

```php
// Shopping cart settings
Stores > Configuration > Sales > Checkout > Shopping Cart

// Quote lifetime (days)
'checkout/cart/delete_quote_after'

// Grouped product image
'checkout/cart/grouped_product_image'

// Configurable product image
'checkout/cart/configurable_product_image'

// Enable multi-address checkout
'multishipping/options/checkout_multiple'

// Number of allowed addresses
'multishipping/options/checkout_multiple_maximum_qty'
```

## Dependencies

### Required Modules

- `Magento_Store` - Store and website context
- `Magento_Customer` - Customer data and addresses
- `Magento_Catalog` - Product information and pricing
- `Magento_Sales` - Order conversion
- `Magento_Tax` - Tax calculation
- `Magento_Directory` - Country/region validation

### Optional Dependencies

- `Magento_SalesRule` - Coupon and promotion support
- `Magento_Checkout` - Checkout UI components
- `Magento_Payment` - Payment method integration
- `Magento_Shipping` - Shipping method integration

## Testing

### Unit Test Example

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Unit\Service;

use PHPUnit\Framework\TestCase;
use Vendor\Module\Service\AddToCartService;
use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\CartManagementInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Quote\Model\Quote;
use Magento\Catalog\Model\Product;

class AddToCartServiceTest extends TestCase
{
    private AddToCartService $service;
    private CartRepositoryInterface $cartRepository;
    private CartManagementInterface $cartManagement;
    private ProductRepositoryInterface $productRepository;

    protected function setUp(): void
    {
        $this->cartRepository = $this->createMock(CartRepositoryInterface::class);
        $this->cartManagement = $this->createMock(CartManagementInterface::class);
        $this->productRepository = $this->createMock(ProductRepositoryInterface::class);

        $this->service = new AddToCartService(
            $this->cartRepository,
            $this->cartManagement,
            $this->productRepository
        );
    }

    public function testAddToCart(): void
    {
        $customerId = 1;
        $productId = 123;
        $qty = 2.0;

        $quote = $this->createMock(Quote::class);
        $product = $this->createMock(Product::class);
        $item = $this->createMock(\Magento\Quote\Model\Quote\Item::class);

        $this->cartManagement->expects($this->once())
            ->method('getCartForCustomer')
            ->with($customerId)
            ->willReturn($quote);

        $this->productRepository->expects($this->once())
            ->method('getById')
            ->with($productId)
            ->willReturn($product);

        $quote->expects($this->once())
            ->method('addProduct')
            ->willReturn($item);

        $item->expects($this->once())
            ->method('getId')
            ->willReturn(456);

        $quote->expects($this->once())->method('collectTotals');
        $this->cartRepository->expects($this->once())->method('save')->with($quote);

        $result = $this->service->addToCart($customerId, $productId, $qty);

        $this->assertEquals(456, $result);
    }
}
```

## Further Reading

- **Architecture Documentation** - [architecture.md](./architecture.md)
- **Execution Flows** - [execution-flows.md](./execution-flows.md)
- **Plugin and Observer Guide** - [plugins-and-observers.md](./plugins-and-observers.md)
- **Integration Guide** - [integrations.md](./integrations.md)
- **Anti-Patterns** - [anti-patterns.md](./anti-patterns.md)
- **Performance Guide** - [performance.md](./performance.md)

---

**Assumptions:**
- Magento 2.4.7+ with PHP 8.2+
- Adobe Commerce or Open Source
- Standard cart configuration (single-address checkout default)
- Full Page Cache (FPC) enabled

**Why This Approach:**
The Quote module uses service contracts extensively to maintain API stability and enable web API exposure. The totals collector pattern allows modular, extensible calculation logic. Repository pattern abstracts persistence concerns.

**Security Impact:**
- All cart operations must verify customer ownership (customer ID matching)
- Guest carts use masked IDs to prevent enumeration attacks
- CSRF protection required on all cart modification forms
- Price validation occurs server-side (never trust client prices)

**Performance Impact:**
- Totals collection is expensive; cache quote totals when possible
- Quote operations invalidate FPC for customer-specific sections
- Use section data for mini cart updates (avoid full page reload)
- Index quote items for fast cart retrieval

**Backward Compatibility:**
Service contracts provide API stability across minor versions. Custom totals collectors and quote plugins are upgrade-safe. Direct model manipulation may break across versions.

**Tests to Add:**
- Unit tests for service implementations
- Integration tests for totals collectors
- API functional tests for cart operations
- MFTF tests for add-to-cart user flows

**Docs to Update:**
- README.md (this file) - Keep use cases current
- Developer documentation for custom totals
- API documentation for REST/GraphQL endpoints

## Related Guides

- [B2B Features Development](../../guides/explanations/b2b-features.md)
- [Service Contracts vs Repositories in Magento 2](../../guides/explanations/service-contracts-repositories.md)
- [Upgrade Guide: Adobe Commerce 2.4.6 to 2.4.7/2.4.8](../../guides/references/upgrade-guide-247-248.md)
- [Custom Payment Method Development: Building Secure Payment Integrations](../../guides/tutorials/custom-payment-method.md)
- [Custom Shipping Method Development: Building Flexible Shipping Solutions](../../guides/tutorials/custom-shipping-method.md)
