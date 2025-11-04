---
title: "Complete Order Management Workflow"
description: "End-to-end tutorial on processing orders from placement to delivery"
type: tutorial
audience:
  - "Magento merchants"
difficulty: Advanced
estimated_time: "20–30 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
tags:
  - orders
  - fulfillment
  - invoicing
  - shipping
  - refunds
  - payments
  - inventory
  - msi
  - high-impact
last_updated: "2025-09-11T07:43:22.002Z"
---

# Complete Order Management Workflow

## Why This Tutorial Matters

Business Impact:
- Streamline order-to-cash, accelerate payment capture, and ship on time to improve customer satisfaction.
- Reduce cancellations, chargebacks, and inventory discrepancies through accurate invoicing, shipping, and refunds.

What You'll Achieve:
- Confidently process a test order end-to-end: review the order, create an invoice, create a shipment with tracking, and issue a partial refund via credit memo—while keeping inventory accurate.

Compatibility:
- Magento Open Source and Adobe Commerce 2.4.x. MSI (Multi-Source Inventory) is enabled by default. RMA features are Adobe Commerce only and not covered here.
- With MSI (Magento 2.3+), stock is reserved at order placement and deducted when you create a shipment. The legacy “Decrease Stock When Order is Placed” setting does not apply with MSI enabled.

## Learning Journey Overview

### Your Situation
You manage daily orders and need a reliable, repeatable process to capture payments, ship accurately, handle exceptions, and keep inventory correct—without slowing down your team or confusing customers.

### What You'll Learn
By completing this tutorial, you will:

- Navigate Sales > Orders and interpret order states vs. statuses
- Create invoices (capture online/offline) and partial invoices
- Create shipments with tracking; perform partial shipments (MSI-aware)
- Issue refunds via Credit Memo (full and partial) and restock returned items
- Handle exceptions: hold, cancel, payment review, and virtual orders
- Verify customer email notifications and print documents
- Understand inventory reservations (MSI) and salable quantity

### Success Criteria
You'll know you've succeeded when:

- The order status progresses from Pending to Processing to Complete (for physical goods)
- The customer receives Order, Invoice, and Shipment emails
- A partial Credit Memo is issued and inventory is restocked for returned items
- Order and item quantities correctly reflect invoiced, shipped, and refunded values
- Operational targets: Processing orders aged >24h are <5% of total; refund turnaround <48h

### Time Investment
- Estimated time: 20–30 minutes
- Skill level after completion: Confident with day-to-day order operations and exception handling
- Business value unlock: Faster capture-to-cash, fewer support tickets, and reduced inventory discrepancies

## Before We Start

### Who This Is For
This tutorial is designed for:

- Store owners and operations managers responsible for order fulfillment
- Customer service agents handling invoicing, shipping, and refunds
- New staff onboarding to Sales > Orders workflows

### What You Need
Make sure you have:

- Admin access with role permissions: Sales > Operations > Orders, Invoices, Shipments, Credit Memos
- A staging store (recommended) with email sending configured (Stores > Configuration > Sales > Sales Emails) or disabled for testing
- At least one enabled payment method for testing (e.g., Check/Money Order) and one shipping method (e.g., Flat Rate)
- One in-stock simple product assigned to a source/stock (MSI) with positive salable quantity

### Preparation Checklist
Before starting, complete these preparation steps:

- Enable Check/Money Order: Stores > Configuration > Sales > Payment Methods > Check/Money Order = Yes
- Enable Flat Rate shipping: Stores > Configuration > Sales > Shipping Methods > Flat Rate = Enabled
- Verify Sales Emails: Stores > Configuration > Sales > Sales Emails > enable Order, Invoice, Shipment, and Credit Memo emails
- Confirm product stock and MSI: Catalog > Products > ensure product is In Stock and Salable Qty > 0; assign to a Source if using MSI
- Ensure cron is running for emails and order processing automation

## Step-by-Step Learning Path

### 1) Place a test order
- Goal: Create a new order to process end-to-end.
- Path: Storefront checkout using your test payment and shipping methods.
- Actions: Add the test product to cart, proceed through checkout, place order.
- Verify: In Admin, go to Sales > Orders and confirm the new order appears. Status is typically Pending (or Processing if your gateway auto-invoices).

### 2) Review the order
- Goal: Validate payment, shipping, and items before invoicing.
- Path: Admin > Sales > Orders > click the order.
- Actions: Check Payment & Shipping Method, Items Ordered, customer email, and Order Comments.
- Verify: Details are accurate; no discrepancies that require Hold or Cancel.

### 3) Create an invoice (capture payment)
- Goal: Capture funds and record revenue for billable items.
- Path: In the order view, click Invoice.
- Actions: Optionally adjust quantities for a partial invoice. Choose Capture Online (if supported by the gateway) or Capture Offline. Click Submit Invoice and Send Email.
- Verify: Order status typically moves to Processing; Invoices tab lists the invoice; the customer receives an invoice email.

### 4) Create a shipment (fulfillment)
- Goal: Deduct inventory, create a shipment, and notify the customer.
- Path: In the order view, click Ship.
- Actions:
  - If MSI is enabled: select Source(s) and quantities per item. Use Source Selection recommendations if available.
  - Add Carrier and Tracking Number(s) as applicable.
  - Check Send Shipment Email and click Submit Shipment.
- Verify: Shipments tab lists the shipment with tracking; the customer receives a shipment email. For non-virtual items, once all items are invoiced and shipped, the order status becomes Complete.

### 5) Print documents (optional)
- Goal: Generate documents for packing or record-keeping.
- Path: From the order view or Sales > Orders grid.
- Actions: Use Actions to Print Invoices or Print Packing Slips.
- Verify: Documents download or print successfully.

### 6) Handle exceptions: Cancel or Hold
- Goal: Pause or stop processing when needed.
- Path: Order view.
- Actions: Before invoicing/fulfillment, click Cancel to void the order or Hold to pause. Use Unhold to resume when ready.
- Verify: Order status updates to Canceled or On Hold accordingly.

### 7) Issue a refund (Credit Memo)
- Goal: Process returns or adjustments and restock items.
- Path: From the order view (or an invoice), click Credit Memo.
- Actions: Adjust quantities, select Return to Stock where appropriate. Choose Refund Online (if supported) or Refund Offline. Add a comment and send email.
- Verify: Credit Memos tab lists the refund; inventory increases for returned items; customer receives credit memo email.

Notes:
- Virtual/Downloadable products: No Shipment step; order can Complete after invoicing.
- Payment Review: If present, complete your gateway’s review/accept/deny flow before invoicing.
- Auto-invoicing: Some gateways create invoices automatically upon capture.

## Practice and Reinforcement

Now that you've learned the core process, let's reinforce your skills:

- Perform a partial shipment (split an order across two shipments)
- Create a partial invoice for one line item and complete it later
- Issue a partial refund for one item and restock it
- Create a Saved View in Sales > Orders to filter Processing orders older than 48 hours

## What You've Accomplished

🎉 Congratulations! You have successfully:

- Reviewed a new order and verified customer and payment details
- Created an invoice and captured payment
- Created a shipment with tracking and sent notifications
- Issued a partial refund via Credit Memo and restocked returned items
- Verified status transitions and document records (Invoices/Shipments/Credit Memos)

### Business Impact
- Expected impact: Reduce time-to-capture by 30–50%, decrease order handling errors, and improve on-time shipment rate.
- Track metrics: Processing orders aging, capture-to-ship lead time, refund turnaround, and cancellation rate.

### Skills Gained
You now have the ability to:

- Operate end-to-end order processing confidently in Sales > Orders
- Capture payments correctly (online/offline) and manage invoices
- Ship with tracking and manage partial shipments under MSI
- Process refunds via Credit Memo and restock inventory
- Troubleshoot common issues that block invoicing or shipping

## Next Steps in Your Journey

Now that you've mastered this process, here's how to build on your success:

### Immediate Actions
- Create Saved Views for Pending, Processing, and Backorder queues; enable shipment and invoice emails if not already active

### Level Up Your Skills
- Configure email templates for brand consistency
- Set up user roles with least privilege for CS agents to minimize errors

### Advanced Applications
- Adobe Commerce: Explore Purchase Orders and Approval workflows
- Integrate shipping carriers for live rates and label printing

## When Things Don't Go as Expected

Even experienced merchants encounter challenges. Here's how to handle common situations:

- Symptom: Invoice button is missing
  - Likely cause: Payment method auto-invoices, or the order is in Payment Review
  - Fix: Check the Invoices tab and your payment method settings; resolve Payment Review before invoicing

- Symptom: Ship button is disabled
  - Likely cause: Items are virtual/downloadable or there is no shippable quantity
  - Fix: Verify product type; for MSI, confirm salable quantity and source assignment; ensure items are invoiced when required by your flow

- Symptom: Cannot create shipment due to insufficient quantity
  - Likely cause: Reservations (MSI) or incorrect source selection
  - Fix: Check Salable Qty and Source assignments; adjust stock or choose a different source; reattempt shipment

- Symptom: Emails not sent
  - Likely cause: Sales Emails disabled or cron not running
  - Fix: Verify Stores > Configuration > Sales > Sales Emails; ensure cron is running; check email queues/logs

- Symptom: Cannot cancel an order
  - Likely cause: Order already invoiced or shipped
  - Fix: Use Credit Memo to refund instead of canceling

- Symptom: Refund Online option unavailable
  - Likely cause: Payment method does not support online refunds
  - Fix: Use Refund Offline and process the refund in your payment gateway if needed

## Continue Learning

### Related Tutorials
- Processing Partial Shipments
- Managing Credit Memos and Returns

### How-To Guides
- Create an Invoice
- Create a Shipment
- Issue a Credit Memo
- Configure Sales Emails

### Reference Materials
- Order States and Statuses
- MSI Overview
- Payment Methods configuration

## Summary

You’ve learned a complete, MSI-aware order management workflow—reviewing orders, invoicing, shipping with tracking, and issuing refunds—while keeping inventory accurate and customers informed.

### Key Takeaways
- Invoice to capture payment; ship to fulfill and deduct stock (with MSI)
- Use Credit Memos for refunds and restocking
- Monitor Processing queues and email notifications to maintain throughput

### Remember
- With MSI, stock is reserved at order placement and deducted at shipment—not at order creation
- Clear Saved Views and role-based permissions reduce errors and speed up operations
