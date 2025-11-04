---
title: "Creating Your First Products"
description: "Complete tutorial on adding simple and configurable products to your catalog"
type: tutorial
audience:
  - "Magento merchants"
difficulty: Advanced
estimated_time: "9 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Product catalog setup"
tags:
  - tutorial
  - product-management
  - catalog
  - high-impact
  - complex
last_updated: "2025-09-11T07:30:34.565Z"
---

# Creating Your First Products

## Why This Tutorial Matters

**Business Impact:**

- Launch products faster to capture revenue sooner
- Improve conversion with accurate pricing, stock, and attribute-driven options
- Reduce catalog maintenance and returns by centralizing variations in configurable products

**What You'll Achieve:**

- Publish a Simple Product and a Configurable Product
- Correctly set price, tax class, stock, visibility, and categories
- Verify both appear on the storefront and are purchasable

## Learning Journey Overview

### Your Situation

You're onboarding Magento 2 and need to publish your first products today with accurate pricing and stock so customers
can buy immediately.

### What You'll Learn

By completing this tutorial, you will:

- Create a Simple Product from Admin (Catalog > Products)
- Create a Configurable Product using the configuration wizard
- Set price, tax class, stock, visibility, categories, and images
- Assign products to websites and categories
- Verify storefront visibility and troubleshoot common issues

### Success Criteria

You'll know you've succeeded when:

- A Simple Product is visible on the storefront and can be added to cart
- A Configurable Product displays swatches/options and allows selection for all variations
- Stock status shows In Stock, products appear in assigned categories and search

### Time Investment

- Estimated time: 20–30 minutes
- Skill level after completion: Confident with creating Simple and Configurable products
- Business value unlock: Faster go-live for new items and fewer catalog errors

## Before We Start

### Who This Is For

This tutorial is designed for:

- Catalog Managers and Merchandisers new to Magento 2
- Store Owners setting up their first products
- Anyone familiar with basic eCommerce concepts (SKU, price, stock)

### What You Need

Make sure you have:

- Admin access with Catalog permissions (Products, Categories, Attributes) to the target website/store view
- Magento 2.4.x (Open Source or Adobe Commerce)
- At least one Website/Store/View configured
- Tax Classes configured (Stores > Tax Rules)
- Shipping Origin set (Stores > Configuration > Sales > Shipping Settings)
- Inventory setup decided: Single Source (Default) or defined Sources/Stocks (Stores > Inventory)
- An Attribute Set ready (or use Default)

### Preparation Checklist

Before starting, complete these preparation steps:

- Confirm you can create/edit attributes (Stores > Attributes > Product)
- For Configurable Products, ensure variation attributes (e.g., Color, Size) exist and are set to Use To Create
  Configurable Product = Yes with input type Dropdown, Visual Swatch, or Text Swatch
- Verify cron is running to process indexes (or set indexers to Update on Save during testing)
- Have sample product images ready (JPG/PNG)

## Step-by-Step Learning Path

### Part 1: Create a Simple Product

1) Go to `Admin sidebar` > `Catalog` > `Products and click Add Product` > `Simple Product`.
2) Set Attribute Set = Default (or your attribute set).
3) Enter Product Name and a unique SKU.
4) Set Price and Tax Class (for example, Taxable Goods).
5) Set Quantity and Stock:
    - Single Source: Set Quantity and Stock Status = In Stock.
    - Multi-Source (MSI): Open the Sources section, Assign Sources, and enter quantities per source. Ensure each
      source's status is In Stock.
6) Set Weight (if the product ships) or mark This item has no weight (virtual).
7) Enable Product = Yes; set Visibility = Catalog, Search.
8) Assign Categories (click Select or create a new category).
9) Upload Images and Videos; optionally designate Base, Small, and Thumbnail.
10) In Search Engine Optimization, review URL Key and Meta Title.
11) In Product in Websites, ensure the correct website is checked.
12) Click Save, then click the View/Preview link to open the storefront page.

Verification for Simple Product

- Confirm the product page loads, shows price and image, indicates In Stock, and can be added to cart.

### Part 2: Create a Configurable Product

1) Go to Admin sidebar > Catalog > Products > Add Product > Configurable Product.
2) Choose Attribute Set and add basics: Name, base SKU, Price (optional; can be set for variants), Tax Class. Set Enable
   Product = Yes and Visibility = Catalog, Search.
3) In the Configurations section, click Create Configurations.
4) Select attributes for variations (e.g., Color, Size). If needed, click Create New Attribute and ensure:
    - Catalog Input Type for Store Owner = Dropdown, Visual Swatch, or Text Swatch
    - Use To Create Configurable Product = Yes
5) Select attribute values (e.g., Color: Red, Blue; Size: S, M, L).
6) Configure Images, Price, and Quantity for variants:
    - Apply to all SKUs or set by attribute (e.g., different image per Color, different price per Size)
    - If you leave quantity blank here, set quantities after generation for each child SKU
7) Click Generate Products to create the child SKUs.
8) Inventory and Sources (MSI):
    - For Single Source: Ensure each child has Quantity > 0 and Stock Status = In Stock
    - For Multi-Source: Open each child (or use mass action) to assign per-source quantities under Sources
    - Child products typically have Visibility = Not Visible Individually (default)
9) Assign Categories to the parent and upload images (parent image + variant images as needed). For color swatch
   display, you can set swatch images in Stores > Attributes > Product > Color.
10) Ensure Product in Websites includes your website, then Save and Preview.

Verification for Configurable Product

- On the storefront, confirm the configurable shows selectable options. Selecting options updates the SKU/price if
  configured. Add to cart successfully.
- Confirm the product appears in assigned categories and in search results.

## Practice and Reinforcement

Now that you've learned the core process, let's reinforce your skills:

- Create a Simple Product with tier pricing (Price > Advanced Pricing > Tier Price)
- Add a new Size attribute value and regenerate configurations for an existing Configurable Product

## What You've Accomplished

🎉 Congratulations! You have successfully:

- Created and published a Simple Product and a Configurable Product with correct stock, pricing, and visibility
- Verified storefront appearance in categories and search, and completed add-to-cart for both product types
- Applied MSI-aware inventory settings to avoid out-of-stock visibility issues

### Business Impact

- Faster time-to-publish for new SKUs, accelerating revenue capture
- Improved product discovery and selection through accurate categories, search, and swatches
- Reduced catalog maintenance by centralizing variations under a single configurable
- KPIs to watch: time-to-publish, product page conversion rate, out-of-stock rate, category/search visibility

### Skills Gained

You now have the ability to:

- Create and configure Simple and Configurable products
- Manage stock for Single Source and Multi-Source (MSI) setups
- Configure variation attributes that power swatches/options
- Troubleshoot common visibility and inventory issues

## Next Steps in Your Journey

Now that you've mastered this process, here's how to build on your success:

### Immediate Actions

- Add related, upsell, and cross-sell products to boost average order value
- Enable and refine swatches for visual clarity on configurable products
- Encourage and moderate product reviews to increase trust and conversion

### Level Up Your Skills

- Learn bulk import via CSV (System > Data Transfer > Import) for faster catalog scaling
- Optimize attribute sets to streamline product creation for your categories
- Configure advanced pricing (special, group, tier) and catalog rules

### Advanced Applications

- Set up scheduled updates for seasonal pricing and content
- Add custom options for simple variations not requiring child SKUs
- Use layered navigation with attributes to improve product discovery

## When Things Don't Go as Expected

Even experienced merchants encounter challenges. Here's how to handle common situations:

- Issue: Product not visible on storefront
    - Fix: Ensure Enable Product = Yes and Visibility = Catalog, Search (or ensure parent configurable is visible)
    - Fix: Check Product in Websites includes your website
    - Fix: Verify Stock Status = In Stock and quantities > 0 (per source if MSI)
    - Fix: Reindex (System > Index Management) and flush cache (System > Cache Management)

- Issue: Cannot create configurations
    - Cause: Variation attributes not compatible
    - Fix: Set attribute input type to Dropdown/Swatch and Use To Create Configurable Product = Yes (Stores >
      Attributes > Product)

- Issue: Price or image not showing for variants
    - Fix: In the configuration wizard, apply price/images at variant level or set defaults on the parent

- Issue: Out-of-stock children not selectable
    - Explanation: Expected behavior
    - Fix: Increase quantity or enable backorders (Stores > Configuration > Catalog > Inventory) if appropriate

## Continue Learning

### Related Tutorials

- Bulk import products with CSV
- Managing categories for better navigation
- Creating product attributes and swatches

### How-To Guides

- Set up tier, group, and special pricing
- Configure tax classes and tax rules
- Use URL rewrites and SEO settings for products

### Reference Materials

- Product types overview (Simple, Configurable)
- Inventory Management (MSI): Sources, Stocks, and Reservations
- Product attributes and attribute sets
- Import/export products
- Pricing features and catalog price rules

## Summary

You created both Simple and Configurable products with correct data, inventory, and visibility. You verified storefront
behavior and learned how to troubleshoot common issues, setting a foundation to scale your catalog efficiently.

### Key Takeaways

- Accurate attributes and MSI-aware stock are critical to visibility and conversion
- Configurable products reduce SKU clutter and speed updates
- Verify, index, and cache-clear if changes don’t appear immediately

### Remember

- If you use multiple sources, set quantities per source under Sources. If using the Default Source only, use the
  Quantity field. Correct stock drives storefront visibility and sales.
