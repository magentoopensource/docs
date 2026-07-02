---
title: "Magento_Sales Architecture"
module: "Magento_Sales"
doc_type: "architecture"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Sales Architecture

## Architectural Overview

The Magento_Sales module implements a sophisticated domain-driven design with clear separation between data access, business logic, and presentation layers. The architecture follows SOLID principles with extensive use of service contracts, repository patterns, and event-driven communication.

## Core Architectural Patterns

### Service Contract Layer

The Sales module exposes all operations through well-defined service contracts in the `Magento\Sales\Api` namespace. This provides API stability and enables:

- **Version compatibility**: Internal implementation changes without breaking clients
- **Web API exposure**: Automatic REST/SOAP endpoint generation
- **Plugin interception**: Extension points at service boundaries
- **Type safety**: Strong contracts for all operations

### Repository Pattern

All entity persistence operations go through repository interfaces:

```php
<?php
namespace Magento\Sales\Model;

use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Model\ResourceModel\Order as OrderResource;
use Magento\Framework\Exception\NoSuchEntityException;
use Magento\Framework\Exception\CouldNotSaveException;

/**
 * Order repository implementation
 *
 * Handles CRUD operations with proper exception handling and extension attributes
 */
class OrderRepository implements OrderRepositoryInterface
{
    private array $registry = [];

    public function __construct(
        private OrderFactory $orderFactory,
        private OrderResource $orderResource,
        private \Magento\Sales\Api\Data\OrderSearchResultInterfaceFactory $searchResultFactory,
        private \Magento\Framework\Api\SearchCriteria\CollectionProcessorInterface $collectionProcessor,
        private \Magento\Sales\Model\ResourceModel\Order\CollectionFactory $orderCollectionFactory,
        private \Magento\Sales\Api\Data\OrderExtensionFactory $orderExtensionFactory
    ) {}

    /**
     * Load order by ID with identity map pattern
     *
     * @param int $id
     * @return OrderInterface
     * @throws NoSuchEntityException
     */
    public function get($id)
    {
        if (isset($this->registry[$id])) {
            return $this->registry[$id];
        }

        $order = $this->orderFactory->create();
        $this->orderResource->load($order, $id);

        if (!$order->getEntityId()) {
            throw new NoSuchEntityException(
                __('The entity that was requested doesn\'t exist. Verify the entity and try again.')
            );
        }

        $this->registry[$id] = $order;
        return $order;
    }

    /**
     * {@inheritdoc}
     */
    public function save(\Magento\Sales\Api\Data\OrderInterface $entity)
    {
        try {
            $this->orderResource->save($entity);
            unset($this->registry[$entity->getEntityId()]);
        } catch (\Exception $e) {
            throw new CouldNotSaveException(
                __('Could not save the order: %1', $e->getMessage()),
                $e
            );
        }

        return $entity;
    }

    /**
     * {@inheritdoc}
     */
    public function getList(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria)
    {
        $collection = $this->orderCollectionFactory->create();
        $this->collectionProcessor->process($searchCriteria, $collection);

        $searchResults = $this->searchResultFactory->create();
        $searchResults->setSearchCriteria($searchCriteria);
        $searchResults->setItems($collection->getItems());
        $searchResults->setTotalCount($collection->getSize());

        return $searchResults;
    }

    /**
     * {@inheritdoc}
     */
    public function delete(\Magento\Sales\Api\Data\OrderInterface $entity)
    {
        try {
            $this->orderResource->delete($entity);
            unset($this->registry[$entity->getEntityId()]);
        } catch (\Exception $e) {
            throw new \Magento\Framework\Exception\CouldNotDeleteException(
                __('Could not delete the order: %1', $e->getMessage())
            );
        }

        return true;
    }
}
```

### Entity-Attribute-Value (EAV) vs Flat Tables

Sales entities use **flat table architecture** (not EAV) for performance:

```sql
-- Order table structure (flat)
CREATE TABLE sales_order (
    entity_id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    state VARCHAR(32),
    status VARCHAR(32),
    customer_id INT UNSIGNED,
    base_grand_total DECIMAL(20,4),
    grand_total DECIMAL(20,4),
    base_currency_code VARCHAR(3),
    order_currency_code VARCHAR(3),
    store_id SMALLINT UNSIGNED,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    -- ... 100+ columns for all order attributes
    PRIMARY KEY (entity_id),
    INDEX (state),
    INDEX (status),
    INDEX (customer_id),
    INDEX (created_at),
    INDEX (increment_id),
    CONSTRAINT FK_SALES_ORDER_CUSTOMER_ID FOREIGN KEY (customer_id)
        REFERENCES customer_entity(entity_id) ON DELETE SET NULL
);
```

This flat structure provides:
- **Fast reads**: Single table query without joins
- **Simple indexing**: Direct column indexes
- **No attribute loading overhead**: All data in one row
- **Grid performance**: Optimized for admin grids

## Domain Model Architecture

### Order Aggregate Root

The Order is the aggregate root containing all related entities:

```php
<?php
namespace Magento\Sales\Model;

use Magento\Sales\Api\Data\OrderInterface;
use Magento\Framework\Model\AbstractModel;

/**
 * Order entity aggregate root
 *
 * Contains all order-related entities and enforces business invariants
 */
class Order extends AbstractModel implements OrderInterface
{
    const STATE_NEW = 'new';
    const STATE_PENDING_PAYMENT = 'pending_payment';
    const STATE_PROCESSING = 'processing';
    const STATE_COMPLETE = 'complete';
    const STATE_CLOSED = 'closed';
    const STATE_CANCELED = 'canceled';
    const STATE_HOLDED = 'holded';
    const STATE_PAYMENT_REVIEW = 'payment_review';

    /**
     * Order items collection
     *
     * @var \Magento\Sales\Model\ResourceModel\Order\Item\Collection|null
     */
    private ?\Magento\Sales\Model\ResourceModel\Order\Item\Collection $items = null;

    /**
     * Order addresses collection
     *
     * @var \Magento\Sales\Model\ResourceModel\Order\Address\Collection|null
     */
    private ?\Magento\Sales\Model\ResourceModel\Order\Address\Collection $addresses = null;

    /**
     * Order payment
     *
     * @var Order\Payment|null
     */
    private ?Order\Payment $payment = null;

    /**
     * Order status history collection
     *
     * @var \Magento\Sales\Model\ResourceModel\Order\Status\History\Collection|null
     */
    private ?\Magento\Sales\Model\ResourceModel\Order\Status\History\Collection $statusHistory = null;

    /**
     * Initialize resource model
     */
    protected function _construct(): void
    {
        $this->_init(\Magento\Sales\Model\ResourceModel\Order::class);
    }

    /**
     * Get all order items (lazy load)
     *
     * @return \Magento\Sales\Model\Order\Item[]
     */
    public function getAllItems(): array
    {
        if ($this->items === null) {
            $this->items = $this->getItemsCollection();
        }

        return $this->items->getItems();
    }

    /**
     * Get items collection
     *
     * @return \Magento\Sales\Model\ResourceModel\Order\Item\Collection
     */
    public function getItemsCollection(): \Magento\Sales\Model\ResourceModel\Order\Item\Collection
    {
        if ($this->items === null) {
            $this->items = $this->_orderItemCollectionFactory->create()
                ->setOrderFilter($this);
        }

        return $this->items;
    }

    /**
     * Get order payment
     *
     * @return Order\Payment
     */
    public function getPayment(): Order\Payment
    {
        if ($this->payment === null) {
            $this->payment = $this->_paymentFactory->create()
                ->setOrder($this);

            $this->_orderResource->loadOrderPayment($this->payment, $this->getId());
        }

        return $this->payment;
    }

    /**
     * Get billing address
     *
     * @return Order\Address|null
     */
    public function getBillingAddress(): ?Order\Address
    {
        foreach ($this->getAddressesCollection() as $address) {
            if ($address->getAddressType() === Order\Address::TYPE_BILLING) {
                return $address;
            }
        }

        return null;
    }

    /**
     * Get shipping address
     *
     * @return Order\Address|null
     */
    public function getShippingAddress(): ?Order\Address
    {
        foreach ($this->getAddressesCollection() as $address) {
            if ($address->getAddressType() === Order\Address::TYPE_SHIPPING) {
                return $address;
            }
        }

        return null;
    }

    /**
     * Can cancel order (business rule enforcement)
     *
     * @return bool
     */
    public function canCancel(): bool
    {
        if ($this->getState() === self::STATE_CANCELED ||
            $this->getState() === self::STATE_COMPLETE ||
            $this->getState() === self::STATE_CLOSED) {
            return false;
        }

        if ($this->getActionFlag(self::ACTION_FLAG_CANCEL) === false) {
            return false;
        }

        // Check if all items can be canceled
        foreach ($this->getAllItems() as $item) {
            if (!$item->canCancel()) {
                return false;
            }
        }

        return true;
    }

    /**
     * Can create invoice (business rule enforcement)
     *
     * @return bool
     */
    public function canInvoice(): bool
    {
        if ($this->getState() === self::STATE_CANCELED ||
            $this->getState() === self::STATE_COMPLETE ||
            $this->getState() === self::STATE_CLOSED) {
            return false;
        }

        if ($this->getActionFlag(self::ACTION_FLAG_INVOICE) === false) {
            return false;
        }

        // Check if any items can be invoiced
        foreach ($this->getAllItems() as $item) {
            if ($item->getQtyToInvoice() > 0 && !$item->getLockedDoInvoice()) {
                return true;
            }
        }

        return false;
    }

    /**
     * Can create shipment (business rule enforcement)
     *
     * @return bool
     */
    public function canShip(): bool
    {
        if ($this->getState() === self::STATE_CANCELED ||
            $this->getState() === self::STATE_COMPLETE ||
            $this->getState() === self::STATE_CLOSED ||
            $this->getState() === self::STATE_HOLDED) {
            return false;
        }

        if ($this->getIsVirtual()) {
            return false;
        }

        if ($this->getActionFlag(self::ACTION_FLAG_SHIP) === false) {
            return false;
        }

        // Check if any items can be shipped
        foreach ($this->getAllItems() as $item) {
            if ($item->getQtyToShip() > 0 && !$item->getIsVirtual()) {
                return true;
            }
        }

        return false;
    }

    /**
     * Can create credit memo (business rule enforcement)
     *
     * @return bool
     */
    public function canCreditmemo(): bool
    {
        if ($this->getState() === self::STATE_CANCELED ||
            $this->getState() === self::STATE_CLOSED ||
            $this->getState() === self::STATE_HOLDED) {
            return false;
        }

        if ($this->getActionFlag(self::ACTION_FLAG_CREDITMEMO) === false) {
            return false;
        }

        // Must have invoices to refund
        if ($this->getTotalInvoiced() <= 0) {
            return false;
        }

        // Check if any items can be refunded
        foreach ($this->getAllItems() as $item) {
            if ($item->getQtyToRefund() > 0) {
                return true;
            }
        }

        return false;
    }
}
```

### State Machine Pattern

Order state transitions are controlled through a state machine:

```php
<?php
namespace Magento\Sales\Model\Order;

/**
 * Order state machine configuration
 */
class Config
{
    /**
     * Statuses per state
     *
     * @var array
     */
    private array $stateStatuses = [
        Order::STATE_NEW => ['pending'],
        Order::STATE_PENDING_PAYMENT => ['pending_payment'],
        Order::STATE_PROCESSING => ['processing', 'fraud'],
        Order::STATE_COMPLETE => ['complete'],
        Order::STATE_CLOSED => ['closed'],
        Order::STATE_CANCELED => ['canceled'],
        Order::STATE_HOLDED => ['holded'],
        Order::STATE_PAYMENT_REVIEW => ['payment_review']
    ];

    /**
     * Get available states for order
     *
     * @return array
     */
    public function getStates(): array
    {
        return [
            Order::STATE_NEW => __('New'),
            Order::STATE_PENDING_PAYMENT => __('Pending Payment'),
            Order::STATE_PROCESSING => __('Processing'),
            Order::STATE_COMPLETE => __('Complete'),
            Order::STATE_CLOSED => __('Closed'),
            Order::STATE_CANCELED => __('Canceled'),
            Order::STATE_HOLDED => __('On Hold'),
            Order::STATE_PAYMENT_REVIEW => __('Payment Review')
        ];
    }

}
```

> **Note:** The `Config` class does not have an `isTransitionAllowed()` method. State transitions are enforced implicitly by the Order model's business methods (`canCancel()`, `canInvoice()`, `canShip()`, etc.), each of which checks the current state against allowed conditions. The following table shows the valid state transitions:
>
> | From State | Allowed Transitions |
> |-----------|-------------------|
> | `new` | `pending_payment`, `processing`, `canceled`, `holded` |
> | `pending_payment` | `processing`, `canceled`, `holded` |
> | `processing` | `complete`, `closed`, `canceled`, `holded` |
> | `holded` | `processing`, `canceled` |
> | `payment_review` | `processing`, `canceled` |
> | `complete` | `closed` |
> | `closed` | (terminal) |
> | `canceled` | (terminal) |

### Order State vs Status

Critical distinction in Sales architecture:

**State** (8 core states, non-extensible):
- System-level workflow state
- Controls what operations are possible (can invoice, can ship, etc.)
- Hard-coded in business logic
- Examples: `new`, `processing`, `complete`, `canceled`

**Status** (unlimited custom statuses):
- Merchant-facing display label
- Maps to a state
- Extensible via UI and code
- Examples: `pending`, `processing`, `complete`, `suspected_fraud`, `awaiting_approval`

```php
<?php
// Status to state mapping
$statusStateMap = [
    'pending' => Order::STATE_NEW,
    'pending_payment' => Order::STATE_PENDING_PAYMENT,
    'processing' => Order::STATE_PROCESSING,
    'suspected_fraud' => Order::STATE_PROCESSING, // Custom status
    'awaiting_approval' => Order::STATE_PROCESSING, // Custom status
    'complete' => Order::STATE_COMPLETE,
    'closed' => Order::STATE_CLOSED,
    'canceled' => Order::STATE_CANCELED,
    'holded' => Order::STATE_HOLDED,
    'payment_review' => Order::STATE_PAYMENT_REVIEW
];
```

## Invoice Architecture

### Invoice Entity Model

```php
<?php
namespace Magento\Sales\Model\Order;

use Magento\Sales\Api\Data\InvoiceInterface;

/**
 * Invoice entity
 *
 * Represents a payment request for an order or partial order
 */
class Invoice extends \Magento\Framework\Model\AbstractModel implements InvoiceInterface
{
    const STATE_OPEN = 1;
    const STATE_PAID = 2;
    const STATE_CANCELED = 3;

    const CAPTURE_ONLINE = 'online';
    const CAPTURE_OFFLINE = 'offline';
    const NOT_CAPTURE = 'not_capture';

    /**
     * Parent order
     *
     * @var \Magento\Sales\Model\Order
     */
    private \Magento\Sales\Model\Order $order;

    /**
     * Invoice items collection
     *
     * @var \Magento\Sales\Model\ResourceModel\Order\Invoice\Item\Collection
     */
    private $items;

    /**
     * Register invoice (prepare for save)
     *
     * Calculates totals and updates order
     *
     * @return $this
     */
    public function register(): self
    {
        if ($this->getId()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Cannot register an existing invoice.')
            );
        }

        // Calculate totals
        $this->collectTotals();

        // Update order totals
        $order = $this->getOrder();
        $order->setTotalInvoiced(
            $order->getTotalInvoiced() + $this->getGrandTotal()
        );
        $order->setBaseTotalInvoiced(
            $order->getBaseTotalInvoiced() + $this->getBaseGrandTotal()
        );

        // Update order items
        foreach ($this->getAllItems() as $item) {
            $orderItem = $item->getOrderItem();
            $orderItem->setQtyInvoiced(
                $orderItem->getQtyInvoiced() + $item->getQty()
            );
        }

        return $this;
    }

    /**
     * Pay invoice (capture payment)
     *
     * @return $this
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function pay(): self
    {
        if ($this->getState() !== self::STATE_OPEN) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Invoice is not in open state.')
            );
        }

        $order = $this->getOrder();
        $payment = $order->getPayment();

        if ($this->getRequestedCaptureCase() === self::CAPTURE_ONLINE) {
            // Capture payment online
            $payment->capture($this);
        }

        $this->setState(self::STATE_PAID);

        // Update order amounts paid
        $order->setTotalPaid(
            $order->getTotalPaid() + $this->getGrandTotal()
        );
        $order->setBaseTotalPaid(
            $order->getBaseTotalPaid() + $this->getBaseGrandTotal()
        );

        return $this;
    }

    /**
     * Cancel invoice
     *
     * @return $this
     */
    public function cancel(): self
    {
        if ($this->getState() === self::STATE_CANCELED) {
            return $this;
        }

        $this->setState(self::STATE_CANCELED);

        $order = $this->getOrder();

        // Revert order totals
        $order->setTotalInvoiced(
            $order->getTotalInvoiced() - $this->getGrandTotal()
        );
        $order->setBaseTotalInvoiced(
            $order->getBaseTotalInvoiced() - $this->getBaseGrandTotal()
        );

        // Revert order item quantities
        foreach ($this->getAllItems() as $item) {
            $orderItem = $item->getOrderItem();
            $orderItem->setQtyInvoiced(
                $orderItem->getQtyInvoiced() - $item->getQty()
            );
        }

        return $this;
    }

    /**
     * Collect invoice totals
     *
     * @return $this
     */
    public function collectTotals(): self
    {
        // Note: Invoice total collection uses the TotalCollectorList
        // (configured in di.xml), not events. There are no
        // sales_order_invoice_collect_totals_before/after events in core.

        // Collect totals from items
        $subtotal = 0;
        $baseSubtotal = 0;
        $taxAmount = 0;
        $baseTaxAmount = 0;

        foreach ($this->getAllItems() as $item) {
            $subtotal += $item->getRowTotal();
            $baseSubtotal += $item->getBaseRowTotal();
            $taxAmount += $item->getTaxAmount();
            $baseTaxAmount += $item->getBaseTaxAmount();
        }

        $this->setSubtotal($subtotal);
        $this->setBaseSubtotal($baseSubtotal);
        $this->setTaxAmount($taxAmount);
        $this->setBaseTaxAmount($baseTaxAmount);

        // Calculate grand total
        $grandTotal = $subtotal + $taxAmount + $this->getShippingAmount()
            - $this->getDiscountAmount();
        $baseGrandTotal = $baseSubtotal + $baseTaxAmount + $this->getBaseShippingAmount()
            - $this->getBaseDiscountAmount();

        $this->setGrandTotal($grandTotal);
        $this->setBaseGrandTotal($baseGrandTotal);

        return $this;
    }
}
```

### Invoice Service Layer

```php
<?php
namespace Magento\Sales\Model\Service;

use Magento\Sales\Api\InvoiceManagementInterface;
use Magento\Sales\Api\InvoiceRepositoryInterface;
use Magento\Sales\Model\Order\Invoice;

/**
 * Invoice management service
 *
 * High-level operations for invoice handling
 */
class InvoiceService implements InvoiceManagementInterface
{
    public function __construct(
        private InvoiceRepositoryInterface $invoiceRepository,
        private \Magento\Sales\Model\Order\Email\Sender\InvoiceSender $invoiceSender,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Capture invoice payment
     *
     * @param int $id Invoice ID
     * @return string Transaction ID
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function setCapture($id): string
    {
        $invoice = $this->invoiceRepository->get($id);

        if ($invoice->getState() !== Invoice::STATE_OPEN) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Invoice is not in open state.')
            );
        }

        $invoice->pay();
        $this->invoiceRepository->save($invoice);

        return $invoice->getOrder()->getPayment()->getLastTransId();
    }

    /**
     * Void invoice
     *
     * @param int $id Invoice ID
     * @return bool
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function setVoid($id): bool
    {
        $invoice = $this->invoiceRepository->get($id);

        if ($invoice->getState() === Invoice::STATE_PAID) {
            // Void payment
            $invoice->getOrder()->getPayment()->void($invoice);
        }

        $invoice->cancel();
        $this->invoiceRepository->save($invoice);

        return true;
    }

    /**
     * Notify customer about invoice
     *
     * @param int $id Invoice ID
     * @return bool
     */
    public function notify($id): bool
    {
        $invoice = $this->invoiceRepository->get($id);

        try {
            $this->invoiceSender->send($invoice);
            return true;
        } catch (\Exception $e) {
            $this->logger->error('Failed to send invoice email: ' . $e->getMessage());
            return false;
        }
    }

    /**
     * Get invoice comments
     *
     * @param int $id Invoice ID
     * @return \Magento\Sales\Api\Data\InvoiceCommentSearchResultInterface
     */
    public function getCommentsList($id)
    {
        $invoice = $this->invoiceRepository->get($id);
        return $invoice->getCommentsCollection();
    }
}
```

## Shipment Architecture

### Shipment Entity and Tracking

```php
<?php
namespace Magento\Sales\Model\Order;

use Magento\Sales\Api\Data\ShipmentInterface;

/**
 * Shipment entity
 *
 * Represents physical shipment of order items
 */
class Shipment extends \Magento\Framework\Model\AbstractModel implements ShipmentInterface
{
    /**
     * Shipment items
     *
     * @var \Magento\Sales\Model\ResourceModel\Order\Shipment\Item\Collection
     */
    private $items;

    /**
     * Tracking numbers
     *
     * @var \Magento\Sales\Model\ResourceModel\Order\Shipment\Track\Collection
     */
    private $tracks;

    /**
     * Register shipment (prepare for save)
     *
     * @return $this
     */
    public function register(): self
    {
        if ($this->getId()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Cannot register an existing shipment.')
            );
        }

        $order = $this->getOrder();

        // Update order items
        foreach ($this->getAllItems() as $item) {
            $orderItem = $item->getOrderItem();
            $orderItem->setQtyShipped(
                $orderItem->getQtyShipped() + $item->getQty()
            );
        }

        // Update order state if fully shipped
        if ($this->isOrderFullyShipped($order)) {
            $order->setState(\Magento\Sales\Model\Order::STATE_COMPLETE)
                ->setStatus($order->getConfig()->getStateDefaultStatus(
                    \Magento\Sales\Model\Order::STATE_COMPLETE
                ));
        }

        return $this;
    }

    /**
     * Add tracking number
     *
     * @param \Magento\Sales\Model\Order\Shipment\Track $track
     * @return $this
     */
    public function addTrack(Shipment\Track $track): self
    {
        $track->setShipment($this)
            ->setOrderId($this->getOrderId());

        if (!$track->getId()) {
            $this->getTracksCollection()->addItem($track);
        }

        return $this;
    }

    /**
     * Get all tracking numbers
     *
     * @return \Magento\Sales\Model\Order\Shipment\Track[]
     */
    public function getAllTracks(): array
    {
        return $this->getTracksCollection()->getItems();
    }

    /**
     * Check if order is fully shipped
     *
     * @param \Magento\Sales\Model\Order $order
     * @return bool
     */
    private function isOrderFullyShipped(\Magento\Sales\Model\Order $order): bool
    {
        foreach ($order->getAllItems() as $item) {
            if ($item->getIsVirtual()) {
                continue;
            }

            if ($item->getQtyToShip() > 0) {
                return false;
            }
        }

        return true;
    }
}
```

### Shipment Tracking

```php
<?php
namespace Magento\Sales\Model\Order\Shipment;

/**
 * Shipment tracking information
 */
class Track extends \Magento\Framework\Model\AbstractModel
{
    /**
     * Add tracking number to shipment
     *
     * @param string $carrier
     * @param string $title
     * @param string $trackNumber
     * @return $this
     */
    public function addData(string $carrier, string $title, string $trackNumber): self
    {
        $this->setCarrierCode($carrier)
            ->setTitle($title)
            ->setTrackNumber($trackNumber);

        return $this;
    }

    /**
     * Get tracking URL
     *
     * @return string|false
     */
    public function getNumberDetail()
    {
        $carrierInstance = $this->_carrierFactory->create(
            $this->getCarrierCode(),
            $this->getStoreId()
        );

        if (!$carrierInstance) {
            $custom = new \Magento\Framework\DataObject();
            $custom->setTracking($this->getTrackNumber());
            $custom->setCarrierTitle($this->getTitle());
            return $custom;
        }

        $carrierInstance->setStore($this->getStoreId());
        $trackingInfo = $carrierInstance->getTrackingInfo($this->getTrackNumber());

        return $trackingInfo;
    }
}
```

## Credit Memo Architecture

### Credit Memo Entity

```php
<?php
namespace Magento\Sales\Model\Order;

use Magento\Sales\Api\Data\CreditmemoInterface;

/**
 * Credit memo entity (refund)
 */
class Creditmemo extends \Magento\Framework\Model\AbstractModel implements CreditmemoInterface
{
    const STATE_OPEN = 1;
    const STATE_REFUNDED = 2;
    const STATE_CANCELED = 3;

    /**
     * Register credit memo (prepare for save)
     *
     * @return $this
     */
    public function register(): self
    {
        if ($this->getId()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Cannot register an existing credit memo.')
            );
        }

        // Collect totals
        $this->collectTotals();

        $order = $this->getOrder();

        // Update order totals
        $order->setTotalRefunded(
            $order->getTotalRefunded() + $this->getGrandTotal()
        );
        $order->setBaseTotalRefunded(
            $order->getBaseTotalRefunded() + $this->getBaseGrandTotal()
        );

        // Update order items
        foreach ($this->getAllItems() as $item) {
            $orderItem = $item->getOrderItem();
            $orderItem->setQtyRefunded(
                $orderItem->getQtyRefunded() + $item->getQty()
            );

            // Return to stock if requested
            if ($item->getBackToStock()) {
                $this->returnItemToStock($item);
            }
        }

        // Process refund
        if ($this->getInvoice() && $this->getInvoice()->getTransactionId()) {
            $order->getPayment()->refund($this);
        }

        // Update order state
        $this->updateOrderState($order);

        return $this;
    }

    /**
     * Update order state after credit memo
     *
     * @param \Magento\Sales\Model\Order $order
     * @return void
     */
    private function updateOrderState(\Magento\Sales\Model\Order $order): void
    {
        // Check if order is fully refunded
        if ($this->isOrderFullyRefunded($order)) {
            $order->setState(\Magento\Sales\Model\Order::STATE_CLOSED)
                ->setStatus($order->getConfig()->getStateDefaultStatus(
                    \Magento\Sales\Model\Order::STATE_CLOSED
                ));
        }
    }

    /**
     * Check if order is fully refunded
     *
     * @param \Magento\Sales\Model\Order $order
     * @return bool
     */
    private function isOrderFullyRefunded(\Magento\Sales\Model\Order $order): bool
    {
        $precision = 0.0001;
        return abs($order->getTotalInvoiced() - $order->getTotalRefunded()) < $precision;
    }

    /**
     * Return item to stock
     *
     * @param Creditmemo\Item $item
     * @return void
     */
    private function returnItemToStock(Creditmemo\Item $item): void
    {
        $this->_stockManagement->backItemQty(
            $item->getProductId(),
            $item->getQty(),
            $item->getOrderItem()->getStore()->getWebsiteId()
        );
    }

    /**
     * Collect credit memo totals
     *
     * @return $this
     */
    public function collectTotals(): self
    {
        // Note: Credit memo total collection uses the TotalCollectorList
        // (configured in di.xml), not events. There are no
        // sales_order_creditmemo_collect_totals_before/after events in core.

        // Collect totals from items
        $subtotal = 0;
        $baseSubtotal = 0;
        $taxAmount = 0;
        $baseTaxAmount = 0;

        foreach ($this->getAllItems() as $item) {
            $subtotal += $item->getRowTotal();
            $baseSubtotal += $item->getBaseRowTotal();
            $taxAmount += $item->getTaxAmount();
            $baseTaxAmount += $item->getBaseTaxAmount();
        }

        $this->setSubtotal($subtotal);
        $this->setBaseSubtotal($baseSubtotal);
        $this->setTaxAmount($taxAmount);
        $this->setBaseTaxAmount($baseTaxAmount);

        // Calculate grand total with adjustments
        $grandTotal = $subtotal + $taxAmount + $this->getShippingAmount()
            - $this->getDiscountAmount()
            + $this->getAdjustmentPositive()
            - $this->getAdjustmentNegative();

        $baseGrandTotal = $baseSubtotal + $baseTaxAmount + $this->getBaseShippingAmount()
            - $this->getBaseDiscountAmount()
            + $this->getBaseAdjustmentPositive()
            - $this->getBaseAdjustmentNegative();

        $this->setGrandTotal($grandTotal);
        $this->setBaseGrandTotal($baseGrandTotal);

        return $this;
    }
}
```

## Order Grid Architecture

### Grid Indexer

The order grid uses a separate flat table for performance:

```php
<?php
namespace Magento\Sales\Model\ResourceModel\Grid;

/**
 * Order grid indexer
 *
 * Maintains sales_order_grid table for fast admin queries
 */
class Collection extends \Magento\Framework\View\Element\UiComponent\DataProvider\SearchResult
{
    /**
     * Initialize grid collection
     */
    protected function _initSelect()
    {
        $this->getSelect()->from(['main_table' => $this->getMainTable()]);

        // Add additional fields
        $this->addFilterToMap('created_at', 'main_table.created_at');
        $this->addFilterToMap('status', 'main_table.status');
        $this->addFilterToMap('base_grand_total', 'main_table.base_grand_total');
        $this->addFilterToMap('grand_total', 'main_table.grand_total');
        $this->addFilterToMap('increment_id', 'main_table.increment_id');
        $this->addFilterToMap('billing_name', 'main_table.billing_name');
        $this->addFilterToMap('shipping_name', 'main_table.shipping_name');

        return $this;
    }
}
```

### Grid Update Mechanism

```php
<?php
namespace Magento\Sales\Model\ResourceModel\Grid;

/**
 * Update order grid on order changes
 */
class OrderGridRefresh
{
    public function __construct(
        private \Magento\Framework\App\ResourceConnection $resourceConnection
    ) {}

    /**
     * Refresh grid row for order
     *
     * @param int $orderId
     * @return void
     */
    public function refresh(int $orderId): void
    {
        $connection = $this->resourceConnection->getConnection();
        $gridTable = $this->resourceConnection->getTableName('sales_order_grid');
        $orderTable = $this->resourceConnection->getTableName('sales_order');
        $orderAddressTable = $this->resourceConnection->getTableName('sales_order_address');

        // Delete existing grid row
        $connection->delete($gridTable, ['entity_id = ?' => $orderId]);

        // Insert updated grid row
        $select = $connection->select()
            ->from(['main_table' => $orderTable], [
                'entity_id',
                'status',
                'store_id',
                'store_name',
                'customer_id',
                'base_grand_total',
                'base_total_paid',
                'grand_total',
                'total_paid',
                'increment_id',
                'base_currency_code',
                'order_currency_code',
                'shipping_name' => new \Zend_Db_Expr('NULL'),
                'billing_name' => new \Zend_Db_Expr('NULL'),
                'created_at',
                'updated_at',
            ]);

        // Add billing name
        $billingAddress = $connection->select()
            ->from($orderAddressTable, [
                'order_id',
                'billing_name' => new \Zend_Db_Expr(
                    "CONCAT_WS(' ', firstname, middlename, lastname)"
                )
            ])
            ->where('address_type = ?', 'billing');

        $select->joinLeft(
            ['billing' => new \Zend_Db_Expr('(' . $billingAddress . ')')],
            'main_table.entity_id = billing.order_id',
            ['billing_name']
        );

        // Add shipping name
        $shippingAddress = $connection->select()
            ->from($orderAddressTable, [
                'order_id',
                'shipping_name' => new \Zend_Db_Expr(
                    "CONCAT_WS(' ', firstname, middlename, lastname)"
                )
            ])
            ->where('address_type = ?', 'shipping');

        $select->joinLeft(
            ['shipping' => new \Zend_Db_Expr('(' . $shippingAddress . ')')],
            'main_table.entity_id = shipping.order_id',
            ['shipping_name']
        );

        $select->where('main_table.entity_id = ?', $orderId);

        $connection->query(
            $connection->insertFromSelect($select, $gridTable)
        );
    }
}
```

## Transaction Management

### Database Transactions

```php
<?php
namespace Magento\Sales\Model\Service;

use Magento\Sales\Api\OrderManagementInterface;
use Magento\Framework\DB\TransactionFactory;

/**
 * Order service with transaction handling
 */
class OrderService implements OrderManagementInterface
{
    public function __construct(
        private TransactionFactory $transactionFactory,
        private \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Place order with transaction
     *
     * @param \Magento\Quote\Api\Data\CartInterface $quote
     * @return \Magento\Sales\Api\Data\OrderInterface
     * @throws \Exception
     */
    public function place(\Magento\Quote\Api\Data\CartInterface $quote): \Magento\Sales\Api\Data\OrderInterface
    {
        $transaction = $this->transactionFactory->create();

        try {
            // Convert quote to order
            $order = $this->quoteManagement->submit($quote);

            // Add related entities to transaction
            $transaction->addObject($order);

            if ($order->getShippingAddress()) {
                $transaction->addObject($order->getShippingAddress());
            }

            if ($order->getBillingAddress()) {
                $transaction->addObject($order->getBillingAddress());
            }

            foreach ($order->getAllItems() as $item) {
                $transaction->addObject($item);
            }

            if ($order->getPayment()) {
                $transaction->addObject($order->getPayment());
            }

            // Save all entities in one transaction
            $transaction->save();

            $this->logger->info('Order placed successfully', [
                'order_id' => $order->getEntityId(),
                'increment_id' => $order->getIncrementId()
            ]);

            return $order;

        } catch (\Exception $e) {
            $this->logger->error('Order placement failed', [
                'quote_id' => $quote->getId(),
                'error' => $e->getMessage()
            ]);

            throw $e;
        }
    }
}
```

## Extension Attributes

### Order Extension Attributes

```xml
<!-- etc/extension_attributes.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Api/etc/extension_attributes.xsd">
    <extension_attributes for="Magento\Sales\Api\Data\OrderInterface">
        <attribute code="shipping_assignments" type="Magento\Sales\Api\Data\ShippingAssignmentInterface[]"/>
        <attribute code="payment_additional_info" type="Magento\Payment\Api\Data\PaymentAdditionalInfoInterface[]"/>
        <attribute code="applied_taxes" type="Magento\Tax\Api\Data\OrderTaxDetailsAppliedTaxInterface[]"/>
        <attribute code="item_applied_taxes" type="Magento\Tax\Api\Data\OrderTaxDetailsItemInterface[]"/>
        <attribute code="converting_from_quote" type="boolean"/>
        <attribute code="gift_message" type="Magento\GiftMessage\Api\Data\MessageInterface"/>
    </extension_attributes>

    <extension_attributes for="Magento\Sales\Api\Data\OrderItemInterface">
        <attribute code="gift_message" type="Magento\GiftMessage\Api\Data\MessageInterface"/>
        <attribute code="applied_taxes" type="Magento\Tax\Api\Data\OrderTaxDetailsItemInterface[]"/>
    </extension_attributes>
</config>
```

### Loading Extension Attributes

```php
<?php
namespace Vendor\Module\Plugin\Sales\Api;

use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Api\OrderRepositoryInterface;

/**
 * Plugin to load custom extension attributes
 */
class OrderRepositoryExtend
{
    public function __construct(
        private \Vendor\Module\Api\CustomDataRepositoryInterface $customDataRepository,
        private \Magento\Sales\Api\Data\OrderExtensionFactory $orderExtensionFactory
    ) {}

    /**
     * Load extension attributes after order get
     *
     * @param OrderRepositoryInterface $subject
     * @param OrderInterface $order
     * @return OrderInterface
     */
    public function afterGet(
        OrderRepositoryInterface $subject,
        OrderInterface $order
    ): OrderInterface {
        $this->loadExtensionAttributes($order);
        return $order;
    }

    /**
     * Load extension attributes after order list
     *
     * @param OrderRepositoryInterface $subject
     * @param \Magento\Sales\Api\Data\OrderSearchResultInterface $searchResult
     * @return \Magento\Sales\Api\Data\OrderSearchResultInterface
     */
    public function afterGetList(
        OrderRepositoryInterface $subject,
        \Magento\Sales\Api\Data\OrderSearchResultInterface $searchResult
    ): \Magento\Sales\Api\Data\OrderSearchResultInterface {
        foreach ($searchResult->getItems() as $order) {
            $this->loadExtensionAttributes($order);
        }

        return $searchResult;
    }

    /**
     * Load custom extension attributes
     *
     * @param OrderInterface $order
     * @return void
     */
    private function loadExtensionAttributes(OrderInterface $order): void
    {
        $extensionAttributes = $order->getExtensionAttributes()
            ?? $this->orderExtensionFactory->create();

        // Load custom data
        $customData = $this->customDataRepository->getByOrderId($order->getEntityId());
        $extensionAttributes->setCustomData($customData);

        $order->setExtensionAttributes($extensionAttributes);
    }
}
```

---

**Assumptions:**
- Adobe Commerce 2.4.7+ with PHP 8.2+
- MySQL 8.0+ for database operations
- Service contract interfaces are stable across minor versions
- Extension attributes follow framework conventions

**Why This Approach:**
- Service contracts provide API stability and web API exposure
- Repository pattern enables caching and identity map
- Flat tables for orders provide optimal grid performance
- State machine enforces business rules consistently
- Transaction management ensures data integrity
- Extension attributes enable non-invasive customization

**Security Impact:**
- All service operations protected by ACL resources
- Repository layer validates entity ownership (customer context)
- PII data (addresses, payment info) handled with encryption at rest
- API endpoints require authentication tokens
- Admin operations require admin session and CSRF tokens

**Performance Impact:**
- Order grid uses flat table with indexes (sub-second queries for millions of orders)
- Identity map in repositories prevents duplicate database queries
- Extension attributes lazy-loaded only when accessed
- Grid indexer can run async to avoid blocking order saves
- Collection queries use proper joins to avoid N+1 problems

**Backward Compatibility:**
- Service contract interfaces stable; internal implementations can change
- Extension attributes don't modify core tables
- Plugins at service layer don't break with core updates
- Database schema changes through declarative schema (backwards compatible)

**Tests to Add:**
- Unit tests: State transitions, business rules (canInvoice, canShip, etc.)
- Integration tests: Repository operations, transaction handling
- API tests: REST/GraphQL endpoint contracts
- Functional tests: End-to-end order lifecycle workflows

**Docs to Update:**
- README.md: High-level service contract usage examples
- EXECUTION_FLOWS.md: Detailed workflow diagrams with this architecture
- Extension attributes usage guide for third-party developers
