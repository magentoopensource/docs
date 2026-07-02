---
title: "Magento_Checkout Overview"
module: "Magento_Checkout"
doc_type: "overview"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Checkout Module

## Overview

The `Magento_Checkout` module is the cornerstone of Magento's order placement workflow, orchestrating the multi-step checkout process from cart review through payment processing to order creation. This module provides a highly extensible, componentized architecture built on Knockout.js UI components, service contracts, and a robust plugin system.

**Target Version:** Magento 2.4.7+ (Adobe Commerce & Open Source)
**PHP Requirements:** 8.1, 8.2, 8.3
**Frontend Stack:** Knockout.js, RequireJS, UI Components, LESS

## Module Purpose

Magento_Checkout serves three primary functions:

1. **Multi-Step Checkout Flow** - Orchestrates shipping, billing, payment, and order review steps with validation and state persistence
2. **Frontend Checkout UI** - Provides a component-based UI framework for rendering checkout steps, forms, and payment methods
3. **Order Placement Contract** - Defines service contracts and integration points for converting quotes to orders

## Key Features

### 1. Checkout Architecture

The checkout is built as a **single-page application (SPA)** using Knockout.js, with each step rendered as a UI component:

- **Shipping Step**: Address entry, shipping method selection, validation
- **Payment Step**: Billing address, payment method selection, payment form rendering
- **Order Review**: Summary sidebar, totals calculation, place order action
- **Success Page**: Order confirmation, registration prompt for guests

### 2. Service Contracts

The module exposes several critical service contracts:

- **`GuestPaymentInformationManagementInterface`** - Place orders for guest customers
- **`PaymentInformationManagementInterface`** - Place orders for registered customers
- **`ShippingInformationManagementInterface`** - Save shipping information and calculate totals
- **`TotalsInformationManagementInterface`** - Calculate totals for address information
- **`GuestShippingInformationManagementInterface`** - Guest shipping operations

### 3. Checkout Session Management

`\Magento\Checkout\Model\Session` extends `\Magento\Framework\Session\SessionManager`:

```php
namespace Magento\Checkout\Model;

use Magento\Quote\Api\Data\CartInterface;
use Magento\Framework\Session\SessionManager;

class Session extends SessionManager
{
    /**
     * @var \Magento\Quote\Model\Quote|null
     */
    protected $_quote;

    /**
     * Get active quote from session
     *
     * @return CartInterface
     * @throws \Magento\Framework\Exception\LocalizedException
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function getQuote(): CartInterface
    {
        if ($this->_quote === null) {
            $quoteId = $this->getQuoteId();
            if ($quoteId) {
                $this->_quote = $this->quoteRepository->getActive($quoteId);
            }
        }
        return $this->_quote;
    }

    /**
     * Reset checkout session state
     *
     * @return $this
     */
    public function clearQuote()
    {
        $this->_quote = null;
        $this->setQuoteId(null);
        $this->setLastSuccessQuoteId(null);
        // Also dispatches 'checkout_quote_destroy' event
        return $this;
    }

    /**
     * Get last order ID placed through checkout
     *
     * @return int|null
     */
    public function getLastOrderId(): ?int
    {
        return $this->storage->getData('last_order_id');
    }
}
```

### 4. Checkout Configuration

The module provides `\Magento\Checkout\Model\ConfigProviderInterface` for injecting data into the checkout UI:

```php
namespace Vendor\CustomCheckout\Model;

use Magento\Checkout\Model\ConfigProviderInterface;
use Magento\Checkout\Model\Session as CheckoutSession;

class CustomConfigProvider implements ConfigProviderInterface
{
    public function __construct(
        private readonly CheckoutSession $checkoutSession,
        private readonly \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig
    ) {}

    /**
     * Provide custom checkout configuration
     *
     * @return array<string, mixed>
     */
    public function getConfig(): array
    {
        $quote = $this->checkoutSession->getQuote();

        return [
            'customCheckout' => [
                'enabled' => $this->scopeConfig->isSetFlag(
                    'custom/checkout/enabled',
                    \Magento\Store\Model\ScopeInterface::SCOPE_STORE
                ),
                'cartItemCount' => $quote->getItemsCount(),
                'grandTotal' => $quote->getGrandTotal(),
                'currencyCode' => $quote->getQuoteCurrencyCode(),
            ],
        ];
    }
}
```

Register in `di.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Checkout\Model\CompositeConfigProvider">
        <arguments>
            <argument name="configProviders" xsi:type="array">
                <item name="custom_checkout_config_provider" xsi:type="object">
                    Vendor\CustomCheckout\Model\CustomConfigProvider
                </item>
            </argument>
        </arguments>
    </type>
</config>
```

### 5. Checkout Agreements (Terms & Conditions)

The module includes a system for displaying and validating checkout agreements:

```php
namespace Magento\CheckoutAgreements\Model;

use Magento\CheckoutAgreements\Api\Data\AgreementInterface;

interface CheckoutAgreementsRepositoryInterface
{
    /**
     * Get list of active checkout agreements for store
     *
     * @param int|null $storeId
     * @return AgreementInterface[]
     */
    public function getList($storeId = null);
}
```

Frontend validation via UI component:

```javascript
define([
    'uiComponent',
    'Magento_Customer/js/model/customer',
    'Magento_Checkout/js/model/agreement-validator'
], function (Component, customer, agreementValidator) {
    'use strict';

    return Component.extend({
        defaults: {
            template: 'Magento_CheckoutAgreements/checkout/agreement'
        },

        /**
         * Validate agreements before place order
         */
        validate: function () {
            return agreementValidator.validate();
        }
    });
});
```

### 6. Address Validation

Address validation during checkout is handled by `Magento\Quote\Model\QuoteAddressValidator`. This class validates that:

- If a `customer_address_id` is provided, it belongs to the current customer
- The referenced customer exists (for logged-in customer quotes)
- Address ownership is correct (a guest cannot reference saved addresses)

```php
namespace Magento\Quote\Model;

use Magento\Customer\Api\AddressRepositoryInterface;
use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Model\Session;
use Magento\Quote\Api\Data\AddressInterface;
use Magento\Quote\Api\Data\CartInterface;

class QuoteAddressValidator
{
    public function __construct(
        AddressRepositoryInterface $addressRepository,
        CustomerRepositoryInterface $customerRepository,
        Session $customerSession
    ) {
        $this->addressRepository = $addressRepository;
        $this->customerRepository = $customerRepository;
        $this->customerSession = $customerSession;
    }

    /**
     * Validates the fields in a specified address data object.
     *
     * @param AddressInterface $addressData
     * @return bool
     */
    public function validate(AddressInterface $addressData): bool
    {
        // Validates customer ID exists and address belongs to customer
        $this->doValidate($addressData, $addressData->getCustomerId());
        return true;
    }

    /**
     * Validate address for a specific cart (added in 2.4.x).
     *
     * @param CartInterface $cart
     * @param AddressInterface $address
     * @return void
     */
    public function validateForCart(CartInterface $cart, AddressInterface $address): void
    {
        // Validates customer/address ownership against the cart's customer
        $this->doValidate($address, $cart->getCustomerIsGuest() ? null : $cart->getCustomer()->getId());
    }
}
```

> **Key point**: `QuoteAddressValidator` does **not** validate field contents (firstname, city, postcode). It validates **ownership** — ensuring the address ID belongs to the logged-in customer. Field-level validation is handled separately by frontend validators and API input processing.

### 7. Minicart Integration

The module provides minicart functionality tightly integrated with checkout:

```xml
<!-- Magento_Checkout::page/html/header.phtml -->
<div class="minicart-wrapper" data-bind="scope: 'minicart_content'">
    <!-- ko template: getTemplate() --><!-- /ko -->
</div>

<script type="text/x-magento-init">
{
    "[data-block='minicart']": {
        "Magento_Checkout/js/view/minicart": {
            "shoppingCartUrl": "<?= $escaper->escapeUrl($block->getShoppingCartUrl()) ?>",
            "maxItemsToDisplay": <?= (int)$block->getMaxItemsToDisplay() ?>
        }
    }
}
</script>
```

### 8. Cart Price Rules During Checkout

Checkout integrates with `Magento_SalesRule` for applying discounts. Discount processing is handled by the totals collector `Magento\SalesRule\Model\Quote\Discount`, which extends `Magento\Quote\Model\Quote\Address\Total\AbstractTotal`.

The collector is registered in `Magento_SalesRule/etc/sales.xml` and runs automatically as part of `TotalsCollector::collect()`:

```php
namespace Magento\SalesRule\Model\Quote;

use Magento\Quote\Model\Quote\Address\Total\AbstractTotal;

class Discount extends AbstractTotal
{
    /**
     * Collect discount totals for each quote item.
     * Called automatically by TotalsCollector during quote save/totals recalculation.
     */
    public function collect(
        \Magento\Quote\Model\Quote $quote,
        \Magento\Quote\Api\Data\ShippingAssignmentInterface $shippingAssignment,
        \Magento\Quote\Model\Quote\Address\Total $total
    ) {
        parent::collect($quote, $shippingAssignment, $total);
        // Applies SalesRule rules to each item via Magento\SalesRule\Model\Validator
        // Sets discount amounts on items and address totals
    }
}
```

> **Key point**: There is no standalone "CartDiscountProcessor" class. Discounts are applied through the totals collector pipeline — `TotalsCollector` iterates all registered collectors including `Discount`, which uses `Magento\SalesRule\Model\Validator` internally to evaluate and apply rules.

## Checkout Flow Summary

1. **Customer lands on `/checkout`** - Session initializes, quote loads
2. **Shipping step renders** - Customer enters address, selects shipping method
3. **Shipping information saved** - API call to `ShippingInformationManagementInterface::saveAddressInformation()`
4. **Payment step renders** - Billing address, payment methods load via `CompositeConfigProvider`
5. **Place order triggered** - `PaymentInformationManagementInterface::savePaymentInformationAndPlaceOrder()`
6. **Order created** - Quote converts to order, inventory reserved, emails sent
7. **Success page** - Customer redirected to `/checkout/onepage/success`

## Configuration Paths

### System Configuration

- **`checkout/options/guest_checkout`** - Enable guest checkout (0/1)
- **`checkout/options/onepage_checkout_enabled`** - Enable onepage checkout (0/1)
- **`checkout/options/enable_agreements`** - Enable terms and conditions (0/1)
- **`checkout/cart/redirect_to_cart`** - Redirect to cart after add-to-cart (0/1)
- **`checkout/cart_link/use_qty`** - Display item quantities in minicart (0/1)
- **`checkout/sidebar/display`** - Display shopping cart sidebar (0/1)
- **`checkout/sidebar/count`** - Maximum items to display in minicart

### Layout Handles

- **`checkout_index_index`** - Main checkout page
- **`checkout_cart_index`** - Shopping cart page
- **`checkout_onepage_success`** - Order success page
- **`checkout_cart_configure`** - Cart item configuration page

## API Endpoints

### REST API

```
POST   /V1/guest-carts/:cartId/shipping-information
POST   /V1/carts/mine/shipping-information
POST   /V1/guest-carts/:cartId/payment-information
POST   /V1/carts/mine/payment-information
GET    /V1/carts/mine/totals
GET    /V1/guest-carts/:cartId/totals
POST   /V1/carts/mine/estimate-shipping-methods
```

### GraphQL

```graphql
mutation SetShippingAddressesOnCart($input: SetShippingAddressesOnCartInput!) {
  setShippingAddressesOnCart(input: $input) {
    cart {
      shipping_addresses {
        firstname
        lastname
        street
        city
        region { code }
        postcode
        country { code }
      }
    }
  }
}

mutation SetPaymentMethodOnCart($input: SetPaymentMethodOnCartInput!) {
  setPaymentMethodOnCart(input: $input) {
    cart {
      selected_payment_method {
        code
        title
      }
    }
  }
}

mutation PlaceOrder($input: PlaceOrderInput!) {
  placeOrder(input: $input) {
    order {
      order_number
    }
  }
}
```

## Extension Points

### 1. Custom Checkout Step

Add custom step via layout XML:

```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceBlock name="checkout.root">
            <arguments>
                <argument name="jsLayout" xsi:type="array">
                    <item name="components" xsi:type="array">
                        <item name="checkout" xsi:type="array">
                            <item name="children" xsi:type="array">
                                <item name="steps" xsi:type="array">
                                    <item name="children" xsi:type="array">
                                        <item name="custom-step" xsi:type="array">
                                            <item name="component" xsi:type="string">
                                                Vendor_Module/js/view/custom-step
                                            </item>
                                            <item name="sortOrder" xsi:type="string">2</item>
                                            <item name="children" xsi:type="array">
                                                <!-- Step content -->
                                            </item>
                                        </item>
                                    </item>
                                </item>
                            </item>
                        </item>
                    </item>
                </argument>
            </arguments>
        </referenceBlock>
    </body>
</page>
```

### 2. Payment Method Integration

Implement `\Magento\Payment\Model\MethodInterface`:

```php
namespace Vendor\Payment\Model;

use Magento\Payment\Model\Method\AbstractMethod;

class CustomPayment extends AbstractMethod
{
    protected $_code = 'custom_payment';
    protected $_isGateway = true;
    protected $_canAuthorize = true;
    protected $_canCapture = true;
    protected $_canRefund = true;

    /**
     * Authorize payment
     *
     * @param \Magento\Payment\Model\InfoInterface $payment
     * @param float $amount
     * @return $this
     */
    public function authorize(
        \Magento\Payment\Model\InfoInterface $payment,
        $amount
    ): self {
        // Implement authorization logic
        return $this;
    }
}
```

### 3. Custom Totals

Add custom total collector:

```php
namespace Vendor\Module\Model\Quote\Address\Total;

use Magento\Quote\Model\Quote\Address\Total\AbstractTotal;

class CustomFee extends AbstractTotal
{
    protected $_code = 'custom_fee';

    /**
     * Collect custom fee total
     */
    public function collect(
        \Magento\Quote\Model\Quote $quote,
        \Magento\Quote\Api\Data\ShippingAssignmentInterface $shippingAssignment,
        \Magento\Quote\Model\Quote\Address\Total $total
    ): self {
        parent::collect($quote, $shippingAssignment, $total);

        $fee = 5.00; // Calculate fee
        $total->setCustomFee($fee);
        $total->setGrandTotal($total->getGrandTotal() + $fee);
        $total->setBaseGrandTotal($total->getBaseGrandTotal() + $fee);

        return $this;
    }
}
```

## Common Customizations

### 1. Disable Specific Payment Method Based on Cart

```php
namespace Vendor\Module\Plugin;

use Magento\Payment\Model\MethodInterface;

class PaymentMethodAvailability
{
    public function __construct(
        private readonly \Magento\Checkout\Model\Session $checkoutSession
    ) {}

    /**
     * Disable payment method based on cart total
     *
     * @param MethodInterface $subject
     * @param bool $result
     * @return bool
     */
    public function afterIsAvailable(
        MethodInterface $subject,
        bool $result
    ): bool {
        if ($subject->getCode() === 'cashondelivery') {
            $quote = $this->checkoutSession->getQuote();
            if ($quote->getGrandTotal() > 1000) {
                return false;
            }
        }
        return $result;
    }
}
```

### 2. Add Custom Validation to Checkout

```php
namespace Vendor\Module\Model;

use Magento\Quote\Api\Data\CartInterface;
use Magento\Framework\Exception\LocalizedException;

class CheckoutValidator
{
    /**
     * Validate cart before checkout
     *
     * @param CartInterface $quote
     * @throws LocalizedException
     */
    public function validate(CartInterface $quote): void
    {
        // Custom validation logic
        $hasVirtualProduct = false;
        foreach ($quote->getAllItems() as $item) {
            if ($item->getIsVirtual()) {
                $hasVirtualProduct = true;
                break;
            }
        }

        if ($hasVirtualProduct && $quote->getItemsQty() > 1) {
            throw new LocalizedException(
                __('Virtual products cannot be purchased with physical products.')
            );
        }
    }
}
```

## Testing Checkout

### Unit Test Example

```php
namespace Vendor\Module\Test\Unit\Model;

use PHPUnit\Framework\TestCase;
use Magento\Checkout\Model\Session;

class CheckoutProcessorTest extends TestCase
{
    private Session $checkoutSession;
    private \Vendor\Module\Model\CheckoutProcessor $processor;

    protected function setUp(): void
    {
        $this->checkoutSession = $this->createMock(Session::class);
        $this->processor = new \Vendor\Module\Model\CheckoutProcessor(
            $this->checkoutSession
        );
    }

    public function testProcessCheckoutWithValidQuote(): void
    {
        $quote = $this->createMock(\Magento\Quote\Model\Quote::class);
        $quote->method('getGrandTotal')->willReturn(100.00);

        $this->checkoutSession
            ->method('getQuote')
            ->willReturn($quote);

        $result = $this->processor->process();
        $this->assertTrue($result);
    }
}
```

### Integration Test Example

```php
namespace Vendor\Module\Test\Integration\Model;

use Magento\TestFramework\Helper\Bootstrap;

class CheckoutFlowTest extends \PHPUnit\Framework\TestCase
{
    /**
     * @magentoDataFixture Magento/Checkout/_files/quote_with_items_saved.php
     */
    public function testGuestCheckoutFlow(): void
    {
        $objectManager = Bootstrap::getObjectManager();
        $quote = $objectManager->create(\Magento\Quote\Model\Quote::class);
        $quote->load('test_order_1', 'reserved_order_id');

        $this->assertGreaterThan(0, $quote->getId());
        $this->assertGreaterThan(0, $quote->getGrandTotal());
    }
}
```

## Dependencies

### Required Modules

- `Magento_Catalog` - Product information
- `Magento_Quote` - Cart/quote management
- `Magento_Customer` - Customer data and authentication
- `Magento_Sales` - Order creation
- `Magento_Payment` - Payment methods
- `Magento_Store` - Multi-store support
- `Magento_Directory` - Country/region data
- `Magento_Ui` - UI components framework
- `Magento_Shipping` - Shipping methods

### Optional Integration

- `Magento_Tax` - Tax calculation
- `Magento_SalesRule` - Cart price rules
- `Magento_GiftMessage` - Gift message support
- `Magento_Captcha` - CAPTCHA on checkout
- `Magento_Multishipping` - Multiple shipping addresses

## Performance Considerations

1. **Full Page Cache** - Checkout pages are excluded from FPC by default
2. **Session Storage** - Use Redis for session storage in production
3. **Quote Cleanup** - Configure `checkout/cart/delete_quote_after` to clean up old quotes
4. **Address Optimization** - Cache country/region data to reduce DB queries
5. **Payment Methods** - Lazy load payment method JavaScript

## Security Considerations

1. **Form Keys** - All checkout forms must include and validate form keys
2. **Quote Validation** - Validate quote ownership before operations
3. **Price Tampering** - Never trust client-side prices; always recalculate server-side
4. **Payment Info** - Never log or expose sensitive payment data
5. **Guest Checkout** - Implement rate limiting on guest checkout to prevent abuse

---

## Assumptions

- **Target Platform**: Adobe Commerce & Magento Open Source 2.4.7+
- **PHP Version**: 8.1, 8.2, 8.3
- **Frontend**: Luma theme or Luma-based custom theme
- **Area**: Frontend (with API service contracts for headless)
- **Scope**: Store-level configuration for most checkout settings

## Why This Approach

The Magento_Checkout module uses a component-based SPA architecture to provide a seamless, single-page checkout experience while maintaining server-side validation and processing. This approach:

- **Separates concerns** - UI components handle presentation, service contracts handle business logic
- **Enables extensibility** - Plugins, observers, and UI component layout allow customization without core modifications
- **Supports multiple frontend implementations** - Same service contracts work for Luma, PWA Studio, and headless
- **Maintains upgrade path** - Following service contracts and avoiding preferences ensures compatibility

## Security Impact

- **CSRF Protection**: All checkout forms include form key validation via `\Magento\Framework\Data\Form\FormKey`
- **Quote Ownership**: Service contracts validate customer/guest owns the quote before operations
- **Payment Security**: PCI compliance maintained by not storing card data; payment methods handle tokenization
- **XSS Prevention**: All checkout output escaped via `$escaper->escapeHtml()` and Knockout.js automatic escaping
- **Rate Limiting**: Consider implementing rate limiting on checkout APIs to prevent abuse

## Performance Impact

- **No FPC**: Checkout pages excluded from Full Page Cache due to dynamic content
- **Session I/O**: Heavy session usage; Redis session storage critical for production
- **DB Queries**: Address validation and totals calculation generate multiple queries; use query profiling
- **JavaScript Bundle**: Knockout.js and checkout components add ~500KB; consider code splitting
- **Core Web Vitals**: Checkout often has lower scores due to dynamic forms; optimize with lazy loading

## Backward Compatibility

- **Service Contracts**: `PaymentInformationManagementInterface`, `ShippingInformationManagementInterface` are stable APIs
- **UI Components**: Adding custom components via layout XML is BC-safe
- **Plugins**: Prefer plugins on service contracts over model classes
- **Breaking Changes**: Magento 2.4.x maintains BC for public APIs; check deprecation notices in release notes

## Tests to Add

1. **Unit Tests**: Mock dependencies, test business logic in isolation
2. **Integration Tests**: Use `@magentoDataFixture` for quote/customer fixtures
3. **API Tests**: Test REST/GraphQL endpoints with WebAPI framework
4. **MFTF Tests**: End-to-end checkout flow tests
5. **Performance Tests**: Load test checkout with Apache JMeter

## Documentation to Update

- **README.md**: Module overview (this file)
- **CHANGELOG.md**: Version history and breaking changes
- **Admin Guide**: System > Configuration > Sales > Checkout settings
- **Developer Guide**: Custom checkout step tutorial, payment method integration
- **API Documentation**: REST/GraphQL schema for checkout endpoints

## Related Guides

- [GraphQL Resolver Patterns in Magento 2](../../guides/explanations/graphql-resolver-patterns.md)
- [Full Page Cache Strategy for High-Performance Magento](../../guides/how-to/full-page-cache-strategy.md)
- [Layout XML Deep Dive: Mastering Magento's View Layer](../../guides/how-to/layout-xml-deep-dive.md)
- [Security Checklist for Custom Modules](../../guides/how-to/security-checklist.md)
- [Comprehensive Testing Strategies for Magento 2](../../guides/how-to/testing-strategies.md)
- [Upgrade Guide: Adobe Commerce 2.4.6 to 2.4.7/2.4.8](../../guides/references/upgrade-guide-247-248.md)
- [Custom Payment Method Development: Building Secure Payment Integrations](../../guides/tutorials/custom-payment-method.md)
- [Custom Shipping Method Development: Building Flexible Shipping Solutions](../../guides/tutorials/custom-shipping-method.md)
