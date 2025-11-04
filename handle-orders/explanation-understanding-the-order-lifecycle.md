---
title: "Understanding the Order Lifecycle"
description: "Conceptual explanation of how orders flow through the system and business processes"
type: explanation
audience: "Magento merchants"
tags:
  - explanation
  - orders
  - complex
last_updated: "2025-09-11T07:56:55.208Z"
---

# Understanding the Order Lifecycle

## Introduction

This guide explains each stage of an order in Magento—from cart and payment, through invoicing and shipment, to refunds—so you can speed fulfillment, reduce errors, and improve cash flow.

In Magento, an order is marked Complete when all items are invoiced and shipped.

Exceptions:
- Orders with only non‑shippable items (virtual, downloadable, and virtual/combined e‑gift cards) complete after invoicing because no shipment is required. Note: Gift Card product type (physical, virtual, combined) is available in Adobe Commerce only.
- Physical gift cards are shippable and require a shipment.
- Mixed carts complete when all shippable items are shipped and all items are invoiced.

Scope & Version Notes (2.4.x): Applies to Magento Open Source and Adobe Commerce 2.4.x. Inventory Management (MSI) is included by default; reservations are used. Adobe Commerce B2B approval workflows are out of scope.

## Prerequisites

- Admin access with permissions to view and manage Orders, Invoices, Shipments, and Credit Memos.
  Admin path: System > Permissions > User Roles.
- At least one active payment method configured.
  Admin path: Stores > Configuration > Sales > Payment Methods.
- At least one shipping method configured and Shipping Origin set.
  Admin paths: Stores > Configuration > Sales > Delivery Methods; Stores > Configuration > Sales > Shipping Settings > Origin.
- For asynchronous gateways (for example, PayPal), ensure the webhook/IPN endpoint is publicly reachable over HTTPS (no maintenance mode, correct base URL/domain, and firewall/IP allowlists updated so gateway systems can connect).
- Ensure Magento cron is configured and running for your environment (required for sales emails, some payment updates, and grid/index refresh). Verify via CLI: bin/magento cron:run and check var/log for errors. Adobe Commerce: System > Support > System Reports can help validate environment health.
  Cron verification:
  • Ensure Magento cron is installed: run "bin/magento cron:install" (once per environment).
  • Check system crontab: run "crontab -l" (look for Magento cron jobs).
  • Verify last runs: check var/log/magento.cron.log (if enabled) and var/log/system.log for cron activity.
  • On Adobe Commerce: System > Support > System Reports to review cron status.

## Key Concepts

- Quote vs Order: A shopping cart becomes a quote in Magento. When the customer submits checkout, the quote converts to an order record.
  Admin path: Sales > Orders.

- Order State vs Status:
  - Definition: State is the system-controlled stage; Status is the merchant-facing label mapped to a state.
  - Admin path: Stores > Settings > Order Status.
  - Default mapping (typical):
    • new → Pending
    • pending_payment → Pending Payment
    • processing → Processing
    • holded → On Hold
    • payment_review → Payment Review (gateway-specific; for example, PayPal can assign the ‘Suspected Fraud’ status under this state)
    • complete → Complete
    • closed → Closed
    • canceled → Canceled
  - Notes: Magento controls the order state (system workflow). You control the order status (labels and communication) mapped to each state. Status-to-state mappings can be customized (Stores > Settings > Order Status). Gateways may assign different default statuses within the same state (for example, 'Suspected Fraud' under payment_review).

- Invoice: The record that items have been billed. When created with an online capture (supported gateways), funds are captured; with offline methods, the invoice records payment received without contacting the gateway.
  - On the Invoice page, if your payment method supports it, set Capture to "Capture Online" to capture funds via the gateway, or "Capture Offline" to record payment received outside Magento. If the Capture option is not shown, the payment method controls capture behavior.
  Admin path: Sales > Orders > open order > Invoice.

- Shipment: The record that items were shipped. In MSI, shipment allocates from specific Sources.
  Admin path: Sales > Orders > open order > Ship.
  Configuration: Stores > Configuration > Catalog > Inventory > Source Selection Algorithm.

- Credit Memo: The record of refunds/returns.
  Admin path: Sales > Orders > open order > Invoices tab > open invoice > Credit Memo.

- Payment Actions: Authorize vs Authorize & Capture. This determines when funds are captured and whether invoices are auto-created.
  Admin path: Stores > Configuration > Sales > Payment Methods.
  Operational tip: Most gateways expire authorizations in 7–30 days. If the authorization expires before you invoice, request a new payment (re‑authorize) or contact the customer. Consider using vault/tokenization to simplify re‑capture where supported.

- Hold/Unhold and Cancel: Hold pauses further actions; Unhold resumes. Cancel voids an order that has no invoices/shipments.
  Admin path: Sales > Orders > open order > Actions (Hold/Unhold/Cancel).

- MSI Reservations (2.3+): When the order is placed, Magento creates reservations that reduce Salable Qty. On shipment, cancellation, or credit memo with Return to Stock, compensating reservations adjust Salable Qty accordingly.

## How It Works

At a glance: Cart → Order (state: new; status: Pending) → Payment authorized and/or captured (state can become processing; an invoice may be auto‑created if captured) → Shipment(s) → Completion (state: complete; status: Complete).

Note: Some payment methods set the order to Processing on successful authorization even if an invoice is not created automatically. Default core behavior: Processing is most commonly set when an invoice and/or shipment is created. Some gateways may set Processing immediately after authorization—check your gateway’s documentation.

1) Cart and Checkout → Order Created
- The customer checks out. Magento converts the quote to an order (state: new; status: typically Pending).
- If the payment gateway processes asynchronously, the status may move to Pending Payment or Payment Review until the gateway confirms.

2) Payment Authorization / Capture
- Authorize & Capture: If the gateway captures immediately, Magento auto-creates an invoice and sets the order to Processing.
- Authorize Only: Funds are authorized, but no invoice is created. Depending on the payment method, the order may move to Processing on successful authorization or remain in New/Pending Payment until you invoice.
- Partial invoices: On the Invoice page, adjust Qty per line to invoice only what you are ready to capture/ship. Remaining items can be invoiced later.
- Caution: Multiple partial invoices (captures) are supported only if your payment gateway allows multiple captures against a single authorization. Confirm gateway capabilities and settings; otherwise, use a single invoice or re‑authorize for the remaining amount.

State cheat sheet:
- New: Order just placed.
- Pending Payment: Awaiting gateway confirmation or offline payment.
- Processing: At least one invoice and/or shipment exists.
- Complete: All items invoiced and all shippable items shipped.
- Closed: All invoiced amounts fully refunded.
- Canceled: No invoices/shipments; order voided.

3) Fulfillment (Shipment)
- Magento allows you to create a shipment even if no invoice exists. Best practice: invoice (capture payment) before shipping to ensure items are paid.
- Partial shipments are supported. In MSI, select quantities per Source during shipment.
- On the Ship page, click Add Tracking Number to record carrier and tracking. Check Email Copy of Shipment to notify the customer.
- Shipment emails: Ensure Stores > Configuration > Sales > Sales Emails > Shipment is enabled. On the Ship page, select Email Copy of Shipment to send immediately. Verify under the order’s Comments History that the shipment email was queued/sent.

4) Order Completion
- When all items on the order are invoiced and shipped, Magento sets the order state to complete (status: Complete).

5) Refunds/Returns (Credit Memo)
- To refund, go to Sales > Orders > open order > Invoices tab > open invoice > Credit Memo.
- Use Refund for online refunds (supported gateways) or Refund Offline if your gateway doesn’t support online refunds. Orders move to the Closed state when the total invoiced amount has been fully refunded (no remaining balance). Partial refunds keep the order in its prior state (Processing or Complete).
- Partial refunds: On the Credit Memo page, set Qty to refund per item and optionally enter Adjustment Refund (to add an additional amount) or Adjustment Fee (to deduct fees). Use Return to Stock per item if you are restocking.
- Shipping/tax refunds: Select Refund Shipping if you are refunding shipping charges; tax is calculated automatically based on refunded items and shipping selection. Choose Refund (online) to send the refund to the gateway, or Refund Offline to record it after processing the refund in your payment provider’s portal.
- Verification: After submitting an online refund, check the order’s Comments History and the Transactions tab for a successful refund transaction ID. If missing, confirm with your PSP.

6) Holds and Cancellations
- Hold an order to prevent changes (Action: Hold). Unhold to resume processing.
- Cancel is available only if no invoices/shipments exist (typically in Pending/Pending Payment). Cancel releases MSI reservations.
- Partial cancellation: If some items are already invoiced or shipped, you cannot cancel those items. Click Actions > Cancel to cancel all remaining uninvoiced/unshipped quantities. Magento automatically cancels only the cancelable quantities. There is no per-line cancel quantity input in the default Admin; for already invoiced items, issue a credit memo instead.

Flow summaries within this lifecycle:
- Flow A: Authorize & Capture
  • Order placed (New) → Gateway captures → Invoice auto-created → State: Processing → Ship one or more shipments → State: Complete after all items shipped and invoiced.
  • Who acts next: Warehouse creates Shipment(s); Finance monitors for refunds if needed.
  • Verify: Sales > Invoices shows invoice; Sales > Transactions shows capture; Sales > Shipments shows shipment(s).
- Flow B: Authorize Only
  • Order placed (New/Pending Payment or Processing depending on gateway) → Manual Invoice to capture funds → State: Processing → Ship → State: Complete after all items shipped and invoiced.
  • Who acts next: Finance creates Invoice (capture) when ready; Warehouse ships after invoice.
  • Verify: Sales > Transactions shows authorization then capture on invoicing; Sales > Invoices/Shipments reflect documents.

Verification after each step:
- Order state/status: Check the header of the order view.
- Documents: Use the Invoices, Shipments, and Credit Memos tabs and the Sales > Invoices / Sales > Shipments grids (filter by Order #) to confirm creation, totals, and timestamps.
- Payments: Review Sales > Transactions for authorization/capture/refund IDs and cross-check the transaction status in your payment provider’s portal.

### Edge Cases
- Virtual/Downloadable/Virtual or Combined Gift Cards only: No shipment is created. Physical Gift Cards require a shipment.
- Mixed carts: Shippable items must be shipped; non‑shippable items require only invoicing. The order completes when all shippable items are shipped and all items are invoiced.
- Multi-address checkout: Each address produces a separate order with its own invoices, shipments, and refunds.
- Payment Review: For methods like PayPal, orders may enter the payment_review state. From the order view, use Accept Payment or Deny Payment (if available) after verifying risk checks in your gateway.

## Benefits and Advantages

- Faster cash flow: For low‑risk, in‑stock SKUs, configure Authorize & Capture to auto‑invoice on order placement and reduce DSO.
- Lower fulfillment errors: Require invoice before shipment so only paid items are picked and packed.
- Reduced cancellations and oversells: MSI reservations hold stock at order time; cancel/credit promptly releases or compensates stock.
- Better customer communication: Enable sales emails for Order, Invoice, Shipment, and Credit Memo to set expectations and reduce support volume.
- Operational focus: Create saved views in Sales > Orders to drive daily triage.
  • Processing queue: Filter Status = Processing; add columns Payment Method and Shipments; sort by Purchase Date ascending. Save the view: click Filters, apply filters, open the Views dropdown, click Save View, name it (for example, "Processing"), then click the star to set it as default.
  • Pending Payment queue: Filter Status = Pending Payment; add column Payment Method; optionally filter by Payment Method for specific gateways. Save as "Pending Payment".
  • Payment Review queue: Filter Status = Payment Review; add column Risk (if surfaced via gateway status in comments) and Payment Method. Save as "Payment Review".
  Implementing these queues reduces time-to-action and helps prevent aged authorizations.

## Common Challenges and Considerations

### Troubleshooting

Payments
- Orders stuck in Pending Payment: Check gateway webhooks/IPN and payment method config (Stores > Configuration > Sales > Payment Methods). Confirm currency configuration under Stores > Configuration > General > Currency Setup (Base, Default, Display) and verify Stores > Currency > Currency Rates are up to date. Also check the order's Comments History and Transactions tab for gateway callback notes or missing transaction IDs before reviewing logs. Review var/log/system.log and var/log/exception.log. If your gateway creates its own log, check var/log for files that include the gateway name (for example, "adyen_*.log", "braintree_*.log"), or consult the gateway’s documentation. If sandboxing, verify callback URL and credentials. Also check Sales > Transactions for the order’s authorization/capture records. Ensure cron is running; stalled crons can delay email sending and certain asynchronous operations.
- Double capture risk: If your gateway auto-invoices, do not also manually invoice. Confirm each payment method’s Payment Action and any auto-invoice behavior.
- Void vs Refund: If payment is only authorized (not captured), use Void (if supported by your payment method). The Void action appears on the Order or Invoice view depending on the integration. If already captured (invoiced), issue a Credit Memo (Refund) instead.
- Sandbox webhook/IPN testing: Ensure your callback URL is publicly reachable (use a tunneling tool if needed), confirm the exact endpoint configured in the gateway, and verify successful callbacks in gateway logs and Magento order comments/history.

Shipping / Inventory (MSI)
- Cannot Ship: Magento allows shipment before invoicing. If the Ship action/button is missing, ensure the order contains shippable items (not virtual/downloadable only), verify MSI source availability for the items, and confirm user permissions.
- Partial shipment with backorders: Allow backorders (Stores > Configuration > Catalog > Inventory > Product Stock Options > Backorders). Ship available quantity now, remainder later.
- Per-product backorders and salable quantity checks: Catalog > Products > open product > Advanced Inventory (or Sources) > enable Backorders for that SKU if needed. Check salable quantity in Catalog > Products grid (Salable Quantity column) and ensure sufficient source quantity is assigned for the shipping source.

Refunds
- Refund button missing: Your gateway may not support online refunds. Use Refund Offline and process the refund in your PSP dashboard.

Permissions
- Action buttons missing: Check System > Permissions > User Roles to ensure the role has Sales > Operations (Orders, Invoices, Shipments, Credit Memos) privileges.

### Preventive Controls

- Choose the right Payment Action per payment method based on the risk profile of the products you plan to accept with that method (for example, use a method set to Authorize Only for high‑fraud or long lead‑time items, and a method set to Authorize & Capture for low‑risk, in‑stock items). If you need different actions by product group, use separate payment methods or websites/store views.
- Document each gateway’s capture/void/refund behavior to avoid double captures; train staff on when to invoice vs ship.
- Monitor webhook/IPN health: set alerts on failed callbacks, keep gateway IP allowlists current, and ensure your base URL and HTTPS configuration are correct.
- Keep currency rates updated: Stores > Currency > Currency Rates, and optionally configure scheduled imports under Stores > Configuration > General > Currency Setup > Scheduled Import Settings.
- Test order flows regularly in a staging/sandbox environment before deploying changes to payment, shipping, or inventory configurations.
- Set a daily report for orders in Pending Payment > 1 hour (Sales > Orders grid filter + export), and alert if count exceeds baseline. Review your gateway dashboard for failed callbacks within the same period.
- Adobe Commerce: Use RMA for returns authorization and tracking (Stores > Settings > Configuration > Sales > RMA). Credit Memo processes the refund; RMA manages the return workflow.

## Use Cases and Scenarios

- Low-risk, in-stock consumer goods: Use Authorize & Capture to auto-invoice on placement, auto-send invoice email, and pick/pack the same day.
  • Checklist: Payment Action = Authorize & Capture; Sales Emails (Order/Invoice/Shipment) = Enabled; Picking policy = pick only items with an invoice; Monitor queue = Processing.
- Preorders or make-to-order: Use Authorize Only at checkout; capture (create invoice) when the item is ready to ship. Mind authorization hold durations (often 7–30 days) to avoid expired authorizations.
  • Checklist: Payment Action = Authorize Only; Sales Emails (Order) = Enabled, (Invoice) = Enabled for capture moment; Set reminder to capture before auth expiry; Picking policy = ship only after invoice.
- High-fraud categories: Enable 3D Secure/AVS at the gateway, set Payment Action to Authorize Only, use Payment Review or manual review, then invoice only approved orders.
  • Checklist: Payment Action = Authorize Only; Gateway risk tools = On (3DS/AVS/CVV); Sales Emails = hold Shipment email until shipped; Queue = Payment Review for manual action; Void unauthorized/denied payments when appropriate.

## Alternatives and Comparisons

- Authorize & Capture
  - Pros: Faster cash flow, fewer manual steps, auto-invoice.
  - Cons: If items later go out of stock or change, you may need refunds.
  - Best for: In-stock, low-fraud SKUs and fast-moving catalogs.
- Authorize Only
  - Pros: Control over capture timing, easier order changes/cancellation before capture.
  - Cons: Authorization may expire; slower cash flow.
  - Best for: Preorders, backorders, custom builds, or high-fraud categories.
- Manual vs Auto Invoicing
  - Use auto when the gateway reliably captures at order placement.
  - Use manual to align capture with allocation/picking and to avoid double capture.
- Single-source vs Multi-source (MSI)
  - Single-source is simpler but less flexible for multi-location fulfillment.
  - Multi-source improves allocation by location and reduces split shipments; requires Source Selection at shipment.

Decision guide:
- Is the item in stock and low fraud risk? Use Authorize & Capture to accelerate cash collection.
- Is lead time uncertain (preorder/backorder/custom)? Use Authorize Only and capture when ready to ship; monitor auth expiry.
- Is fraud risk high or review needed? Use Authorize Only with Payment Review; capture only after approval.
- Do you need to avoid split shipments or coordinate allocation? Favor manual invoicing aligned to picking/packing.

## Business Impact

### For Merchants

- KPIs: Days Sales Outstanding (DSO), Authorization-to-Capture Conversion Rate, Refund Rate, Chargeback Rate.
- Targets: Auth→Capture > 98% for in‑stock items; Avg time New→Processing < 15 min (synchronous) or < 2h (asynchronous). Refund Rate < 3% for shipped orders. Chargeback Rate < 0.5%.
- Map states to SLAs: For example, orders should move from Pending/Pending Payment to Processing within X minutes when payments are synchronous.

### For Customers

- KPIs: Time to Ship from Processing, On-time Delivery %, Proactive Email Notifications Sent.
- Clear status emails reduce WISMO (“where is my order?”) contacts and improve satisfaction.

### For Operations

- KPIs: Pick/Pack SLA from Processing, Partial Shipment Frequency, Orders stuck >24h in Pending Payment.
- Use state/status dashboards to prioritize Processing orders for fulfillment.

## Common Misconceptions

- “Processing means shipped” → Processing indicates the order is being worked: it typically means at least one invoice and/or shipment exists. Best practice is to invoice before shipping, but Magento can enter Processing if a shipment is created first.
- “Complete means delivered” → Complete means Magento recorded all items as invoiced and shipped; delivery happens after shipment with the carrier.
- “You can cancel any order” → After invoicing/shipping, use a credit memo; Cancel is only for orders with no invoices/shipments.
- “Credit memo always returns stock” → Stock increases only if you select Return to Stock for the refunded items.

## Implementation Considerations

- Payment Action: Stores > Configuration > Sales > Payment Methods > select your method > Payment Action (Authorize Only vs Authorize & Capture).
- Sales Emails: Stores > Configuration > Sales > Sales Emails (Order, Invoice, Shipment, Credit Memo notifications).
- Order Status Management: Stores > Settings > Order Status (create custom statuses and assign to states).
- Inventory and Backorders: Stores > Configuration > Catalog > Inventory > Product Stock Options (Backorders, Out-of-Stock Threshold). Per-product overrides under Catalog > Products.
- MSI Sources and Stocks (2.3+): Configure Sources and Stocks under Stores > Inventory. At shipment, select source quantities.
- Source Selection Algorithm (MSI): Stores > Configuration > Catalog > Inventory > Source Selection Algorithm — choose how sources are suggested during shipment (for example, Priority, Distance Priority if configured).
- Permissions: System > Permissions > User Roles (control who can invoice, ship, and issue credit memos).

## Related Concepts

- Orders (Admin)
- Invoices (Admin)
- Shipments (Admin)
- Credit Memos (Admin)
- Order Status and State
- Inventory Management (MSI) Basics
- Payment Methods configuration
