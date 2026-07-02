---
title: "Custom Payment Method Development: Building Secure Payment Integrations"
description: "Complete guide to developing custom payment methods in Magento 2: gateway architecture, payment adapter implementation, vault integration, refunds, void operations, testing strategies, and PCI compliance"
type: "tutorial"
tier: 1
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "90 minutes"
topics:
  - payment-gateway
  - payment-method
  - vault
  - tokenization
  - refunds
  - pci-compliance
  - security
  - testing
  - service-contracts
last_updated: "2026-02-07"
---

# Custom Payment Method Development: Building Secure Payment Integrations

## Learning Objectives

By completing this tutorial, you will:

- Understand Magento 2's payment gateway architecture and payment facade pattern
- Implement a complete custom payment method with gateway commands
- Build payment adapters for third-party payment processors
- Integrate vault (tokenization) for stored payment methods
- Implement refund, void, and capture operations correctly
- Handle payment authorization and settlement flows
- Implement proper error handling and transaction logging
- Follow PCI compliance requirements for payment data handling
- Write comprehensive tests for payment methods
- Deploy payment methods securely to production

## Introduction

Payment method development in Magento 2 requires careful attention to security, compliance, and reliability. Unlike other extensions, payment methods handle sensitive financial data and must be implemented with rigorous standards.

### Payment Gateway Architecture

Magento 2 uses a **gateway-based architecture** that separates payment operations into discrete, testable command classes:

- **Gateway Commands**: Authorization, capture, sale, refund, void, cancel
- **Request Builders**: Construct API requests from order/payment data
- **Handlers**: Process gateway responses and update payment/order state
- **Validators**: Validate responses and ensure data integrity
- **Adapters**: Abstract third-party gateway client libraries

This architecture provides:
- **Separation of concerns**: Each operation is isolated
- **Testability**: Commands can be unit tested independently
- **Reusability**: Request builders and handlers are shared across commands
- **Flexibility**: Easy to add new operations or modify existing ones

### Payment Method vs Payment Gateway

**Payment Method**: The customer-facing payment option (e.g., "Credit Card", "PayPal Express")
**Payment Gateway**: The backend service that processes transactions (e.g., Stripe API, Braintree SDK)

One gateway can support multiple payment methods (e.g., Braintree supports cards, PayPal, Venmo).

### PCI Compliance Overview

**PCI DSS** (Payment Card Industry Data Security Standard) governs how cardholder data is stored, transmitted, and processed.

**Key principles for Magento developers:**
- **Never store** CVV/CVC codes (prohibited by PCI)
- **Never store** full card numbers (use tokenization)
- **Use HTTPS** for all payment communication
- **Validate input** to prevent injection attacks
- **Log sanitized data** only (mask card numbers, omit CVV)
- **Use gateway-hosted forms** or tokenization to minimize PCI scope

**SAQ A-EP** (Self-Assessment Questionnaire) is the most common level for Magento merchants using payment gateways that host the payment form or tokenize cards before submission.

## Payment Method Structure

### Module Structure

A complete payment method module includes:

```
Vendor/PaymentGateway/
├── registration.php
├── composer.json
├── etc/
│   ├── module.xml
│   ├── config.xml                    # Default configuration
│   ├── adminhtml/
│   │   └── system.xml                # Admin configuration fields
│   └── di.xml                        # Gateway command pool, virtual types
├── Gateway/
│   ├── Config/                       # Gateway configuration reader
│   │   └── Config.php
│   ├── Http/
│   │   ├── Client/                   # HTTP client adapter
│   │   │   └── ClientAdapter.php
│   │   └── TransferFactory.php       # HTTP transfer object factory
│   ├── Request/                      # Request builders
│   │   ├── AuthorizationRequest.php
│   │   ├── CaptureRequest.php
│   │   ├── RefundRequest.php
│   │   └── VoidRequest.php
│   ├── Response/                     # Response handlers
│   │   ├── TransactionIdHandler.php
│   │   ├── CardDetailsHandler.php
│   │   └── VaultDetailsHandler.php
│   ├── Validator/                    # Response validators
│   │   ├── ResponseCodeValidator.php
│   │   └── GeneralResponseValidator.php
│   ├── Command/                      # Gateway commands (optional explicit classes)
│   │   └── CaptureStrategyCommand.php
│   └── ErrorMapper.php               # Map gateway errors to Magento messages
├── Model/
│   ├── Ui/
│   │   └── ConfigProvider.php        # Frontend configuration provider
│   └── Adminhtml/
│       └── Source/
│           └── PaymentAction.php     # Payment action options
├── Observer/
│   └── DataAssignObserver.php        # Assign additional payment data
└── view/
    ├── frontend/
    │   ├── layout/
    │   │   └── checkout_index_index.xml
    │   ├── web/
    │   │   ├── js/
    │   │   │   └── view/payment/
    │   │   │       └── method-renderer.js  # Knockout payment renderer
    │   │   └── template/payment/
    │   │       └── form.html                # Payment form template
    │   └── requirejs-config.js
    └── adminhtml/
        └── templates/
            └── info.phtml                     # Admin payment info display
```

### Module Registration and Dependencies

**registration.php**

```php
<?php
declare(strict_types=1);

use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Vendor_PaymentGateway',
    __DIR__
);
```

**composer.json**

```json
{
    "name": "vendor/magento2-payment-gateway",
    "description": "Custom payment gateway integration for Magento 2",
    "type": "magento2-module",
    "version": "1.0.0",
    "license": "proprietary",
    "require": {
        "php": "~8.2.0||~8.3.0",
        "magento/framework": "^103.0",
        "magento/module-payment": "^100.4",
        "magento/module-checkout": "^100.4",
        "magento/module-quote": "^101.2",
        "magento/module-sales": "^103.0",
        "magento/module-vault": "^101.2",
        "guzzlehttp/guzzle": "^7.5"
    },
    "autoload": {
        "files": [
            "registration.php"
        ],
        "psr-4": {
            "Vendor\\PaymentGateway\\": ""
        }
    }
}
```

**etc/module.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Vendor_PaymentGateway">
        <sequence>
            <module name="Magento_Payment"/>
            <module name="Magento_Checkout"/>
            <module name="Magento_Vault"/>
            <module name="Magento_Sales"/>
        </sequence>
    </module>
</config>
```

### Payment Method Configuration

**etc/config.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Store:etc/config.xsd">
    <default>
        <payment>
            <vendor_gateway>
                <active>0</active>
                <model>VendorGatewayFacade</model>
                <title>Credit Card (Custom Gateway)</title>
                <payment_action>authorize</payment_action>
                <can_authorize>1</can_authorize>
                <can_capture>1</can_capture>
                <can_void>1</can_void>
                <can_refund>1</can_refund>
                <can_cancel>1</can_cancel>
                <can_use_checkout>1</can_use_checkout>
                <can_use_internal>1</can_use_internal>
                <can_capture_partial>1</can_capture_partial>
                <can_refund_partial_per_invoice>1</can_refund_partial_per_invoice>
                <is_gateway>1</is_gateway>
                <order_status>pending</order_status>
                <sort_order>10</sort_order>
                <debugReplaceKeys>api_key,api_secret,card_number,cvv</debugReplaceKeys>
                <paymentInfoKeys>cc_type,cc_last4,cc_exp_month,cc_exp_year</paymentInfoKeys>
            </vendor_gateway>
        </payment>
    </default>
</config>
```

**Configuration flags explained:**

- `payment_action`: `authorize` (auth only), `authorize_capture` (sale), `order` (async)
- `can_authorize`: Supports authorization without capture
- `can_capture`: Supports capturing authorized amounts
- `can_void`: Can void authorized (uncaptured) transactions
- `can_refund`: Can refund captured amounts
- `can_capture_partial`: Supports partial captures
- `can_refund_partial_per_invoice`: Supports partial refunds per invoice
- `is_gateway`: Uses gateway command architecture
- `debugReplaceKeys`: Keys to mask in debug logs (PCI compliance)
- `paymentInfoKeys`: Data to store in `sales_order_payment.additional_information`

### Admin Configuration

**etc/adminhtml/system.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system_file.xsd">
    <system>
        <section id="payment">
            <group id="vendor_gateway" translate="label" type="text" sortOrder="100" showInDefault="1" showInWebsite="1" showInStore="1">
                <label>Custom Payment Gateway</label>
                <field id="active" translate="label" type="select" sortOrder="10" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Enabled</label>
                    <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
                </field>
                <field id="title" translate="label" type="text" sortOrder="20" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Title</label>
                </field>
                <field id="payment_action" translate="label" type="select" sortOrder="30" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Payment Action</label>
                    <source_model>Vendor\PaymentGateway\Model\Adminhtml\Source\PaymentAction</source_model>
                </field>
                <field id="api_endpoint" translate="label" type="select" sortOrder="40" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>API Endpoint</label>
                    <source_model>Vendor\PaymentGateway\Model\Adminhtml\Source\Environment</source_model>
                </field>
                <field id="api_key" translate="label" type="obscure" sortOrder="50" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>API Key</label>
                    <backend_model>Magento\Config\Model\Config\Backend\Encrypted</backend_model>
                </field>
                <field id="api_secret" translate="label" type="obscure" sortOrder="60" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>API Secret</label>
                    <backend_model>Magento\Config\Model\Config\Backend\Encrypted</backend_model>
                </field>
                <field id="debug" translate="label" type="select" sortOrder="70" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Debug Mode</label>
                    <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
                </field>
                <field id="allowspecific" translate="label" type="allowspecific" sortOrder="80" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Payment from Applicable Countries</label>
                    <source_model>Magento\Payment\Model\Config\Source\Allspecificcountries</source_model>
                </field>
                <field id="specificcountry" translate="label" type="multiselect" sortOrder="90" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Payment from Specific Countries</label>
                    <source_model>Magento\Directory\Model\Config\Source\Country</source_model>
                    <can_be_empty>1</can_be_empty>
                </field>
                <field id="min_order_total" translate="label" type="text" sortOrder="100" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Minimum Order Total</label>
                </field>
                <field id="max_order_total" translate="label" type="text" sortOrder="110" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Maximum Order Total</label>
                </field>
                <field id="sort_order" translate="label" type="text" sortOrder="120" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Sort Order</label>
                </field>
            </group>
        </section>
    </system>
</config>
```

**Security notes:**
- API credentials use `type="obscure"` and `backend_model="Magento\Config\Model\Config\Backend\Encrypted"`
- Credentials are encrypted in the database using Magento's encryption key (`app/etc/env.php`)
- Never expose credentials in frontend code or logs

## Gateway Command Architecture

### Dependency Injection Configuration

The core of the payment gateway is configured via `di.xml` using virtual types.

**etc/di.xml** (Part 1: Facade and Command Pool)

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Payment Method Facade -->
    <virtualType name="VendorGatewayFacade" type="Magento\Payment\Model\Method\Adapter">
        <arguments>
            <argument name="code" xsi:type="const">Vendor\PaymentGateway\Gateway\Config\Config::CODE</argument>
            <argument name="formBlockType" xsi:type="string">Magento\Payment\Block\Form\Cc</argument>
            <argument name="infoBlockType" xsi:type="string">Vendor\PaymentGateway\Block\Info</argument>
            <argument name="valueHandlerPool" xsi:type="object">VendorGatewayValueHandlerPool</argument>
            <argument name="commandPool" xsi:type="object">VendorGatewayCommandPool</argument>
            <argument name="validatorPool" xsi:type="object">VendorGatewayValidatorPool</argument>
        </arguments>
    </virtualType>

    <!-- Command Pool -->
    <virtualType name="VendorGatewayCommandPool" type="Magento\Payment\Gateway\Command\CommandPool">
        <arguments>
            <argument name="commands" xsi:type="array">
                <item name="authorize" xsi:type="string">VendorGatewayAuthorizeCommand</item>
                <item name="capture" xsi:type="string">VendorGatewayCaptureStrategyCommand</item>
                <item name="sale" xsi:type="string">VendorGatewaySaleCommand</item>
                <item name="void" xsi:type="string">VendorGatewayVoidCommand</item>
                <item name="refund" xsi:type="string">VendorGatewayRefundCommand</item>
                <item name="cancel" xsi:type="string">VendorGatewayVoidCommand</item>
                <item name="vault_authorize" xsi:type="string">VendorGatewayVaultAuthorizeCommand</item>
                <item name="vault_sale" xsi:type="string">VendorGatewayVaultSaleCommand</item>
            </argument>
        </arguments>
    </virtualType>

    <!-- Authorize Command -->
    <virtualType name="VendorGatewayAuthorizeCommand" type="Magento\Payment\Gateway\Command\GatewayCommand">
        <arguments>
            <argument name="requestBuilder" xsi:type="object">VendorGatewayAuthorizationRequest</argument>
            <argument name="transferFactory" xsi:type="object">Vendor\PaymentGateway\Gateway\Http\TransferFactory</argument>
            <argument name="client" xsi:type="object">Vendor\PaymentGateway\Gateway\Http\Client\TransactionAuthorize</argument>
            <argument name="handler" xsi:type="object">VendorGatewayAuthorizationHandler</argument>
            <argument name="validator" xsi:type="object">VendorGatewayResponseValidator</argument>
        </arguments>
    </virtualType>

    <!-- Sale Command (Authorize + Capture in one call) -->
    <virtualType name="VendorGatewaySaleCommand" type="VendorGatewayAuthorizeCommand">
        <arguments>
            <argument name="requestBuilder" xsi:type="object">VendorGatewaySaleRequest</argument>
        </arguments>
    </virtualType>

    <!-- Capture Strategy Command -->
    <virtualType name="VendorGatewayCaptureStrategyCommand" type="Vendor\PaymentGateway\Gateway\Command\CaptureStrategyCommand">
        <arguments>
            <argument name="commandPool" xsi:type="object">VendorGatewayCommandPool</argument>
        </arguments>
    </virtualType>

    <!-- Void Command -->
    <virtualType name="VendorGatewayVoidCommand" type="Magento\Payment\Gateway\Command\GatewayCommand">
        <arguments>
            <argument name="requestBuilder" xsi:type="object">VendorGatewayVoidRequest</argument>
            <argument name="transferFactory" xsi:type="object">Vendor\PaymentGateway\Gateway\Http\TransferFactory</argument>
            <argument name="client" xsi:type="object">Vendor\PaymentGateway\Gateway\Http\Client\TransactionVoid</argument>
            <argument name="handler" xsi:type="object">VendorGatewayVoidHandler</argument>
            <argument name="validator" xsi:type="object">VendorGatewayResponseValidator</argument>
        </arguments>
    </virtualType>

    <!-- Refund Command -->
    <virtualType name="VendorGatewayRefundCommand" type="Magento\Payment\Gateway\Command\GatewayCommand">
        <arguments>
            <argument name="requestBuilder" xsi:type="object">VendorGatewayRefundRequest</argument>
            <argument name="transferFactory" xsi:type="object">Vendor\PaymentGateway\Gateway\Http\TransferFactory</argument>
            <argument name="client" xsi:type="object">Vendor\PaymentGateway\Gateway\Http\Client\TransactionRefund</argument>
            <argument name="handler" xsi:type="object">VendorGatewayRefundHandler</argument>
            <argument name="validator" xsi:type="object">VendorGatewayResponseValidator</argument>
        </arguments>
    </virtualType>

</config>
```

### Capture Strategy Command

Magento's capture operation has two modes:
1. **Capture** (post-authorization): Capture a previously authorized transaction
2. **Sale**: Authorize and capture in one operation (if no prior authorization exists)

**Gateway/Command/CaptureStrategyCommand.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Command;

use Magento\Payment\Gateway\Command\CommandPoolInterface;
use Magento\Payment\Gateway\CommandInterface;
use Magento\Payment\Gateway\Data\PaymentDataObjectInterface;
use Magento\Payment\Gateway\Helper\SubjectReader;
use Magento\Sales\Model\Order\Payment;

/**
 * Determines whether to execute 'sale' or 'capture' command based on payment state
 */
class CaptureStrategyCommand implements CommandInterface
{
    private const SALE = 'sale';
    private const CAPTURE = 'capture';

    public function __construct(
        private readonly CommandPoolInterface $commandPool
    ) {}

    /**
     * @param array $commandSubject
     * @return void
     * @throws \Magento\Payment\Gateway\Command\CommandException
     */
    public function execute(array $commandSubject): void
    {
        /** @var PaymentDataObjectInterface $paymentDO */
        $paymentDO = SubjectReader::readPayment($commandSubject);

        /** @var Payment $payment */
        $payment = $paymentDO->getPayment();

        // If no transaction ID exists, this is a sale (authorize + capture)
        // If transaction ID exists, this is capturing a previous authorization
        $command = $payment->getAuthorizationTransaction()
            ? self::CAPTURE
            : self::SALE;

        $this->commandPool->get($command)->execute($commandSubject);
    }
}
```

**Why this pattern?**

When admin clicks "Invoice" in order view:
- If payment was **authorized** → execute `capture` (capture the authorization)
- If payment is new (no authorization) → execute `sale` (auth + capture in one API call)

## Request Builders

Request builders construct the payload sent to the payment gateway. They read data from the payment object, order, quote, and configuration.

**Gateway/Request/AuthorizationRequest.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Request;

use Magento\Payment\Gateway\Data\PaymentDataObjectInterface;
use Magento\Payment\Gateway\Helper\SubjectReader;
use Magento\Payment\Gateway\Request\BuilderInterface;
use Magento\Sales\Model\Order\Payment;
use Vendor\PaymentGateway\Gateway\Config\Config;

/**
 * Builds authorization request payload
 */
class AuthorizationRequest implements BuilderInterface
{
    public function __construct(
        private readonly Config $config
    ) {}

    /**
     * @param array $buildSubject
     * @return array
     */
    public function build(array $buildSubject): array
    {
        /** @var PaymentDataObjectInterface $paymentDO */
        $paymentDO = SubjectReader::readPayment($buildSubject);

        /** @var Payment $payment */
        $payment = $paymentDO->getPayment();
        $order = $paymentDO->getOrder();
        $amount = SubjectReader::readAmount($buildSubject);

        return [
            'transaction_type' => 'authorize',
            'merchant_id' => $this->config->getMerchantId(),
            'amount' => $this->formatAmount($amount),
            'currency' => $order->getCurrencyCode(),
            'order_id' => $order->getOrderIncrementId(),
            'card_number' => $payment->getAdditionalInformation('cc_number'),
            'card_exp_month' => $payment->getCcExpMonth(),
            'card_exp_year' => $payment->getCcExpYear(),
            'card_cvv' => $payment->getCcCid(),
            'billing_address' => $this->buildAddress($order->getBillingAddress()),
            'customer_email' => $order->getBillingAddress()->getEmail(),
            'customer_ip' => $order->getRemoteIp(),
        ];
    }

    /**
     * Format amount to gateway format (e.g., 10.00 -> 1000 for cents-based gateways)
     */
    private function formatAmount(float $amount): int
    {
        return (int) round($amount * 100);
    }

    /**
     * Build address array from order address
     */
    private function buildAddress(\Magento\Payment\Gateway\Data\AddressAdapterInterface $address): array
    {
        return [
            'first_name' => $address->getFirstname(),
            'last_name' => $address->getLastname(),
            'address_line1' => $address->getStreetLine1(),
            'address_line2' => $address->getStreetLine2(),
            'city' => $address->getCity(),
            'state' => $address->getRegionCode(),
            'postal_code' => $address->getPostcode(),
            'country' => $address->getCountryId(),
        ];
    }
}
```

**Critical security note:**

In production, **never send raw card data** to your server. Use one of these approaches:

1. **Gateway-hosted tokenization**: Customer enters card on gateway's iframe/redirect, returns token
2. **JavaScript tokenization**: Use gateway's JS SDK to tokenize card in browser before form submission
3. **Direct API tokenization**: Only for PCI-compliant servers (requires SAQ D)

**Modified approach for tokenization:**

```php
// Instead of reading raw card data, read the token:
'payment_token' => $payment->getAdditionalInformation('payment_token'),

// Remove these lines (never send raw card data):
// 'card_number' => $payment->getAdditionalInformation('cc_number'),
// 'card_cvv' => $payment->getCcCid(),
```

### Composite Request Builders

For complex requests, use composite builders that delegate to specialized builders.

**etc/di.xml** (Request Builder Composition)

```xml
<virtualType name="VendorGatewayAuthorizationRequest" type="Magento\Payment\Gateway\Request\BuilderComposite">
    <arguments>
        <argument name="builders" xsi:type="array">
            <item name="transaction" xsi:type="string">Vendor\PaymentGateway\Gateway\Request\TransactionDataBuilder</item>
            <item name="customer" xsi:type="string">Vendor\PaymentGateway\Gateway\Request\CustomerDataBuilder</item>
            <item name="payment" xsi:type="string">Vendor\PaymentGateway\Gateway\Request\PaymentDataBuilder</item>
            <item name="address" xsi:type="string">Vendor\PaymentGateway\Gateway\Request\AddressDataBuilder</item>
            <item name="vault" xsi:type="string">Vendor\PaymentGateway\Gateway\Request\VaultDataBuilder</item>
        </argument>
    </arguments>
</virtualType>
```

Each builder handles one concern:
- `TransactionDataBuilder`: Transaction type, amount, currency, order ID
- `CustomerDataBuilder`: Customer email, IP, user agent
- `PaymentDataBuilder`: Payment token, card type, card metadata
- `AddressDataBuilder`: Billing/shipping addresses
- `VaultDataBuilder`: Tokenization flags for vault integration

## Response Handlers

Response handlers process the gateway's response and update Magento's payment and order state.

**Gateway/Response/TransactionIdHandler.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Response;

use Magento\Payment\Gateway\Helper\SubjectReader;
use Magento\Payment\Gateway\Response\HandlerInterface;
use Magento\Sales\Model\Order\Payment;

/**
 * Handles transaction ID from gateway response
 */
class TransactionIdHandler implements HandlerInterface
{
    /**
     * @param array $handlingSubject
     * @param array $response
     * @return void
     */
    public function handle(array $handlingSubject, array $response): void
    {
        $paymentDO = SubjectReader::readPayment($handlingSubject);

        /** @var Payment $payment */
        $payment = $paymentDO->getPayment();

        // Set transaction ID (critical for capture, void, refund operations)
        $payment->setTransactionId($response['transaction_id']);

        // Set whether transaction is closed (cannot be modified further)
        $payment->setIsTransactionClosed(false);

        // Store raw gateway response for debugging
        $payment->setTransactionAdditionalInfo(
            Payment::TRANSACTION_RAW_DETAILS,
            $response
        );
    }
}
```

**Gateway/Response/CardDetailsHandler.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Response;

use Magento\Payment\Gateway\Helper\SubjectReader;
use Magento\Payment\Gateway\Response\HandlerInterface;
use Magento\Sales\Model\Order\Payment;

/**
 * Stores card metadata (last4, type, expiration) from gateway response
 */
class CardDetailsHandler implements HandlerInterface
{
    /**
     * @param array $handlingSubject
     * @param array $response
     * @return void
     */
    public function handle(array $handlingSubject, array $response): void
    {
        $paymentDO = SubjectReader::readPayment($handlingSubject);

        /** @var Payment $payment */
        $payment = $paymentDO->getPayment();

        // Store card details for display in admin and customer account
        $payment->setCcLast4($response['card']['last4']);
        $payment->setCcType($this->mapCardType($response['card']['brand']));
        $payment->setCcExpMonth($response['card']['exp_month']);
        $payment->setCcExpYear($response['card']['exp_year']);

        // Store additional info (merchant-safe data only, no PCI data)
        $payment->setAdditionalInformation('card_bin', $response['card']['bin']);
        $payment->setAdditionalInformation('card_country', $response['card']['country']);
        $payment->setAdditionalInformation('card_funding', $response['card']['funding']); // credit/debit/prepaid
    }

    /**
     * Map gateway card brand to Magento card type
     */
    private function mapCardType(string $brand): string
    {
        return match (strtolower($brand)) {
            'visa' => 'VI',
            'mastercard', 'master' => 'MC',
            'amex', 'american express' => 'AE',
            'discover' => 'DI',
            'jcb' => 'JCB',
            'diners', 'diners club' => 'DN',
            default => 'OT',
        };
    }
}
```

**Composite Response Handler** (etc/di.xml)

```xml
<virtualType name="VendorGatewayAuthorizationHandler" type="Magento\Payment\Gateway\Response\HandlerChain">
    <arguments>
        <argument name="handlers" xsi:type="array">
            <item name="transaction_id" xsi:type="string">Vendor\PaymentGateway\Gateway\Response\TransactionIdHandler</item>
            <item name="card_details" xsi:type="string">Vendor\PaymentGateway\Gateway\Response\CardDetailsHandler</item>
            <item name="vault_details" xsi:type="string">Vendor\PaymentGateway\Gateway\Response\VaultDetailsHandler</item>
            <item name="fraud_details" xsi:type="string">Vendor\PaymentGateway\Gateway\Response\FraudDetailsHandler</item>
        </argument>
    </arguments>
</virtualType>
```

## Response Validators

Validators ensure the gateway response is valid before handlers process it. If validation fails, the payment is declined.

**Gateway/Validator/ResponseCodeValidator.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Validator;

use Magento\Payment\Gateway\Helper\SubjectReader;
use Magento\Payment\Gateway\Validator\AbstractValidator;
use Magento\Payment\Gateway\Validator\ResultInterface;
use Magento\Payment\Gateway\Validator\ResultInterfaceFactory;

/**
 * Validates gateway response code
 */
class ResponseCodeValidator extends AbstractValidator
{
    private const SUCCESS_CODES = ['approved', 'authorized'];

    public function __construct(
        ResultInterfaceFactory $resultFactory
    ) {
        parent::__construct($resultFactory);
    }

    /**
     * @param array $validationSubject
     * @return ResultInterface
     */
    public function validate(array $validationSubject): ResultInterface
    {
        $response = SubjectReader::readResponse($validationSubject);

        $isValid = isset($response['status'])
            && in_array(strtolower($response['status']), self::SUCCESS_CODES, true);

        $errorMessages = [];

        if (!$isValid) {
            $errorMessages[] = $response['message'] ?? __('Payment gateway rejected the transaction.');
        }

        return $this->createResult($isValid, $errorMessages);
    }
}
```

**Gateway/Validator/GeneralResponseValidator.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Validator;

use Magento\Payment\Gateway\Helper\SubjectReader;
use Magento\Payment\Gateway\Validator\AbstractValidator;
use Magento\Payment\Gateway\Validator\ResultInterface;
use Magento\Payment\Gateway\Validator\ResultInterfaceFactory;

/**
 * Validates general response structure and required fields
 */
class GeneralResponseValidator extends AbstractValidator
{
    private const REQUIRED_FIELDS = [
        'status',
        'transaction_id',
        'amount',
    ];

    public function __construct(
        ResultInterfaceFactory $resultFactory
    ) {
        parent::__construct($resultFactory);
    }

    /**
     * @param array $validationSubject
     * @return ResultInterface
     */
    public function validate(array $validationSubject): ResultInterface
    {
        $response = SubjectReader::readResponse($validationSubject);

        $isValid = true;
        $errorMessages = [];

        // Check required fields
        foreach (self::REQUIRED_FIELDS as $field) {
            if (!isset($response[$field])) {
                $isValid = false;
                $errorMessages[] = __('Missing required field: %1', $field);
            }
        }

        // Validate amount matches (prevent response tampering)
        if (isset($response['amount'])) {
            $requestAmount = SubjectReader::readAmount($validationSubject);
            $responseAmount = $response['amount'] / 100; // Convert cents to dollars

            if (abs($requestAmount - $responseAmount) > 0.01) {
                $isValid = false;
                $errorMessages[] = __('Amount mismatch: requested %1, received %2', $requestAmount, $responseAmount);
            }
        }

        return $this->createResult($isValid, $errorMessages);
    }
}
```

**Validator Pool** (etc/di.xml)

```xml
<virtualType name="VendorGatewayValidatorPool" type="Magento\Payment\Gateway\Validator\ValidatorPool">
    <arguments>
        <argument name="validators" xsi:type="array">
            <item name="country" xsi:type="string">VendorGatewayCountryValidator</item>
            <item name="currency" xsi:type="string">VendorGatewayCurrencyValidator</item>
        </argument>
    </arguments>
</virtualType>

<virtualType name="VendorGatewayResponseValidator" type="Magento\Payment\Gateway\Validator\ValidatorComposite">
    <arguments>
        <argument name="validators" xsi:type="array">
            <item name="general" xsi:type="string">Vendor\PaymentGateway\Gateway\Validator\GeneralResponseValidator</item>
            <item name="response_code" xsi:type="string">Vendor\PaymentGateway\Gateway\Validator\ResponseCodeValidator</item>
        </argument>
    </arguments>
</virtualType>
```

## HTTP Client Adapter

The HTTP client adapter abstracts the gateway's HTTP API client.

**Gateway/Http/Client/TransactionAuthorize.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Http\Client;

use Magento\Payment\Gateway\Http\ClientInterface;
use Magento\Payment\Gateway\Http\TransferInterface;
use Psr\Log\LoggerInterface;
use Vendor\PaymentGateway\Gateway\Config\Config;
use GuzzleHttp\Client;
use GuzzleHttp\Exception\GuzzleException;

/**
 * HTTP client for authorization requests
 */
class TransactionAuthorize implements ClientInterface
{
    public function __construct(
        private readonly Config $config,
        private readonly LoggerInterface $logger,
        private readonly Client $httpClient
    ) {}

    /**
     * @param TransferInterface $transferObject
     * @return array
     * @throws \Magento\Payment\Gateway\Http\ClientException
     */
    public function placeRequest(TransferInterface $transferObject): array
    {
        $request = $transferObject->getBody();

        try {
            $response = $this->httpClient->post(
                $this->config->getApiEndpoint() . '/authorize',
                [
                    'json' => $request,
                    'headers' => [
                        'Authorization' => 'Bearer ' . $this->config->getApiKey(),
                        'Content-Type' => 'application/json',
                        'X-API-Version' => '2024-01-01',
                    ],
                    'timeout' => 30,
                ]
            );

            $body = (string) $response->getBody();
            $result = json_decode($body, true);

            if (json_last_error() !== JSON_ERROR_NONE) {
                throw new \RuntimeException('Invalid JSON response from gateway');
            }

            $this->logTransaction($request, $result);

            return $result;

        } catch (GuzzleException $e) {
            $this->logger->critical('Payment gateway request failed', [
                'exception' => $e->getMessage(),
                'request' => $this->sanitizeLogData($request),
            ]);

            // Return error response in expected format
            return [
                'status' => 'error',
                'message' => __('Payment gateway is unavailable. Please try again later.'),
                'error_code' => 'gateway_unavailable',
            ];
        }
    }

    /**
     * Log transaction details (sanitized)
     */
    private function logTransaction(array $request, array $response): void
    {
        if (!$this->config->isDebugEnabled()) {
            return;
        }

        $this->logger->debug('Payment Gateway Transaction', [
            'request' => $this->sanitizeLogData($request),
            'response' => $this->sanitizeLogData($response),
        ]);
    }

    /**
     * Sanitize sensitive data for logging (PCI compliance)
     */
    private function sanitizeLogData(array $data): array
    {
        $sanitized = $data;

        $sensitiveKeys = [
            'card_number',
            'card_cvv',
            'cvv',
            'api_key',
            'api_secret',
            'authorization',
            'payment_token',
        ];

        foreach ($sensitiveKeys as $key) {
            if (isset($sanitized[$key])) {
                $sanitized[$key] = '***REDACTED***';
            }
        }

        // Mask card number if present (show only last 4)
        if (isset($data['card_number']) && strlen($data['card_number']) >= 4) {
            $sanitized['card_number'] = 'XXXX-XXXX-XXXX-' . substr($data['card_number'], -4);
        }

        return $sanitized;
    }
}
```

**Gateway/Http/TransferFactory.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Http;

use Magento\Payment\Gateway\Http\TransferBuilder;
use Magento\Payment\Gateway\Http\TransferFactoryInterface;
use Magento\Payment\Gateway\Http\TransferInterface;
use Vendor\PaymentGateway\Gateway\Config\Config;

/**
 * Transfer factory for HTTP requests
 */
class TransferFactory implements TransferFactoryInterface
{
    public function __construct(
        private readonly TransferBuilder $transferBuilder,
        private readonly Config $config
    ) {}

    /**
     * @param array $request
     * @return TransferInterface
     */
    public function create(array $request): TransferInterface
    {
        return $this->transferBuilder
            ->setBody($request)
            ->setMethod('POST')
            ->setHeaders([
                'Content-Type' => 'application/json',
                'Authorization' => 'Bearer ' . $this->config->getApiKey(),
            ])
            ->setUri($this->config->getApiEndpoint())
            ->build();
    }
}
```

## Vault Integration (Tokenization)

Vault integration allows customers to save payment methods for future use. This requires tokenizing card data with the gateway and storing the token in Magento's vault.

### Vault Configuration

**etc/config.xml** (add to payment section)

```xml
<vendor_gateway_vault>
    <model>VendorGatewayVaultFacade</model>
    <title>Stored Payment Methods</title>
</vendor_gateway_vault>
```

**etc/di.xml** (Vault Configuration)

```xml
<!-- Vault Facade -->
<virtualType name="VendorGatewayVaultFacade" type="Magento\Vault\Model\Method\Vault">
    <arguments>
        <argument name="config" xsi:type="object">Vendor\PaymentGateway\Gateway\Config\Config</argument>
        <argument name="valueHandlerPool" xsi:type="object">VendorGatewayVaultValueHandlerPool</argument>
        <argument name="vaultProvider" xsi:type="object">VendorGatewayFacade</argument>
        <argument name="code" xsi:type="const">Vendor\PaymentGateway\Gateway\Config\Config::CODE_VAULT</argument>
    </arguments>
</virtualType>

<!-- Vault Value Handler Pool -->
<virtualType name="VendorGatewayVaultValueHandlerPool" type="Magento\Payment\Gateway\Config\ValueHandlerPool">
    <arguments>
        <argument name="handlers" xsi:type="array">
            <item name="default" xsi:type="string">VendorGatewayVaultConfigValueHandler</item>
        </argument>
    </arguments>
</virtualType>

<virtualType name="VendorGatewayVaultConfigValueHandler" type="Magento\Payment\Gateway\Config\ConfigValueHandler">
    <arguments>
        <argument name="configInterface" xsi:type="object">Vendor\PaymentGateway\Gateway\Config\Config</argument>
    </arguments>
</virtualType>
```

### Vault Response Handler

**Gateway/Response/VaultDetailsHandler.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Response;

use Magento\Payment\Gateway\Helper\SubjectReader;
use Magento\Payment\Gateway\Response\HandlerInterface;
use Magento\Sales\Model\Order\Payment;
use Magento\Vault\Api\Data\PaymentTokenInterface;
use Magento\Vault\Api\Data\PaymentTokenInterfaceFactory;
use Magento\Vault\Model\PaymentTokenManagement;

/**
 * Handles vault tokenization from gateway response
 */
class VaultDetailsHandler implements HandlerInterface
{
    public function __construct(
        private readonly PaymentTokenInterfaceFactory $paymentTokenFactory,
        private readonly PaymentTokenManagement $paymentTokenManagement
    ) {}

    /**
     * @param array $handlingSubject
     * @param array $response
     * @return void
     */
    public function handle(array $handlingSubject, array $response): void
    {
        $paymentDO = SubjectReader::readPayment($handlingSubject);

        /** @var Payment $payment */
        $payment = $paymentDO->getPayment();

        // Check if customer requested to save payment method
        $saveCard = $payment->getAdditionalInformation('is_active_payment_token_enabler');

        if (!$saveCard || !isset($response['payment_token'])) {
            return;
        }

        // Create payment token
        /** @var PaymentTokenInterface $paymentToken */
        $paymentToken = $this->paymentTokenFactory->create();

        $paymentToken->setGatewayToken($response['payment_token']);
        $paymentToken->setExpiresAt($this->getExpirationDate($response['card']['exp_year'], $response['card']['exp_month']));
        $paymentToken->setTokenDetails($this->convertDetailsToJSON([
            'type' => $payment->getCcType(),
            'maskedCC' => $payment->getCcLast4(),
            'expirationDate' => sprintf(
                '%02d/%04d',
                $response['card']['exp_month'],
                $response['card']['exp_year']
            ),
        ]));

        // Link payment token to payment
        $extensionAttributes = $payment->getExtensionAttributes();
        $extensionAttributes->setVaultPaymentToken($paymentToken);
    }

    /**
     * Get expiration date for vault token
     */
    private function getExpirationDate(int $year, int $month): string
    {
        $expDate = new \DateTime(
            sprintf('%d-%02d-01 00:00:00', $year, $month),
            new \DateTimeZone('UTC')
        );
        $expDate->add(new \DateInterval('P1M'));

        return $expDate->format('Y-m-d 00:00:00');
    }

    /**
     * Convert payment token details to JSON
     */
    private function convertDetailsToJSON(array $details): string
    {
        $json = json_encode($details);

        if (json_last_error() !== JSON_ERROR_NONE) {
            throw new \RuntimeException('Failed to encode payment token details');
        }

        return $json;
    }
}
```

### Vault Authorize Command

When a customer uses a saved payment method, Magento executes the `vault_authorize` or `vault_sale` command.

**Gateway/Request/VaultAuthorizationRequest.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Request;

use Magento\Payment\Gateway\Data\PaymentDataObjectInterface;
use Magento\Payment\Gateway\Helper\SubjectReader;
use Magento\Payment\Gateway\Request\BuilderInterface;
use Magento\Vault\Model\PaymentToken;
use Vendor\PaymentGateway\Gateway\Config\Config;

/**
 * Builds authorization request for vault payments
 */
class VaultAuthorizationRequest implements BuilderInterface
{
    public function __construct(
        private readonly Config $config
    ) {}

    /**
     * @param array $buildSubject
     * @return array
     */
    public function build(array $buildSubject): array
    {
        /** @var PaymentDataObjectInterface $paymentDO */
        $paymentDO = SubjectReader::readPayment($buildSubject);

        $payment = $paymentDO->getPayment();
        $order = $paymentDO->getOrder();
        $amount = SubjectReader::readAmount($buildSubject);

        // Get payment token from extension attributes
        $extensionAttributes = $payment->getExtensionAttributes();
        $paymentToken = $extensionAttributes->getVaultPaymentToken();

        if (!$paymentToken instanceof PaymentToken) {
            throw new \RuntimeException('Payment token not found for vault payment');
        }

        return [
            'transaction_type' => 'authorize',
            'merchant_id' => $this->config->getMerchantId(),
            'amount' => (int) round($amount * 100),
            'currency' => $order->getCurrencyCode(),
            'order_id' => $order->getOrderIncrementId(),
            'payment_token' => $paymentToken->getGatewayToken(), // Use saved token
            'customer_email' => $order->getBillingAddress()->getEmail(),
            'customer_ip' => $order->getRemoteIp(),
        ];
    }
}
```

**etc/di.xml** (Vault Commands)

```xml
<!-- Vault Authorize Command -->
<virtualType name="VendorGatewayVaultAuthorizeCommand" type="Magento\Payment\Gateway\Command\GatewayCommand">
    <arguments>
        <argument name="requestBuilder" xsi:type="object">VendorGatewayVaultAuthorizationRequest</argument>
        <argument name="transferFactory" xsi:type="object">Vendor\PaymentGateway\Gateway\Http\TransferFactory</argument>
        <argument name="client" xsi:type="object">Vendor\PaymentGateway\Gateway\Http\Client\TransactionAuthorize</argument>
        <argument name="handler" xsi:type="object">VendorGatewayAuthorizationHandler</argument>
        <argument name="validator" xsi:type="object">VendorGatewayResponseValidator</argument>
    </arguments>
</virtualType>

<!-- Vault Sale Command -->
<virtualType name="VendorGatewayVaultSaleCommand" type="VendorGatewayVaultAuthorizeCommand">
    <arguments>
        <argument name="requestBuilder" xsi:type="object">VendorGatewayVaultSaleRequest</argument>
    </arguments>
</virtualType>
```

## Refund and Void Operations

### Refund Request and Handler

**Gateway/Request/RefundRequest.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Request;

use Magento\Payment\Gateway\Data\PaymentDataObjectInterface;
use Magento\Payment\Gateway\Helper\SubjectReader;
use Magento\Payment\Gateway\Request\BuilderInterface;
use Magento\Sales\Model\Order\Payment;
use Vendor\PaymentGateway\Gateway\Config\Config;

/**
 * Builds refund request payload
 */
class RefundRequest implements BuilderInterface
{
    public function __construct(
        private readonly Config $config
    ) {}

    /**
     * @param array $buildSubject
     * @return array
     */
    public function build(array $buildSubject): array
    {
        /** @var PaymentDataObjectInterface $paymentDO */
        $paymentDO = SubjectReader::readPayment($buildSubject);

        /** @var Payment $payment */
        $payment = $paymentDO->getPayment();
        $amount = SubjectReader::readAmount($buildSubject);

        // Get parent transaction (the capture/sale transaction to refund)
        $parentTransactionId = $payment->getParentTransactionId();

        if (!$parentTransactionId) {
            throw new \RuntimeException('Parent transaction ID not found for refund');
        }

        return [
            'transaction_type' => 'refund',
            'merchant_id' => $this->config->getMerchantId(),
            'transaction_id' => $parentTransactionId,
            'amount' => (int) round($amount * 100),
            'reason' => $payment->getCreditmemo()->getCustomerNote() ?? 'Refund requested by merchant',
        ];
    }
}
```

**Gateway/Response/RefundHandler.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Response;

use Magento\Payment\Gateway\Helper\SubjectReader;
use Magento\Payment\Gateway\Response\HandlerInterface;
use Magento\Sales\Model\Order\Payment;

/**
 * Handles refund response
 */
class RefundHandler implements HandlerInterface
{
    /**
     * @param array $handlingSubject
     * @param array $response
     * @return void
     */
    public function handle(array $handlingSubject, array $response): void
    {
        $paymentDO = SubjectReader::readPayment($handlingSubject);

        /** @var Payment $payment */
        $payment = $paymentDO->getPayment();

        // Set refund transaction ID
        $payment->setTransactionId($response['refund_id']);

        // Refund transaction is closed (cannot be modified)
        $payment->setIsTransactionClosed(true);
        $payment->setShouldCloseParentTransaction(true);

        // Store refund details
        $payment->setTransactionAdditionalInfo(
            Payment::TRANSACTION_RAW_DETAILS,
            [
                'refund_id' => $response['refund_id'],
                'refund_amount' => $response['amount'] / 100,
                'refund_date' => $response['refunded_at'],
                'reason' => $response['reason'] ?? null,
            ]
        );
    }
}
```

### Void Request and Handler

Void operations cancel an authorization before capture. Once a transaction is captured, you cannot void it (you must refund instead).

**Gateway/Request/VoidRequest.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Request;

use Magento\Payment\Gateway\Data\PaymentDataObjectInterface;
use Magento\Payment\Gateway\Helper\SubjectReader;
use Magento\Payment\Gateway\Request\BuilderInterface;
use Magento\Sales\Model\Order\Payment;
use Vendor\PaymentGateway\Gateway\Config\Config;

/**
 * Builds void request payload
 */
class VoidRequest implements BuilderInterface
{
    public function __construct(
        private readonly Config $config
    ) {}

    /**
     * @param array $buildSubject
     * @return array
     */
    public function build(array $buildSubject): array
    {
        /** @var PaymentDataObjectInterface $paymentDO */
        $paymentDO = SubjectReader::readPayment($buildSubject);

        /** @var Payment $payment */
        $payment = $paymentDO->getPayment();

        // Get authorization transaction to void
        $authTransactionId = $payment->getParentTransactionId() ?: $payment->getLastTransId();

        if (!$authTransactionId) {
            throw new \RuntimeException('Authorization transaction ID not found for void');
        }

        return [
            'transaction_type' => 'void',
            'merchant_id' => $this->config->getMerchantId(),
            'transaction_id' => $authTransactionId,
        ];
    }
}
```

**Gateway/Response/VoidHandler.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Response;

use Magento\Payment\Gateway\Helper\SubjectReader;
use Magento\Payment\Gateway\Response\HandlerInterface;
use Magento\Sales\Model\Order\Payment;

/**
 * Handles void response
 */
class VoidHandler implements HandlerInterface
{
    /**
     * @param array $handlingSubject
     * @param array $response
     * @return void
     */
    public function handle(array $handlingSubject, array $response): void
    {
        $paymentDO = SubjectReader::readPayment($handlingSubject);

        /** @var Payment $payment */
        $payment = $paymentDO->getPayment();

        // Set void transaction ID
        $payment->setTransactionId($response['void_id']);

        // Void transaction is closed
        $payment->setIsTransactionClosed(true);
        $payment->setShouldCloseParentTransaction(true);

        // Store void details
        $payment->setTransactionAdditionalInfo(
            Payment::TRANSACTION_RAW_DETAILS,
            [
                'void_id' => $response['void_id'],
                'voided_at' => $response['voided_at'],
                'original_transaction_id' => $response['original_transaction_id'],
            ]
        );
    }
}
```

## Testing Payment Methods

### Unit Tests

**Test/Unit/Gateway/Request/AuthorizationRequestTest.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Test\Unit\Gateway\Request;

use Magento\Payment\Gateway\Data\OrderAdapterInterface;
use Magento\Payment\Gateway\Data\PaymentDataObject;
use Magento\Sales\Model\Order\Payment;
use PHPUnit\Framework\TestCase;
use Vendor\PaymentGateway\Gateway\Config\Config;
use Vendor\PaymentGateway\Gateway\Request\AuthorizationRequest;

class AuthorizationRequestTest extends TestCase
{
    private AuthorizationRequest $builder;
    private Config $config;

    protected function setUp(): void
    {
        $this->config = $this->createMock(Config::class);
        $this->builder = new AuthorizationRequest($this->config);
    }

    public function testBuildReturnsCorrectStructure(): void
    {
        $this->config->method('getMerchantId')->willReturn('test_merchant_123');

        $payment = $this->createMock(Payment::class);
        $payment->method('getAdditionalInformation')->willReturn('tok_test123');
        $payment->method('getCcExpMonth')->willReturn('12');
        $payment->method('getCcExpYear')->willReturn('2025');

        $order = $this->createMock(OrderAdapterInterface::class);
        $order->method('getOrderIncrementId')->willReturn('000000001');
        $order->method('getCurrencyCode')->willReturn('USD');

        $paymentDO = new PaymentDataObject($order, $payment);

        $buildSubject = [
            'payment' => $paymentDO,
            'amount' => 100.50,
        ];

        $result = $this->builder->build($buildSubject);

        $this->assertEquals('authorize', $result['transaction_type']);
        $this->assertEquals('test_merchant_123', $result['merchant_id']);
        $this->assertEquals(10050, $result['amount']); // 100.50 * 100 cents
        $this->assertEquals('USD', $result['currency']);
        $this->assertEquals('000000001', $result['order_id']);
    }
}
```

### Integration Tests

**Test/Integration/Model/PaymentTest.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Test\Integration\Model;

use Magento\Framework\Exception\LocalizedException;
use Magento\Payment\Gateway\Command\CommandPoolInterface;
use Magento\Payment\Model\Method\Adapter;
use Magento\Quote\Model\Quote;
use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;

/**
 * @magentoAppArea frontend
 */
class PaymentTest extends TestCase
{
    private Adapter $paymentMethod;
    private CommandPoolInterface $commandPool;

    protected function setUp(): void
    {
        $objectManager = Bootstrap::getObjectManager();
        $this->paymentMethod = $objectManager->get('VendorGatewayFacade');
        $this->commandPool = $objectManager->get('VendorGatewayCommandPool');
    }

    /**
     * @magentoConfigFixture current_store payment/vendor_gateway/active 1
     * @magentoConfigFixture current_store payment/vendor_gateway/api_key test_key
     */
    public function testPaymentMethodIsAvailable(): void
    {
        /** @var Quote $quote */
        $quote = Bootstrap::getObjectManager()->create(Quote::class);
        $quote->setStoreId(1);
        $quote->setGrandTotal(100.00);

        $this->assertTrue($this->paymentMethod->isAvailable($quote));
    }

    public function testCommandPoolHasRequiredCommands(): void
    {
        $requiredCommands = ['authorize', 'capture', 'void', 'refund', 'sale'];

        foreach ($requiredCommands as $command) {
            $this->assertNotNull(
                $this->commandPool->get($command),
                "Command '$command' is missing from command pool"
            );
        }
    }

    /**
     * @magentoDataFixture Magento/Sales/_files/order.php
     */
    public function testAuthorizeCommand(): void
    {
        $objectManager = Bootstrap::getObjectManager();

        /** @var \Magento\Sales\Model\Order $order */
        $order = $objectManager->create(\Magento\Sales\Model\Order::class);
        $order->loadByIncrementId('100000001');

        $payment = $order->getPayment();
        $payment->setMethod('vendor_gateway');
        $payment->setAdditionalInformation('payment_token', 'tok_test123');

        try {
            $this->paymentMethod->authorize($payment, 100.00);
            $this->assertNotEmpty($payment->getTransactionId());
        } catch (LocalizedException $e) {
            // Expected if gateway is not accessible in test environment
            $this->markTestSkipped('Gateway not accessible: ' . $e->getMessage());
        }
    }
}
```

### Mock Gateway for Testing

For automated testing, implement a mock gateway that simulates responses.

**Gateway/Http/Client/MockClient.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentGateway\Gateway\Http\Client;

use Magento\Payment\Gateway\Http\ClientInterface;
use Magento\Payment\Gateway\Http\TransferInterface;

/**
 * Mock client for testing (used when debug mode is enabled in test environment)
 */
class MockClient implements ClientInterface
{
    /**
     * @param TransferInterface $transferObject
     * @return array
     */
    public function placeRequest(TransferInterface $transferObject): array
    {
        $request = $transferObject->getBody();

        return match ($request['transaction_type']) {
            'authorize' => [
                'status' => 'authorized',
                'transaction_id' => 'mock_txn_' . uniqid(),
                'amount' => $request['amount'],
                'currency' => $request['currency'],
                'card' => [
                    'last4' => '4242',
                    'brand' => 'visa',
                    'exp_month' => $request['card_exp_month'] ?? '12',
                    'exp_year' => $request['card_exp_year'] ?? '2025',
                    'bin' => '424242',
                    'country' => 'US',
                    'funding' => 'credit',
                ],
                'payment_token' => 'mock_tok_' . uniqid(),
            ],
            'void' => [
                'status' => 'voided',
                'void_id' => 'mock_void_' . uniqid(),
                'voided_at' => date('Y-m-d H:i:s'),
                'original_transaction_id' => $request['transaction_id'],
            ],
            'refund' => [
                'status' => 'refunded',
                'refund_id' => 'mock_refund_' . uniqid(),
                'amount' => $request['amount'],
                'refunded_at' => date('Y-m-d H:i:s'),
                'reason' => $request['reason'] ?? null,
            ],
            default => [
                'status' => 'error',
                'message' => 'Unknown transaction type',
            ],
        };
    }
}
```

**etc/di.xml** (Mock Client for Testing)

```xml
<!-- Use mock client in test environment -->
<type name="Vendor\PaymentGateway\Gateway\Http\Client\TransactionAuthorize">
    <arguments>
        <argument name="httpClient" xsi:type="object">Vendor\PaymentGateway\Gateway\Http\Client\MockClient</argument>
    </arguments>
</type>
```

## PCI Compliance and Security

### Never Store These

**Prohibited by PCI DSS:**
- Full card number (PAN) - Use tokenization
- CVV/CVC/CID codes - Never store, even encrypted
- Full magnetic stripe data

**Allowed (with encryption):**
- Card last 4 digits
- Expiration date
- Cardholder name
- Payment token (from gateway)

### Secure Configuration Storage

```xml
<!-- etc/adminhtml/system.xml -->
<field id="api_key" translate="label" type="obscure" sortOrder="50" showInDefault="1" showInWebsite="1" showInStore="0">
    <label>API Key</label>
    <backend_model>Magento\Config\Model\Config\Backend\Encrypted</backend_model>
    <config_path>payment/vendor_gateway/api_key</config_path>
</field>
```

Credentials are encrypted using `Magento\Framework\Encryption\EncryptorInterface` with the key from `app/etc/env.php`.

### Input Validation

```php
// Validate card number format (if handling raw cards)
if (!preg_match('/^[0-9]{13,19}$/', $cardNumber)) {
    throw new LocalizedException(__('Invalid card number format'));
}

// Validate CVV
if (!preg_match('/^[0-9]{3,4}$/', $cvv)) {
    throw new LocalizedException(__('Invalid CVV format'));
}

// Validate expiration
if ($year < date('Y') || ($year == date('Y') && $month < date('m'))) {
    throw new LocalizedException(__('Card has expired'));
}
```

### HTTPS Enforcement

```php
// In ConfigProvider or validator
if (!$this->request->isSecure()) {
    throw new LocalizedException(__('Payment information must be transmitted securely'));
}
```

### Log Sanitization

```php
// Always sanitize logs
private function sanitizeLogData(array $data): array
{
    $sensitiveKeys = [
        'card_number',
        'card_cvv',
        'cvv',
        'api_key',
        'api_secret',
        'authorization',
        'payment_token',
    ];

    foreach ($sensitiveKeys as $key) {
        if (isset($data[$key])) {
            $data[$key] = '***REDACTED***';
        }
    }

    return $data;
}
```

### Frontend Security

**Never send raw card data to your server:**

```javascript
// BAD: Sending raw card data to Magento server
var payload = {
    card_number: cardNumber,
    cvv: cvv,
    exp_month: expMonth,
    exp_year: expYear
};

// GOOD: Tokenize in browser using gateway's SDK
gatewaySDK.tokenize({
    card_number: cardNumber,
    cvv: cvv,
    exp_month: expMonth,
    exp_year: expYear
}).then(function(token) {
    // Send only the token to Magento
    var payload = {
        payment_token: token.id
    };
});
```

## Deployment and Production Considerations

### Configuration Checklist

Before deploying to production:

1. **Credentials**
   - Switch from sandbox to production API endpoint
   - Use production API keys (never commit to repository)
   - Rotate test credentials

2. **Debug Mode**
   - Disable debug logging in production
   - Implement proper error handling (log internally, show generic messages to customers)

3. **Testing**
   - Run full payment flow tests (authorize, capture, void, refund)
   - Test vault functionality
   - Test declined transactions
   - Test 3D Secure flows (if applicable)

4. **Monitoring**
   - Set up alerts for payment failures
   - Monitor transaction success rates
   - Track authorization vs capture ratios
   - Monitor refund/void patterns

5. **Compliance**
   - Complete PCI SAQ (Self-Assessment Questionnaire)
   - Implement security patches promptly
   - Review logs for PCI-prohibited data
   - Ensure HTTPS is enforced

### Error Handling

```php
// Gateway/ErrorMapper.php
class ErrorMapper
{
    private const ERROR_CODES = [
        'card_declined' => 'Your card was declined. Please use a different payment method.',
        'insufficient_funds' => 'Your card has insufficient funds.',
        'expired_card' => 'Your card has expired.',
        'invalid_cvv' => 'The CVV code is invalid.',
        'invalid_expiry' => 'The expiration date is invalid.',
        'processing_error' => 'An error occurred processing your payment. Please try again.',
    ];

    public function getMessageForCode(string $code): string
    {
        return self::ERROR_CODES[$code] ?? self::ERROR_CODES['processing_error'];
    }
}
```

### Performance Optimization

**Timeout Configuration:**

```php
// HTTP client timeout should balance reliability vs performance
'timeout' => 30, // 30 seconds for payment operations
'connect_timeout' => 5, // 5 seconds to establish connection
```

**Caching:**

Never cache payment credentials or transaction data. Do cache:
- Gateway configuration (API endpoints, merchant ID)
- Card type mappings
- Country/currency validators

### Transaction State Management

**Authorization flow:**
1. Customer places order → Order status: `pending`
2. Authorization succeeds → Order status: `processing`, Payment: authorized
3. Invoice created → Payment: captured, Order status: `processing` or `complete`

**Failure handling:**
- Authorization fails → Order canceled, inventory restored
- Capture fails → Order remains authorized, manual intervention required
- Refund fails → Log error, notify merchant, investigate gateway

---

## Assumptions

- **Target versions**: Magento 2.4.7+, PHP 8.2+
- **Payment gateway**: REST API with JSON payloads (adapt for SOAP/XML if needed)
- **Tokenization**: Gateway provides tokenization API (required for PCI compliance)
- **Environment**: Production deployment requires PCI SAQ completion

## Why This Approach

**Gateway architecture**: Separates concerns, enables testing, follows Magento best practices

**Vault integration**: Required for recurring payments, subscriptions, and improved checkout UX

**Capture strategy command**: Handles both sale (authorize+capture) and deferred capture flows

**Mock client**: Enables automated testing without live gateway access

**Sanitized logging**: PCI compliance and debugging balance

## Security Impact

**Critical security requirements:**
- Never store CVV codes (PCI prohibited)
- Use tokenization to avoid storing full card numbers
- Encrypt API credentials in database
- Sanitize all payment logs
- Enforce HTTPS for all payment operations
- Validate all input data (card numbers, CVV, expiration)
- Implement rate limiting on payment endpoints (not shown, requires Magento security extension or WAF)

**Authorization requirements:**
- Payment methods are available to all authenticated customers
- Admin refund/void operations require `Magento_Sales::sales_invoice` and `Magento_Sales::creditmemo` ACL permissions

**CSRF protection:**
- All admin payment operations (capture, refund, void) are protected by Magento's form key mechanism
- Frontend checkout uses CSRF tokens automatically

## Performance Impact

**Cacheability:**
- Payment configuration is cacheable (config cache)
- Payment forms are not cached (contain order-specific data)
- Vault tokens are cached per customer (improves checkout performance)

**Database impact:**
- Each transaction creates records in `sales_payment_transaction`, `sales_order_payment`
- Vault tokens stored in `vault_payment_token`, `vault_payment_token_order_payment_link`
- Properly indexed tables ensure fast transaction lookups

**API latency:**
- Payment authorization: 500-2000ms (external gateway call)
- Capture/void/refund: 500-1500ms (external gateway call)
- Vault token creation: +200ms (database write)
- Use async capture for large merchants (not shown, requires message queue)

**Core Web Vitals impact:**
- Payment form rendering: Minimal (static template)
- Payment submission: Blocks order placement (unavoidable)
- Use loading indicators and prevent double-submission

## Backward Compatibility

**API stability:**
- Gateway command interfaces are stable in Magento 2.4.x
- Vault interfaces are stable in Magento 2.4.x
- Payment method configuration structure is stable

**Database schema:**
- Declarative schema ensures safe upgrades
- No direct SQL queries (uses service contracts)

**Upgrade path:**
- Magento 2.4.7 → 2.4.8: No breaking changes expected
- Magento 2.4.x → 2.4.9: Monitor deprecation notices in gateway interfaces

**Migration considerations:**
- Moving from legacy payment method to gateway architecture requires refactoring
- Vault tokens are not backward compatible with legacy saved cards (re-tokenization required)

## Tests to Add

**Unit tests:**
- Request builders (all commands)
- Response handlers (all operations)
- Validators (response structure, amount matching)
- Error mapper

**Integration tests:**
- Payment method availability
- Authorization flow
- Capture flow (sale and deferred)
- Void operation
- Refund operation (full and partial)
- Vault save and reuse
- Multi-currency support

**Functional tests (MFTF):**
- Complete checkout with credit card
- Saved payment method checkout
- Admin capture invoice
- Admin refund credit memo
- Admin void order
- Declined card handling

**API tests:**
- REST API order placement with payment method
- GraphQL checkout with payment method

**Security tests:**
- CVV not logged or stored
- API credentials encrypted in database
- Payment logs sanitized
- HTTPS enforcement

## Documentation to Update

**Merchant documentation:**
- `README.md`: Installation, configuration, API credential setup
- `CONFIGURATION.md`: Admin settings explanation
- `TROUBLESHOOTING.md`: Common errors and resolutions

**Developer documentation:**
- `ARCHITECTURE.md`: Gateway architecture, command flow, extension points
- `API.md`: Gateway API client adapter interface
- `TESTING.md`: How to run tests, mock gateway usage
- `SECURITY.md`: PCI compliance requirements, security audit checklist

**Admin user guide:**
- How to configure payment method
- How to process refunds
- How to void orders
- Troubleshooting declined transactions

**Code comments:**
- Inline docblocks for all classes (already shown in examples)
- XML configuration comments for complex virtual types

## Related Documentation

### Related Guides

- [Custom Shipping Method Development: Building Flexible Shipping Solutions](custom-shipping-method.md)
- [Service Contracts vs Repositories in Magento 2](../explanations/service-contracts-repositories.md)
- [Security Checklist for Custom Modules](../how-to/security-checklist.md)
- [Comprehensive Testing Strategies for Magento 2](../how-to/testing-strategies.md)

### Related Module Documentation

- [Magento_Checkout Overview](../../modules/checkout/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
- [Magento_Quote Overview](../../modules/quote/README.md)
