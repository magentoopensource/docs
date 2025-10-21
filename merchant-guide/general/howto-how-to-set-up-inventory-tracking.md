---
title: "How to Set Up Inventory Tracking"
description: "Configure stock levels, low stock alerts, and backorder management"
type: how-to
audience:
  - "Magento merchants"
difficulty: Intermediate
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Product catalog setup"
tags:
  - how-to
  - product-management
  - inventory
  - customer-management
  - content-management
last_updated: "2025-09-11T09:02:17.587Z"
---

# How to Set Up Inventory Tracking

## Overview

Set up reliable inventory tracking in Magento 2 to prevent overselling, reduce out-of-stock disappointments, and automate low-stock visibility. This how-to helps a Growing Merchant configure stock levels, low stock alerts, and backorders to improve conversion and customer trust.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Product catalog setup
- Server cron enabled (required for alerts, reservations/indexers, and emails)
- An admin role with permissions for Stores > Configuration, Inventory (Sources/Stocks), Sales > Orders, and Reports

## What You'll Accomplish

By following this guide, you will:

- Set up inventory tracking across your catalog using Magento's inventory (MSI) features
- Improve store performance and customer experience with low-stock alerts and backorder policies

## Step-by-Step Instructions

### Step 0: Choose your inventory scope (single vs. multi-source)

(Here, "scope" refers to your inventory model—single location vs. multiple locations—not the Magento configuration scope.)

If you have one stock location, use the Default Source and Default Stock. If you have multiple locations, go to Stores > Inventory > Sources to add each location (name, code, contact, pickup, and address), then go to Stores > Inventory > Stocks to create/adjust a Stock that aggregates the sources your website uses. After creating or editing a Stock, open the Sales Channels tab and assign the appropriate Website(s). Save the Stock to enable sales from that stock on the website.

### Step 1: Enable stock visibility and urgency messaging

Note on scope and cache: Use the scope switcher (upper-left) to select the target Website before making changes. After saving, if prompted, go to System > Cache Management and refresh the Configuration and Page Cache.

Go to Stores > Configuration > Catalog > Inventory > Stock Options. Set Display Out of Stock Products = Yes (recommended for SEO and back-in-stock alerts). Set Stores > Configuration > Catalog > Inventory > Stock Options > Only X left Threshold = a small number (e.g., 5) to show urgency on product pages.

Business tip: Keeping out-of-stock product pages live preserves indexed URLs and captures demand via stock alerts. Urgency messaging on low-inventory SKUs can lift product page conversion rates when inventory is tight.

### Step 2: Define global product stock controls

Go to Stores > Configuration > Catalog > Inventory > Product Stock Options and set:

- Manage Stock = Yes
- Backorders = No Backorders (or Allow Qty Below 0 / Allow Qty Below 0 and Notify Customer)
- Out-of-Stock Threshold = 0 (use a negative number like -5 to allow five backorders)
- Minimum Qty Allowed in Shopping Cart / Maximum Qty Allowed in Shopping Cart = per your business rules
- Notify for Quantity Below = your low-stock trigger (e.g., 10)

Click Save Config.

Revenue guidance: Use backorders when supplier lead times are predictable and margins cover potential cancellations or expedited shipping. Appropriate backorders can capture otherwise lost sales; poor use can increase cancellations and support costs.

### Step 3: Configure customer back-in-stock alerts

Note on scope and cache: Product Alerts are configured per Website. Use the scope switcher (upper-left) to select the intended Website before changing settings. After saving, refresh caches if prompted.

Go to Stores > Configuration > Catalog > Catalog > Product Alerts. Set Allow Alert When Product Comes Back in Stock = Yes. Choose Default Sender and Email Templates. Ensure cron is running to send alerts.

Scope and eligibility: Product Alerts are website-scoped. By default, only registered customers can subscribe to stock alerts.

KPI tip: Track subscribers per SKU, open/click rates, conversion rate from alert emails, and time-to-restock to quantify recovered revenue and inform purchasing.

Verification: From the storefront, log in as a customer and click "Notify me when this product is in stock" for SKU HDB-001. Set the product to In Stock with Qty > 0. Run cron or wait for cron to process product alerts. Confirm the back-in-stock email is received.

### Step 4: Assign sources and quantities to a product

Go to Catalog > Products, edit SKU HDB-001. In the product, open the Sources section, click Assign Sources, select your source(s), set Status = In Stock, set Qty (e.g., 150), and Save.

### Step 5: Adjust per-product Advanced Inventory (optional overrides)

On the same product, open Advanced Inventory. If needed, uncheck Use Config Settings to override: Backorders (e.g., allow for prelaunch SKUs), Notify for Quantity Below (e.g., set 25 for high-velocity items), Out-of-Stock Threshold (e.g., -5 if you allow five backorders). Save.

### Step 6: Set low-stock alert threshold for operational monitoring

Using either global Notify for Quantity Below or product-level overrides, confirm thresholds reflect your reorder points.

Replenishment guidance: Set Notify for Quantity Below to your Reorder Point = (average daily sales × supplier lead time in days) + safety stock. This reduces stockouts and rush shipping costs.

Admin notifications: To receive low-stock emails, go to Stores > Configuration > Catalog > Inventory and configure the Low Stock Email settings (enable sending, set recipient email, sender, and template). These emails are sent by cron; ensure server cron is enabled.

### Step 7: Validate salable quantity with a test order

From the storefront, purchase 2 units of HDB-001. After placing the order, go to Catalog > Products. Click Columns and enable Salable Quantity. Find SKU HDB-001 and confirm Salable Quantity decreased by 2. Note: In MSI, placing an order creates a reservation that reduces Salable Quantity immediately. Source Quantity (per warehouse) decreases when the order is shipped. If indexers are on schedule, allow cron to process; alternatively, reindex for immediate verification.

### Step 8: Trigger and confirm low-stock status

Lower the product qty to cross below the threshold (e.g., set qty from 12 to 9 when Notify for Quantity Below = 10) or place enough test orders to reach it. Go to Reports > Products > Low Stock and verify HDB-001 appears. If emails are supported and configured, check the inbox.

### Step 9: Test backorder behavior (if enabled)

Set the product qty to 0 while Backorders are allowed globally or on the product. From the storefront, add HDB-001 to the cart and proceed to checkout. With backorders enabled, confirm the notice appears on the product page and/or in the cart (e.g., "We don't have as many as you requested, so we'll backorder the remaining items") before purchase.

### Step 10: Finalize operational settings

Ensure Display Out of Stock Products remains as intended, confirm daily/weekly review cadence for Reports > Low Stock, and document reorder points per category/SKU. Train staff to go to Sales > Orders > View Order > Credit Memo, check Return to Stock for each item being refunded, and submit the credit memo.

Operating rhythm: Assign a daily check of Reports > Products > Low Stock to an inventory coordinator and a weekly review of SKUs with high alert subscribers to inform buy-plans.

## Verification

To confirm everything is working correctly:

1. Catalog > Products grid shows expected Salable Quantity for HDB-001 after a test order
1. Reports > Products > Low Stock includes HDB-001 after crossing the threshold
1. A test backorder can be placed and the backorder notice appears
1. A test stock alert email is received by the customer (if enabled)

## Common Issues and Solutions

- Salable Qty not decreasing after order: Ensure cron is running and reservations are enabled; reindex at System > Tools > Index Management (Inventory indexers). Verify the order is not canceled.
- Low Stock report empty: Verify Notify for Quantity Below > 0, reindex, and confirm the Stock is assigned to the Website in Stores > Inventory > Stocks > Sales Channels.
- Back-in-stock alert not sent: Confirm Product Alerts are enabled at the correct website scope; ensure cron is running; verify the customer is subscribed and the product returned to In Stock with Qty > 0.
- Cannot add to cart when backorders enabled: Check product-level Advanced Inventory overrides; ensure Out-of-Stock Threshold is negative if you plan to allow N backorders.
- Out-of-stock items not visible on storefront: Ensure Display Out of Stock Products = Yes at the website scope and clear caches under System > Cache Management.

## Related Resources

- [Magento User Guide](https://docs.magento.com/user-guide/)

---

*Need help? [Contact support]() or check our [troubleshooting guide]().*
