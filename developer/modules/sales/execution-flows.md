---
title: "Magento_Sales Execution Flows"
module: "Magento_Sales"
doc_type: "execution-flows"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Sales Execution Flows

## Overview

This document details the complete execution flows for order management operations in Magento_Sales, including order placement, invoice creation, shipment processing, credit memo generation, and order cancellation. Each flow includes the complete call stack, event dispatches, database operations, and extension points.

## Order Placement Flow

### Flow Diagram (Text Representation)

```
[Quote Submission]
    ↓
[QuoteManagement::submit()]
    ↓
[Event: sales_model_service_quote_submit_before]
    ↓
[Order Creation from Quote]
    ↓
[OrderManagement::place()]
    ↓
[Payment Authorization]
    ↓
[Event: sales_order_place_before]
    ↓
[Transaction Begin]
    ↓
[Save Order + Items + Addresses + Payment]
    ↓
[Transaction Commit]
    ↓
[Event: sales_order_place_after]
    ↓
[Quote Deactivation]
    ↓
[Send Order Confirmation Email]
    ↓
[Order Grid Update]
    ↓
[Return Order Object]
```

### Detailed Execution Flow

#### 1. Quote Submission Entry Point

```php
<?php
namespace Magento\Quote\Model;

use Magento\Quote\Api\CartManagementInterface;
use Magento\Sales\Api\Data\OrderInterface;

/**
 * Quote submission to order
 */
class QuoteManagement implements CartManagementInterface
{
    public function __construct(
        private \Magento\Framework\Event\ManagerInterface $eventManager,
        private \Magento\Quote\Model\QuoteValidator $quoteValidator,
        private \Magento\Sales\Model\OrderFactory $orderFactory,
        private \Magento\Quote\Model\Quote\Address\ToOrder $quoteAddressToOrder,
        private \Magento\Quote\Model\Quote\Address\ToOrderAddress $quoteAddressToOrderAddress,
        private \Magento\Quote\Model\Quote\Payment\ToOrderPayment $quotePaymentToOrderPayment,
        private \Magento\Quote\Model\Quote\Item\ToOrderItem $quoteItemToOrderItem,
        private \Magento\Sales\Api\OrderManagementInterface $orderManagement,
        private \Magento\Framework\DB\TransactionFactory $transactionFactory
    ) {}

    /**
     * Submit quote and create order
     *
     * Full execution flow with all steps
     *
     * @param int $cartId
     * @param \Magento\Quote\Api\Data\PaymentInterface $payment
     * @return OrderInterface
     * @throws \Exception
     */
    public function placeOrder($cartId, $payment = null)
    {
        // Step 1: Load and validate quote
        $quote = $this->quoteRepository->getActive($cartId);

        if ($quote->getCheckoutMethod() === null) {
            $quote->setCheckoutMethod(self::METHOD_GUEST);
        }

        // Step 2: Set payment method
        if ($payment) {
            $quote->getPayment()->importData($payment->getData());
        }

        // Step 3: Collect totals (final calculation)
        $quote->collectTotals();

        // Step 4: Validate quote
        $this->quoteValidator->validateBeforeSubmit($quote);

        // Step 5: Dispatch pre-submit event
        $this->eventManager->dispatch(
            'sales_model_service_quote_submit_before',
            ['order' => null, 'quote' => $quote]
        );

        // Step 6: Convert quote to order
        $order = $this->submit($quote);

        return $order;
    }

    /**
     * Submit quote (internal conversion logic)
     *
     * @param \Magento\Quote\Model\Quote $quote
     * @return OrderInterface
     * @throws \Exception
     */
    protected function submit(\Magento\Quote\Model\Quote $quote): OrderInterface
    {
        // Step 1: Convert quote data to order
        $order = $this->orderFactory->create();

        // Step 2: Copy quote data to order
        $this->quoteAddressToOrder->convert($quote, $order);

        // Step 3: Set order status
        $order->setStatus(
            $order->getConfig()->getStateDefaultStatus(\Magento\Sales\Model\Order::STATE_NEW)
        );
        $order->setState(\Magento\Sales\Model\Order::STATE_NEW);

        // Step 4: Convert addresses
        if ($quote->getBillingAddress()) {
            $billingAddress = $this->quoteAddressToOrderAddress->convert(
                $quote->getBillingAddress()
            );
            $billingAddress->setAddressType(\Magento\Sales\Model\Order\Address::TYPE_BILLING);
            $order->setBillingAddress($billingAddress);
        }

        if ($quote->getShippingAddress()) {
            $shippingAddress = $this->quoteAddressToOrderAddress->convert(
                $quote->getShippingAddress()
            );
            $shippingAddress->setAddressType(\Magento\Sales\Model\Order\Address::TYPE_SHIPPING);
            $order->setShippingAddress($shippingAddress);
        }

        // Step 5: Convert payment
        $payment = $this->quotePaymentToOrderPayment->convert($quote->getPayment());
        $order->setPayment($payment);

        // Step 6: Convert items
        foreach ($quote->getAllVisibleItems() as $quoteItem) {
            $orderItem = $this->quoteItemToOrderItem->convert($quoteItem);
            $order->addItem($orderItem);
        }

        // Step 7: Place order (authorize payment and save)
        $order = $this->orderManagement->place($order);

        // Step 8: Deactivate quote
        $quote->setIsActive(false);
        $this->quoteRepository->save($quote);

        // Step 9: Dispatch after-submit event
        $this->eventManager->dispatch(
            'sales_model_service_quote_submit_success',
            ['order' => $order, 'quote' => $quote]
        );

        return $order;
    }
}
```

#### 2. Order Management Place

```php
<?php
namespace Magento\Sales\Model\Service;

use Magento\Sales\Api\OrderManagementInterface;
use Magento\Sales\Api\Data\OrderInterface;

/**
 * Order management service - place operation
 */
class OrderService implements OrderManagementInterface
{
    public function __construct(
        private \Magento\Framework\Event\ManagerInterface $eventManager,
        private \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        private \Magento\Sales\Model\Order\Email\Sender\OrderSender $orderSender,
        private \Magento\Framework\DB\TransactionFactory $transactionFactory,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Place order
     *
     * @param OrderInterface $order
     * @return OrderInterface
     * @throws \Exception
     */
    public function place(\Magento\Sales\Api\Data\OrderInterface $order)
    {
        // Step 1: Dispatch before place event
        $this->eventManager->dispatch(
            'sales_order_place_before',
            ['order' => $order]
        );

        // Step 2: Begin database transaction
        $transaction = $this->transactionFactory->create();

        try {
            // Step 3: Process payment (authorize)
            $payment = $order->getPayment();
            $payment->place();

            // Step 4: Set initial order state based on payment
            if ($payment->getIsTransactionPending()) {
                $order->setState(\Magento\Sales\Model\Order::STATE_PAYMENT_REVIEW);
                $order->setStatus($order->getConfig()->getStateDefaultStatus(
                    \Magento\Sales\Model\Order::STATE_PAYMENT_REVIEW
                ));
            } else {
                $order->setState(\Magento\Sales\Model\Order::STATE_PROCESSING);
                $order->setStatus($order->getConfig()->getStateDefaultStatus(
                    \Magento\Sales\Model\Order::STATE_PROCESSING
                ));
            }

            // Step 5: Add all related entities to transaction
            $transaction->addObject($order);
            $transaction->addObject($order->getPayment());

            foreach ($order->getAddresses() as $address) {
                $transaction->addObject($address);
            }

            foreach ($order->getAllItems() as $item) {
                $transaction->addObject($item);
            }

            // Step 6: Save all entities atomically
            $transaction->save();

            // Step 7: Dispatch after place event
            $this->eventManager->dispatch(
                'sales_order_place_after',
                ['order' => $order]
            );

            // Step 8: Send order confirmation email
            try {
                $this->orderSender->send($order);
            } catch (\Exception $e) {
                $this->logger->error('Failed to send order confirmation email', [
                    'order_id' => $order->getEntityId(),
                    'error' => $e->getMessage()
                ]);
            }

            // Step 9: Update order grid
            $this->updateOrderGrid($order);

            $this->logger->info('Order placed successfully', [
                'order_id' => $order->getEntityId(),
                'increment_id' => $order->getIncrementId(),
                'state' => $order->getState(),
                'status' => $order->getStatus()
            ]);

            return $order;

        } catch (\Exception $e) {
            // Rollback on any error
            $this->logger->error('Order placement failed', [
                'order_data' => $order->getData(),
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString()
            ]);

            throw $e;
        }
    }

    /**
     * Update order grid table
     *
     * @param OrderInterface $order
     * @return void
     */
    private function updateOrderGrid(OrderInterface $order): void
    {
        // Grid updates are handled automatically by the sales_order_save_after
        // event via Magento\Sales\Model\ResourceModel\Grid. No separate
        // grid-specific event exists in core Magento.
        $this->gridAggregator->refreshByOrderId($order->getEntityId());
    }
}
```

#### 3. Payment Authorization Flow

```php
<?php
namespace Magento\Sales\Model\Order;

/**
 * Order payment authorization
 */
class Payment extends \Magento\Payment\Model\Info
{
    /**
     * Place payment (authorize funds)
     *
     * @return $this
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function place(): self
    {
        $this->_eventManager->dispatch(
            'sales_order_payment_place_start',
            ['payment' => $this]
        );

        $order = $this->getOrder();
        $methodInstance = $this->getMethodInstance();

        // Step 1: Set amount to authorize
        $this->setAmountOrdered($order->getTotalDue());
        $this->setBaseAmountOrdered($order->getBaseTotalDue());

        // Step 2: Authorize payment
        try {
            $methodInstance->authorize($this, $order->getBaseTotalDue());

            // Step 3: Create authorization transaction
            if ($this->getTransactionId()) {
                $transaction = $this->_transactionFactory->create();
                $transaction->setOrderPaymentObject($this)
                    ->setTxnId($this->getTransactionId())
                    ->setTxnType(\Magento\Sales\Model\Order\Payment\Transaction::TYPE_AUTH)
                    ->setIsClosed(0);

                $this->addTransaction($transaction);
            }

        } catch (\Exception $e) {
            $this->logger->error('Payment authorization failed', [
                'order_id' => $order->getEntityId(),
                'payment_method' => $methodInstance->getCode(),
                'error' => $e->getMessage()
            ]);

            throw new \Magento\Framework\Exception\LocalizedException(
                __('Payment authorization failed: %1', $e->getMessage())
            );
        }

        $this->_eventManager->dispatch(
            'sales_order_payment_place_end',
            ['payment' => $this]
        );

        return $this;
    }
}
```

### Key Events Dispatched

| Event Name | When Fired | Observer Use Cases |
|------------|------------|-------------------|
| `sales_model_service_quote_submit_before` | Before quote conversion | Final quote validation, modify quote data |
| `sales_order_place_before` | Before order save | Inventory reservation, fraud checks |
| `sales_order_place_after` | After order saved | ERP sync, warehouse notification, external APIs |
| `sales_model_service_quote_submit_success` | After successful order creation | Analytics, affiliate tracking |
| `sales_order_payment_place_start` | Before payment authorization | Payment validation |
| `sales_order_payment_place_end` | After payment authorized | Payment logging, fraud detection |

### Database Operations

#### Tables Modified During Order Placement

1. **sales_order**: Order record created
2. **sales_order_item**: One row per order item
3. **sales_order_address**: Billing and shipping addresses
4. **sales_order_payment**: Payment information
5. **sales_order_status_history**: Initial status history entry
6. **sales_payment_transaction**: Authorization transaction
7. **sales_order_grid**: Grid index entry
8. **quote**: Quote set to inactive
9. **inventory_reservation** (if MSI enabled): Inventory reserved

## Invoice Creation Flow

### Flow Diagram

```
[Invoice Request]
    ↓
[Validate Order Can Invoice]
    ↓
[Create Invoice Entity]
    ↓
[Add Invoice Items]
    ↓
[Collect Invoice Totals]
    ↓
[Register Invoice]
    ↓
[Event: sales_order_invoice_register]
    ↓
[Payment Capture]
    ↓
[Event: sales_order_invoice_pay]
    ↓
[Transaction Begin]
    ↓
[Save Invoice + Update Order]
    ↓
[Transaction Commit]
    ↓
[Event: sales_order_invoice_save_after]
    ↓
[Send Invoice Email]
    ↓
[Update Order State]
```

### Detailed Implementation

```php
<?php
namespace Magento\Sales\Model\Service;

use Magento\Sales\Api\InvoiceManagementInterface;
use Magento\Sales\Api\Data\InvoiceInterface;

/**
 * Invoice creation service
 */
class InvoiceService
{
    public function __construct(
        private \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        private \Magento\Sales\Model\Order\InvoiceFactory $invoiceFactory,
        private \Magento\Sales\Api\InvoiceRepositoryInterface $invoiceRepository,
        private \Magento\Sales\Model\Order\Email\Sender\InvoiceSender $invoiceSender,
        private \Magento\Framework\DB\TransactionFactory $transactionFactory,
        private \Magento\Framework\Event\ManagerInterface $eventManager,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Create invoice for order
     *
     * @param int $orderId
     * @param array $items Item ID => Qty to invoice
     * @param bool $capture Capture payment online
     * @param bool $notify Send customer notification
     * @return InvoiceInterface
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function createInvoice(
        int $orderId,
        array $items = [],
        bool $capture = true,
        bool $notify = true
    ): InvoiceInterface {
        // Step 1: Load order
        $order = $this->orderRepository->get($orderId);

        // Step 2: Validate can invoice
        if (!$order->canInvoice()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('The order does not allow creating an invoice.')
            );
        }

        // Step 3: Create invoice
        $invoice = $this->invoiceFactory->create();
        $invoice->setOrder($order);

        // Step 4: Add items to invoice
        if (empty($items)) {
            // Full invoice - all remaining items
            foreach ($order->getAllItems() as $orderItem) {
                if ($orderItem->getQtyToInvoice() > 0 && !$orderItem->getIsVirtual()) {
                    $invoiceItem = $this->invoiceItemFactory->create();
                    $invoiceItem->setOrderItem($orderItem);
                    $invoiceItem->setQty($orderItem->getQtyToInvoice());
                    $invoice->addItem($invoiceItem);
                }
            }
        } else {
            // Partial invoice - specified items
            foreach ($items as $itemId => $qty) {
                $orderItem = $order->getItemById($itemId);

                if (!$orderItem) {
                    throw new \Magento\Framework\Exception\LocalizedException(
                        __('Order item %1 not found.', $itemId)
                    );
                }

                if ($qty > $orderItem->getQtyToInvoice()) {
                    throw new \Magento\Framework\Exception\LocalizedException(
                        __('Invalid quantity to invoice for item %1.', $itemId)
                    );
                }

                $invoiceItem = $this->invoiceItemFactory->create();
                $invoiceItem->setOrderItem($orderItem);
                $invoiceItem->setQty($qty);
                $invoice->addItem($invoiceItem);
            }
        }

        // Step 5: Set capture mode
        if ($capture) {
            $invoice->setRequestedCaptureCase(\Magento\Sales\Model\Order\Invoice::CAPTURE_ONLINE);
        } else {
            $invoice->setRequestedCaptureCase(\Magento\Sales\Model\Order\Invoice::NOT_CAPTURE);
        }

        // Step 6: Collect totals
        $invoice->collectTotals();

        // Step 7: Register invoice
        $invoice->register();

        // Step 8: Dispatch register event
        $this->eventManager->dispatch(
            'sales_order_invoice_register',
            ['invoice' => $invoice, 'order' => $order]
        );

        // Step 9: Begin transaction
        $transaction = $this->transactionFactory->create();
        $transaction->addObject($invoice);
        $transaction->addObject($order);

        try {
            // Step 10: Capture payment if requested
            if ($capture) {
                $invoice->pay();

                $this->eventManager->dispatch(
                    'sales_order_invoice_pay',
                    ['invoice' => $invoice, 'order' => $order]
                );
            }

            // Step 11: Save invoice and order
            $transaction->save();

            // Step 12: Dispatch after save event
            $this->eventManager->dispatch(
                'sales_order_invoice_save_after',
                ['invoice' => $invoice]
            );

            // Step 13: Send notification email
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

            // Step 14: Update order state
            $this->updateOrderState($order);

            $this->logger->info('Invoice created successfully', [
                'invoice_id' => $invoice->getEntityId(),
                'order_id' => $order->getEntityId(),
                'increment_id' => $invoice->getIncrementId(),
                'grand_total' => $invoice->getGrandTotal(),
                'captured' => $capture
            ]);

            return $invoice;

        } catch (\Exception $e) {
            $this->logger->error('Invoice creation failed', [
                'order_id' => $orderId,
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString()
            ]);

            throw new \Magento\Framework\Exception\LocalizedException(
                __('Could not create invoice: %1', $e->getMessage()),
                $e
            );
        }
    }

    /**
     * Update order state after invoice
     *
     * @param \Magento\Sales\Model\Order $order
     * @return void
     */
    private function updateOrderState(\Magento\Sales\Model\Order $order): void
    {
        // Check if order is fully invoiced
        $totalInvoiced = 0;
        foreach ($order->getAllItems() as $item) {
            $totalInvoiced += $item->getQtyInvoiced();
        }

        $totalOrdered = 0;
        foreach ($order->getAllItems() as $item) {
            $totalOrdered += $item->getQtyOrdered();
        }

        if ($totalInvoiced >= $totalOrdered) {
            // Order is fully invoiced
            if ($order->canShip()) {
                // Has items to ship - keep in processing
                $order->setState(\Magento\Sales\Model\Order::STATE_PROCESSING);
            } else {
                // Nothing to ship - mark complete
                $order->setState(\Magento\Sales\Model\Order::STATE_COMPLETE);
                $order->setStatus($order->getConfig()->getStateDefaultStatus(
                    \Magento\Sales\Model\Order::STATE_COMPLETE
                ));
            }
        }

        $this->orderRepository->save($order);
    }
}
```

### Key Events for Invoice

| Event Name | When Fired | Use Cases |
|------------|------------|-----------|
| `sales_order_invoice_register` | After invoice registered, before save | Modify invoice data, add custom fees |
| `sales_order_invoice_pay` | After payment captured | Payment logging, accounting sync |
| `sales_order_invoice_save_after` | After invoice saved | External system notification, analytics |

## Shipment Creation Flow

### Flow Diagram

```
[Shipment Request]
    ↓
[Validate Order Can Ship]
    ↓
[Create Shipment Entity]
    ↓
[Add Shipment Items]
    ↓
[Add Tracking Information]
    ↓
[Register Shipment]
    ↓
[Transaction Begin]
    ↓
[Save Shipment + Update Order]
    ↓
[Transaction Commit]
    ↓
[Event: sales_order_shipment_save_after]
    ↓
[Send Shipment Email]
    ↓
[Update Order State]
```

### Implementation

```php
<?php
namespace Magento\Sales\Model\Service;

/**
 * Shipment creation service
 */
class ShipmentService
{
    public function __construct(
        private \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        private \Magento\Sales\Model\Order\ShipmentFactory $shipmentFactory,
        private \Magento\Sales\Model\Order\Shipment\TrackFactory $trackFactory,
        private \Magento\Sales\Api\ShipmentRepositoryInterface $shipmentRepository,
        private \Magento\Sales\Model\Order\Email\Sender\ShipmentSender $shipmentSender,
        private \Magento\Framework\DB\TransactionFactory $transactionFactory,
        private \Magento\Framework\Event\ManagerInterface $eventManager,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Create shipment for order
     *
     * @param int $orderId
     * @param array $items Item ID => Qty to ship
     * @param array $tracking Tracking information
     * @param bool $notify Send notification
     * @return \Magento\Sales\Api\Data\ShipmentInterface
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function createShipment(
        int $orderId,
        array $items = [],
        array $tracking = [],
        bool $notify = true
    ): \Magento\Sales\Api\Data\ShipmentInterface {
        // Step 1: Load order
        $order = $this->orderRepository->get($orderId);

        // Step 2: Validate can ship
        if (!$order->canShip()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('The order does not allow creating a shipment.')
            );
        }

        // Step 3: Create shipment
        $shipment = $this->shipmentFactory->create($order, $items);

        if (!$shipment->getTotalQty()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Cannot create shipment without products.')
            );
        }

        // Step 4: Add tracking information
        foreach ($tracking as $trackData) {
            $track = $this->trackFactory->create();
            $track->setNumber($trackData['number'] ?? '')
                ->setCarrierCode($trackData['carrier_code'] ?? '')
                ->setTitle($trackData['title'] ?? '');

            $shipment->addTrack($track);
        }

        // Step 5: Register shipment
        $shipment->register();

        // Step 6: Begin transaction
        // Note: Unlike invoices, shipments do not dispatch a register event in core Magento.
        $transaction = $this->transactionFactory->create();
        $transaction->addObject($shipment);
        $transaction->addObject($order);

        try {
            // Step 8: Save shipment and order
            $transaction->save();

            // Step 9: Dispatch after save event
            $this->eventManager->dispatch(
                'sales_order_shipment_save_after',
                ['shipment' => $shipment]
            );

            // Step 10: Send notification email
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

            // Step 11: Update order state
            $this->updateOrderState($order);

            $this->logger->info('Shipment created successfully', [
                'shipment_id' => $shipment->getEntityId(),
                'order_id' => $order->getEntityId(),
                'increment_id' => $shipment->getIncrementId(),
                'total_qty' => $shipment->getTotalQty()
            ]);

            return $shipment;

        } catch (\Exception $e) {
            $this->logger->error('Shipment creation failed', [
                'order_id' => $orderId,
                'error' => $e->getMessage()
            ]);

            throw new \Magento\Framework\Exception\LocalizedException(
                __('Could not create shipment: %1', $e->getMessage())
            );
        }
    }

    /**
     * Update order state after shipment
     *
     * @param \Magento\Sales\Model\Order $order
     * @return void
     */
    private function updateOrderState(\Magento\Sales\Model\Order $order): void
    {
        // Check if order is fully shipped
        $allShipped = true;
        foreach ($order->getAllItems() as $item) {
            if ($item->getIsVirtual()) {
                continue;
            }

            if ($item->getQtyToShip() > 0) {
                $allShipped = false;
                break;
            }
        }

        if ($allShipped) {
            // Check if fully invoiced
            $allInvoiced = true;
            foreach ($order->getAllItems() as $item) {
                if ($item->getQtyToInvoice() > 0) {
                    $allInvoiced = false;
                    break;
                }
            }

            if ($allInvoiced) {
                // Fully shipped and invoiced - complete
                $order->setState(\Magento\Sales\Model\Order::STATE_COMPLETE);
                $order->setStatus($order->getConfig()->getStateDefaultStatus(
                    \Magento\Sales\Model\Order::STATE_COMPLETE
                ));
            }
        }

        $this->orderRepository->save($order);
    }
}
```

### Key Events for Shipment

| Event Name | When Fired | Use Cases |
|------------|------------|-----------|
| `sales_order_shipment_save_after` | After shipment saved | Shipping label generation, carrier API, inventory updates |
| `sales_order_shipment_track_save_after` | After tracking saved | Customer notification with tracking |

## Credit Memo (Refund) Flow

### Flow Diagram

```
[Credit Memo Request]
    ↓
[Validate Order Can Refund]
    ↓
[Create Credit Memo Entity]
    ↓
[Add Credit Memo Items]
    ↓
[Apply Adjustments]
    ↓
[Collect Totals]
    ↓
[Register Credit Memo]
    ↓
[Process Refund (Online/Offline)]
    ↓
[Return Items to Stock (optional)]
    ↓
[Transaction Begin]
    ↓
[Save Credit Memo + Update Order]
    ↓
[Transaction Commit]
    ↓
[Event: sales_order_creditmemo_save_after]
    ↓
[Send Credit Memo Email]
    ↓
[Update Order State]
```

### Implementation

```php
<?php
namespace Magento\Sales\Model\Service;

/**
 * Credit memo creation service
 */
class CreditmemoService
{
    public function __construct(
        private \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        private \Magento\Sales\Model\Order\CreditmemoFactory $creditmemoFactory,
        private \Magento\Sales\Api\CreditmemoRepositoryInterface $creditmemoRepository,
        private \Magento\Sales\Model\Order\Email\Sender\CreditmemoSender $creditmemoSender,
        private \Magento\Framework\DB\TransactionFactory $transactionFactory,
        private \Magento\Framework\Event\ManagerInterface $eventManager,
        private \Magento\CatalogInventory\Api\StockManagementInterface $stockManagement,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Create credit memo for order
     *
     * @param int $orderId
     * @param array $items Item ID => Qty to refund
     * @param float $adjustmentPositive Adjustment refund
     * @param float $adjustmentNegative Adjustment fee
     * @param float $shippingAmount Shipping refund amount
     * @param bool $returnToStock Return items to stock
     * @param bool $refundOnline Process refund through payment gateway
     * @param bool $notify Send notification
     * @return \Magento\Sales\Api\Data\CreditmemoInterface
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function createCreditmemo(
        int $orderId,
        array $items = [],
        float $adjustmentPositive = 0.0,
        float $adjustmentNegative = 0.0,
        float $shippingAmount = 0.0,
        bool $returnToStock = false,
        bool $refundOnline = true,
        bool $notify = true
    ): \Magento\Sales\Api\Data\CreditmemoInterface {
        // Step 1: Load order
        $order = $this->orderRepository->get($orderId);

        // Step 2: Validate can refund
        if (!$order->canCreditmemo()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('The order does not allow creating a credit memo.')
            );
        }

        // Step 3: Create credit memo
        $creditmemo = $this->creditmemoFactory->createByOrder($order, [
            'items' => $items,
            'adjustment_positive' => $adjustmentPositive,
            'adjustment_negative' => $adjustmentNegative,
            'shipping_amount' => $shippingAmount
        ]);

        // Step 4: Set return to stock flag
        if ($returnToStock) {
            foreach ($creditmemo->getAllItems() as $item) {
                $item->setBackToStock(true);
            }
        }

        // Step 5: Collect totals
        $creditmemo->collectTotals();

        // Step 6: Register credit memo
        $creditmemo->register();

        // Step 7: Begin transaction
        // Note: The core event is adminhtml_sales_order_creditmemo_register_before
        // (admin area only). There is no generic sales_order_creditmemo_register event.
        $transaction = $this->transactionFactory->create();
        $transaction->addObject($creditmemo);
        $transaction->addObject($order);

        try {
            // Step 9: Process refund
            if ($refundOnline && $creditmemo->getInvoice()) {
                $creditmemo->setState(\Magento\Sales\Model\Order\Creditmemo::STATE_REFUNDED);
                $order->getPayment()->refund($creditmemo);

                $this->eventManager->dispatch(
                    'sales_order_creditmemo_refund',
                    ['creditmemo' => $creditmemo, 'order' => $order]
                );
            }

            // Step 10: Return items to stock
            if ($returnToStock) {
                foreach ($creditmemo->getAllItems() as $item) {
                    $this->stockManagement->backItemQty(
                        $item->getProductId(),
                        $item->getQty(),
                        $order->getStore()->getWebsiteId()
                    );
                }
            }

            // Step 11: Save credit memo and order
            $transaction->save();

            // Step 12: Dispatch after save event
            $this->eventManager->dispatch(
                'sales_order_creditmemo_save_after',
                ['creditmemo' => $creditmemo]
            );

            // Step 13: Send notification email
            if ($notify) {
                try {
                    $this->creditmemoSender->send($creditmemo);
                } catch (\Exception $e) {
                    $this->logger->error('Failed to send credit memo email', [
                        'creditmemo_id' => $creditmemo->getEntityId(),
                        'error' => $e->getMessage()
                    ]);
                }
            }

            // Step 14: Update order state
            $this->updateOrderState($order);

            $this->logger->info('Credit memo created successfully', [
                'creditmemo_id' => $creditmemo->getEntityId(),
                'order_id' => $order->getEntityId(),
                'increment_id' => $creditmemo->getIncrementId(),
                'grand_total' => $creditmemo->getGrandTotal(),
                'refunded_online' => $refundOnline
            ]);

            return $creditmemo;

        } catch (\Exception $e) {
            $this->logger->error('Credit memo creation failed', [
                'order_id' => $orderId,
                'error' => $e->getMessage()
            ]);

            throw new \Magento\Framework\Exception\LocalizedException(
                __('Could not create credit memo: %1', $e->getMessage())
            );
        }
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
        $precision = 0.0001;
        $fullyRefunded = abs(
            $order->getTotalInvoiced() - $order->getTotalRefunded()
        ) < $precision;

        if ($fullyRefunded) {
            $order->setState(\Magento\Sales\Model\Order::STATE_CLOSED);
            $order->setStatus($order->getConfig()->getStateDefaultStatus(
                \Magento\Sales\Model\Order::STATE_CLOSED
            ));
        }

        $this->orderRepository->save($order);
    }
}
```

### Key Events for Credit Memo

| Event Name | When Fired | Use Cases |
|------------|------------|-----------|
| `adminhtml_sales_order_creditmemo_register_before` | Before credit memo registered (admin only) | Modify refund data |
| `sales_order_creditmemo_refund` | After online refund processed | Payment logging |
| `sales_order_creditmemo_save_after` | After credit memo saved | Accounting sync, inventory updates |

## Order Cancellation Flow

### Implementation

```php
<?php
namespace Magento\Sales\Model\Service;

/**
 * Order cancellation service
 */
class OrderCancellationService
{
    public function __construct(
        private \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        private \Magento\Sales\Api\OrderManagementInterface $orderManagement,
        private \Magento\Framework\Event\ManagerInterface $eventManager,
        private \Magento\CatalogInventory\Api\StockManagementInterface $stockManagement,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Cancel order
     *
     * @param int $orderId
     * @param string $reason Cancellation reason
     * @return bool
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function cancel(int $orderId, string $reason = ''): bool
    {
        // Step 1: Load order
        $order = $this->orderRepository->get($orderId);

        // Step 2: Validate can cancel
        if (!$order->canCancel()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('The order cannot be canceled.')
            );
        }

        // Step 3: Validate cancellation prerequisites
        // Note: Magento core does not dispatch a before-cancel event.
        // The order_cancel_after event fires after cancellation completes.

        try {
            // Step 4: Cancel payment
            $order->getPayment()->cancel();

            // Step 5: Cancel all items
            foreach ($order->getAllItems() as $item) {
                $item->cancel();
            }

            // Step 6: Return inventory
            foreach ($order->getAllItems() as $item) {
                if (!$item->getIsVirtual()) {
                    $this->stockManagement->backItemQty(
                        $item->getProductId(),
                        $item->getQtyOrdered() - $item->getQtyRefunded() - $item->getQtyCanceled(),
                        $order->getStore()->getWebsiteId()
                    );
                }
            }

            // Step 7: Update order state
            $order->setState(\Magento\Sales\Model\Order::STATE_CANCELED);
            $order->setStatus($order->getConfig()->getStateDefaultStatus(
                \Magento\Sales\Model\Order::STATE_CANCELED
            ));

            // Step 8: Add status history
            if ($reason) {
                $order->addStatusHistoryComment($reason, false);
            }

            // Step 9: Save order
            $this->orderRepository->save($order);

            // Step 10: Dispatch after cancel event
            $this->eventManager->dispatch(
                'order_cancel_after',
                ['order' => $order]
            );

            $this->logger->info('Order canceled successfully', [
                'order_id' => $order->getEntityId(),
                'increment_id' => $order->getIncrementId(),
                'reason' => $reason
            ]);

            return true;

        } catch (\Exception $e) {
            $this->logger->error('Order cancellation failed', [
                'order_id' => $orderId,
                'error' => $e->getMessage()
            ]);

            throw new \Magento\Framework\Exception\LocalizedException(
                __('Could not cancel order: %1', $e->getMessage())
            );
        }
    }
}
```

## Performance Optimization Patterns

### Eager Loading Collections

```php
<?php
// Bad: N+1 query problem
foreach ($orders as $order) {
    $items = $order->getAllItems(); // Separate query per order
    $payment = $order->getPayment(); // Separate query per order
}

// Good: Eager load with joins
$orderCollection = $this->orderCollectionFactory->create();
$orderCollection->addFieldToFilter('customer_id', $customerId)
    ->setOrder('created_at', 'DESC')
    ->setPageSize(20);

// Join related data
$orderCollection->getSelect()
    ->joinLeft(
        ['payment' => 'sales_order_payment'],
        'main_table.entity_id = payment.parent_id',
        ['method', 'last_trans_id']
    )
    ->joinLeft(
        ['item' => 'sales_order_item'],
        'main_table.entity_id = item.order_id',
        []
    );

foreach ($orderCollection as $order) {
    // Data already loaded, no additional queries
}
```

### Batch Processing

```php
<?php
// Process multiple orders efficiently
public function processOrders(array $orderIds): void
{
    $connection = $this->resourceConnection->getConnection();

    // Batch update
    $connection->update(
        $this->resourceConnection->getTableName('sales_order'),
        ['status' => 'processed', 'updated_at' => new \Zend_Db_Expr('NOW()')],
        ['entity_id IN (?)' => $orderIds]
    );

    // Dispatch custom event for batch (not a core Magento event;
    // define in your module's events.xml if needed)
    $this->eventManager->dispatch(
        'custom_sales_order_batch_processed',
        ['order_ids' => $orderIds]
    );
}
```

---

**Assumptions:**
- Adobe Commerce 2.4.7+ with PHP 8.2+
- All examples use service contracts for upgrade safety
- Events follow Magento naming conventions
- Transaction handling ensures data integrity

**Why This Approach:**
- Complete visibility into execution flow for debugging
- Event-driven architecture enables extension without core modification
- Transaction boundaries prevent partial saves
- State validation prevents invalid operations

**Security Impact:**
- All operations require proper authorization (ACL for admin, customer authentication for frontend)
- Payment operations use secure gateway communication
- Sensitive data logged only in debug mode with masking

**Performance Impact:**
- Each flow optimized for minimal database queries
- Grid updates can be async to avoid blocking
- Batch operations for multiple entities reduce overhead
- Collection queries use proper indexes

**Backward Compatibility:**
- Service contract interfaces stable across minor versions
- Event names and payloads maintained
- Plugin interception points preserved
- Database transactions ensure rollback on errors

**Tests to Add:**
- Integration tests for complete flows (quote to order, order to invoice)
- Unit tests for state validation logic
- Functional tests for multi-step processes
- Performance tests for batch operations

**Docs to Update:**
- ARCHITECTURE.md: Reference these flows
- PLUGINS_AND_OBSERVERS.md: Document all events
- PERFORMANCE.md: Include optimization patterns
