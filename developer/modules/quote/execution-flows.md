---
title: "Magento_Quote Execution Flows"
module: "Magento_Quote"
doc_type: "execution-flows"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Quote Execution Flows

## Overview

This document details the execution flows for critical Quote module operations, tracing the complete path from user action through controllers, services, models, and database persistence. Each flow includes sequence diagrams in text format, code examples, and performance considerations.

**Target Version:** Magento 2.4.7+ | Adobe Commerce & Open Source
**PHP Version:** 8.2+

## Flow 1: Add Product to Cart (Customer)

### User Story
A logged-in customer adds a simple product to their cart from the product detail page.

### Sequence Flow

```
Customer Browser
    ↓ POST /checkout/cart/add (product=123, qty=2)
Magento\Checkout\Controller\Cart\Add
    ↓ validate form key (CSRF protection)
FormKey\Validator::validate()
    ↓ validate product exists and is saleable
Magento\Catalog\Api\ProductRepositoryInterface::getById(123)
    ↓ get or create customer cart
Magento\Quote\Api\CartManagementInterface::getCartForCustomer($customerId)
    ├─> Check cache for active quote
    ├─> If not cached: QuoteRepository::getForCustomer()
    │       ├─> ResourceModel\Quote::loadByCustomerId()
    │       └─> Return Quote object
    └─> If no quote: Create new empty cart
            └─> Quote::setCustomer(), setStore(), setIsActive(1)
    ↓ add product to quote
Quote::addProduct($product, $buyRequest)
    ├─> Product\Type::prepareForCartAdvanced()
    │       ├─> Validate qty, stock, product status
    │       ├─> Process product options
    │       └─> Return prepared product(s)
    ├─> Quote::_addCatalogProduct()
    │       ├─> Check for existing item (merge or new)
    │       ├─> Quote\Item::setProduct(), setQty(), setSku()
    │       ├─> Quote\Item::addOption() for buy request
    │       └─> ItemsCollection::addItem()
    └─> Return Quote\Item or error string
    ↓ recalculate totals
Quote::collectTotals()
    ├─> TotalsCollector::collect()
    │       ├─> Subtotal collector (priority 100)
    │       ├─> Shipping collector (priority 200)
    │       ├─> Tax collector (priority 300)
    │       ├─> Discount collector (priority 400)
    │       └─> GrandTotal collector (priority 500)
    └─> Update Quote totals from Address totals
    ↓ persist quote
QuoteRepository::save($quote)
    ├─> Event: sales_quote_save_before
    ├─> ResourceModel\Quote::save()
    │       ├─> Save quote header (quote table)
    │       ├─> Save items (quote_item table)
    │       ├─> Save addresses (quote_address table)
    │       └─> Save payment (quote_payment table)
    ├─> Event: sales_quote_save_after
    └─> Clear repository cache
    ↓ update mini cart section
CustomerData\Cart::getSectionData()
    ├─> Reload quote
    ├─> Build cart summary
    └─> Return JSON (items, qty, subtotal)
    ↓ return response
HTTP 302 Redirect → /checkout/cart
Customer Browser
    ↓ displays cart with new item
```

### Implementation Example

```php
<?php
declare(strict_types=1);

namespace Magento\Checkout\Controller\Cart;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\Controller\Result\Redirect;

/**
 * Add product to cart controller
 */
class Add extends \Magento\Framework\App\Action\Action implements HttpPostActionInterface
{
    public function __construct(
        \Magento\Framework\App\Action\Context $context,
        private readonly \Magento\Quote\Api\CartManagementInterface $cartManagement,
        private readonly \Magento\Quote\Api\CartRepositoryInterface $cartRepository,
        private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
        private readonly \Magento\Framework\Data\Form\FormKey\Validator $formKeyValidator,
        private readonly \Magento\Customer\Model\Session $customerSession,
        private readonly \Magento\Framework\Message\ManagerInterface $messageManager,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {
        parent::__construct($context);
    }

    /**
     * Add product to cart action
     *
     * @return Redirect
     */
    public function execute(): Redirect
    {
        $resultRedirect = $this->resultRedirectFactory->create();

        // Validate CSRF token
        if (!$this->formKeyValidator->validate($this->getRequest())) {
            $this->messageManager->addErrorMessage(__('Invalid form key.'));
            return $resultRedirect->setPath('*/*/');
        }

        $productId = (int)$this->getRequest()->getParam('product');
        $qty = (float)($this->getRequest()->getParam('qty') ?: 1);

        if (!$productId) {
            $this->messageManager->addErrorMessage(__('Product not specified.'));
            return $resultRedirect->setPath('*/*/');
        }

        try {
            // Load product
            $product = $this->productRepository->getById($productId);

            // Get or create customer cart
            $customerId = (int)$this->customerSession->getCustomerId();
            $quote = $this->cartManagement->getCartForCustomer($customerId);

            // Prepare buy request
            $params = $this->getRequest()->getParams();
            $buyRequest = new \Magento\Framework\DataObject($params);

            // Add product to cart
            $item = $quote->addProduct($product, $buyRequest);

            // Check for error message (string return)
            if (is_string($item)) {
                throw new \Magento\Framework\Exception\LocalizedException(__($item));
            }

            // Collect totals and save
            $quote->collectTotals();
            $this->cartRepository->save($quote);

            // Success message
            $this->messageManager->addSuccessMessage(
                __('You added %1 to your shopping cart.', $product->getName())
            );

            // Dispatch event for observers
            $this->_eventManager->dispatch(
                'checkout_cart_add_product_complete',
                ['product' => $product, 'request' => $this->getRequest(), 'response' => $this->getResponse()]
            );

        } catch (\Magento\Framework\Exception\LocalizedException $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
            $this->logger->error('Add to cart error: ' . $e->getMessage());
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage(__('We cannot add this item to your cart.'));
            $this->logger->critical('Add to cart critical error: ' . $e->getMessage());
        }

        return $resultRedirect->setPath('checkout/cart');
    }
}
```

### Performance Considerations

- **Database Queries:** 3-5 queries (customer lookup, product load, quote load/save)
- **Cache Impact:** Invalidates customer section data (mini cart)
- **Indexer Impact:** None (quote operations don't trigger reindex)
- **Optimization:** Cache product data, use lazy loading for quote items

---

## Flow 2: Add Configurable Product to Cart

### User Story
Customer selects color and size options for a configurable product and adds it to cart.

### Sequence Flow

```
Customer Browser
    ↓ POST /checkout/cart/add (product=456, super_attribute[color]=23, super_attribute[size]=56, qty=1)
Controller\Cart\Add::execute()
    ↓ load configurable product
ProductRepository::getById(456)
    ↓ get customer cart
CartManagement::getCartForCustomer()
    ↓ prepare buy request with super attributes
DataObject(['product' => 456, 'super_attribute' => [color => 23, size => 56], 'qty' => 1])
    ↓ add product to quote
Quote::addProduct($configurableProduct, $buyRequest)
    ↓ prepare for cart (configurable-specific)
Configurable\Product\Type::prepareForCartAdvanced()
    ├─> Validate super attributes provided
    ├─> Load simple product based on attributes
    │       └─> getProductByAttributes([color => 23, size => 56])
    ├─> Check simple product stock, status
    ├─> Return [configurable product, simple product]
    └─> Set parent-child relationship
    ↓ add configurable product as parent item
Quote::_addCatalogProduct($configurableProduct)
    ├─> Create Quote\Item for configurable
    ├─> Set sku, name, price from configurable
    ├─> Add option: info_buyRequest (serialized)
    ├─> Add option: product_type = configurable
    └─> Add to items collection
    ↓ add simple product as child item
Quote::_addCatalogProduct($simpleProduct)
    ├─> Create Quote\Item for simple
    ├─> Set parentItem to configurable item
    ├─> Set sku, price from simple
    ├─> Add option: simple_product
    └─> Add to parent's children collection
    ↓ collect totals (uses simple product price)
Quote::collectTotals()
    ↓ save quote with parent-child items
QuoteRepository::save()
```

### Implementation Deep Dive

```php
<?php
declare(strict_types=1);

namespace Magento\ConfigurableProduct\Model\Product\Type;

use Magento\Catalog\Model\Product;
use Magento\Framework\DataObject;

/**
 * Configurable product type
 */
class Configurable extends \Magento\Catalog\Model\Product\Type\AbstractType
{
    /**
     * Prepare product for cart
     *
     * @param DataObject $buyRequest
     * @param Product $product
     * @param string|null $processMode
     * @return array|string Array of products or error message
     */
    public function prepareForCartAdvanced(DataObject $buyRequest, $product, $processMode = null)
    {
        $superAttributes = $buyRequest->getData('super_attribute');

        // Validate super attributes provided
        if (!is_array($superAttributes) || empty($superAttributes)) {
            return __('Please specify the product\'s required option(s).')->render();
        }

        // Get simple product by attributes
        $simpleProduct = $this->getProductByAttributes($superAttributes, $product);

        if (!$simpleProduct || !$simpleProduct->getId()) {
            return __('The required options you selected are not available.')->render();
        }

        // Check simple product is saleable
        if (!$simpleProduct->isSaleable()) {
            return __('This product is currently out of stock.')->render();
        }

        // Prepare configurable product
        $product->setCustomOptions([]);
        $this->_prepareProduct($buyRequest, $product, $processMode);

        // Prepare simple product
        $this->_prepareProduct($buyRequest, $simpleProduct, $processMode);

        // Add attributes option to configurable
        $product->addCustomOption('attributes', serialize($superAttributes));
        $product->addCustomOption('product_qty_' . $simpleProduct->getId(), $buyRequest->getQty());
        $product->addCustomOption('simple_product', $simpleProduct->getId(), $simpleProduct);

        // Add parent product option to simple
        $simpleProduct->addCustomOption('parent_product_id', $product->getId());

        // Set simple product as child
        $product->addCustomOption('child_product', $simpleProduct->getId(), $simpleProduct);

        // Return both products (parent, child)
        $products = [$product, $simpleProduct];

        return $products;
    }

    /**
     * Get simple product by selected attributes
     *
     * @param array $attributes
     * @param Product $product
     * @return Product|null
     */
    private function getProductByAttributes(array $attributes, Product $product): ?Product
    {
        $collection = $this->getUsedProductCollection($product)
            ->addAttributeToSelect('*')
            ->setFlag('has_stock_status_filter', true);

        foreach ($attributes as $attributeId => $value) {
            $collection->addAttributeToFilter($attributeId, $value);
        }

        return $collection->getFirstItem();
    }
}
```

### Database Operations

```sql
-- Load configurable product
SELECT * FROM catalog_product_entity WHERE entity_id = 456;
SELECT * FROM catalog_product_entity_varchar WHERE entity_id = 456; -- attributes

-- Load simple product by attributes
SELECT cpe.* FROM catalog_product_entity cpe
INNER JOIN catalog_product_super_link cpsl ON cpsl.product_id = cpe.entity_id
INNER JOIN catalog_product_entity_int cpei_color ON cpei_color.entity_id = cpe.entity_id
INNER JOIN catalog_product_entity_int cpei_size ON cpei_size.entity_id = cpe.entity_id
WHERE cpsl.parent_id = 456
  AND cpei_color.attribute_id = 93 AND cpei_color.value = 23
  AND cpei_size.attribute_id = 144 AND cpei_size.value = 56;

-- Save quote items (parent and child)
INSERT INTO quote_item (quote_id, product_id, product_type, sku, name, qty, price, parent_item_id)
VALUES (789, 456, 'configurable', 'CONFIG-SKU', 'Configurable Product', 1, 59.99, NULL);

INSERT INTO quote_item (quote_id, product_id, product_type, sku, name, qty, price, parent_item_id)
VALUES (789, 457, 'simple', 'SIMPLE-SKU-RED-L', 'Configurable Product-Red-Large', 1, 59.99, 1001);

-- Save item options
INSERT INTO quote_item_option (item_id, product_id, code, value)
VALUES (1001, 456, 'info_buyRequest', '{"product":456,"super_attribute":{"color":23,"size":56},"qty":1}');

INSERT INTO quote_item_option (item_id, product_id, code, value)
VALUES (1001, 456, 'attributes', 'a:2:{s:5:"color";i:23;s:4:"size";i:56;}');

INSERT INTO quote_item_option (item_id, product_id, code, value)
VALUES (1001, 456, 'simple_product', '457');
```

---

## Flow 3: Update Cart Item Quantity

### User Story
Customer changes quantity of an item in cart from 2 to 5.

### Sequence Flow

```
Customer Browser
    ↓ POST /checkout/cart/updatePost (cart[item_id]=1001, cart[item_id][qty]=5)
Controller\Cart\UpdatePost::execute()
    ↓ validate form key
FormKeyValidator::validate()
    ↓ get customer cart
CartRepository::getForCustomer($customerId)
    ↓ parse cart data
$cartData = $request->getParam('cart')
    ↓ iterate cart data
foreach ($cartData as $itemId => $itemInfo)
    ↓ get item from quote
Quote::getItemById($itemId)
    ├─> Validate item exists
    └─> Validate item belongs to this quote
    ↓ check quantity
if ($itemInfo['qty'] > 0)
    ├─> Update quantity
    │   Quote\Item::setQty($itemInfo['qty'])
    └─> Trigger item qty update
        Quote\Item::calcRowTotal()
else
    └─> Remove item
        Quote::removeItem($itemId)
    ↓ recalculate totals
Quote::collectTotals()
    ├─> Reset all totals to zero
    ├─> Execute subtotal collector
    │   Subtotal::collect()
    │       ├─> Iterate items
    │       ├─> Item::calcRowTotal() [price * qty]
    │       ├─> Sum row totals
    │       └─> Set address subtotal
    ├─> Execute shipping collector
    ├─> Execute tax collector
    ├─> Execute discount collector
    └─> Execute grand total collector
    ↓ save updated quote
QuoteRepository::save($quote)
    ├─> Event: sales_quote_save_before
    ├─> UPDATE quote SET items_count=X, items_qty=Y, subtotal=Z
    ├─> UPDATE quote_item SET qty=5, row_total=price*5
    ├─> Event: sales_quote_save_after
    └─> Clear cache
    ↓ return success
MessageManager::addSuccessMessage('Cart updated.')
Redirect → /checkout/cart
```

### Implementation

```php
<?php
declare(strict_types=1);

namespace Magento\Checkout\Controller\Cart;

use Magento\Framework\App\Action\HttpPostActionInterface;

class UpdatePost extends \Magento\Framework\App\Action\Action implements HttpPostActionInterface
{
    public function __construct(
        \Magento\Framework\App\Action\Context $context,
        private readonly \Magento\Quote\Api\CartRepositoryInterface $cartRepository,
        private readonly \Magento\Customer\Model\Session $customerSession,
        private readonly \Magento\Framework\Data\Form\FormKey\Validator $formKeyValidator,
        private readonly \Magento\Framework\Message\ManagerInterface $messageManager
    ) {
        parent::__construct($context);
    }

    /**
     * Update cart quantities
     *
     * @return \Magento\Framework\Controller\Result\Redirect
     */
    public function execute()
    {
        $resultRedirect = $this->resultRedirectFactory->create();

        if (!$this->formKeyValidator->validate($this->getRequest())) {
            return $resultRedirect->setPath('*/*/');
        }

        $cartData = $this->getRequest()->getParam('cart');
        if (!is_array($cartData)) {
            return $resultRedirect->setPath('*/*/');
        }

        try {
            $customerId = (int)$this->customerSession->getCustomerId();
            $quote = $this->cartRepository->getForCustomer($customerId);

            $updateAction = false;

            foreach ($cartData as $itemId => $itemInfo) {
                $item = $quote->getItemById($itemId);
                if (!$item) {
                    continue;
                }

                $qty = isset($itemInfo['qty']) ? (float)$itemInfo['qty'] : 0;

                if ($qty > 0) {
                    // Update quantity
                    $item->setQty($qty);
                    $updateAction = true;
                } else {
                    // Remove item
                    $quote->removeItem($itemId);
                    $updateAction = true;
                }
            }

            if ($updateAction) {
                // Recalculate totals
                $quote->collectTotals();

                // Save quote
                $this->cartRepository->save($quote);

                $this->messageManager->addSuccessMessage(__('Cart updated.'));

                // Dispatch event
                $this->_eventManager->dispatch(
                    'checkout_cart_update_items_after',
                    ['cart' => $quote, 'info' => $cartData]
                );
            }

        } catch (\Magento\Framework\Exception\LocalizedException $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage(__('We cannot update the shopping cart.'));
        }

        return $resultRedirect->setPath('*/*/');
    }
}
```

---

## Flow 4: Apply Coupon Code

### User Story
Customer enters a coupon code "SAVE10" in the cart to receive a 10% discount.

### Sequence Flow

```
Customer Browser
    ↓ POST /checkout/cart/couponPost (coupon_code=SAVE10)
Controller\Cart\CouponPost::execute()
    ↓ get customer cart
CartRepository::getForCustomer()
    ↓ apply coupon
CouponManagement::set($cartId, 'SAVE10')
    ├─> Validate coupon exists
    │   SalesRule\Model\Coupon::loadByCode('SAVE10')
    ├─> Validate rule is active
    │   SalesRule\Model\Rule::load($coupon->getRuleId())
    ├─> Validate date range
    ├─> Validate usage limits
    └─> Set coupon code on quote
        Quote::setCouponCode('SAVE10')
    ↓ collect totals (applies discount)
Quote::collectTotals()
    ├─> Subtotal collector (before discount)
    │   Subtotal = $100.00
    ├─> Discount collector (applies coupon)
    │   Discount::collect()
    │       ├─> Load sales rules matching quote
    │       ├─> RuleProcessor::process()
    │       │   ├─> Validate rule conditions (cart subtotal, customer group, etc.)
    │       │   ├─> Calculate discount (10% of $100 = $10)
    │       │   ├─> Set item discounts
    │       │   └─> Update totals
    │       ├─> Total::setDiscountAmount(-10.00)
    │       └─> Total::setDiscountDescription('SAVE10')
    └─> Grand total collector
        GrandTotal = Subtotal + Shipping + Tax - Discount
                   = $100 + $5 + $8 - $10 = $103
    ↓ save quote
QuoteRepository::save()
    ├─> UPDATE quote SET coupon_code='SAVE10'
    ├─> UPDATE quote_address SET discount_amount=-10.00
    └─> Event: sales_quote_save_after
    ↓ verify coupon applied
if (Quote::getCouponCode() === 'SAVE10')
    MessageManager::addSuccessMessage('Coupon applied.')
else
    MessageManager::addErrorMessage('Coupon is invalid.')
```

### Implementation

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Model;

use Magento\Quote\Api\CouponManagementInterface;
use Magento\Framework\Exception\CouldNotSaveException;
use Magento\Framework\Exception\NoSuchEntityException;

class CouponManagement implements CouponManagementInterface
{
    public function __construct(
        private readonly CartRepositoryInterface $quoteRepository,
        private readonly \Magento\SalesRule\Model\CouponFactory $couponFactory,
        private readonly \Magento\SalesRule\Model\RuleRepository $ruleRepository
    ) {}

    /**
     * Set coupon code for cart
     *
     * @param int $cartId
     * @param string $couponCode
     * @return bool
     * @throws NoSuchEntityException
     * @throws CouldNotSaveException
     */
    public function set($cartId, $couponCode)
    {
        $quote = $this->quoteRepository->get($cartId);

        // Trim and uppercase coupon code
        $couponCode = trim($couponCode);

        if (empty($couponCode)) {
            throw new CouldNotSaveException(__('The coupon code is not valid.'));
        }

        try {
            // Validate coupon exists
            $coupon = $this->couponFactory->create();
            $coupon->loadByCode($couponCode);

            if (!$coupon->getId()) {
                throw new NoSuchEntityException(__('The coupon code "%1" is not valid.', $couponCode));
            }

            // Validate rule is active
            $rule = $this->ruleRepository->getById($coupon->getRuleId());

            if (!$rule->getIsActive()) {
                throw new CouldNotSaveException(__('The coupon code "%1" is not valid.', $couponCode));
            }

            // Validate usage limits
            if ($coupon->getUsageLimit() && $coupon->getTimesUsed() >= $coupon->getUsageLimit()) {
                throw new CouldNotSaveException(
                    __('The coupon code "%1" has reached its usage limit.', $couponCode)
                );
            }

            // Set coupon on quote
            $quote->setCouponCode($couponCode);

            // Collect totals to apply discount
            $quote->collectTotals();

            // Save quote
            $this->quoteRepository->save($quote);

            // Verify coupon was actually applied (rule conditions may have failed)
            if ($quote->getCouponCode() !== $couponCode) {
                throw new CouldNotSaveException(
                    __('The coupon code "%1" is not valid for this cart.', $couponCode)
                );
            }

            return true;

        } catch (NoSuchEntityException | CouldNotSaveException $e) {
            throw $e;
        } catch (\Exception $e) {
            throw new CouldNotSaveException(
                __('Could not apply coupon code: %1', $e->getMessage()),
                $e
            );
        }
    }

    /**
     * Remove coupon from cart
     *
     * @param int $cartId
     * @return bool
     */
    public function remove($cartId)
    {
        $quote = $this->quoteRepository->get($cartId);
        $quote->setCouponCode('');
        $quote->collectTotals();
        $this->quoteRepository->save($quote);

        return true;
    }
}
```

---

## Flow 5: Totals Calculation Deep Dive

### Collector Execution Order

```
Quote::collectTotals()
    ↓
TotalsCollector::collect($quote)
    ↓ for each address
    TotalsCollector::collectAddressTotals($quote, $address)
        ↓ create Total object (empty container)
        Total::__construct()
        ↓ execute collectors by priority
        ├─> [100] Subtotal::collect()
        │       ├─> Iterate items
        │       ├─> Item::calcRowTotal() [price * qty]
        │       ├─> Sum item totals
        │       ├─> Total::setSubtotal($sum)
        │       └─> Total::setBaseSubtotal($baseSum)
        │
        ├─> [200] Shipping::collect()
        │       ├─> Get selected shipping method
        │       ├─> ShippingMethodManagement::getRate()
        │       ├─> Total::setShippingAmount($rate)
        │       └─> Total::setBaseShippingAmount($baseRate)
        │
        ├─> [300] Tax::collect()
        │       ├─> TaxCalculation::calcTaxAmount()
        │       ├─> Apply tax to items
        │       ├─> Apply tax to shipping
        │       ├─> Total::setTaxAmount($tax)
        │       └─> Total::setBaseTaxAmount($baseTax)
        │
        ├─> [400] Discount::collect()
        │       ├─> Load applicable sales rules
        │       ├─> RuleProcessor::process($items, $rules)
        │       ├─> Calculate discount per item
        │       ├─> Total::setDiscountAmount(-$discount)
        │       └─> Total::setDiscountDescription($codes)
        │
        └─> [500] GrandTotal::collect()
                ├─> Sum all total amounts
                ├─> Grand = Subtotal + Shipping + Tax - Discount
                ├─> Total::setGrandTotal($grand)
                └─> Total::setBaseGrandTotal($baseGrand)
        ↓ update address with totals
        Address::addData($total->getData())
        ├─> Address::setSubtotal()
        ├─> Address::setShippingAmount()
        ├─> Address::setTaxAmount()
        ├─> Address::setDiscountAmount()
        └─> Address::setGrandTotal()
```

### Custom Totals Collector Example

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Quote\Address\Total;

use Magento\Quote\Model\Quote;
use Magento\Quote\Api\Data\ShippingAssignmentInterface;
use Magento\Quote\Model\Quote\Address\Total;
use Magento\Quote\Model\Quote\Address\Total\AbstractTotal;

/**
 * Handling fee collector
 *
 * Adds a $2.99 handling fee to all orders.
 * Executes after subtotal, before tax.
 */
class HandlingFee extends AbstractTotal
{
    /**
     * Collector code
     *
     * @var string
     */
    protected string $_code = 'handling_fee';

    /**
     * Handling fee amount
     */
    private const FEE_AMOUNT = 2.99;

    /**
     * Collect handling fee
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

        // Only apply to shipping address
        if ($address->getAddressType() !== \Magento\Quote\Model\Quote\Address::ADDRESS_TYPE_SHIPPING) {
            return $this;
        }

        // Skip if no items
        $items = $shippingAssignment->getItems();
        if (!count($items)) {
            return $this;
        }

        $fee = self::FEE_AMOUNT;
        $baseFee = $fee; // Assume same currency

        // Add fee to totals
        $total->setTotalAmount($this->getCode(), $fee);
        $total->setBaseTotalAmount($this->getCode(), $baseFee);
        $total->setHandlingFeeAmount($fee);
        $total->setBaseHandlingFeeAmount($baseFee);

        // Add to grand total
        $total->setGrandTotal($total->getGrandTotal() + $fee);
        $total->setBaseGrandTotal($total->getBaseGrandTotal() + $baseFee);

        return $this;
    }

    /**
     * Fetch handling fee for display
     *
     * @param Quote $quote
     * @param Total $total
     * @return array
     */
    public function fetch(Quote $quote, Total $total): array
    {
        $fee = $total->getHandlingFeeAmount();

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
}
```

Register collector in `etc/sales.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Sales:etc/sales.xsd">
    <section name="quote">
        <group name="totals">
            <!-- sort_order determines execution order (after subtotal=100, before tax=300) -->
            <item name="handling_fee"
                  instance="Vendor\Module\Model\Quote\Address\Total\HandlingFee"
                  sort_order="250"/>
        </group>
    </section>
</config>
```

---

## Flow 6: Guest Cart to Customer Cart Merge

### User Story
A guest adds items to cart, then logs in. Guest cart merges with existing customer cart.

### Sequence Flow

```
Guest Session
    ↓ add items to cart
    GuestCartManagement::createEmptyCart()
        ├─> Create new quote (customer_id=NULL)
        ├─> QuoteIdMask::createMaskedId($quoteId)
        └─> Return masked ID (UUID)
    ↓ add products (masked ID: abc123)
    GuestCartRepository::get('abc123')
        └─> QuoteIdMask::getQuoteIdByMaskedId('abc123') → quoteId=999
    Quote(999)::addProduct($product1, qty=2)
    Quote(999)::addProduct($product2, qty=1)
    Guest cart now has 2 items
Customer Login
    ↓ POST /customer/account/loginPost
    Customer\Account\LoginPost::execute()
        ├─> Authenticate customer
        ├─> Set customer session
        └─> Dispatch: customer_login
Observer\AfterCustomerLogin::execute()
    ↓ get guest cart ID from cookie
    $guestQuoteId = Cookie::get('guest-cart')
    ↓ check if customer has existing cart
    try {
        $customerQuote = QuoteRepository::getForCustomer($customerId)
    } catch (NoSuchEntityException) {
        $customerQuote = null
    }
    ↓ load guest quote
    $guestQuote = QuoteRepository::get($guestQuoteId)
    ↓ if customer has cart: merge
    if ($customerQuote && $customerQuote->getItemsCount() > 0)
        CustomerQuote::merge($guestQuote)
            ├─> Event: sales_quote_merge_before
            ├─> Iterate guest quote items
            │   foreach ($guestQuote->getAllVisibleItems() as $guestItem)
            │       ├─> Check if item exists in customer cart
            │       │   foreach ($customerQuote->getAllItems() as $customerItem)
            │       │       if ($customerItem->compare($guestItem))
            │       │           ├─> Merge quantities
            │       │           └─> $customerItem->setQty($customerItem->getQty() + $guestItem->getQty())
            │       └─> If not found: add as new item
            │           $newItem = clone $guestItem
            │           $customerQuote->addItem($newItem)
            ├─> Merge coupon code
            │   if (!$customerQuote->getCouponCode() && $guestQuote->getCouponCode())
            │       $customerQuote->setCouponCode($guestQuote->getCouponCode())
            └─> Event: sales_quote_merge_after
    else
        ↓ no customer cart: assign guest cart to customer
        $guestQuote->setCustomer($customer)
        $guestQuote->setCustomerId($customerId)
        $guestQuote->setCustomerEmail($customer->getEmail())
        $guestQuote->setCustomerGroupId($customer->getGroupId())
    ↓ collect totals and save
    $customerQuote->collectTotals()
    QuoteRepository::save($customerQuote)
    ↓ delete guest quote
    QuoteRepository::delete($guestQuote)
    ↓ delete masked ID
    QuoteIdMask::delete($guestQuoteId)
Customer now has merged cart
```

### Implementation

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

/**
 * Merge guest cart into customer cart on login
 */
class AfterCustomerLogin implements ObserverInterface
{
    public function __construct(
        private readonly \Magento\Quote\Api\CartRepositoryInterface $quoteRepository,
        private readonly \Magento\Checkout\Model\Session $checkoutSession,
        private readonly \Magento\Quote\Model\QuoteIdMaskFactory $quoteIdMaskFactory,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Execute merge logic
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        $customer = $observer->getEvent()->getCustomer();

        try {
            // Get guest quote ID
            $guestQuoteId = $this->checkoutSession->getQuoteId();
            if (!$guestQuoteId) {
                return;
            }

            $guestQuote = $this->quoteRepository->get($guestQuoteId);

            // Skip if guest quote is empty
            if (!$guestQuote->getItemsCount()) {
                return;
            }

            // Check if customer has existing active quote
            try {
                $customerQuote = $this->quoteRepository->getForCustomer($customer->getId());
            } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
                // No existing customer quote - assign guest quote to customer
                $guestQuote->setCustomer($customer);
                $guestQuote->setCustomerId($customer->getId());
                $guestQuote->setCustomerEmail($customer->getEmail());
                $guestQuote->setCustomerGroupId($customer->getGroupId());
                $guestQuote->setCustomerIsGuest(0);

                $guestQuote->collectTotals();
                $this->quoteRepository->save($guestQuote);

                $this->checkoutSession->setQuoteId($guestQuote->getId());
                return;
            }

            // Customer has existing quote - merge
            if ($customerQuote->getId() && $customerQuote->getItemsCount()) {
                $customerQuote->merge($guestQuote);
                $customerQuote->collectTotals();
                $this->quoteRepository->save($customerQuote);

                // Delete guest quote
                $this->quoteRepository->delete($guestQuote);

                $this->checkoutSession->setQuoteId($customerQuote->getId());
            } else {
                // Customer quote is empty - use guest quote
                $guestQuote->setCustomer($customer);
                $guestQuote->collectTotals();
                $this->quoteRepository->save($guestQuote);

                $this->checkoutSession->setQuoteId($guestQuote->getId());
            }

        } catch (\Exception $e) {
            $this->logger->error('Error merging guest cart: ' . $e->getMessage());
        }
    }
}
```

---

## Flow 7: Quote to Order Conversion

### User Story
Customer completes checkout and places order. Quote converts to order.

### Sequence Flow

```
Customer Browser
    ↓ POST /checkout/onepage/saveOrder
Checkout\Controller\Onepage\SaveOrder::execute()
    ↓ validate quote
QuoteValidator::validateBeforeSubmit($quote)
    ├─> Check quote has items
    ├─> Check billing address complete
    ├─> Check shipping address (if not virtual)
    ├─> Check shipping method selected
    ├─> Check payment method selected
    └─> Validate all items are saleable
    ↓ reserve order ID
Quote::reserveOrderId()
    ├─> Generate increment ID
    ├─> Quote::setReservedOrderId($incrementId)
    └─> Save quote
    ↓ dispatch before submit event
Event: sales_model_service_quote_submit_before
    ↓ submit quote (create order)
CartManagement::placeOrder($cartId, $paymentMethod)
    ↓
    QuoteManagement::submit($quote)
        ├─> Convert quote to order
        │   QuoteToOrderConverter::convert($quote)
        │       ├─> Create Order entity
        │       ├─> Order::setCustomerId($quote->getCustomerId())
        │       ├─> Order::setStoreId($quote->getStoreId())
        │       ├─> Order::setQuoteId($quote->getId())
        │       ├─> Copy addresses
        │       │   QuoteAddressToOrderAddress::convert($quoteAddress)
        │       ├─> Copy payment
        │       │   QuotePaymentToOrderPayment::convert($quotePayment)
        │       └─> Return Order
        │
        ├─> Convert quote items to order items
        │   foreach ($quote->getAllVisibleItems() as $quoteItem)
        │       QuoteItemToOrderItem::convert($quoteItem)
        │           ├─> Create OrderItem
        │           ├─> OrderItem::setSku($quoteItem->getSku())
        │           ├─> OrderItem::setQtyOrdered($quoteItem->getQty())
        │           ├─> OrderItem::setPrice($quoteItem->getPrice())
        │           ├─> Copy options
        │           └─> Return OrderItem
        │
        ├─> Copy totals
        │   Order::setSubtotal($quote->getSubtotal())
        │   Order::setGrandTotal($quote->getGrandTotal())
        │   Order::setShippingAmount($quote->getShippingAmount())
        │   Order::setTaxAmount($quote->getTaxAmount())
        │   Order::setDiscountAmount($quote->getDiscountAmount())
        │
        ├─> Save order
        │   OrderRepository::save($order)
        │       ├─> INSERT INTO sales_order
        │       ├─> INSERT INTO sales_order_item (for each item)
        │       ├─> INSERT INTO sales_order_address (billing, shipping)
        │       ├─> INSERT INTO sales_order_payment
        │       └─> Event: sales_order_save_after
        │
        └─> Dispatch submit success event
            Event: sales_model_service_quote_submit_success
                └─> Observers: send order email, update inventory, etc.
    ↓ deactivate quote
    Quote::setIsActive(0)
    QuoteRepository::save($quote)
    ↓ return order ID
    Order::getId()
Redirect → /checkout/onepage/success?orderId=XXX
```

### Performance Metrics

- **Total Time:** 500-1500ms (depending on payment gateway)
- **Database Queries:** 15-25 queries
  - Quote load: 3-5 queries
  - Order save: 10-15 queries (header + items + addresses + payment)
  - Quote deactivate: 1 query
- **Cache Impact:** None (order placement doesn't invalidate FPC)
- **Events Dispatched:** 5-8 events (observers may send emails, update inventory)

---

## Performance Best Practices

### 1. Minimize Totals Collection

Totals collection is expensive. Only call `collectTotals()` when necessary:

```php
// BAD: Collect totals multiple times
$quote->addProduct($product1);
$quote->collectTotals(); // Unnecessary
$quote->addProduct($product2);
$quote->collectTotals(); // Unnecessary
$this->quoteRepository->save($quote);

// GOOD: Collect totals once after all changes
$quote->addProduct($product1);
$quote->addProduct($product2);
$quote->collectTotals(); // Only once
$this->quoteRepository->save($quote);
```

### 2. Use Repository Caching

Quote repository caches loaded quotes. Reuse quote instances:

```php
// BAD: Load quote multiple times
$quote1 = $this->quoteRepository->get($quoteId);
$quote1->addProduct($product);
$quote2 = $this->quoteRepository->get($quoteId); // Reload from DB
$quote2->collectTotals();

// GOOD: Load once, reuse
$quote = $this->quoteRepository->get($quoteId);
$quote->addProduct($product);
$quote->collectTotals();
$this->quoteRepository->save($quote);
```

### 3. Lazy Load Products

Don't load full product data unless needed:

```php
// BAD: Load full product with all attributes
$product = $this->productRepository->getById($productId);

// GOOD: Load minimal data
$product = $this->productRepository->getById($productId, false, $storeId, false);
```

---

**Assumptions:**
- Magento 2.4.7+ with PHP 8.2+
- MySQL/MariaDB database
- Redis session storage
- Standard single-address checkout flow

**Why This Approach:**
Detailed execution flows with code examples provide implementation-ready guidance. Sequence diagrams trace complete request-to-database paths for debugging and optimization.

**Security Impact:**
- All cart operations validate customer ownership
- Form key (CSRF) validation on all POST requests
- Coupon validation prevents invalid/expired code application
- Quote-to-order conversion validates payment method

**Performance Impact:**
- Totals collection is most expensive operation (5-15ms per call)
- Quote repository caching reduces database queries
- Mini cart updates use customer section data (no full page reload)
- Large carts (100+ items) require optimization

**Backward Compatibility:**
Service contracts ensure API stability. Event names and observer execution order are stable. Totals collector registration via sales.xml is upgrade-safe.

**Tests to Add:**
- Integration tests for each flow (add to cart, update, coupon, merge, place order)
- API functional tests for REST/GraphQL endpoints
- Performance tests for large carts (100+ items)
- MFTF tests for user flows

**Docs to Update:**
- Flow diagrams for cart operations
- Performance optimization guide
- Debugging guide with query analysis
