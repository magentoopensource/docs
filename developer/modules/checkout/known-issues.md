---
title: "Magento_Checkout Known Issues"
module: "Magento_Checkout"
doc_type: "known-issues"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Checkout Known Issues

## Overview

This document catalogs known issues, bugs, and quirks in the `Magento_Checkout` module across Magento 2.4.x versions. Each issue includes symptoms, root cause, workarounds, and permanent fixes where available.

**Target Version:** Magento 2.4.7+ (Adobe Commerce & Open Source)

---

## Issue 1: Quote Address Street Array Index Issue

### Severity: MEDIUM | Affected Versions: 2.4.0 - 2.4.6

### Symptoms

- Checkout fails with error: "Notice: Undefined offset: 1 in Quote/Address.php"
- Street address lines beyond first line are lost
- Billing/shipping addresses incomplete after save

### Root Cause

The `street` field in quote addresses is stored as an array, but some address population methods expect a string. When street has only one line, accessing `$street[1]` triggers an undefined offset notice.

```php
// Problematic code in Magento\Quote\Model\Quote\Address
public function getStreet()
{
    $street = parent::getStreet();
    if (!is_array($street)) {
        $street = explode("\n", $street);
    }
    // Accessing $street[1] without checking if it exists
    return $street;
}
```

### Workaround

Validate street array before access:

```php
namespace Vendor\Module\Plugin;

use Magento\Quote\Model\Quote\Address;

class AddressStreetFix
{
    /**
     * Fix street array access
     *
     * @param Address $subject
     * @param array|string|null $result
     * @return array
     */
    public function afterGetStreet(
        Address $subject,
        $result
    ): array {
        if (!is_array($result)) {
            $result = $result ? explode("\n", $result) : [];
        }

        // Ensure array has at least 2 elements
        while (count($result) < 2) {
            $result[] = '';
        }

        return $result;
    }
}
```

**Register in `di.xml`:**

```xml
<type name="Magento\Quote\Model\Quote\Address">
    <plugin name="vendor_module_address_street_fix"
            type="Vendor\Module\Plugin\AddressStreetFix"
            sortOrder="10"/>
</type>
```

### Permanent Fix

This issue is partially addressed in Magento 2.4.7 with improved street field handling. Verify your specific implementation.

---

## Issue 2: Checkout Totals Not Updating After Shipping Method Change

### Severity: HIGH | Affected Versions: 2.4.0 - 2.4.7

### Symptoms

- Customer changes shipping method
- Shipping cost updates on frontend
- Grand total doesn't recalculate
- Tax calculation incorrect
- Order placed with wrong total

### Root Cause

Race condition in Knockout.js observable subscriptions causes totals update to fire before shipping method is fully saved. The `totals` observable updates before the server confirms the new shipping method.

```javascript
// Problematic sequence:
// 1. Customer selects new shipping method
// 2. Frontend updates UI immediately (optimistic)
// 3. API call to save shipping method starts
// 4. Totals collector runs with old shipping cost
// 5. API call completes
// 6. Totals are now stale
```

### Workaround

Force totals refresh after shipping method save:

```javascript
// Vendor/Module/view/frontend/web/js/view/shipping-method-refresh.js
define([
    'uiComponent',
    'Magento_Checkout/js/model/quote',
    'Magento_Checkout/js/action/get-totals',
    'Magento_Checkout/js/model/shipping-service'
], function (Component, quote, getTotalsAction, shippingService) {
    'use strict';

    return Component.extend({
        initialize: function () {
            this._super();

            // Subscribe to shipping method changes
            quote.shippingMethod.subscribe(function (newMethod) {
                if (newMethod) {
                    // Wait for server response before refreshing totals
                    setTimeout(function () {
                        getTotalsAction([]);
                    }, 1000);
                }
            });

            return this;
        }
    });
});
```

**Alternative: Server-Side Fix**

Add plugin to force totals collection:

```php
namespace Vendor\Module\Plugin;

use Magento\Checkout\Api\ShippingInformationManagementInterface;
use Magento\Checkout\Api\Data\ShippingInformationInterface;

class ForceTotalsRefresh
{
    public function __construct(
        private readonly \Magento\Quote\Api\CartRepositoryInterface $quoteRepository,
        private readonly \Magento\Quote\Api\CartTotalRepositoryInterface $cartTotalsRepository
    ) {}

    /**
     * Force totals recalculation after shipping method save
     */
    public function afterSaveAddressInformation(
        ShippingInformationManagementInterface $subject,
        \Magento\Checkout\Api\Data\PaymentDetailsInterface $result,
        int $cartId,
        ShippingInformationInterface $addressInformation
    ): \Magento\Checkout\Api\Data\PaymentDetailsInterface {

        // Force fresh totals calculation
        $quote = $this->quoteRepository->getActive($cartId);
        $quote->setTotalsCollectedFlag(false);
        $quote->collectTotals();
        $this->quoteRepository->save($quote);

        // Update result with fresh totals
        $totals = $this->cartTotalsRepository->get($cartId);
        $result->setTotals($totals);

        return $result;
    }
}
```

---

## Issue 3: Guest Email Validation Race Condition

### Severity: MEDIUM | Affected Versions: 2.4.0 - 2.4.6

### Symptoms

- Guest enters email that matches existing customer
- "Email already exists" warning doesn't appear
- Guest can complete checkout with existing customer email
- Duplicate customer records created

### Root Cause

Email availability check is debounced on frontend, but customer can click "Next" before the check completes. Backend doesn't re-validate email uniqueness during order placement.

### Workaround

Add server-side email validation:

```php
namespace Vendor\Module\Plugin;

use Magento\Checkout\Api\GuestPaymentInformationManagementInterface;
use Magento\Quote\Api\Data\PaymentInterface;
use Magento\Quote\Api\Data\AddressInterface;
use Magento\Framework\Exception\LocalizedException;

class ValidateGuestEmail
{
    public function __construct(
        private readonly \Magento\Customer\Api\CustomerRepositoryInterface $customerRepository,
        private readonly \Magento\Framework\Api\SearchCriteriaBuilder $searchCriteriaBuilder,
        private readonly \Magento\Quote\Api\CartRepositoryInterface $quoteRepository
    ) {}

    /**
     * Validate guest email before placing order
     */
    public function beforeSavePaymentInformationAndPlaceOrder(
        GuestPaymentInformationManagementInterface $subject,
        string $cartId,
        string $email,
        PaymentInterface $paymentMethod,
        ?AddressInterface $billingAddress = null
    ): array {

        // Check if email belongs to existing customer
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('email', $email)
            ->create();

        $customers = $this->customerRepository->getList($searchCriteria);

        if ($customers->getTotalCount() > 0) {
            throw new LocalizedException(
                __('A customer with this email address already exists. Please log in or use a different email.')
            );
        }

        return [$cartId, $email, $paymentMethod, $billingAddress];
    }
}
```

---

## Issue 4: Payment Method Disappears After Address Change

### Severity: HIGH | Affected Versions: 2.4.0 - 2.4.7

### Symptoms

- Customer selects payment method
- Changes billing/shipping address
- Payment method selection resets
- Customer must re-select payment method

### Root Cause

Address change triggers payment method re-evaluation. If payment method's `isAvailable()` check depends on address data (e.g., country restrictions), the selected method may become unavailable and is cleared from the quote.

```php
// Magento\Payment\Model\Method\AbstractMethod::isAvailable()
public function isAvailable(\Magento\Quote\Api\Data\CartInterface $quote = null): bool
{
    if (!$this->isActive($quote ? $quote->getStoreId() : null)) {
        return false;
    }

    // Payment method checks specific countries
    $allowedCountries = $this->getConfigData('allowspecific');
    if ($allowedCountries && $quote) {
        $billingCountry = $quote->getBillingAddress()->getCountryId();
        // If country changes, method may become unavailable
    }

    return true;
}
```

### Workaround

Preserve payment method if still available:

```php
namespace Vendor\Module\Plugin;

use Magento\Checkout\Api\ShippingInformationManagementInterface;
use Magento\Checkout\Api\Data\ShippingInformationInterface;

class PreservePaymentMethod
{
    public function __construct(
        private readonly \Magento\Quote\Api\CartRepositoryInterface $quoteRepository,
        private readonly \Magento\Payment\Api\PaymentMethodListInterface $paymentMethodList,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Preserve payment method after address change if still available
     */
    public function afterSaveAddressInformation(
        ShippingInformationManagementInterface $subject,
        \Magento\Checkout\Api\Data\PaymentDetailsInterface $result,
        int $cartId,
        ShippingInformationInterface $addressInformation
    ): \Magento\Checkout\Api\Data\PaymentDetailsInterface {

        try {
            $quote = $this->quoteRepository->getActive($cartId);
            $currentPaymentMethod = $quote->getPayment()->getMethod();

            if ($currentPaymentMethod) {
                // Check if current payment method is still available
                $availableMethods = $this->paymentMethodList->getList($cartId);
                $methodCodes = array_map(function ($method) {
                    return $method->getCode();
                }, $availableMethods);

                if (!in_array($currentPaymentMethod, $methodCodes)) {
                    // Method no longer available, clear it
                    $quote->getPayment()->setMethod(null);
                    $this->quoteRepository->save($quote);

                    $this->logger->warning('Payment method cleared after address change', [
                        'quote_id' => $cartId,
                        'payment_method' => $currentPaymentMethod,
                        'new_country' => $quote->getBillingAddress()->getCountryId()
                    ]);
                }
            }

        } catch (\Exception $e) {
            $this->logger->error('Failed to preserve payment method', [
                'error' => $e->getMessage()
            ]);
        }

        return $result;
    }
}
```

---

## Issue 5: Minicart Quantity Update Race Condition

### Severity: MEDIUM | Affected Versions: 2.4.0 - 2.4.7

### Symptoms

- Customer updates quantity in minicart
- Minicart shows updated quantity
- Checkout shows old quantity
- Customer must reload checkout to see correct quantity

### Root Cause

Minicart uses AJAX to update quantity, but checkout page caches quote data in JavaScript. The `window.checkoutConfig` object is not automatically refreshed after cart updates.

### Workaround

Force checkout data refresh on cart updates:

```javascript
// Vendor/Module/view/frontend/web/js/cart-update-listener.js
define([
    'uiComponent',
    'Magento_Customer/js/customer-data',
    'Magento_Checkout/js/model/quote'
], function (Component, customerData, quote) {
    'use strict';

    return Component.extend({
        initialize: function () {
            this._super();

            // Listen for cart updates
            var cart = customerData.get('cart');
            cart.subscribe(function (updatedCart) {
                // Reload page if items or quantity changed
                if (window.location.pathname.indexOf('/checkout') !== -1) {
                    window.location.reload();
                }
            });

            return this;
        }
    });
});
```

**Better Solution: Real-time Quote Sync**

```php
namespace Vendor\Module\Controller\Cart;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\Controller\Result\JsonFactory;

class UpdateCheckout implements HttpPostActionInterface
{
    public function __construct(
        private readonly \Magento\Checkout\Model\Session $checkoutSession,
        private readonly \Magento\Quote\Api\CartRepositoryInterface $quoteRepository,
        private readonly JsonFactory $resultJsonFactory
    ) {}

    /**
     * Get updated quote data for checkout refresh
     */
    public function execute(): \Magento\Framework\Controller\Result\Json
    {
        $result = $this->resultJsonFactory->create();

        try {
            $quote = $this->checkoutSession->getQuote();
            $quote->setTotalsCollectedFlag(false);
            $quote->collectTotals();
            $this->quoteRepository->save($quote);

            $result->setData([
                'success' => true,
                'totals' => $quote->getTotals(),
                'items' => $this->getQuoteItems($quote)
            ]);

        } catch (\Exception $e) {
            $result->setData([
                'success' => false,
                'error' => $e->getMessage()
            ]);
        }

        return $result;
    }

    /**
     * Get quote items data
     */
    private function getQuoteItems(\Magento\Quote\Model\Quote $quote): array
    {
        $items = [];
        foreach ($quote->getAllVisibleItems() as $item) {
            $items[] = [
                'item_id' => $item->getId(),
                'name' => $item->getName(),
                'qty' => $item->getQty(),
                'price' => $item->getPrice()
            ];
        }
        return $items;
    }
}
```

---

## Issue 6: Virtual Products Break Shipping Step

### Severity: HIGH | Affected Versions: 2.4.0 - 2.4.6

### Symptoms

- Cart contains both virtual and physical products
- Shipping step shows but cannot be completed
- Error: "Please specify a shipping method"
- Shipping methods don't load

### Root Cause

Quote is considered virtual if all items are virtual, but logic doesn't properly handle mixed carts. Shipping step renders but shipping rates aren't calculated for mixed quote types.

```php
// Magento\Quote\Model\Quote::isVirtual()
public function isVirtual(): bool
{
    $isVirtual = true;
    foreach ($this->getAllItems() as $item) {
        if (!$item->getProduct()->getIsVirtual()) {
            $isVirtual = false;
            break;
        }
    }
    return $isVirtual;
}
```

### Workaround

Ensure mixed carts properly collect shipping rates:

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

class EnsureShippingForMixedCart implements ObserverInterface
{
    /**
     * Ensure shipping rates collected for mixed carts
     */
    public function execute(Observer $observer): void
    {
        /** @var \Magento\Quote\Model\Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        // Check if cart has both virtual and physical items
        $hasVirtual = false;
        $hasPhysical = false;

        foreach ($quote->getAllItems() as $item) {
            if ($item->getProduct()->getIsVirtual()) {
                $hasVirtual = true;
            } else {
                $hasPhysical = true;
            }

            if ($hasVirtual && $hasPhysical) {
                break;
            }
        }

        // For mixed carts, ensure shipping address exists and rates are collected
        if ($hasVirtual && $hasPhysical) {
            $shippingAddress = $quote->getShippingAddress();
            if ($shippingAddress && !$shippingAddress->getShippingMethod()) {
                $shippingAddress->setCollectShippingRates(true);
            }
        }
    }
}
```

**Register Observer:**

```xml
<event name="sales_quote_collect_totals_before">
    <observer name="vendor_module_mixed_cart_shipping"
              instance="Vendor\Module\Observer\EnsureShippingForMixedCart"/>
</event>
```

---

## Issue 7: Session Lock Timeout During High Traffic

### Severity: CRITICAL | Affected Versions: 2.4.0 - 2.4.7

### Symptoms

- Checkout hangs at "Processing..." indefinitely
- Error logs: "Unable to acquire lock on session"
- Happens during peak traffic hours
- Multiple concurrent requests from same customer

### Root Cause

PHP session locking mechanism causes concurrent AJAX requests from checkout to queue. If one request takes too long, subsequent requests wait for session lock, eventually timing out.

```php
// Session lock flow:
// Request 1: Acquire lock → Process → Release lock (5s)
// Request 2: Wait for lock → Wait... → Wait... → Timeout (30s)
// Request 3: Wait for lock → Wait... → Wait... → Timeout (30s)
```

### Workaround

Use Redis session storage with optimized locking:

**`app/etc/env.php`:**

```php
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
],
```

**Alternative: Close Session Early**

```php
namespace Vendor\Module\Controller\Ajax;

use Magento\Framework\App\Action\HttpPostActionInterface;

class QuickResponse implements HttpPostActionInterface
{
    public function __construct(
        private readonly \Magento\Framework\Session\SessionManagerInterface $session
    ) {}

    public function execute()
    {
        // Process data that doesn't need session write
        $data = $this->processData();

        // Close session early to release lock
        $this->session->writeClose();

        // Continue with slow operations (external API calls, etc.)
        $result = $this->slowOperation($data);

        return $this->jsonResponse($result);
    }
}
```

---

## Issue 8: Custom Customer Attributes Lost During Checkout

### Severity: MEDIUM | Affected Versions: 2.4.0 - 2.4.7

### Symptoms

- Custom customer attributes defined in `customer_eav_attribute` table
- Attributes set during registration or account update
- Attributes not available in checkout
- Order doesn't include custom attribute values

### Root Cause

Custom customer attributes aren't automatically included in checkout config provider. The `window.checkoutConfig.customerData` object only includes default attributes.

### Workaround

Create custom config provider:

```php
namespace Vendor\Module\Model\Checkout;

use Magento\Checkout\Model\ConfigProviderInterface;
use Magento\Customer\Model\Session as CustomerSession;

class CustomAttributeConfigProvider implements ConfigProviderInterface
{
    public function __construct(
        private readonly CustomerSession $customerSession,
        private readonly \Magento\Customer\Api\CustomerRepositoryInterface $customerRepository,
        private readonly \Magento\Framework\Api\ExtensibleDataObjectConverter $dataObjectConverter
    ) {}

    /**
     * Provide custom customer attributes to checkout
     */
    public function getConfig(): array
    {
        $config = [];

        if ($this->customerSession->isLoggedIn()) {
            $customerId = $this->customerSession->getCustomerId();

            try {
                $customer = $this->customerRepository->getById($customerId);

                $config['customCustomerData'] = [
                    'loyalty_tier' => $customer->getCustomAttribute('loyalty_tier')
                        ? $customer->getCustomAttribute('loyalty_tier')->getValue()
                        : null,
                    'tax_exempt' => $customer->getCustomAttribute('tax_exempt')
                        ? $customer->getCustomAttribute('tax_exempt')->getValue()
                        : false,
                    'preferred_delivery_time' => $customer->getCustomAttribute('preferred_delivery_time')
                        ? $customer->getCustomAttribute('preferred_delivery_time')->getValue()
                        : null,
                ];

            } catch (\Exception $e) {
                // Log error but don't break checkout
            }
        }

        return $config;
    }
}
```

**Register Config Provider:**

```xml
<type name="Magento\Checkout\Model\CompositeConfigProvider">
    <arguments>
        <argument name="configProviders" xsi:type="array">
            <item name="custom_attribute_config_provider" xsi:type="object">
                Vendor\Module\Model\Checkout\CustomAttributeConfigProvider
            </item>
        </argument>
    </arguments>
</type>
```

---

## Issue 9: Incorrect Tax Calculation After Coupon Applied

### Severity: HIGH | Affected Versions: 2.4.0 - 2.4.6

### Symptoms

- Customer applies discount coupon
- Discount applied to subtotal
- Tax calculated on original subtotal (before discount)
- Grand total incorrect

### Root Cause

Tax collector runs before discount collector in totals collection sequence. Tax is calculated on pre-discount amount.

### Workaround

Adjust totals collector sort order:

```xml
<!-- etc/sales.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Sales:etc/sales.xsd">
    <section name="quote">
        <group name="totals">
            <!-- Ensure discount runs before tax -->
            <item name="discount" instance="Magento\SalesRule\Model\Quote\Discount" sort_order="300"/>
            <item name="tax" instance="Magento\Tax\Model\Sales\Total\Quote\Tax" sort_order="400"/>
        </group>
    </section>
</config>
```

**Note:** In Magento 2.4.7, this is corrected in core, but verify your specific tax configuration.

---

## Issue 10: Place Order Button Disabled After Failed Payment

### Severity: MEDIUM | Affected Versions: 2.4.0 - 2.4.7

### Symptoms

- Payment authorization fails (declined card, insufficient funds, etc.)
- Error message displayed to customer
- "Place Order" button remains disabled
- Customer must reload page to retry

### Root Cause

JavaScript component sets `isPlaceOrderActionAllowed(false)` when placing order, but doesn't re-enable on error.

```javascript
// Magento_Checkout/js/view/payment/default.js
placeOrder: function () {
    this.isPlaceOrderActionAllowed(false); // Disable button

    this.getPlaceOrderDeferredObject()
        .done(function () {
            // Success - redirect
        }).fail(function () {
            // Error - button stays disabled!
        });
}
```

### Workaround

Re-enable button on error:

```javascript
// Vendor/Module/view/frontend/web/js/view/payment/method-renderer/custom.js
define([
    'Magento_Checkout/js/view/payment/default'
], function (Component) {
    'use strict';

    return Component.extend({
        defaults: {
            template: 'Vendor_Module/payment/custom'
        },

        placeOrder: function (data, event) {
            var self = this;

            if (event) {
                event.preventDefault();
            }

            if (this.validate() && this.isPlaceOrderActionAllowed() === true) {
                this.isPlaceOrderActionAllowed(false);

                this.getPlaceOrderDeferredObject()
                    .done(function () {
                        self.afterPlaceOrder();

                        if (self.redirectAfterPlaceOrder) {
                            redirectOnSuccessAction.execute();
                        }
                    }).fail(function () {
                        // Re-enable button on error
                        self.isPlaceOrderActionAllowed(true);
                    });

                return true;
            }

            return false;
        }
    });
});
```

---

## Issue Severity Classification

| Severity | Definition | Response Time |
|----------|------------|---------------|
| CRITICAL | Prevents order placement or causes data loss | Immediate |
| HIGH | Degrades checkout experience significantly | 1-2 business days |
| MEDIUM | Causes inconvenience but has workaround | 1-2 weeks |
| LOW | Cosmetic or minor functional issue | As resources permit |

---

## Reporting New Issues

When reporting checkout issues:

1. **Magento Version**: Exact version (e.g., 2.4.7-p1)
2. **Reproduction Steps**: Detailed step-by-step instructions
3. **Expected Behavior**: What should happen
4. **Actual Behavior**: What actually happens
5. **Environment**: PHP version, web server, session storage
6. **Extensions**: List of installed checkout-related extensions
7. **Logs**: Relevant error logs from `var/log/`
8. **Screenshots**: Visual evidence of issue

---

## Assumptions

- **Target Platform**: Adobe Commerce & Magento Open Source 2.4.7+
- **PHP Version**: 8.1, 8.2, 8.3
- **Session Storage**: Redis (recommended for production)
- **Deployment Mode**: Production mode for performance testing

## Why Document Known Issues

- **Faster Troubleshooting**: Teams can quickly identify and resolve common problems
- **Reduce Support Tickets**: Known issues with workarounds reduce customer support load
- **Informed Development**: Developers can avoid triggering known bugs
- **Upgrade Planning**: Understanding version-specific issues helps plan upgrades

## Security Impact

- **Session Management**: Session lock timeout issues can be exploited for DoS attacks
- **Email Validation**: Guest email race condition could lead to account hijacking
- **Payment Failures**: Failed payment retry issues may expose payment data in logs

## Performance Impact

- **Session Locks**: Critical performance bottleneck during high traffic
- **Totals Collection**: Race conditions cause unnecessary recalculation
- **Quote Refresh**: Excessive quote reloads impact database performance

## Backward Compatibility

- **Workarounds**: All workarounds use plugins/observers to maintain upgrade path
- **Configuration**: System config changes won't break on upgrade
- **Data Integrity**: Fixes don't modify core database schema

## Tests to Add

1. **Regression Tests**: Verify known issues don't reappear after fixes
2. **Load Tests**: Test session locking under concurrent load
3. **Integration Tests**: Test mixed cart scenarios (virtual + physical)
4. **API Tests**: Verify payment method availability logic

## Documentation to Update

- **Troubleshooting Guide**: Include symptoms and solutions for known issues
- **Release Notes**: Document which issues are fixed in each version
- **Developer Guide**: Best practices to avoid triggering known bugs
- **Upgrade Guide**: Version-specific issues to watch during upgrades
