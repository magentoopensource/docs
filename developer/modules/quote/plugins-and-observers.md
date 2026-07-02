---
title: "Magento_Quote Plugins & Observers"
module: "Magento_Quote"
doc_type: "plugins-and-observers"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Quote Plugins and Observers

## Overview

The Magento_Quote module provides extensive extension points through plugins (interceptors) and observers (event listeners). This document catalogs all major events, demonstrates plugin patterns, and provides production-ready examples for common customizations.

**Target Version:** Magento 2.4.7+ | Adobe Commerce & Open Source
**PHP Version:** 8.2+

## Plugin Architecture

### Why Plugins Over Preferences

Plugins (interceptors) are the preferred extension mechanism in Magento 2 because they:

1. **Preserve Upgrade Path** - Multiple plugins can modify the same method without conflicts
2. **Maintain BC** - Don't require class inheritance, so base class changes don't break customizations
3. **Support Multiple Extensions** - Multiple modules can plugin the same method
4. **Enable Debugging** - Plugin chain is visible in di:compile output

### Plugin Types

```php
// Before plugin - modify arguments before method execution
public function beforeMethodName($subject, $arg1, $arg2, ...): array

// Around plugin - wrap method execution, control flow
public function aroundMethodName($subject, callable $proceed, $arg1, $arg2, ...)

// After plugin - modify return value after method execution
public function afterMethodName($subject, $result, $arg1, $arg2, ...)
```

### Plugin Registration

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Quote\Api\CartRepositoryInterface">
        <plugin name="vendor_module_quote_repository"
                type="Vendor\Module\Plugin\Quote\QuoteRepositoryExtend"
                sortOrder="10"
                disabled="false"/>
    </type>
</config>
```

## Core Quote Plugins

### 1. Quote Repository Plugins

#### Validate Business Rules Before Save

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\Data\CartInterface;
use Magento\Framework\Exception\LocalizedException;

/**
 * Quote repository validation plugin
 *
 * Enforces custom business rules before quote save.
 */
class QuoteRepositoryExtend
{
    private const MAX_ITEMS_PER_CART = 100;
    private const MAX_QUANTITY_PER_ITEM = 999;

    public function __construct(
        private readonly \Psr\Log\LoggerInterface $logger,
        private readonly \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig
    ) {}

    /**
     * Validate quote before save
     *
     * @param CartRepositoryInterface $subject
     * @param CartInterface $quote
     * @return array
     * @throws LocalizedException
     */
    public function beforeSave(
        CartRepositoryInterface $subject,
        CartInterface $quote
    ): array {
        // Validate item count
        if ($quote->getItemsCount() > self::MAX_ITEMS_PER_CART) {
            throw new LocalizedException(
                __('Cart cannot contain more than %1 items.', self::MAX_ITEMS_PER_CART)
            );
        }

        // Validate individual item quantities
        foreach ($quote->getAllItems() as $item) {
            if ($item->getQty() > self::MAX_QUANTITY_PER_ITEM) {
                throw new LocalizedException(
                    __(
                        'Item "%1" quantity cannot exceed %2.',
                        $item->getName(),
                        self::MAX_QUANTITY_PER_ITEM
                    )
                );
            }
        }

        // Validate minimum order amount
        $minOrderAmount = $this->scopeConfig->getValue(
            'sales/minimum_order/amount',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        );

        if ($minOrderAmount && $quote->getGrandTotal() < $minOrderAmount) {
            throw new LocalizedException(
                __('Minimum order amount is %1.', $minOrderAmount)
            );
        }

        return [$quote];
    }

    /**
     * Log quote save for audit trail
     *
     * @param CartRepositoryInterface $subject
     * @param void $result
     * @param CartInterface $quote
     * @return void
     */
    public function afterSave(
        CartRepositoryInterface $subject,
        $result,
        CartInterface $quote
    ): void {
        $this->logger->info('Quote saved', [
            'quote_id' => $quote->getId(),
            'customer_id' => $quote->getCustomerId(),
            'items_count' => $quote->getItemsCount(),
            'grand_total' => $quote->getGrandTotal()
        ]);
    }
}
```

#### Add Extension Attributes After Load

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\Data\CartInterface;

/**
 * Populate custom extension attributes on quote load
 */
class AddExtensionAttributesExtend
{
    public function __construct(
        private readonly \Vendor\Module\Model\LoyaltyPointsCalculator $loyaltyCalculator,
        private readonly \Magento\Quote\Api\Data\CartExtensionFactory $cartExtensionFactory
    ) {}

    /**
     * Add loyalty points extension attribute
     *
     * @param CartRepositoryInterface $subject
     * @param CartInterface $quote
     * @return CartInterface
     */
    public function afterGet(
        CartRepositoryInterface $subject,
        CartInterface $quote
    ): CartInterface {
        $extensionAttributes = $quote->getExtensionAttributes()
            ?? $this->cartExtensionFactory->create();

        // Calculate loyalty points earned for this cart
        $loyaltyPoints = $this->loyaltyCalculator->calculatePoints($quote);
        $extensionAttributes->setLoyaltyPoints($loyaltyPoints);

        $quote->setExtensionAttributes($extensionAttributes);

        return $quote;
    }

    /**
     * Also add extension attributes for customer quote
     *
     * @param CartRepositoryInterface $subject
     * @param CartInterface $quote
     * @return CartInterface
     */
    public function afterGetForCustomer(
        CartRepositoryInterface $subject,
        CartInterface $quote
    ): CartInterface {
        return $this->afterGet($subject, $quote);
    }
}
```

### 2. Cart Management Plugins

#### Prevent Cart Creation for Blocked Customers

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Api\CartManagementInterface;
use Magento\Framework\Exception\LocalizedException;

/**
 * Validate customer can create cart
 */
class CartManagementExtend
{
    public function __construct(
        private readonly \Magento\Customer\Api\CustomerRepositoryInterface $customerRepository
    ) {}

    /**
     * Prevent cart creation for blocked customers
     *
     * @param CartManagementInterface $subject
     * @param int $customerId
     * @return array
     * @throws LocalizedException
     */
    public function beforeCreateEmptyCartForCustomer(
        CartManagementInterface $subject,
        int $customerId
    ): array {
        $customer = $this->customerRepository->getById($customerId);

        // Check custom attribute (example: customer_status)
        $status = $customer->getCustomAttribute('customer_status');
        if ($status && $status->getValue() === 'blocked') {
            throw new LocalizedException(
                __('Your account is currently blocked. Please contact support.')
            );
        }

        return [$customerId];
    }
}
```

#### Track Cart Abandonment

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Api\CartManagementInterface;
use Magento\Quote\Api\Data\CartInterface;

/**
 * Track when carts are created for abandonment tracking
 */
class TrackCartCreationExtend
{
    public function __construct(
        private readonly \Vendor\Module\Model\CartAbandonmentTracker $abandonmentTracker
    ) {}

    /**
     * Track cart creation
     *
     * @param CartManagementInterface $subject
     * @param int $result Quote ID
     * @param int $customerId
     * @return int
     */
    public function afterCreateEmptyCartForCustomer(
        CartManagementInterface $subject,
        int $result,
        int $customerId
    ): int {
        // Register cart for abandonment tracking
        $this->abandonmentTracker->trackCartCreation($result, $customerId);

        return $result;
    }
}
```

### 3. Cart Item Repository Plugins

#### Enforce Product Limits

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Api\CartItemRepositoryInterface;
use Magento\Quote\Api\Data\CartItemInterface;
use Magento\Framework\Exception\LocalizedException;

/**
 * Enforce product-specific purchase limits
 */
class CartItemRepositoryExtend
{
    public function __construct(
        private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
        private readonly \Magento\Quote\Api\CartRepositoryInterface $cartRepository
    ) {}

    /**
     * Validate product limits before adding to cart
     *
     * @param CartItemRepositoryInterface $subject
     * @param CartItemInterface $cartItem
     * @return array
     * @throws LocalizedException
     */
    public function beforeSave(
        CartItemRepositoryInterface $subject,
        CartItemInterface $cartItem
    ): array {
        $product = $this->productRepository->getById($cartItem->getProductId());

        // Check custom attribute: max_qty_per_customer
        $maxQty = $product->getCustomAttribute('max_qty_per_customer');
        if ($maxQty && $cartItem->getQty() > $maxQty->getValue()) {
            throw new LocalizedException(
                __(
                    'Maximum quantity for "%1" is %2.',
                    $product->getName(),
                    $maxQty->getValue()
                )
            );
        }

        // Check total quantity in cart (existing + new)
        $quote = $this->cartRepository->get($cartItem->getQuoteId());
        $existingQty = 0;

        foreach ($quote->getAllItems() as $item) {
            if ($item->getProductId() === $cartItem->getProductId() && $item->getId() !== $cartItem->getItemId()) {
                $existingQty += $item->getQty();
            }
        }

        $totalQty = $existingQty + $cartItem->getQty();
        if ($maxQty && $totalQty > $maxQty->getValue()) {
            throw new LocalizedException(
                __(
                    'Total quantity for "%1" cannot exceed %2. You already have %3 in cart.',
                    $product->getName(),
                    $maxQty->getValue(),
                    $existingQty
                )
            );
        }

        return [$cartItem];
    }
}
```

### 4. Totals Collector Plugins

#### Add Custom Data to Totals

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Model\Quote\TotalsCollector;
use Magento\Quote\Model\Quote;
use Magento\Quote\Model\Quote\Address\Total;

/**
 * Add custom data to totals for frontend display
 */
class TotalsCollectorExtend
{
    public function __construct(
        private readonly \Vendor\Module\Model\RewardPointsCalculator $rewardsCalculator
    ) {}

    /**
     * Add reward points to totals
     *
     * @param TotalsCollector $subject
     * @param Total $result
     * @param Quote $quote
     * @return Total
     */
    public function afterCollect(
        TotalsCollector $subject,
        Total $result,
        Quote $quote
    ): Total {
        // Calculate reward points earned
        $rewardPoints = $this->rewardsCalculator->calculatePoints($quote->getGrandTotal());

        // Add to totals for frontend display
        $result->setRewardPointsEarned($rewardPoints);

        return $result;
    }
}
```

### 5. Quote Model Plugins

#### Prevent Direct Model Manipulation

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Model\Quote;

/**
 * Enforce repository usage for quote saves
 *
 * ANTI-PATTERN PREVENTION: Direct save() on models bypasses plugins
 * on repository. This plugin warns developers.
 */
class PreventDirectSaveExtend
{
    public function __construct(
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Log warning when quote saved directly
     *
     * @param Quote $subject
     * @return array
     */
    public function beforeSave(Quote $subject): array
    {
        // Get backtrace to identify caller
        $backtrace = debug_backtrace(DEBUG_BACKTRACE_IGNORE_ARGS, 5);

        // Check if save() called from repository
        $calledFromRepository = false;
        foreach ($backtrace as $trace) {
            if (isset($trace['class']) && str_contains($trace['class'], 'Repository')) {
                $calledFromRepository = true;
                break;
            }
        }

        if (!$calledFromRepository) {
            $this->logger->warning(
                'Quote saved directly without repository. Use CartRepositoryInterface::save() instead.',
                ['quote_id' => $subject->getId(), 'backtrace' => $backtrace]
            );
        }

        return [];
    }
}
```

## Observer Patterns

### Quote Events

#### Event: `sales_quote_save_before`

Dispatched before quote is saved to database.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

/**
 * Validate quote before save
 */
class QuoteSaveBeforeObserver implements ObserverInterface
{
    /**
     * Execute validation
     *
     * @param Observer $observer
     * @return void
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        // Custom validation: prevent saving quotes with zero grand total
        if ($quote->getGrandTotal() <= 0 && $quote->getItemsCount() > 0) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Cannot save quote with zero total.')
            );
        }
    }
}
```

Register in `etc/events.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="sales_quote_save_before">
        <observer name="vendor_module_quote_save_before"
                  instance="Vendor\Module\Observer\QuoteSaveBeforeObserver"/>
    </event>
</config>
```

#### Event: `sales_quote_save_after`

Dispatched after quote saved to database.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

/**
 * Sync quote to external CRM system
 */
class QuoteSaveAfterObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Vendor\Module\Service\CrmSyncService $crmSync,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Sync quote to CRM
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        // Only sync active customer quotes
        if (!$quote->getIsActive() || !$quote->getCustomerId()) {
            return;
        }

        try {
            $this->crmSync->syncCart($quote);
        } catch (\Exception $e) {
            // Don't fail quote save if CRM sync fails
            $this->logger->error('CRM sync failed for quote: ' . $e->getMessage(), [
                'quote_id' => $quote->getId()
            ]);
        }
    }
}
```

#### Event: `sales_quote_load_after`

Dispatched after quote loaded from database.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

/**
 * Populate custom data after quote load
 */
class QuoteLoadAfterObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Vendor\Module\Model\GiftMessageRepository $giftMessageRepository
    ) {}

    /**
     * Load gift messages
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        // Load gift messages for quote items
        $giftMessages = $this->giftMessageRepository->getByQuoteId($quote->getId());
        $quote->setData('gift_messages', $giftMessages);
    }
}
```

#### Event: `sales_quote_collect_totals_before`

Dispatched before totals collection.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

/**
 * Reset custom totals before recalculation
 */
class CollectTotalsBeforeObserver implements ObserverInterface
{
    /**
     * Reset custom fee
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        // Reset custom totals that will be recalculated
        foreach ($quote->getAllAddresses() as $address) {
            $address->setHandlingFee(0);
            $address->setBaseHandlingFee(0);
        }
    }
}
```

#### Event: `sales_quote_collect_totals_after`

Dispatched after totals collection.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

/**
 * Validate totals after collection
 */
class CollectTotalsAfterObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Validate and log totals
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        // Validate totals consistency
        $calculatedGrandTotal = $quote->getSubtotal()
            + $quote->getShippingAddress()->getShippingAmount()
            + $quote->getShippingAddress()->getTaxAmount()
            - abs($quote->getShippingAddress()->getDiscountAmount());

        if (abs($calculatedGrandTotal - $quote->getGrandTotal()) > 0.01) {
            $this->logger->warning('Quote totals mismatch', [
                'quote_id' => $quote->getId(),
                'calculated' => $calculatedGrandTotal,
                'stored' => $quote->getGrandTotal()
            ]);
        }
    }
}
```

### Cart Events

#### Event: `checkout_cart_add_product_complete`

Dispatched after product added to cart.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

/**
 * Track add-to-cart events
 */
class CartAddProductObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Vendor\Module\Model\Analytics\EventTracker $eventTracker
    ) {}

    /**
     * Track add-to-cart event
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        $product = $observer->getEvent()->getProduct();
        $request = $observer->getEvent()->getRequest();

        $this->eventTracker->track('add_to_cart', [
            'product_id' => $product->getId(),
            'sku' => $product->getSku(),
            'name' => $product->getName(),
            'price' => $product->getFinalPrice(),
            'qty' => $request->getParam('qty', 1)
        ]);
    }
}
```

#### Event: `checkout_cart_update_items_after`

Dispatched after cart items updated.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

/**
 * Send cart update notification
 */
class CartUpdateItemsObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Vendor\Module\Model\Notification\CartUpdateNotifier $notifier
    ) {}

    /**
     * Notify customer of cart update
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $cart */
        $cart = $observer->getEvent()->getCart();
        $info = $observer->getEvent()->getInfo();

        // Send notification if customer opted in
        if ($cart->getCustomer() && $cart->getCustomer()->getCustomAttribute('cart_notifications')) {
            $this->notifier->sendCartUpdateEmail($cart, $info);
        }
    }
}
```

#### Event: `sales_quote_remove_item`

Dispatched when item removed from cart.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote\Item;

/**
 * Track item removal
 */
class QuoteRemoveItemObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Vendor\Module\Model\Analytics\EventTracker $eventTracker
    ) {}

    /**
     * Track item removal
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Item $quoteItem */
        $quoteItem = $observer->getEvent()->getQuoteItem();

        $this->eventTracker->track('remove_from_cart', [
            'product_id' => $quoteItem->getProductId(),
            'sku' => $quoteItem->getSku(),
            'qty' => $quoteItem->getQty(),
            'quote_id' => $quoteItem->getQuoteId()
        ]);
    }
}
```

### Merge Events

#### Event: `sales_quote_merge_before`

Dispatched before guest cart merged into customer cart.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

/**
 * Validate merge compatibility
 */
class QuoteMergeBeforeObserver implements ObserverInterface
{
    /**
     * Validate quotes can be merged
     *
     * @param Observer $observer
     * @return void
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();
        /** @var Quote $source */
        $source = $observer->getEvent()->getSource();

        // Validate stores match
        if ($quote->getStoreId() !== $source->getStoreId()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Cannot merge carts from different stores.')
            );
        }

        // Validate total items won't exceed limit
        $totalItems = $quote->getItemsCount() + $source->getItemsCount();
        if ($totalItems > 100) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Merged cart would exceed maximum item limit.')
            );
        }
    }
}
```

#### Event: `sales_quote_merge_after`

Dispatched after quotes merged.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

/**
 * Log cart merge for analytics
 */
class QuoteMergeAfterObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Log merge event
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();
        /** @var Quote $source */
        $source = $observer->getEvent()->getSource();

        $this->logger->info('Cart merge completed', [
            'customer_cart_id' => $quote->getId(),
            'guest_cart_id' => $source->getId(),
            'customer_id' => $quote->getCustomerId(),
            'merged_items' => $source->getItemsCount(),
            'total_items' => $quote->getItemsCount()
        ]);
    }
}
```

### Submit Events

#### Event: `sales_model_service_quote_submit_before`

Dispatched before quote submitted (converted to order).

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

/**
 * Final validation before order placement
 */
class QuoteSubmitBeforeObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Vendor\Module\Model\FraudDetection $fraudDetection
    ) {}

    /**
     * Validate quote before order placement
     *
     * @param Observer $observer
     * @return void
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        // Fraud detection check
        $fraudScore = $this->fraudDetection->calculateRisk($quote);
        if ($fraudScore > 0.8) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('This order requires manual review. Please contact support.')
            );
        }

        // Validate quote hasn't been modified since checkout started
        $checkoutStartTime = $quote->getData('checkout_started_at');
        if ($checkoutStartTime) {
            $timeSinceStart = time() - strtotime($checkoutStartTime);
            if ($timeSinceStart > 3600) { // 1 hour
                throw new \Magento\Framework\Exception\LocalizedException(
                    __('Your session has expired. Please review your cart and try again.')
                );
            }
        }
    }
}
```

#### Event: `sales_model_service_quote_submit_success`

Dispatched after order created successfully.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Model\Order;
use Magento\Quote\Model\Quote;

/**
 * Post-order creation tasks
 */
class QuoteSubmitSuccessObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Vendor\Module\Model\LoyaltyPoints $loyaltyPoints,
        private readonly \Vendor\Module\Model\Notification\OrderNotifier $notifier
    ) {}

    /**
     * Execute post-order tasks
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Order $order */
        $order = $observer->getEvent()->getOrder();
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        // Award loyalty points
        if ($order->getCustomerId()) {
            $this->loyaltyPoints->awardPoints(
                $order->getCustomerId(),
                $order->getGrandTotal()
            );
        }

        // Send internal notifications
        $this->notifier->notifyWarehouse($order);
        $this->notifier->notifyAccounting($order);
    }
}
```

#### Event: `sales_model_service_quote_submit_failure`

Dispatched if order creation fails.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

/**
 * Handle order placement failure
 */
class QuoteSubmitFailureObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Psr\Log\LoggerInterface $logger,
        private readonly \Vendor\Module\Model\Notification\ErrorNotifier $notifier
    ) {}

    /**
     * Log and notify on failure
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();
        /** @var \Exception $exception */
        $exception = $observer->getEvent()->getException();

        $this->logger->error('Order placement failed', [
            'quote_id' => $quote->getId(),
            'customer_id' => $quote->getCustomerId(),
            'error' => $exception->getMessage(),
            'trace' => $exception->getTraceAsString()
        ]);

        // Notify support team of critical failures
        if ($quote->getGrandTotal() > 1000) {
            $this->notifier->notifyHighValueFailure($quote, $exception);
        }
    }
}
```

## Advanced Patterns

### Conditional Plugin Execution

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\Data\CartInterface;

/**
 * Conditionally execute plugin based on configuration
 */
class ConditionalPluginExtend
{
    public function __construct(
        private readonly \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig
    ) {}

    /**
     * Only execute if feature enabled
     *
     * @param CartRepositoryInterface $subject
     * @param CartInterface $quote
     * @return array
     */
    public function beforeSave(
        CartRepositoryInterface $subject,
        CartInterface $quote
    ): array {
        // Check if feature enabled
        $enabled = $this->scopeConfig->isSetFlag(
            'vendor_module/general/enable_validation',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        );

        if (!$enabled) {
            return [$quote];
        }

        // Feature enabled - execute validation
        // ... validation logic ...

        return [$quote];
    }
}
```

### Plugin Chain Awareness

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\Data\CartInterface;

/**
 * Plugin that cooperates with other plugins
 */
class ChainAwarePluginExtend
{
    /**
     * Around plugin that preserves other plugin behavior
     *
     * @param CartRepositoryInterface $subject
     * @param callable $proceed
     * @param CartInterface $quote
     * @return void
     */
    public function aroundSave(
        CartRepositoryInterface $subject,
        callable $proceed,
        CartInterface $quote
    ): void {
        // Pre-processing
        $originalData = $quote->getData();

        try {
            // Call next plugin in chain or original method
            $proceed($quote);

            // Post-processing on success
            // ... custom logic ...

        } catch (\Exception $e) {
            // Restore original data on failure
            $quote->setData($originalData);
            throw $e;
        }
    }
}
```

### Observer Priority via Sort Order

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="sales_quote_save_after">
        <!-- Execute first (lower sort order) -->
        <observer name="vendor_module_critical"
                  instance="Vendor\Module\Observer\CriticalObserver"
                  sortOrder="10"/>

        <!-- Execute second -->
        <observer name="vendor_module_logging"
                  instance="Vendor\Module\Observer\LoggingObserver"
                  sortOrder="20"/>

        <!-- Execute last (higher sort order) -->
        <observer name="vendor_module_cleanup"
                  instance="Vendor\Module\Observer\CleanupObserver"
                  sortOrder="100"/>
    </event>
</config>
```

## Testing Plugins and Observers

### Unit Test for Plugin

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Unit\Plugin\Quote;

use PHPUnit\Framework\TestCase;
use Vendor\Module\Plugin\Quote\QuoteRepositoryExtend;
use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Model\Quote;

class QuoteRepositoryExtendTest extends TestCase
{
    private QuoteRepositoryExtend $plugin;
    private CartRepositoryInterface $subject;

    protected function setUp(): void
    {
        $logger = $this->createMock(\Psr\Log\LoggerInterface::class);
        $scopeConfig = $this->createMock(\Magento\Framework\App\Config\ScopeConfigInterface::class);

        $this->plugin = new QuoteRepositoryExtend($logger, $scopeConfig);
        $this->subject = $this->createMock(CartRepositoryInterface::class);
    }

    public function testBeforeSaveValidatesItemCount(): void
    {
        $quote = $this->createMock(Quote::class);
        $quote->expects($this->once())
            ->method('getItemsCount')
            ->willReturn(150); // Exceeds limit

        $this->expectException(\Magento\Framework\Exception\LocalizedException::class);
        $this->expectExceptionMessage('Cart cannot contain more than 100 items.');

        $this->plugin->beforeSave($this->subject, $quote);
    }

    public function testBeforeSaveAllowsValidQuote(): void
    {
        $quote = $this->createMock(Quote::class);
        $quote->method('getItemsCount')->willReturn(50);
        $quote->method('getAllItems')->willReturn([]);
        $quote->method('getGrandTotal')->willReturn(100.00);

        $result = $this->plugin->beforeSave($this->subject, $quote);

        $this->assertEquals([$quote], $result);
    }
}
```

### Integration Test for Observer

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Integration\Observer;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;

class QuoteSaveAfterObserverTest extends TestCase
{
    /**
     * @magentoDataFixture Magento/Quote/_files/quote.php
     */
    public function testObserverExecutesOnQuoteSave(): void
    {
        $objectManager = Bootstrap::getObjectManager();
        $quote = $objectManager->create(\Magento\Quote\Model\Quote::class);
        $quote->load('test01', 'reserved_order_id');

        // Trigger save to fire observer
        $quote->setGrandTotal(100.00);
        $quote->save();

        // Verify observer executed (example: check log, external service, etc.)
        // ... assertions ...

        $this->assertTrue(true);
    }
}
```

---

**Assumptions:**
- Magento 2.4.7+ with PHP 8.2+
- Plugins registered via di.xml
- Observers registered via events.xml
- Service contracts used as plugin targets (not models)

**Why This Approach:**
Plugins on service contracts maintain upgrade path. Observers enable decoupled reactions to events. BeforeSave plugins enforce business rules; afterSave observers handle side effects. Around plugins control execution flow.

**Security Impact:**
- Plugins can enforce authorization before cart operations
- Observers can log audit trails for compliance
- Validation plugins prevent malicious data injection
- Never trust client data - validate in plugins/observers

**Performance Impact:**
- Each plugin adds method call overhead (~0.1ms)
- Observers execute synchronously (block request if slow)
- Use queues for heavy observer tasks (email, CRM sync)
- Avoid loading large datasets in observers

**Backward Compatibility:**
Plugin and observer signatures must remain stable. Event names are API contracts. Adding plugin methods is safe; changing signatures breaks extensions.

**Tests to Add:**
- Unit tests for plugin logic
- Integration tests for observer execution
- Functional tests for event dispatch
- Performance tests for plugin chains

**Docs to Update:**
- Plugin catalog with use cases
- Event reference with parameters
- Observer best practices
- Plugin vs observer decision guide
