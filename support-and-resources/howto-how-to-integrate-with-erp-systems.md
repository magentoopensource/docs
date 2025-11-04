---
title: "How to Integrate with ERP Systems"
description: "Connect Magento with enterprise resource planning systems for unified operations"
type: how-to
audience:
  - "Magento merchants"
difficulty: Intermediate
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
tags:
  - how-to
last_updated: "2025-09-17T09:34:06.693Z"
---

# How to Integrate with ERP Systems

## Overview

Integrate Magento with your ERP to keep inventory accurate, automate order flow, and speed fulfillment—reducing
stockouts and errors while improving customer experience and operational efficiency.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Magento version 2.3+ (note: MSI steps require 2.3+)
- Staging/sandbox environments for both Magento and ERP (do not test in production)
- Admin user with permission to create/activate Integrations
- Server CLI access to run bin/magento and manage cron
- ERP/iPaaS credentials and endpoint URLs
- Backup/rollback plan and a recent database/filesystem backup
- Test data (sample products, customers, orders)
- Network allowlists/IPs if your ERP/iPaaS requires them

## What You'll Accomplish

By following this guide, you will:

- Set up a secure, least-privilege integration between Magento and your ERP
- Improve your store's performance and customer experience

## Step-by-Step Instructions

### Step 0: Define scope, data entities, and SLAs

List the entities to sync (products, inventory, prices, customers, orders, shipments). For each, define direction (
ERP→Magento or Magento→ERP), frequency (e.g., inventory every 10 minutes), and SLA (e.g., orders to ERP within 5
minutes). Capture KPIs: order latency, inventory accuracy, error rate.

### Step 1: Choose your integration approach

Always integrate and validate in a staging environment first. Do not connect your ERP to production until end-to-end
tests pass and a rollback plan is in place.

Evaluate: 1) Prebuilt connector (Magento Marketplace or ERP vendor), 2) iPaaS (middleware), 3) Direct API (custom).
Prefer connectors for common ERPs to reduce time-to-value; choose iPaaS for multi-system orchestration. Confirm
availability of a staging environment.

Decision criteria and trade-offs:

- Prebuilt connector: fastest time-to-value, predictable cost, limited customization. Best for common ERPs (SAP Business
  One, NetSuite, Microsoft Dynamics) and standard flows.
- iPaaS: good for multi-system orchestration (ERP + WMS + CRM), visual mapping, error handling; higher monthly cost.
- Direct API (custom): maximum control, highest upfront/ongoing engineering cost; choose only if unique requirements
  exist or no reliable connector is available.

Business impact guidance:

- Inventory should be near real-time to reduce oversell risk and cancellations.
- Catalog and media can be batched to reduce indexing load and hosting costs.

### Step 2: Create a Magento API Integration with least-privilege access

In Magento Admin, go to System > Extensions > Integrations > Add New Integration. Enter a name (e.g., ERP Connector),
contact email, and callback/identity URLs if your connector requires OAuth callbacks.

In the API tab, set Resource Access to Custom and grant only needed scopes. Common scopes for these flows:

- Catalog > Products
- Catalog > Inventory (Stock)
- Sales > Operations > Orders, Invoices, Shipments
- Customers > All Customers
- If on Magento 2.3+ with MSI, also include Inventory (MSI) resources (e.g., Source Items, Stocks). Labels can vary by
  version/module; select the least-privilege options that cover your flows.

Save. Click Activate, then Allow to generate keys. Copy Consumer Key, Consumer Secret, Access Token, and Access Token
Secret.

Security best practices:

- Use Integrations with least-privilege scopes instead of admin tokens.
- Rotate Integration tokens/keys at least quarterly; revoke unused Integrations.
- Restrict callback and identity URLs to trusted domains; enable IP allowlisting on ERP/iPaaS if supported.
- Store credentials in a secrets manager; never hard-code keys in code repositories.

### Step 3: Configure ERP or middleware with Magento credentials

In your ERP connector, add Magento as a connection using your store base URL (https) and the OAuth credentials. Match
Magento settings: Stores > Settings > Configuration > General > General > Locale Options (Timezone) and General >
Currency Setup (Base/Default/Display Currency). Verify connection with a test call (e.g., GET /V1/store/storeViews).

### Step 4: Map and schedule ERP→Magento product, price, and inventory syncs

Map fields: SKU (required), name, description, price, tax class, visibility, status, categories, and images. For
inventory, map source/stock (MSI) and quantities. Enable delta updates when supported.

Product modeling and attributes:

- Product types: Define how configurables map (parent product + child simples with attribute options), bundles, and
  grouped products.
- Attribute sets: Ensure required attributes exist in Magento with matching attribute codes and correct input types.
- Media: Provide base, small, thumbnail images and swatch images; confirm external URL vs binary upload method supported
  by your connector.
- Price scope: Verify Stores > Settings > Configuration > Catalog > Catalog > Price > Catalog Price Scope (Global vs
  Website) aligns with ERP price lists.

Set schedules:

- Catalog 1–2 times/day (or on change)
- Prices hourly if needed
- Inventory every 5–10 minutes

Why it matters: Near-real-time inventory minimizes oversell risk and cancellations, protecting margin and customer
lifetime value. Catalog and media updates can be batched to reduce indexing load and hosting costs.

### Step 5: Map and schedule Magento→ERP order and customer exports

Configure trigger and idempotency:

- Trigger: Export on order placement (state = new) or when payment is captured (status = Processing). Document which
  event applies per payment method (offline vs online capture) to avoid misses/duplicates.
- Idempotency: Use increment_id as the unique key; reject duplicates in ERP.

Map order data:

- Order fields: increment_id, items with SKU/qty, totals, tax, payment, shipping, discounts
- Taxes/discounts: Include tax_amount, discount_amount, shipping_discount_amount, and coupon_code
- Shipping methods: Map Magento method codes (e.g., flatrate_flatrate, freeshipping_freeshipping) to ERP
  carriers/service levels
- Payment methods: Map Magento codes (e.g., checkmo, braintree, paypal_express) to ERP payment terms

Export customers as needed to maintain CRM data.

### Step 6: Enable ERP→Magento fulfillment (shipment/tracking) and invoicing

Configure ERP to send shipment confirmations back to Magento, including carrier and tracking numbers. If invoicing is
ERP-driven, map invoice creation to Magento. Ensure the Integration has Sales permissions for shipments and invoices.

Operational considerations:

- Email notifications: If you want Magento to send shipment/invoice emails, enable in Stores > Settings >
  Configuration > Sales > Sales Emails.
- Partial shipments: Confirm your ERP can post multiple shipments per order with per-item quantities and tracking
  numbers.
- Credit memos/returns: If ERP manages returns, map Magento credit memo creation back from ERP.

### Step 7: Prepare Magento for automation (cron, indexers, inventory)

In Magento Admin, go to System > Tools > Index Management and set relevant indexers to Update by Schedule.

Cron setup and verification:

- From the server, run bin/magento cron:install (if not installed), then bin/magento cron:run.
- Verify three Magento cron entries exist via crontab -l (two for Magento and one for update/cron log depending on
  version).
- Check recent execution in var/log/cron.log (if enabled).

MSI considerations:

- Single vs Multi-Source: If single-source, update the Default Source. If multi-source, specify the selling Source(s)
  your ERP updates and ensure Source-to-Stock assignments reflect sales channels (Stores > Inventory).
- Reservations: Magento places inventory reservations on order placement. Ensure your ERP integration decrements source
  quantities and confirm reservations are cleared upon shipment/invoice.

### Step 8: Run an end-to-end sandbox test

Create a test product in ERP; allow it to sync to Magento. In Magento, place a test order for that product. In ERP,
process fulfillment and send shipment/tracking back. Confirm Magento shows shipment, tracking, and correct stock
decrement. Use a unique marker (e.g., test coupon) to trace the order.

### Step 9: Go live, monitor, and tune

Enable production schedules. Set alerts in your connector for failed runs. Monitor KPIs (order latency, shipment
latency, inventory accuracy, error rate) for 72 hours.

Targets and actions:

- KPIs: Order export latency < 5 minutes (P95); Inventory sync latency < 10 minutes (P95); Error rate < 1% per 1,000
  records; Oversell incidents = 0.
- Alerts: Notify on failed runs, 5xx errors > 3 in 10 minutes, or queue backlog > 2x normal.
- Monitoring: Use connector dashboards, Magento logs (var/log), and ERP job monitors. Schedule daily spot checks on a
  random set of orders. Tune frequencies, payload sizes, and retries/backoff as needed.

## Verification

Use this checklist to confirm end-to-end data integrity:

1. API connectivity: Call GET /V1/store/storeViews and GET /V1/products?searchCriteria[currentPage]=1 using your
   connector; both should return 200 responses.
2. Product sync: Verify product appears in Catalog > Products with correct attributes, price, visibility, and images.
3. Inventory: Check salable quantity in Catalog > Products (Columns > Salable Quantity) and, for MSI, verify source
   quantities in Stores > Inventory > Sources.
4. Order export: Place a test order; confirm ERP received the order with correct increment_id, items, totals, and
   addresses.
5. Shipment: Post a shipment from ERP; verify it in Sales > Shipments and that tracking appears on the order and in the
   customer account.
6. Indexers: Run bin/magento indexer:status to confirm Ready status and schedule mode.
7. Cron: Inspect var/log/cron.log (if enabled) and confirm recent run times; ensure cron is executing on schedule.
8. Errors: Check var/log/system.log and var/log/exception.log for integration-related errors; resolve any recurring
   messages.

## Common Issues and Solutions

### Authentication & Access

- Symptom: 401 Unauthorized when calling Magento APIs
    - Likely cause: Integration not activated, incorrect base URL, or expired/rotated tokens
    - Fix: Reactivate the Integration (System > Extensions > Integrations), verify the exact store base URL (https), and
      reissue tokens. Ensure server time is accurate to avoid OAuth timestamp issues.

- Symptom: ERP/iPaaS cannot complete OAuth callback
    - Likely cause: Callback/Identity URLs not whitelisted or blocked by firewall
    - Fix: Update Integration settings with correct URLs and allowlist connector IPs/domains.

### Catalog & Attributes

- Symptom: Product import fails due to missing attributes/attribute set
    - Likely cause: Attributes do not exist in Magento or wrong input types
    - Fix: Create required attributes and attribute sets with matching codes and types; re-map in connector.

- Symptom: Images do not appear on the storefront
    - Likely cause: Connector posting external URLs without permissions or missing base/small/thumbnail roles
    - Fix: Use supported media method (external URL vs binary upload) and set image roles correctly.

### Inventory / MSI

- Symptom: Salable Quantity does not change after sync
    - Likely cause: Updating the wrong Source/Stock or reservations not cleared
    - Fix: Update the correct selling Source(s); verify Source-to-Stock assignments. Ensure shipments/invoices are
      created so reservations are cleared.

- Symptom: Inventory updates succeed but items still out of stock
    - Likely cause: Indexers not running
    - Fix: Set indexers to Update by Schedule (System > Tools > Index Management) and ensure cron is running.

### Orders & Exports

- Symptom: Duplicate orders in ERP
    - Likely cause: Export triggered on both state=new and status=Processing without idempotency
    - Fix: Choose a single trigger and enforce uniqueness by increment_id in ERP.

- Symptom: ERP rejects shipping/payment methods
    - Likely cause: Unmapped Magento method codes
    - Fix: Map Magento shipping/payment codes to ERP carriers/service levels and payment terms.

### Performance & Timeouts

- Symptom: API timeouts or slow bulk updates
    - Likely cause: Payloads too large or peak-time contention
    - Fix: Use pagination and smaller batch sizes, enable delta updates, schedule heavy jobs off-peak, and configure
      retries with backoff in the connector.

### Cron & Indexers

- Symptom: Data visible in Admin but not on storefront
    - Likely cause: Indexers not up-to-date or cron not running
    - Fix: Verify with bin/magento indexer:status; set to Update by Schedule and confirm cron entries with crontab -l
      and var/log/cron.log.
