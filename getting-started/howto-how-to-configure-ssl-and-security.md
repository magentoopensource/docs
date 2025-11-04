---
title: "How to Configure SSL and Security"
description: "Set up SSL certificates and basic security measures for customer trust"
type: how-to
audience:
  - "Magento merchants"
  - "Advanced users"
difficulty: Advanced
estimated_time: "45–90 minutes (depends on hosting, CDN, and number of stores)"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Basic PHP knowledge"
  - "Magento development environment"
tags:
  - how-to
  - customer-management
  - security
  - high-impact
  - complex
last_updated: "2025-09-12T12:39:43.749Z"
---

# How to Configure SSL and Security

## Overview

Secure your Magento 2 store with SSL and baseline security controls to protect customer data, improve conversion rates,
and meet payment and privacy expectations. This how-to shows new store owners and growing merchants how to configure
HTTPS, enforce secure cookies, and enable core protections (2FA and reCAPTCHA) so customers see a trusted padlock and
checkout completes without security warnings.

This guide applies to Magento Open Source and Adobe Commerce 2.4.x. HTTPS removes browser security warnings, improves
trust, and is a Google ranking signal. Strong security reduces fraud and chargebacks and can increase checkout
completion rates. Merchants typically see higher checkout completion when browsers show a valid padlock and no mixed
content. 

Enabling reCAPTCHA reduces bot signups and fake orders; enforcing Admin 2FA reduces risk of account takeover
and fraudulent configuration changes. Fewer abandoned checkouts and blocked bot traffic can lift conversion and reduce
chargebacks/support costs. Even a 0.5–1% checkout lift is material for most stores. Example ROI: If your monthly revenue
is $250,000 and conversion increases 0.7%, estimated lift ≈ $1,750/month. If bot signups drop and save 10 agent
hours/month at $30/hour, add ≈ $300/month in support savings.

## Prerequisites

Before you begin, make sure you have:

### Technical access

- Access to Magento Admin
- Server/hosting control panel access or ability to request changes from your hosting support
- Ability to run CLI commands (SSH) or coordinate with your developer/host
- CDN access (if applicable)

### Operational readiness

- Schedule a low-traffic maintenance window
- Back up your database and app/etc/env.php and app/etc/config.php
- Ensure you have an Admin super user and SSH access
- Have a rollback plan (restore base URLs and revert server/CDN redirect rules)

## What You'll Accomplish

By following this guide, you will:

- Configure HTTPS, redirects, and secure cookies across your store
- Enable Admin 2FA and Google reCAPTCHA to block bots and account takeovers
- Verify a clean padlock (no mixed content) so checkout is trusted

## Step-by-Step Instructions

### Quick Start (Most Stores)

- Step 0–2: Install a valid SSL certificate and verify https:// loads without errors
- Step 3: Enforce a single HTTP→HTTPS redirect at your CDN or web server (not both)
- Step 4: Update Magento Base URLs to https and set Use Secure URLs (frontend/admin)
- Step 5: Enable secure cookies, set Use SID on Frontend = No, and enable session validations
- Step 6–7: Require Admin 2FA and enable Google reCAPTCHA (test checkout thoroughly)
- Step 8: Flush Magento caches and purge Varnish/Fastly/Cloudflare caches
- Step 11: Verify padlock, no mixed content, webhooks, and all payments/shipping methods

### Step 0: Decide your SSL approach

Choose one:

1. Hosting-managed SSL (AutoSSL/Let’s Encrypt) via your control panel or support ticket.
2. Let’s Encrypt with Certbot if you have server access.
3. Commercial SSL from a CA if you require organization validation (OV/EV) for compliance or procurement.

Note: Most modern browsers do not prominently display EV/OV organization names in the address bar. If using a CDN (e.g.,
Cloudflare), set SSL/TLS mode to Full (strict) and ensure the origin also has a valid certificate.

Warning: Do not use Cloudflare "Flexible" SSL. It can cause redirect loops, mixed content, and checkout failures,
harming SEO and revenue.

### Step 1: Install the SSL certificate on your web server

On managed hosting, enable SSL in your panel or ask support. For Let’s Encrypt, run Certbot appropriate for your stack (
Apache/Nginx) and domain; ensure auto-renew is configured. For a commercial cert, install the certificate, key, and
intermediate chain per your server’s documentation; restart the web server when done.

TLS best practices: Allow only TLS 1.2 and 1.3; disable TLS 1.0/1.1. Prefer modern ECDHE + AEAD ciphers. Test your site
at https://www.ssllabs.com/ssltest/ and aim for an A grade. Set an automated expiry monitor (for example, a cron that
runs certbot renew --dry-run with alerts, or an external uptime/expiry monitor) so certificates renew before expiry.

### Step 2: Verify the certificate works at your domain

Open https://yourdomain in a browser and check the padlock details. Alternatively, run: curl -I https://yourdomain and
confirm a 200/301 response. If you see trust errors, reinstall the intermediate chain or correct the domain/SAN.

### Step 3: Enforce HTTP to HTTPS redirects at the edge or origin

Decide where to enforce redirects: If using a CDN (Cloudflare, Fastly), enable the CDN’s HTTPS redirect and do not add
origin redirects. If not using a CDN, add a single redirect at your web server. Do not enable redirects in both places.

- Do: Enable the HTTPS redirect in Cloudflare OR add a single redirect rule in Nginx/Apache (not both).
- Example (Nginx server block):

```
if ($scheme = http) { return 301 https://$host$request_uri; }
```

- Example (Apache .htaccess — direct origin, no proxy):

```
RewriteCond %{HTTPS} !=on
RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
```

- Example (Apache .htaccess — behind proxy/CDN that sets X-Forwarded-Proto):

```
RewriteCond %{HTTP:X-Forwarded-Proto} !https
RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
```

Do not combine both conditions.

In Magento, also go to Stores > Configuration > General > Web > Url Options and set Auto-redirect to Base URL. If you
already enforce redirects at the CDN or web server, set Auto-redirect to Base URL = No to avoid duplicate redirects.
After final verification, switch to Yes (301 Moved Permanently) only if Magento must enforce the canonical host.

Cloudflare users: Never use Flexible SSL. Use Full (strict) with a valid origin certificate to prevent redirect loops
and checkout/SEO issues.

After one week of error-free operation, convert any temporary 302 redirects to 301 (permanent) at your CDN/web server to
consolidate SEO signals.

### Step 4: Update Magento base URLs to HTTPS

Admin path: Stores > Configuration > General > Web

Before editing, in the Configuration page (top-left), set Scope to the target Website (not Default Config). Click OK
when prompted to change scope.

Do this as sub-steps:

1) In Configuration (upper-left), change Scope to the target Website (not Default Config) and confirm.
2) Edit Base URLs and Base URLs (Secure) for that Website only. Repeat for each Website/domain.

In Magento Admin, go to Stores > Configuration > General > Web and make these changes:

- Base URLs: Base URL = https://yourdomain/
- Base URLs (Secure): Base URL = https://yourdomain/
- Base URLs (Secure): Use Secure URLs on Storefront = Yes
- Base URLs (Secure): Use Secure URLs in Admin = Yes
- If you previously customized Base URL for Static View Files or Base URL for User Media, mirror them under Base URLs (
  Secure) with https. Otherwise, leave them blank to use defaults.
- If TLS terminates at a load balancer/CDN (Fastly, Cloudflare, Nginx proxy), ensure it forwards X-Forwarded-Proto:
  https. Set Base URLs (Secure) > Offloader header = X-Forwarded-Proto (or your header name). Only set Offloader header
  if TLS terminates before Magento (CDN/load balancer/proxy); use X-Forwarded-Proto unless your provider documents a
  different header.

Save.

If you can’t access Admin, use CLI:

```
bin/magento config:set web/unsecure/base_url https://yourdomain/
bin/magento config:set web/secure/base_url https://yourdomain/
bin/magento config:set web/secure/use_in_frontend 1
bin/magento config:set web/secure/use_in_adminhtml 1
bin/magento config:set web/secure/offloader_header X-Forwarded-Proto
bin/magento cache:flush
```

CLI audit and fix (if previously customized):

- Check:

```
bin/magento config:show web/unsecure/base_static_url
bin/magento config:show web/unsecure/base_media_url
bin/magento config:show web/secure/base_static_url
bin/magento config:show web/secure/base_media_url
```

- If set, update secure values to https or clear to use defaults:

```
bin/magento config:set web/secure/base_static_url https://yourdomain/static/
bin/magento config:set web/secure/base_media_url https://yourdomain/media/
```

(Set unsecure values blank or https as per your policy.)

Multi-site note: Each website with its own domain requires its own SSL certificate, DNS, redirects, and separate
reCAPTCHA keys. Repeat configuration per website/domain to avoid downtime or SEO issues on secondary storefronts.

### Step 5: Enable secure cookies and session protections

Admin path: Stores > Configuration > General > Web > Default Cookie Settings

- Go to Stores > Configuration > General > Web > Default Cookie Settings. Set Use HTTP Only = Yes and Cookie Secure =
  Yes. Set Cookie SameSite according to your integrations: Lax (default) is safe for most sites. Use None (requires
  Cookie Secure = Yes) if you rely on cross-site flows (for example, embedded checkout/iFrames, SSO). Leave Cookie
  Domain blank unless you must scope cookies to a specific domain. If needed, enter your exact domain without protocol (
  for example, example.com). Set Cookie Lifetime as needed (for example, 3600).
- Go to Stores > Configuration > General > Web > URL Options and set Use SID on Frontend = No.
- Go to Stores > Configuration > General > Web > Session Validation Settings and enable appropriate validations (for
  example, Validate HTTP_USER_AGENT = Yes).

Save and flush cache. Test Admin and Storefront logins after changing.

### Step 6: Enforce Two-Factor Authentication for Admin

Admin path: Stores > Configuration > Security > 2FA

In Magento 2.4+, Admin 2FA is required. Go to Stores > Configuration > Security > 2FA. Choose Providers to Use (for
example, Google Authenticator, Email, Duo) and optionally enable Force providers for all users. Ensure transactional
email works before enabling the Email provider to avoid enrollment delays. Require 2FA for all Admin users, then log out
and log in to enroll devices. Ensure all admin users enroll.

### Step 7: Enable Google reCAPTCHA for storefront and Admin

Obtain site and secret keys from Google reCAPTCHA for your domain.

Admin paths:

- Stores > Configuration > Security > Google reCAPTCHA Admin Panel
- Stores > Configuration > Security > Google reCAPTCHA Storefront

Configure v2 or v3 in each section for the relevant forms. For Admin Panel, enable for Admin Login (and other admin
forms as desired). For Storefront, enable for customer login, registration, contact, and checkout forms where
applicable. Explicitly enable protection for Place Order to cover checkout.

Caution: Enabling reCAPTCHA on Place Order may interfere with some payment methods (wallets/APMs like Apple Pay/Google
Pay, certain hosted/redirect flows). Test each payment method end-to-end in staging before enabling in production. If
false positives occur, adjust the score threshold (v3) or disable reCAPTCHA for Place Order and rely on protections on
other checkout forms.

For v3 Invisible, start with Score Threshold 0.5 and adjust based on false positives/negatives. Rule of thumb: If
legitimate customers are challenged or blocked, lower to 0.3; if bots get through, raise to 0.7–0.9. Review reCAPTCHA
admin analytics weekly during rollout.

Use a separate key pair per domain/environment. If using Google reCAPTCHA, disable native CAPTCHA to avoid double
prompts: Stores > Configuration > Security > CAPTCHA. Set Enable CAPTCHA on Frontend = No and (optionally) Admin = No,
depending on your reCAPTCHA coverage. Save and test challenges in a private browser window.

### Step 8: Flush Magento caches and deploy static content if needed

Run bin/magento cache:flush and bin/magento cache:clean. If your site runs in production mode or you changed
theme/static URLs, run bin/magento setup:static-content:deploy for your active locales. In developer mode, this is
usually not required. Verify the site loads styles/scripts over HTTPS.

Check mode:

```
bin/magento deploy:mode:show
```

If production and you changed theme/static URLs:

```
bin/magento setup:static-content:deploy -f en_US <add other locales>
bin/magento cache:flush
```

Edge/CDN cache purge: If you use Varnish or Fastly, purge the cache after switching to HTTPS to prevent cached http
assets.

- Fastly: In Admin go to Stores > Configuration > Advanced > System > Full Page Cache > Fastly Configuration and click
  Purge All (or use your Fastly dashboard).
- Varnish: Issue a PURGE via your hosting panel or Varnish management tool.
- Cloudflare: Perform a “Purge Everything”.

### Step 9: Optional: Change the Admin URL

For security-by-hardening, change the Admin frontName. Use bin/magento setup:config:set
--backend-frontname=your-new-admin and update bookmarks. Communicate the change to authorized admins only.

### Step 10: Optional: Enable HSTS

After confirming everything works over HTTPS, configure your server/CDN to send Strict-Transport-Security with an
initial short max-age (for example, 300). Increase to 15552000 (180 days) after a week of successful operation. Include
preload and subdomains only when fully ready.

Example headers:

- Initial: Strict-Transport-Security: max-age=300
- After a week: Strict-Transport-Security: max-age=15552000; includeSubDomains
- Preload (only when all subdomains are HTTPS): Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

Set via your web server (Apache/Nginx) or your CDN’s security headers feature.

### Step 11: Final end-to-end test

Test homepage, product page, cart, checkout, account login, password reset, and Admin login. Confirm HTTPS padlock,
reCAPTCHA, and 2FA work; watch browser console for mixed content; ensure payment test transactions complete without
security warnings.

Mixed content check and remediation:

- Open browser DevTools Console and look for mixed content warnings.
- In Admin, review Content > Pages and Content > Blocks for http:// links; change to https://, protocol-relative (//),
  or use the {{store url=""}} directive to generate your store’s base URL. Example: <a href="{{store url="contact"}}">
  Contact</a>
- CLI search for http values: bin/magento config:show | grep "http://"
- Database (backup first): SELECT path, scope, scope_id, value FROM core_config_data WHERE value LIKE 'http://%';

Payments & shipping regression tests:

- Test each enabled payment method (credit card gateways, wallets like Apple Pay/Google Pay, PayPal, BNPL) and each
  shipping method end-to-end in sandbox/staging and then production. Verify redirects/return URLs use https and webhooks
  are delivered.

Integrations: Update callback/webhook URLs at your payment provider (for example, PayPal, Adyen, Stripe) and any
third-party services (ERP/CRM, marketing forms) to use https://. Verify webhook deliveries after cutover.

If you use a CDN, purge its cache after switching to HTTPS to avoid serving cached http assets.

SEO follow-up:

- Regenerate and serve your sitemap over HTTPS (Marketing > SEO & Search > Site Map), then submit the HTTPS sitemap URL
  in Google Search Console and Bing Webmaster Tools.
- Verify canonical tags, hreflang, and structured data fetch correctly over HTTPS.

## Verification

Use this checklist to confirm everything is working correctly:

- Storefront
    - Homepage, category, product pages load with a padlock (no mixed content)
    - Customer login, registration, contact forms load and reCAPTCHA triggers as configured
    - Add to cart, cart page, and checkout steps load over HTTPS without warnings
- Checkout
    - Place a test order with a sandbox/test payment method without errors
    - Order confirmation page and emails are delivered correctly
- Admin
    - Admin login requires 2FA and allows enrollment for all users
    - Admin pages load over HTTPS; configuration saves and caches flush without error
- External
    - Webhooks/callbacks from payment provider(s) deliver successfully to https URLs
    - SSL Labs test returns A grade and no legacy TLS versions are enabled
- KPIs to monitor for 14 days
    - Conversion rate and checkout drop-off by step
    - Payment decline/error rates by method; false-positive reCAPTCHA events
    - Bot/signup volume and WAF/captcha challenge counts
    - Organic traffic and indexation (Search Console): crawl errors after HTTPS switch

## Common Issues and Solutions

- Redirect loop after enabling CDN: Ensure Cloudflare SSL is Full (strict) and origin has a valid cert; remove duplicate
  redirects at origin.
- Admin login loop after cookie changes: Clear cookies for your domain; set Cookie Domain blank; set Use SID on
  Storefront = No; flush cache.
- No padlock/mixed content: Search CMS content for http:// assets; update to https and redeploy static content; clear
  caches.
- 2FA lockout (Email provider): Verify transactional email works; use another provider (for example, Google
  Authenticator). In emergency (non-production) you can disable 2FA via CLI: bin/magento module:disable
  Magento_TwoFactorAuth && bin/magento setup:upgrade && bin/magento cache:flush. Re-enable after fixing with:
  bin/magento module:enable Magento_TwoFactorAuth && bin/magento setup:upgrade && bin/magento cache:flush.
