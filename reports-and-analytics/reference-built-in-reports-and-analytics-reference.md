---
title: "Built-in Reports and Analytics Reference"
description: "Complete reference of Magento's reporting capabilities and data export options"
type: reference
audience: "Magento merchants"
tags:
  - reference
  - analytics
last_updated: "2025-09-15T09:40:33.032Z"
---

# Built-in Reports and Analytics Reference

## Overview

Use Magento’s built‑in Reports to monitor sales performance, product demand, customer growth, reviews, and on‑site
search. Export CSV or Excel XML (.xml) from report grids, or use REST/GraphQL entity endpoints (orders, invoices,
customers, products) to retrieve raw data for your own aggregations. There are no dedicated REST endpoints for Admin
report summaries. Retrieve entities (orders, invoices, shipments, products, customers) and perform your own aggregations
in your BI tool. Accurate reporting drives better inventory planning, campaign ROI tracking, and service‑level
performance.

Business outcomes and KPIs examples:

- Inventory protection: Use Products > Bestsellers to inform weekly reorders; Products > Low Stock to prevent stockouts.
- Financial close: Use Sales > Orders/Tax/Invoiced/Refunds for monthly revenue, tax, and cash‑flow reconciliation.
- Marketing ROI: Use Sales > Coupons to evaluate discount performance and margin impact.
- Merchandising and search: Use Search Terms to prioritize new products and synonyms that improve findability.
- Search demand quality: Filter Search Terms for "Number of results = 0" and add synonyms or new products; redirect
  high‑volume terms to landing pages.

Scope: Built‑in Admin Reports vs Advanced Reporting (optional)

- This page focuses on the built‑in Admin Reports available in core. Advanced Reporting (optional SaaS) provides
  additional dashboards and is configured at Stores > Configuration > General > Advanced Reporting. Availability and
  metrics differ from built‑in reports.
- This page covers only built‑in Admin Reports. Advanced Reporting setup and metrics are not covered here.

### Quick report catalog (UI paths)

- Reports > Sales
    - Orders — Revenue and order counts by period.
    - Invoiced — Billed revenue (cash basis proxy) by period.
    - Shipping — Shipping charges by carrier/method.
    - Refunds — Returned amounts (credit memos) by period.
    - Tax — Tax collected by rate/class.
    - Coupons — Discount usage and performance.
- Reports > Products
    - Bestsellers — Top sellers by quantity.
    - Ordered — Quantity ordered by SKU.
    - Low Stock — Reorder candidates (threshold‑based; supports MSI Stock selection).
    - Views — Most viewed products.
    - Downloads (if enabled) — Download counts for digital products.
- Reports > Customers
    - Accounts — Total customer accounts by period.
    - New — New accounts by period.
- Reports > Reviews
    - By Customers — Review volume and ratings by customer.
    - By Products — Review volume and ratings by product.
- Marketing > SEO & Search
    - Search Terms — On‑site search demand and zero‑result terms.

Note: Available reports and labels can vary by version/edition and enabled modules.

### How to export a report

1. Go to the desired report (for example, Reports > Sales > Orders).
2. Set filters (these vary by report). For example, Sales reports use Period (Day/Month/Year), From/To dates, and Order
   Status; Products > Low Stock uses Stock (Sales Channel) and Quantity Threshold; Marketing > SEO & Search > Search
   Terms uses Store/View and Search Query filters.
3. Click Show Report to refresh the grid.
4. In the Export dropdown, choose CSV or Excel XML.
5. Click Export to download the file.

### Programmatic access (APIs)

- REST (Admin): Use REST admin endpoints for storewide operational data (orders, invoices, shipments, credit memos,
  customers, products). There are no dedicated REST endpoints for Admin report summaries; retrieve entities and
  aggregate in your BI tool or spreadsheet.
    - Examples: /V1/orders, /V1/invoices, /V1/shipments, /V1/creditmemo, /V1/customers/search, /V1/products
    - Authentication: Use an Admin token — POST /V1/integration/admin/token with admin credentials; include the token as
      Bearer in subsequent requests.
    - Pagination and filtering: Use searchCriteria[currentPage] and searchCriteria[pageSize] and add date/status
      filters. Example: GET /V1/orders?searchCriteria[currentPage]=1&searchCriteria[pageSize]
      =200&searchCriteria[filterGroups][0][filters][0][field]
      =created_at&searchCriteria[filterGroups][0][filters][0][condition_type]
      =from&searchCriteria[filterGroups][0][filters][0][value]=2025-09-01 00:00:00
- GraphQL (Customer context): Limitation — Only the signed‑in customer’s data is available (no admin‑wide sales data for
  orders/invoices/shipments/credit memos via GraphQL). Use GraphQL primarily for storefront and customer self‑service.
    - Examples: customer { orders }, products (as available for your use case)
- When to export vs build: For ad‑hoc or low volume, export CSV/Excel from report grids and aggregate in a spreadsheet.
  For recurring, large‑volume, or multi‑store needs, use REST with a BI/ETL tool (consider team skills, data volume, and
  frequency to avoid overbuilding).
- Note: Built‑in aggregated report views (e.g., Sales totals, Bestsellers) are not exposed as dedicated report APIs.

### Lightweight automation playbook

- Weekly replenishment: Export Products > Bestsellers and Low Stock; share with Purchasing for reorders. Owner:
  Purchasing. Cadence: Weekly. Trigger: Days of cover below target or quantity below threshold.
- Monthly finance package: Export Sales > Orders/Tax/Invoiced/Refunds for accounting close and reconciliation. Owner:
  Finance. Cadence: Monthly. Trigger: End of period.
- Monthly merchandising: Export Marketing > Search Terms to refine synonyms and add high‑demand products. Owner:
  Merchandising/SEO. Cadence: Monthly. Trigger: New zero‑result or rising‑volume queries.
- Verification: After export, confirm total orders and revenue match the same‑period filter in Sales > Orders. Check
  currency column = Base Currency. Spot‑check three orders for amounts and statuses.

### Edition and version notes

- The Reports menu is available in both Adobe Commerce and Magento Open Source.
- Some capabilities (for example, Advanced Reporting) require additional configuration and service availability.
- UI labels and exact report availability vary by version—verify in your Admin.

## Configuration

Prerequisites and settings

- Cron: Ensure Magento cron is configured and running so statistics and (optional) Advanced Reporting can update.
- Time zone and currency: Set at Stores > Configuration > General > General > Locale Options (Time Zone) and Stores >
  Configuration > General > Currency Setup. Sales report amounts are calculated in Base Currency. Period grouping uses
  the Admin time zone (Stores > Configuration > General > General > Locale Options).
- Enable Reports data collection: Stores > Configuration > General > Reports > General Options > Enable Reports = Yes.
  Required for Product Views and some aggregated reports to collect data going forward (not retroactive).
- Refresh statistics: Go to Reports > Statistics, select the desired items, and click Refresh. Use this if totals appear
  stale after changing orders or configurations.
- Permissions: Assign report access via System > Permissions > User Roles. Grant Resources under Reports (Sales,
  Products, Customers, Reviews, Marketing) as needed.
- Advanced Reporting (optional): Configure at Stores > Configuration > General > Advanced Reporting. Requires working
  cron and a publicly accessible base URL. Advanced Reporting prerequisites: Publicly accessible base URL (no basic
  auth/IP allowlist), correct Base URL in Stores > Configuration > General > Web, cron active (bin/magento cron:run),
  and at least one order in the last 24 hours. Initial data sync can take up to 24 hours.

Troubleshooting

- Report is empty: Verify date range and order statuses; click Show Report; refresh Reports > Statistics; confirm cron
  is running; check the store view.
- Totals don’t match the Orders grid: Ensure the same statuses/period are selected; refresh statistics; verify time zone
  settings.
- Export is slow or times out: Narrow the date range; filter by status; export in smaller batches.
- Advanced Reporting shows no data: Confirm it’s enabled, cron is active, and the base URL is publicly accessible; allow
  time for initial data sync.
- Low Stock with MSI: Select the Stock (Sales Channel) in the report filter. “Notify for Quantity Below” can be set
  globally or per Source Item; ensure thresholds are configured for accurate results (Stores > Configuration > Catalog >
  Inventory and Catalog > Products > Sources).
- Large export performance: Increase PHP memory_limit and max_execution_time, and consider exporting by month/quarter.
  Avoid browser timeouts by narrowing date ranges.
- Month‑end close checklist (Finance):
    - Use Sales > Orders with the period and statuses your business recognizes as “booked” to get order counts and
      revenue (Base Currency).
    - Cross‑check Sales > Invoiced for billed amounts and Sales > Refunds for credit memos in the same period/time zone.
    - Reconcile tax using Sales > Tax by rate/class; verify the period boundary uses the Admin time zone.
    - If discrepancies persist, refresh Reports > Statistics and re‑run the exports.