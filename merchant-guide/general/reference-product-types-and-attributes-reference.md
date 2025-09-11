---
title: "Product Types and Attributes Reference"
description: "Reference for product types, key attribute settings, and configuration procedures for Commerce/Open Source 2.4.x"
type: reference
audience:
  - Magento merchants
  - New Magento users
tags:
  - reference
  - catalog
  - product-types
  - attributes
  - configuration
last_updated: "2025-09-11T17:15:57.531Z"
---

# Product Types and Attributes Reference

## Overview

Use this reference to select the correct product type and configure product attributes to power search, filters, and merchandising. Applies to Adobe Commerce and Magento Open Source 2.4.x unless noted. Edition-specific features are marked Commerce-only. Following these guidelines improves findability, conversion, and catalog efficiency.

Choosing the right structure has measurable ROI. For example, Configurable products (with per-variant stock and pricing) improve size/color discovery and reduce oversells compared to Simple products with Customizable Options. Properly configured filterable attributes increase product discovery in layered navigation and site search, often improving category conversion and reducing support tickets about unavailable sizes or colors.

Measure impact:
- Benchmark before/after: category conversion rate, product discovery rate (sessions with filter use), search exit rate, and count of OOS-related support tickets.
- Track changes when enabling new filters or restructuring products (for example, switching to Configurable). Expect uplift in category conversion and fewer OOS inquiries when variation stock is accurate and filterable.

### Product Types at a Glance
- Simple: Single SKU with price, inventory, and shipping. Use for standalone items.
- Configurable: Parent product with variations (for example, color/size). Requires variation attributes (Dropdown/Visual/Text Swatch) with Global scope and 'Use To Create Configurable Product' = Yes.
- Grouped: Collection of simple products presented together; each item is purchased and inventoried separately (no price on parent).
- Bundle: Build-your-own kits with selectable components; supports dynamic or fixed pricing and shipping.
- Virtual: No shipment (services, warranties). Inventory optional.
- Downloadable: Digital files with access control and link management.
- Gift Card (Commerce-only): Virtual, physical, or combined gift cards with codes and balances.

Note: Gift Card is available only in Adobe Commerce. Before creating Gift Card products, configure code pools and emails at Stores > Configuration > Sales > Gift Cards (Commerce-only). See official guide for code generation and delivery.

### Choosing the Right Product Type

Quick decision flow:
- Is it physical vs digital/service? Physical: Simple/Configurable/Grouped/Bundle. Digital/service: Virtual or Downloadable; Gift Card (Commerce-only) for store credit.
- Need per-option stock/pricing/media (for example, size/color)? Use Configurable.
- Customer should build their own kit from components? Use Bundle.
- Display a set of separate SKUs together but purchase individually? Use Grouped.
- Selling store credit or vouchers? Use Gift Card (Commerce-only).

- Multiple sizes/colors with stock per variant: Use Configurable (attributes: size, color).
- Customizable kit with selectable components: Use Bundle.
- Upselling a set of related items with independent SKUs: Use Grouped.
- Service with no shipping: Use Virtual. For subscriptions/recurring billing, install a subscription/recurring payments extension; the product type remains Virtual unless the extension specifies otherwise.
- Digital downloads (software, media): Use Downloadable.
- One-time voucher/currency product: Use Gift Card (Commerce-only).
- Single SKU with simple options and no inventory per option: Simple with Customizable Options (not filterable; no stock by option).

Configurable vs Simple with Customizable Options — quick decision:
- Use Configurable when you need: clean, unified PDP with swatches; optional variant-specific media/pricing; per-variant stock and analytics; size/color filtering in layered navigation; reduced oversells and returns. Child SKUs do not get separate SEO URLs by default (unless you set child Visibility to Catalog/Search and merchandise them, which can create duplicate content and is generally not recommended).
- Use Simple with Custom Options only when: options are cosmetic or non-inventory (for example, engraving) and you do not need filtering or per-option stock/reporting.

Performance/cost trade-offs:
- Configurable products increase SKU count (one child per variant), affecting import times, index size, and ERP/PIM sync volume.
- Bundles add price calculation overhead at runtime.
- Choose the minimal-complexity type that meets requirements to balance merchandising benefits with operational costs.

## Configuration

### Attribute Basics
- Attribute: A data field for products (for example, Brand, Color, Size).
- Attribute Set: A template of attributes applied to products in a category or type.
- Scope: Product attributes can be Global or Store View. Use Global for variation and filterable attributes.
- Price scope: Product pricing uses Stores > Configuration > Catalog > Catalog > Price > Catalog Price Scope (Global or Website).
- Input Types: Text, Text Area, Dropdown, Multiple Select, Visual Swatch, Text Swatch, Boolean, Price, Date, and more.
- Storefront impact: Attributes power search, sorting, and layered navigation filters. Prefer Dropdown/Swatch for filterable facets and configurable variations. Attributes like Brand can also power SEO-friendly landing pages (for example, Brand pages) and internal linking.

### Key Attribute Properties (Stores > Attributes > Product)
- Scope: Set to Global for attributes used in configurable variations.
- Catalog Input Type: Use Dropdown/Visual/Text Swatch for filters and variations.
- Use in Layered Navigation: Select 'Filterable (with results)' to show only filters that return results, or 'Filterable (no results)' to always show filters (use sparingly). Applies to Dropdown and Multiple Select attributes. Price filtering is available by default and controlled by Stores > Configuration > Catalog > Catalog > Layered Navigation (price step), not by this attribute toggle.
- Use in Search Results Layered Navigation: Available for Dropdown and Multiple Select attributes that are set to Filterable. Set to Yes to also show the filter on search results pages.
- Position: Controls the display order of the filter in layered navigation (lower number appears higher).
- Use in Search: Include attribute in quick/full-text search as needed.
- Search Weight: Increase for high-signal attributes (for example, Brand, SKU, MPN) to boost search relevance.
- Used for Sorting in Product Listing: Enable for numeric/text fields you want to sort by.
- Visible on Catalog Pages on Storefront: Show on product detail pages when relevant.
- Use for Promo Rule Conditions: Make available in cart/catalog price rules.
- Comparable on Storefront: Show in compare tables if useful.
- Use To Create Configurable Product: Set to Yes to make the attribute available in the Create Configurations wizard.
- Attribute Code: Lowercase, no spaces; keep stable to avoid integration and SEO issues.

### How to Create a Product Attribute
1) Go to Stores > Attributes > Product > Add New Attribute.
2) Set Default Label and Attribute Code.
3) Choose Catalog Input Type (for example, Visual Swatch for Color) and add options.
4) Set Scope = Global. Use in Layered Navigation = choose 'Filterable (with results)' or 'Filterable (no results)' if needed; set Use in Search Results Layered Navigation and Position as appropriate.
5) Save Attribute.
6) Assign the attribute to your Attribute Set(s): Stores > Attributes > Attribute Set > open your set > drag the attribute from Unassigned to the correct group > Save.

### How to Create or Update an Attribute Set
1) Go to Stores > Attributes > Attribute Set > Add Attribute Set.
2) Name it (for example, Apparel) and base it on Default.
3) Drag your attributes from Unassigned to the appropriate groups. Save.

### How to Create a Product
1) Go to Catalog > Products > Add Product (choose type).
2) Complete required fields: Name, SKU, Price, Weight (required for shippable products: Simple, Configurable children, Grouped items, and Bundle if Ship Separately), Tax Class (for example, Taxable Goods; confirm with your tax rules), Visibility (Catalog, Search), Websites.
3) Assign Categories and upload Images/Videos (set Base, Small, Thumbnail roles; set swatch images if applicable). For color swatches, upload swatch images per attribute option at Stores > Attributes > Product > [Color] > Manage Swatch (Values of Your Attribute). To use product images for swatches on PDPs, enable Stores > Configuration > Catalog > Catalog > Storefront > Use Product Image for Swatch if Possible = Yes. Attribute option swatches (set on the attribute) take precedence. The product image role Swatch is used when the config is enabled and no option-specific swatch overrides.
4) Inventory:
   - Single Source: Set Quantity and Stock Status.
   - Multiple Sources: Open Sources, Assign Sources, and set quantities per source.
   Note: Verify stock-to-website mapping at Stores > Inventory > Stocks. Ensure the product's assigned Source(s) belong to the Stock serving your website; otherwise, Salable Quantity will be 0.
5) For Configurable products:
   a) Prerequisite: Ensure variation attributes (for example, Size, Color) already have the full set of options created (Stores > Attributes > Product > [Attribute] > Manage Swatch/Options).
   b) Click Create Configurations and select variation attributes (only Global Dropdown/Visual/Text Swatch attributes with 'Use To Create Configurable Product' = Yes appear).
   c) Generate variations and set images/price/qty at the appropriate level.
   d) Set child product Visibility = Not Visible Individually to avoid duplicate listings.
6) Save and ensure product is Enabled.

### Inventory (MSI)
- Overview: Magento Inventory (MSI) supports Single Source and Multiple Source. A product is salable only if it has quantity at a Source that belongs to the website's Stock. Ensure products are assigned to at least one Source mapped to the website's Stock (Stores > Inventory > Stocks).
- Salable Quantity: Reflects reservations across orders; may differ from physical Quantity.
- Sources (Stores > Inventory > Sources): Physical locations with on-hand quantity.
- Stocks (Stores > Inventory > Stocks): Aggregate one or more Sources and assign them to sales channels (websites). Each website can be linked to only one Stock.
- UI: Catalog > Products > Edit Product > Sources panel > Assign Sources and set Qty per source.

### Best Practices
- Standardize attribute codes (lowercase, underscores) and maintain a catalog-wide naming convention.
- Define attribute sets per category (for example, Apparel, Electronics) to speed product onboarding.
- Make filterable facets Dropdown/Swatch to improve layered navigation performance and UX.
- Prioritize 4–6 high-intent filters per category based on search term reports, top faceted navigation interactions, and observed conversion lift (for example, Size, Color, Brand, Price, Fit) to maximize conversion without facet bloat.
- Keep variation attributes Global and limited to a few key facets (for example, Color, Size) to reduce SKU explosion.
- Align attributes with ERP/PIM field names where possible to streamline integrations.
- Search tuning: Only enable 'Use in Search' for attributes customers actually search (for example, SKU, Brand, MPN). Keep Search Weight high (6–10) only for 3–5 top-signal fields to avoid noise and index bloat.
- Configurable product quick checklist:
  - Variation attributes are Global, Dropdown/Visual/Text Swatch, and 'Use To Create Configurable Product' = Yes.
  - Attribute options (for example, size scale, color set) are created before product setup.
  - Child product Visibility = Not Visible Individually.
  - Images/price/qty set at the appropriate level (parent or child as needed).
  - Inventory assigned to Sources that belong to the website's Stock; Salable Qty > 0.
- Operational onboarding SOP:
  1) Define the attribute set per category and lock naming conventions.
  2) Pre-load option lists for key attributes (for example, Size scale, Brand, Color set).
  3) Map ERP/PIM fields to Magento attributes and verify codes/scopes match.
  4) Import a template SKU to validate images, prices, tax class, and inventory mapping.
  5) QA in a staging environment and monitor indexers during initial bulk import.

### Verify & Troubleshoot
- Product not visible: Check Status = Enabled, Visibility = Catalog/Search, assigned to correct Website and Category.
- Filters not showing:
  - Attribute type: Must be Dropdown or Multiple Select for attribute filters.
  - Attribute settings: Set Use in Layered Navigation to 'Filterable (with results)'. For search pages, set Use in Search Results Layered Navigation = Yes.
  - Stock display: If options disappear due to OOS, enable Stores > Configuration > Catalog > Inventory > Stock Options > Display Out of Stock Products.
  - Position: Adjust Position for filter order.
  - Reindex/cache: Update indexers and clear caches.
- Variation attributes missing: Ensure Scope = Global, input type is Dropdown/Visual/Text Swatch, and 'Use To Create Configurable Product' = Yes.
- MSI/Salability:
  - Confirm product is assigned to at least one Source with qty.
  - Confirm Sources are linked to the website's Stock (Stores > Inventory > Stocks).
  - Compare Salable Quantity vs Quantity to account for reservations.
  - MSI reservations: If Salable Qty is 0 but Quantity > 0, check reservations: run bin/magento inventory:reservation:list <ORDER_INCREMENT_ID>. If needed, compensate with bin/magento inventory:reservation:compensate <ORDER_INCREMENT_ID> after canceling/refunding orders. Ensure all order state transitions are completed and cron is running.
- Reindex/Cache: System > Tools > Index Management (indexes Up to date), System > Tools > Cache Management (flush if needed). Key indexers: Catalog Search (after attribute search changes), Product EAV (after attribute changes), Category Products (after category assignments), Product Price (after price changes), Inventory (after source/stock changes). Ensure Status = Ready and Mode = Update by Schedule, and cron is running.

Revenue Safeguards:
1) Validate Salable Qty > 0 for top SKUs before sale events.
2) Enable Display Out of Stock Products during launches to keep filters stable and avoid disappearing options.
3) Monitor Indexer status during imports to avoid empty categories or stale prices.

Quick verification checklist:
1) Status = Enabled and correct Visibility.
2) Websites and Categories assigned.
3) Attribute types and Layered Navigation settings correct (including Search Results setting and Position).
4) Variation attributes are Global and eligible for configurations.
5) Sources assigned with qty; Stock mapped to website; Salable Qty > 0.
6) Reindex and clear caches; verify cron.

## See Also

- Adobe Commerce Admin User Guide — Catalog > Products: https://experienceleague.adobe.com/docs/commerce-admin/catalog/products/products.html
- Product Attributes: https://experienceleague.adobe.com/docs/commerce-admin/catalog/product-attributes/product-attributes.html
- Attribute Sets: https://experienceleague.adobe.com/docs/commerce-admin/catalog/product-attributes/attribute-sets.html
- Layered Navigation: https://experienceleague.adobe.com/docs/commerce-admin/catalog/navigation/layered-navigation.html
- Product Types (overview): https://experienceleague.adobe.com/docs/commerce-admin/catalog/products/types/types.html

---

*This reference was last updated on 2025-09-11T17:15:57.531Z. For the most current information, please check the [official documentation](https://experienceleague.adobe.com/docs/commerce.html).*