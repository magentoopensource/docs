---
title: "How to Add Legal Pages and Compliance"
description: "Create privacy policy, terms of service, and comply with regional regulations"
type: how-to
audience:
  - "Magento merchants"
difficulty: Advanced
estimated_time: "20–30 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Legal-reviewed content for each policy (Privacy Policy, Terms, Cookie Policy, and any regional pages)"
  - "An admin user with permissions to manage Content, Stores Configuration, and Widgets"
  - "Knowledge of your active theme and footer containers (or ability to consult your developer)"
  - "If you use a Consent Management Platform (CMP), access to CMP settings or your Tag Manager"
  - "Legal note: This guide is for technical setup only and is not legal advice. Work with your legal counsel to determine required disclosures and consent mechanisms for your jurisdictions."
  - "Version note: This guide applies to Magento Open Source/Adobe Commerce 2.4.x (Luma/Blank). If you use PWA Studio, Hyvä, or a heavily customized frontend, UI labels and behavior may differ."
tags:
  - how-to
  - high-impact
last_updated: "2025-09-11T11:27:40.779Z"
---

# How to Add Legal Pages and Compliance

## Overview

Add legally required pages and consent features in Magento 2 to build customer trust, reduce compliance risk, and prevent checkout friction. This how-to guides you to create Privacy Policy and Terms of Service pages, enable checkout terms acceptance, and configure cookie consent for regional regulations.

Clear legal pages and consent improve trust signals and reduce support tickets about data rights. Making store policies explicit at checkout can reduce disputes and chargebacks. Improved trust and transparency can lift checkout conversion by reducing hesitation while lowering compliance risk.

Quick mapping: regulation to actions
- GDPR/LGPD/UK GDPR: In Magento: publish Privacy Policy; enable checkout terms; enable cookie banner. Outside Magento: use a CMP/Tag Manager to gate third-party scripts and manage consent categories—Magento alone does not block third-party scripts.
- ePrivacy/Cookie laws (EU/UK): In Magento: display a consent banner before setting non-essential cookies (Magento banner). Outside Magento: gate analytics/ads until consent via a CMP/Tag Manager; Magento native features do not block third-party scripts.
- CCPA/CPRA (California): In Magento: add a “Do Not Sell or Share My Personal Information” page and link to it. Outside Magento: include a link labeled “Do Not Sell or Share My Personal Information” or “Your Privacy Choices” on the homepage and any page where you collect personal information. Many merchants implement this as a persistent footer link on all pages for consistency. Update Privacy Policy disclosures and reference GPC (Global Privacy Control) if supported.
- DACH (Germany/Austria/Switzerland): In Magento: add an Impressum/Legal Notice page; keep Returns/Revocation info easily accessible.
- All regions: In Magento: display Terms & Conditions acceptance at checkout and keep policies linked in the footer.

Non-compliance can be costly: GDPR/UK GDPR fines can reach up to 4% of global annual turnover; CCPA/CPRA penalties can be up to $7,500 per intentional violation.

Scope map (what you configure where)
- Global: Create CMS Pages and Checkout Agreements once, then assign to specific Store Views.
- Website: Enable Terms & Conditions; enable Cookie Restriction Mode; configure website-specific cookie settings.
- Store View: Assign localized CMS pages and agreements; manage translated content.
- Theme: Widgets are bound to a specific Design Theme.

Typical time to complete including testing across sites: 20–30 minutes.

Regional checklist
- EU/UK: Publish Privacy Policy, enable checkout Terms & Conditions, enable cookie banner (Cookie Restriction Mode), and use a CMP to gate non-essential scripts.
- US (California): Create a “Do Not Sell or Share My Personal Information”/“Your Privacy Choices” page and link it sitewide; update Privacy Policy disclosures and reference GPC if supported.
- DACH: Add an Impressum/Legal Notice and ensure Returns/Revocation policy is easy to find.
- All: Keep policy links in the footer and require Terms acceptance at checkout.

Caching best practices
- After saving Widgets, CMS Blocks/Pages, or Configuration: go to System > Tools > Cache Management, Select All, choose Actions: Refresh, and Submit. Avoid “Flush Cache Storage” in production unless strictly necessary.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Legal-reviewed content for each policy (Privacy Policy, Terms, Cookie Policy, and any regional pages)
- An admin user with permissions to manage Content, Stores Configuration, and Widgets
- Knowledge of your active theme and footer containers (or ability to consult your developer)
- If you use a Consent Management Platform (CMP), access to CMP settings or your Tag Manager
- Legal note: This guide is for technical setup only and is not legal advice. Work with your legal counsel to determine required disclosures and consent mechanisms for your jurisdictions.
- Version note: This guide applies to Magento Open Source/Adobe Commerce 2.4.x (Luma/Blank). If you use PWA Studio, Hyvä, or a heavily customized frontend, UI labels and behavior may differ.
- Adobe Commerce (Content Staging): To schedule policy changes with an effective date, open the CMS Page, click Schedule New Update, set the start time/date, and publish. Repeat for all localized variants.

## What You'll Accomplish

By following this guide, you will:

- Create compliant legal pages (Privacy Policy, Terms and Conditions, optional Cookie/Do Not Sell pages)
- Require Terms acceptance at checkout and add legal links to your footer
- Enable cookie consent (Cookie Restriction Mode) per website

## Step-by-Step Instructions

### Step 1: Plan scope and gather content

Decide which store views and regions need distinct policies. Confirm which websites will require cookie consent and checkout terms. Gather approved text for Privacy Policy, Terms and Conditions (you may label the CMS page “Terms of Service” for branding), and any regional pages (e.g., Cookie Policy, Do Not Sell or Share My Personal Information).




### Step 2: Create the Privacy Policy page

1) In Admin, go to Content > Elements > Pages > Add New Page.
2) Set Page Title: Privacy Policy. Set URL Key: privacy-policy.
3) Choose the correct Store View(s); for multi-language, create one page per store view with translated content and URL Key.
4) Set Status: Enabled.
5) In Content, paste your Privacy Policy. Add headings, links to contact options, and update date. Include essentials:
   - What data you collect, purposes, and lawful basis (where applicable)
   - Cookies/trackers used and how to manage preferences
   - Data sharing, retention, and international transfers
   - Data subject rights and how to submit requests (email/form link)
   - Your company name, address, and contact information
   Optional: In Search Engine Optimization, set Meta Title and Meta Description. Keep the URL Key simple (e.g., privacy-policy).
6) Save.




### Step 3: Create the Terms and Conditions page

1) Go to Content > Elements > Pages > Add New Page.
2) Set Page Title: Terms and Conditions (you can display “Terms of Service” on the page for branding). URL Key: terms-of-service.
3) Assign the correct Store View(s); create localized variants as needed.
4) Status: Enabled.
5) Add your Terms content. Include effective date and clear acceptance language. Include essentials:
   - Order, payment, and billing terms; shipping and delivery expectations
   - Returns, refunds, cancellations, and warranties
   - Limitation of liability and dispute resolution
   - Governing law and contact information
6) Save.




### Step 4: Optional: Create Cookie Policy and Do Not Sell pages

For regions that require them: repeat the CMS page process to add a Cookie Policy (URL Key: cookie-policy) and a Do Not Sell or Share My Personal Information page (URL Key: do-not-sell-or-share). Include mechanisms (form or instructions) for submitting data requests/opt-outs, if applicable. For California (CPRA), include a link labeled “Do Not Sell or Share My Personal Information” or “Your Privacy Choices” on the homepage and any page where you collect personal information. Many merchants implement this as a persistent footer link on all pages for consistency. Reference GPC (Global Privacy Control) if supported.

Option (no CMP): If you don’t have a CMP, you can route opt-out and privacy requests via the built-in Contact Us form. Link your “Do Not Sell or Share” page to the Contact Us page at {{store url='contact'}} and instruct users to select “Privacy Request” in the subject/message. Configure routing at Stores > Configuration > General > Contacts (set Send Emails To to your privacy inbox).

Note: “Terms and Conditions” matches Magento’s checkout agreement terminology; you can continue to brand your CMS page as “Terms of Service” if desired.

Optional (region-specific): Also consider creating an Impressum/Legal Notice page (common in DACH) and ensure a clearly labeled Returns/Revocation Policy page is accessible from the footer.

Tip (US sites): Test placing “Your Privacy Choices” in the header utility bar to increase visibility and reduce support inquiries; keep footer links as the baseline.




### Step 5: Enable and configure checkout Terms and Conditions

Scope summary: The ‘Enable Terms and Conditions’ setting is per Website (Stores > Configuration > Sales > Checkout). Checkout Agreements are global records that you assign to specific Store Views. Create one agreement per language/content variant and assign it to the Store Views that should display it. Rule of thumb: For every Website where ‘Enable Terms and Conditions’ = Yes, make sure at least one enabled agreement is assigned to each of its Store Views.

1) Go to Stores > Configuration > Sales > Checkout.
2) Expand Checkout Options and set Enable Terms and Conditions: Yes. Save Config.
3) Go to Stores > Settings > Terms and Conditions > Add New Condition.
4) Fill Name (e.g., Checkout Terms) and Checkbox Text (plain text, e.g., I agree to the Terms and Conditions). Set Status: Enabled.
5) Assign Store Views (select all store views where this agreement should appear). In Content, paste the full terms text. Set Show Content As: Text or HTML. The agreement content opens in a modal during checkout; set Content Height to 300–400 (pixels) and Sort Order as needed. If desired, include an HTML link within Content to your Terms CMS page, but keep the full terms in Content for clarity and compliance.
6) Save. If you have multiple websites, repeat configuration and create agreements for each set of Store Views as needed.

Notes & Tips
- Third-party checkouts: If you use a one-step checkout or a payment provider’s custom checkout, verify it renders the agreement checkbox and blocks order placement when unchecked.
- Express checkout enforcement: Test PayPal Express, Apple Pay, Google Pay, Braintree, and any one-step checkout to ensure the T&C checkbox is rendered and enforced before place order. If a provider’s flow bypasses Magento’s agreement step, review the extension’s settings or add a pre-submit validation to require agreement acceptance. Coordinate with your vendor as needed.
- Conversion tip: Test checkbox microcopy variants (e.g., “I have read and agree to the Terms and Conditions” vs. shorter versions) and monitor checkout completion rate.




### Step 6: Add footer links using a CMS Page Link widget

1) Go to Content > Elements > Widgets > Add Widget.
2) Type: CMS Page Link. Design Theme: select your active theme. Continue. Important: Widgets are bound to the selected Design Theme. If different Store Views use different themes, create a separate widget per theme and assign the appropriate Store Views to each widget.
3) Set Widget Title (e.g., Privacy Policy Link). Assign to Store Views that should display this link.
4) Under Layout Updates, set Display On: All Pages. For Container, choose 'Footer Links'. If you don't see it, choose 'Page Footer'. Container names vary by theme. If you’re unsure, check your theme’s layout XML for available containers or use the CMS block method in Step 7.
5) In Layout Updates, set Sort Order to control link sequence (lower numbers render first) and group links as desired (e.g., Privacy, Terms, Cookie Policy, Do Not Sell).
6) In Widget Options, select the CMS Page (Privacy Policy) and set Anchor Text (e.g., Privacy Policy). For CPRA, ensure you also create a link labeled “Do Not Sell or Share My Personal Information” or “Your Privacy Choices” pointing to your opt-out page.
7) Save. Then go to System > Tools > Cache Management, Select All > Actions: Refresh, and Submit. Avoid “Flush Cache Storage” in production. If you later change the active theme, revisit Widgets and re-create or reassign the widget to the new theme.

Add a header link (CPRA)
1) Go to Content > Elements > Blocks > Add New Block, and create a small block containing an <a> link labeled “Your Privacy Choices” pointing to {{store url='do-not-sell-or-share'}}.
2) Go to Content > Elements > Widgets > Add Widget. Type: CMS Static Block. Design Theme: your active theme. Continue.
3) Assign the widget to relevant Store Views. Under Layout Updates, set Display On: All Pages and choose a header container (varies by theme, e.g., ‘Header Panel’ or ‘Header’). Set Sort Order for positioning.
4) Select your block in Widget Options. Save.
5) Go to System > Tools > Cache Management and Refresh invalidated caches.

Notes & Tips
- If you do not see an obvious “Footer Links” container, consult your theme’s layout XML to find available containers. As a fallback, use a footer CMS block (see Step 7).
- Note: When switching themes, widgets do not migrate automatically. Create new widgets for the new theme and disable the old ones.
- Optimization: Consider an A/B test for header link vs. footer-only placement and track support ticket volume tagged “privacy.” If header placement reduces tickets, keep it.




### Step 7: Alternative: Add links via a footer CMS static block

If your theme uses a footer CMS block: 1) Go to Content > Elements > Blocks and locate the footer block (often named Footer Links or Footer). 2) Edit the block for the correct Store View and add HTML links to your new pages. 3) Save, then go to System > Tools > Cache Management and Refresh invalidated caches.

Example HTML:
<ul class="footer-links">
  <li><a href="{{store url='privacy-policy'}}">Privacy Policy</a></li>
  <li><a href="{{store url='terms-of-service'}}">Terms and Conditions</a></li>
  <li><a href="{{store url='cookie-policy'}}">Cookie Policy</a></li>
  <li><a href="{{store url='do-not-sell-or-share'}}">Do Not Sell or Share My Personal Information</a></li>
</ul>

Tip: For US-targeted sites, consider adding a small header utility link labeled “Your Privacy Choices” in addition to the footer link to improve visibility and reduce support inquiries. Where possible, A/B test header vs. footer-only placement and monitor privacy-related ticket volume.




### Step 8: Enable cookie consent (Cookie Restriction Mode)

Important: Magento’s Cookie Restriction Mode shows a basic banner and controls Magento’s own cookie behavior only. It does not block third-party scripts (e.g., GA/GA4, Meta Pixel) or provide category-level consent. Use a CMP or tag manager consent conditions to block non-essential scripts until consent. Non-compliance with EU/UK cookie laws can carry significant penalties; consult legal counsel.

Note on scope: At the top-left Scope switcher, choose the target Website. If a field is greyed out, uncheck ‘Use Default’ to edit it for that Website. Repeat for each Website that requires different cookie settings.

1) Go to Stores > Configuration > General > Web.
2) Expand Default Cookie Settings.
3) Set Cookie Restriction Mode: Yes (note scope is typically Website). Only adjust Cookie Lifetime and Path/Domain if you understand the implications. Leave Domain blank to use the current domain unless you have a multi-subdomain strategy. Incorrect values can cause login/session issues.
4) Save Config. Then go to System > Tools > Cache Management and Refresh invalidated cache types (for example, Configuration, Page Cache). Use “Flush Cache Storage” only if strictly necessary.
5) Test in a private window. If your operations require granular consent (e.g., separate categories for Analytics, Advertising, Functional) or automatic script blocking (e.g., via Google Tag Manager), integrate a Consent Management Platform (CMP) and configure it to load or block tags based on consent.
6) In Default Cookie Settings, set Use HTTP Only: Yes to reduce XSS risk for Magento cookies. Ensure your storefront uses HTTPS: go to Stores > Configuration > General > Web > Base URLs (Secure) and set Use Secure URLs in Storefront = Yes and Use Secure URLs in Admin = Yes so cookies are set with the Secure flag. Save Config.
7) Cookie SameSite: Go to Stores > Configuration > General > Web > Default Cookie Settings > Cookie SameSite and set an appropriate value (commonly Lax). If you rely on cross-site flows (e.g., embedded payment or SSO), you may need None (requires HTTPS). Consult your payment provider’s guidance and test checkout thoroughly.

Customization (banner text/translation)
- The banner text is controlled by theme translations. To change it per store view, add a translation in app/design/<Vendor>/<theme>/i18n/<locale>.csv, for example: "We use cookies to make your experience better","<Your custom text>". Then deploy static content and refresh caches via System > Tools > Cache Management. Alternatively, override the template in your theme for full control. Alternatively, add a translation in a language pack at app/i18n/<Vendor>/<language_pack>/<locale>.csv if you manage translations outside the theme.

Notes & Tips
- Google Consent Mode v2: If you use Google Tag Manager, enable Consent Mode v2. Configure default denied states for ad_storage/analytics_storage, and update states on consent. Measure impact by comparing modeled conversions in GA4 and Google Ads conversion value/ROAS before vs. after adoption to quantify ROI while respecting user choices.
- CMP gating examples: Gate Google Analytics/GA4, Meta Pixel, Google Ads/Conversion Linker, Heatmaps (e.g., Hotjar), and marketing automation (e.g., HubSpot) under Analytics/Advertising categories; keep essential Magento cookies always allowed.
- Frontend compatibility: The built-in Cookie Restriction Mode and Checkout Agreements render on the default Magento frontend. If you use PWA Studio, a headless storefront, Hyvä, or a custom one-step checkout, ensure your frontend implements equivalent cookie consent and agreement UI/logic, or integrate a CMP that controls those surfaces.
- Testing: Clear cookies or use a fresh browser profile to test banner appearance per website. After accepting, the banner should no longer show and necessary cookies should be set. Cookie Restriction Mode text/styling is theme-controlled; customize via your theme or i18n translation files.




### Step 9: Scope correctly for multi-language/multi-region stores

1) For each store view (language/region), duplicate policy pages with localized content and assign the appropriate Store View.
2) Create separate Widgets pointing to each localized page and assign to the corresponding Store View.
3) For each website, configure Checkout Terms and Cookie Restriction Mode separately via the scope selector.




### Step 10: Clear caches and test end-to-end

1) Go to System > Tools > Cache Management. Prefer “Select All” > “Refresh” to invalidate and let Magento warm caches. Use “Flush Cache Storage” only if necessary to avoid cache stampedes on production.
2) If you use Varnish or a CDN, purge external caches selectively—ideally only affected URLs (policy pages, footer) rather than a full purge.
3) Open a private/incognito window and visit your storefront. Check the footer links and open each policy page.
4) Add a product to cart and proceed to checkout to verify the Terms checkbox behavior.
5) Confirm cookie banner behavior on a fresh session.
6) Repeat tests for each store view/website.



## Verification

To confirm everything is working correctly:

- Footer links: On each store view, verify links point to localized pages and return HTTP 200.
- Terms & Conditions: At checkout, attempt to place an order without checking the box; it should be blocked with a clear error. Open the agreement link and confirm content displays as configured (Text/HTML) in a modal with the correct height.
- Negative test (agreement): Attempt to place the order with the checkbox unchecked. Expect an inline error such as “You must agree to the terms and conditions before placing the order.” Then check the box and confirm the order proceeds.
- Cookies: In a new session, confirm the cookie banner appears. After acceptance, use your browser’s developer tools (Application/Storage > Cookies) to confirm Magento cookies (e.g., PHPSESSID, section_data_ids) are present and the consent flag cookie user_allowed_save_cookie=1 is set. Before acceptance, this cookie should be absent or 0.
- Accessibility: Verify the cookie banner and agreement modal are keyboard-navigable (Tab/Shift+Tab), the checkbox has an accessible label, focus is trapped inside the modal while open, color contrast meets WCAG, and screen readers announce the banner and modal headings.
- Admin save: Reopen CMS pages and the T&C condition to verify Store View assignments and Status = Enabled.
- If changes don’t appear on production, purge Varnish/CDN caches selectively and retest.

Evidence of acceptance (recommended)
- Out of the box, Magento enforces acceptance but does not persist the agreement text/version to the order record.
- Work with your developer or an extension to store: agreement IDs, version/effective date, checkbox text, and acceptance timestamp on the order (e.g., as order attributes or a status history comment). This creates evidentiary value for disputes and audits and can improve chargeback reversal rates.

KPIs to monitor (post-implementation)
- Checkout abandonment related to agreement errors (should decrease). Track a GA4 event when the T&C checkbox is shown/checked and segment your checkout funnel by this event.
- Number/time-to-resolution of privacy/data-rights tickets (should decrease/improve). If using the Contact form for privacy requests, label and track these in your helpdesk/CRM. Consider A/B testing header vs footer-only “Your Privacy Choices” placement and track resulting ticket volume.
- Opt-in rates by consent category from your CMP and impact on analytics completeness. Fire GA4 events on consent accept/decline and enable Google Consent Mode v2 to preserve modeled conversions while honoring choices; monitor pre/post changes in GA4 modeled conversions and Google Ads conversion value/ROAS.
- Chargeback/dispute rates related to terms visibility (should decrease). Persisting agreement version/timestamp on the order improves dispute evidence and can increase win rates and recovered revenue.

### Completion Checklist
- Policy pages created and enabled per store view (Privacy, Terms, Cookie, Do Not Sell/Share, regional pages as needed).
- Footer links visible and pointing to the correct localized pages on each store view.
- T&C enabled at Website scope; checkout blocks order placement when unchecked; modal shows full terms.
- Cookie banner shows on first visit for each website and disappears after acceptance; scripts gated appropriately.
- Magento cache invalidated (avoid full flush on production where possible); Varnish/CDN caches purged selectively; retested in private windows.
- Tested across each website/store view and any custom or third-party checkout/frontends.

## Common Issues and Solutions

- Footer links not visible: Ensure the widget/block is assigned to the correct Store View and Container, then refresh invalidated caches (System > Tools > Cache Management). See Step 6 (Widget Store Views/Container) or Step 7 (Footer CMS block).
- T&C checkbox not showing: Set Stores > Configuration > Sales > Checkout > Enable Terms and Conditions = Yes at the Website scope (Step 5, item 2), and confirm the agreement Status = Enabled and assigned to the correct Store Views (Step 5, item 5).
- Cookie banner not showing: Verify Cookie Restriction Mode = Yes at the Website scope (Step 8, item 3), clear browser cookies, and test again. Check that JavaScript bundling/minification is not causing errors (view browser console).
- 404 on policy page: Confirm the page is assigned to the current Store View and the URL Key matches the link (Steps 2–4). If using widgets, confirm the widget’s Design Theme matches the storefront theme (Step 6, item 2).

## Related Resources

- [Adobe Commerce and Magento Open Source User Guide](https://experienceleague.adobe.com/docs/commerce-admin/user-guides/home.html)
- [Checkout Agreements](https://experienceleague.adobe.com/docs/commerce-admin/stores-sales/checkout/checkout-agreements.html)
- [Default Cookie Settings (Cookie Restriction Mode)](https://experienceleague.adobe.com/docs/commerce-admin/config/general/web.html#default-cookie-settings)

---

Need help? Contact Adobe Commerce Support at https://experienceleague.adobe.com/?support-tab=support#support or see the Commerce Knowledge Base at https://experienceleague.adobe.com/docs/commerce-knowledge-base/kb/overview.html.
