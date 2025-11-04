---
title: "How to Maintain PCI Compliance"
description: "Ensure your store meets payment card industry security standards"
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
  - payment
  - security
  - high-impact
  - complex
last_updated: "2025-09-12T10:48:44.617Z"
---

# How to Maintain PCI Compliance

## Overview

Maintain PCI compliance to accept cards safely, avoid fines, and increase customer trust. This guide shows a low-effort
path to PCI compliance for Magento stores by outsourcing card data, enforcing HTTPS, hardening Admin, and completing
required scans/SAQs. Choosing a hosted redirect flow (often SAQ A) typically reduces annual PCI questions from hundreds
to about 20–30 and can cut assessment time from weeks to hours—lowering compliance effort and risk exposure.

Merchants switching from on-site card forms to hosted redirect/hosted fields typically reduce annual PCI questionnaire
effort by 70–90% and lower breach exposure from form-jacking. Enabling 3DS2 and network tokenization (when supported by
your gateway) can improve authorization rates by 1–3% in regulated markets and reduce fraud losses, translating to lower
compliance cost, direct revenue lift, and improved margins.

Plan on 90–120 minutes to complete the technical steps and validation (excludes acquirer/QSA coordination).

Version compatibility: Written for Magento 2.4.x. If you’re on an older version, upgrade to 2.4.6+ for built‑in 2FA and
security updates.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Payment provider account
- Valid TLS certificate for all store domains and subdomains used during checkout
- SSH access to the application server (or Cloud project) with permissions to run Composer and the `bin/magento` CLI
- A staging environment (or a maintenance window) for safe deployment and testing

## What You'll Accomplish

By following this guide, you will:

- Reduce your PCI scope by using hosted/tokenized payments
- Enforce HTTPS/TLS 1.2+ across your store
- Harden Admin access (2FA, reCAPTCHA, least privilege)
- Pass security scans and complete the correct SAQ

## Step-by-Step Instructions

### Step 0: Select a hosted or tokenized payment flow to minimize PCI scope

Decide on a gateway and integration type that keeps card data off your servers.

Prefer one of the following integration types:

- Hosted payment page (full redirect)
- Gateway-hosted fields/iframes via the official Magento extension (e.g., PayPal Smart Buttons, Braintree Hosted Fields,
  Adyen Drop-in)

Avoid custom on-site card forms.

Business value: Hosted redirects can qualify for SAQ A, reducing PCI effort substantially (fewer questions, faster
assessments) and lowering breach exposure. Merchants moving to hosted redirect or hosted fields often see 70–90% less
PCI questionnaire effort and fewer form‑jacking risks.

Confirm with your acquirer/QSA which SAQ applies. Full redirect or gateway-hosted payment pages typically qualify for
SAQ A. Hosted fields/JS integrations often require SAQ A-EP even when card data never touches your server.

Verify:

- Confirm your gateway’s Magento extension supports hosted or tokenized flows.
- Record the intended SAQ type after consulting your acquirer/QSA.

### Step 1: Enforce HTTPS everywhere with modern TLS

In Magento Admin, go to Stores > Configuration > General > Web. Under Base URLs (Secure), set Secure Base URL to your
HTTPS URL with a trailing slash (e.g., https://example.com/). Set Use Secure URLs on Storefront = Yes and Use Secure
URLs in Admin = Yes. Save and then flush the Magento cache.

At your web server/CDN, force HTTP→HTTPS redirects and disable TLS 1.0/1.1. Optionally enable HSTS at the edge (start
with a short max-age, use includeSubDomains only if all subdomains support HTTPS; consider preload only after
validation).

If you’re on Adobe Commerce on cloud infrastructure: manage TLS versions, HSTS, and WAF in Fastly. Ensure TLS 1.0/1.1
are disabled in your Fastly service, configure HSTS there, and enable WAF rules. Consider an IP allowlist for /admin via
Fastly WAF. See: https://experienceleague.adobe.com/docs/commerce-cloud-service/user-guide/cdn/fastly.html

Cookie security: In Admin, go to Stores > Configuration > General > Web > Default Cookie Settings and set Use HTTP
Only = Yes. With HTTPS enforced, Magento will set the Secure flag on session cookies.

Test your domain with https://www.ssllabs.com/ssltest/ and remediate any grade lower than A. Ensure only TLS 1.2/1.3 are
enabled and weak ciphers are disabled.

Verify:

- Visiting http URLs redirects to https.
- SSL Labs shows grade A with TLS 1.2/1.3 only.
- Also see the final Verification checklist.

### Step 2: Install and configure the official gateway extension

Install the vendor’s official Magento 2 extension (Marketplace or Composer).

#### Install and deploy

Recommended order:

1) Enable maintenance mode and ensure backups: run `bin/magento maintenance:enable`; create a DB/code backup or snapshot
   and retain your `composer.lock` as a rollback anchor.
2) Composer install (example): `composer require adyen/module-payment:^<latest-major>` — check the gateway’s docs for
   the current supported major version for your Magento version.
3) Upgrade Magento: `bin/magento setup:upgrade`.
4) Check deploy mode with `bin/magento deploy:mode:show`. If production, run `bin/magento setup:di:compile` and
   `bin/magento setup:static-content:deploy -f` (optionally specify locales).
5) Then run: `bin/magento cache:clean`. Use `bin/magento cache:flush` only if specifically required (it clears the
   entire cache storage).
6) Disable maintenance mode with `bin/magento maintenance:disable`.

#### Configure in Admin

1) In Admin: go to Stores > Configuration > Sales > Payment Methods and locate your gateway.
2) Enter API credentials (start with sandbox). Enable tokenization/vault if available. If your gateway uses Magento
   Vault, ensure it is enabled at Stores > Configuration > Sales > Payment Methods > Vault (Enable Vault = Yes). Some
   gateways provide their own “Save card” setting—enable as required.
3) Enable 3D Secure 2.x. Business value: 3DS2 meets EU PSD2 SCA requirements, reduces chargebacks, and with frictionless
   flows can increase approval rates in regulated markets. Enable TRA/low‑value exemptions and trusted beneficiary lists
   where supported to maximize frictionless approvals while staying compliant.
4) Set payment action. Guidance: Authorize Only (best for preorders, backorders, split shipments) reduces refunds on
   out-of-stock items and supports split shipments; Authorize and Capture (best for in‑stock, ship‑now workflows where
   cash flow is prioritized) immediately improves cash flow.
5) Configure gateway webhooks/notifications (e.g., authorized/captured/refunded). Whitelist gateway IPs (if required)
   and configure the exact callback URL provided by your payment extension. Do not use generic REST endpoints. Examples:
   Adyen: `{base_url}/adyen/notifications/notify`; Braintree: `{base_url}/braintree/webhook/receive`; PayPal: configure
   webhooks/IPN in the PayPal dashboard as documented by the PayPal extension.

Verify:

- The payment method appears at checkout in sandbox mode and processes a test authorization.
- Trigger a webhook event (e.g., capture/refund in the gateway dashboard) and confirm Magento updates order status
  automatically.
- Also see the final Verification checklist.

### Step 3: Verify no card data touches your servers

In sandbox, place a test order. Open your browser’s developer tools (Network tab) and confirm card iframes/scripts load
from gateway domains (e.g., *.adyen.com, *.braintreegateway.com, *.paypal.com), not your store’s domain.

On the server, inspect `var/log` and your web server logs. Ensure logs never contain PAN or CVV. If any sensitive data
appears, stop testing, purge logs securely, and remediate the integration immediately.

Ensure webhook signature/secret validation is enabled in your gateway settings (e.g., Adyen HMAC signatures, Stripe
signing secret, Braintree webhook signature). In Admin > Stores > Configuration > Sales > Payment
Methods > [Your Gateway], set Debug/Logging = No in production. Verify that only masked PAN and tokens appear anywhere.

Database verification (tokenization):

- Run: `SELECT entity_id, payment_method_code, public_hash, type, token_details FROM vault_payment_token LIMIT 5;`
  Verify `token_details` contains masked card data (e.g., ****1111) and no PAN/CVV.
- If your gateway links tokens per order, also check: `SELECT * FROM vault_payment_token_order_payment_link LIMIT 5;`
  Ensure only token references are present.

Verify:

- No PAN/CVV present in application or server logs.
- Only tokens/masked data are stored; no primary account numbers are saved anywhere.
- Also see the final Verification checklist.

### Step 4: Harden Magento Admin and checkout

- Enforce 2FA (Magento 2.4+): Go to Stores > Configuration > Security > 2FA and enable the providers you support (e.g.,
  Google Authenticator, Duo Security, WebAuthn (security keys)). Test by logging in with a non‑enrolled admin account to
  confirm 2FA enrollment is enforced. If a user is locked out or enrollment must be reset, use:
  `bin/magento twofactorauth:reset <username>`.
- Change the Admin URL: Stores > Configuration > Advanced > Admin, set a custom Admin path.
- Configure least-privilege roles: System > Permissions > User Roles.
- Enable Google reCAPTCHA separately for Admin and Storefront:
    - Stores > Configuration > Security > Google reCAPTCHA Admin Panel
    - Stores > Configuration > Security > Google reCAPTCHA Storefront
      Provide separate keys and settings. Use reCAPTCHA v3 on checkout to minimize friction, but start with a moderate
      threshold (e.g., 0.3–0.5) and monitor conversion. If legitimate customers are challenged, lower the threshold or
      scope reCAPTCHA to specific checkout actions.
- Strengthen password and session security: Stores > Configuration > Advanced > Admin > Security. Set a strong password
  policy (min length and complexity), set Admin Account Sharing = No, set an appropriate Session Lifetime, and ensure
  Add Secret Key to URLs = Yes.
- Add a Content Security Policy (CSP) to restrict scripts to trusted domains (your store, CDN, payment gateway). If
  using a CDN/WAF, set a script-src policy that includes only your domain, CDN, and gateway (e.g., *.adyen.com, *
  .paypal.com). Test in report‑only mode first, then enforce. This materially reduces risk of card‑skimming malware and
  is especially important for SAQ A‑EP.
- Consider IP allowlisting for Admin via your firewall/CDN.

Verify:

- A new-device admin login requires a second factor.
- reCAPTCHA behaves as configured (low friction on checkout; challenges on high-risk forms).
- Password policy and session controls are enforced, and the Admin URL has changed.
- Also see the final Verification checklist.

### Step 5: Keep Magento and extensions updated; run Adobe Security Scan

Apply the latest Magento security patches and extension updates in staging, then production, following your
change-control process.

Sign in to your Magento/Adobe Commerce account and register your site in the Security Scan
tool: https://experienceleague.adobe.com/docs/commerce-operations/security/security-scan.html. Follow the “Register your
site” steps to verify ownership and schedule recurring (e.g., weekly) scans. Review findings and remediate promptly.

Verify:

- Adobe Commerce Security Scan shows no critical/high issues and scans are scheduled to run automatically.
- Also see the final Verification checklist.

### Step 6: Schedule quarterly ASV scans and complete the correct SAQ

Most acquirers require quarterly external ASV scans for any merchant with Internet-facing domains in scope. Confirm
scope and targets with your acquirer/QSA to ensure you scan all required hosts (e.g., www, admin subdomains, APIs).

Use PCI SSC SAQ instructions to determine your SAQ type (A, A-EP, D, etc.). Complete the SAQ in your acquirer’s portal (
or approved platform) and retain the Attestation of Compliance (AOC).

Examples (confirm with your acquirer/QSA):

- PayPal Smart Buttons with full redirect → typically SAQ A
- Adyen/Braintree hosted fields (card entry iframe on your site) → typically SAQ A‑EP
- Custom on-site card forms or direct post → SAQ D for merchants

Operational value: Correctly scoping to SAQ A (when eligible) often eliminates internal network controls from the
questionnaire and can cut annual compliance preparation from several weeks to a few hours.

Verify:

- Latest ASV scan passed for all in-scope hosts and is scheduled quarterly.
- Current SAQ and AOC are completed and stored in your compliance repository.
- Also see the final Verification checklist.

### Step 7: Document and operationalize compliance

Create a simple compliance runbook that includes:

- Gateway AOC on file, SAQ/ASV schedule, update cadence, key contacts, change control, and incident response steps.
- Quarterly access review of admin users/roles and monthly log review procedures.
- Backup/restore testing schedule and annual security training tracking.
- Credential hygiene: rotate API keys and admin credentials regularly.
- Ownership and cadence (RACI-style): Security scan review (weekly, SecOps), SAQ/AOC update (annual, Compliance Lead),
  access review (quarterly, IT), patching (monthly, DevOps).
- Incident response: Immediately isolate affected systems (take checkout offline), preserve logs and evidence, notify
  your acquirer within 24 hours, engage a PCI-approved forensic investigator (PFI) if directed, rotate all credentials
  and API keys, and communicate with customers per legal counsel. Assign a single incident commander.

Train staff annually and after role changes.

Verify:

- The runbook exists, owners are assigned, and tasks have scheduled reminders.
- Also see the final Verification checklist.

## Verification

Use this checklist to confirm everything is working correctly:

1) Step 0: SAQ type confirmed with acquirer/QSA and documented.
2) Step 1: SSL Labs reports grade A; only TLS 1.2/1.3 enabled; HTTP→HTTPS redirects enforced; HSTS tested safely;
   HTTPOnly cookies enabled.
3) Step 2: Sandbox order succeeds; webhooks update order status automatically; 3DS2 flows work as expected; payment
   action aligns with operations.
4) Step 3: Browser shows payment fields from gateway domain; database stores only tokens/masked data; no PAN/CVV in
   logs; webhook signatures validated; debug logs disabled in production.
5) Step 4: 2FA enforced on admin login from a new device; reCAPTCHA tuned for low checkout friction; password/session
   policies applied; Admin URL changed and (if used) IP allowlisting active; CSP enforced after report-only testing.
6) Step 5: Adobe Commerce Security Scan shows no critical/high issues; recurring scans scheduled.
7) Step 6: Quarterly ASV scan passed for all in-scope hosts; correct SAQ completed; AOC stored; SAQ scope validated with
   acquirer/QSA.
8) Step 7: Compliance runbook maintained with owners and cadence; training records up to date; incident response plan
   documented.

## Common Issues and Solutions

- reCAPTCHA causes checkout friction: Use v3 on checkout and v2 only on high-risk forms; tune score thresholds (start
  around 0.3–0.5) and monitor conversion.
- 3DS2 challenges reduce conversions: Enable frictionless flow where supported and apply SCA exemptions (e.g., trusted
  devices, low-risk/TRA, low-value) if your gateway supports them.
- ASV scan fails due to TLS: Disable TLS 1.0/1.1 on load balancer/web server; prefer modern ECDHE ciphers; re-run scan.
- Security Scan flags admin exposure: Restrict the /admin path by IP at WAF/CDN and rename the Admin URL.
- Logs contain sensitive data: Disable debug logging for payment modules in production; rotate and purge logs securely;
  investigate and remediate any leakage immediately; verify webhook signature validation is enabled.
