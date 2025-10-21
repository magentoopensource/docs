---
title: "Promotions and Discounts Reference"
description: "Complete reference of promotional tools, cart rules, and discount mechanisms"
type: reference
audience: "Magento merchants"
tags:
  - reference
  - promotions
last_updated: "2025-09-12T07:03:48.570Z"
---

# Promotions and Discounts Reference

## Overview

Use Magento 2 promotions to drive conversion, average order value (AOV), and retention. This reference covers Cart Price Rules (checkout discounts), Catalog Price Rules (pre-cart price adjustments), coupons, free shipping, labels, and measurement. Applies to: Magento Open Source and Adobe Commerce 2.4.x; Adobe Commerce-only features are flagged. This page is a living reference and is updated regularly to reflect the latest capabilities.

Cart Price Rules apply discounts at checkout and can use coupons. Catalog Price Rules adjust product prices before the cart and do not use coupons.

### Promotion Types at a Glance
- Cart Price Rules (Open Source and Adobe Commerce): Discounts at checkout; supports coupons.
- Catalog Price Rules (Open Source and Adobe Commerce): Pre-cart pricing adjustments; no coupon codes.
- Free Shipping via rule (Open Source and Adobe Commerce): Use Cart Price Rule "Free Shipping" action.
- Coupon Auto-Generation (Open Source and Adobe Commerce): In Cart Price Rules, set Coupon = Specific Coupon and Use Auto Generation = Yes, Save the rule, then reopen and click Manage Coupon Codes.
- Scheduled Updates/Staging (Adobe Commerce only): Time-based versions of rules.
- Gift Cards (Adobe Commerce only): Create gift card products at Catalog > Products > Add Product > Gift Card. Manage balances at Marketing > Gift Card Accounts. Tip: Most merchants exclude gift card products from discounts via Cart Price Rule Actions > Conditions.
- Customer Segments (Adobe Commerce only): Target promotions by segment.

Edition differences:
- Open Source and Adobe Commerce: Cart Price Rules, Catalog Price Rules, Coupons, rule-based Free Shipping.
- Adobe Commerce only: Scheduled Updates/Staging, Customer Segments, Gift Cards.
- Adobe Commerce B2B: Shared Catalog custom prices are considered alongside Catalog Price Rules; the storefront uses the lowest applicable price (custom price vs. rule-adjusted price). Test with each Shared Catalog and customer group.

### Choose the right promotion for your goal
- Increase first purchase conversion: Cart Price Rule with Specific Coupon for new customers; limit Uses per Customer = 1. Business note: Drives conversion but watch discount depth to protect margin.
- Boost AOV:
  - Set a threshold 10–20% above current AOV. Example: AOV $80 → threshold about $90.
  - Use a percent discount (e.g., 10%).
  - Optionally add Free Shipping for qualifying carts.
  - A/B test two thresholds for 7 days and pick the higher net-margin result.
  Business note: Typically increases revenue per order; validate gross margin after discount and shipping subsidies.
- Move aging inventory: Catalog Price Rule targeting category/attribute for a fixed period. Business note: Moves overstock; monitor sell-through and margin.
- Reward loyalty/VIPs: Cart Price Rule scoped to selected Customer Groups. Business note: Improves retention and LTV; keep offers targeted.
- Stimulate multi-unit purchase: Buy X get Y (BXGY) Cart Price Rule limited to specific SKUs/categories. Business note: Increases units/order and clears specific lines; monitor gross margin per order.
- B2B (Adobe Commerce): Shared Catalog custom prices are considered alongside Catalog Price Rules; the storefront uses the lowest applicable price. Test with each Shared Catalog and customer group. Business note: Ensures negotiated pricing remains competitive; verify rule outcomes per account.

## Configuration

### Prerequisites
- Cron enabled and running (required for applying/scheduling Catalog Price Rules and some background tasks).
- Indexers healthy; production sites should use Update by Schedule. CLI checks: bin/magento indexer:status (to check status) and bin/magento indexer:show-mode (to confirm update mode).
- Catalog Price Scope: Stores > Configuration > Catalog > Catalog > Price > Catalog Price Scope (Global or Website). For multi-website setups with different prices/currencies, set to Website and test Catalog Rules per website.
- For Free Shipping promotions, ensure an eligible shipping method is configured:
  - Go to Stores > Configuration > Sales > Shipping Methods > Free Shipping
  - Set Enable = Yes
  - Keep your normal Minimum Order Amount in place. When a Cart Price Rule grants Free Shipping, the Free Shipping method will be offered even if the Minimum Order Amount is higher. Do not set this to 0 unless you intend to offer free shipping on all orders.
  - Save config and clear cache
  - Then, in the Cart Price Rule, set Free Shipping = "For shipment with matching items" (or "For matching items only" as needed)
  - Note: Many third‑party carrier methods ignore Magento's free-shipping flag. If you use external carrier integrations, test that the Free Shipping method appears and is selected as expected.
  - Note: "Apply to Shipping Amount" includes shipping in the discount calculation; it does not make shipping free. To make shipping free, set Actions > Free Shipping and enable the Free Shipping carrier.
- Permissions: User role with access to Marketing > Promotions > Cart Price Rules and Catalog Price Rules.

### Cart Price Rules (Admin)
Path: Admin > Marketing > Promotions > Cart Price Rules > Add New Rule

1) Rule Information
- Rule Name: [e.g., 10% off over $100]
- Description: [internal notes]
- Websites: [select]
- Customer Groups: [e.g., NOT LOGGED IN, General]
- Coupon: Select "No Coupon" or "Specific Coupon". If "Specific Coupon", enter Coupon Code. Set Use Auto Generation = Yes to mass-generate codes.
- Uses per Coupon / Uses per Customer: [enter limits]
- From/To: Date-only (inclusive). Time-of-day scheduling is not supported here; use Scheduled Updates (Adobe Commerce) for start/end times.
- Priority: [lower number = higher priority]
- Public in RSS Feed: [Yes/No] (visible only if RSS is enabled at Stores > Configuration > Catalog > RSS Feeds)

2) Conditions
- Define cart conditions (e.g., Subtotal greater than 100). Conditions must be true for the rule to trigger.

3) Actions
- Apply: [Percent of product price discount | Fixed amount discount | Fixed amount discount for whole cart | Buy X get Y]
- Discount Amount: [value]
- Maximum Qty Discount is Applied To: [cap]
- Discount Qty Step (Buy X): [for BXGY]
- Apply to Shipping Amount: [Yes/No]
- Free Shipping: [No | For matching items only | For shipment with matching items]
- Stop Further Rules Processing: [Yes/No]
- Conditions: [limit the products affected by the discount]

Note: "Apply to Shipping Amount" includes shipping in the discount calculation; it does not make shipping free. To make shipping free, set Free Shipping and enable the Free Shipping carrier under Stores > Configuration > Sales > Shipping Methods > Free Shipping.

4) Labels
- Default Rule Label for All Store Views and optional per-store view labels.

5) Save the rule.

Tips
- Use the rule Priority and Stop Further Rules Processing to control stacking with other rules.
- Ensure Websites and Customer Groups selections match your intended audience.
- Scope: Rule Conditions = cart-level trigger. Actions > Conditions = which items receive the discount.
- For configurable and bundle products, use Actions > Conditions to target the specific items discounted (child SKUs). Rule-level Conditions evaluate the cart as a whole; Actions > Conditions limit which items receive the discount.
- Bundles: For Dynamic Price bundles, discounts apply to child items; for Fixed Price bundles, discounts apply to the parent only. Configurables: Discounts are applied to the simple child selected at add-to-cart.
- Example: To discount only items in Category = Tees when the cart subtotal is over $100, put "Subtotal greater than 100" under Conditions (cart scope) and put "Category is Tees" under Actions > Conditions (item scope).
- Adobe Commerce only: Save the rule, then open the Customer Segments tab on the rule to include/exclude segments (segments are created at Customers > Segments).

### Catalog Price Rules (Admin)
Path: Admin > Marketing > Promotions > Catalog Price Rules > Add New Rule

1) Rule Information
- Name, Description, Websites, Customer Groups, From/To dates, Priority.

2) Conditions
- Product attribute conditions (e.g., Category is 12; Attribute Set is "Shoes").

3) Actions
- Apply: [Apply as percentage of original | Apply as fixed amount | Adjust final price to discount value | Adjust final price to this percentage]
- Discount Amount: [value]
- Stop Further Rules Processing: [Yes/No]

4) Labels
- Default and per-store view labels.

5) Save and Apply
- Click Save and Apply to rebuild rule index. Applying and scheduled activations require cron.
- Relevant indexers: catalogrule_rule and catalogrule_product. If needed, reindex via CLI: bin/magento indexer:reindex catalogrule_rule catalogrule_product
- Performance note: Applying Catalog Price Rules can trigger heavy indexing on large catalogs. Schedule updates during low-traffic windows and ensure cron frequency is adequate.

Notes
- Catalog Price Rules adjust product prices before they enter the cart; no coupons are used.
- Price interactions: Catalog Price Rules are calculated from the Original (base) price. The storefront uses the lowest applicable price among: Special Price, Tier Price, Group Price (and Shared Catalog custom price in B2B), and the Catalog Rule adjusted price. "Stop Further Rules Processing" affects stacking with other Catalog Rules only; it does not prevent comparison against Special/Tier/Group/Shared Catalog prices.
- Special/Tier/Group prices can influence the resulting final price. Test outcomes on products that already have Special, Tier, or Group prices to confirm the intended result.

### Coupons
Within a Cart Price Rule (Admin > Marketing > Promotions > Cart Price Rules):
- Coupon: Select "Specific Coupon" to enable coupon fields.
- Coupon Code: Enter a manual code OR set Use Auto Generation = Yes.
- Manage Coupon Codes: After saving the rule, open it again and click Manage Coupon Codes to:
  - Generate: Specify Quantity, Code Length, Code Format, Prefix/Suffix, and Dash Every X Characters.
  - Export: Download generated codes as CSV for marketing use.
- Strategy: Use auto-generated, unique single-use codes for influencer/affiliate channels to prevent leakage. Set Uses per Customer = 1 and Uses per Coupon = 1 for limited offers. For sitewide promos, use a generic code with strict end dates and priority settings.
- Leakage control: For high-value offers, require login by targeting Customer Groups (exclude NOT LOGGED IN) and/or Customer Segments (Adobe Commerce). Pair with Uses per Customer = 1.
- Tracking: Reports > Sales > Coupons shows usage by rule and code.
- Limitation: Only one coupon code can be applied to a cart at a time. Additional codes replace the existing one. Use rule Priority and Stop Further Rules Processing to control stacking with automatic (no-coupon) rules.
- Ensure coupon codes are unique across all rules and websites. Duplicate codes in active rules can cause unexpected validation failures.

### Labels and Store Views
- Use clear, customer-friendly labels (e.g., "10% off orders $100+") under Labels.
- Configure per-store view labels when wording should differ by locale or brand.

### Best-practice guardrails
- Set Uses per Coupon and Uses per Customer to prevent abuse.
- Use Conditions in Actions to exclude low-margin or MAP-restricted products.
- Prefer percent discounts over fixed amounts for mixed-price catalogs.
- Cap Maximum Qty Discount is Applied To for BXGY and bulk discounts.
- Use Priority and Stop Further Rules Processing to avoid unintended stacking.
- Timebox promotions with From/To dates and review overlap.
- Margin protection: Exclude low-margin SKUs or categories, prevent double-discounts by setting Stop Further Rules Processing = Yes on key rules, and avoid stacking with sitewide promotions.
- Quick margin check example: If your average gross margin is 40% and shipping subsidy per order is $5, on a $100 AOV order a 10% discount reduces revenue by $10. New gross margin dollars ≈ ($100 − $10) × 40% − $5 = $25. Compare this to baseline ($100 × 40% − $5 = $35). Adjust discount or threshold until post-discount margin dollars meet your target.

### Test and Verify
- Create a test customer in each targeted customer group.
- For Cart Price Rules: Add qualifying products; verify the discount and rule label appear in cart/checkout totals.
- For Catalog Price Rules: After Save and Apply, confirm cron runs and, if needed, reindex via CLI; verify frontend prices reflect the discount. Note: Catalog rule labels are not rendered on the storefront by default and typically require theme customization.
- Verify tax behavior: Stores > Configuration > Sales > Tax > Calculation Settings > "Apply Discount On Prices" (Excluding/Including Tax) matches policy.
- Dates are inclusive and start at 00:00 store time. Cart and Catalog rules do not support time-of-day; to activate/deactivate at a specific hour, use Scheduled Updates (Adobe Commerce) or enable/disable the rule manually.
- For multi-website setups: Test in each website/store view with its currency, validate rounding on catalog rule prices post currency conversion, and confirm tax settings per website (Stores > Configuration > Sales > Tax) match your discount calculation policy.
- Multi-currency: Rule thresholds and calculations use the website base currency. Validate rounding and thresholds per website and storefront display currency.
- With multiple rules, confirm Priority and Stop Further Rules Processing produce intended results.
- Note: Apply to Shipping Amount does not make shipping free. It only includes shipping in the discount calculation. To make shipping free, use Actions > Free Shipping.

### Reporting and Measurement
- Coupons usage report: Admin > Reports > Sales > Coupons. Track redemptions, revenue, and AOV.
- Key KPIs: redemption rate, incremental revenue, AOV lift, margin impact, and attach rate of promoted items.
- For Catalog Rules without coupons: Compare period-over-period AOV and conversion on affected categories/SKUs, analyze cohorts during rule windows, and monitor margin impact.
- Channel attribution: Tag campaigns (UTMs) and map orders to codes/channels to understand performance and reduce leakage. Combine coupon redemption data with order cohorts to isolate incremental lift versus baseline.
- If your instance uses Report Statistics, go to Reports > Statistics and Refresh to ensure current data. Export coupon reports to CSV and join with UTM-tagged order data for channel-level ROI.
- Name codes with campaign prefixes (e.g., IG-SUMMER10) and tag links with UTMs. This enables channel-level ROI in reports when you export coupon usage and join to order sources.
- Export report data to analyze performance by channel or code batch.

### API Quick Reference (REST)
- Apply coupon to customer cart: PUT /V1/carts/mine/coupons/{couponCode}
- Remove coupon from customer cart: DELETE /V1/carts/mine/coupons
- Apply coupon to guest cart: PUT /V1/guest-carts/{cartId}/coupons/{couponCode}
- Remove coupon from guest cart: DELETE /V1/guest-carts/{cartId}/coupons
- Auth: /V1/carts/mine/* endpoints require a customer token (Authorization: Bearer <token>).
- Guest flow: Create a cart via POST /V1/guest-carts to obtain cartId, then call the guest coupon endpoints with that cartId.
- Developer reference for SalesRule and CatalogRule endpoints: https://developer.adobe.com/commerce/webapi/rest/
- GraphQL: Apply coupon: mutation applyCouponToCart(input: { cart_id: "<cartId>", coupon_code: "<code>" }) { cart { applied_coupons { code } } } | Remove coupon: mutation removeCouponFromCart(input: { cart_id: "<cartId>" }) { cart { applied_coupons { code } } } | Docs: https://developer.adobe.com/commerce/webapi/graphql/

### Troubleshooting
- Rule not applying: Verify Websites and Customer Groups; ensure the cart meets both Conditions and Actions filters.
- Coupon invalid: Check From/To dates, Uses per Coupon/Customer limits, and code spelling/case.
- Catalog Rule price not updated: Click Save and Apply, confirm cron is running, reindex catalogrule_rule and catalogrule_product, then clear full-page cache. If storefront prices still do not reflect changes, purge Varnish/CDN and browser cache (System > Tools > Cache Management > Flush Magento Cache) and ensure your deployment purges external caches.
- Discounts stacking unexpectedly: Set Stop Further Rules Processing = Yes on the highest-priority rule.
- Rule order: Lower Priority number runs first. "Stop Further Rules Processing" stops only lower-priority rules after the current rule applies. If the rule doesn’t apply, processing continues.
- Tax discrepancy: Review "Apply Discount On Prices" and "Apply to Shipping Amount" in Actions.

### Promotion recipes (copy-ready examples)
1) 10% off orders over $100 with coupon
- Type: Cart Price Rule
- Rule Information: Coupon = Specific Coupon (e.g., SAVE10), Uses per Customer = 1, Priority = 1
- Conditions: Subtotal greater than 100
- Actions: Apply = Percent of product price discount; Discount Amount = 10; Apply to Shipping Amount = No; Stop Further Rules Processing = Yes
- Labels: "10% off $100+"

2) Buy 2 get 1 free on Category "Tees"
- Type: Cart Price Rule
- Rule Information: No Coupon; Priority = 1
- Conditions (Rule Conditions): [leave blank, or add cart-level limits as needed]
- Actions:
  - Apply = Buy X get Y (discount amount is Y)
  - Discount Qty Step (Buy X) = 2
  - Discount Amount (Y) = 1
  - Maximum Qty Discount is Applied To = 0 (no limit) or set a cap as desired
  - Actions > Conditions: Category is Tees (ensures both X and Y items are from Tees)
- Labels: "Buy 2, get 1 free — Tees"
- Note: The free item is the lowest-priced matching item.

3) 15% off Clearance for 7 days (Catalog)
- Type: Catalog Price Rule
- Rule Information: From/To = [7-day window], Priority = 1
- Conditions: Product attribute "Clearance" = Yes (or Category = Clearance)
- Actions: Apply = By Percentage of the Original Price; Discount Amount = 15; Stop Further Rules Processing = Yes
- Labels: "15% off Clearance"
- Scheduling note: For staged activations with versioning, use Scheduled Updates (Adobe Commerce only).

## See Also

- Adobe Commerce Promotions overview: https://experienceleague.adobe.com/docs/commerce-admin/marketing/promotions/promotions.html
- Cart Price Rules: https://experienceleague.adobe.com/docs/commerce-admin/marketing/promotions/cart-price-rules/cart-price-rules.html
- Catalog Price Rules: https://experienceleague.adobe.com/docs/commerce-admin/marketing/promotions/catalog-price-rules/catalog-price-rules.html
- Coupons Report: https://experienceleague.adobe.com/docs/commerce-admin/reports/sales/coupons.html
- REST API (Coupons and Rules): https://developer.adobe.com/commerce/webapi/rest/

---

*This reference was last updated on 2025-09-12T07:03:48.570Z. For the most current procedures and UI, see Adobe Commerce Promotions documentation: https://experienceleague.adobe.com/docs/commerce-admin/marketing/promotions/promotions.html.*
