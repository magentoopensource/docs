---
title: "Magento_Customer Integrations"
module: "Magento_Customer"
doc_type: "integrations"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Customer Integrations

## Overview

The Magento_Customer module serves as a foundational integration point for nearly every customer-facing feature in Adobe Commerce and Magento Open Source. This document catalogs critical integration patterns between Customer and other core modules, including Quote, Sales, Newsletter, Wishlist, Review, Tax, and Checkout.

Understanding these integrations is essential for building custom extensions that interact with customer data, implementing cross-module functionality, and debugging issues that span multiple modules.

**Target Versions:** Magento 2.4.7+ / Adobe Commerce 2.4.7+ with PHP 8.2+

---

## Table of Contents

1. [Integration Architecture Overview](#integration-architecture-overview)
2. [Customer ↔ Quote Integration](#customer--quote-integration)
3. [Customer ↔ Sales Integration](#customer--sales-integration)
4. [Customer ↔ Wishlist Integration](#customer--wishlist-integration)
5. [Customer ↔ Newsletter Integration](#customer--newsletter-integration)
6. [Customer ↔ Review Integration](#customer--review-integration)
7. [Customer ↔ Tax Integration](#customer--tax-integration)
8. [Customer ↔ Checkout Integration](#customer--checkout-integration)
9. [Customer ↔ CatalogPermissions (Adobe Commerce)](#customer--catalogpermissions-adobe-commerce)
10. [Customer ↔ Reward Points (Adobe Commerce)](#customer--reward-points-adobe-commerce)
11. [Customer ↔ Store Credit (Adobe Commerce)](#customer--store-credit-adobe-commerce)
12. [REST API Integrations](#rest-api-integrations)
13. [GraphQL Integrations](#graphql-integrations)

---

## Integration Architecture Overview

### Integration Patterns

The Customer module integrates with other modules through several patterns:

1. **Service Contract Dependencies:** Other modules depend on `CustomerRepositoryInterface` and `AccountManagementInterface`
2. **Event-Driven Integration:** Modules listen to customer events (`customer_save_after`, `customer_login`, etc.)
3. **Plugin-Based Extension:** Modules extend customer functionality via plugins
4. **Foreign Key Relationships:** Database tables link via `customer_id` foreign keys
5. **Session Context:** `Magento\Customer\Model\Session` provides customer context across modules
6. **Extension Attributes:** Modules add custom fields to `CustomerInterface` and `AddressInterface`

### Dependency Graph

```
Magento_Customer (core)
    ├─→ Magento_Quote (depends on Customer)
    │   └─→ Magento_Checkout
    ├─→ Magento_Sales (depends on Customer)
    ├─→ Magento_Wishlist (depends on Customer)
    ├─→ Magento_Newsletter (depends on Customer)
    ├─→ Magento_Review (depends on Customer)
    ├─→ Magento_Tax (uses customer groups)
    ├─→ Magento_CustomerBalance (Adobe Commerce, depends on Customer)
    └─→ Magento_Reward (Adobe Commerce, depends on Customer)
```

---

## Customer ↔ Quote Integration

### Overview

The Quote module manages shopping cart functionality and depends heavily on Customer for cart ownership, address population, and customer-specific pricing.

### Database Relationships

**quote table:**
- `customer_id` (FK → `customer_entity.entity_id`)
- `customer_email`
- `customer_firstname`
- `customer_lastname`
- `customer_group_id`

**quote_address table:**
- Links to `customer_address_entity` for address data

### Integration Points

#### 1. Cart Assignment to Customer

When a customer logs in, guest cart merges with customer cart:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Checkout\Model\Session as CheckoutSession;
use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Model\QuoteIdMaskFactory;

class CartMergeService
{
    public function __construct(
        private readonly CheckoutSession $checkoutSession,
        private readonly CartRepositoryInterface $cartRepository,
        private readonly QuoteIdMaskFactory $quoteIdMaskFactory
    ) {}

    /**
     * Merge guest cart with customer cart on login
     *
     * @param CustomerInterface $customer
     * @return void
     */
    public function mergeCartsOnLogin(CustomerInterface $customer): void
    {
        $quote = $this->checkoutSession->getQuote();

        if ($quote->getId() && !$quote->getCustomerId()) {
            // Assign guest cart to customer
            $quote->setCustomer($customer);
            $quote->setCustomerId((int)$customer->getId());
            $quote->setCustomerEmail($customer->getEmail());
            $quote->setCustomerFirstname($customer->getFirstname());
            $quote->setCustomerLastname($customer->getLastname());
            $quote->setCustomerGroupId((int)$customer->getGroupId());
            $quote->setCustomerIsGuest(0);

            $this->cartRepository->save($quote);
        }
    }
}
```

**Event Hook:** `customer_login` observer merges carts automatically.

#### 2. Customer Address Population

Customer addresses auto-populate shipping/billing forms:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\AddressRepositoryInterface;
use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Model\Quote\Address as QuoteAddress;

class QuoteAddressService
{
    public function __construct(
        private readonly CustomerRepositoryInterface $customerRepository,
        private readonly AddressRepositoryInterface $addressRepository,
        private readonly CartRepositoryInterface $cartRepository,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder
    ) {}

    /**
     * Populate quote addresses from customer default addresses
     *
     * @param int $customerId
     * @param int $quoteId
     * @return void
     */
    public function populateAddressesFromCustomer(int $customerId, int $quoteId): void
    {
        $customer = $this->customerRepository->getById($customerId);
        $quote = $this->cartRepository->get($quoteId);

        // Get customer addresses
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('parent_id', $customerId)
            ->create();
        $addresses = $this->addressRepository->getList($searchCriteria)->getItems();

        foreach ($addresses as $address) {
            if ($address->isDefaultBilling()) {
                $billingAddress = $quote->getBillingAddress();
                $this->populateQuoteAddress($billingAddress, $address);
            }

            if ($address->isDefaultShipping()) {
                $shippingAddress = $quote->getShippingAddress();
                $this->populateQuoteAddress($shippingAddress, $address);
            }
        }

        $this->cartRepository->save($quote);
    }

    private function populateQuoteAddress(
        QuoteAddress $quoteAddress,
        \Magento\Customer\Api\Data\AddressInterface $customerAddress
    ): void {
        $quoteAddress->setCustomerAddressId($customerAddress->getId());
        $quoteAddress->setFirstname($customerAddress->getFirstname());
        $quoteAddress->setLastname($customerAddress->getLastname());
        $quoteAddress->setStreet($customerAddress->getStreet());
        $quoteAddress->setCity($customerAddress->getCity());
        $quoteAddress->setRegionId($customerAddress->getRegionId());
        $quoteAddress->setPostcode($customerAddress->getPostcode());
        $quoteAddress->setCountryId($customerAddress->getCountryId());
        $quoteAddress->setTelephone($customerAddress->getTelephone());
    }
}
```

#### 3. Customer Group Pricing

Quote items reflect customer group pricing:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Customer\Model\Session as CustomerSession;
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

class ApplyCustomerGroupPricingObserver implements ObserverInterface
{
    public function __construct(
        private readonly CustomerSession $customerSession
    ) {}

    /**
     * Apply customer group pricing to quote
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        if ($this->customerSession->isLoggedIn()) {
            $customerGroupId = (int)$this->customerSession->getCustomerGroupId();
            $quote->setCustomerGroupId($customerGroupId);

            // Trigger price recalculation
            $quote->collectTotals();
        }
    }
}
```

---

## Customer ↔ Sales Integration

### Overview

Sales module links orders, invoices, shipments, and credit memos to customers for order history and reorder functionality.

### Database Relationships

**sales_order table:**
- `customer_id` (FK → `customer_entity.entity_id`)
- `customer_email`
- `customer_firstname`
- `customer_lastname`
- `customer_group_id`
- `customer_is_guest` (0 = registered, 1 = guest)

### Integration Points

#### 1. Order History Display

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Api\OrderRepositoryInterface;

class OrderHistoryService
{
    public function __construct(
        private readonly OrderRepositoryInterface $orderRepository,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder
    ) {}

    /**
     * Get customer order history
     *
     * @param int $customerId
     * @return OrderInterface[]
     */
    public function getCustomerOrders(int $customerId): array
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('customer_id', $customerId)
            ->addFilter('state', 'canceled', 'neq')
            ->addSortOrder('created_at', 'DESC')
            ->setPageSize(10)
            ->create();

        $orders = $this->orderRepository->getList($searchCriteria);
        return $orders->getItems();
    }
}
```

#### 2. Reorder Functionality

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartManagementInterface;
use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Sales\Model\Order\ItemFactory;

class ReorderService
{
    public function __construct(
        private readonly OrderRepositoryInterface $orderRepository,
        private readonly CartManagementInterface $cartManagement,
        private readonly CartRepositoryInterface $cartRepository,
        private readonly ItemFactory $orderItemFactory
    ) {}

    /**
     * Create new cart from previous order
     *
     * @param int $orderId
     * @param int $customerId
     * @return int Cart/Quote ID
     */
    public function reorder(int $orderId, int $customerId): int
    {
        $order = $this->orderRepository->get($orderId);

        // Verify order belongs to customer
        if ((int)$order->getCustomerId() !== $customerId) {
            throw new \Magento\Framework\Exception\AuthorizationException(
                __('Order does not belong to customer.')
            );
        }

        // Create new quote
        $quoteId = $this->cartManagement->createEmptyCartForCustomer($customerId);
        $quote = $this->cartRepository->get($quoteId);

        // Add items from order
        foreach ($order->getItems() as $orderItem) {
            if ($orderItem->getParentItem()) {
                continue; // Skip child items
            }

            $quote->addProduct(
                $orderItem->getProduct(),
                (float)$orderItem->getQtyOrdered()
            );
        }

        $this->cartRepository->save($quote);
        return $quoteId;
    }
}
```

#### 3. Customer Lifetime Value Calculation

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Sales\Api\OrderRepositoryInterface;

class CustomerLifetimeValueService
{
    public function __construct(
        private readonly OrderRepositoryInterface $orderRepository,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder
    ) {}

    /**
     * Calculate customer lifetime value
     *
     * @param int $customerId
     * @return float
     */
    public function calculateLifetimeValue(int $customerId): float
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('customer_id', $customerId)
            ->addFilter('state', ['complete', 'processing'], 'in')
            ->create();

        $orders = $this->orderRepository->getList($searchCriteria);

        $lifetimeValue = 0.0;
        foreach ($orders->getItems() as $order) {
            $lifetimeValue += (float)$order->getGrandTotal();
        }

        return $lifetimeValue;
    }
}
```

---

## Customer ↔ Wishlist Integration

### Overview

Wishlist module allows customers to save products for future purchase.

### Database Relationships

**wishlist table:**
- `customer_id` (FK → `customer_entity.entity_id`)

### Integration Points

#### 1. Wishlist Creation on Registration

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Wishlist\Model\WishlistFactory;

class CreateWishlistOnRegistrationObserver implements ObserverInterface
{
    public function __construct(
        private readonly WishlistFactory $wishlistFactory
    ) {}

    /**
     * Create wishlist for new customer
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var CustomerInterface $customer */
        $customer = $observer->getEvent()->getCustomer();

        $wishlist = $this->wishlistFactory->create();
        $wishlist->setCustomerId((int)$customer->getId());
        $wishlist->save();
    }
}
```

#### 2. Wishlist to Cart Transfer

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Checkout\Model\Cart;
use Magento\Wishlist\Model\ResourceModel\Item\CollectionFactory;

class WishlistToCartService
{
    public function __construct(
        private readonly CollectionFactory $wishlistItemCollectionFactory,
        private readonly Cart $cart
    ) {}

    /**
     * Move all wishlist items to cart
     *
     * @param int $customerId
     * @return int Number of items added
     */
    public function moveAllToCart(int $customerId): int
    {
        $collection = $this->wishlistItemCollectionFactory->create();
        $collection->addCustomerIdFilter($customerId);

        $itemsAdded = 0;
        foreach ($collection as $item) {
            try {
                $this->cart->addProduct($item->getProduct(), $item->getBuyRequest());
                $item->delete();
                $itemsAdded++;
            } catch (\Exception $e) {
                // Log error but continue processing
                continue;
            }
        }

        $this->cart->save();
        return $itemsAdded;
    }
}
```

---

## Customer ↔ Newsletter Integration

### Overview

Newsletter module manages email subscription preferences tied to customer accounts.

### Database Relationships

**newsletter_subscriber table:**
- `customer_id` (FK → `customer_entity.entity_id`)

### Integration Points

#### 1. Auto-Subscribe on Registration

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Newsletter\Model\SubscriberFactory;

class SubscribeNewsletterOnRegistrationObserver implements ObserverInterface
{
    public function __construct(
        private readonly SubscriberFactory $subscriberFactory
    ) {}

    /**
     * Subscribe customer to newsletter if checkbox checked
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var CustomerInterface $customer */
        $customer = $observer->getEvent()->getCustomer();

        // Check if customer opted in during registration
        $isSubscribed = $observer->getEvent()->getData('is_subscribed');

        if ($isSubscribed) {
            $subscriber = $this->subscriberFactory->create();
            $subscriber->subscribeCustomerById((int)$customer->getId());
        }
    }
}
```

#### 2. Sync Newsletter Status with Customer Account

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Newsletter\Model\SubscriberFactory;

class NewsletterSyncService
{
    public function __construct(
        private readonly SubscriberFactory $subscriberFactory
    ) {}

    /**
     * Check if customer is subscribed to newsletter
     *
     * @param int $customerId
     * @return bool
     */
    public function isSubscribed(int $customerId): bool
    {
        $subscriber = $this->subscriberFactory->create();
        $subscriber->loadByCustomerId($customerId);

        return $subscriber->isSubscribed();
    }

    /**
     * Update newsletter subscription
     *
     * @param int $customerId
     * @param bool $subscribe
     * @return void
     */
    public function updateSubscription(int $customerId, bool $subscribe): void
    {
        $subscriber = $this->subscriberFactory->create();
        $subscriber->loadByCustomerId($customerId);

        if ($subscribe && !$subscriber->isSubscribed()) {
            $subscriber->subscribeCustomerById($customerId);
        } elseif (!$subscribe && $subscriber->isSubscribed()) {
            $subscriber->unsubscribeCustomerById($customerId);
        }
    }
}
```

---

## Customer ↔ Review Integration

### Overview

Review module links product reviews to customer accounts for review history and moderation.

### Database Relationships

**review_detail table:**
- `customer_id` (FK → `customer_entity.entity_id`, nullable for guest reviews)

### Integration Points

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Review\Model\ResourceModel\Review\CollectionFactory;

class CustomerReviewService
{
    public function __construct(
        private readonly CollectionFactory $reviewCollectionFactory
    ) {}

    /**
     * Get customer's product reviews
     *
     * @param int $customerId
     * @return \Magento\Review\Model\ResourceModel\Review\Collection
     */
    public function getCustomerReviews(int $customerId)
    {
        $collection = $this->reviewCollectionFactory->create();
        $collection->addFieldToFilter('customer_id', $customerId)
            ->addStatusFilter(\Magento\Review\Model\Review::STATUS_APPROVED)
            ->setDateOrder();

        return $collection;
    }
}
```

---

## Customer ↔ Tax Integration

### Overview

Tax module uses customer group and address to determine applicable tax rates.

### Integration Points

#### 1. Customer Group Tax Class

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Api\GroupRepositoryInterface;
use Magento\Tax\Api\TaxCalculationInterface;

class CustomerTaxService
{
    public function __construct(
        private readonly CustomerRepositoryInterface $customerRepository,
        private readonly GroupRepositoryInterface $groupRepository,
        private readonly TaxCalculationInterface $taxCalculation
    ) {}

    /**
     * Get tax rate for customer
     *
     * @param int $customerId
     * @param int $productTaxClassId
     * @return float
     */
    public function getTaxRateForCustomer(int $customerId, int $productTaxClassId): float
    {
        $customer = $this->customerRepository->getById($customerId);
        $customerGroup = $this->groupRepository->getById((int)$customer->getGroupId());

        $request = new \Magento\Framework\DataObject([
            'country_id' => $customer->getAddresses()[0]->getCountryId() ?? 'US',
            'region_id' => $customer->getAddresses()[0]->getRegionId() ?? null,
            'postcode' => $customer->getAddresses()[0]->getPostcode() ?? null,
            'customer_class_id' => $customerGroup->getTaxClassId(),
            'product_class_id' => $productTaxClassId
        ]);

        return $this->taxCalculation->getCalculatedRate($productTaxClassId);
    }
}
```

---

## Customer ↔ Checkout Integration

### Overview

Checkout module orchestrates quote conversion to order, using customer data for order attribution and address population.

### Integration Points

**Multi-Address Checkout:** Customers can ship to multiple addresses:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\AddressRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Multishipping\Model\Checkout\Type\Multishipping;

class MultiAddressCheckoutService
{
    public function __construct(
        private readonly AddressRepositoryInterface $addressRepository,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder,
        private readonly Multishipping $multishipping
    ) {}

    /**
     * Get customer addresses for multi-shipping checkout
     *
     * @param int $customerId
     * @return \Magento\Customer\Api\Data\AddressInterface[]
     */
    public function getShippingAddresses(int $customerId): array
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('parent_id', $customerId)
            ->create();

        return $this->addressRepository->getList($searchCriteria)->getItems();
    }
}
```

---

## Customer ↔ CatalogPermissions (Adobe Commerce)

### Overview

Adobe Commerce B2B uses customer groups to control catalog visibility and pricing.

### Integration Points

**Shared Catalogs:** Customer groups assigned to shared catalogs see specific product sets and prices.

**Category Permissions:** Customer groups can be granted/denied access to specific categories.

---

## Customer ↔ Reward Points (Adobe Commerce)

### Overview

Adobe Commerce reward points system tracks points earned and redeemed by customers.

### Database Relationships

**magento_reward table:**
- `customer_id` (FK → `customer_entity.entity_id`)

### Integration Points

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Reward\Model\RewardFactory;

class RewardPointsService
{
    public function __construct(
        private readonly RewardFactory $rewardFactory
    ) {}

    /**
     * Get customer reward points balance
     *
     * @param int $customerId
     * @return int
     */
    public function getPointsBalance(int $customerId): int
    {
        $reward = $this->rewardFactory->create();
        $reward->setCustomerId($customerId);
        $reward->loadByCustomerId();

        return (int)$reward->getPointsBalance();
    }
}
```

---

## Customer ↔ Store Credit (Adobe Commerce)

### Overview

Adobe Commerce store credit allows customers to maintain credit balances for future purchases.

### Database Relationships

**magento_customerbalance table:**
- `customer_id` (FK → `customer_entity.entity_id`)

---

## REST API Integrations

### Customer Endpoints

**Create Customer:**
```http
POST /rest/V1/customers
Content-Type: application/json

{
  "customer": {
    "email": "customer@example.com",
    "firstname": "John",
    "lastname": "Doe",
    "websiteId": 1
  },
  "password": "SecurePass123!"
}
```

**Get Customer by ID:**
```http
GET /rest/V1/customers/123
Authorization: Bearer {admin_token}
```

**Get Current Customer:**
```http
GET /rest/V1/customers/me
Authorization: Bearer {customer_token}
```

**Update Customer:**
```http
PUT /rest/V1/customers/123
Authorization: Bearer {admin_token}
Content-Type: application/json

{
  "customer": {
    "id": 123,
    "email": "customer@example.com",
    "firstname": "John",
    "lastname": "Smith"
  }
}
```

---

## GraphQL Integrations

### Customer Queries

**Get Customer Data:**
```graphql
{
  customer {
    id
    email
    firstname
    lastname
    group_id
    addresses {
      id
      firstname
      lastname
      street
      city
      region {
        region_code
        region
      }
      postcode
      country_code
      telephone
      default_shipping
      default_billing
    }
  }
}
```

**Create Customer:**
```graphql
mutation {
  createCustomer(
    input: {
      email: "customer@example.com"
      firstname: "John"
      lastname: "Doe"
      password: "SecurePass123!"
    }
  ) {
    customer {
      id
      email
      firstname
      lastname
    }
  }
}
```

---

## Assumptions

- **Target Platform:** Adobe Commerce / Magento Open Source 2.4.7+
- **PHP Version:** 8.2+
- **Integration Scope:** Standard core module integrations
- **API Access:** REST/GraphQL endpoints enabled

## Why This Approach

These integration patterns follow Magento's service contract architecture, ensuring loose coupling between modules while maintaining data consistency. Event-driven integrations allow third-party modules to extend functionality without core modifications.

## Security Impact

- **Authorization:** Verify customer ownership before accessing related data (orders, wishlist, reviews)
- **API Access:** Customer tokens scoped to customer-owned resources
- **Data Exposure:** Integrate only necessary customer data; avoid over-fetching PII

## Performance Impact

- **Lazy Loading:** Load related data (addresses, orders) only when needed
- **Caching:** Use FPC for customer-independent data; private content for customer-specific data
- **Database Queries:** Use repositories with search criteria instead of loading entire collections

## Backward Compatibility

- **Service Contracts:** Integration interfaces stable across 2.4.x
- **Database Schema:** Foreign key relationships maintained
- **Events:** Integration events never removed

## Tests to Add

- **Integration:** Test cross-module workflows (cart merge, order history, wishlist)
- **API:** Test REST/GraphQL endpoints for customer and related resources
- **MFTF:** Test end-to-end flows (login → add to cart → checkout)

## Docs to Update

- **README.md:** Overview of module integrations
- **INTEGRATIONS.md:** This document
- **API documentation:** REST/GraphQL endpoint examples
