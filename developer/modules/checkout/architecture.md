---
title: "Magento_Checkout Architecture"
module: "Magento_Checkout"
doc_type: "architecture"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Checkout Architecture

## Architectural Overview

The `Magento_Checkout` module implements a **multi-layered, component-based architecture** that separates business logic, data persistence, and presentation concerns. The architecture follows SOLID principles and leverages Magento's service contract pattern to provide a stable API for both frontend checkout UI and headless/API-based implementations.

**Core Architectural Patterns:**

1. **Service Layer** - Service contracts define stable APIs for checkout operations
2. **UI Component Architecture** - Knockout.js components render dynamic checkout UI
3. **Session-Based State** - Checkout session maintains quote and customer state across requests
4. **Event-Driven Extension** - Observers and plugins enable customization without core modifications
5. **Multi-Step Workflow** - Step-based architecture with validation at each stage

---

## Layer Architecture

### 1. Service Contract Layer (API)

The service contract layer defines the public API for checkout operations. These interfaces provide BC-guaranteed contracts for both REST/GraphQL APIs and internal Magento modules.

#### Primary Service Contracts

**`\Magento\Checkout\Api\PaymentInformationManagementInterface`**

Handles payment information and order placement for authenticated customers:

```php
namespace Magento\Checkout\Api;

use Magento\Quote\Api\Data\PaymentInterface;
use Magento\Quote\Api\Data\AddressInterface;

interface PaymentInformationManagementInterface
{
    /**
     * Actual 2.4.7 signatures — no PHP return types or typed params
     */

    /**
     * @param int $cartId
     * @param \Magento\Quote\Api\Data\PaymentInterface $paymentMethod
     * @param \Magento\Quote\Api\Data\AddressInterface|null $billingAddress
     * @return int Order ID
     */
    public function savePaymentInformationAndPlaceOrder(
        $cartId,
        \Magento\Quote\Api\Data\PaymentInterface $paymentMethod,
        \Magento\Quote\Api\Data\AddressInterface $billingAddress = null
    );

    /**
     * @param int $cartId
     * @param \Magento\Quote\Api\Data\PaymentInterface $paymentMethod
     * @param \Magento\Quote\Api\Data\AddressInterface|null $billingAddress
     * @return int
     */
    public function savePaymentInformation(
        $cartId,
        \Magento\Quote\Api\Data\PaymentInterface $paymentMethod,
        \Magento\Quote\Api\Data\AddressInterface $billingAddress = null
    );

    /**
     * @param int $cartId
     * @return \Magento\Checkout\Api\Data\PaymentDetailsInterface
     */
    public function getPaymentInformation($cartId);
}
```

**Implementation:**

```php
namespace Magento\Checkout\Model;

use Magento\Checkout\Api\PaymentInformationManagementInterface;
use Magento\Quote\Api\Data\PaymentInterface;
use Magento\Quote\Api\Data\AddressInterface;
use Magento\Framework\Exception\CouldNotSaveException;
use Magento\Quote\Api\CartManagementInterface;
use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\BillingAddressManagementInterface;
use Magento\Quote\Api\PaymentMethodManagementInterface;

class PaymentInformationManagement implements PaymentInformationManagementInterface
{
    public function __construct(
        private readonly BillingAddressManagementInterface $billingAddressManagement,
        private readonly PaymentMethodManagementInterface $paymentMethodManagement,
        private readonly CartManagementInterface $cartManagement,
        private readonly CartRepositoryInterface $cartRepository,
        private readonly PaymentDetailsFactory $paymentDetailsFactory,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * {@inheritdoc}
     */
    public function savePaymentInformationAndPlaceOrder(
        $cartId,
        \Magento\Quote\Api\Data\PaymentInterface $paymentMethod,
        \Magento\Quote\Api\Data\AddressInterface $billingAddress = null
    ) {
        try {
            // Save payment information first
            $this->savePaymentInformation($cartId, $paymentMethod, $billingAddress);

            // Place order
            $orderId = $this->cartManagement->placeOrder($cartId);

            if (!$orderId) {
                throw new CouldNotSaveException(
                    __('A server error stopped your order from being placed. Please try again.')
                );
            }

            return $orderId;

        } catch (\Exception $e) {
            $this->logger->critical($e);
            throw new CouldNotSaveException(
                __('A server error stopped your order from being placed. Please try again.'),
                $e
            );
        }
    }

    /**
     * {@inheritdoc}
     */
    public function savePaymentInformation(
        $cartId,
        \Magento\Quote\Api\Data\PaymentInterface $paymentMethod,
        \Magento\Quote\Api\Data\AddressInterface $billingAddress = null
    ) {
        // Set billing address if provided
        if ($billingAddress) {
            $billingAddress->setCustomerAddressId(null);
            $this->billingAddressManagement->assign($cartId, $billingAddress);
        }

        // Set payment method
        $this->paymentMethodManagement->set($cartId, $paymentMethod);

        return $cartId;
    }

    /**
     * {@inheritdoc}
     */
    public function getPaymentInformation($cartId)
    {
        $quote = $this->cartRepository->getActive($cartId);

        $paymentDetails = $this->paymentDetailsFactory->create();
        $paymentDetails->setPaymentMethods($this->paymentMethodManagement->getList($cartId));
        $paymentDetails->setTotals($quote->getTotals());

        return $paymentDetails;
    }
}
```

**`\Magento\Checkout\Api\ShippingInformationManagementInterface`**

Manages shipping address and method selection:

```php
namespace Magento\Checkout\Api;

use Magento\Checkout\Api\Data\ShippingInformationInterface;

interface ShippingInformationManagementInterface
{
    /**
     * Save shipping information and calculate totals
     *
     * @param int $cartId
     * @param ShippingInformationInterface $addressInformation
     * @return \Magento\Checkout\Api\Data\PaymentDetailsInterface
     * @throws \Magento\Framework\Exception\InputException
     * @throws \Magento\Framework\Exception\StateException
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function saveAddressInformation(
        int $cartId,
        ShippingInformationInterface $addressInformation
    ): Data\PaymentDetailsInterface;
}
```

**Implementation with Validation:**

```php
namespace Magento\Checkout\Model;

use Magento\Checkout\Api\ShippingInformationManagementInterface;
use Magento\Checkout\Api\Data\ShippingInformationInterface;
use Magento\Framework\Exception\InputException;
use Magento\Framework\Exception\StateException;
use Magento\Framework\Exception\NoSuchEntityException;

class ShippingInformationManagement implements ShippingInformationManagementInterface
{
    public function __construct(
        private readonly \Magento\Quote\Api\CartRepositoryInterface $quoteRepository,
        private readonly \Magento\Customer\Api\AddressRepositoryInterface $addressRepository,
        private readonly PaymentDetailsFactory $paymentDetailsFactory,
        private readonly \Magento\Quote\Model\QuoteAddressValidator $addressValidator,
        private readonly \Psr\Log\LoggerInterface $logger,
        private readonly \Magento\Quote\Api\Data\CartExtensionFactory $cartExtensionFactory,
        private readonly \Magento\Quote\Model\Quote\TotalsCollector $totalsCollector
    ) {}

    /**
     * {@inheritdoc}
     */
    public function saveAddressInformation(
        int $cartId,
        ShippingInformationInterface $addressInformation
    ): \Magento\Checkout\Api\Data\PaymentDetailsInterface {

        $quote = $this->quoteRepository->getActive($cartId);

        if ($quote->isVirtual()) {
            throw new NoSuchEntityException(
                __('Cart contains virtual product(s) only. Shipping address is not applicable.')
            );
        }

        $shippingAddress = $addressInformation->getShippingAddress();
        $billingAddress = $addressInformation->getBillingAddress();

        // Validate shipping address
        $this->validateAddress($shippingAddress);

        // Set shipping address
        $shippingAddress->setCustomerAddressId(null);
        $quote->setShippingAddress($shippingAddress);

        // Set shipping method
        $carrierCode = $addressInformation->getShippingCarrierCode();
        $methodCode = $addressInformation->getShippingMethodCode();
        $quote->getShippingAddress()->setShippingMethod($carrierCode . '_' . $methodCode);

        // Set billing address if provided
        if ($billingAddress) {
            $billingAddress->setCustomerAddressId(null);
            $quote->setBillingAddress($billingAddress);
        }

        // Collect totals
        $quote->setTotalsCollectedFlag(false);
        $quote->collectTotals();

        // Save quote
        $this->quoteRepository->save($quote);

        // Return payment details
        $paymentDetails = $this->paymentDetailsFactory->create();
        $paymentDetails->setPaymentMethods($this->getPaymentMethods($quote));
        $paymentDetails->setTotals($quote->getTotals());

        return $paymentDetails;
    }

    /**
     * Validate address
     *
     * @param \Magento\Quote\Api\Data\AddressInterface $address
     * @return void
     * @throws InputException
     */
    private function validateAddress(\Magento\Quote\Api\Data\AddressInterface $address): void
    {
        $errors = $this->addressValidator->validate($address);
        if ($errors) {
            throw new InputException(
                __('Unable to save address. Please check input data. %1', implode(' ', $errors))
            );
        }
    }

    /**
     * Get available payment methods for quote
     *
     * @param \Magento\Quote\Model\Quote $quote
     * @return \Magento\Quote\Api\Data\PaymentMethodInterface[]
     */
    private function getPaymentMethods(\Magento\Quote\Model\Quote $quote): array
    {
        $paymentMethods = [];
        $methods = $quote->getPayment()->getAvailableMethods();

        foreach ($methods as $method) {
            $paymentMethods[] = $method;
        }

        return $paymentMethods;
    }
}
```

#### Data Transfer Objects (DTOs)

**`\Magento\Checkout\Api\Data\ShippingInformationInterface`**

```php
namespace Magento\Checkout\Api\Data;

use Magento\Quote\Api\Data\AddressInterface;

interface ShippingInformationInterface
{
    /**
     * Actual 2.4.7 signatures — no PHP return types or typed params
     */

    /** @return \Magento\Quote\Api\Data\AddressInterface */
    public function getShippingAddress();

    /** @return $this */
    public function setShippingAddress(\Magento\Quote\Api\Data\AddressInterface $address);

    /** @return \Magento\Quote\Api\Data\AddressInterface|null */
    public function getBillingAddress();

    /** @return $this */
    public function setBillingAddress(\Magento\Quote\Api\Data\AddressInterface $address);

    /** @return string */
    public function getShippingCarrierCode();

    /** @return $this */
    public function setShippingCarrierCode($code);

    /** @return string */
    public function getShippingMethodCode();

    /** @return $this */
    public function setShippingMethodCode($code);
}
```

**Implementation (Model):**

```php
namespace Magento\Checkout\Model;

use Magento\Checkout\Api\Data\ShippingInformationInterface;
use Magento\Quote\Api\Data\AddressInterface;
use Magento\Framework\Model\AbstractExtensibleModel;

class ShippingInformation extends AbstractExtensibleModel implements ShippingInformationInterface
{
    private const SHIPPING_ADDRESS = 'shipping_address';
    private const BILLING_ADDRESS = 'billing_address';
    private const SHIPPING_CARRIER_CODE = 'shipping_carrier_code';
    private const SHIPPING_METHOD_CODE = 'shipping_method_code';

    /**
     * {@inheritdoc}
     */
    public function getShippingAddress()
    {
        return $this->getData(self::SHIPPING_ADDRESS);
    }

    /** {@inheritdoc} */
    public function setShippingAddress(\Magento\Quote\Api\Data\AddressInterface $address)
    {
        return $this->setData(self::SHIPPING_ADDRESS, $address);
    }

    /** {@inheritdoc} */
    public function getBillingAddress()
    {
        return $this->getData(self::BILLING_ADDRESS);
    }

    /** {@inheritdoc} */
    public function setBillingAddress(\Magento\Quote\Api\Data\AddressInterface $address)
    {
        return $this->setData(self::BILLING_ADDRESS, $address);
    }

    /** {@inheritdoc} */
    public function getShippingCarrierCode()
    {
        return $this->getData(self::SHIPPING_CARRIER_CODE) ?? '';
    }

    /** {@inheritdoc} */
    public function setShippingCarrierCode($code)
    {
        return $this->setData(self::SHIPPING_CARRIER_CODE, $code);
    }

    /** {@inheritdoc} */
    public function getShippingMethodCode()
    {
        return $this->getData(self::SHIPPING_METHOD_CODE) ?? '';
    }

    /** {@inheritdoc} */
    public function setShippingMethodCode($code)
    {
        return $this->setData(self::SHIPPING_METHOD_CODE, $code);
    }
}
```

---

### 2. Business Logic Layer (Model)

The model layer implements core checkout business logic, including quote management, session handling, and checkout type determination.

#### Checkout Session

**`\Magento\Checkout\Model\Session`**

The checkout session maintains state across requests and provides access to the active quote:

```php
namespace Magento\Checkout\Model;

use Magento\Framework\Session\SessionManager;
use Magento\Quote\Api\Data\CartInterface;
use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Exception\NoSuchEntityException;

class Session extends SessionManager
{
    private const QUOTE_ID_KEY = 'quote_id';
    private const LAST_ORDER_ID_KEY = 'last_order_id';
    private const LAST_SUCCESS_QUOTE_ID_KEY = 'last_success_quote_id';
    private const LAST_REAL_ORDER_ID_KEY = 'last_real_order_id';

    /**
     * @var CartInterface|null
     */
    protected $_quote;

    /**
     * @var bool
     */
    protected $_loadInactive = false;

    public function __construct(
        private readonly CartRepositoryInterface $quoteRepository,
        private readonly \Magento\Quote\Model\QuoteFactory $quoteFactory,
        private readonly \Magento\Customer\Model\Session $customerSession,
        private readonly \Magento\Store\Model\StoreManagerInterface $storeManager,
        // ... parent constructor parameters
    ) {
        parent::__construct(...);
    }

    /**
     * Get checkout quote instance
     *
     * @return CartInterface
     * @throws LocalizedException
     * @throws NoSuchEntityException
     */
    public function getQuote(): CartInterface
    {
        if ($this->_quote === null) {
            $this->_quote = $this->loadQuote();
        }
        return $this->_quote;
    }

    /**
     * Load quote from repository
     *
     * @return CartInterface
     * @throws LocalizedException
     * @throws NoSuchEntityException
     */
    private function loadQuote(): CartInterface
    {
        $quoteId = $this->getQuoteId();

        if ($quoteId) {
            try {
                $quote = $this->_loadInactive
                    ? $this->quoteRepository->get($quoteId)
                    : $this->quoteRepository->getActive($quoteId);

                // Validate quote belongs to current customer
                if ($this->customerSession->isLoggedIn()) {
                    if ((int)$quote->getCustomerId() !== (int)$this->customerSession->getCustomerId()) {
                        $quote = $this->createNewQuote();
                    }
                }

                return $quote;

            } catch (NoSuchEntityException $e) {
                $this->setQuoteId(null);
            }
        }

        // Create new quote if none exists
        return $this->createNewQuote();
    }

    /**
     * Create new quote instance
     *
     * @return CartInterface
     */
    private function createNewQuote(): CartInterface
    {
        $quote = $this->quoteFactory->create();
        $quote->setStoreId($this->storeManager->getStore()->getId());

        if ($this->customerSession->isLoggedIn()) {
            $quote->setCustomerId($this->customerSession->getCustomerId());
            $quote->setCustomerGroupId($this->customerSession->getCustomerGroupId());
        }

        return $quote;
    }

    /**
     * Get quote ID from session
     *
     * @return int|null
     */
    public function getQuoteId(): ?int
    {
        return $this->getData(self::QUOTE_ID_KEY);
    }

    /**
     * Set quote ID in session
     *
     * @param int|null $quoteId
     * @return $this
     */
    public function setQuoteId(?int $quoteId): self
    {
        $this->setData(self::QUOTE_ID_KEY, $quoteId);
        return $this;
    }

    /**
     * Replace existing quote with new one
     *
     * @param CartInterface $quote
     * @return $this
     */
    public function replaceQuote(CartInterface $quote): self
    {
        $this->_quote = $quote;
        $this->setQuoteId($quote->getId());
        return $this;
    }

    /**
     * Clear checkout session
     *
     * @return $this
     */
    public function clearQuote(): self
    {
        $this->_quote = null;
        $this->setQuoteId(null);
        return $this;
    }

    /**
     * Get last order ID
     *
     * @return int|null
     */
    public function getLastOrderId(): ?int
    {
        return $this->getData(self::LAST_ORDER_ID_KEY);
    }

    /**
     * Set last order ID
     *
     * @param int $orderId
     * @return $this
     */
    public function setLastOrderId(int $orderId): self
    {
        $this->setData(self::LAST_ORDER_ID_KEY, $orderId);
        return $this;
    }

    /**
     * Get last real order ID (increment ID)
     *
     * @return string|null
     */
    public function getLastRealOrderId(): ?string
    {
        return $this->getData(self::LAST_REAL_ORDER_ID_KEY);
    }

    /**
     * Set last real order ID
     *
     * @param string $orderIncrementId
     * @return $this
     */
    public function setLastRealOrderId(string $orderIncrementId): self
    {
        $this->setData(self::LAST_REAL_ORDER_ID_KEY, $orderIncrementId);
        return $this;
    }

    /**
     * Allow loading inactive quotes
     *
     * @param bool $load
     * @return $this
     */
    public function setLoadInactive(bool $load = true): self
    {
        $this->_loadInactive = $load;
        return $this;
    }
}
```

#### Checkout Type Resolver

Determines whether checkout is guest or registered:

```php
namespace Magento\Checkout\Model;

use Magento\Quote\Api\Data\CartInterface;
use Magento\Customer\Model\Session as CustomerSession;

// Note: CheckoutTypeResolver is NOT a real Magento class. Guest checkout
// logic is handled by \Magento\Checkout\Helper\Data::isAllowedGuestCheckout().
// The example below shows the conceptual logic:
class GuestCheckoutHelper
{
    public function __construct(
        private readonly CustomerSession $customerSession,
        private readonly \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig
    ) {}

    /**
     * Check if guest checkout is allowed (conceptual example)
     *
     * @param CartInterface $quote
     * @return bool
     * @see \Magento\Checkout\Helper\Data::isAllowedGuestCheckout()
     */
    public function isGuestCheckoutAllowed(CartInterface $quote): bool
    {
        // Check if customer is logged in
        if ($this->customerSession->isLoggedIn()) {
            return false;
        }

        // Check if guest checkout is enabled globally
        if (!$this->scopeConfig->isSetFlag(
            'checkout/options/guest_checkout',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        )) {
            return false;
        }

        // Check if any product in quote disallows guest checkout
        foreach ($quote->getAllItems() as $item) {
            $product = $item->getProduct();
            if ($product && !$product->getAllowGuestCheckout()) {
                return false;
            }
        }

        return true;
    }

    /**
     * Check if registration is required
     *
     * @param CartInterface $quote
     * @return bool
     */
    public function isRegistrationRequired(CartInterface $quote): bool
    {
        return !$this->isGuestCheckoutAllowed($quote)
            && !$this->customerSession->isLoggedIn();
    }
}
```

---

### 3. UI Component Layer (View)

The UI component layer implements the frontend checkout interface using Knockout.js components and RequireJS modules.

#### Checkout UI Component Structure

The checkout page is structured as nested UI components defined in layout XML:

**`checkout_index_index.xml`**

```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      layout="1column"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <update handle="checkout_cart_item_renderers"/>
    <body>
        <referenceBlock name="content">
            <block name="checkout.root"
                   template="Magento_Checkout::onepage.phtml"
                   class="Magento\Checkout\Block\Onepage"
                   cacheable="false">
                <arguments>
                    <argument name="jsLayout" xsi:type="array">
                        <item name="components" xsi:type="array">
                            <item name="checkout" xsi:type="array">
                                <item name="component" xsi:type="string">
                                    Magento_Checkout/js/view/checkout
                                </item>
                                <item name="config" xsi:type="array">
                                    <item name="template" xsi:type="string">
                                        Magento_Checkout/checkout
                                    </item>
                                </item>
                                <item name="children" xsi:type="array">
                                    <!-- Sidebar component -->
                                    <item name="sidebar" xsi:type="array">
                                        <item name="component" xsi:type="string">
                                            Magento_Checkout/js/view/sidebar
                                        </item>
                                        <item name="children" xsi:type="array">
                                            <item name="summary" xsi:type="array">
                                                <item name="component" xsi:type="string">
                                                    Magento_Checkout/js/view/summary
                                                </item>
                                            </item>
                                        </item>
                                    </item>
                                    <!-- Steps component -->
                                    <item name="steps" xsi:type="array">
                                        <item name="component" xsi:type="string">
                                            uiComponent
                                        </item>
                                        <item name="children" xsi:type="array">
                                            <!-- Shipping step -->
                                            <item name="shipping-step" xsi:type="array">
                                                <item name="component" xsi:type="string">
                                                    Magento_Checkout/js/view/shipping
                                                </item>
                                                <item name="sortOrder" xsi:type="string">1</item>
                                            </item>
                                            <!-- Payment step -->
                                            <item name="billing-step" xsi:type="array">
                                                <item name="component" xsi:type="string">
                                                    Magento_Checkout/js/view/billing
                                                </item>
                                                <item name="sortOrder" xsi:type="string">2</item>
                                            </item>
                                        </item>
                                    </item>
                                </item>
                            </item>
                        </item>
                    </argument>
                </arguments>
            </block>
        </referenceBlock>
    </body>
</page>
```

#### Checkout Step Component

**`Magento_Checkout/js/view/shipping.js`**

```javascript
define([
    'ko',
    'uiComponent',
    'underscore',
    'Magento_Checkout/js/model/step-navigator',
    'Magento_Checkout/js/model/quote',
    'Magento_Checkout/js/action/set-shipping-information',
    'Magento_Checkout/js/model/shipping-service'
], function (
    ko,
    Component,
    _,
    stepNavigator,
    quote,
    setShippingInformationAction,
    shippingService
) {
    'use strict';

    return Component.extend({
        defaults: {
            template: 'Magento_Checkout/shipping',
            shippingFormTemplate: 'Magento_Checkout/shipping-address/form',
            shippingMethodListTemplate: 'Magento_Checkout/shipping-address/shipping-method-list',
            shippingMethodItemTemplate: 'Magento_Checkout/shipping-address/shipping-method-item'
        },

        visible: ko.observable(true),
        errorValidationMessage: ko.observable(false),

        /**
         * Initialize component
         */
        initialize: function () {
            this._super();

            // Register step in step navigator
            stepNavigator.registerStep(
                'shipping',
                null,
                'Shipping',
                this.visible,
                _.bind(this.navigate, this),
                this.sortOrder
            );

            return this;
        },

        /**
         * Navigate to this step
         */
        navigate: function () {
            this.visible(true);
        },

        /**
         * Navigate to next step
         */
        navigateToNextStep: function () {
            if (this.validateShippingInformation()) {
                setShippingInformationAction().done(
                    function () {
                        stepNavigator.next();
                    }
                );
            }
        },

        /**
         * Validate shipping information
         *
         * @returns {Boolean}
         */
        validateShippingInformation: function () {
            var shippingAddress = quote.shippingAddress();
            var shippingMethod = quote.shippingMethod();

            if (!shippingAddress) {
                this.errorValidationMessage('Please specify a shipping address.');
                return false;
            }

            if (!shippingMethod) {
                this.errorValidationMessage('Please specify a shipping method.');
                return false;
            }

            // Validate address fields
            if (!shippingAddress.firstname) {
                this.errorValidationMessage('First name is required.');
                return false;
            }

            if (!shippingAddress.lastname) {
                this.errorValidationMessage('Last name is required.');
                return false;
            }

            if (!shippingAddress.street || !shippingAddress.street[0]) {
                this.errorValidationMessage('Street address is required.');
                return false;
            }

            if (!shippingAddress.city) {
                this.errorValidationMessage('City is required.');
                return false;
            }

            if (!shippingAddress.postcode) {
                this.errorValidationMessage('Postal code is required.');
                return false;
            }

            if (!shippingAddress.countryId) {
                this.errorValidationMessage('Country is required.');
                return false;
            }

            return true;
        },

        /**
         * Get shipping rates
         *
         * @returns {Array}
         */
        getShippingRates: function () {
            return shippingService.getShippingRates();
        }
    });
});
```

#### Configuration Provider

Injects backend data into frontend checkout components:

```php
namespace Magento\Checkout\Model;

use Magento\Checkout\Model\ConfigProviderInterface;
use Magento\Checkout\Model\Session as CheckoutSession;
use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Store\Model\ScopeInterface;

class DefaultConfigProvider implements ConfigProviderInterface
{
    public function __construct(
        private readonly CheckoutSession $checkoutSession,
        private readonly ScopeConfigInterface $scopeConfig,
        private readonly \Magento\Customer\Model\Session $customerSession,
        private readonly \Magento\Quote\Api\PaymentMethodManagementInterface $paymentMethodManagement,
        private readonly \Magento\Checkout\Model\Cart\ImageProvider $imageProvider
    ) {}

    /**
     * {@inheritdoc}
     */
    public function getConfig(): array
    {
        $quote = $this->checkoutSession->getQuote();
        $quoteId = $quote->getId();

        $output = [
            'quoteData' => [
                'entity_id' => $quoteId,
                'store_id' => $quote->getStoreId(),
                'customer_id' => $quote->getCustomerId(),
                'items_count' => $quote->getItemsCount(),
                'items_qty' => $quote->getItemsQty(),
                'grand_total' => $quote->getGrandTotal(),
                'base_grand_total' => $quote->getBaseGrandTotal(),
                'base_currency_code' => $quote->getBaseCurrencyCode(),
                'quote_currency_code' => $quote->getQuoteCurrencyCode(),
            ],
            'quoteItemData' => $this->getQuoteItemData($quote),
            'isCustomerLoggedIn' => $this->customerSession->isLoggedIn(),
            'selectedShippingMethod' => $this->getSelectedShippingMethod($quote),
            'storeCode' => $quote->getStore()->getCode(),
            'totalsData' => $quote->getTotals(),
            'paymentMethods' => $this->getPaymentMethods($quoteId),
            'checkoutUrl' => $this->scopeConfig->getValue(
                'checkout/options/onepage_checkout_enabled',
                ScopeInterface::SCOPE_STORE
            ) ? '/checkout' : '/checkout/multishipping',
        ];

        return $output;
    }

    /**
     * Get quote item data with images
     *
     * @param \Magento\Quote\Model\Quote $quote
     * @return array
     */
    private function getQuoteItemData(\Magento\Quote\Model\Quote $quote): array
    {
        $items = [];
        foreach ($quote->getAllVisibleItems() as $item) {
            $items[] = [
                'item_id' => $item->getId(),
                'name' => $item->getName(),
                'qty' => $item->getQty(),
                'price' => $item->getPrice(),
                'product_type' => $item->getProductType(),
                'thumbnail' => $this->imageProvider->getImage($item)->getImageUrl(),
            ];
        }
        return $items;
    }

    /**
     * Get selected shipping method
     *
     * @param \Magento\Quote\Model\Quote $quote
     * @return string|null
     */
    private function getSelectedShippingMethod(\Magento\Quote\Model\Quote $quote): ?string
    {
        $shippingMethod = $quote->getShippingAddress()->getShippingMethod();
        return $shippingMethod ?: null;
    }

    /**
     * Get available payment methods
     *
     * @param int $quoteId
     * @return array
     */
    private function getPaymentMethods(int $quoteId): array
    {
        $paymentMethods = [];
        $methods = $this->paymentMethodManagement->getList($quoteId);

        foreach ($methods as $method) {
            $paymentMethods[] = [
                'code' => $method->getCode(),
                'title' => $method->getTitle(),
            ];
        }

        return $paymentMethods;
    }
}
```

---

### 4. Data Persistence Layer

#### Quote Address Management

Shipping and billing addresses are stored in `quote_address` table:

```php
namespace Magento\Quote\Model\ResourceModel\Quote;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class Address extends AbstractDb
{
    /**
     * Initialize resource model
     *
     * @return void
     */
    protected function _construct(): void
    {
        $this->_init('quote_address', 'address_id');
    }

    /**
     * Save address with items
     *
     * @param \Magento\Quote\Model\Quote\Address $address
     * @return $this
     */
    protected function _afterSave(\Magento\Framework\Model\AbstractModel $address): self
    {
        parent::_afterSave($address);

        // Save address items
        foreach ($address->getAllItems() as $item) {
            $item->setParentId($address->getId());
            $item->save();
        }

        // Save shipping rates
        foreach ($address->getAllShippingRates() as $rate) {
            $rate->setAddressId($address->getId());
            $rate->save();
        }

        return $this;
    }
}
```

---

## Component Communication Patterns

### 1. Event Bus Pattern

Checkout components communicate via Knockout.js observables and custom events:

```javascript
define([
    'ko',
    'Magento_Checkout/js/model/quote'
], function (ko, quote) {
    'use strict';

    // Subscribe to shipping address changes
    quote.shippingAddress.subscribe(function (newAddress) {
        console.log('Shipping address changed:', newAddress);
        // Trigger recalculation of shipping methods
    });

    // Subscribe to totals changes
    quote.totals.subscribe(function (newTotals) {
        console.log('Totals updated:', newTotals);
    });
});
```

### 2. Action Pattern

Actions encapsulate API calls and state updates:

```javascript
define([
    'Magento_Checkout/js/model/quote',
    'Magento_Checkout/js/model/url-builder',
    'mage/storage',
    'Magento_Checkout/js/model/error-processor',
    'Magento_Customer/js/model/customer',
    'Magento_Checkout/js/model/full-screen-loader'
], function (
    quote,
    urlBuilder,
    storage,
    errorProcessor,
    customer,
    fullScreenLoader
) {
    'use strict';

    return function (messageContainer) {
        var serviceUrl,
            payload;

        payload = {
            addressInformation: {
                shipping_address: quote.shippingAddress(),
                billing_address: quote.billingAddress(),
                shipping_method_code: quote.shippingMethod().method_code,
                shipping_carrier_code: quote.shippingMethod().carrier_code
            }
        };

        if (customer.isLoggedIn()) {
            serviceUrl = urlBuilder.createUrl('/carts/mine/shipping-information', {});
        } else {
            serviceUrl = urlBuilder.createUrl('/guest-carts/:cartId/shipping-information', {
                cartId: quote.getQuoteId()
            });
        }

        fullScreenLoader.startLoader();

        return storage.post(
            serviceUrl,
            JSON.stringify(payload)
        ).done(function (response) {
            quote.setTotals(response.totals);
            quote.setPaymentMethods(response.payment_methods);
            fullScreenLoader.stopLoader();
        }).fail(function (response) {
            errorProcessor.process(response, messageContainer);
            fullScreenLoader.stopLoader();
        });
    };
});
```

---

## Assumptions

- **Target Platform**: Adobe Commerce & Magento Open Source 2.4.7+
- **PHP Version**: 8.1, 8.2, 8.3
- **Frontend**: Luma theme with Knockout.js UI components
- **Session Storage**: Redis or database (production should use Redis)
- **Areas**: Frontend (checkout UI), webapi_rest (API), webapi_graphql (GraphQL)

## Why This Architecture

The multi-layered architecture with service contracts provides:

- **API Stability**: Service contracts guarantee BC across versions
- **Separation of Concerns**: Business logic, data persistence, and UI are decoupled
- **Testability**: Each layer can be unit tested in isolation
- **Extensibility**: Plugins, observers, and UI component layout allow customization
- **Headless Support**: Same service contracts work for Luma, PWA Studio, and custom frontends

## Security Impact

- **Quote Validation**: All service contracts validate quote ownership before operations
- **CSRF Protection**: Form keys required for all POST requests
- **Input Validation**: DTOs validated via `\Magento\Framework\Validator\Factory`
- **XSS Prevention**: Knockout.js auto-escapes bindings; manual escaping via `$escaper`
- **Session Hijacking**: Session regeneration on customer login

## Performance Impact

- **No FPC**: Checkout pages excluded from Full Page Cache
- **Session I/O**: Heavy session usage; Redis critical for production
- **API Calls**: Each step makes API call to save data and calculate totals
- **JavaScript Bundle**: ~500KB for checkout components; consider RequireJS optimization
- **Database Queries**: Quote totals collection triggers multiple queries; use query profiling

## Backward Compatibility

- **Service Contracts**: All interfaces in `Api/` namespace are BC-guaranteed
- **UI Components**: Layout XML modifications are BC-safe
- **Plugins**: Prefer plugins on service contracts over model classes
- **Extension Attributes**: Use extension attributes for adding data to DTOs

## Tests to Add

1. **Unit Tests**: Mock dependencies, test service contract implementations
2. **Integration Tests**: Test quote manipulation with fixtures
3. **API Tests**: Test REST/GraphQL endpoints with WebAPI framework
4. **JavaScript Tests**: Jasmine tests for UI components
5. **MFTF Tests**: End-to-end checkout flow

## Documentation to Update

- **Architecture Diagram**: Component interaction flow
- **Service Contract Documentation**: Method signatures, return types, exceptions
- **UI Component Guide**: How to add custom checkout step
- **API Documentation**: REST/GraphQL schema
