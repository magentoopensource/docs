---
title: "Designing a Professional Store Frontend"
description: "Complete tutorial on customizing theme, layout, and visual design"
type: tutorial
audience:
  - "Magento merchants"
  - "Advanced users"
difficulty: Intermediate
estimated_time: "9 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Basic PHP knowledge"
  - "Magento development environment"
tags:
  - tutorial
  - complex
last_updated: "2025-09-15T10:39:04.225Z"
---

# Designing a Professional Store Frontend

Note: Applies to Magento Open Source and Adobe Commerce 2.4.3+. Page Builder is available in Open Source 2.4.3+ and
Adobe Commerce. UI labels reflect 2.4.6. If your Admin looks different, check your version or ask your administrator.

Scope: This is a no‑code tutorial that uses Admin-only tools (Page Builder or the WYSIWYG editor). For developer
theming (child themes, LESS/CSS), see the developer documentation.

## Why This Tutorial Matters

**Business Impact:** A professional storefront increases conversion, average order value, and brand trust. With the
right theme and layout, you can launch campaigns faster and reduce reliance on developers. Improving homepage structure
and CTAs typically increases CTR by 10–20% and conversion by 2–5%. Scheduling seasonal themes can reduce launch time
from days to hours, lowering operational costs.

**What You'll Achieve:**

- Assign a theme to specific store views
- Configure logo, favicon, typography, and basic branding
- Build a high‑converting homepage using Page Builder (or TinyMCE)
- Optimize category and product page layouts
- Schedule seasonal theme changes
- Validate changes and resolve common display issues

## Learning Journey Overview

### Your Situation

You need to refresh your storefront without writing code. You may have a new theme from a vendor, want to update
branding, or need to launch a seasonal campaign across multiple store views.

### What You'll Learn

By completing this tutorial, you will:

- Assign a theme per website/store view (Content > Design > Configuration)
- Update branding (logo, favicon, header/footer) for each store view
- Create a homepage with Page Builder and publish it as the default CMS page
- Use Blocks and Widgets to promote categories, products, and promos
- Adjust page layouts for categories and products
- Schedule and preview design changes
- Clear caches, validate changes, and troubleshoot display issues

### Success Criteria

You'll know you've succeeded when:

- A theme is assigned to the correct Store View in Content > Design > Configuration
- Your homepage is live and set as Stores > Configuration > General > Web > Default Pages > CMS Home Page
- Logo and favicon are visible on the storefront
- Category and product page layouts reflect your chosen configurations
- A scheduled design change has been created and previewed
- The storefront renders correctly after cache flush, with expected content and styles
- KPIs to monitor post-launch: homepage CTR, bounce rate, add‑to‑cart rate, and category‑to‑product click‑through

### Time Investment

- Estimated time: 45–90 minutes depending on the number of pages and whether Page Builder is enabled
- Skill level after completion: Confident in no‑code storefront configuration and content updates
- Business value unlock: Faster campaign launches, improved conversion, reduced developer dependency

## Before We Start

### Who This Is For

This tutorial is designed for:

- Store owners and ecommerce managers responsible for site experience
- Content editors/merchandisers using Page Builder to manage the homepage and landing pages

### What You Need

Make sure you have:

- Magento Admin login with permissions for Content and Design
- A tested theme installed (ask your developer/vendor)
- Brand assets: logo (PNG/SVG), favicon (16×16 or 32×32 PNG/ICO), hero images, and color references
- A staging site to test changes
- If your site runs in Production mode, access to a developer/host to deploy static content when needed

### Preparation Checklist

Before starting, complete these preparation steps:

1) Confirm your target Store View in the Admin store switcher.
2) Back up current theme settings or take screenshots of Design Configuration.
3) Upload media assets to Content > Media Gallery.
4) Optionally disable Full Page Cache at System > Cache Management during setup; re‑enable after you finish.
5) Inform your developer if your site runs in Production mode and coordinate any required static content deploy.

## Step-by-Step Learning Path

Step 1 — Assign your theme

- Go to Content > Design > Configuration.
- In the table, find the row for your target Store View and click Edit.
- Under Applied Theme, select your theme and click Save Configuration.
- If prompted, flush caches at System > Cache Management.

Step 2 — Configure branding (logo, favicon, header/footer)

- From the same Store View design configuration, open the Header section and upload the Logo Image. Set Logo Image Width
  and Height as needed.
- Open HTML Head and upload the Favicon Icon. Set Default Page Title and Title Suffix for SEO/branding.
- Open Footer to update Copyright text and footer links as needed.
- Save Configuration and flush cache if prompted.

Step 3 — Create and set your homepage

- Go to Content > Elements > Pages and click Add New Page.
- Title: Home. Set Enable Page to Yes.
- Content: If Page Builder is available (2.4.3+), click Content and then Edit with Page Builder. Add a Row, then a
  Banner with a hero image, headline, supporting text, a primary CTA button, and link to a key category or landing page.
  If Page Builder is not available, use the WYSIWYG editor to add similar content and links.
- Click Save.
- Set as the storefront home page: go to Stores > Settings > Configuration > General > Web > Default Pages. Set CMS Home
  Page to "Home" (or your page title). Click Save Config and flush caches if prompted.

Step 4 — Promote content with Blocks and Widgets

- Blocks: go to Content > Elements > Blocks > Add New Block. Create a block such as "Homepage Promo". Build content
  using Page Builder (or WYSIWYG) and click Save.
- Widgets: go to Content > Elements > Widgets > Add Widget. Choose Type: Catalog Products List or CMS Static Block.
  Select your Design Theme and click Continue.
- In Storefront Properties, set the Title, Assign to Store Views, and Sort Order.
- Under Layout Updates, choose where to display it (for example, Specified Page > CMS Home Page) and select a container
  such as Main Content Area or Sidebar Main. Configure widget options (for example, show New products, a specific
  category, or a Static Block) and click Save.

Step 5 — Optimize category and product layouts

- Categories: go to Catalog > Categories. Select a category. Open the Design tab to set the Page Layout (for example, 2
  columns with left bar). In Display Settings, adjust products per page, default sorting, and mode (Grid/List). Click
  Save.
- Products: go to Catalog > Products. Edit a product. Open the Design tab and set the Page Layout if needed (for
  example, 1 column for focused PDP). Click Save.

Step 6 — Schedule design changes (optional)

- Go to Content > Design > Schedule and click Add New Design Change.
- Select the Website/Store View, choose the Applied Theme, and set the date range (for a seasonal campaign or holiday
  refresh). Click Save.
- Note: Adobe Commerce customers can also use Content Staging for CMS content timing. Design Schedule is available in
  both editions.

Step 7 — Validate and publish

- Go to System > Cache Management and click Flush Magento Cache (and clear Full Page Cache/Varnish if used).
- Open the storefront in an incognito/private window. Use the store switcher to confirm the correct Store View.
- If assets or styles are missing in Production mode, contact your developer/host to run:
    - bin/magento setup:static-content:deploy -f
    - bin/magento cache:flush

## Practice and Reinforcement

Now that you've learned the core process, let's reinforce your skills:

- Replace the homepage hero banner and link to your top‑converting category.
- Create a "Featured Collection" block and add it via a widget to the homepage.
- Change a category to a 1‑column layout and compare click‑through to product pages for 7 days.

## What You've Accomplished

🎉 **Congratulations!** You have successfully:

- Assigned a theme to the correct Store View
- Configured branding (logo, favicon, header/footer)
- Built and published a homepage and set it as the default CMS page
- Added promotional content via Blocks and Widgets
- Optimized category and product layouts
- Scheduled and validated a theme change

### Business Impact

- Faster campaign launches with less developer involvement
- Improved homepage CTR and conversion from clearer messaging and CTAs
- More consistent brand presentation across store views

### Skills Gained

You now have the ability to:

- Configure themes and branding without code
- Build high‑impact CMS pages using Page Builder
- Place targeted promotions with Blocks and Widgets
- Tune layouts for categories and product pages
- Schedule and validate design changes safely

## Next Steps in Your Journey

Now that you've mastered this process, here's how to build on your success:

### Immediate Actions

- Build a campaign landing page for your next promotion using Page Builder templates
- Add a "Top Sellers" product list widget to key pages
- Review analytics to measure CTR, bounce rate, and add‑to‑cart rate

### Level Up Your Skills

- Create reusable Page Builder templates for seasonal campaigns
- Localize branding and content per store view for international markets
- Establish a content governance checklist for safe deployments

### Advanced Applications

- Multi‑store theming and localization across regions
- A/B test hero banners and CTA placements with your analytics/experimentation tools
- Use Content Staging (Adobe Commerce) to orchestrate complex campaign timelines

## When Things Don't Go as Expected

Even experienced merchants encounter challenges. Here's how to handle common situations:

- Theme not visible on storefront — Ensure you edited the correct Store View in Content > Design > Configuration. Flush
  caches. In Production mode, request a static content deploy.
- Logo/favicon not updating — Hard refresh the browser (Ctrl/Cmd+Shift+R). Verify you uploaded at the Store View scope,
  not Default Config. Clear Full Page Cache/Varnish.
- Homepage not displaying your new page — Confirm Stores > Configuration > General > Web > Default Pages > CMS Home Page
  points to your new page.
- Page Builder not available — You may be on a version earlier than 2.4.3 or Page Builder is disabled. Use the WYSIWYG
  editor or ask an admin to enable Page Builder.
- Widgets not showing — Check widget Storefront Properties scope, Layout Update placement, and container. Clear caches
  after changes.

## Continue Learning

### Related Tutorials

- Building Landing Pages with Page Builder

### How-To Guides

- Create and Place Widgets (Product Lists, Static Blocks)
- Design Configuration Fields by Scope

### Reference Materials

- Multi‑Store Theming and Localization Best Practices

## Summary

A polished storefront can be delivered without code by assigning the right theme, configuring branding, and building
rich content with Page Builder. Use widgets and layout settings to promote products and streamline navigation. Validate
changes, monitor KPIs, and iterate based on results.

### Key Takeaways

- Work at the correct Store View scope before making changes
- Page Builder enables faster, more flexible content updates
- Cache management and validation are essential for accurate storefront results

### Remember

- Always test on staging first and coordinate static content deploys in Production
- Document your design settings so you can roll back quickly if needed
