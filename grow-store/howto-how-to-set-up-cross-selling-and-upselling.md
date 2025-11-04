---
title: "How to Set Up Cross-Selling and Upselling"
description: "Configure related products and recommendations to increase average order value"
type: how-to
audience:
  - "Magento merchants"
difficulty: Intermediate
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Product catalog setup"
tags:
  - how-to
  - product-management
last_updated: "2025-09-14T18:39:27.339Z"
---

# How to Set Up Cross-Selling and Upselling

## Overview

Increase average order value and improve the shopping experience by configuring relevant cross-sells, upsells, and
related products that guide customers to complementary and higher-value items.

Applicability: These steps apply to Magento Open Source and Adobe Commerce 2.4.x. Adobe Commerce also offers automation
via Marketing > Promotions > Related Product Rules and Marketing > Product Recommendations for scalable, AI-driven
suggestions.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Product catalog setup
- Admin role with Catalog > Products edit permission
- Staging environment available to test changes before production

## What You'll Accomplish

By following this guide, you will:

- Successfully set up cross-selling and upselling
- Improve your store's performance and customer experience

## Step-by-Step Instructions

### Step 0: Set goals and choose target products

This quick planning step helps you focus on the highest ROI opportunities.

Identify a pilot category or top 10 products where recommendations will have the biggest impact. Define success
metrics (e.g., +10% AOV, +5% attach rate in 30 days). Prioritize SKUs with high traffic and low attach rate, strong
margins, and categories with natural complements (e.g., shoes → insoles, socks). Map for each base product: 3–5 related
products, 2–4 upsells, and 2–4 cross-sells.

### Step 1: Open the product editor in Magento Admin

Go to Catalog > Products. Search for and click the base product (e.g., RUN-SHOE-100). Ensure the product is Enabled, In
Stock, and Visible Individually. Confirm it is assigned to the correct Website.

Ensure the Store View selector (top-left) is set to the desired scope (All Store Views or the target Website). Product
links are global, but product status, visibility, and price can be website-specific.

### Step 2: Add Related Products (on-page complements)

Scroll to the Related Products, Up-Sells, and Cross-Sells section. Click Add Related Products. Use filters to find
complementary items (e.g., SHOE-INSOLE-DELUXE, SHOE-CLEAN-KIT). Select 3–5 items and click Add Selected Products. Use
the Position column (or drag-and-drop) to set display order; storefront sorting typically respects this order (
theme-dependent).

Selection playbook: Choose accessories frequently bought together within ~20–40% of the base product’s price for strong
relevance without cannibalizing the main purchase.

### Step 3: Add Upsell Products (premium alternatives)

In the same section, click Add Up-Sell Products. Search for premium or higher-margin alternatives (e.g.,
RUN-SHOE-200-PRO, RUN-SHOE-NEW-TECH). Select 2–4 items and click Add Selected Products. Use the Position column (or
drag-and-drop) to set the priority order; confirm on your storefront.

Selection playbook: Favor newer models or bundles with a +15–30% price delta and higher margin/value to the customer.

### Step 4: Add Cross-Sell Products (cart complements)

Click Add Cross-Sell Products. Select items suited for impulse purchase at checkout (e.g., ATH-SOCKS-PACK,
SPORT-BOTTLE-700ML). Choose 2–4 and click Add Selected Products. Use the Position column (or drag-and-drop) to set
order; most themes respect this.

Selection playbook: Low-cost add-ons under ~15% of cart value to minimize friction and increase attach rate.

### Step 5: Save and repeat for key catalog items

Click Save at the top right. Repeat the process for additional high-traffic products. Focus first on top 10–20 SKUs to
maximize impact quickly.

### Step 6: Optional: Bulk update via CSV import

Prepare a product CSV with these columns for each base SKU: sku, related_skus, upsell_skus, crosssell_skus. Example
file:

```
sku,related_skus,upsell_skus,crosssell_skus
RUN-SHOE-100,"SHOE-INSOLE-DELUXE,SHOE-CLEAN-KIT","RUN-SHOE-200-PRO,RUN-SHOE-NEW-TECH","ATH-SOCKS-PACK,SPORT-BOTTLE-700ML"
```

Go to System > Data Transfer > Import. Set Entity Type = Products, Import Behavior = Add/Update. Upload the file,
Validate, then Import. Reindex if needed.

### Step 7: Refresh cache and confirm indexing

Go to System > Cache Management and click Flush Magento Cache. Then open System > Index Management and confirm key
indexers are Ready, especially:

- Product EAV
- Category Products
- Catalog Search (if applicable)

If indexers are set to Update by Schedule, ensure cron is running. CLI alternative: run bin/magento indexer:reindex from
your Magento root.

### Step 8: Validate storefront display on product page

On the storefront, open a configured product page. Verify Related Products and Upsell Products blocks display the exact
items you added, in the order you set. Check mobile and desktop views.

If blocks don’t appear:

1) Ensure linked products are Enabled, Visible (Catalog, Search), and In Stock.
2) Confirm all products are assigned to the same Website as the base product.
3) Stores > Configuration > Catalog > Inventory > Stock Options: If Display Out of Stock Products = No, out-of-stock
   linked items won’t show.
4) Some themes remove or relocate these blocks—test with a reference theme (e.g., Luma) or consult your theme docs.

### Step 9: Validate cross-sells on the cart page

Add the base product to the cart. Open the cart page (not just the mini-cart). Confirm Cross-Sell items appear. If you
added multiple cross-sells, confirm limits and ordering are respected by the theme.

### Step 10: Set up measurement and monitor impact

Record a baseline AOV from Reports > Sales > Orders for the prior period. After deployment, monitor AOV weekly. Use
Reports > Products > Products in Cart to identify popular combinations. In your analytics, create events or segments to
track clicks and add-to-cart from recommendation blocks.

Implementation tips:

- Add a unique CSS class or data attribute to recommendation links (e.g., data-rec-block="related|upsell|crosssell") and
  configure GA4/Adobe Analytics click events.
- Track per-block KPIs: CTR, add-to-cart rate, conversion rate, revenue per session, and AOV.
- Review weekly for 4 weeks; prune low performers and replace with better candidates.
- Consider an A/B test (show vs hide blocks) for ~2 weeks to quantify impact.

## Verification

Use this checklist to confirm everything is working correctly:

- Admin: Product saved with related/upsell/cross-sell links and Position set
- Indexers: Product EAV, Category Products, and Catalog Search are Ready
- Cache: Magento cache flushed; cron running if using Update by Schedule
- Storefront: Blocks visible on product and cart pages; order respected; works on mobile and desktop
- Analytics: Events fire for clicks/add-to-cart from recommendation blocks

## Common Issues and Solutions

- Links not saving: Verify you clicked Save and resolved any validation errors. Check role permissions at System >
  Permissions > User Roles for Catalog > Products.
- Items not visible: Confirm Website assignment and stock status; reindex and refresh invalidated caches. Check Display
  Out of Stock Products setting.
- Order not respected: Ensure Position values are set. Some themes override order—review theme settings/layout or test
  with a reference theme.
- Import fails: Validate the CSV, ensure headers match exactly, and check import error details. Review logs at
  var/log/system.log and var/log/exception.log. Confirm delimiter/enclosure settings align with your file.
- Performance: Limit to 3–5 related and 2–4 upsells/cross-sells to keep pages fast and focused.

