---
title: "How to Optimize Products for Search"
description: "Improve product visibility with SEO-friendly URLs, meta data, and content"
type: how-to
audience:
  - "Magento merchants"
difficulty: Intermediate
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Product catalog setup"
tags:
  - how-to
  - product-management
  - content-management
  - seo
last_updated: "2025-09-15T10:59:05.323Z"
---

# How to Optimize Products for Search

## Overview

Optimize product pages so shoppers and search engines can find them faster. This how-to guides Magento merchants to create SEO-friendly URLs, compelling meta data, and high-quality content that improves organic visibility and on-site search findability—leading to more qualified traffic and higher conversion.

Business impact and KPIs to monitor:
- Higher organic CTR and sessions to product pages
- Increased on-site search conversion rate; reduced zero-result searches and search exits
- Improved average position for priority product URLs in search results
- Suggested 30–60 day targets to benchmark impact:
  - +10–20% organic sessions to product pages
  - +1–2 percentage points SERP CTR for key product URLs
  - -30% zero-result searches; -10–20% search exit rate

How to measure and prioritize:
- Use Google Search Console (Search Results report) to monitor impressions, average position, and CTR for product URLs.
- Use GA4 or Adobe Analytics to measure site search behavior and conversion. In GA4, track view_search_results and purchase events to derive search conversion and exit rates.
- Prioritize optimization for top revenue SKUs and product URLs with high impressions but low CTR to maximize impact.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Product catalog setup
- Enabled cron on the server for scheduled sitemap generation and indexers
- Administrator role with permissions for Catalog, Marketing, and Stores configuration
- Web server rewrite module enabled (Apache mod_rewrite or equivalent on Nginx)

Note: Screens and features apply to Magento Open Source and Adobe Commerce 2.4.x. Adobe Live Search is an Adobe Commerce add-on that changes native search behavior and overrides some native Catalog Search settings.

If you use Adobe Live Search:
- Manage synonyms, search suggestions/recommendations, and merchandising in Live Search.
- Relevance tuning and attribute boosting are configured in Live Search, not via native Catalog Search settings like Search Weight.
- Native Catalog Search UI options for suggestions/recommendations do not apply when Live Search is enabled.

## What You'll Accomplish

By following this guide, you will:

- Configure global SEO settings for clean URLs and canonical tags
- Optimize product URL keys with 301 redirects
- Write high-quality meta titles/descriptions and on-page content
- Tune attributes and synonyms to improve on-site search relevance
- Generate and submit an XML sitemap and verify indexing

## Step-by-Step Instructions

### Step 1: Set global SEO URL and canonical settings

Group and apply the following settings by Admin path:

- Go to: Stores > Configuration > General > Web > Search Engine Optimization
  - Set: Use Web Server Rewrites = Yes

- Go to: Stores > Configuration > General > Web > Url Options
  - Set: Add Store Code to URLs = No
  - Set: Auto-redirect to Base URL = Yes (301 Moved Permanently)

- Go to: Stores > Configuration > Catalog > Catalog > Search Engine Optimization
  - Set: Use Categories Path for Product URLs = No
  - Set: Use Canonical Link Meta Tag for Products = Yes; Categories = Yes
  - Set: Product URL Suffix = .html; Category URL Suffix = .html (include the leading dot)
  - Set: Create Permanent Redirect for URLs if URL Key Changed = Yes
  - Note: Avoid changing URL suffixes on a live site without testing and 301 redirects—doing so can cause widespread 404s and SEO loss.
  - Warning: Changing "Use Categories Path for Product URLs" on a live site can rewrite many product URLs. Test in a staging environment first, export current URLs, prepare bulk 301 redirects, and update XML sitemaps before deploying.
  - Bulk redirects (for large changes): Implement 301s at your web server/CDN for scale. Examples:
    - Apache: `Redirect 301 /old-url.html /new-url.html`
    - Nginx: `rewrite ^/old-url.html$ /new-url.html permanent;`
    - For small batches: Marketing > SEO & Search > URL Rewrites > Add URL Rewrite

- Click: Save Config

- Go to: Stores > Configuration > Catalog > Catalog > Catalog Search
  - Review and set: Minimum Query Length, Maximum Query Length
  - Optional: Enable Search Suggestions = Yes; Enable Search Recommendations = Yes (tune to your catalog and customers)
  - If Adobe Live Search is enabled, native Catalog Search settings (suggestions/recommendations) are not used; manage these behaviors in the Live Search admin.

Why this matters: Canonicals and clean URLs prevent duplicate content, consolidate ranking signals, and improve crawl efficiency; tuned search settings improve on-site findability and conversion.

### Step 2: Ensure product is visible in search

- Go to: Catalog > Products > [Product]
  - Ensure: Enable Product = Yes
  - Set: Visibility = Catalog, Search
  - Product in Websites: select the target Website(s)
  - Categories: assign at least one relevant, enabled category (recommended for discovery/navigation)
  - Ensure availability:
    - Quantity > 0 or Backorders allowed
    - If using MSI: in the product’s Sources section, assign the correct sources and quantities. Ensure the product is assigned to the target Website(s) in Product in Websites, and that the Website is linked to the correct Stock (Stores > Inventory > Stocks).
  - Optional: If you want out-of-stock items to appear in search: Stores > Configuration > Catalog > Inventory > Stock Options > Display Out-of-Stock Products = Yes
- Click: Save

Why this matters: If a product isn’t enabled, assigned to a website, salable, or set to Catalog, Search it won’t show in on-site search results.

### Step 3: Optimize product URL key with 301 redirect

- Go to: Catalog > Products > [Product] > Search Engine Optimization
  - Set: URL Key = lowercase, hyphen-separated; no special characters; keep under 60 characters (e.g., "mens-running-shoes-nike-pegasus-40"). Avoid stop words (the, and, for) and redundant category words if not needed.
  - Set: Create Permanent Redirect for old URL = Yes
- For multi-language sites: use the Store View selector (top-left) to set localized URL Keys per store view.
- Click: Save

Bulk redirects tip: When changing many URL keys or suffixes, export old/new URLs to create a mapping and implement redirects at the web server or CDN layer for performance and scale. For smaller sets, use Marketing > SEO & Search > URL Rewrites.

Why this matters: A clean URL improves relevance and click-through; 301 redirects preserve link equity and prevent traffic loss from broken links.

### Step 4: Write meta title and meta description

- Go to: Catalog > Products > [Product] > Search Engine Optimization
  - Set: Meta Title = 50–60 characters, include primary keyword and brand/model if applicable
  - Set: Meta Description = 150–160 characters emphasizing benefits and a clear CTA
- For multi-language sites: switch Store View and provide localized Meta Title and Meta Description.
- Click: Save

Scale it with templates (recommended for large catalogs):
- Go to: Stores > Configuration > Catalog > Catalog > Product Fields Auto-Generation
  - Use attribute placeholders to auto-generate defaults, for example:
    - Meta Title: {{name}} | {{brand}}
    - Meta Description: Shop {{name}} by {{brand}}. {{short_description}}
  - These defaults apply when product-level fields are empty; you can override per product as needed.

Why this matters: Strong meta tags improve SERP relevance and CTR, driving more qualified organic traffic. Templates save time and ensure consistency across large catalogs.

### Step 5: Improve product descriptions and image alt text

- Go to: Catalog > Products > [Product]
  - Refine: Short Description and Description with shopper-focused copy, features/benefits, and naturally integrated keywords
  - Differentiate: Avoid copy-pasting manufacturer descriptions. Add unique value propositions (USPs), sizing/fit guidance, care instructions, returns/warranty info, and a brief FAQ to address objections.
- Go to: Images and Videos
  - Click each image and set: Alt Text = descriptive text (e.g., "Men’s blue mesh running shoe, side view")
- Click: Save

Advanced (multi-language/international): Magento does not add hreflang tags out of the box. Work with your developer or an extension to implement hreflang linking equivalent product URLs across store views/locales to improve geotargeting and avoid duplicate-content issues.

Why this matters: Rich, relevant content aids ranking and conversions; alt text improves accessibility and image SEO.

### Step 6: Configure key attributes for search relevance

- Go to: Stores > Attributes > Product > [Attribute] > Storefront Properties
  - Set: Use in Search = Yes
  - Set: Search Weight = higher (e.g., 5–10) for critical attributes (Brand, Material, Size)
  - Set: Use in Layered Navigation = Filterable (with results)
  - Set: Use in Search Results Layered Navigation = Yes (to show the filter on search results pages)
  - Ensure: The attribute is assigned to the product’s attribute set
- Note: Layered Navigation is only available for attributes with Catalog Input Type = Dropdown, Multiple Select, or Price. The "Use in Search Results Layered Navigation" option appears only after setting "Use in Layered Navigation" to Filterable and for supported attribute types.
- Note: After changing Use in Search or Search Weight, reindex the Catalog Search index (System > Tools > Index Management) or via CLI (see Step 11).
- If you use Adobe Live Search: configure attribute boosting and relevance in Live Search instead of relying on native Search Weight.
- Click: Save Attribute

Why this matters: Proper attribute configuration boosts search relevance and enables filters shoppers use to convert.

### Step 7: Add shopper-language search synonyms

- Go to: Marketing > SEO & Search > Search Synonyms
  - Click: New Synonym Group
  - Set: Scope = Website
  - Enter terms (comma-separated): sneakers, trainers, running shoes
  - Click: Save
- How to find high-impact synonyms:
  - Review Admin > Marketing > SEO & Search > Search Terms for frequent queries and misspellings
  - In Google Search Console, check Search Results > Queries for how users describe your products
  - Talk to Customer Service and Sales for regional terms and common alternatives
  - Prioritize synonyms that drive zero-result searches to reduce abandonment
- Note: If you use Adobe Live Search, manage synonyms in the Live Search admin instead of native synonyms.

Why this matters: Synonyms capture how customers actually search, reducing zero-results and improving conversions.

### Step 8: Create and submit XML sitemap

- Go to: Stores > Configuration > Catalog > XML Sitemap
  - Under Generation Settings: Enabled = Yes; configure Frequency and Start Time (cron must be enabled)
  - Under Products Options: Add Images into Sitemap = All
  - Under Sitemap File Limits: set Maximum No. of URLs per File (up to 50,000) and Maximum File Size (up to 10,485,760 bytes). Magento splits files automatically when limits are reached.
  - Click: Save Config
- Go to: Marketing > SEO & Search > Site Map
  - Click: Add Sitemap
  - Set: Filename = sitemap.xml; Path = /
  - Set: Store View = [Your Store View]
  - Click: Save, then click: Generate
- Note: On Adobe Commerce on cloud infrastructure, the web root may be read-only. Set Path = /media/sitemap/ (ensure the directory exists and is writable) and submit the full URL (e.g., https://yourdomain.com/media/sitemap/sitemap.xml) to Google Search Console.
- Confirm: The sitemap URL you configured (e.g., https://yourdomain.com/sitemap.xml or https://yourdomain.com/media/sitemap/sitemap.xml) is accessible
- For multi-storeview sites: create a separate sitemap per Store View and submit each one. Use a separate Google Search Console URL-prefix property that matches each store view’s exact base URL, and submit the corresponding sitemap URL in that property.
- Submit: In Google Search Console, open your property > Indexing > Sitemaps > enter the sitemap URL (e.g., https://yourdomain.com/sitemap.xml) > Submit.
- Add to robots.txt for discovery (recommended):
  - Go to: Content > Design > Configuration > [Store View] > Search Engine Robots
  - In "Edit Custom instruction of robots.txt File", add: `Sitemap: https://yourdomain.com/sitemap.xml` (use the exact URL per store view)

Why this matters: Sitemaps help search engines discover and index product URLs and images faster.

### Step 9: Set robots to allow indexing

- Go to: Content > Design > Configuration
  - Click: Edit on your target Store View
  - Open: Search Engine Robots
  - Set: Default Robots = Index, Follow
  - Click: Save Configuration

Why this matters: Ensures your products are crawlable and indexable in production.

### Step 10: Enable breadcrumbs on product pages

- Go to: Stores > Configuration > Catalog > Catalog
  - Set: Breadcrumbs on Product Pages = Yes
  - Click: Save Config

Why this matters: Breadcrumbs strengthen internal linking and can enhance rich snippets, aiding SEO and navigation.

### Step 11: Reindex and flush cache

- Go to: System > Tools > Index Management
  - Select relevant indexes (e.g., Product EAV, Category Products, Catalog Search) and choose Actions: Reindex Data
- Go to: System > Cache Management
  - If any cache types are Invalidated, select them and click Refresh. Only use Flush Magento Cache if changes do not appear after a refresh.

CLI (recommended for production):
- SSH to your server and run:
```
bin/magento indexer:reindex
bin/magento cache:clean
bin/magento cache:flush full_page
```
- Verify indexer modes:
```
bin/magento indexer:show-mode
```
  - Use Update by Schedule in production for heavy catalogs:
```
bin/magento indexer:set-mode schedule
```

Why this matters: Reindexing and proper cache refresh ensure your changes are visible on the storefront and in search without unnecessary performance impact.

### Step 12: Validate results

- On the storefront: search for the product name and a synonym you added
- Open the product page: confirm the URL format, meta tags, and visible breadcrumbs
- Test an old product URL: verify it 301-redirects to the new URL
- Over the next weeks: monitor Google Search Console and Admin > Marketing > SEO & Search > Search Terms for trends

For a full checklist, see the Verification section below for technical checks, robots/sitemaps, and structured data tests.

Why this matters: Validation confirms correct implementation and surfaces issues before they impact traffic.

## Verification

To confirm everything is working correctly:

- View page source on a product page and confirm: a single "link rel=\"canonical\"" points to the product URL without category path
- Verify meta tags: the title tag and meta description tag match your entries
- Check image alt text: inspect an image in browser dev tools and confirm the alt attribute is descriptive
- Test old URL redirect: try the previous product URL and confirm an HTTP 301 to the new URL (e.g., curl -I https://yourdomain.com/old-url)
- Robots: visit https://yourdomain.com/robots.txt and ensure rules don’t block product URLs; also include "Disallow: /catalogsearch/" to prevent internal search pages from being indexed
- Sitemap: open your configured sitemap URL (e.g., https://yourdomain.com/sitemap.xml or https://yourdomain.com/media/sitemap/sitemap.xml) and confirm it loads without errors
- Synonyms: on the storefront, search for a synonym you added and confirm relevant results. Then in Admin go to Marketing > SEO & Search > Search Terms to verify the query was recorded and review Results/Uses
- Structured data: test a product URL in Google’s Rich Results Test and ensure Product schema includes name, price, availability, and ratings. If missing, work with your theme/developer to add JSON-LD
- Track KPIs weekly:
  - In Google Search Console: impressions, average position, and CTR for key product URLs
  - In GA4/Adobe Analytics: search refinement/exit and purchase rate after search (view_search_results → purchase)
  - In Magento: Marketing > SEO & Search > Search Terms for query volume and results count

## Common Issues and Solutions

- Product not appearing in search
  - Fix: Set Visibility = Catalog, Search and ensure key attributes have Use in Search = Yes; verify the product is enabled and salable (stock/backorders/MSI source assignment); confirm Product in Websites includes the target site; refresh invalidated caches and reindex Catalog Search if product data was changed
- 404 after changing URL suffix
  - Fix: Restore correct suffix (e.g., .html), create 301 redirects for changed URLs, and avoid live changes without testing
- Synonyms not working
  - Fix: If using native search: clear Magento cache and retest in the storefront (no reindex required). Verify the synonym group Scope matches the target Website. If using Adobe Live Search: configure synonyms in the Live Search admin and allow a few minutes for propagation.
- Sitemap 404 or not generated
  - Fix: Ensure Filename = sitemap.xml and Path = / (or /media/sitemap/ on cloud); verify the folder is writable; re-generate and confirm URL accessibility; cron must be enabled for scheduled generation
- Site set to noindex
  - Fix: Content > Design > Configuration > [Store View] > Search Engine Robots = Index, Follow
