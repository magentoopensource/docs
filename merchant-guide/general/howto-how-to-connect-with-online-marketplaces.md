---
title: "How to Connect with Online Marketplaces"
description: "Integrate with Amazon, eBay, and other marketplaces to expand sales channels"
type: how-to
audience:
  - "Magento merchants"
difficulty: Intermediate
estimated_time: "1–3 hours for initial connector setup; 1–3 days to validate listings and order flows"
prerequisites:
  - "Access to Magento Admin Panel"
tags:
  - how-to
last_updated: "2025-09-16T19:04:50.958Z"
---

# How to Connect with Online Marketplaces

## Overview

Expand revenue and improve customer experience by syncing your Magento store with Amazon, eBay, and other marketplaces. This guide helps a growing merchant connect, list products, and automate orders, inventory, and pricing—reducing manual work and preventing overselling.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Active Amazon Seller Central and/or eBay Seller Hub accounts with Professional/Business plan and approval to list in your categories
- GTINs (UPC/EAN/ISBN) for products or approved GTIN exemptions
- Defined shipping, return, and tax policies (eBay business policies; Amazon fulfillment settings)
- A staging environment that matches production for safe installation and testing
- Composer and CLI access and the ability to deploy code (on‑prem) or Git access for Adobe Commerce on cloud projects
- Magento cron enabled and running (required for all sync tasks)
- A backup and rollback plan

## What You'll Accomplish

By following this guide, you will:

- Connect your Magento store to Amazon and/or eBay using a vetted connector
- List a small test set of products and validate attribute/category mappings
- Automate price, inventory, and order sync end-to-end

## Step-by-Step Instructions

### Step 0: Choose your integration model

Decide whether Magento should create and manage listings (full sync) or only import orders from existing marketplace listings. 

- Orders-only: fastest launch with minimal catalog work; lower risk of listing policy violations; limited branding and no centralized price control. Best if you already have compliant listings.
- Full sync: centralized price/stock control, consistent content, eligibility for repricing strategies; requires attribute/category mapping and more upfront work. 

Tip: Orders-only yields faster time-to-revenue; full sync maximizes margin control and reduces operational rework long-term.

### Step 1: Select a marketplace connector for Magento 2

Evaluate connectors using this checklist:

- Magento 2.x compatibility and MSI (multi-source) inventory support
- Amazon SP-API and eBay API support; sandbox/test options
- Order import with tax/fee handling; FBA/MFN support; eBay business policies support
- Price rules (markup, min margin, MAP), repricing options, multi-store/multi-currency
- Listing creation and linking to existing listings; variation (parent/child) support
- SLA-backed support, documentation quality, and total cost (license + per-order/GMV fees)

Confirm support for your exact Magento version, order import, pricing rules, and target marketplaces. Review documentation and pricing.

### Step 2: Install the connector in staging via Composer

Obtain the package name from the vendor’s documentation or Magento Marketplace. Install and validate in a non-production environment first.

On-prem/self-hosted (staging) example:
- php -d memory_limit=-1 composer require <vendor/package:^version>
- php bin/magento module:enable <Module_Name>
- php bin/magento setup:upgrade
- php bin/magento setup:di:compile
- php bin/magento setup:static-content:deploy -f
- php bin/magento cache:clean

For production on-prem, use maintenance mode and a deployment window:
- php bin/magento maintenance:enable
- php -d memory_limit=-1 composer require <vendor/package:^version>
- php bin/magento module:enable <Module_Name>
- php bin/magento setup:upgrade
- php bin/magento setup:di:compile
- php bin/magento setup:static-content:deploy -f
- php bin/magento cache:clean
- php bin/magento maintenance:disable

Adobe Commerce on cloud:
- Run composer require <vendor/package:^version> in your local dev environment
- Commit composer.json and composer.lock, then push to the integration branch
- Let cloud build/deploy hooks run (they execute setup:upgrade, compile, deploy)
- Verify on the integration environment, then promote to staging and production via the normal pipeline
- Do not SSH into the production container to install modules

### Step 3: Connect your Amazon and/or eBay seller accounts

In Magento Admin, open the connector’s configuration area (location varies by vendor; typically under Stores > Configuration > <Connector Vendor> or via a top-level <Vendor> menu). From there, click Add Account to start the OAuth/consent flow.

- Amazon: Select the correct region during authorization (North America, Europe, Far East) to avoid invalid marketplace IDs. Ensure the connector is registered as an SP-API application and authorize it in the correct Seller Central account and region.
- eBay: Approve access from eBay Seller Hub. Choose the correct site (e.g., eBay US, eBay UK) if the connector supports multiple sites.

Save credentials and confirm the account status shows Connected in the connector.

### Step 4: Configure global sync settings

Set time zone, currency, and sync frequency. Map stock source (MSI) to the channel. Define price rules or channel price adjustments. Configure order import settings (status mapping, customer creation, tax handling, shipping mappings, FBA vs. merchant-fulfilled).

Pricing and margin:
- Implement channel-specific price rules to cover marketplace fees and shipping costs (for example, +12% markup with a minimum margin floor). This helps protect profitability while remaining competitive.

Taxes and currency:
- Taxes: In many regions, marketplaces collect/remit tax. Configure the connector to import marketplace-collected tax as order tax and avoid double-charging in Magento. Some connectors map imported orders to a non-taxable customer group—enable if required.
- Currency: If marketplace currency differs from your website currency, enable connector-side conversion or set website/store-view pricing scope accordingly so posted prices match the marketplace.

### Step 5: Map categories and required attributes

For each marketplace category you plan to use, map Magento categories and attributes to marketplace-required fields (Brand, Condition, Item Specifics for eBay; Product Type/Category attributes for Amazon). Validate that UPC/EAN/ISBN is present or set up GTIN exemptions.

Variations and brand requirements:
- Amazon: Define parent (configurable) and child (simple) relationships that match category variation themes (e.g., Size, Color). Ensure your Brand is authorized or enrolled in Brand Registry; some brands/categories are gated.
- eBay: Map Item Specifics per category and include mandatory specifics to avoid listing rejection.

### Step 6: Prepare and link products

Clean SKUs and ensure each product has correct identifiers, images, and descriptions.

Linking and duplicate prevention:
- Amazon: Link products to existing ASINs using UPC/EAN/ISBN or ASIN. Only create new ASINs if no match exists and you meet new listing requirements. Avoid duplicate ASIN creation.
- eBay: Link to existing listings by Item ID or create new listings using templates; ensure SKU uniqueness to prevent duplicate listings.

For existing marketplace listings, use the connector’s linking tool to associate them with Magento SKUs. For new listings, create listing templates (title, bullets, images, policies).

### Step 7: Publish a small test set

Select 5–10 SKUs and submit them to the marketplace (or enable linking). Use the connector’s preview/validation to catch errors. Confirm the status transitions (Queued > Submitted > Active) and resolve any attribute/category issues before scaling.

### Step 8: Enable inventory and price sync

Turn on quantity and price synchronization in the connector. Make a small price change in Magento to a test SKU and verify it updates on the marketplace within the documented sync window. Confirm stock decrements after a Magento or marketplace test sale.

Operational guidance:
- Ensure Magento cron is running (Admin > Stores > Configuration > Advanced > System > Cron or via CLI) so sync jobs execute reliably.
- Expected latency: Price/qty updates typically reflect within 5–30 minutes depending on connector schedule and API throttling.
- Set a safe sync frequency to avoid API rate-limits; many connectors provide backoff controls.
- Make sure only one system owns listing, inventory, and order sync per channel. Do not run multiple connectors or overlapping ERP/WMS updates against the same marketplace listings.

### Step 9: Test order import and fulfillment

Place a low-value order on the marketplace. Confirm the order imports into Magento with correct items, taxes, and shipping method. Ship the order from Magento (or via FBA if configured) and verify tracking information updates back to the marketplace.

Exceptions and post-order flows:
- Cancellations and refunds: Decide where refunds will be initiated (marketplace vs Magento). Configure the connector to sync cancellations, returns, and partial refunds. Test a cancellation and a partial refund to ensure statuses and amounts sync back correctly.

### Step 10: Roll out to your full catalog

Scale listing to additional SKUs in batches by category. Monitor connector logs and marketplace health dashboards for rejections or policy flags. Set alerts for order import failures and low stock. Document your operational runbook.

Monitoring and KPIs:
- Alerts: order import failures, low stock (below safety stock), listing errors above threshold, API auth failures.
- KPIs: Buy Box win rate (Amazon), late shipment rate, cancel rate, out-of-stock rate, GMV by channel, return rate by SKU, and contribution margin per channel.

## Verification

Use this checklist to confirm everything is working correctly:

1. Listings: Confirm test SKUs show Active on Amazon/eBay and in the connector with no validation errors.
2. Price/Inventory: Change price/qty for a test SKU in Magento; verify the update on the marketplace within the expected sync window.
3. Orders: Place a marketplace test order; verify it appears in Magento with correct tax/fees and can be shipped; confirm tracking syncs back to the marketplace and the order status updates.

## Common Issues and Solutions

- Listings rejected: Missing GTIN or invalid attribute. Solution: Add UPC/EAN/ISBN or apply for GTIN exemption; verify category-required attributes and Item Specifics.
- Orders not importing: Connector cron not running or order status filter excludes Pending/Unshipped. Solution: Verify cron and job schedules; adjust order status mapping to include relevant states.
- Price not updating: Wrong website scope or currency mismatch. Solution: Set website price scope correctly and map marketplace to the correct website/store view; review currency conversion settings.
- OAuth token expired: Reauthorize the account in connector settings; set calendar reminders before token expiry.
- API throttling: Reduce sync frequency, stagger jobs, or enable exponential backoff in the connector; limit batch sizes during peak hours.

## Related Resources

- [Magento User Guide](https://docs.magento.com/user-guide/)

---

*Need help? [Contact support]() or check our [troubleshooting guide]().*
