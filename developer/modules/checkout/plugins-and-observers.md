---
title: "Magento_Checkout Plugins & Observers"
module: "Magento_Checkout"
doc_type: "plugins-and-observers"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Checkout Plugins and Observers

## Overview

The `Magento_Checkout` module provides numerous extension points through the event-observer pattern and plugin system. Understanding these extension mechanisms is critical for customizing checkout behavior without modifying core files or breaking upgrade paths.

**Target Version:** Magento 2.4.7+ (Adobe Commerce & Open Source)

---

## Event-Observer Pattern

### Critical Checkout Events

#### 1. Quote Submission Events

**`sales_model_service_quote_submit_before`**

Dispatched before quote is converted to order. Last chance to modify quote data or cancel order placement.

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;
use Magento\Framework\Exception\LocalizedException;

class ValidateQuoteBeforeSubmit implements ObserverInterface
{
    public function __construct(
        private readonly \Psr\Log\LoggerInterface $logger,
        private readonly \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig
    ) {}

    /**
     * Validate quote before order placement
     *
     * @param Observer $observer
     * @return void
     * @throws LocalizedException
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        $this->logger->info('Validating quote before submission', [
            'quote_id' => $quote->getId(),
            'grand_total' => $quote->getGrandTotal()
        ]);

        // Example: Prevent orders over certain amount without manager approval
        $maxOrderAmount = $this->scopeConfig->getValue(
            'sales/maximum_order/amount',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        );

        if ($maxOrderAmount && $quote->getGrandTotal() > $maxOrderAmount) {
            $approvalCode = $quote->getData('manager_approval_code');

            if (!$approvalCode || !$this->validateApprovalCode($approvalCode)) {
                throw new LocalizedException(
                    __('Orders over %1 require manager approval code.', $maxOrderAmount)
                );
            }
        }

        // Example: Validate custom quote attribute
        if (!$quote->getData('required_custom_field')) {
            throw new LocalizedException(
                __('Required custom field is missing from quote.')
            );
        }
    }

    /**
     * Validate manager approval code
     *
     * @param string $code
     * @return bool
     */
    private function validateApprovalCode(string $code): bool
    {
        // Implement approval code validation logic
        return strlen($code) === 8 && ctype_alnum($code);
    }
}
```

**Registration (`events.xml`):**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="sales_model_service_quote_submit_before">
        <observer name="vendor_module_validate_quote_before_submit"
                  instance="Vendor\Module\Observer\ValidateQuoteBeforeSubmit"/>
    </event>
</config>
```

**`sales_model_service_quote_submit_success`**

Dispatched after order successfully created from quote.

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Api\Data\OrderInterface;
use Magento\Quote\Model\Quote;

class ProcessOrderAfterPlacement implements ObserverInterface
{
    public function __construct(
        private readonly \Psr\Log\LoggerInterface $logger,
        private readonly \Vendor\Module\Service\OrderNotificationService $notificationService,
        private readonly \Vendor\Module\Api\ExternalSystemInterface $externalSystem
    ) {}

    /**
     * Process order after successful placement
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var OrderInterface $order */
        $order = $observer->getEvent()->getOrder();

        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        $this->logger->info('Order placed successfully', [
            'order_id' => $order->getEntityId(),
            'increment_id' => $order->getIncrementId(),
            'grand_total' => $order->getGrandTotal()
        ]);

        try {
            // Send order data to external system
            $this->externalSystem->syncOrder($order);

            // Send internal notifications
            $this->notificationService->notifyWarehouse($order);

            // Copy custom data from quote to order
            if ($customData = $quote->getData('custom_delivery_instructions')) {
                $order->setData('custom_delivery_instructions', $customData);
                $order->save();
            }

        } catch (\Exception $e) {
            $this->logger->error('Failed to process order after placement', [
                'order_id' => $order->getEntityId(),
                'error' => $e->getMessage()
            ]);
            // Don't throw - order already placed
        }
    }
}
```

**`sales_model_service_quote_submit_failure`**

Dispatched when order placement fails.

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

class HandleOrderPlacementFailure implements ObserverInterface
{
    public function __construct(
        private readonly \Psr\Log\LoggerInterface $logger,
        private readonly \Vendor\Module\Service\AlertService $alertService
    ) {}

    /**
     * Handle order placement failure
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
            'customer_email' => $quote->getCustomerEmail(),
            'error' => $exception->getMessage(),
            'trace' => $exception->getTraceAsString()
        ]);

        // Send alert to operations team for high-value cart failures
        if ($quote->getGrandTotal() > 1000) {
            $this->alertService->notifyOrderFailure($quote, $exception);
        }

        // Mark quote for review
        try {
            $quote->setData('placement_failed', true);
            $quote->setData('failure_reason', $exception->getMessage());
            $quote->save();
        } catch (\Exception $e) {
            $this->logger->error('Failed to update quote after placement failure', [
                'error' => $e->getMessage()
            ]);
        }
    }
}
```

#### 2. Checkout Session Events

**`checkout_quote_init`**

Dispatched when quote is initialized in checkout session.

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

class InitializeQuoteData implements ObserverInterface
{
    public function __construct(
        private readonly \Magento\Customer\Model\Session $customerSession,
        private readonly \Vendor\Module\Service\LoyaltyService $loyaltyService
    ) {}

    /**
     * Initialize custom quote data on checkout
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        // Apply loyalty discount for logged-in customers
        if ($this->customerSession->isLoggedIn()) {
            $customerId = $this->customerSession->getCustomerId();
            $loyaltyDiscount = $this->loyaltyService->calculateDiscount($customerId, $quote);

            if ($loyaltyDiscount > 0) {
                $quote->setData('loyalty_discount', $loyaltyDiscount);
                $quote->setTotalsCollectedFlag(false);
                $quote->collectTotals();
            }
        }
    }
}
```

**`checkout_cart_product_add_after`**

Dispatched after product added to cart.

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote\Item;

class TrackProductAddedToCart implements ObserverInterface
{
    public function __construct(
        private readonly \Vendor\Module\Service\AnalyticsService $analyticsService,
        private readonly \Magento\Checkout\Model\Session $checkoutSession
    ) {}

    /**
     * Track product added to cart for analytics
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        // Note: checkout_cart_product_add_after passes 'items' (array), 'product', and 'request'
        // — not 'quoteItem'. Use $observer->getEvent()->getItems() to get the items array.
        $items = $observer->getEvent()->getItems();
        $product = $observer->getEvent()->getProduct();
        $quote = $this->checkoutSession->getQuote();

        // Send to analytics service
        $this->analyticsService->trackEvent('product_added_to_cart', [
            'product_id' => $product->getId(),
            'sku' => $product->getSku(),
            'name' => $product->getName(),
            'price' => $product->getFinalPrice(),
            'quantity' => $quoteItem->getQty(),
            'cart_total' => $quote->getGrandTotal()
        ]);
    }
}
```

#### 3. Address Events

**`sales_quote_address_save_before`**

Dispatched before quote address is saved.

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote\Address;

class NormalizeAddressData implements ObserverInterface
{
    public function __construct(
        private readonly \Magento\Directory\Model\RegionFactory $regionFactory
    ) {}

    /**
     * Normalize address data before saving
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Address $address */
        $address = $observer->getEvent()->getQuoteAddress();

        // Normalize postcode format
        $postcode = $address->getPostcode();
        if ($postcode) {
            $normalizedPostcode = strtoupper(str_replace(' ', '', $postcode));
            $address->setPostcode($normalizedPostcode);
        }

        // Ensure region_id is set if region text provided
        if ($address->getRegion() && !$address->getRegionId()) {
            $region = $this->regionFactory->create();
            $region->loadByCode($address->getRegion(), $address->getCountryId());

            if ($region->getId()) {
                $address->setRegionId($region->getId());
            }
        }

        // Normalize phone number
        $telephone = $address->getTelephone();
        if ($telephone) {
            $normalizedPhone = preg_replace('/[^0-9+]/', '', $telephone);
            $address->setTelephone($normalizedPhone);
        }
    }
}
```

#### 4. Totals Collection Events

**`sales_quote_collect_totals_before`**

Dispatched before quote totals are collected.

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

class PrepareQuoteForTotalsCollection implements ObserverInterface
{
    public function __construct(
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Prepare quote before totals collection
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        $this->logger->debug('Starting totals collection', [
            'quote_id' => $quote->getId(),
            'items_count' => $quote->getItemsCount()
        ]);

        // Clear custom totals that will be recalculated
        $quote->setData('custom_fee', null);
        $quote->setData('loyalty_discount', null);
    }
}
```

**`sales_quote_collect_totals_after`**

Dispatched after quote totals are collected.

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

class ApplyAdditionalDiscounts implements ObserverInterface
{
    public function __construct(
        private readonly \Vendor\Module\Service\PromotionService $promotionService,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Apply additional discounts after totals collection
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        // Calculate flash sale discount
        $flashSaleDiscount = $this->promotionService->calculateFlashSaleDiscount($quote);

        if ($flashSaleDiscount > 0) {
            $grandTotal = $quote->getGrandTotal();
            $newGrandTotal = max(0, $grandTotal - $flashSaleDiscount);

            $quote->setGrandTotal($newGrandTotal);
            $quote->setBaseGrandTotal($newGrandTotal);
            $quote->setData('flash_sale_discount', $flashSaleDiscount);

            $this->logger->info('Applied flash sale discount', [
                'quote_id' => $quote->getId(),
                'discount' => $flashSaleDiscount,
                'new_total' => $newGrandTotal
            ]);
        }
    }
}
```

#### 5. Order Creation Events

**`checkout_submit_all_after`**

Dispatched after order is created but before it's saved.

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Api\Data\OrderInterface;
use Magento\Quote\Model\Quote;

class CopyQuoteDataToOrder implements ObserverInterface
{
    /**
     * Copy custom data from quote to order
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var OrderInterface $order */
        $order = $observer->getEvent()->getOrder();

        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        // Copy custom fields from quote to order
        $customFields = [
            'delivery_date',
            'delivery_time_slot',
            'gift_message',
            'special_instructions',
            'loyalty_discount',
            'referral_code'
        ];

        foreach ($customFields as $field) {
            if ($value = $quote->getData($field)) {
                $order->setData($field, $value);
            }
        }

        // Copy extension attributes
        $quoteExtension = $quote->getExtensionAttributes();
        $orderExtension = $order->getExtensionAttributes();

        if ($quoteExtension && $orderExtension) {
            if ($giftWrap = $quoteExtension->getGiftWrap()) {
                $orderExtension->setGiftWrap($giftWrap);
            }
        }
    }
}
```

**`sales_order_place_after`**

Dispatched after order is saved to database.

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Api\Data\OrderInterface;

class SendOrderNotifications implements ObserverInterface
{
    public function __construct(
        private readonly \Vendor\Module\Service\SlackNotifier $slackNotifier,
        private readonly \Vendor\Module\Service\SmsService $smsService,
        private readonly \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig
    ) {}

    /**
     * Send notifications after order placement
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var OrderInterface $order */
        $order = $observer->getEvent()->getOrder();

        // Send Slack notification for high-value orders
        $highValueThreshold = $this->scopeConfig->getValue(
            'sales/notifications/high_value_threshold',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        );

        if ($order->getGrandTotal() >= $highValueThreshold) {
            $this->slackNotifier->notifyNewOrder($order);
        }

        // Send SMS to customer if enabled
        if ($this->scopeConfig->isSetFlag('sales/notifications/sms_enabled')) {
            $this->smsService->sendOrderConfirmation($order);
        }
    }
}
```

#### 6. Checkout Success Events

**`checkout_onepage_controller_success_action`**

Dispatched when success page is loaded.

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Api\Data\OrderInterface;

class TrackConversion implements ObserverInterface
{
    public function __construct(
        private readonly \Vendor\Module\Service\AnalyticsService $analyticsService,
        private readonly \Magento\Checkout\Model\Session $checkoutSession
    ) {}

    /**
     * Track conversion on success page
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        $orderId = $this->checkoutSession->getLastOrderId();

        if ($orderId) {
            /** @var OrderInterface $order */
            $order = $observer->getEvent()->getOrder();

            // Send conversion data to analytics
            $this->analyticsService->trackConversion([
                'order_id' => $order->getIncrementId(),
                'revenue' => $order->getGrandTotal(),
                'tax' => $order->getTaxAmount(),
                'shipping' => $order->getShippingAmount(),
                'items' => $this->getOrderItems($order)
            ]);
        }
    }

    /**
     * Get order items for tracking
     *
     * @param OrderInterface $order
     * @return array
     */
    private function getOrderItems(OrderInterface $order): array
    {
        $items = [];

        foreach ($order->getAllVisibleItems() as $item) {
            $items[] = [
                'sku' => $item->getSku(),
                'name' => $item->getName(),
                'price' => $item->getPrice(),
                'quantity' => $item->getQtyOrdered()
            ];
        }

        return $items;
    }
}
```

---

## Plugin System (Interceptors)

### Critical Checkout Plugins

#### 1. Payment Information Management Plugin

Extend payment information to add custom validation.

```php
namespace Vendor\Module\Plugin;

use Magento\Checkout\Api\PaymentInformationManagementInterface;
use Magento\Quote\Api\Data\PaymentInterface;
use Magento\Quote\Api\Data\AddressInterface;
use Magento\Framework\Exception\LocalizedException;

class PaymentInformationManagementExtend
{
    public function __construct(
        private readonly \Vendor\Module\Service\FraudDetectionService $fraudDetection,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Validate payment before placing order
     *
     * @param PaymentInformationManagementInterface $subject
     * @param int $cartId
     * @param PaymentInterface $paymentMethod
     * @param AddressInterface|null $billingAddress
     * @return void
     * @throws LocalizedException
     */
    public function beforeSavePaymentInformationAndPlaceOrder(
        PaymentInformationManagementInterface $subject,
        int $cartId,
        PaymentInterface $paymentMethod,
        ?AddressInterface $billingAddress = null
    ): void {

        $this->logger->info('Validating payment before order placement', [
            'cart_id' => $cartId,
            'payment_method' => $paymentMethod->getMethod()
        ]);

        // Run fraud detection
        $fraudScore = $this->fraudDetection->calculateRiskScore($cartId, $billingAddress);

        if ($fraudScore > 80) {
            throw new LocalizedException(
                __('Your order requires additional verification. Please contact customer service.')
            );
        }

        // Validate payment method specific requirements
        if ($paymentMethod->getMethod() === 'cashondelivery') {
            $this->validateCashOnDeliveryRestrictions($cartId);
        }
    }

    /**
     * Add custom data to order after placement
     *
     * @param PaymentInformationManagementInterface $subject
     * @param int $orderId
     * @return int
     */
    public function afterSavePaymentInformationAndPlaceOrder(
        PaymentInformationManagementInterface $subject,
        int $orderId
    ): int {

        $this->logger->info('Order placed successfully', [
            'order_id' => $orderId
        ]);

        // Log successful payment for fraud tracking
        $this->fraudDetection->logSuccessfulPayment($orderId);

        return $orderId;
    }

    /**
     * Validate cash on delivery restrictions
     *
     * @param int $cartId
     * @throws LocalizedException
     */
    private function validateCashOnDeliveryRestrictions(int $cartId): void
    {
        // Example: Restrict COD based on cart total or delivery location
        // Implementation here
    }
}
```

**Plugin Registration (`di.xml`):**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Checkout\Api\PaymentInformationManagementInterface">
        <plugin name="vendor_module_payment_information_extend"
                type="Vendor\Module\Plugin\PaymentInformationManagementExtend"
                sortOrder="10"/>
    </type>
</config>
```

#### 2. Shipping Information Management Plugin

Add custom logic around shipping information.

```php
namespace Vendor\Module\Plugin;

use Magento\Checkout\Api\ShippingInformationManagementInterface;
use Magento\Checkout\Api\Data\ShippingInformationInterface;
use Magento\Framework\Exception\LocalizedException;

class ShippingInformationManagementExtend
{
    public function __construct(
        private readonly \Vendor\Module\Service\DeliveryEstimator $deliveryEstimator,
        private readonly \Magento\Quote\Api\CartRepositoryInterface $quoteRepository,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Validate shipping information before saving
     *
     * @param ShippingInformationManagementInterface $subject
     * @param int $cartId
     * @param ShippingInformationInterface $addressInformation
     * @return array
     * @throws LocalizedException
     */
    public function beforeSaveAddressInformation(
        ShippingInformationManagementInterface $subject,
        int $cartId,
        ShippingInformationInterface $addressInformation
    ): array {

        $shippingAddress = $addressInformation->getShippingAddress();

        // Validate delivery to address is possible
        if (!$this->deliveryEstimator->canDeliver($shippingAddress)) {
            throw new LocalizedException(
                __('We cannot deliver to the specified address. Please choose a different location.')
            );
        }

        // Validate shipping method is available for this address
        $shippingMethod = $addressInformation->getShippingCarrierCode() . '_' .
                         $addressInformation->getShippingMethodCode();

        if (!$this->isShippingMethodAvailable($cartId, $shippingAddress, $shippingMethod)) {
            throw new LocalizedException(
                __('The selected shipping method is not available for this address.')
            );
        }

        return [$cartId, $addressInformation];
    }

    /**
     * Add estimated delivery date after saving shipping info
     *
     * @param ShippingInformationManagementInterface $subject
     * @param \Magento\Checkout\Api\Data\PaymentDetailsInterface $result
     * @param int $cartId
     * @param ShippingInformationInterface $addressInformation
     * @return \Magento\Checkout\Api\Data\PaymentDetailsInterface
     */
    public function afterSaveAddressInformation(
        ShippingInformationManagementInterface $subject,
        \Magento\Checkout\Api\Data\PaymentDetailsInterface $result,
        int $cartId,
        ShippingInformationInterface $addressInformation
    ): \Magento\Checkout\Api\Data\PaymentDetailsInterface {

        try {
            // Calculate estimated delivery date
            $quote = $this->quoteRepository->getActive($cartId);
            $shippingAddress = $quote->getShippingAddress();
            $shippingMethod = $shippingAddress->getShippingMethod();

            $estimatedDate = $this->deliveryEstimator->estimateDeliveryDate(
                $shippingAddress,
                $shippingMethod
            );

            // Store in quote for later use
            $quote->setData('estimated_delivery_date', $estimatedDate);
            $this->quoteRepository->save($quote);

            $this->logger->info('Estimated delivery date calculated', [
                'cart_id' => $cartId,
                'delivery_date' => $estimatedDate
            ]);

        } catch (\Exception $e) {
            $this->logger->error('Failed to calculate delivery estimate', [
                'error' => $e->getMessage()
            ]);
            // Don't fail the request if delivery estimate fails
        }

        return $result;
    }

    /**
     * Check if shipping method is available for address
     *
     * @param int $cartId
     * @param \Magento\Quote\Api\Data\AddressInterface $address
     * @param string $shippingMethod
     * @return bool
     */
    private function isShippingMethodAvailable(
        int $cartId,
        \Magento\Quote\Api\Data\AddressInterface $address,
        string $shippingMethod
    ): bool {
        // Implementation to check shipping method availability
        return true;
    }
}
```

#### 3. Quote Repository Plugin

Intercept quote operations to add custom logic.

```php
namespace Vendor\Module\Plugin;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\Data\CartInterface;

class QuoteRepositoryExtend
{
    public function __construct(
        private readonly \Vendor\Module\Service\QuoteAuditService $auditService,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Log quote changes before save
     *
     * @param CartRepositoryInterface $subject
     * @param CartInterface $quote
     * @return array
     */
    public function beforeSave(
        CartRepositoryInterface $subject,
        CartInterface $quote
    ): array {

        // Audit quote changes
        if ($quote->getId()) {
            $this->auditService->logQuoteChange($quote);
        }

        return [$quote];
    }

    /**
     * Clear cache after quote save
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

        $this->logger->debug('Quote saved', [
            'quote_id' => $quote->getId(),
            'items_count' => $quote->getItemsCount(),
            'grand_total' => $quote->getGrandTotal()
        ]);

        // Clear custom cache related to this quote
        // Implementation here
    }
}
```

#### 4. Checkout Session Plugin

Extend checkout session functionality.

```php
namespace Vendor\Module\Plugin;

use Magento\Checkout\Model\Session;
use Magento\Quote\Api\Data\CartInterface;

class CheckoutSessionExtend
{
    public function __construct(
        private readonly \Vendor\Module\Service\SessionTracker $sessionTracker,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Track quote access
     *
     * @param Session $subject
     * @param CartInterface $result
     * @return CartInterface
     */
    public function afterGetQuote(
        Session $subject,
        CartInterface $result
    ): CartInterface {

        // Track session activity
        if ($result->getId()) {
            $this->sessionTracker->recordQuoteAccess($result->getId());
        }

        return $result;
    }

    /**
     * Log quote clearing
     *
     * @param Session $subject
     * @param Session $result
     * @return Session
     */
    public function afterClearQuote(
        Session $subject,
        Session $result
    ): Session {

        $this->logger->info('Checkout session cleared');

        return $result;
    }
}
```

#### 5. Cart Management Plugin

Add logic around cart operations.

```php
namespace Vendor\Module\Plugin;

use Magento\Quote\Api\CartManagementInterface;
use Magento\Quote\Api\Data\PaymentInterface;

class CartManagementExtend
{
    public function __construct(
        private readonly \Vendor\Module\Service\InventoryChecker $inventoryChecker,
        private readonly \Vendor\Module\Service\PriceVerifier $priceVerifier,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Perform final validations before placing order
     *
     * @param CartManagementInterface $subject
     * @param int $cartId
     * @param PaymentInterface|null $paymentMethod
     * @return array
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function beforePlaceOrder(
        CartManagementInterface $subject,
        int $cartId,
        ?PaymentInterface $paymentMethod = null
    ): array {

        $this->logger->info('Final validation before order placement', [
            'cart_id' => $cartId
        ]);

        // Verify inventory availability (race condition check)
        if (!$this->inventoryChecker->verifyAvailability($cartId)) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Some products are no longer available in requested quantities.')
            );
        }

        // Verify prices haven't changed
        if (!$this->priceVerifier->verifyPrices($cartId)) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Product prices have changed. Please review your cart.')
            );
        }

        return [$cartId, $paymentMethod];
    }

    /**
     * Log order ID after successful placement
     *
     * @param CartManagementInterface $subject
     * @param int $orderId
     * @return int
     */
    public function afterPlaceOrder(
        CartManagementInterface $subject,
        int $orderId
    ): int {

        $this->logger->info('Order placement completed', [
            'order_id' => $orderId
        ]);

        return $orderId;
    }
}
```

---

## Best Practices for Plugins and Observers

### 1. Plugin vs Observer Decision Matrix

| Use Case | Recommendation | Rationale |
|----------|---------------|-----------|
| Modify method arguments | Plugin (`before`) | Direct access to parameters |
| Modify method return value | Plugin (`after`, `around`) | Can transform result |
| Cancel operation | Plugin (`around`) or Observer | Both can throw exceptions |
| Execute async task | Observer | Decouple from main flow |
| Log operations | Observer | Don't need to modify data |
| Validate input | Plugin (`before`) | Early exit possible |
| Multiple customizations | Observers | Better for composition |

### 2. Plugin Ordering Strategy

```xml
<!-- di.xml -->
<type name="Magento\Checkout\Api\PaymentInformationManagementInterface">
    <!-- Low sortOrder = runs first for before plugins, last for after plugins -->
    <plugin name="validation_plugin"
            type="Vendor\Module\Plugin\ValidationPlugin"
            sortOrder="10"/>

    <plugin name="logging_plugin"
            type="Vendor\Module\Plugin\LoggingPlugin"
            sortOrder="100"/>

    <plugin name="notification_plugin"
            type="Vendor\Module\Plugin\NotificationPlugin"
            sortOrder="200"/>
</type>
```

### 3. Avoid `around` Plugins When Possible

**Bad Example (using `around`):**

```php
public function aroundSavePaymentInformationAndPlaceOrder(
    PaymentInformationManagementInterface $subject,
    \Closure $proceed,
    int $cartId,
    PaymentInterface $paymentMethod,
    ?AddressInterface $billingAddress = null
): int {
    // Validation
    $this->validate($cartId);

    // Call original method
    $orderId = $proceed($cartId, $paymentMethod, $billingAddress);

    // Post-processing
    $this->process($orderId);

    return $orderId;
}
```

**Good Example (using `before` and `after`):**

```php
public function beforeSavePaymentInformationAndPlaceOrder(...): void
{
    $this->validate($cartId);
}

public function afterSavePaymentInformationAndPlaceOrder(..., int $orderId): int
{
    $this->process($orderId);
    return $orderId;
}
```

### 4. Exception Handling in Observers

```php
public function execute(Observer $observer): void
{
    try {
        // Observer logic
        $this->doSomething($observer);

    } catch (\Exception $e) {
        // Log error but don't re-throw unless critical
        $this->logger->error('Observer failed', [
            'observer' => get_class($this),
            'error' => $e->getMessage()
        ]);

        // Only throw if operation must stop
        // throw $e;
    }
}
```

---

## Assumptions

- **Target Platform**: Adobe Commerce & Magento Open Source 2.4.7+
- **PHP Version**: 8.1, 8.2, 8.3
- **Areas**: global (observers), frontend/webapi_rest (plugins)
- **Event Dispatch**: Synchronous unless explicitly queued

## Why Plugins and Observers

- **Upgrade Safety**: No core file modifications required
- **Composition**: Multiple extensions can hook into same points
- **Maintainability**: Clear separation of core logic and customizations
- **Performance**: Observers can be disabled; plugins are compiled
- **Testability**: Easy to mock and test in isolation

## Security Impact

- **Validation in Plugins**: Always validate input in `before` plugins
- **Authorization**: Check permissions before modifying data
- **Sensitive Data**: Never log payment details or PII
- **Exception Messages**: Don't expose system details to users

## Performance Impact

- **Observer Overhead**: Each observer adds latency; keep logic minimal
- **Plugin Chain**: Long plugin chains increase execution time
- **Database Queries**: Avoid N+1 queries in observers/plugins
- **Async Processing**: Use message queues for non-critical operations

## Backward Compatibility

- **Event Payload**: Don't rely on undocumented event data
- **Plugin Signatures**: Match method signatures exactly
- **Service Contracts**: Only plugin service contracts, not models
- **Extension Attributes**: Use extension attributes for adding data

## Tests to Add

1. **Unit Tests**: Test plugin/observer logic with mocked dependencies
2. **Integration Tests**: Test actual plugin/observer execution
3. **MFTF Tests**: Verify behavior changes in UI
4. **Performance Tests**: Measure impact on checkout performance

## Documentation to Update

- **Extension Guide**: Document all extension points
- **Plugin Registry**: List all plugins with purpose and order
- **Event Reference**: Document event payloads and timing
- **Migration Guide**: How to replace deprecated hooks
