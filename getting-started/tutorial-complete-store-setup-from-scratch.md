---
title: "Complete Store Setup from Scratch"
description: "Configure a new Magento 2.4.x store from first login to placing a successful test order, including taxes, shipping, payments, catalog, theme, and launch checks. This tutorial starts after Magento Open Source/Adobe Commerce 2.4.x is installed and you can sign in to the Admin."
type: tutorial
audience:
  - "Magento merchants"
  - "New Magento users"
difficulty: Beginner-Intermediate
estimated_time: "90–120 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Applies to Magento Open Source and Adobe Commerce 2.4.x"
tags:
  - tutorial
  - high-impact
  - complex
last_updated: "2025-09-11T19:18:34.022Z"
---

# Complete Store Setup from Scratch

## Why This Tutorial Matters

**Business Impact:**
- Launch a conversion-ready store faster with a proven setup sequence
- Reduce rework and avoid checkout failures that jeopardize early revenue
- Establish compliant tax, shipping, and payment configurations from day one

**What You'll Achieve:**
- A checkout-ready store with core settings, catalog, payments, and shipping configured
- A successfully placed test order with correct totals and emails
- A concise launch checklist to move from sandbox to production safely

## Learning Journey Overview

### Your Situation
You're signed into a fresh Magento 2.4.x Admin and need a clear, end-to-end path to configure settings, add a product, enable payments/shipping, and validate checkout before going live.

### What You'll Learn
By completing this tutorial, you will:

- Configure store info, locale, and currency
- Set up taxes, shipping methods, and payment methods
- Create categories and your first product
- Assign a theme and configure checkout and emails
- Place and verify a test order end-to-end

### Success Criteria
You'll know you've succeeded when:

1) A shopper can place a test order (guest or customer) with correct tax, shipping, and totals
2) Order-related emails (order, invoice, shipment) are received
3) The order appears in Sales > Orders with expected statuses and amounts

### Time Investment
- **Estimated time:** 90–120 minutes
- **Skill level after completion:** Confident beginner
- **Business value unlock:** Conversion-ready store with validated checkout

## Before We Start

### Who This Is For
This tutorial is designed for:

- Merchants and operators responsible for launching a new Magento store
- New Magento users who need a safe, repeatable setup checklist

### What You Need
Make sure you have:

- Magento Open Source 2.4.x Admin access
- A domain with an SSL certificate (or a secure staging URL)
- Payment provider sandbox credentials (for example, PayPal)
- Shipping carrier account details (optional for live rates)
- Basic tax requirements for your selling regions

### Preparation Checklist
Before starting, complete these preparation steps:

- Confirm Admin login and 2FA work for your user
- Ensure Base URLs point to your domain: Stores > Configuration > General > Web
- Verify SSL is installed and Use Secure URLs is enabled (Storefront and Admin)
- Confirm caches are enabled: System > Cache Management

## Step-by-Step Learning Path

### 1) Set Store Info, Locale, and Currency
- Go to: Stores > Settings > Configuration > General > General > Store Information and enter your business details
- Set locale: Stores > Configuration > General > Locale Options (Timezone, Locale, First Day of Week)
- Set currency: Stores > Configuration > General > Currency Setup (Base, Default, Display)
- If using multiple currencies: Stores > Currency > Currency Rates and update rates

Checkpoint: Storefront shows the correct language/locale and displays prices in your chosen currency.

### 2) Configure Websites, Stores, and Views
- Go to: Stores > Settings > All Stores
- Create additional Website/Store/Store Views if needed
- Set the Default Store View and confirm your store switcher reflects the intended structure

Checkpoint: Your store structure matches your business (e.g., single site, multiple views for languages).

### 3) Configure Taxes
- Add tax rates: Stores > Taxes > Tax Zones and Rates
- Create tax rules: Stores > Taxes > Tax Rules (map product and customer tax classes to rates)
- Set calculation/display: Stores > Configuration > Sales > Tax (Calculation Settings, Default Tax Destination, and Display Settings)

Checkpoint: Tax applies correctly in cart/checkout when you enter a taxable address.

### 4) Configure Shipping
- Set shipping origin: Stores > Configuration > Sales > Shipping Settings > Origin
- Enable methods: Stores > Configuration > Sales > Delivery Methods
  - Start with Flat Rate and/or Free Shipping (configure thresholds if needed)
  - Optionally configure carriers (UPS, USPS, FedEx, DHL) with your account credentials
- Ensure products intended for shipping have a weight set (non-virtual)

Checkpoint: At checkout, at least one shipping method appears for your test address.

### 5) Configure Payments
- Go to: Stores > Configuration > Sales > Payment Methods
- For testing, enable Check / Money Order (simple, no external credentials)
- Optionally configure PayPal using Sandbox credentials (set Environment = Sandbox)

Checkpoint: Your chosen payment method(s) appear during checkout.

### 6) Build Your Catalog: Categories and First Product
- Categories: Catalog > Categories
  - Use the existing Default Category or create a new root category and set it as default for the store
- Product: Catalog > Products > Add Product (Simple Product)
  - Required: Name, SKU, Price, Weight (if shippable), Quantity, Category assignment, Visibility = Catalog, Search, Enable Product = Yes

Checkpoint: Product is visible on the storefront category page and product detail page.

### 7) Verify Inventory (MSI)
- Go to: Stores > Inventory > Sources and Stocks
  - Confirm Default Source and Default Stock exist
  - Ensure your product is assigned and shows Salable Quantity > 0

Checkpoint: Product shows In Stock on the storefront and can be added to cart.

### 8) Assign a Theme
- Go to: Content > Design > Configuration
- Edit the Global (Default) scope or specific store view
- Select your theme and Save Configuration
- If prompted, clear caches: System > Cache Management > Flush Magento Cache

Checkpoint: Storefront renders with your selected theme.

### 9) Configure Checkout and Emails
- Checkout: Stores > Configuration > Sales > Checkout
  - Enable Guest Checkout if desired and review checkout options
- Email Identities: Stores > Configuration > General > Store Email Addresses (set From names and addresses)
- Transactional Emails: Stores > Configuration > Sales > Sales Emails (enable and confirm sender identities)

Checkpoint: You are ready to validate order and invoice emails by placing a test order.

### 10) Place a Test Order and Verify
- On the storefront, add your product to cart and proceed to checkout
- Enter a taxable address (if applicable), select a shipping method, and choose Check / Money Order
- Place the order and confirm the order success page
- In Admin: Sales > Orders
  - Verify order totals, create an Invoice, then create a Shipment (practice complete order flow)
- Confirm order/invoice/shipment emails are received

Checkpoint: End-to-end order flow completes and emails send as expected.

### 11) Launch Readiness Checklist
- Security and URLs: Stores > Configuration > General > Web
  - Use Secure URLs (Storefront and Admin) = Yes
  - Search Engine Optimization > Use Web Server Rewrites = Yes
- Robots: Content > Design > Configuration > Edit scope > Search Engine Robots = INDEX, FOLLOW (production only)
- Caching and Indexes:
  - System > Cache Management: Enable all caches and Flush Magento Cache
  - System > Tools > Index Management: Reindex any out-of-date indexes
- Base URLs: Stores > Configuration > General > Web: Confirm Base URLs (secure and unsecure) match your production domain

Checkpoint: Storefront loads over HTTPS, friendly URLs work, and search engines can index production.

## Practice and Reinforcement

Now that you've learned the core process, let's reinforce your skills:

- Create a configurable product with Size and Color attributes and place a test order
- Enable Free Shipping for orders over a threshold and verify the method appears only when eligible
- Change the Default Display Currency and update Stores > Currency > Currency Rates; confirm price display on storefront

## What You've Accomplished

🎉 **Congratulations!** You have successfully:

- Configured general settings (store info, locale, currency)
- Set up taxes, shipping, and payment methods
- Built a basic catalog and verified inventory
- Assigned a theme and configured checkout and emails
- Placed and processed a complete test order (order, invoice, shipment)

### Business Impact
- Reduced time-to-first-sale with a validated checkout
- Lower launch risk via a structured checklist and test workflow

### Skills Gained
You now have the ability to:

- Navigate core Admin areas confidently
- Diagnose and resolve common visibility, tax, shipping, and payment issues
- Prepare a store for production with essential security and SEO settings

## Next Steps in Your Journey

Now that you've mastered this process, here's how to build on your success:

### Immediate Actions
- Switch payment methods from sandbox/test to live only after all QA passes
- Create a simple go-live checklist (payments, shipping, taxes, emails, SSL, robots, cache)
- Set up basic monitoring (order failure alerts, email deliverability checks)

### Level Up Your Skills
- Build out category structure and product attributes for scalable catalog management
- Create custom transactional email templates and brand your communications
- Generate and submit an XML sitemap (Marketing > SEO & Search > Site Map) to search engines

### Advanced Applications
- Enable persistent cart to improve conversion (Stores > Configuration > Customers > Persistent Shopping Cart)
- Set related/upsell/cross-sell products to increase AOV
  - Configure related products per item
- Implement basic SEO: metadata on homepage/categories, friendly URLs, and robots best practices

## When Things Don't Go as Expected

Even experienced merchants encounter challenges. Here's how to handle common situations:

### Payments
- Symptom: Payment method not visible at checkout
  - Likely cause: Method disabled, scope mismatch, or unsupported currency/country
  - Fix: Stores > Configuration > Sales > Payment Methods; enable method at the correct scope and ensure store currency and Allowed Countries align with the provider

### Shipping
- Symptom: No shipping methods available
  - Likely cause: Shipping Origin missing, product has no weight, or method restrictions
  - Fix: Set Origin (Stores > Configuration > Sales > Shipping Settings); ensure product Weight > 0 and In Stock; review method restrictions in Sales > Delivery Methods

### Taxes
- Symptom: Taxes not applied or incorrect
  - Likely cause: Missing tax rates/rules or default destination mismatch
  - Fix: Stores > Taxes > Tax Zones and Rates; Stores > Taxes > Tax Rules; verify Default Tax Destination Calculation (Stores > Configuration > Sales > Tax) and customer address

### Catalog & Inventory
- Symptom: Product not visible on storefront
  - Likely cause: Disabled product, wrong visibility, not assigned to website/category, or no salable qty
  - Fix: Catalog > Products: Enable Product, set Visibility = Catalog, Search; assign to website and category; ensure In Stock and quantity > 0; if needed, System > Tools > Index Management: Reindex

### Caching & Indexing
- Symptom: Changes not appearing
  - Likely cause: Cache not flushed or indexes outdated
  - Fix: System > Cache Management: Flush Magento Cache; System > Tools > Index Management: Reindex updated indexes

### Access & Security (2FA)
- Symptom: Locked out by 2FA
  - Fix: Ask an Admin with higher privileges to reset your 2FA or use backup codes per your configured provider

## Continue Learning

### Related Tutorials
- Configure Taxes by Region
- Shipping Methods Deep Dive
- Payment Methods and PayPal Sandbox
- Theme Customization Basics
- Inventory (MSI) for Multi-Location

### How-To Guides
- Create Configurable Products
- Set Up Email Templates
- Manage URL Rewrites and Sitemaps

### Reference Materials
- Magento 2.4 User Guide: Configuration, Catalog, Sales, Marketing

## Summary

A structured, end-to-end setup reduces launch risk and speeds time-to-first-sale. With core configuration, catalog, payments, shipping, checkout, and emails in place—and a successful test order—you are ready to finalize production settings and go live.

### Key Takeaways
- Follow a proven sequence to avoid rework and missed settings
- Validate end-to-end with a real test order and emails
- Lock in production essentials: HTTPS, rewrites, robots, caches, and indexes

### Remember
- Switch to live payments only after passing your QA checklist
- Set Search Engine Robots to INDEX, FOLLOW only on production
- Revisit taxes, shipping, and payment configurations whenever your business rules change