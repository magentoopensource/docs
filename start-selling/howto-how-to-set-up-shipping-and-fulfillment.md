---
title: "How to Set Up Shipping and Fulfillment"
description: "Configure shipping methods, rates, and automated fulfillment processes"
type: how-to
audience:
  - "Magento merchants"
difficulty: Advanced
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Product catalog setup"
tags:
  - how-to
  - shipping
  - high-impact
last_updated: "2025-09-11T11:07:31.298Z"
---

# How to Set Up Shipping and Fulfillment

## Overview

Set up shipping and fulfillment to reduce cart abandonment, speed up delivery, and unlock revenue. This guide shows new
store owners and growing merchants how to configure shipping methods, rates, and basic fulfillment automation in Magento
2 so customers see clear options at checkout and you can ship orders efficiently.

Note: This guide focuses on core Magento capabilities (shipping methods, rules, labels, and notifications). For advanced
automation (e.g., rate shopping, dimensional packing, multi-warehouse, or 3PL/ship station integrations), consider
extensions or platforms like ShipperHQ, ShipStation, or a 3PL integration to reduce handling time and errors.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Product catalog setup
- Ensure shippable products have Weight > 0 (Catalog > Products > Edit > Weight)
- In Stores > Configuration, use the Scope selector (top-left) to switch to the target Website when configuring Shipping
  Methods and Table Rates
- Confirm Stores > Configuration > General > Country Options: set Allow Countries and Default Country for your service
  area

## What You'll Accomplish

By following this guide, you will:

- Successfully set up shipping and fulfillment
- Improve your store's performance and customer experience

## Step-by-Step Instructions

### Step 0: Set your shipping origin and store address

In Admin, go to Stores > Configuration. Under Sales > Shipping Settings, open the Origin section. Enter country,
region/state, city, postcode, and street address. Save. Ensure a valid phone number is set in Stores > Configuration >
General > Store Information. Complete this step first to ensure accurate quotes and labels.

### Step 1: Decide your initial shipping strategy

Choose one path to launch:

- Flat Rate: Simple; best for light, uniform products.
- Free Shipping (threshold): Boosts conversion above a spend level.
- Table Rates: Tier by weight/price/destination.
- Live Carrier Rates: Accurate quotes and labels (UPS/FedEx/USPS/DHL).
  You can combine methods (e.g., Free over $50; else Flat $5).

Business guidance: Set a free-shipping threshold slightly above current AOV (about 10–20%) to lift order value while
protecting margin. Aim to keep average shipping cost under ~10% of order value.

### Step 2: Enable Flat Rate shipping (optional)

Go to Stores > Configuration > Sales > Shipping Methods > Flat Rate. Set Enabled = Yes. Method Name = Flat Rate. Type =
Per Order (or Per Item). Price = e.g., 5.00. Calculate Handling Fee = Fixed or Percent. Handling Fee = e.g., 0.50 (
fixed) or 2 (percent). Do not include the % symbol. Set Ship to Applicable Countries and (if Specific) Ship to Specific
Countries. Save. After saving, go to System > Cache Management > Flush Magento Cache.

### Step 3: Configure Free Shipping with a threshold (optional)

Use one of these approaches:

Option A (Simple, global):

- Enable method: Stores > Configuration > Sales > Shipping Methods > Free Shipping
- Enabled = Yes
- Minimum Order Amount = e.g., 50
- Include Tax to Amount = Yes/No per your policy
- Configure countries and Save
- System > Cache Management > Flush Magento Cache

Option B (Flexible with targeting):

- In Free Shipping method, set Minimum Order Amount = 0 (keep the method available)
- Create rule: Marketing > Cart Price Rules > Add New Rule
    - Name: Free shipping over $50
    - Active = Yes
    - Websites and Customer Groups = all relevant
    - Conditions: Subtotal (excluding tax) equals or greater than 50 (or your value)
    - Actions: Free Shipping = For entire order; Stop Further Rules Processing = Yes if you don’t want other discounts
      to stack
    - Set From/To Dates if needed, then Save
- System > Cache Management > Flush Magento Cache

### Step 4: Set up Table Rates (optional, for tiered pricing)

Go to Stores > Configuration > Sales > Shipping Methods > Table Rates. In the Scope selector (top-left), switch to the
target Website. Set Enabled = Yes. Condition = choose Price vs. Destination, Weight vs. Destination, or # of Items vs.
Destination. Save.

Click Export CSV to download the template at this Website scope. The CSV includes columns: Country, Region/State,
Zip/Postal Code, Condition Value, Price. Use '*' as a wildcard for ZIPs (e.g., 90*). Condition Value corresponds to your
selected Condition (price/weight/items). Only one row is matched—the most specific wins.

Fill the CSV with country, region, ZIP patterns, and thresholds with corresponding shipping prices. Return to Table
Rates and Import the CSV at the same Website scope. Save and then go to System > Cache Management > Flush Magento Cache.
Test with various addresses and order sizes.

### Step 5: Configure a carrier for live rates and labels (optional)

Example UPS: Stores > Configuration > Sales > Shipping Methods > UPS. Enabled = Yes. UPS Type = United Parcel Service
XML. Enter Access License Number, User ID, Password, and Shipper Number. Packaging = Package. Allow methods (e.g.,
Ground, 2nd Day). Handling Fee (optional). Free Method (optional) = None (use cart rule instead). Negotiated Rates = Yes
if available. Debug = Yes during testing.

Use production (non-sandbox) credentials for launch. For carriers that offer Sandbox Mode (e.g., FedEx), set Sandbox
Mode = No when going live. Configure shipping labels within each carrier’s settings: enable label printing, complete
required credentials, and set packaging defaults and dimensions there. Standardize package sizes where possible and
enable negotiated rates to reduce costs 10–30%.

### Step 6: Control method visibility and ordering at checkout

For each enabled method, set:

- Ship to Applicable Countries = All Allowed Countries or Specific Countries. If Specific, set Ship to Specific
  Countries accordingly.
- Show Method if Not Applicable = No
- Sort Order for display priority (lower shows first)
  Save after each change and go to System > Cache Management > Flush Magento Cache if prompted.

### Step 7: Enable fulfillment basics: labels, packing slips, notifications

Go to Stores > Configuration > Sales > Sales Emails and enable Shipment and Shipment Comments emails (set Sender and
templates). Ensure Store Email Addresses are set.

Configure shipping labels within each carrier: Stores > Configuration > Sales > Shipping Methods > [Carrier] (e.g.,
UPS/FedEx/USPS/DHL). Enable label printing and complete required credentials and packaging defaults there.

Print packing slips from:

- Sales > Shipments > open a shipment > Print Packing Slip
- Or bulk: Sales > Shipments, select shipments > Actions > Print Packing Slips

### Step 8: Place a domestic test order to validate rates

On the storefront, add a shippable product to cart. Use Estimate Shipping or proceed to checkout. Enter a domestic
address matching your service area. Verify Flat/Free/Table/Carrier methods appear with correct pricing. Complete the
order using a test payment method or a low-value live method.

If using carriers, set Debug = Yes in the carrier config while testing. Review var/log/shipping.log and
var/log/system.log for any errors or denials.

### Step 9: Create a shipment, label, and tracking for the test order

Admin > Sales > Orders > open the test order. Click Ship. If using a carrier with labels, choose Create Shipping Label,
define package weight/dimensions if required, and click Submit Shipment. Print the label. Ensure a tracking number is
generated. Send shipment email if not automatic.

### Step 10: Test threshold and regional edge cases

Adjust cart value below and above your Free Shipping threshold to ensure correct method toggling. Test a restricted
country or state to confirm methods are hidden. If using Table Rates, try ZIP patterns from the CSV to confirm matching
behavior.

Troubleshooting during testing:

- If you see "Sorry, no quotes are available for this order at this time", verify product weights (> 0), origin address,
  allowed countries, and carrier credentials. Check units (lbs/kg) and required dimensions.
- Check var/log/shipping.log and var/log/system.log for detailed error messages.

### Step 11: Publish your shipping policy and monitor KPIs

Add a clear Shipping & Delivery page with delivery windows, thresholds, and cutoffs. Link it from the header/footer and
reference on PDP/cart.

- Create a CMS page: Content > Pages > Add New Page (Title: Shipping & Delivery). Publish and link it in Content >
  Blocks or your theme’s header/footer.
- Monitor KPIs in Reports > Sales > Orders and related views. Suggested targets: cart abandonment < 70%, shipping cost <
  10% of order value, on-time ship rate > 95%. Review weekly and adjust handling fees, thresholds, and methods as
  needed.

## Verification

To confirm everything is working correctly:

1. In checkout, verify only intended methods show for a supported address (e.g., US) and that prices match your
   configuration
1. Test a destination outside your service area to confirm methods are hidden
1. Place two orders around the free shipping threshold (e.g., $49 and $51) to validate rule logic
1. Create a shipment and confirm the shipment email is received and the tracking link works
1. In Admin, confirm all configurations are saved at the correct Website scope and caches are clean (System > Cache
   Management > Flush Magento Cache)

## Common Issues and Solutions

- Issue: "Sorry, no quotes are available for this order at this time"
    - Fix: Add product weights (> 0); confirm origin address; verify allowed countries; enable the shipping method;
      check carrier credentials and service availability for the destination; review var/log/shipping.log
- Issue: Free Shipping not appearing at threshold
    - Fix: Ensure Free Shipping method is Enabled; for rules, use condition Subtotal (Excl. Tax); clear cache; check
      rule priority and set Stop Further Rules Processing = Yes if needed
- Issue: Table Rates not applied
    - Fix: Switch to the correct Website scope before export/import; verify CSV columns and ZIP wildcards; re-save
      config and Flush Magento Cache
- Issue: Label creation fails
    - Fix: Provide package weight/dimensions; confirm production credentials; disable Sandbox Mode for go-live (where
      applicable); check var/log/shipping.log for API errors
