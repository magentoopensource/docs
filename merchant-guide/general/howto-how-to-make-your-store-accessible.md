---
title: "How to Make Your Store Accessible"
description: "Implement accessibility features for users with disabilities"
type: how-to
audience:
  - "Magento merchants"
difficulty: Intermediate
estimated_time: "2–4 hours (merchant tasks) + developer time as needed"
prerequisites:
  - "Access to Magento Admin Panel"
tags:
  - how-to
last_updated: "2025-09-16T20:08:03.772Z"
---

# How to Make Your Store Accessible

## Overview

Make your Magento 2 storefront accessible to customers with disabilities to increase conversions, reduce legal risk, and improve SEO. This how-to guides a Growing Merchant through configuring built-in settings, optimizing content, and validating results to move your store toward WCAG 2.1 AA outcomes with minimal theme changes where possible. This guide moves your storefront toward WCAG 2.1 AA alignment using configuration and content updates. Full compliance may require theme and extension code changes and a formal audit.

Business value and KPIs to track:
- Checkout completion rate, form submission success rate, and average time to checkout
- Support ticket volume for access issues and spam/bot signups
- Expect quick wins from improved contrast, clearer labels, and lower reCAPTCHA friction

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Staging environment with production-like theme, data, and extensions
- Admin role with permissions for Stores > Configuration and Content
- Chrome with Lighthouse and an accessibility extension (axe or WAVE) installed
- Google reCAPTCHA keys for your chosen type (v2 Checkbox, v2 Invisible, or v3 Invisible)
- Developer access for theme/CSS changes and to verify theme version via composer

## What You'll Accomplish

By following this guide, you will: Configure key accessibility settings, improve content for screen readers and keyboard users, and validate improvements with audits on home, category, product, and checkout pages.

You will specifically:
- Enable breadcrumbs on CMS pages
- Configure Google reCAPTCHA on key forms with verified behavior and no double challenges
- Add meaningful alt text for top revenue products (at least top 20 images)
- Correct headings and link text on Home and key landing pages
- Ensure captions/transcripts for product videos
- Achieve at least 4.5:1 contrast on key buttons/text (3:1 for large text) on priority pages
- Validate keyboard-only checkout path without traps
- Capture before/after audit results and log remaining developer tickets

## Step-by-Step Instructions

### Step 0: Establish an accessibility baseline with audits

In a staging environment that mirrors production, open your storefront in Chrome. Run Lighthouse (DevTools > Lighthouse) on Home, a top Category, a top Product, and Checkout pages. Install and run the WAVE or axe extension on the same pages. Note critical issues (contrast, missing alt text, form labels, headings). Estimated time: 20–30 minutes.

### Step 1: Verify active theme and assess accessibility posture

In Magento Admin, go to Content > Design > Configuration and confirm your active theme. Magento Admin does not display the installed theme version; ask your developer to run `composer show` or check `composer.lock` on the server to verify the theme version. Check the theme vendor’s release notes for accessibility improvements. If your theme is outdated or known to have issues, schedule an update. If the theme lacks accessibility claims, plan a later evaluation or replacement.

### Step 2: Enable breadcrumbs for CMS pages

In Admin, go to Stores > Configuration > General > Web > Default Pages. Set Show Breadcrumbs for CMS Pages to Yes. Save Config and clear cache if prompted (System > Cache Management > Flush Magento Cache).

### Step 3: Configure accessible reCAPTCHA for storefront forms

Go to Stores > Configuration > Security > Google reCAPTCHA Storefront. Enter your site and secret keys (obtain from Google reCAPTCHA). Select the reCAPTCHA type that matches your Google keys: v2 Checkbox (most explicit with an available audio challenge), v2 Invisible, or v3 Invisible. Enable for customer login, registration, forgot password, newsletter subscription, Contact Us, and checkout-related forms.

- If using Google reCAPTCHA, disable native CAPTCHA to avoid double challenges: Stores > Configuration > Customers > Customer Configuration > CAPTCHA > Enable CAPTCHA = No.
- For v3 Invisible: set Minimum Score to 0.3–0.5 to avoid blocking legitimate customers. Review failed actions in Google’s dashboard and adjust if needed. v3 does not show challenges.
- After saving, go to System > Cache Management and Flush Magento Cache.
- Conversion guidance: Use v3 Invisible for checkout to minimize friction; use v2 Checkbox for high‑abuse forms like Contact Us to deter spam with minimal impact.

### Step 4: Add descriptive alt text to product images

Go to Catalog > Products. Edit a top-selling product. In Images and Videos, click each image and fill the Alt Text field with a concise description (e.g., "Men’s waterproof hiking jacket, blue").

Best practices and prioritization:
- Leave alt text empty for purely decorative imagery to avoid screen reader noise.
- Use concise, specific alt text that conveys differentiators (material, color, key feature).
- Prioritize top revenue SKUs first (top 80% revenue); schedule the rest.
- For large catalogs, plan a sprint using product export lists and internal SOPs; assign to content ops.

Save. Batch through your top 20 products to start.

### Step 5: Fix headings and link text in CMS content (Page Builder or WYSIWYG)

Go to Content > Pages and edit your Home and key Landing pages. Use Page Builder Heading content type with appropriate levels (H1 for page title once; H2/H3 for sections). For links, replace vague text like "Click here" with meaningful labels like "Shop men’s jackets." For images, open the Image content type and set Image Description (alt text). Save and publish.

Also ensure product swatches and attribute labels are accessible:
- Go to Stores > Attributes > Product. Open your Color/Size attributes.
- Ensure each option label is human-readable (e.g., "Navy" not "#001F3F"). For Visual Swatch, verify the "Admin" and "Default Store View" labels are set.
- Save and reindex if prompted.

### Step 6: Ensure product videos include captions and transcripts

Upload captions to YouTube/Vimeo for your product videos. In Magento, open a product, go to Images and Videos, click Add Video, and paste the video URL. Ensure captions are enabled on the video platform and add a transcript link in the product description if needed. Save.

### Step 7: Improve color contrast for text and buttons on key pages

Use Page Builder to fix contrast on specific CMS blocks (text and buttons). Select text or button elements and adjust colors to meet at least a 4.5:1 contrast ratio (3:1 for large text). Validate with a contrast checker.

For sitewide elements (buttons, links, header/footer, checkout), create a child theme and adjust variables/CSS (e.g., primary button background/hover, link color, visible focus outline). Log a developer ticket with screenshots and measured contrast ratios.

### Step 8: Name form fields clearly and verify error messages

Test storefront forms (account registration, checkout). Ensure field labels are visible and meaningful. For core forms, update labels via i18n translation (app/i18n CSV or a language pack) or extension settings. Avoid relying on placeholders; ensure visible `<label>` text. Trigger validation errors and confirm they are clear and programmatically associated to fields. If issues appear, document and assign to a developer for code-based changes.

### Step 9: Run a keyboard-only checkout test

Perform this first in a staging environment. From your storefront, use only Tab, Shift+Tab, Space, and Enter to: open navigation, select a category, enter a product page, choose options, add to cart, open mini cart, proceed to checkout, and complete a test order using a sandbox payment method. If focus gets trapped or lost, record the step and theme component for remediation.

### Step 10: Perform a quick screen reader scan

In staging, with NVDA (Windows) or VoiceOver (Mac), navigate a product page and the checkout. Check headings outline, alt text announcements, button/link names, and feedback messages (e.g., add-to-cart confirmation). Note any silent or mislabeled controls for developer follow-up.

### Step 11: Re-run audits and document improvements

Re-run Lighthouse, WAVE, or axe on the same set of pages. Compare scores and issue lists to your baseline. Capture before/after screenshots. Create tickets for remaining theme-level issues (e.g., focus management in modals). Share results with stakeholders: a simple slide showing baseline vs. current scores, resolved issue counts, and next-step dev tickets with ETA helps secure budget for remaining fixes.

## Verification

Use this checklist to confirm each change:
- Breadcrumbs: Open a CMS page and confirm the breadcrumb trail renders and is keyboard-focusable.
- reCAPTCHA: Trigger each enabled form; confirm expected behavior (v2 challenge/audio available when needed; v3 submits without challenge) and successful submission. Verify native CAPTCHA is disabled if using Google reCAPTCHA.
- Alt text: Inspect images in dev tools (ensure img elements have appropriate alt attributes) and with a screen reader; confirm meaningful announcements.
- Headings/links: Use a headings outline (e.g., NVDA Insert+F7) and verify a single H1, logical H2/H3 structure, and meaningful link names.
- Contrast: Validate text and buttons with a contrast checker and document ratios (≥4.5:1 for normal text, ≥3:1 for large text).
- Keyboard checkout: Complete end-to-end with only Tab/Shift+Tab/Enter/Space; verify visible focus, no traps, and that modals return focus to the trigger on close.
- Screen reader: Confirm add-to-cart confirmations, error messages, and inline validation are announced.

## Common Issues and Solutions

- Double CAPTCHA prompts: Disable native CAPTCHA when Google reCAPTCHA is enabled (Customers > Customer Configuration > CAPTCHA > Enable CAPTCHA = No).
- v3 blocking legitimate users: Lower Minimum Score to 0.3–0.4 and retest; review action metrics in Google’s console.
- Keyboard trap in mini cart or modals: Update the theme or extension; ensure focus is trapped inside while open and returns to the triggering control on close.
- Low contrast buttons from theme: Implement child theme CSS overrides and adjust design tokens/variables.
- Autoplaying sliders: Disable autoplay in Page Builder or add a visible pause/stop control.

## Related Resources

- [Adobe Commerce User Guide](https://experienceleague.adobe.com/en/docs/commerce-admin)

---

Need help? Contact your support channel or review the Adobe Commerce documentation linked above.
