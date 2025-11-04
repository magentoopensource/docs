---
title: "How to Optimize Checkout for Higher Conversions"
description: "Reduce cart abandonment and improve checkout completion rates"
type: how-to
audience:
  - "Magento merchants"
difficulty: Advanced
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Payment provider account"
  - "SSL certificate"
tags:
  - how-to
  - checkout
  - high-impact
  - complex
last_updated: "2025-09-11T07:15:32.459Z"
---

# How to Optimize Checkout for Higher Conversions

## Overview

Reduce cart abandonment and increase revenue by streamlining Magento 2 checkout. This guide helps Magento merchants
enable guest checkout, reduce form fields, add express wallets (PayPal/Apple Pay/Google Pay), and enable Instant
Purchase and persistent cart—improving conversion with measurable results.

Business impact:

- Express wallets can reduce checkout time by 30–60% and lift mobile conversion by 5–15%.
- Guest checkout can increase completion 10–20% for first-time buyers.
- Fewer fields typically reduce drop-off on the shipping step and lower support contacts.

Scope and compatibility:

- Supported versions/editions: Magento Open Source and Adobe Commerce 2.4.x.
- Note: Braintree requires the official Braintree extension (Marketplace/Adobe) on 2.4.x.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Payment provider account
- SSL certificate
- PayPal and/or Braintree sandbox credentials ready for testing:
    - PayPal: API Username/Password/Signature (for core PayPal Express), or Client ID/Secret depending on integration
    - Braintree: Merchant ID, Public Key, Private Key
- Test cards and wallet test accounts (e.g., PayPal sandbox buyer; Apple Pay on a supported device; Google Pay test
  environment)
- If using Braintree wallets: Official Braintree Payments extension installed and enabled
- Optional (for measurement): GA4 implemented via Google Tag Manager or a GA4 extension
- Plan 30–60 minutes to complete, depending on wallet setup and verification steps

## What You'll Accomplish

By following this guide, you will:

- Configure checkout for higher conversions using proven settings
- Reduce checkout time and form friction for customers

## Step-by-Step Instructions

Note: After any configuration change, go to System > Cache Management and click Flush Magento Cache. If prompted,
refresh invalidated caches.

### Step 0: Enable guest checkout and streamline checkout options

Go to Stores > Configuration > Sales > Checkout > Checkout Options. Set:

- Allow Guest Checkout = Yes
- Display Billing Address On = Payment Page (keeps billing off the method tile)
- Enable Terms and Conditions = No (unless required by law)

If you do not need multi-address orders, go to Stores > Configuration > Sales > Multishipping Settings and set:

- Allow Shipping to Multiple Addresses = No

Save Config, then flush cache.

### Step 1: Reduce required form fields for faster entry

Go to Stores > Configuration > Customers > Customer Configuration > Name and Address Options. Set:

- Number of Lines in a Street Address = 1 or 2
- Show Prefix = No, Show Middle Name (initial) = No, Show Suffix = No
- Show Company = No, Show Fax = No
- Show Telephone = Optional (or Required if your fraud/risk model needs it)
- Show Tax/VAT Number = No (unless you sell B2B)

Save Config, then flush cache.

### Step 2: Set default country and minimize location friction

Go to Stores > Configuration > General > General > Country Options. Set:

- Default Country = your primary market
- Allow Countries = only markets you ship to
- Zip/Postal Code is Optional for Selected Countries = select countries where it isn’t required

Save Config, then flush cache.

### Step 3: Simplify shipping choices to one clear option

Go to Stores > Configuration > Sales > Delivery Methods. Enable one simple method customers understand:

- Free Shipping: Enabled = Yes and Minimum Order Amount per your threshold, or
- Flat Rate: Enabled = Yes with a simple Rate

Tips:

- Disable unused carriers to avoid choice overload.
- Set Sort Order = 0 (lowest number) on your preferred method so it’s preselected. Keep backup methods enabled with a
  higher Sort Order for edge cases.

Then go to Stores > Configuration > Sales > Shipping Settings > Origin and ensure your origin is correctly set.

Save Config, then flush cache.

### Step 4: Add an express wallet (choose one primary path)

Option A – Core PayPal Express Checkout (NVP)

1) Go to Stores > Configuration > Sales > Payment Methods > PayPal > PayPal Express Checkout.
2) Enable PayPal Express Checkout and enter API Username, API Password, and Signature from your PayPal account.
3) Surface buttons earlier:
    - Display on Shopping Cart = Yes
    - Display on Product Detail Page = Yes (if desired)
      Note: “Display on Mini Cart” is not a standard core PayPal setting. If your integration or theme adds mini cart
      buttons, follow that provider’s instructions.
4) Save Config, then flush cache.

Option B – Braintree (Marketplace extension) with PayPal, Apple Pay, Google Pay
Prerequisite: Install and enable the official Braintree Payments extension. Paths below appear after installation.

1) Go to Stores > Configuration > Sales > Payment Methods > Braintree.
2) Enable Braintree and enter Merchant ID, Public Key, and Private Key.
3) Enable Vault = Yes (required for saved cards/Instant Purchase).
4) (Optional) Enable PayPal via Braintree and configure button placement per the extension’s settings.
5) Apple Pay:
    - In the Braintree Control Panel, enable Apple Pay, upload the certificate, and complete domain verification for
      your site.
    - Ensure your site is served over HTTPS.
    - Test with Safari on a supported iOS/macOS device with a card in Apple Wallet.
6) Google Pay:
    - In the Braintree Control Panel, enable Google Pay, select allowed card networks, and set Environment = Sandbox for
      testing.
7) Save Config, then flush cache.

### Step 5: Enable Instant Purchase for returning customers

Go to Stores > Configuration > Sales > Sales. Set:

- Enable Instant Purchase = Yes

Prerequisites for the Instant Purchase button to appear:

- The customer is logged in and has a vaulted payment method supported by Instant Purchase (e.g., Braintree with Vault).
- The customer has default billing and shipping addresses saved in their account.

Save Config, then flush cache.

### Step 6: Turn on Persistent Shopping Cart

Go to Stores > Configuration > Customers > Persistent Shopping Cart. Set:

- Enable Persistence = Yes
- Persistence Lifetime (seconds) = your typical purchase window (e.g., 604800 for 7 days)
- Enable Remember Me = Yes (allow customers to opt in)
- (Optional) Clear Persistence on Sign Out = Yes

Align cookie lifetime: Go to Stores > Configuration > General > Web > Default Cookie Settings and ensure Cookie
Lifetime (seconds) is equal to or greater than your Persistence Lifetime.

Business note: Persistent carts support recovery of returning buyers and improve the effectiveness of cart reminder
emails.

Save Config, then flush cache.

### Step 7: Optimize cart behavior to keep shoppers in flow

Go to Stores > Configuration > Sales > Checkout > Shopping Cart. Set:

- After Adding a Product Redirect to Shopping Cart = No (so shoppers aren’t pulled away from browsing)
- Number of Cross-sell Items to Display = 0–2 (keep distractions minimal)

Go to Stores > Configuration > Sales > Checkout > Mini Cart. Set:

- Display Mini Cart = Yes
- Maximum Number of Items to Display = your preference (e.g., 3–5)

Shipping preselection reminder: Ensure your preferred delivery method has the lowest Sort Order under Stores >
Configuration > Sales > Delivery Methods (see Step 3).

Save Config, then flush cache.

### Step 8: Validate end-to-end with test orders

- Open a private browser window.
- As a guest, add a product and proceed to checkout. Confirm minimal fields and that express wallet buttons appear on
  cart (and PDP if enabled).
- Complete a test order using PayPal sandbox or a low-value product.
- Create a test customer account, save a payment method (via Braintree Vault), and place an order using Instant
  Purchase.
- For Apple Pay: test in Safari on a supported device with Apple Pay set up; confirm the Apple Pay button appears and
  completes authorization.
- For Google Pay (via Braintree): ensure Environment = Sandbox and complete a test authorization.
- Record time-to-complete and note any friction.

### Step 9: Measure conversion impact

If using GA4 via Google Tag Manager or a GA4 extension, verify events in Google Analytics > Admin > DebugView (or
Realtime) to confirm begin_checkout and purchase fire. If GA4 isn’t set up, implement GA4 first via GTM or an extension.

Key KPIs and targets:

- Checkout completion rate = purchase events ÷ begin_checkout events (target +3–10% improvement)
- Mobile conversion rate (target +5–15%)
- Average checkout duration (target -20–40%)

Compare a 7–14 day baseline before changes to the same duration after changes, keeping seasonality consistent. In
Magento, validate order uplift via Reports > Sales > Orders.

## Verification

Use this checklist to confirm everything is working correctly:

- Guest checkout is available on checkout
- Checkout shows only 1–2 street lines; Prefix/Middle/Suffix/Fax are hidden
- Cart does not redirect after add-to-cart; mini cart shows recent item(s)
- Only one shipping method visible (or your preferred method is preselected)
- PayPal/Braintree wallet buttons visible on Cart (and PDP if enabled)
- Instant Purchase button visible for a logged-in test customer with a vaulted payment method and default addresses
- Persistent cart retains items across browser sessions within your set lifetime

## Common Issues and Solutions

- Apple Pay button not showing
    - Test in Safari on a device with Apple Pay set up; ensure domain verification is completed in Braintree and your
      site is served over HTTPS.
- Google Pay not available (via Braintree)
    - Enable Google Pay in the Braintree Control Panel, ensure card networks are enabled, and set Environment = Sandbox
      for testing.
- PayPal buttons missing on cart
    - Confirm Display on Shopping Cart = Yes in PayPal settings; clear caches; check the browser console for theme/JS
      conflicts.
- Instant Purchase not visible
    - Ensure the customer is logged in, has a vaulted payment method, and has default billing/shipping addresses.
- Guest checkout disabled unexpectedly
    - Confirm Allow Guest Checkout = Yes and review any product-specific restrictions that require customer accounts.
- Multiple shipping options appearing
    - Disable unused carriers and verify your preferred carrier has the lowest Sort Order.
