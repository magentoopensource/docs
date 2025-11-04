---
title: "Understanding Customer Segmentation and Targeting"
description: "Explanation of Customer Groups (Open Source and Commerce), Customer Segments (Adobe Commerce), behavioral targeting with Dynamic Blocks, and personalization strategies."
type: explanation
audience:
  - "Magento merchants"
  - "Advanced users"
tags:
  - explanation
  - customer-management
last_updated: "2025-09-16T13:47:08.894Z"
---

# Understanding Customer Segmentation and Targeting

## Introduction

Customer segmentation and targeting help you present the right prices, content, and offers to the right shoppers at the
right time. In Magento Open Source, use Customer Groups, price rules, and CMS techniques for basic
targeting. Effective targeting increases conversion rate, average order value, and customer lifetime value while
reducing discount waste.

Note: Customer Segments and Dynamic Blocks are available in Adobe Commerce. Magento Open Source merchants can use
Customer Groups and price rules, or third‑party extensions, to approximate some targeting.

## Key Concepts

- Customer Groups (Open Source and Commerce): Static assignment of customers used primarily for pricing, tax classes,
  and access settings. Typically maintained by admin or integrations.
- Customer Segments (Adobe Commerce): Dynamic audiences built from rules that evaluate customer behavior and
  attributes (e.g., browsing, order history, cart contents, location). Can target registered customers and visitors (
  guests).
- Dynamic Blocks (Adobe Commerce): Content blocks whose visibility is controlled by Customer Segment rules. Created
  under Content and placed with Page Builder on pages, blocks, or categories.
- Page Builder: Visual editor used to assemble and place content, including Dynamic Blocks, without code.
- Cart Price Rules: Promotions that can be limited to specific Customer Segments (Adobe Commerce) or Customer Groups (
  Open Source and Commerce).
- Related Product Rules (Adobe Commerce): Rule-based merchandising for upsell/cross-sell; can complement segment
  targeting.
- Caching and personalization: Full Page Cache/Varnish work with Dynamic Blocks to render per-segment content without
  disabling cache.
- Scope: Customer Segments are scoped by website and can apply to Registered Customers and/or Visitors.

## How It Works

Quick flow: Create Segment (Customers > Segments) -> Build Dynamic Block (Content > Elements > Dynamic Blocks) -> Place
via Page Builder on Pages/Categories -> (Optional) Limit a Cart Price Rule to the segment -> Test with matched
accounts/sessions.

Adobe Commerce end-to-end workflow

1) Create a Customer Segment

- Go to Admin: Customers > Segments > Add Segment.
- Enter Segment Name, set Status to Active, choose Website scope, and set Apply To: Registered Customers or Visitors (
  guests).
- In Conditions, define rules (examples: Customer Address, Order History, Shopping Cart, Product Browsing).
- Save. Use Matched Customers to verify registered customer membership.

2) Build targeted content with a Dynamic Block

- Go to Content > Elements > Dynamic Blocks > Add Dynamic Block.
- Enter a Title. In Display Settings, select the Customer Segments that should see this block.
- Add promotional content (banner, copy, CTA) using Page Builder elements.

3) Place the Dynamic Block on a page or category

- For a CMS page: Content > Pages > Edit > Content. Use Page Builder to drag the Dynamic Block content type into the
  layout and choose your Dynamic Block.
- For a category: Catalog > Categories > Content. Use Page Builder to add the Dynamic Block.
- Save and, if needed, refresh caches under System > Cache Management.

4) (Optional) Target a promotion to a segment

- Go to Marketing > Promotions > Cart Price Rules > Add New Rule.
- Define rule basics and conditions. In the Customer Segments section, choose the segments that qualify for the rule (
  Adobe Commerce).

5) Test and verify

- Create a test customer that meets the segment conditions.
- View the storefront in a private window, sign in (or browse as guest if the segment applies to visitors), and confirm
  the targeted content and/or promotion displays as expected.

Magento Open Source approximation

- Use Customer Groups and CMS Blocks with Widgets to place group-restricted messaging.
- Limit Cart/Catalog Price Rules by Customer Group.
- Consider third‑party extensions for behavioral segmentation if needed.

## Benefits and Advantages

- Higher conversion rate: Show relevant offers to shoppers with clear purchase intent.
- Increased average order value: Cross-sell based on cart contents or browsing history.
- Reduced discount leakage: Limit promotions to qualified segments instead of sitewide.
- Faster go-to-market: Launch targeted campaigns without code changes using Page Builder.
- Improved customer experience: Personalized content for new vs returning customers, VIPs, or B2B accounts.

## Common Challenges and Considerations

- Caching: With Full Page Cache/Varnish, use Dynamic Blocks and Page Builder for per-segment rendering; avoid hardcoding
  conditional content in templates.
- Guest segmentation: Conditions for visitors are session-based and harder to validate; test in incognito and across
  browsers.
- Data freshness: Segment membership updates after conditions run; allow time and reindex if needed.
- Scope: Segments are scoped by website; verify the correct website in multi-site setups.
- Privacy/consent: Ensure your cookie and privacy policy covers behavior-based targeting (GDPR/CCPA). Avoid PII in
  segment names.
- Fall-back content: Always provide default content for users who do not match any segment.

## Use Cases and Scenarios

- Cart value booster (quick win): If cart subtotal > $100, show free shipping threshold reminder or bundle offer.
- Category retargeting (quick win): If a shopper viewed >3 products in Running Shoes, show a 10% welcome offer on that
  category for first-time buyers.
- VIP loyalty (quick win): Customers with lifetime value > $500 see early-access banners and higher-tier discounts.
- Replenishment: Past purchasers of consumables 30–60 days ago see a reorder CTA on the homepage.
- B2B pricing nudges: Logged-in customers in Wholesale group see volume discount blocks and MOQ messaging.
- Seasonal geo-targeting: Visitors from specific regions see localized shipping cut-off dates.
- New vs returning: New visitors see educational content; returning customers see curated recommendations.

## Alternatives and Comparisons

- Magento Open Source: Use Customer Groups + Cart/Catalog Price Rules, coupon codes, and CMS Blocks with Widgets for
  basic targeting. Consider third‑party extensions for behavioral segmentation.
- Adobe Commerce: Use Customer Segments + Dynamic Blocks + Related Product Rules for native targeting. Enhance with Live
  Search and Product Recommendations for AI‑driven relevance.
- External tools: Connect a CDP or experimentation tool for advanced audience building and A/B testing.

## Business Impact

### For Merchants

- Revenue: Targeted offers can lift conversion by 5–15% and AOV by 3–10% depending on category and seasonality.
- Margin: Reduce sitewide discounting by limiting offers to high-intent segments.
- Speed: Launch new campaigns in hours via Page Builder without developer cycles.
- Example ROI model: If monthly sessions = 200,000, baseline CR = 2.0%, AOV = $80, revenue = $320,000. A targeted
  cart-abandoner segment lifting CR to 2.2% (+10% relative) adds ≈$32,000/month. Limiting a 10% discount to high-intent
  segments can improve gross margin by 1–2 points.

### For Customers

- Relevance: See content and offers aligned with intent (e.g., accessories related to items in cart).
- Clarity: Clear next-best actions reduce friction and improve satisfaction.

### For Operations

- Efficiency: Reusable segments and blocks reduce campaign setup time.
- Governance: Edition-scoped features, approvals, and content staging support controlled rollouts.

## Common Misconceptions

- “Customer Groups and Customer Segments are the same.” False—groups are static for pricing/tax; segments are dynamic
  and behavior-based.
- “Personalization works the same for guests and logged-in users.” False—registered customers support richer conditions
  and verification.
- “Full page caching breaks personalization.” Not if you use Dynamic Blocks and Page Builder, which support per-segment
  rendering.

## Implementation Considerations

Prerequisites

- Edition: Adobe Commerce for Customer Segments and Dynamic Blocks.
- Access: Admin role with permissions for Customers, Marketing, and Content.
- Theme: Page Builder enabled for easiest placement.

Checklist

- Define business goals and KPIs (e.g., +5% CR in category X).
- Create segments; document conditions and website scope.
- Build Dynamic Blocks with default fallbacks.
- Place blocks in Pages/Categories where intent is strongest.
- Configure Cart Price Rules if promotional.
- QA in staging with test accounts and guest sessions.
- Deploy with Content Staging if time-bound; monitor results and iterate.

Rollback

- Keep default content active; disable Dynamic Blocks or segments to revert instantly.

## Related Concepts

- Marketing > Promotions > Cart Price Rules
- Content Staging and Page Builder
- Customer Groups and Shared Catalog (B2B)

*Understanding these concepts is crucial for making informed decisions about your Magento implementation. For practical
guidance, review the Further Reading section and related how-to topics such as Create a Customer Segment, Create a
Dynamic Block, and Place a Dynamic Block with Page Builder.*