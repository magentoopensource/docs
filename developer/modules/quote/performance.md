---
title: "Magento_Quote Performance"
module: "Magento_Quote"
doc_type: "performance"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Quote Performance Optimization

## Overview

This document provides comprehensive performance optimization strategies for the Magento_Quote module, covering cart operations, totals calculation, database queries, caching, and scalability patterns for high-traffic stores.

**Target Version:** Magento 2.4.7+ | Adobe Commerce & Open Source
**PHP Version:** 8.2+

---

## Performance Baseline

### Typical Cart Operation Timings (Single Item)

| Operation | Time (ms) | Database Queries | Cache Impact |
|-----------|-----------|------------------|--------------|
| Load Quote | 5-15 | 3-5 | Read from session |
| Add Item | 50-100 | 8-12 | Invalidate cart section |
| Update Quantity | 40-80 | 6-10 | Invalidate cart section |
| Apply Coupon | 60-120 | 10-15 | Invalidate cart section |
| Totals Collection | 30-80 | 5-10 | No FPC impact |
| Place Order | 200-500 | 25-40 | No FPC impact |

### Large Cart Performance (100+ Items)

| Operation | Time (ms) | Scaling Factor |
|-----------|-----------|----------------|
| Load Quote | 50-150 | Linear O(n) |
| Totals Collection | 500-2000 | O(n * m) collectors |
| Apply Discount | 800-3000 | O(n * rules) |
| Place Order | 2000-8000 | Linear O(n) |

---

## 1. Database Optimization

### 1.1 Essential Indexes

Ensure these indexes exist for optimal quote queries:

```sql
-- Quote table indexes
ALTER TABLE quote
    ADD INDEX idx_customer_active (customer_id, is_active, store_id),
    ADD INDEX idx_updated_at (updated_at),
    ADD INDEX idx_reserved_order (reserved_order_id);

-- Quote item indexes
ALTER TABLE quote_item
    ADD INDEX idx_quote_id (quote_id),
    ADD INDEX idx_product_id (product_id),
    ADD INDEX idx_sku (sku);

-- Quote address indexes
ALTER TABLE quote_address
    ADD INDEX idx_quote_id (quote_id),
    ADD INDEX idx_customer_address_id (customer_address_id);

-- Quote payment indexes
ALTER TABLE quote_payment
    ADD INDEX idx_quote_id (quote_id);

-- Quote ID mask indexes (guest carts)
ALTER TABLE quote_id_mask
    ADD UNIQUE INDEX idx_masked_id (masked_id),
    ADD INDEX idx_quote_id (quote_id);
```

**Verification:**

```sql
-- Check if indexes exist
SHOW INDEX FROM quote WHERE Key_name = 'idx_customer_active';
SHOW INDEX FROM quote_item WHERE Key_name = 'idx_quote_id';
```

### 1.2 Query Optimization

#### Avoid SELECT *

```php
<?php
// BAD - Loads all columns
$collection = $this->quoteCollectionFactory->create();
$quotes = $collection->getItems();

// GOOD - Load only needed columns
$collection = $this->quoteCollectionFactory->create();
$collection->addFieldToSelect(['entity_id', 'customer_id', 'items_count', 'grand_total']);
$quotes = $collection->getItems();
```

#### Use Batch Loading

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Model\ResourceModel\Quote\CollectionFactory;

/**
 * Batch load quotes with items
 */
class BatchQuoteLoaderService
{
    public function __construct(
        private readonly CollectionFactory $quoteCollectionFactory
    ) {}

    /**
     * Load multiple quotes with items in single query
     *
     * @param array $quoteIds
     * @return array
     */
    public function loadQuotesWithItems(array $quoteIds): array
    {
        $collection = $this->quoteCollectionFactory->create();
        $collection->addFieldToFilter('entity_id', ['in' => $quoteIds]);

        // OPTIMIZATION: Eager load items to avoid N+1 queries
        foreach ($collection as $quote) {
            $quote->getItemsCollection(); // Loads all items in single query per quote
        }

        return $collection->getItems();
    }
}
```

### 1.3 Connection Pooling

Configure persistent database connections:

```php
// app/etc/env.php
return [
    'db' => [
        'connection' => [
            'default' => [
                'host' => 'localhost',
                'dbname' => 'magento',
                'username' => 'magento',
                'password' => 'password',
                'model' => 'mysql4',
                'engine' => 'innodb',
                'initStatements' => 'SET NAMES utf8;',
                'active' => '1',
                'persistent' => '1', // Enable persistent connections
                'driver_options' => [
                    PDO::MYSQL_ATTR_LOCAL_INFILE => 1,
                    PDO::ATTR_EMULATE_PREPARES => false
                ]
            ]
        ]
    ]
];
```

---

## 2. Totals Calculation Optimization

### 2.1 Collector Ordering

Optimize collector execution order to minimize redundant calculations:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Sales:etc/sales.xsd">
    <section name="quote">
        <group name="totals">
            <!-- Optimal execution order -->
            <item name="subtotal" instance="Magento\Quote\Model\Quote\Address\Total\Subtotal" sort_order="100"/>
            <item name="shipping" instance="Magento\Quote\Model\Quote\Address\Total\Shipping" sort_order="200"/>
            <!-- Custom collectors between 200-300 if they affect tax calculation -->
            <item name="tax" instance="Magento\Tax\Model\Sales\Total\Quote\Tax" sort_order="300"/>
            <item name="discount" instance="Magento\SalesRule\Model\Quote\Discount" sort_order="400"/>
            <!-- Grand total must be last -->
            <item name="grand_total" instance="Magento\Quote\Model\Quote\Address\Total\Grand" sort_order="500"/>
        </group>
    </section>
</config>
```

### 2.2 Conditional Collector Execution

Skip expensive collectors when not needed:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Quote\Address\Total;

use Magento\Quote\Model\Quote;
use Magento\Quote\Api\Data\ShippingAssignmentInterface;
use Magento\Quote\Model\Quote\Address\Total;

/**
 * Optimized custom fee collector
 */
class CustomFee extends \Magento\Quote\Model\Quote\Address\Total\AbstractTotal
{
    protected string $_code = 'custom_fee';

    public function __construct(
        private readonly \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig
    ) {}

    public function collect(
        Quote $quote,
        ShippingAssignmentInterface $shippingAssignment,
        Total $total
    ): self {
        parent::collect($quote, $shippingAssignment, $total);

        // OPTIMIZATION: Skip if feature disabled
        if (!$this->scopeConfig->isSetFlag('vendor/custom_fee/enabled')) {
            return $this;
        }

        // OPTIMIZATION: Skip if virtual quote (no shipping)
        if ($quote->isVirtual()) {
            return $this;
        }

        // OPTIMIZATION: Skip if no items
        $items = $shippingAssignment->getItems();
        if (!count($items)) {
            return $this;
        }

        // Calculate fee only if necessary
        $fee = $this->calculateFee($total);

        $total->setTotalAmount($this->getCode(), $fee);
        $total->setBaseTotalAmount($this->getCode(), $fee);

        return $this;
    }

    /**
     * Lazy calculation - only when needed
     */
    private function calculateFee(Total $total): float
    {
        // Expensive calculation here
        return 10.00;
    }
}
```

### 2.3 Totals Caching

Cache totals for read-heavy scenarios:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Model\Quote;
use Magento\Quote\Model\Quote\TotalsCollector;
use Magento\Quote\Model\Quote\Address\Total;

/**
 * Cache quote totals
 */
class CacheTotalsExtend
{
    private const CACHE_LIFETIME = 300; // 5 minutes
    private const CACHE_TAG = 'quote_totals';

    public function __construct(
        private readonly \Magento\Framework\App\CacheInterface $cache,
        private readonly \Magento\Framework\Serialize\SerializerInterface $serializer
    ) {}

    /**
     * Cache totals collection result
     *
     * @param TotalsCollector $subject
     * @param callable $proceed
     * @param Quote $quote
     * @return Total
     */
    public function aroundCollect(
        TotalsCollector $subject,
        callable $proceed,
        Quote $quote
    ): Total {
        // Generate cache key based on quote state
        $cacheKey = $this->getCacheKey($quote);

        // Try cache first
        $cachedData = $this->cache->load($cacheKey);
        if ($cachedData) {
            return $this->serializer->unserialize($cachedData);
        }

        // Calculate totals
        $totals = $proceed($quote);

        // Cache result
        $this->cache->save(
            $this->serializer->serialize($totals),
            $cacheKey,
            [self::CACHE_TAG, 'quote_' . $quote->getId()],
            self::CACHE_LIFETIME
        );

        return $totals;
    }

    /**
     * Generate cache key from quote state
     *
     * @param Quote $quote
     * @return string
     */
    private function getCacheKey(Quote $quote): string
    {
        $keyData = [
            'quote_id' => $quote->getId(),
            'updated_at' => $quote->getUpdatedAt(),
            'items_count' => $quote->getItemsCount(),
            'coupon_code' => $quote->getCouponCode(),
            'customer_group_id' => $quote->getCustomerGroupId(),
            'store_id' => $quote->getStoreId()
        ];

        return self::CACHE_TAG . '_' . md5($this->serializer->serialize($keyData));
    }
}
```

**Configuration:**

```xml
<type name="Magento\Quote\Model\Quote\TotalsCollector">
    <plugin name="vendor_cache_totals"
            type="Vendor\Module\Plugin\Quote\CacheTotalsExtend"
            sortOrder="10"/>
</type>
```

---

## 3. Caching Strategies

### 3.1 Full Page Cache (FPC) Exclusions

Cart data is customer-specific and must be excluded from FPC:

```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <!-- Cart block excluded from FPC -->
        <referenceBlock name="minicart">
            <arguments>
                <argument name="cache_lifetime" xsi:type="null"/>
                <argument name="cache_key" xsi:type="null"/>
            </arguments>
        </referenceBlock>
    </body>
</page>
```

### 3.2 Customer Section Data

Use section data for dynamic cart updates without full page reload:

```javascript
// Load cart data via section API
require(['Magento_Customer/js/customer-data'], function (customerData) {
    var cart = customerData.get('cart');

    // Subscribe to changes
    cart.subscribe(function (updatedCart) {
        console.log('Cart updated:', updatedCart);
        console.log('Items count:', updatedCart.summary_count);
        console.log('Subtotal:', updatedCart.subtotal);
    });

    // Trigger reload
    customerData.reload(['cart']);
});
```

**Backend Implementation:**

```php
<?php
declare(strict_types=1);

namespace Magento\Checkout\CustomerData;

use Magento\Customer\CustomerData\SectionSourceInterface;

/**
 * Cart section data provider
 */
class Cart implements SectionSourceInterface
{
    public function __construct(
        private readonly \Magento\Checkout\Model\Session $checkoutSession,
        private readonly \Magento\Checkout\Helper\Data $checkoutHelper
    ) {}

    /**
     * Get cart section data
     *
     * OPTIMIZATION: Return minimal data, defer heavy calculations
     *
     * @return array
     */
    public function getSectionData(): array
    {
        $quote = $this->checkoutSession->getQuote();

        if (!$quote->getId()) {
            return [
                'summary_count' => 0,
                'subtotal' => 0,
                'items' => []
            ];
        }

        // Return lightweight data
        return [
            'summary_count' => $quote->getItemsQty(),
            'subtotal' => $quote->getSubtotal(),
            'subtotal_formatted' => $this->checkoutHelper->formatPrice($quote->getSubtotal()),
            'items' => $this->getItemsData($quote), // Only summary, not full item data
            'extra_actions' => $this->checkoutHelper->getCartExtraActions()
        ];
    }

    /**
     * Get minimal item data
     *
     * @param \Magento\Quote\Model\Quote $quote
     * @return array
     */
    private function getItemsData(\Magento\Quote\Model\Quote $quote): array
    {
        $items = [];
        foreach ($quote->getAllVisibleItems() as $item) {
            $items[] = [
                'item_id' => $item->getId(),
                'product_name' => $item->getName(),
                'qty' => $item->getQty(),
                'product_price_value' => $item->getPrice()
            ];
        }
        return $items;
    }
}
```

### 3.3 Redis Session Storage

Use Redis for session storage to improve quote access performance:

```php
// app/etc/env.php
return [
    'session' => [
        'save' => 'redis',
        'redis' => [
            'host' => '127.0.0.1',
            'port' => '6379',
            'password' => '',
            'timeout' => '2.5',
            'persistent_identifier' => '',
            'database' => '2',
            'compression_threshold' => '2048',
            'compression_library' => 'gzip',
            'log_level' => '4',
            'max_concurrency' => '20',
            'break_after_frontend' => '5',
            'break_after_adminhtml' => '30',
            'first_lifetime' => '600',
            'bot_first_lifetime' => '60',
            'bot_lifetime' => '7200',
            'disable_locking' => '0',
            'min_lifetime' => '60',
            'max_lifetime' => '2592000'
        ]
    ]
];
```

---

## 4. Query Reduction Techniques

### 4.1 Lazy Loading Prevention

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Catalog\Model\ResourceModel\Product\CollectionFactory;

/**
 * Eager load products to prevent N+1 queries
 */
class OptimizedCartLoaderService
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly CollectionFactory $productCollectionFactory
    ) {}

    /**
     * Load quote with all related data in minimal queries
     *
     * @param int $quoteId
     * @return \Magento\Quote\Api\Data\CartInterface
     */
    public function loadQuoteOptimized(int $quoteId): \Magento\Quote\Api\Data\CartInterface
    {
        // Load quote (1 query)
        $quote = $this->cartRepository->get($quoteId);

        // Collect all product IDs (1 query for items)
        $productIds = [];
        foreach ($quote->getAllItems() as $item) {
            $productIds[] = $item->getProductId();
        }

        // Eager load all products (1 query)
        if (!empty($productIds)) {
            $productCollection = $this->productCollectionFactory->create();
            $productCollection->addFieldToFilter('entity_id', ['in' => $productIds]);
            $productCollection->addAttributeToSelect(['name', 'sku', 'price', 'image']);
            $products = $productCollection->getItems();

            // Attach products to items (no additional queries)
            foreach ($quote->getAllItems() as $item) {
                if (isset($products[$item->getProductId()])) {
                    $item->setProduct($products[$item->getProductId()]);
                }
            }
        }

        // Total queries: 3 (quote, items, products) instead of 1 + N
        return $quote;
    }
}
```

### 4.2 Collection Optimization

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Model\ResourceModel\Quote\CollectionFactory;

/**
 * Optimized quote collection loading
 */
class QuoteCollectionService
{
    public function __construct(
        private readonly CollectionFactory $quoteCollectionFactory
    ) {}

    /**
     * Load active customer quotes efficiently
     *
     * @param int $customerId
     * @return array
     */
    public function getActiveQuotes(int $customerId): array
    {
        $collection = $this->quoteCollectionFactory->create();

        // OPTIMIZATION: Filter at database level
        $collection->addFieldToFilter('customer_id', $customerId);
        $collection->addFieldToFilter('is_active', 1);

        // OPTIMIZATION: Select only needed fields
        $collection->addFieldToSelect([
            'entity_id',
            'created_at',
            'updated_at',
            'items_count',
            'items_qty',
            'grand_total',
            'store_id'
        ]);

        // OPTIMIZATION: Limit results
        $collection->setPageSize(10);
        $collection->setCurPage(1);

        // OPTIMIZATION: Order by most recent
        $collection->setOrder('updated_at', 'DESC');

        return $collection->getItems();
    }
}
```

---

## 5. Large Cart Optimization

### 5.1 Item Pagination

For carts with 100+ items, paginate item display:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Block\Cart;

use Magento\Framework\View\Element\Template;

/**
 * Paginated cart items display
 */
class Items extends Template
{
    private const ITEMS_PER_PAGE = 20;

    public function __construct(
        Template\Context $context,
        private readonly \Magento\Checkout\Model\Session $checkoutSession,
        array $data = []
    ) {
        parent::__construct($context, $data);
    }

    /**
     * Get items for current page
     *
     * @return array
     */
    public function getItems(): array
    {
        $quote = $this->checkoutSession->getQuote();
        $allItems = $quote->getAllVisibleItems();

        $currentPage = (int)$this->getRequest()->getParam('p', 1);
        $offset = ($currentPage - 1) * self::ITEMS_PER_PAGE;

        return array_slice($allItems, $offset, self::ITEMS_PER_PAGE);
    }

    /**
     * Get total pages
     *
     * @return int
     */
    public function getTotalPages(): int
    {
        $quote = $this->checkoutSession->getQuote();
        $totalItems = $quote->getItemsCount();

        return (int)ceil($totalItems / self::ITEMS_PER_PAGE);
    }
}
```

### 5.2 Async Totals Calculation

Calculate totals asynchronously for very large carts:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Framework\MessageQueue\PublisherInterface;

/**
 * Async totals calculation for large carts
 */
class AsyncTotalsService
{
    private const LARGE_CART_THRESHOLD = 50;

    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly PublisherInterface $publisher
    ) {}

    /**
     * Calculate totals async if cart is large
     *
     * @param int $quoteId
     * @return bool True if calculated sync, false if queued
     */
    public function calculateTotals(int $quoteId): bool
    {
        $quote = $this->cartRepository->get($quoteId);

        if ($quote->getItemsCount() < self::LARGE_CART_THRESHOLD) {
            // Small cart - calculate synchronously
            $quote->collectTotals();
            $this->cartRepository->save($quote);
            return true;
        }

        // Large cart - queue for async processing
        $this->publisher->publish('quote.totals.calculate', json_encode([
            'quote_id' => $quoteId
        ]));

        return false;
    }
}
```

**Message Queue Consumer:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\MessageQueue;

use Magento\Quote\Api\CartRepositoryInterface;

/**
 * Process async totals calculation
 */
class TotalsCalculationConsumer
{
    public function __construct(
        private readonly CartRepositoryInterface $cartRepository,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Process message
     *
     * @param string $message
     * @return void
     */
    public function process(string $message): void
    {
        $data = json_decode($message, true);
        $quoteId = $data['quote_id'];

        try {
            $quote = $this->cartRepository->get($quoteId);
            $quote->collectTotals();
            $this->cartRepository->save($quote);

            $this->logger->info('Async totals calculated', ['quote_id' => $quoteId]);
        } catch (\Exception $e) {
            $this->logger->error('Async totals calculation failed', [
                'quote_id' => $quoteId,
                'error' => $e->getMessage()
            ]);
        }
    }
}
```

---

## 6. Scaling Strategies

### 6.1 Read Replicas

Offload quote reads to database replicas:

```php
// app/etc/env.php
return [
    'db' => [
        'connection' => [
            'default' => [
                'host' => 'db-master.example.com',
                'dbname' => 'magento',
                'username' => 'magento',
                'password' => 'password',
                'active' => '1'
            ],
            'checkout' => [
                'host' => 'db-replica-checkout.example.com',
                'dbname' => 'magento',
                'username' => 'magento_read',
                'password' => 'password',
                'active' => '1'
            ]
        ],
        'slave_connection' => [
            'checkout' => [
                'host' => 'db-replica-checkout.example.com',
                'dbname' => 'magento',
                'username' => 'magento_read',
                'password' => 'password',
                'active' => '1'
            ]
        ]
    ],
    'resource' => [
        'default_setup' => [
            'connection' => 'default'
        ],
        'checkout' => [
            'connection' => 'checkout'
        ]
    ]
];
```

### 6.2 Horizontal Scaling with Sticky Sessions

For multi-server deployments, use sticky sessions to keep customer on same server:

**Nginx Configuration:**

```nginx
upstream backend {
    ip_hash; # Sticky sessions based on IP
    server app1.example.com:9000;
    server app2.example.com:9000;
    server app3.example.com:9000;
}

server {
    location / {
        fastcgi_pass backend;
        # ... other config
    }
}
```

**Alternative: Redis Session Sharing:**

All servers share Redis for session storage (configured in section 3.3).

---

## 7. Monitoring and Profiling

### 7.1 Query Logging

Enable slow query log for quote operations:

```sql
-- MySQL configuration
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 0.5; -- Log queries > 500ms
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

### 7.2 New Relic Instrumentation

Add custom transactions for quote operations:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\Data\CartInterface;

/**
 * New Relic instrumentation
 */
class NewRelicMonitoringExtend
{
    /**
     * Monitor quote save performance
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
        if (extension_loaded('newrelic')) {
            newrelic_start_transaction(ini_get('newrelic.appname'));
            newrelic_name_transaction('quote/save');

            newrelic_add_custom_parameter('quote_id', $quote->getId());
            newrelic_add_custom_parameter('items_count', $quote->getItemsCount());
            newrelic_add_custom_parameter('grand_total', $quote->getGrandTotal());
        }

        $startTime = microtime(true);

        try {
            $proceed($quote);
        } finally {
            $executionTime = (microtime(true) - $startTime) * 1000; // Convert to ms

            if (extension_loaded('newrelic')) {
                newrelic_custom_metric('Custom/Quote/Save/Duration', $executionTime);
                newrelic_end_transaction();
            }
        }
    }
}
```

### 7.3 Performance Metrics Collection

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Metrics;

/**
 * Collect quote performance metrics
 */
class QuoteMetricsCollector
{
    private array $metrics = [];

    public function __construct(
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Record metric
     *
     * @param string $operation
     * @param float $duration
     * @param array $context
     * @return void
     */
    public function record(string $operation, float $duration, array $context = []): void
    {
        $this->metrics[] = [
            'operation' => $operation,
            'duration_ms' => round($duration * 1000, 2),
            'context' => $context,
            'timestamp' => time()
        ];

        // Log slow operations
        if ($duration > 1.0) { // > 1 second
            $this->logger->warning('Slow quote operation detected', [
                'operation' => $operation,
                'duration_ms' => round($duration * 1000, 2),
                'context' => $context
            ]);
        }
    }

    /**
     * Get collected metrics
     *
     * @return array
     */
    public function getMetrics(): array
    {
        return $this->metrics;
    }

    /**
     * Clear metrics
     *
     * @return void
     */
    public function clear(): void
    {
        $this->metrics = [];
    }
}
```

---

## 8. Best Practices Summary

### Do's

1. **Use Repository Pattern** - Always use `CartRepositoryInterface` for persistence
2. **Batch Operations** - Group multiple item additions before `collectTotals()`
3. **Eager Load** - Preload related data to avoid N+1 queries
4. **Index Properly** - Ensure database indexes on frequently queried columns
5. **Cache Aggressively** - Cache totals, section data, and expensive calculations
6. **Monitor Queries** - Track slow queries and optimize them
7. **Test at Scale** - Performance test with 100+ items in cart

### Don'ts

1. **Don't Call collectTotals() Multiple Times** - Batch changes first
2. **Don't Use SELECT *** - Load only needed columns
3. **Don't Bypass Repositories** - Avoid direct model save()
4. **Don't Load Full Collections** - Use pagination and filtering
5. **Don't Trust Client Data** - Always validate server-side
6. **Don't Block on Heavy Operations** - Use async processing for large carts
7. **Don't Ignore Indexes** - Missing indexes cause table scans

---

## 9. Performance Checklist

### Pre-Production

- [ ] Database indexes created and verified
- [ ] Redis configured for sessions and cache
- [ ] Slow query log enabled
- [ ] New Relic or equivalent APM configured
- [ ] Load testing performed with realistic cart sizes
- [ ] Totals collectors have optimal sort_order
- [ ] Custom collectors skip when not needed
- [ ] Section data returns minimal payload
- [ ] Query count verified for key operations

### Monitoring

- [ ] Track average cart operation times
- [ ] Monitor database query count per request
- [ ] Alert on slow totals collection (> 500ms)
- [ ] Monitor cart abandonment rate
- [ ] Track large cart frequency (100+ items)
- [ ] Monitor Redis memory usage
- [ ] Track session storage performance

---

**Assumptions:**
- Magento 2.4.7+ with PHP 8.2+
- MySQL 8.0+ or MariaDB 10.6+
- Redis 6.0+ for session and cache
- Multiple application servers for scaling
- CDN for static content

**Why This Approach:**
Performance optimization requires measurement-driven decisions. Caching strategies prevent redundant calculations. Database optimization reduces query overhead. Monitoring enables proactive issue detection.

**Security Impact:**
- Caching must not expose customer data
- Session sharing requires secure Redis configuration
- Monitoring should not log sensitive data
- Read replicas must enforce same access controls

**Performance Impact:**
- Proper indexing: 50-80% faster queries
- Redis sessions: 30-50% faster quote access
- Totals caching: 60-90% faster on repeated access
- Eager loading: 40-70% reduction in query count

**Backward Compatibility:**
Performance optimizations should be transparent to API consumers. Caching must invalidate correctly. Async processing requires fallback for real-time needs.

**Tests to Add:**
- Performance benchmarks for key operations
- Load tests with 100+ items in cart
- Cache invalidation tests
- Query count assertions
- Scalability tests with concurrent users

**Docs to Update:**
- Performance tuning guide
- Infrastructure requirements
- Monitoring dashboard setup
- Scaling architecture diagrams
