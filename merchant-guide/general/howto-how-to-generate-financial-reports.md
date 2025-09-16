---
title: "How to Generate Financial Reports"
description: "Create comprehensive financial reports for tax, accounting, and business planning"
type: how-to
audience:
  - "Magento merchants"
difficulty: Intermediate
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
tags:
  - how-to
last_updated: "2025-09-16T18:46:43.692Z"
---

# How to Generate Financial Reports

## Overview

Generate accurate, exportable financial reports from Magento to speed up tax filing, accounting reconciliation, and planning—reducing month-end close time and improving cash-flow visibility.

Map your outputs to accounting processes for a faster, cleaner close:
- Invoices CSV = revenue recognition and tax payable
- Credit Memos CSV = refunds/contra-revenue
- Tax report = tax payable by tax rate/code
- Orders report = demand trend tracking (for planning)

Tip: For reconciliation, use invoice date and base currency; for demand KPIs, use order date. Using the Invoiced report (invoice date) typically reduces payment-provider variances by >90%, cutting reconciliation time by 1–3 hours per close and lowering audit risk.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel

## What You'll Accomplish

By following this guide, you will:

- Generate accurate sales, tax, invoice, and refund reports
- Reduce month-end close time and improve cash-flow visibility

## Step-by-Step Instructions

### Step 0: Refresh report statistics

In Admin, go to Reports > Statistics. Select all report types you plan to use (Orders, Invoices, Tax, etc.). Click Refresh Statistics and confirm. Wait until the Updated At column shows the current timestamp.

Tip: Ensure your store time zone aligns with your payment provider’s report time zone (or note the difference). Use Base currency for accounting and reconciliation.



### Step 1: Generate a Sales Orders summary

Go to Reports > Sales > Orders. Set From and To dates for your reporting period. Choose Show By (Day/Month/Year) based on how you want totals grouped. Set Order Status to Specified, then select Processing and Complete. Exclude Pending Payment, Canceled, and Closed unless you explicitly want to include fully refunded orders in this report. Click Show Report. Use Export to download CSV if needed.

Note: Order Status must be set to Specified before you can multi-select statuses.



### Step 1a: Generate an Invoiced report for revenue capture

Go to Reports > Sales > Invoiced. Set From and To dates (Invoice Date), choose Show By (Day/Month/Year), and set Order Status to Specified if needed. Click Show Report, then use Export to download CSV. Use Total Invoiced and Total Refunded from this report as the basis for payment provider reconciliation.



### Step 2: Produce a Tax report by tax rate/code

Navigate to Reports > Sales > Tax. Set From and To dates, choose the period grouping, and set Order Status to Specified if needed. This report summarizes collected tax by tax rate/code (not by jurisdiction). Click Show Report. Export CSV for filing support.

Note: The native Tax report summarizes by tax rate/code. If you file by jurisdiction (city/county), export Sales > Invoices and Credit Memos with order shipping/billing addresses, or use a tax service integration (e.g., Avalara, Vertex, TaxJar) for jurisdiction-level reports.



### Step 3: Export transaction-level Invoices for accounting

Go to Sales > Invoices. Filter by Invoice Date (From, To). Optionally filter by Status (Paid/Open/Canceled) and Purchased From (Store). Click Export > CSV to download detailed invoice-level data including totals and taxes.



### Step 4: Export Credit Memos (refunds) to capture adjustments

Go to Sales > Credit Memos. Filter by the period. Click Export > CSV. Keep this file with your invoice export to reflect net sales after refunds.



### Step 5: Verify refunds in the Refunds report

Go to Reports > Sales > Refunds. Set the same period and statuses. Click Show Report. Export if you need a summarized view of refunds separate from the transaction-level credit memo export.



### Step 6: Create a Product Ordered report for planning

Go to Reports > Products > Ordered. Set From and To dates and any store filters. Click Show Report. Export CSV to analyze top sellers, slow movers, and to inform replenishment and promotions. Create a top-20 SKU list to prioritize reorders and featured placements; identify bottom-20 SKUs for markdowns or bundle promotions; align inventory buys to supplier lead times to improve turns and reduce stockouts.



### Step 7: Optional: Export Orders grid for a master dataset

Go to Sales > Orders. Filter by the reporting period and desired statuses. Click Export > CSV. Use this as a master dataset to cross-reference with invoices and credit memos if your accounting process needs additional fields.

Note: The Orders grid totals are by order-created date and include both Base and Order (Purchased) currency columns. Use a consistent currency and date basis across reports when building a master dataset.



### Step 8: Reconcile against payment provider statements

Download your provider’s settlement report for the same period. In Magento, use Reports > Sales > Invoiced with the same date basis (Invoice Date) and statuses. Compare Total Invoiced minus Total Refunded to the provider’s gross captures for the period. Account for gateway fees to reconcile to net deposits. If Payment Action is Authorize (capture later), align to the capture/invoice date, not order date. Investigate any variance >1%.



### Step 9: Archive and document your close

Save all CSV exports with a consistent naming convention (e.g., 2025-08/AllStores/Invoiced_2025-08_Base.csv). Store them in a secure, backed-up folder. Record reconciliation notes and any adjustments made for audit purposes.

- Restrict access to exported financial reports to finance/admin roles.
- Store exports in an encrypted, backed-up location.
- Retain monthly exports and reconciliation notes for at least 7 years (consult your local regulations).



## Verification

- Confirm that Reports > Statistics shows Updated At within the last 24 hours for the reports you ran.
- In Sales > Orders (report), verify that Total Invoiced - Total Refunded (for the period) approximately equals Reports > Sales > Invoiced "Total Invoiced - Total Refunded" (same date basis).
- Sum Grand Total (Base) in Sales > Invoices export for the period and compare to Reports > Sales > Invoiced totals.
- If variances exceed 1%, check date basis (order vs invoice date), statuses, currency (base vs order), and offline vs online refunds.

## Common Issues and Solutions

- Totals don’t match gateway: Ensure you’re using the Invoiced report (invoice date), not Orders. Check Payment Action (Authorize vs Authorize and Capture).
- Refunds missing: Confirm credit memos were issued as Online refunds; Offline refunds won’t appear on gateway statements.
- Double counting: Exclude Canceled and Closed in Orders report to avoid including fully refunded orders twice.
- Time zone mismatch: Align store time zone and provider report time zone.
- Currency mismatch: Use Base currency for accounting; avoid mixing Base and Order (Purchased) currency.
- Stale data: Run Reports > Statistics > Refresh for relevant reports or ensure cron is running.
- Export too large/timeouts: Narrow the date range or export in batches.

## Related Resources

- Adobe Commerce User Guide - Stores & Sales Introduction: https://experienceleague.adobe.com/docs/commerce-admin/stores-sales/introduction.html
- Sales Reports overview: https://experienceleague.adobe.com/docs/commerce-admin/stores-sales/sales-reports/sales-reports.html
- Orders and Invoices: https://experienceleague.adobe.com/docs/commerce-admin/stores-sales/order-management/orders/orders.html#invoices

---

*Need help? [Contact support]() or check our [troubleshooting guide](#common-issues-and-solutions).*