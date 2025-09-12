---
title: "How to Process Returns and Refunds"
description: "Manage customer returns, RMA process, and issue refunds efficiently"
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
tags:
  - how-to
  - customer-management
last_updated: "2025-09-12T11:17:58.223Z"
---

# How to Process Returns and Refunds

## Overview

Process returns and refunds efficiently to protect revenue, keep inventory accurate, and improve customer satisfaction for a growing Magento store.

## Prerequisites

Before you begin, make sure you have:

- Admin access with permissions for Sales > Orders, Invoices, Credit Memos, and (Adobe Commerce only) Sales > Returns
- A payment method that supports online refunds (optional but required for Online Refunds)
- Adobe Commerce is required for RMAs and Store Credit (Customer Balance); Open Source supports refunds via Credit Memos only
- A test order/invoice to practice in a staging environment

## What You'll Accomplish

By following this guide, you will:

- Process returns and refunds correctly using Credit Memos and (Adobe Commerce) RMAs
- Improve your store's performance and customer experience

## Step-by-Step Instructions

### Step 0: Decide your processing path

Choose one:
- Path A (Open Source and Adobe Commerce): Refund-only via Credit Memo for quick refunds without formal authorization.
- Path B (Adobe Commerce only): Full RMA workflow for authorization, inspection, exchanges, or store credit.

Decision guide:
- Use Credit Memo for simple refunds (no inspection or exchange required) or when the item is not being returned (e.g., service issues, partial refunds).
- Use RMA when you need pre-authorization, physical inspection, exchanges, store credit, or to capture return reasons/conditions for analytics.

Business guidance: Credit Memos minimize handling cost and speed resolution. RMAs improve quality control, enable exchanges/store credit to retain revenue, and provide better data on return drivers.

### Step 1: Confirm payment and email settings

In Admin, go to Stores > Configuration. Verify your payment method supports refunds (for example, PayPal/Braintree capture and refund settings) under Sales > Payment Methods. Online Refund is available only if the original payment was captured and the gateway supports refunds; otherwise use Refund Offline and process the funds in your gateway. Enable Sales emails under Sales > Sales Emails (Order, Invoice, Credit Memo). Click Save Config. If the system prompts that the cache is invalidated, go to System > Tools > Cache Management and click Flush Magento Cache.

### Step 2: Optional (Adobe Commerce): Enable RMA and Store Credit

Go to Stores > Configuration > Sales > RMA Settings. Set Enable RMA on Storefront = Yes and Enable RMA on Product Level as needed. Then go to Stores > Configuration > Customers > Customer Configuration > Store Credit Options (Customer Balance) and set Enable Store Credit = Yes if you plan to offer store credit. Save.

### Step 3: Locate the order and review its status

Navigate to Sales > Orders. Search by order number, email, or customer name. Open the order and confirm it is invoiced and, if applicable, shipped. To issue a refund (Credit Memo), the order must have at least one invoice. If there is no invoice, create one first. Note the items and quantities requested for return.

### Step 4: Path A: Create a Credit Memo (refund-only)

From the order view, click Invoices and open the invoice. Click Credit Memo. For each item, set Qty to refund and tick Return to Stock for items coming back. The Return to Stock checkbox adds the quantity back to inventory. In Multi-Source Inventory (MSI), Magento returns the quantity to the source from which the order was shipped. Set Refund Shipping and Adjustment Refund/Fee if needed.

- Adjustment Refund: Adds an extra amount to the refund (for example, goodwill credit).
- Adjustment Fee: Deducts an amount from the refund (for example, a restocking fee).
- Refund Shipping: Refunds the shipping amount (and shipping tax if applicable). Apply per your policy.

If the gateway supports it, click Refund to send funds back online; otherwise click Refund Offline and process the refund in your gateway. Caution: Avoid duplicate refunds—do not issue an Online Refund in Magento and also refund the same amount directly in your payment gateway.

### Step 5: Path B (Adobe Commerce): Create and authorize an RMA

Go to Sales > Returns (RMA). You can create an RMA only for orders that have at least one shipment (order status Processing or Complete). Click New Return. Select the customer/order, choose items and quantities, set Reason, Item Condition, and Resolution (Refund, Exchange, Store Credit). Submit to create an RMA with a number. Review and set status to Authorized; send the RMA email with return instructions.

Tip: Configure your RMA Reason, Item Condition, and Resolution values at Stores > Other Settings > RMA Reasons, RMA Item Conditions, and RMA Resolutions before starting.

### Step 6: Receive and inspect returned items (RMA)

When the package arrives, open the RMA. Mark items as Received and set the verified condition and accepted quantities. Add internal comments if discrepancies exist.

### Step 7: Issue refund from RMA via Credit Memo

From the RMA, click Create Credit Memo. Confirm quantities, tick Return to Stock per item, set shipping and adjustments, and choose Refund (online) or Refund Offline as appropriate.

### Step 8: Handle exchanges or store credit (optional, Adobe Commerce)

- Exchanges: Open the RMA and click Create Exchange Order, select replacement items (correct size/color), and complete the order, setting pricing per your policy.
- Store Credit: On the Credit Memo form, select Refund to Store Credit to add the amount to the customer’s balance. Refund to Store Credit is available for registered customers only; guests are not eligible for Store Credit.

Best practices: Offer a small incentive (for example, bonus credit) to encourage store credit or exchanges over cash refunds, and expedite replacement shipping to increase loyalty and retained revenue.

### Step 9: Validate inventory and sources (Multi-Source Inventory, MSI)

Go to Catalog > Products and open the returned SKU. Confirm Quantity and Salable Quantity increased. In Multi-Source Inventory, returned quantities are added back to the source they were shipped from. To move stock between sources, edit source item quantities for the SKU at Catalog > Products > [open product] > Sources, or use an import to update source quantities.

### Step 10: Notify the customer and document

Ensure the Credit Memo and/or RMA emails were sent. Add a comment to the order/RMA noting the refund amount, date/time, and next steps. Avoid including any sensitive payment details; use the system’s masked payment details and transaction IDs for reference. Attach any internal notes for audit.

### Step 11: Verify payment reconciliation

Go to Sales > Transactions to confirm the refund transaction and ID. You can also open the order and review related invoices and credit memos. Match the refunded amount (items, tax, shipping) to the Credit Memo. Export to accounting if integrated.

### Step 12: Measure the impact

Track metrics weekly: average return cycle time, percentage of online vs offline refunds, restock rate, and top return reasons. Export data from Sales > Credit Memos and (Adobe Commerce) Sales > Returns grids to CSV for analysis. Categorize return reasons to spot product/content issues and update product pages, size guides, and packaging. Use cycle-time insights to streamline carrier choices and warehouse handling. Tune your return policy and incentives to shift refunds toward exchanges or store credit where appropriate.

## Verification

To confirm everything is working correctly:

1. Place a test order and generate an invoice.
2. Create a Credit Memo with Return to Stock checked for one item.
3. If supported, process an Online Refund and verify a transaction appears under Sales > Transactions.
4. Confirm Credit Memo email is enabled (Stores > Configuration > Sales > Sales Emails) and sent (check order Comments History).
5. Verify product Quantity and Salable Quantity increased (and correct source in MSI).
6. For RMA (Adobe Commerce): Create an RMA for a shipped order, authorize it, receive items, and create a Credit Memo; verify status changes and notification emails.

## Common Issues and Solutions

- Refund button not available: Ensure an invoice exists for the order and the payment was captured.
- Online Refund not available: The gateway may not support refunds or the payment was not captured; use Refund Offline and process the refund in the gateway.
- Return to Stock not visible: Product Manage Stock must be Yes; downloadable/virtual items cannot be returned to stock.
- Cannot create RMA: Order must have a shipment (Processing or Complete) and RMA must be enabled in configuration.
- Store Credit option missing: Enable Store Credit (Customer Balance) and ensure the customer is registered (not a guest).
- Duplicate refund risk: Do not click Refund (online) in Magento and also refund the same amount in your payment gateway; choose one method.

## Related Resources

- Credit Memos: https://experienceleague.adobe.com/docs/commerce-admin/stores-sales/order-management/credit-memos/credit-memos.html
- Returns (RMA): https://experienceleague.adobe.com/docs/commerce-admin/stores-sales/order-management/returns/returns.html
- Store Credit (Customer Balance): https://experienceleague.adobe.com/docs/commerce-admin/customers/customer-accounts/store-credit.html
- Inventory Management: https://experienceleague.adobe.com/docs/commerce-admin/inventory.html

---

*Need help? [Contact support]() or check our [troubleshooting guide]().*
