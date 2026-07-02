---
title: "Magento_Checkout Integrations"
module: "Magento_Checkout"
doc_type: "integrations"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Checkout Integrations

## Overview

The `Magento_Checkout` module is a central hub that integrates with virtually every major Magento module. Understanding these integrations is critical for building custom checkout flows, troubleshooting issues, and ensuring proper data flow from cart to order.

**Target Version:** Magento 2.4.7+ (Adobe Commerce & Open Source)

---

## Integration 1: Magento_Quote

The Quote module is the foundation of checkout, managing the shopping cart and quote lifecycle.

### Quote to Checkout Integration

**Quote Entity Structure:**

```php
namespace Magento\Quote\Model;

use Magento\Quote\Api\Data\CartInterface;

class Quote extends \Magento\Framework\Model\AbstractModel implements CartInterface
{
    /**
     * Get all visible quote items
     *
     * @return \Magento\Quote\Model\Quote\Item[]
     */
    public function getAllVisibleItems(): array
    {
        $items = [];
        foreach ($this->getAllItems() as $item) {
            if (!$item->getParentItemId()) {
                $items[] = $item;
            }
        }
        return $items;
    }

    /**
     * Collect totals
     *
     * @return $this
     */
    public function collectTotals(): self
    {
        if ($this->getTotalsCollectedFlag()) {
            return $this;
        }

        $this->setTotalsCollectedFlag(false);
        $this->_eventManager->dispatch(
            'sales_quote_collect_totals_before',
            ['quote' => $this]
        );

        // Collect totals for each address
        foreach ($this->getAllAddresses() as $address) {
            $address->setCollectShippingRates(true);
            $address->collectTotals();
        }

        $this->setTotalsCollectedFlag(true);
        $this->_eventManager->dispatch(
            'sales_quote_collect_totals_after',
            ['quote' => $this]
        );

        return $this;
    }

    /**
     * Check if quote is virtual (no physical products)
     *
     * @return bool
     */
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
}
```

### Totals Collection Integration

**Total Collector Implementation:**

```php
namespace Vendor\Module\Model\Quote\Address\Total;

use Magento\Quote\Model\Quote\Address\Total\AbstractTotal;
use Magento\Quote\Model\Quote;
use Magento\Quote\Api\Data\ShippingAssignmentInterface;
use Magento\Quote\Model\Quote\Address\Total;

class CustomFee extends AbstractTotal
{
    /**
     * @var string
     */
    protected $_code = 'custom_fee';

    public function __construct(
        private readonly \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig,
        private readonly \Magento\Store\Model\StoreManagerInterface $storeManager
    ) {}

    /**
     * Collect custom fee total
     *
     * @param Quote $quote
     * @param ShippingAssignmentInterface $shippingAssignment
     * @param Total $total
     * @return $this
     */
    public function collect(
        Quote $quote,
        ShippingAssignmentInterface $shippingAssignment,
        Total $total
    ): self {
        parent::collect($quote, $shippingAssignment, $total);

        if (!$this->isEnabled()) {
            return $this;
        }

        $items = $shippingAssignment->getItems();
        if (!count($items)) {
            return $this;
        }

        // Calculate custom fee
        $fee = $this->calculateFee($quote);

        // Add fee to total
        $total->setCustomFee($fee);
        $total->setBaseCustomFee($fee);

        // Update grand total
        $total->setGrandTotal($total->getGrandTotal() + $fee);
        $total->setBaseGrandTotal($total->getBaseGrandTotal() + $fee);

        // Store fee in quote for later retrieval
        $quote->setCustomFee($fee);

        return $this;
    }

    /**
     * Fetch custom fee for totals display
     *
     * @param Quote $quote
     * @param Total $total
     * @return array
     */
    public function fetch(Quote $quote, Total $total): array
    {
        $fee = $quote->getCustomFee();

        if (!$fee) {
            return [];
        }

        return [
            'code' => $this->getCode(),
            'title' => __('Handling Fee'),
            'value' => $fee
        ];
    }

    /**
     * Get label for total
     *
     * @return \Magento\Framework\Phrase
     */
    public function getLabel(): \Magento\Framework\Phrase
    {
        return __('Handling Fee');
    }

    /**
     * Calculate fee amount
     *
     * @param Quote $quote
     * @return float
     */
    private function calculateFee(Quote $quote): float
    {
        $subtotal = $quote->getSubtotal();
        $feePercent = $this->scopeConfig->getValue(
            'checkout/custom_fee/percent',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        );

        return $subtotal * ($feePercent / 100);
    }

    /**
     * Check if custom fee is enabled
     *
     * @return bool
     */
    private function isEnabled(): bool
    {
        return $this->scopeConfig->isSetFlag(
            'checkout/custom_fee/enabled',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        );
    }
}
```

**Registration (`sales.xml`):**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Sales:etc/sales.xsd">
    <section name="quote">
        <group name="totals">
            <item name="custom_fee" instance="Vendor\Module\Model\Quote\Address\Total\CustomFee" sort_order="500"/>
        </group>
    </section>
</config>
```

---

## Integration 2: Magento_Payment

Payment integration is critical for processing transactions during checkout.

### Payment Method Integration

**Payment Method Implementation:**

```php
namespace Vendor\Payment\Model;

use Magento\Payment\Model\Method\AbstractMethod;
use Magento\Payment\Model\InfoInterface;
use Magento\Framework\Exception\LocalizedException;

class Gateway extends AbstractMethod
{
    const CODE = 'vendor_gateway';

    protected $_code = self::CODE;
    protected $_isGateway = true;
    protected $_canAuthorize = true;
    protected $_canCapture = true;
    protected $_canCapturePartial = true;
    protected $_canRefund = true;
    protected $_canRefundInvoicePartial = true;
    protected $_canVoid = true;
    protected $_canUseInternal = false;
    protected $_canUseCheckout = true;
    protected $_canUseForMultishipping = true;

    public function __construct(
        private readonly \Vendor\Payment\Gateway\Http\Client $httpClient,
        private readonly \Vendor\Payment\Gateway\Request\AuthorizationRequest $authRequest,
        private readonly \Vendor\Payment\Gateway\Response\AuthorizationHandler $authHandler,
        \Magento\Framework\Model\Context $context,
        \Magento\Framework\Registry $registry,
        \Magento\Framework\Api\ExtensionAttributesFactory $extensionFactory,
        \Magento\Framework\Api\AttributeValueFactory $customAttributeFactory,
        \Magento\Payment\Helper\Data $paymentData,
        \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig,
        \Magento\Payment\Model\Method\Logger $logger,
        ?\Magento\Framework\Model\ResourceModel\AbstractResource $resource = null,
        ?\Magento\Framework\Data\Collection\AbstractDb $resourceCollection = null,
        array $data = []
    ) {
        parent::__construct(
            $context,
            $registry,
            $extensionFactory,
            $customAttributeFactory,
            $paymentData,
            $scopeConfig,
            $logger,
            $resource,
            $resourceCollection,
            $data
        );
    }

    /**
     * Authorize payment
     *
     * @param InfoInterface $payment
     * @param float $amount
     * @return $this
     * @throws LocalizedException
     */
    public function authorize(InfoInterface $payment, $amount): self
    {
        if ($amount <= 0) {
            throw new LocalizedException(__('Invalid amount for authorization.'));
        }

        try {
            // Build authorization request
            $request = $this->authRequest->build([
                'payment' => $payment,
                'amount' => $amount
            ]);

            // Send to gateway
            $response = $this->httpClient->placeRequest($request);

            // Handle response
            $this->authHandler->handle([
                'payment' => $payment,
                'response' => $response
            ]);

            // Set transaction ID
            $payment->setTransactionId($response['transaction_id']);
            $payment->setIsTransactionClosed(false);

            return $this;

        } catch (\Exception $e) {
            $this->_logger->error('Authorization failed', [
                'error' => $e->getMessage()
            ]);
            throw new LocalizedException(
                __('Payment authorization failed. Please try again or use a different payment method.')
            );
        }
    }

    /**
     * Capture payment
     *
     * @param InfoInterface $payment
     * @param float $amount
     * @return $this
     * @throws LocalizedException
     */
    public function capture(InfoInterface $payment, $amount): self
    {
        if ($amount <= 0) {
            throw new LocalizedException(__('Invalid amount for capture.'));
        }

        $authTransactionId = $payment->getParentTransactionId();

        try {
            $request = [
                'transaction_id' => $authTransactionId,
                'amount' => $amount
            ];

            $response = $this->httpClient->placeRequest($request);

            $payment->setTransactionId($response['transaction_id']);
            $payment->setIsTransactionClosed(true);

            return $this;

        } catch (\Exception $e) {
            $this->_logger->error('Capture failed', [
                'error' => $e->getMessage()
            ]);
            throw new LocalizedException(
                __('Payment capture failed. Please contact support.')
            );
        }
    }

    /**
     * Check if payment method is available
     *
     * @param \Magento\Quote\Api\Data\CartInterface|null $quote
     * @return bool
     */
    public function isAvailable(\Magento\Quote\Api\Data\CartInterface $quote = null): bool
    {
        if (!parent::isAvailable($quote)) {
            return false;
        }

        if (!$quote) {
            return false;
        }

        // Example: Disable for orders below minimum
        $minAmount = $this->_scopeConfig->getValue(
            'payment/' . self::CODE . '/min_order_total',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        );

        if ($minAmount && $quote->getGrandTotal() < $minAmount) {
            return false;
        }

        // Example: Restrict by country
        $allowedCountries = $this->_scopeConfig->getValue(
            'payment/' . self::CODE . '/allowspecific',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        );

        if ($allowedCountries) {
            $specificCountries = explode(',', $this->_scopeConfig->getValue(
                'payment/' . self::CODE . '/specificcountry',
                \Magento\Store\Model\ScopeInterface::SCOPE_STORE
            ));

            $billingCountry = $quote->getBillingAddress()->getCountryId();
            if (!in_array($billingCountry, $specificCountries)) {
                return false;
            }
        }

        return true;
    }

    /**
     * Validate payment method
     *
     * @return $this
     * @throws LocalizedException
     */
    public function validate(): self
    {
        parent::validate();

        $info = $this->getInfoInstance();

        // Validate required fields
        if (!$info->getAdditionalInformation('card_token')) {
            throw new LocalizedException(__('Payment token is required.'));
        }

        return $this;
    }
}
```

**Payment Config Provider:**

```php
namespace Vendor\Payment\Model;

use Magento\Checkout\Model\ConfigProviderInterface;
use Magento\Framework\App\Config\ScopeConfigInterface;

class ConfigProvider implements ConfigProviderInterface
{
    const CODE = 'vendor_gateway';

    public function __construct(
        private readonly ScopeConfigInterface $scopeConfig,
        private readonly \Magento\Framework\UrlInterface $urlBuilder
    ) {}

    /**
     * Provide payment configuration to checkout
     *
     * @return array
     */
    public function getConfig(): array
    {
        return [
            'payment' => [
                self::CODE => [
                    'isActive' => $this->scopeConfig->isSetFlag(
                        'payment/' . self::CODE . '/active',
                        \Magento\Store\Model\ScopeInterface::SCOPE_STORE
                    ),
                    'apiKey' => $this->scopeConfig->getValue(
                        'payment/' . self::CODE . '/api_key',
                        \Magento\Store\Model\ScopeInterface::SCOPE_STORE
                    ),
                    'sandboxMode' => $this->scopeConfig->isSetFlag(
                        'payment/' . self::CODE . '/sandbox',
                        \Magento\Store\Model\ScopeInterface::SCOPE_STORE
                    ),
                    'tokenizeUrl' => $this->urlBuilder->getUrl('vendorpayment/tokenize/process'),
                    'ccTypes' => ['VI', 'MC', 'AE', 'DI'],
                    'hasVerification' => true
                ]
            ]
        ];
    }
}
```

---

## Integration 3: Magento_Shipping

Shipping method integration provides delivery options during checkout.

### Shipping Carrier Integration

**Custom Carrier Implementation:**

```php
namespace Vendor\Shipping\Model\Carrier;

use Magento\Quote\Model\Quote\Address\RateRequest;
use Magento\Shipping\Model\Carrier\AbstractCarrier;
use Magento\Shipping\Model\Carrier\CarrierInterface;
use Magento\Shipping\Model\Rate\Result;

class CustomCarrier extends AbstractCarrier implements CarrierInterface
{
    protected $_code = 'customcarrier';

    public function __construct(
        private readonly \Magento\Shipping\Model\Rate\ResultFactory $rateResultFactory,
        private readonly \Magento\Quote\Model\Quote\Address\RateResult\MethodFactory $rateMethodFactory,
        \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig,
        \Magento\Quote\Model\Quote\Address\RateResult\ErrorFactory $rateErrorFactory,
        \Psr\Log\LoggerInterface $logger,
        array $data = []
    ) {
        parent::__construct($scopeConfig, $rateErrorFactory, $logger, $data);
    }

    /**
     * Collect shipping rates
     *
     * @param RateRequest $request
     * @return Result|bool
     */
    public function collectRates(RateRequest $request)
    {
        if (!$this->isActive()) {
            return false;
        }

        /** @var Result $result */
        $result = $this->rateResultFactory->create();

        // Calculate shipping rates
        $rates = $this->calculateRates($request);

        foreach ($rates as $rateData) {
            $method = $this->rateMethodFactory->create();
            $method->setCarrier($this->_code);
            $method->setCarrierTitle($this->getConfigData('title'));
            $method->setMethod($rateData['method']);
            $method->setMethodTitle($rateData['title']);
            $method->setPrice($rateData['price']);
            $method->setCost($rateData['cost']);

            $result->append($method);
        }

        return $result;
    }

    /**
     * Calculate shipping rates based on request
     *
     * @param RateRequest $request
     * @return array
     */
    private function calculateRates(RateRequest $request): array
    {
        $rates = [];

        // Get destination info
        $destCountry = $request->getDestCountryId();
        $destRegion = $request->getDestRegionId();
        $destPostcode = $request->getDestPostcode();
        $weight = $request->getPackageWeight();

        // Standard Shipping
        $rates[] = [
            'method' => 'standard',
            'title' => 'Standard Shipping (5-7 business days)',
            'price' => $this->calculateStandardRate($weight, $destCountry),
            'cost' => $this->calculateStandardCost($weight)
        ];

        // Express Shipping
        if ($this->isExpressAvailable($destCountry)) {
            $rates[] = [
                'method' => 'express',
                'title' => 'Express Shipping (2-3 business days)',
                'price' => $this->calculateExpressRate($weight, $destCountry),
                'cost' => $this->calculateExpressCost($weight)
            ];
        }

        // Next Day Shipping (if available for region)
        if ($this->isNextDayAvailable($destCountry, $destRegion)) {
            $rates[] = [
                'method' => 'nextday',
                'title' => 'Next Day Shipping',
                'price' => $this->calculateNextDayRate($weight, $destCountry),
                'cost' => $this->calculateNextDayCost($weight)
            ];
        }

        return $rates;
    }

    /**
     * Calculate standard shipping rate
     *
     * @param float $weight
     * @param string $country
     * @return float
     */
    private function calculateStandardRate(float $weight, string $country): float
    {
        $baseRate = $this->getConfigData('standard_base_rate') ?: 5.00;
        $perKgRate = $this->getConfigData('standard_per_kg_rate') ?: 2.00;

        $rate = $baseRate + ($weight * $perKgRate);

        // International surcharge
        if ($country !== 'US') {
            $rate += $this->getConfigData('international_surcharge') ?: 10.00;
        }

        return round($rate, 2);
    }

    /**
     * Check if express shipping is available for country
     *
     * @param string $country
     * @return bool
     */
    private function isExpressAvailable(string $country): bool
    {
        $availableCountries = explode(',', $this->getConfigData('express_countries') ?: 'US,CA');
        return in_array($country, $availableCountries);
    }

    /**
     * Check if next day shipping is available
     *
     * @param string $country
     * @param string $region
     * @return bool
     */
    private function isNextDayAvailable(string $country, string $region): bool
    {
        // Next day only available in US, specific states
        if ($country !== 'US') {
            return false;
        }

        $availableRegions = explode(',', $this->getConfigData('nextday_regions') ?: 'CA,NY,TX');
        return in_array($region, $availableRegions);
    }

    /**
     * Get allowed shipping methods
     *
     * @return array
     */
    public function getAllowedMethods(): array
    {
        return [
            'standard' => 'Standard Shipping',
            'express' => 'Express Shipping',
            'nextday' => 'Next Day Shipping'
        ];
    }
}
```

**Shipping Rate Modification Plugin:**

```php
namespace Vendor\Module\Plugin;

use Magento\Quote\Model\Quote\Address\RateRequest;
use Magento\Shipping\Model\Rate\Result;

class ShippingRatesModifier
{
    public function __construct(
        private readonly \Magento\Customer\Model\Session $customerSession,
        private readonly \Vendor\Module\Service\LoyaltyService $loyaltyService,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Apply discounts to shipping rates for loyalty members
     *
     * @param \Magento\Shipping\Model\Shipping $subject
     * @param Result $result
     * @param RateRequest $request
     * @return Result
     */
    public function afterCollectRates(
        \Magento\Shipping\Model\Shipping $subject,
        Result $result,
        RateRequest $request
    ): Result {

        if (!$this->customerSession->isLoggedIn()) {
            return $result;
        }

        $customerId = $this->customerSession->getCustomerId();
        $discountPercent = $this->loyaltyService->getShippingDiscount($customerId);

        if ($discountPercent > 0) {
            foreach ($result->getAllRates() as $rate) {
                $originalPrice = $rate->getPrice();
                $discountedPrice = $originalPrice * (1 - $discountPercent / 100);

                $rate->setPrice($discountedPrice);

                $this->logger->info('Applied loyalty shipping discount', [
                    'customer_id' => $customerId,
                    'carrier' => $rate->getCarrier(),
                    'method' => $rate->getMethod(),
                    'original_price' => $originalPrice,
                    'discounted_price' => $discountedPrice
                ]);
            }
        }

        return $result;
    }
}
```

---

## Integration 4: Magento_Sales

Integration with Sales module for order creation and management.

### Quote to Order Conversion

**Order Converter:**

```php
namespace Vendor\Module\Model\Convert;

use Magento\Quote\Model\Quote;
use Magento\Sales\Api\Data\OrderInterface;

class QuoteToOrder
{
    public function __construct(
        private readonly \Magento\Quote\Model\Quote\Address\ToOrder $addressToOrder,
        private readonly \Magento\Quote\Model\Quote\Address\ToOrderAddress $addressToOrderAddress,
        private readonly \Magento\Quote\Model\Quote\Item\ToOrderItem $itemToOrderItem,
        private readonly \Magento\Quote\Model\Quote\Payment\ToOrderPayment $paymentToOrderPayment,
        private readonly \Magento\Sales\Api\Data\OrderInterfaceFactory $orderFactory,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Convert quote to order
     *
     * @param Quote $quote
     * @return OrderInterface
     */
    public function convert(Quote $quote): OrderInterface
    {
        $this->logger->info('Converting quote to order', [
            'quote_id' => $quote->getId()
        ]);

        // Create order from billing address
        $order = $this->addressToOrder->convert($quote->getBillingAddress());

        // Set addresses
        $order->setBillingAddress(
            $this->addressToOrderAddress->convert($quote->getBillingAddress())
        );

        if (!$quote->isVirtual()) {
            $order->setShippingAddress(
                $this->addressToOrderAddress->convert($quote->getShippingAddress())
            );
        }

        // Set payment
        $order->setPayment(
            $this->paymentToOrderPayment->convert($quote->getPayment())
        );

        // Convert items
        foreach ($quote->getAllVisibleItems() as $quoteItem) {
            $orderItem = $this->itemToOrderItem->convert($quoteItem);
            $order->addItem($orderItem);

            // Convert child items
            if ($quoteItem->getHasChildren()) {
                foreach ($quoteItem->getChildren() as $childItem) {
                    $orderChildItem = $this->itemToOrderItem->convert($childItem);
                    $orderChildItem->setParentItem($orderItem);
                    $order->addItem($orderChildItem);
                }
            }
        }

        // Copy custom data
        $this->copyCustomData($quote, $order);

        return $order;
    }

    /**
     * Copy custom data from quote to order
     *
     * @param Quote $quote
     * @param OrderInterface $order
     * @return void
     */
    private function copyCustomData(Quote $quote, OrderInterface $order): void
    {
        $customFields = [
            'custom_fee',
            'loyalty_discount',
            'delivery_date',
            'gift_message'
        ];

        foreach ($customFields as $field) {
            if ($value = $quote->getData($field)) {
                $order->setData($field, $value);
            }
        }
    }
}
```

---

## Integration 5: Magento_Customer

Customer integration for logged-in checkout and address management.

### Customer Data Integration

**Customer Address Loader:**

```php
namespace Vendor\Module\Model\Checkout;

use Magento\Customer\Api\AddressRepositoryInterface;
use Magento\Customer\Api\Data\AddressInterface;
use Magento\Customer\Model\Session as CustomerSession;

class CustomerAddressProvider
{
    public function __construct(
        private readonly CustomerSession $customerSession,
        private readonly AddressRepositoryInterface $addressRepository,
        private readonly \Magento\Framework\Api\SearchCriteriaBuilder $searchCriteriaBuilder
    ) {}

    /**
     * Get customer addresses for checkout
     *
     * @return AddressInterface[]
     */
    public function getAddresses(): array
    {
        if (!$this->customerSession->isLoggedIn()) {
            return [];
        }

        $customerId = $this->customerSession->getCustomerId();

        try {
            $searchCriteria = $this->searchCriteriaBuilder
                ->addFilter('parent_id', $customerId)
                ->create();

            $addressSearchResult = $this->addressRepository->getList($searchCriteria);
            return $addressSearchResult->getItems();

        } catch (\Exception $e) {
            return [];
        }
    }

    /**
     * Get default shipping address
     *
     * @return AddressInterface|null
     */
    public function getDefaultShippingAddress(): ?AddressInterface
    {
        if (!$this->customerSession->isLoggedIn()) {
            return null;
        }

        $customer = $this->customerSession->getCustomerData();
        $defaultShippingId = $customer->getDefaultShipping();

        if (!$defaultShippingId) {
            return null;
        }

        try {
            return $this->addressRepository->getById($defaultShippingId);
        } catch (\Exception $e) {
            return null;
        }
    }

    /**
     * Get default billing address
     *
     * @return AddressInterface|null
     */
    public function getDefaultBillingAddress(): ?AddressInterface
    {
        if (!$this->customerSession->isLoggedIn()) {
            return null;
        }

        $customer = $this->customerSession->getCustomerData();
        $defaultBillingId = $customer->getDefaultBilling();

        if (!$defaultBillingId) {
            return null;
        }

        try {
            return $this->addressRepository->getById($defaultBillingId);
        } catch (\Exception $e) {
            return null;
        }
    }
}
```

---

## Integration 6: Magento_Tax

Tax calculation integration for accurate totals.

### Tax Calculation in Checkout

**Tax Observer:**

```php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

class ApplyCustomTaxRules implements ObserverInterface
{
    public function __construct(
        private readonly \Magento\Tax\Model\Calculation $taxCalculation,
        private readonly \Magento\Customer\Model\Session $customerSession,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Apply custom tax rules after tax collection
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        // Example: Apply tax exemption for specific customer groups
        if ($this->customerSession->isLoggedIn()) {
            $customerGroupId = $this->customerSession->getCustomerGroupId();

            if ($this->isTaxExemptGroup($customerGroupId)) {
                foreach ($quote->getAllAddresses() as $address) {
                    $address->setTaxAmount(0);
                    $address->setBaseTaxAmount(0);
                }

                $quote->setTaxAmount(0);
                $quote->setBaseTaxAmount(0);

                $this->logger->info('Tax exemption applied', [
                    'quote_id' => $quote->getId(),
                    'customer_group' => $customerGroupId
                ]);
            }
        }
    }

    /**
     * Check if customer group is tax exempt
     *
     * @param int $customerGroupId
     * @return bool
     */
    private function isTaxExemptGroup(int $customerGroupId): bool
    {
        // Tax exempt group IDs (e.g., wholesale, government)
        $exemptGroups = [4, 5, 6];
        return in_array($customerGroupId, $exemptGroups);
    }
}
```

---

## Integration 7: Magento_Inventory (MSI)

Multi-Source Inventory integration for stock management.

### Inventory Reservation During Checkout

**Inventory Check Plugin:**

```php
namespace Vendor\Module\Plugin;

use Magento\Quote\Api\CartManagementInterface;
use Magento\InventorySalesApi\Api\AreProductsSalableInterface;
use Magento\Framework\Exception\LocalizedException;

class ValidateInventoryBeforeOrder
{
    public function __construct(
        private readonly AreProductsSalableInterface $areProductsSalable,
        private readonly \Magento\Quote\Api\CartRepositoryInterface $cartRepository,
        private readonly \Magento\Store\Model\StoreManagerInterface $storeManager,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Validate inventory before placing order
     *
     * @param CartManagementInterface $subject
     * @param int $cartId
     * @return void
     * @throws LocalizedException
     */
    public function beforePlaceOrder(
        CartManagementInterface $subject,
        int $cartId
    ): void {

        $quote = $this->cartRepository->getActive($cartId);
        $stockId = $this->getStockIdForWebsite($quote->getStore()->getWebsiteId());

        $skus = [];
        $qtys = [];

        foreach ($quote->getAllVisibleItems() as $item) {
            $skus[] = $item->getSku();
            $qtys[$item->getSku()] = $item->getQty();
        }

        // Check if products are salable
        $salableResults = $this->areProductsSalable->execute($skus, $stockId);

        foreach ($salableResults as $salableResult) {
            if (!$salableResult->isSalable()) {
                throw new LocalizedException(
                    __('Product %1 is no longer available in requested quantity.', $salableResult->getSku())
                );
            }
        }

        $this->logger->info('Inventory validation passed', [
            'cart_id' => $cartId,
            'stock_id' => $stockId
        ]);
    }

    /**
     * Get stock ID for website
     *
     * @param int $websiteId
     * @return int
     */
    private function getStockIdForWebsite(int $websiteId): int
    {
        // Map website to stock (simplified)
        return 1; // Default stock
    }
}
```

---

## Assumptions

- **Target Platform**: Adobe Commerce & Magento Open Source 2.4.7+
- **PHP Version**: 8.1, 8.2, 8.3
- **Modules Enabled**: All standard commerce modules
- **MSI**: Multi-Source Inventory enabled for inventory management

## Why These Integrations

- **Modularity**: Each module handles specific domain logic
- **Reusability**: Service contracts allow multiple consumers
- **Extensibility**: Plugins and observers enable customization
- **Data Integrity**: Transactions ensure consistency across modules

## Security Impact

- **Payment Data**: Never log or expose sensitive payment information
- **Customer Data**: Validate permissions before accessing customer addresses
- **Inventory**: Prevent overselling with proper reservation mechanisms
- **Tax Calculation**: Ensure tax rules cannot be manipulated client-side

## Performance Impact

- **Totals Collection**: Triggered multiple times; optimize collectors
- **Shipping Rates**: Cache when possible to reduce API calls
- **Address Validation**: Use efficient queries for address lookups
- **Inventory Checks**: Implement proper indexing for salability checks

## Backward Compatibility

- **Service Contracts**: All integration points use stable APIs
- **Extension Attributes**: Use for adding data without BC breaks
- **Totals Collectors**: Register via `sales.xml` with proper sort order
- **Payment Methods**: Follow AbstractMethod pattern for compatibility

## Tests to Add

1. **Integration Tests**: Test module interactions with fixtures
2. **API Tests**: Test service contract integrations
3. **Unit Tests**: Mock dependencies for isolated testing
4. **MFTF Tests**: End-to-end checkout with integrations

## Documentation to Update

- **Integration Map**: Visual diagram of module interactions
- **API Documentation**: Service contract dependencies
- **Extension Guide**: How to extend integrations
- **Troubleshooting**: Common integration issues
