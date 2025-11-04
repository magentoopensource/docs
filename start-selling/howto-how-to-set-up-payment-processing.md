---
title: "How to Set Up Payment Processing"
description: "Configure payment gateways and secure payment processing for customer transactions"
type: how-to
audience:
  - "Magento merchants"
  - "Advanced users"
difficulty: Advanced
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Basic PHP knowledge"
  - "Magento development environment"
  - "Payment provider account"
  - "SSL certificate"
tags:
  - how-to
  - customer-management
  - payment
  - high-impact
  - complex
last_updated: "2025-09-12T09:44:08.379Z"
---

# How to Set Up Payment Processing

## Overview

Set up secure, reliable payment processing in Magento 2 to accept cards and digital wallets, reduce cart abandonment,
and unlock immediate revenue. A well-optimized payment mix typically increases checkout conversion by 2–5% and reduces
payment-related support tickets by 20–40%.

PCI considerations: To reduce PCI scope, prefer redirect or hosted fields integrations provided by your gateway. Avoid
collecting card data on your server. Confirm which SAQ (A or A-EP) applies to your chosen gateway integration and
complete it with your acquirer.

## Prerequisites

Before you begin, make sure you have:

- Merchant prerequisites:
    - Admin access to Magento (Access to Magento Admin Panel)
    - Payment provider account(s)
    - Valid SSL certificate
- Technical prerequisites (if installing extensions):
    - Staging site with SSH/command-line access
    - Composer installed
    - Backup of code and database before installing extensions
    - Magento cron configured and running (required by many payment extensions)

## What You'll Accomplish

By following this guide, you will:

- Successfully set up payment processing in Magento 2
- Improve your store's checkout performance and customer experience

## Step-by-Step Instructions

### Step 0: Enable HTTPS across storefront and Admin

In Admin, go to Stores > Configuration > General > Web (Scope: Website).
Under Base URLs (Secure):

1) Set Base URL (Secure) to https://<your-domain>/
2) Set Use Secure URLs on Storefront = Yes
3) Set Use Secure URLs in Admin = Yes
4) (Optional) Enable HTTP Strict Transport Security (HSTS) = Yes
   Click Save Config, then go to System > Cache Management > Flush Magento Cache. Ensure your SSL certificate is valid
   and your web server forces HTTPS.

### Step 1: Decide your payment mix and payment action

Choose at least one wallet (e.g., PayPal) and one direct card processor (e.g., Stripe, Adyen, Braintree, Authorize.Net).
Evaluate providers on:

- Regional coverage and local payment methods
- Fees and settlement currency support
- Approval/authorization rates and risk tools (AVS/CVV/3DS2)
- Features (vault/saved cards, subscriptions, Apple Pay/Google Pay)

Choose Payment Action:

- Authorize and Capture: Best for same-day shipping or digital goods where you fulfill immediately.
- Authorize Only: Best if you ship later, have backorders, or inventory variability. Note that authorizations expire (
  typically ~7 days for PayPal and 7–30 days for cards depending on issuer). Ensure capture occurs before expiry.

Note any regional requirements (e.g., SCA/3DS2 in the EEA) and enable 3DS where required.

### Step 2: Gather sandbox credentials from providers

Create or log in to your provider accounts and generate sandbox/test API credentials. For PayPal, create sandbox
business and buyer accounts and note API Username/Password/Signature (Classic API credentials required by Magento's core
PayPal integration). For card gateways, collect Publishable/Secret keys or Merchant ID/Key pairs. Keep webhook signing
secret keys handy if applicable.

### Step 3: Enable and configure PayPal in Magento

In Admin, go to Stores > Configuration > Sales > Payment Methods. Expand the PayPal section and select the solution you
need. Under PayPal, select PayPal Express Checkout (core integration) or other PayPal methods as needed. Set Enable to
Yes. Set Sandbox Mode to Yes. Enter your PayPal sandbox API credentials. Configure Payment Action and Payment from
Applicable Countries (and Payment from Specific Countries if applicable). Save and flush cache.

### Step 4: Install your card gateway extension (Composer)

On your staging server, run Composer to install the provider’s Magento 2 extension (check the provider’s docs for the
exact package name).

Example commands:

- composer require adyen/module-payment
- composer require stripe/stripe-payments

Then enable and register the module:

- bin/magento module:enable Adyen_Payment && bin/magento setup:upgrade

In production mode, also run:

- bin/magento setup:di:compile && bin/magento setup:static-content:deploy -f

Finally:

- bin/magento cache:flush

Note: Magento 2.4.0+ removed several bundled payment methods (e.g., Braintree, Authorize.Net, eWay, CyberSource,
Worldpay). Install the official Marketplace extension from your provider and verify version compatibility with your
Magento version. Always install and test on staging before deploying to production.

### Step 5: Configure the card gateway settings

Go to Stores > Configuration > Sales > Payment Methods and open your installed gateway. Note: Payment Methods are
configured at Website scope—use the scope selector (top-left) to choose the correct Website before entering keys.

Enable it and set Mode to Sandbox/Test. Enter API keys (publishable/secret or merchant credentials). Configure Payment
Action, Currencies (if available in the gateway)—otherwise ensure Store > Configuration > General > Currency Setup
matches your gateway’s settlement currency, Payment from Applicable Countries, 3D Secure/SCA options, and Vault/saved
cards if needed.

If your gateway supports saved cards, enable Magento Vault at Stores > Configuration > Sales > Payment Methods > Stored
Payment Methods (Vault), and enable "Enable Vault" in your gateway’s settings. Save and flush cache.

### Step 6: Set up webhooks/IPN in the provider

#### PayPal (IPN)

For core PayPal methods, configure IPN (not Webhooks) in your PayPal Business account. Go
to https://www.paypal.com/businessmanage/preferences/notifications and set the IPN Notification URL to:

- https://<your-domain>/paypal/ipn/

Note: The core PayPal integration in Magento does not display this IPN URL in Admin. Ensure your site uses HTTPS and
test IPN delivery from the PayPal console.

#### Other gateways (Webhooks)

For third-party gateways, use the webhook endpoint displayed in the extension settings in Magento. Subscribe to events
for payment authorized, captured, refunded, and failed. If a signing secret is provided, paste it into the Magento
gateway settings. Test delivery from the provider console.
Examples:

- Adyen: Use the endpoint shown under Stores > Configuration > Sales > Payment Methods > Adyen > Webhooks (
  Notifications)
- Stripe: Use the endpoint shown under Stores > Configuration > Sales > Payment Methods > Stripe > Webhooks (e.g.,
  https://<your-domain>/stripe/webhook)

### Step 7: Test end-to-end checkout in sandbox

On your storefront, place two orders: one using PayPal sandbox buyer and one using a test card. Use your gateway’s test
card matrix (include 3DS-required cards) and follow their instructions for triggering authorization, capture, and
failure scenarios. Example (generic): Visa 4111111111111111 with a future expiry and any 3-digit CVV.

Confirm taxes/shipping are applied. Complete payment. In Admin, verify order status and invoices/transactions. Issue a
test refund from Magento and confirm it appears in the provider.

Expected results:

- Capture flow: Order state Processing, Invoice auto-created, Transaction type: capture.
- Authorize flow: Order state Processing or Payment Review (gateway dependent), Invoice not created until capture,
  Transaction type: authorization. Verify the Transaction is visible under Sales > Transactions.

### Step 8: Switch to production (go live)

Payment Methods are configured per Website scope. Switch the Configuration scope selector to each Website and enter the
correct live credentials for each site.

When ready to go live:

- In each payment method, change Mode to Production/Live and replace sandbox keys with live credentials.
- Verify HTTPS and the production domain are fully configured.
- If enabling wallets like Apple Pay/Google Pay, complete any required domain verification steps in your gateway before
  enabling in production.
- Click Save and flush cache. Optionally place a low-value live test order.
- Update Payment Failed Emails (Stores > Configuration > Sales > Checkout) to notify your team.
- Document and securely store keys. Consider a short maintenance/change window to monitor immediately after cutover.

### Step 9: Monitor, optimize, and scale

Enable and review gateway debug logs (var/log/system.log, var/log/debug.log, and any module-specific logs like
var/log/stripe*.log or var/log/adyen/*.log). Monitor conversion, decline rates, and error logs daily in the first week.

KPI benchmarks and levers:

- Authorization rate > 90%: Improve by enabling 3DS exemptions where eligible, retrying soft declines, and routing
  high-risk orders through 3DS.
- 3DS friction rate < 10% for trusted traffic: Tune risk rules and exemptions.
- Chargeback rate < 0.9%: Use AVS/CVV rules and post-authorization fraud screening.

Reorder payment methods via Sort Order so top performers appear first. Add wallets (Apple Pay/Google Pay) through your
gateway for mobile conversion—many require domain verification prior to enablement. Schedule periodic webhook/IPN tests
and key rotations.

## Verification

Use this checklist to confirm everything is working correctly:

1. Place sandbox orders (one PayPal, one card) and complete checkout.
2. Open Sales > Orders and confirm the expected state and invoice creation based on Payment Action (see Step 7 Expected
   results).
3. Open Sales > Transactions and confirm authorization/capture entries are present for each order.
4. Issue a partial and a full refund from Sales > Credit Memos; confirm the refunds appear in the provider dashboard
   within 5 minutes.
5. From the provider dashboard, re-deliver a recent webhook/IPN event to confirm Magento receives it (HTTP 200) and
   updates the order accordingly.

## Common Issues and Solutions

- Payments fail with "Unable to place order": Enable gateway Debug/Logging and check var/log/*.log; verify API keys
  match the correct mode (Sandbox vs Live).
- Orders not updating after payment: Verify webhook/IPN URL and signing secret; confirm HTTPS and 200 responses; check
  that Magento cron is running.
- 3DS challenge not shown: Ensure 3DS is enabled in gateway config and test with 3DS-required test cards.
- Currency mismatch: Ensure Stores > Configuration > General > Currency Setup matches gateway-enabled currencies.
- Duplicate captures: Ensure only one system performs capture (Magento or provider), not both.