---
title: "Magento_Sales Anti-Patterns"
module: "Magento_Sales"
doc_type: "anti-patterns"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Sales Anti-Patterns

## Overview

This document catalogs common anti-patterns, mistakes, and problematic approaches when working with the Magento_Sales module. Each anti-pattern includes the problematic code, explanation of why it's wrong, the correct approach, and real-world consequences of using the anti-pattern.

## Direct Model Manipulation Anti-Patterns

### Anti-Pattern 1: Direct Order Save Without Service Contracts

**Problematic Code:**

```php
<?php
// BAD: Direct model manipulation bypasses business logic
$order = $this->orderFactory->create()->load($orderId);
$order->setState(\Magento\Sales\Model\Order::STATE_COMPLETE);
$order->setStatus('complete');
$order->save();
```

**Why This Is Wrong:**
- Bypasses all plugins and observers
- Skips state validation (canComplete(), state machine rules)
- No event dispatching (sales_order_save_before/after)
- No transaction handling - partial saves possible
- Breaks extension attribute handling
- No API compatibility - direct saves don't work via REST/GraphQL

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Sales\Api\OrderManagementInterface;
use Magento\Framework\Exception\LocalizedException;

/**
 * Proper order state management
 */
class OrderStateManagement
{
    public function __construct(
        private OrderRepositoryInterface $orderRepository,
        private OrderManagementInterface $orderManagement
    ) {}

    /**
     * Complete order using service contracts
     *
     * @param int $orderId
     * @return void
     * @throws LocalizedException
     */
    public function completeOrder(int $orderId): void
    {
        $order = $this->orderRepository->get($orderId);

        // Validate state transition is allowed
        if ($order->getState() === \Magento\Sales\Model\Order::STATE_COMPLETE) {
            throw new LocalizedException(__('Order is already complete.'));
        }

        if (!$order->canShip() && !$order->canInvoice()) {
            // Order can be completed - all items shipped/invoiced
            $order->setState(\Magento\Sales\Model\Order::STATE_COMPLETE);
            $order->setStatus(
                $order->getConfig()->getStateDefaultStatus(
                    \Magento\Sales\Model\Order::STATE_COMPLETE
                )
            );

            // Use repository save - triggers all plugins and events
            $this->orderRepository->save($order);
        } else {
            throw new LocalizedException(
                __('Order cannot be completed - items pending shipment/invoice.')
            );
        }
    }
}
```

**Real-World Consequences:**
- Integration failures: Third-party modules miss order state changes
- Inventory issues: Stock not updated because observers didn't fire
- Payment problems: Payment gateway not notified of completion
- Reporting errors: Analytics/BI systems miss state changes
- Audit trail gaps: No history comments generated

---

### Anti-Pattern 2: Direct Order Status Change Without State

**Problematic Code:**

```php
<?php
// BAD: Changing status without updating state
$order = $this->orderFactory->create()->load($orderId);
$order->setStatus('custom_status');
$order->save();
// State remains old value - inconsistent!
```

**Why This Is Wrong:**
- State and status become inconsistent
- Business logic checks state, not status
- canInvoice(), canShip() checks fail
- Order workflow breaks
- Admin UI shows wrong available actions

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Sales\Model\Order;

/**
 * Proper status/state management
 */
class OrderStatusManager
{
    /**
     * Set custom status while maintaining state consistency
     *
     * @param Order $order
     * @param string $status
     * @param string $comment
     * @return void
     */
    public function setOrderStatus(
        Order $order,
        string $status,
        string $comment = ''
    ): void {
        // Validate status exists and maps to current state
        $statusState = $order->getConfig()->getStateDefaultStatus($order->getState());
        $availableStatuses = $order->getConfig()->getStateStatuses($order->getState());

        if (!in_array($status, $availableStatuses, true)) {
            throw new \InvalidArgumentException(
                sprintf(
                    'Status "%s" is not valid for state "%s"',
                    $status,
                    $order->getState()
                )
            );
        }

        // Set status (state remains unchanged)
        $order->setStatus($status);

        // Add status history comment
        if ($comment) {
            $order->addCommentToStatusHistory(
                $comment,
                $status,
                false // Not visible to customer
            );
        }

        $this->orderRepository->save($order);
    }

    /**
     * Create custom status mapped to processing state
     *
     * This should be done via data patch, not runtime code
     */
    public function createCustomStatus(): void
    {
        // Example: Add "awaiting_approval" status to "processing" state
        $status = $this->statusFactory->create();
        $status->setData([
            'status' => 'awaiting_approval',
            'label' => 'Awaiting Approval'
        ]);

        $this->statusResource->save($status);

        // Assign to processing state
        $status->assignState(
            Order::STATE_PROCESSING,
            false, // Not default
            true   // Visible on storefront
        );
    }
}
```

---

### Anti-Pattern 3: Direct Collection Load Without Search Criteria

**Problematic Code:**

```php
<?php
// BAD: Direct collection usage bypasses API contracts
$orderCollection = $this->orderCollectionFactory->create();
$orderCollection->addFieldToFilter('customer_id', $customerId);
$orderCollection->addFieldToFilter('status', 'processing');
$orderCollection->load();

foreach ($orderCollection as $order) {
    // Process orders
}
```

**Why This Is Wrong:**
- Not compatible with Web API
- Bypasses collection processors (ACL, store scope)
- No extension attribute loading
- Breaks when collection implementation changes
- Doesn't support GraphQL resolvers
- Can't add custom filters via plugins

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Api\SortOrderBuilder;

/**
 * Order search service using search criteria
 */
class OrderSearchService
{
    public function __construct(
        private OrderRepositoryInterface $orderRepository,
        private SearchCriteriaBuilder $searchCriteriaBuilder,
        private SortOrderBuilder $sortOrderBuilder
    ) {}

    /**
     * Get customer orders with filters
     *
     * @param int $customerId
     * @param array $statuses
     * @param int $pageSize
     * @param int $currentPage
     * @return \Magento\Sales\Api\Data\OrderSearchResultInterface
     */
    public function getCustomerOrders(
        int $customerId,
        array $statuses = [],
        int $pageSize = 20,
        int $currentPage = 1
    ): \Magento\Sales\Api\Data\OrderSearchResultInterface {
        $this->searchCriteriaBuilder
            ->addFilter('customer_id', $customerId, 'eq');

        if (!empty($statuses)) {
            $this->searchCriteriaBuilder
                ->addFilter('status', $statuses, 'in');
        }

        // Add sort order
        $sortOrder = $this->sortOrderBuilder
            ->setField('created_at')
            ->setDirection('DESC')
            ->create();

        $searchCriteria = $this->searchCriteriaBuilder
            ->addSortOrder($sortOrder)
            ->setPageSize($pageSize)
            ->setCurrentPage($currentPage)
            ->create();

        return $this->orderRepository->getList($searchCriteria);
    }

    /**
     * Get orders by date range
     *
     * @param string $fromDate
     * @param string $toDate
     * @return \Magento\Sales\Api\Data\OrderSearchResultInterface
     */
    public function getOrdersByDateRange(
        string $fromDate,
        string $toDate
    ): \Magento\Sales\Api\Data\OrderSearchResultInterface {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('created_at', $fromDate, 'gteq')
            ->addFilter('created_at', $toDate, 'lteq')
            ->create();

        return $this->orderRepository->getList($searchCriteria);
    }
}
```

---

## Invoice/Shipment/Credit Memo Anti-Patterns

### Anti-Pattern 4: Manual Invoice Creation Without Validation

**Problematic Code:**

```php
<?php
// BAD: Creating invoice without proper validation
$invoice = $this->invoiceFactory->create();
$invoice->setOrderId($orderId);
$invoice->setState(\Magento\Sales\Model\Order\Invoice::STATE_PAID);
$invoice->setGrandTotal($amount);
$invoice->save();
```

**Why This Is Wrong:**
- No order validation (canInvoice check skipped)
- Invoice items not created
- Order totals not updated
- No payment capture
- No customer notification
- Inventory not updated

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Sales\Api\Data\InvoiceInterface;
use Magento\Sales\Model\Order;
use Magento\Framework\Exception\LocalizedException;

/**
 * Proper invoice creation service
 */
class InvoiceCreationService
{
    public function __construct(
        private \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        private \Magento\Sales\Model\Service\InvoiceService $invoiceService,
        private \Magento\Framework\DB\TransactionFactory $transactionFactory,
        private \Magento\Sales\Model\Order\Email\Sender\InvoiceSender $invoiceSender,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Create invoice properly
     *
     * @param int $orderId
     * @param array $items Item ID => Qty to invoice
     * @param bool $captureOnline
     * @param bool $notify
     * @return InvoiceInterface
     * @throws LocalizedException
     */
    public function createInvoice(
        int $orderId,
        array $items = [],
        bool $captureOnline = true,
        bool $notify = true
    ): InvoiceInterface {
        $order = $this->orderRepository->get($orderId);

        // Validate order can be invoiced
        if (!$order->canInvoice()) {
            throw new LocalizedException(
                __('The order does not allow an invoice to be created.')
            );
        }

        // Use InvoiceService for proper invoice creation
        $invoice = $this->invoiceService->prepareInvoice($order, $items);

        if (!$invoice->getTotalQty()) {
            throw new LocalizedException(
                __('You can\'t create an invoice without products.')
            );
        }

        // Set capture case
        $invoice->setRequestedCaptureCase(
            $captureOnline
                ? \Magento\Sales\Model\Order\Invoice::CAPTURE_ONLINE
                : \Magento\Sales\Model\Order\Invoice::NOT_CAPTURE
        );

        // Register invoice (updates order, creates invoice items)
        $invoice->register();

        // Save invoice and order in transaction
        $transaction = $this->transactionFactory->create();
        $transaction->addObject($invoice)
            ->addObject($order)
            ->save();

        // Send customer notification
        if ($notify) {
            try {
                $this->invoiceSender->send($invoice);
            } catch (\Exception $e) {
                $this->logger->error('Failed to send invoice email', [
                    'invoice_id' => $invoice->getEntityId(),
                    'error' => $e->getMessage()
                ]);
            }
        }

        return $invoice;
    }
}
```

---

### Anti-Pattern 5: Shipment Without Order Item Validation

**Problematic Code:**

```php
<?php
// BAD: Creating shipment without checking available quantities
$shipment = $this->shipmentFactory->create();
$shipment->setOrderId($orderId);

foreach ($orderItems as $itemId => $qty) {
    $shipmentItem = $this->shipmentItemFactory->create();
    $shipmentItem->setOrderItemId($itemId);
    $shipmentItem->setQty($qty); // No validation!
    $shipment->addItem($shipmentItem);
}

$shipment->save();
```

**Why This Is Wrong:**
- Can ship more than ordered
- Can ship already shipped items
- Can ship canceled items
- Can ship virtual items
- Order state not updated
- Customer not notified

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Sales\Api\Data\ShipmentInterface;
use Magento\Framework\Exception\LocalizedException;

/**
 * Proper shipment creation with validation
 */
class ShipmentCreationService
{
    public function __construct(
        private \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        private \Magento\Sales\Model\Order\ShipmentFactory $shipmentFactory,
        private \Magento\Sales\Model\Order\Shipment\TrackFactory $trackFactory,
        private \Magento\Framework\DB\TransactionFactory $transactionFactory,
        private \Magento\Sales\Model\Order\Email\Sender\ShipmentSender $shipmentSender,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Create shipment with proper validation
     *
     * @param int $orderId
     * @param array $items Item ID => Qty
     * @param array $tracking Tracking info
     * @param bool $notify
     * @return ShipmentInterface
     * @throws LocalizedException
     */
    public function createShipment(
        int $orderId,
        array $items = [],
        array $tracking = [],
        bool $notify = true
    ): ShipmentInterface {
        $order = $this->orderRepository->get($orderId);

        // Validate order can ship
        if (!$order->canShip()) {
            throw new LocalizedException(
                __('Cannot create shipment for this order.')
            );
        }

        // Validate and prepare items
        $validatedItems = $this->validateShipmentItems($order, $items);

        // Create shipment with validated items
        $shipment = $this->shipmentFactory->create(
            $order,
            $validatedItems
        );

        if (!$shipment->getTotalQty()) {
            throw new LocalizedException(
                __('Cannot create shipment without products.')
            );
        }

        // Add tracking information
        foreach ($tracking as $trackData) {
            $track = $this->trackFactory->create();
            $track->setNumber($trackData['number'] ?? '')
                ->setCarrierCode($trackData['carrier_code'] ?? '')
                ->setTitle($trackData['title'] ?? '');

            $shipment->addTrack($track);
        }

        // Register shipment (updates order quantities)
        $shipment->register();

        // Save in transaction
        $transaction = $this->transactionFactory->create();
        $transaction->addObject($shipment)
            ->addObject($order)
            ->save();

        // Send notification
        if ($notify) {
            try {
                $this->shipmentSender->send($shipment);
            } catch (\Exception $e) {
                $this->logger->error('Failed to send shipment email', [
                    'shipment_id' => $shipment->getEntityId(),
                    'error' => $e->getMessage()
                ]);
            }
        }

        return $shipment;
    }

    /**
     * Validate shipment items
     *
     * @param \Magento\Sales\Model\Order $order
     * @param array $items
     * @return array
     * @throws LocalizedException
     */
    private function validateShipmentItems(
        \Magento\Sales\Model\Order $order,
        array $items
    ): array {
        $validatedItems = [];

        foreach ($items as $itemId => $qty) {
            $orderItem = $order->getItemById($itemId);

            if (!$orderItem) {
                throw new LocalizedException(
                    __('Order item %1 not found.', $itemId)
                );
            }

            // Check if item is virtual
            if ($orderItem->getIsVirtual()) {
                throw new LocalizedException(
                    __('Cannot ship virtual item %1.', $orderItem->getSku())
                );
            }

            // Check available quantity
            $qtyAvailable = $orderItem->getQtyToShip();
            if ($qty > $qtyAvailable) {
                throw new LocalizedException(
                    __(
                        'Cannot ship %1 of item %2. Only %3 available.',
                        $qty,
                        $orderItem->getSku(),
                        $qtyAvailable
                    )
                );
            }

            // Check item is not canceled
            if ($orderItem->getQtyCanceled() > 0 &&
                $orderItem->getQtyToShip() <= 0) {
                throw new LocalizedException(
                    __('Cannot ship canceled item %1.', $orderItem->getSku())
                );
            }

            $validatedItems[$itemId] = $qty;
        }

        return $validatedItems;
    }
}
```

---

## Payment Processing Anti-Patterns

### Anti-Pattern 6: Direct Payment Capture Without Gateway Communication

**Problematic Code:**

```php
<?php
// BAD: Marking payment as captured without gateway interaction
$order = $this->orderFactory->create()->load($orderId);
$payment = $order->getPayment();
$payment->setAmountPaid($order->getGrandTotal());
$payment->save();
$order->save();
```

**Why This Is Wrong:**
- No actual payment capture from gateway
- Payment gateway and Magento out of sync
- Funds never actually captured
- No transaction record created
- Fraud detection bypassed
- Reconciliation impossible

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Sales\Model\Order;
use Magento\Framework\Exception\LocalizedException;

/**
 * Proper payment capture service
 */
class PaymentCaptureService
{
    public function __construct(
        private \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        private \Magento\Sales\Api\InvoiceManagementInterface $invoiceManagement,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Capture payment properly through gateway
     *
     * @param int $orderId
     * @return bool
     * @throws LocalizedException
     */
    public function capturePayment(int $orderId): bool
    {
        $order = $this->orderRepository->get($orderId);

        // Validate order can be invoiced
        if (!$order->canInvoice()) {
            throw new LocalizedException(
                __('Order cannot be invoiced.')
            );
        }

        $payment = $order->getPayment();
        $methodInstance = $payment->getMethodInstance();

        // Validate payment method supports capture
        if (!$methodInstance->canCapture()) {
            throw new LocalizedException(
                __('Payment method does not support capture.')
            );
        }

        try {
            // Create invoice with online capture
            // This properly calls payment gateway
            $invoice = $this->invoiceService->prepareInvoice($order);
            $invoice->setRequestedCaptureCase(
                \Magento\Sales\Model\Order\Invoice::CAPTURE_ONLINE
            );
            $invoice->register();

            // Payment capture happens during invoice pay()
            $invoice->pay();

            // Save invoice and order
            $transaction = $this->transactionFactory->create();
            $transaction->addObject($invoice)
                ->addObject($order)
                ->save();

            $this->logger->info('Payment captured successfully', [
                'order_id' => $order->getEntityId(),
                'transaction_id' => $payment->getLastTransId(),
                'amount' => $invoice->getGrandTotal()
            ]);

            return true;

        } catch (\Exception $e) {
            $this->logger->error('Payment capture failed', [
                'order_id' => $order->getEntityId(),
                'error' => $e->getMessage()
            ]);

            throw new LocalizedException(
                __('Payment capture failed: %1', $e->getMessage())
            );
        }
    }
}
```

---

## Order State Machine Anti-Patterns

### Anti-Pattern 7: Bypassing State Validation

**Problematic Code:**

```php
<?php
// BAD: Force state change without validation
$order->setState(\Magento\Sales\Model\Order::STATE_COMPLETE);
$order->save();
// Order may have pending shipments or invoices!
```

**Why This Is Wrong:**
- State inconsistent with order operations
- Business logic broken (canShip/canInvoice return wrong values)
- Workflow automation breaks
- Reports show incorrect data
- Customer sees wrong order status

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Sales\Model\Order;
use Magento\Framework\Exception\LocalizedException;

/**
 * Order state machine with proper validation
 */
class OrderStateMachine
{
    public function __construct(
        private \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Transition order to complete state with validation
     *
     * @param int $orderId
     * @return void
     * @throws LocalizedException
     */
    public function completeOrder(int $orderId): void
    {
        $order = $this->orderRepository->get($orderId);

        // Validate current state allows completion
        $validFromStates = [
            Order::STATE_PROCESSING,
            Order::STATE_PAYMENT_REVIEW
        ];

        if (!in_array($order->getState(), $validFromStates, true)) {
            throw new LocalizedException(
                __(
                    'Cannot complete order from state "%1".',
                    $order->getState()
                )
            );
        }

        // Validate all items are fully processed
        if (!$this->isOrderFullyProcessed($order)) {
            throw new LocalizedException(
                __('Order has pending shipments or invoices.')
            );
        }

        // Transition state
        $order->setState(Order::STATE_COMPLETE);
        $order->setStatus(
            $order->getConfig()->getStateDefaultStatus(Order::STATE_COMPLETE)
        );

        $order->addCommentToStatusHistory(
            'Order completed - all items shipped and invoiced.',
            false,
            false
        );

        $this->orderRepository->save($order);

        $this->logger->info('Order completed', [
            'order_id' => $order->getEntityId(),
            'from_state' => $order->getOrigData('state'),
            'to_state' => $order->getState()
        ]);
    }

    /**
     * Check if order is fully processed
     *
     * @param Order $order
     * @return bool
     */
    private function isOrderFullyProcessed(Order $order): bool
    {
        // Check all items are invoiced
        foreach ($order->getAllItems() as $item) {
            if ($item->getQtyToInvoice() > 0) {
                return false;
            }
        }

        // Check all items are shipped (for physical products)
        foreach ($order->getAllItems() as $item) {
            if (!$item->getIsVirtual() && $item->getQtyToShip() > 0) {
                return false;
            }
        }

        return true;
    }

    /**
     * Valid state transition map
     *
     * @return array
     */
    private function getValidTransitions(): array
    {
        return [
            Order::STATE_NEW => [
                Order::STATE_PENDING_PAYMENT,
                Order::STATE_PROCESSING,
                Order::STATE_HOLDED,
                Order::STATE_CANCELED
            ],
            Order::STATE_PENDING_PAYMENT => [
                Order::STATE_PROCESSING,
                Order::STATE_HOLDED,
                Order::STATE_CANCELED
            ],
            Order::STATE_PROCESSING => [
                Order::STATE_COMPLETE,
                Order::STATE_HOLDED,
                Order::STATE_CANCELED,
                Order::STATE_CLOSED
            ],
            Order::STATE_HOLDED => [
                Order::STATE_PROCESSING,
                Order::STATE_CANCELED
            ],
            Order::STATE_PAYMENT_REVIEW => [
                Order::STATE_PROCESSING,
                Order::STATE_CANCELED
            ],
            Order::STATE_COMPLETE => [
                Order::STATE_CLOSED
            ],
            Order::STATE_CLOSED => [],
            Order::STATE_CANCELED => []
        ];
    }

    /**
     * Validate state transition
     *
     * @param string $fromState
     * @param string $toState
     * @return bool
     */
    public function isTransitionValid(string $fromState, string $toState): bool
    {
        $validTransitions = $this->getValidTransitions();
        return in_array(
            $toState,
            $validTransitions[$fromState] ?? [],
            true
        );
    }
}
```

---

## Performance Anti-Patterns

### Anti-Pattern 8: Loading Full Order in Loop

**Problematic Code:**

```php
<?php
// BAD: N+1 query problem
$orderIds = [1, 2, 3, 4, 5, /* ... 1000 IDs */];

foreach ($orderIds as $orderId) {
    $order = $this->orderRepository->get($orderId); // 1000 queries!
    $total = $order->getGrandTotal();
    $items = $order->getAllItems(); // 1000 more queries!
    // Process order
}
```

**Why This Is Wrong:**
- 2000+ database queries for 1000 orders
- Extremely slow (seconds to minutes)
- High database load
- Memory inefficient
- Doesn't scale

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Sales\Api\OrderRepositoryInterface;

/**
 * Efficient bulk order loading
 */
class BulkOrderLoader
{
    public function __construct(
        private OrderRepositoryInterface $orderRepository,
        private SearchCriteriaBuilder $searchCriteriaBuilder,
        private \Magento\Framework\App\ResourceConnection $resourceConnection
    ) {}

    /**
     * Load multiple orders efficiently
     *
     * @param array $orderIds
     * @return array
     */
    public function loadOrders(array $orderIds): array
    {
        // Use search criteria with IN filter
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('entity_id', $orderIds, 'in')
            ->create();

        $searchResult = $this->orderRepository->getList($searchCriteria);

        return $searchResult->getItems();
    }

    /**
     * Get order totals efficiently (single query)
     *
     * @param array $orderIds
     * @return array [order_id => grand_total]
     */
    public function getOrderTotals(array $orderIds): array
    {
        $connection = $this->resourceConnection->getConnection();
        $select = $connection->select()
            ->from(
                $this->resourceConnection->getTableName('sales_order'),
                ['entity_id', 'grand_total']
            )
            ->where('entity_id IN (?)', $orderIds);

        return $connection->fetchPairs($select);
    }

    /**
     * Process orders in batches
     *
     * @param array $orderIds
     * @param callable $processor
     * @param int $batchSize
     * @return void
     */
    public function processBatch(
        array $orderIds,
        callable $processor,
        int $batchSize = 100
    ): void {
        $batches = array_chunk($orderIds, $batchSize);

        foreach ($batches as $batch) {
            $orders = $this->loadOrders($batch);

            foreach ($orders as $order) {
                $processor($order);
            }

            // Clear memory after each batch
            unset($orders);
            gc_collect_cycles();
        }
    }
}
```

---

### Anti-Pattern 9: Loading Entire Order Collection Without Limit

**Problematic Code:**

```php
<?php
// BAD: Loading all orders into memory
$orderCollection = $this->orderCollectionFactory->create();
$orderCollection->addFieldToFilter('status', 'processing');
// No page size set - loads ALL orders!

foreach ($orderCollection as $order) {
    // Process order
}
// Out of memory error with 10,000+ orders
```

**Why This Is Wrong:**
- Memory exhaustion with large datasets
- Long execution time
- Database overload
- PHP timeout
- Potential server crash

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

/**
 * Efficient order batch processing
 */
class OrderBatchProcessor
{
    private const BATCH_SIZE = 100;

    public function __construct(
        private \Magento\Sales\Model\ResourceModel\Order\CollectionFactory $orderCollectionFactory,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Process orders in batches
     *
     * @param array $filters
     * @param callable $processor
     * @return int Total processed
     */
    public function processOrders(array $filters, callable $processor): int
    {
        $currentPage = 1;
        $totalProcessed = 0;

        do {
            // Load one page of orders
            $collection = $this->orderCollectionFactory->create();

            foreach ($filters as $field => $value) {
                $collection->addFieldToFilter($field, $value);
            }

            $collection->setPageSize(self::BATCH_SIZE);
            $collection->setCurPage($currentPage);
            $collection->load();

            // Process batch
            foreach ($collection as $order) {
                try {
                    $processor($order);
                    $totalProcessed++;
                } catch (\Exception $e) {
                    $this->logger->error('Order processing failed', [
                        'order_id' => $order->getId(),
                        'error' => $e->getMessage()
                    ]);
                }
            }

            // Clear collection from memory
            $collection->clear();
            unset($collection);

            $currentPage++;

        } while ($currentPage <= $collection->getLastPageNumber());

        return $totalProcessed;
    }

    /**
     * Process orders with iterator (memory efficient)
     *
     * @param array $filters
     * @param callable $processor
     * @return int
     */
    public function processWithIterator(array $filters, callable $processor): int
    {
        $connection = $this->resourceConnection->getConnection();
        $select = $connection->select()
            ->from($this->resourceConnection->getTableName('sales_order'), ['entity_id']);

        foreach ($filters as $field => $value) {
            $select->where("$field = ?", $value);
        }

        // Fetch IDs only (minimal memory)
        $orderIds = $connection->fetchCol($select);
        $totalProcessed = 0;

        // Process in batches
        foreach (array_chunk($orderIds, self::BATCH_SIZE) as $batch) {
            $orders = $this->loadOrders($batch);

            foreach ($orders as $order) {
                try {
                    $processor($order);
                    $totalProcessed++;
                } catch (\Exception $e) {
                    $this->logger->error('Order processing failed', [
                        'order_id' => $order->getId(),
                        'error' => $e->getMessage()
                    ]);
                }
            }

            unset($orders);
            gc_collect_cycles();
        }

        return $totalProcessed;
    }
}
```

---

## Transaction Handling Anti-Patterns

### Anti-Pattern 10: No Transaction for Multi-Entity Operations

**Problematic Code:**

```php
<?php
// BAD: Saving related entities without transaction
$order->setState(\Magento\Sales\Model\Order::STATE_COMPLETE);
$order->save(); // Saved

$invoice->setState(\Magento\Sales\Model\Order\Invoice::STATE_PAID);
$invoice->save(); // If this fails, order already changed!

$shipment->register();
$shipment->save(); // Inconsistent state if this fails
```

**Why This Is Wrong:**
- Partial saves leave inconsistent state
- No rollback on errors
- Data integrity compromised
- Orphaned records
- Impossible to recover

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Framework\DB\TransactionFactory;
use Magento\Framework\Exception\LocalizedException;

/**
 * Proper transaction handling for multi-entity operations
 */
class TransactionalOrderService
{
    public function __construct(
        private TransactionFactory $transactionFactory,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Complete order with invoice and shipment atomically
     *
     * @param \Magento\Sales\Model\Order $order
     * @param \Magento\Sales\Model\Order\Invoice $invoice
     * @param \Magento\Sales\Model\Order\Shipment $shipment
     * @return void
     * @throws LocalizedException
     */
    public function completeOrderWithInvoiceAndShipment(
        \Magento\Sales\Model\Order $order,
        \Magento\Sales\Model\Order\Invoice $invoice,
        \Magento\Sales\Model\Order\Shipment $shipment
    ): void {
        // Begin transaction
        $transaction = $this->transactionFactory->create();

        try {
            // Add all entities to transaction
            $transaction->addObject($order);
            $transaction->addObject($invoice);
            $transaction->addObject($shipment);

            // Add invoice items
            foreach ($invoice->getAllItems() as $item) {
                $transaction->addObject($item);
            }

            // Add shipment items
            foreach ($shipment->getAllItems() as $item) {
                $transaction->addObject($item);
            }

            // Validate all operations
            if (!$order->canInvoice()) {
                throw new LocalizedException(__('Order cannot be invoiced.'));
            }

            if (!$order->canShip()) {
                throw new LocalizedException(__('Order cannot be shipped.'));
            }

            // Update order state
            $order->setState(\Magento\Sales\Model\Order::STATE_COMPLETE);
            $order->setStatus($order->getConfig()->getStateDefaultStatus(
                \Magento\Sales\Model\Order::STATE_COMPLETE
            ));

            // Register invoice
            $invoice->register();
            $invoice->pay();

            // Register shipment
            $shipment->register();

            // Save all atomically
            $transaction->save();

            $this->logger->info('Order completed with invoice and shipment', [
                'order_id' => $order->getId(),
                'invoice_id' => $invoice->getId(),
                'shipment_id' => $shipment->getId()
            ]);

        } catch (\Exception $e) {
            $this->logger->error('Transaction failed', [
                'order_id' => $order->getId(),
                'error' => $e->getMessage()
            ]);

            // Transaction automatically rolled back
            throw new LocalizedException(
                __('Failed to complete order: %1', $e->getMessage())
            );
        }
    }
}
```

---

## Data Integrity Anti-Patterns

### Anti-Pattern 11: Modifying Order Totals Without Recalculation

**Problematic Code:**

```php
<?php
// BAD: Changing item price without recalculating order totals
$orderItem = $order->getItemById($itemId);
$orderItem->setPrice(99.99);
$orderItem->save();
// Order grand_total now incorrect!
```

**Why This Is Wrong:**
- Order totals incorrect
- Tax amount wrong
- Discounts not recalculated
- Subtotal inconsistent
- Invoices/refunds fail

**Correct Approach:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Sales\Model\Order;
use Magento\Framework\Exception\LocalizedException;

/**
 * Proper order total recalculation
 */
class OrderTotalRecalculation
{
    /**
     * Update item price and recalculate order
     *
     * @param Order $order
     * @param int $itemId
     * @param float $newPrice
     * @return void
     * @throws LocalizedException
     */
    public function updateItemPrice(
        Order $order,
        int $itemId,
        float $newPrice
    ): void {
        // Validate order state allows modification
        if ($order->getState() !== Order::STATE_PENDING_PAYMENT) {
            throw new LocalizedException(
                __('Cannot modify order items after payment.')
            );
        }

        $orderItem = $order->getItemById($itemId);
        if (!$orderItem) {
            throw new LocalizedException(__('Order item not found.'));
        }

        // Update item prices
        $orderItem->setPrice($newPrice);
        $orderItem->setBasePrice($newPrice);
        $orderItem->setRowTotal($newPrice * $orderItem->getQtyOrdered());
        $orderItem->setBaseRowTotal($newPrice * $orderItem->getQtyOrdered());

        // Recalculate order totals
        $this->recalculateOrderTotals($order);

        // Save order
        $this->orderRepository->save($order);
    }

    /**
     * Recalculate all order totals
     *
     * @param Order $order
     * @return void
     */
    private function recalculateOrderTotals(Order $order): void
    {
        $subtotal = 0;
        $baseSubtotal = 0;
        $taxAmount = 0;
        $baseTaxAmount = 0;
        $discountAmount = 0;
        $baseDiscountAmount = 0;

        // Sum item totals
        foreach ($order->getAllVisibleItems() as $item) {
            $subtotal += $item->getRowTotal();
            $baseSubtotal += $item->getBaseRowTotal();
            $taxAmount += $item->getTaxAmount();
            $baseTaxAmount += $item->getBaseTaxAmount();
            $discountAmount += abs($item->getDiscountAmount());
            $baseDiscountAmount += abs($item->getBaseDiscountAmount());
        }

        // Update order totals
        $order->setSubtotal($subtotal);
        $order->setBaseSubtotal($baseSubtotal);
        $order->setTaxAmount($taxAmount);
        $order->setBaseTaxAmount($baseTaxAmount);
        $order->setDiscountAmount(-$discountAmount);
        $order->setBaseDiscountAmount(-$baseDiscountAmount);

        // Calculate grand total
        $grandTotal = $subtotal
            + $taxAmount
            + $order->getShippingAmount()
            - $discountAmount;

        $baseGrandTotal = $baseSubtotal
            + $baseTaxAmount
            + $order->getBaseShippingAmount()
            - $baseDiscountAmount;

        $order->setGrandTotal($grandTotal);
        $order->setBaseGrandTotal($baseGrandTotal);

        // Update total due
        $totalPaid = $order->getTotalPaid() ?? 0;
        $order->setTotalDue($grandTotal - $totalPaid);
        $order->setBaseTotalDue($baseGrandTotal - ($order->getBaseTotalPaid() ?? 0));
    }
}
```

---

## Summary: Anti-Pattern Checklist

**Before Making Changes, Verify:**

- [ ] Using service contracts (repositories, management interfaces)?
- [ ] Using search criteria instead of direct collections?
- [ ] State/status validation before changes?
- [ ] Transaction wrapping for multi-entity operations?
- [ ] Proper event dispatching (not bypassed)?
- [ ] Using InvoiceService/ShipmentFactory instead of direct creation?
- [ ] Payment operations going through gateway?
- [ ] Batch processing with pagination for large datasets?
- [ ] Extension attributes for custom data (not direct table changes)?
- [ ] Error handling and logging present?

---

**Assumptions:**
- Adobe Commerce 2.4.7+ with PHP 8.2+
- All examples show Commerce-specific patterns
- Service contracts are the correct approach for all operations
- Transaction handling essential for data integrity

**Why This Approach:**
- Service contracts maintain upgrade safety
- Proper validation prevents data corruption
- Transactions ensure atomicity
- Batch processing prevents memory issues
- Event dispatching enables extensibility

**Security Impact:**
- Proper validation prevents unauthorized order modifications
- Transaction boundaries prevent race conditions
- Service contracts enforce ACL
- Logging provides audit trail

**Performance Impact:**
- Batch processing scales to large datasets
- Proper indexing with search criteria
- Transaction overhead minimal vs data integrity
- Memory management prevents crashes

**Backward Compatibility:**
- Service contracts stable across versions
- Anti-patterns may "work" but break on upgrades
- Direct model manipulation bypasses future improvements
- Proper patterns future-proof code

**Tests to Add:**
- Unit tests: Validation logic, state machine
- Integration tests: Transaction rollback, multi-entity operations
- Performance tests: Batch processing benchmarks
- Negative tests: Verify errors thrown for invalid operations

**Docs to Update:**
- README.md: Link to anti-patterns
- Code review checklist: Include these patterns
- Developer onboarding: Required reading
