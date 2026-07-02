---
title: "Email Templates Customization"
description: "Master Magento email template architecture, transactional email customization, SMTP configuration, and queue management for reliable email delivery."
type: "how-to"
tier: 2
difficulty: "intermediate"
version: "2.4.7+"
estimated_time: "45 minutes"
topics:
  - email templates
  - transactional emails
  - smtp
  - email queue
  - template variables
last_updated: "2026-02-07"
---

# Email Templates Customization

## Overview

Magento's email system is built on a robust template engine that supports multi-store configurations, locale-specific content, and transactional email workflows. This guide covers the complete email architecture, from template structure to queue management and SMTP integration.

**What you'll learn:**
- Email template architecture and the template rendering pipeline
- Creating and customizing transactional email templates in modules
- Working with email variables, blocks, and directives
- SMTP configuration and third-party email service integration
- Email queue management and asynchronous sending
- Testing and previewing email templates
- Performance optimization and troubleshooting

**Prerequisites:**
- Magento 2.4.7+ (Adobe Commerce or Open Source)
- Understanding of Magento module structure
- Basic knowledge of XML configuration and PHTML templates
- Familiarity with dependency injection

---

## Email Template Architecture

### Template System Components

Magento's email system consists of several key components:

1. **Template Models**: `Magento\Email\Model\Template` handles template loading and rendering
2. **Transport Builder**: `Magento\Framework\Mail\Template\TransportBuilder` constructs email messages
3. **Email Queue**: Asynchronous email sending can be achieved via the message queue framework (`Magento\Framework\MessageQueue`)
4. **Template Filter**: `Magento\Email\Model\Template\Filter` processes template directives
5. **SMTP Transport**: Configurable transport layer for email delivery

### Email Flow

```
[Trigger Event] → [Observer/Controller]
       ↓
[TransportBuilder] → [Template Loading]
       ↓
[Variable Processing] → [Template Rendering]
       ↓
[Transport Creation] → [Queue (optional)]
       ↓
[SMTP Delivery] → [Customer Inbox]
```

---

## Email Template Structure

### Template File Format

Email templates use HTML with Magento-specific directives and variables.

**File**: `view/frontend/email/custom_notification.html`

```html
<!--@subject {{trans "Order Confirmation for %store_name" store_name=$store.getFrontendName()}} @-->
<!--@vars
{
    "var order": "Magento\\Sales\\Model\\Order",
    "var store": "Magento\\Store\\Model\\Store",
    "var customer": "Magento\\Customer\\Model\\Customer",
    "var shipment": "Magento\\Sales\\Model\\Order\\Shipment|null"
}
@-->
<!--@styles
body {
    background: #f6f6f6;
    font-family: Arial, sans-serif;
}
.email-container {
    max-width: 600px;
    margin: 0 auto;
    background: #ffffff;
}
@-->

{{template config_path="design/email/header_template"}}

<table class="email-container">
    <tr>
        <td class="email-heading">
            <h1>{{trans "Hello %customer_name," customer_name=$order.getCustomerName()}}</h1>
        </td>
    </tr>
    <tr>
        <td class="email-content">
            <p>{{trans "Thank you for your order from %store_name." store_name=$store.getFrontendName()}}</p>

            {{if order.getIsNotVirtual()}}
            <p>{{trans "Your order will be shipped to:"}}</p>
            {{block class="Magento\\Sales\\Block\\Order\\Email\\Address"
                    area="frontend"
                    template="Magento_Sales::email/order/address.phtml"
                    address=$order.getShippingAddress()}}
            {{/if}}

            <h2>{{trans "Order #%increment_id" increment_id=$order.getIncrementId()}}</h2>

            {{block class="Magento\\Sales\\Block\\Order\\Email\\Items"
                    area="frontend"
                    template="Magento_Sales::email/order/items.phtml"
                    order=$order}}

            {{block class="Magento\\Framework\\View\\Element\\Template"
                    area="frontend"
                    template="Magento_Sales::email/order/totals.phtml"
                    order=$order}}
        </td>
    </tr>
</table>

{{template config_path="design/email/footer_template"}}
```

### Template Directives

**Common Directives:**

```html
<!-- Variable output with escaping -->
{{var customer_name}}

<!-- Translation -->
{{trans "Welcome to %store" store=$store_name}}

<!-- Conditional logic -->
{{if order.getShippingMethod()}}
    Shipping: {{var order.getShippingDescription()}}
{{else}}
    No shipping required
{{/if}}

<!-- Loops -->
{{for item in order.getAllItems()}}
    <li>{{var item.getName()}} - {{var item.getQty()}}</li>
{{/for}}

<!-- Include another template -->
{{template config_path="design/email/header_template"}}

<!-- Embed a block -->
{{block class="Magento\\Cms\\Block\\Block" block_id="email_footer"}}

<!-- Dependent variables -->
{{depend customer_id}}
    Customer ID: {{var customer_id}}
{{/depend}}

<!-- Store configuration -->
{{config path="general/store_information/phone"}}

<!-- Media URL -->
{{media url="email/logo.png"}}

<!-- Store URL -->
{{store url=""}}

<!-- Custom variables -->
{{customVar code="my_custom_var"}}
```

---

## Creating Custom Email Templates

### Step 1: Define Email Template in Module

**File**: `Vendor/CustomEmail/etc/email_templates.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Email:etc/email_templates.xsd">

    <!-- Custom notification email -->
    <template id="custom_email_notification_template"
              label="Custom Notification Email"
              file="custom_notification.html"
              type="html"
              module="Vendor_CustomEmail"
              area="frontend"/>

    <!-- Admin notification -->
    <template id="custom_email_admin_notification"
              label="Admin Alert Email"
              file="admin_alert.html"
              type="html"
              module="Vendor_CustomEmail"
              area="adminhtml"/>

    <!-- Text-only template -->
    <template id="custom_email_text_notification"
              label="Text Notification"
              file="text_notification.txt"
              type="text"
              module="Vendor_CustomEmail"
              area="frontend"/>
</config>
```

### Step 2: Create System Configuration

**File**: `Vendor/CustomEmail/etc/adminhtml/system.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system_file.xsd">
    <system>
        <section id="custom_email" translate="label" type="text" sortOrder="300" showInDefault="1" showInWebsite="1" showInStore="1">
            <label>Custom Email Settings</label>
            <tab>general</tab>
            <resource>Vendor_CustomEmail::config</resource>

            <group id="notification" translate="label" type="text" sortOrder="10" showInDefault="1" showInWebsite="1" showInStore="1">
                <label>Notification Settings</label>

                <field id="enabled" translate="label comment" type="select" sortOrder="10" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Enable Notifications</label>
                    <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
                    <config_path>custom_email/notification/enabled</config_path>
                </field>

                <field id="template" translate="label comment" type="select" sortOrder="20" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Email Template</label>
                    <source_model>Magento\Config\Model\Config\Source\Email\Template</source_model>
                    <config_path>custom_email/notification/template</config_path>
                    <comment>Email template for customer notifications</comment>
                </field>

                <field id="sender" translate="label comment" type="select" sortOrder="30" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Email Sender</label>
                    <source_model>Magento\Config\Model\Config\Source\Email\Identity</source_model>
                    <config_path>custom_email/notification/sender</config_path>
                </field>

                <field id="copy_to" translate="label comment" type="text" sortOrder="40" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Send Email Copy To</label>
                    <comment>Comma-separated list of email addresses</comment>
                    <config_path>custom_email/notification/copy_to</config_path>
                    <validate>validate-emails</validate>
                </field>

                <field id="copy_method" translate="label" type="select" sortOrder="50" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Send Email Copy Method</label>
                    <source_model>Magento\Config\Model\Config\Source\Email\Method</source_model>
                    <config_path>custom_email/notification/copy_method</config_path>
                </field>
            </group>
        </section>
    </system>
</config>
```

### Step 3: Create Email Sender Service

**File**: `Vendor/CustomEmail/Model/EmailSender.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\CustomEmail\Model;

use Magento\Framework\App\Area;
use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Framework\Mail\Template\TransportBuilder;
use Magento\Framework\Translate\Inline\StateInterface;
use Magento\Store\Model\ScopeInterface;
use Magento\Store\Model\StoreManagerInterface;
use Psr\Log\LoggerInterface;

class EmailSender
{
    private const XML_PATH_EMAIL_ENABLED = 'custom_email/notification/enabled';
    private const XML_PATH_EMAIL_TEMPLATE = 'custom_email/notification/template';
    private const XML_PATH_EMAIL_SENDER = 'custom_email/notification/sender';
    private const XML_PATH_EMAIL_COPY_TO = 'custom_email/notification/copy_to';
    private const XML_PATH_EMAIL_COPY_METHOD = 'custom_email/notification/copy_method';

    public function __construct(
        private readonly TransportBuilder $transportBuilder,
        private readonly StateInterface $inlineTranslation,
        private readonly ScopeConfigInterface $scopeConfig,
        private readonly StoreManagerInterface $storeManager,
        private readonly LoggerInterface $logger
    ) {
    }

    /**
     * Send custom notification email
     *
     * @param array $emailData
     * @param int|null $storeId
     * @return bool
     */
    public function sendNotificationEmail(array $emailData, ?int $storeId = null): bool
    {
        if (!$this->isEnabled($storeId)) {
            return false;
        }

        try {
            $this->inlineTranslation->suspend();

            $storeId = $storeId ?? (int)$this->storeManager->getStore()->getId();
            $sender = $this->getSender($storeId);
            $template = $this->getTemplate($storeId);

            // Build transport
            $transport = $this->transportBuilder
                ->setTemplateIdentifier($template)
                ->setTemplateOptions([
                    'area' => Area::FRONTEND,
                    'store' => $storeId,
                ])
                ->setTemplateVars($emailData['variables'] ?? [])
                ->setFromByScope($sender, $storeId)
                ->addTo($emailData['recipient_email'], $emailData['recipient_name'] ?? '')
                ->getTransport();

            // Handle BCC/CC copies
            $this->addEmailCopies($storeId);

            // Send email
            $transport->sendMessage();

            $this->inlineTranslation->resume();

            return true;

        } catch (\Exception $e) {
            $this->logger->error('Email sending failed: ' . $e->getMessage(), [
                'exception' => $e,
                'email_data' => $emailData
            ]);
            $this->inlineTranslation->resume();
            return false;
        }
    }

    /**
     * Send email asynchronously via queue
     *
     * @param array $emailData
     * @param int|null $storeId
     * @return bool
     */
    public function sendNotificationEmailAsync(array $emailData, ?int $storeId = null): bool
    {
        if (!$this->isEnabled($storeId)) {
            return false;
        }

        try {
            $storeId = $storeId ?? (int)$this->storeManager->getStore()->getId();
            $sender = $this->getSender($storeId);
            $template = $this->getTemplate($storeId);

            // Build transport but don't send immediately
            $this->transportBuilder
                ->setTemplateIdentifier($template)
                ->setTemplateOptions([
                    'area' => Area::FRONTEND,
                    'store' => $storeId,
                ])
                ->setTemplateVars($emailData['variables'] ?? [])
                ->setFromByScope($sender, $storeId)
                ->addTo($emailData['recipient_email'], $emailData['recipient_name'] ?? '');

            $this->addEmailCopies($storeId);

            // Add to queue instead of sending immediately
            $transport = $this->transportBuilder->getTransport();

            // Queue is handled automatically by Magento's email queue system
            // when using TransportBuilder with proper configuration

            return true;

        } catch (\Exception $e) {
            $this->logger->error('Email queue failed: ' . $e->getMessage(), [
                'exception' => $e,
                'email_data' => $emailData
            ]);
            return false;
        }
    }

    /**
     * Add CC/BCC copies based on configuration
     */
    private function addEmailCopies(int $storeId): void
    {
        $copyTo = $this->getCopyTo($storeId);
        $copyMethod = $this->getCopyMethod($storeId);

        if (!empty($copyTo)) {
            foreach ($copyTo as $email) {
                if ($copyMethod === 'bcc') {
                    $this->transportBuilder->addBcc($email);
                } else {
                    $this->transportBuilder->addCc($email);
                }
            }
        }
    }

    private function isEnabled(?int $storeId = null): bool
    {
        return $this->scopeConfig->isSetFlag(
            self::XML_PATH_EMAIL_ENABLED,
            ScopeInterface::SCOPE_STORE,
            $storeId
        );
    }

    private function getTemplate(?int $storeId = null): string
    {
        return (string)$this->scopeConfig->getValue(
            self::XML_PATH_EMAIL_TEMPLATE,
            ScopeInterface::SCOPE_STORE,
            $storeId
        );
    }

    private function getSender(?int $storeId = null): string
    {
        return (string)$this->scopeConfig->getValue(
            self::XML_PATH_EMAIL_SENDER,
            ScopeInterface::SCOPE_STORE,
            $storeId
        );
    }

    private function getCopyTo(?int $storeId = null): array
    {
        $copyTo = $this->scopeConfig->getValue(
            self::XML_PATH_EMAIL_COPY_TO,
            ScopeInterface::SCOPE_STORE,
            $storeId
        );

        if (empty($copyTo)) {
            return [];
        }

        return array_filter(array_map('trim', explode(',', $copyTo)));
    }

    private function getCopyMethod(?int $storeId = null): string
    {
        return (string)$this->scopeConfig->getValue(
            self::XML_PATH_EMAIL_COPY_METHOD,
            ScopeInterface::SCOPE_STORE,
            $storeId
        );
    }
}
```

---

## Advanced Template Variables

### Creating Custom Variable Providers

**File**: `Vendor/CustomEmail/Model/Template/VariableProvider.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\CustomEmail\Model\Template;

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Store\Model\StoreManagerInterface;

class VariableProvider
{
    public function __construct(
        private readonly CustomerRepositoryInterface $customerRepository,
        private readonly StoreManagerInterface $storeManager,
        private readonly ScopeConfigInterface $scopeConfig
    ) {
    }

    /**
     * Get variables for order confirmation email
     *
     * @param \Magento\Sales\Model\Order $order
     * @return array
     */
    public function getOrderVariables(\Magento\Sales\Model\Order $order): array
    {
        $store = $this->storeManager->getStore($order->getStoreId());

        return [
            'order' => $order,
            'store' => $store,
            'customer' => $this->getCustomerData($order),
            'billing' => $order->getBillingAddress(),
            'shipping' => $order->getShippingAddress(),
            'payment_html' => $this->getPaymentHtml($order),
            'formattedBillingAddress' => $this->formatAddress($order->getBillingAddress()),
            'formattedShippingAddress' => $this->formatAddress($order->getShippingAddress()),
            'store_name' => $store->getFrontendName(),
            'store_phone' => $this->scopeConfig->getValue(
                'general/store_information/phone',
                \Magento\Store\Model\ScopeInterface::SCOPE_STORE,
                $order->getStoreId()
            ),
            'created_at_formatted' => $order->getCreatedAtFormatted(2),
            'items' => $this->formatOrderItems($order),
        ];
    }

    private function getCustomerData(\Magento\Sales\Model\Order $order): ?object
    {
        if (!$order->getCustomerId()) {
            return null;
        }

        try {
            return $this->customerRepository->getById($order->getCustomerId());
        } catch (\Exception $e) {
            return null;
        }
    }

    private function formatAddress($address): string
    {
        if (!$address) {
            return '';
        }

        return $address->format('html');
    }

    private function formatOrderItems(\Magento\Sales\Model\Order $order): array
    {
        $items = [];
        foreach ($order->getAllVisibleItems() as $item) {
            $items[] = [
                'name' => $item->getName(),
                'sku' => $item->getSku(),
                'qty' => (int)$item->getQtyOrdered(),
                'price' => $order->formatPrice($item->getPrice()),
                'row_total' => $order->formatPrice($item->getRowTotal()),
            ];
        }
        return $items;
    }

    private function getPaymentHtml(\Magento\Sales\Model\Order $order): string
    {
        $payment = $order->getPayment();
        return $payment->getMethodInstance()->getTitle();
    }
}
```

---

## SMTP Configuration

### Step 1: Create SMTP Transport Configuration

**File**: `app/etc/env.php` (configuration example)

```php
<?php
return [
    // ... other configuration

    'system' => [
        'default' => [
            'system' => [
                'smtp' => [
                    'disable' => '0',
                    'host' => 'smtp.example.com',
                    'port' => '587',
                    'username' => 'user@example.com',
                    'password' => 'encrypted_password',
                    'auth' => 'login', // 'login', 'plain', 'crammd5'
                    'ssl' => 'tls', // 'tls', 'ssl', or empty
                ]
            ]
        ]
    ]
];
```

### Step 2: Custom SMTP Transport Module

**File**: `Vendor/SmtpConfig/etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Override default mail transport -->
    <type name="Magento\Framework\Mail\TransportInterface">
        <plugin name="smtp_transport_plugin"
                type="Vendor\SmtpConfig\Plugin\Mail\TransportExtend"
                sortOrder="10"/>
    </type>

    <!-- SMTP configuration provider -->
    <type name="Vendor\SmtpConfig\Model\Config">
        <arguments>
            <argument name="scopeConfig" xsi:type="object">Magento\Framework\App\Config\ScopeConfigInterface</argument>
        </arguments>
    </type>
</config>
```

**File**: `Vendor/SmtpConfig/Plugin/Mail/TransportExtend.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\SmtpConfig\Plugin\Mail;

use Magento\Framework\Mail\TransportInterface;
use Vendor\SmtpConfig\Model\Config;
use Laminas\Mail\Transport\Smtp;
use Laminas\Mail\Transport\SmtpOptions;
use Psr\Log\LoggerInterface;

class TransportExtend
{
    public function __construct(
        private readonly Config $smtpConfig,
        private readonly LoggerInterface $logger
    ) {
    }

    /**
     * Replace default transport with SMTP transport
     */
    public function aroundSendMessage(
        TransportInterface $subject,
        callable $proceed
    ): void {
        if (!$this->smtpConfig->isEnabled()) {
            $proceed();
            return;
        }

        try {
            $message = $subject->getMessage();

            $transport = new Smtp();
            $transport->setOptions(new SmtpOptions([
                'host' => $this->smtpConfig->getHost(),
                'port' => $this->smtpConfig->getPort(),
                'connection_class' => $this->smtpConfig->getAuth(),
                'connection_config' => [
                    'username' => $this->smtpConfig->getUsername(),
                    'password' => $this->smtpConfig->getPassword(),
                    'ssl' => $this->smtpConfig->getSsl(),
                ],
            ]));

            $transport->send($message);

        } catch (\Exception $e) {
            $this->logger->error('SMTP transport failed: ' . $e->getMessage(), [
                'exception' => $e
            ]);
            throw $e;
        }
    }
}
```

---

## Email Queue Management

### Enabling Asynchronous Email Sending

**File**: `app/etc/env.php`

```php
<?php
return [
    // ... other configuration

    'queue' => [
        'consumers_wait_for_messages' => 1,
    ],

    'cron_consumers_runner' => [
        'cron_run' => true,
        'max_messages' => 1000,
        'consumers' => [
            'emailSender'
        ]
    ]
];
```

### Custom Queue Consumer

**File**: `Vendor/CustomEmail/etc/queue_consumer.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework-message-queue:etc/consumer.xsd">
    <consumer name="custom.email.sender"
              queue="custom.email.queue"
              connection="amqp"
              handler="Vendor\CustomEmail\Model\Queue\EmailHandler::process"
              consumerInstance="Magento\Framework\MessageQueue\Consumer"
              maxMessages="100"/>
</config>
```

**File**: `Vendor/CustomEmail/etc/queue_topology.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework-message-queue:etc/topology.xsd">
    <exchange name="custom.email" type="topic" connection="amqp">
        <binding id="customEmailBinding" topic="custom.email.send" destinationType="queue" destination="custom.email.queue"/>
    </exchange>
</config>
```

### Monitoring Email Queue

```bash
# Check queue status
bin/magento queue:consumers:list

# View pending messages
mysql -e "SELECT * FROM queue_message WHERE status = 'new' ORDER BY created_at DESC LIMIT 10;"

# Start consumer manually
bin/magento queue:consumers:start emailSender --max-messages=100

# Clear failed messages (careful!)
mysql -e "DELETE FROM queue_message WHERE status = 'error' AND updated_at < DATE_SUB(NOW(), INTERVAL 7 DAY);"
```

---

## Testing and Debugging

### Email Preview in Admin

**Create Preview Controller:**

**File**: `Vendor/CustomEmail/Controller/Adminhtml/Email/Preview.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\CustomEmail\Controller\Adminhtml\Email;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Email\Model\TemplateFactory;
use Magento\Framework\App\Area;
use Magento\Framework\Controller\ResultInterface;
use Magento\Framework\View\Result\PageFactory;

class Preview extends Action
{
    public const ADMIN_RESOURCE = 'Vendor_CustomEmail::email_preview';

    public function __construct(
        Context $context,
        private readonly PageFactory $resultPageFactory,
        private readonly TemplateFactory $templateFactory
    ) {
        parent::__construct($context);
    }

    public function execute(): ResultInterface
    {
        $templateId = (int)$this->getRequest()->getParam('id');

        $template = $this->templateFactory->create();
        $template->load($templateId);

        if (!$template->getId()) {
            $this->messageManager->addErrorMessage(__('Template not found.'));
            return $this->resultRedirectFactory->create()->setPath('*/*/');
        }

        // Set preview variables
        $variables = $this->getPreviewVariables();

        $template->setVars($variables);
        $processedTemplate = $template->processTemplate();

        $resultPage = $this->resultPageFactory->create();
        $resultPage->getConfig()->getTitle()->prepend(__('Email Preview'));

        $block = $resultPage->getLayout()->createBlock(
            \Magento\Framework\View\Element\Template::class
        );
        $block->setTemplate('Vendor_CustomEmail::email/preview.phtml');
        $block->setData('email_content', $processedTemplate);

        return $resultPage;
    }

    private function getPreviewVariables(): array
    {
        return [
            'customer_name' => 'John Doe',
            'store_name' => 'Demo Store',
            'order_id' => '000000123',
            // Add more preview data
        ];
    }
}
```

### CLI Test Command

**File**: `Vendor/CustomEmail/Console/Command/TestEmail.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\CustomEmail\Console\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Vendor\CustomEmail\Model\EmailSender;

class TestEmail extends Command
{
    private const ARGUMENT_EMAIL = 'email';
    private const OPTION_STORE = 'store';

    public function __construct(
        private readonly EmailSender $emailSender,
        ?string $name = null
    ) {
        parent::__construct($name);
    }

    protected function configure(): void
    {
        $this->setName('custom:email:test')
            ->setDescription('Send test email')
            ->addArgument(
                self::ARGUMENT_EMAIL,
                InputArgument::REQUIRED,
                'Recipient email address'
            )
            ->addOption(
                self::OPTION_STORE,
                's',
                InputOption::VALUE_OPTIONAL,
                'Store ID',
                null
            );
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $email = $input->getArgument(self::ARGUMENT_EMAIL);
        $storeId = $input->getOption(self::OPTION_STORE)
            ? (int)$input->getOption(self::OPTION_STORE)
            : null;

        $output->writeln('<info>Sending test email...</info>');

        $emailData = [
            'recipient_email' => $email,
            'recipient_name' => 'Test Recipient',
            'variables' => [
                'customer_name' => 'Test Customer',
                'order_id' => '000000999',
                'store_name' => 'Test Store',
            ]
        ];

        $result = $this->emailSender->sendNotificationEmail($emailData, $storeId);

        if ($result) {
            $output->writeln('<info>Email sent successfully!</info>');
            return Command::SUCCESS;
        }

        $output->writeln('<error>Email sending failed. Check logs.</error>');
        return Command::FAILURE;
    }
}
```

**Register command:**

**File**: `Vendor/CustomEmail/etc/di.xml`

```xml
<type name="Magento\Framework\Console\CommandList">
    <arguments>
        <argument name="commands" xsi:type="array">
            <item name="custom_email_test" xsi:type="object">Vendor\CustomEmail\Console\Command\TestEmail</item>
        </argument>
    </arguments>
</type>
```

---

## Performance Optimization

### Email Template Caching

```php
<?php
// Cache processed templates
$cacheKey = 'email_template_' . $templateId . '_' . $storeId;
$cachedContent = $cache->load($cacheKey);

if ($cachedContent === false) {
    $cachedContent = $template->processTemplate();
    $cache->save($cachedContent, $cacheKey, ['email_template'], 3600);
}
```

### Batch Email Sending

```php
<?php
declare(strict_types=1);

namespace Vendor\CustomEmail\Model;

use Magento\Framework\Mail\Template\TransportBuilder;

class BatchEmailSender
{
    private array $emailQueue = [];
    private const BATCH_SIZE = 50;

    public function __construct(
        private readonly TransportBuilder $transportBuilder
    ) {
    }

    public function addToQueue(array $emailData): void
    {
        $this->emailQueue[] = $emailData;

        if (count($this->emailQueue) >= self::BATCH_SIZE) {
            $this->flush();
        }
    }

    public function flush(): void
    {
        foreach ($this->emailQueue as $emailData) {
            try {
                // Send email
                $this->sendSingleEmail($emailData);
            } catch (\Exception $e) {
                // Log error but continue with batch
                continue;
            }
        }

        $this->emailQueue = [];
    }

    private function sendSingleEmail(array $emailData): void
    {
        // Implementation
    }
}
```

---

## Security Considerations

### Input Sanitization

```php
<?php
// Always escape variables in templates
{{var customer_name|escape}}

// For URLs
{{var redirect_url|escape:'url'}}

// For HTML attributes
<a href="{{var link|escape:'htmlAttr'}}">Link</a>
```

### Preventing Email Injection

```php
<?php
declare(strict_types=1);

namespace Vendor\CustomEmail\Model;

class EmailValidator
{
    /**
     * Validate email address to prevent header injection
     */
    public function validateEmail(string $email): bool
    {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            return false;
        }

        // Check for newline characters that could inject headers
        if (preg_match('/[\r\n]/', $email)) {
            return false;
        }

        return true;
    }

    /**
     * Sanitize email subject
     */
    public function sanitizeSubject(string $subject): string
    {
        // Remove any newline or carriage return characters
        return str_replace(["\r", "\n"], '', $subject);
    }
}
```

---

## Troubleshooting

### Common Issues

**1. Emails not sending:**
```bash
# Check mail.log
tail -f var/log/mail.log

# Check exception.log
tail -f var/log/exception.log

# Test SMTP connection
telnet smtp.example.com 587

# Check queue consumer
bin/magento queue:consumers:list
ps aux | grep emailSender
```

**2. Template variables not rendering:**
```php
// Ensure variables are properly set
$transport->setTemplateVars([
    'var customer_name' => 'John', // WRONG
    'customer_name' => 'John',     // CORRECT
]);

// Check template syntax
{{var customer_name}}  // CORRECT
{{customer_name}}      // WRONG
```

**3. Email queue stuck:**
```bash
# Check queue table
mysql -e "SELECT COUNT(*), status FROM queue_message GROUP BY status;"

# Restart consumer
supervisorctl restart magento_queue_emailSender

# Or manually
bin/magento queue:consumers:start emailSender
```

---

## Summary

**Key Takeaways:**
- Email templates use HTML with Magento directives (trans, var, block, if, for)
- TransportBuilder is the primary interface for constructing emails
- Always use scope-aware configuration for multi-store setups
- Leverage email queue for high-volume sending
- SMTP can be configured via plugins without modifying core
- Test emails using CLI commands and preview functionality
- Sanitize all inputs to prevent email injection attacks
- Monitor queue consumers and mail logs for debugging

---

## Assumptions

- **Magento Version:** 2.4.7+ (Adobe Commerce or Open Source)
- **PHP Version:** 8.2+
- **Email Transport:** Default Sendmail or custom SMTP
- **Queue Backend:** Database or RabbitMQ for async sending
- **Environment:** Multi-store configuration with scope-aware settings

## Why This Approach

- **Service-Oriented:** EmailSender encapsulates all email logic with clean dependencies
- **Configuration-Driven:** Templates and settings are configurable per store scope
- **Queue Support:** Async sending prevents blocking operations during checkout/order flow
- **Testability:** CLI commands and preview functionality enable easy testing
- **Security:** Input validation and escaping prevent injection attacks
- **Performance:** Batch sending and template caching reduce overhead

## Security Impact

- **CSRF Protection:** Email forms require form keys when user-triggered
- **XSS Prevention:** All template variables must use escape filters
- **Email Injection:** Validate email addresses and sanitize subjects
- **PII Handling:** Email content may contain customer data; ensure GDPR compliance
- **Secrets Management:** SMTP credentials stored in env.php (encrypted recommended)
- **Logging:** Avoid logging email content or recipient addresses

## Performance Impact

- **FPC:** Email sending bypasses full page cache (backend operation)
- **Queue:** Async sending prevents blocking checkout/order placement
- **Database:** Queue table grows with pending messages; monitor and purge old records
- **SMTP:** External SMTP may add latency; use connection pooling
- **Template Rendering:** Cache processed templates when possible

## Backward Compatibility

- **API Stability:** TransportBuilder and email template APIs are stable
- **Template Format:** Email template format unchanged since Magento 2.3
- **Queue System:** Message queue interface is BC-compliant
- **Configuration Paths:** Email configuration paths are stable
- **Upgrade Path:** Custom email modules upgrade cleanly to Magento 2.4.8+

## Tests to Add

**Unit Tests:**
```php
// Test email sender
testSendNotificationEmailSuccess()
testSendNotificationEmailDisabled()
testEmailVariableProvider()

// Test validators
testEmailValidation()
testSubjectSanitization()
```

**Integration Tests:**
```php
// Test email sending
testEmailTransportBuildsCorrectly()
testEmailQueueProcessing()
testEmailTemplateRendering()
```

**Functional Tests (MFTF):**
```xml
<!-- Test email configuration -->
<test name="AdminConfigureCustomEmailTest">
<test name="AdminPreviewEmailTemplateTest">
```

## Docs to Update

- **README.md:** Installation, configuration, and usage examples
- **CHANGELOG.md:** Version history and breaking changes
- **docs/CONFIGURATION.md:** System.xml field descriptions and SMTP setup
- **docs/TEMPLATES.md:** Template directive reference and variable guide
- **Admin User Guide:** Screenshots of email configuration panel and preview feature

## Related Documentation

### Related Guides

- [Cron Jobs Implementation: Building Reliable Scheduled Tasks in Magento 2](cron-jobs.md)
- [Plugin System Deep Dive: Mastering Magento 2 Interception](../tutorials/plugin-system-deep-dive.md)

### Related Module Documentation

- [Magento_Customer Overview](../../modules/customer/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
