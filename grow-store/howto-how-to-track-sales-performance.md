---
title: "How to Track Sales Performance"
description: "Monitor revenue, product performance, and sales trends for business insights"
type: how-to
audience:
  - "Magento merchants"
difficulty: Intermediate
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Product catalog setup"
  - "Cron is enabled and running (required for report aggregation and Advanced Reporting)"
  - "Your Admin role has access to Reports (System > Permissions > User Roles)"
  - "Store Timezone is set (Stores > Configuration > General > Locale Options)"
tags:
  - how-to
  - product-management
  - performance
last_updated: "2025-09-12T09:25:39.620Z"
---

# How to Track Sales Performance

## Overview

Use Magento’s Sales, Products, and Customers reports (and Advanced Reporting if enabled) to monitor booked vs. realized
revenue, identify top/lagging products, and evaluate promotions—so you can adjust inventory, pricing, and campaigns each
week.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Product catalog setup
- Cron is enabled and running (required for report aggregation and Advanced Reporting)
- Your Admin role has access to Reports (System > Permissions > User Roles)
- Store Timezone is set (Stores > Configuration > General > Locale Options)

## What You'll Accomplish

By following this guide, you will:

- Learn a repeatable process to track sales performance using Magento reports
- Improve your store's performance and customer experience

## Step-by-Step Instructions

### Step 1: Confirm reporting scope and time range

Decide the revenue stage you will track (Orders for booked revenue, Invoices for realized revenue, or net of Refunds).
Choose the store scope and period:

- Scope: Use the Scope selector in the Admin header (All Store Views, a specific Website, or Store View) so figures
  match your reporting needs.
- Time range: Choose a standard period (for example, Last 7 days or Last 30 days) to enable consistent comparisons.

### Step 2: Check the Dashboard for a quick health snapshot

Go to Admin sidebar > Dashboard. Use the Date Range selector (for example, Last 7 Days) and confirm the Scope (All Store
Views or a specific Store View). Review the revenue and orders charts for the selected period to spot trends.

Act on it: If revenue dips versus the prior period, adjust campaigns or on-site promotions. If orders spike, review
inventory and fulfillment capacity.

### Step 3: Run the Sales Orders report (booked revenue trend)

Go to Reports > Sales > Orders, then:

- Select Scope from the Admin header to match your reporting needs
- Set From and To dates (for example, last 30 days)
- Set Period to Day, Month, or Year (Week is not available in the native report)
- Set Order Status to the statuses that represent valid booked orders for your workflow (for example,
  Pending/Processing/Complete; exclude Canceled and Closed if not relevant)
- Click Show Report
- Optional: Export > CSV for sharing

Note: Some reports use aggregated data. If you recently received orders, go to Reports > Statistics and Refresh the
relevant reports to include the latest data.

Act on it: Use trends to adjust merchandising and ad budgets. For example, if booked revenue is rising in a category,
feature those products and increase inventory.

### Step 4: Analyze realized revenue and refunds

- Go to Reports > Sales > Invoiced to view invoiced totals for the same Scope, date range, and Period
- Then go to Reports > Sales > Refunds for the same parameters to understand returns
- Record both for comparison with Orders

Note: Some reports use aggregated data. If you recently invoiced or refunded orders, refresh the relevant statistics
under Reports > Statistics.

Act on it: If realized revenue lags booked revenue, review payment capture and fulfillment bottlenecks. If refunds are
high, investigate root causes (quality, sizing, shipping damage) and update product pages or policies.

### Step 5: Evaluate product performance (winners and laggards)

- Go to Reports > Products > Ordered
- Set the same Scope, date range, and Period, then click Show Report
- Note the top 10 products by quantity and by revenue
- Check Reports > Products > Bestsellers to confirm consistency

Note: If you recently received orders, refresh statistics under Reports > Statistics to include the latest data.

Act on it: Reorder top sellers and feature them on the homepage/category pages. For laggards, improve
images/descriptions, test price adjustments, or bundle with top sellers.

### Step 6: Assess promotion impact

- Go to Reports > Sales > Coupons
- Select the same Scope and date range, then click Show Report
- Review revenue, discount amounts, and uses per coupon

Act on it: Calculate net lift = Revenue − Discount Amount − incremental Shipping/Handling. Pause codes with negative net
lift; increase budget or visibility for high-ROI codes.

Note: If data looks stale, refresh the Coupons statistics under Reports > Statistics.

### Step 7: Check shipping, tax, and cost drivers (optional)

- Run Reports > Sales > Shipping to identify carriers/methods driving the highest shipping spend relative to revenue
- Run Reports > Sales > Tax to review tax collected by rate/region

Note: Product Cost is not included in these reports. For margin analysis, ensure the Cost attribute is maintained and
use custom reporting or Advanced Reporting.

Act on it: Promote lower-cost shipping methods when appropriate, negotiate carrier rates in high-cost regions, and
validate tax settings for compliance.

### Step 8: Review customer activity (optional)

- Go to Reports > Customers > Orders to see orders by customer and average amounts
- Go to Reports > Customers > New to see new customer signups in the period

Act on it: Identify high-value customers for loyalty offers and target new customers with onboarding or cross-sell
campaigns.

### Step 9: Enable and use Advanced Reporting (if available)

- Go to Stores > Configuration > General > Advanced Reporting
- Set Enable to Yes and complete required fields (Industry and Time of day to send data). Ensure your store Timezone is
  set at Stores > Configuration > General > Locale Options
- Save the configuration and ensure cron is running

Prerequisites and timing:

- Your storefront base URL must be publicly accessible (no maintenance mode or HTTP authentication)
- HTTPS is recommended
- Initial data sync can take up to 24 hours after enabling and saving configuration
- Ensure at least one completed order exists for the selected scope

After the initial data sync, go to Reports > Advanced Reporting to view Sales, Customers, and Products dashboards with
trend analyses.

Act on it: Use these dashboards to spot multi-week trends and seasonality, then update inventory buys, pricing, and
campaign plans.

### Step 10: Export and compute key KPIs for your weekly snapshot

Export Sales > Orders and Products > Ordered to CSV. In a spreadsheet, compute and document:

- AOV = 'Revenue' / 'Orders' (use the columns from the Sales > Orders report)
- Refund rate = 'Total Refunded' / 'Total Invoiced' (use Sales > Orders columns, or cross-reference Sales > Refunds and
  Sales > Invoiced)
- Top 10 products by quantity and by revenue
- Optional: Shipping cost % Revenue = (Shipping Amount / Revenue)

Include in your weekly snapshot: Orders, Revenue, AOV, Refund rate, Coupon ROI, and Shipping cost % Revenue. Share every
Monday with Ops, Merch, and Marketing to set priorities.

### Step 11: Refresh statistics and set a reporting cadence

- Go to Reports > Statistics
- Select relevant reports and click Refresh or Refresh Lifetime Statistics
- Schedule a recurring calendar reminder (weekly) to rerun Steps 3–10 using the same Scope and filters

## Verification

To confirm everything is working correctly:

- Cross-check total Orders count in Sales > Orders with the Orders grid for the same date range and statuses
- Reconcile 'Total Invoiced' with a sample of invoices in Sales > Invoices for the same period
- Confirm a known coupon code appears in Reports > Sales > Coupons with the expected Usage count
- Compare Dashboard revenue with Sales > Orders 'Revenue' for the same period and Scope

## Common Issues and Solutions

- No data in reports: Refresh Reports > Statistics and ensure the correct Scope and date range are selected
- Coupons report empty: Verify orders actually used coupon codes and the selected statuses include those orders
- Advanced Reporting unavailable: Ensure cron is running, base URL is public, and wait up to 24 hours after enabling
- Numbers don’t match between Dashboard and Sales > Orders: Align Scope, date range, and statuses; Dashboard uses its
  own filters
- Export shows partial data: Increase PHP memory limit and try exporting in smaller date ranges
