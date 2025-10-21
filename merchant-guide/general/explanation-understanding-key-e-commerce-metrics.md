---
title: "Understanding Key E-commerce Metrics"
description: "Explanation of important KPIs and how to interpret business data for growth"
type: explanation
audience: "Magento merchants"
tags:
  - explanation
last_updated: "2025-09-12T10:01:36.100Z"
---

# Understanding Key E-commerce Metrics

## Introduction

Focus on a handful of KPIs that directly impact revenue and customer experience. This guide shows which metrics to track, how to calculate them with Magento data, and how to run a weekly review that turns insights into actions. By the end, you can select the right KPIs for your stage, read what they mean, and make confident, ROI-driven decisions.

Quick start this quarter:
- Conversion Rate (CVR) = Orders / Sessions. Data: Orders (Admin > Reports > Sales > Orders); Sessions (analytics, e.g., GA4).
- Average Order Value (AOV) = Revenue / Orders. Data: Admin > Reports > Sales > Orders.
- Refund Rate = Refunded orders / Total orders. Data: Admin > Reports > Sales > Refunds.
- Repeat Purchase Rate (RPR) = Customers with ≥2 orders / Total customers. Data: Admin > Customers + Orders export.
- Revenue vs Target = Actual revenue vs plan. Data: Admin > Reports > Sales > Orders (align on gross/net definition).

## Key Concepts

Use this dictionary to standardize how your team calculates KPIs and where to find inputs in Magento.

- Conversion Rate (CVR)
  - Formula: Orders / Sessions.
  - Magento data: Orders (Admin > Reports > Sales > Orders). Sessions from your analytics tool (e.g., GA4).
  - Notes: Filter out test orders and bot sessions.

- Average Order Value (AOV)
  - Formula: Revenue / Orders.
  - Magento data: Revenue from Admin > Reports > Sales > Orders. Decide whether to use Grand Total excluding tax/shipping.
  - Notes: Define gross vs net (tax, shipping, discounts) and stay consistent.

- Gross Merchandise Value (GMV)
  - Formula: Sum of order totals over a period (typically gross before refunds).
  - Magento data: Admin > Reports > Sales > Orders.

- Customer Acquisition Cost (CAC)
  - Formula: Marketing spend / New customers acquired.
  - Magento data: New customers (Admin > Reports > Customers > New). Spend from ad platforms.

- Customer Lifetime Value (CLV)
  - Simple estimate: AOV × Purchase Frequency (orders per customer per year) × Gross Margin % × Average Customer Lifespan (years).
  - Magento data: AOV and orders/customer from Reports and customer exports; margin from your finance system.

- Repeat Purchase Rate (RPR)
  - Formula: Customers with ≥2 orders / Total customers in period.
  - Magento data: Admin > Customers and Orders export.

- Cart Abandonment Rate
  - Formula: (Carts created − Orders) / Carts created.
  - Magento data: Adobe Commerce: Reports > Marketing > Products in Cart; otherwise use analytics events.

- Checkout Abandonment Rate
  - Formula: (Checkout sessions − Orders) / Checkout sessions.
  - Data: Analytics checkout funnel (e.g., GA4).

- Refund Rate
  - Formula: Refunded orders / Total orders.
  - Magento data: Admin > Reports > Sales > Refunds.

- Return Rate (RMA, Adobe Commerce only)
  - Formula: RMAs / Orders.
  - Magento data: RMA feature (Stores > Configuration > Sales > RMA Settings) and RMA reports/exports.

For each KPI, document: the exact formula (gross/net rules), the data source, an owner, a review cadence, and a target.

## How It Works

Follow this operating cadence to select, measure, and review KPIs using Magento data:

1) Set targets
- Pick up to 5 KPIs aligned to your current goal (e.g., raise CVR, increase AOV, reduce refunds).

2) Define calculation rules
- Write formulas and scope: include/exclude tax, shipping, discounts, canceled/unpaid orders, and refunds.

3) Configure data sources
- Magento Admin reports: Admin > Reports > Sales / Customers / Products.
- Analytics: GA4 (sessions, funnels, attribution).
- Ad platforms: spend for CAC/ROAS.

4) Ensure data quality
- Admin > Reports > Statistics: Make sure statistics refresh via cron (or refresh manually on schedule).
- Stores > Configuration > General > General > Locale Options: Confirm timezone matches analytics.
- Stores > Configuration > General > Currency Setup: Confirm base/allowed currencies.
- Exclude test orders and filter bots in analytics.

5) Build a dashboard
- Weekly KPI view with trend vs target, owner, and notes. Keep it lightweight and consistent.

6) Run a weekly review (30 minutes)
- Inspect deltas vs target, find root causes, and assign actions directly in Magento (e.g., promotions, payment methods, checkout settings).
- Timebox owners and due dates.

7) Validate numbers
- Cross-check Magento revenue vs analytics and payment provider disbursements; investigate >5% variance and document acceptable thresholds.

## Benefits and Advantages

Understand the business outcomes you can expect from a focused KPI practice.

- Faster decisions: Clear targets and a weekly review reduce time-to-action.
- Higher revenue: Prioritize fixes that raise CVR and AOV.
- Lower acquisition costs: Optimize spend using CAC vs CLV by channel.
- Operational efficiency: Fewer ad-hoc reports; one shared KPI language across teams.

## Common Challenges and Considerations

Avoid these pitfalls to keep your KPIs trustworthy and actionable.

- Data alignment: Use the same timezone and currency across Magento and analytics.
- Order states: Exclude canceled and test orders from revenue. Decide whether to include pending/unpaid orders.
- Refunds/chargebacks: Decide if revenue is net of refunds; reconcile with Reports > Sales > Refunds.
- Multi-store: Always use the store-view switcher to avoid mixing data across stores.
- Bot traffic: Filter bots in analytics to avoid inflated sessions and deflated CVR.
- Coupon impact: Large discounts reduce AOV and margin; analyze with Reports > Sales > Coupons (Adobe Commerce) or order exports.
- Analytics sampling: Avoid sampled reports when computing KPIs.

## Use Cases and Scenarios

Map KPI movements to concrete Magento actions that drive revenue.

Scenario 1: CVR below target
- Diagnose
  - Review GA4 landing-page bounce and checkout-funnel drop-offs.
- Actions in Magento
  - Enable additional payment methods to reduce friction: Stores > Configuration > Sales > Payment Methods.
  - Simplify checkout fields: Stores > Configuration > Sales > Checkout.
  - Improve on-site search relevance: Marketing > SEO & Search > Search Terms and Search Synonyms; adjust redirects and synonyms.

Scenario 2: Low AOV
- Diagnose
  - Identify bundling and attachment opportunities: Reports > Products > Bestsellers and (Adobe Commerce) Reports > Marketing > Products in Cart.
- Actions in Magento
  - Add Related Products, Up-Sells, and Cross-Sells: Catalog > Products > Edit a product > Related Products, Up-Sells, and Cross-Sells.
  - Offer a free shipping threshold slightly above current AOV: Stores > Configuration > Sales > Shipping Methods > Free Shipping.
  - Create cart price rules for bundles or thresholds: Marketing > Promotions > Cart Price Rules.

Scenario 3: High refund/return rate
- Diagnose
  - Identify top-returned SKUs and reasons: Reports > Sales > Refunds; analyze order and RMA data (Adobe Commerce).
- Actions in Magento
  - Improve PDP sizing/fit info and attributes: Catalog > Attributes > Product and product content.
  - Update return policy and expectations: Content > Pages.
  - For Adobe Commerce with RMA: Configure RMA reasons and workflows: Stores > Configuration > Sales > RMA Settings.

## Alternatives and Comparisons

Choose the right tools to measure and act on your KPIs.

- Magento Admin Reports: Built-in operational reports; best for day-to-day store metrics and order-level truths.
- Analytics platforms (e.g., GA4): Session-based metrics and funnels; best for CVR and channel attribution.
- Adobe Commerce Business Intelligence (MBI, paid): Centralized data modeling and dashboards; best for CLV cohorts and multi-source blending.
- BI tools (Looker Studio, Power BI): Flexible reporting when connected to the Magento database or scheduled exports.

## Business Impact

Quantify how better KPIs translate to outcomes for each audience.

### For Merchants

- Raise revenue by prioritizing initiatives with highest impact on CVR and AOV.
- Allocate ad spend based on CAC vs CLV by channel.
- Example: With 100,000 monthly sessions and AOV $80, a 0.3 pp CVR lift (2.0% → 2.3%) adds ~300 orders and ~$24,000 in monthly revenue.

### For Customers

- Faster, simpler checkout and more relevant offers through data-driven improvements.

### For Operations

- Predictable inventory and staffing via stable KPI tracking and forecasts.

## Common Misconceptions

Clear up myths that lead to poor decisions.

- "More traffic always equals more revenue": Not if CVR is low. Fix the funnel first.
- "Revenue equals orders": Refunds and cancellations matter; track net revenue if that’s your definition.
- "CLV is too complex": A simple estimate can guide smarter CAC caps.
- "All channels contribute equally": Use attribution and cohort analysis to see real contribution.

## Implementation Considerations

Check these prerequisites, edition notes, and verification steps before you rely on KPIs.

- Prerequisites: Cron enabled so Reports > Statistics can refresh; user roles with access to Reports; analytics tracking installed and validated.
- Version/edition notes: Some reports (e.g., RMAs, Abandoned Carts, Products in Cart) require Adobe Commerce.
- Verification: Monthly reconciliation of Magento revenue vs payment provider disbursements; document acceptable variance thresholds (e.g., <2%).
- Governance: Maintain a KPI dictionary and change log whenever formulas or data sources change.

## Related Concepts

Explore supporting practices that make KPIs more actionable.

- Cohort analysis (new vs returning customers)
- Segmentation (high-CLV segments)
- A/B testing and experimentation
- Attribution models (last-click vs data-driven)

## Further Reading

Deepen your knowledge with vendor-maintained resources.

- Commerce Admin Reports overview (Adobe Experience League): https://experienceleague.adobe.com/en/docs/commerce-admin/reports
- GA4 eCommerce measurement guide (Google): https://developers.google.com/analytics/devguides/collection/ga4/ecommerce
- Adobe Commerce Business Intelligence (MBI) overview: https://experienceleague.adobe.com/en/docs/commerce-business-intelligence/mbi/overview

---

*Understanding these concepts is crucial for making informed decisions about your Magento implementation. For practical guidance, see our related how-to guides: https://experienceleague.adobe.com/en/docs/commerce-admin/reports and tutorials: https://experienceleague.adobe.com/en/docs/commerce-learn/tutorials/overview.*