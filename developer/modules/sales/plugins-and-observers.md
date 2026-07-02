---
title: "Magento_Sales Plugins & Observers"
module: "Magento_Sales"
doc_type: "plugins-and-observers"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Sales Plugins and Observers

## Overview

The Magento_Sales module provides extensive extension points through plugins (interceptors) and event observers. This document catalogs all critical extension points, provides complete implementation examples, and explains best practices for extending sales functionality without modifying core code.

## Plugin Architecture in Sales

### Plugin Types and When to Use Each

**Before Plugin**: Modify input parameters, add validation, prevent execution
**After Plugin**: Modify return values, execute post-processing
**Around Plugin**: Complete control over execution (use sparingly, prefer before/after)

### Critical Plugin Targets

#### 1. Order Repository Plugins

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Sales\Api;

use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Api\Data\OrderExtensionFactory;
use Magento\Sales\Api\Data\OrderSearchResultInterface;
use Magento\Sales\Api\OrderRepositoryInterface;
use Vendor\Module\Api\CustomOrderDataRepositoryInterface;

/**
 * Plugin to load/save custom order data
 *
 * Demonstrates extension attributes pattern for orders
 */
class OrderRepositoryExtend
{
    public function __construct(
        private OrderExtensionFactory $orderExtensionFactory,
        private CustomOrderDataRepositoryInterface $customOrderDataRepository
    ) {}

    /**
     * Load custom data after order get
     *
     * @param OrderRepositoryInterface $subject
     * @param OrderInterface $order
     * @return OrderInterface
     */
    public function afterGet(
        OrderRepositoryInterface $subject,
        OrderInterface $order
    ): OrderInterface {
        $this->loadCustomData($order);
        return $order;
    }

    /**
     * Load custom data for order list
     *
     * @param OrderRepositoryInterface $subject
     * @param OrderSearchResultInterface $searchResult
     * @return OrderSearchResultInterface
     */
    public function afterGetList(
        OrderRepositoryInterface $subject,
        OrderSearchResultInterface $searchResult
    ): OrderSearchResultInterface {
        foreach ($searchResult->getItems() as $order) {
            $this->loadCustomData($order);
        }

        return $searchResult;
    }

    /**
     * Save custom data before order save
     *
     * @param OrderRepositoryInterface $subject
     * @param OrderInterface $order
     * @return array
     */
    public function beforeSave(
        OrderRepositoryInterface $subject,
        OrderInterface $order
    ): array {
        // Extract custom data before save
        $extensionAttributes = $order->getExtensionAttributes();

        if ($extensionAttributes && $extensionAttributes->getCustomOrderData()) {
            // Store for after save
            $order->setData('_custom_order_data', $extensionAttributes->getCustomOrderData());
        }

        return [$order];
    }

    /**
     * Save custom data after order save
     *
     * @param OrderRepositoryInterface $subject
     * @param OrderInterface $order
     * @return OrderInterface
     */
    public function afterSave(
        OrderRepositoryInterface $subject,
        OrderInterface $order
    ): OrderInterface {
        // Save custom data if present
        if ($customData = $order->getData('_custom_order_data')) {
            $customData->setOrderId($order->getEntityId());
            $this->customOrderDataRepository->save($customData);

            // Reload to ensure extension attributes are current
            $this->loadCustomData($order);
        }

        return $order;
    }

    /**
     * Load custom extension attributes
     *
     * @param OrderInterface $order
     * @return void
     */
    private function loadCustomData(OrderInterface $order): void
    {
        $extensionAttributes = $order->getExtensionAttributes()
            ?? $this->orderExtensionFactory->create();

        try {
            $customData = $this->customOrderDataRepository->getByOrderId(
                (int)$order->getEntityId()
            );
            $extensionAttributes->setCustomOrderData($customData);
        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            // No custom data exists yet
            $extensionAttributes->setCustomOrderData(null);
        }

        $order->setExtensionAttributes($extensionAttributes);
    }
}
```

**Plugin Configuration (etc/di.xml):**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Sales\Api\OrderRepositoryInterface">
        <plugin name="vendor_module_order_repository_custom_data"
                type="Vendor\Module\Plugin\Sales\Api\OrderRepositoryExtend"
                sortOrder="10"/>
    </type>
</config>
```

#### 2. Order Management Plugins

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Sales\Api;

use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Api\OrderManagementInterface;
use Psr\Log\LoggerInterface;

/**
 * Plugin for order management operations
 *
 * Adds custom validation and business logic
 */
class OrderManagementExtend
{
    public function __construct(
        private LoggerInterface $logger,
        private \Vendor\Module\Service\FraudDetectionService $fraudDetection,
        private \Vendor\Module\Service\InventoryValidation $inventoryValidation,
        private \Vendor\Module\Api\ExternalOrderSystemInterface $externalSystem
    ) {}

    /**
     * Validate order before placement
     *
     * @param OrderManagementInterface $subject
     * @param OrderInterface $order
     * @return array
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function beforePlace(
        OrderManagementInterface $subject,
        OrderInterface $order
    ): array {
        // Custom fraud detection
        $fraudScore = $this->fraudDetection->calculateRiskScore($order);

        if ($fraudScore > 0.8) {
            $this->logger->warning('High fraud score detected', [
                'order_increment_id' => $order->getIncrementId(),
                'fraud_score' => $fraudScore,
                'customer_email' => $order->getCustomerEmail()
            ]);

            // Set order to payment review state
            $order->setState(\Magento\Sales\Model\Order::STATE_PAYMENT_REVIEW);
            $order->setStatus('suspected_fraud');

            $order->addCommentToStatusHistory(
                sprintf('Order flagged for review - Fraud score: %.2f', $fraudScore),
                false,
                false
            );
        }

        // Validate inventory availability (beyond default Magento checks)
        if (!$this->inventoryValidation->validateRealTimeStock($order)) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('One or more products are no longer available in the requested quantity.')
            );
        }

        return [$order];
    }

    /**
     * Execute post-order logic
     *
     * @param OrderManagementInterface $subject
     * @param OrderInterface $result
     * @return OrderInterface
     */
    public function afterPlace(
        OrderManagementInterface $subject,
        OrderInterface $result
    ): OrderInterface {
        try {
            // Send order to external system (ERP, WMS, etc.)
            $this->externalSystem->createOrder($result);

            $this->logger->info('Order synchronized to external system', [
                'order_id' => $result->getEntityId(),
                'increment_id' => $result->getIncrementId()
            ]);

        } catch (\Exception $e) {
            // Log error but don't fail order placement
            $this->logger->error('Failed to sync order to external system', [
                'order_id' => $result->getEntityId(),
                'error' => $e->getMessage()
            ]);

            // Add comment to order
            $result->addCommentToStatusHistory(
                'Warning: Failed to sync to external system. Will retry via cron.',
                false,
                false
            );
        }

        return $result;
    }

    /**
     * Before order cancellation
     *
     * @param OrderManagementInterface $subject
     * @param int $id
     * @return array
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function beforeCancel(
        OrderManagementInterface $subject,
        $id
    ): array {
        // Custom cancellation validation
        $order = $this->orderRepository->get($id);

        // Check if already shipped to warehouse
        if ($this->externalSystem->isOrderShipped($order->getIncrementId())) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Cannot cancel order: Already shipped from warehouse.')
            );
        }

        return [$id];
    }

    /**
     * After order cancellation
     *
     * @param OrderManagementInterface $subject
     * @param bool $result
     * @return bool
     */
    public function afterCancel(
        OrderManagementInterface $subject,
        bool $result
    ): bool {
        // Notify external systems of cancellation
        // Implementation details...

        return $result;
    }
}
```

#### 3. Invoice Management Plugins

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Sales\Model\Service;

use Magento\Sales\Api\Data\InvoiceInterface;
use Magento\Sales\Api\InvoiceRepositoryInterface;

/**
 * Plugin for invoice operations
 *
 * Adds custom invoice processing logic
 */
class InvoiceServiceExtend
{
    public function __construct(
        private \Vendor\Module\Service\AccountingIntegration $accounting,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * After invoice save - sync to accounting system
     *
     * @param InvoiceRepositoryInterface $subject
     * @param InvoiceInterface $invoice
     * @return InvoiceInterface
     */
    public function afterSave(
        InvoiceRepositoryInterface $subject,
        InvoiceInterface $invoice
    ): InvoiceInterface {
        try {
            // Send invoice to accounting system
            $this->accounting->createInvoice($invoice);

            $this->logger->info('Invoice synced to accounting system', [
                'invoice_id' => $invoice->getEntityId(),
                'increment_id' => $invoice->getIncrementId(),
                'order_id' => $invoice->getOrderId()
            ]);

        } catch (\Exception $e) {
            $this->logger->error('Failed to sync invoice to accounting', [
                'invoice_id' => $invoice->getEntityId(),
                'error' => $e->getMessage()
            ]);
        }

        return $invoice;
    }
}
```

#### 4. Payment Capture Plugins

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Sales\Model\Order;

use Magento\Sales\Model\Order\Payment;

/**
 * Plugin for payment capture operations
 *
 * Adds fraud detection and custom payment logic
 */
class PaymentExtend
{
    public function __construct(
        private \Vendor\Module\Service\PaymentFraudDetection $fraudDetection,
        private \Vendor\Module\Service\PaymentReconciliation $reconciliation,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Before payment capture - additional fraud checks
     *
     * @param Payment $subject
     * @param \Magento\Sales\Model\Order\Invoice|null $invoice
     * @return array
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function beforeCapture(
        Payment $subject,
        $invoice = null
    ): array {
        $order = $subject->getOrder();

        // Real-time fraud detection at capture
        $fraudResult = $this->fraudDetection->validateCapture($subject, $order);

        if (!$fraudResult->isValid()) {
            $this->logger->critical('Payment capture blocked by fraud detection', [
                'order_id' => $order->getEntityId(),
                'payment_method' => $subject->getMethod(),
                'fraud_reasons' => $fraudResult->getReasons()
            ]);

            throw new \Magento\Framework\Exception\LocalizedException(
                __('Payment capture blocked for security reasons.')
            );
        }

        return [$invoice];
    }

    /**
     * After payment capture - reconciliation
     *
     * @param Payment $subject
     * @param Payment $result
     * @return Payment
     */
    public function afterCapture(
        Payment $subject,
        Payment $result
    ): Payment {
        try {
            // Record payment for reconciliation
            $this->reconciliation->recordPayment($result);

            $this->logger->info('Payment captured and reconciled', [
                'order_id' => $result->getOrder()->getEntityId(),
                'transaction_id' => $result->getLastTransId(),
                'amount' => $result->getAmountOrdered()
            ]);

        } catch (\Exception $e) {
            $this->logger->error('Payment reconciliation failed', [
                'order_id' => $result->getOrder()->getEntityId(),
                'error' => $e->getMessage()
            ]);
        }

        return $result;
    }
}
```

#### 5. Shipment Tracking Plugins

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Sales\Model\Order\Shipment;

use Magento\Sales\Model\Order\Shipment\Track;

/**
 * Plugin for shipment tracking
 *
 * Integrates with carrier APIs for real-time tracking
 */
class TrackExtend
{
    public function __construct(
        private \Vendor\Module\Service\CarrierIntegration $carrierIntegration,
        private \Magento\Sales\Api\ShipmentRepositoryInterface $shipmentRepository,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * After tracking save - notify carrier
     *
     * @param Track $subject
     * @param Track $result
     * @return Track
     */
    public function afterSave(
        Track $subject,
        Track $result
    ): Track {
        try {
            // Register tracking with carrier API
            $this->carrierIntegration->registerTracking(
                $result->getCarrierCode(),
                $result->getTrackNumber(),
                $result->getShipment()
            );

            $this->logger->info('Tracking registered with carrier', [
                'shipment_id' => $result->getParentId(),
                'carrier' => $result->getCarrierCode(),
                'tracking_number' => $result->getTrackNumber()
            ]);

        } catch (\Exception $e) {
            $this->logger->error('Failed to register tracking with carrier', [
                'tracking_number' => $result->getTrackNumber(),
                'error' => $e->getMessage()
            ]);
        }

        return $result;
    }
}
```

## Event Observer Architecture

### Core Sales Events Catalog

#### Order Events

| Event Name | When Fired | Event Data | Use Cases |
|------------|------------|------------|-----------|
| `sales_order_place_before` | Before order is saved during placement | `order` | Final validation, fraud checks |
| `sales_order_place_after` | After order is saved | `order` | External system sync, notifications |
| `sales_order_save_before` | Before any order save | `order`, `data_object` | Audit logging, data validation |
| `sales_order_save_after` | After any order save | `order`, `data_object` | Cache invalidation, index updates |
| `sales_order_load_after` | After order loaded from DB | `order`, `data_object` | Inject runtime data |
| `sales_order_delete_before` | Before order deletion | `order`, `data_object` | Archive data, cleanup |
| `sales_order_delete_after` | After order deletion | `order`, `data_object` | Cleanup related data |

#### Order State Change Events

| Event Name | When Fired | Event Data | Use Cases |
|------------|------------|------------|-----------|
| `sales_order_state_change_before` | Before state changes | `order`, `state` | Validate state transition |
| `order_cancel_after` | After order canceled | `order` | Inventory restoration, notifications |

#### Invoice Events

| Event Name | When Fired | Event Data | Use Cases |
|------------|------------|------------|-----------|
| `sales_order_invoice_register` | After invoice registered, before save | `invoice`, `order` | Modify invoice data |
| `sales_order_invoice_pay` | After payment captured | `invoice`, `order` | Payment logging |
| `sales_order_invoice_save_before` | Before invoice save | `invoice`, `data_object` | Validation |
| `sales_order_invoice_save_after` | After invoice save | `invoice`, `data_object` | Accounting sync |
| `sales_order_invoice_cancel` | After invoice canceled | `invoice` | Payment reversal |

#### Shipment Events

| Event Name | When Fired | Event Data | Use Cases |
|------------|------------|------------|-----------|
| `sales_order_shipment_save_before` | Before shipment save | `shipment`, `data_object` | Validation |
| `sales_order_shipment_save_after` | After shipment save | `shipment`, `data_object` | Carrier notification |
| `sales_order_shipment_track_save_after` | After tracking saved | `track`, `data_object` | Customer notification |

#### Credit Memo Events

| Event Name | When Fired | Event Data | Use Cases |
|------------|------------|------------|-----------|
| `adminhtml_sales_order_creditmemo_register_before` | Before credit memo registered (admin only) | `creditmemo`, `order` | Modify refund data |
| `sales_order_creditmemo_refund` | After refund processed | `creditmemo`, `order` | Payment logging |
| `sales_order_creditmemo_save_before` | Before credit memo save | `creditmemo`, `data_object` | Validation |
| `sales_order_creditmemo_save_after` | After credit memo save | `creditmemo`, `data_object` | Accounting sync |

### Observer Implementation Examples

#### 1. Order Placement Observer

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer\Sales;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Model\Order;

/**
 * Observer for order placement
 *
 * Executes custom logic after order is placed
 */
class OrderPlaceAfterObserver implements ObserverInterface
{
    public function __construct(
        private \Vendor\Module\Service\ErpIntegration $erpIntegration,
        private \Vendor\Module\Service\WarehouseNotification $warehouseNotification,
        private \Vendor\Module\Model\OrderQueueFactory $orderQueueFactory,
        private \Vendor\Module\Model\ResourceModel\OrderQueue $orderQueueResource,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Execute after order placement
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Order $order */
        $order = $observer->getEvent()->getOrder();

        try {
            // 1. Queue order for ERP sync (async processing)
            $this->queueOrderForErpSync($order);

            // 2. Notify warehouse (real-time)
            if ($this->shouldNotifyWarehouse($order)) {
                $this->warehouseNotification->notifyNewOrder($order);
            }

            // 3. Custom business logic
            $this->processCustomOrderLogic($order);

            $this->logger->info('Post-order processing completed', [
                'order_id' => $order->getEntityId(),
                'increment_id' => $order->getIncrementId()
            ]);

        } catch (\Exception $e) {
            // Log error but don't fail order placement
            $this->logger->error('Post-order processing failed', [
                'order_id' => $order->getEntityId(),
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString()
            ]);
        }
    }

    /**
     * Queue order for ERP synchronization
     *
     * @param Order $order
     * @return void
     */
    private function queueOrderForErpSync(Order $order): void
    {
        $queueEntry = $this->orderQueueFactory->create();
        $queueEntry->setData([
            'order_id' => $order->getEntityId(),
            'increment_id' => $order->getIncrementId(),
            'status' => 'pending',
            'retry_count' => 0,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        $this->orderQueueResource->save($queueEntry);
    }

    /**
     * Determine if warehouse notification is needed
     *
     * @param Order $order
     * @return bool
     */
    private function shouldNotifyWarehouse(Order $order): bool
    {
        // Don't notify for virtual/downloadable orders
        if ($order->getIsVirtual()) {
            return false;
        }

        // Only notify during business hours
        $currentHour = (int)date('H');
        if ($currentHour < 8 || $currentHour > 18) {
            return false;
        }

        return true;
    }

    /**
     * Custom order processing logic
     *
     * @param Order $order
     * @return void
     */
    private function processCustomOrderLogic(Order $order): void
    {
        // Check for special handling requirements
        foreach ($order->getAllItems() as $item) {
            if ($item->getProduct()->getData('requires_special_handling')) {
                $order->addCommentToStatusHistory(
                    'Order contains items requiring special handling',
                    false,
                    false
                );
                break;
            }
        }

        // Apply custom order tags
        if ($order->getGrandTotal() > 1000) {
            $order->setData('is_high_value', true);
        }
    }
}
```

**Observer Configuration (etc/events.xml):**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="sales_order_place_after">
        <observer name="vendor_module_order_place_after"
                  instance="Vendor\Module\Observer\Sales\OrderPlaceAfterObserver"/>
    </event>
</config>
```

#### 2. Invoice Save Observer

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer\Sales;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Model\Order\Invoice;

/**
 * Observer for invoice save operations
 *
 * Syncs invoices to external accounting system
 */
class InvoiceSaveAfterObserver implements ObserverInterface
{
    public function __construct(
        private \Vendor\Module\Service\AccountingSync $accountingSync,
        private \Vendor\Module\Model\InvoiceSyncQueueFactory $queueFactory,
        private \Vendor\Module\Model\ResourceModel\InvoiceSyncQueue $queueResource,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Execute after invoice save
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Invoice $invoice */
        $invoice = $observer->getEvent()->getInvoice();

        // Only sync paid invoices
        if ($invoice->getState() !== Invoice::STATE_PAID) {
            return;
        }

        try {
            // Immediate sync attempt
            try {
                $this->accountingSync->syncInvoice($invoice);

                $this->logger->info('Invoice synced to accounting system', [
                    'invoice_id' => $invoice->getEntityId(),
                    'increment_id' => $invoice->getIncrementId(),
                    'order_id' => $invoice->getOrderId()
                ]);

            } catch (\Exception $e) {
                // Queue for retry if immediate sync fails
                $this->queueInvoiceForRetry($invoice);

                $this->logger->warning('Invoice queued for accounting sync retry', [
                    'invoice_id' => $invoice->getEntityId(),
                    'error' => $e->getMessage()
                ]);
            }

        } catch (\Exception $e) {
            $this->logger->error('Invoice sync processing failed', [
                'invoice_id' => $invoice->getEntityId(),
                'error' => $e->getMessage()
            ]);
        }
    }

    /**
     * Queue invoice for retry
     *
     * @param Invoice $invoice
     * @return void
     */
    private function queueInvoiceForRetry(Invoice $invoice): void
    {
        $queueEntry = $this->queueFactory->create();
        $queueEntry->setData([
            'invoice_id' => $invoice->getEntityId(),
            'order_id' => $invoice->getOrderId(),
            'status' => 'pending',
            'retry_count' => 0,
            'next_retry_at' => date('Y-m-d H:i:s', strtotime('+5 minutes')),
            'created_at' => date('Y-m-d H:i:s')
        ]);

        $this->queueResource->save($queueEntry);
    }
}
```

#### 3. Shipment Track Observer

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer\Sales;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Model\Order\Shipment\Track;

/**
 * Observer for shipment tracking
 *
 * Sends customer notifications with tracking information
 */
class ShipmentTrackSaveAfterObserver implements ObserverInterface
{
    public function __construct(
        private \Magento\Sales\Api\ShipmentRepositoryInterface $shipmentRepository,
        private \Vendor\Module\Service\CustomerNotification $customerNotification,
        private \Vendor\Module\Service\CarrierTracking $carrierTracking,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Execute after tracking save
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Track $track */
        $track = $observer->getEvent()->getTrack();

        try {
            $shipment = $this->shipmentRepository->get($track->getParentId());
            $order = $shipment->getOrder();

            // Get real-time tracking info from carrier
            $trackingInfo = $this->carrierTracking->getTrackingInfo(
                $track->getCarrierCode(),
                $track->getTrackNumber()
            );

            // Send enhanced notification to customer
            $this->customerNotification->sendShipmentNotification(
                $order,
                $shipment,
                $track,
                $trackingInfo
            );

            $this->logger->info('Shipment notification sent', [
                'order_id' => $order->getEntityId(),
                'shipment_id' => $shipment->getEntityId(),
                'tracking_number' => $track->getTrackNumber(),
                'carrier' => $track->getCarrierCode()
            ]);

        } catch (\Exception $e) {
            $this->logger->error('Shipment notification failed', [
                'track_id' => $track->getEntityId(),
                'error' => $e->getMessage()
            ]);
        }
    }
}
```

#### 4. Credit Memo Observer

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer\Sales;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Model\Order\Creditmemo;

/**
 * Observer for credit memo operations
 *
 * Handles refund processing and accounting
 */
class CreditmemoSaveAfterObserver implements ObserverInterface
{
    public function __construct(
        private \Vendor\Module\Service\RefundProcessing $refundProcessing,
        private \Vendor\Module\Service\CustomerService $customerService,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Execute after credit memo save
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Creditmemo $creditmemo */
        $creditmemo = $observer->getEvent()->getCreditmemo();

        try {
            // Process refund in external system
            $this->refundProcessing->processRefund($creditmemo);

            // Update customer account (store credit, etc.)
            if ($creditmemo->getBaseCustomerBalanceReturnMax() > 0) {
                $this->customerService->addStoreCredit(
                    $creditmemo->getOrder()->getCustomerId(),
                    $creditmemo->getBaseCustomerBalanceReturnMax()
                );
            }

            // Log refund for auditing
            $this->logger->info('Credit memo processed', [
                'creditmemo_id' => $creditmemo->getEntityId(),
                'order_id' => $creditmemo->getOrderId(),
                'grand_total' => $creditmemo->getGrandTotal(),
                'adjustment_positive' => $creditmemo->getAdjustmentPositive(),
                'adjustment_negative' => $creditmemo->getAdjustmentNegative()
            ]);

        } catch (\Exception $e) {
            $this->logger->error('Credit memo post-processing failed', [
                'creditmemo_id' => $creditmemo->getEntityId(),
                'error' => $e->getMessage()
            ]);
        }
    }
}
```

#### 5. Order Cancellation Observer

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer\Sales;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Model\Order;

/**
 * Observer for order cancellation
 *
 * Handles cleanup and notifications
 */
class OrderCancelAfterObserver implements ObserverInterface
{
    public function __construct(
        private \Vendor\Module\Service\ErpIntegration $erpIntegration,
        private \Vendor\Module\Service\InventoryManagement $inventoryManagement,
        private \Vendor\Module\Service\PaymentProcessing $paymentProcessing,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Execute after order cancellation
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Order $order */
        $order = $observer->getEvent()->getOrder();

        try {
            // Notify ERP of cancellation
            $this->erpIntegration->cancelOrder($order->getIncrementId());

            // Release inventory reservations
            $this->inventoryManagement->releaseReservation($order);

            // Void/cancel payment authorization
            if ($order->getPayment()->getAuthorizationTransaction()) {
                $this->paymentProcessing->voidAuthorization($order->getPayment());
            }

            $this->logger->info('Order cancellation processed', [
                'order_id' => $order->getEntityId(),
                'increment_id' => $order->getIncrementId(),
                'state' => $order->getState(),
                'status' => $order->getStatus()
            ]);

        } catch (\Exception $e) {
            $this->logger->error('Order cancellation post-processing failed', [
                'order_id' => $order->getEntityId(),
                'error' => $e->getMessage()
            ]);
        }
    }
}
```

## Best Practices for Plugins and Observers

### 1. Plugin vs Observer Decision Tree

```
Does the operation need to:
├─ Modify input parameters? → Use BEFORE plugin
├─ Modify return values? → Use AFTER plugin
├─ Completely replace logic? → Use AROUND plugin (rare)
└─ React to state change without modifying data? → Use OBSERVER
```

### 2. Performance Considerations

```php
<?php
// BAD: Heavy processing in observer
public function execute(Observer $observer): void
{
    $order = $observer->getEvent()->getOrder();

    // This blocks order save!
    $this->externalApi->syncOrder($order); // May take 5-10 seconds
}

// GOOD: Queue for async processing
public function execute(Observer $observer): void
{
    $order = $observer->getEvent()->getOrder();

    // Fast queue operation
    $this->messageQueue->publish('sales.order.sync', [
        'order_id' => $order->getEntityId()
    ]);
}
```

### 3. Error Handling

```php
<?php
// CRITICAL: Never let observer exceptions fail core operations
public function execute(Observer $observer): void
{
    try {
        // Your custom logic
        $this->doSomething($observer->getEvent()->getOrder());
    } catch (\Exception $e) {
        // Log error but DON'T re-throw
        $this->logger->error('Observer failed', [
            'observer' => self::class,
            'error' => $e->getMessage()
        ]);

        // Optionally: Add order comment for visibility
        $order = $observer->getEvent()->getOrder();
        $order->addCommentToStatusHistory(
            'Warning: ' . $e->getMessage(),
            false,
            false
        );
    }
}
```

### 4. Plugin Sort Order

```xml
<!-- etc/di.xml -->
<config>
    <!-- Validation plugins: sortOrder < 100 -->
    <type name="Magento\Sales\Api\OrderManagementInterface">
        <plugin name="vendor_module_validation"
                type="Vendor\Module\Plugin\ValidationExtend"
                sortOrder="50"/>
    </type>

    <!-- Business logic plugins: sortOrder 100-500 -->
    <type name="Magento\Sales\Api\OrderManagementInterface">
        <plugin name="vendor_module_business_logic"
                type="Vendor\Module\Plugin\BusinessLogicExtend"
                sortOrder="200"/>
    </type>

    <!-- Logging/monitoring plugins: sortOrder > 500 -->
    <type name="Magento\Sales\Api\OrderManagementInterface">
        <plugin name="vendor_module_logging"
                type="Vendor\Module\Plugin\LoggingExtend"
                sortOrder="600"/>
    </type>
</config>
```

### 5. Avoiding Plugin Conflicts

```php
<?php
// BAD: Modifying same property in multiple plugins causes conflicts
public function afterGet(OrderRepositoryInterface $subject, OrderInterface $order): OrderInterface
{
    $order->setCustomAttribute('processed', true);
    return $order;
}

// GOOD: Use extension attributes (designed for this purpose)
public function afterGet(OrderRepositoryInterface $subject, OrderInterface $order): OrderInterface
{
    $extensionAttributes = $order->getExtensionAttributes();
    $extensionAttributes->setProcessed(true);
    $order->setExtensionAttributes($extensionAttributes);
    return $order;
}
```

## Testing Plugins and Observers

### Unit Test for Plugin

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Unit\Plugin\Sales\Api;

use PHPUnit\Framework\TestCase;
use Vendor\Module\Plugin\Sales\Api\OrderRepositoryExtend;
use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Api\OrderRepositoryInterface;

class OrderRepositoryExtendTest extends TestCase
{
    private OrderRepositoryExtend $plugin;
    private $customOrderDataRepository;
    private $orderExtensionFactory;

    protected function setUp(): void
    {
        $this->customOrderDataRepository = $this->createMock(
            \Vendor\Module\Api\CustomOrderDataRepositoryInterface::class
        );
        $this->orderExtensionFactory = $this->createMock(
            \Magento\Sales\Api\Data\OrderExtensionFactory::class
        );

        $this->plugin = new OrderRepositoryExtend(
            $this->orderExtensionFactory,
            $this->customOrderDataRepository
        );
    }

    public function testAfterGetLoadsCustomData(): void
    {
        $orderId = 123;
        $order = $this->createMock(OrderInterface::class);
        $order->method('getEntityId')->willReturn($orderId);

        $subject = $this->createMock(OrderRepositoryInterface::class);

        $customData = new \stdClass();
        $customData->value = 'test';

        $this->customOrderDataRepository
            ->expects($this->once())
            ->method('getByOrderId')
            ->with($orderId)
            ->willReturn($customData);

        $result = $this->plugin->afterGet($subject, $order);

        $this->assertSame($order, $result);
    }
}
```

### Integration Test for Observer

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Integration\Observer\Sales;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;
use Magento\Sales\Model\Order;

class OrderPlaceAfterObserverTest extends TestCase
{
    private $eventManager;
    private $orderQueueResource;

    protected function setUp(): void
    {
        $objectManager = Bootstrap::getObjectManager();
        $this->eventManager = $objectManager->get(\Magento\Framework\Event\ManagerInterface::class);
        $this->orderQueueResource = $objectManager->get(
            \Vendor\Module\Model\ResourceModel\OrderQueue::class
        );
    }

    /**
     * @magentoDataFixture Magento/Sales/_files/order.php
     */
    public function testOrderQueuedForErpSync(): void
    {
        $objectManager = Bootstrap::getObjectManager();
        $order = $objectManager->create(Order::class);
        $order->loadByIncrementId('100000001');

        // Dispatch event
        $this->eventManager->dispatch('sales_order_place_after', ['order' => $order]);

        // Verify order was queued
        $connection = $this->orderQueueResource->getConnection();
        $select = $connection->select()
            ->from($this->orderQueueResource->getMainTable())
            ->where('order_id = ?', $order->getId());

        $queueEntry = $connection->fetchRow($select);

        $this->assertNotEmpty($queueEntry);
        $this->assertEquals('pending', $queueEntry['status']);
    }
}
```

---

**Assumptions:**
- Adobe Commerce 2.4.7+ with PHP 8.2+
- Plugins configured via di.xml with proper sort orders
- Observers configured via events.xml
- Async processing via message queues where appropriate

**Why This Approach:**
- Plugins for data modification maintain upgrade safety
- Observers for reactive logic don't block critical operations
- Extension attributes for custom data avoid core table modifications
- Error handling prevents custom code from failing core operations

**Security Impact:**
- Plugin validation can enforce additional authorization checks
- Observers should not expose sensitive data in logs
- External API calls should use secure authentication
- PII in custom data must follow same protection as core data

**Performance Impact:**
- Observers execute synchronously; use queues for heavy operations
- Plugin chains add overhead; keep count reasonable (< 10 per method)
- Extension attribute loading is lazy when implemented properly
- Database queries in observers/plugins should use connection pool

**Backward Compatibility:**
- Plugin method signatures must match intercepted methods
- Event data structure is stable across minor versions
- Extension attributes via extension_attributes.xml are BC-safe
- Observer event names maintained across versions

**Tests to Add:**
- Unit tests: Plugin logic, observer logic
- Integration tests: Event dispatching, data persistence
- Functional tests: End-to-end flows with plugins/observers
- Performance tests: Plugin chain overhead measurement

**Docs to Update:**
- README.md: Link to this document
- ARCHITECTURE.md: Reference extension points
- Custom module documentation: Document your plugins/observers
