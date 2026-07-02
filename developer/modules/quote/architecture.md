---
title: "Magento_Quote Architecture"
module: "Magento_Quote"
doc_type: "architecture"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Quote Architecture

## Architectural Overview

The Magento_Quote module implements a layered architecture with clear separation between service contracts (API layer), business logic (model layer), and persistence (resource layer). The architecture follows Domain-Driven Design principles with the Quote aggregate root managing cart state, items, addresses, and totals.

**Target Version:** Magento 2.4.7+ | Adobe Commerce & Open Source
**PHP Version:** 8.2+

## Architectural Principles

### 1. Service Contract Driven

All cart operations exposed through stable service contracts in the `Api` namespace. Implementations reside in `Model` namespace, allowing internal refactoring without breaking API consumers.

### 2. Aggregate Root Pattern

The `Quote` entity serves as the aggregate root, managing the consistency of its child entities (items, addresses, payment, shipping rates). All modifications flow through the Quote to maintain invariants.

### 3. Totals Collector Pattern

Totals calculation uses a collector chain with ordered execution. Each collector focuses on a single concern (subtotal, tax, shipping, discount) and contributes to the total calculation independently.

### 4. Repository Pattern

Persistence abstraction through repositories (`CartRepositoryInterface`) decouples business logic from storage mechanisms. Supports caching, validation, and event dispatching.

### 5. Guest Cart Masking

Guest carts use masked IDs (UUIDs) to prevent enumeration attacks and maintain security. Masked IDs map to internal quote IDs via `quote_id_mask` table.

## System Components

### Service Layer (API)

#### Core Service Contracts

```
Api/
├── CartRepositoryInterface         # Quote CRUD operations
├── CartManagementInterface         # Cart lifecycle management
├── CartItemRepositoryInterface     # Item management
├── CartTotalRepositoryInterface    # Totals retrieval
├── CouponManagementInterface       # Coupon operations
├── PaymentMethodManagementInterface # Payment method management
├── ShippingMethodManagementInterface # Shipping method management
├── ShipmentEstimationInterface     # Shipping cost estimation
├── GuestCartRepositoryInterface    # Guest cart access
├── GuestCartManagementInterface    # Guest cart management
└── Data/                           # Data transfer objects
    ├── CartInterface               # Quote data
    ├── CartItemInterface           # Item data
    ├── AddressInterface            # Address data
    ├── PaymentInterface            # Payment data
    ├── TotalsInterface             # Totals data
    └── ShippingMethodInterface     # Shipping method data
```

#### Service Contract Design

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Api;

use Magento\Quote\Api\Data\CartInterface;
use Magento\Framework\Exception\NoSuchEntityException;
use Magento\Framework\Exception\CouldNotSaveException;

/**
 * Quote repository service contract
 *
 * Provides stable API for quote persistence operations.
 * Implementation may change, but interface remains stable.
 *
 * @api
 * @since 100.0.0
 */
interface CartRepositoryInterface
{
    /**
     * Get quote by ID (actual 2.4.7 signature — no PHP return type)
     *
     * @param int $cartId
     * @return \Magento\Quote\Api\Data\CartInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function get($cartId);

    /**
     * @param \Magento\Framework\Api\SearchCriteriaInterface $searchCriteria
     * @return \Magento\Quote\Api\Data\CartSearchResultsInterface
     */
    public function getList(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria);

    /**
     * @param int $customerId
     * @param int[] $sharedStoreIds
     * @return \Magento\Quote\Api\Data\CartInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function getForCustomer($customerId, array $sharedStoreIds = []);

    /**
     * @param int $cartId
     * @param int[] $sharedStoreIds
     * @return \Magento\Quote\Api\Data\CartInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function getActive($cartId, array $sharedStoreIds = []);

    /**
     * @param int $customerId
     * @param int[] $sharedStoreIds
     * @return \Magento\Quote\Api\Data\CartInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function getActiveForCustomer($customerId, array $sharedStoreIds = []);

    /**
     * Save quote and all child entities (items, addresses, payment).
     *
     * @param \Magento\Quote\Api\Data\CartInterface $quote
     * @return void
     */
    public function save(\Magento\Quote\Api\Data\CartInterface $quote);

    /**
     * @param \Magento\Quote\Api\Data\CartInterface $quote
     * @return void
     */
    public function delete(\Magento\Quote\Api\Data\CartInterface $quote);
}
```

### Model Layer

#### Quote Entity (Aggregate Root)

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Model;

use Magento\Quote\Api\Data\CartInterface;
use Magento\Framework\Model\AbstractExtensibleModel;

/**
 * Quote entity - shopping cart aggregate root
 *
 * Manages cart state, items, addresses, payment, and totals.
 * All modifications should flow through this class to maintain invariants.
 *
 * @method int getEntityId()
 * @method Quote setEntityId(int $value)
 * @method int getStoreId()
 * @method Quote setStoreId(int $value)
 */
class Quote extends AbstractExtensibleModel implements CartInterface
{
    /**
     * Quote items collection
     *
     * @var \Magento\Quote\Model\ResourceModel\Quote\Item\Collection
     */
    protected $_items;

    /**
     * Quote addresses collection
     *
     * @var \Magento\Quote\Model\ResourceModel\Quote\Address\Collection
     */
    protected $_addresses;

    /**
     * Billing address cache
     *
     * @var Quote\Address|null
     */
    protected $_billingAddress;

    /**
     * Shipping address cache
     *
     * @var Quote\Address|null
     */
    protected $_shippingAddress;

    /**
     * Payment instance
     *
     * @var Quote\Payment|null
     */
    protected $_payment;

    /**
     * Add product to quote
     *
     * Validates product, creates quote item, and adds to items collection.
     * Returns error message string if validation fails.
     *
     * @param \Magento\Catalog\Model\Product $product
     * @param \Magento\Framework\DataObject|float|array $request
     * @param string|null $processMode
     * @return Quote\Item|string
     */
    public function addProduct(
        \Magento\Catalog\Model\Product $product,
        $request = null,
        $processMode = null
    ) {
        if ($request === null) {
            $request = 1;
        }
        if (is_numeric($request)) {
            $request = new \Magento\Framework\DataObject(['qty' => $request]);
        }
        if (!$request instanceof \Magento\Framework\DataObject) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('We found an invalid request for adding product to quote.')
            );
        }

        $cartCandidates = $product->getTypeInstance()
            ->prepareForCartAdvanced($request, $product, $processMode);

        // String return means error
        if (is_string($cartCandidates)) {
            return $cartCandidates;
        }

        // Product successfully prepared
        if (!is_array($cartCandidates)) {
            $cartCandidates = [$cartCandidates];
        }

        $parentItem = null;
        $errors = [];
        $items = [];

        foreach ($cartCandidates as $candidate) {
            $item = $this->_addCatalogProduct($candidate, $candidate->getCartQty());

            if (is_string($item)) {
                $errors[] = $item;
                continue;
            }

            $items[] = $item;

            if (!$parentItem) {
                $parentItem = $item;
            }
            if ($parentItem && $candidate->getParentProductId()) {
                $item->setParentItem($parentItem);
            }

            $this->_eventManager->dispatch(
                'sales_quote_product_add_after',
                ['items' => $items]
            );
        }

        if (!empty($errors)) {
            return implode("\n", $errors);
        }

        return $parentItem;
    }

    /**
     * Collect totals for quote
     *
     * Executes all registered totals collectors in priority order.
     * Resets existing totals before recalculation.
     *
     * @return $this
     */
    public function collectTotals()
    {
        // Note: Quote::collectTotals() delegates to TotalsCollector.
        // The events sales_quote_collect_totals_before and
        // sales_quote_collect_totals_after are dispatched inside
        // TotalsCollector::collect(), NOT directly in this method.
        $this->totalsCollector->collect($this);
        $this->addData($this->totalsCollector->getData());

        return $this;
    }

    /**
     * Get all quote items
     *
     * @return Quote\Item[]
     */
    public function getAllItems(): array
    {
        $items = [];
        foreach ($this->getItemsCollection() as $item) {
            if (!$item->isDeleted()) {
                $items[] = $item;
            }
        }
        return $items;
    }

    /**
     * Get billing address
     *
     * @return Quote\Address
     */
    public function getBillingAddress(): Quote\Address
    {
        if ($this->_billingAddress === null) {
            foreach ($this->getAddressesCollection() as $address) {
                if ($address->getAddressType() === Address::ADDRESS_TYPE_BILLING) {
                    $this->_billingAddress = $address;
                    break;
                }
            }
        }

        if ($this->_billingAddress === null) {
            $this->_billingAddress = $this->_addressFactory->create();
            $this->_billingAddress->setAddressType(Address::ADDRESS_TYPE_BILLING);
            $this->addAddress($this->_billingAddress);
        }

        return $this->_billingAddress;
    }

    /**
     * Get shipping address
     *
     * @return Quote\Address
     */
    public function getShippingAddress(): Quote\Address
    {
        if ($this->_shippingAddress === null) {
            foreach ($this->getAddressesCollection() as $address) {
                if ($address->getAddressType() === Address::ADDRESS_TYPE_SHIPPING) {
                    $this->_shippingAddress = $address;
                    break;
                }
            }
        }

        if ($this->_shippingAddress === null) {
            $this->_shippingAddress = $this->_addressFactory->create();
            $this->_shippingAddress->setAddressType(Address::ADDRESS_TYPE_SHIPPING);
            $this->addAddress($this->_shippingAddress);
        }

        return $this->_shippingAddress;
    }

    /**
     * Check if quote is virtual (no physical products)
     *
     * @return bool
     */
    public function isVirtual(): bool
    {
        $isVirtual = true;
        $countItems = 0;

        foreach ($this->getItemsCollection() as $item) {
            if ($item->isDeleted() || $item->getParentItemId()) {
                continue;
            }
            $countItems++;
            if (!$item->getProduct()->getIsVirtual()) {
                $isVirtual = false;
            }
        }

        return $countItems === 0 ? false : $isVirtual;
    }

    /**
     * Merge quotes (guest cart into customer cart)
     *
     * @param Quote $quote Source quote to merge from
     * @return $this
     */
    public function merge(Quote $quote): self
    {
        $this->_eventManager->dispatch(
            'sales_quote_merge_before',
            ['quote' => $this, 'source' => $quote]
        );

        foreach ($quote->getAllVisibleItems() as $item) {
            $found = false;
            foreach ($this->getAllItems() as $quoteItem) {
                if ($quoteItem->compare($item)) {
                    $quoteItem->setQty($quoteItem->getQty() + $item->getQty());
                    $found = true;
                    break;
                }
            }

            if (!$found) {
                $newItem = clone $item;
                $this->addItem($newItem);
                if ($item->getHasChildren()) {
                    foreach ($item->getChildren() as $child) {
                        $newChild = clone $child;
                        $newChild->setParentItem($newItem);
                        $this->addItem($newChild);
                    }
                }
            }
        }

        // Merge coupon code if not already set
        if (!$this->getCouponCode() && $quote->getCouponCode()) {
            $this->setCouponCode($quote->getCouponCode());
        }

        $this->_eventManager->dispatch(
            'sales_quote_merge_after',
            ['quote' => $this, 'source' => $quote]
        );

        return $this;
    }
}
```

#### Quote Item

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Model\Quote;

use Magento\Quote\Api\Data\CartItemInterface;
use Magento\Framework\Model\AbstractExtensibleModel;

/**
 * Quote item entity
 *
 * Represents a single product in the cart with quantity, pricing,
 * and configuration options.
 */
class Item extends AbstractExtensibleModel implements CartItemInterface
{
    /**
     * Parent item (for child products of configurable/bundle)
     *
     * @var Item|null
     */
    protected $_parentItem;

    /**
     * Child items
     *
     * @var Item[]
     */
    protected $_children = [];

    /**
     * Product instance
     *
     * @var \Magento\Catalog\Model\Product|null
     */
    protected $_product;

    /**
     * Quote instance
     *
     * @var \Magento\Quote\Model\Quote|null
     */
    protected $_quote;

    /**
     * Item options collection
     *
     * @var \Magento\Quote\Model\Quote\Item\Option[]
     */
    protected $_options = [];

    /**
     * Calculate row total
     *
     * Row total = (Price - Discount) * Qty
     *
     * @return $this
     */
    public function calcRowTotal(): self
    {
        $qty = $this->getQty();
        $price = $this->getCalculationPrice();
        $basePrice = $this->getBaseCalculationPrice();

        $rowTotal = $price * $qty;
        $baseRowTotal = $basePrice * $qty;

        $this->setRowTotal($this->priceCurrency->round($rowTotal));
        $this->setBaseRowTotal($this->priceCurrency->round($baseRowTotal));

        return $this;
    }

    /**
     * Get product instance
     *
     * @return \Magento\Catalog\Model\Product
     */
    public function getProduct(): \Magento\Catalog\Model\Product
    {
        if ($this->_product === null) {
            $this->_product = $this->productRepository->getById(
                $this->getProductId(),
                false,
                $this->getQuote()->getStoreId()
            );
        }
        return $this->_product;
    }

    /**
     * Compare item with another item
     *
     * Used for merging quotes - determines if items represent same product
     * with same configuration.
     *
     * @param Item $item
     * @return bool
     */
    public function compare(Item $item): bool
    {
        if ($this->getProductId() !== $item->getProductId()) {
            return false;
        }

        foreach ($this->getOptions() as $option) {
            if ($option->getCode() === 'info_buyRequest') {
                $requestData = $option->getValue();
                $itemRequestData = $item->getOptionByCode('info_buyRequest');
                if ($itemRequestData && $requestData !== $itemRequestData->getValue()) {
                    return false;
                }
            }
        }

        return true;
    }

    /**
     * Get buy request for item
     *
     * Contains original product configuration (options, qty, etc.)
     *
     * @return \Magento\Framework\DataObject
     */
    public function getBuyRequest(): \Magento\Framework\DataObject
    {
        $option = $this->getOptionByCode('info_buyRequest');
        if ($option) {
            $request = new \Magento\Framework\DataObject(
                $this->serializer->unserialize($option->getValue())
            );
            return $request;
        }
        return new \Magento\Framework\DataObject();
    }

    /**
     * Set parent item
     *
     * @param Item $parentItem
     * @return $this
     */
    public function setParentItem(Item $parentItem): self
    {
        $this->_parentItem = $parentItem;
        $parentItem->addChild($this);
        return $this;
    }

    /**
     * Add child item
     *
     * @param Item $child
     * @return $this
     */
    protected function addChild(Item $child): self
    {
        $this->_children[] = $child;
        return $this;
    }

    /**
     * Get child items
     *
     * @return Item[]
     */
    public function getChildren(): array
    {
        return $this->_children;
    }

    /**
     * Check if item has children
     *
     * @return bool
     */
    public function getHasChildren(): bool
    {
        return !empty($this->_children);
    }
}
```

### Totals Collection Architecture

#### TotalsCollector

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Model\Quote;

use Magento\Quote\Model\Quote;
use Magento\Quote\Api\Data\ShippingAssignmentInterface;

/**
 * Totals collector orchestrator
 *
 * Executes all registered totals collectors in priority order.
 * Each collector contributes to address totals independently.
 */
class TotalsCollector
{
    /**
     * Totals collector instances
     *
     * @var Address\Total\AbstractTotal[]
     */
    private array $collectors;

    /**
     * Collect totals for quote
     *
     * @param Quote $quote
     * @return Address\Total
     */
    public function collect(Quote $quote): Address\Total
    {
        $total = $this->totalFactory->create(Address\Total::class);

        foreach ($quote->getAllAddresses() as $address) {
            $this->collectAddressTotals($quote, $address);
        }

        return $total;
    }

    /**
     * Collect totals for address
     *
     * @param Quote $quote
     * @param Quote\Address $address
     * @return Address\Total
     */
    public function collectAddressTotals(
        Quote $quote,
        Quote\Address $address
    ): Address\Total {
        $total = $this->totalFactory->create(Address\Total::class);
        $this->_setCollectorData($total);

        $shippingAssignment = $this->shippingAssignmentFactory->create([
            'shipping' => $this->shippingFactory->create([
                'address' => $address
            ])
        ]);

        // Execute collectors in priority order
        foreach ($this->getCollectors() as $collector) {
            $collector->collect($quote, $shippingAssignment, $total);
        }

        // Update address totals
        $address->addData($total->getData());
        $address->setAppliedTaxes($total->getAppliedTaxes());

        return $total;
    }

    /**
     * Get collectors sorted by priority
     *
     * @return Address\Total\AbstractTotal[]
     */
    private function getCollectors(): array
    {
        if ($this->collectors === null) {
            $this->collectors = $this->totalsSortedResolver->getSortedCollectorsList();
        }
        return $this->collectors;
    }
}
```

#### Abstract Total Collector

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Model\Quote\Address\Total;

use Magento\Quote\Model\Quote;
use Magento\Quote\Api\Data\ShippingAssignmentInterface;
use Magento\Quote\Model\Quote\Address\Total;

/**
 * Abstract totals collector
 *
 * Base class for all totals collectors. Provides common functionality
 * and defines collector contract.
 */
abstract class AbstractTotal
{
    /**
     * Totals collector code (subtotal, tax, shipping, etc.)
     *
     * @var string
     */
    protected string $_code;

    /**
     * Collect totals
     *
     * Main entry point for totals calculation. Modify $total object
     * to contribute to quote totals.
     *
     * @param Quote $quote
     * @param ShippingAssignmentInterface $shippingAssignment
     * @param Total $total
     * @return $this
     */
    abstract public function collect(
        Quote $quote,
        ShippingAssignmentInterface $shippingAssignment,
        Total $total
    ): self;

    /**
     * Fetch totals for display
     *
     * Return array of totals to display in cart/checkout.
     *
     * @param Quote $quote
     * @param Total $total
     * @return array
     */
    public function fetch(Quote $quote, Total $total): array
    {
        return [];
    }

    /**
     * Get collector code
     *
     * @return string
     */
    public function getCode(): string
    {
        return $this->_code;
    }

    /**
     * Set total amount
     *
     * Helper to add amount to total.
     *
     * @param Total $total
     * @param string $code
     * @param float $amount
     * @return $this
     */
    protected function _addAmount(Total $total, string $code, float $amount): self
    {
        $total->setTotalAmount($code, $amount);
        return $this;
    }

    /**
     * Set base total amount
     *
     * @param Total $total
     * @param string $code
     * @param float $amount
     * @return $this
     */
    protected function _addBaseAmount(Total $total, string $code, float $amount): self
    {
        $total->setBaseTotalAmount($code, $amount);
        return $this;
    }
}
```

#### Subtotal Collector Example

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Model\Quote\Address\Total;

use Magento\Quote\Model\Quote;
use Magento\Quote\Api\Data\ShippingAssignmentInterface;
use Magento\Quote\Model\Quote\Address\Total;

/**
 * Subtotal totals collector
 *
 * Calculates subtotal from all items in cart.
 * Executes first in collector chain (priority 100).
 */
class Subtotal extends AbstractTotal
{
    /**
     * Collector code
     *
     * @var string
     */
    protected string $_code = 'subtotal';

    /**
     * Collect subtotal
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

        $subtotal = 0;
        $baseSubtotal = 0;
        $itemsQty = 0;

        foreach ($items as $item) {
            if ($item->getParentItem()) {
                continue;
            }

            // Calculate item row total
            $item->calcRowTotal();

            $subtotal += $item->getRowTotal();
            $baseSubtotal += $item->getBaseRowTotal();
            $itemsQty += $item->getQty();
        }

        $total->setSubtotal($subtotal);
        $total->setBaseSubtotal($baseSubtotal);
        $total->setItemsQty($itemsQty);
        $total->setItemsCount(count($items));

        $address->setSubtotal($subtotal);
        $address->setBaseSubtotal($baseSubtotal);

        return $this;
    }

    /**
     * Fetch subtotal for display
     *
     * @param Quote $quote
     * @param Total $total
     * @return array
     */
    public function fetch(Quote $quote, Total $total): array
    {
        return [
            'code' => $this->getCode(),
            'title' => __('Subtotal'),
            'value' => $total->getSubtotal()
        ];
    }
}
```

### Repository Pattern

#### Quote Repository Implementation

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Model;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\Data\CartInterface;
use Magento\Framework\Exception\NoSuchEntityException;
use Magento\Framework\Exception\CouldNotSaveException;

/**
 * Quote repository implementation
 *
 * Handles quote persistence with caching, validation, and event dispatching.
 */
class QuoteRepository implements CartRepositoryInterface
{
    /**
     * Quote factory
     *
     * @var QuoteFactory
     */
    private QuoteFactory $quoteFactory;

    /**
     * Resource model
     *
     * @var ResourceModel\Quote
     */
    private ResourceModel\Quote $quoteResource;

    /**
     * Quote cache
     *
     * @var array<int, Quote>
     */
    private array $quotesById = [];

    /**
     * Customer quote cache
     *
     * @var array<int, Quote>
     */
    private array $quotesByCustomerId = [];

    /**
     * Get quote by ID
     *
     * @param int $cartId
     * @return CartInterface
     * @throws NoSuchEntityException
     */
    public function get($cartId)
    {
        if (!isset($this->quotesById[$cartId])) {
            $quote = $this->quoteFactory->create();
            $this->quoteResource->load($quote, $cartId);

            if (!$quote->getId()) {
                throw new NoSuchEntityException(
                    __('No such entity with cartId = %1', $cartId)
                );
            }

            $this->quotesById[$cartId] = $quote;
        }

        return $this->quotesById[$cartId];
    }

    /**
     * {@inheritdoc}
     */
    public function getForCustomer($customerId, array $sharedStoreIds = [])
    {
        $cacheKey = $this->getCacheKeyForCustomer($customerId, $sharedStoreIds);

        if (!isset($this->quotesByCustomerId[$cacheKey])) {
            $quote = $this->quoteFactory->create();
            $this->quoteResource->loadByCustomerId($quote, $customerId);

            if (!$quote->getId()) {
                throw new NoSuchEntityException(
                    __('No such entity with customerId = %1', $customerId)
                );
            }

            $this->quotesByCustomerId[$cacheKey] = $quote;
        }

        return $this->quotesByCustomerId[$cacheKey];
    }

    /**
     * {@inheritdoc}
     */
    public function save(\Magento\Quote\Api\Data\CartInterface $quote)
    {
        try {
            $this->quoteResource->save($quote);

            // Clear cache
            unset($this->quotesById[$quote->getId()]);
            if ($quote->getCustomerId()) {
                $cacheKey = $this->getCacheKeyForCustomer($quote->getCustomerId(), []);
                unset($this->quotesByCustomerId[$cacheKey]);
            }
        } catch (\Exception $e) {
            throw new CouldNotSaveException(
                __('Could not save quote: %1', $e->getMessage()),
                $e
            );
        }
    }

    /**
     * {@inheritdoc}
     */
    public function delete(\Magento\Quote\Api\Data\CartInterface $quote)
    {
        $quoteId = $quote->getId();
        $customerId = $quote->getCustomerId();

        $this->quoteResource->delete($quote);

        unset($this->quotesById[$quoteId]);
        if ($customerId) {
            $cacheKey = $this->getCacheKeyForCustomer($customerId, []);
            unset($this->quotesByCustomerId[$cacheKey]);
        }
    }

    /**
     * Get cache key for customer
     *
     * @param int $customerId
     * @param int[] $sharedStoreIds
     * @return string
     */
    private function getCacheKeyForCustomer(int $customerId, array $sharedStoreIds): string
    {
        $key = $customerId;
        if ($sharedStoreIds) {
            $key .= '_' . implode('_', $sharedStoreIds);
        }
        return (string)$key;
    }
}
```

### Guest Cart Architecture

#### Masked ID Pattern

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Model;

/**
 * Quote ID mask
 *
 * Maps masked IDs (UUIDs) to internal quote IDs for guest carts.
 * Prevents quote ID enumeration attacks.
 */
class QuoteIdMask extends \Magento\Framework\Model\AbstractModel
{
    /**
     * Get quote ID for masked ID
     *
     * @param string $maskedId
     * @return int|null
     */
    public function getQuoteIdByMaskedId(string $maskedId): ?int
    {
        $this->load($maskedId, 'masked_id');
        return $this->getQuoteId() ?: null;
    }

    /**
     * Get masked ID for quote ID
     *
     * @param int $quoteId
     * @return string|null
     */
    public function getMaskedIdByQuoteId(int $quoteId): ?string
    {
        $this->load($quoteId, 'quote_id');
        return $this->getMaskedId() ?: null;
    }

    /**
     * Create masked ID for quote
     *
     * @param int $quoteId
     * @return string
     */
    public function createMaskedId(int $quoteId): string
    {
        $maskedId = $this->randomDataGenerator->getUniqueHash();
        $this->setQuoteId($quoteId);
        $this->setMaskedId($maskedId);
        $this->save();
        return $maskedId;
    }
}
```

#### Guest Cart Repository

```php
<?php
declare(strict_types=1);

namespace Magento\Quote\Model;

use Magento\Quote\Api\GuestCartRepositoryInterface;
use Magento\Quote\Api\Data\CartInterface;
use Magento\Framework\Exception\NoSuchEntityException;

/**
 * Guest cart repository
 *
 * Provides guest cart operations using masked IDs.
 */
class GuestCartRepository implements GuestCartRepositoryInterface
{
    public function __construct(
        private readonly CartRepositoryInterface $quoteRepository,
        private readonly QuoteIdMaskFactory $quoteIdMaskFactory
    ) {}

    /**
     * Get cart by masked ID
     *
     * @param string $cartId Masked ID
     * @return CartInterface
     * @throws NoSuchEntityException
     */
    public function get($cartId)
    {
        $quoteIdMask = $this->quoteIdMaskFactory->create();
        $quoteId = $quoteIdMask->getQuoteIdByMaskedId($cartId);

        if (!$quoteId) {
            throw new NoSuchEntityException(
                __('No such entity with cartId = %1', $cartId)
            );
        }

        return $this->quoteRepository->get($quoteId);
    }
}
```

## Data Flow Patterns

### Add to Cart Flow

```
Controller/API
    ↓
CartItemRepository::save()
    ↓
Quote::addProduct()
    ↓ (validate product)
Product\Type::prepareForCartAdvanced()
    ↓ (create item)
Quote::_addCatalogProduct()
    ↓ (add to collection)
Quote\Item\Collection::addItem()
    ↓ (recalculate)
Quote::collectTotals()
    ↓ (persist)
QuoteRepository::save()
```

### Totals Calculation Flow

```
Quote::collectTotals()
    ↓
TotalsCollector::collect()
    ↓ (for each address)
TotalsCollector::collectAddressTotals()
    ↓ (for each collector by priority)
    ├─> Subtotal::collect()
    ├─> Shipping::collect()
    ├─> Tax::collect()
    ├─> Discount::collect()
    └─> GrandTotal::collect()
    ↓ (update address)
Address::addData(Total::getData())
```

### Quote-to-Order Conversion

```
CartManagement::placeOrder()
    ↓ (validate)
QuoteValidator::validateBeforeSubmit()
    ↓ (reserve order ID)
Quote::reserveOrderId()
    ↓ (dispatch event)
Event: sales_model_service_quote_submit_before
    ↓ (convert)
QuoteManagement::submit()
    ↓ (create order)
QuoteToOrder::convert()
    ↓ (convert items)
QuoteItemToOrderItem::convert()
    ↓ (dispatch event)
Event: sales_model_service_quote_submit_success
    ↓ (deactivate quote)
Quote::setIsActive(false)
```

## Extension Mechanisms

### 1. Plugins (Interceptors)

Modify service contract behavior without inheritance.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin;

use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\Data\CartInterface;

class QuoteRepositoryExtend
{
    /**
     * Before save - validate custom business rules
     *
     * @param CartRepositoryInterface $subject
     * @param CartInterface $quote
     * @return array
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function beforeSave(
        CartRepositoryInterface $subject,
        CartInterface $quote
    ): array {
        // Custom validation
        if ($quote->getItemsCount() > 100) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Cart cannot contain more than 100 items.')
            );
        }

        return [$quote];
    }

    /**
     * After save - log or sync to external system
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
        // Log cart save
        $this->logger->info('Quote saved', ['quote_id' => $quote->getId()]);
    }
}
```

### 2. Observers

React to quote events.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

class QuoteSaveAfterObserver implements ObserverInterface
{
    /**
     * Execute observer
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        $quote = $observer->getEvent()->getQuote();

        // Custom logic after quote save
        // Example: Sync to external CRM
        $this->crmSync->syncCart($quote);
    }
}
```

### 3. Extension Attributes

Add custom data to quotes via API.

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Api/etc/extension_attributes.xsd">
    <extension_attributes for="Magento\Quote\Api\Data\CartInterface">
        <attribute code="loyalty_points" type="int"/>
        <attribute code="gift_message" type="string"/>
    </extension_attributes>
</config>
```

## Performance Considerations

### Quote Caching Strategy

- **Quote repository caches** loaded quotes in-memory to avoid repeated DB queries
- **FPC excludes** customer-specific cart data (uses private content/sections)
- **Session storage** persists quote ID, not entire quote object

### Totals Calculation Optimization

- **Collector ordering** minimizes redundant calculations (subtotal before discount)
- **Lazy loading** defers product loading until needed
- **Delta updates** only recalculate affected items on quantity change

### Database Indexes

```sql
-- Key indexes on quote table
INDEX(customer_id, is_active, store_id)  -- Customer quote lookup
INDEX(updated_at)                         -- Abandoned cart cleanup
INDEX(reserved_order_id)                  -- Order association

-- Key indexes on quote_item table
INDEX(quote_id)                           -- Items for quote
INDEX(product_id, store_id)               -- Product availability checks
```

---

**Assumptions:**
- Magento 2.4.7+ with PHP 8.2+
- Standard quote configuration (single-address default)
- MySQL/MariaDB database
- Redis for session storage

**Why This Approach:**
Service contracts provide API stability and web API exposure. Aggregate root pattern maintains quote consistency. Totals collector chain enables modular, extensible calculations. Repository pattern abstracts persistence and enables caching.

**Security Impact:**
- Guest cart masked IDs prevent enumeration
- Quote ownership validation required on all operations
- Price validation server-side (never trust client)
- CSRF protection on all cart modification forms

**Performance Impact:**
- Quote operations invalidate FPC for affected customer sections
- Totals collection is expensive; cache when possible
- Large carts (100+ items) require optimization
- Database indexes critical for customer quote lookup

**Backward Compatibility:**
Service contracts guarantee API stability. Totals collectors registered via di.xml are upgrade-safe. Direct model manipulation may break across versions.

**Tests to Add:**
- Unit tests for Quote model methods
- Integration tests for totals collectors
- API functional tests for cart operations
- Performance tests for large carts (100+ items)

**Docs to Update:**
- Architecture diagrams for totals collection flow
- Extension point documentation for custom collectors
- API documentation for extension attributes
