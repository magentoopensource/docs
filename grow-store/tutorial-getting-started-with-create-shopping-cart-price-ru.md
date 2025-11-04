---
title: "Getting Started with Create Shopping Cart Price Rules"
description: "Configure cart-level promotions and discount rules"
type: tutorial
audience:
  - "Magento merchants"
difficulty: Advanced
estimated_time: "9 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
tags:
  - tutorial
  - promotions
  - high-impact
last_updated: "2025-09-10T13:54:38.947Z"
---

# Getting Started with Create Shopping Cart Price Rules

## Why This Tutorial Matters

Note on terminology: In the Magento Admin, this feature is labeled Cart Price Rules. You'll find it under Marketing > Promotions > Cart Price Rules.

Business Impact:
- Cart Price Rules let you run targeted promotions at checkout—boosting conversion, increasing average order value (AOV), and moving inventory without changing product prices.
- Use threshold discounts to lift AOV (for example, $20 off $150), sitewide codes to drive conversion, and SKU/category conditions to clear inventory.
- Control margin by scoping rules to specific websites, customer groups, and by preventing discount stacking.

What You'll Achieve:
- Create, schedule, and QA a working cart-level discount with a coupon code.
- Control stacking behavior using Priority and Stop Further Rules Processing.
- Generate unique coupon codes for campaigns and measure coupon performance.

## Learning Journey Overview

### Your Situation
- You need a reliable way to launch cart-level promotions (with or without coupon codes) and verify they work across your websites and customer groups.

### What You'll Learn
By completing this tutorial, you will:
- Navigate to Marketing > Promotions > Cart Price Rules
- Configure Rule Information (websites, customer groups, coupon settings, dates, priority)
- Define Conditions and Actions (Apply types, Free Shipping, Apply to Shipping Amount)
- Add Labels for storefront clarity
- Generate and test coupon codes
- Control rule stacking with Priority and Stop Further Rules Processing
- Validate results on storefront and in reports

### Success Criteria
You'll know you've succeeded when:
- A test coupon applies the expected discount in the cart and checkout
- The discount label displays correctly
- The promotion is restricted to the intended website and customer groups
- The rule does not stack with lower-priority discounts when configured to stop further processing
- Coupon usage appears in Reports > Sales > Coupons

### Time Investment
- Estimated time: 9 minutes
- Skill level after completion: Intermediate
- Business value unlock: Launch margin-safe promotions that increase conversion and AOV, with measurable results via coupon reports

## Before We Start

### Who This Is For
This tutorial is designed for:
- Merchants, Ecommerce Managers, and Marketers who run promotions
- Store Admins configuring discounts for one or more websites/store views

### What You Need
Make sure you have:
- Magento 2.4.x Admin access with Marketing and Sales permissions
- At least one website and customer group to target
- A test product and a test customer account
- A staging environment to test before production (recommended)

### Preparation Checklist
Before starting, complete these preparation steps:
- Decide the promotion goal (for example, 10% off orders over $100)
- Determine target customer groups and websites
- Choose coupon type: single code or auto-generated unique codes
- Set start/end dates and time zone
- Decide stacking behavior and rule priority

Note: This tutorial applies to Magento Open Source and Adobe Commerce 2.4.x. Field labels may vary slightly across minor versions (for example, Stop Further Rules Processing vs Discard Subsequent Rules).

## Step-by-Step Learning Path

1) Navigate to the rule creation screen
- Admin sidebar > Marketing > Promotions > Cart Price Rules > Add New Rule

2) Rule Information
- Rule Name: Summer 10% over $100
- Description: 10% off subtotal over $100 with code SUMMER10
- Active: Yes
- Websites: Select target website(s)
- Customer Groups: For example, NOT LOGGED IN, General (select as needed)
- Coupon: Specific Coupon
- Coupon Code: SUMMER10 (or set Use Auto Generation = Yes for bulk codes)
- Uses per Coupon: e.g., 1000
- Uses per Customer: 1
- From/To: Set start and end dates
- Priority: 1 (lower number = higher priority)

3) Conditions (order-qualification logic)
- Leave blank to apply to all carts, or add thresholds:
  - Click Add (+) > Subtotal > is equal or greater than > 100

4) Actions (how the discount applies)
- Apply: Percent of product price discount
- Discount Amount: 10
- Maximum Qty Discount is Applied To: [blank or a number]
- Discount Qty Step (Buy X): [blank unless doing Buy X Get Y]
- Apply to Shipping Amount: No (set to Yes if the discount should include shipping)
- Free Shipping: No (or select For matching items only, if applicable)
- Apply the rule only to cart items matching the following conditions: [leave blank unless restricting to specific SKUs/categories]
- Stop Further Rules Processing (may appear as Discard Subsequent Rules): Yes if you want to prevent other lower-priority rules from applying

5) Labels (storefront text)
- Default Rule Label for All Store Views: Summer 10% Off
- Per Store View Labels: Optional, if you need localized or tailored text

6) Save the rule
- Click Save. The rule activates within the configured date range and scope.

7) Generate multiple coupons (optional)
- If Use Auto Generation = Yes: After saving, a Manage Coupon Codes tab appears. Open it, set Quantity, Code Length, and Format, then Generate and Export.
- Note: The Manage Coupon Codes tab appears only after the rule is saved with Use Auto Generation enabled.

8) Test on the storefront
- Add qualifying products to the cart
- Apply the coupon SUMMER10
- Verify the discount amount and the label on the Cart and Checkout totals

9) Verify reporting
- Go to Reports > Sales > Coupons to confirm coupon usage after placing a test order

## Practice and Reinforcement

Now that you've learned the core process, let's reinforce your skills:
- Create a Buy X Get Y rule: Buy 2, get 1 free of the same SKU
  - Apply = Buy X get Y free (discount amount is Y), Discount Amount = 1, Discount Qty Step (Buy X) = 2
- Create a fixed cart discount: $15 off orders over $150
  - Apply = Fixed amount discount for whole cart, Discount Amount = 15, add a Subtotal ≥ 150 condition
- Generate 500 unique coupons and export them for an email campaign

## What You've Accomplished

🎉 Congratulations! You have successfully:
- Created and activated a Cart Price Rule with a coupon code
- Configured conditions, actions, and labels
- Controlled stacking via priority and Stop Further Rules Processing
- Tested the promotion end-to-end and verified reporting

### Business Impact
- You can now launch targeted, margin-safe promotions that increase conversion and AOV, and use reporting to quantify performance.

### Skills Gained
You now have the ability to:
- Configure Cart Price Rules end-to-end, including coupon generation
- Scope promotions to websites and customer groups
- Prevent unintended discount stacking to protect margin
- Validate and measure promo performance in the Admin

## Next Steps in Your Journey

Now that you've mastered this process, here's how to build on your success:

### Immediate Actions
- Launch a welcome coupon for new subscribers (single code, 10% off). Measure redemptions weekly in Reports > Sales > Coupons.
- Run a weekend threshold promotion ($20 off $150) with Stop Further Rules Processing to protect margins.
- Export 1,000 unique coupons for paid ads; track code-specific ROI and attribute redemptions to campaigns.

### Level Up Your Skills
- Localize labels per store view for international sites
- Use conditions to include/exclude categories or SKUs to move specific inventory
- Combine with customer groups to reward logged-in or VIP customers only

### Advanced Applications
- Offer free shipping for specific items or thresholds via Actions > Free Shipping
- Build Buy X Get Y cross-sell bundles by restricting Actions to matching items
- Run multi-website A/B tests by cloning rules with different labels, codes, or thresholds, then compare coupon performance

## When Things Don't Go as Expected

Even experienced merchants encounter challenges. Here's how to handle common situations:
- Coupon not applying: Verify Active = Yes, date range, website, and customer groups. Confirm Uses per Coupon/Customer not exceeded. Ensure Conditions evaluate to true (Subtotal vs including shipping/tax). If using specific item conditions, confirm products match attributes.
- Wrong discount amount: Check Apply type (Percent vs Fixed) and tax settings (Stores > Configuration > Sales > Tax > Calculation Settings > Apply Discount on Prices). Apply to Shipping Amount changes the base for the discount.
- Multiple discounts stacking: Increase Priority (lower number executes first) and enable Stop Further Rules Processing (or Discard Subsequent Rules) on the high-value promo.
- Free shipping not triggered: In Actions, set Free Shipping appropriately and ensure your shipping methods allow free shipping when flagged.
- Auto-generated coupons tab missing: Save the rule with Use Auto Generation = Yes to reveal Manage Coupon Codes.
- Scope issues on multi-website: Confirm Websites selection matches the store you test on and that you are testing in the correct storefront.
- Testing tips: Clear/refresh the cart, re-add items, and sign in/out as needed. If storefront labels don't update, refresh caches.

## Continue Learning

### Related Tutorials
- Catalog Price Rules vs Cart Price Rules: when to use each
- Customer Segmentation (Adobe Commerce) to target high-value shoppers

### How-To Guides
- Reporting: Sales by Coupon and Sales reports to measure promo performance
- Abandoned cart follow-ups using coupons

### Reference Materials
- Admin paths: Marketing > Promotions > Cart Price Rules; Reports > Sales > Coupons
- Notes on labels: Stop Further Rules Processing may appear as Discard Subsequent Rules in some versions

## Summary

You created a cart-level promotion, controlled who sees it and when, prevented unintended stacking, and verified performance with reports. These steps are the foundation for reliable, margin-aware promotional campaigns.

### Key Takeaways
- Cart Price Rules drive conversion and AOV without changing list prices
- Priority and Stop Further Rules Processing prevent double-discounting
- Coupon reports provide measurable ROI for your campaigns

### Remember
- Test in a staging environment first
- Align dates/time zone, scope, and customer groups before launch
- Document your rule naming, labeling, and priorities for team consistency
