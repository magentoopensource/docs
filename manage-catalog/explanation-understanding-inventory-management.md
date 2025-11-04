---
title: "Understanding Inventory Management"
description: "Explanation of stock tracking, multi-source inventory, and inventory strategies"
type: explanation
audience: "Magento merchants"
tags:
  - explanation
  - inventory
  - msi
  - fulfillment
  - shipping
  - product-management
  - operations
last_updated: "2025-09-10T14:32:45.962Z"
---

# Understanding Inventory Management

## Introduction

Inventory Management (MSI) in Magento Open Source and Adobe Commerce controls product availability, allocation, and
fulfillment across one or many locations (Sources). By understanding Salable Quantity, Stocks (website-level
availability), and Source Selection Algorithms, you can increase in-stock rate, reduce shipping costs, and fulfill
faster.

Applies to: Magento Open Source and Adobe Commerce 2.4.x. Inventory Management (MSI) is available in 2.3 and later and
is enabled by default.

## Key Concepts

This section highlights the core MSI entities and terminology used throughout the Admin and storefront.

- SKU: Unique identifier for a sellable product.
- Source: A physical location that stores inventory (for example, warehouse, store, dropship vendor). Admin path:
  Stores > Inventory > Sources.
- Source Item: The quantity and status of a specific SKU at a specific Source.
- Stock: A virtual grouping of one or more Sources mapped to a Sales Channel (website). Admin path: Stores > Inventory >
  Stocks.
- Sales Channel: A website that sells products. Stocks are assigned to websites to control availability per site.
- Salable Quantity: The sellable amount of a SKU on a Stock (website). It aggregates quantities across assigned Sources
  and subtracts reservations and thresholds.
- Reservation: A record created at order placement that decreases Salable Quantity to prevent overselling before
  shipment.
- Source Selection Algorithm (SSA): Logic that chooses which Sources fulfill a shipment (for example, Source Priority,
  Distance Priority).
- Backorders: Option to continue selling when Salable Quantity is at or below zero.

Callout: A Source is a physical place (warehouse/store). A Stock is a logical group of Sources that serves a website (
Sales Channel). Products are sellable on a website if their assigned Sources belong to that website’s Stock.

## How It Works

From order placement to shipment, MSI allocates inventory and updates availability in a predictable lifecycle.

1. Order placement: When a customer places an order, Adobe Commerce/Magento Open Source creates a Reservation that
   reduces the SKU’s Salable Quantity on the assigned Stock (website). Physical Source Item quantities do not change
   yet.
2. Shipment: When you create a shipment, the system applies the Source Selection Algorithm to pick one or more Sources
   and deducts Source Item quantities accordingly. The corresponding Reservation is released.
3. Cancel/Refund: If an order is canceled or refunded before shipment, the system writes a compensating Reservation to
   restore Salable Quantity.
4. Salable Quantity calculation: Salable Quantity ≈ Σ(available Source Item qty across the Stock’s Sources) − Σ(active
   reservations) − Out-of-Stock Threshold. If Backorders are enabled, Salable Quantity can go negative.
5. Where to view: Add the Salable Quantity column in Catalog > Products (Columns > Salable Quantity) to monitor
   website-level availability.

## Benefits and Advantages

MSI features map directly to measurable business outcomes you can track and optimize.

- Lower shipping costs by shipping from the nearest Source using Distance Priority SSA (typical 5–15% savings).
- Higher in-stock rate via aggregated inventory across multiple Sources, improving conversion.
- Faster fulfillment by routing orders to Sources with availability and proximity, reducing order cycle time.
- Fewer oversells through Reservations and Out-of-Stock Thresholds, lowering cancellation rate.
- Flexible omnichannel options (for example, store pickup) by modeling stores as Sources.
- Example KPIs: in-stock rate, oversell incidents per 1,000 orders, average delivery time, shipping cost per order,
  order cancellation rate due to OOS.

## Common Challenges and Considerations

Be aware of frequent pitfalls and configure MSI to mitigate risk.

- Backorders: If enabled without clear expectations, you may oversell. Configure at Stores > Configuration > Catalog >
  Inventory > Product Stock Options > Backorders.
- Out-of-Stock Threshold: Set a safety buffer to account for shrinkage, damage, or returns.
- ERP/WMS Integration: Ensure external systems respect Reservations. If orders are imported after shipment, run
  consistency checks to avoid mismatches.
- Indexing and performance: For high volume, set inventory indexers to Update by Schedule to keep storefront responsive.
- Multi-website mapping: Assign the correct Stock to each website to avoid cross-website availability leakage.

## Use Cases and Scenarios

These common setups illustrate how to apply MSI to meet business goals.

- Single warehouse: One Source, one Stock mapped to one website — simplest setup for many SMBs. Benefit: straightforward
  operations with minimal configuration.
- Multi-warehouse: Regional Sources assigned to a single Stock; Distance Priority SSA reduces shipping time/cost.
  Benefit: lower shipping cost and faster delivery.
- Dropship vendor: Model each vendor as a Source; set low priority so in‑house inventory is used first. Benefit: expand
  catalog without carrying inventory; protect margin by preferring owned stock.
- Store pickup: Each store is a Source; enable In-Store Delivery to offer pickup. Benefit: increase store traffic and
  reduce last‑mile costs.
- Pre-sale with backorders: Enable backorders for select SKUs to capture demand before replenishment. Benefit: monetize
  demand early while communicating ship dates.

## Alternatives and Comparisons

Choose the right approach based on complexity, systems, and scale.

- MSI-only: Best for first-party owned inventory with simple to moderate complexity; minimal integration overhead.
- MSI + ERP/WMS: Use MSI to expose availability and allocate on the storefront; use ERP/WMS as the system of record for
  master inventory and fulfillment. Sync via REST APIs (/V1/inventory/source-items) or message bus.
- Legacy single-source (pre-2.3): Not recommended; MSI is standard since 2.3 and supports multi-location operations out
  of the box.

## Business Impact

Define a simple measurement plan to quantify results from MSI configuration and tuning.

- Establish baseline and target for: oversell incidents per 1,000 orders, average delivery time, shipping cost per
  order, in-stock rate, and order cancellation rate due to OOS. Compare pre/post changes to attribute improvements.

### For Merchants

- Reduce shipping cost per order by routing to the closest Source (typical 5–15% savings depending on geography and
  carrier mix).
- Increase in-stock rate and conversion by pooling inventory across Sources and websites where appropriate.
- Improve gross margin by prioritizing in-house inventory over dropship vendors when both are available.

### For Customers

- Faster delivery estimates based on nearby fulfillment.
- Fewer out-of-stock messages and order cancellations.
- Clear backorder messaging when items ship later, maintaining trust.

### For Operations

- Higher pick/pack efficiency by allocating orders to Sources with stock concentration.
- Reduced customer service workload via fewer oversells and cancellations.
- Easier inventory reconciliation with Reservations and built-in consistency checks.

## Common Misconceptions

Clarify frequently confused terms to avoid configuration mistakes.

- "Qty" vs "Salable Qty": Source Item Qty is physical per Source; Salable Quantity is aggregated and
  reservation-adjusted per website (Stock).
- "Stock" vs "Source": Stock is a logical grouping tied to websites; Source is a physical location where inventory is
  stored.
- "Backorders" do not create inventory; they allow selling into negative Salable Quantity and require clear messaging
  and lead times.

## Implementation Considerations

Use these steps and references to set up MSI correctly and keep data consistent.

- Create Sources: Admin > Stores > Inventory > Sources > Add New Source. Define contact info, address, and enable.
- Create Stocks: Admin > Stores > Inventory > Stocks > Add New Stock. Assign Sources and map to the target website(s).
- Assign products to Sources: Admin > Catalog > Products > Edit product > Sources section. Assign Sources and set
  quantities/status for each Source.
- Configure inventory options: Admin > Stores > Configuration > Catalog > Inventory. Set Backorders, Out-of-Stock
  Threshold, and Notify for Quantity Below.
- Choose SSA: Admin > Stores > Configuration > Catalog > Inventory > Source Selection Algorithm Options. Source Priority
  is default; Distance Priority requires accurate Source location data.
- APIs: Update source items via REST POST /V1/inventory/source-items; manage sources via /V1/inventory/sources.
- Indexers: For scale, set indexers to Update by Schedule (Admin > System > Index Management) to maintain storefront
  performance.
- Troubleshooting reservations: Use CLI to detect and fix mismatches:
    - bin/magento inventory:reservation:list-inconsistencies
    - bin/magento inventory:reservation:create-compensations
- Multi-website mapping: Ensure each website is assigned the correct Stock, and only the intended Sources are included.

### Quick Start

- Add your first Source (Stores > Inventory > Sources) and enable it with accurate address/location.
- Create a Stock (Stores > Inventory > Stocks) and assign your Source; map the Stock to your website.
- Edit top SKUs (Catalog > Products) to assign the Source and set quantities/status.
- Configure Backorders and thresholds (Stores > Configuration > Catalog > Inventory) per your policy.
- Choose SSA (Stores > Configuration > Catalog > Inventory > Source Selection Algorithm Options). Start with Source
  Priority; consider Distance Priority when you have multiple regions.
- Place a test order on the storefront and verify Salable Quantity changes and SSA selection at shipment.

## Related Concepts

- In-Store Pickup (In-Store Delivery)
- Product types and inventory (Configurable, Bundle, Grouped)
- Backorders and stock notifications
- Distance Priority SSA setup and source geocoding
