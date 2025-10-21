---
title: "Streamlining Store Operations"
description: "Complete guide to optimizing workflows, automation, and operational efficiency"
type: tutorial
audience:
  - "Magento merchants"
difficulty: Intermediate
estimated_time: "9 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
tags:
  - tutorial
last_updated: "2025-09-15T10:11:08.759Z"
---

# Streamlining Store Operations

Scope: Applies to Magento Open Source and Adobe Commerce 2.4.x (Admin-only features; no extensions required).

Note: This tutorial uses only features available in Magento Open Source 2.4.x. If you use Adobe Commerce, you can also leverage B2B and advanced features (not covered here).

## Why This Tutorial Matters

**Business Impact:** Reduce order handling time, prevent stockouts, and standardize fulfillment to improve customer satisfaction and lower cost per order.

**What You'll Achieve:**
- Cut daily order processing time by 20–40% using saved grid views and mass actions
- Reduce out-of-stock incidents with inventory thresholds and weekly Low Stock reviews
- Standardize invoicing and shipment emails to improve customer communication

## Learning Journey Overview

### Your Situation
You process 10–500 orders/day, manage a growing catalog, and need faster picking/packing, fewer manual updates, and consistent customer notifications—without adding headcount or custom code.

### What You'll Learn
By completing this tutorial, you will:

- Create and use saved Admin grid views for Orders and Products
- Perform common mass actions efficiently on orders and products
- Configure Sales Emails for orders, invoices, and shipments
- Set inventory thresholds and review Low Stock reports
- Structure inventory using Sources and Stocks (MSI)
- Update product attributes in bulk safely
- Optimize indexers for performance with Update by Schedule

### Success Criteria
You'll know you've succeeded when:
- You have at least two saved views on Sales > Orders and Catalog > Products
- New orders trigger Order and Shipment emails automatically
- Low Stock report is empty or reviewed weekly; no unexpected stockouts in the next 30 days
- Indexers are set to Update by Schedule and all are in Ready status

### Time Investment
- Estimated time: 30–45 minutes for first-time setup
- Skill level after completion: Confident day-to-day operator with efficient order and inventory workflows
- Business value unlock: Faster fulfillment, fewer errors, improved customer communications—no extensions required
- Note: Ongoing usage typically adds ~5–10 minutes per week for reviews and adjustments

## Before We Start

### Who This Is For
This tutorial is designed for:

- Store owners and operations managers
- Fulfillment leads and warehouse supervisors
- Catalog managers responsible for product updates

### What You Need
Make sure you have:

- Magento Admin login with permissions to Sales, Catalog, Stores > Configuration, and System tools
- Outgoing email configured by your host or SMTP provider (to send transactional emails)
- If using multiple locations, a simple plan for your inventory Sources and Stocks (MSI)
- Optional: A staging site to test configuration changes before production

### Preparation Checklist
Before starting, complete these preparation steps:
1) Confirm email sender identities: Stores > Configuration > General > Store Email Addresses
2) Verify you can send emails: Stores > Configuration > Advanced > System > Mail Sending Settings (or confirm SMTP with your host)
3) Review Indexers: System > Tools > Index Management (note current mode)
4) Export or screenshot key configuration settings as a lightweight backup

## Step-by-Step Learning Path

### Orders & Fulfillment

1) Create saved Orders grid views
- Go to Sales > Orders.
- Click Filters, set Status = Pending, set Purchase Date to Today (or as needed).
- Click Columns and enable the columns your team needs (e.g., Grand Total, Shipping Address, Shipping Method).
- In the Views selector (top right), click Save Current View and name it "New Orders Today".
- Repeat to create an "Awaiting Shipment" view (e.g., Status = Processing, no Shipment created).

2) Use mass actions to batch process safely
- Go to Sales > Orders and select your "New Orders Today" view.
- Select multiple orders (check the boxes), open Actions and choose safe batch operations like Print Invoices or Print Packingslips.
- Tip: If the Invoice action is unavailable for an order, the payment method may auto-capture; open the order and check the Invoices tab.

### Customer Communications

3) Configure Sales Emails
- Go to Stores > Configuration > Sales > Sales Emails.
- Ensure Order, Invoice, Shipment, and Credit Memo emails are Enabled.
- Choose appropriate Email Sender identities (configured under Stores > Configuration > General > Store Email Addresses).
- Save Config.
- Test by placing a small storefront order and confirming that Order and Shipment emails arrive.

### Inventory & MSI

4) Set low stock thresholds and review reports
- Go to Stores > Configuration > Catalog > Inventory > Product Stock Options.
- Set Notify for Quantity Below to a sensible buffer (e.g., 5–10, depending on sales velocity) and Save Config.
- Weekly: Go to Reports > Products > Low Stock, review and restock items before they sell out.

5) Define Sources and Stocks (for multi-location)
- Go to Stores > Inventory > Sources and Add New Source for each physical location (enable and fill address/contact details).
- Go to Stores > Inventory > Stocks and either:
  - Use the default Stock for single-location, or
  - Create a new Stock, assign your website, and add the relevant Sources.
- When fulfilling orders, Magento's Source Selection Algorithm (SSA) will suggest optimal source(s) based on your configuration.

### Catalog Efficiency

6) Bulk update product attributes
- Go to Catalog > Products.
- Filter/select the products to change.
- In Actions, choose Update Attributes.
- Adjust fields (e.g., Tax Class, Visibility, or attribute values) and click Save.

7) Create a "Out of Stock" product view
- Go to Catalog > Products.
- Click Filters, set Stock Status = Out of Stock (and add any other useful filters like Visibility or Website).
- Adjust Columns to include Qty, Stock Status, Status, and Visibility.
- Save the view as "Needs Restock" to quickly identify products requiring attention.

### Performance

8) Optimize indexers for stability and speed
- Go to System > Tools > Index Management.
- Select all indexers, choose Actions = Set to Update by Schedule, and Submit.
- Confirm all indexers show Ready status after the next cron run. If not, verify the server cron is running.

## Practice and Reinforcement

Now that you've learned the core process, let's reinforce your skills:
- Create two additional Orders views: "Awaiting Payment" (Status = Pending) and "Shipped Today" (Shipment Date = Today).
- Bulk update 10 products to the same Tax Class and confirm the change via Catalog > Products.
- Trigger a test order on the storefront and verify Order and Shipment emails are received.

## What You've Accomplished

🎉 **Congratulations!** You have successfully:

- Created saved views for Orders and Products to accelerate daily work
- Standardized transactional emails for key order events
- Implemented inventory thresholds and review habits to prevent stockouts
- Organized inventory across one or more locations and enabled efficient picking
- Optimized indexers for balanced performance

### Business Impact
- 20–40% reduction in daily order handling time via saved views and mass actions
- Fewer out-of-stock incidents through proactive monitoring
- Improved customer satisfaction from timely, consistent emails

### Skills Gained
You now have the ability to:
- Create and manage Admin grid views
- Execute bulk order and product operations safely
- Configure and test transactional emails
- Structure inventory with MSI Sources and Stocks
- Manage indexer modes and verify indexer health

## Next Steps in Your Journey

Now that you've mastered this process, here's how to build on your success:

### Immediate Actions
- Schedule weekly Low Stock report reviews
- Audit Sales Emails for branding and tone alignment with your store

### Level Up Your Skills
- Create role-based Admin users and permissions (System > Permissions > User Roles) for pick/pack/ship workflows to reduce errors

### Advanced Applications
- Connect carrier/shipping integrations for label printing and tracking
- Automate exports from Sales > Orders for accounting or ERP reconciliation

## When Things Don't Go as Expected

Even experienced merchants encounter challenges. Here's how to handle common situations:

Orders grid and saved views
- Can't save a grid view: Ensure your admin role has permissions for the area; try clearing Admin cache (System > Cache Management > Flush Magento Cache).

Mass actions
- "Invoice" action missing: The payment method may auto-capture; open the order and check the Invoices tab.

Sales Emails
- Emails not sending: Confirm Stores > Configuration > Sales > Sales Emails are enabled; verify Mail Sending Settings under Stores > Configuration > Advanced > System; confirm cron and SMTP with your host/developer.

Inventory / MSI
- SSA not suggesting sources: Ensure multiple Sources are enabled and assigned to the Stock serving your website.
- Low Stock report empty but items sell out: Raise Notify for Quantity Below or shorten your review cadence.

Indexers
- Indexers stuck: System > Tools > Index Management > Reindex Data. If issues persist, verify your server cron is running and check for long-running reindex processes.

## Continue Learning

### Related Tutorials
- Efficient Catalog Management
- Building Email Templates

### How-To Guides
- Create and Assign Attribute Sets
- Configure Sales Emails
- Manage Indexers

### Reference Materials
- Admin Grid Views and Filters
- Inventory Management (MSI) concepts

## Summary

### Key Takeaways
- Saved grid views and mass actions can save hours each week
- Sales Emails reduce support tickets and improve customer trust
- Low Stock monitoring prevents lost sales
- Indexers set to schedule keep the site fast during updates

### Remember
- Test changes on a staging site when possible and review operational reports weekly

---

*Found this tutorial helpful? [Share your feedback]() or [suggest improvements]().*

*Questions or stuck? [Get help from the community]() or [contact support]().*
