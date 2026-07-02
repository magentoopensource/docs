---
title: "Magento_Quote Known Issues"
module: "Magento_Quote"
doc_type: "known-issues"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Quote Known Issues

## Overview

This document catalogs known issues, bugs, edge cases, and unexpected behaviors in the Magento_Quote module across versions. Each issue includes symptoms, root cause analysis, workarounds, and permanent fixes where available.

**Target Version:** Magento 2.4.7+ | Adobe Commerce & Open Source
**PHP Version:** 8.2+

---

## Issue Categories

1. Quote Merging Issues
2. Abandoned Cart Problems
3. Multi-Currency Cart Issues
4. Multi-Store Cart Synchronization
5. Quote Lifecycle Edge Cases
6. Performance Bottlenecks
7. Data Migration Issues

---

## 1. Quote Merging Issues

### Issue 1.1: Duplicate Items After Guest-to-Customer Merge

**Severity:** Medium
**Affected Versions:** 2.4.0 - 2.4.6
**Status:** Partially Fixed in 2.4.7

**Symptoms:**

- Customer logs in with guest cart containing items
- Customer already has cart with same items
- After merge, items appear duplicated instead of quantities combined
- Customer sees 2 line items for same product configuration

**Example:**

```
Guest cart: Product A (SKU-123) qty 2
Customer cart: Product A (SKU-123) qty 1
After merge:
  - Product A (SKU-123) qty 2
  - Product A (SKU-123) qty 1
Expected:
  - Product A (SKU-123) qty 3
```

**Root Cause:**

The `Quote::merge()` method uses `Item::compare()` to identify duplicate items, but comparison logic doesn't properly handle all product types. Specifically:

1. Configurable products with same options may not match due to option serialization differences
2. Bundle products with different child item order fail comparison
3. Custom options with array values have inconsistent comparison

**Code Analysis:**

```php
<?php
// Magento\Quote\Model\Quote\Item::compare()
// Problem: Option comparison uses strict equality on serialized data

public function compare(Item $item): bool
{
    if ($this->getProductId() !== $item->getProductId()) {
        return false;
    }

    foreach ($this->getOptions() as $option) {
        if ($option->getCode() === 'info_buyRequest') {
            $requestData = $option->getValue();
            $itemRequestData = $item->getOptionByCode('info_buyRequest');

            // ISSUE: Serialized data may differ even if logically identical
            if ($itemRequestData && $requestData !== $itemRequestData->getValue()) {
                return false;
            }
        }
    }

    return true;
}
```

**Workaround:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Model\Quote;

/**
 * Fix duplicate items on merge
 */
class FixDuplicateMergeExtend
{
    public function __construct(
        private readonly \Magento\Framework\Serialize\SerializerInterface $serializer
    ) {}

    /**
     * After merge, consolidate duplicate items
     *
     * @param Quote $subject
     * @param Quote $result
     * @param Quote $source
     * @return Quote
     */
    public function afterMerge(Quote $subject, Quote $result, Quote $source): Quote
    {
        $items = $result->getAllItems();
        $consolidated = [];

        foreach ($items as $item) {
            if ($item->getParentItem()) {
                continue;
            }

            $key = $this->getItemKey($item);

            if (isset($consolidated[$key])) {
                // Duplicate found - combine quantities
                $existingItem = $consolidated[$key];
                $existingItem->setQty($existingItem->getQty() + $item->getQty());
                $result->removeItem($item->getId());
            } else {
                $consolidated[$key] = $item;
            }
        }

        return $result;
    }

    /**
     * Generate consistent key for item comparison
     *
     * @param \Magento\Quote\Model\Quote\Item $item
     * @return string
     */
    private function getItemKey(\Magento\Quote\Model\Quote\Item $item): string
    {
        $key = $item->getProductId() . '_' . $item->getProductType();

        // Add options to key
        $option = $item->getOptionByCode('info_buyRequest');
        if ($option) {
            $buyRequest = $this->serializer->unserialize($option->getValue());

            // Sort array to ensure consistent key
            ksort($buyRequest);
            $key .= '_' . md5($this->serializer->serialize($buyRequest));
        }

        return $key;
    }
}
```

**Permanent Fix:**

Magento Core Team should implement normalized option comparison in `Item::compare()`.

```php
<?php
// Proposed fix for Magento\Quote\Model\Quote\Item

public function compare(Item $item): bool
{
    if ($this->getProductId() !== $item->getProductId()) {
        return false;
    }

    // Get buy requests
    $thisBuyRequest = $this->getBuyRequest();
    $itemBuyRequest = $item->getBuyRequest();

    // Normalize for comparison (sort arrays, remove whitespace, etc.)
    $thisNormalized = $this->normalizeBuyRequest($thisBuyRequest);
    $itemNormalized = $this->normalizeBuyRequest($itemBuyRequest);

    return $thisNormalized === $itemNormalized;
}

private function normalizeBuyRequest(array $buyRequest): array
{
    // Remove qty - not relevant for comparison
    unset($buyRequest['qty']);

    // Recursively sort arrays
    array_walk_recursive($buyRequest, function(&$value) {
        if (is_array($value)) {
            ksort($value);
        }
    });

    return $buyRequest;
}
```

---

### Issue 1.2: Quote Merge Fails Across Different Stores

**Severity:** High
**Affected Versions:** All versions
**Status:** Won't Fix (by design)

**Symptoms:**

- Customer shops on Store A (US), adds items
- Customer logs in on Store B (UK)
- Guest cart from Store A doesn't merge into Store B cart
- Items lost or customer has separate carts per store

**Root Cause:**

Quotes are scoped to stores for currency, tax, and pricing reasons. Merging cross-store carts would require:

1. Currency conversion
2. Price recalculation
3. Tax rule re-evaluation
4. Potential product availability differences

**Workaround:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

/**
 * Notify customer of cart in different store
 */
class CrossStoreMergeNotificationObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Magento\Quote\Api\CartRepositoryInterface $cartRepository,
        private readonly \Magento\Store\Model\StoreManagerInterface $storeManager,
        private readonly \Magento\Framework\Message\ManagerInterface $messageManager
    ) {}

    public function execute(Observer $observer): void
    {
        $customer = $observer->getEvent()->getCustomer();
        $currentStoreId = $this->storeManager->getStore()->getId();

        // Check for carts in other stores
        $connection = $this->cartRepository->getConnection();
        $select = $connection->select()
            ->from('quote', ['store_id', 'entity_id', 'items_count'])
            ->where('customer_id = ?', $customer->getId())
            ->where('is_active = ?', 1)
            ->where('store_id != ?', $currentStoreId);

        $otherCarts = $connection->fetchAll($select);

        if (!empty($otherCarts)) {
            $this->messageManager->addNoticeMessage(
                __(
                    'You have items in your cart on another store. Please visit that store to complete your purchase.'
                )
            );
        }
    }
}
```

**Alternative Solution:**

Provide customer with option to transfer items across stores (manual action):

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Framework\Exception\LocalizedException;

/**
 * Transfer cart items across stores
 */
class CrossStoreCartTransferService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly ProductRepositoryInterface $productRepository
    ) {}

    /**
     * Transfer items from source store cart to target store cart
     *
     * @param int $customerId
     * @param int $sourceStoreId
     * @param int $targetStoreId
     * @return int Number of items transferred
     * @throws LocalizedException
     */
    public function transferCart(
        int $customerId,
        int $sourceStoreId,
        int $targetStoreId
    ): int {
        // Load source cart
        $sourceQuote = $this->cartRepository->getForCustomer($customerId, [$sourceStoreId]);

        // Load or create target cart
        try {
            $targetQuote = $this->cartRepository->getForCustomer($customerId, [$targetStoreId]);
        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            $targetQuote = $this->cartManagement->createEmptyCartForCustomer($customerId);
            $targetQuote->setStoreId($targetStoreId);
        }

        $itemsTransferred = 0;

        foreach ($sourceQuote->getAllVisibleItems() as $item) {
            try {
                // Load product in target store context
                $product = $this->productRepository->getById(
                    $item->getProductId(),
                    false,
                    $targetStoreId
                );

                // Check if product available in target store
                if (!$product->isAvailable()) {
                    continue;
                }

                // Add to target cart with same configuration
                $buyRequest = $item->getBuyRequest();
                $result = $targetQuote->addProduct($product, $buyRequest);

                if (!is_string($result)) {
                    $itemsTransferred++;
                }

            } catch (\Exception $e) {
                // Skip items that can't be transferred
                continue;
            }
        }

        if ($itemsTransferred > 0) {
            $targetQuote->collectTotals();
            $this->cartRepository->save($targetQuote);

            // Deactivate source cart
            $sourceQuote->setIsActive(0);
            $this->cartRepository->save($sourceQuote);
        }

        return $itemsTransferred;
    }
}
```

---

## 2. Abandoned Cart Problems

### Issue 2.1: Abandoned Carts Not Cleaning Up

**Severity:** Medium
**Affected Versions:** All versions
**Status:** Configuration Issue

**Symptoms:**

- Database table `quote` grows indefinitely
- Millions of old inactive quotes
- Database performance degradation
- Slow quote queries

**Root Cause:**

Default Magento configuration doesn't aggressively clean abandoned carts. Configuration path:

```
Stores > Configuration > Sales > Checkout > Shopping Cart
Quote Lifetime (days): 30 (default)
```

Cron job `sales_clean_quotes` runs but only marks quotes inactive, doesn't delete.

**Workaround:**

Create custom cron to delete old quotes:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Cron;

use Magento\Quote\Model\ResourceModel\Quote\CollectionFactory;
use Magento\Framework\Stdlib\DateTime\DateTime;

/**
 * Delete old abandoned quotes
 */
class CleanOldQuotes
{
    private const DAYS_TO_KEEP = 90;

    public function __construct(
        private readonly CollectionFactory $quoteCollectionFactory,
        private readonly DateTime $dateTime,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Execute cleanup
     *
     * @return void
     */
    public function execute(): void
    {
        $timestamp = $this->dateTime->gmtTimestamp() - (self::DAYS_TO_KEEP * 86400);
        $datetime = $this->dateTime->gmtDate('Y-m-d H:i:s', $timestamp);

        $collection = $this->quoteCollectionFactory->create();
        $collection->addFieldToFilter('is_active', 0);
        $collection->addFieldToFilter('updated_at', ['lt' => $datetime]);

        $deletedCount = 0;

        foreach ($collection as $quote) {
            try {
                $quote->delete();
                $deletedCount++;
            } catch (\Exception $e) {
                $this->logger->error('Failed to delete quote: ' . $e->getMessage(), [
                    'quote_id' => $quote->getId()
                ]);
            }
        }

        $this->logger->info('Cleaned up old quotes', [
            'deleted_count' => $deletedCount
        ]);
    }
}
```

Register cron in `crontab.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Cron:etc/crontab.xsd">
    <group id="default">
        <job name="vendor_module_clean_old_quotes" instance="Vendor\Module\Cron\CleanOldQuotes" method="execute">
            <schedule>0 2 * * *</schedule> <!-- Daily at 2 AM -->
        </job>
    </group>
</config>
```

---

### Issue 2.2: Abandoned Cart Email Sent Multiple Times

**Severity:** Low
**Affected Versions:** Extensions only
**Status:** Extension Issue

**Symptoms:**

- Customer receives multiple abandoned cart emails for same cart
- Email sent every hour instead of once
- Customer annoyance

**Root Cause:**

Third-party abandoned cart extensions don't properly track sent emails. Need to flag quotes as "email sent" to prevent duplicates.

**Solution:**

Add custom attribute to track email status:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Quote\Setup\QuoteSetupFactory;

class AddAbandonedEmailFlagAttribute implements DataPatchInterface
{
    public function __construct(
        private readonly QuoteSetupFactory $quoteSetupFactory
    ) {}

    public function apply()
    {
        $quoteSetup = $this->quoteSetupFactory->create();

        $quoteSetup->addAttribute('quote', 'abandoned_email_sent', [
            'type' => 'int',
            'label' => 'Abandoned Cart Email Sent',
            'visible' => false,
            'required' => false,
            'default' => 0
        ]);

        $quoteSetup->addAttribute('quote', 'abandoned_email_sent_at', [
            'type' => 'datetime',
            'label' => 'Abandoned Cart Email Sent At',
            'visible' => false,
            'required' => false
        ]);

        return $this;
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

Use in abandoned cart email service:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;

class AbandonedCartEmailService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly \Magento\Framework\Mail\Template\TransportBuilder $transportBuilder
    ) {}

    public function sendAbandonedCartEmail(int $quoteId): void
    {
        $quote = $this->cartRepository->get($quoteId);

        // Check if email already sent
        if ($quote->getData('abandoned_email_sent')) {
            return;
        }

        // Send email...
        $this->transportBuilder
            ->setTemplateIdentifier('abandoned_cart_email')
            ->setTemplateOptions(['area' => 'frontend', 'store' => $quote->getStoreId()])
            ->setTemplateVars(['quote' => $quote])
            ->setFromByScope('sales')
            ->addTo($quote->getCustomerEmail())
            ->getTransport()
            ->sendMessage();

        // Mark email sent
        $quote->setData('abandoned_email_sent', 1);
        $quote->setData('abandoned_email_sent_at', date('Y-m-d H:i:s'));
        $this->cartRepository->save($quote);
    }
}
```

---

## 3. Multi-Currency Cart Issues

### Issue 3.1: Currency Not Updating on Store Switch

**Severity:** Medium
**Affected Versions:** 2.4.0 - 2.4.5
**Status:** Fixed in 2.4.6

**Symptoms:**

- Customer adds items to cart in USD
- Customer switches store view to EUR
- Cart still shows USD prices
- Totals incorrect

**Root Cause:**

Quote currency fields (`quote_currency_code`, `store_to_quote_rate`) not updated on store switch.

**Workaround (for versions < 2.4.6):**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Model\Quote;
use Magento\Store\Model\Store;

/**
 * Update quote currency on store switch
 */
class UpdateCurrencyOnStoreChangeExtend
{
    public function __construct(
        private readonly \Magento\Quote\Api\CartRepositoryInterface $cartRepository,
        private readonly \Magento\Directory\Model\CurrencyFactory $currencyFactory
    ) {}

    /**
     * After store set on quote, update currency
     *
     * @param Quote $subject
     * @param Quote $result
     * @param Store $store
     * @return Quote
     */
    public function afterSetStore(Quote $subject, Quote $result, $store): Quote
    {
        $currentCurrency = $result->getQuoteCurrencyCode();
        $storeCurrency = $store->getCurrentCurrencyCode();

        if ($currentCurrency !== $storeCurrency) {
            // Update currency
            $result->setQuoteCurrencyCode($storeCurrency);

            // Calculate exchange rate
            $baseCurrency = $this->currencyFactory->create()->load($store->getBaseCurrencyCode());
            $quoteCurrency = $this->currencyFactory->create()->load($storeCurrency);
            $rate = $baseCurrency->getRate($quoteCurrency);

            $result->setStoreToQuoteRate($rate);

            // Recalculate prices
            foreach ($result->getAllItems() as $item) {
                $product = $item->getProduct();
                $product->setStoreId($store->getId());

                // Reload price in new currency
                $price = $product->getFinalPrice($item->getQty());
                $item->setPrice($price);
                $item->calcRowTotal();
            }

            // Recalculate totals
            $result->collectTotals();
        }

        return $result;
    }
}
```

---

## 4. Multi-Store Cart Synchronization

### Issue 4.1: Customer Has Multiple Active Quotes

**Severity:** Low
**Affected Versions:** All versions
**Status:** By Design

**Symptoms:**

- Customer has active quote in Store A
- Customer has active quote in Store B
- Only one quote visible depending on current store
- Data inconsistency

**Root Cause:**

Magento allows one active quote per customer per store. This is intentional but can confuse customers.

**Solution:**

Provide unified cart view across stores:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Block\Cart;

use Magento\Framework\View\Element\Template;

/**
 * Display all customer carts across stores
 */
class MultiStoreCartView extends Template
{
    public function __construct(
        Template\Context $context,
        private readonly \Magento\Quote\Model\ResourceModel\Quote\CollectionFactory $quoteCollectionFactory,
        private readonly \Magento\Customer\Model\Session $customerSession,
        private readonly \Magento\Store\Model\StoreManagerInterface $storeManager,
        array $data = []
    ) {
        parent::__construct($context, $data);
    }

    /**
     * Get all active carts for customer
     *
     * @return \Magento\Quote\Model\Quote[]
     */
    public function getCustomerCarts(): array
    {
        if (!$this->customerSession->isLoggedIn()) {
            return [];
        }

        $customerId = $this->customerSession->getCustomerId();

        $collection = $this->quoteCollectionFactory->create();
        $collection->addFieldToFilter('customer_id', $customerId);
        $collection->addFieldToFilter('is_active', 1);
        $collection->addFieldToFilter('items_count', ['gt' => 0]);

        return $collection->getItems();
    }

    /**
     * Get store name for quote
     *
     * @param \Magento\Quote\Model\Quote $quote
     * @return string
     */
    public function getStoreName(\Magento\Quote\Model\Quote $quote): string
    {
        try {
            $store = $this->storeManager->getStore($quote->getStoreId());
            return $store->getName();
        } catch (\Exception $e) {
            return 'Unknown Store';
        }
    }
}
```

---

## 5. Quote Lifecycle Edge Cases

### Issue 5.1: Quote Reserved Order ID Conflicts

**Severity:** High
**Affected Versions:** 2.4.0 - 2.4.4
**Status:** Fixed in 2.4.5

**Symptoms:**

- Order placement fails with "Duplicate entry for reserved_order_id"
- Multiple quotes have same reserved order ID
- Database constraint violation

**Root Cause:**

Race condition when multiple processes try to reserve order IDs simultaneously.

**Temporary Workaround:**

Retry order placement with new reserved ID:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartManagementInterface;
use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Framework\Exception\LocalizedException;

class OrderPlacementRetryService
{
    private const MAX_RETRIES = 3;

    public function __construct(
        private readonly CartManagementInterface $cartManagement,
        private readonly CartRepositoryInterface $cartRepository
    ) {}

    public function placeOrder(int $quoteId): int
    {
        $attempts = 0;

        while ($attempts < self::MAX_RETRIES) {
            try {
                return $this->cartManagement->placeOrder($quoteId);
            } catch (\Exception $e) {
                // Check if duplicate reserved order ID error
                if (str_contains($e->getMessage(), 'reserved_order_id')) {
                    $attempts++;

                    if ($attempts >= self::MAX_RETRIES) {
                        throw $e;
                    }

                    // Generate new reserved order ID
                    $quote = $this->cartRepository->get($quoteId);
                    $quote->setReservedOrderId(null);
                    $quote->reserveOrderId();
                    $this->cartRepository->save($quote);

                    // Retry
                    continue;
                }

                // Different error - rethrow
                throw $e;
            }
        }

        throw new LocalizedException(__('Failed to place order after %1 attempts', self::MAX_RETRIES));
    }
}
```

---

## 6. Performance Bottlenecks

### Issue 6.1: Slow Totals Collection with Many Items

**Severity:** Medium
**Affected Versions:** All versions
**Status:** Optimization Needed

**Symptoms:**

- Cart with 100+ items is very slow
- Totals calculation takes 5-10 seconds
- Page timeout on cart view
- Poor user experience

**Root Cause:**

Totals collectors iterate all items multiple times. With 100+ items:

- Subtotal collector: O(n)
- Tax collector: O(n) with tax calculation per item
- Discount collector: O(n * m) where m = number of sales rules
- Shipping collector: O(n) for weight calculation

**Optimization:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Model\Quote\TotalsCollector;
use Magento\Quote\Model\Quote;

/**
 * Cache totals for large carts
 */
class CacheTotalsForLargeCartsExtend
{
    private const CACHE_LIFETIME = 300; // 5 minutes
    private const LARGE_CART_THRESHOLD = 50;

    public function __construct(
        private readonly \Magento\Framework\App\CacheInterface $cache,
        private readonly \Magento\Framework\Serialize\SerializerInterface $serializer
    ) {}

    /**
     * Cache totals for large carts
     *
     * @param TotalsCollector $subject
     * @param callable $proceed
     * @param Quote $quote
     * @return \Magento\Quote\Model\Quote\Address\Total
     */
    public function aroundCollect(
        TotalsCollector $subject,
        callable $proceed,
        Quote $quote
    ) {
        // Only cache for large carts
        if ($quote->getItemsCount() < self::LARGE_CART_THRESHOLD) {
            return $proceed($quote);
        }

        $cacheKey = 'quote_totals_' . $quote->getId() . '_' . $quote->getUpdatedAt();

        // Try to load from cache
        $cachedTotals = $this->cache->load($cacheKey);
        if ($cachedTotals) {
            return $this->serializer->unserialize($cachedTotals);
        }

        // Calculate totals
        $totals = $proceed($quote);

        // Cache result
        $this->cache->save(
            $this->serializer->serialize($totals),
            $cacheKey,
            ['quote_totals'],
            self::CACHE_LIFETIME
        );

        return $totals;
    }
}
```

---

## 7. Data Migration Issues

### Issue 7.1: Quote Data Lost During Upgrade

**Severity:** High
**Affected Versions:** Upgrade from 2.3.x to 2.4.x
**Status:** Migration Script Needed

**Symptoms:**

- After upgrade, active customer carts are empty
- quote_item table has orphaned rows
- Customer complaints of lost carts

**Root Cause:**

Schema changes or data migration scripts failed during upgrade.

**Verification:**

```sql
-- Check for orphaned quote items
SELECT qi.item_id, qi.quote_id
FROM quote_item qi
LEFT JOIN quote q ON qi.quote_id = q.entity_id
WHERE q.entity_id IS NULL;

-- Check for quotes missing items
SELECT q.entity_id, q.items_count, COUNT(qi.item_id) AS actual_items
FROM quote q
LEFT JOIN quote_item qi ON q.entity_id = qi.quote_id
WHERE q.items_count > 0
GROUP BY q.entity_id
HAVING actual_items = 0;
```

**Recovery Script:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Console\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

/**
 * Repair quote data after migration
 */
class RepairQuoteDataCommand extends Command
{
    public function __construct(
        private readonly \Magento\Quote\Model\ResourceModel\Quote\CollectionFactory $quoteCollectionFactory,
        private readonly \Magento\Framework\App\ResourceConnection $resource
    ) {
        parent::__construct();
    }

    protected function configure()
    {
        $this->setName('quote:repair:data');
        $this->setDescription('Repair quote data after migration');
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $connection = $this->resource->getConnection();

        // Fix items_count
        $output->writeln('Fixing items_count...');
        $connection->query("
            UPDATE quote q
            SET items_count = (
                SELECT COUNT(*)
                FROM quote_item qi
                WHERE qi.quote_id = q.entity_id
                  AND qi.parent_item_id IS NULL
            )
        ");

        // Fix items_qty
        $output->writeln('Fixing items_qty...');
        $connection->query("
            UPDATE quote q
            SET items_qty = (
                SELECT COALESCE(SUM(qi.qty), 0)
                FROM quote_item qi
                WHERE qi.quote_id = q.entity_id
                  AND qi.parent_item_id IS NULL
            )
        ");

        // Deactivate empty quotes
        $output->writeln('Deactivating empty quotes...');
        $connection->query("
            UPDATE quote
            SET is_active = 0
            WHERE items_count = 0
              AND is_active = 1
        ");

        $output->writeln('Quote data repair completed.');
        return Command::SUCCESS;
    }
}
```

---

**Assumptions:**
- Magento 2.4.7+ with PHP 8.2+
- MySQL/MariaDB database
- Standard quote table schema
- Third-party extensions may introduce additional issues

**Why This Approach:**
Real-world issues documented with root cause analysis. Provides immediate workarounds while advocating for permanent fixes. Includes detection scripts and verification queries.

**Security Impact:**
- Quote ownership validation critical for multi-customer installations
- Abandoned cart cleanup prevents data accumulation
- Reserved order ID conflicts can be exploited

**Performance Impact:**
- Abandoned cart cleanup improves database performance
- Totals caching dramatically improves large cart performance
- Multi-currency conversions add overhead

**Backward Compatibility:**
Workarounds designed to be non-invasive. Can be removed when upgrading to fixed versions. Migration scripts preserve data integrity.

**Tests to Add:**
- Integration tests for quote merge scenarios
- Performance tests for large carts (100+ items)
- Data integrity tests for migrations
- Unit tests for currency conversion logic

**Docs to Update:**
- Known issues changelog
- Upgrade guide with verification steps
- Performance tuning guide
- Troubleshooting runbook
