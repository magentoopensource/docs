---
title: "Magento_Quote Integrations"
module: "Magento_Quote"
doc_type: "integrations"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Quote Integrations

## Overview

The Magento_Quote module integrates deeply with core modules to provide complete shopping cart functionality. This document details integration points, data flows, dependencies, and best practices for extending quote behavior through module integrations.

**Target Version:** Magento 2.4.7+ | Adobe Commerce & Open Source
**PHP Version:** 8.2+

## Module Dependencies

### Required Dependencies

```xml
<!-- module.xml — Note: Magento_Quote has NO <sequence> element in 2.4.7. -->
<!-- Dependencies are declared only in composer.json, not in module.xml. -->
<module name="Magento_Quote"/>
```

> **Note:** The actual `module.xml` for Magento_Quote contains no `<sequence>` block.
> Module load-order dependencies are managed via Composer's `require` section, not XML sequence.

### Optional Dependencies

- **Magento_SalesRule** - Promotions and discounts
- **Magento_Payment** - Payment method integration
- **Magento_Shipping** - Shipping method integration
- **Magento_Checkout** - Checkout UI and flow
- **Magento_InventorySales** - MSI stock validation
- **Magento_GiftMessage** - Gift message support
- **Magento_Reward** - Reward points (Adobe Commerce)
- **Magento_CustomerBalance** - Store credit (Adobe Commerce)

## Integration: Magento_Catalog

### Product to Quote Item Conversion

When adding products to cart, Magento_Quote integrates with Magento_Catalog to:

1. Load product data
2. Validate product salability
3. Process product options (configurable, bundle, grouped)
4. Calculate prices
5. Apply catalog price rules

#### Product Type Integration

```php
<?php
declare(strict_types=1);

namespace Magento\Catalog\Model\Product\Type;

/**
 * Abstract product type class for quote integration
 */
abstract class AbstractType
{
    /**
     * Prepare product for cart
     *
     * Validates product, processes options, returns product(s) to add.
     *
     * @param \Magento\Framework\DataObject $buyRequest
     * @param \Magento\Catalog\Model\Product $product
     * @param string|null $processMode
     * @return array|string Array of products or error message
     */
    public function prepareForCartAdvanced(
        \Magento\Framework\DataObject $buyRequest,
        $product,
        $processMode = null
    );

    /**
     * Check if product is saleable
     *
     * @param \Magento\Catalog\Model\Product $product
     * @return bool
     */
    public function isSalable($product);

    /**
     * Get product options for quote item
     *
     * @param \Magento\Catalog\Model\Product $product
     * @return array
     */
    public function getOrderOptions($product);
}
```

#### Custom Product Type Example

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Product\Type;

use Magento\Catalog\Model\Product\Type\AbstractType;

/**
 * Custom product type with special cart behavior
 */
class Subscription extends AbstractType
{
    const TYPE_CODE = 'subscription';

    /**
     * Prepare subscription product for cart
     *
     * @param \Magento\Framework\DataObject $buyRequest
     * @param \Magento\Catalog\Model\Product $product
     * @param string|null $processMode
     * @return array|string
     */
    public function prepareForCartAdvanced(
        \Magento\Framework\DataObject $buyRequest,
        $product,
        $processMode = null
    ) {
        // Validate subscription term selected
        $term = $buyRequest->getData('subscription_term');
        if (!$term) {
            return __('Please select a subscription term.')->render();
        }

        // Validate term is valid option
        $validTerms = $product->getCustomAttribute('subscription_terms');
        if (!$validTerms || !in_array($term, explode(',', $validTerms->getValue()))) {
            return __('Invalid subscription term selected.')->render();
        }

        // Calculate subscription price
        $basePrice = $product->getPrice();
        $termDiscount = $this->getTermDiscount($term);
        $subscriptionPrice = $basePrice * (1 - $termDiscount);

        // Set custom price
        $product->setCustomPrice($subscriptionPrice);
        $product->setOriginalCustomPrice($subscriptionPrice);

        // Add subscription options
        $product->addCustomOption('subscription_term', $term);
        $product->addCustomOption('subscription_start_date', date('Y-m-d'));

        return parent::prepareForCartAdvanced($buyRequest, $product, $processMode);
    }

    /**
     * Get discount for subscription term
     *
     * @param string $term
     * @return float
     */
    private function getTermDiscount(string $term): float
    {
        return match($term) {
            '3_months' => 0.05,   // 5% discount
            '6_months' => 0.10,   // 10% discount
            '12_months' => 0.15,  // 15% discount
            default => 0.0
        };
    }

    /**
     * Check if subscription is saleable
     *
     * @param \Magento\Catalog\Model\Product $product
     * @return bool
     */
    public function isSalable($product): bool
    {
        // Check parent salability
        if (!parent::isSalable($product)) {
            return false;
        }

        // Check subscription-specific rules
        // Example: Check if subscription service is available
        return $this->subscriptionService->isAvailable();
    }
}
```

### Price Integration

Quote items retrieve prices from catalog product with catalog price rules applied.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Quote;

use Magento\Quote\Model\Quote\Item;

/**
 * Item price calculator with catalog rule integration
 */
class ItemPriceCalculator
{
    public function __construct(
        private readonly \Magento\Catalog\Model\Product\PriceModifier $priceModifier,
        private readonly \Magento\CatalogRule\Model\ResourceModel\RuleFactory $ruleFactory
    ) {}

    /**
     * Calculate final price for quote item
     *
     * @param Item $item
     * @return float
     */
    public function calculatePrice(Item $item): float
    {
        $product = $item->getProduct();
        $quote = $item->getQuote();

        // Get base price (regular, special, tier)
        $price = $product->getFinalPrice($item->getQty());

        // Apply catalog price rules
        $rulePrice = $this->ruleFactory->create()->getRulePrice(
            new \DateTime(),
            $quote->getStore()->getWebsiteId(),
            $quote->getCustomerGroupId(),
            $product->getId()
        );

        if ($rulePrice !== null && $rulePrice < $price) {
            $price = $rulePrice;
        }

        return $price;
    }
}
```

## Integration: Magento_Customer

### Customer Data Association

Quote stores customer information for personalization, pricing, and order placement.

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Model;

use Magento\Customer\Api\Data\CustomerInterface;

/**
 * Quote customer association
 */
class Quote
{
    /**
     * Set customer data on quote
     *
     * @param CustomerInterface $customer
     * @return $this
     */
    public function setCustomer(CustomerInterface $customer): self
    {
        $this->setCustomerId($customer->getId());
        $this->setCustomerEmail($customer->getEmail());
        $this->setCustomerFirstname($customer->getFirstname());
        $this->setCustomerLastname($customer->getLastname());
        $this->setCustomerGroupId($customer->getGroupId());
        $this->setCustomerTaxClassId($customer->getTaxClassId());
        $this->setCustomerIsGuest(0);

        return $this;
    }

    /**
     * Load customer addresses to quote
     *
     * @param CustomerInterface $customer
     * @return $this
     */
    public function assignCustomerWithAddressChange(CustomerInterface $customer): self
    {
        $this->setCustomer($customer);

        // Load default billing address
        if ($customer->getDefaultBilling()) {
            $billingAddress = $this->customerAddressRepository->getById(
                $customer->getDefaultBilling()
            );
            $this->getBillingAddress()->importCustomerAddressData($billingAddress);
        }

        // Load default shipping address
        if ($customer->getDefaultShipping()) {
            $shippingAddress = $this->customerAddressRepository->getById(
                $customer->getDefaultShipping()
            );
            $this->getShippingAddress()->importCustomerAddressData($shippingAddress);
        }

        return $this;
    }
}
```

### Customer Group Pricing

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

/**
 * Apply customer group pricing when customer logs in
 */
class CustomerLoginQuoteObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Magento\Quote\Api\CartRepositoryInterface $quoteRepository
    ) {}

    /**
     * Recalculate quote prices with customer group
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        $customer = $observer->getEvent()->getCustomer();

        try {
            $quote = $this->quoteRepository->getForCustomer($customer->getId());

            // Set customer group for pricing
            $quote->setCustomerGroupId($customer->getGroupId());

            // Recalculate prices with customer group
            foreach ($quote->getAllItems() as $item) {
                $product = $item->getProduct();
                $product->setCustomerGroupId($customer->getGroupId());

                // Reload product price with customer group
                $finalPrice = $product->getPriceModel()->getFinalPrice(
                    $item->getQty(),
                    $product
                );
                $item->setPrice($finalPrice);
            }

            // Recalculate totals
            $quote->collectTotals();
            $this->quoteRepository->save($quote);

        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            // No quote for customer - skip
        }
    }
}
```

## Integration: Magento_Tax

### Tax Calculation in Totals

Tax collector integrates with tax calculation module to apply taxes to quote items and shipping.

```php
<?php
declare(strict_types=1);

namespace Magento\Tax\Model\Sales\Total\Quote;

use Magento\Quote\Model\Quote;
use Magento\Quote\Api\Data\ShippingAssignmentInterface;
use Magento\Quote\Model\Quote\Address\Total;

/**
 * Tax totals collector
 */
class Tax extends \Magento\Quote\Model\Quote\Address\Total\AbstractTotal
{
    protected string $_code = 'tax';

    public function __construct(
        private readonly \Magento\Tax\Model\Calculation $taxCalculation,
        private readonly \Magento\Tax\Api\TaxCalculationInterface $taxCalculationService,
        private readonly \Magento\Tax\Model\Config $taxConfig
    ) {}

    /**
     * Collect tax amounts for quote
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

        $address = $shippingAssignment->getShipping()->getAddress();
        $items = $shippingAssignment->getItems();

        if (!count($items)) {
            return $this;
        }

        // Get tax calculation basis (shipping origin or destination)
        $taxBasis = $this->taxConfig->getAlgorithm($quote->getStore());

        // Calculate tax for each item
        $itemTaxes = [];
        foreach ($items as $item) {
            if ($item->getParentItem()) {
                continue;
            }

            $taxDetails = $this->taxCalculationService->calculateTax(
                $this->getQuoteDetailsItemDataObject($item),
                $quote->getStoreId(),
                $quote->getCustomerId()
            );

            $itemTax = $taxDetails->getTaxAmount();
            $itemTaxes[$item->getId()] = $itemTax;

            // Set tax on item
            $item->setTaxAmount($itemTax);
            $item->setBaseTaxAmount($itemTax);
            $item->setTaxPercent($taxDetails->getTaxRate());
        }

        // Calculate shipping tax
        $shippingTax = 0;
        if ($address->getShippingAmount() > 0) {
            $shippingTaxDetails = $this->taxCalculationService->calculateShippingTax(
                $address,
                $quote->getStoreId()
            );
            $shippingTax = $shippingTaxDetails->getTaxAmount();
        }

        // Sum total tax
        $totalTax = array_sum($itemTaxes) + $shippingTax;

        // Set tax totals
        $total->setTaxAmount($totalTax);
        $total->setBaseTaxAmount($totalTax);
        $total->setShippingTaxAmount($shippingTax);
        $total->setBaseShippingTaxAmount($shippingTax);

        // Store applied taxes for display
        $total->setAppliedTaxes($this->getAppliedTaxes($itemTaxes, $shippingTax));

        return $this;
    }
}
```

### Tax Display Configuration

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Block\Cart;

/**
 * Cart totals display with tax configuration
 */
class Totals extends \Magento\Framework\View\Element\Template
{
    public function __construct(
        \Magento\Framework\View\Element\Template\Context $context,
        private readonly \Magento\Tax\Model\Config $taxConfig,
        private readonly \Magento\Checkout\Model\Session $checkoutSession,
        array $data = []
    ) {
        parent::__construct($context, $data);
    }

    /**
     * Get totals for display
     *
     * @return array
     */
    public function getTotals(): array
    {
        $quote = $this->checkoutSession->getQuote();
        $totals = [];

        // Subtotal
        $totals[] = [
            'code' => 'subtotal',
            'label' => __('Subtotal'),
            'value' => $this->getSubtotalDisplay($quote)
        ];

        // Shipping
        if (!$quote->isVirtual()) {
            $totals[] = [
                'code' => 'shipping',
                'label' => __('Shipping'),
                'value' => $this->getShippingDisplay($quote)
            ];
        }

        // Tax (display based on configuration)
        $totals[] = $this->getTaxDisplay($quote);

        // Discount
        if ($quote->getShippingAddress()->getDiscountAmount() != 0) {
            $totals[] = [
                'code' => 'discount',
                'label' => __('Discount (%1)', $quote->getCouponCode()),
                'value' => -abs($quote->getShippingAddress()->getDiscountAmount())
            ];
        }

        // Grand Total
        $totals[] = [
            'code' => 'grand_total',
            'label' => __('Grand Total'),
            'value' => $quote->getGrandTotal()
        ];

        return $totals;
    }

    /**
     * Get tax display based on configuration
     *
     * @param Quote $quote
     * @return array
     */
    private function getTaxDisplay(Quote $quote): array
    {
        $displayType = $this->taxConfig->getCartTaxDisplayType();

        if ($displayType == \Magento\Tax\Model\Config::DISPLAY_TYPE_BOTH) {
            // Display subtotal excluding and including tax
            return [
                'code' => 'tax',
                'label' => __('Tax'),
                'value' => $quote->getShippingAddress()->getTaxAmount(),
                'full_info' => [
                    'excl_tax' => $quote->getSubtotal(),
                    'incl_tax' => $quote->getSubtotal() + $quote->getShippingAddress()->getTaxAmount()
                ]
            ];
        }

        return [
            'code' => 'tax',
            'label' => __('Tax'),
            'value' => $quote->getShippingAddress()->getTaxAmount()
        ];
    }
}
```

## Integration: Magento_SalesRule

### Coupon and Promotion Application

SalesRule discount collector applies promotions to quote during totals collection.

```php
<?php
declare(strict_types=1);

namespace Magento\SalesRule\Model\Quote;

use Magento\Quote\Model\Quote;
use Magento\Quote\Api\Data\ShippingAssignmentInterface;
use Magento\Quote\Model\Quote\Address\Total;

/**
 * Discount totals collector
 */
class Discount extends \Magento\Quote\Model\Quote\Address\Total\AbstractTotal
{
    protected string $_code = 'discount';

    public function __construct(
        private readonly \Magento\SalesRule\Model\Validator $validator,
        private readonly \Magento\SalesRule\Model\RulesApplier $rulesApplier
    ) {}

    /**
     * Collect discount totals
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

        $address = $shippingAssignment->getShipping()->getAddress();
        $items = $shippingAssignment->getItems();

        if (!count($items)) {
            return $this;
        }

        // Initialize validator with quote context
        $this->validator->init(
            $quote->getStore()->getWebsiteId(),
            $quote->getCustomerGroupId(),
            $quote->getCouponCode()
        );

        // Get applicable rules
        $applicableRules = $this->validator->getApplicableRules($items);

        // Apply rules to items
        $discountAmount = 0;
        $baseDiscountAmount = 0;
        $appliedRuleIds = [];

        foreach ($items as $item) {
            if ($item->getParentItem()) {
                continue;
            }

            // Apply all applicable rules to item
            $itemDiscount = $this->rulesApplier->applyRules(
                $item,
                $applicableRules,
                $address
            );

            $discountAmount += $itemDiscount->getDiscountAmount();
            $baseDiscountAmount += $itemDiscount->getBaseDiscountAmount();
            $appliedRuleIds = array_merge($appliedRuleIds, $itemDiscount->getAppliedRuleIds());
        }

        // Set discount totals
        $total->setDiscountAmount(-abs($discountAmount));
        $total->setBaseDiscountAmount(-abs($baseDiscountAmount));
        $total->setDiscountDescription($this->getDiscountDescription($quote, $appliedRuleIds));

        return $this;
    }

    /**
     * Get discount description for display
     *
     * @param Quote $quote
     * @param array $ruleIds
     * @return string
     */
    private function getDiscountDescription(Quote $quote, array $ruleIds): string
    {
        if ($quote->getCouponCode()) {
            return $quote->getCouponCode();
        }

        return implode(', ', $ruleIds);
    }
}
```

### Custom Promotion Rule

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\SalesRule;

use Magento\Quote\Model\Quote\Item\AbstractItem;

/**
 * Custom sales rule condition for subscription products
 */
class SubscriptionCondition extends \Magento\Rule\Model\Condition\AbstractCondition
{
    /**
     * Validate if item is subscription product
     *
     * @param AbstractItem $item
     * @return bool
     */
    public function validate(\Magento\Framework\Model\AbstractModel $item): bool
    {
        $product = $item->getProduct();
        return $product->getTypeId() === 'subscription';
    }

    /**
     * Get input type for admin UI
     *
     * @return string
     */
    public function getInputType(): string
    {
        return 'select';
    }

    /**
     * Get value element type
     *
     * @return string
     */
    public function getValueElementType(): string
    {
        return 'select';
    }
}
```

## Integration: Magento_Shipping

### Shipping Rate Collection

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Model\Quote\Address;

use Magento\Quote\Model\Quote\Address;

/**
 * Shipping rate collector
 */
class RateCollector
{
    public function __construct(
        private readonly \Magento\Shipping\Model\Config $shippingConfig,
        private readonly \Magento\Shipping\Model\CarrierFactory $carrierFactory
    ) {}

    /**
     * Collect shipping rates for address
     *
     * @param Address $address
     * @return $this
     */
    public function collectRates(Address $address): self
    {
        $request = $this->createRateRequest($address);

        // Get active carriers
        $carriers = $this->shippingConfig->getActiveCarriers(
            $address->getQuote()->getStoreId()
        );

        $allRates = [];

        foreach ($carriers as $carrierCode => $carrierConfig) {
            $carrier = $this->carrierFactory->get($carrierCode);

            if (!$carrier) {
                continue;
            }

            // Check if carrier is available for this quote
            if (!$carrier->isActive()) {
                continue;
            }

            // Collect rates from carrier
            $result = $carrier->collectRates($request);

            if ($result && $result->getAllRates()) {
                $allRates = array_merge($allRates, $result->getAllRates());
            }
        }

        // Save rates to address
        $address->removeAllShippingRates();
        foreach ($allRates as $rate) {
            $address->addShippingRate($rate);
        }

        return $this;
    }

    /**
     * Create rate request from address
     *
     * @param Address $address
     * @return \Magento\Quote\Model\Quote\Address\RateRequest
     */
    private function createRateRequest(Address $address): \Magento\Quote\Model\Quote\Address\RateRequest
    {
        $request = new \Magento\Quote\Model\Quote\Address\RateRequest();

        $quote = $address->getQuote();

        $request->setAllItems($address->getAllItems());
        $request->setDestCountryId($address->getCountryId());
        $request->setDestRegionId($address->getRegionId());
        $request->setDestPostcode($address->getPostcode());
        $request->setDestCity($address->getCity());
        $request->setPackageValue($address->getBaseSubtotal());
        $request->setPackageValueWithDiscount($address->getBaseSubtotalWithDiscount());
        $request->setPackageWeight($address->getWeight());
        $request->setPackageQty($address->getItemQty());
        $request->setStoreId($quote->getStoreId());
        $request->setWebsiteId($quote->getStore()->getWebsiteId());
        $request->setBaseCurrency($quote->getStore()->getBaseCurrency());
        $request->setPackageCurrency($quote->getStore()->getCurrentCurrency());
        $request->setCustomerGroupId($quote->getCustomerGroupId());

        return $request;
    }
}
```

## Integration: Magento_Payment

### Payment Method Availability

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Model;

use Magento\Quote\Api\PaymentMethodManagementInterface;

/**
 * Payment method management for quotes
 */
class PaymentMethodManagement implements PaymentMethodManagementInterface
{
    public function __construct(
        \Magento\Quote\Api\CartRepositoryInterface $quoteRepository,
        \Magento\Payment\Model\Checks\ZeroTotal $zeroTotalValidator,
        \Magento\Payment\Model\MethodList $methodList
    ) {
        $this->quoteRepository = $quoteRepository;
        $this->zeroTotalValidator = $zeroTotalValidator;
        $this->methodList = $methodList;
    }

    /**
     * Get available payment methods for quote
     *
     * @param int $cartId
     * @return \Magento\Quote\Api\Data\PaymentMethodInterface[]
     */
    public function getList(int $cartId): array
    {
        $quote = $this->quoteRepository->get($cartId);

        $availableMethods = [];
        $methods = $this->paymentConfig->getActiveMethods();

        foreach ($methods as $methodCode => $methodConfig) {
            $method = $this->methodFactory->create($methodCode);

            // Check if method is available for this quote
            if ($method->isAvailable($quote)) {
                $availableMethods[] = $method;
            }
        }

        return $availableMethods;
    }

    /**
     * Set payment method on quote
     *
     * @param int $cartId
     * @param \Magento\Quote\Api\Data\PaymentInterface $method
     * @return int Quote ID
     */
    public function set(int $cartId, \Magento\Quote\Api\Data\PaymentInterface $method): int
    {
        $quote = $this->quoteRepository->get($cartId);

        // Validate payment method is available
        $paymentMethod = $this->methodFactory->create($method->getMethod());
        if (!$paymentMethod->isAvailable($quote)) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Payment method is not available.')
            );
        }

        // Set payment on quote
        $quote->getPayment()->importData($method->getData());
        $this->quoteRepository->save($quote);

        return $quote->getId();
    }
}
```

## Integration: Magento_InventorySales (MSI)

### Stock Validation

```php
<?php
declare(strict_types=1);

namespace Magento\InventorySales\Plugin\Quote;

use Magento\Quote\Api\CartItemRepositoryInterface;
use Magento\Quote\Api\Data\CartItemInterface;
use Magento\Framework\Exception\LocalizedException;

/**
 * Validate stock availability for quote items (MSI)
 */
class ValidateStockExtend
{
    public function __construct(
        private readonly \Magento\InventorySalesApi\Api\IsProductSalableInterface $isProductSalable,
        private readonly \Magento\InventorySalesApi\Api\GetProductSalableQtyInterface $getProductSalableQty,
        private readonly \Magento\InventoryCatalogApi\Api\DefaultStockProviderInterface $defaultStockProvider
    ) {}

    /**
     * Validate stock before adding item to cart
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
        $sku = $cartItem->getSku();
        $qty = $cartItem->getQty();
        $stockId = $this->defaultStockProvider->getId();

        // Check if product is salable
        if (!$this->isProductSalable->execute($sku, $stockId)) {
            throw new LocalizedException(
                __('Product "%1" is out of stock.', $cartItem->getName())
            );
        }

        // Check if requested quantity is available
        $salableQty = $this->getProductSalableQty->execute($sku, $stockId);
        if ($qty > $salableQty) {
            throw new LocalizedException(
                __(
                    'The requested qty of "%1" is not available. Available: %2',
                    $cartItem->getName(),
                    $salableQty
                )
            );
        }

        return [$cartItem];
    }
}
```

## Integration: Magento_Sales (Order Conversion)

### Quote to Order Converter

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Model\Quote;

use Magento\Quote\Model\Quote;
use Magento\Sales\Api\Data\OrderInterface;

/**
 * Convert quote to order
 */
class ToOrderConverter
{
    public function __construct(
        private readonly \Magento\Sales\Api\Data\OrderInterfaceFactory $orderFactory,
        private readonly \Magento\Quote\Model\Quote\Address\ToOrderAddress $addressConverter,
        private readonly \Magento\Quote\Model\Quote\Payment\ToOrderPayment $paymentConverter,
        private readonly \Magento\Quote\Model\Quote\Item\ToOrderItem $itemConverter
    ) {}

    /**
     * Convert quote to order
     *
     * @param Quote $quote
     * @return OrderInterface
     */
    public function convert(Quote $quote): OrderInterface
    {
        $order = $this->orderFactory->create();

        // Copy quote data to order
        $order->setQuoteId($quote->getId());
        $order->setStoreId($quote->getStoreId());
        $order->setCustomerId($quote->getCustomerId());
        $order->setCustomerEmail($quote->getCustomerEmail());
        $order->setCustomerFirstname($quote->getCustomerFirstname());
        $order->setCustomerLastname($quote->getCustomerLastname());
        $order->setCustomerGroupId($quote->getCustomerGroupId());
        $order->setCustomerIsGuest($quote->getCustomerIsGuest());

        // Copy totals
        $order->setSubtotal($quote->getSubtotal());
        $order->setBaseSubtotal($quote->getBaseSubtotal());
        $order->setGrandTotal($quote->getGrandTotal());
        $order->setBaseGrandTotal($quote->getBaseGrandTotal());

        // Convert addresses
        $billingAddress = $this->addressConverter->convert($quote->getBillingAddress());
        $order->setBillingAddress($billingAddress);

        if (!$quote->isVirtual()) {
            $shippingAddress = $this->addressConverter->convert($quote->getShippingAddress());
            $order->setShippingAddress($shippingAddress);

            // Copy shipping information
            $order->setShippingMethod($quote->getShippingAddress()->getShippingMethod());
            $order->setShippingDescription($quote->getShippingAddress()->getShippingDescription());
            $order->setShippingAmount($quote->getShippingAddress()->getShippingAmount());
            $order->setBaseShippingAmount($quote->getShippingAddress()->getBaseShippingAmount());
        }

        // Convert payment
        $payment = $this->paymentConverter->convert($quote->getPayment());
        $order->setPayment($payment);

        // Convert items
        foreach ($quote->getAllVisibleItems() as $quoteItem) {
            $orderItem = $this->itemConverter->convert($quoteItem);
            $order->addItem($orderItem);
        }

        return $order;
    }
}
```

---

**Assumptions:**
- Magento 2.4.7+ with PHP 8.2+
- Standard module configuration (all core modules enabled)
- MSI (Multi-Source Inventory) enabled for stock management
- Standard catalog, customer, tax configuration

**Why This Approach:**
Service contracts enable clean integration boundaries. Product type interface allows extensible cart behavior. Totals collectors provide modular integration with tax, shipping, discounts. Quote-to-order converter maintains data consistency.

**Security Impact:**
- Validate customer ownership when loading quote data
- Payment method availability checks prevent unauthorized method selection
- Stock validation prevents overselling
- Tax calculation uses server-side rules (never trust client)

**Performance Impact:**
- Tax calculation can be expensive (cache tax rates)
- Shipping rate collection queries multiple carriers (async where possible)
- Discount calculation iterates all sales rules (index rules by customer group)
- Quote-to-order conversion requires multiple database writes (use transaction)

**Backward Compatibility:**
Integration points via service contracts are stable. Product type interface is stable. Totals collector registration is upgrade-safe. Event names for integration are API contracts.

**Tests to Add:**
- Integration tests for each module integration
- API tests for cross-module workflows
- Unit tests for converter classes
- Functional tests for quote-to-order flow

**Docs to Update:**
- Integration architecture diagrams
- Module dependency matrix
- Extension point catalog
- API integration guide
