---
title: "Magento_Checkout Execution Flows"
module: "Magento_Checkout"
doc_type: "execution-flows"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Checkout Execution Flows

## Overview

This document details the critical execution flows in the `Magento_Checkout` module, from initial cart access through order placement. Understanding these flows is essential for debugging checkout issues, customizing checkout behavior, and ensuring proper integration with payment and shipping methods.

**Target Version:** Magento 2.4.7+ (Adobe Commerce & Open Source)

---

## Flow 1: Complete Checkout Process (End-to-End)

This is the master flow that encompasses all sub-flows from cart to order confirmation.

### High-Level Sequence

```
Cart Page → Checkout Page Load → Shipping Step → Payment Step → Place Order → Success Page
```

### Detailed Step-by-Step Flow

#### Step 1: Customer Navigates to Checkout (`/checkout`)

**Entry Point:** `\Magento\Checkout\Controller\Index\Index::execute()`

```php
namespace Magento\Checkout\Controller\Index;

use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Framework\Controller\ResultInterface;
use Magento\Framework\View\Result\PageFactory;
use Magento\Checkout\Model\Session as CheckoutSession;
use Magento\Framework\Exception\LocalizedException;

class Index implements HttpGetActionInterface
{
    public function __construct(
        private readonly PageFactory $resultPageFactory,
        private readonly CheckoutSession $checkoutSession,
        private readonly \Magento\Quote\Api\CartRepositoryInterface $quoteRepository,
        private readonly \Magento\Framework\Controller\Result\RedirectFactory $resultRedirectFactory,
        private readonly \Magento\Framework\Message\ManagerInterface $messageManager
    ) {}

    /**
     * Execute checkout page load
     *
     * @return ResultInterface
     */
    public function execute(): ResultInterface
    {
        try {
            // Get active quote
            $quote = $this->checkoutSession->getQuote();

            // Validate quote has items
            if (!$quote->hasItems() || $quote->getHasError()) {
                throw new LocalizedException(__('We can\'t initialize checkout.'));
            }

            // Validate minimum order amount
            if (!$quote->validateMinimumAmount()) {
                throw new LocalizedException(
                    __('Minimum order amount is %1', $quote->getMinimumOrderAmount())
                );
            }

            // Reserve order ID for the quote
            if (!$quote->getReservedOrderId()) {
                $quote->reserveOrderId()->save();
            }

            // Render checkout page
            $resultPage = $this->resultPageFactory->create();
            $resultPage->getConfig()->getTitle()->set(__('Checkout'));

            return $resultPage;

        } catch (LocalizedException $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
            return $this->redirectToCart();
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage(
                __('We can\'t initialize checkout.')
            );
            return $this->redirectToCart();
        }
    }

    /**
     * Redirect to cart page
     *
     * @return ResultInterface
     */
    private function redirectToCart(): ResultInterface
    {
        $resultRedirect = $this->resultRedirectFactory->create();
        $resultRedirect->setPath('checkout/cart');
        return $resultRedirect;
    }
}
```

**What Happens:**

1. System loads active quote from checkout session
2. Validates quote has items and no errors
3. Validates minimum order amount requirement
4. Reserves order increment ID if not already reserved
5. Renders checkout page with Knockout.js components

#### Step 2: Checkout Page Initialization (Frontend)

**JavaScript Entry Point:** `Magento_Checkout/js/view/checkout.js`

```javascript
define([
    'uiComponent',
    'Magento_Customer/js/model/customer',
    'Magento_Checkout/js/model/quote',
    'Magento_Checkout/js/model/step-navigator',
    'Magento_Checkout/js/model/sidebar'
], function (
    Component,
    customer,
    quote,
    stepNavigator,
    sidebarModel
) {
    'use strict';

    return Component.extend({
        defaults: {
            template: 'Magento_Checkout/checkout',
            visible: true
        },

        /**
         * Initialize checkout
         */
        initialize: function () {
            this._super();

            // Load customer data
            if (customer.isLoggedIn()) {
                customer.getCustomerData();
            }

            // Load quote data
            this.loadQuoteData();

            // Initialize sidebar
            sidebarModel.initialize();

            // Navigate to first step
            stepNavigator.navigateToFirstStep();

            return this;
        },

        /**
         * Load quote data from window.checkoutConfig
         */
        loadQuoteData: function () {
            var checkoutConfig = window.checkoutConfig;

            quote.totals(checkoutConfig.totalsData);
            quote.setItems(checkoutConfig.quoteItemData);
            quote.shippingMethod(checkoutConfig.selectedShippingMethod);

            if (checkoutConfig.quoteData.entity_id) {
                quote.setQuoteId(checkoutConfig.quoteData.entity_id);
            }
        }
    });
});
```

**What Happens:**

1. Checkout component initializes
2. Customer data loaded if logged in
3. Quote data loaded from `window.checkoutConfig` (injected by ConfigProvider)
4. Sidebar summary initialized
5. Navigation to first step (shipping)

#### Step 3: Shipping Address & Method Selection

**Frontend Component:** `Magento_Checkout/js/view/shipping.js`

Customer enters shipping address and selects shipping method, then clicks "Next".

**Action Triggered:** `Magento_Checkout/js/action/set-shipping-information.js`

```javascript
define([
    'jquery',
    'Magento_Checkout/js/model/quote',
    'Magento_Checkout/js/model/url-builder',
    'mage/storage',
    'Magento_Checkout/js/model/error-processor',
    'Magento_Customer/js/model/customer',
    'Magento_Checkout/js/model/full-screen-loader',
    'Magento_Checkout/js/action/get-totals',
    'Magento_Checkout/js/model/shipping-save-processor/default'
], function (
    $,
    quote,
    urlBuilder,
    storage,
    errorProcessor,
    customer,
    fullScreenLoader,
    getTotalsAction,
    defaultProcessor
) {
    'use strict';

    return function (messageContainer) {
        var serviceUrl,
            payload,
            payloadExtender;

        // Build payload
        payload = {
            addressInformation: {
                shipping_address: quote.shippingAddress(),
                billing_address: quote.billingAddress(),
                shipping_method_code: quote.shippingMethod().method_code,
                shipping_carrier_code: quote.shippingMethod().carrier_code
            }
        };

        // Build service URL
        if (customer.isLoggedIn()) {
            serviceUrl = urlBuilder.createUrl('/carts/mine/shipping-information', {});
        } else {
            serviceUrl = urlBuilder.createUrl('/guest-carts/:cartId/shipping-information', {
                cartId: quote.getQuoteId()
            });
            payload.addressInformation.extension_attributes = {
                guest_email: quote.guestEmail
            };
        }

        fullScreenLoader.startLoader();

        // Make API call
        return storage.post(
            serviceUrl,
            JSON.stringify(payload)
        ).done(function (response) {
            // Update quote with response data
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

**Backend Endpoint:** `POST /rest/V1/carts/mine/shipping-information`

**Service Contract:** `\Magento\Checkout\Api\ShippingInformationManagementInterface::saveAddressInformation()`

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
        private readonly \Magento\Quote\Model\QuoteAddressValidator $addressValidator,
        private readonly PaymentDetailsFactory $paymentDetailsFactory,
        private readonly \Magento\Quote\Api\CartTotalRepositoryInterface $cartTotalsRepository,
        private readonly \Magento\Quote\Api\PaymentMethodManagementInterface $paymentMethodManagement,
        private readonly \Psr\Log\LoggerInterface $logger,
        private readonly \Magento\Customer\Api\AddressRepositoryInterface $addressRepository,
        private readonly \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig
    ) {}

    /**
     * {@inheritdoc}
     */
    public function saveAddressInformation(
        int $cartId,
        ShippingInformationInterface $addressInformation
    ): \Magento\Checkout\Api\Data\PaymentDetailsInterface {

        // Load quote
        $quote = $this->quoteRepository->getActive($cartId);

        if ($quote->isVirtual()) {
            throw new NoSuchEntityException(
                __('Cart contains virtual product(s) only. Shipping address is not applicable.')
            );
        }

        // Get addresses from request
        $shippingAddress = $addressInformation->getShippingAddress();
        $billingAddress = $addressInformation->getBillingAddress();

        // Validate shipping address
        $this->validateShippingAddress($shippingAddress);

        // Set shipping address on quote
        $shippingAddress->setCustomerAddressId(null);
        $quote->setShippingAddress($shippingAddress);

        // Set shipping method
        $carrierCode = $addressInformation->getShippingCarrierCode();
        $methodCode = $addressInformation->getShippingMethodCode();
        $shippingMethod = $carrierCode . '_' . $methodCode;

        $quote->getShippingAddress()->setShippingMethod($shippingMethod);
        $quote->getShippingAddress()->setCollectShippingRates(true);

        // Set billing address if provided
        if ($billingAddress) {
            $quote->setBillingAddress($billingAddress);
        }

        // Process extension attributes (e.g., guest email)
        $extensionAttributes = $addressInformation->getExtensionAttributes();
        if ($extensionAttributes && $extensionAttributes->getGuestEmail()) {
            $quote->setCustomerEmail($extensionAttributes->getGuestEmail());
        }

        // Collect totals
        $quote->setTotalsCollectedFlag(false);
        $quote->collectTotals();

        // Save quote
        $this->quoteRepository->save($quote);

        // Build and return payment details
        return $this->buildPaymentDetails($quote);
    }

    /**
     * Validate shipping address
     *
     * @param \Magento\Quote\Api\Data\AddressInterface $address
     * @return void
     * @throws InputException
     */
    private function validateShippingAddress(
        \Magento\Quote\Api\Data\AddressInterface $address
    ): void {
        $errors = $this->addressValidator->validate($address);

        if (!empty($errors)) {
            throw new InputException(
                __('Unable to save address. Please check input data. %1', implode(' ', $errors))
            );
        }
    }

    /**
     * Build payment details response
     *
     * @param \Magento\Quote\Model\Quote $quote
     * @return \Magento\Checkout\Api\Data\PaymentDetailsInterface
     */
    private function buildPaymentDetails(
        \Magento\Quote\Model\Quote $quote
    ): \Magento\Checkout\Api\Data\PaymentDetailsInterface {

        $paymentDetails = $this->paymentDetailsFactory->create();

        // Get available payment methods
        $paymentMethods = $this->paymentMethodManagement->getList($quote->getId());
        $paymentDetails->setPaymentMethods($paymentMethods);

        // Get updated totals
        $totals = $this->cartTotalsRepository->get($quote->getId());
        $paymentDetails->setTotals($totals);

        return $paymentDetails;
    }
}
```

**Events Dispatched:**

- `sales_quote_address_save_before`
- `sales_quote_address_save_after`
- `sales_quote_collect_totals_before`
- `sales_quote_collect_totals_after`
- `sales_quote_save_before`
- `sales_quote_save_after`

**What Happens:**

1. Frontend validates address and shipping method
2. API call to `POST /rest/V1/carts/mine/shipping-information`
3. Backend validates shipping address fields
4. Shipping address saved to quote
5. Shipping method set on quote shipping address
6. Totals collected (shipping cost, tax, grand total)
7. Quote saved to database
8. Payment methods and updated totals returned to frontend
9. Frontend updates UI with new totals and payment methods
10. User navigates to payment step

#### Step 4: Payment Method Selection & Billing Address

**Frontend Component:** `Magento_Checkout/js/view/payment.js`

Customer selects payment method and enters billing address (if different from shipping).

**Payment Method Renderer Example:** `Magento_Checkout/js/view/payment/default.js`

```javascript
define([
    'ko',
    'Magento_Checkout/js/view/payment/default',
    'Magento_Checkout/js/model/quote'
], function (ko, Component, quote) {
    'use strict';

    return Component.extend({
        defaults: {
            template: 'Magento_Checkout/payment/default'
        },

        /**
         * Get payment method data
         *
         * @returns {Object}
         */
        getData: function () {
            return {
                'method': this.item.method,
                'po_number': null,
                'additional_data': null
            };
        },

        /**
         * Check if payment method is available
         *
         * @returns {Boolean}
         */
        isAvailable: function () {
            return quote.totals().grand_total > 0;
        },

        /**
         * Place order
         */
        placeOrder: function (data, event) {
            var self = this;

            if (event) {
                event.preventDefault();
            }

            if (this.validate() && additionalValidators.validate()) {
                this.isPlaceOrderActionAllowed(false);

                this.getPlaceOrderDeferredObject()
                    .done(function () {
                        self.afterPlaceOrder();

                        if (self.redirectAfterPlaceOrder) {
                            redirectOnSuccessAction.execute();
                        }
                    }).fail(function () {
                        self.isPlaceOrderActionAllowed(true);
                    });

                return true;
            }

            return false;
        },

        /**
         * Get place order deferred object
         *
         * @returns {jQuery.Deferred}
         */
        getPlaceOrderDeferredObject: function () {
            return $.when(
                placeOrderAction(this.getData(), this.messageContainer)
            );
        }
    });
});
```

#### Step 5: Place Order

**Action Triggered:** `Magento_Checkout/js/action/place-order.js`

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

    return function (paymentData, messageContainer) {
        var serviceUrl,
            payload;

        // Build payload with payment and billing address
        payload = {
            cartId: quote.getQuoteId(),
            paymentMethod: paymentData,
            billingAddress: quote.billingAddress()
        };

        // Build service URL
        if (customer.isLoggedIn()) {
            serviceUrl = urlBuilder.createUrl('/carts/mine/payment-information', {});
        } else {
            serviceUrl = urlBuilder.createUrl('/guest-carts/:cartId/payment-information', {
                cartId: quote.getQuoteId()
            });
            payload.email = quote.guestEmail;
        }

        fullScreenLoader.startLoader();

        // Make API call to place order
        return storage.post(
            serviceUrl,
            JSON.stringify(payload)
        ).fail(function (response) {
            errorProcessor.process(response, messageContainer);
            fullScreenLoader.stopLoader();
        });
    };
});
```

**Backend Endpoint:** `POST /rest/V1/carts/mine/payment-information`

**Service Contract:** `\Magento\Checkout\Api\PaymentInformationManagementInterface::savePaymentInformationAndPlaceOrder()`

```php
namespace Magento\Checkout\Model;

use Magento\Checkout\Api\PaymentInformationManagementInterface;
use Magento\Quote\Api\Data\PaymentInterface;
use Magento\Quote\Api\Data\AddressInterface;
use Magento\Framework\Exception\CouldNotSaveException;

class PaymentInformationManagement implements PaymentInformationManagementInterface
{
    public function __construct(
        private readonly \Magento\Quote\Api\BillingAddressManagementInterface $billingAddressManagement,
        private readonly \Magento\Quote\Api\PaymentMethodManagementInterface $paymentMethodManagement,
        private readonly \Magento\Quote\Api\CartManagementInterface $cartManagement,
        private readonly PaymentDetailsFactory $paymentDetailsFactory,
        private readonly \Magento\Quote\Api\CartTotalRepositoryInterface $cartTotalsRepository,
        private readonly \Psr\Log\LoggerInterface $logger,
        private readonly \Magento\Quote\Api\CartRepositoryInterface $cartRepository
    ) {}

    /**
     * {@inheritdoc}
     */
    public function savePaymentInformationAndPlaceOrder(
        int $cartId,
        PaymentInterface $paymentMethod,
        ?AddressInterface $billingAddress = null
    ): int {

        $this->logger->info('Starting order placement for cart: ' . $cartId);

        try {
            // Save payment information
            $this->savePaymentInformation($cartId, $paymentMethod, $billingAddress);

            // Place order
            $orderId = $this->cartManagement->placeOrder($cartId);

            if (!$orderId) {
                throw new CouldNotSaveException(
                    __('A server error stopped your order from being placed. Please try again.')
                );
            }

            $this->logger->info('Order placed successfully. Order ID: ' . $orderId);

            return $orderId;

        } catch (\Magento\Framework\Exception\LocalizedException $e) {
            $this->logger->error('Order placement failed: ' . $e->getMessage());
            throw new CouldNotSaveException(__($e->getMessage()), $e);
        } catch (\Exception $e) {
            $this->logger->critical('Unexpected error during order placement', [
                'exception' => $e
            ]);
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
        int $cartId,
        PaymentInterface $paymentMethod,
        ?AddressInterface $billingAddress = null
    ): int {

        // Get quote
        $quote = $this->cartRepository->getActive($cartId);

        // Set billing address
        if ($billingAddress) {
            $billingAddress->setCustomerAddressId(null);
            $this->billingAddressManagement->assign($cartId, $billingAddress);
        }

        // Set payment method
        $this->paymentMethodManagement->set($cartId, $paymentMethod);

        return $cartId;
    }
}
```

**Quote to Order Conversion:** `\Magento\Quote\Model\QuoteManagement::placeOrder()`

```php
namespace Magento\Quote\Model;

use Magento\Quote\Api\CartManagementInterface;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Exception\CouldNotSaveException;

class QuoteManagement implements CartManagementInterface
{
    public function __construct(
        private readonly \Magento\Framework\Event\ManagerInterface $eventManager,
        private readonly \Magento\Quote\Model\Quote\Address\ToOrder $quoteAddressToOrder,
        private readonly \Magento\Quote\Model\Quote\Address\ToOrderAddress $quoteAddressToOrderAddress,
        private readonly \Magento\Quote\Model\Quote\Item\ToOrderItem $quoteItemToOrderItem,
        private readonly \Magento\Quote\Model\Quote\Payment\ToOrderPayment $quotePaymentToOrderPayment,
        private readonly \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        private readonly QuoteRepository $quoteRepository,
        private readonly \Magento\Framework\DB\TransactionFactory $transactionFactory,
        private readonly \Magento\Quote\Model\CustomerManagement $customerManagement,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * {@inheritdoc}
     */
    public function placeOrder($cartId, PaymentInterface $paymentMethod = null)
    {
        // Load quote
        $quote = $this->quoteRepository->getActive($cartId);

        // Validate quote
        $this->validateQuote($quote);

        // Dispatch before submit event
        $this->eventManager->dispatch(
            'sales_model_service_quote_submit_before',
            ['quote' => $quote]
        );

        try {
            // Start database transaction
            $transaction = $this->transactionFactory->create();

            // Convert quote to order
            $order = $this->submit($quote);

            // Save order
            $this->orderRepository->save($order);
            $transaction->addObject($order);

            // Deactivate quote
            $quote->setIsActive(false);
            $this->quoteRepository->save($quote);
            $transaction->addObject($quote);

            // Commit transaction
            $transaction->save();

            // Dispatch after submit event
            $this->eventManager->dispatch(
                'sales_model_service_quote_submit_success',
                ['order' => $order, 'quote' => $quote]
            );

            return $order->getEntityId();

        } catch (\Exception $e) {
            $this->logger->critical($e);

            // Dispatch failure event
            $this->eventManager->dispatch(
                'sales_model_service_quote_submit_failure',
                ['quote' => $quote, 'exception' => $e]
            );

            throw new CouldNotSaveException(
                __('An error occurred on the server. Please try again.'),
                $e
            );
        }
    }

    /**
     * Submit quote and create order
     *
     * @param Quote $quote
     * @return \Magento\Sales\Api\Data\OrderInterface
     * @throws LocalizedException
     */
    protected function submit(Quote $quote): \Magento\Sales\Api\Data\OrderInterface
    {
        // Create order from quote
        $order = $this->quoteAddressToOrder->convert($quote->getShippingAddress());

        // Set billing address
        $order->setBillingAddress(
            $this->quoteAddressToOrderAddress->convert($quote->getBillingAddress())
        );

        // Set shipping address (if not virtual)
        if (!$quote->isVirtual()) {
            $order->setShippingAddress(
                $this->quoteAddressToOrderAddress->convert($quote->getShippingAddress())
            );
        }

        // Set payment
        $order->setPayment(
            $this->quotePaymentToOrderPayment->convert($quote->getPayment())
        );

        // Convert quote items to order items
        foreach ($quote->getAllItems() as $quoteItem) {
            if ($quoteItem->getParentItem()) {
                continue;
            }

            $orderItem = $this->quoteItemToOrderItem->convert($quoteItem);
            $order->addItem($orderItem);

            // Add child items
            if ($quoteItem->getHasChildren()) {
                foreach ($quoteItem->getChildren() as $childItem) {
                    $orderChildItem = $this->quoteItemToOrderItem->convert($childItem);
                    $orderChildItem->setParentItem($orderItem);
                    $order->addItem($orderChildItem);
                }
            }
        }

        // Set customer data
        if ($quote->getCustomerId()) {
            $order->setCustomerId($quote->getCustomerId());
            $order->setCustomerIsGuest(false);
        } else {
            $order->setCustomerIsGuest(true);
        }

        $order->setCustomerEmail($quote->getCustomerEmail());
        $order->setCustomerFirstname($quote->getCustomerFirstname());
        $order->setCustomerLastname($quote->getCustomerLastname());

        // Dispatch order creation event
        $this->eventManager->dispatch(
            'checkout_submit_all_after',
            ['order' => $order, 'quote' => $quote]
        );

        return $order;
    }

    /**
     * Validate quote before placing order
     *
     * @param Quote $quote
     * @return void
     * @throws LocalizedException
     */
    private function validateQuote(Quote $quote): void
    {
        if (!$quote->hasItems()) {
            throw new LocalizedException(__('The cart is empty. Please add items to proceed.'));
        }

        if ($quote->getHasError()) {
            throw new LocalizedException(__('The cart contains errors. Please review and correct them.'));
        }

        if (!$quote->validateMinimumAmount()) {
            throw new LocalizedException(
                __('Order amount must be at least %1', $quote->getStore()->getCurrentCurrency()->format($quote->getMinimumOrderAmount(), [], false))
            );
        }

        // Validate shipping method for non-virtual quotes
        if (!$quote->isVirtual() && !$quote->getShippingAddress()->getShippingMethod()) {
            throw new LocalizedException(__('Please specify a shipping method.'));
        }

        // Validate payment method
        if (!$quote->getPayment()->getMethod()) {
            throw new LocalizedException(__('Please specify a payment method.'));
        }
    }
}
```

**Events Dispatched During Order Placement:**

1. `sales_model_service_quote_submit_before` - Before order creation
2. `checkout_submit_all_after` - After order object created
3. `sales_order_place_before` - Before order saved
4. `sales_order_place_after` - After order saved
5. `sales_order_save_after` - After order committed to DB
6. `sales_model_service_quote_submit_success` - After successful order placement
7. `checkout_onepage_controller_success_action` - On success page load

**What Happens:**

1. Payment method and billing address saved to quote
2. Quote validated (items, minimum amount, shipping method, payment method)
3. Database transaction started
4. Quote converted to order (address, items, payment, totals)
5. Order saved to database
6. Quote deactivated (`is_active = 0`)
7. Transaction committed
8. Inventory reserved via `InventoryReservation` module
9. Order confirmation email sent via `Sales/Model/Order/Email/Sender/OrderSender`
10. Customer redirected to success page

#### Step 6: Order Success Page

**Controller:** `\Magento\Checkout\Controller\Onepage\Success::execute()`

```php
namespace Magento\Checkout\Controller\Onepage;

use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Framework\Controller\ResultInterface;
use Magento\Framework\View\Result\PageFactory;
use Magento\Checkout\Model\Session as CheckoutSession;

class Success implements HttpGetActionInterface
{
    public function __construct(
        private readonly PageFactory $resultPageFactory,
        private readonly CheckoutSession $checkoutSession,
        private readonly \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        private readonly \Magento\Framework\Controller\Result\RedirectFactory $resultRedirectFactory
    ) {}

    /**
     * Display order success page
     *
     * @return ResultInterface
     */
    public function execute(): ResultInterface
    {
        // Get last order ID from session
        $orderId = $this->checkoutSession->getLastOrderId();

        if (!$orderId) {
            // No order in session, redirect to home
            $resultRedirect = $this->resultRedirectFactory->create();
            $resultRedirect->setPath('/');
            return $resultRedirect;
        }

        // Load order
        $order = $this->orderRepository->get($orderId);

        // Dispatch success event
        $this->eventManager->dispatch(
            'checkout_onepage_controller_success_action',
            ['order' => $order]
        );

        // Render success page
        $resultPage = $this->resultPageFactory->create();
        $resultPage->getConfig()->getTitle()->set(__('Success'));

        return $resultPage;
    }
}
```

**Success Block:** `\Magento\Checkout\Block\Onepage\Success`

```php
namespace Magento\Checkout\Block\Onepage;

use Magento\Framework\View\Element\Template;
use Magento\Checkout\Model\Session as CheckoutSession;

class Success extends Template
{
    public function __construct(
        Template\Context $context,
        private readonly CheckoutSession $checkoutSession,
        private readonly \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        array $data = []
    ) {
        parent::__construct($context, $data);
    }

    /**
     * Get order increment ID
     *
     * @return string|null
     */
    public function getOrderId(): ?string
    {
        return $this->checkoutSession->getLastRealOrderId();
    }

    /**
     * Check if customer can view order
     *
     * @return bool
     */
    public function canViewOrder(): bool
    {
        return $this->checkoutSession->getLastOrderId() !== null;
    }

    /**
     * Get view order URL
     *
     * @return string
     */
    public function getViewOrderUrl(): string
    {
        return $this->getUrl('sales/order/view', [
            'order_id' => $this->checkoutSession->getLastOrderId()
        ]);
    }
}
```

---

## Flow 2: Guest Checkout vs. Registered Customer Checkout

### Guest Checkout Flow

**Key Differences:**

1. **Guest email required** - Captured during shipping step
2. **Different API endpoints** - `/guest-carts/:cartId/...` instead of `/carts/mine/...`
3. **Cart ID in URL** - Guest cart ID passed explicitly in API calls
4. **No customer association** - Order placed without customer_id
5. **No saved addresses** - Cannot select from address book

**Email Capture:**

```javascript
// Magento_Checkout/js/view/form/element/email.js
define([
    'ko',
    'Magento_Ui/js/form/element/abstract',
    'Magento_Checkout/js/model/quote'
], function (ko, Abstract, quote) {
    'use strict';

    return Abstract.extend({
        defaults: {
            template: 'Magento_Checkout/form/element/email',
            email: '',
            emailCheckTimeout: 500
        },

        /**
         * Initialize email component
         */
        initialize: function () {
            this._super();

            // Subscribe to email changes
            this.email.subscribe(function (newEmail) {
                quote.guestEmail = newEmail;

                // Check if email already exists
                this.checkEmailAvailability(newEmail);
            }, this);

            return this;
        },

        /**
         * Check if email is already registered
         */
        checkEmailAvailability: function (email) {
            clearTimeout(this.emailCheckHandler);

            this.emailCheckHandler = setTimeout(function () {
                $.ajax({
                    url: '/rest/V1/customers/isEmailAvailable',
                    type: 'POST',
                    data: JSON.stringify({
                        customerEmail: email
                    }),
                    contentType: 'application/json'
                }).done(function (isEmailAvailable) {
                    if (!isEmailAvailable) {
                        // Email exists, show login prompt
                        this.showLoginPrompt();
                    }
                }.bind(this));
            }.bind(this), this.emailCheckTimeout);
        }
    });
});
```

### Registered Customer Checkout Flow

**Key Advantages:**

1. **Address book** - Select from saved addresses
2. **Faster checkout** - Pre-filled customer data
3. **Order history** - View orders in account
4. **Reorder capability** - Easy reordering from past orders

**Address Selection:**

```javascript
// Magento_Checkout/js/view/shipping-address/address-renderer/default.js
define([
    'ko',
    'uiComponent',
    'Magento_Checkout/js/action/select-shipping-address',
    'Magento_Checkout/js/model/quote',
    'Magento_Customer/js/customer-data'
], function (
    ko,
    Component,
    selectShippingAddressAction,
    quote,
    customerData
) {
    'use strict';

    return Component.extend({
        defaults: {
            template: 'Magento_Checkout/shipping-address/address-renderer/default'
        },

        /**
         * Check if this address is selected
         *
         * @returns {Boolean}
         */
        isSelected: ko.computed(function () {
            var selectedAddress = quote.shippingAddress();
            var addressData = this.address();

            if (selectedAddress && addressData) {
                return selectedAddress.getKey() === addressData.getKey();
            }

            return false;
        }, this),

        /**
         * Select this address
         */
        selectAddress: function () {
            selectShippingAddressAction(this.address());
        },

        /**
         * Edit this address
         */
        editAddress: function () {
            // Trigger edit address modal
            this.showAddressForm(this.address());
        }
    });
});
```

---

## Flow 3: Virtual Product Checkout

Virtual products (downloadable, services) skip shipping step entirely.

**Virtual Quote Detection:**

```php
// \Magento\Quote\Model\Quote::isVirtual()
public function isVirtual(): bool
{
    $isVirtual = true;

    foreach ($this->getAllItems() as $item) {
        if (!$item->getIsVirtual()) {
            $isVirtual = false;
            break;
        }
    }

    return $isVirtual;
}
```

**Layout Modification for Virtual Checkout:**

```xml
<!-- checkout_index_index.xml -->
<body>
    <referenceBlock name="checkout.root">
        <arguments>
            <argument name="jsLayout" xsi:type="array">
                <item name="components" xsi:type="array">
                    <item name="checkout" xsi:type="array">
                        <item name="children" xsi:type="array">
                            <item name="steps" xsi:type="array">
                                <item name="children" xsi:type="array">
                                    <!-- Remove shipping step for virtual quotes -->
                                    <item name="shipping-step" xsi:type="array">
                                        <item name="visible" xsi:type="helper" helper="Magento\Checkout\Helper\Data::isVirtualQuote"/>
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
```

---

## Flow 4: Multi-Shipping Checkout (Legacy)

Multi-shipping checkout allows customers to ship items to multiple addresses.

**Entry Point:** `/checkout/multishipping`

**Flow:**

1. Customer selects addresses for each item
2. Selects shipping method for each address
3. Reviews order summary
4. Places order - creates **multiple orders** (one per shipping address)

**Note:** Multi-shipping is considered legacy and rarely used in modern implementations.

---

## Assumptions

- **Target Platform**: Adobe Commerce & Magento Open Source 2.4.7+
- **PHP Version**: 8.1, 8.2, 8.3
- **Frontend**: Luma theme with Knockout.js UI components
- **Session Storage**: Redis (production)
- **Database Transaction Isolation**: READ COMMITTED

## Why These Flows

The checkout flows are designed to:

- **Minimize round-trips** - Batch operations where possible (shipping info + totals in one call)
- **Validate early** - Client-side validation before API calls
- **Maintain data integrity** - Database transactions ensure atomic order creation
- **Support extensibility** - Events at each step allow customization
- **Handle errors gracefully** - Try-catch blocks with rollback on failure

## Security Impact

- **Quote Validation**: Every API call validates quote ownership
- **CSRF Protection**: Form keys validated on all POST requests
- **Payment Security**: Payment methods handle sensitive data, never exposed in checkout session
- **Rate Limiting**: Consider implementing rate limiting on place order endpoint
- **Fraud Detection**: Integration with fraud prevention services via payment methods

## Performance Impact

- **Database Transactions**: Order placement uses transaction, may cause locks under high load
- **Session I/O**: Heavy session usage; Redis critical for performance
- **Totals Calculation**: Triggered multiple times; cache quote items where possible
- **Inventory Reservation**: May cause contention on `inventory_reservation` table
- **Email Sending**: Async via `sales_order_place_after` event recommended

## Backward Compatibility

- **Service Contracts**: All service contracts are BC-guaranteed
- **Events**: Event names and payloads are stable
- **Quote Structure**: Quote tables maintain BC across versions
- **API Endpoints**: REST/GraphQL endpoints follow semantic versioning

## Tests to Add

1. **Unit Tests**: Test service contract implementations with mocked dependencies
2. **Integration Tests**: Test full checkout flow with fixtures
3. **API Tests**: Test REST endpoints with WebAPI framework
4. **MFTF Tests**: End-to-end checkout flows for guest and registered customers
5. **Load Tests**: Concurrent order placement under stress

## Documentation to Update

- **Flowcharts**: Visual representation of checkout flows
- **API Documentation**: REST/GraphQL endpoint contracts
- **Event Reference**: List of events dispatched during checkout
- **Troubleshooting Guide**: Common checkout errors and resolutions
