---
title: "How to Implement GDPR Compliance"
description: "Set up data protection features and privacy controls for EU customers"
type: how-to
audience:
  - "Magento merchants"
  - "Advanced users"
difficulty: Advanced
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Basic PHP knowledge"
  - "Magento development environment"
tags:
  - how-to
  - customer-management
  - high-impact
last_updated: "2025-09-11T12:22:38.537Z"
---

# How to Implement GDPR Compliance

## Overview

Implement GDPR controls in Magento 2 to unlock EU sales, build customer trust, and reduce regulatory risk. This how-to
guides new store owners and growing merchants to configure privacy features, cookie consent, and data-subject request
workflows with measurable outcomes.

Supported versions: Magento Open Source and Adobe Commerce 2.4.x. Some settings may differ by version; where relevant,
notes are provided.

Estimated effort: Plan 60–120 minutes for the baseline configuration, plus 30–60 minutes if integrating a Consent
Management Platform (CMP) and tag manager.

Recommended KPIs to track:

- Cookie consent acceptance rate and opt-out rate
- Share of orders with recorded checkout agreement consent
- DSAR (data subject access request) handling time (target: internal 7 days; legal maximum 30 days)
- Accuracy of analytics attribution post-consent (e.g., reduction in “direct/none” after implementing consented
  tracking)

## Prerequisites

Before you begin, make sure you have:

- Required: Access to Magento Admin Panel
- Optional (for Step 4 only): Basic PHP knowledge and a Magento development environment

Tip on configuration scope: Many privacy settings are Website or Store View scoped. Use the scope switcher in the
upper-left of Stores > Configuration to set values at the intended scope (e.g., EU website or store view) before saving.

## What You'll Accomplish

By following this guide, you will:

- Implement GDPR baseline controls in Magento 2 (privacy policy, cookie consent, and consent at key touchpoints)
- Establish DSAR (data subject access request) export and deletion/anonymization procedures
- Reduce compliance risk while maintaining marketing performance

## Step-by-Step Instructions

### Step 0: Create a GDPR-ready Privacy Policy page and footer link

Admin: Content > Elements > Pages > Add New Page. Set Title "Privacy Policy" and URL Key "privacy-policy". Add your
approved policy content. Set Store View(s) to your EU stores. Save and enable. Add a footer link: Content > Elements >
Blocks > search for "Footer Links" (or a similarly named footer block). Edit and add a link using a store-aware
directive, for example: <a href="{{store url='privacy-policy'}}">Privacy Policy</a>. If your theme lacks a footer block,
go to Content > Design > Configuration > [EU Store View] > Footer > Miscellaneous HTML and add the anchor link. Flush
cache (System > Cache Management > Flush Magento Cache).

### Step 1: Enable cookie consent (baseline) or integrate a CMP

Option A — Native banner (baseline; Website scope): Stores > Configuration > General > Web > Default Cookie Settings.
Set Cookie Restriction Mode = Yes at the Website scope for your EU site(s). Optionally adjust Cookie Lifetime to a
minimal necessary duration. Save, then flush cache. Note: Native mode shows a consent banner and blocks Magento cookies
but does not manage third-party scripts.

Option B — CMP integration (recommended if using analytics/ads): Install and configure a CMP to block all non-essential
tags by default and gate third-party scripts via consent.

- Install CMP script: Content > Design > Configuration > [Store View] > HTML Head > Scripts and Style Sheets. Paste your
  CMP script.
- Configure categories: In the CMP, set default-deny for non-essential categories (e.g., analytics, ads,
  personalization) until consent is granted.
- Tag Manager setup: In Google Tag Manager (or equivalent), add a Consent Initialization tag, then set tag firing
  conditions based on CMP consent signals (e.g., analytics_storage, ad_storage). Ensure analytics/ads tags only fire
  when consent is granted.
- Validate: Use GTM Preview/Debug to confirm no marketing tags fire before consent; verify they fire after accepting
  consent.
- Update banner text to link to your Cookie Policy and Privacy Policy.

Business impact: A CMP preserves attribution accuracy and ad optimization by ensuring tracking resumes only after
consent, improving ROAS measurement and reducing wasted ad spend.

### Step 2: Require Privacy/Terms agreement at checkout

Enable agreements (Website scope): Stores > Configuration > Sales > Checkout > Checkout Options > Enable Terms and
Conditions = Yes. Use the scope selector to set this at the correct Website.

Create agreement: Stores > Settings > Terms and Conditions > Add New Condition. Name: "Privacy & Cookies Policy" (
include versioning, e.g., "v2, effective 2025-09-01"). Status: Enabled. Checkbox Text: "I agree to the Privacy & Cookies
Policy". Content: Summarize and link to /privacy-policy and your cookie policy. Applied: Automatically. Assign to
relevant Store Views/languages and create separate agreements per language store view to support localization and audit
trails. Save and clear cache.

Business impact: Clear policy acknowledgment can reduce misunderstandings, cancellations, and chargebacks, lowering
support effort and dispute fees.

### Step 3: Enable newsletter double opt-in

Stores > Configuration > Customers > Newsletter > Subscription Options (Website scope): Set Need to Confirm = Yes.
Optionally allow guest subscriptions as per your policy. Ensure your mail transport and cron are configured so
confirmation emails send.

Verification:

- Subscribe with a test email and confirm receipt of the confirmation email within 5 minutes.
- Click the confirmation link and verify status flips to Subscribed in Customers > All Customers > [Customer] >
  Newsletter tab.
- Ensure cron is running (bin/magento cron:run) and transactional emails are configured (Stores > Configuration >
  Advanced > System > Mail Sending Settings).
- Consider updating the newsletter email templates to reference the Privacy Policy (Marketing > Communications > Email
  Templates).

### Step 4: Add a consent checkbox to the Contact Us form (developer)

Implement via a custom module and layout XML to avoid brittle template overrides. Add a child block/field to the contact
form with a required checkbox labeled with a link to your Privacy Policy (input name example: gdpr_consent). Implement
server-side validation: create a small plugin/observer on Magento\Contact\Controller\Index\Post to check the request
parameter and throw a localized error if not present. Ensure compatibility with reCAPTCHA (Stores > Configuration >
Security > Google reCAPTCHA) and form key validation. Deploy static content and clear cache. Test on staging first.

Consent evidence capture:

- Update the Contact Form email template (Marketing > Communications > Email Templates > Add New Template > Load
  default: Contact Form) to include: "Consent: {{var gdpr_consent ? 'Yes' : 'No'}}" and a timestamp/IP if available.
- Optionally, create a custom table to log submissions with consent value and IP for audit, with a defined retention
  period per your policy.

### Step 5: Set up a DSAR export workflow

For a specific requester: 1) Verify identity via your SOP. 2) Export customer profile: System > Data Transfer > Export >
Entity Type = Customers Main File. Filter by email and export CSV. 3) Export addresses similarly (Entity Type = Customer
Addresses). 4) Export orders if requested: Sales > Orders grid > filter by customer > Export = CSV.

Also consider additional data sources:

- Reviews: Marketing > User Content > Reviews (filter by customer)
- Newsletter status: Customers > All Customers > [Customer] > Newsletter tab
- Quotes/abandoned carts: Sales > Carts > Abandoned Carts (filter by email)
- Wishlists (via Reports or database where applicable)
- Third parties: payment gateways, shipping providers, analytics, email service providers—request exports where
  applicable

Consolidate files, review for third-party data, and securely transmit to the verified requester. Document the request
and response date. Define an SLA: legal maximum 30 days; set an internal target of 7 days. Consider
extensions/automation to reduce handling time and errors.

### Step 6: Set up account deletion/anonymization process

If deletion is requested and permitted by your retention policy: Customers > All Customers > search for the account >
select > Actions > Delete. If deletion is blocked due to order retention rules or customizations, use a
GDPR/anonymization extension to pseudonymize personal fields while preserving order records for accounting.

Before deletion/anonymization:

- Back up the database or export the customer record
- Test the process in staging first
- Document the lawful basis and any retention exceptions (e.g., accounting) before proceeding

Update your SOP to handle exceptions and record actions taken.

### Step 7: Configure data retention and log cleaning

Stores > Configuration > Advanced > System > Log: Enable Log Cleaning = Yes. Set Save Log, Days to your policy (e.g.,
180 days). Save and flush cache. Note: If Advanced > System > Log is not present in your version, skip this setting and
enforce retention via database-level jobs or reporting cleanup. Verify availability under Stores > Configuration >
Advanced > System in your installation.

Ensure cron is configured so log cleaning runs automatically (bin/magento cron:install on server; verify crontab).
Document your retention schedule in your privacy policy and review quarterly. Business impact: Shorter retention reduces
database size, improving report speed and lowering hosting costs.

### Step 8: Reference privacy in transactional emails and remove trackers until consent

Marketing > Communications > Email Templates > Add New Template. Load a default footer template, customize to include a
short privacy note and link to /privacy-policy.

Assign the footer across all transactional emails at Stores > Configuration > General > Design > Transactional Emails >
Footer Template. Use Stores > Configuration > Sales > Sales Emails only when replacing specific sales email body
templates (order, invoice, shipment, etc.).

Ensure any tracking pixels or link decorators that set cookies are disabled unless consent is granted (use your CMP/Tag
Manager conditions) so no trackers load before consent.

## Verification

Use this checklist to confirm everything is working correctly:

1. Privacy Policy link appears in EU store footer and resolves correctly per store view (store-aware URL).
2. New visitor sees cookie banner; analytics/ads do not fire before consent; they load only after consent (validate in
   GTM Preview/Network tab).
3. Checkout shows Terms checkbox; order can’t be placed until checked; agreement text matches the correct
   language/version.
4. Newsletter sign-up sends confirmation email; status updates to Subscribed after clicking the link; cron and mail
   sending settings verified.
5. Contact form blocks submission without consent; received email includes consent=Yes and timestamp (and IP if added).
6. DSAR export includes all relevant entities (customers, addresses, orders, reviews, quotes); excludes other customers’
   data; request is fulfilled within internal SLA.
7. Deletion/anonymization behaves per policy in staging; production backups exist prior to action.
8. Log cleaning job runs on schedule; data older than policy threshold is removed.
9. Transactional emails include a privacy link; no tracking pixels or decorators load without consent.

## Common Issues and Solutions

- Cookie banner not showing: Confirm Cookie Restriction Mode is enabled at the correct Website scope and cache is
  flushed. If using a CMP, ensure the CMP script is added to the correct Store View head and loads without errors.
- Marketing tags firing before consent: Set CMP categories to default-deny and configure a Consent Initialization tag in
  GTM; ensure tags require consent states (e.g., analytics_storage='granted').
- Can’t find Terms and Conditions: Navigate to Stores > Settings > Terms and Conditions (not "Other Settings"). Ensure
  the feature is enabled at Website scope.
- Agreement not visible at checkout: Verify the agreement is Enabled, Applied Automatically, and assigned to the correct
  Store Views/languages.
- Newsletter confirmation not sending: Ensure cron is running and Mail Sending Settings are configured. Check that the
  Need to Confirm option is set at the intended Website scope.
- Contact form errors: Ensure your module validates gdpr_consent server-side, preserves form key validation, and is
  compatible with Google reCAPTCHA.
- DSAR exports missing data: Include reviews, quotes, newsletter status, and third-party systems. Filter exports by
  email to avoid including other customers’ data.
- Log cleaning option missing: Your installation may not include the Magento_Log module; manage retention via
  database-level jobs and verify availability under Advanced > System.
- Email footer not updating: Assign the Footer Template under General > Design > Transactional Emails; clear caches and
  ensure you are not overriding per-email body templates unless intended.
