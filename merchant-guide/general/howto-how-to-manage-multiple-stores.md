---
title: "How to Manage Multiple Stores"
description: "Efficiently operate multiple stores, websites, and brands from one installation"
type: how-to
audience:
  - "Magento merchants"
  - "New Magento users"
difficulty: Intermediate
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
tags:
  - how-to
last_updated: "2025-09-12T11:31:14.828Z"
---

# How to Manage Multiple Stores

## Overview

Operate multiple brands, regions, or catalogs from one Magento 2 installation to reduce operating cost, improve customer experience per market, and scale faster with consistent governance.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel

## What You'll Accomplish

By following this guide, you will:

- Successfully how to manage multiple stores

## Step-by-Step Instructions

### Step 0: Plan your multi-store structure

Decide what you need: Websites for separate domains, pricing, checkout; Stores for different catalogs under the same checkout; Store Views for language/currency. Example: Create two Websites (US, UK), each with one Store and one Store View. Note decisions for price scope, account sharing, and inventory strategy. [Estimated time: 10 min]




### Step 1: Create the Website container

Go to Stores > All Stores > Create Website. Enter a Name (e.g., UK Storefront), a unique Code (e.g., uk), and Sort Order. Save. Repeat for any additional websites.




### Step 2: Create a Root Category for the new store

Go to Catalog > Categories. Click Add Root Category. Name it (e.g., UK Catalog Root), set Enable Category &#x3D; Yes and Include in Menu &#x3D; Yes. Save. Add child categories as needed.




### Step 3: Create the Store and assign the Root Category

Go to Stores > All Stores > Create Store. Choose the Website you created, enter a Name (e.g., UK Store), and set Root Category to the one you just created. Save.




### Step 4: Create Store Views for language/currency

Go to Stores > All Stores > Create Store View. Select the Store, enter Name (e.g., English UK), Code (e.g., en_uk), Status &#x3D; Enabled. Save. Create additional views for other languages if needed.




### Step 5: Configure Base URLs for each Website

Go to Stores > Configuration. In the scope switcher (top left), select the new Website. Navigate to General > Web. Expand Base URLs and Base URLs (Secure). Enter the full domain URL (e.g., https://uk.brand.co.uk/). Set Use Secure URLs in Storefront/Admin &#x3D; Yes. Save. Coordinate DNS/SSL and web server vhost if not already done.




### Step 6: Set locale, time zone, and currency per Store View/Website

For language and formatting, select the Store View in the scope switcher and go to General > Locale Options. Set Locale (e.g., English (United Kingdom)) and Timezone. For currency, switch to Website scope: General > Currency Setup. Set Base Currency, Default Display Currency, and Allowed Currencies. Save.




### Step 7: Configure tax, payment, and shipping per Website

Switch to the target Website scope. Go to Sales > Tax to set tax classes/rules per region. Then go to Sales > Payment Methods to enable region-appropriate methods (e.g., PayPal for UK) and Sales > Shipping Methods for carriers/zones. Save after each section.




### Step 8: Assign inventory sources and stock to the Website (MSI)

Go to Stores > Inventory > Stocks. Edit or create a Stock that includes the Sources serving this region. Assign the Website to this Stock. Ensure Sources have quantity for relevant products (Stores > Inventory > Sources or per product). Save.




### Step 9: Assign products to the Website and categories

Open a product (Catalog > Products). In Product in Websites, select the new Website. Assign the product to categories under the Store’s Root Category. If Price Scope &#x3D; Website, enter website-specific prices. Save. Repeat for key products (or use bulk actions/import).




### Step 10: Set default store view and optional URL code behavior

Go to Stores > All Stores. Edit the Website to confirm the Default Store and Default Store View. If using separate domains, go to Stores > Configuration > General > Web > URL Options and set Add Store Code to URLs &#x3D; No. If sharing a domain with paths, set it to Yes and configure rewrites.




### Step 11: Clear cache and let indexes update

Go to System > Cache Management. Click Flush Magento Cache. Ensure cron is running so indexers update automatically. If necessary, schedule a developer to run CLI reindex. Refresh storefront pages.




### Step 12: Verify storefronts end-to-end

Visit each domain. Confirm theme/logo, language, currency, categories, and product pricing. Place a test order with website-specific payment and shipping. In Admin, go to Sales > Orders and verify the Website column and totals. Check Stores > Currency > Currency Rates reflect expected conversions.





## Verification

To confirm everything is working correctly:

1. Navigate to the admin panel and verify changes are visible
1. Check that all configurations are saved correctly
1. Test the functionality from a customer perspective

## Common Issues and Solutions


## Related Resources

- [Magento User Guide](https://docs.magento.com/user-guide/)

---

*Need help? [Contact support]() or check our [troubleshooting guide]().*