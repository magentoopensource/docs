---
title: "Getting Started with Manage Order Statuses"
description: "Configure and assign custom order statuses to align order communication and back-office processes."
type: tutorial
audience:
  - "Magento merchants"
  - "Advanced users"
difficulty: Beginner
estimated_time: "9 minutes"
prerequisites:
  - "Admin access with permission to Stores > Settings > Order Status"
  - "At least one test order available (Pending or Processing)"
  - "Optional: Email sending configured to validate notifications"
tags:
  - tutorial
  - high-impact
  - complex
last_updated: "2025-09-10T13:53:26.753Z"
---

# Getting Started with Manage Order Statuses

## Why This Tutorial Matters

**Business Impact:** Clear, custom order statuses reduce "Where is my order?" (WISMO) inquiries, help warehouse and support teams prioritize, and improve customer trust with transparent updates.

**What You'll Achieve:** Create a custom order status, assign it to the right order state (for example, Processing), control whether customers can see it on the storefront, and update an order to validate both Admin workflow and customer-facing behavior.

## Learning Journey Overview

### Your Situation
You need clearer milestones between Processing and Complete so your team and customers know exactly what’s happening with each order.

### What You'll Learn
By completing this tutorial, you will:

- Create a custom order status with store-view specific labels
- Assign the status to an order state and decide if it should be the default
- Control whether the status is visible on the storefront
- Update an order to the new status and notify the customer
- Filter and export orders by your custom status

### Success Criteria
You'll know you've succeeded when:

- Your custom status appears under Stores > Settings > Order Status
- It is assigned to the correct Order State and, if desired, set as default
- You can set an order to this status from the Comments History and the customer sees it in their account
- You can filter the Orders grid by the new status

### Time Investment
- **Estimated time:** 9 minutes
- **Skill level after completion:** Confident in configuring and assigning order statuses
- **Business value unlock:** Faster order triage and clearer customer updates

## Before We Start

### Who This Is For
This tutorial is designed for:

- Store Owners and Operations Managers seeking clearer order workflows
- Customer Service Leads who need precise statuses visible to customers

### What You Need
Make sure you have:

- Admin access to Stores > Settings > Order Status
- At least one test order (Pending or Processing) to verify status updates
- Optional: Sales Emails enabled to verify customer notifications (Stores > Settings > Configuration > Sales > Sales Emails)

> Applies to Magento Open Source and Adobe Commerce 2.4.x. UI labels may differ slightly in earlier versions.

### Preparation Checklist
Before starting, complete these preparation steps:

1) Confirm your Admin role can access Stores > Settings > Order Status
2) Create or identify a test order
3) If testing notifications, ensure Sales Emails are enabled and a transactional email sender is configured

## Step-by-Step Learning Path

1) Navigate to Admin: Stores > Settings > Order Status
2) Click "Create New Status". Enter:
   - "Status Code": awaiting_pickup (lowercase, no spaces)
   - "Status Label": Awaiting Pickup
   - (Optional) "Store View Specific Labels": add translations or variants per store view
   Click "Save Status".
3) Click "Assign Status to State". Set:
   - "Order State": Processing (or the state that matches your workflow)
   - "Order Status": Awaiting Pickup
   - "Use Order Status As Default": leave unchecked unless you want this to be the default status whenever an order enters this state
   - "Visible on Storefront": check if customers should see this status in their account
   Click "Save Status Assignment".

   Best practices and warnings:
   - Avoid setting a custom status as default for a state in production without testing. This affects how statuses are applied when orders change state (for example, after invoicing or shipment).
   - Keep internal-only statuses (for example, "Awaiting Fraud Review") hidden from the storefront to prevent customer confusion.

4) Verify on an order:
   - Go to Sales > Orders > open a Processing order > scroll to the "Comments History" section.
   - In "Status", select "Awaiting Pickup". Optionally check "Notify Customer by Email", add a comment, and click "Submit Comment".
   - Confirm the new status displays on the order. If notifications are enabled, verify the customer email and the status display in the customer’s account.
5) Filter by status:
   - Go to Sales > Orders, open Filters, set "Status" to "Awaiting Pickup", and apply filters to validate reporting and triage.

## Practice and Reinforcement

Now that you've learned the core process, let's reinforce your skills:

- Create a status "Awaiting Vendor Confirmation" assigned to state: On Hold; set "Visible on Storefront" off and note why it’s internal-only
- Add store-view specific labels for a non-English store view and verify display
- Export orders filtered by your new status to share with your warehouse team

## What You've Accomplished

🎉 **Congratulations!** You have successfully:

- Created a custom order status and labels
- Assigned the status to a state with correct storefront visibility
- Updated an order and optionally notified the customer
- Filtered orders by the new status for operational triage

### Business Impact
Expected impact: Reduce WISMO tickets by 10–30% through clearer statuses; speed internal triage by enabling status-based queues; improve SLA adherence by filtering operations dashboards.

### Skills Gained
You now have the ability to:

- Configure and manage custom order statuses in Admin
- Assign statuses to states with the correct defaults and visibility
- Operationalize statuses for customer communication and team workflows

## Next Steps in Your Journey

Now that you've mastered this process, here's how to build on your success:

### Immediate Actions
- Review all current statuses and retire or rename unused ones
- Align status names with your support macros and internal SOPs

### Level Up Your Skills
- Create saved filters in the Orders grid by status for each team (for example, warehouse, customer service)
- Document when each team should move orders between statuses

### Advanced Applications
- Use the REST API to pull orders by status into your BI/reporting: GET /V1/orders?searchCriteria[filter_groups][0][filters][0][field]=status&searchCriteria[filter_groups][0][filters][0][value]=awaiting_pickup&searchCriteria[filter_groups][0][filters][0][condition_type]=eq
- With GraphQL, query orders and note that "status" and "state" are distinct fields; do not conflate them in logic
- Align default status selections with your payment/shipping flows to minimize manual updates

## When Things Don't Go as Expected

Even experienced merchants encounter challenges. Here's how to handle common situations:

- Status not available on order: Ensure it’s assigned to the order’s current "Order State" (Stores > Settings > Order Status > "Assign Status to State").
- Customer can’t see status: Verify "Visible on Storefront" is checked in the status-to-state assignment, and that you added a comment when changing the status.
- Can’t delete a status: Unassign it from all states first; you cannot delete default or in-use statuses.
- Unexpected default status applied: Review "Use Order Status As Default" on assignments; avoid changing defaults unless tested on staging.
- Emails not sent: Check Sales Emails settings (Stores > Settings > Configuration > Sales > Sales Emails) and confirm "Notify Customer by Email" was selected when submitting the comment.

## Continue Learning

### Related Tutorials
- Configure Sales Emails
- Create Saved Filters in Orders Grid

### How-To Guides
- Assign Order Status to State
- Manage Store View Labels for Order Statuses

### Reference Materials
- Concept: Order States vs. Order Statuses (differences and mapping)
- Orders REST API Overview; retrieving orders by status and understanding "state" vs "status"
- GraphQL: Order queries highlighting distinct "status" and "state" fields

## Summary

Custom order statuses help you communicate clearly with customers and streamline internal workflows. You created a status, assigned it to the right state with appropriate visibility, verified it on an order, and learned how to use it for filtering and reporting.

### Key Takeaways
- Use exact UI labels and assign statuses to the correct state to enable selection on orders
- Only set a custom status as default after thorough testing
- Hide internal-only statuses from the storefront to reduce confusion

### Remember
Test changes in a staging environment and align status names and defaults with your fulfillment and support processes before rolling into production.

---

*Found this tutorial helpful? [Share your feedback]() or [suggest improvements]().*

*Questions or stuck? [Get help from the community]() or [contact support]().*
