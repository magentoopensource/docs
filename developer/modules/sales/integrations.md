---
title: "Magento_Sales Integrations"
module: "Magento_Sales"
doc_type: "integrations"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Sales Integrations

## Overview

The Magento_Sales module integrates deeply with numerous other Magento modules to provide complete order management functionality. This document details all major integration points, data flows, dependency relationships, and best practices for extending these integrations.

## Core Module Integrations

### Integration with Magento_Quote

The Quote module provides the cart/checkout foundation that converts to orders.

#### Quote to Order Conversion Flow

```php
<?php
namespace Magento\Quote\Model;

use Magento\Sales\Api\Data\OrderInterface;

/**
 * Quote to order conversion
 *
 * Core integration point between Quote and Sales modules
 */
class QuoteManagement
{
    public function __construct(
        private \Magento\Quote\Model\Quote\Address\ToOrder $quoteAddressToOrder,
        private \Magento\Quote\Model\Quote\Address\ToOrderAddress $quoteAddressToOrderAddress,
        private \Magento\Quote\Model\Quote\Item\ToOrderItem $quoteItemToOrderItem,
        private \Magento\Quote\Model\Quote\Payment\ToOrderPayment $quotePaymentToOrderPayment,
        private \Magento\Sales\Model\OrderFactory $orderFactory,
        private \Magento\Framework\DataObject\Copy $objectCopyService
    ) {}

    /**
     * Convert quote to order
     *
     * @param \Magento\Quote\Model\Quote $quote
     * @return OrderInterface
     */
    protected function submitQuote(\Magento\Quote\Model\Quote $quote): OrderInterface
    {
        $order = $this->orderFactory->create();

        // Copy quote data to order using data mapping
        $this->quoteAddressToOrder->convert($quote, $order);

        // Convert addresses
        if ($quote->getBillingAddress()) {
            $billingAddress = $this->quoteAddressToOrderAddress->convert(
                $quote->getBillingAddress()
            );
            $order->setBillingAddress($billingAddress);
        }

        if (!$quote->getIsVirtual() && $quote->getShippingAddress()) {
            $shippingAddress = $this->quoteAddressToOrderAddress->convert(
                $quote->getShippingAddress()
            );
            $order->setShippingAddress($shippingAddress);
        }

        // Convert payment
        $payment = $this->quotePaymentToOrderPayment->convert($quote->getPayment());
        $order->setPayment($payment);

        // Convert items
        foreach ($quote->getAllVisibleItems() as $quoteItem) {
            $orderItem = $this->quoteItemToOrderItem->convert($quoteItem);
            $order->addItem($orderItem);
        }

        return $order;
    }
}
```

#### Field Mapping Configuration

```xml
<!-- etc/fieldset.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:DataObject/etc/fieldset.xsd">

    <!-- Quote to Order field mapping -->
    <scope id="sales_convert_quote">
        <fieldset id="sales_convert_quote">
            <field name="store_id">
                <aspect name="to_order"/>
            </field>
            <field name="base_currency_code">
                <aspect name="to_order"/>
            </field>
            <field name="store_currency_code">
                <aspect name="to_order"/>
            </field>
            <field name="order_currency_code">
                <aspect name="to_order"/>
            </field>
            <field name="grand_total">
                <aspect name="to_order"/>
            </field>
            <field name="base_grand_total">
                <aspect name="to_order"/>
            </field>
            <field name="customer_id">
                <aspect name="to_order"/>
            </field>
            <field name="customer_email">
                <aspect name="to_order"/>
            </field>
            <field name="customer_firstname">
                <aspect name="to_order"/>
            </field>
            <field name="customer_lastname">
                <aspect name="to_order"/>
            </field>
            <field name="customer_taxvat">
                <aspect name="to_order"/>
            </field>
            <field name="customer_dob">
                <aspect name="to_order"/>
            </field>
            <field name="customer_gender">
                <aspect name="to_order"/>
            </field>
            <field name="customer_group_id">
                <aspect name="to_order"/>
            </field>
            <field name="remote_ip">
                <aspect name="to_order"/>
            </field>
            <field name="x_forwarded_for">
                <aspect name="to_order"/>
            </field>
        </fieldset>
    </scope>

    <!-- Quote Item to Order Item -->
    <scope id="sales_convert_quote_item">
        <fieldset id="sales_convert_quote_item">
            <field name="product_id">
                <aspect name="to_order_item"/>
            </field>
            <field name="sku">
                <aspect name="to_order_item"/>
            </field>
            <field name="name">
                <aspect name="to_order_item"/>
            </field>
            <field name="weight">
                <aspect name="to_order_item"/>
            </field>
            <field name="qty">
                <aspect name="to_order_item">
                    <attribute name="qty_ordered"/>
                </aspect>
            </field>
            <field name="price">
                <aspect name="to_order_item"/>
            </field>
            <field name="base_price">
                <aspect name="to_order_item"/>
            </field>
            <field name="row_total">
                <aspect name="to_order_item"/>
            </field>
            <field name="base_row_total">
                <aspect name="to_order_item"/>
            </field>
            <field name="tax_amount">
                <aspect name="to_order_item"/>
            </field>
            <field name="base_tax_amount">
                <aspect name="to_order_item"/>
            </field>
        </fieldset>
    </scope>
</config>
```

#### Custom Field Mapping

```php
<?php
namespace Vendor\Module\Plugin\Quote\Model\Quote\Address;

use Magento\Quote\Model\Quote\Address\ToOrder;
use Magento\Sales\Api\Data\OrderInterface;

/**
 * Plugin to add custom field mapping
 */
class ToOrderExtend
{
    /**
     * Add custom fields to order conversion
     *
     * @param ToOrder $subject
     * @param OrderInterface $result
     * @param \Magento\Quote\Model\Quote\Address $address
     * @return OrderInterface
     */
    public function afterConvert(
        ToOrder $subject,
        OrderInterface $result,
        \Magento\Quote\Model\Quote\Address $address
    ): OrderInterface {
        // Map custom quote fields to order
        $quote = $address->getQuote();

        if ($customField = $quote->getData('custom_field')) {
            $result->setData('custom_field', $customField);
        }

        // Map extension attributes
        $quoteExtension = $quote->getExtensionAttributes();
        if ($quoteExtension && $quoteExtension->getCustomData()) {
            $orderExtension = $result->getExtensionAttributes();
            $orderExtension->setCustomData($quoteExtension->getCustomData());
            $result->setExtensionAttributes($orderExtension);
        }

        return $result;
    }
}
```

### Integration with Magento_Payment

Payment module handles all payment method operations during order processing.

#### Payment Authorization During Order Placement

```php
<?php
namespace Magento\Sales\Model\Order;

/**
 * Order payment operations
 *
 * Integrates with payment methods for authorization/capture
 */
class Payment extends \Magento\Payment\Model\Info
{
    /**
     * Place payment (authorize)
     *
     * @return $this
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function place(): self
    {
        $order = $this->getOrder();
        $methodInstance = $this->getMethodInstance();

        // Set amounts to authorize
        $this->setAmountOrdered($order->getTotalDue());
        $this->setBaseAmountOrdered($order->getBaseTotalDue());

        // Dispatch pre-authorization event
        $this->_eventManager->dispatch(
            'sales_order_payment_place_start',
            ['payment' => $this]
        );

        try {
            // Call payment method authorization
            $methodInstance->authorize($this, $order->getBaseTotalDue());

            // Create authorization transaction
            if ($this->getTransactionId()) {
                $transaction = $this->_transactionFactory->create();
                $transaction->setOrderPaymentObject($this)
                    ->setTxnId($this->getTransactionId())
                    ->setTxnType(\Magento\Sales\Model\Order\Payment\Transaction::TYPE_AUTH)
                    ->setIsClosed(0)
                    ->setAdditionalInformation(
                        \Magento\Sales\Model\Order\Payment\Transaction::RAW_DETAILS,
                        $this->getAdditionalInformation()
                    );

                $this->addTransaction($transaction);
                $this->setParentTransactionId($transaction->getTransactionId());
            }

        } catch (\Magento\Framework\Exception\LocalizedException $e) {
            $this->_logger->error('Payment authorization failed', [
                'order_id' => $order->getEntityId(),
                'method' => $methodInstance->getCode(),
                'error' => $e->getMessage()
            ]);

            throw $e;
        }

        // Dispatch post-authorization event
        $this->_eventManager->dispatch(
            'sales_order_payment_place_end',
            ['payment' => $this]
        );

        return $this;
    }

    /**
     * Capture payment
     *
     * @param \Magento\Sales\Model\Order\Invoice|null $invoice
     * @return $this
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function capture($invoice = null): self
    {
        if ($invoice === null) {
            $invoice = $this->_invoice;
        }

        $methodInstance = $this->getMethodInstance();
        $amount = $invoice ? $invoice->getBaseGrandTotal() : $this->getBaseAmountOrdered();

        // Dispatch pre-capture event
        $this->_eventManager->dispatch(
            'sales_order_payment_capture',
            ['payment' => $this, 'invoice' => $invoice]
        );

        try {
            // Call payment method capture
            $methodInstance->setStore($this->getOrder()->getStoreId());
            $methodInstance->capture($this, $amount);

            // Update payment amounts
            $this->setBaseAmountPaid($this->getBaseAmountPaid() + $amount);
            $this->setAmountPaid(
                $this->getAmountPaid() + $this->getOrder()->getStore()->convertPrice($amount)
            );

            // Create capture transaction
            if ($this->getTransactionId()) {
                $transaction = $this->_transactionFactory->create();
                $transaction->setOrderPaymentObject($this)
                    ->setTxnId($this->getTransactionId())
                    ->setTxnType(\Magento\Sales\Model\Order\Payment\Transaction::TYPE_CAPTURE)
                    ->setParentTxnId($this->getParentTransactionId())
                    ->setIsClosed(1);

                $this->addTransaction($transaction);
            }

        } catch (\Exception $e) {
            $this->_logger->error('Payment capture failed', [
                'order_id' => $this->getOrder()->getEntityId(),
                'method' => $methodInstance->getCode(),
                'amount' => $amount,
                'error' => $e->getMessage()
            ]);

            throw new \Magento\Framework\Exception\LocalizedException(
                __('Payment capture failed: %1', $e->getMessage())
            );
        }

        return $this;
    }

    /**
     * Refund payment
     *
     * @param \Magento\Sales\Model\Order\Creditmemo $creditmemo
     * @return $this
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function refund($creditmemo): self
    {
        $methodInstance = $this->getMethodInstance();
        $baseAmount = $creditmemo->getBaseGrandTotal();

        try {
            // Call payment method refund
            $methodInstance->refund($this, $baseAmount);

            // Update payment amounts
            $this->setBaseAmountRefunded($this->getBaseAmountRefunded() + $baseAmount);
            $this->setAmountRefunded(
                $this->getAmountRefunded() + $creditmemo->getGrandTotal()
            );

            // Create refund transaction
            if ($this->getTransactionId()) {
                $transaction = $this->_transactionFactory->create();
                $transaction->setOrderPaymentObject($this)
                    ->setTxnId($this->getTransactionId())
                    ->setTxnType(\Magento\Sales\Model\Order\Payment\Transaction::TYPE_REFUND)
                    ->setParentTxnId($this->getParentTransactionId())
                    ->setIsClosed(1);

                $this->addTransaction($transaction);
            }

        } catch (\Exception $e) {
            $this->_logger->error('Payment refund failed', [
                'order_id' => $this->getOrder()->getEntityId(),
                'creditmemo_id' => $creditmemo->getEntityId(),
                'amount' => $baseAmount,
                'error' => $e->getMessage()
            ]);

            throw new \Magento\Framework\Exception\LocalizedException(
                __('Payment refund failed: %1', $e->getMessage())
            );
        }

        return $this;
    }
}
```

#### Custom Payment Method Integration

```php
<?php
namespace Vendor\Module\Model\Payment;

use Magento\Payment\Model\Method\AbstractMethod;

/**
 * Custom payment method
 *
 * Integrates with Sales module for order payment
 */
class CustomPayment extends AbstractMethod
{
    protected $_code = 'custom_payment';
    protected $_isGateway = true;
    protected $_canAuthorize = true;
    protected $_canCapture = true;
    protected $_canRefund = true;
    protected $_canVoid = true;

    /**
     * Authorize payment
     *
     * @param \Magento\Payment\Model\InfoInterface $payment
     * @param float $amount
     * @return $this
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function authorize(\Magento\Payment\Model\InfoInterface $payment, $amount)
    {
        if (!$this->canAuthorize()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('The authorize action is not available.')
            );
        }

        /** @var \Magento\Sales\Model\Order $order */
        $order = $payment->getOrder();

        try {
            // Call external payment gateway
            $response = $this->gatewayClient->authorize([
                'amount' => $amount,
                'currency' => $order->getBaseCurrencyCode(),
                'order_id' => $order->getIncrementId(),
                'customer_email' => $order->getCustomerEmail(),
                'billing_address' => $this->formatAddress($order->getBillingAddress())
            ]);

            if (!$response->isSuccess()) {
                throw new \Magento\Framework\Exception\LocalizedException(
                    __('Payment authorization failed: %1', $response->getMessage())
                );
            }

            // Store transaction details
            $payment->setTransactionId($response->getTransactionId());
            $payment->setIsTransactionClosed(false);
            $payment->setAdditionalInformation('gateway_response', $response->getData());

        } catch (\Exception $e) {
            $this->_logger->error('Payment gateway authorization failed', [
                'order_id' => $order->getEntityId(),
                'amount' => $amount,
                'error' => $e->getMessage()
            ]);

            throw new \Magento\Framework\Exception\LocalizedException(
                __('Payment gateway error: %1', $e->getMessage())
            );
        }

        return $this;
    }

    /**
     * Capture payment
     *
     * @param \Magento\Payment\Model\InfoInterface $payment
     * @param float $amount
     * @return $this
     */
    public function capture(\Magento\Payment\Model\InfoInterface $payment, $amount)
    {
        $authTransactionId = $payment->getParentTransactionId();

        if ($authTransactionId) {
            // Capture existing authorization
            $response = $this->gatewayClient->capture(
                $authTransactionId,
                $amount
            );
        } else {
            // Sale (authorize + capture in one step)
            $response = $this->gatewayClient->sale([
                'amount' => $amount,
                'currency' => $payment->getOrder()->getBaseCurrencyCode(),
                'order_id' => $payment->getOrder()->getIncrementId()
            ]);
        }

        if (!$response->isSuccess()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Payment capture failed: %1', $response->getMessage())
            );
        }

        $payment->setTransactionId($response->getTransactionId());
        $payment->setIsTransactionClosed(true);

        return $this;
    }

    /**
     * Refund payment
     *
     * @param \Magento\Payment\Model\InfoInterface $payment
     * @param float $amount
     * @return $this
     */
    public function refund(\Magento\Payment\Model\InfoInterface $payment, $amount)
    {
        $captureTransactionId = $payment->getParentTransactionId();

        $response = $this->gatewayClient->refund(
            $captureTransactionId,
            $amount
        );

        if (!$response->isSuccess()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Payment refund failed: %1', $response->getMessage())
            );
        }

        $payment->setTransactionId($response->getTransactionId());
        $payment->setIsTransactionClosed(true);

        return $this;
    }
}
```

### Integration with Magento_Tax

Tax calculation and application to orders, invoices, and credit memos.

#### Tax Collection for Orders

```php
<?php
namespace Magento\Tax\Model\Sales\Total\Quote;

/**
 * Tax totals collector
 *
 * Applied during quote collection, carried to order
 */
class Tax extends \Magento\Quote\Model\Quote\Address\Total\AbstractTotal
{
    /**
     * Collect tax totals
     *
     * @param \Magento\Quote\Model\Quote $quote
     * @param \Magento\Quote\Api\Data\ShippingAssignmentInterface $shippingAssignment
     * @param \Magento\Quote\Model\Quote\Address\Total $total
     * @return $this
     */
    public function collect(
        \Magento\Quote\Model\Quote $quote,
        \Magento\Quote\Api\Data\ShippingAssignmentInterface $shippingAssignment,
        \Magento\Quote\Model\Quote\Address\Total $total
    ) {
        $this->clearValues($total);

        if (!$shippingAssignment->getItems()) {
            return $this;
        }

        $address = $shippingAssignment->getShipping()->getAddress();

        // Calculate tax for each item
        $taxDetails = $this->taxCalculation->calculateTax(
            $this->getAddressQuoteDetails($shippingAssignment, $total),
            $quote->getStoreId()
        );

        // Apply tax to items
        foreach ($taxDetails->getItems() as $itemTaxDetails) {
            $quoteItem = $this->getQuoteItem($itemTaxDetails->getCode(), $shippingAssignment);

            if ($quoteItem) {
                $quoteItem->setTaxAmount($itemTaxDetails->getRowTax());
                $quoteItem->setBaseTaxAmount($itemTaxDetails->getRowTax());
                $quoteItem->setTaxPercent($itemTaxDetails->getTaxPercent());

                // Store applied taxes for order
                $appliedTaxes = [];
                foreach ($itemTaxDetails->getAppliedTaxes() as $appliedTax) {
                    $appliedTaxes[] = [
                        'title' => $appliedTax->getTitle(),
                        'percent' => $appliedTax->getPercent(),
                        'amount' => $appliedTax->getAmount(),
                        'rates' => $appliedTax->getRates()
                    ];
                }

                $quoteItem->setAppliedTaxes($appliedTaxes);
            }
        }

        // Set total tax amounts
        $total->setTaxAmount($taxDetails->getTaxAmount());
        $total->setBaseTaxAmount($taxDetails->getTaxAmount());
        $total->setGrandTotal($total->getGrandTotal() + $taxDetails->getTaxAmount());
        $total->setBaseGrandTotal($total->getBaseGrandTotal() + $taxDetails->getTaxAmount());

        return $this;
    }
}
```

#### Tax in Order Entity

```php
<?php
// Tax data stored in order
$order->setTaxAmount($taxAmount);
$order->setBaseTaxAmount($baseTaxAmount);
$order->setShippingTaxAmount($shippingTaxAmount);
$order->setBaseShippingTaxAmount($baseShippingTaxAmount);

// Tax applied to order items
foreach ($order->getAllItems() as $item) {
    $item->setTaxAmount($itemTaxAmount);
    $item->setBaseTaxAmount($baseItemTaxAmount);
    $item->setTaxPercent($taxPercent);

    // Detailed tax breakdown
    $item->setAppliedTaxes([
        [
            'title' => 'US-CA-*-Rate 1',
            'percent' => 8.25,
            'amount' => 16.50,
            'rates' => [
                [
                    'code' => 'US-CA-*-Rate 1',
                    'title' => 'California State Tax',
                    'percent' => 8.25
                ]
            ]
        ]
    ]);
}
```

### Integration with Magento_Shipping

Shipping method selection and rate calculation integration.

#### Shipping in Orders

```php
<?php
namespace Magento\Sales\Model\Order;

/**
 * Shipping information in orders
 */
class Shipping
{
    /**
     * Get shipping information from order
     *
     * @param \Magento\Sales\Model\Order $order
     * @return array
     */
    public function getShippingInfo(\Magento\Sales\Model\Order $order): array
    {
        return [
            'method' => $order->getShippingMethod(),
            'description' => $order->getShippingDescription(),
            'amount' => $order->getShippingAmount(),
            'base_amount' => $order->getBaseShippingAmount(),
            'tax_amount' => $order->getShippingTaxAmount(),
            'base_tax_amount' => $order->getBaseShippingTaxAmount(),
            'discount_amount' => $order->getShippingDiscountAmount(),
            'base_discount_amount' => $order->getBaseShippingDiscountAmount()
        ];
    }
}
```

#### Shipment Label Generation

```php
<?php
namespace Vendor\Module\Model\Shipment;

use Magento\Sales\Model\Order\Shipment;

/**
 * Generate shipping labels for shipments
 */
class LabelGenerator
{
    public function __construct(
        private \Magento\Shipping\Model\CarrierFactory $carrierFactory,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Generate shipping label
     *
     * @param Shipment $shipment
     * @return string Base64 encoded label
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function generateLabel(Shipment $shipment): string
    {
        $order = $shipment->getOrder();
        $shippingMethod = $order->getShippingMethod();

        // Parse carrier code from shipping method
        list($carrierCode, $methodCode) = explode('_', $shippingMethod, 2);

        // Get carrier instance
        $carrier = $this->carrierFactory->create($carrierCode, $order->getStoreId());

        if (!$carrier || !$carrier->isShippingLabelsAvailable()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Shipping labels are not available for this carrier.')
            );
        }

        try {
            // Request label from carrier
            $request = $this->buildLabelRequest($shipment, $carrier);
            $response = $carrier->requestToShipment($request);

            if ($response->hasErrors()) {
                throw new \Magento\Framework\Exception\LocalizedException(
                    __('Carrier error: %1', implode(', ', $response->getErrors()))
                );
            }

            // Save tracking number
            if ($trackingNumber = $response->getInfo()[0]['tracking_number'] ?? null) {
                $track = $this->trackFactory->create();
                $track->setNumber($trackingNumber)
                    ->setCarrierCode($carrierCode)
                    ->setTitle($carrier->getConfigData('title'));

                $shipment->addTrack($track);
            }

            // Return label content
            return $response->getInfo()[0]['label_content'] ?? '';

        } catch (\Exception $e) {
            $this->logger->error('Shipping label generation failed', [
                'shipment_id' => $shipment->getEntityId(),
                'carrier' => $carrierCode,
                'error' => $e->getMessage()
            ]);

            throw new \Magento\Framework\Exception\LocalizedException(
                __('Failed to generate shipping label: %1', $e->getMessage())
            );
        }
    }

    /**
     * Build label request for carrier
     *
     * @param Shipment $shipment
     * @param \Magento\Shipping\Model\Carrier\AbstractCarrier $carrier
     * @return \Magento\Framework\DataObject
     */
    private function buildLabelRequest(
        Shipment $shipment,
        \Magento\Shipping\Model\Carrier\AbstractCarrier $carrier
    ): \Magento\Framework\DataObject {
        $order = $shipment->getOrder();

        $request = new \Magento\Framework\DataObject([
            'order_shipment' => $shipment,
            'shipper_contact_person_name' => $carrier->getConfigData('contact_name'),
            'shipper_contact_person_first_name' => $carrier->getConfigData('contact_first_name'),
            'shipper_contact_person_last_name' => $carrier->getConfigData('contact_last_name'),
            'shipper_contact_company_name' => $carrier->getConfigData('company_name'),
            'shipper_contact_phone_number' => $carrier->getConfigData('phone_number'),
            'shipper_email' => $carrier->getConfigData('email'),
            'shipper_address_street' => $carrier->getConfigData('street'),
            'shipper_address_city' => $carrier->getConfigData('city'),
            'shipper_address_state_or_province_code' => $carrier->getConfigData('region_id'),
            'shipper_address_postal_code' => $carrier->getConfigData('postcode'),
            'shipper_address_country_code' => $carrier->getConfigData('country_id'),
            'recipient_contact_person_name' => $order->getShippingAddress()->getName(),
            'recipient_contact_phone_number' => $order->getShippingAddress()->getTelephone(),
            'recipient_email' => $order->getCustomerEmail(),
            'recipient_address_street' => implode(' ', $order->getShippingAddress()->getStreet()),
            'recipient_address_city' => $order->getShippingAddress()->getCity(),
            'recipient_address_state_or_province_code' => $order->getShippingAddress()->getRegionCode(),
            'recipient_address_postal_code' => $order->getShippingAddress()->getPostcode(),
            'recipient_address_country_code' => $order->getShippingAddress()->getCountryId(),
            'packages' => $this->buildPackagesData($shipment),
            'order_id' => $order->getIncrementId(),
            'reference_data' => $order->getIncrementId()
        ]);

        return $request;
    }

    /**
     * Build packages data for shipment
     *
     * @param Shipment $shipment
     * @return array
     */
    private function buildPackagesData(Shipment $shipment): array
    {
        $packages = [];
        $totalWeight = 0;
        $totalValue = 0;

        foreach ($shipment->getAllItems() as $item) {
            $orderItem = $item->getOrderItem();
            $totalWeight += $orderItem->getWeight() * $item->getQty();
            $totalValue += $orderItem->getPrice() * $item->getQty();
        }

        $packages[] = [
            'params' => [
                'weight' => $totalWeight,
                'customs_value' => $totalValue,
                'length' => 10,
                'width' => 10,
                'height' => 10,
                'weight_units' => 'POUND',
                'dimension_units' => 'INCH',
                'content_type' => 'MERCHANDISE',
                'content_type_other' => ''
            ],
            'items' => []
        ];

        return $packages;
    }
}
```

### Integration with Magento_Inventory (MSI)

Multi-Source Inventory integration for stock management.

#### Inventory Reservation During Order Placement

```php
<?php
namespace Magento\InventorySales\Plugin\Sales\OrderManagement;

use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Api\OrderManagementInterface;

/**
 * Plugin to create inventory reservations
 */
class AppendReservationsAfterOrderPlacementExtend
{
    public function __construct(
        private \Magento\InventorySalesApi\Api\PlaceReservationsForSalesEventInterface $placeReservations,
        private \Magento\InventoryReservations\Model\ReservationBuilder $reservationBuilder,
        private \Magento\InventorySales\Model\GetSkuFromOrderItem $getSkuFromOrderItem
    ) {}

    /**
     * Create inventory reservations after order placement
     *
     * @param OrderManagementInterface $subject
     * @param OrderInterface $order
     * @return OrderInterface
     */
    public function afterPlace(
        OrderManagementInterface $subject,
        OrderInterface $order
    ): OrderInterface {
        $reservations = [];

        foreach ($order->getItems() as $item) {
            if (!$item->getIsVirtual()) {
                $sku = $this->getSkuByProductId->execute((int)$item->getProductId());

                // Create reservation (negative qty = reserved)
                $reservations[] = $this->reservationBuilder
                    ->setSku($sku)
                    ->setQuantity(-(float)$item->getQtyOrdered())
                    ->setStockId($this->getStockIdForOrder($order))
                    ->setMetadata(json_encode([
                        'event_type' => 'order_placed',
                        'object_type' => 'order',
                        'object_id' => $order->getEntityId(),
                        'object_increment_id' => $order->getIncrementId()
                    ]))
                    ->build();
            }
        }

        if ($reservations) {
            $this->placeReservations->execute($reservations);
        }

        return $order;
    }

    /**
     * Get stock ID for order
     *
     * @param OrderInterface $order
     * @return int
     */
    private function getStockIdForOrder(OrderInterface $order): int
    {
        // Get stock ID based on order's sales channel (website)
        return $this->stockByWebsiteId->execute((int)$order->getStore()->getWebsiteId());
    }
}
```

#### Inventory Compensation During Order Cancellation

```php
<?php
namespace Magento\InventorySales\Plugin\Sales\OrderManagement;

use Magento\Sales\Api\OrderManagementInterface;

/**
 * Plugin to compensate inventory reservations on cancellation
 */
class CancelOrderReservationExtend
{
    /**
     * Compensate reservation after order cancellation
     *
     * @param OrderManagementInterface $subject
     * @param bool $result
     * @param int $orderId
     * @return bool
     */
    public function afterCancel(
        OrderManagementInterface $subject,
        bool $result,
        $orderId
    ): bool {
        if ($result) {
            $order = $this->orderRepository->get($orderId);
            $reservations = [];

            foreach ($order->getItems() as $item) {
                if (!$item->getIsVirtual()) {
                    $sku = $this->getSkuByProductId->execute((int)$item->getProductId());

                    // Create compensation (positive qty = released)
                    $reservations[] = $this->reservationBuilder
                        ->setSku($sku)
                        ->setQuantity((float)$item->getQtyOrdered() - $item->getQtyRefunded())
                        ->setStockId($this->getStockIdForOrder($order))
                        ->setMetadata(json_encode([
                            'event_type' => 'order_canceled',
                            'object_type' => 'order',
                            'object_id' => $order->getEntityId()
                        ]))
                        ->build();
                }
            }

            if ($reservations) {
                $this->placeReservations->execute($reservations);
            }
        }

        return $result;
    }
}
```

### Integration with Magento_Customer

Customer data and account integration.

#### Customer Order History

```php
<?php
namespace Vendor\Module\Model\Customer;

use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;

/**
 * Customer order history service
 */
class OrderHistory
{
    public function __construct(
        private OrderRepositoryInterface $orderRepository,
        private SearchCriteriaBuilder $searchCriteriaBuilder
    ) {}

    /**
     * Get customer order history
     *
     * @param int $customerId
     * @param int $pageSize
     * @param int $currentPage
     * @return \Magento\Sales\Api\Data\OrderSearchResultInterface
     */
    public function getOrderHistory(
        int $customerId,
        int $pageSize = 20,
        int $currentPage = 1
    ): \Magento\Sales\Api\Data\OrderSearchResultInterface {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('customer_id', $customerId, 'eq')
            ->addSortOrder('created_at', 'DESC')
            ->setPageSize($pageSize)
            ->setCurrentPage($currentPage)
            ->create();

        return $this->orderRepository->getList($searchCriteria);
    }

    /**
     * Get customer lifetime value
     *
     * @param int $customerId
     * @return float
     */
    public function getCustomerLifetimeValue(int $customerId): float
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('customer_id', $customerId, 'eq')
            ->addFilter('state', ['complete', 'processing'], 'in')
            ->create();

        $orders = $this->orderRepository->getList($searchCriteria);

        $lifetimeValue = 0.0;
        foreach ($orders->getItems() as $order) {
            $lifetimeValue += $order->getGrandTotal();
        }

        return $lifetimeValue;
    }
}
```

## External System Integrations

### ERP Integration Pattern

```php
<?php
namespace Vendor\Module\Service;

use Magento\Sales\Api\Data\OrderInterface;

/**
 * ERP integration service
 *
 * Pattern for integrating orders with external ERP systems
 */
class ErpIntegration
{
    public function __construct(
        private \Vendor\Module\Gateway\ErpClient $erpClient,
        private \Vendor\Module\Model\ErpOrderMapper $orderMapper,
        private \Vendor\Module\Model\ErpSyncQueueFactory $queueFactory,
        private \Vendor\Module\Model\ResourceModel\ErpSyncQueue $queueResource,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Sync order to ERP
     *
     * @param OrderInterface $order
     * @return bool
     */
    public function syncOrder(OrderInterface $order): bool
    {
        try {
            // Map Magento order to ERP format
            $erpOrderData = $this->orderMapper->mapOrderToErp($order);

            // Send to ERP
            $response = $this->erpClient->createOrder($erpOrderData);

            if (!$response->isSuccess()) {
                throw new \RuntimeException(
                    'ERP order creation failed: ' . $response->getError()
                );
            }

            // Store ERP order ID on Magento order
            $order->setData('erp_order_id', $response->getErpOrderId());
            $this->orderRepository->save($order);

            $this->logger->info('Order synced to ERP', [
                'order_id' => $order->getEntityId(),
                'erp_order_id' => $response->getErpOrderId()
            ]);

            return true;

        } catch (\Exception $e) {
            $this->logger->error('ERP sync failed', [
                'order_id' => $order->getEntityId(),
                'error' => $e->getMessage()
            ]);

            // Queue for retry
            $this->queueForRetry($order);

            return false;
        }
    }

    /**
     * Queue order for ERP sync retry
     *
     * @param OrderInterface $order
     * @return void
     */
    private function queueForRetry(OrderInterface $order): void
    {
        $queueEntry = $this->queueFactory->create();
        $queueEntry->setData([
            'order_id' => $order->getEntityId(),
            'status' => 'pending',
            'retry_count' => 0,
            'next_retry_at' => date('Y-m-d H:i:s', strtotime('+5 minutes')),
            'created_at' => date('Y-m-d H:i:s')
        ]);

        $this->queueResource->save($queueEntry);
    }
}
```

### Warehouse Management System (WMS) Integration

```php
<?php
namespace Vendor\Module\Service;

use Magento\Sales\Api\Data\ShipmentInterface;

/**
 * WMS integration for shipment processing
 */
class WmsIntegration
{
    public function __construct(
        private \Vendor\Module\Gateway\WmsClient $wmsClient,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Notify WMS of new order
     *
     * @param \Magento\Sales\Api\Data\OrderInterface $order
     * @return bool
     */
    public function notifyNewOrder(\Magento\Sales\Api\Data\OrderInterface $order): bool
    {
        try {
            $wmsOrder = [
                'order_number' => $order->getIncrementId(),
                'customer_name' => $order->getCustomerName(),
                'shipping_address' => [
                    'name' => $order->getShippingAddress()->getName(),
                    'street' => $order->getShippingAddress()->getStreet(),
                    'city' => $order->getShippingAddress()->getCity(),
                    'region' => $order->getShippingAddress()->getRegion(),
                    'postcode' => $order->getShippingAddress()->getPostcode(),
                    'country' => $order->getShippingAddress()->getCountryId(),
                    'telephone' => $order->getShippingAddress()->getTelephone()
                ],
                'items' => []
            ];

            foreach ($order->getItems() as $item) {
                $wmsOrder['items'][] = [
                    'sku' => $item->getSku(),
                    'name' => $item->getName(),
                    'qty' => $item->getQtyOrdered(),
                    'weight' => $item->getWeight()
                ];
            }

            $response = $this->wmsClient->createPickList($wmsOrder);

            if (!$response->isSuccess()) {
                throw new \RuntimeException($response->getError());
            }

            $this->logger->info('Order sent to WMS', [
                'order_id' => $order->getEntityId(),
                'wms_pick_list_id' => $response->getPickListId()
            ]);

            return true;

        } catch (\Exception $e) {
            $this->logger->error('WMS notification failed', [
                'order_id' => $order->getEntityId(),
                'error' => $e->getMessage()
            ]);

            return false;
        }
    }

    /**
     * Import shipment from WMS
     *
     * @param string $wmsShipmentId
     * @return ShipmentInterface|null
     */
    public function importShipment(string $wmsShipmentId): ?ShipmentInterface
    {
        try {
            // Get shipment data from WMS
            $wmsShipment = $this->wmsClient->getShipment($wmsShipmentId);

            // Find corresponding Magento order
            $order = $this->orderRepository->get($wmsShipment->getOrderId());

            // Create Magento shipment
            $shipment = $this->shipmentFactory->create($order);

            // Add tracking information
            foreach ($wmsShipment->getTrackingNumbers() as $tracking) {
                $track = $this->trackFactory->create();
                $track->setNumber($tracking->getNumber())
                    ->setCarrierCode($tracking->getCarrier())
                    ->setTitle($tracking->getCarrierTitle());

                $shipment->addTrack($track);
            }

            // Save shipment
            $shipment->register();
            $this->shipmentRepository->save($shipment);

            $this->logger->info('Shipment imported from WMS', [
                'order_id' => $order->getEntityId(),
                'shipment_id' => $shipment->getEntityId(),
                'wms_shipment_id' => $wmsShipmentId
            ]);

            return $shipment;

        } catch (\Exception $e) {
            $this->logger->error('WMS shipment import failed', [
                'wms_shipment_id' => $wmsShipmentId,
                'error' => $e->getMessage()
            ]);

            return null;
        }
    }
}
```

---

**Assumptions:**
- Adobe Commerce 2.4.7+ with PHP 8.2+
- MSI (Multi-Source Inventory) enabled for inventory management
- Custom integrations use message queues for async processing
- External systems provide REST/SOAP APIs

**Why This Approach:**
- Field mapping via fieldset.xml for upgrade-safe conversions
- Extension attributes for custom data without table modifications
- Plugin-based integrations maintain modularity
- Message queues prevent blocking order operations

**Security Impact:**
- External API credentials stored in encrypted core_config_data
- API authentication tokens rotated regularly
- PII data encrypted in transit to external systems
- API rate limiting to prevent abuse

**Performance Impact:**
- Quote to order conversion optimized with batch operations
- External integrations queued for async processing
- Inventory reservations use indexed tables
- Tax calculations cached when possible

**Backward Compatibility:**
- Field mappings stable across minor versions
- Extension attributes backwards compatible
- Service contract interfaces maintained
- Plugin interception points preserved

**Tests to Add:**
- Integration tests: Quote to order conversion with all data
- Unit tests: Field mapping, tax calculation
- Functional tests: End-to-end order with external integrations
- API tests: External system communication

**Docs to Update:**
- README.md: Link integration overview
- ARCHITECTURE.md: Reference integration patterns
- Custom module docs: Document your integrations
