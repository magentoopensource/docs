---
title: "Getting Started with Categories (Product Catalog)"
description: "Organize products with categories and navigation structure"
type: tutorial
audience:
  - "Magento merchants"
difficulty: Beginner
estimated_time: "9 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "At least one Admin role with access to Catalog > Categories"
  - "Optional: One or more sample products to assign (enabled, in stock, visible in catalog)"
tags:
  - tutorial
  - product-management
  - high-impact
  - complex
last_updated: "2025-09-10T13:25:03.872Z"
---

# Getting Started with Categories (Product Catalog)

## Why This Tutorial Matters

**Business Impact:** A well-structured category tree improves product findability, boosts conversion rates, and strengthens SEO by creating clean, keyword-rich URLs. Clear navigation reduces time-to-product, increases product views per session, and supports higher average order value.

**What You'll Achieve:** You'll create a category structure, configure display settings, assign products, enable layered navigation, optimize URLs and metadata, and verify that categories appear correctly in the storefront menu.

## Learning Journey Overview

### Your Situation
You have products live or in progress, but customers struggle to browse effectively. You need a clear category hierarchy that mirrors how shoppers think (for example, Men > Shoes > Sneakers), shows relevant filters, and supports SEO.

### What You'll Learn
By completing this tutorial, you will:

- Create root and subcategories (Admin sidebar > Catalog > Categories).
- Configure category visibility (Enable Category, Include in Menu) and display settings (Display Mode, Anchor).
- Assign products to categories and control product order (Position).
- Optimize category URLs and metadata for SEO (URL Key, Meta Title, Meta Description).
- Verify storefront navigation, layered filters, and category pages.
- Troubleshoot common issues (category not visible, filters missing, 404 after rename).

### Success Criteria
You'll know you've succeeded when:

- Your new category appears in the top navigation and is clickable.
- Assigned products display in the correct category with the intended sort order.
- Layered navigation filters (for example, size, color, price) appear on category pages for Anchor categories.
- The category page URL is readable (for example, /men/shoes/sneakers) and loads without 404 errors.

### Time Investment
- Estimated time: 9 minutes
- Skill level after completion: Able to create and configure categories, assign products, and troubleshoot visibility
- Business value unlock: Faster path-to-product and improved SEO through structured navigation

## Before We Start

### Who This Is For
This tutorial is designed for merchants and catalog managers using Magento Open Source or Adobe Commerce 2.4.x who manage site navigation and merchandising.

### What You Need
Make sure you have:

- Admin access with permissions for Catalog > Categories
- Magento 2.4.x (Open Source or Adobe Commerce)
- Optional: One or more sample products to assign (Status: Enabled; Visibility: Catalog or Catalog, Search; In Stock)

### Preparation Checklist
Before starting, complete these preparation steps:

- Sketch your category hierarchy (2–3 levels deep is ideal).
- Decide category naming conventions (customer-friendly, keyword-rich).
- Identify which categories should show filters (set Anchor = Yes for shoppable categories).
- If using multiple store views, decide which root category each store will use.

## Step-by-Step Learning Path

1) Open Categories
- Go to Admin sidebar > Catalog > Categories.
- Use the Store View switcher (top-left) to select the correct scope (for example, Default Store View).

2) Create a Root Category (optional)
- Click Add Root Category.
- In General, enter Name (for example, "Default Category" or your site name).
- Set Enable Category = Yes.
- Set Include in Menu = Yes (to show in top navigation).
- Click Save.
- To assign this root category to a store: go to Admin > Stores > All Stores > click your Store (under the Stores column) > set Root Category to the one you created > Save.

3) Create a Subcategory
- In Catalog > Categories, select the parent category.
- Click Add Subcategory.
- In General:
  - Name: for example, "Sneakers".
  - Enable Category = Yes.
  - Include in Menu = Yes.
- In Content: add a Category Image (optional) and a concise Description to support SEO and conversions.
- In Display Settings:
  - Display Mode: Products only (or Static block and products if you plan to add a CMS banner).
  - Anchor: Yes (enables layered navigation filters on the category page).
  - Available Product Listing Sort By: Position, Product Name, Price (select applicable).
  - Default Product Listing Sort By: Position (recommended for manual merchandising).
- In Search Engine Optimization:
  - URL Key: lowercase-hyphenated (for example, sneakers).
  - Meta Title and Meta Description: concise and keyword-targeted.
- Click Save.

4) Assign Products to the Category
- Open the Products in Category section.
- Use filters and checkboxes to include or exclude products from the category.
- Use the Position column to control product order (lower numbers show first). Drag-and-drop may be available depending on version.
- Ensure each product is Enabled, In Stock, and has Visibility = Catalog or Catalog, Search.
- Click Save.

5) Verify on the Storefront
- Visit the storefront and open the top navigation. Confirm the category appears.
- Open the category page. Confirm products display and filters (layered navigation) appear for Anchor categories.
- Confirm URL format matches your URL Key.

6) Cache and Indexing (if changes do not appear)
- Go to Admin > System > Cache Management and click Flush Magento Cache.
- If indexers are set to Update by Schedule, allow cron to run. If needed, go to Admin > System > Tools > Index Management and reindex Category Products and related indexers.

Best practices embedded:
- Set Anchor = Yes for shoppable categories to enable filters.
- Set Default Product Listing Sort By = Position to highlight best-sellers and promotions.
- Craft URL Keys with primary keywords and write compelling Meta Descriptions to improve SERP click-through.

## Practice and Reinforcement

Now that you've learned the core process, let's reinforce your skills:

- Create a seasonal subcategory (for example, "Back to School"), assign five products, and verify it appears in the menu.
- Switch Anchor to No on a category and note the difference in filters; then turn it back to Yes.
- Adjust product Positions to feature top sellers first and verify the order on the storefront.

## What You've Accomplished

🎉 Congratulations! You have successfully:

- Created a structured category tree, configured display and SEO settings, assigned products, verified storefront visibility, and learned how to clear cache or reindex when needed.

### Business Impact
Customers can now find products faster via clear navigation and filters, reducing bounce and improving conversion. Clean URLs and on-page content improve organic search visibility.

### Skills Gained
You now have the ability to:

- Plan and create a category tree
- Enable layered navigation via Anchor settings
- Assign products to categories and control ordering
- Configure SEO-friendly URLs and metadata for categories
- Perform basic cache and indexing management

## Next Steps in Your Journey

Now that you've mastered this process, here's how to build on your success:

### Immediate Actions
- Build out all primary and secondary categories based on your plan.
- Merchandise top revenue categories by setting product positions.

### Level Up Your Skills
- Add a CMS static block banner to key categories (Display Mode: Static block and products).
- Audit category names and URL Keys for keyword relevance and clarity.

### Advanced Applications
- Multi-store: Assign different root categories per store for localized navigation (Admin > Stores > All Stores).
- SEO: Map redirects when renaming categories to avoid 404s (Marketing > SEO & Search > URL Rewrites).

## When Things Don't Go as Expected

Even experienced merchants encounter challenges. Here's how to handle common situations:

- Category not visible in menu: Ensure Enable Category = Yes and Include in Menu = Yes; confirm you're editing the correct Store View; flush cache.
- Filters not showing: Set Anchor = Yes; ensure product attributes are set to Use in Layered Navigation and have values on products.
- Products not appearing: Verify products are Enabled, In Stock, assigned to the category, and have Visibility that includes Catalog.
- 404 after changing URL Key: Magento creates URL rewrites automatically; flush cache. If the issue persists, check Marketing > SEO & Search > URL Rewrites for conflicts.
- Changes not visible: If indexers are set to Update by Schedule, wait for cron or reindex via System > Tools > Index Management.

## Continue Learning

### Related Tutorials
- Product Attributes and Layered Navigation
- Merchandising with Product Positions

### How-To Guides
- Configure URL Keys and Meta Data for Categories
- Assign a Root Category to a Store View

### Reference Materials
- Catalog > Categories field reference
- Layered Navigation configuration
- URL Rewrites management

## Summary

A clean category structure drives shopper discovery and SEO. Use Anchor categories for filters, Position for merchandising, and verify changes with cache and indexing routines.

### Key Takeaways
- Navigate to Admin sidebar > Catalog > Categories to build and manage your category tree.
- Enable Category and Include in Menu control visibility; Anchor controls filters.
- Use Position to curate product order and boost conversion.
- Keep URL Keys readable and stable; add redirects when renaming.

### Remember
Always work in the correct Store View, clear cache if changes don’t appear, and reindex when necessary to ensure storefront accuracy.

---

*Found this tutorial helpful? [Share your feedback]() or [suggest improvements]().*

*Questions or stuck? [Get help from the community]() or [contact support]().*
