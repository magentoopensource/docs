---
title: "How to Set Up Site Search and Navigation"
description: "Configure search functionality, filters, and intuitive navigation menus"
type: how-to
audience:
  - "Magento merchants"
difficulty: Intermediate
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
tags:
  - how-to
  - complex
last_updated: "2025-09-12T06:40:57.543Z"
---

# How to Set Up Site Search and Navigation

## Overview

Set up fast, accurate site search and intuitive navigation to help shoppers find products quickly, reduce bounce, and increase conversion. This how-to guides a growing merchant through configuring Magento’s built-in search, filters (layered navigation), and menu-driven category navigation for measurable business impact.

Track KPIs such as search-to-product-view rate, zero-results rate, and conversion rate from search. Aim for under 10% zero-results rate and increased search-driven revenue after tuning. If metrics underperform: lower zero-results by adding synonyms and redirects for top queries; boost relevance by raising Search Weight for high-intent attributes (e.g., SKU, Brand); and reduce friction by enabling autocomplete and cleaning stop words. Reducing zero-results by 5 percentage points can lift search-driven revenue by 3–8% by salvaging high-intent sessions.

Note on search engine compatibility: Magento 2.4.0–2.4.3 require Elasticsearch 7.x. Magento 2.4.4–2.4.5 support OpenSearch 1.x and Elasticsearch 7.x. Magento 2.4.6+ support OpenSearch 2.x and Elasticsearch 7.17.x. Always verify exact versions in the System Requirements for your Magento version: https://experienceleague.adobe.com/docs/commerce-operations/installation-guide/system-requirements.html. UI labels may vary slightly by version.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Elasticsearch/OpenSearch installed and reachable from the web server (version supported by your Magento version)
- Admin credentials with permission to change Catalog and Configuration settings
- Cron configured and running (for scheduled indexing)
- Ability to flush caches and run reindex (CLI access recommended for production)
- Network access from the web server to your OpenSearch/Elasticsearch host and port (default 9200) is allowed by firewalls/security groups
- If using HTTPS, a valid certificate is installed on the search cluster
- Quick connectivity test (from the web server): curl -k https://HOST:PORT or curl http://HOST:PORT should return version info

## What You'll Accomplish

By following this guide, you will:

- Set up site search and navigation using Magento’s built-in capabilities
- Improve your store's performance and customer experience

## Step-by-Step Instructions

### Step 1: Select and save the search engine

Go to Stores > Configuration > Catalog > Catalog > Catalog Search. In Search Engine, select the supported engine for your Magento version (Elasticsearch 7 for 2.4.0–2.4.3; OpenSearch or Elasticsearch for 2.4.4+).

Configure the connection fields (labels vary by version):
- Server Hostname
- Server Port
- Index Prefix
- Enable SSL/Use HTTPS
- Enable HTTP Authentication (if required)
- Username and Password (if your cluster requires authentication)
- Timeout

Click Test Connection (if available), then click Save Config, and flush caches when prompted.

### Step 2: Tune core search behavior

In the same Catalog Search section:
- Set Minimum Query Length and Maximum Query Length.
- Enable Search Suggestions = Yes to show “Did you mean” on the search results page, and set Search Suggestions Count (e.g., 5–10).
- Enable Search Recommendations = Yes to show “Related search terms” on the search results page, and set Search Recommendations Count.
- Enable Search Autocomplete = Yes to show the dropdown while typing, and set Autocomplete Limit (also labeled “Maximum Number of Autocomplete Results” in some versions) to control how many items appear in the typing dropdown.

Save.

### Step 3: Add search synonyms

Go to Marketing > SEO & Search > Search Synonyms. Select the Store View. Click Add Synonym Group. Enter comma-separated equivalents (e.g., tee, t-shirt, tshirt). Also consider brand abbreviations (e.g., "NKB" -> "Nike"), regional spellings ("colour" -> "color"), and common misspellings to capture high-intent traffic. Save.

After you save: Reindex Catalog Search (System > Tools > Index Management > Catalog Search > Reindex Data) or wait for the next scheduled reindex.

### Step 4: Add stop words

Go to Marketing > SEO & Search > Stop Words. Select the Store View. Click Add New Stop Word, add common words to ignore (e.g., the, and), and Save.

After you save: Reindex Catalog Search or wait for the next scheduled reindex.

### Step 5: Boost relevance for key attributes

Go to Stores > Attributes > Product. Open high-intent attributes (e.g., Brand, SKU, Model). Under Storefront Properties, set Use in Search = Yes and raise Search Weight (e.g., 5–10 for Brand/SKU). Save each attribute.

Note: The Search Weight field appears only after setting Use in Search = Yes. Valid weights are 1 (lowest) to 10 (highest).

After you save: Reindex Product EAV and Catalog Search or wait for the next scheduled reindex.

### Step 6: Enable attributes for layered navigation

In Stores > Attributes > Product, open attributes you want as filters (Brand, Size, Color). For each attribute you want as a filter:
- Catalog Input Type = Dropdown, Multiple Select, Visual Swatch, or Text Swatch
- Scope = Global
- Storefront Properties: Use in Layered Navigation = Filterable (with results)
- Optional: Use in Search Results Layered Navigation = Yes
- Save

Important: For an attribute to be used in layered navigation, its Scope must be Global (Stores > Attributes > Product > [Attribute] > Properties > Scope = Global). If set to Website or Store View, layered navigation options are unavailable.

Product data readiness checklist:
- Add the attribute to all relevant Attribute Sets (Stores > Attributes > Attribute Set)
- Ensure products in the category have non-empty values for the attribute in the current Store View
- Reindex Category Products and Product EAV

Optional global settings: Go to Stores > Configuration > Catalog > Catalog > Layered Navigation. Set Price Navigation Step Calculation (Automatic or Manual), Price Navigation Step, and Display Product Count = Yes to show counts next to filters. Save.

After you save: Reindex Product EAV and Category Products or wait for the next scheduled reindex.

### Step 7: Turn on layered navigation per category

Go to Catalog > Categories. Select a main category. Under Display Settings, set Is Anchor = Yes. Save. Repeat for key categories.

Note: Filters show only if products in the category have values for the filterable attributes and there are at least two distinct values. Verify attribute scope and values for the current Store View.

After you save: Reindex Category Products or wait for the next scheduled reindex.

### Step 8: Build a clear category hierarchy for top navigation

Do:
- Create or reorganize 6–10 top-level categories under your store’s root category
- For each category: set Is Active = Yes and Include in Menu = Yes
- Add 1–2 levels of subcategories where needed
- Use the Position field to control order
- If you create a new root category, assign it at Stores > All Stores > (Your Store) > Root Category so it appears in the store’s top navigation
- Save

Best practices:
- Use shopper-friendly labels (e.g., "Men", "Women", "Sale") and limit top-level items to 5–8 for scannability
- Place highest-margin or seasonal categories first
- Track header menu click-through rate (CTR) and revenue per category in your analytics, then reorder monthly to surface highest-CTR or highest-margin categories. Reordering top navigation can shift 2–5% more traffic to profitable categories

### Step 9: Set category display mode and sorting

Still in Catalog > Categories, for each major category under Display Settings, set Display Mode to Products Only or Static Block and Products. Under Display Settings, set Default Product Listing Sort By and Available Product Listing Sort By.

Set Default Product Listing Sort By to Position, Product Name, or Price (the options available out of the box). Note: Relevance applies only to search results. Best Sellers requires a custom attribute or an extension.

Save.

### Step 10: Reindex and refresh caches

Ensure cron is running and indexers are set to Update by Schedule. If needed, go to System > Tools > Index Management, select the relevant indexers, choose Reindex Data, and Submit.

Reindex guidance:
- After synonyms/stop words/search settings: Catalog Search
- After attribute changes (Use in Search, Search Weight, Layered Navigation): Product EAV, Catalog Search
- After category Is Anchor and hierarchy changes: Category Products

Then go to System > Cache Management and refresh Page Cache and Block HTML Output (or Flush Magento Cache if unsure).

### Step 11: Validate storefront search and filters

On the storefront, perform searches for Brand, SKU, and a descriptive term. Check search suggestions (“Did you mean”) and confirm relevant products appear. Type the first 2–3 letters of a popular query and confirm the autocomplete dropdown shows up to the configured Autocomplete Limit. Open a major category and apply 2–3 filters (Brand, Price, Size). Clear filters and repeat.

### Step 12: Monitor search terms and optimize

After sufficient traffic (often within 24–72 hours), go to Reports > Marketing > Search Terms to analyze search term performance. To manage redirects or terms, go to Marketing > SEO & Search > Search Terms. Add synonyms or adjust attribute Search Weight as needed.

Note: Search Terms, Synonyms, and Stop Words are scoped per Store View. Review and optimize per storefront locale (use the store switcher at the top-left of each page).

Create high-impact redirects for common navigational queries:
- "sale" -> /sale
- "gift card" -> /gift-card
- "new" -> /new-arrivals
- "clearance" -> /outlet

For navigational or support queries (e.g., "returns", "size chart"), create Search Term entries with Redirect URLs to the appropriate CMS pages or category landing pages to improve conversion and reduce support contacts. Systematic synonym/redirect updates on top 50 queries typically improve search conversion 5–15% over 60 days.

Suggested 30/60/90-day optimization cadence:
- Days 1–30: Fix zero-results queries; add synonyms for top 20 queries; enable autocomplete and tune Autocomplete Limit; raise Search Weight for SKU/Brand
- Days 31–60: Add redirects for navigational queries; expand synonyms to top 50 queries; prune stop words; validate layered navigation coverage on top categories
- Days 61–90: Review analytics, reorder top navigation by CTR/margin; A/B test Autocomplete Limit and suggestions count; refine attribute weights and filter set

## Verification

To confirm everything is working correctly:

- Run a search for a known SKU and confirm the product appears on the first results page.
- Type the first 2–3 letters of a popular query and confirm the autocomplete dropdown shows up to the configured Autocomplete Limit.
- Open a major category and confirm filter facets (Brand, Price, Size) appear with product counts.
- On the search results page, confirm layered navigation appears if enabled for attributes.
- Confirm category pages respect the configured default sort (Position, Product Name, or Price).
- Navigate to the admin panel and verify changes are visible and saved correctly.

## Common Issues and Solutions

- Issue: No search results returned.
  - Solution: Verify Elasticsearch/OpenSearch service is running and reachable; check Stores > Configuration > Catalog > Catalog Search connection settings; reindex Catalog Search; check var/log/system.log and var/log/exception.log for connection errors.
- Issue: Filters not showing in category.
  - Solution: Ensure category Is Anchor = Yes; attribute input type is Dropdown/Multiple Select/Visual Swatch/Text Swatch; Scope = Global; Use in Layered Navigation = Filterable (with results); attribute is in the correct Attribute Sets; products have attribute values; reindex Product EAV and Category Products; clear cache.
- Issue: Synonyms not applied.
  - Solution: Confirm Store View selection matches storefront; reindex Catalog Search; clear caches; ensure synonyms are created for the correct Store View.
- Issue: Slow search.
  - Solution: Verify search engine hardware resources; reduce Autocomplete Limit and Search Suggestions Count; ensure production mode and full-page cache are enabled; consider upgrading to OpenSearch on Magento 2.4.4+ if supported.

## Related Resources

- Adobe Commerce (Magento) User Guide: https://experienceleague.adobe.com/docs/commerce-admin/user-guides/home.html
- Adobe Commerce Live Search (Commerce only): https://experienceleague.adobe.com/docs/commerce-merchant-services/live-search/overview.html

---

*Need help? [Contact support]() or check our [troubleshooting guide]().*
