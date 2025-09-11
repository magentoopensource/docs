---
title: "How to Improve Site Loading Speed"
description: "Optimize images, caching, and performance for faster page loads"
type: how-to
audience:
  - "Magento merchants"
difficulty: Intermediate
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
tags:
  - how-to
  - performance
last_updated: "2025-09-11T12:39:05.599Z"
---

# How to Improve Site Loading Speed

## Overview

Improve conversion and SEO by reducing your Magento 2 storefront loading times. Faster pages lower bounce rate, increase add-to-cart and checkout completion, and reduce infrastructure costs. This how-to shows a growing merchant how to optimize images, caching, and frontend assets using Magento Admin, with optional advanced steps for developers/CDN.

Improving Core Web Vitals such as LCP and INP can raise conversion rates and lower ad cost-per-click via better Quality Score. As a benchmark, many merchants see a 1–3% conversion uplift per ~0.5s LCP improvement on mobile. Using a CDN with Full Page Cache (FPC) often offloads 50–80% of origin traffic, reducing hosting costs.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Staging environment to test changes
- Maintenance window and backup/restore plan
- CDN account and DNS control (for Step 4)
- Varnish server available (for Step 2)
- SSH access for a developer to run Magento CLI commands (for Step 10 and Production-only settings)
- Admin permissions to edit Configuration at the correct scope (Default/Website/Store)

## What You'll Accomplish

By following this guide, you will:

- Improve site loading speed with Magento 2 best practices
- Improve your store's performance and customer experience

## Step-by-Step Instructions

### Step 0: Measure your baseline performance

Priority: Quick Win

Open Google PageSpeed Insights and test your Home, a busy Category, and a high-traffic Product page (mobile). Record LCP, INP, CLS, TTFB, and overall score. Note that INP replaced FID in 2024. For consistency, run PageSpeed Insights for mobile. In Chrome DevTools, emulate a mid-tier mobile device and Fast 3G/Slow 4G network conditions.




### Step 1: Enable and refresh Magento caches

Priority: Quick Win

Go to System > Tools > Cache Management. Click Select All. From the Actions dropdown, choose Enable and submit. Then click Flush Magento Cache. Also click Flush Catalog Images Cache to reset resized images.




### Step 2: Configure Full Page Cache (FPC)

Priority: Developer/Advanced

Go to Stores > Configuration > Advanced > System > Full Page Cache. Set Caching Application to Built-in Application if you do not have Varnish.

If you already have Varnish available, select Varnish Cache, choose the correct Varnish version (6.x), set TTL (keep default 86400 unless advised otherwise), and Save Config. Click Export VCL for your version, provide the VCL to your developer to upload to the Varnish server, and reload the Varnish service. Verify that your store domain is routed through Varnish.




### Step 3: Minify CSS and JavaScript (HTTP/2-friendly)

Priority: Quick Win (Admin). Developer required in Production mode.

Go to Stores > Configuration > Advanced > Developer. Under CSS Settings, set Minify CSS Files = Yes and Merge CSS Files = No. Under JavaScript Settings, set Minify JavaScript Files = Yes, Enable JavaScript Bundling = No, Merge JavaScript Files = No, and Move JS code to the bottom of the page = No (recommended default). If you enable Yes for Move JS code to the bottom, thoroughly test checkout, cart, and dynamic widgets on a staging site first. Under Template Settings, set Minify HTML = Yes. Save Config and Flush Cache.

Note: In Production mode, these fields may be locked or hidden. Have your developer change them via CLI (bin/magento config:set ...) or config.php and deploy. If fields are unavailable, ask your developer to run:

```
bin/magento config:set dev/css/minify_files 1
bin/magento config:set dev/js/minify_files 1
bin/magento config:set dev/js/enable_js_bundling 0
bin/magento config:set dev/js/merge_files 0
bin/magento config:set dev/css/merge_css_files 0
bin/magento config:set dev/template/minify_html 1
# Optional (recommended default): keep scripts at top to avoid breakage
# bin/magento config:set dev/js/move_script_to_bottom 0
bin/magento cache:flush
```




### Step 4: Offload static and media files to a CDN (recommended)

Priority: Developer/Advanced

Ensure your CDN is set up to pull from your origin. In Magento Admin, go to Stores > Configuration > General > Web. Expand Base URLs and Base URLs (Secure). Set Base URL for Static View Files and Base URL for User Media Files to your CDN URLs (HTTPS, trailing slash). In the same page, set Use Secure URLs on Storefront = Yes and Use Secure URLs in Admin = Yes. Ensure your CDN serves HTTPS. Configure CORS headers on the CDN for fonts and static assets (for example, Access-Control-Allow-Origin: * or your domain). Save Config and Flush Cache. Then purge the CDN cache for /static and /media.

Business impact: A CDN typically cuts global asset latency by 50–80% and offloads the majority of static traffic from your server, improving SEO and lowering origin costs.




### Step 5: Identify and replace heavy images

Priority: Quick Win

Go to Content > Media > Media Gallery. Sort by file size and open the largest items used on Home/Category/Product pages. For each, download and compress using an optimizer (target WebP or high-quality JPEG). Maintain correct dimensions for display (avoid oversized originals). When replacing an image used in CMS content, keep the same file path and filename to avoid broken links. Consider using an image CDN or automation (for example, automatically serve WebP/AVIF and resize by device) to reduce manual effort. Replace the file in Media Gallery (or update CMS blocks/banners) and Save. After replacing a batch, Flush Catalog Images Cache.




### Step 6: Verify lazy loading and responsive images

Priority: Quick Win

Open a Category page in your browser and View Source or use DevTools Elements panel. Check that product and content images include loading="lazy" and responsive attributes (srcset/sizes). If missing, consult your theme vendor or developer to enable native lazy loading and responsive images. Avoid custom code that disables lazy loading.




### Step 7: Reduce third-party scripts and pixel tags

Priority: Quick Win

Go to Content > Design > Configuration, edit your active theme scope. Under HTML Head, open the Scripts and Style Sheets field and review the contents. Remove unused widgets, legacy trackers, and synchronous script tags. Move remaining marketing tags into Google Tag Manager where possible and ensure they load asynchronously/deferred. Business impact: every blocking script delays LCP/INP and can reduce add-to-cart and checkout completion rates. Consolidating tags and deferring non-critical scripts often improves LCP by 200–500ms.




### Step 8: Ensure static files are signed for safe long caching

Priority: Quick Win (Admin). Developer required in Production mode.

Go to Stores > Configuration > Advanced > Developer > Static Files Settings. Set Sign Static Files = Yes. Save Config and Flush Cache. This enables cache-busting query parameters so browsers/CDNs can cache assets longer without risking stale files after deployments.

Note: In Production mode, these fields may be read-only or hidden. Ask your developer to set these values via CLI or config.php and deploy. If the field is unavailable in Production mode, ask your developer to run:

```
bin/magento config:set dev/static/sign 1
bin/magento cache:flush
```




### Step 9: Confirm indexers and finish with a cache refresh

Priority: Quick Win

Go to System > Tools > Index Management. Ensure all critical indexers are in Update by Schedule (recommended) and statuses are not Reindex Required. If any are required, select them and choose Reindex Data. Then return to System > Tools > Cache Management and Flush Magento Cache.




### Step 10: Optional (developer): Switch to Production mode and deploy assets

Priority: Developer/Advanced

Ask your developer to set Production mode and deploy static content for active locales and themes. This ensures pre-compiled templates and minified assets are served efficiently.

Developer CLI on the server:

```
bin/magento maintenance:enable
bin/magento deploy:mode:set production
bin/magento setup:di:compile
bin/magento setup:static-content:deploy -f en_US [add other locales] --jobs=4
bin/magento cache:flush
bin/magento maintenance:disable
# Verify
bin/magento deploy:mode:show
```




### Step 11: Re-measure and compare results

Priority: Quick Win

Repeat PageSpeed Insights tests for the same Home, Category, and Product URLs (mobile). Also perform a repeat-load test in your browser: load a category page twice and observe the second TTFB. Record improvements. Targets (mobile): LCP ≤ 2.5s, INP ≤ 200ms, CLS ≤ 0.1. If targets are not met, prioritize images, third-party scripts, and FPC/Varnish configuration and iterate.




## Verification

To confirm everything is working correctly:

1. Navigate to the admin panel and verify changes are visible
1. Check that all configurations are saved correctly
1. Test the functionality from a customer perspective
- Use browser DevTools > Network to confirm HTTP/2 or HTTP/3 and Content-Encoding: br or gzip for static assets.
- Check a category page response headers for X-Magento-Cache-Debug: HIT (logged-out) and an Age header increasing (if Varnish/CDN in front).
- Confirm static assets load from your CDN domain and include a version/signing query parameter.
- Run Lighthouse or PageSpeed Insights again and compare LCP/INP/CLS.
- Validate no mixed-content warnings appear in the console.

## Common Issues and Solutions

- Issue: X-Magento-Cache-Debug shows MISS.
  - Fix: Ensure FPC is enabled, the Varnish VCL is applied, and avoid cache-busting cookies on the storefront.
- Issue: Mixed content after setting CDN.
  - Fix: Enable Use Secure URLs on Storefront/Admin and update hard-coded http:// links in CMS content.
- Issue: JS errors after moving JS to bottom.
  - Fix: Revert the setting or defer only non-critical scripts; test on staging.
- Issue: Fonts blocked from CDN.
  - Fix: Add Access-Control-Allow-Origin header on CDN for font MIME types.
- Issue: Page speed regressions for logged-in users.
  - Fix: Ensure private content and hole-punching are working; avoid making dynamic blocks cacheable.

## Related Resources

- [Magento User Guide](https://experienceleague.adobe.com/docs/commerce-admin/user-guides/home.html)

---

*Need help? [Contact support](https://experienceleague.adobe.com/docs/commerce-knowledge-base/kb/overview.html) or check our [troubleshooting guide](https://experienceleague.adobe.com/docs/commerce-operations/performance-best-practices/overview.html).*