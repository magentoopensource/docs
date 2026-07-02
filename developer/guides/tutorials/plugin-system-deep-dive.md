---
title: "Plugin System Deep Dive: Mastering Magento 2 Interception"
description: "Complete guide to Magento 2's plugin system (interceptors): Before, After, Around plugins, execution order, best practices, performance optimization, and production-ready examples"
type: "tutorial"
tier: 1
difficulty: "intermediate"
version: "2.4.7+"
estimated_time: "60 minutes"
topics:
  - plugins
  - interceptors
  - dependency-injection
  - before-plugin
  - after-plugin
  - around-plugin
  - design-patterns
  - best-practices
  - performance
last_updated: "2026-02-07"
---

# Plugin System Deep Dive: Mastering Magento 2 Interception

## Learning Objectives

By completing this tutorial, you will:

- Understand the plugin (interceptor) pattern and how Magento implements it
- Master Before, After, and Around plugin types with real-world examples
- Control plugin execution order using `sortOrder` and dependencies
- Choose the correct extension mechanism: plugins vs observers vs preferences
- Avoid common plugin pitfalls that break functionality or performance
- Optimize plugin performance and minimize generated code bloat
- Implement production-ready plugins following SOLID principles
- Debug plugin chains and diagnose interception issues

## Introduction

Magento 2's plugin system (also called interceptors) is the primary mechanism for extending core and third-party functionality **without modifying original code**. Plugins enable you to run custom code before, after, or around any public method in Magento, making them essential for building upgrade-safe, modular extensions.

### What Are Plugins?

Plugins are PHP classes that intercept method calls on **public methods** of **non-final classes**. When a method is intercepted, Magento's generated code routes the call through your plugin, allowing you to:

- Modify input arguments (Before plugin)
- Modify return values (After plugin)
- Completely replace method logic (Around plugin)

### Why Use Plugins?

**Advantages:**
- **Upgrade-safe**: No direct modification of core/third-party code
- **Modular**: Multiple plugins can intercept the same method
- **Flexible**: Choose Before/After/Around based on requirements
- **Testable**: Plugins are DI-injected classes with explicit dependencies

**Limitations:**
- Only work on **public methods** of **non-final classes**
- Cannot intercept: final methods, static methods, constructors, private/protected methods
- Performance cost: Generated code overhead (mitigated by compiled mode)

### When to Use Plugins vs Alternatives

| Mechanism | Use Case | Example |
|-----------|----------|---------|
| **Plugin** | Modify method behavior, arguments, or return values | Change product price calculation |
| **Observer** | React to events without return value | Send email after order placed |
| **Preference** | Replace entire class (last resort) | Override core class with major changes |
| **Event Dispatch** | Notify other modules of state changes | Dispatch custom event after import |
| **Service Contract** | Define new API contract | Create new repository interface |

**Rule of thumb**: Prefer plugins for method interception, observers for event-driven logic, and avoid preferences.

## Plugin Types: Before, After, Around

Magento provides three plugin types, each with distinct capabilities and use cases.

### Before Plugin

**Purpose**: Modify method arguments before the original method executes.

**Signature**: `public function before<MethodName>(<Type> $subject, <argument1>, <argument2>, ...)`

**Return**: Array of modified arguments (or empty array to keep original)

**Use Case**: Validation, argument transformation, logging input

**Example**: Add customer group discount to product price calculation

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Catalog\Model\Product;

use Magento\Catalog\Model\Product;
use Magento\Customer\Model\Session as CustomerSession;
use Psr\Log\LoggerInterface;

class PriceExtend
{
    public function __construct(
        private readonly CustomerSession $customerSession,
        private readonly LoggerInterface $logger
    ) {
    }

    /**
     * Modify finalPrice calculation arguments based on customer group
     *
     * @param Product $subject The product instance being intercepted
     * @param float $qty Quantity (original argument)
     * @return array|null Modified arguments [qty] or null to keep original
     */
    public function beforeGetFinalPrice(Product $subject, $qty = 1.0): ?array
    {
        $customerGroupId = $this->customerSession->getCustomerGroupId();

        // Log for debugging
        $this->logger->debug('Before getFinalPrice', [
            'product_id' => $subject->getId(),
            'original_qty' => $qty,
            'customer_group' => $customerGroupId
        ]);

        // Example: Apply bulk discount for wholesale customer group (ID 3)
        if ($customerGroupId === 3 && $qty < 10) {
            $this->logger->info('Adjusting quantity to 10 for wholesale pricing');
            return [10.0]; // Force minimum quantity for wholesale price tier
        }

        return null; // Return null to keep original arguments unchanged
    }
}
```

**Key Points:**
- Method name: `before` + PascalCase original method name
- First parameter: `$subject` (the intercepted object instance)
- Subsequent parameters: Match original method signature **exactly**
- Return: Array of modified arguments in same order as original method, or `null` to keep original
- **Important:** Returning `null` keeps original arguments. Returning `[]` would replace arguments with an empty array, which would break the method call. The interceptor checks `if ($beforeResult !== null)` before replacing arguments.

### After Plugin

**Purpose**: Modify the return value after the original method executes.

**Signature**: `public function after<MethodName>(<Type> $subject, <result>, <argument1>, <argument2>, ...)`

**Return**: Modified result (must match original return type)

**Use Case**: Transform output, add data, logging result

**Example**: Add custom attribute to product collection

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Catalog\Model\ResourceModel\Product;

use Magento\Catalog\Model\ResourceModel\Product\Collection;
use Magento\Framework\DB\Select;
use Psr\Log\LoggerInterface;

class CollectionExtend
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {
    }

    /**
     * Add custom attribute to product collection after load
     *
     * @param Collection $subject The collection instance
     * @param Collection $result The collection after _afterLoad (method returns $this)
     * @return Collection Modified collection
     */
    public function afterLoad(Collection $subject, Collection $result): Collection
    {
        // Only process if collection has items
        if ($result->count() === 0) {
            return $result;
        }

        // Add custom data to each product
        foreach ($result->getItems() as $product) {
            $customValue = $this->calculateCustomValue($product);
            $product->setData('custom_attribute', $customValue);
        }

        $this->logger->debug('Added custom attribute to product collection', [
            'product_count' => $result->count()
        ]);

        return $result;
    }

    /**
     * Example business logic
     */
    private function calculateCustomValue($product): string
    {
        // Complex calculation based on product data
        return 'custom_' . $product->getId();
    }
}
```

**Key Points:**
- Method name: `after` + PascalCase original method name
- First parameter: `$subject` (the intercepted object)
- **Second parameter**: `$result` (return value from original method)
- Subsequent parameters: Original method arguments (for reference only)
- **Must return** value compatible with original method's return type
- Cannot access `$subject` state changed by original method (use `$result`)

### Around Plugin

**Purpose**: Completely control method execution (call original, skip it, or replace it).

**Signature**: `public function around<MethodName>(<Type> $subject, callable $proceed, <argument1>, <argument2>, ...)`

**Return**: Modified result (must match original return type)

**Use Case**: Conditional execution, caching, performance optimization, complete behavior replacement

**Example**: Add caching layer to expensive product recommendation

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Catalog\Model\Product;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Catalog\Model\Product;
use Magento\Framework\App\CacheInterface;
use Magento\Framework\Serialize\SerializerInterface;
use Psr\Log\LoggerInterface;

class RecommendationExtend
{
    private const CACHE_KEY_PREFIX = 'product_recommendations_';
    private const CACHE_LIFETIME = 3600; // 1 hour
    private const CACHE_TAG = 'product_recommendations';

    public function __construct(
        private readonly CacheInterface $cache,
        private readonly SerializerInterface $serializer,
        private readonly LoggerInterface $logger
    ) {
    }

    /**
     * Add caching layer around expensive getRecommendations call
     *
     * @param Recommendation $subject
     * @param callable $proceed Original method callable
     * @param ProductInterface $product
     * @param int $limit
     * @return ProductInterface[] Array of recommended products
     */
    public function aroundGetRecommendations(
        Recommendation $subject,
        callable $proceed,
        ProductInterface $product,
        int $limit = 5
    ): array {
        $cacheKey = $this->getCacheKey($product->getId(), $limit);

        // Try cache first
        $cachedData = $this->cache->load($cacheKey);
        if ($cachedData !== false) {
            $this->logger->debug('Cache hit for product recommendations', [
                'product_id' => $product->getId(),
                'limit' => $limit
            ]);

            try {
                return $this->serializer->unserialize($cachedData);
            } catch (\InvalidArgumentException $e) {
                $this->logger->error('Failed to unserialize cached recommendations', [
                    'error' => $e->getMessage()
                ]);
                // Fall through to original method
            }
        }

        // Cache miss - call original method
        $this->logger->info('Cache miss, executing expensive recommendation logic', [
            'product_id' => $product->getId()
        ]);

        $result = $proceed($product, $limit); // Call original method

        // Store in cache
        try {
            $this->cache->save(
                $this->serializer->serialize($result),
                $cacheKey,
                [self::CACHE_TAG],
                self::CACHE_LIFETIME
            );
        } catch (\InvalidArgumentException $e) {
            $this->logger->error('Failed to cache recommendations', [
                'error' => $e->getMessage()
            ]);
        }

        return $result;
    }

    /**
     * Generate unique cache key
     */
    private function getCacheKey(int $productId, int $limit): string
    {
        return self::CACHE_KEY_PREFIX . $productId . '_' . $limit;
    }
}
```

**Key Points:**
- Method name: `around` + PascalCase original method name
- Second parameter: `callable $proceed` (invokes original method + subsequent plugins)
- **Must call** `$proceed(...)` to execute original method (unless intentionally skipping)
- Arguments passed to `$proceed()` can be modified
- **Most powerful** but also **most dangerous** (can break plugin chain if misused)
- Higher performance cost than Before/After

**Common Around Plugin Mistake:**

```php
// WRONG: Not calling $proceed breaks plugin chain
public function aroundSomeMethod($subject, callable $proceed, $arg)
{
    return 'my value'; // Original method and other plugins never execute!
}

// CORRECT: Always call $proceed unless you have specific reason
public function aroundSomeMethod($subject, callable $proceed, $arg)
{
    $result = $proceed($arg); // Execute original + other plugins
    return $this->modifyResult($result);
}
```

## Plugin Registration (di.xml)

Plugins are declared in `etc/di.xml` (global scope) or `etc/frontend/di.xml` / `etc/adminhtml/di.xml` (area-specific).

### Basic Plugin Registration

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Plugin for product price calculation -->
    <type name="Magento\Catalog\Model\Product">
        <plugin name="vendor_module_product_price_plugin"
                type="Vendor\Module\Plugin\Catalog\Model\Product\PriceExtend"
                sortOrder="10"
                disabled="false" />
    </type>

</config>
```

**Attributes:**

| Attribute | Required | Description | Example |
|-----------|----------|-------------|---------|
| `name` | Yes | Unique plugin identifier (vendor_module_scope_plugin) | `vendor_module_product_price_plugin` |
| `type` | Yes | Fully qualified plugin class name | `Vendor\Module\Plugin\Class` |
| `sortOrder` | No | Execution order (default: 10) | `10` (lower = earlier) |
| `disabled` | No | Enable/disable plugin (default: false) | `true` to disable |

### Plugin Naming Convention

Follow this pattern for plugin `name` attribute:

```
{vendor}_{module}_{scope}_{functionality}_plugin
```

Examples:
- `acme_catalog_product_price_plugin`
- `acme_checkout_cart_validation_plugin`
- `acme_customer_authentication_plugin`

### Area-Specific Plugins

Limit plugin scope to frontend or adminhtml:

**Frontend only** (`etc/frontend/di.xml`):

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Only active on storefront -->
    <type name="Magento\Checkout\Model\Cart">
        <plugin name="vendor_module_cart_discount_plugin"
                type="Vendor\Module\Plugin\Checkout\Model\Cart\DiscountExtend"
                sortOrder="20" />
    </type>

</config>
```

**Admin only** (`etc/adminhtml/di.xml`):

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Only active in admin panel -->
    <type name="Magento\Sales\Api\OrderRepositoryInterface">
        <plugin name="vendor_module_order_audit_plugin"
                type="Vendor\Module\Plugin\Sales\Api\OrderRepositoryExtend"
                sortOrder="30" />
    </type>

</config>
```

**Global** (`etc/di.xml`): Active in all areas (frontend, adminhtml, crontab, webapi_rest, webapi_soap).

## Plugin Execution Order

When multiple plugins intercept the same method, execution order is determined by `sortOrder` (ascending).

### Execution Flow

Given three plugins on `ClassName::methodName()`:

```xml
<type name="ClassName">
    <plugin name="pluginA" type="PluginA" sortOrder="10" />
    <plugin name="pluginB" type="PluginB" sortOrder="20" />
    <plugin name="pluginC" type="PluginC" sortOrder="30" />
</type>
```

**Execution sequence** when `methodName()` is called:

1. `PluginA::beforeMethodName()` (sortOrder 10)
2. `PluginB::beforeMethodName()` (sortOrder 20)
3. `PluginC::beforeMethodName()` (sortOrder 30)
4. `PluginA::aroundMethodName()` (sortOrder 10, calls `$proceed`)
5. `PluginB::aroundMethodName()` (sortOrder 20, calls `$proceed`)
6. `PluginC::aroundMethodName()` (sortOrder 30, calls `$proceed`)
7. **Original method executes**
8. `PluginC::aroundMethodName()` returns (sortOrder 30, reverse order)
9. `PluginB::aroundMethodName()` returns (sortOrder 20)
10. `PluginA::aroundMethodName()` returns (sortOrder 10)
11. `PluginA::afterMethodName()` (sortOrder 10)
12. `PluginB::afterMethodName()` (sortOrder 20)
13. `PluginC::afterMethodName()` (sortOrder 30)

**Key Observations:**

- **Before plugins**: Execute in sortOrder (10 → 20 → 30)
- **Around plugins**: Execute in sortOrder on entry, **reverse** on exit (like nested function calls)
- **After plugins**: Execute in sortOrder (10 → 20 → 30), same order as before plugins
- **Original method**: Executes after all Around plugins call `$proceed()`

### Controlling Execution Order

**Default sortOrder**: If not specified, defaults to `10`.

**Strategic sortOrder values**:

| sortOrder | Purpose | Example |
|-----------|---------|---------|
| `1-9` | Early validation, security checks | Authentication plugin |
| `10-50` | Standard business logic | Price calculation |
| `51-90` | Post-processing, logging | Audit trail |
| `91-100` | Final transformations | Output formatting |

**Example**: Ensure validation runs before business logic:

```xml
<type name="Magento\Checkout\Model\Cart">
    <!-- Validation runs first -->
    <plugin name="vendor_module_cart_validation"
            type="Vendor\Module\Plugin\Checkout\Validation"
            sortOrder="5" />

    <!-- Business logic runs second -->
    <plugin name="vendor_module_cart_discount"
            type="Vendor\Module\Plugin\Checkout\Discount"
            sortOrder="20" />

    <!-- Logging runs last -->
    <plugin name="vendor_module_cart_logger"
            type="Vendor\Module\Plugin\Checkout\Logger"
            sortOrder="90" />
</type>
```

### Debugging Plugin Order

View compiled plugin chain:

```bash
# Inspect generated interceptor
cat generated/code/Magento/Catalog/Model/Product/Interceptor.php
```

The `Interceptor.php` file shows the exact plugin sequence Magento will execute.

## When to Use Plugins vs Observers vs Preferences

### Decision Matrix

| Requirement | Solution | Reason |
|-------------|----------|--------|
| Modify method arguments | **Before Plugin** | Direct access to arguments |
| Modify return value | **After Plugin** | Direct access to result |
| Replace method entirely | **Around Plugin** (or Preference) | Can skip original method |
| React to event (no return value) | **Observer** | Event-driven, decoupled |
| Add data to object | **After Plugin** or **Extension Attributes** | Depends on persistence need |
| Validate input | **Before Plugin** or **Validator** | Plugins for interception, Validators for reusable logic |
| Replace entire class | **Preference** (last resort) | Only if plugins insufficient |

### Plugin Use Cases

**Best suited for:**

1. **Price calculations**: Modify product price based on customer group
2. **Inventory checks**: Add custom stock validation
3. **URL generation**: Customize product URLs
4. **API responses**: Transform REST/GraphQL output
5. **Data enrichment**: Add calculated fields to collections
6. **Caching**: Wrap expensive methods with cache layer
7. **Logging**: Track method calls for debugging

**Example: Plugin for API Response Transformation**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Catalog\Api;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Framework\Api\SearchResultsInterface;

class ProductRepositoryExtend
{
    /**
     * Add computed field to product data after getById
     *
     * @param ProductRepositoryInterface $subject
     * @param ProductInterface $result
     * @param int $productId
     * @return ProductInterface
     */
    public function afterGetById(
        ProductRepositoryInterface $subject,
        ProductInterface $result,
        int $productId
    ): ProductInterface {
        // Add custom attribute to API response
        $result->setCustomAttribute('api_version', '2.0');
        $result->setCustomAttribute('response_time', microtime(true));

        return $result;
    }

    /**
     * Add computed fields to product list after getList
     *
     * @param ProductRepositoryInterface $subject
     * @param SearchResultsInterface $result
     * @return SearchResultsInterface
     */
    public function afterGetList(
        ProductRepositoryInterface $subject,
        SearchResultsInterface $result
    ): SearchResultsInterface {
        foreach ($result->getItems() as $product) {
            $product->setCustomAttribute('list_position', $this->calculatePosition($product));
        }

        return $result;
    }

    private function calculatePosition(ProductInterface $product): int
    {
        // Business logic
        return (int) $product->getId();
    }
}
```

### Observer Use Cases

**Best suited for:**

1. **Email notifications**: Send email after order placement
2. **Third-party integrations**: Sync data to external system
3. **Logging**: Record events for audit trail
4. **Cache invalidation**: Clear cache after data changes
5. **Analytics**: Track user behavior

**Example: Observer for Order Notification**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer\Sales;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Api\Data\OrderInterface;
use Psr\Log\LoggerInterface;

class OrderPlacedNotification implements ObserverInterface
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {
    }

    /**
     * Execute when sales_order_place_after event dispatches
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var OrderInterface $order */
        $order = $observer->getData('order');

        $this->logger->info('Order placed', [
            'order_id' => $order->getEntityId(),
            'customer_email' => $order->getCustomerEmail(),
            'grand_total' => $order->getGrandTotal()
        ]);

        // Send notification to external system
        // No return value needed
    }
}
```

**Registration** (`etc/events.xml`):

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="sales_order_place_after">
        <observer name="vendor_module_order_notification"
                  instance="Vendor\Module\Observer\Sales\OrderPlacedNotification" />
    </event>
</config>
```

### Preference Use Cases (Avoid When Possible)

**Only use preferences when:**

1. You must override **private/protected methods** (plugins can't intercept these)
2. You need to replace **entire class behavior** (multiple methods)
3. The class is **final** (plugins don't work on final classes)
4. You're fixing a **core bug** temporarily until patch available

**Preference risks:**

- **BC breaks**: Only one preference allowed per class (last one wins)
- **Upgrade issues**: Core changes conflict with your preference
- **Conflicts**: Other modules' preferences clash with yours

**Example: Preference for final class** (hypothetical):

```xml
<!-- Only if ClassName is final and you have no other choice -->
<preference for="Magento\Framework\SomeClass"
            type="Vendor\Module\Model\SomeClassCustom" />
```

**Better alternative**: Request Magento core to make class non-final or add plugin points.

## Common Plugin Mistakes

### Mistake 1: Wrong Return Type in After Plugin

```php
// WRONG: Changing return type
public function afterGetName(Product $subject, string $result): array
{
    return ['modified_name']; // Original returns string, not array!
}

// CORRECT: Return same type
public function afterGetName(Product $subject, string $result): string
{
    return 'Modified: ' . $result;
}
```

### Mistake 2: Not Calling $proceed in Around Plugin

```php
// WRONG: Breaking plugin chain
public function aroundSave(ProductRepository $subject, callable $proceed, ProductInterface $product): ProductInterface
{
    // Validation logic
    if (!$this->isValid($product)) {
        throw new \InvalidArgumentException('Invalid product');
    }

    // Forgot to call $proceed - original save never happens!
    return $product;
}

// CORRECT: Always call $proceed
public function aroundSave(ProductRepository $subject, callable $proceed, ProductInterface $product): ProductInterface
{
    if (!$this->isValid($product)) {
        throw new \InvalidArgumentException('Invalid product');
    }

    return $proceed($product); // Call original method
}
```

### Mistake 3: Modifying $subject State in Before Plugin

```php
// WRONG: Before plugins should not modify $subject state
public function beforeSave(Product $subject): ?array
{
    $subject->setName('Modified'); // Side effect!
    return null;
}

// CORRECT: Modify via arguments or use After/Around plugin
public function aroundSave(Product $subject, callable $proceed): Product
{
    $subject->setName('Modified'); // OK in Around plugin
    return $proceed();
}
```

### Mistake 4: Plugin on Final/Private/Static Methods

```php
// These will NOT work:
// - final public methods
// - private/protected methods
// - static methods
// - __construct() method

// Check if method is pluginable:
class SomeClass {
    final public function finalMethod() {} // Cannot plugin
    private function privateMethod() {} // Cannot plugin
    public static function staticMethod() {} // Cannot plugin
    public function __construct() {} // Cannot plugin

    public function pluginableMethod() {} // CAN plugin
}
```

### Mistake 5: Circular Dependencies

```php
// WRONG: Plugin creates circular dependency
namespace Vendor\Module\Plugin;

class ProductExtend {
    public function __construct(
        private readonly \Magento\Catalog\Model\Product $product // Circular!
    ) {}

    public function afterGetName(\Magento\Catalog\Model\Product $subject, string $result): string {
        return $this->product->getSku(); // Infinite loop!
    }
}

// CORRECT: Inject factory or avoid injecting plugged class
public function __construct(
    private readonly \Magento\Catalog\Model\ProductFactory $productFactory
) {}
```

## Performance Considerations

### Plugin Performance Impact

**Cost per plugin type**:

| Type | Overhead | Reason |
|------|----------|--------|
| Before | Low | Simple argument pass-through |
| After | Low | Simple result pass-through |
| Around | **High** | Wraps original method, adds call stack depth |

**General overhead**:
- Each plugin adds ~0.01-0.1ms per call (production mode)
- Around plugins: 2-5x slower than Before/After
- Compiled interceptors mitigate cost (use `production` mode)

### Optimization Strategies

#### 1. Minimize Around Plugins

```php
// AVOID: Around plugin for simple transformation
public function aroundGetPrice(Product $subject, callable $proceed): float
{
    $price = $proceed();
    return $price * 1.1; // 10% markup
}

// BETTER: Use After plugin
public function afterGetPrice(Product $subject, float $result): float
{
    return $result * 1.1;
}
```

#### 2. Avoid Heavy Operations in Plugins

```php
// AVOID: Database query in plugin on frequently-called method
public function afterGetName(Product $subject, string $result): string
{
    $customData = $this->resource->getConnection()->fetchOne(
        "SELECT custom_field FROM custom_table WHERE product_id = ?",
        [$subject->getId()]
    ); // Executes on EVERY getName() call!

    return $result . ' - ' . $customData;
}

// BETTER: Cache result or load in collection
private array $cache = [];

public function afterGetName(Product $subject, string $result): string
{
    $productId = $subject->getId();

    if (!isset($this->cache[$productId])) {
        $this->cache[$productId] = $this->getCustomData($productId);
    }

    return $result . ' - ' . $this->cache[$productId];
}
```

#### 3. Use Area-Specific Plugins

```php
// AVOID: Global plugin active everywhere
<!-- etc/di.xml -->
<type name="Magento\Catalog\Model\Product">
    <plugin name="frontend_specific_plugin" type="FrontendPlugin" />
</type>

// BETTER: Frontend-only plugin
<!-- etc/frontend/di.xml -->
<type name="Magento\Catalog\Model\Product">
    <plugin name="frontend_specific_plugin" type="FrontendPlugin" />
</type>
```

#### 4. Conditional Logic Early in Plugin

```php
// AVOID: Processing every call
public function afterLoad(Collection $subject, Collection $result): Collection
{
    foreach ($result->getItems() as $item) {
        // Expensive operation on every item
        $this->processItem($item);
    }
    return $result;
}

// BETTER: Early exit if not applicable
public function afterLoad(Collection $subject, Collection $result): Collection
{
    // Exit early if wrong area or empty collection
    if ($this->state->getAreaCode() !== Area::AREA_FRONTEND || $result->count() === 0) {
        return $result;
    }

    foreach ($result->getItems() as $item) {
        $this->processItem($item);
    }

    return $result;
}
```

### Generated Code Management

Plugins generate `Interceptor.php` files in `generated/code/`. Keep generated code clean:

```bash
# Clear generated code after di.xml changes
bin/magento setup:di:compile

# Verify plugin is registered
grep -r "YourPluginClass" generated/metadata/
```

## Production-Ready Plugin Example

Complete example following all best practices:

**Module structure**:
```
Vendor/Module/
├── Plugin/
│   └── Sales/
│       └── Api/
│           └── OrderRepositoryExtend.php
├── etc/
│   └── di.xml
├── registration.php
└── module.xml
```

**Plugin/Sales/Api/OrderRepositoryExtend.php**:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Sales\Api;

use Magento\Framework\Exception\LocalizedException;
use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Api\OrderRepositoryInterface;
use Psr\Log\LoggerInterface;

/**
 * Plugin to add audit trail when orders are saved
 *
 * @see OrderRepositoryInterface
 */
class OrderRepositoryExtend
{
    /**
     * @param LoggerInterface $logger
     * @param \Magento\Framework\Stdlib\DateTime\DateTime $dateTime
     * @param \Magento\Backend\Model\Auth\Session $authSession
     */
    public function __construct(
        private readonly LoggerInterface $logger,
        private readonly \Magento\Framework\Stdlib\DateTime\DateTime $dateTime,
        private readonly \Magento\Backend\Model\Auth\Session $authSession
    ) {
    }

    /**
     * Validate order before save
     *
     * @param OrderRepositoryInterface $subject
     * @param OrderInterface $order
     * @return array
     * @throws LocalizedException
     */
    public function beforeSave(
        OrderRepositoryInterface $subject,
        OrderInterface $order
    ): array {
        // Validation logic
        if ($order->getGrandTotal() < 0) {
            throw new LocalizedException(
                __('Order grand total cannot be negative.')
            );
        }

        $this->logger->debug('Order validation passed', [
            'order_id' => $order->getEntityId(),
            'grand_total' => $order->getGrandTotal()
        ]);

        // Return null to keep original arguments unchanged
        return null;
    }

    /**
     * Add audit trail after order is saved
     *
     * @param OrderRepositoryInterface $subject
     * @param OrderInterface $result Saved order
     * @param OrderInterface $order Original order argument
     * @return OrderInterface
     */
    public function afterSave(
        OrderRepositoryInterface $subject,
        OrderInterface $result,
        OrderInterface $order
    ): OrderInterface {
        // Get admin user who made the change
        $adminUser = $this->authSession->getUser();
        $adminUsername = $adminUser ? $adminUser->getUserName() : 'system';

        // Log audit trail
        $this->logger->info('Order saved', [
            'order_id' => $result->getEntityId(),
            'admin_user' => $adminUsername,
            'timestamp' => $this->dateTime->gmtDate(),
            'status' => $result->getStatus(),
            'state' => $result->getState()
        ]);

        // Add audit data as custom attribute (optional)
        $result->setData('last_modified_by', $adminUsername);
        $result->setData('last_modified_at', $this->dateTime->gmtDate());

        return $result;
    }
}
```

**etc/di.xml**:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <type name="Magento\Sales\Api\OrderRepositoryInterface">
        <plugin name="vendor_module_order_audit_plugin"
                type="Vendor\Module\Plugin\Sales\Api\OrderRepositoryExtend"
                sortOrder="100"
                disabled="false" />
    </type>

</config>
```

**Why this example is production-ready**:

1. **Type declarations**: All parameters and return types explicitly declared (PHP 8.2+ strict types)
2. **Constructor DI**: Dependencies injected, testable
3. **Readonly properties**: Immutable dependencies (PHP 8.1+)
4. **Exception handling**: Validation with meaningful exception messages
5. **Logging**: Debug and info logs for troubleshooting
6. **Documentation**: PHPDoc with @see, @param, @return, @throws
7. **Naming**: Clear plugin name following convention
8. **sortOrder**: Explicit sortOrder for predictable execution
9. **Service contract**: Plugs interface, not implementation (upgrade-safe)
10. **No side effects**: Before plugin doesn't modify $subject

## Debugging Plugins

### Check Plugin Registration

```bash
# List all plugins for a class
bin/magento dev:di:info "Magento\Catalog\Model\Product"

# Output shows all configured plugins
```

### Inspect Generated Interceptor

```bash
# View generated plugin wrapper
cat generated/code/Magento/Catalog/Model/Product/Interceptor.php

# Shows exact plugin execution order
```

### Enable Debug Logging

```php
// In plugin method
$this->logger->debug('Plugin executed', [
    'method' => __METHOD__,
    'subject_class' => get_class($subject),
    'arguments' => func_get_args()
]);
```

### Test Plugin in Isolation

```php
// Unit test for plugin
namespace Vendor\Module\Test\Unit\Plugin;

use PHPUnit\Framework\TestCase;
use Vendor\Module\Plugin\MyPluginExtend;

class MyPluginExtendTest extends TestCase
{
    private MyPluginExtend $plugin;

    protected function setUp(): void
    {
        $this->plugin = new MyPluginExtend(
            $this->createMock(\Psr\Log\LoggerInterface::class)
        );
    }

    public function testAfterGetName(): void
    {
        $subjectMock = $this->createMock(\Magento\Catalog\Model\Product::class);
        $originalResult = 'Product Name';

        $result = $this->plugin->afterGetName($subjectMock, $originalResult);

        $this->assertStringContainsString('Modified', $result);
    }
}
```

## Testing Plugins

### Unit Test Example

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Unit\Plugin\Catalog\Model\Product;

use Magento\Catalog\Model\Product;
use PHPUnit\Framework\TestCase;
use Psr\Log\LoggerInterface;
use Vendor\Module\Plugin\Catalog\Model\Product\PriceExtend;

class PriceExtendTest extends TestCase
{
    private PriceExtend $plugin;
    private LoggerInterface $loggerMock;
    private Product $productMock;

    protected function setUp(): void
    {
        $this->loggerMock = $this->createMock(LoggerInterface::class);
        $this->productMock = $this->createMock(Product::class);

        $this->plugin = new PriceExtend($this->loggerMock);
    }

    public function testAfterGetPriceAppliesMarkup(): void
    {
        $originalPrice = 100.0;
        $expectedPrice = 110.0; // 10% markup

        $result = $this->plugin->afterGetPrice($this->productMock, $originalPrice);

        $this->assertEquals($expectedPrice, $result);
    }

    public function testAfterGetPriceLogsDebugInfo(): void
    {
        $this->productMock->method('getId')->willReturn(123);

        $this->loggerMock->expects($this->once())
            ->method('debug')
            ->with(
                $this->stringContains('Price modified'),
                $this->arrayHasKey('product_id')
            );

        $this->plugin->afterGetPrice($this->productMock, 100.0);
    }
}
```

### Integration Test Example

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Integration\Plugin;

use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;

class PricePluginIntegrationTest extends TestCase
{
    private ProductRepositoryInterface $productRepository;

    protected function setUp(): void
    {
        $this->productRepository = Bootstrap::getObjectManager()
            ->get(ProductRepositoryInterface::class);
    }

    /**
     * @magentoDataFixture Magento/Catalog/_files/product_simple.php
     * @magentoAppArea frontend
     */
    public function testPluginModifiesProductPrice(): void
    {
        $product = $this->productRepository->get('simple');

        $originalPrice = 10.0;
        $this->assertEquals($originalPrice, $product->getPrice());

        // After plugin should apply 10% markup
        $finalPrice = $product->getFinalPrice();
        $this->assertEquals(11.0, $finalPrice);
    }
}
```

## Assumptions

- Target: Adobe Commerce / Magento Open Source 2.4.7
- PHP: 8.2+
- Environment: Development and production
- Scope: Backend (PHP modules, DI, plugins)
- Extensions: Custom modules in app/code or Composer packages

## Why This Approach

- **Plugins over Preferences**: Upgrade-safe; multiple plugins can coexist
- **Service Contracts**: Plugin interfaces, not implementations, for BC
- **Before/After over Around**: Lower performance cost when possible
- **Type Declarations**: PHP 8.2+ strict types for reliability
- **Constructor DI**: Explicit dependencies, testable
- **Logging**: Debug and info logs for production troubleshooting
- **Early Exit**: Conditional logic at start of method reduces overhead

Alternatives considered:
- **Observers**: No return value; can't modify method behavior
- **Preferences**: Only one allowed; BC risk; conflicts common
- **Event Dispatch**: Custom events for decoupling; plugins for interception
- **Rewrites (Magento 1)**: Deprecated; plugins replace rewrites in Magento 2

## Security Impact

### Authentication/Authorization

- Plugins on admin controllers: Ensure ACL checks not bypassed
- Plugins on API endpoints: Verify token/OAuth validation occurs
- Example: Plugin on `OrderRepositoryInterface::save()` must not skip ACL

### CSRF/Form Keys

- Plugins on POST handlers: Ensure form key validation not skipped
- Do not remove CSRF protection in plugins

### XSS Escaping

- Plugins modifying HTML output: Escape all user input
- Example: `afterGetName()` returning HTML must use `escapeHtml()`

### PII/GDPR

- Plugins logging customer data: Ensure PII not logged in plain text
- Anonymize logs: `$this->logger->info('Customer action', ['customer_id' => hash('sha256', $customerId)])`

### Secrets Management

- Never log API keys, passwords, tokens in plugins
- Use `LoggerInterface` with log level awareness (debug vs info)

## Performance Impact

### Full Page Cache (FPC)

- Plugins on cacheable blocks: May break FPC
- ESI holes for dynamic content: Use Varnish ESI instead of plugins
- Cache invalidation: Plugins modifying cached data must invalidate cache tags

### Database Load

- Avoid N+1 queries in plugins on collection load
- Use `afterLoad` to batch-fetch data, not per-item queries

### Core Web Vitals (CWV)

- Plugins on frontend rendering: Minimize processing time
- Use profiler to measure plugin overhead: `bin/magento dev:profiler:enable`

### Cacheability

- Plugins on `CacheInterface`: Can wrap cache operations for logging/metrics
- Ensure cache keys unique: Include all parameters affecting output

## Backward Compatibility

### API/DB Schema Changes

- Plugins on data interfaces: Ensure added attributes don't break serialization
- Use extension attributes for adding data to entities

### Upgrade Path

- Magento 2.4.6 → 2.4.7: Plugins remain compatible if targeting service contracts
- Magento 2.4.7 → 2.4.8/2.4.9: Verify plugin methods still exist; check deprecation notices

### Migration Notes

- Review Magento release notes for deprecated methods
- Test plugins against new minor versions before upgrading
- Use `@deprecated` annotations in your plugin methods if replacing with new approach

## Tests to Add

### Unit Tests

- Test each plugin method in isolation
- Mock `$subject`, verify return types
- Test edge cases (null values, empty collections, exceptions)

### Integration Tests

- Test plugin in real Magento environment with DI
- Verify plugin execution order with multiple plugins
- Test area-specific plugins (frontend vs adminhtml)

### Functional Tests (MFTF)

- Test end-to-end flows affected by plugins
- Example: Product add to cart with price plugin active

### Performance Tests

- Measure plugin overhead with `bin/magento dev:profiler:enable`
- Compare before/after plugin activation
- Target: <5ms overhead per plugin in production mode

## Documentation to Update

### Code Comments

- PHPDoc for all plugin methods: `@param`, `@return`, `@throws`, `@see`
- Explain **why** plugin is needed, not just **what** it does

### Module README

```markdown
## Plugins

This module registers the following plugins:

- `Vendor\Module\Plugin\Catalog\Model\Product\PriceExtend`: Applies 10% markup to product prices for customer group 3 (Wholesale). Registered on `Magento\Catalog\Model\Product::getPrice()`.

sortOrder: 20 (runs after core price calculations)
```

### CHANGELOG

```markdown
## [1.0.0] - 2026-02-05
### Added
- Plugin on `ProductRepositoryInterface::save()` for order audit trail
- Plugin on `Product::getFinalPrice()` for dynamic pricing
```

### Admin User Guide

- Document observable behavior changes caused by plugins
- Example: "Wholesale customers (group 3) automatically receive 10% discount on all products"

## Related Documentation

### Related Guides

- [Service Contracts vs Repositories in Magento 2](../explanations/service-contracts-repositories.md)
- [Comprehensive Testing Strategies for Magento 2](../how-to/testing-strategies.md)
- [Declarative Schema & Data Patches: Modern Database Management in Magento 2](declarative-schema-data-patches.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Customer Overview](../../modules/customer/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
