---
title: "Getting Started with Product Import and Export"
description: "Bulk import/export products using CSV files and data management"
type: tutorial
audience:
  - "Magento merchants"
difficulty: Intermediate
estimated_time: "25–45 minutes"
prerequisites:
  - "Access to the Admin with appropriate permissions (System > Permissions > User Roles)"
  - "Attribute sets, websites, and stores configured. (You can create products via import even if the catalog is empty.)"
tags:
  - tutorial
  - product-management
  - inventory-management
  - data-transfer
  - high-impact
last_updated: "2025-09-10T13:54:44.117Z"
---

# Getting Started with Product Import and Export

## Why This Tutorial Matters

This tutorial shows you how to bulk create and update products using CSV import/export so you can launch collections faster, keep prices accurate, and reduce manual work.

**Business Impact:** 
- Reduce manual data entry, speed up catalog launches, and standardize data quality to decrease returns and support tickets.
- By standardizing your import process you cut catalog update time from hours to minutes, reduce data errors that lead to returns, and accelerate campaign launches. Example: Updating 500 SKUs manually (30 seconds each) takes ~4 hours; with import (~10 minutes prep + ~5 minutes processing) it's ~15 minutes, saving >3.5 hours per update cycle.

**What You'll Achieve:** 
- Set up a repeatable import/export workflow to bulk create, update, or retire products using CSV.

## Learning Journey Overview

### Your Situation
You need a reliable, fast way to add or update many products at once, keep images and attributes consistent, and ensure storefront accuracy after changes.

### What You'll Learn
By completing this tutorial, you will:

- Export a product CSV template
- Prepare a valid CSV with required columns
- Import products and images
- Handle Multi‑Source Inventory (MSI) source‑level stock
- Verify results and troubleshoot common issues

### Success Criteria
You'll know you've succeeded when:
- Import completes with 0 errors and all rows processed.
- Products appear in the catalog with correct price, visibility, and images.
- Category assignments and stock/salable quantity display as expected.
- Indexers show "Ready" and the storefront reflects updates.

### Time Investment
- **Estimated time:** 25–45 minutes
- **Skill level after completion:** Confident with core product import/export and basic troubleshooting
- **Business value unlock:** Faster catalog updates, fewer errors, scalable operations

## Before We Start

### Who This Is For
This tutorial is designed for:

- Merchants and catalog managers who need to update many products at once.
- Agencies onboarding catalogs from suppliers or migrating from another platform.
- Ops teams maintaining seasonal price and inventory changes.

### What You Need
Make sure you have:

- Admin access with permission to Import/Export (System > Permissions > User Roles).
- A CSV editor (Google Sheets, Excel) and ability to save UTF‑8 CSV.
- Attribute sets and required attributes defined.
- Website/store views created.
- File system or SFTP access to place product images on the server (recommended: pub/media/import).
- Sufficient PHP upload limits (upload_max_filesize and post_max_size greater than your CSV size).

### Preparation Checklist
Before starting, complete these preparation steps:

1) In Admin, export a small sample of products to get a valid header template (System > Data Transfer > Export > Entity Type: Products).
2) Create/confirm attribute sets and required attributes (Stores > Attributes > Product; Stores > Attribute Set).
3) Place product images on server: pub/media/import, and use those filenames in the CSV image columns.
4) Plan import behavior: Add/Update vs Replace vs Delete.
5) Schedule a maintenance window if importing thousands of SKUs.

## Step-by-Step Learning Path

Follow these steps:

1) Export a product CSV template
- Go to Admin: System > Data Transfer > Export.
- Entity Type: Products. Optionally filter to export a small set.
- Click Continue and download the CSV. Use its headers as your template to avoid mapping errors.

2) Prepare your CSV
- Keep the exported headers exactly as-is; encode as UTF‑8.
- Include required fields for each product, such as:
  - sku, attribute_set_code, product_type (simple, configurable, virtual, downloadable, bundle, grouped)
  - name, price, tax_class_name
  - status (1 = Enabled, 2 = Disabled), visibility
  - weight (required for shippable product types)
  - categories (use full paths like "Default Category/Gear/Bags"; separate multiple with |)
  - websites (use website codes, e.g., base)
  - image, small_image, thumbnail, additional_images (filenames located in pub/media/import)
- Tips:
  - Use exact attribute option labels or values that exist in your store. Create missing options before import.
  - For configurable products, prepare simple children first and plan configurable_variations if building via CSV (see Next Steps).

3) Import products
- Go to System > Data Transfer > Import.
- Entity Type: Products.
- Import Behavior: Add/Update (recommended for upserts). Use Replace for full overwrites and Delete to remove products.
- Choose your CSV file. Click Check Data to validate. If successful, click Import.
- After import, check System > Tools > Cache Management and refresh caches if needed.

4) (MSI) Import source items (quantities per source)
- If you use Multi‑Source Inventory, go to System > Data Transfer > Import.
- Entity Type: Stock Sources (source items).
- Use a CSV with columns like: source_code, sku, quantity, status (1 = In Stock, 0 = Out of Stock).
- Click Check Data, then Import.

5) Verify results
- Confirm products in Catalog > Products and on the storefront.
- Check System > Tools > Index Management; ensure indexers show Ready. Reindex if necessary.
- Validate prices, visibility, categories, images, and salable quantity.

## Practice and Reinforcement

Now that you've learned the core process, let's reinforce your skills:

Practice: Export one existing product, duplicate the row with a new SKU, change name, price, and image; import using Add/Update; verify it appears on the storefront and is purchasable.

## What You've Accomplished

🎉 **Congratulations!** You have successfully:

- Exported a product template and prepared a valid CSV.
- Imported products with images using the Add/Update behavior.
- Updated inventory at the source level (MSI) when applicable.
- Verified indexers, caches, and storefront output.

### Business Impact
- Your team can now perform bulk updates in minutes, reduce catalog errors, and accelerate campaign and product launches.

### Skills Gained
You now have the ability to:

- Create and update products at scale via CSV.
- Manage images and categories through data import.
- Maintain MSI quantities using source item imports.
- Validate and troubleshoot import results confidently.

## Next Steps in Your Journey

Now that you've mastered this process, here's how to build on your success:

### Immediate Actions
- Document your import template and store it in version control.
- Schedule monthly export snapshots for rollback and auditing.

### Level Up Your Skills
- Learn Advanced Pricing import to bulk manage tier and customer group prices.
- Automate recurring imports via Scheduled Import/Export (if available) or CLI/cron-driven scripts.

### Advanced Applications
- Build configurable products from CSV using configurable_variations and configurable_variation_labels.
- Run price promotions at scale by importing tier/group prices.
- Schedule regular stock updates from supplier feeds to prevent oversells and lost sales.

## When Things Don't Go as Expected

Even experienced merchants encounter challenges. Here's how to handle common situations:

- Problem: CSV fails validation.
  - Solution: Ensure UTF‑8 encoding, keep headers exact from the export template, and remove stray formulas or hidden characters.

- Problem: SKU not found when updating.
  - Solution: Use Add/Update behavior or include a row that creates the missing SKU.

- Problem: Images missing after import.
  - Solution: Place files in pub/media/import; ensure filenames and extensions in image/small_image/thumbnail/additional_images match exactly (case-sensitive). Refresh caches.

- Problem: Categories not applied.
  - Solution: Use full category paths starting from Default Category and separate multiple paths with |. Verify categories exist.

- Problem: Attribute option value doesn't exist.
  - Solution: Create the option first (Stores > Attributes > Product), then re-import.

- Problem: Salable quantity is 0 with MSI.
  - Solution: Import source items with quantity and status = 1 (In Stock), and ensure products are assigned to the correct source and stock.

- Problem: Import times out or file is too large.
  - Solution: Increase PHP limits (upload_max_filesize, post_max_size, max_execution_time) or split the CSV into smaller batches.

- Risk mitigation:
  - Export a full product snapshot before major imports to enable rollback.
  - Test imports on a staging environment before running on production.

## Continue Learning

### Related Tutorials
- Advanced Pricing Import (tier and group prices)
- Building Configurable Products via CSV
- Scheduled Imports/Exports and automation options

### How-To Guides
- Fix common import errors
- Manage attribute sets and required attributes

### Reference Materials
- Sample product CSV template
- MSI Source Items CSV format and field definitions

## Summary



### Key Takeaways
- Export first to get a valid template.
- Keep headers exact and values valid to avoid mapping errors.
- Use Add/Update for safe upserts; Replace for full overwrites.
- Store images in pub/media/import and reference filenames in CSV.
- Verify indexers and storefront after import.

### Remember
- Always back up with an export before bulk changes, and test on staging when possible.

---

*Found this tutorial helpful? [Share your feedback]() or [suggest improvements]().*

*Questions or stuck? [Get help from the community]() or [contact support]().*
