---
title: "Getting Started: Create and Manage Invoices"
description: "Learn how to create full and partial invoices, capture payments online or offline, and manage invoice emails and PDFs."
type: tutorial
audience:
  - "Magento merchants"
difficulty: Beginner
estimated_time: "9 minutes"
prerequisites:
  - "Admin user with Sales > Operations > Orders and Invoices permissions"
  - "At least one configured payment method (Stores > Configuration > Sales > Payment Methods)"
  - "Sales emails enabled for Invoices (Stores > Configuration > Sales > Sales Emails)"
  - "A test order in a state eligible for invoicing (e.g., not fully invoiced)"
tags:
  - tutorial
  - high-impact
last_updated: "2025-09-10T13:54:09.753Z"
---

# Getting Started: Create and Manage Invoices

Applies to: Adobe Commerce and Magento Open Source 2.4.x

## Why This Tutorial Matters

**Business Impact:** Invoicing turns orders into cash. Accurate full or partial invoices accelerate settlement, improve cash flow, and reduce chargebacks.

**What You'll Achieve:** You will be able to create full and partial invoices, capture payment online or record it offline, email customers their invoices, and manage invoices at scale from the Invoices grid.

## Learning Journey Overview

### Your Situation
You're receiving paid orders and need to convert them into captured revenue. Some orders ship in multiple parts, and your payment provider may require you to invoice to capture funds.

### What You'll Learn
By completing this tutorial, you will:

- Create a full invoice for an order (Sales > Orders > View > Invoice)
- Create a partial invoice for selected items or quantities
- Choose Capture Online vs Capture Offline appropriately
- Email, print, and export invoices
- Find and manage invoices in Sales > Operations > Invoices
- Verify payment capture and invoice status
- Resolve common issues (Invoice button missing, capture options unavailable)

### Success Criteria
You'll know you've succeeded when:

- You create one full and one partial invoice without errors
- Order status updates appropriately and an invoice number is generated
- Payment is captured online (transaction ID present) or recorded offline
- Customer receives the invoice email (noted in order Comments History)
- The invoice appears in Sales > Operations > Invoices and can be printed (PDF)

### Time Investment
- Estimated time: 9 minutes
- Skill level after completion: Confident with day-to-day invoicing and basic troubleshooting
- Business value unlock: Faster cash collection, fewer billing errors, and improved customer communication

## Before We Start

### Who This Is For
This tutorial is designed for:

- Store Owners and General Managers
- Operations and Fulfillment Managers
- Finance/Accounts Receivable staff

### What You Need
Make sure you have:

- Admin access with permissions to Sales > Operations > Orders and Invoices
- At least one payment method configured (Stores > Configuration > Sales > Payment Methods)
- Sales emails for Invoices enabled (Stores > Configuration > Sales > Sales Emails)
- A test order that is not fully invoiced
- Cron configured and running if you rely on queued emails

### Preparation Checklist
Before starting, complete these preparation steps:

1) Confirm your payment method's Payment Action (Authorize vs Authorize and Capture).
2) Place a test order using that method.
3) Enable Invoice emails and set the correct sender and template (Stores > Configuration > Sales > Sales Emails).
4) Verify you can send emails (for example, test a password reset to confirm outbound email works).
5) Ensure you can access Sales > Operations > Invoices.

## Step-by-Step Learning Path

Step 1 — Create a full invoice
1. In Admin, go to Sales > Orders.
2. Click the order you want to invoice. The order should show an Invoice button and not be fully invoiced.
3. Click Invoice.
4. On the New Invoice page:
   - Review Items to Invoice. Keep the full quantities to invoice the entire order.
   - In the Totals section, set Capture Case:
     - Capture Online (recommended if your payment method supports online capture)
     - Capture Offline (use if you already collected payment outside the gateway)
   - Optional: In Invoice Comments, add a note and select Email Copy of Invoice to notify the customer.
5. Click Submit Invoice.
Result: An invoice number is created. The order status typically moves to Processing. If Capture Online was used, a transaction ID is recorded.

Step 2 — Create a partial invoice
1. Sales > Orders > open the order > click Invoice.
2. In Items to Invoice, adjust Qty to invoice only the items/quantities you are ready to bill.
3. Choose the appropriate Capture Case. Note: Some gateways do not allow multiple captures; use an Authorize-only payment action to enable multiple partial captures when supported.
4. Optional: Add a comment and email the invoice to the customer.
5. Click Submit Invoice.
Result: A partial invoice is created. The remaining quantities remain open for subsequent invoices.

Step 3 — Email, print, and export invoices
- Email: Open the invoice (Sales > Operations > Invoices > View) and click Send Email.
- Print: From Sales > Operations > Invoices, select one or more invoices, then Actions > Print Invoices to download a PDF.
- Export: Use the Export control (CSV or XML) on the Invoices grid for reconciliation.

Step 4 — Verify payment and invoice status
- On the order, check the Invoices tab to see your invoice(s).
- Check order Comments History for "Notified customer" when emailing.
- Go to Sales > Operations > Transactions to verify the capture transaction (when Capture Online is used).

## Practice and Reinforcement

Now that you've learned the core process, let's reinforce your skills:

- Create and email a full invoice for a test order.
- Create a partial invoice (for example, invoice only 1 of 2 items). Then create a second invoice for the remainder.
- Print both invoices as a single PDF from the Invoices grid.
- Export the Invoices grid to CSV and confirm totals.

## What You've Accomplished

🎉 Congratulations! You have successfully:

- Created full and partial invoices with correct capture settings
- Notified customers with invoice emails
- Printed and exported invoices for accounting
- Verified payment capture and invoice status

### Business Impact
Reduced days sales outstanding (DSO) by capturing payments at the right time, fewer invoice errors on partial shipments, and improved customer satisfaction via timely communications.

### Skills Gained
You now have the ability to:

- Confidently create, email, print, and export invoices
- Choose the correct capture option for your payment method
- Verify transactions and troubleshoot common invoicing issues

## Next Steps in Your Journey

Now that you've mastered this process, here's how to build on your success:

### Immediate Actions
- Review and set Payment Action for each payment method.
- Enable and brand your invoice email templates.

### Level Up Your Skills
- Automate invoice email communications with comments for preorders or backorders.

### Advanced Applications
- Align invoicing with multi-shipment (MSI) workflows and B2B partial fulfillment.
- Use Sales > Operations > Transactions to reconcile gateway captures.

## When Things Don't Go as Expected

Even experienced merchants encounter challenges. Here's how to handle common situations:

Issue: Invoice button is missing on the order
- Cause: The order is already fully invoiced or the payment method used Authorize and Capture (auto-invoiced).
- Fix: Check the Invoices tab on the order. For new invoices, use an order that is not fully invoiced.

Issue: Capture Online option is not available
- Cause: The payment method does not support online capture or is configured for Authorize and Capture (already captured).
- Fix: Switch the method to Authorize (if supported) or use Capture Offline when you have collected payment externally.

Issue: Customer did not receive the invoice email
- Cause: Invoice emails disabled or email queue/crons misconfigured.
- Fix: Stores > Configuration > Sales > Sales Emails > Invoice = Enabled. Ensure cron is running. Resend from the invoice page.

Issue: Cannot create a second partial invoice
- Cause: Your gateway may not allow multiple captures.
- Fix: Consult your payment provider documentation. Consider Authorize-only with multiple captures if supported.

Issue: Order state is Pending Payment
- Cause: Asynchronous payment (for example, some PayPal flows) not confirmed yet.
- Fix: Wait for the payment notification or manually capture in the gateway if applicable.

## Continue Learning

### Related Tutorials
- Manage Orders
- Create Shipments
- Create Credit Memos (Refunds)

### How-To Guides
- Configure Payment Methods
- Configure Sales Emails

### Reference Materials
- Invoices grid fields
- Transactions view
- Order status and state mapping

## Summary

You can now convert orders into invoices, capture payments appropriately, and manage invoice communications and documents from a single place. These skills help you accelerate cash flow and reduce billing errors.

### Key Takeaways
- Use Capture Online when your gateway supports it; use Capture Offline for external payments
- Partial invoices align billing with partial shipments and backorders
- Always verify transactions and email notifications

### Remember
- Keep payment actions aligned with your fulfillment strategy
- Test invoicing workflows in a staging environment before going live

---

*Found this tutorial helpful? [Share your feedback]() or [suggest improvements]().*

*Questions or stuck? [Get help from the community]() or [contact support]().*
