---
title: "How to Automate Business Workflows"
description: "Set up automated processes for orders, inventory, customer communications, and more"
type: how-to
audience:
  - "Magento merchants"
  - "Advanced users"
difficulty: Intermediate
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Basic PHP knowledge"
  - "Magento development environment"
  - "Product catalog setup"
tags:
  - how-to
  - inventory
  - customer-management
  - orders
last_updated: "2025-09-26T08:16:58.476Z"
---

# How to Automate Business Workflows

## Overview

Automate key Magento 2 workflows to reduce manual effort, speed order processing, prevent stockouts, and deliver timely customer communications. This guide shows a growing merchant how to configure no-code automations and add a lightweight custom module for advanced tasks, improving fulfillment times and customer experience.

Scope: This guide applies to Magento Open Source and Adobe Commerce 2.4.x. Notes are included where behavior differs (for example, the Content Staging cron group on Adobe Commerce, Cloud-managed cron, and MSI inventory specifics).

Why it matters: Each automation directly impacts KPIs.
- Auto-invoicing accelerates pick/pack, shortening time-to-fulfill and improving cash flow.
- Auto-cancel of stale pending payments frees inventory from limbo, reducing oversell risk and improving conversion.
- Inventory alerts reduce out-of-stock events and lost sales; smart backorders can support pre-sell and cash flow.
- Strong email deliverability raises open/click rates, improving repeat purchase revenue and LTV.

Planning time: Merchant configuration typically takes 20–40 minutes. Optional developer automations (custom module + webhook) typically take 1.5–3 hours including staging and deployment.

## Prerequisites

Before you begin, make sure you have:

- Merchant prerequisites:
  - Admin access with permission to Stores > Configuration
  - Ability to place test orders and access test customer inboxes
  - SMTP/transactional email provider credentials (host, port, username, TLS/SSL)
  - Product catalog set up with at least one test product
- Developer prerequisites (for Steps 5–6):
  - Access to a staging environment and the production deployment process
  - Magento filesystem owner CLI access on staging/production
  - Magento development environment and basic PHP knowledge
  - Backups enabled or a VCS rollback plan for code and configuration

## What You'll Accomplish

By following this guide, you will:

- Automate key business workflows (emails, invoicing, inventory alerts)
- Improve order processing speed and customer experience

## Step-by-Step Instructions

### Step 0: Enable and verify Magento cron

Note: This step is a prerequisite for later automations.

1) In CLI, run: 
- bin/magento cron:install
- Note (Adobe Commerce on cloud): Do not run cron:install. Cron is managed via your .magento.app.yaml. See Adobe Commerce on cloud > Configure cron.
2) Verify the crontab: crontab -l (should include Magento cron entries)
3) Manually trigger to test: bin/magento cron:run --group=default && bin/magento cron:run --group=index. If you use Adobe Commerce with Content Staging, also run: bin/magento cron:run --group=staging.
4) Check var/log/system.log and var/log/cron.log for successful execution.




### Step 1: Configure transactional emails (orders, invoices, shipments)

1) Set sender identities first: Go to Stores > Configuration > General > Store Email Addresses and set names/emails for General Contact and Sales Representative (and others you use).
2) Ensure mail is enabled: Go to Stores > Configuration > Advanced > System > Mail Sending Settings and confirm Disable Email Communications = No.
3) Go to Admin > Stores > Configuration > Sales > Sales Emails.
4) Enable Order, Invoice, Shipment, and Credit Memo emails; select appropriate email templates and sender identities.
5) For customer emails, go to Customers > Customer Configuration > Create New Account Options and set the Welcome email template; verify Forgot Password template under Password Options.
6) Save config and clear cache if prompted. Send test orders to verify.




### Step 2: Set payment actions to drive auto-invoicing where appropriate

1) Go to Stores > Configuration > Sales > Payment Methods.
2) For each online gateway that supports capture on authorization, set Payment Action to Authorize and Capture.
3) For offline methods (e.g., Check/Money Order), leave as Authorize or manual to keep control.
4) Save and test a checkout with a capture-enabled method.

Note: Many gateways create invoices automatically on capture, but this behavior is integration-specific. Confirm your gateway settings and test that an invoice is created and the Invoice Email is sent (Sales > Sales Emails).




### Step 3: Automate inventory alerts and stock behavior

1) Go to Stores > Configuration > Catalog > Inventory.
2) Under Stock Options and Product Stock Options, set Display Out of Stock Products (per your merchandising plan), Backorders policy, and Notify for Quantity Below (global threshold; override per product when needed).
   - Merchant tip: Displaying OOS products with an “Email me when available” option can capture demand and preserve SEO value. Allow backorders only when you have reliable lead times to avoid cancellations and refunds.
3) Ensure cron is enabled so Product Alerts (stock/price) emails send. The Reports > Products > Low Stock report pulls current data when you view it.
4) If your edition doesn’t email low stock alerts, plan to add either: (a) an extension that emails the report, or (b) a small custom cron job to notify your team (see next steps).
5) MSI note (2.4.x): With Multi-Source Inventory enabled, “Low Stock” evaluates Salable Quantity. Per-product thresholds can be set under Catalog > Products > [Product] > Sources and Advanced Inventory. The global “Notify for Quantity Below” remains under Stores > Configuration > Catalog > Inventory > Product Stock Options.




### Step 4: Harden email deliverability

1) Configure a reliable SMTP/transactional email service using a trusted SMTP extension.
   - After installation, go to Stores > Configuration > Advanced > System > SMTP (or the extension’s config) and enter your provider settings (host, port, authentication, TLS/SSL).
   - Send a test order and verify delivery in your provider dashboard.
2) Set proper SPF/DKIM/DMARC for your sender domain to improve inbox placement.
3) Monitor impact: Track bounce rate and open/click rates for transactional emails. Improving deliverability increases reach and can lift repeat purchase revenue and LTV.




### Step 5: Create a custom module: auto-cancel stale Pending Payment orders

In your development environment:

1) Create module skeleton: app/code/Vendor/AutoCancel/

- registration.php:
```php
<?php
use Magento\Framework\Component\ComponentRegistrar;
ComponentRegistrar::register(ComponentRegistrar::MODULE, 'Vendor_AutoCancel', __DIR__);
```

- etc/module.xml:
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
  <module name="Vendor_AutoCancel" setup_version="1.0.0"/>
</config>
```

- etc/crontab.xml:
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:config/etc/crontab.xsd">
  <group id="default">
    <job name="vendor_autocancel_pending" instance="Vendor\AutoCancel\Cron\CancelPending" method="execute" schedule="*/10 * * * *"/>
  </group>
</config>
```

- etc/adminhtml/system.xml (adds a configurable hours threshold):
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system_file.xsd">
  <system>
    <section id="vendor_autocancel" translate="label" sortOrder="900" showInDefault="1" showInWebsite="1" showInStore="1">
      <label>Auto Cancel Settings</label>
      <tab>sales</tab>
      <resource>Vendor_AutoCancel::config</resource>
      <group id="general" translate="label" sortOrder="10" showInDefault="1" showInWebsite="1" showInStore="1">
        <label>General</label>
        <field id="hours" translate="label" type="text" sortOrder="10" showInDefault="1" showInWebsite="1" showInStore="1">
          <label>Cancel after (hours)</label>
          <comment>Orders in pending_payment state older than this will be canceled by cron.</comment>
          <validate>validate-number validate-greater-than-zero</validate>
          <config_path>vendor_autocancel/general/hours</config_path>
        </field>
      </group>
    </section>
  </system>
</config>
```

- Cron/CancelPending.php (uses service contracts, state filter, and logging):
```php
<?php
namespace Vendor\AutoCancel\Cron;

use Magento\Sales\Model\Order;
use Magento\Sales\Model\ResourceModel\Order\CollectionFactory as OrderCollectionFactory;
use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Framework\Stdlib\DateTime\DateTime;
use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Store\Model\ScopeInterface;
use Psr\Log\LoggerInterface;

class CancelPending
{
    private OrderCollectionFactory $orders;
    private OrderRepositoryInterface $orderRepository;
    private DateTime $date;
    private ScopeConfigInterface $scopeConfig;
    private LoggerInterface $logger;

    public function __construct(
        OrderCollectionFactory $orders,
        OrderRepositoryInterface $orderRepository,
        DateTime $date,
        ScopeConfigInterface $scopeConfig,
        LoggerInterface $logger
    ) {
        $this->orders = $orders;
        $this->orderRepository = $orderRepository;
        $this->date = $date;
        $this->scopeConfig = $scopeConfig;
        $this->logger = $logger;
    }

    public function execute(): bool
    {
        try {
            $hours = (int)($this->scopeConfig->getValue('vendor_autocancel/general/hours', ScopeInterface::SCOPE_STORE) ?? 48);
            if ($hours <= 0) {
                $hours = 48;
            }
            $cutoffTs = strtotime("-{$hours} hours", $this->date->gmtTimestamp());
            $cutoff = gmdate('Y-m-d H:i:s', $cutoffTs);

            $collection = $this->orders->create()
                ->addFieldToFilter('state', Order::STATE_PENDING_PAYMENT)
                ->addFieldToFilter('created_at', ['lt' => $cutoff]);

            foreach ($collection as $order) {
                try {
                    if ($order->canCancel()) {
                        $order->cancel();
                        $order->addCommentToStatusHistory('Auto-canceled after timeout.');
                        $this->orderRepository->save($order);
                    }
                } catch (\Throwable $e) {
                    $this->logger->error('Auto-cancel failed', [
                        'order_id' => $order->getId(),
                        'message' => $e->getMessage(),
                    ]);
                }
            }
        } catch (\Throwable $e) {
            $this->logger->error('Auto-cancel cron error: ' . $e->getMessage());
        }
        return true;
    }
}
```

2) Enable and deploy: 
- bin/magento module:enable Vendor_AutoCancel && bin/magento setup:upgrade && bin/magento cache:flush
- If in production mode, also run: bin/magento setup:di:compile
3) Verify cron runs and observe orders transitioning to Canceled after the threshold (test in staging).

Timeout guidance by payment method (choose a value that balances revenue protection with inventory availability):
- Card/real-time gateways (Auth+Capture at checkout): 24–48 hours
- PayPal pending eCheck or similar delayed capture: 48–72 hours
- Bank transfer/offline methods: 5–7 days (or cancel manually)




### Step 6: Optional: push order updates to external systems via webhook

1) Add etc/events.xml in your module:
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
  <event name="sales_order_save_after">
    <observer name="vendor_order_webhook" instance="Vendor\AutoCancel\Observer\OrderWebhook"/>
  </event>
</config>
```

2) Add admin configuration for the webhook (extend etc/adminhtml/system.xml):
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system_file.xsd">
  <system>
    <section id="vendor_autocancel">
      <group id="webhook" translate="label" sortOrder="20" showInDefault="1" showInWebsite="1" showInStore="1">
        <label>Webhook</label>
        <field id="endpoint" translate="label" type="text" sortOrder="10" showInDefault="1" showInWebsite="1" showInStore="1">
          <label>Endpoint URL</label>
          <config_path>vendor_autocancel/webhook/endpoint</config_path>
        </field>
        <field id="secret" translate="label" type="obscure" sortOrder="20" showInDefault="1" showInWebsite="1" showInStore="1">
          <label>Shared Secret</label>
          <config_path>vendor_autocancel/webhook/secret</config_path>
        </field>
        <field id="timeout" translate="label" type="text" sortOrder="30" showInDefault="1" showInWebsite="1" showInStore="1">
          <label>Timeout (seconds)</label>
          <config_path>vendor_autocancel/webhook/timeout</config_path>
          <validate>validate-number validate-greater-than-zero</validate>
        </field>
      </group>
    </section>
  </system>
</config>
```

3) Create Observer/OrderWebhook.php that posts JSON to your automation platform (with status-change guard, timeout, and HMAC signature):
```php
<?php
namespace Vendor\AutoCancel\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\HTTP\Client\Curl;
use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Store\Model\ScopeInterface;
use Psr\Log\LoggerInterface;

class OrderWebhook implements ObserverInterface
{
    private Curl $curl;
    private ScopeConfigInterface $scopeConfig;
    private LoggerInterface $logger;

    public function __construct(Curl $curl, ScopeConfigInterface $scopeConfig, LoggerInterface $logger)
    {
        $this->curl = $curl;
        $this->scopeConfig = $scopeConfig;
        $this->logger = $logger;
    }

    public function execute(Observer $observer)
    {
        $order = $observer->getEvent()->getOrder();

        // Avoid duplicate calls: only act when status changes to complete
        if (!$order->dataHasChangedFor('status') || $order->getStatus() !== 'complete') {
            return;
        }

        $endpoint = (string)($this->scopeConfig->getValue('vendor_autocancel/webhook/endpoint', ScopeInterface::SCOPE_STORE) ?? '');
        $secret = (string)($this->scopeConfig->getValue('vendor_autocancel/webhook/secret', ScopeInterface::SCOPE_STORE) ?? '');
        $timeout = (int)($this->scopeConfig->getValue('vendor_autocancel/webhook/timeout', ScopeInterface::SCOPE_STORE) ?? 5);
        if (!$endpoint) {
            return;
        }

        $payload = json_encode([
            'increment_id' => $order->getIncrementId(),
            'grand_total' => $order->getGrandTotal(),
            'customer_email' => $order->getCustomerEmail(),
            'status' => $order->getStatus(),
        ]);

        try {
            $this->curl->addHeader('Content-Type', 'application/json');
            if ($secret !== '') {
                $sig = hash_hmac('sha256', (string)$payload, $secret);
                $this->curl->addHeader('X-Signature', $sig);
            }
            $this->curl->setTimeout($timeout > 0 ? $timeout : 5);
            $this->curl->post($endpoint, (string)$payload);

            $status = (int)$this->curl->getStatus();
            if ($status < 200 || $status >= 300) {
                $this->logger->error('Order webhook non-2xx response', [
                    'order_id' => $order->getId(),
                    'status' => $status,
                    'body' => $this->curl->getBody(),
                ]);
            }
        } catch (\Throwable $e) {
            $this->logger->error('Order webhook error: ' . $e->getMessage(), [
                'order_id' => $order->getId(),
            ]);
        }
    }
}
```

4) Deploy, then complete an order; confirm the external system receives a single POST with the expected JSON and signature.




### Step 7: Deploy, test, and measure impact

1) Deploy to staging; run functional tests (place test orders, simulate low stock).
2) Deploy to production during low traffic; monitor logs and email deliverability for 24–48 hours.
3) Track KPIs: average time-to-fulfill, cancellation rate, low-stock incidents, email delivery rate and open rate. Iterate thresholds and schedules as needed.
4) Rollback plan: If issues arise, disable the custom module (bin/magento module:disable Vendor_AutoCancel && bin/magento setup:upgrade), revert code via VCS, revert configuration changes under Stores > Configuration, clear cache, and retest.




## Verification

To confirm everything is working correctly:

- Verify Sales Emails: Place a test order; confirm Order, Invoice, and Shipment emails are received. If using custom templates, check Marketing > Email Templates for correct assignment.
- Verify auto-invoicing: Place an order with a capture-enabled payment method. Under Sales > Orders > View > Invoices, confirm an invoice was created and the Invoice Email was sent.
- Verify inventory alerts and report: Lower a product’s salable quantity below your threshold; check Reports > Products > Low Stock shows the product. If using product alerts, subscribe to stock/price alerts and confirm the email is received when triggered by cron.
- Verify auto-cancel: In staging, create a Pending Payment order and temporarily set the cancel threshold to 0.05 hours (~3 minutes). Confirm it moves to Canceled with the history comment “Auto-canceled after timeout.”
- Verify webhook: Complete an order; confirm your endpoint receives a single POST with the expected JSON and the X-Signature header. Ensure no duplicate calls occur on subsequent order edits.

## Common Issues and Solutions

- Emails not sending:
  - Check Stores > Configuration > Advanced > System > Mail Sending Settings (Disable Email Communications = No).
  - Verify SMTP provider credentials in your SMTP extension settings.
  - Review var/log/exception.log for mail errors and your provider dashboard for bounces.
- Cron not running:
  - Ensure the system crontab includes Magento entries (crontab -l) and uses the correct PHP binary path.
  - Review var/log/cron.log for errors. On Adobe Commerce on cloud, manage cron via .magento.app.yaml.
- Orders not auto-invoicing:
  - Confirm the payment method supports automatic capture and that the gateway is configured to create invoices on capture.
  - Test and verify that Invoice Email is sent (Sales > Sales Emails).
- Auto-cancel not working:
  - Verify the cron schedule in etc/crontab.xml and that your cron is running.
  - Ensure the collection filter uses state = pending_payment and that the threshold hours are set > 0.
- Duplicate webhooks:
  - Ensure the observer checks dataHasChangedFor('status') and only fires on the intended status transition (e.g., to complete).
  - Consider idempotency keys on the receiving system.

## Related Resources

- Adobe Commerce and Magento Open Source User Guide: https://experienceleague.adobe.com/docs/commerce.html

---

Need help? Visit Adobe Experience League (https://experienceleague.adobe.com/docs/commerce.html) or see the Common Issues and Solutions section above.
